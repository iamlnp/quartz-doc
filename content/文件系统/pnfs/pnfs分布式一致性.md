---
年份:
  - "2026"
月份:
  - 10月
imageNameKey: pnfs分布式一致性
创建时间: 2026-10-29 15:32
修改时间: 2026-10-29 15:41
tags:
  - pnfs
---
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
# 1 NFSv4.1 字节范围锁  
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
# 2 共享保留（Share Reservation）  
| 属性   | 说明                                        |
|------|-------------------------------------------|
| 协议依据 | RFC 8881 §8.2 / §9.1 / §18.16             |
| 访问模式 | READ、WRITE、BOTH                           |
| 拒绝模式 | NONE、READ、WRITE、BOTH                      |
| 冲突处理 | 不兼容的 OPEN 返回 NFS4ERR_SHARE_DENIED         |
| 持久化  | open_state_table_set_cat() → RonDB        |
| 关键文件 | include/open_state.h、src/mds/open_state.c |
共享保留在 OPEN 阶段建立，与字节范围锁互补：前者控制"谁能打开文件做什么"， 后者控制"打开后哪些字节范围互斥"。  

# 3 文件委托（File Delegation）  
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
# 4 目录委托（Directory Delegation）  
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
# 5 pNFS Layout 回收  
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
# 6 Stripe Lease（条带租约）  
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

# 7 跨 MDS 缓存一致性（Active-Active 集群）  
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
# 8 NFSv4.1 回调基础设施  
所有 recall / notify 机制依赖 NFSv4.1 的**回通道**（backchannel），复用 前通道 TCP 连接：

|回调类型|用途|触发场景|
|---|---|---|
| `CB_RECALL` |回收文件委托|委托冲突|
| `CB_LAYOUTRECALL` |回收 pNFS layout|layout 冲突 / DS 下线 / 文件删除|
| `CB_NOTIFY` |目录委托通知|目录变更事件|
| `CB_GETATTR` |查询委托持有者属性|WRITE 委托回收时|

关键文件：`include/nfs4_cb.h`、`include/nfs4_cb_sec.h`、`src/mds/nfs4_cb.c`  

# 9 租约与宽限期  
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

# 10 跨 MDS 协调层  
协调层将协议状态持久化到 RonDB，使所有 MDS 节点可见：

|状态类型|RonDB 行结构|说明|
|---|---|---|
|Open / Share| `mds_coord_open_row` |含 owner_mds_id + boot_epoch（fencing 用）|
|Lock| `mds_coord_lock_row` |含冲突测试逻辑|
|Delegation| `mds_coord_deleg_row` |含冲突扫描|
|Layout| `mds_coord_layout_*` |通过 stateid 查询|

关键文件：`include/mds_coordination.h`

# 11 机制对照表  
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
