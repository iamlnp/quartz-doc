
# 1 Active-Active  
Lattice 的核心设计目标是 **Active-Active元数据**：
- 任意数量的 MDS 守护进程可以同时为同一文件系统提供服务
- 没有领导者（leader）
- 没有全局锁服务
- 没有故障转移事件

```
                    ┌─────────────────────────────┐
                    │   NFS 客户端 (Linux pNFS)    │
                    └──────────┬──────────────────┘
                               │ NFSv4.1/4.2
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
        ┌──────────┐    ┌──────────┐    ┌──────────┐
        │ MDS 1    │    │ MDS 2    │    │ MDS N    │
        │ 拥有      │    │ 拥有      │    │ 拥有      │
        │ /shard1  │    │ /shard2  │    │ /shardN  │
        └────┬─────┘    └────┬─────┘    └────┬─────┘
             │               │               │
             └───────────────┼───────────────┘
                             ▼
                    ┌─────────────────┐
                    │  RonDB 共享目录   │
                    │  (事务型 KV 存储)  │
                    └─────────────────┘
```

命名空间通过 **子树分区（subtree partitioning）** 被切分为多个分片，每个 MDS 拥有一个 独立的子树。客户端看到的是**一个统一的文件系统**，但内部每个分片由不同的 MDS 管理。  
##### 1.1.1.1.1 分区命名空间  
###### 1.1.1.1.1.1 子树映射（Subtree Map）
核心数据结构是 `subtree_map`（`include/subtree_map.h`），维护命名空间的所有权分区：

```c
struct subtree_entry {
    char                path[MDS_MAX_PATH];    // 子树路径，如 "/shard1"
    uint32_t            owner_mds_id;          // 拥有该子树的 MDS ID
    uint64_t            version;               // CAS 版本号
    enum subtree_state  state;                 // ACTIVE / MIGRATING / FROZEN
    _Atomic uint64_t    op_count;              // 操作计数（用于负载均衡）
    uint64_t            root_fileid;           // 子树根目录的 fileid
    uint32_t            partition_id;          // 目录分区行 ID
    bool                pm_backed;             // 是否持久化到后端
};
```

**子树状态机**：

| 状态                  | 说明              |
| ------------------- | --------------- |
| `SUBTREE_ACTIVE`    | 正常服务状态          |
| `SUBTREE_MIGRATING` | 正在迁移到其他 MDS     |
| `SUBTREE_FROZEN`    | 已冻结，拒绝新操作（迁移准备） |

###### 1.1.1.1.1.2 分片播种（Shard Seeding）
集群启动时，`subtree_map_seed_shards()` 在命名空间根目录下创建分片目录：

```
/               ← 根目录（所有 MDS 共享服务）
├── shard1/     ← MDS 1 拥有（junction 目录，sticky bit 标记）
│   ├── file_a
│   └── dir_b/
├── shard2/     ← MDS 2 拥有（junction 目录）
│   └── ...
└── shardN/     ← MDS N 拥有（junction 目录）
    └── ...
```

分片行同时被持久化到 RonDB 的分区映射表中（`mds_cluster_partition_put`），确保重启后 恢复。播种过程是幂等的：
- 已存在的内存条目保持不变
- 使用 upsert 写入后端（因为初始分片布局无所有者竞争）
- 根目录 "/" 使用 insert-only 声明（只有一个节点能成功）

###### 1.1.1.1.1.3 Junction 目录（交叉点目录）
每个分片根目录是一个 **junction 目录**，用 **sticky bit（`S_ISVTX`，01000）** 作为 带内标记：

```c
// src/fsal_obj/referral.c
int referral_is_junction(struct mds_catalogue *cat, uint64_t fileid)
{
    struct mds_inode inode;
    mds_cat_ns_getattr(cat, fileid, &inode);
    
    if (inode.type != MDS_FTYPE_DIR)
        return 0;
    
    return (inode.mode & S_ISVTX) ? 1 : 0;  // sticky bit = junction 标记
}
```

**设计优势：**
- 无需额外的数据库表或元数据字段
- Sticky bit 是目录的合法属性，普通用户不会设置
- 安全的带内信号

创建 junction 目录：
```c
enum mds_status referral_create_junction(struct mds_catalogue *cat,
                                         uint64_t parent_fileid,
                                         const char *name,
                                         uint32_t target_mds_id)
{
    // 1. 创建普通目录
    mds_cat_ns_create(cat, NULL, parent_fileid, name,
                      MDS_FTYPE_DIR, 0777, 0, 0, NULL, &out);
    
    // 2. 设置 sticky bit 标记为 junction
    out.mode = 0777 | S_ISVTX;
    mds_cat_ns_setattr(cat, NULL, out.fileid, &out, MDS_ATTR_MODE);
}
```

`referral_ensure_junction()` 提供更健壮的语义：如果普通目录已存在，会升级为 junction 而不是返回 `MDS_ERR_EXISTS`。  
##### 1.1.1.1.2 fs_locations Referral 机制  
###### 1.1.1.1.2.1 NFSv4.1 协议基础  
根据 RFC 8881 §8.4.3 和 §11.12，NFSv4.1 定义了 **文件系统引用（fs_locations）** 机制：
`FATTR4_FS_LOCATIONS`（属性 24）结构：
```
fs_locations4 {
    pathname4       fs_root;      // 当前文件系统在引用服务器中的路径
    fs_location4    locations<>;  // 目标服务器列表
};

fs_location4 {
    utf8str_cis     server<>;     // 目标服务器地址（IP 或主机名）
    pathname4       rootpath;     // 目标服务器上的路径
};
```

当客户端遇到 referral 目录时，服务器返回 `NFS4ERR_MOVED`，告知客户端需要"跨越"到 另一个文件系统实例。  
###### 1.1.1.1.2.2 Lattice 实现流程  
**完整请求路径：**
```
客户端                         MDS 1 (非拥有者)           MDS 2 (拥有者)
  │                                │                        │
  │── LOOKUP /shard2 ─────────────>│                        │
  │                                │── 检测 junction ──>    │
  │                                │   (sticky bit 已设置)   │
  │                                │                        │
  │<── NFS4ERR_MOVED ─────────────│                        │
  │    + fs_locations:             │                        │
  │      server: "mds2-host"       │                        │
  │      rootpath: "/shard2"       │                        │
  │                                │                        │
  │── 跨越 referral ──────────────────────────────────────>│
  │   (新的 SETCLIENTID/PUTFH)     │                        │
  │                                │                        │
  │<── 正常响应 ──────────────────────────────────────────│
  │    (MDS 2 直接服务)             │                        │
```

###### 1.1.1.1.2.3 FSID 隔离
每个分片导出为独立的文件系统（不同的 FSID），使客户端将其视为不同的超级块：

```c
// include/referral.h
static inline uint64_t referral_fsid_major(uint32_t owner_mds_id) {
    return (owner_mds_id == 0) ? 1 : (uint64_t)owner_mds_id;
}

static inline uint64_t referral_fsid_minor(uint32_t owner_mds_id) {
    (void)owner_mds_id;
    return 0;
}
```

**FSID 分配规则：**

|分片|owner_mds_id|FSID major|FSID minor|
|---|---|---|---|
| `/shard1` |1|1|0|
| `/shard2` |2|2|0|
| `/shardN` |N|N|0|

客户端通过 FSID 区分不同的文件系统实例，确保跨分片访问被视为跨越文件系统边界。  

##### 1.1.1.1.3 Referral 拓扑强制（Referral Topology Enforcement）  
###### 1.1.1.1.3.1 问题背景  
由于所有 MDS 共享同一个 RonDB 目录，**任何 MDS 都能服务任何 fileid**。这意味着：
- 客户端可能持有缓存的 FH（file handle）
- 在 referral submount 过期后，客户端继续对错误的 MDS 发起操作
- 绕过了 referral 拓扑，导致：
    - 负载不均衡（wrong-MDS load）
    - `nosharecache` 跨实例可见性竞争

###### 1.1.1.1.3.2 referral_strict 机制  
当 `referral_strict = true`（默认开启）时，compound 调度器检查每个操作的 `current_fh`：

```c
// src/mds/compound.c:1234-1264
if (cd->cfg_referral_strict && cd->current_fh_set &&
    cd->current_fh.owner_mds_id != 0 &&
    cd->current_fh.owner_mds_id != cd->mds_id) {
    
    // FH 属于其他 MDS → 准备拒绝
    bool fh_moved = true;

    switch (op->opnum) {
    // 这些操作放行（协议基础设施操作）
    case OP_SEQUENCE:
    case OP_EXCHANGE_ID:
    case OP_CREATE_SESSION:
    case OP_DESTROY_SESSION:
    case OP_DESTROY_CLIENTID:
    case OP_BIND_CONN_TO_SESSION:
    case OP_RECLAIM_COMPLETE:
    case OP_PUTFH:
    case OP_PUTROOTFH:
    case OP_PUTPUBFH:
    case OP_GETFH:
    case OP_SAVEFH:
    case OP_RESTOREFH:
    case OP_SECINFO:
    case OP_SECINFO_NO_NAME:
    case OP_RENEW:
    case OP_RELEASE_LOCKOWNER:
        fh_moved = false;
        break;
        
    case OP_GETATTR:
        // referral 发现放行（RFC 8881 §8.5.1）
        if (nfs4_bitmap_test(op->arg.getattr.requested,
                             FATTR4_FS_LOCATIONS))
            fh_moved = false;
        break;
        
    default:
        break;  // 其他操作 → NFS4ERR_MOVED
    }

    if (fh_moved) {
        atomic_fetch_add_explicit(&g_branch_metrics.nfs_moved, 1,
                                  memory_order_relaxed);
        return NFS4ERR_MOVED;  // 强制客户端重新走路径
    }
}
```

**效果：**

如果客户端试图通过 MDS 1 操作 `/shard2/file`（属于 MDS 2），MDS 1 返回 `NFS4ERR_MOVED`，迫使客户端重新 LOOKUP 路径并跟随 junction referral 到 MDS 2。  

**例外放行：**

| 操作类型                             | 放行原因                            |
| -------------------------------- | ------------------------------- |
| FH/会话管理（PUTFH、GETFH、SEQUENCE 等）  | 协议基础设施，不涉及数据操作                  |
| GETATTR 请求 `FATTR4_FS_LOCATIONS` | RFC 8881 §8.5.1 规定的 referral 发现 |
| `owner_mds_id == 0`              |                                 |

##### 1.1.1.1.4 路径解析（Fileid → Path）  
当 MDS 收到一个 FH 但不确定其所属子树时，`referral_resolve_path()` 通过 `parent_fileid` 链向上追溯到根目录，重建绝对路径：

```c
// src/fsal_obj/referral.c:260-325
enum mds_status referral_resolve_path(struct mds_catalogue *cat,
                                      uint64_t fileid,
                                      char *path_buf, size_t buf_len)
{
    char components[REFERRAL_PATH_MAX_DEPTH][256];
    int depth = 0;
    uint64_t cur = fileid;

    // 从 fileid 开始，逐级向上查找 parent_fileid
    while (cur != MDS_FILEID_ROOT && depth < REFERRAL_PATH_MAX_DEPTH) {
        struct mds_inode inode;
        mds_cat_ns_getattr(cat, cur, &inode);
        
        if (inode.parent_fileid == 0 || inode.parent_fileid == cur)
            break;

        // 在每一级通过 readdir 找到子目录的名称
        struct find_name_arg fna = { .target = cur, .found = false };
        mds_cat_ns_readdir(cat, inode.parent_fileid,
                           NULL, 0, NULL, find_name_cb, &fna);
        
        snprintf(components[depth], sizeof(components[depth]),
                 "%s", fna.name);
        depth++;
        cur = inode.parent_fileid;
    }

    // 深度限制检查
    if (cur != MDS_FILEID_ROOT && depth >= REFERRAL_PATH_MAX_DEPTH)
        return MDS_ERR_INVAL;  // 拒绝截断路径

    // 拼接绝对路径
    size_t off = 0;
    for (int i = depth - 1; i >= 0; i--) {
        int n = snprintf(path_buf + off, buf_len - off,
                         "/%s", components[i]);
        off += n;
    }
    return MDS_OK;
}
```

**关键设计：**

- 最大深度 32 级（`REFERRAL_PATH_MAX_DEPTH`），占用 8 KiB 栈空间
- Junction 目录通常在根目录下的一级，因此 32 级足够
- **绝不返回截断路径**：如果深度超限，返回错误而不是部分路径（避免客户端跟随错误的 fs_locations）

**使用场景：**

即使客户端只提供了 opaque FH（没有路径），MDS 也能确定该文件属于哪个分片，从而正确 执行 referral 拓扑强制。

##### 1.1.1.1.5 配置项  
| 配置键                       | 类型      | 默认值   | 说明                                               |
| ------------------------- | ------- | ----- | ------------------------------------------------ |
| `referral_strict`         | bool    | true  | 强制 referral 拓扑：拒绝非拥有者 MDS 上的操作（返回 NFS4ERR_MOVED） |
| `hide_referral_junctions` | bool    | false | 在 READDIR 中隐藏 /shardN junction 目录（仅影响根目录列表的美观性）  |
| `cluster_size`            | uint32  | 1     | MDS 节点数（>1 时自动启用分片和 referral）                    |
| `shard_enabled`           | bool    | false | 分片主开关                                            |
| `auto_split_*`            | various | -     | 自动拆分相关参数（见 config-keys.md）                       |
| **配置示例：**                 |         |       |                                                  |
```ini
# /etc/lattice/mds.conf

# 启用 3 节点 active-active 集群
cluster_size = 3
shard_enabled = true

# 强制 referral 拓扑（推荐保持开启）
referral_strict = true

# 隐藏 /shardN 目录（可选，美化根目录列表）
hide_referral_junctions = false
```

# 2 子树迁移（Subtree Migration）
当需要重新平衡负载时，Lattice 支持在线迁移子树所有权：

```
状态机:  ACTIVE → MIGRATING → FROZEN → (数据传输) → ACTIVE (新 owner)
```

**迁移流程：**

|阶段|操作|说明|
|---|---|---|
|1. Freeze|源 MDS 冻结子树|设置 `state = SUBTREE_FROZEN`，阻止新操作|
|2. Stream|数据传输|源 MDS 将子树数据流式传输到目标 MDS|
|3. Commit|更新所有权|通过 CAS 更新分区映射的 `owner_mds_id` |
|4. Unfreeze|激活子树|目标 MDS 设置 `state = SUBTREE_ACTIVE` |
|5. Junction|创建 referral|源 MDS 创建/更新 junction 目录，指向目标 MDS|

**迁移后的 referral 更新：**

迁移完成后，源 MDS 的 junction 目录自动将后续访问 referral 到新的拥有者。客户端无需 重新挂载，只需跟随新的 referral 即可透明地切换到目标 MDS。

**相关 API：**

```c
// include/subtree_map.h
enum mds_status subtree_map_transfer_owner_if_migrating(
    struct subtree_map *map,
    const char *path,
    uint32_t new_owner,
    uint64_t expected_ver);
```

该函数仅在 `state == SUBTREE_MIGRATING` 时成功，确保所有权转移发生在迁移流程中。
# 3 跨子树重命名（2PC）  
在 pNFS-lattice 的多 MDS 架构中，每个 MDS 通过 `subtree_map` 拥有若干子树（shard）。当 NFS 客户端发起 `RENAME` 操作，且源路径和目的路径分属**不同 MDS 拥有的子树**时，就产生了跨子树重命名问题。这本质上是一个**分布式事务**问题——需要保证两个子树上的元数据变更要么全部成功，要么全部回滚。
# 4 子树拆分与合并  
pNFS-lattice 的多 MDS 架构通过 `subtree_map` 将命名空间划分为多个子树（partition），每个子树由一个 MDS 拥有。初始部署时通常配置 `/shard1`... `/shardN` 对应 MDS 1...N。但在运行过程中，某些子树可能成为热点（例如 `/data/hot_dataset` 访问量远超其他区域），此时需要将该子目录**拆分**为独立子树并迁移到专用 MDS 上；反之，当负载降低后，又需要将其**合并**回父级子树。  

```txt
                ┌──────────────────────┐
                │   mds-admin CLI       │
                │  subtree split / merge │
                └──────────┬───────────┘
                           │
                ┌──────────▼───────────┐
                │  cluster_transport    │
                │  CT_MSG_SPLIT_ADMIN   │
                └──────────┬───────────┘
                           │
          ┌────────────────▼────────────────┐
          │     subtree_split_execute()      │
          │                                  │
          │  validate → check → register     │
          │           │                      │
          │     dest == self?                │
          │      │ Yes      │ No             │
          │      ▼          ▼                │
          │   完成    migration_initiate()   │
          │            (Freeze-Stream-       │
          │             Commit-Unfreeze)     │
          │              │                   │
          │         失败 → 回滚              │
          └─────────────────────────────────┘

自动路径:
┌──────────────────────────────────────────┐
│  split_evaluator (后台线程)               │
│  每 5 分钟扫描 → 检测热点子树             │
│  → 生成提案 (PENDING)                    │
│  → 管理员审批 (APPROVED)                 │
│  → 自动执行 subtree_split_execute()       │
└──────────────────────────────────────────┘

合并路径:
┌──────────────────────────────────────────┐
│  subtree_merge_execute()                  │
│  验证 → 检查同 MDS → remove_subtree      │
│  （远程合并未实现）                        │
└──────────────────────────────────────────┘
```

# 5 故障转移  
pNFS-lattice 的故障转移采用 **Active-Standby** 主备模型：每个 Primary MDS 配一个 Standby MDS，Standby 通过事务 delta 日志复制（`txn_delta`）保持与 Primary 的数据同步。当 Primary 故障时，Standby 自动提升为新的 Primary，接管其子树所有权并进入 NFSv4.1 Grace Period 允许客户端恢复。  

**核心文件：**

| 文件                   | 职责                               |
| -------------------- | -------------------------------- |
| failover.h           | 故障转移 API 头文件                     |
| failover.c           | 提升/降级核心逻辑（553 行）                 |
| failover_watchdog.h  | 看门狗 API 头文件                      |
| failover_watchdog.c  | 心跳过期检测线程（403 行）                  |
| cluster_membership.c | 成员管理、promote_standby、remove_self |
| subtree_map.c        | 子树接管的 CAS 操作                     |
##### 5.1.1.1.1 角色状态机  
```
FAILOVER_STANDBY ──► FAILOVER_PROMOTING ──► FAILOVER_PRIMARY       ▲                                        │       │                                        │       └──── (controlled demote) ◄──── FAILOVER_DEMOTING
```

|角色|值|说明|
|---|---|---|
| `FAILOVER_STANDBY` |0|备用状态，接收复制、监控伙伴心跳|
| `FAILOVER_PROMOTING` |1|提升中（瞬态），正在执行接管流程|
| `FAILOVER_PRIMARY` |2|主状态，正在服务子树|
| `FAILOVER_DEMOTING` |3|降级中（瞬态），用于滚动升级场景|

**关键约束：** 当前版本 **不实现自动 failback（降级回切）**。Demotion 仅支持受控场景（`failover_controlled_demote`），需要显式协议和 resync 保证。  

##### 5.1.1.1.2 故障检测：看门狗（Watchdog）  
核心实现在 `failover_watchdog.c`。
###### 5.1.1.1.2.1 心跳写入（Writer 侧）  
每个 MDS 运行一个心跳线程，**每 5 秒**通过 `mds_cluster_node_heartbeat()` 刷新 Catalogue（RonDB）中自己节点注册行的 `last_heartbeat_ns`（`CLOCK_REALTIME` 纳秒时间戳）。  

###### 5.1.1.1.2.2 存活轮询（Reader 侧）  
Standby 上运行一个看门狗 pthread，核心参数：

|参数|默认值|含义|
|---|---|---|
| `poll_interval_ms` |2000 ms|轮询间隔|
| `stale_timeout_ms` |15000 ms|过期阈值（= 3 次心跳）|
| `min_observe_ms` |20000 ms|启动保护期，防止误判|
每个轮询周期的逻辑（`watchdog_fn`）：
1. **睡眠**一个 poll_interval
2. 检查角色是否为 `FAILOVER_STANDBY`（非 STANDBY 则退出）
3. 检查是否在**启动保护期**（`min_observe_ms` 内不提升）
4. 调用 `mds_cluster_node_scan_stale()` 扫描伙伴的心跳时间戳
5. 若伙伴心跳**过期且在 REALTIME 时钟域内** → 调用 `failover_promote()`
6. 提升成功 → 看门狗线程退出；被拒绝 → 下个周期重试

###### 5.1.1.1.2.3 时钟域安全  
心跳时间戳使用 `CLOCK_REALTIME`。定义了一个 **REALTIME 地板值** `FAILOVER_HB_REALTIME_FLOOR_NS`（2020-01-01T00:00:00Z）。如果伙伴的时间戳低于此值，说明是旧版 `CLOCK_MONOTONIC` 域（滚动升级场景），视为**不确定（INDETERMINATE）**，跳过该 tick，**绝不宣布伙伴过期**——防止对活着的 Primary 发起脑裂式提升。  

##### 5.1.1.1.3 故障转移提升——八阶段严格流程
核心函数 `failover_promote()`，实现严格的八阶段提升序列：
###### 5.1.1.1.3.1 Phase 1: 角色守卫  
```bash
必须处于 FAILOVER_STANDBY 状态，否则返回 MDS_ERR_PERM
```
###### 5.1.1.1.3.2 Phase 2: 预检查（promote_prechecks）  
包含三个子检查：
- **2a 自隔离（Self-fencing）**：通过 `cluster_membership_is_self_fenced()` 检查自身是否已被隔离（如心跳写入失败表明已被取代）
- **2b 复制健康门**：通过 `health_repl_is_ok()` 检查复制链路是否健康，确保 Standby 数据足够新
- **3 伙伴死亡确认**：调用 `detect_cb`（如果非 NULL）确认伙伴确实已死
- **3a 线协议兼容检查**：确保 self 与 partner 的 `wire_compat_version` 一致，防止版本不匹配时提升
###### 5.1.1.1.3.3 Phase 4a: 快照伙伴路径  
调用 snapshot_partner_paths() 记录伙伴当前拥有的所有子树路径，用于后续失败时的定向回滚。  

###### 5.1.1.1.3.4 Phase 5: 加载客户端恢复记录  
通过 `failover_load_recovery_clients() `从协调后端加载伙伴服务过的所有客户端的恢复记录。  

###### 5.1.1.1.3.5 Phase 6: 进入宽限期  
调用 `grace_enter_with_clients()`，携带已知的客户端列表进入 NFSv4.1 宽限期：
- 已知客户端可通过 CLAIM_PREVIOUS 恢复状态
- 所有客户端回收完成或超时后，宽限期结束  

###### 5.1.1.1.3.6 Phase 7: 发布 ACTIVE + ACTIVE_SERVING
调用` cluster_membership_promote_standby()` 将自身在集群成员表中从 NODE_STANDBY 原子更新为 NODE_ACTIVE + NODE_ACTIVE_SERVING。**仅在 takeover + recovery + grace 全部完成后才发布**，确保对等节点看到 ACTIVE_SERVING 时该节点已准备好服务。

###### 5.1.1.1.3.7 Phase 8: 角色转换为 PRIMARY
```txt
ctx->role = FAILOVER_PRIMARY;
```

##### 5.1.1.1.4 受控降级（Controlled Demote）  
`failover_controlled_demote() `用于计划内的角色交接（如滚动升级），流程：  
1. **角色守卫**：必须为 PRIMARY
2. **要求活跃复制连接**：无复制则拒绝（`MDS_ERR_PERM`），确保 Standby 数据是最新的
3. **线协议兼容检查**：在任何状态变更前检查
4. **角色设为 DEMOTING**
5. **静默（Quiesce）**：调用 `subtree_map_freeze_owned()` 冻结所有本地子树，NFS 客户端收到 `NFS4ERR_DELAY`
6. **复制同步屏障**：RonDB 原生多节点一致性，无需额外同步步骤
7. **从成员表自删除**：调用 `cluster_membership_remove_self()`，此时仍为 `ACTIVE + ACTIVE_SERVING`，Standby 的 partner-loss 观察者检测到此删除事件后触发提升
8. **终态**：角色设为 `STANDBY`，节点准备好停止

##### 5.1.1.1.5 竞争安全——多 Standby 场景  
当多个 Standby 同时竞争同一个死 Primary 的子树时：
1. 两个 Standby（A、B）同时调用 `failover_promote()`
2. A 的 CAS 先到达存储 → 成功获取所有 partition 所有权 → 成为 PRIMARY
3. B 的 CAS 全部返回 `MDS_ERR_STALE` → `failover_promote()` 返回 `MDS_ERR_STALE`
4. B 保持 `STANDBY` 角色，其看门狗继续观察
这确保了**绝不会两个节点同时认为自己是 Primary**。
##### 5.1.1.1.6 Partner-Loss 观察者（权威模式）  
除了看门狗轮询，还支持**事件驱动**的提升路径（`cluster_membership.h`）  
- 通过 `cluster_membership_set_partner_loss_cb()` 注册观察者
- 当 RonDB watch 机制检测到伙伴的 `ACTIVE + ACTIVE_SERVING` 成员被删除时，触发 `partner_loss_cb`
- 回调中直接调用 `failover_promote()`
- 用于权威模式（etcd/RonDB watch-driven），替代 `detect_cb`

受控降级时的自删除（Step 7）正是利用了这个机制来触发 Standby 的提升。  

##### 5.1.1.1.7 设计要点总结  
| 设计要点     | 实现方式                                            |
|----------|-------------------------------------------------|
| 防脑裂      | CAS 竞争 + 自隔离 + 复制健康门 + 时钟域检查                    |
| 快速检测     | 看门狗 2s 轮询 + 15s 过期阈值（3 次心跳）                     |
| 启动安全     | 20s 最小观察期，防止 Standby 早于 Primary 初始化完成时误判        |
| 滚动升级安全   | REALTIME 地板值检测旧时钟域，不宣布旧 Primary 过期              |
| 客户端连续    | 宽限期 + 恢复记录收养，客户端可通过 CLAIM_PREVIOUS 恢复           |
| 回滚安全     | 快照 + 定向回滚，任何阶段失败都能归还子树                          |
| Failback | 未实现——需要显式的 failback 协议与 resync 保证（标记为 Deferred） |
