---
年份:
  - "2026"
月份:
  - 10月
imageNameKey: pnfs数据备份与冗余机制
创建时间: 2026-10-28 17:46
修改时间: 2026-10-28 17:46
tags:
  - pnfs
---
pNFS-lattice 的"冗余"分四层：**DS 数据冗余（mirror）**、**元数据冗余（RonDB 副本）**、**MDS 服务冗余（active-standby failover）**、**inline 数据冗余（随元数据）**。以下均以源码为准逐层说明。

# 1 Mirror（镜像）机制：DS 数据冗余  
| 配置                                   | 说明                           |
| ------------------------------------ | ---------------------------- |
| mirror_count = 1（默认）                 | 无冗余，每个 stripe 只有一份数据         |
| mirror_count = 2                     | 每个 stripe 有 2 份副本，分布在不同 DS 上 |
| mirror_count = N（上限 MDS_MAX_MIRRORS=4） | 每个 stripe 有 N 份副本            |
| 数据文件命名格式：`{fileid}_{stripe}_{mirror}` | 放置时硬约束：同一 stripe 的不同 mirror 不共置同一 DS |

```bash
DS-1                          DS-2                          DS-3
┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
│ /export/ds/data/│          │ /export/ds/data/│          │ /export/ds/data/│
│   5_0_0 (镜像0)  │◄────────►│   5_0_1 (镜像1) │          │                 │
│   6_0_0         │          │                 │          │                 │
└─────────────────┘          └─────────────────┘          └─────────────────┘

mirror_count = 2 的文件 5
```

写入语义：客户端直写路径和 MDS 代理路径**都写所有 mirror**（任一失败即报错）；读取时任选一个好 mirror，失败按序切换。

# 2 DS 故障时系统会做什么（自动）vs 不会做什么  
**源码事实：DS 故障后没有任何"自动重建数据"的线程。** resilver / rebalance / tiering 都是 `src/modules/` 下的企业版可选模块，由管理员通过 mds-admin 显式触发（main.c 注释原文："Resilver worker (admin-triggered, not auto-started)"）；社区版构建只有 stub，API 存在但不可用。

## 2.1 自动执行的部分（ds_health + layout_recall）
```bash
1. ds_health_monitor 周期探测 (NFSv3 NULL, 默认 5s 周期, 默认连续 6 次失败)
   → 防抖动: OFFLINE 后冷却 30s 起指数翻倍至 8min 封顶;
     恢复需连续成功 max(3, threshold/2) 次才重新标 ONLINE
2. 标记 DS_OFFLINE (经 commit_queue COMMIT_OP_DS_STATE 顺带复制到 Standby)
3. layout_recall_for_ds 回调:
   → 扫 mds_ds_layout_idx 找到该 DS 上全部 (clientid, fileid)
   → 对每个唯一 clientid 发 CB_LAYOUTRECALL(ALL)（尽力而为）
   → 无论回调成败, 权威撤销全部受影响 layout_state 行
   → ds_io_limits 递增 io_generation → 新 deviceid 迫使客户端重新 GETDEVICEINFO
4. 后续 LAYOUTGET 的 placement 过滤掉 OFFLINE DS；
   既有文件若 stripe 恰好全落在故障 DS 上，则该文件数据不可访问，
   直到管理员介入（见 2.2）
```

## 2.2 需要管理员介入的部分（resilver）
```bash
DS-1 (健康)                   DS-2 (故障)                   DS-3 (新)
┌─────────────────┐          ┌─────────────────┐          ┌─────────────────┐
│   5_0_0 (镜像0)  │──────────│   5_0_1 (丢失)  │─────────►│   5_0_1 (重建)   │
└─────────────────┘          └─────────────────┘          └─────────────────┘
      源数据                    ✗                     mds-admin 触发 resilver

流程: 扫 degraded mirror 文件 → 从健康 mirror proxy_read → proxy_write 到
      placement_select_replacement 选出的替换 DS → 单事务更新 stripe_map
      + gc_enqueue 旧槽位(MDS_GC_SWEEP_SLOT) → 客户端后续 LAYOUTGET 拿到新映射
约束: 单 stripe 文件, 无活跃 writer/layout, dense 文件
```

# 3 元数据冗余：RonDB 副本  
- 所有目录元数据（inode、dirent、stripe map、DS registry、协议状态行）存于 RonDB/NDB，副本数由集群配置 `NoOfReplicas=2` 决定（实验室配置即 2 副本，容忍 1 个数据节点故障）。MDS 本身无本地权威状态，任意 MDS 挂掉不影响元数据安全。
- **inline 文件**（小文件，inline_enabled，≤64KiB）的数据直接存在 catalogue 行里，因此自动继承 RonDB 的副本冗余；代价是数据与元数据同路径，失去 pNFS 直连 DS 的优势（RonDB 模式强制 inline_enabled=false，全部数据走 DS）。

# 4 MDS 服务冗余：active-standby failover  
- 每个 Primary MDS 可配一个 Standby，二者组成 failover pair；Standby 通过复制链路跟随 Primary。
- 故障检测：心跳写 RonDB `mds_node_registry.last_heartbeat_ns`（5s），Standby 看门狗 2s 轮询、15s 判过期、20s 启动保护期；REALTIME 时钟域地板值防滚动升级误判。
- 提升为 8 阶段严格流程（角色守卫 → 预检查/自隔离/复制健康门/线协议兼容 → 快照伙伴子树 → 加载客户端恢复记录 → 进入 NFSv4.1 grace period → 发布 ACTIVE_SERVING → PRIMARY），多 Standby 竞争靠 partition 所有权 CAS 决出唯一胜者，防脑裂。
- 接管子树用 `subtree_map_failover_take_over`：先对 mds_partition_map 逐条 CAS，全部成功才继续，否则返回 STALE 放弃提升。

# 5 备份  
| 层面 | 现状（源码核对） |
|---|---|
| 在线备份 API | `mds_backup_create / verify / restore / info`（`src/mds/backup.c`）**全部为 stub，返回 MDS_ERR_NOSUPPORT**；注释说明计划用 NDB Backup API 实现，尚未落地 |
| RonDB 集群备份 | 用 RonDB 原生 `ndb_mgm -e "START BACKUP"`（ndb_mgm 是 NDB 管理客户端，连接 mgmd 执行管理操作），备份的是元数据集群本身 |
| MySQL 逻辑备份 | 通过 mysqld API 节点可 mysqldump（表引擎 ndbcluster） |
| DS 数据备份 | 文件级：DS 上的 `data/{fileid}_{stripe}_{mirror}` 是普通文件，可用任意文件级/块级手段备份；恢复时需与 stripe map 一致，建议停机或配合 mirror |
| 元数据检索导出 | `mds-find` / `mds-apid`（HTTP 只读查询 API）可用于审计式导出 |

# 6 生产环境建议  
| 场景            | 推荐配置                     |
|----------------|--------------------------|
| 开发/测试         | mirror_count = 1（节省存储）   |
| 生产环境（重要数据）    | mirror_count = 2 或 3     |
| HPC-Shared 宽条带文件 | 走宽条带（stripe_count 高、mirror_count=1）+ DS 底层 RAID/EC；当前不支持镜像 |
| DS 底层存储       | 建议使用 RAID 或分布式存储（如 Ceph）兜底单盘/单机故障 |
| DS 故障后         | 数据**不会**自动重建：先在剩余 mirror 上继续服务，再由管理员触发 resilver（企业版模块） |
| 元数据备份         | 定期 `ndb_mgm START BACKUP`；mds_backup_* API 就绪前勿依赖应用层在线备份 |
