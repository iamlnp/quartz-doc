---
领域:
  - 协议
版本:
  - NA
状态:
  - 进行中
开发人员:
  - 刘乃朋
年份:
  - "2026"
imageNameKey: pnfs调研
tags:
  - LN技术评估
  - pnfs
---

# 1 pnfs  
pNFS（Parallel NFS，并行 NFS）,是 **NFSv4.1（RFC5661）原生标准扩展**，核心思想：**元数据路径和数据路径分离**，客户端拿到 layout 之后，可以绕过 NFS 元数据服务器，**直接并行访问后端存储节点**，解决传统 NFS 单服务器转发数据的瓶颈。

## 1.1 传统 NFS vs pNFS

- **普通 NFS**：所有读写数据流都经过 NFS 服务端，服务端成为带宽 / CPU 单点瓶颈
- **pNFS**：
    1. 元数据操作（open、stat、create、unlink、rename）走 MDS（元数据服务器）
    2. 客户端调用 `LAYOUTGET`，从 MDS 获取 **layout**（文件在后端存储的分片映射、权限 stateid、设备地址）
    3. 客户端直接和 Data Server / 块设备 / 对象存储通信读写数据，**数据不经过 MDS**
    4. MDS 可以随时 `LAYOUTRECALL` 收回 layout（比如文件元数据变更、truncate、删除、权限变动），强制客户端切回 MDS 通路

| Layout 类型                       | 编号 | 后端存储             | 说明                                                |
|---------------------------------|----|------------------|---------------------------------------------------|
| File Layout                     | 1  | 多台 NFS 数据服务器     | 文件条带分布在多个 NFS 节点，最经典；nfs-ganesha、NetApp、PanFS 常用  |
| Object Layout                   | 2  | OSD 对象存储         | OSD 协议，Ceph 早期原型思路，工业落地很少                         |
| Block Layout                    | 3  | 共享块存储 (iSCSI/FC) | 客户端直接读写块；Linux 内核 NFS 服务端 + XFS 支持；需要客户端可直连共享 LUN |
| Flexible File Layout(FlexFiles) | -  | 增强版 File Layout  | RFC8435，支持镜像、纠删、故障分片修复，现在主流新实现都用这个                |
## 1.2 关键流程（一次写）
1. Client → MDS：`OPEN` / `CREATE` 创建文件（元数据走 MDS）
2. Client → MDS：`LAYOUTGET`，请求文件布局
3. MDS 返回 layout：offset、长度、设备列表、stateid、条带信息
4. Client → 多个 Data Server：直接并行 WRITE（不走 MDS）
5. 发生元数据变更（unlink、rename、setattr）：MDS 发 `LAYOUTRECALL`，客户端停止直写，交还 layout，后续 IO 切回 MDS
6. Client 完成：`LAYOUTRETURN` 归还布局

## 1.3 重要核心概念
- **Layout**：文件到底层存储的地址映射 + 访问凭证（stateid），有有效期，可以被召回
- **Layout Recall**：MDS 强控手段，是 pNFS 一致性的关键；压测（fsstress）很多 bug 都出在 recall 竞争、layout 泄漏、IO 悬挂
- **Stateid**：pNFS 权限句柄，绑定 layout，防止越权访问数据设备
- **DS（Data Server）**：数据节点，只负责数据读写，不处理元数据

## 1.4 优缺点

✅ 优势
- 带宽横向扩展：多客户端、多 DS 并行 IO，适合 HPC、AI 大文件流式读写
- MDS 只处理元数据，CPU、网卡不再被数据流占满
- 兼容标准 NFS 语义，上层应用无需修改代码

❌ 代价
- 网络拓扑更复杂：客户端需要同时连通 MDS + 全部 DS
- 一致性逻辑变复杂：Layout Recall、IO 悬停、回收竞争是高频故障点
- 运维排障难：IO 可能走两条路径；抓包需要同时看 MDS 和 DS 流量
- 小 IO、频繁元数据操作场景收益不大（你前面那条 creat/write/unlink 小 IO workload，就是专门打元数据 + recall 路径）
# 2 pNFS-Lattice  
项目官网：  
1. [https://pnfs-lattice.org](https://link.wtturl.cn/?target=https%3A%2F%2Fpnfs-lattice.org&scene=im&aid=582478&lang=zh) 
2. [https://pnfs-hydra.io](https://link.wtturl.cn/?target=https%3A%2F%2Fpnfs-hydra.io&scene=im&aid=582478&lang=zh)
3. https://github.com/PEAK-AIO/pnfs-lattice  

由 PEAK:AIO + LANL（洛斯阿拉莫斯国家实验室）联合主导、Linux Foundation 下开源的新一代 **pNFS 元数据服务（MDS）项目**，定位：**弹性、可水平扩缩容、无单点的 pNFS MDS**，专门解决传统 pNFS MDS 元数据瓶颈问题，面向 AI / HPC 大集群场景

- **nfs-ganesha**：完整 NFS/pNFS 服务端（协议栈 + MDS + FSAL 对接后端存储），MDS 实例是有状态的、静态部署；集群方案靠外部分布式存储承接元数据
- **Lattice**：**只做弹性 MDS（元数据平面）**，原生设计目标是把 MDS 本身做成无状态、可随时启停、上千节点弹性扩缩的 “元数据集群”；**不负责 Data Server**，兼容标准 NFSv4.2 pNFS FlexFile Layout v2，DS 可以复用现有 ganesha DS 或者其他 pNFS Data Server

## 2.1 系统架构  
### 2.1.1 架构图
```bash
       ┌──────────────────────────────────────────┐
       │        MDS Cluster (active-active)        │
       │  ┌────────┐  ┌────────┐  ┌────────┐     │
       │  │ MDS-0  │  │ MDS-1  │  │ MDS-N  │     │
       │  └───┬────┘  └───┬────┘  └───┬────┘     │
       │      └───────────┼───────────┘           │
       └──────────────────┼───────────────────────┘
                          │ NDB C++ API
            ┌─────────────┴─────────────┐
            │     RonDB Cluster          │
            │  (共享元数据存储)            │
            │  inodes · dirents · stripe  │
            │  ds_registry · node_reg     │
            └─────────────┬─────────────┘
                          │
       ┌──────────────────┼──────────────────┐
┌──────▼──────┐   ┌──────▼──────┐   ┌───────▼──────┐
│  NFS DS-0   │   │  NFS DS-1   │   │  NFS DS-N    │
│flex TCP/RDMA│   │flex TCP/RDMA│   │flex TCP/RDMA  │
└─────────────┘   └─────────────┘   └──────────────┘
```

**三层分离设计**:
1. **MDS 层** — 元数据协调，active-active 多节点
2. **Catalogue 层** — 分布式事务元数据存储（RonDB/FoundationDB/DAOS/memdb）
3. **DS 层** — 标准 Linux knfsd，存储实际数据块
### 2.1.2 概念说明  
#### 2.1.2.1 MDS 空间 = 命名空间（Namespace）

每个 MDS 管理的是**命名空间的一个子树**，包括：

|内容|说明|
|---|---|
|目录树结构|文件/目录的层级关系|
|文件属性|权限、属主、大小、时间戳等|
|inode 元数据|文件标识符、链接计数等|
|Layout 映射|文件数据分布在哪些 DS 上|

在多 MDS 架构中，全局命名空间通过 **subtree map**（子树映射）被分割：  
```bash
/（全局根）
├── referral → MDS-1 管理的子树（如 /home/user1/）
├── referral → MDS-2 管理的子树（如 /home/user2/）
└── referral → MDS-3 管理的子树（如 /data/project/）
```

所有 MDS 通过 **RonDB**（分布式数据库）共享元数据，任何一个 MDS 都能看到全局命名空间，但每个 MDS 主要"负责"一个子树的所有权。  

##### 2.1.2.1.1 Shard和MDS的对应关系  
1. 每个 Shard 对应一个 MDS, 每个 shard 通过 referral 指向拥有它的 MDS, 比如：
```bash
/shard1  →  referral to MDS-1 (10.10.10.50)
/shard2  →  referral to MDS-2 (10.10.10.51)
/shard3  →  referral to MDS-3 (10.10.10.55)
```
从 `subtree_map.h `可以看到：

```c
struct subtree_entry {
    char                path[MDS_MAX_PATH];    // 如 "/shard1"
    uint32_t            owner_mds_id;          // 拥有的 MDS ID
    // ...
};
```

2. Shard 里面的操作默认只会落到这个 MDS 上

| 配置项             | 默认值   | 行为                                                                              |
|-----------------|-------|---------------------------------------------------------------------------------|
| referral_strict | true  | 操作会检查 filehandle 的 owner_mds_id，如果不是当前 MDS，返回 NFS4ERR_MOVED，强制客户端跳转到 owning MDS |
| referral_strict | false | 历史模式（serve-anywhere），任何 MDS 都可以处理任何操作                                           |
从 `compound_namespace.c`：
```c
if (subtree_map_lookup_exact(cd->smap, jpath, &se) == MDS_OK &&
    se.owner_mds_id != cd->mds_id) {
    return NFS4ERR_MOVED;  // 强制跳转到 owning MDS
}
```

3. Shard 内部会继续分片， 但需要手动或自动触发 split
```c
// 可以将 /data 分片为 /data/users、/data/projects 等子分片
enum mds_status subtree_split_execute(
    struct subtree_map *map,
    struct mds_catalogue *cat,
    const char *parent_path,    // "/data"
    const char *child_path,     // "/data/users"
    uint64_t child_fileid,
    uint32_t dest_mds_id,       // 可以分配给不同的 MDS
    // ...
);
```

**分片层级示例**：
```txt
/ (root)
├── /shard1 (MDS-1)
│   ├── /shard1/users (MDS-1) ← 可以进一步分片
│   └── /shard1/projects (MDS-2) ← 迁移到其他 MDS
├── /shard2 (MDS-2)
└── /shard3 (MDS-3)
```

**但通常不会自动递归分片**，分片是按需进行的（负载均衡或数据局部性优化）。  

#### 2.1.2.2 数据空间 = DS 的本地存储  
数据空间是**每个 DS 独立的本地存储**，不是多节点共享的：  
```bash
DS-1 (192.168.100.52)          DS-2 (192.168.100.53)
┌─────────────────────┐        ┌─────────────────────┐
│ /export/ds/         │        │ /export/ds/         │
│   data/             │        │   data/             │
│     5_0_0  (inode5) │        │     6_0_0  (inode6) │
│     7_0_0  (inode7) │        │     8_1_0  (inode8) │
└─────────────────────┘        └─────────────────────┘
       本地存储                       本地存储
```
**关键特征：**

| 特征         | 说明                                       |
| ---------- | ---------------------------------------- |
| **物理分布**   | 每个 DS 有自己的磁盘，数据文件存储在本地                   |
| **数据分散**   | 文件按 `inode_stripe_mirror` 格式分散存储在不同 DS 上 |
| **MDS 可见** | 所有 MDS 通过 NFS 挂载可以访问所有 DS 的数据            |
| **客户端直访**  | pNFS 客户端通过 Layout 直接访问 DS，不经过 MDS        |
### 2.1.3 源码模块结构  
| 目录               | 职责               | 核心文件                                                              |
| ---------------- | ---------------- | ----------------------------------------------------------------- |
| `src/mds/`       | NFSv4.1/4.2 协议处理 | compound.c (调度器), rpc_server.c, main.c                            |
| `src/catalogue/` | 可插拔元数据后端         | catalogue_rondb.c（RonDB C 包装，C++ shim 为企业版库）、catalogue_memdb.c、catalogue_dispatch.c、catalogue_factory.c、rondb_schema.c |
| `src/cluster/`   | MDS 间协调          | cluster_transport.c, rename_2pc.c, failover.c                     |
| `src/fsal_obj/`  | FSAL 对象抽象        | layout.c, placement.c                                             |
| `src/common/`    | 共享基础设施           | config.c, log.c, threadpool.c                                     |
| `src/tools/`     | CLI 工具           | mds-admin, mds-find, mds-apid, mds-backend-probe                  |
| `src/modules/`   | 可选模块             | ds_gc, ds_prealloc, layout_cache, observability 等                 |
| `include/`       | 公共 C 头文件         | 90+ 头文件，每个子系统一个                                                   |
| `tests/`         | 测试套件             | 86 个单元测试 + 集成测试 + 一致性测试                                           |

### 2.1.4 [pnfs元数据分布管理](pnfs元数据分布管理.md)  

### 2.1.5 [pnfs数据分布管理](pnfs数据分布管理.md)  

### 2.1.6 [pnfs数据备份与冗余机制](pnfs数据备份与冗余机制.md)  

### 2.1.7 [pnfs数据访问](pnfs数据访问.md)  

### 2.1.8 分布式锁  
Lattice 不引入独立的分布式锁管理器（DLM），而是利用 NFSv4.1/pNFS 协议内建的 状态模型（委托、锁、布局回收）配合 RonDB 的事务能力实现多客户端、多 MDS 节点 之间的一致性。  
```
┌─────────────────────────────────────────────────────────────────────┐
│                        一致性机制全景                                │
├──────────────────────┬──────────────────────────────────────────────┤
│ 协议层（NFSv4.1）      │  NFSv4.1 字节范围锁、共享保留、文件委托、     │
│                      │  目录委托、Layout 回收                        │
├──────────────────────┼──────────────────────────────────────────────┤
│ 数据路径层（pNFS）     │  Stripe Lease（条带租约）、Layout Recall      │
├──────────────────────┼──────────────────────────────────────────────┤
│ 集群层（Active-Active）│  TTL 缓存过期、change counter 校验、         │
│                      │  变更时 invalidate、RonDB 事务序列化、        │
│                      │  2PC 跨分片重命名、Catalog Delta 日志         │
├──────────────────────┼──────────────────────────────────────────────┤
│ 回调基础设施           │  CB_RECALL / CB_LAYOUTRECALL / CB_NOTIFY /   │
│                      │  CB_GETATTR                                   │
└──────────────────────┴──────────────────────────────────────────────┘
```

Lattice 的核心设计选择是**以数据库事务替代分布式锁**：  
1. **协议内建状态模型**：NFSv4.1 已经定义了完整的委托、锁、layout 状态机， Lattice 忠实实现这些机制而非发明新协议。
2. **RonDB 作为序列化点**：所有协议状态持久化到 RonDB，利用数据库的行级 事务锁实现并发写入的序列化。这消除了对独立锁服务的需求。
3. **TTL + 版本号的双重缓存校验**：内存缓存通过 TTL 限定过期窗口，通过 change counter 实现精确校验，在保证性能的同时维持一致性。
4. **回通道驱动异步回收**：所有 recall 操作通过 NFSv4.1 backchannel 异步 通知客户端，不阻塞 MDS 主路径。
5. **Active-Active 无领导者**：任意数量的 MDS 可同时服务同一命名空间， 无需选举、无需故障转移事件，一致性由底层事务存储保证。
#### 2.1.8.1 NFSv4.1 字节范围锁  
| 属性       | 说明                                                                     |
|----------|------------------------------------------------------------------------|
| 协议依据     | RFC 8881 §18.10 – §18.12（LOCK / LOCKT / LOCKU）                         |
| 锁类型      | READ_LT（读锁）、WRITE_LT（写锁）；阻塞变体在内部归一化为非阻塞                                |
| 锁粒度      | 按 (fileid, clientid, lock_owner) + 字节偏移/长度                             |
| 冲突语义     | POSIX 风格——重叠锁替换而非拒绝                                                    |
| 并发控制     | 16 条带互斥锁（按 fileid 哈希）                                                  |
| 跨 MDS 可见 | 通过 lock_table_set_cat() 持久化到 RonDB                                     |
| 协调接口     | mds_coord_lock_put / del / test / scan_file / scan_owner / reap_client |
| 关键文件     | include/lock_state.h、src/mds/lock_state.c、include/mds_coordination.h   |
```
客户端 A  LOCK(W, 0, 1024)  ──►  MDS-1  ──┐
                                           ├──► RonDB lock_table（共享行）
客户端 B  LOCK(R, 512, 2048) ──►  MDS-2  ──┘
           │
           └─ 冲突检测：scan_file → 字节范围重叠 + 类型冲突 → NFS4ERR_DENIED
```
#### 2.1.8.2 共享保留（Share Reservation）  
| 属性   | 说明                                        |
|------|-------------------------------------------|
| 协议依据 | RFC 8881 §8.2 / §9.1 / §18.16             |
| 访问模式 | READ、WRITE、BOTH                           |
| 拒绝模式 | NONE、READ、WRITE、BOTH                      |
| 冲突处理 | 不兼容的 OPEN 返回 NFS4ERR_SHARE_DENIED         |
| 持久化  | open_state_table_set_cat() → RonDB        |
| 关键文件 | include/open_state.h、src/mds/open_state.c |
共享保留在 OPEN 阶段建立，与字节范围锁互补：前者控制"谁能打开文件做什么"， 后者控制"打开后哪些字节范围互斥"。  

#### 2.1.8.3 文件委托（File Delegation）  
委托是 NFSv4.1 中**客户端缓存一致性**的核心机制。MDS 将文件的读或写权限 "委托"给客户端，委托期间该客户端可本地处理相关操作而无需往返 MDS。  

| 委托类型                | 并发持有       | 冲突条件             |
|---------------------|------------|------------------|
| OPEN_DELEGATE_READ  | 多客户端可同时持有  | 任何其他客户端持有任意委托即冲突 |
| OPEN_DELEGATE_WRITE | 同一文件仅一个持有者 | 任何其他客户端持有任意委托即冲突 |
**生命周期 :**   
```
MDS 授予委托 ──► 客户端本地处理读/写
       │
       ▼
另一个客户端请求冲突操作
       │
       ▼
deleg_check_conflict() ──► 检测到冲突
       │
       ▼
deleg_recall_file() ──► CB_RECALL（回通道）
       │
       ├── 客户端主动 DELEGRETURN ──► 委托归还 ✓
       │
       └── 超时未归还 ──► 强制撤销（deleg_revoke）
```

**CB_GETATTR**:  
当 WRITE 委托被 recall 时，MDS 可通过 `CB_GETATTR` 向持有者查询当前 文件大小和 change 属性，确保撤销前获取最新状态（RFC 8881 §20.1）。  

**待回收账本（Pending-Recall Ledger）**:  
支持跨 session 重建的 CB_RECALL 重传（pynfs DSESS9003 测试场景）， 确保客户端不会因网络抖动而遗漏回收通知。  

**配置与关键文件**：
  
| 配置项                                            | 默认值   | 说明       |
|------------------------------------------------|-------|----------|
| file_delegations_enabled                       | false | 是否启用文件委托 |
| 关键文件：include/delegation.h、src/mds/delegation.c |
#### 2.1.8.4 目录委托（Directory Delegation）  
|属性|说明|
|---|---|
|协议依据|RFC 8881 §10.9|
|键|(dir_fileid, clientid)|
|通知事件|`NOTIFY4_REMOVE_ENTRY`、`NOTIFY4_ADD_ENTRY`、`NOTIFY4_RENAME_ENTRY`|
|写入冲突门控|`dir_deleg_is_writer_present()` 防止重复授予|
|通知流程|目录变更 → `dir_deleg_notify_dir()` → `CB_NOTIFY`（mask 匹配）或 `CB_RECALL`（降级）|

配置与关键文件：

| 配置项                           | 默认值     | 说明          |
| ----------------------------- | ------- | ----------- |
| `dir_delegations_enabled`     | `false` | 是否启用目录委托    |
| `dir_deleg_recall_timeout_ms` | —       | recall 超时时间 |

关键文件：`include/dir_delegation.h`、`src/mds/dir_delegation.c`
#### 2.1.8.5 pNFS Layout 回收  
Layout 回收是 pNFS 特有的**数据路径**缓存一致性机制。当客户端持有的 layout 与新请求冲突时，MDS 通过 `CB_LAYOUTRECALL` 回调要求客户端归还。  

1. 触发场景

| 场景     | 函数                                     | 说明                                         |
|--------|----------------------------------------|--------------------------------------------|
| 字节范围冲突 | layout_recall_byte_range_for_holders() | LAYOUTGET 时扫描重叠范围，RW vs 任意 / READ vs RW 冲突 |
| DS 下线  | layout_recall_for_ds()                 | 回收引用该 DS 的所有 layout                        |
| 文件最终删除 | layout_recall_revoke_all_for_unlink()  | unlink 前回收该文件的全部 layout                    |
2. 回收范围
`CB_LAYOUTRECALL` 支持三种作用域：
- **文件级**：回收特定文件的 layout
- **FSID 级**：回收整个文件系统的 layout
- **ALL**：回收客户端所有 layout

3. DS 侧隔离
通过 `layout_recall_set_proxy()` 在回收后将 DS 上的 backing 文件 chown 到 fencing uid/gid，防止已撤销的客户端继续写入（RFC 8435 §14）。

关键文件：`include/layout_recall.h`、`src/mds/layout_recall.c`、 `src/mds/nfs4_cb.c`、`src/mds/compound_layout.c`  
#### 2.1.8.6 Stripe Lease（条带租约）  
Stripe Lease 是 Lattice 自定义的**独占写保障**机制，工作在单个条带粒度。

```
文件 F，条带数 4

客户端 A 写入 stripe[0] 的 byte[0..4096]
  ──► 获取 lease(fileid=F, stripe=0, range=[0,4096])

客户端 B 写入 stripe[0] 的 byte[2048..6144]
  ──► 检测到重叠 lease，且持有者为不同客户端
  ──► 返回 NFS4ERR_LAYOUTTRYLATER

客户端 C 写入 stripe[1] 的 byte[0..4096]
  ──► 不同条带，无冲突 ✓
```

|属性|说明|
|---|---|
|键|(fileid, stripe_index)|
|冲突域|同一条带内的重叠字节范围|
|冲突返回| `NFS4ERR_LAYOUTTRYLATER` |
|主动缩窄| `stripe_lease_prefix_conflict_free_length()` 允许部分授予|
配置：

|配置项|默认值|说明|
|---|---|---|
| `stripe_lease_duration_ms` | `30000` |租约有效期；设为 0 禁用|
关键文件：`include/lease_table.h`、`include/lease_stripe_map.h`、 `src/mds/lease_table.c`

#### 2.1.8.7 跨 MDS 缓存一致性（Active-Active 集群）  
Lattice 支持多个 MDS 同时服务同一命名空间（无领导者、无故障转移事件）。 集群内的一致性通过以下分层策略实现： 
1. TTL 过期
当 `cluster_size > 1` 时，inode 和 dirent 缓存自动启用正条目 TTL：

|配置项|默认值|说明|
|---|---|---|
| `positive_cache_ttl_ms` | `1000` |缓存条目最大存活时间（ms）|
这限制了来自对端 MDS 变更的过期条目遮蔽新 fileid 的窗口。  

2. 变更时失效
每次命名空间变更操作立即 invalidate 本地缓存：

| 操作      | 失效的缓存                                       |
| ------- | ------------------------------------------- |
| CREATE  | `inode_cache`、`dirent_cache`                |
| REMOVE  | `inode_cache`、`dirent_cache`、`layout_cache` |
| RENAME  | `inode_cache`、`dirent_cache`（源 + 目标目录）      |
| LINK    | `inode_cache`、`dirent_cache`                |
| SETATTR | `inode_cache`                               |
3. change counter 校验
缓存读取时校验目录服务的 `change` 计数器（行版本号）。若计数器与缓存 不一致，则回源 RonDB 重新加载。这保证了即使 TTL 未过期，也能检测到 对端 MDS 的并发修改。  

4. RonDB 事务序列化
所有变更操作编译为对 RonDB 的**单一事务**。并发写入者通过 NDB 的 **行级锁**自然序列化。这意味着：
- 不需要额外的分布式锁服务
- 数据库事务系统本身提供了序列化保证
- 委托、锁、layout 等协议状态行通过 `*_table_set_cat()` 写入 RonDB， 对所有 MDS 节点可见

5. 2PC 跨分片重命名
当 rename 的源目录和目标目录位于不同 MDS 分片时，使用两阶段提交：

```
MDS-1（源）                    MDS-2（目标）
    │                              │
    ├── PrepareRename ────────────►│
    │                              ├── 检查冲突、预留
    │◄─────────── 准备完成 ─────────┤
    │                              │
    ├── CommitRename ─────────────►│
    │                              ├── 执行变更
    │◄─────────── 提交确认 ─────────┤
    │                              │
    ├── 执行变更                    │
    │                              │
```

若 Prepare 阶段任一方失败，则 AbortRename 回滚。

关键文件：`src/cluster/cluster_transport.c`、`include/cluster_transport.h`、 `proto/mds_cluster.proto`  

6. Catalog Delta 日志
锁定变更包装器在变更成功后发射语义重放记录（delta），包含操作类型和完整参数（见 [pnfs元数据分布管理](pnfs元数据分布管理.md) 2.1.4.9）：

|Delta 类型|说明|
|---|---|
| `INODE_UPSERT` |节点属性更新|
| `DIRENT_PUT` |目录项添加|
| `DIRENT_DELETE` |目录项删除|
| `LAYOUT_GRANT` |布局授予|
| `LAYOUT_RETURN` |布局归还|
|…|更多操作类型|
Delta 流带有 (stream_id, sequence_no) 标识，支持多 MDS 间的顺序回放。
关键文件：`include/catalog_delta.h`  

7. Parent-Touch 覆盖层
`parent_touch` 子系统确保跨 MDS 的 `change_info` 永不回退。它维护一个 **只升不降**的属性覆盖层，即使目录的实际属性因缓存延迟尚未同步， `change_info.atomic` 和 `change` 值也不会出现倒退。

关键文件：`include/parent_touch.h`  
#### 2.1.8.8 NFSv4.1 回调基础设施  
所有 recall / notify 机制依赖 NFSv4.1 的**回通道**（backchannel），复用 前通道 TCP 连接：

|回调类型|用途|触发场景|
|---|---|---|
| `CB_RECALL` |回收文件委托|委托冲突|
| `CB_LAYOUTRECALL` |回收 pNFS layout|layout 冲突 / DS 下线 / 文件删除|
| `CB_NOTIFY` |目录委托通知|目录变更事件|
| `CB_GETATTR` |查询委托持有者属性|WRITE 委托回收时|

关键文件：`include/nfs4_cb.h`、`include/nfs4_cb_sec.h`、`src/mds/nfs4_cb.c`  

#### 2.1.8.9 租约与宽限期  
|属性|说明|
|---|---|
|客户端租约|按 clientid 跟踪，可配置|
|租约超时清理|`deleg_revoke_client()`、`open_state_revoke_client()`、`lock_release_all_for_client()`、session 清理|
|宽限期（Grace）|守护进程重启后阻止非 reclaim 的 OPEN，防止旧状态与新状态冲突|
|Courtesy 客户端|RFC 8881 §8.4.3——过期客户端的状态在冲突时按需撤销|
配置：

| 配置项              | 默认值  | 说明         |
| ---------------- | ---- | ---------- |
| `lease_time_sec` | `90` | 客户端租约时长（秒） |
关键文件：`include/session.h`、`include/grace.h`  

#### 2.1.8.10 跨 MDS 协调层  
协调层将协议状态持久化到 RonDB，使所有 MDS 节点可见：

|状态类型|RonDB 行结构|说明|
|---|---|---|
|Open / Share| `mds_coord_open_row` |含 owner_mds_id + boot_epoch（fencing 用）|
|Lock| `mds_coord_lock_row` |含冲突测试逻辑|
|Delegation| `mds_coord_deleg_row` |含冲突扫描|
|Layout| `mds_coord_layout_*` |通过 stateid 查询|

关键文件：`include/mds_coordination.h`

#### 2.1.8.11 机制对照表  
| 机制           | 类型                                | 作用域              | 关键文件                                | 状态       |
|--------------|-----------------------------------|------------------|-------------------------------------|----------|
| 字节范围锁        | NFSv4.1 LOCK                      | 文件字节范围           | lock_state.h/c                      | 生产可用     |
| 共享保留         | NFSv4.1 OPEN                      | 文件级              | open_state.h/c                      | 生产可用     |
| 文件委托         | NFSv4.1 delegation                | 文件级              | delegation.h/c                      | 已交付，默认关闭 |
| 目录委托         | NFSv4.1 delegation                | 目录级              | dir_delegation.h/c                  | 已交付，默认关闭 |
| Layout 回收    | pNFS CB_LAYOUTRECALL              | 文件字节范围           | layout_recall.h/c、nfs4_cb.h/c       | 生产可用     |
| Stripe Lease | 自定义条带租约                           | (fileid, stripe) | lease_table.h、lease_stripe_map.h    | 生产可用     |
| 跨 MDS 一致性    | TTL + change counter + invalidate | 集群全局             | inode_cache.h、dirent_cache.h、main.c | 生产可用     |
| 2PC 重命名      | 两阶段提交                             | 跨分片 / 跨 MDS      | cluster_transport.h、rename_2pc.c    | 生产可用     |
| 协调状态         | RonDB 持久化协议行                      | 集群全局             | mds_coordination.h                  | 生产可用     |

### 2.1.9 进程模型（线程组）  
单个 `pnfs-mds` 进程内包含多个线程组：  

| 线程组               | 来源                         | 角色                         |
| ----------------- | -------------------------- | -------------------------- |
| RPC listeners     | `rpc_server.c`             | 接受连接，解码 COMPOUND           |
| Compound workers  | `compound.c`               | 执行 compound_process()      |
| Catalogue I/O     | `catalogue_rondb.c` + 企业版 C++ shim | 驱动 NDB 事务（社区版链接 `src/common/rondb_stubs.c`） |
| DS GC             | `ds_gc.c`                  | 清理孤立 stripe 行              |
| DS health         | `ds_health.c`              | NFS NULL 探测 + I/O 限制探测     |
| Cluster transport | `cluster_transport.c`      | gRPC 对等消息                  |
| Sessions/DRC      | `session.c`                | NFSv4.1 会话表                |
| Backchannel       | `nfs4_cb.c`                | CB_RECALL, CB_LAYOUTRECALL |
| Metrics           | `metrics_http_lite.c`      | Prometheus /metrics 端点     |
### 2.1.10 Compound 处理管线  
每个 NFSv4 请求是一个 COMPOUND（包含多个 op），处理流程：  
```bash
RPC listener (accept + decode)
  → Worker pool (XDR decode + auth)
    → compound_init (构建 compound_data)
      → per-op dispatch (op_open / op_read / ...)
        → encode result (XDR + RPC reply)
        → [on mutate] catalogue txn (单次 NDB 提交)
        → [on layout] placement + stripe map write
```

Op 处理器按主题拆分到不同文件：
- `compound_namespace.c` — ACCESS, LOOKUP, GETATTR, CREATE, REMOVE, RENAME, LINK, READDIR
- `compound_data_io.c` — OPEN/CLOSE, READ/WRITE, COMMIT
- `compound_layout.c` — LAYOUTGET/RETURN/COMMIT, GETDEVICEINFO
- `compound_session.c` — EXCHANGE_ID, CREATE_SESSION, SEQUENCE
- `compound_nfsv42.c` — ALLOCATE, COPY, CLONE, SEEK 等 NFSv4.2 操作
### 2.1.11 可插拔 Catalogue 层
通过 vtable 模式实现后端无关性：  
```bash
include/mds_catalogue.h          ← 公共 API（后端无关）
include/catalogue_internal.h     ← 内部 vtable 定义（4 个 vtable）
src/catalogue/catalogue_dispatch.c ← 分发器（路由到活动后端）
src/catalogue/catalogue_factory.c  ← 后端注册工厂
```

**vtable 结构** (catalogue_internal.h):

- `mds_catalogue_ops` — 生命周期（close, probe, bootstrap）
- `mds_authority_ops` — 命名空间数据平面（create, remove, rename, lookup, readdir...）
- `mds_coordination_ops` — 恢复关键状态（layout, delegation, lock, session, open state）
- `mds_cluster_ops` — 多 MDS 服务（节点注册、心跳、分区映射）

### 2.1.12 并发与一致性  
- **单事务保证**: 每个变更操作（mkdir, rmdir, link, rename, setattr）编译为**一个 NDB 事务**
- **原子父 inode 更新**: 使用 NDB `interpretedUpdateTuple` 避免 read-modify-write 竞争
- **Striped 锁**: `pthread_mutex_t` 数组，16 或 1024 条纹（open_state 默认 1024）
- **NDB 排他行锁**: 用于 setattr 等需要序列化的路径
- **Any-MDS 路由**: 任何 MDS 可服务任何文件句柄，通过 RonDB 共享权威 + `parent_touch` 叠加层保证一致性

### 2.1.13 NFS Layout 路径  
```bash
OPEN(create) → cat_create() → 分配 fileid, 写 dirent + inode
  → [预分配器开启时] 从 per-DS 前瞻池拉取 stripe 坐标

LAYOUTGET → placement_select_ex() → 基于 mds_shard_map / ds_health 视图计算放置
  → stripe map 持久化到 stripe_maps 表
  → byte-range 冲突召回扫描
  → 返回 flex-files layout 给客户端

Client I/O → 直接与 DS 通信（MDS 不参与）

LAYOUTCOMMIT → 客户端告知新文件大小 → 一个事务更新 inode
LAYOUTRETURN / CB_LAYOUTRECALL → 任一方释放 layout
```

### 2.1.14 集群特性  
- **Active-active**: 分区命名空间 + `fs_locations` referral（NFS4ERR_MOVED）
- **跨子树重命名**: 两阶段提交 (2PC) over inter-MDS gRPC
- **子树迁移**: 通过 admin RPC 实时转移所有权
- **故障转移**: RonDB 原生 partner-loss 检测 + 自动提升 + NFSv4.1 grace period
- **复制**: 由 NDB 集群原生处理，无应用层 delta 传输

#### 2.1.14.1 Active-Active  
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
##### 2.1.14.1.1 分区命名空间  
###### 2.1.14.1.1.1 子树映射（Subtree Map）
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

###### 2.1.14.1.1.2 分片播种（Shard Seeding）
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

###### 2.1.14.1.1.3 Junction 目录（交叉点目录）
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
##### 2.1.14.1.2 fs_locations Referral 机制  
###### 2.1.14.1.2.1 NFSv4.1 协议基础  
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
###### 2.1.14.1.2.2 Lattice 实现流程  
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

###### 2.1.14.1.2.3 FSID 隔离
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

##### 2.1.14.1.3 Referral 拓扑强制（Referral Topology Enforcement）  
###### 2.1.14.1.3.1 问题背景  
由于所有 MDS 共享同一个 RonDB 目录，**任何 MDS 都能服务任何 fileid**。这意味着：
- 客户端可能持有缓存的 FH（file handle）
- 在 referral submount 过期后，客户端继续对错误的 MDS 发起操作
- 绕过了 referral 拓扑，导致：
    - 负载不均衡（wrong-MDS load）
    - `nosharecache` 跨实例可见性竞争

###### 2.1.14.1.3.2 referral_strict 机制  
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

##### 2.1.14.1.4 路径解析（Fileid → Path）  
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

##### 2.1.14.1.5 配置项  
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

#### 2.1.14.2 子树迁移（Subtree Migration）
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
#### 2.1.14.3 跨子树重命名（2PC）  
在 pNFS-lattice 的多 MDS 架构中，每个 MDS 通过 `subtree_map` 拥有若干子树（shard）。当 NFS 客户端发起 `RENAME` 操作，且源路径和目的路径分属**不同 MDS 拥有的子树**时，就产生了跨子树重命名问题。这本质上是一个**分布式事务**问题——需要保证两个子树上的元数据变更要么全部成功，要么全部回滚。
#### 2.1.14.4 子树拆分与合并  
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

#### 2.1.14.5 故障转移  
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
##### 2.1.14.5.1 角色状态机  
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

##### 2.1.14.5.2 故障检测：看门狗（Watchdog）  
核心实现在 `failover_watchdog.c`。
###### 2.1.14.5.2.1 心跳写入（Writer 侧）  
每个 MDS 运行一个心跳线程，**每 5 秒**通过 `mds_cluster_node_heartbeat()` 刷新 Catalogue（RonDB）中自己节点注册行的 `last_heartbeat_ns`（`CLOCK_REALTIME` 纳秒时间戳）。  

###### 2.1.14.5.2.2 存活轮询（Reader 侧）  
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

###### 2.1.14.5.2.3 时钟域安全  
心跳时间戳使用 `CLOCK_REALTIME`。定义了一个 **REALTIME 地板值** `FAILOVER_HB_REALTIME_FLOOR_NS`（2020-01-01T00:00:00Z）。如果伙伴的时间戳低于此值，说明是旧版 `CLOCK_MONOTONIC` 域（滚动升级场景），视为**不确定（INDETERMINATE）**，跳过该 tick，**绝不宣布伙伴过期**——防止对活着的 Primary 发起脑裂式提升。  

##### 2.1.14.5.3 故障转移提升——八阶段严格流程
核心函数 `failover_promote()`，实现严格的八阶段提升序列：
###### 2.1.14.5.3.1 Phase 1: 角色守卫  
```bash
必须处于 FAILOVER_STANDBY 状态，否则返回 MDS_ERR_PERM
```
###### 2.1.14.5.3.2 Phase 2: 预检查（promote_prechecks）  
包含三个子检查：
- **2a 自隔离（Self-fencing）**：通过 `cluster_membership_is_self_fenced()` 检查自身是否已被隔离（如心跳写入失败表明已被取代）
- **2b 复制健康门**：通过 `health_repl_is_ok()` 检查复制链路是否健康，确保 Standby 数据足够新
- **3 伙伴死亡确认**：调用 `detect_cb`（如果非 NULL）确认伙伴确实已死
- **3a 线协议兼容检查**：确保 self 与 partner 的 `wire_compat_version` 一致，防止版本不匹配时提升
###### 2.1.14.5.3.3 Phase 4a: 快照伙伴路径  
调用 snapshot_partner_paths() 记录伙伴当前拥有的所有子树路径，用于后续失败时的定向回滚。  

###### 2.1.14.5.3.4 Phase 5: 加载客户端恢复记录  
通过 `failover_load_recovery_clients() `从协调后端加载伙伴服务过的所有客户端的恢复记录。  

###### 2.1.14.5.3.5 Phase 6: 进入宽限期  
调用 `grace_enter_with_clients()`，携带已知的客户端列表进入 NFSv4.1 宽限期：
- 已知客户端可通过 CLAIM_PREVIOUS 恢复状态
- 所有客户端回收完成或超时后，宽限期结束  

###### 2.1.14.5.3.6 Phase 7: 发布 ACTIVE + ACTIVE_SERVING
调用` cluster_membership_promote_standby()` 将自身在集群成员表中从 NODE_STANDBY 原子更新为 NODE_ACTIVE + NODE_ACTIVE_SERVING。**仅在 takeover + recovery + grace 全部完成后才发布**，确保对等节点看到 ACTIVE_SERVING 时该节点已准备好服务。

###### 2.1.14.5.3.7 Phase 8: 角色转换为 PRIMARY
```txt
ctx->role = FAILOVER_PRIMARY;
```

##### 2.1.14.5.4 受控降级（Controlled Demote）  
`failover_controlled_demote() `用于计划内的角色交接（如滚动升级），流程：  
1. **角色守卫**：必须为 PRIMARY
2. **要求活跃复制连接**：无复制则拒绝（`MDS_ERR_PERM`），确保 Standby 数据是最新的
3. **线协议兼容检查**：在任何状态变更前检查
4. **角色设为 DEMOTING**
5. **静默（Quiesce）**：调用 `subtree_map_freeze_owned()` 冻结所有本地子树，NFS 客户端收到 `NFS4ERR_DELAY`
6. **复制同步屏障**：RonDB 原生多节点一致性，无需额外同步步骤
7. **从成员表自删除**：调用 `cluster_membership_remove_self()`，此时仍为 `ACTIVE + ACTIVE_SERVING`，Standby 的 partner-loss 观察者检测到此删除事件后触发提升
8. **终态**：角色设为 `STANDBY`，节点准备好停止

##### 2.1.14.5.5 竞争安全——多 Standby 场景  
当多个 Standby 同时竞争同一个死 Primary 的子树时：
1. 两个 Standby（A、B）同时调用 `failover_promote()`
2. A 的 CAS 先到达存储 → 成功获取所有 partition 所有权 → 成为 PRIMARY
3. B 的 CAS 全部返回 `MDS_ERR_STALE` → `failover_promote()` 返回 `MDS_ERR_STALE`
4. B 保持 `STANDBY` 角色，其看门狗继续观察
这确保了**绝不会两个节点同时认为自己是 Primary**。
##### 2.1.14.5.6 Partner-Loss 观察者（权威模式）  
除了看门狗轮询，还支持**事件驱动**的提升路径（`cluster_membership.h`）  
- 通过 `cluster_membership_set_partner_loss_cb()` 注册观察者
- 当 RonDB watch 机制检测到伙伴的 `ACTIVE + ACTIVE_SERVING` 成员被删除时，触发 `partner_loss_cb`
- 回调中直接调用 `failover_promote()`
- 用于权威模式（etcd/RonDB watch-driven），替代 `detect_cb`

受控降级时的自删除（Step 7）正是利用了这个机制来触发 Standby 的提升。  

##### 2.1.14.5.7 设计要点总结  
| 设计要点     | 实现方式                                            |
|----------|-------------------------------------------------|
| 防脑裂      | CAS 竞争 + 自隔离 + 复制健康门 + 时钟域检查                    |
| 快速检测     | 看门狗 2s 轮询 + 15s 过期阈值（3 次心跳）                     |
| 启动安全     | 20s 最小观察期，防止 Standby 早于 Primary 初始化完成时误判        |
| 滚动升级安全   | REALTIME 地板值检测旧时钟域，不宣布旧 Primary 过期              |
| 客户端连续    | 宽限期 + 恢复记录收养，客户端可通过 CLAIM_PREVIOUS 恢复           |
| 回滚安全     | 快照 + 定向回滚，任何阶段失败都能归还子树                          |
| Failback | 未实现——需要显式的 failback 协议与 resync 保证（标记为 Deferred） |

### 2.1.15 性能优化  
- **零 LMDB 代码**: 生产二进制中无 LMDB 实现
- **直接 catalogue 分发**: 移除 commit_queue 写线程，直接从工作线程写 RonDB
- **Per-thread fileid 池**: `_Thread_local` 批量分配，零竞争
- **融合 LOOKUP**: dirent + inode 在 1 个 NDB 事务中读取
- **NDB 连接池**: 自动调整为 worker_threads 大小
- **Wave-2 堆暂存**: op/result union 不再内联最坏情况载荷，每 worker 暂存从 ~37.8 MB 降至 ~1.11 MB
- **Striped inode/dirent 缓存**: 16 条纹 LRU
- **零拷贝 RPC 分发**: 工作线程直接从 recv_buf 读取
- **Per-thread XDR 缓冲区**: 消除每 RPC ~1MB 的 alloc/free

## 2.2 项目推荐阅读顺序  
### 2.2.1 第一阶段：理解全貌
`README.md` — 项目概览、架构图、快速入门
`docs/architecture.md `— 核心文档，23 节的架构圣经，包含系统视图、进程模型、compound 处理、catalogue 设计、并发模型、网络、可观测性等
`docs/QUICKSTART.md` — 快速构建运行
### 2.2.2 第二阶段：深入核心模块
`include/pnfs_mds.h` — 顶层头文件，定义全局常量、inode 结构、枚举
`include/mds_catalogue.h `— 后端无关的 catalogue API（1193 行，非常详细的注释）
`include/catalogue_internal.h `— vtable 定义 + 7 条 slot 合约（C1-C7），理解可插拔后端的关键
`src/mds/compound.c `— COMPOUND 调度器入口（2638 行）
`src/mds/main.c `— 守护进程入口（2602 行）
### 2.2.3 第三阶段：理解数据路径
`src/mds/compound_namespace.c` — 命名空间操作（CREATE/REMOVE/RENAME/LINK/READDIR）
`src/mds/compound_data_io.c`— OPEN/CLOSE/READ/WRITE
`src/mds/compound_layout.c` — pNFS layout 路径（3543 行，最复杂的文件之一）
`src/catalogue/catalogue_rondb.c` — RonDB C 包装层（企业版另含 C++ shim 库；社区版链接 src/common/rondb_stubs.c）
### 2.2.4 第四阶段：理解集群与高级特性
`src/cluster/ `— 跨 MDS 协调（transport, 2PC, failover, migration）
`docs/striping-guide.m`d — HPC-Shared 宽条带文件
`docs/find-api.md` — 元数据搜索 API
`docs/pnfs-lab-deploy-guide.md` — 集群部署指南
### 2.2.5 第五阶段：测试与质量保证
`tests/catalogue_conformance/` — 后端一致性测试套件
`tests/unit/ `— 86 个单元测试（使用 CMocka 框架）
`CONTRIBUTING.md `— 代码规范、编译纪律、提交规范

## 2.3 部署简要指南  
### 2.3.1 部署方式概览
项目提供三种部署路径：

|方式|适用场景|复杂度|
|---|---|---|
|**预编译二进制**|快速评估|低|
|** `scripts/pnfs-lab` 自动化编排**|实验室/集群部署|中|
|**手动安装**|深度理解/定制|高|
### 2.3.2 3.2 快速评估（预编译二进制）  
从 https://pnfs-lattice.peakaio.com 下载预编译包（支持 Rocky 9.7 和 Ubuntu 24.04），包含 MDS 守护进程 + RonDB + 单节点默认配置。  
### 2.3.3 从源码构建  
```bash
# 依赖: GCC >= 11.1, CMake >= 3.20, RonDB 26.02+, libntirpc, OpenSSL
cmake -B build -DCMAKE_BUILD_TYPE=Release -DENABLE_RONDB=ON
cmake --build build -j$(nproc)
ctest --test-dir build --output-on-failure   # 运行测试（memdb 后端，无需 RonDB）
```

**构建选项**:
- `-DENABLE_RONDB=ON` — RonDB 后端
- `-DENABLE_FDB=ON` — FoundationDB 后端
- `-DENABLE_DAOS=ON` — DAOS 后端
- `-DENABLE_TESTS=ON` — 构建测试
- `-DENABLE_EBPF=ON` — eBPF 监控

用快捷命令：  
```bash
./scripts/pnfs-lab full-single    # 单 MDS 实验室
./scripts/pnfs-lab full-multi     # 多 MDS + 2 RonDB 数据节点
```

### 2.3.4 配置参考  
核心配置文件 `/etc/pnfs-mds/mds.conf`，关键键：

| 键                          | 默认值     | 说明                                           |
| -------------------------- | ------- | -------------------------------------------- |
| `mds_id`                   | 1       | 唯一节点 ID                                      |
| `nfs_port`                 | 2049    | NFSv4.1 监听端口                                 |
| `catalogue_backend`        | rondb   | 后端选择: rondb/memdb/fdb/daos                   |
| `worker_threads`           | 16      | COMPOUND 分发线程数                               |
| `ds_count`                 | —       | DS 数量                                        |
| `ds[N]`                    | —       | DS 地址: `host:/export`                        |
| `stripe_unit_bytes`        | 65536   | 条带单元大小                                       |
| `lease_time_sec`           | 90      | NFSv4 lease 时间                               |
| `workload_profile`         | default | 调优预设: default/hpc/ai_training/genomics/media |
| `inode_cache_size`         | 0       | inode LRU 缓存（0=禁用）                           |
| `file_delegations_enabled` | false   | 文件委托                                         |
### 2.3.5 服务管理  
```bash
sudo systemctl start/stop/restart pnfs-mds
sudo systemctl status pnfs-mds
journalctl -u pnfs-mds -f          # 实时日志
```

## 2.4 [pnfs编译部署](pnfs编译部署.md)
## 2.5 常用命令  
1. 检查rondb集群状态
```bash
# 137.33.24.134是mgmd节点
/opt/rondb/bin/ndb_mgm --ndb-connectstring=137.33.24.134:1186 -e show
```

2. 检查进程信息
```bash
# mysqld: sql服务
# ndbmtd: 数据node
# ndb_mgmd: 管理node
[root@lzhnode1 ~]# ps -aux |grep rondb
root      338927  1.2  1.8 3932084 618324 ?      Sl   Oct23  49:21 /opt/rondb/bin/mysqld --defaults-file=/etc/rondb/my.cnf --user=root
root     1021988  0.0  0.0  44272 11896 ?        Ss   Oct25   0:21 /opt/rondb/bin/ndbmtd --ndb-connectstring=137.33.24.134:1186 --ndb-nodeid=1 --nodaemon
root     1021991 99.1  2.2 3963788 744684 ?      Rl   Oct25 1478:38 /opt/rondb/bin/ndbmtd --ndb-connectstring=137.33.24.134:1186 --ndb-nodeid=1 --nodaemon
root        1746  1.1  0.0 793360 31996 ?        Ssl  10:29   0:09 /opt/rondb/bin/ndb_mgmd -f /etc/rondb/config.ini --configdir=/var/lib/rondb/mgm --nodaemon
```

3. systemd信息
```bash
systemctl status rondb-ndbmtd
systemctl status rondb-mgmd
systemctl status rondb-mysqld

systemctl restart rondb-ndbmtd
systemctl restart rondb-mgmd
systemctl restart rondb-mysqld
```

