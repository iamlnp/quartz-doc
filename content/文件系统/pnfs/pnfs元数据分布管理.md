元数据分布机制回答的核心问题是：**元数据如何在 MDS、DS、客户端三者之间产生、流转、同步和回收**。以下按生命周期阶段展开。

# 1 元数据产生机制：DS 放置（Placement）

当客户端首次对文件发起 LAYOUTGET 时，MDS 需要决定该文件的条带映射——即数据分布在哪些 DS 上。这是元数据分布的**起点**。

```txt
┌─────────────────────────────────────────────────────────────────┐
│  LAYOUTGET 触发 DS 放置决策                                       │
│                                                                 │
│  1. 读取 DS 注册表 (ds_cache → ds_registry)                      │
│     → 过滤 ONLINE + 兼容 mode/transport 的 DS 集合                │
│                                                                 │
│  2. 选择放置策略 (placement_policy_enabled ? 配置策略 : RR)        │
│     ┌──────────────────────────────────────────────────────┐     │
│     │ PLACEMENT_RR          : 全局原子计数器轮转              │     │
│     │ PLACEMENT_WEIGHTED_RR : 加权蓄水池 (total - used)      │     │
│     │ PLACEMENT_CAPACITY    : 最大空闲 DS 优先               │     │
│     └──────────────────────────────────────────────────────┘     │
│                                                                 │
│  3. 核心分配公式:                                                 │
│     ds_idx = (start_key + s * mirror_count + m) % online_count   │
│     其中 start_key = fileid (使不同文件均匀散布)                    │
│     约束: 同一 stripe 的不同 mirror 不在同一 DS                     │
│                                                                 │
│  4. 降级处理 (placement_select2):                                  │
│     请求 stripe_count > ONLINE DS 数 → 缩减到实际可用数            │
│     返回 effective_stripe_count (调用方必须用此值持久化)             │
│                                                                 │
│  5. 持久化: cat_stripe_map_put(header) + cat_stripe_map_put(entries)│
│     → 写入 mds_stripe_maps + mds_stripe_entries 表                │
└─────────────────────────────────────────────────────────────────┘
```

**放置结果即元数据分布的物理映射**：一个 `stripe_count × mirror_count` 的矩阵，每个条目记录 `(ds_id, nfs_fh)`。

### 1.1 放置策略详细实现（`placement.c`）

```txt
┌──────────────────────────────────────────────────────────────────┐
│  placement_select_ex2 策略分发器                                    │
│                                                                  │
│  单 DS 快速路径:                                                    │
│    sc==1 && mirror_count==1 && ds_count==1 && ONLINE              │
│    → 直接填充，跳过所有分配/扫描                                      │
│                                                                  │
│  栈/堆分配阈值:                                                     │
│    ds_count ≤ 64 (MDS_PLACEMENT_STACK_MAX) → 栈分配 scratch 数组   │
│    ds_count > 64 → malloc 分配，累加 placement_heap_fallback_total │
│    设计意图: 栈帧不超过 ~1 KiB                                      │
│                                                                  │
│  权重优先级 (fill_online_with_free):                                 │
│    1. 管理员配置 ds->weight (非零) → 第三方 DS 无容量上报时使用         │
│    2. 上报空闲空间 total_bytes - used_bytes → 有容量上报的 DS         │
│    3. 常量 1 → 未上报任何信息的 DS 保持可选                           │
│                                                                  │
│  Phase 3 降级 (D5):                                                │
│    sc * mirror_count > online_count                                │
│    → 缩减 stripe_count，累加 placement_degraded_total 指标           │
│                                                                  │
│  主分配循环:                                                        │
│    for each stripe s:                                              │
│      选择头部 DS (taken[] 位图保证 stripe 级去重)                     │
│        WEIGHTED_RR → mds_wrr_weighted_pick(pool, online_count)     │
│        CAPACITY    → mds_wrr_capacity_pick(pool, online_count)     │
│      若选中槽位已被占用 → 线性前探找下一个未设置槽位                     │
│      镜像槽位: 从 pick+1 开始扫描，跳过已占用槽位                      │
│        → 保证 no-two-mirrors-on-same-DS 不变量                      │
└──────────────────────────────────────────────────────────────────┘
```

### 1.2 HPC-Shared 两级兼容性过滤（`ds_filter_compatible_preferred`）

HPC 场景下，文件可能使用 RDMA/GPUDirect DS 以获得最佳性能。放置时执行两级过滤：

```txt
┌──────────────────────────────────────────────────────────────────┐
│  两级过滤流程                                                       │
│                                                                  │
│  第一级 (必需): ds_filter_compatible                                │
│    → 过滤 ONLINE + 匹配 required_mode + 至少一个匹配 transport       │
│    → legacy 记录 (transport==0) 视为 TCP-capable                    │
│                                                                  │
│  第二级 (偏好): 在必需集合内构建偏好子集                               │
│    → 匹配 preferred_transport (至少一个 bit)                        │
│    → 匹配 preferred_caps (所有 bits, 如 DS_CAP_GPUDIRECT)           │
│                                                                  │
│  决策:                                                            │
│    偏好子集非空 → 仅使用偏好子集 (RDMA/GPUDirect DS)                 │
│    偏好子集为空 → 降级使用完整必需集合 (优雅回退)                      │
│    preferred_transport==0 && preferred_caps==0                     │
│      → 直接走第一级过滤                                             │
│                                                                  │
│  常量:                                                            │
│    DS_TRANSPORT_TCP  = 0x01                                       │
│    DS_TRANSPORT_RDMA = 0x02                                       │
│    DS_CAP_GPUDIRECT  = 0x01                                       │
└──────────────────────────────────────────────────────────────────┘
```

### 1.3 替换选择与分层选择

| 函数 | 用途 | 算法 |
|---|---|---|
| `placement_select_replacement` | Resilver 替换故障 DS | 若 target_ds_id≠0 验证其 ONLINE 且不在排除列表；否则自动选首个合格 ONLINE DS |
| `placement_select_for_tier` | Tiering 分层 mover | 按 target_tier 过滤 + 排除列表，选空闲空间最大（least-used-bytes）的 DS |

### 1.4 关键限制常量

| 常量 | 值 | 说明 |
|---|---|---|
| `MDS_MAX_DS_NODES` | 256 | 最大 DS 节点数 |
| `MDS_MAX_MIRRORS` | 4 | 每文件最大镜像数 |
| `MDS_MAX_STRIPES` | 1024 | 宽条带上限（HPC N-to-1） |
| `MDS_PLACEMENT_STACK_MAX` | 64 | 栈分配 scratch 数组上限 |

# 2 元数据写入机制

元数据修改不是直接写后端，而是经过一个分层管道：

```txt
┌──────────────────────────────────────────────────────────────────┐
│  RPC 工作线程                                                     │
│  (compound_layoutget / op_open / op_create / ...)                │
│       │                                                          │
│       ▼                                                          │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Layer 0: Per-Compound 快照缓存                            │    │
│  │                                                          │    │
│  │  compound_inode_get(fileid):                              │    │
│  │    1. 查 current/saved inline cache (请求内)               │    │
│  │    2. 查 global inode_cache (LRU 全局缓存)                 │    │
│  │    3. 查 catalogue (后端读)                                │    │
│  │  → 同一 COMPOUND 内同 fileid 最多读一次 catalogue           │    │
│  │                                                          │    │
│  │  CREATE 时 inode_cache warmup:                             │    │
│  │    → 创建后立即填充缓存，后续 GETATTR 无需再读 catalogue      │    │
│  └──────────────────────┬───────────────────────────────────┘    │
│                         │                                        │
│                         ▼                                        │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Layer 1: 内存写（即时生效）                                 │    │
│  │                                                          │    │
│  │  open_state_table → 本地 hash 表插入                       │    │
│  │  layout_cache     → LRU 缓存 put                          │    │
│  │  deleg_table      → 本地 hash 表插入                       │    │
│  │  stripe_lease_table → 租约获取                             │    │
│  │                                                          │    │
│  │  效果: 本 MDS 立即可见；其他 MDS 不可见                      │    │
│  └──────────────────────┬───────────────────────────────────┘    │
│                         │ write-through                          │
│                         ▼                                        │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Layer 2: Commit Queue（批量提交管道）                       │    │
│  │                                                          │    │
│  │  commit_queue_submit(op) → 不可变操作描述符入队              │    │
│  │  写入线程: 攒批 (≤128 ops 或 ≤1MiB 或 ≤2ms)                │    │
│  │  → 通过 catalogue vtable 分发到后端                         │    │
│  │  → 可选: 通过 repl_conn 复制到 Standby MDS                  │    │
│  │                                                          │    │
│  │  核心不变量: NFS 回复不先于 catalogue 写入发送                │    │
│  └──────────────────────┬───────────────────────────────────┘    │
│                         │                                        │
│                         ▼                                        │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Layer 3: Catalogue 后端（共享持久化）                       │    │
│  │                                                          │    │
│  │  RonDB: 单行自提交 (每个 cat_* 调用独立原子提交)              │    │
│  │  Fused 操作: 多行单事务 (见下表)                             │    │
│  │                                                          │    │
│  │  效果: 所有 MDS 立即可见（RonDB 分布式一致性）               │    │
│  └──────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

### 2.1 Fused 操作列表（单事务多行原子写入）

| Fused 操作 | 功能 | 涉及表 |
|---|---|---|
| `ns_create_with_layout` | CREATE + LAYOUTGET 合并 | inode + dirent + stripe_map + layout_state |
| `ns_create_wide` | 宽条带文件创建 | inode + dirent + stripe_map (header + entries) |
| `layoutget_fused` | LAYOUTGET 单事务 | stripe_map + layout_state |
| `ns_remove_known_gc` | 最终 unlink + GC 入队 | inode + dirent + gc_queue + stripe_map |
| `ns_readdir_plus` | READDIR + GETATTR 合并 | dirent + inode (避免二次读取) |
| `ns_readdir_plus_from` | cookie 续读 READDIR | 同上 + cookie 定位 |

### 2.2 原子父目录更新（RonDB `interpretedUpdateTuple`）

并发 CREATE/UNLINK 时，父目录的 nlink/mtime/ctime/change 需要原子更新。RonDB 后端使用 `interpretedUpdateTuple` 的 `incValue`/`subValue` 操作，在存储引擎层完成增量修改，消除跨 MDS 的 read-modify-write 竞争。

### 2.3 Commit Queue 批处理参数

| 参数 | 默认值 | 说明 |
|---|---|---|
| `commit_batch_size` | 128 | 单批最大操作数 |
| `commit_batch_max_bytes` | 1 MiB | 单批最大字节数 |
| `commit_flush_ms` | 2 | 最大攒批等待时间 |
| `commit_queue_depth` | 4096 | 队列深度上限 |

**关键机制**：

- `transient_state_cache=true` 时跳过 Layer 2/3，内存即权威（单 MDS 安全）
- `transient_state_cache=false` 时 write-through 到 RonDB（多 MDS 必须）
- Fused 操作在单后端事务中完成多行原子写入

# 3 跨 MDS 元数据同步机制

多 MDS 部署下，元数据如何在节点间保持一致？

```txt
┌──────────────────────────────────────────────────────────────────┐
│               跨 MDS 元数据同步架构                                │
│                                                                  │
│   ┌────────────────────────────────────────────────────────┐     │
│   │  机制 A: 共享 Catalogue（主路径）                         │     │
│   │                                                        │     │
│   │  RonDB/FDB/DAOS 作为唯一权威 (shared authority)          │     │
│   │  所有 MDS 直接读写同一后端                                │     │
│   │  → inode/dirent/stripe_map/ds_registry 天然一致          │     │
│   │  → layout_state/open_state/delegation 通过协调表共享      │     │
│   └────────────────────────────────────────────────────────┘     │
│                                                                  │
│   ┌────────────────────────────────────────────────────────┐     │
│   │  机制 B: Delta Broadcast + Catalog Image（加速路径）      │     │
│   │                                                        │     │
│   │  每次 catalogue 写操作产生一条 delta 记录:                 │     │
│   │    (stream_id=MDS_id, seqno=单调递增, type, payload)     │     │
│   │                                                        │     │
│   │  写入 mds_delta_broadcast 表 ──► 其他 MDS 的 poller       │     │
│   │  定期轮询 ──► catalog_image_apply(delta)                  │     │
│   │  ──► 内存 image 逐步追上 authority                        │     │
│   │                                                        │     │
│   │  四种生命周期模式:                                        │     │
│   │    OFF     : 无 image，全部读 authority                    │     │
│   │    SHADOW  : image 回放但回复来自 authority                 │     │
│   │    COMPARE : 热读比较 image vs authority                   │     │
│   │    PRIMARY : 热读 image，失败回退 authority                 │     │
│   └────────────────────────────────────────────────────────┘     │
│                                                                  │
│   ┌────────────────────────────────────────────────────────┐     │
│   │  机制 C: 2PC Journal（跨 MDS 操作）                       │     │
│   │                                                        │     │
│   │  跨分片 RENAME / HARDLINK 需要两阶段提交:                  │     │
│   │                                                        │     │
│   │  Phase 1 (PREPARE):                                    │     │
│   │    协调方写 mds_rename_journal(PREPARED)                  │     │
│   │    参与方写 mds_rename_journal(PREPARED)                  │     │
│   │    获取分布式锁 (mds_ns_locks, 按 class 排序避免死锁)      │     │
│   │                                                        │     │
│   │  Phase 2 (COMMIT):                                     │     │
│   │    协调方执行本地操作 → journal(COMMITTED)                 │     │
│   │    通知参与方执行 → journal(COMMITTED)                     │     │
│   │    释放分布式锁                                           │     │
│   │                                                        │     │
│   │  崩溃恢复: 启动时扫描 journal                             │     │
│   │    ABORTED → 删除                                        │     │
│   │    PREPARED+COORDINATOR → abort (源端完整)                │     │
│   │    COMMITTED+COORDINATOR → 重试 commit 或清理源端          │     │
│   │    PREPARED+PARTICIPANT → 保留 (结果未知)                  │     │
│   └────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

### 3.1 共享权威模式 vs 遗留模式（2PC Rename）

```txt
┌──────────────────────────────────────────────────────────────────┐
│  rename_2pc_cat_shared_authority() 分支                            │
│                                                                  │
│  共享权威模式 (RonDB/FDB/DAOS):                                    │
│    → inode 是全局的，不需要复制到参与方                               │
│    → COMMIT 时仅移动 dirent，源端 inode 保留                        │
│    → 利用后端行级锁保证并发安全                                       │
│                                                                  │
│  遗留模式 (memdb/standalone):                                       │
│    → PREPARE 时将 inode + stripe_map 打包到 payload                 │
│    → 参与方在 PREPARE 阶段即写入本地存储                               │
│    → COMMIT 时删除源端 dirent + inode                                │
│    → payload 上限: R2PC_MAX_PAYLOAD_ENTRIES = 16 个 stripe 条目      │
└──────────────────────────────────────────────────────────────────┘
```

### 3.2 集群成员生命周期（`cluster_membership.c`）

```txt
┌──────────────────────────────────────────────────────────────────┐
│  节点生命周期状态机                                                  │
│                                                                  │
│  JOINING ──► IDLE ──► ACTIVE_SERVING ──► DRAINING ──► DRAINED   │
│                ▲                           │                     │
│                └───────────────────────────┘                     │
│                      (undrain 回退)                               │
│                                                                  │
│  角色:                                                            │
│    NODE_ACTIVE  (0) : 活跃服务节点                                 │
│    NODE_STANDBY (1) : 热备节点                                     │
│                                                                  │
│  关键约束:                                                         │
│    - leave 操作拒绝 SERVING/DRAINING 状态的节点                      │
│    - leave 要求节点不拥有任何 subtree                                │
│    - force_remove 绕过生命周期检查，但拒绝有 subtree 的 active 节点    │
│    - 协调者判定: mds_id 最小的节点为 coordinator                      │
│    - can_own_subtrees: 必须同时满足 ACTIVE + ACTIVE_SERVING          │
└──────────────────────────────────────────────────────────────────┘
```

### 3.3 节点注册与心跳（`mds_cluster.h`）

| 操作 | 说明 |
|---|---|
| `node_register` | 条件 upsert：仅当现有 `boot_epoch` 更低时替换，防止旧实例覆盖新实例 |
| `node_heartbeat` | 更新 `last_heartbeat_ns`；若 `boot_epoch` 不匹配返回 `MDS_ERR_STALE` |
| `node_deregister` | 条件删除：仅当 `boot_epoch` 匹配时删除 |
| `node_scan_stale` | 扫描超过阈值的节点，用于故障检测 |

`boot_epoch` 是 `CLOCK_REALTIME` 纳秒时间戳，跨重启单调递增。

### 3.4 子树自动分裂评估（`subtree_split.c`）

```txt
┌──────────────────────────────────────────────────────────────────┐
│  split_evaluator 自动热子树检测                                     │
│                                                                  │
│  配置参数:                                                         │
│    auto_split_threshold  : 10000 (ops 计数阈值)                    │
│    auto_split_interval   : 300s (评估周期)                          │
│    auto_split_cooldown   : 600s (冷却期)                            │
│    auto_split_sustained  : 2 (连续热周期数)                         │
│    auto_split_min_children : 4 (最小子目录数)                        │
│                                                                  │
│  评估流程:                                                         │
│  1. 扫描所有本地 subtree                                            │
│  2. 对每个 op_count > threshold 的 subtree:                         │
│     a. 检查 min_children 约束                                       │
│     b. 需要 sustained_intervals 个连续热周期                          │
│     c. 非热周期时对 pending proposal 执行热度衰减                      │
│     d. 冷却期过期后重新开始追踪                                        │
│  3. 满足条件 → 生成 split_proposal (PROPOSAL_PENDING)               │
│  4. auto_execute=true 时自动执行分裂                                  │
│                                                                  │
│  分裂执行 (subtree_split_execute):                                   │
│  1. map 验证                                                       │
│  2. catalogue getattr + junction 检查                                │
│  3. 注册子树到 subtree_map                                          │
│  4. 若目标 MDS 不同 → 触发跨 MDS 迁移                                │
│  5. 失败时回滚                                                      │
│                                                                  │
│  合并 (subtree_merge_execute): 仅支持本地合并；跨 MDS 合并未实现         │
└──────────────────────────────────────────────────────────────────┘
```

### 3.5 集群间传输协议（`cluster_transport.h`）

```txt
┌──────────────────────────────────────────────────────────────────┐
│  Wire Protocol: 5 字节头 + 变长 payload                             │
│  [msg_type u8][payload_len u32 BE]                                │
│  所有多字节字段大端序                                                 │
│  最大 payload: 4 MiB                                               │
│                                                                  │
│  92 种消息类型 (按功能分组):                                          │
│                                                                  │
│  2PC 协议 (1-5):                                                   │
│    PREPARE / VOTE / COMMIT / ABORT / ACK                          │
│                                                                  │
│  迁移协议 (6-12):                                                   │
│    MIG_BEGIN / MIG_INODE / MIG_FINISH / MIG_ACK / MIG_NACK        │
│    MIG_ADMIN_REQ / MIG_ADMIN_RESP                                 │
│                                                                  │
│  Resilver (13-15, 33):                                             │
│    RESILVER_START / STATUS / RESP / STATUS_EXT                    │
│                                                                  │
│  集群成员 (16-28):                                                  │
│    NODE_JOIN / LEAVE / CLUSTER_STATUS_REQ/RESP                    │
│    MEMBERSHIP_RESP / SET_LIFECYCLE                                 │
│    SPLIT_ADMIN_REQ/RESP / ASSIGN_ADMIN_REQ/RESP                   │
│    DRAIN_SELF / STANDBY_DETACH / FORCE_REMOVE                     │
│                                                                  │
│  管理查询 (29-32, 81-84):                                           │
│    SUBTREE_LIST / MIG_PROGRESS / CONFIG_SHOW                      │
│    DS_CAPACITY_SHOW / DS_SET_WEIGHT / DS_CAPACITY_PROBE            │
│                                                                  │
│  Rebalance (34-36): START / STATUS_REQ / RESP                     │
│  Tiering (37-40): START / STOP / STATUS_REQ / RESP                │
│  Quota (41-44): SET / GET_REQ / GET_RESP / DEL                    │
│  Failover/Upgrade (45-48): REQ/RESP / STATUS_REQ/RESP             │
│  DS 管理 (49-68): LIST / ADD / SET_STATE / REMOVE / VALIDATE      │
│    CLR / PROVISION / INFO / PATCH_READY / CLEAR                   │
│  Metrics (69-70): REQ / RESP                                      │
│  跨子树 nlink (71-76): INC/DEC REQ/RESP                           │
│  Split 评估 (77-80): PROPOSALS / APPROVE / AUTO                   │
│  HPC-Shared (89-92): MODE_SET / STATUS_REQ/RESP                   │
│                                                                  │
│  传输层安全:                                                        │
│    - 无 TLS + 无 allowlist → 仅允许 loopback                        │
│    - 有 TLS → 允许配置的对等方列表                                    │
│    - 每连接 30s poll 超时防 DoS                                      │
└──────────────────────────────────────────────────────────────────┘
```

# 4 分片路由机制：操作如何落到正确的 MDS

多 MDS 部署中，命名空间被分成多个 shard（分片），每个 shard 由一个 MDS 拥有。`home_shard_id` 嵌入在 inode 的 RonDB 行中（`rondb_schema.c`），使得单次 inode 读即可完成路由判断。

```txt
┌──────────────────────────────────────────────────────────────────┐
│  分片路由流程                                                      │
│                                                                  │
│  1. 客户端 PUTFH(filehandle)                                      │
│     │                                                            │
│     ▼                                                            │
│  2. MDS 从 FH 提取 fileid → 查 mds_shard_fileid 表                │
│     → 得到 home_shard_id (inode 行中存储)                          │
│     │                                                            │
│     ▼                                                            │
│  3. 查 subtree_map: home_shard_id → owner_mds_id                  │
│     │                                                            │
│     ├── owner_mds_id == self → 正常处理                            │
│     │                                                            │
│     └── owner_mds_id != self (referral_strict=true):              │
│         → 返回 NFS4ERR_MOVED                                      │
│         → 客户端根据 fs_locations referral 重新连接到正确 MDS       │
│                                                                  │
│  4. 跨分片 LOOKUP:                                                │
│     本地 dirent 不存在 → 查 mds_ext_dirents 表                     │
│     → 获得 (owner_mds_id, target_fileid, anchor_id)               │
│     → 返回 referral 给客户端                                      │
└──────────────────────────────────────────────────────────────────┘
```

### 4.1 最长前缀匹配算法（`subtree_map.c`）

```txt
┌──────────────────────────────────────────────────────────────────┐
│  longest_prefix_match 实现细节                                     │
│                                                                  │
│  数据结构:                                                         │
│    - entries: 动态数组 (初始容量 16)                                 │
│    - 按 subtree_path 降序长度排序 (resort 后维护)                    │
│    - lock: pthread_rwlock_t 保护并发访问                             │
│                                                                  │
│  匹配算法:                                                         │
│    线性扫描所有 entries:                                             │
│      is_path_prefix(entry.path, target_path):                     │
│        "/" 匹配一切                                                  │
│        "/data" 匹配 "/data", "/data/", "/data/foo"                  │
│        "/data" 不匹配 "/datafile" (边界检查: 下一字符必须是 / 或 \0)   │
│    第一个命中即为最长前缀 (因为已按长度降序排列)                        │
│                                                                  │
│  Referral 构建 (referral_build):                                   │
│    → RFC 8881 S11.11 fs_locations 编码                              │
│    → referral_fsid_major(owner_mds_id) 每 shard 独立 fsid           │
│    → 节点主机名表 (nodes[]) 用于生成 referral 地址                    │
└──────────────────────────────────────────────────────────────────┘
```

### 4.2 Fileid 批量分配

```txt
┌──────────────────────────────────────────────────────────────────┐
│  Fileid 分配策略                                                   │
│                                                                  │
│  MDS_FILEID_BATCH = 1024                                          │
│    → 每次从 catalogue 全局计数器预分配 1024 个 fileid                 │
│    → 减少分布式计数器竞争                                            │
│                                                                  │
│  Per-Thread 池:                                                    │
│    → _Thread_local 本地批量分配                                     │
│    → 零竞争: 每个 RPC 工作线程从自己的池中取 fileid                   │
│    → 池耗尽时再次从 catalogue 批量申请                               │
│                                                                  │
│  根 inode: MDS_FILEID_ROOT = 2 (POSIX 惯例)                       │
│                                                                  │
│  Shard 路由表:                                                     │
│    → shard_fileid_map: fileid → shard_id 映射                     │
│    → 用于跨 shard FH 解析                                          │
│    → 通过 ext_dirent 表支持跨 shard LOOKUP                         │
└──────────────────────────────────────────────────────────────────┘
```

### 4.3 Partition Map CAS 语义

| 操作 | 说明 |
|---|---|
| `partition_put(insert_only=true)` | 根节点声明：仅当行不存在时插入，竞争时返回 `MDS_ERR_EXISTS` |
| `partition_put(insert_only=false)` | 仅用于从未拥有的 shard seed，不覆盖已有行 |
| `partition_cas(partition_id, expected_owner, new_owner, new_state)` | **唯一**的所有权变更方式；owner 不匹配时返回 `MDS_ERR_STALE` |
| `partition_list` | 启动时失败是**致命错误**（有界重试后），永不视为空映射 |

### 4.4 冻结文件集（Migration 期间）

```txt
┌──────────────────────────────────────────────────────────────────┐
│  frozen_fileids 管理                                               │
│                                                                  │
│  数据结构: 排序数组 (qsort 插入, bsearch 查找)                       │
│  复杂度: O(log n) 查找                                             │
│                                                                  │
│  生命周期:                                                         │
│    subtree_map_freeze_fileids(fids, count) → 排序插入               │
│    subtree_map_is_fileid_frozen(fid) → 二分查找                     │
│    subtree_map_unfreeze_fileids(fids, count) → 移除                 │
│                                                                  │
│  用途: Migration 期间阻止冻结 fileid 的元数据变更                     │
│    → DFS 预收集所有 fileid → 注册冻结 → 流式传输 → 完成后解冻          │
└──────────────────────────────────────────────────────────────────┘
```

# 5 元数据缓存一致性机制

内存缓存与共享 Catalogue 之间的一致性维护：

| 缓存 | 写入策略 | 失效触发 | 一致性保证 |
|---|---|---|---|
| layout_cache | LAYOUTGET 命中时 put | remove / setattr(shrink) 时 invalidate | 最终一致；HPC-Shared 文件 stripe_map 生命周期内稳定 |
| open_state_table | open/close 时本地写 + write-through | CLOSE / lease 过期 / 崩溃恢复 | shared_state_supported 时 RonDB 权威；否则内存权威 |
| deleg_table | grant/return 时本地写 + write-through | recall / revoke / file unlink | 同上 |
| session_table | EXCHANGE_ID/CREATE_SESSION 时本地写 | lease 过期 / DESTROY_SESSION | 内存权威（clientid 按 MDS 隔离） |
| ds_cache | 启动时全量加载 + 管理员命令时 reload | ds add/remove/set-state/provision | 启动时一致；管理员操作后显式刷新 |
| stripe_lease_table | 纯内存，不持久化 | lease 过期 / LAYOUTRETURN | 单 MDS 内存权威 |
| inode_cache | per-compound 快照 + 全局 LRU | COMPOUND 结束 | 同一 COMPOUND 内 fileid 最多读一次 catalogue |
| catalog_image | delta 回放构建 | image_feed_stop / 模式切换 | 依赖 delta stream 完整性 |

### 5.1 Catalog Image 实现细节（`catalog_image.c`）

```txt
┌──────────────────────────────────────────────────────────────────┐
│  Catalog Image 内部数据结构                                         │
│                                                                  │
│  inode_map:                                                      │
│    - 开放寻址哈希表 (线性探测)                                       │
│    - 初始容量: 1024 (INODE_INIT_CAP)                               │
│    - 负载因子: 3/4 (LOAD_FACTOR_NUM/DEN) 时扩容                     │
│    - 扩容: 容量翻倍 + 全量 rehash                                    │
│    - 删除: 墓碑标记 + 后续链重新哈希                                  │
│    - 哈希函数: MurmurHash3 64-bit finalizer                        │
│                                                                  │
│  dirent_map:                                                     │
│    - 开放寻址哈希表                                                 │
│    - 初始容量: 2048 (DIRENT_INIT_CAP)                              │
│    - 哈希函数: hash_u64(parent) XOR 多项式哈希(name, base=31)       │
│    - READDIR: 收集 → qsort(strcmp) → start_after 过滤              │
│                                                                  │
│  并发控制 (Phase 9C):                                               │
│    - pthread_rwlock_t 保护 inodes + dirents                        │
│    - _get 操作: 读锁                                                │
│    - _put/_del/apply 操作: 写锁                                     │
│                                                                  │
│  Gap 检测:                                                         │
│    - high_water[MAX_STREAMS=16]: 每 stream 已回放最大 seqno          │
│    - expected_next[MAX_STREAMS=16]: 每 stream 期望下一 seqno         │
│    - has_gap[MAX_STREAMS=16]: 每 stream 是否存在间隙                 │
│    - catalog_image_is_complete(stream_id, authority_seqno):          │
│        !has_gap[stream_id] && high_water >= authority_seqno          │
│                                                                  │
│  INODE_UPSERT payload 格式 (133 字节):                               │
│    fileid(8) + type(1) + mode(4) + nlink(4) + uid(8) + gid(8)     │
│    + size(8) + space_used(8) + atime(12) + mtime(12) + ctime(12)  │
│    + change(8) + generation(8) + flags(4) + create_verf(8)         │
│    + parent_fileid(8) + stripe_count(4) + stripe_unit(4)           │
│    + mirror_count(4) = 133 bytes                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 5.2 Stripe Lease Table 实现（`lease_table.c`）

```txt
┌──────────────────────────────────────────────────────────────────┐
│  分片哈希表设计                                                     │
│                                                                  │
│  SLT_SHARD_COUNT = 16 (每 shard 一个 pthread_mutex)                │
│  哈希键: mix(fileid, stripe_index)                                 │
│    → fileid ^ (stripe_index * 0x9e3779b97f4a7c15)  (黄金比例常数)    │
│    → (mix ^ (mix >> 16)) & 0xF                                    │
│  设计意图: 同一 fileid 的不同 stripe 分散到不同 shard                  │
│    → 宽条带布局的并发租约请求不互相阻塞                                │
│                                                                  │
│  冲突域: 匹配物理 I/O 范围                                          │
│    → 不同客户端写同一文件的不相交 DS stripe → 不冲突                   │
│    → ds_id 仅用于追踪/可观测性，不参与冲突判定                         │
│                                                                  │
│  过期策略: 惰性驱逐                                                   │
│    → check_conflict / acquire 时先清理本 shard 过期条目               │
│    → shard_evict_expired: 遍历桶链，释放 expiry_ns <= now 的条目      │
│                                                                  │
│  续租路径:                                                          │
│    → 查找匹配 (fileid, clientid, stripe_index, ds_offset) 的现有条目  │
│    → 更新 ds_length, ds_id, expiry_ns                               │
│                                                                  │
│  前缀无冲突长度计算:                                                   │
│    stripe_lease_prefix_conflict_free_length                        │
│    → 按 stripe-index 顺序遍历 slices                                 │
│    → 首个冲突时计算 conflict_file_start                               │
│    → 返回 max(0, conflict_file_start - lease_offset)                │
│    → 用途: op_layoutget 主动缩窄授权范围                               │
│       而非返回 NFS4ERR_LAYOUTTRYLATER                                │
└──────────────────────────────────────────────────────────────────┘
```

transient_state_cache 开关（`pnfs_mds.h`）：
- false（默认）：open/delegation/lock 状态 write-through 到 RonDB，多 MDS 安全
- true：仅内存，单 MDS 部署优化，崩溃后客户端通过 reclaim 重建

# 6 元数据回收机制

元数据不会永久驻留，以下机制负责回收：

```txt
┌──────────────────────────────────────────────────────────────────┐
│  元数据回收路径                                                     │
│                                                                  │
│  A. GC 队列 (ds_gc)                                               │
│     触发: 文件最终 unlink (nlink→0)                                │
│     流程: gc_enqueue(fileid, ds_id, nfs_fh, sweep_hint)           │
│           → mds_gc_queue 表                                       │
│           → GC coordinator 线程 batch peek → 推入有界环形队列         │
│           → ds_gc_workers 个 worker 线程消费 → proxy unlink DS 文件  │
│           → gc_dequeue                                            │
│     并行: ds_gc_workers 个 worker 并行消费 (上限 32)                 │
│     分布: 每个 MDS 只 drain 自己 owner_mds_id 的行                   │
│     批量: ds_gc_batch_size (默认 256, 上限 4096)                    │
│     队列: DS_GC_QUEUE_CAP = 4096 有界环形缓冲                        │
│     指标: 每 6 个 tick 刷新 gc_pending gauge                        │
│                                                                  │
│  B. 异步删除清单 (remove_manifest)                                  │
│     触发: remove_async=true 时的 unlink                            │
│     流程: submit(dir_fileid, name, child_fileid, generation)       │
│           → 1. 检查容量 (atomic total < max_pending, 上限 ~1M)       │
│           → 2. 分配 tombstone (DRAINING 状态，对 executor 不可见)     │
│           → 3. 插入 stripe 哈希链                                   │
│           → 4. 提交 manifest 行 (durable-before-ack)                │
│           → 5. 成功后转 PENDING，广播 cond                           │
│           → coordinator 线程: batch peek → claim → 喂给 workers      │
│           → worker 线程: execute → finalize_inode                   │
│     优势: 将 DS 对象清理从 unlink 热路径移出                          │
│     哈希: 16 stripes, 每 stripe 独立 mutex + cond                   │
│     哈希函数: splitmix64(dir_fileid) XOR fnv1a(name)                │
│     孤儿清理: 每 30s scrub 一次，回收 stale tombstone                 │
│     强制排空: force_drain_entry 最多等待 5s (500 × 10ms slices)       │
│                                                                  │
│  C. Layout state 回收                                              │
│     触发: LAYOUTRETURN / CB_LAYOUTRECALL / DS 故障 / lease 过期      │
│     流程: layout_return → layout_state 行删除 + ds_layout_idx 删除   │
│           → stripe_lease_release_all_for                            │
│                                                                  │
│  D. Session/Open/Delegation 回收                                   │
│     触发: lease 过期 (session_table_start_reaper)                   │
│     流程: open_state_close_all_for_client                           │
│           → deleg_revoke_client                                     │
│           → session_destroy_client                                  │
│           → 可选: DS fencing (proxy chown 中立 uid)                  │
│                                                                  │
│  E. 客户端恢复记录回收                                               │
│     触发: RECLAIM_COMPLETE 后                                       │
│     流程: mds_coord_recovery_del(clientid)                          │
└──────────────────────────────────────────────────────────────────┘
```

### 6.1 GC Sweep Hint 编码（`pnfs_mds.h`）

```txt
┌──────────────────────────────────────────────────────────────────┐
│  sweep_hint 位域编码                                                │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │ Bit 31 (FLAG)  │ Bits 30-4 (SC)  │ Bits 3-0 (MC)       │     │
│  │ 1=槽位模式      │ stripe_count    │ mirror_count         │     │
│  │ 0=几何模式      │                 │                      │     │
│  └─────────────────────────────────────────────────────────┘     │
│                                                                  │
│  宏定义:                                                          │
│    MDS_GC_SWEEP_SLOT_FLAG  = (1U << 31)                           │
│    MDS_GC_SWEEP_GEOM(sc,mc) = ((sc)<<4 | (mc)&0xF)                │
│    MDS_GC_SWEEP_SLOT(s,m)  = (SLOT_FLAG | (s)<<4 | (m)&0xF)       │
│    MDS_GC_SWEEP_IS_SLOT(h) = ((h) & SLOT_FLAG)                    │
│    MDS_GC_SWEEP_SC(h)      = (((h)>>4) & 0x07FFFFFF)              │
│    MDS_GC_SWEEP_MC(h)      = ((h) & 0xF)                          │
│                                                                  │
│  三种清理模式:                                                      │
│    IS_SLOT → sweep_single_slot: 清理单个 (stripe,mirror) 文件       │
│    非零几何 → sweep_known_geometry: 遍历所有 (sc,mc) 组合            │
│    零值 → sweep_legacy_dense: 假设密集条带，首个缺失即停               │
│                                                                  │
│  约束上限: 64 DSes, 64×64 槽位扫描                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 6.2 Remove Manifest 状态机

```txt
┌──────────────────────────────────────────────────────────────────┐
│  rm_state 状态转换                                                  │
│                                                                  │
│  submit():                                                       │
│    [不可见] ──► DRAINING ──► PENDING                              │
│    (tombstone 分配)  (对 executor 不可见)  (对 executor 可见)        │
│                                                                  │
│  worker claim:                                                    │
│    PENDING ──► DRAINING ──► [执行 finalize] ──► 删除 tombstone      │
│                                                                  │
│  执行结果:                                                         │
│    OK → rm_finalize_inode:                                        │
│      1. layout_cache invalidate                                   │
│      2. LAYOUTCOMMIT aggregator drop                              │
│      3. GC enqueue (per unique DS in stripe map)                  │
│      4. stripe_map_del                                            │
│      5. inode_del                                                 │
│      6. quota update                                              │
│    STALE/NOTFOUND → 检查 DELETE_PENDING flag (Phase-R)             │
│    其他 → bump_retry, 回退 PENDING                                 │
│                                                                  │
│  INDOUBT 返回 (-2): commit 结果未知，tombstone 保留                   │
│  最大等待: RM_WAIT_SLICES(500) × RM_WAIT_SLICE_MS(10) = 5s         │
└──────────────────────────────────────────────────────────────────┘
```

# 7 元数据重分布机制：Rebalance & Resilver

当 DS 拓扑变化（故障、扩容、均衡）时，元数据需要重新映射：

```txt
┌──────────────────────────────────────────────────────────────────┐
│  Rebalance (管理员触发, 镜像搬迁)                                    │
│                                                                  │
│  场景: 将 DS-A 上的镜像数据迁移到 DS-B                               │
│  约束: 单 stripe 文件, 无活跃 layout/writer, dense 文件             │
│                                                                  │
│  流程:                                                           │
│  1. scan: cat_stripe_map_scan → 找到 DS-A 上的文件                  │
│  2. 检查: open_state(无活跃 writer) + layout_state(无活跃 layout)   │
│  3. copy: proxy_read(DS-A) → proxy_write(DS-B)                   │
│  4. atomic swap: stripe_map_put(新 DS-B entry)                    │
│                + gc_enqueue(旧 DS-A entry)                        │
│     → 通过 commit_queue 或 rebalance_move 单事务完成                 │
│  5. 重复直到所有候选处理完毕                                          │
│                                                                  │
│  状态机: IDLE → SCANNING → COPYING → DONE/ERROR                    │
│  跳过计数: skipped_active_layout / skipped_active_writer            │
│           / skipped_sparse / skipped_multi_stripe / skipped_error  │
│                                                                  │
│  注: 社区版为 stub 实现 (rebalance_stub.c)                          │
├──────────────────────────────────────────────────────────────────┤
│  Resilver (管理员触发, 降级修复)                                      │
│                                                                  │
│  场景: DS-A 故障后, 从其健康镜像副本重建到替换 DS-B                     │
│  约束: 同上                                                        │
│                                                                  │
│  流程:                                                           │
│  1. scan: 找到所有 degraded mirror 的文件                           │
│  2. 从健康 mirror 读取数据                                           │
│  3. 写入 target_ds                                                  │
│  4. 更新 stripe_map + gc_enqueue 旧条目                              │
│  5. placement_select_replacement 选择替换 DS                         │
│                                                                  │
│  状态机: IDLE → SCANNING → COPYING → DONE/ERROR                    │
│  注: 社区版为 stub 实现 (resilver_stub.c)                            │
├──────────────────────────────────────────────────────────────────┤
│  DS 故障时的元数据连锁反应                                            │
│                                                                  │
│  ds_health_monitor 检测到 DS 连续失败                                 │
│  → ds_registry.state = DS_OFFLINE (通过 commit_queue 复制)          │
│  → ds_fail_cb 回调触发                                             │
│  → layout_recall_for_ds():                                        │
│    1. ds_layout_idx_scan(ds_id) → 收集所有 (clientid, fileid)       │
│    2. 对每个唯一 clientid 发送 CB_LAYOUTRECALL(ALL)                  │
│    3. 强制撤销所有受影响的 layout_state 行                            │
│  → ds_cache_invalidate() 刷新本地缓存                               │
│  → 后续 LAYOUTGET 不再选择该 DS (placement 过滤 OFFLINE)              │
└──────────────────────────────────────────────────────────────────┘
```

### 7.1 DS 健康监控防抖机制（`ds_health.c`）

```txt
┌──────────────────────────────────────────────────────────────────┐
│  Anti-Flapping 算法                                                │
│                                                                  │
│  故障检测:                                                         │
│    默认轮询间隔: 5000ms (DS_HEALTH_DEFAULT_INTERVAL)                │
│    故障阈值: 6 次连续失败 (DS_HEALTH_DEFAULT_THRESHOLD)              │
│    探测方式: NFSv3 NULL RPC over TCP                                │
│      → NFS_PROGRAM=100003, NFS_VERSION=3, NFS_NULL_PROC=0          │
│      → 44 字节报文: 4B record mark + 40B RPC header                 │
│      → 非阻塞 connect + poll 超时 + TCP_NODELAY                     │
│                                                                  │
│  退避冷却:                                                         │
│    cooldown_ms = COOLDOWN_BASE_MS(30s) << min(flap_count, 4)       │
│    上限: COOLDOWN_CAP_MS = 480000ms (8 分钟)                        │
│    → 冷却期间跳过该 DS 的探测                                        │
│                                                                  │
│  恢复条件:                                                         │
│    需要 max(fail_threshold/2, RECOVERY_MIN=3) 次连续成功探测          │
│    → 即至少 3 次连续 OK 才标记为 ONLINE                               │
│                                                                  │
│  抖动计数器重置:                                                     │
│    连续 60s (DS_HEALTH_FLAP_RESET_NS) ONLINE → flap_count 归零       │
│                                                                  │
│  状态字段:                                                         │
│    consecutive_failures / marked_offline / recovery_successes       │
│    cooldown_until_ns / flap_count / last_online_ns                  │
└──────────────────────────────────────────────────────────────────┘
```

### 7.2 DS Cache 权重管理（`ds_cache.c`）

```txt
┌──────────────────────────────────────────────────────────────────┐
│  权重优先级链 (ds_cache_overlay_weights)                              │
│                                                                  │
│  1. 管理员覆盖 ds->weight (非零) → 无条件优先                         │
│  2. 容量派生 auto_weight (非零) → 来自 ds_capacity_probe             │
│  3. 空闲字节启发式 → placement 路径中计算                              │
│                                                                  │
│  容量聚合:                                                         │
│    ds_cache_aggregate_capacity(stale_sec):                         │
│      → 遍历所有 eligible 条目: present + ONLINE + total_bytes≠0     │
│        + 未过期 (last_observed_unix_sec > now - stale_sec)           │
│      → 饱和加法 (sat_add_u64, 上限 UINT64_MAX)                      │
│      → 返回 (total_bytes, used_bytes, ds_count)                     │
│                                                                  │
│  跨 MDS 观测合并:                                                    │
│    ds_cache_apply_remote_observations(list, count):                 │
│      → 合并对端 MDS 的容量观测                                        │
│      → 跳过 phantom-zero 行 (total=0 的虚假报告)                     │
│                                                                  │
│  快照: ds_cache_snapshot_ids → 复制所有 present ds_id                │
└──────────────────────────────────────────────────────────────────┘
```

# 8 Device ID 编码机制：元数据寻址桥梁

Device ID 是连接**逻辑元数据**和**物理 DS 地址**的桥梁：

```txt
┌──────────────────────────────────────────────────────────────────┐
│  Device ID 结构 (16 bytes)                                        │
│  ┌──────────────┬──────────────┬───────────────┬──────────────┐  │
│  │ mds_id (4B)  │ ds_id (4B)   │io_generation  │ zeros (4B)   │  │
│  │ Big-Endian   │ Big-Endian   │  (4B) BE      │              │  │
│  └──────────────┴──────────────┴───────────────┴──────────────┘  │
│                                                                  │
│  编码: build_deviceid(mds_id, ds_id, io_generation)               │
│  解码: deviceid_to_ds_id(deviceid) → ds_id                        │
│                                                                  │
│  用途:                                                           │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  LAYOUTGET 响应:                                         │     │
│  │    ff_data_server4.deviceid = build_deviceid(...)         │     │
│  │    → 客户端收到 deviceid                                  │     │
│  │                                                         │     │
│  │  GETDEVICEINFO 请求:                                     │     │
│  │    客户端发送 deviceid                                    │     │
│  │    MDS 解码 → ds_id → 查 ds_cache → 返回 host/port/...   │     │
│  └─────────────────────────────────────────────────────────┘     │
│                                                                  │
│  io_generation: DS 重新注册时递增, 使旧 deviceid 失效               │
│  客户端缓存旧 deviceid 的 GETDEVICEINFO 结果 → 下次发现 generation   │
│  不匹配 → 重新 GETDEVICEINFO                                       │
└──────────────────────────────────────────────────────────────────┘
```

# 9 Delta 记录机制：元数据变更追踪

每次 Catalogue 写操作都会产生语义化的 delta 记录（`catalog_delta.h`）：

```txt
┌──────────────────────────────────────────────────────────────────┐
│  Delta 记录格式 (25 字节头 + 变长 payload)                          │
│                                                                  │
│  type(1B) + stream_id(4B) + seqno(8B) + timestamp_ns(8B)         │
│  + payload_len(4B) + payload(变长)                                 │
│                                                                  │
│  22 种语义类型:                                                     │
│  INODE_UPSERT / INODE_DELETE / DIRENT_PUT / DIRENT_DELETE         │
│  STRIPE_MAP_PUT / STRIPE_MAP_DELETE                               │
│  XATTR_PUT / XATTR_DELETE                                        │
│  DS_REGISTRY_PUT / DS_REGISTRY_DELETE                             │
│  DS_PROVISION_PUT / DS_PROVISION_DELETE                           │
│  QUOTA_RULE_PUT / QUOTA_USAGE_PUT                                 │
│  GC_ENQUEUE / GC_DEQUEUE                                         │
│  LAYOUT_GRANT / LAYOUT_RETURN                                     │
│  CLIENT_RECOVERY_PUT / CLIENT_RECOVERY_DELETE                     │
│  RENAME_JOURNAL / PARTITION_MAP_UPDATE                            │
│                                                                  │
│  消费方:                                                          │
│  1. 持久日志 (catalog_replay_mode=JOURNAL) → 崩溃重建               │
│  2. delta_broadcast 表 → 跨 MDS 的 catalog_image 回放              │
│  3. 文件日志 (catalog_replay_mode=LOG) → 调试/审计                  │
│                                                                  │
│  stream_id 设计:                                                   │
│  单 MDS: stream_id=0                                              │
│  多 MDS: stream_id=MDS_id, 每个 MDS 独立单调 seqno                  │
│  → catalog_image 通过 per-stream high_water 追踪回放进度             │
└──────────────────────────────────────────────────────────────────┘
```

### 9.1 Delta Builder 实现（`catalog_delta.c`）

```txt
┌──────────────────────────────────────────────────────────────────┐
│  序列化/反序列化                                                     │
│                                                                  │
│  Builder 内部结构:                                                   │
│    buf: uint8_t* 输出缓冲区                                         │
│    len: 当前使用长度                                                 │
│    cap: 分配容量 (初始 BUILDER_INIT_CAP = 4096)                      │
│    count: 已追加记录数                                               │
│                                                                  │
│  扩容策略: builder_ensure(need)                                     │
│    → 容量不足时翻倍，直到 len + need 可容纳                             │
│    → 初始 cap=0 时从 4096 开始                                       │
│                                                                  │
│  序列化格式: 大端序编码                                                │
│    fdb_put_u32 / fdb_put_u64 写入各字段                              │
│    每条记录占 CATALOG_DELTA_HDR_SIZE(25) + payload_len 字节            │
│                                                                  │
│  反序列化: catalog_delta_deserialize(buf, len, cb, ctx)              │
│    → 逐条解析 25B 头 + payload                                       │
│    → 对每条记录调用 cb(&rec, ctx)                                     │
│    → cb 返回非零值时提前终止                                           │
│    → 截断 payload 或尾部残留字节 → 返回 -1                             │
│                                                                  │
│  API:                                                              │
│    create / append / count / serialize / reset / destroy            │
│    serialize 返回的 buf 由 builder 持有 (borrowed, not owned)          │
│    reset 清零 len 和 count，保留分配 (复用场景)                         │
└──────────────────────────────────────────────────────────────────┘
```

# 10 Catalogue 后端抽象层

### 10.1 后端工厂（`catalogue_factory.c`）

```txt
┌──────────────────────────────────────────────────────────────────┐
│  四种后端注册                                                       │
│                                                                  │
│  ┌──────────────────┬─────────────────────┬──────────────────┐   │
│  │ Backend ID       │ Constructor         │ Guard Macro      │   │
│  ├──────────────────┼─────────────────────┼──────────────────┤   │
│  │ MDS_BACKEND_RONDB│ catalogue_rondb_open│ HAVE_RONDB       │   │
│  │ MDS_BACKEND_MEMDB│ catalogue_memdb_    │ HAVE_MEMDB_      │   │
│  │                  │ open_cfg            │ BACKEND          │   │
│  │ MDS_BACKEND_FDB  │ catalogue_fdb_open  │ HAVE_FDB         │   │
│  │ MDS_BACKEND_DAOS │ catalogue_daos_open │ HAVE_DAOS        │   │
│  └──────────────────┴─────────────────────┴──────────────────┘   │
│                                                                  │
│  未编译的后端: .open = NULL (已知但未编译)                             │
│  mds_catalogue_backend_available(): 查找且 .open != NULL            │
│  mds_catalogue_open(cfg): 按 cfg->catalogue_backend 分发            │
└──────────────────────────────────────────────────────────────────┘
```

### 10.2 四组 Vtable 接口

```txt
┌──────────────────────────────────────────────────────────────────┐
│  struct mds_catalogue 的四个 vtable 指针                             │
│                                                                  │
│  1. mds_authority_ops (72 个函数指针)                                │
│     → 数据面: namespace / inode / dirent / stripe_map /            │
│       DS registry / quota / GC / prealloc / shard routing          │
│     → 关键操作:                                                     │
│       ns_create_wide, ns_remove_known_gc (fused)                   │
│       stripe_map_get/put/scan                                      │
│       gc_enqueue/peek_batch                                        │
│       shard_fileid_get/put/del                                     │
│       ext_dirent_get/put/del (跨 shard)                             │
│       link_anchor_put/del (跨 shard link)                           │
│                                                                  │
│  2. mds_coordination_ops (42 个函数指针)                             │
│     → 恢复关键状态: 2PC journal / layout_state /                    │
│       client_recovery / open / lock / delegation / session / DRC    │
│     → 关键操作:                                                     │
│       layoutget_fused (可选融合 LAYOUTGET)                            │
│       layout_grant_union (饱和 UNION 续租, 非覆盖)                    │
│       journal_put/get/del/scan (2PC 日志)                            │
│       lock_test (冲突检测)                                           │
│                                                                  │
│  3. mds_cluster_ops (8 个函数指针)                                   │
│     → 多 MDS 服务: node_registry / heartbeat / partition_map        │
│     → 关键操作:                                                     │
│       node_register/heartbeat/deregister/list/scan_stale            │
│       partition_list/put/cas                                        │
│                                                                  │
│  4. mds_catalogue_ops (6 个函数指针)                                 │
│     → 生命周期: close / probe / bootstrap /                         │
│       backend_handle / image_feed_start / image_feed_stop           │
│                                                                  │
│  能力位:                                                            │
│    MDS_CAT_CAP_SHARED_AUTHORITY (1<<0): 所有 MDS 共享单一权威          │
│    MDS_CAT_CAP_MULTI_PROCESS    (1<<1): 存储可被多进程访问              │
└──────────────────────────────────────────────────────────────────┘
```

### 10.3 Slot 合约（C1-C7）

每个后端必须遵守的七条绑定规则：

| 编号 | 规则 | 说明 |
|---|---|---|
| C1 | 可重入性 | 回调可重入同一 catalogue 句柄；后端必须使用 materialise-then-deliver 模式 |
| C2 | 重试安全 | 条目仅在事务尝试完成后投递，不在重试内部投递 |
| C3 | 连贯对象谓词 | stripe header+entries、layout-grant 验证、目录空判断、GC 重验证——全部在**一个事务**内决定 |
| C4 | 状态透传 | 分发器永不扁平化 slot 的返回状态 |
| C5 | 能力真实性 | 填充的 slot 必须实际实现该操作；禁止"存在但返回 NOSUPPORT" |
| C6 | 事务令牌语义 | `mds_cat_txn` 仅为分组上下文；多行原子性**仅在** fused slot 中存在 |
| C7 | 关闭时所有权 | `ops->close(cat)` 释放后端资源；分发器拥有并释放 `struct mds_catalogue` |

# 11 核心不变量

### 11.1 NFS 回复不先于 Catalogue 写入

**在 MDS 向客户端发送任何表示"成功"的 NFS 回复之前，使该回复为"真"的所有 Catalogue 写操作必须已经持久化完成。** 这是一个**写前日志（Write-Ahead）** 原则的变体。

### 11.2 Catalogue Vtable 合约不变量

```txt
┌──────────────────────────────────────────────────────────────────┐
│  结构不变量                                                        │
│                                                                  │
│  I1. 单权威: 共享权威模式下，所有 MDS 读写同一后端，无 per-node 元数据    │
│  I2. 行级锁: 并发修改由后端行级锁序列化 (RonDB NDB 事务)               │
│  I3. Fused 原子性: 多行操作在单后端事务中完成，无中间状态可见             │
│  I4. 事务令牌: mds_cat_txn 仅为分组上下文，不保证多行原子性              │
│      → 仅 fused slot 提供多行原子性                                   │
│                                                                  │
│  状态不变量                                                        │
│                                                                  │
│  I5. Boot Epoch 单调性: 跨重启单调递增，防止旧实例覆盖新实例             │
│  I6. Partition CAS 唯一性: 所有权变更仅通过 partition_cas              │
│  I7. 根声明幂等性: insert_only=true 保证恰好一个 MDS 赢得根             │
│  I8. 启动致命性: partition_list 启动失败是致命的 (有界重试后)            │
│                                                                  │
│  分布不变量                                                        │
│                                                                  │
│  I9. 镜像隔离: 同一 stripe 的不同 mirror 不在同一 DS                   │
│  I10. Fileid 唯一性: 全局计数器 + per-thread 批量分配保证不重复          │
│  I11. 降级一致性: effective_stripe_count 必须被调用方持久化              │
│  I12. 权重优先级: operator > auto_weight > free-bytes heuristic       │
│                                                                  │
│  恢复不变量                                                        │
│                                                                  │
│  I13. 2PC 崩溃安全: PREPARED+COORDINATOR → abort;                    │
│       COMMITTED+COORDINATOR → retry commit                          │
│  I14. Tombstone 持久性: durable-before-ack，manifest 行先于回复持久化    │
│  I15. INDOUBT 保守性: commit 结果未知时保留 tombstone，不丢弃            │
│  I16. 孤儿清理: 30s 周期 scrub 回收 stale tombstone                   │
│                                                                  │
│  缓存不变量                                                        │
│                                                                  │
│  I17. Per-Compound 快照: 同一 COMPOUND 内同 fileid 最多读一次 catalogue │
│  I18. CREATE Warmup: 创建后立即填充 inode_cache，后续 GETATTR 免读       │
│  I19. Delta Gap 检测: has_gap[stream] 标记缺失序列，阻止不完整回放        │
│  I20. Image 完整性: is_complete = !has_gap && high_water >= authority   │
└──────────────────────────────────────────────────────────────────┘
```

### 11.3 故障转移不变量

```txt
┌──────────────────────────────────────────────────────────────────┐
│  Failover 协议不变量                                                │
│                                                                  │
│  F1. 自 fencing: promote 前检查自身状态，防止脑裂                      │
│  F2. 复制健康门控: repl_conn 不健康时拒绝 promote                      │
│  F3. CAS 竞争: 两个 standby 同时 promote → 恰好一个 CAS 成功           │
│      → 失败方收到 MDS_ERR_STALE，保持 standby                          │
│  F4. 回滚安全: takeover 后任何失败 → 每条路径 CAS 回原 owner            │
│  F5. 客户端恢复: promote 时加载 partner 的 recovery records 并 adopt   │
│  F6. 优雅降级: 受控 demote 需要 live replication + wire_compat 检查    │
│  F7. 看门狗宽限期: 启动后 20s 内不触发 promote (CLUSTER_STARTUP_       │
│      DEADLINE_MS = 10000)                                           │
│  F8. 心跳合理性: last_heartbeat_ns < 2020-01-01 UTC → INDETERMINATE  │
│      → 永不触发 promote (防止旧 CLOCK_MONOTONIC 域误判)                │
└──────────────────────────────────────────────────────────────────┘
```

# 12 迁移协议：子树所有权转移

### 12.1 实时迁移状态机（`migration.c`）

```txt
┌──────────────────────────────────────────────────────────────────┐
│  migration_state 状态转换                                           │
│                                                                  │
│  IDLE ──► FREEZING ──► STREAMING ──► COMMITTING ──► DONE         │
│              │                                │                   │
│              └────────── FAILED ◄─────────────┘                   │
│                     │                                            │
│                     └──► 解冻 + 恢复 ACTIVE                        │
│                                                                  │
│  8 步协议:                                                         │
│  1. 验证 ACTIVE + 自有 subtree                                      │
│  2. 冻结 (set MIGRATING)                                           │
│  2b. DFS 预收集所有 fileid → 注册到 frozen_fileids                    │
│  3. send BeginMigration (dest_mds_id, path, estimated_inodes)      │
│  4. DFS 流式传输所有 inode chunks                                    │
│     → 每 chunk: fileid + inode(109B) + dirents + stripe_map         │
│       + xattrs + inline_data                                       │
│  5. send FinishMigration (total_inodes)                             │
│  6. subtree_map_transfer_owner_if_migrating (CAS)                   │
│  7. 本地根转 junction (sticky bit 01000)                             │
│  8. 清除 frozen_fileids                                             │
│                                                                  │
│  Wire Format (mig_inode_chunk):                                     │
│    [fileid 8B][inode_serial 109B][dirent_count 4B]                  │
│    For each dirent: [child_fileid 8B][type 1B][name_len 2B][name]   │
│    [has_stripe_map 1B]                                              │
│    If yes: [sc 4B][su 4B][mc 4B] + entries[ds_id 4B + fh_len 4B   │
│    + fh]                                                            │
│    [xattr_count 4B] + xattrs[name_len 2B + name + val_len 4B + val]│
│    [has_inline_data 1B] + inline_data[len 4B + data]                │
│                                                                  │
│  安全约束:                                                           │
│    stripe_count ≤ MDS_MAX_STRIPES (1024)                            │
│    mirror_count ≤ MDS_MAX_MIRRORS (4)                               │
│    xattr name ≤ 255 bytes                                           │
│    inline_data ≤ MDS_INLINE_DATA_MAX (64 KiB)                       │
└──────────────────────────────────────────────────────────────────┘
```

### 12.2 迁移进度追踪（`migration_tracker`）

```txt
┌──────────────────────────────────────────────────────────────────┐
│  migration_tracker 结构                                            │
│                                                                  │
│  字段:                                                            │
│    state: _Atomic int (migration_state 枚举)                       │
│    path: char[MDS_MAX_PATH=4096]                                  │
│    total_inodes: _Atomic uint32_t                                 │
│    done_inodes: _Atomic uint32_t                                  │
│    begin_lock: pthread_mutex_t (防止并发 begin)                      │
│                                                                  │
│  API:                                                             │
│    create / destroy                                               │
│    begin(path, total) → 设置 path + total + state=FREEZING          │
│    update(done_delta) → 原子累加 done_inodes                        │
│    finish(final_state) → 设置最终状态                                │
│    get_progress → 快照 (state, path, total, done)                   │
└──────────────────────────────────────────────────────────────────┘
```

# 13 关键配置参数速查

| 分类 | 参数 | 默认值 | 说明 |
|---|---|---|---|
| **放置** | `placement_policy` | RR | RR / WEIGHTED_RR / CAPACITY |
| | `placement_policy_enabled` | false | 启用配置策略 |
| | `default_stripe_count` | 1 | 默认条带数 |
| | `default_mirror_count` | 1 | 默认镜像数 |
| | `stripe_unit_bytes` | - | 条带单元大小 |
| **HPC** | `hpc_max_stripe_count` | 128 | HPC 宽条带上限 |
| | `hpc_xdr_form` | AUTO | LEGACY / STRIPED / AUTO |
| | `hpc_getattr_mode` | STRICT | STRICT / OPTIMISTIC |
| **提交管道** | `commit_batch_size` | 128 | 单批最大操作数 |
| | `commit_flush_ms` | 2 | 最大攒批等待 |
| | `commit_queue_depth` | 4096 | 队列深度 |
| **租约** | `lease_time_sec` | - | 状态租约时间 |
| | `stripe_lease_duration_ms` | 30000 | 条带租约时长 |
| **缓存** | `inode_cache_size` | - | inode LRU 缓存大小 |
| | `layout_cache_size` | 1024 | layout 缓存条目数 |
| | `transient_state_cache` | false | 内存即权威模式 |
| **异步删除** | `remove_async` | false | 启用异步删除 |
| | `remove_async_workers` | - | worker 线程数 |
| | `remove_async_claim_ttl_ms` | - | claim 租约 TTL |
| **自动分裂** | `auto_split_enabled` | false | 启用自动分裂 |
| | `auto_split_threshold` | 10000 | ops 计数阈值 |
| | `auto_split_sustained` | 2 | 连续热周期数 |
| **DS 健康** | `ds_health_fail_threshold` | 6 | 故障判定阈值 |
| | `ds_heartbeat_ms` | - | 心跳间隔 |
| **分片** | `shard_enabled` | false | 启用命名空间分片 |
| | `referral_strict` | true | 严格 referral 模式 |

# 14 版本与兼容性

| 常量 | 值 | 说明 |
|---|---|---|
| `PNFS_MDS_VERSION_MAJOR` | 0 | 主版本号 |
| `PNFS_MDS_VERSION_MINOR` | 1 | 次版本号 |
| `MDS_WIRE_COMPAT_VERSION` | 1 | 线协议兼容版本 |
| `MDS_SCHEMA_VERSION` | 3 | Catalogue schema 版本 |
| GCC 最低要求 | 11.1 | `_Static_assert` 强制检查 |
