---
年份:
  - "2026"
月份:
  - 10月
imageNameKey: pnfs数据分布管理
创建时间: 2026-10-28 17:37
修改时间: 2026-10-28 17:37
tags:
  - pnfs
---
pNFS-lattice 采用 **MDS + DS 分离架构**：MDS（Metadata Server）管理元数据和布局信息，DS（Data Server）存储实际数据文件。一个文件的完整数据被切分成多个条带，分布到多个 DS 上，客户端可以绕过 MDS 直接与各 DS 通信完成 I/O。  

pNFS 的数据分布解决的核心问题是：**一个文件的字节如何分散到多个数据服务器（DS）上的多个物理文件中**。这涉及四个维度：  

| 维度                  | 含义                 | 决定时机  |
| ------------------- | ------------------ | ----- |
| 条带宽度 (stripe_count) | 文件数据分布在几个 DS 上     | 文件创建时 |
| 条带大小 (stripe_unit)  | 每个 DS 文件承载的连续字节块大小 | 文件创建时 |
| 镜像数量 (mirror_count) | 每个条带存几份副本          | 文件创建时 |
| DS 选择 (placement)   | 每个条带的每个镜像放在哪个 DS 上 | 文件创建时 |

这四个维度一旦确定就永久固定在文件的 stripe map 中，不会随 DS 的上下线而改变。  

# 1 Stripe  
Stripe 是 pNFS-lattice 实现并行 I/O 的核心机制。文件创建时确定条带几何并持久化到 catalogue；LAYOUTGET 时将 stripe map 以 flex-files 布局形式下发给客户端；客户端可直接对 DS 做条带化 I/O（需内核支持），或通过 MDS 代理访问
## 1.1 核心概念
Stripe 是 pNFS（parallel NFS, RFC 8881 / RFC 8435）的核心数据分布机制。一个文件的逻辑数据被切分成固定大小的块（**stripe unit**），轮流分布到多个数据服务器（DS）上，每个 DS 存储文件的一个 **stripe**。  
关键参数定义在 `pnfs_mds.h`：  

| 参数           | 含义                     | 上限                     |
|--------------|------------------------|------------------------|
| stripe_count | 每个文件的条带数（数据分布在几个 DS 上） | MDS_MAX_STRIPES = 1024 |
| stripe_unit  | 每个条带的大小（字节），即轮转粒度      | 默认 65536，推荐 1 MiB      |
| mirror_count | 每个 stripe 的副本数         | MDS_MAX_MIRRORS = 4    |

数据文件命名约定（见 `proxy_io.c`）：  
```txt
{mount_path}/data/{fileid}_{stripe}_{mirror}
```

**寻址方式** — 稀疏寻址（sparse addressing）：
```txt
stripe_idx  = (offset / stripe_unit) % stripe_count
local_off   = offset   // DS 文件内偏移 == 文件逻辑偏移
```

即每个 stripe 的 DS 文件内偏移等于文件的逻辑偏移，而不是密集打包。这样 MDS 代理 I/O 和客户端直接 I/O 可以混合访问同一文件而不产生数据错乱。  
## 1.2 Stripe 的放置（Placement）策略  - 决定数据分布到哪些 DS  
Placement 在文件首次需要布局时运行，决定 `stripe_count × mirror_count` 个 DS 槽位分别由哪些 DS 承载， 核心实现在 `placement.c`，由 ` placement.h` 声明接口。  

**放置策略类型：**

| 策略                    | 选择逻辑                                          | 适用场景      |
|-----------------------|-----------------------------------------------|-----------|
| PLACEMENT_RR          | 轮询 — 全局原子计数器轮转（并发调用各得唯一起点）               | 默认策略，简单均匀 |
| PLACEMENT_WEIGHTED_RR | 加权随机 — 按 DS 权重做加权 pick（`mds_wrr_weighted_pick`） | 异构 DS 集群  |
| PLACEMENT_CAPACITY    | 容量优先 — 选剩余空间最大的 DS（`mds_wrr_capacity_pick`）   | 均衡存储利用率   |

**WRR 权重优先级**（`placement.c::fill_online_with_free`，高到低）：① operator 配置 `ds_weight.<id>` → ② 容量自动权重 auto_weight（`placement_capacity_weighting=PROPORTIONAL` 时由 statvfs 探测推导：`max(1, floor((1-used/total)×100))`，bounded [1,100]，满盘 DS 权重为 1 保证仍可选中）→ ③ 上报空闲字节 (total-used) → ④ 常量 1。
注意：WEIGHTED_RR/CAPACITY 的选核函数在 `src/modules/wrr/` 可选模块中，**社区版构建只有 stub（恒返回 0），这两种策略会降级为 online 集合上的纯 RR**；多条带放置用本地 taken 位图保证各 stripe 落在不同 DS，同一 stripe 的 mirror 不共置。

**降级与持久化**：可用 DS 数 < stripe_count × mirror_count 时缩减到实际可放条数（最小 1），返回 effective_stripe_count 打点 `pnfs_mds_placement_degraded_total`；**调用方必须用有效条带数持久化**，否则幻影条带会把 LAYOUTGET 卡进 NFS4ERR_DELAY 死循环。另有搬迁配套选核：`placement_select_replacement`（镜像替换，可校验管理员指定 target）与 `placement_select_for_tier`（分层迁移，tier 内最空者）。

**放置流程**:  
以 op_layoutget 中的 placement fallback 路径为例（`compound_layout.c`）
```txt
步骤 1：读取 DS 注册表
   cat_ds_list() → 获取所有已注册 DS 的信息数组
   ↓
步骤 2：兼容性过滤
   layout_filter_for_inode()
   ├─ 过滤条件：DS 状态 == ONLINE
   ├─ 模式匹配：DS_MODE_GENERIC
   └─ 传输匹配：TCP / RDMA / 两者
   ↓
步骤 3：I/O 限制过滤
   layout_drop_iolimit_ineligible()
   └─ 移除不满足 I/O 限制条件的 DS
   ↓
步骤 4：确定几何参数
   if (placement_policy_enabled && 配置了默认值)
       stripe_count = cfg_default_stripe_count
       mirror_count = cfg_default_mirror_count
   else
       stripe_count = 1, mirror_count = 1
   stripe_unit = cfg_stripe_unit ?: 65536
   ↓
步骤 5：DS 权重叠加（仅 WRR 策略）
   ds_cache_overlay_weights()
   └─ 从 DS 缓存读取 operator 配置的权重
   ↓
步骤 6：执行放置算法
   placement_select_ex2()（policy_enabled）或 placement_select2()（纯 RR）
   ├─ 输出：entries[] 数组（填充 ds_id）
   ├─ 输出：实际 stripe_count（可能已降级，必须用于持久化）
   ├─ 公式：idx = (start + s*mirror_count + m) % online_count
   │    LAYOUTGET 路径 start = 全局原子计数器(每次前进 sc×mc)
   │    ds_prealloc 宽条带预热路径 start = fileid % online_count
   │    （placement_select_rr_at2，使宽文件散布全池而非挤头部热点）
   └─ 栈暂存上限 64 个 DS，超出走堆并打点 heap_fallback_total
      （MDS_PLACEMENT_STACK_MAX / MDS_MAX_DS_NODES=256）
   ↓
步骤 7：持久化 Stripe Map
   mds_cat_stripe_map_put(cat, NULL, fileid,
       stripe_count, stripe_unit, mirror_count, entries)
```

## 1.3 Stripe Map 的存储与查询
Stripe map 描述了一个文件的完整条带布局：`stripe_count × mirror_count` 个 `mds_ds_map_entry` 条目， 描述了每个 (stripe, mirror) 槽位对应哪个 DS, 按 **行优先**（stripe 为行，mirror 为列）排列：  
```txt
entries[0] = stripe 0, mirror 0 → ds_id=2
entries[1] = stripe 0, mirror 1 → ds_id=5
entries[2] = stripe 1, mirror 0 → ds_id=3
entries[3] = stripe 1, mirror 1 → ds_id=7
...
```

**存储层级：**  

| 级   | 适用条件                             | 存储位置                                                 | 访问开销      |
|-----|----------------------------------|------------------------------------------------------|-----------|
| 内联  | stripe_count=1 && mirror_count=1 | inode 行内联字段（`MDS_IFLAG_INLINE_STRIPE`，schema v9：inline_ds_id / inline_fh / inline_stripe_unit） | 零额外查询（一次 inode 读同时服务 LAYOUTGET / unlink fence / GC；create/remove 不写/删 stripe 行） |
| 缓存  | HPC-Shared 宽条带文件                 | 内存 16-shard LRU（layout_cache，默认 1024 条目，条目为堆拷贝）   | 内存哈希查找    |
| 持久化 | 所有多条带/镜像文件                       | RonDB / FDB 的 mds_stripe_maps + mds_stripe_entries 表 | 1-2 次网络往返 |

**Catalogue 后端实现**
1. RonDB: `catalogue_rondb.c` 实现了 layoutget_fused，将 stripe_get + layout_grant 合并到一个 NDB 事务中，减少一次网络往返
2. FoundationDB: `catalogue_fdb_ext.c` 使用 stripe_rows_write / stripe_get_body 将 stripe header + entries 编码为 FDB key-value 对

## 1.4 LAYOUTGET 中的 Stripe 处理  
核心流程在 `compound_layout.c`：  
1. **快速路径** — 如果文件是同一 compound 中刚创建的（`stripe_cached`），直接使用缓存的 stripe 信息，跳过 catalogue 查询
2. **Fused 路径** — 通过 `layoutget_fused` 在一个事务中完成 stripe map 获取 + layout grant
3. **常规路径** — `cat_stripe_map_get` 查询 stripe map
4. **降级路径** — 如果没有 stripe map 且文件为空（首次 LAYOUTGET），进入 placement 选择 DS 并持久化；如果 stripe map 缺失但文件非空，**拒绝放置回退**（防止把已有数据搁浅），返回 NFS4ERR_IO

注意：**既有文件永不重条带**——新增 DS 不会改变已创建文件的 stripe map（`compound_layout.c` Issue #6 注释）；重分布只能靠管理员触发的 rebalance/resilver（企业版模块）。

**线格式（Wire Form**）: 对于 flex-files 布局（RFC 8435），有两种 XDR 形式：

|形式|说明|客户端要求|
|---|---|---|
| `LEGACY` |每个 DS 一个 `ff_mirror4` 条目（`mirror_count = stripe_count × mirror_count`）|所有 Linux 内核|
| `STRIPED` |1 个 `ff_mirror4` 条目，`ds_count = stripe_count` |Linux 6.18+|

选择逻辑：AUTO 模式下，当 `mirror_count == 1 && stripe_count > 1` 时自动选择 STRIPED 形式。  

## 1.5 Stripe 级别的租约与冲突检测  
`lease_table.h` 实现了 per-stripe 粒度的租约表：  
- 冲突域是 `(fileid, stripe_index)`，而非传统的 `(fileid, offset)`
- 写同一文件不同 stripe 的客户端不会冲突
- 写同一 stripe 的重叠字节范围的客户端返回 `NFS4ERR_LAYOUTTRYLATER`

`ease_stripe_map.h`（应为 `lease_stripe_map.h`）提供了纯辅助函数，将逻辑字节范围分解为 per-stripe 切片：  
```c
struct stripe_slice {
    uint32_t stripe_index;   // 条带编号
    uint64_t ds_offset;      // 在条带内的偏移
    uint64_t ds_length;      // 在条带内的长度
};
```
# 2 Device ID — DS 的网络寻址  
客户端拿到 stripe map 中的 `ds_id` 后，需要通过 `GETDEVICEINFO` 获取该 DS 的网络地址。  
**Device ID 结构：**  
16 字节（`compound_layout.c`）：  
```txt
┌──────────────┬──────────────┬──────────────────┬──────────┐
│ mds_id (4B)  │ ds_id (4B)   │ generation (4B)  │ zeros(4B)│
│ 大端          │ 大端          │ 大端              │          │
└──────────────┴──────────────┴──────────────────┴──────────┘
```

- `mds_id`：多 MDS 集群中标识哪个 MDS 创建的
- `ds_id`：指向具体的数据服务器
- `generation`：DS 的 I/O 限制（rsize/wsize）变化时递增

`generation` 的设计目的：当 DS 的 I/O 限制发生变化时，新的布局携带不同的 device ID，客户端发现 ID 变化后必须重新发起 `GETDEVICEINFO`，从而获取最新的 rsize/wsize 值。generation 来自 `ds_io_limits_get()`（Wave 5 T5.1），由 NFSv3 FSINFO 探测（`ds_iolimit_probe_ms`，默认 60s，探测关闭时回退 legacy 1MiB 常量）在 rtmax/wtmax 变化时递增。而 `deviceid_to_ds_id()` 只提取中间 4 字节的 `ds_id`，忽略 generation，所以同一 DS 的所有 generation 版本都解析到同一个 DS 注册行。  

**GETDEVICEINFO 返回内容**：  
```c
struct nfs4_res_getdeviceinfo {
    uint32_t layout_type;
    uint32_t ds_count;           // 始终为 1
    struct {
        uint32_t ds_id;
        uint32_t endpoint_count;
        struct nfs4_ds_endpoint {
            char     netid[8];   // "tcp" 或 "rdma"
            char     host[256];
            uint16_t port;
        } endpoints[2];
        uint32_t rsize;          // DS 读 I/O 限制
        uint32_t wsize;          // DS 写 I/O 限制
    } ds[2];
};
```