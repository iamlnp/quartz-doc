### 2.1.4 元数据分布  
元数据分布机制回答的核心问题是：**元数据如何在 MDS、DS、客户端三者之间产生、流转、同步和回收**。以下按生命周期阶段展开，全部内容已与 `pnfs-lattice` 源码核对（catalogue 层 `mds_catalogue.h` / `mds_coordination.h`，RonDB schema v10，35 张表）。

#### 2.1.4.1 元数据产生机制：DS 放置（Placement）  
当客户端首次对文件发起 LAYOUTGET 且该文件还没有 stripe map（空文件）时，MDS 需要决定该文件的条带映射——即数据分布在哪些 DS 上。这是元数据分布的**起点**（`src/mds/compound_layout.c` op_layoutget；另有 HPC 宽条带 CREATE 预热路径和 ds_prealloc 预分配路径也会调用放置）。

```txt
┌─────────────────────────────────────────────────────────────────┐
│  LAYOUTGET 触发 DS 放置决策（compound_layout.c）                   │
│                                                                 │
│  1. 读取 DS 注册表: cat_ds_list()                                │
│     → layout_filter_for_inode() 过滤 ONLINE + mode/transport     │
│     → layout_drop_iolimit_ineligible() 移除 I/O 限制不达标 DS     │
│                                                                 │
│  2. 确定几何:                                                    │
│     placement_policy_enabled && 配置非零                          │
│       → stripe_count = default_stripe_count (默认1)              │
│         mirror_count = default_mirror_count (默认1)              │
│     否则 → 1×1（历史行为）                                        │
│     stripe_unit = cfg_stripe_unit ?: 65536                       │
│                                                                 │
│  3. 权重叠加（WRR 前置）: ds_cache_overlay_weights()              │
│     → operator 权重 ds_weight.<id> 覆盖到内存权重字段              │
│                                                                 │
│  4. 放置算法:                                                    │
│     policy_enabled ? placement_select_ex2(policy,...)            │
│                    : placement_select2(...)   (纯 RR)             │
│                                                                 │
│  5. 持久化: mds_cat_stripe_map_put(fileid, sc, su, mc, entries)  │
│     1×1 文件走 inode 内联（MDS_IFLAG_INLINE_STRIPE, schema v9）   │
│     多条带/镜像写 mds_stripe_maps + mds_stripe_entries 表         │
└─────────────────────────────────────────────────────────────────┘
```

**核心分配公式**（`placement.c::placement_select_rr_at2`）：
```txt
idx = (start + s * mirror_count + m) % online_count
```
- `start` 有两个来源：
  - `placement_select2`（LAYOUTGET 主路径）：`start = atomic_fetch_add(g_rr_counter, need)`，全局原子计数器每次调用前进 `need = stripe_count × mirror_count`，保证并发调用互不重叠；
  - `placement_select_rr_at2(start_key = fileid)`（`ds_prealloc_batch` 宽条带预热专用）：以 fileid 取模作窗口起点，使宽文件在 50 个 DS 的集群上均匀散布而不是挤在头部热点。
- 同一 stripe 的不同 mirror 落在 online 数组的连续槽位，天然保证**不在同一 DS**（前提 `online_count >= mirror_count`）。

**三种放置策略**（`enum mds_placement_policy`，仅 `placement_policy_enabled=true` 时生效，对新文件生效、旧文件保持原 stripe map）：

| 策略 | 选择逻辑 | 权重来源（优先级从高到低） |
|---|---|---|
| PLACEMENT_RR | 全局原子计数器轮转 | 无权重 |
| PLACEMENT_WEIGHTED_RR | 按权重加权随机（`mds_wrr_weighted_pick`） | ① operator 配置 `ds_weight.<id>` → ② 容量自动权重 auto_weight（`placement_capacity_weighting=PROPORTIONAL` 时由 statvfs 探测推导：`max(1, floor((1-used/total)×100))`，bounded [1,100]）→ ③ 上报空闲字节 (total-used) → ④ 常量 1 |
| PLACEMENT_CAPACITY | 选空闲字节最大的 DS（`mds_wrr_capacity_pick`） | 同上 |

注意：WRR/CAPACITY 的选核函数在 `src/modules/wrr/` 可选模块中，**社区版构建只有 stub（恒返回 0），外围 walk-forward 逻辑会把这两种策略降级为 online 集合上的纯 RR**；且 WEIGHTED_RR/CAPACITY 的多条带放置用本地 taken 位图保证 stripe 级 DS 互异。

**降级处理**（`placement_select2` / `placement_select_ex2`，stripe_count 为 in/out 参数）：
- `online_count < mirror_count` → `MDS_ERR_NOSPC`；
- `online_count < stripe_count × mirror_count` → 缩减到 `sc = online_count / mirror_count`（最小 1），并 `atomic_fetch_add(pnfs_mds_placement_degraded_total)`；
- **调用方必须用返回的 effective_stripe_count 持久化**：若按请求值持久化，calloc 未填充的"幻影条带"永远无法就绪，会把 LAYOUTGET 卡进 NFS4ERR_DELAY 死循环。

**其他工程细节**：
- 栈上暂存数组上限 64 个 DS（`MDS_PLACEMENT_STACK_MAX`），超出走堆并打点 `pnfs_mds_placement_heap_fallback_total`；集群上限 `MDS_MAX_DS_NODES=256`。
- 兼容性过滤器三件套：`ds_filter_compatible`（mode+transport 硬过滤）、`ds_filter_with_caps`（再加 capabilities）、`ds_filter_compatible_preferred`（硬地板 + 软偏好，用于 HPC-Shared 文件：有 RDMA|GPUDirect DS 时优先用，没有则优雅回退 TCP 全集）。
- 镜像/分层搬迁的配套选核：`placement_select_replacement`（指定 target 则校验 ONLINE + 不在排除表，否则第一个可用者）与 `placement_select_for_tier`（tier 内空闲最多者）。
- **既有文件不重条带**：新增 DS 不会改变已创建文件的 stripe map（`compound_layout.c` Issue #6 注释）。

**放置结果即元数据分布的物理映射**：一个 `stripe_count × mirror_count` 的矩阵，每个条目记录 `(ds_id, nfs_fh)`。schema v9 起，1×1 文件的这唯一一条目不落 side table，而是内联在 inode 行（`inline_ds_id` / `inline_fh` / `inline_stripe_unit`，置 `MDS_IFLAG_INLINE_STRIPE`），LAYOUTGET / unlink fence / GC 一次 inode 读即可服务，create/remove 不再写/删 stripe 行。
#### 2.1.4.2 元数据写入机制  
元数据修改不是直接写后端，而是经过一个分层管道：
```txt
┌──────────────────────────────────────────────────────────────────┐
│  RPC 工作线程                                                       │
│  (compound_layoutget / op_open / op_create / ...)                 │
│       │                                                          │
│       ▼                                                          │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Layer 1: 内存写（即时生效）                                 │    │
│  │                                                          │    │
│  │  open_state_table → 本地 hash 表插入 (默认 2^20 桶/1024 锁条)│    │
│  │  layout_cache     → 16-shard LRU put                      │    │
│  │  deleg_table      → 本地 hash 表插入 (16 条带锁)            │    │
│  │  stripe_lease_table → 纯内存租约获取                        │    │
│  │                                                          │    │
│  │  效果: 本 MDS 立即可见；其他 MDS 不可见                      │    │
│  └──────────────────────┬───────────────────────────────────┘    │
│                         │ write-through                            │
│                         ▼                                          │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Layer 2: Commit Queue（同步直通分发器）                     │    │
│  │                                                          │    │
│  │  现实现: commit_queue_submit() 在【调用方线程】上同步        │    │
│  │  经 catalogue vtable 直发后端——无写线程、无队列、无攒批。      │    │
│  │  历史形态(写入线程/环形缓冲/txn_delta_builder)已在            │    │
│  │  Level-B 清理中移除；batch_size/batch_max_bytes/flush_ms/    │    │
│  │  queue_depth 参数仅为 API 兼容保留，传入即忽略。              │    │
│  │                                                          │    │
│  │  18 种操作描述符: CREATE/REMOVE/RENAME/LINK/SETATTR/         │    │
│  │  INLINE_WRITE/INLINE_DEL/INODE_PUT/RECOVERY_PUT/DEL/         │    │
│  │  DS_STATE/LAYOUT_STATE_PUT/DEL/STRIPE_MAP_PUT/               │    │
│  │  REBALANCE_MOVE/QUOTA_ADJUST/DS_REGISTER/DS_REMOVE           │    │
│  │                                                          │    │
│  │  核心不变量: NFS 回复不先于 catalogue 写入发送                │    │
│  │  (submit 阻塞到后端落库才返回，天然成立)                      │    │
│  └──────────────────────┬───────────────────────────────────┘    │
│                         │                                          │
│                         ▼                                          │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Layer 3: Catalogue 后端（共享持久化）                       │    │
│  │                                                          │    │
│  │  事务契约 (mds_catalogue.h "Transaction control"):         │    │
│  │  · 每个 mds_cat_* / mds_coord_* 调用自包含原子提交            │    │
│  │  · mds_cat_txn 只是分组令牌，abort 不会回滚任何已提交行         │    │
│  │  · 多行原子性【只】存在于融合操作:                            │    │
│  │    ns_create_wide        = inode+dirent+stripe map+parent   │    │
│  │    ns_create_with_layout = ns_create + layout_state 行      │    │
│  │    ns_remove_known_gc    = dirent+inode+GC 行（最终unlink）  │    │
│  │    ns_rename[_flags]     = 双侧 dirent + 双亲 (+ victim)    │    │
│  │    coord_layoutget_fused = stripe map 读 + layout grant     │    │
│  │    coord_layout_grant_union = 续租区间饱和并集（防欠召回）     │    │
│  │                                                          │    │
│  │  效果: 所有 MDS 立即可见（RonDB 分布式一致性）               │    │
│  └──────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

**关键开关**：
- `transient_state_cache`（默认 false，`pnfs_mds.h`）：false 时 open/layout 状态 write-through 到 RonDB 共享协议状态表（`mds_open_state`、`mds_delegations`、`mds_byte_range_locks`、`mds_sessions` 等，行内带 `owner_mds_id + owner_boot_epoch` 供 fencing/故障收割）；true 时跳过 NDB 写、内存即权威，仅单 MDS 安全（多 MDS 部署 main.c 会打日志告警）。
- `mds_coord_shared_state_supported(cat)` 为 false 的后端（如 memdb 之外的部分场景），协议状态表一律 NOSUPPORT，等同内存权威。
- CREATE+LAYOUTGET 融合：`commit_op_create.layout_pregrant` 让 writer 在 CREATE 同一事务里顺带持久化 layout_state（best-effort，失败不阻塞 CREATE）；`layoutget_fused` 则在后端单事务里完成 stripe map 读取 + grant，省一次网络往返。
#### 2.1.4.3 跨 MDS 元数据同步机制  
多 MDS 部署下，元数据如何在节点间保持一致？  
```txt
┌──────────────────────────────────────────────────────────────────┐
│               跨 MDS 元数据同步架构                                │
│                                                                  │
│   ┌────────────────────────────────────────────────────────┐     │
│   │  机制 A: 共享 Catalogue（主路径，唯一权威）                │     │
│   │                                                        │     │
│   │  RonDB(FDB/DAOS 同构) 作为 shared authority             │     │
│   │  所有 MDS 直接读写同一后端                                │     │
│   │  → inode/dirent/stripe_map/ds_registry 天然一致          │     │
│   │  → 协议状态(open/deleg/lock/session/layout) 经协调表共享  │     │
│   │  → 权威判断: mds_catalogue_shared_authority()           │     │
│   └────────────────────────────────────────────────────────┘     │
│                                                                  │
│   ┌────────────────────────────────────────────────────────┐     │
│   │  机制 B: Delta Broadcast + Catalog Image（加速路径）      │     │
│   │                                                        │     │
│   │  写入侧: 锁定变更包装器在变更成功后调用 rondb_emit_delta   │     │
│   │    → 组装 (type, stream_id=mds_id, seqno++, ts, payload) │     │
│   │    → 插入 mds_delta_broadcast                            │     │
│   │       PK=(source_mds_id, seqno), 带 boot_epoch            │     │
│   │    · changefeed_enabled 时 delta 插入失败 ⇒ 该变更        │     │
│   │      整体向调用方报失败（防止 authority 与 image 永久分叉） │     │
│   │    · seqno 取自 mds_meta.delta_seqno_counter，             │     │
│   │      定期批量落盘 + 启动时加安全余量；崩溃后撞主键则         │     │
│   │      自增重试(≤128次)保证 delta 不丢                      │     │
│   │    · 当前 C 包装器在 rename 等锁定路径发射                  │     │
│   │      DIRENT_DELETE/PUT，镜像 shim 层可覆盖更多类型          │     │
│   │                                                        │     │
│   │  消费侧: 后台 poller 线程                                │     │
│   │    1. 从 mds_node_registry 发现外国 MDS 流(≤16 个)        │     │
│   │    2. 按 per-stream last_applied 游标批量拉取(每批100)    │     │
│   │    3. catalog_image_apply: 只回放 INODE_UPSERT/DELETE +   │     │
│   │       DIRENT_PUT/DELETE（image v1 范围 = inode+dirent）   │     │
│   │    4. 每 60s 按所有流的最小 applied seqno trim 广播表      │     │
│   │                                                        │     │
│   │  catalog_image: 只读加速镜像，【不是】权威；内部 rwlock；    │     │
│   │  四种生命周期模式 (catalog_image_mode):                   │     │
│   │    OFF     : 无 image，全部读 authority（默认）            │     │
│   │    SHADOW  : image 回放但回复来自 authority                 │     │
│   │    COMPARE : 热读比较 image vs authority                   │     │
│   │    PRIMARY : 热读 image，失败回退 authority                 │     │
│   │  前置条件: mode != OFF 要求 catalog_replay_mode=JOURNAL    │     │
│   │  (持久日志，与 authority 写协同提交)                        │     │
│   └────────────────────────────────────────────────────────┘     │
│                                                                  │
│   ┌────────────────────────────────────────────────────────┐     │
│   │  机制 C: 2PC Journal（跨 MDS/跨分片 rename、hardlink）    │     │
│   │                                                        │     │
│   │  共享日志表 mds_rename_journal (通用 mds_coord_journal):  │     │
│   │    txn_id | state(PREPARED/COMMITTED/ABORTED) |          │     │
│   │    role(COORDINATOR/PARTICIPANT) | remote_mds_id |        │     │
│   │    双侧 parent/name | inode 快照 payload | created_at     │     │
│   │                                                        │     │
│   │  Phase 1 (PREPARE): 协调方+参与方各写 PREPARED 日志       │     │
│   │  Phase 2 (COMMIT):  通知参与方执行 → journal 记 COMMITTED │     │
│   │  崩溃恢复: 启动时 rename_2pc_recover() 扫描 journal,      │     │
│   │            PREPARED → 决定提交或 ABORTED                  │     │
│   │                                                        │     │
│   │  并发控制事实: 跨 MDS rename 现以单把 TOPOLOGY 锁          │     │
│   │  (mds_ns_locks class 0, 全局单例) 串行化——更早的          │     │
│   │  PARENT_NAME(src)+(dst) 细粒度锁已证明冗余并移除           │     │
│   │  (省 4 次 NDB 往返/次 rename)。锁的一般规则仍是按           │     │
│   │  (resource_class, partition_hint, key) 升序获取防死锁:     │     │
│   │  class 0 TOPOLOGY / 1 PARENT_NAME / 2 DIR_MUTATION /      │     │
│   │  3 INODE_ATTR / 4 FILE_STATE / 5 LOCK_TABLE /              │     │
│   │  6 CLIENT_SESSION / 7 SLOT_DRC                             │     │
│   └────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

#### 2.1.4.4 分片路由机制：操作如何落到正确的 MDS  
多 MDS 部署中，命名空间被分成多个 shard（分片），每个 shard 由一个 MDS 拥有。**与直觉不同，运行时路由并不依赖"查表"，而是依赖线 FH 内嵌的 owner 信息**：
```txt
┌──────────────────────────────────────────────────────────────────┐
│  分片路由流程（实际实现）                                            │
│                                                                  │
│  0. FH 格式: v1 FH 在 LOOKUP/GETFH 时烙入 owner_mds_id            │
│     → 客户端缓存的深部 FH 自带路由信息，0 额外 catalogue I/O        │
│                                                                  │
│  1. 客户端 PUTFH(filehandle)                                      │
│     · 不做 referral 检查（junction 目录 inode 是本地元数据）        │
│     · fileid==0 视为畸形 FH → NFS4ERR_BADHANDLE                  │
│     · 安装 current_fh; 单分片部署跳过 fileid→shard 解析            │
│       (省 ~200µs/PUTFH); 多分片时可选                              │
│       mds_cat_shard_fileid_get() —— 注意该 vtable 槽位            │
│       RonDB 后端【不填充】(无此表)，FDB/DAOS/memdb 才有实现        │
│     · compound_inode_get 读 inode: 命中 icache 则免 NDB PK 读;    │
│       fileid 已删(单调不复用) → NFS4ERR_STALE (RFC 8881 §15.1.2.4)│
│                                                                  │
│  2. MOVED 门控: compound 调度器逐 op 检查                         │
│     referral_strict=true && current_fh.owner_mds_id != self      │
│     → 白名单放行(SEQUENCE/EXCHANGE_ID/CREATE_SESSION/PUTFH/       │
│       GETFH/GETATTR(FS_LOCATIONS 时, RFC 8881 §8.5.1) 等)        │
│     → 其余 op 一律 NFS4ERR_MOVED + fs_locations referral         │
│                                                                  │
│  3. 路径级路由: LOOKUP 命中 junction 目录(目录 sticky bit 标记)   │
│     → NFS4ERR_MOVED + fs_locations                               │
│     跨分片目录项经 mds_ext_dirents 表:                           │
│     (parent, name) → (owner_mds_id, target_fileid, anchor_id)    │
│                                                                  │
│  4. 路由数据结构:                                                  │
│     · subtree_map: 内存最长前缀匹配表; pm_backed 条目持久化在      │
│       mds_partition_map (partition_id, owner_mds_id,             │
│       state=ACTIVE/MIGRATING/FROZEN, subtree_path);              │
│       "/" 根所有权 insert-only 抢占, /shardN 幂等 upsert 播种,    │
│       所有权变更先 CAS 落库再改内存                                 │
│     · home_shard_id: 内嵌在 mds_inodes 行 (inode 线像 137B 基础   │
│       + v8 synth 尾 + v9 inline stripe 尾 = 285B)                 │
│     · mds_shard_map: 请求级最长前缀路由 → 每 shard 独立 cat+cq    │
│       (MDS_SHARD_MAX=64), compound_data.current_shard 跨整个      │
│       COMPOUND 持有                                               │
└──────────────────────────────────────────────────────────────────┘
```

#### 2.1.4.5 元数据缓存一致性机制  
内存缓存与共享 Catalogue 之间的一致性维护：  

| 缓存 | 写入策略 | 失效触发 | 一致性保证 |
|---|---|---|---|
| layout_cache | LAYOUTGET 产生新 stripe map 时 put（16-shard LRU，默认 1024 条目，条目为堆拷贝） | remove(final unlink) / setattr(shrink) 时 invalidate；缓存命中即跳过 catalogue 往返 | 最终一致；HPC-Shared 文件 stripe map 生命周期内稳定 |
| open_state_table | open/close 本地写 + 经 `mds_coord_open_*` write-through（默认 2^20 桶、1024 锁条） | CLOSE / lease 过期 / 崩溃恢复 / failover 收割 | `shared_state_supported && !transient` 时 RonDB 权威；否则内存权威 |
| deleg_table | grant/return 本地写 + `mds_coord_deleg_*` write-through（16 条带锁） | recall / revoke / file unlink | 同上 |
| lock_state | LOCK/LOCKU 本地 + `mds_coord_lock_*` write-through（16 条带锁） | LOCKU / lease 过期 / reap_client | 同上 |
| session_table | EXCHANGE_ID/CREATE_SESSION 本地写（默认 65536 桶） | lease 过期 / DESTROY_SESSION；reaper 线程收割 | clientid 按属主 MDS 亲和；内存权威 + 可选持久化 |
| ds_cache | 启动全量加载 + 管理员命令 reload | ds add/remove/set-state/provision/patch-ready | 启动时一致；管理员操作后显式刷新；叠加 weight/capacity（含远端观测合并）供 WRR 与 GETATTR SPACE_* 聚合 |
| stripe_lease_table | 纯内存，不持久化（key=(fileid, stripe_index, ds_offset)） | lease 过期 / LAYOUTRETURN release_all | 单 MDS 内存权威；冲突 → LAYOUTTRYLATER 或按 prefix_conflict_free_length 缩窄授予 |
| inode_cache / dirent_cache | 16 条纹 LRU | 命名空间变更即时失效（CREATE/REMOVE/RENAME/LINK/SETATTR 各失效对应集合） | 单 MDS 正条目无界；多 MDS（cluster_size>1）默认加 ~1s 正条目 TTL + 5s 负条目 TTL |

transient_state_cache 开关（`pnfs_mds.h`，默认 false）：
- false（默认）：open/delegation/lock 状态 write-through 到 RonDB，多 MDS 安全
- true：仅内存（`open_state_table_set_skip_ndb`），单 MDS 部署优化，崩溃后客户端通过 reclaim 重建

#### 2.1.4.6 元数据回收机制  
元数据不会永久驻留，以下机制负责回收：  
```txt
┌──────────────────────────────────────────────────────────────────┐
│  元数据回收路径                                                     │
│                                                                  │
│  A. GC 队列 (ds_gc, src/modules/ds_gc)                            │
│     触发: 文件最终 unlink (nlink→0)，经融合操作 ns_remove_known_gc │
│     在同一后端事务里提交 dirent+inode+GC 行，关闭"name 已删但     │
│     DS 对象未入队"的崩溃窗口（不支持融合时走 legacy 分立路径）      │
│     流程: gc_enqueue_hint(fileid, ds_id, nfs_fh, sweep_hint)      │
│           → mds_gc_queue 表 (gc_seq 单调, ix_gc_queue_seq 有序索引)│
│           → 协调线程 peek_batch(默认256) → 有界队列                 │
│           → ds_gc_workers(默认4, [1,32]) 个 worker 各自消费不相交行 │
│           → proxy unlink DS 文件 → gc_dequeue                     │
│     分布: owner_mds_id 列使每个 MDS 只 drain 自己的行               │
│           (0=未分配, 任何 MDS 可收)                                 │
│     sweep_hint: MDS_GC_SWEEP_GEOM(sc,mc) 整文件回收 /               │
│                 MDS_GC_SWEEP_SLOT(s,m)(bit31) 单槽回收,             │
│                 0=legacy dense sweep(仅对 1×1 文件正确)             │
│     语义: DS 侧 ENOENT 视为成功; 永久性错误阻断 dequeue,            │
│           队列不会静默丢工作                                        │
│                                                                  │
│  B. 异步删除清单 (remove_manifest, schema v10)                      │
│     触发: remove_async=true 时的 unlink                           │
│     流程: 1) 内存 TOMBSTONE (dir_fileid, name) 立即遮蔽            │
│             LOOKUP/OPEN/READDIR                                    │
│           2) mds_remove_pending 单行插入 (durable-before-ack)      │
│           3) 立即回复客户端; DRAINER 后台 claim(租约 claim_ttl)    │
│             → 执行受守卫的 ns_remove + 最终 unlink 清理            │
│     崩溃安全: manifest 行先于 ACK 落库 → MDS 崩溃不会复活           │
│             已确认的删除; 启动时先 reload 重建 tombstone 再开 RPC   │
│     INDOUBT: 提交结果不可判 → 返回 NFS4ERR_IO 且【禁止】回退        │
│             同步路径(防重复 GC)                                    │
│     部署门: v1 仅单 MDS 或 strict-subtree 多 MDS                   │
│             (tombstone 可见性是节点本地的)                          │
│                                                                  │
│  C. Layout state 回收                                              │
│     触发: LAYOUTRETURN / CB_LAYOUTRECALL(字节范围冲突、DS 故障) /   │
│           最终 unlink 强制撤销 / lease 过期                         │
│     流程: revoke_layout → layout_state 行删除 + ds_layout_idx 删除 │
│           + stripe_lease_release_all_for;                          │
│           字节范围撤销后对 DS 文件 fencing (chown 中立身份,          │
│           RFC 8435 §14)                                            │
│                                                                  │
│  D. Session/Open/Delegation 回收                                   │
│     触发: lease 过期 (session reaper)                               │
│     流程: open_state_close_all_for_client → deleg_revoke_client    │
│           → session_destroy_client → 可选 DS fencing                │
│                                                                  │
│  E. 孤儿 inode 与恢复记录                                          │
│     · MDS_IFLAG_UNLINK_ORPHAN: rename 覆盖最后链接但仍有本地         │
│       open → 保留 inode 行, 最后一次 CLOSE 才 finalize              │
│       (GC DS 对象 + stripe 行 + inode 行 + quota)                  │
│     · MDS_IFLAG_DELETE_PENDING: delete-at-ack, 行待后台 finalize   │
│     · 客户端恢复记录: RECLAIM_COMPLETE → mds_coord_recovery_del;    │
│       failover 时按 owner CAS 批量转移 (recovery_transfer)          │
└──────────────────────────────────────────────────────────────────┘
```

#### 2.1.4.7 元数据重分布机制：Rebalance & Resilver  
当 DS 拓扑变化（故障、扩容、均衡）时，元数据需要重新映射。**注意：rebalance / resilver / tiering 均为管理员触发的企业版可选模块（社区版构建只有 stub，API 存在但不可用）**：

```txt
┌──────────────────────────────────────────────────────────────────┐
│  Rebalance (管理员触发, 镜像搬迁)                                    │
│  场景: 将 DS-A 上的镜像数据迁移到 DS-B                               │
│  约束: 单 stripe 文件, 无活跃 layout/writer, dense 文件             │
│  流程:                                                             │
│  1. scan: cat_stripe_map_scan → 找到 DS-A 上的文件                  │
│  2. 检查: open_state(无活跃 writer) + layout_state(无活跃 layout)   │
│  3. copy: proxy_read(DS-A) → proxy_write(DS-B)                    │
│  4. atomic swap: COMMIT_OP_REBALANCE_MOVE 单事务完成                │
│       mds_cat_stripe_map_put(新 entry) + gc_enqueue_hint(旧槽位)    │
│       gc_sweep_hint = MDS_GC_SWEEP_SLOT(stripe, mirror)            │
│  5. 重复直到所有候选处理完毕                                          │
├──────────────────────────────────────────────────────────────────┤
│  Resilver (管理员触发, 降级修复; main.c 注释明确 "not auto-started") │
│  场景: DS-A 故障后, 从其健康镜像副本重建到替换 DS-B                    │
│  流程: 找到 degraded mirror 文件 → 从健康 mirror 读 → 写 target_ds  │
│       → 更新 stripe_map + gc_enqueue 旧条目                         │
│       → placement_select_replacement 选替换 DS(可指定 target 校验)   │
├──────────────────────────────────────────────────────────────────┤
│  DS 故障时的元数据连锁反应（自动部分）                                │
│                                                                  │
│  ds_health_monitor (NFSv3 NULL 探测, DS 导出的是 v3):              │
│  · 周期 ds_heartbeat_ms (默认 5000ms), 单次超时 = 周期/2           │
│  · 连续失败 ≥ ds_health_fail_threshold (默认 6) → DS_OFFLINE      │
│  · 防抖动: OFFLINE 后冷却 30s 起, 每次 flap 翻倍至 8min 封顶;       │
│    恢复需连续成功 ≥ max(3, threshold/2); 稳定在线 60s 清零 flap     │
│  · 状态变更: 优先走 CQ COMMIT_OP_DS_STATE(ds_get+ds_put 改写)      │
│    (顺带复制到 Standby), 无 CQ 时直发 catalogue vtable             │
│  → ds_fail_cb 回调 (main.c 注册为 layout_recall_for_ds):           │
│    1. mds_coord_ds_layout_idx_scan(ds_id) → 收集所有 (clientid, fileid)│
│    2. 按唯一 clientid 快照 backchannel 目标, 发送 CB_LAYOUTRECALL(ALL)│
│    3. 无论 CB 成败, 权威撤销全部受影响 layout_state 行              │
│       (未响应客户端在下次 SEQUENCE/LAYOUTGET 时发现)                │
│  → ds_io_limits 层: I/O 限制探测(ds_iolimit_probe_ms, 默认60s,      │
│    NFSv3 FSINFO 取真实 rtmax/wtmax) 变化时递增 io_generation →       │
│    新 LAYOUTGET 携带新 deviceid → 客户端重新 GETDEVICEINFO          │
│  → 后续 placement 过滤 OFFLINE DS; DS 恢复探测成功后自动标 ONLINE    │
│    并重新进入候选集 (resilver/rebalance 由管理员决定何时重建数据)     │
└──────────────────────────────────────────────────────────────────┘
```

#### 2.1.4.8 Device ID 编码机制：元数据寻址桥梁  
Device ID 是连接**逻辑元数据**和**物理 DS 地址**的桥梁：  
```txt
┌──────────────────────────────────────────────────────────────────┐
│  Device ID 结构 (16 bytes, compound_layout.c::build_deviceid)      │
│  ┌──────────────┬──────────────┬───────────────┬──────────────┐  │
│  │ mds_id (4B)  │ ds_id (4B)   │ generation(4B)│ zeros (4B)   │  │
│  │ Big-Endian   │ Big-Endian   │ Big-Endian    │              │  │
│  └──────────────┴──────────────┴───────────────┴──────────────┘  │
│                                                                  │
│  编码: build_deviceid(mds_id, ds_id, io_generation)               │
│  解码: deviceid_to_ds_id(deviceid) → 取 ds_id 4B (忽略 generation) │
│                                                                  │
│  ⚠️ generation 的语义: DS 的 I/O 限制代际 (Wave 5 T5.1),           │
│     来自 ds_io_limits_get(ds_id, ..., &io_gen) —— 即 rtmax/wtmax  │
│     探测值变化时递增, 使旧 deviceid 失效、客户端重新 GETDEVICEINFO  │
│     以拿到新 rsize/wsize; 【不是】DS 重新注册计数器。               │
│     I/O 限制探测: ds_iolimit_probe_ms (默认 60000), 关闭时回退     │
│     legacy 1MiB 常量。                                             │
│                                                                  │
│  用途:                                                           │
│  ┌─────────────────────────────────────────────────────────┐     │
│  │  LAYOUTGET 响应:                                         │     │
│  │    ff_data_server4.deviceid = build_deviceid(...)         │     │
│  │    → 客户端收到 deviceid + nfs_fh + ffl_user/ffl_group    │     │
│  │                                                         │     │
│  │  GETDEVICEINFO 请求:                                     │     │
│  │    客户端发送 deviceid                                    │     │
│  │    MDS 解码 → ds_id → 查 ds_cache/ds_registry →           │     │
│  │    返回 host/port/tcp_port/rdma_port/rsize/wsize         │     │
│  │    endpoint 集由 ds_getdev_transport (tcp/rdma/both) 决定  │     │
│  └─────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────┘
```

#### 2.1.4.9 Delta 记录机制：元数据变更追踪  
每次 Catalogue 写操作都会产生语义化的 delta 记录（`catalog_delta.h`）：  
```txt
┌──────────────────────────────────────────────────────────────────┐
│  Delta 记录格式 (25 字节头 + 变长 payload)                          │
│                                                                  │
│  type(1B) + stream_id(4B) + seqno(8B) + timestamp_ns(8B)         │
│  + payload_len(4B) + payload(变长)                                 │
│                                                                  │
│  22 种语义类型 (enum catalog_delta_type, 1..22):                  │
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
│  三个消费方:                                                       │
│  1. mds_delta_broadcast 表 → 跨 MDS 的 catalog_image 回放(见2.1.4.3)│
│  2. 持久日志 (catalog_replay_mode=JOURNAL) → 崩溃重建,             │
│     与 authority 写协同提交; mode!=OFF 的强制前置条件               │
│  3. 文件日志 (catalog_replay_mode=LOG) → 调试/审计(非持久)          │
│                                                                  │
│  stream/seqno 设计:                                                │
│  单 MDS: stream_id=0; 多 MDS: stream_id=MDS_id, 每流独立单调 seqno  │
│  RonDB 实现: seqno 由 mds_meta.delta_seqno_counter 分配,           │
│    定期批量落盘(DELTA_PERSIST_INTERVAL), 启动加载时加安全余量;      │
│    崩溃后若与上一 incarnation 的行撞主键, 自增 seqno 重试(≤128)     │
│  → catalog_image 以 per-stream high_water 追踪回放进度             │
└──────────────────────────────────────────────────────────────────┘
```
#### 2.1.4.10 核心不变量  
##### 2.1.4.10.1 NFS 回复不先于 catalogue 写入  
**在 MDS 向客户端发送任何表示"成功"的 NFS 回复之前，使该回复为"真"的所有 Catalogue 写操作必须已经持久化完成。**  这是一个**写前日志（Write-Ahead）**原则的变体。当前实现靠 `commit_queue_submit()` 的同步阻塞语义天然成立；`mds_backup_*` 等异步化改造若发生，此不变量的实现机制必须同步重做。

##### 2.1.4.10.2 effective_stripe_count 必须持久化  
放置降级后，调用方必须用返回的有效条带数写 stripe map。持久化请求值会记录 calloc 零填充的幻影条带，导致 LAYOUTGET 永远 NFS4ERR_DELAY。

##### 2.1.4.10.3 同一 stripe 的 mirror 不共置同一 DS  
放置算法的数组连续槽位分配 + WEIGHTED_RR/CAPACITY 的 taken 位图共同保证；`online_count < mirror_count` 时宁可 NOSPC 也不破坏。

##### 2.1.4.10.4 remove_async 下 durable-before-ack  
tombstone + mds_remove_pending 行必须先于客户端 ACK 落库，否则 MDS 崩溃会复活已确认的删除。

##### 2.1.4.10.5 changefeed 下 delta 与变更同生死  
`changefeed_enabled` 时 delta 插入失败 ⇒ 整个变更向调用方报失败，authority 与 image 不允许静默分叉。

##### 2.1.4.10.6 pm_backed 所有权先落库后改内存  
subtree 所有权变更先对 mds_partition_map CAS，成功后才更新内存条目；任何后续 refresh 都不会回滚内存状态。
