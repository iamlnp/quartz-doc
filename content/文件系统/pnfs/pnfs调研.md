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

### 2.1.2 数据流  
```bash
1. Client → MDS    OPEN("/data/file")       — MDS 在 RonDB 中解析 inode，返回 stateid
2. Client → MDS    LAYOUTGET(stateid)       — MDS 返回 flex file layout: DS 地址 + 每 stripe 的 NFS FH
3. Client → DS     NFS READ/WRITE           — 客户端直接与 DS 通信（并行 stripes）
4. Client → MDS    LAYOUTRETURN on close    — 释放 layout 状态
```
### 2.1.3 概念说明  
#### 2.1.3.1 MDS 空间 = 命名空间（Namespace）

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

##### 2.1.3.1.1 Shard和MDS的对应关系  
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

#### 2.1.3.2 数据空间 = DS 的本地存储  
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

#### 2.1.3.3 数据访问路径  
pNFS 模式（客户端支持 LAYOUTGET）：  
```bash
客户端 → MDS（获取 Layout）→ 返回"文件 X 的数据在 DS-1 的 stripe 0"
客户端 → DS-1（直接读写数据）
```
 Proxy I/O 模式（客户端不支持 pNFS）：  
```bash
客户端 → MDS（读写请求）
MDS → DS-1（通过 NFS 挂载代理读写）
MDS → 客户端（返回结果）
```

### 2.1.4 数据分布  
Stripe 是 pNFS（parallel NFS, RFC 8881 / RFC 8435）的核心数据分布机制。一个文件的逻辑数据被切分成固定大小的块（**stripe unit**），轮流分布到多个数据服务器（DS）上，每个 DS 存储文件的一个 **stripe**。  
关键参数定义在 `pnfs_mds.h`：  

| 参数           | 含义                     | 上限                     |
|--------------|------------------------|------------------------|
| stripe_count | 每个文件的条带数（数据分布在几个 DS 上） | MDS_MAX_STRIPES = 1024 |
| stripe_unit  | 每个条带的大小（字节），即轮转粒度      | 默认 65536，推荐 1 MiB      |
| mirror_count | 每个 stripe 的副本数         | MDS_MAX_MIRRORS = 4    |

### 2.1.5 数据备份与冗余机制  
pNFS-lattice **支持数据镜像**，但需要主动配置：
#### 2.1.5.1 Mirror（镜像）机制  
| 配置                                   | 说明                           |
| ------------------------------------ | ---------------------------- |
| mirror_count = 1（默认）                 | 无冗余，每个 stripe 只有一份数据         |
| mirror_count = 2                     | 每个 stripe 有 2 份副本，分布在不同 DS 上 |
| mirror_count = N                     | 每个 stripe 有 N 份副本            |
| 数据文件命名格式：`{inode}_{stripe}_{mirror}` |                              |
```bash
DS-1                          DS-2                          DS-3
┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
│ /export/ds/data/│          │ /export/ds/data/│          │ /export/ds/data/│
│   5_0_0 (镜像0)  │◄────────►│   5_0_1 (镜像1) │          │                 │
│   6_0_0         │          │                 │          │                 │
└─────────────────┘          └─────────────────┘          └─────────────────┘

mirror_count = 2 的文件 5
```

#### 2.1.5.2 Resilver（自动数据重建）  
当一个 DS 故障时，系统会**自动从镜像副本重建数据**：
```bash
DS-1 (健康)                   DS-2 (故障)                   DS-3 (新)
┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
│   5_0_0 (镜像0)  │──────────│   5_0_1 (丢失)  │─────────►│   5_0_1 (重建)   │
└─────────────────┘          └─────────────────┘          └─────────────────┘
      源数据                         ✗                        自动复制
```

- 当 DS-2 故障时，resilver 线程会从 DS-1 复制数据到 DS-3
- 自动更新 stripe map，将故障 DS 替换为新 DS
- 整个过程对客户端透明

#### 2.1.5.3 DS 健康监控  
`ds_health.c` 实现了：
- 定期心跳检测（ds_heartbeat_ms 配置）
- 故障阈值（连续 N 次失败后标记 OFFLINE）
- 自动恢复检测（DS 恢复后自动标记 ONLINE）
- 防抖动机制（避免频繁状态切换）

#### 2.1.5.4 生产环境建议  
| 场景            | 推荐配置                     |
|---------------|--------------------------|
| 开发/测试         | mirror_count = 1（节省存储）   |
| 生产环境（重要数据）    | mirror_count = 2 或 3     |
| HPC-Shared 文件 | 当前不支持镜像（代码限制）            |
| DS 底层存储       | 建议使用 RAID 或分布式存储（如 Ceph） |

### 2.1.6 源码模块结构  
| 目录               | 职责               | 核心文件                                                              |
| ---------------- | ---------------- | ----------------------------------------------------------------- |
| `src/mds/`       | NFSv4.1/4.2 协议处理 | compound.c (调度器), rpc_server.c, main.c                            |
| `src/catalogue/` | 可插拔元数据后端         | catalogue_rondb_shim.cpp, catalogue_memdb.c, catalogue_dispatch.c |
| `src/cluster/`   | MDS 间协调          | cluster_transport.c, rename_2pc.c, failover.c                     |
| `src/fsal_obj/`  | FSAL 对象抽象        | layout.c, placement.c                                             |
| `src/common/`    | 共享基础设施           | config.c, log.c, threadpool.c                                     |
| `src/tools/`     | CLI 工具           | mds-admin, mds-find, mds-apid, mds-backend-probe                  |
| `src/modules/`   | 可选模块             | ds_gc, ds_prealloc, layout_cache, observability 等                 |
| `include/`       | 公共 C 头文件         | 90+ 头文件，每个子系统一个                                                   |
| `tests/`         | 测试套件             | 82 个单元测试 + 集成测试 + 一致性测试                                           |
### 2.1.7 进程模型（线程组）  
单个 `pnfs-mds` 进程内包含多个线程组：  

| 线程组               | 来源                         | 角色                         |
| ----------------- | -------------------------- | -------------------------- |
| RPC listeners     | `rpc_server.c`             | 接受连接，解码 COMPOUND           |
| Compound workers  | `compound.c`               | 执行 compound_process()      |
| Catalogue I/O     | `catalogue_rondb_shim.cpp` | 驱动 NDB 事务                  |
| DS GC             | `ds_gc.c`                  | 清理孤立 stripe 行              |
| DS health         | `ds_health.c`              | NFS NULL 探测 + I/O 限制探测     |
| Cluster transport | `cluster_transport.c`      | gRPC 对等消息                  |
| Sessions/DRC      | `session.c`                | NFSv4.1 会话表                |
| Backchannel       | `nfs4_cb.c`                | CB_RECALL, CB_LAYOUTRECALL |
| Metrics           | `metrics_http_lite.c`      | Prometheus /metrics 端点     |
### 2.1.8 Compound 处理管线  
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
### 2.1.9 可插拔 Catalogue 层
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

### 2.1.10 并发与一致性  
- **单事务保证**: 每个变更操作（mkdir, rmdir, link, rename, setattr）编译为**一个 NDB 事务**
- **原子父 inode 更新**: 使用 NDB `interpretedUpdateTuple` 避免 read-modify-write 竞争
- **Striped 锁**: `pthread_mutex_t` 数组，16 或 1024 条纹（open_state 默认 1024）
- **NDB 排他行锁**: 用于 setattr 等需要序列化的路径
- **Any-MDS 路由**: 任何 MDS 可服务任何文件句柄，通过 RonDB 共享权威 + `parent_touch` 叠加层保证一致性

### 2.1.11 NFS Layout 路径  
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

### 2.1.12 集群特性  
- **Active-active**: 分区命名空间 + `fs_locations` referral（NFS4ERR_MOVED）
- **跨子树重命名**: 两阶段提交 (2PC) over inter-MDS gRPC
- **子树迁移**: 通过 admin RPC 实时转移所有权
- **故障转移**: RonDB 原生 partner-loss 检测 + 自动提升 + NFSv4.1 grace period
- **复制**: 由 NDB 集群原生处理，无应用层 delta 传输
### 2.1.13 性能优化  
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
`src/catalogue/catalogue_rondb_shim.cpp` — RonDB C++ → C ABI 桥接
### 2.2.4 第四阶段：理解集群与高级特性
`src/cluster/ `— 跨 MDS 协调（transport, 2PC, failover, migration）
`docs/striping-guide.m`d — HPC-Shared 宽条带文件
`docs/find-api.md` — 元数据搜索 API
`docs/pnfs-lab-deploy-guide.md` — 集群部署指南
### 2.2.5 第五阶段：测试与质量保证
`tests/catalogue_conformance/` — 后端一致性测试套件
`tests/unit/ `— 82 个单元测试（使用 CMocka 框架）
`CONTRIBUTING.md `— 代码规范、编译纪律、提交规范

## 2.3 部署指南  
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


## 2.4 编译部署 - 单节点 
### 2.4.1 环境概览
#### 2.4.1.1 系统环境

|项目|实际值|项目要求|说明|
|---|---|---|---|
|OS|CentOS 8.4|Ubuntu 22.04+ / Rocky 9+|低版本 CentOS 可用但需处理兼容性|
|GCC|8.4.1|>= 11.1|需临时降低门控，实际编译无错误|
|CMake|3.28.4（pip 安装）|>= 3.20|系统自带 3.18.2 不满足|
|libntirpc|3.2|需要|已安装|
|OpenSSL|1.1.1g|需要|已安装|
|nfs-utils|2.3.3|需要|已安装（DS 角色）|
#### 2.4.1.2 部署拓扑

```
┌──────────────────────────────────────────────────────────┐
│  localhost (单节点)                                       │
│                                                          │
│  ┌──────────────────┐      ┌──────────────────┐         │
│  │  pnfs-mds (MDS)  │      │  knfsd (DS)      │         │
│  │  NFS  :2050      │─────>│  NFS  :2049      │         │
│  │  gRPC :50051     │      │  /export/ds      │         │
│  │  HTTP :9090      │      └──────────────────┘         │
│  │  mount:20048     │                                    │
│  └──────────────────┘                                    │
│         │                                                │
│  ┌──────┴───────────┐                                   │
│  │  memdb 后端       │  (内存元数据，重启丢失)             │
│  └──────────────────┘                                   │
│         ▲                                                │
│  ┌──────┴───────────┐                                   │
│  │  NFSv4.1 Client  │  mount -o port=2050               │
│  └──────────────────┘                                   │
└──────────────────────────────────────────────────────────┘
```

**数据路径**：Client → MDS (NFSv4.1, port 2050) → 获取 layout → Client → DS (NFSv3, port 2049) 直接读写  
### 2.4.2 编译步骤

#### 2.4.2.1 升级 CMake
系统自带 CMake 3.18.2 不满足 >= 3.20 的要求，通过 pip 升级：
```bash
pip3 install cmake
# 验证
cmake --version  # 输出: cmake version 3.28.4
```

#### 2.4.2.2 处理 GCC 版本门控
项目要求 GCC >= 11.1，但系统只有 GCC 8.4.1。需临时修改两处：
**文件 1**: `CMakeLists.txt` 第 138 行
```cmake
# 修改前
if(CMAKE_C_COMPILER_VERSION VERSION_LESS "11.1")
# 修改后
if(CMAKE_C_COMPILER_VERSION VERSION_LESS "8.0")
```

**文件 2**: `include/pnfs_mds.h` 第 28 行
```c
// 修改前
# if PNFS_GCC_VERSION < 110100
_Static_assert(0, "pnfs-mds requires GCC >= 11.1 ...");

// 修改后
# if PNFS_GCC_VERSION < 80000
_Static_assert(0, "pnfs-mds requires GCC >= 8.0 ...");
```

> **注意**: 编译完成后应恢复原始门控值，避免提交不兼容的变更。
#### 2.4.2.3 配置和编译
```bash
cd /home/code/pnfs-lattice

cmake -B build \
    -DCMAKE_BUILD_TYPE=Release \
    -DENABLE_RONDB=OFF \
    -DENABLE_MEMDB_BACKEND=ON \
    -DENABLE_TESTS=ON \
    -DENABLE_EBPF=OFF \
    -DENABLE_ADMIN_CLI=ON \
    -DENABLE_FDB=OFF \
    -DENABLE_DAOS=OFF

cmake --build build -j$(nproc)
```

#### 2.4.2.4 编译产出

|二进制|大小|说明|
|---|---|---|
| `build/src/mds/pnfs-mds` |836 KB|MDS 守护进程（主程序）|
| `build/src/tools/mds-admin` |328 KB|管理 CLI 工具|
| `build/src/tools/mds-find` |170 KB|元数据搜索 CLI|
| `build/src/tools/mds-apid` |—|只读 HTTP 查询 API|

#### 2.4.2.5 运行测试
```bash
ctest --test-dir build --output-on-failure -L unit
```

结果：**76 个单元测试全部通过 (100%)**，6 个 FDB/DAOS 测试正确跳过。  
### 2.4.3 部署步骤

#### 2.4.3.1 设置数据服务器 (DS)
DS 使用本机 knfsd 提供 NFSv3 导出：
```bash
# 创建导出目录
mkdir -p /export/ds
chmod 0777 /export/ds

# 配置 NFS 导出
cat > /etc/exports.d/pnfs-ds.exports << 'EOF'
/export/ds 127.0.0.0/8(rw,sync,no_subtree_check,no_root_squash,insecure)
/export/ds 10.0.0.0/8(rw,sync,no_subtree_check,no_root_squash,insecure)
EOF

# [14:45:53] root@ceph-221:/home/code/pnfs-lattice/deploy/mnt/pnfs# cat /etc/exports.d/pnfs-ds.exports
# /export/ds 127.0.0.0/8(rw,sync,no_subtree_check,no_root_squash,insecure)
# /export/ds 10.0.0.0/8(rw,sync,no_subtree_check,no_root_squash,insecure)

# 启动 NFS 服务器
systemctl start nfs-server
exportfs -ra

# 验证
showmount -e localhost
# 输出: /export/ds 10.0.0.0/8,127.0.0.0/8
```

**说明**：  
1. `/etc/exports.d/pnfs-ds.exports` — NFS 导出配置文件，这是 knfsd（Linux 内核 NFS 服务器）的**导出配置文件**。NFS 服务器不会自动把目录共享出去，必须通过 exports 文件明确声明"哪些目录、对哪些客户端、以什么权限开放"。
2. exportfs -ra — 重新导出所有文件系统，让内核 NFS 服务器重新读取配置文件并生效。类似于 systemctl reload。如果只写了配置文件但不执行 exportfs -ra，NFS 服务器并不知道有新的导出需要发布。
3. 为什么用 localhost (127.0.0.1)
    1. **MDS 需要回环挂载 DS**：pnfs-mds 启动时需要通过 NFSv3 挂载 DS 的导出目录（`ds[0] = 127.0.0.1:/export/ds`），目的是获取 DS 的文件句柄（file handle），以便在 pNFS layout 响应中告诉客户端"数据写到哪里"。
    2. **端口 2049 冲突**：knfsd 已占用 2049 端口，所以 MDS 必须使用 2050。客户端挂载 MDS 时需指定 `port=2050`。
    3. **开发环境简化**：生产环境中 MDS、DS、Client 分布在不同物理机上（参见 `docs/MANUAL_INSTALL.md` 的 6 节点拓扑），各自使用真实 IP。单节点部署用 localhost 模拟整个集群。

**生产环境对比**：

| |开发环境|生产环境|
|--|---|---|
|DS 地址| `127.0.0.1:/export/ds` | `10.10.10.52:/export/ds` |
|MDS 地址| `127.0.0.1:2050` | `10.10.10.50:2049` |
|Client|同一台机器|独立客户端节点|
|DS 数量|1|2+|
|MDS 数量|1|3+ (集群)|
#### 2.4.3.2 创建 MDS 配置

```bash
mkdir -p deploy/etc deploy/log deploy/mnt/ds0 deploy/mnt/pnfs

cat > deploy/etc/mds.conf << 'EOF'
# pnfs-mds 单节点开发配置
mds_id = 1
hostname = 127.0.0.1
nfs_port = 2050
grpc_port = 50051

# 元数据后端: memdb (内存)
catalogue_backend = memdb

# 单节点
cluster_size = 1
cluster_bind_addr = 127.0.0.1

# 工作线程
worker_threads = 4

# 启用 pNFS layout
serve_layouts = true

# 数据服务器配置
ds_count = 1
ds[0] = 127.0.0.1:/export/ds
ds_mount_path_fmt = /home/code/pnfs-lattice/deploy/mnt/ds%u

# NFSv4 租约
lease_time_sec = 90
grace_period_sec = 0

# 内联数据 (小文件优化)
inline_enabled = true
inline_max_size = 65536

# 缓存
inode_cache_size = 16384
dirent_cache_size = 32768
positive_cache_ttl_ms = 1000

# 可观测性
metrics_http_port = 9090

# 日志
log_file = /home/code/pnfs-lattice/deploy/log/mds.log
log_level = info

# mountd 兼容
mountd_compat_enabled = true
mountd_compat_port = 20048
mountd_compat_exports = /
EOF
```

#### 2.4.3.3 挂载 DS 导出
MDS 需要回环挂载 DS 导出以准备文件句柄：
```bash
mount -t nfs -o vers=3 127.0.0.1:/export/ds deploy/mnt/ds0
```

#### 2.4.3.4 启动 MDS
```bash
./build/src/mds/pnfs-mds ./deploy/etc/mds.conf &
```
验证端口：
```bash
ss -tlnp | grep pnfs-mds
# 0.0.0.0:2050   - NFS 监听
# 127.0.0.1:50051 - gRPC 集群通信
# 0.0.0.0:9090   - Prometheus 指标
```

#### 2.4.3.5 客户端挂载
```bash
mount -t nfs4 -o vers=4.1,port=2050 127.0.0.1:/ deploy/mnt/pnfs
```

**DS导出和客户端挂载的关系**：  
1. DS 导出和客户端挂载是**两条独立的数据路径**，各自承担不同职责。  
```bash
客户端
  │
  │  路径 A: NFSv4.1 挂载 MDS (控制面)
  │  mount -t nfs4 -o port=2050 127.0.0.1:/ /mnt/pnfs
  │
  ├──── MDS (pnfs-mds :2050)
  │     处理: OPEN, CLOSE, LOOKUP, GETATTR, LAYOUTGET, LAYOUTRETURN
  │     职责: 元数据、命名空间、layout 发放
  │
  │  路径 B: NFSv3 直连 DS (数据面)
  │     由 pNFS layout 自动触发，用户无感知
  │
  └──── DS (knfsd :2049)
        处理: READ, WRITE (NFSv3)
        职责: 实际文件数据存储
```

2. **客户端挂载 MDS 是入口，DS 导出是实际数据的存放地。** 具体流程：  
```bash
1. 客户端挂载 MDS
   mount -t nfs4 -o port=2050 127.0.0.1:/ /mnt/pnfs
   → 获得命名空间视图（目录树、文件名、属性）

2. 客户端写入文件
   echo "data" > /mnt/pnfs/file.txt

3. 客户端向 MDS 请求 OPEN + LAYOUTGET
   → MDS 在 memdb/RonDB 中创建 inode
   → MDS 返回 layout: "数据写到 DS 127.0.0.1:/export/ds 的 NFSv3 端口"

4. 客户端根据 layout，直接向 DS 发起 NFSv3 WRITE
   → 数据写入 /export/ds/data/{inode}_{stripe}_{mirror}

5. 客户端向 MDS 发送 LAYOUTRETURN
   → MDS 记录 layout 状态
```

3. 验证两条路径独立存在， 客户端在 `/mnt/pnfs/demo/hello.txt` 写入 "Hello pnfs-lattice!"，这段数据**不在 MDS 上**，而是通过路径 B 直接写入 DS 的 `/export/ds/data/5_0_0` 文件中。MDS 只保存了"这个文件叫 hello.txt，inode 是 5，数据在 DS 上"这个元信息。
```bash
# 路径 A: 客户端挂载点 (看到的是 MDS 提供的命名空间)
ls /home/code/pnfs-lattice/deploy/mnt/pnfs/
# 输出: demo/  (目录结构由 MDS 管理)

# 路径 B: DS 数据目录 (看到的是实际文件数据)
ls /export/ds/data/
# 输出: 5_0_0  7_0_0  (命名格式: inode_stripe_mirror)
```

4. MDS 也要挂载 DS的原因：  
MDS 配置中有这一行：
```
ds[0] = 127.0.0.1:/export/dsds_mount_path_fmt = deploy/mnt/ds%u
```
MDS 自己也需要挂载 DS，但目的**不是读写数据**，而是：

|目的|说明|
|---|---|
|获取文件句柄|MDS 通过 `name_to_handle_at()` 在 DS 挂载点上获取 DS 文件的 file handle，放入 layout 响应中|
|创建 DS 端文件|客户端 OPEN(CREATE) 新文件时，MDS 在 DS 挂载点上预创建对应的空文件（`ds_prepare`）|
|健康检查|MDS 定期探测 DS 挂载点是否可达|
**MDS 挂载 DS 是管理操作，不经过这条路径传输用户数据。**  
```bash
              客户端挂载 MDS                MDS 挂载 DS
              (用户可见)                   (管理用途)
              ─────────────               ────────────
协议:         NFSv4.1                     NFSv3
端口:         2050                        2049
目的:         获取命名空间 + layout         获取 DS 文件句柄
数据传输:      仅元数据操作                  不传用户数据
用户感知:      直接 (mount 点)               无感知

              客户端直连 DS
              (数据面，layout 驱动)
              ──────────────
协议:         NFSv3
端口:         2049
目的:         实际文件读写
用户感知:     无感知 (内核自动路由)
```

三条连接中，用户只需要执行一次 `mount`（挂载 MDS），其余两条连接由内核和 MDS 自动建立和管理。  

![923](assets/2026-10-22-pnfs调研-IMG.png)

### 2.4.4 使用方法  
#### 2.4.4.1 基本文件操作

```bash
MNT=deploy/mnt/pnfs

# 创建目录
mkdir $MNT/mydata

# 写入文件
echo "Hello pnfs-lattice!" > $MNT/mydata/test.txt
cat $MNT/mydata/test.txt

# 硬链接 (共享 inode)
ln $MNT/mydata/test.txt $MNT/mydata/link.txt
ls -li $MNT/mydata/

# 符号链接
ln -s test.txt $MNT/mydata/symlink.txt

# 重命名
mv $MNT/mydata/test.txt $MNT/mydata/renamed.txt

# 删除
rm $MNT/mydata/link.txt
```

#### 2.4.4.2 扩展属性 (xattr)
```bash
# 设置
setfattr -n user.comment -v "important data" $MNT/mydata/renamed.txt

# 读取
getfattr -n user.comment --only-values $MNT/mydata/renamed.txt

# 列出所有
getfattr -d $MNT/mydata/renamed.txt
```

#### 2.4.4.3 大文件写入
```bash
# 写入 100MB 文件
dd if=/dev/urandom of=$MNT/mydata/big.bin bs=1M count=100

# 验证 DS 上的数据文件
ls -la /export/ds/data/
# 输出示例:
# -rw-rw-rw- 1 root root 104857600 ... 5_0_0
# 文件命名: {inode}_{stripe}_{mirror}
```

#### 2.4.4.4 可观测性
```bash
# Prometheus 指标
curl http://127.0.0.1:9090/metrics

# 关键指标
curl -s http://127.0.0.1:9090/metrics | grep -v "^#" | grep -v " 0$"
# pnfs_mds_nfs_ops_total     - 总 NFS 操作数
# pnfs_mds_nfs_op{op="..."}  - 按操作类型统计
# pnfs_mds_ds_fh_cache_hits  - DS 文件句柄缓存命中
# pnfs_mds_layoutget_delay_* - Layout 获取延迟
```

#### 2.4.4.5 服务控制
```bash
# 停止 MDS
kill $(pgrep pnfs-mds)

# 卸载客户端
umount deploy/mnt/pnfs

# 停止 DS
systemctl stop nfs-server
umount deploy/mnt/ds0
```
### 2.4.5 端口分配  
| 端口    | 服务            | 用途            |
|-------|---------------|---------------|
| 2049  | knfsd         | DS NFS 服务     |
| 2050  | pnfs-mds      | MDS NFS 服务    |
| 50051 | pnfs-mds      | gRPC 集群通信     |
| 9090  | pnfs-mds      | Prometheus 指标 |
| 20048 | mountd_compat | showmount 兼容  |
### 2.4.6 文件目录结构  
```bash
/home/code/pnfs-lattice/
├── build/                          # 编译产出
│   └── src/
│       ├── mds/pnfs-mds           # MDS 守护进程
│       └── tools/
│           ├── mds-admin          # 管理工具
│           ├── mds-find           # 搜索工具
│           └── mds-apid           # HTTP API
├── deploy/                         # 部署目录
│   ├── etc/mds.conf               # MDS 配置
│   ├── log/mds.log                # 运行日志
│   └── mnt/
│       ├── ds0/                   # DS 回环挂载
│       └── pnfs/                  # 客户端挂载点
└── /export/ds/                     # DS 数据目录
    └── data/                       # 实际文件数据
        └── {inode}_{stripe}_{mirror}
```


## 2.5 编译部署 - 多节点   
说明：基于V4.2的版本  
### 2.5.1 编译  

- [lattice pnfs编译详细过程](assets/lattice%20pnfs编译详细过程.md)  

#### 2.5.1.1 安装工具链  

| 项目        | 原始状态                  | 目标要求   |
| --------- | --------------------- | ------ |
| OS        | CentOS 8 (glibc 2.28) | -      |
| GCC       | 8.4.1                 | ≥ 11.1 |
| CMake     | 未安装                   | ≥ 3.20 |
| RonDB     | 未安装                   | 26.02+ |
| libntirpc | 未安装                   | v5.8   |
CMake — 系统无可用包，通过 pip 安装：
`pip3 install cmake   # → cmake 3.28.4`
   GCC 14.2.1 — 系统只有 GCC 8.4.1，从 Rocky Linux 8 仓库下载 gcc-toolset-14 RPM 并解压到
   /opt/gcc14：
```shell
gcc-toolset-14-gcc-14.2.1-11.el8_10.x86_64.rpm
gcc-toolset-14-gcc-c++-14.2.1-11.el8_10.x86_64.rpm
gcc-toolset-14-binutils-2.41-4.el8_10.x86_64.rpm
gcc-toolset-14-runtime-14.0-1.el8_10.x86_64.rpm
gcc-toolset-14-libstdc++-devel-14.2.1-11.el8_10.x86_64.rpm
```
   安装路径：`/opt/gcc14/opt/rh/gcc-toolset-14/root/usr/bin/gcc`  
#### 2.5.1.2 安装开发依赖  
从 Rocky Linux 8 仓库下载并安装以下 RPM:  
  
| 包名                               | 用途               |
| -------------------------------- | ---------------- |
| openssl-devel-1.1.1k-18.el8_10   | OpenSSL 头文件和库    |
| krb5-devel-1.18.2-34.el8_10      | Kerberos/GSSAPI  |
| libcom_err-devel-1.45.6-7.el8_10 | et/com_err.h 头文件 |
| libtirpc-devel-1.1.4-12.el8_10   | TI-RPC 开发库       |
#### 2.5.1.3 从源码构建 libntirpc v5.8  
RHEL/CentOS 不提供 libntirpc 包，使用项目自带脚本 deps/build-ntirpc.sh：
```shell
git clone --depth 1 -b v5.8 https://github.com/nfs-ganesha/ntirpc.git
cmake -S ntirpc -B ntirpc/build -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_INSTALL_PREFIX=/usr/local -DUSE_GSS=OFF -DTIRPC_EPOLL=ON
cmake --build ntirpc/build -j$(nproc)
cmake --install ntirpc/build
```
   安装到 `/usr/local/lib64/libntirpc.so` 和 `/usr/local/include/ntirpc/`。  
#### 2.5.1.4 安装 RonDB 26.02.4  
按 docs/INSTALL_ROCKY.md 文档下载：
```shell
wget https://repo.hops.works/master/rondb-26.02.4-linux-glibc2.28-x86_64.tar.gz
tar xzf rondb-26.02.4-linux-glibc2.28-x86_64.tar.gz -C /opt/
ln -sfn /opt/rondb-26.02.4-linux-glibc2.28-x86_64 /opt/rondb
```
关键文件：
   - 头文件：/opt/rondb/include/storage/ndb/ndbapi/NdbApi.hpp
   - 库文件：/opt/rondb/lib/libndbclient.so → libndbclient.so.6.1.0
   - 管理工具：/opt/rondb/bin/ndb_mgm（验证版本：RonDB-26.02.4）

检查Rondb的安装情况：  
```bash
[root@lzhnode3 ~]# /opt/rondb/bin/ndb_mgm --version
RonDB distrib RonDB-26.02.4, for Linux (x86_64)
```
#### 2.5.1.5 CMake 配置  
```
GCC14_ROOT=/opt/gcc14/opt/rh/gcc-toolset-14/root/usr
CC=$GCC14_ROOT/bin/gcc CXX=$GCC14_ROOT/bin/g++ \
LD_LIBRARY_PATH=$GCC14_ROOT/lib64:$GCC14_ROOT/lib:/opt/rondb/lib \
cmake -B build \
  -DCMAKE_BUILD_TYPE=Release \
  -DENABLE_RONDB=ON \
  -DRonDB_ROOT=/opt/rondb \
  -DENABLE_MEMDB_BACKEND=ON \
  -DENABLE_EBPF=OFF \
  -DENABLE_ADMIN_CLI=ON \
  -DENABLE_TESTS=ON \
  -DENABLE_FDB=OFF \
  -DENABLE_DAOS=OFF
```

CMake 输出确认：
```shell
-- Found RonDB: /opt/rondb/lib/libndbclient.so
-- RonDB found -- RonDB catalogue backend enabled
```
#### 2.5.1.6 编译
`cmake --build build -j$(nproc)`  

最终产物：  
![661](assets/1-2026-10-22-pnfs调研-IMG.png)  

检查是否包含RonDB动态库  
```bash
[root@lzhnode3 pnfs-lattice]# nm build/src/mds/pnfs-mds | grep catalogue_rondb
00000000004731e0 T catalogue_rondb_alloc_fileid_range
0000000000473240 T catalogue_rondb_alloc_gc_seq_range
```

复用编译命令
```shell
GCC14_ROOT=/opt/gcc14/opt/rh/gcc-toolset-14/root/usr
export PATH= $GCC14_ROOT/bin:$ PATH
export LD_LIBRARY_PATH= $GCC14_ROOT/lib64:$ GCC14_ROOT/lib:/opt/rondb/lib
export CC=$GCC14_ROOT/bin/gcc
export CXX=$GCC14_ROOT/bin/g++
cmake --build build -j$(nproc)
```
### 2.5.2 部署  
推荐拓扑，mds和ds角色不在相同机器

| Host    | Mgmt IP        | Internal IP | Role                             |
|---------|----------------|-------------|----------------------------------|
| mds1    | 192.168.100.50 | 10.10.10.50 | pnfs-mds + RonDB data node 1     |
| mds2    | 192.168.100.51 | 10.10.10.51 | pnfs-mds + RonDB data node 2     |
| mds3    | 192.168.100.55 | 10.10.10.55 | pnfs-mds + RonDB management node |
| ds1     | 192.168.100.52 | 10.10.10.52 | NFS data server (/export/ds)     |
| ds2     | 192.168.100.53 | 10.10.10.53 | NFS data server (/export/ds)     |
| client1 | 192.168.100.54 | 10.10.10.54 | NFSv4.2 client (/mnt/pnfs)       |

==🟣3节点实验室环境，以下部署过程以3节点为例==：
- Node A (==🟠Mgmt IP== - 10.128.137.132, ==🟢InternalIP== - 137.33.24.132) — mds1 + RonDB 数据节点 + ds1 (knfsd export)
- Node B (==🟠Mgmt IP== - 10.128.137.133, ==🟢InternalIP== - 137.33.24.133) — mds2 + RonDB 数据节点 + ds2 (knfsd export)
- Node C (==🟠Mgmt IP== - 10.128.137.134, ==🟢InternalIP== - 137.33.24.134) — mds3 + RonDB 管理节点（无数据节点）
- Node D (==🟡Mgmt IP== - 10.131.1.221, 用户的客户端)
RonDB 集群三类进程的关系：  

| 进程           | 角色             |
| ------------ | -------------- |
| ndb_mgmd     | 管理服务器，维护集群配置   |
| ndbmtd       | 数据节点，实际存储和处理数据 |
| mysqld (API) | SQL 接口节点       |
#### 2.5.2.1 在Node C配置和启动RonDB的mgmd(管理节点)  
```bash
mkdir -p /etc/rondb /var/lib/rondb/mgm /usr/local/lib/pnfs-lab
```

1. 以下是**RonDB 集群配置文件** (`config.ini`) 的编写说明，该文件必须在集群所有节点上保持一致，且以管理节点 (`mgmd`) 为权威来源。整个配置文件定义了 RonDB 集群的拓扑结构和资源分配，是 pNFS-lattice 系统中 MDS 通过 NDB 存储元数据的基础，在Node C创建完毕后，拷贝到Node A和Node B.
详细配置文件的说明见：  
- [[内部链接/RonDB集群配置文件说明|RonDB集群配置文件说明]]  

```bash
sudo tee /etc/rondb/config.ini >/dev/null <<'EOF'
[ndb_mgmd]
NodeId=65
HostName=137.33.24.134
PortNumber=1186
DataDir=/var/lib/rondb/mgm

[ndbd default]
NoOfReplicas=2
DataDir=/var/lib/rondb/data
FileSystemPath=/var/lib/rondb/data
DataMemory=2G
SharedGlobalMemory=256M
RedoBuffer=64M
TransactionMemory=128M
IndexMemory=64M

[ndbd]
NodeId=1
HostName=137.33.24.132

[ndbd]
NodeId=2
HostName=137.33.24.133

# API slots pinned to each MDS (two per MDS for connection pooling)
# 这些是 pNFS 的 MDS（Metadata Server）节点访问 RonDB 数据节点的通道。MDS 通过这些 API 节点与 NDB 数据节点通信，执行元数据读写操作
# 每个 MDS 节点各分配了 2 个 API 连接槽位，用于连接池化
[api]
NodeId=101
HostName=137.33.24.132
[api]
NodeId=102
HostName=137.33.24.132
[api]
NodeId=103
HostName=137.33.24.133
[api]
NodeId=104
HostName=137.33.24.133
[api]
NodeId=105
HostName=137.33.24.134
[api]
NodeId=106
HostName=137.33.24.134

# 预留了 16 个无固定主机绑定的 API 槽位，供 ndb_mgm（管理客户端）、mds-admin（MDS 管理工具）等临时连接使用。
# 这些槽位没有指定 HostName，任何节点都可以按需占用
[api]
NodeId=107
[api]
NodeId=108
[api]
NodeId=109
[api]
NodeId=110
[api]
NodeId=111
[api]
NodeId=112
[api]
NodeId=113
[api]
NodeId=114
[api]
NodeId=115
[api]
NodeId=116
[api]
NodeId=117
[api]
NodeId=118
[api]
NodeId=119
[api]
NodeId=120
[api]
NodeId=121
[api]
NodeId=122
EOF
```

总结如下：

| 区段             | 作用                                      |
|----------------|-----------------------------------------|
| [ndb_mgmd]     | 定义管理节点                                  |
| [ndbd default] | 数据节点的全局参数（2 副本、2G 数据内存等）                |
| [ndbd] × 2     | 两个数据节点                                  |
| [api] × 6      | 绑定到 3 个 MDS 的固定 API 槽位（每 MDS 2 个，用于连接池） |
| [api] × 16     | 自由 API 槽位，供管理工具等临时连接                    |
2. 编写启动脚本如下：  
```bash
sudo tee /usr/local/lib/pnfs-lab/rondb-mgmd-start.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
args=()
if [ ! -e /var/lib/rondb/mgm/ndb_65_config.bin.1 ]; then
    args+=(--initial)
fi
exec /opt/rondb/bin/ndb_mgmd \
    -f /etc/rondb/config.ini \
    --configdir=/var/lib/rondb/mgm \
    "${args[@]}" \
    --nodaemon
EOF
sudo chmod 0755 /usr/local/lib/pnfs-lab/rondb-mgmd-start.sh
```

3. 安装systemd unit
```bash
sudo tee /etc/systemd/system/rondb-mgmd.service >/dev/null <<'EOF'
[Unit]
Description=RonDB management server for pnfs lab
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
Environment=LD_LIBRARY_PATH=/opt/rondb/lib:/opt/rondb/lib/mysql
ExecStart=/usr/local/lib/pnfs-lab/rondb-mgmd-start.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now rondb-mgmd.service
sudo systemctl status rondb-mgmd.service --no-pager
```

#### 2.5.2.2 在Node A和Node B配置和启动RonDB的data nodes
先对Node A(mds1):  
```bash
sudo mkdir -p /var/lib/rondb/data /usr/local/lib/pnfs-lab

sudo tee /usr/local/lib/pnfs-lab/rondb-ndbmtd-start.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
NODE_ID=1            # <-- set to 2 on mds2
args=()
if [ ! -d /var/lib/rondb/data ] || \
   [ -z "$(find /var/lib/rondb/data -mindepth 1 -maxdepth 1 -print -quit 2>/dev/null)" ]; then
    args+=(--initial)
fi
exec /opt/rondb/bin/ndbmtd \
    --ndb-connectstring=137.33.24.134:1186 \
    --ndb-nodeid="${NODE_ID}" \
    "${args[@]}" \
    --nodaemon
EOF
sudo chmod 0755 /usr/local/lib/pnfs-lab/rondb-ndbmtd-start.sh

sudo tee /etc/systemd/system/rondb-ndbmtd.service >/dev/null <<'EOF'
[Unit]
Description=RonDB data node for pnfs lab
After=network-online.target rondb-mgmd.service
Wants=network-online.target

[Service]
Type=simple
Environment=LD_LIBRARY_PATH=/opt/rondb/lib:/opt/rondb/lib/mysql
ExecStart=/usr/local/lib/pnfs-lab/rondb-ndbmtd-start.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now rondb-ndbmtd.service
```

对Node B(mds2):  
```bash
sudo mkdir -p /var/lib/rondb/data /usr/local/lib/pnfs-lab

sudo tee /usr/local/lib/pnfs-lab/rondb-ndbmtd-start.sh >/dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
NODE_ID=2
args=()
if [ ! -d /var/lib/rondb/data ] || \
   [ -z "$(find /var/lib/rondb/data -mindepth 1 -maxdepth 1 -print -quit 2>/dev/null)" ]; then
    args+=(--initial)
fi
exec /opt/rondb/bin/ndbmtd \
    --ndb-connectstring=137.33.24.134:1186 \
    --ndb-nodeid="${NODE_ID}" \
    "${args[@]}" \
    --nodaemon
EOF
sudo chmod 0755 /usr/local/lib/pnfs-lab/rondb-ndbmtd-start.sh

sudo tee /etc/systemd/system/rondb-ndbmtd.service >/dev/null <<'EOF'
[Unit]
Description=RonDB data node for pnfs lab
After=network-online.target rondb-mgmd.service
Wants=network-online.target

[Service]
Type=simple
Environment=LD_LIBRARY_PATH=/opt/rondb/lib:/opt/rondb/lib/mysql
ExecStart=/usr/local/lib/pnfs-lab/rondb-ndbmtd-start.sh
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now rondb-ndbmtd.service
```

#### 2.5.2.3 确认RonDB启动成功  
在Node C:  
```bash
/opt/rondb/bin/ndb_mgm --ndb-connectstring=137.33.24.134:1186 -e show
```

说明：  
mgmd进程： `/opt/rondb/bin/ndb_mgmd` , 检查方式： `systemctl status rondb-mgmd`
data nodes进程：  `/opt/rondb/bin/ndbmtd`, 检查方式： `systemctl status rondb-ndbmtd`
##### 2.5.2.3.1 问题1  - node A和node B连接失败
检查结果：  
```bash
[root@lzhnode3 ~]# /opt/rondb/bin/ndb_mgm --ndb-connectstring=137.33.24.134:1186 -e show
Connected to management server at 137.33.24.134 port 1186 (using cleartext)
Cluster Configuration
---------------------
[ndbd(NDB)]        2 node(s)
id=1 (not connected, accepting connect from 137.33.24.132)
id=2 (not connected, accepting connect from 137.33.24.133)

[ndb_mgmd(MGM)]    1 node(s)
id=65      @137.33.24.134  (RonDB-26.02.4)

[mysqld(API)]      22 node(s)
id=101 (not connected, accepting connect from 137.33.24.132)
id=102 (not connected, accepting connect from 137.33.24.132)
id=103 (not connected, accepting connect from 137.33.24.133)
id=104 (not connected, accepting connect from 137.33.24.133)
id=105 (not connected, accepting connect from 137.33.24.134)
id=106 (not connected, accepting connect from 137.33.24.134)
id=107 (not connected, accepting connect from any host)
id=108 (not connected, accepting connect from any host)
id=109 (not connected, accepting connect from any host)
id=110 (not connected, accepting connect from any host)
id=111 (not connected, accepting connect from any host)
```

说明：  
```bash
RonDB 集群当前状态不健康，存在严重问题：

   管理节点 (MGM)
   - id=65 @ 137.33.24.134 — 正常运行，版本 RonDB-26.02.4

   数据节点 (NDB)
   - id=1 @ 137.33.24.132 — 未连接
   - id=2 @ 137.33.24.133 — 未连接

   两个数据节点都处于 not connected 状态，意味着集群无法提供任何数据服务。RonDB
   是分布式数据库，至少需要半数以上的数据节点在线才能正常工作（当前 2
   个全部离线，集群完全不可用）。
```

原因：  
node A上RonDB没有启动成功  

##### 2.5.2.3.2 mysqld连接失败  
MySQL API节点全部未连接，这意味着集群无法提供 SQL 服务
```bash
[root@lzhnode2 ~]# /opt/rondb/bin/ndb_mgm --ndb-connectstring=137.33.24.134:1186 -e show
Connected to management server at 137.33.24.134 port 1186 (using cleartext)
Cluster Configuration
---------------------
[ndbd(NDB)]        2 node(s)
id=1       @137.33.24.132  (RonDB-26.02.4, Nodegroup: 0, *)
id=2       @137.33.24.133  (RonDB-26.02.4, Nodegroup: 0)

[ndb_mgmd(MGM)]    1 node(s)
id=65      @137.33.24.134  (RonDB-26.02.4)

[mysqld(API)]      22 node(s)
id=101 (not connected, accepting connect from 137.33.24.132)
id=102 (not connected, accepting connect from 137.33.24.132)
id=103 (not connected, accepting connect from 137.33.24.133)
id=104 (not connected, accepting connect from 137.33.24.133)
id=105 (not connected, accepting connect from 137.33.24.134)
id=106 (not connected, accepting connect from 137.33.24.134)
id=107 (not connected, accepting connect from any host)
id=108 (not connected, accepting connect from any host)
id=109 (not connected, accepting connect from any host)
id=110 (not connected, accepting connect from any host)
id=111 (not connected, accepting connect from any host)
id=112 (not connected, accepting connect from any host)
id=113 (not connected, accepting connect from any host)
id=114 (not connected, accepting connect from any host)
id=115 (not connected, accepting connect from any host)
id=116 (not connected, accepting connect from any host)
id=117 (not connected, accepting connect from any host)
id=118 (not connected, accepting connect from any host)
id=119 (not connected, accepting connect from any host)
id=120 (not connected, accepting connect from any host)
id=121 (not connected, accepting connect from any host)
id=122 (not connected, accepting connect from any host)
```
解决（下面是其中一个节点的处理，其余节点处理相同）：  
- [[内部链接/mysqld连接失败问题解决|mysqld连接失败问题解决]]  
解决完毕后：  
```bash
[root@lzhnode1 ~]# /opt/rondb/bin/ndb_mgm --ndb-connectstring=137.33.24.134:1186 -e show
Connected to management server at 137.33.24.134 port 1186 (using cleartext)
Cluster Configuration
---------------------
[ndbd(NDB)]        2 node(s)
id=1       @137.33.24.132  (RonDB-26.02.4, Nodegroup: 0, *)
id=2       @137.33.24.133  (RonDB-26.02.4, Nodegroup: 0)

[ndb_mgmd(MGM)]    1 node(s)
id=65      @137.33.24.134  (RonDB-26.02.4)

[mysqld(API)]      22 node(s)
id=101     @137.33.24.132  (RonDB-26.02.4)
id=102 (not connected, accepting connect from 137.33.24.132)
id=103     @137.33.24.133  (RonDB-26.02.4)
id=104 (not connected, accepting connect from 137.33.24.133)
id=105     @137.33.24.134  (RonDB-26.02.4)
id=106 (not connected, accepting connect from 137.33.24.134)
id=107 (not connected, accepting connect from any host)
id=108 (not connected, accepting connect from any host)
id=109 (not connected, accepting connect from any host)
id=110 (not connected, accepting connect from any host)
id=111 (not connected, accepting connect from any host)
id=112 (not connected, accepting connect from any host)
id=113 (not connected, accepting connect from any host)
id=114 (not connected, accepting connect from any host)
id=115 (not connected, accepting connect from any host)
id=116 (not connected, accepting connect from any host)
id=117 (not connected, accepting connect from any host)
id=118 (not connected, accepting connect from any host)
id=119 (not connected, accepting connect from any host)
id=120 (not connected, accepting connect from any host)
id=121 (not connected, accepting connect from any host)
id=122 (not connected, accepting connect from any host)
```

说明：  
当前状态正常：
✅ 所有核心组件已连接并运行：
- 2 个 NDB 数据节点（132、133）- 正常
- 1 个管理节点（134）- 正常
- 3 个 MySQL API 节点（每个节点各 1 个）- 正常

关于未连接的 API 槽位： 剩余 19 个未连接的槽位是设计如此，完全正常：
- id=102、104、106：每个节点的备用 API 槽位，预留给连接池使用（按需启动）
- id=107-122：16 个动态槽位，供临时连接和管理工具使用

因此： 集群已完全恢复正常运行。三个节点都可以通过 MySQL 接口访问 NDB分布式数据库，数据副本同步正常（Nodegroup 0 包含两个数据节点）。

##### 2.5.2.3.3 rondb连接不正常  
三个问题：  
1. node1连接失败
2. node2状态为starting,未就绪
3. mysqld连接失败
```bash
[root@lzhnode2 ~]# /opt/rondb/bin/ndb_mgm --ndb-connectstring=137.33.24.134:1186 -e show
Connected to management server at 137.33.24.134 port 1186 (using cleartext)
Cluster Configuration
---------------------
[ndbd(NDB)]        2 node(s)
id=1 (not connected, accepting connect from 137.33.24.132)
id=2       @137.33.24.133  (RonDB-26.02.4, starting, Nodegroup: 0)

[ndb_mgmd(MGM)]    1 node(s)
id=65      @137.33.24.134  (RonDB-26.02.4)

[mysqld(API)]      22 node(s)
id=101 (not connected, accepting connect from 137.33.24.132)
id=102 (not connected, accepting connect from 137.33.24.132)
id=103 (not connected, accepting connect from 137.33.24.133)
id=104 (not connected, accepting connect from 137.33.24.133)
id=105 (not connected, accepting connect from 137.33.24.134)
id=106 (not connected, accepting connect from 137.33.24.134)
```

原因：内存不足导致rondb服务异常 ,解决方案，stop ceph系列占用内存的服务：  
```bash
systemctl stop ceph-osd.target
```
##### 2.5.2.3.4 问题2  - ndbmtd 内存占用过大
解决：  
1. 检查配置： `cat /etc/rondb/config.ini | grep -A 20 '\[ndbd default\]'`
2.  使用 rondb-mem-config.sh 生成合理的内存配置
```bash
[ndbd default]
AutomaticMemoryConfig=0
DataMemory=10240M
DiskPageBufferMemory=3584M
SharedGlobalMemory=256M
RedoBuffer=64M
TransactionMemory=128M
```

或者在原来的基础上加上：  
```bash
# 关闭自动内存
AutomaticMemoryConfig=0
# 限制总内存上限     
TotalMemoryConfig=8G
MaxNoOfExecutionThreads=4
```

3. 更新 `/etc/rondb/config.ini`
4. 重启管理节点：`systemctl restart rondb-mgmd`
5. 滚动重启数据节点: `/opt/rondb/bin/ndb_mgm -e "<node_id> restart"`
    - ndb_mgm 是 RonDB/NDB 集群的管理客户端（Management Client）, 连接到管理服务器（ndb_mgmd），用于查看集群状态、执行管理操作
    - 常用命令：
        - `/opt/rondb/bin/ndb_mgm --ndb-connectstring=137.33.24.134:1186 -e show` # 查看集群整体状态
        - 进入交互模式： `/opt/rondb/bin/ndb_mgm --ndb-connectstring=137.33.24.134:1186`
```bash
# 滚动重启两个数据节点（先等第一个 Started 再重启第二个）
/opt/rondb/bin/ndb_mgm -e "1 restart"
/opt/rondb/bin/ndb_mgm -e "2 restart"
```

#### 2.5.2.4 配置NFS data servers ds1和ds2  

在Node A和Node B启动ds1和ds2, 命令如下：  
```bash
ssh root@10.128.137.132    # and 10.128.137.133
# Debian/Ubuntu
sudo apt update
sudo apt install -y nfs-kernel-server nfs-common

# centos
yum install -y nfs-utils
systemctl enable --now nfs-server

# 创建目录（类似 mkdir -p，但可同时设置权限）
sudo install -d -m 0777 /export/ds /export/ds/data  # /export/ds 和 /export/ds/data：权限 0777（所有人可读写执行），用于 DS 数据存储
sudo install -d -m 0755 /etc/exports.d # 权限 0755，存放额外的 NFS 导出配置文件片段

# 导出规则：
# /export/ds: 要导出的目录
# 137.33.24.0/24: 允许访问的客户端网段
# rw: 读写权限
# sync: 同步写入（数据先落盘再返回，保证一致性）
# no_subtree_check: 不检查子目录是否在导出范围内，提升性能
# no_root_squash: 客户端的 root 用户保留 root 权限（不做降权映射）
# insecure: 允许客户端使用 非特权端口（>1024）连接，某些 NFS 客户端需要
sudo tee /etc/exports.d/pnfs-ds.exports >/dev/null <<'EOF'
/export/ds 137.33.24.0/24(rw,sync,no_subtree_check,no_root_squash,insecure)
EOF

# enable：设置开机自启; --now：立即启动服务（等同于同时执行 start）
sudo systemctl enable --now nfs-kernel-server
sudo exportfs -ra # -r: 重新导出所有文件系统（重新读取 /etc/exports 及 /etc/exports.d/*.exports）, -a: 作用于所有条目
sudo exportfs -v  # 以详细模式列出当前所有活跃的 NFS 导出及其选项，用于确认配置是否生效

# Raise knfsd thread pool from the default 8 to 32
# Linux 内核 NFS 服务默认启动 8 个线程 处理请求
# 这里将其提升到 32 个线程，以应对更高的并发 I/O 负载
# if [ -w ... ] 检查该 proc 文件是否可写（即内核 NFS 模块已加载）
# 写入 /proc/fs/nfsd/threads 会动态调整运行中的 nfsd 线程数

if [ -w /proc/fs/nfsd/threads ]; then
    echo 32 | sudo tee /proc/fs/nfsd/threads >/dev/null
fi
```

检查导出是否生效：可以看到 `/export/ds 137.33.24.0/24`和`/export/ds 137.33.24.0/24 `  
```bash
[root@lzhnode1 ~]# showmount -e 137.33.24.132
Export list for 137.33.24.132:
/export/ds 137.33.24.0/24


[root@lzhnode2 system]# showmount -e 137.33.24.133
Export list for 137.33.24.133:
/export/ds 137.33.24.0/24
```

#### 2.5.2.5 在所有MDS节点安装pnfs-mds  
1. 确保对应MDS节点包含pnfs-mds和mds-admin,
2. 安装二进制和nfs client包
```bash
# sudo apt update
# sudo apt install -y libntirpc-dev libgssapi-krb5-2 libssl3 nfs-common attr
# 上述是Debian/Ubuntu的命令，cenots的nfs client包如下：
yum install -y libtirpc-devel krb5-libs openssl nfs-utils attr
sudo install -d -m 0755 /usr/local/bin /etc/pnfs-mds /var/lib/pnfs-mds
sudo install -m 0755 /home/code/pnfs-lattice/build/src/mds/pnfs-mds /usr/local/bin/pnfs-mds
sudo install -m 0755 /home/code/pnfs-lattice/build/src/tools/mds-admin /usr/local/bin/mds-admin
```

3. 在所有节点创建RonDB的后台配置文件
```bash
# 137.33.24.134是Rondb的管理节点ip
sudo tee /etc/pnfs-mds/rondb.conf >/dev/null <<'EOF'
connect_string = 137.33.24.134:1186
schema_name = pnfs_mds
EOF
```

4. 在每个节点创建pnfs-mds的配置文件
如下是Node A的配置：
```bash
sudo tee /etc/pnfs-mds/mds.conf >/dev/null <<'EOF'
mds_id = 1
hostname = 137.33.24.132
nfs_port = 2050  # 注意nfs-server使用了2049端口
grpc_port = 50051
catalogue_backend = rondb
catalogue_backend_conf = /etc/pnfs-mds/rondb.conf
worker_threads = 8
inline_enabled = false
lease_time_sec = 90
grace_period_sec = 0
ds_heartbeat_ms = 5000
cluster_size = 3
cluster_bind_addr = 137.33.24.132
prealloc_pool_size = 4096
inode_cache_size = 32768
transient_state_cache = true
cluster_peer[0] = 137.33.24.132
cluster_peer[1] = 137.33.24.133
cluster_peer[2] = 137.33.24.134
ds_count = 2
ds[0] = 137.33.24.132:/export/ds
ds[1] = 137.33.24.133:/export/ds
ds_mount_path_fmt = /mnt/ds%u
EOF
```

Node B和Node C只需要修改如下项目，其余相同：  
```bash
# Node B
mds_id = 2
hostname = 137.33.24.133
cluster_bind_addr = 137.33.24.133
# Node C
mds_id = 3
hostname = 137.33.24.134
cluster_bind_addr = 137.33.24.134
```

5. 在每个MDS挂载DS导出
为了准备文件句柄，MDS需要使用nfsv3挂载每个DS导出，这些挂载DS导出并不是客户端数据路径，客户端依然通过pNFS向这些DS直接写入数据  
```bash
sudo install -d -m 0755 /mnt/ds0 /mnt/ds1
sudo mount -t nfs -o vers=3,nconnect=8 137.33.24.132:/export/ds /mnt/ds0
sudo mount -t nfs -o vers=3,nconnect=8 137.33.24.133:/export/ds /mnt/ds1
```

通过向/etc/fstab写入如下配置，可以在设备重启后实现自动挂载：  
```bash
echo '137.33.24.132:/export/ds /mnt/ds0 nfs vers=3,nconnect=8,_netdev 0 0' | sudo tee -a /etc/fstab
echo '137.33.24.133:/export/ds /mnt/ds1 nfs vers=3,nconnect=8,_netdev 0 0' | sudo tee -a /etc/fstab
```

6. 在所有MDS节点安装pnfs-mds的systemd unit并启动
```bash
sudo tee /etc/systemd/system/pnfs-mds.service >/dev/null <<'EOF'
[Unit]
Description=pnfs-mds daemon
After=network-online.target rondb-mgmd.service rondb-ndbmtd.service remote-fs.target
Wants=network-online.target

[Service]
Type=simple
Environment=LD_LIBRARY_PATH=/opt/rondb/lib:/opt/rondb/lib/mysql
ExecStart=/usr/local/bin/pnfs-mds /etc/pnfs-mds/mds.conf
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable --now pnfs-mds.service
sudo systemctl status pnfs-mds.service --no-pager
```

#### 2.5.2.6 在客户端上mount对应的命名空间，挂载完成后，可以看到
```bash
# sudo apt update
# sudo apt install -y nfs-common attr

sudo yum install -y nfs-utils attr


sudo install -d -m 0755 /mnt/pnfs
# 注意使用port=2050, 默认端口被nfs-server使用
sudo mount -t nfs4 -o vers=4.2,port=2050,nconnect=8 137.33.24.134:/ /mnt/pnfs
mount | grep /mnt/pnfs
ls /mnt/pnfs

# 结果：
[root@lzhnode3 ~]# ls /mnt/pnfs
shard1  shard2  shard3
[root@lzhnode3 ~]# mount | grep /mnt/pnfs
137.33.24.134:/ on /mnt/pnfs type nfs4 (rw,relatime,vers=4.2,rsize=65536,wsize=65536,namlen=255,hard,proto=tcp,nconnect=8,port=2050,timeo=600,retrans=2,sec=sys,clientaddr=137.33.24.134,local_lock=none,addr=137.33.24.134)
```
集群是多MDS，每个命名空间的根目录是普通的可写目录，其中每个 MDS 各持有一个 referral 条目。你应该能看到每个 MDS 对应一个条目：  
```
shard1  shard2  shard3
```
每个 `shardN` 都是一个 NFSv4 `fs_locations` referral，指向拥有该分片的 MDS（shard1 → mds1 @ 137.33.24.132，shard2 → mds2 @ 137.33.24.133，shard3 → mds3 @ 137.33.24.134）。内核在你第一次 `cd` 进入某个分片时会跨越这个 referral。

直接在根目录写入也可以——共享的 RonDB catalogue 使得根级别的条目从每个 MDS 都可见——但**建议优先在分片内工作**，这样通过 referral 将元数据负载分散到各个 MDS 上，而不是全部集中到你挂载的那个 MDS。
### 2.5.3 冒烟测试  

在客户端上，可以使用某个shard进行读写操作：  
```bash
SHARD=/mnt/pnfs/shard1    # or shard2 / shard3
cd "$SHARD"
sudo mkdir -p smoke && cd smoke
echo hello | sudo tee f1 >/dev/null
sudo cat f1
sudo ln f1 f1.hl
sudo ln -s f1 f1.sym
sudo readlink f1.sym
sudo setfattr -n user.foo -v bar f1
sudo getfattr -n user.foo --only-values f1; echo
sudo dd if=/dev/urandom of=big.bin bs=1M count=10 status=none
sudo stat -c '%s bytes' big.bin
cd "$SHARD"     # leave the smoke dir before removing it
sudo rm -rf "$SHARD/smoke"
```

### 2.5.4 服务管理  
1. pnfs-mds管理
```bash
sudo systemctl restart pnfs-mds
sudo systemctl status pnfs-mds
sudo journalctl -u pnfs-mds -f
```

2. stop路径上的所有服务
```bash
# on client1
sudo umount /mnt/pnfs || true

# on each MDS
sudo systemctl stop pnfs-mds
sudo umount /mnt/ds0 /mnt/ds1 || true

# on Rondb data node
sudo systemctl stop rondb-ndbmtd

# on Rondb 管理 node
sudo systemctl stop rondb-mgmd
```

启动服务，则依照  
a. Rondb管理node: `sudo systemctl start rondb-mgmd`  
b. Rondb数据node: `sudo systemctl start rondb-ndbmtd`  
c. `ndb_mgm -e show` 检查data node全部连接完毕后，在所有pnfs-mds节点执行： mount所有ds和 `systemctl start pnfs-mds`  
d. 客户端挂载pnfs

## 2.6 常用命令  
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

