---
年份:
  - "2026"
月份:
  - 10月
imageNameKey: pnfs数据访问
创建时间: 2026-10-28 17:49
修改时间: 2026-10-28 17:52
tags:
  - pnfs
---
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

pNFS-lattice 有两条互斥的 I/O 路径，由 MDS 在 LAYOUTGET 时决定走哪一条：  
```txt
           ┌─────────────────────────┐
           │      Client (rank)       │
           │  open() / read() / write()│
           └────────┬────────────────┘
                    │
                 LAYOUTGET?
               ╱          ╲
         成功 ╱                ╲ 失败/不支持
             ▼                    ▼
┌──────────────────┐   ┌──────────────────┐
│  Path A: 客户端   │   │  Path B: MDS 代理 │
│  直接 DS I/O      │   │  代理 I/O        │
│  (pNFS 路径)      │   │  (Proxy 路径)    │
└────┬─────┬───────┘   └────┬─────┬───────┘
     │     │                 │     │
READ/WRITE               READ/WRITE
直接到DS                  通过MDS转发
     │     │                 │     │
┌────┘     └────┐     ┌────┘     └────┐
│  DS0   DS1    │     │  DS0   DS1    │
│  (数据条带)    │     │  (数据条带)   │
└──────────────┘      └──────────────┘
```

# 1 客户端直接 DS I/O（pNFS 路径）  
## 1.1 触发条件  
客户端发起 LAYOUTGET 时，MDS 在 `op_layoutget` 中检查三个开关：  

| 条件                                         | 效果                                         |
|--------------------------------------------|--------------------------------------------|
| a->layout_type 不是 files 或 flex-files       | 返回 NFS4ERR_NOTSUPP                         |
| cd->cfg_serve_layouts == false             | 返回 NFS4ERR_LAYOUTUNAVAILABLE → 客户端走 Path B |
| HPC 文件且 cd->cfg_hpc_serve_layouts == false | 返回 NFS4ERR_LAYOUTUNAVAILABLE → 客户端走 Path B |
| inline 文件且无 DS_PENDING                     | 返回 NFS4ERR_LAYOUTUNAVAILABLE → 客户端走 Path B |
只有全部通过，MDS 才会下发布局，客户端才能直连 DS。  

## 1.2 布局获取交互（LAYOUTGET）  
1. **客户端 → MDS**：  
```txt
COMPOUND {
  PUTFH(file_handle)
  LAYOUTGET(
    signal_available = true,
    layout_type = LAYOUT4_FLEX_FILES,   // 或 LAYOUT4_NFSV4_1_FILES
    iomode = LAYOUTIOMODE4_RW,
    offset = 0,
    length = UINT64_MAX,
    minlength = 4096,                   // 客户端想写的范围
    stateid = {seqid=0, other=...},
    maxcount = 4096                     // Linux 内核硬编码上限
  )
}
```

2. **MDS 内部处理**:  （`compound_layout.c`）：  
MDS **不按照客户端请求的范围回复**，而是授权一个**读写、长生命周期的窗口**（`layout_select_grant_range()`：从请求 offset 起，上限受 `layout_grant_max_length_bytes` 约束，若与 stripe lease 冲突还会按 `stripe_lease_prefix_conflict_free_length()` 主动缩窄为无冲突前缀而不是拒绝）, 这样客户端可以跨 open/close 周期复用同一个布局，无需再次 LAYOUTGET。  

3. **MDS → 客户端（线上格式）：**
MDS 根据 `hpc_xdr_form` 配置选择器（auto/legacy/striped，`compound_layout.c`）决定线格式：  
**Legacy 形式（pre-6.18 客户端）**：  
```txt
ff_layout4 body:
  stripe_unit (8 bytes)
  ff_mirror_count = stripe_count × mirror_count
  for each mirror:
    ds_count = 1
    ff_data_server4:
      deviceid[16]          ← 编码了 mds_id + ds_id + io_generation
      efficiency (4)
      stateid (20)
      nfs_fh (variable)     ← DS 上的数据文件句柄
      ffl_user (string)     ← 合成 UID
      ffl_group (string)    ← 合成 GID
  ff_flags (4)
  stats_hint (4)
```
每个 `ff_mirror4` 只含 1 个 DS。客户端把多个 mirror 视为**副本**（镜像语义），从任一个读取。  

**Striped 形式（Linux 6.18+ 客户端）**:  

```txt
ff_layout4 body:
  stripe_unit (8 bytes)
  ff_mirror_count = 1
  ff_mirror4[0]:
    ds_count = stripe_count
    for each DS:
      ff_data_server4:
        deviceid[16]
        nfs_fh (variable)
        ffl_user / ffl_group
  ff_flags (4)
  stats_hint (4)
```
单个 `ff_mirror4` 包含所有 DS。客户端用 `dss_id = offset / stripe_unit` 计算每个 I/O 应该发到哪个 DS。  
## 1.3 设备信息获取（GETDEVICEINFO）  
客户端从 LAYOUTGET 响应中提取 `deviceid`，然后向 MDS 发送 GETDEVICEINFO 获取 DS 的网络地址：
1. **客户端 → MDS**
```txt
GETDEVICEINFO(
  deviceid[16],
  layout_type = LAYOUT4_FLEX_FILES,
  maxcount = 4096
)
```
2. **mds处理（`op_getdeviceinfo`）**
```c
ds_id = deviceid_to_ds_id(a->deviceid);   // 从 16 字节中提取 ds_id
cat_ds_get(cd, ds_id, &info);             // 查 DS 注册表
```
3. **MDS → 客户端：**
```txt
GETDEVICEINFO result:
  layout_type = LAYOUT4_FLEX_FILES
  ds_count = 1
  ds[0]:
    addr = "10.0.0.2"
    endpoint_count = 1~2
    endpoints[]:
      {netid="tcp",  host="10.0.0.2", port=2049}
      {netid="rdma", host="10.0.0.2", port=20049}  // 可选
    rsize = 1048576   // 读块大小
    wsize = 1048576   // 写块大小
```
客户端**缓存**这个映射，后续不再重复查询同一 deviceid。  
## 1.4 **客户端直接 I/O 的数据分布**
拿到布局和 DS 地址后，客户端**绕过 MDS**，直接向 DS 发送 NFS READ/WRITE：  
**条带寻址公式**（与 Linux 内核 flex-files 客户端一致）：  
```txt
对于逻辑文件偏移量 offset:
  stripe_idx = (offset / stripe_unit) % stripe_count
  ds_index   = stripe_idx * mirror_count + mirror_idx  // 选择哪个 DS
  local_off  = offset                                   // 稀疏寻址
```

**关键特征：==🔴稀疏寻址** — DS 文件内的偏移等于文件的逻辑偏移，不是紧凑排列==。  
示例：`stripe_unit=1MiB, stripe_count=4, mirror_count=1`  
```txt
逻辑文件:
  [0────1M───][1M───2M───][2M───3M───][3M───4M───][4M───5M───]...

DS 文件:
  DS0: [0────1M───]              [4M───5M───]          ...
  DS1:            [1M───2M───]              [5M───6M───]
  DS2:                       [2M───3M───]
  DS3:                                  [3M───4M───]
```
每个 DS 文件内部有大量**空洞**（sparse holes），只存储属于自己的那些 stripe unit。  

**为什么用稀疏而非密集？** 密集打包的公式是 `stripe_pos × stripe_unit + intra_offset`，这要求客户端和 MDS 代理对"条带内第几个位置"有完全一致的理解。如果客户端直接 I/O 用密集寻址而 MDS 代理 I/O 用稀疏寻址（或反过来），同一字节会被写到 DS 文件的不同位置，导致数据损坏。稀疏寻址让两条路径天然一致。  

**写操作**：客户端向目标 stripe 的**所有 mirror** 都写入相同数据（写时复制语义）。
**读操作**：客户端从目标 stripe 的**任一 mirror** 读取（失败则尝试下一个 mirror）。

## 1.5 布局提交交互（LAYOUTCOMMIT）  
客户端完成写操作后，向 MDS 发送 LAYOUTCOMMIT 报告文件变化：
1. 客户端 → MDS：
```txt
LAYOUTCOMMIT(
  stateid = <layout_stateid>,
  offset = 0,
  length = UINT64_MAX,
  new_offset = true,                    // 有数据增长
  last_write_offset = 1048575,          // 最后一个写字节偏移
  time_modify_set = true,
  time_modify = 2026-10-27T10:00:00
)
```

2. MDS 处理（op_layoutcommit）：
    - **默认路径（社区版实际行为）**：所有文件（含 HPC-Shared）都是同步单次 `cat_setattr`（SIZE_EXTEND 采用 max(old,new) 增长式合并 + MTIME + FLAGS），一次 LAYOUTCOMMIT 一次 NDB 写。`layout_commit_aggregator` 模块在社区版构建中只有 stub（init 返回 NULL 句柄），聚合器不生效。
    - **企业版路径**：`cd->lcommit_agg != NULL` 时 HPC-Shared 文件的路由到内存聚合器（默认 4096 桶、200ms 定时刷盘），合并 N 个并发写入者的 `max(size)` 和 `latest(mtime)`，摊薄写 NDB 的频率
    - **GETATTR 一致性**：`hpc_getattr_mode=STRICT`（默认）时 GETATTR 强制先 flush 该 fileid 的聚合桶再回复，保住 POSIX `stat()` 语义；`OPTIMISTIC` 直接回 `max(persisted, aggregated)`，更快但有偏差

## 1.6 布局归还交互（LAYOUTRETURN）  
客户端主动释放布局：  
```txt
LAYOUTRETURN(
  reclaim = false,
  layout_type = LAYOUT4_FLEX_FILES,
  iomode = LAYOUTIOMODE4_RW,
  return_type = LAYOUTRETURN4_FILE,
  stateid = <layout_stateid>
)
```
MDS 在 `op_layoutreturn` 中：  
1. 验证 stateid seqid（旧 → `NFS4ERR_OLD_STATEID`，新 → `NFS4ERR_BAD_STATEID`）
2. 删除 layout_state 行
3. 释放 stripe lease
4. 返回 `lrs_present = false`（整个布局已清除）
MDS **不会**在 LAYOUTRETURN 时做 DS 端文件 fencing（因为可能有其他客户端共享同一文件的布局）。  
## 1.7 服务端回调（CB_LAYOUTRECALL）  
MDS 可以主动回收客户端的布局：  
1. 场景 1 — DS 故障（`layout_recall_for_ds`）： 
```bash
ds_health 回调触发 →
  1. 扫描 ds_layout_idx 找到该 DS 上所有 (clientid, fileid)
  2. 收集唯一 clientid 列表
  3. 对每个客户端发送 CB_LAYOUTRECALL(LAYOUTRECALL4_ALL)
  4. 无论 CB 是否成功，强制撤销所有受影响的 layout_state 行
     （未响应的客户端在下次 SEQUENCE/LAYOUTGET 时发现撤销）
```
注意：此路径**不做 DS 文件 fencing**——DS 已故障，chown 无从谈起；fencing 发生在字节范围冲突撤销和 unlink 强制撤销路径（`fence_ds_file_for_fileid`，RFC 8435 §14）。

2. 场景 2 — 字节范围冲突（`layout_recall_byte_range_for_holders`）：
```txt
新 LAYOUTGET 到达，发现已有其他客户端持有重叠范围 →
  1. 收集冲突 holder 列表（按 iomode + range 过滤）
  2. 对每个 holder 发送 CB_LAYOUTRECALL(LAYOUTRECALL4_FILE, offset, length)
     其中 offset/length 是请求范围与持有范围的交集
  3. 强制撤销冲突的 layout_state 行
```

3. 场景 3 — 文件删除（`layout_recall_revoke_all_for_unlink`）：
```txt
REMOVE 操作 →
  1. 收集该 fileid 的所有 layout holder
  2. 跳过 CB（避免死锁 RPC worker pool）
  3. 直接撤销所有 layout_state 行 + fencing DS 文件
```

# 2 MDS 代理 I/O（Proxy 路径）  
以下任一情况导致客户端走 MDS 代理路径：
## 2.1 触发条件

| 触发                                 |
| ---------------------------------- |
| 客户端不支持 pNFS（不发 LAYOUTGET）          |
| MDS 返回 `NFS4ERR_LAYOUTUNAVAILABLE` |
| `serve_layouts=false` 全局开关         |
| HPC 文件且 `hpc_serve_layouts=false`  |
| inline 文件（无 DS 后端）                 |
| DS 文件 FH 未就绪                       |
## 2.2 代理 READ 的数据分布  
1. 客户端 → MDS：
```txt
COMPOUND {
  PUTFH(file_handle)
  READ(stateid, offset=0, count=65536)
}
```

2. MDS 内部处理（`mds_proxy_read`）：
```txt
1. proxy_stripe_map_get() → 获取 stripe_count, stripe_unit, mirror_count, entries[]
2. 循环跨越 stripe 边界:
   while (total_read < bounded_count):
     a. compute_stripe_addr(offset + total_read, ...)
        → stripe_idx = ((offset+total_read) / stripe_unit) % stripe_count
        → local_offset = offset + total_read   // 稀疏寻址
     b. 截断到当前 stripe_unit 边界
        chunk = min(remaining_data, stripe_unit - intra_stripe_offset)
     c. 镜像容错: for m in 0..mirror_count:
        → 打开 DS 文件: {mount}/data/{fileid}_{stripe_idx}_{m}
        → pread(fd, buf, chunk, local_offset)
        → 成功则 break; 失败则尝试下一个 mirror
     d. 短读补零（稀疏空洞）
```

**关键点**：MDS 代理 READ 与客户端直接 READ 使用**完全相同的条带寻址公式**，保证两种路径的数据一致性。  

## 2.3 代理 WRITE 的数据分布  
1. 客户端 → MDS：
```txt
COMPOUND {
  PUTFH(file_handle)
  WRITE(stateid, offset=0, data=...)
}
```
2. MDS 内部处理（mds_proxy_write）：
```txt
1. proxy_stripe_map_get() → 获取条带映射
2. 循环跨越 stripe 边界:
   while (total_written < data_len):
     a. compute_stripe_addr(offset + total_written, ...)
        → stripe_idx, local_offset
     b. 截断到 stripe_unit 边界
     c. 写入所有 mirror:
        for m in 0..mirror_count:
          → 打开 DS 文件: {mount}/data/{fileid}_{stripe_idx}_{m}
          → pwrite(fd, data, chunk, local_offset)
          → 任何一个 mirror 写失败 → 返回 MDS_ERR_IO
```

**与 READ 的区别**：WRITE 必须写到**所有 mirror**（不是任一个），保证副本一致性。  
## 2.4 代理路径的 DS 文件命名  
所有 DS 数据文件遵循统一命名约定（`build_ds_path`）：  
```txt
{ds_mount_path}/data/{fileid}_{stripe_index}_{mirror_index}
```
例如：`/mnt/ds0/data/12345_0_0`、`/mnt/ds0/data/12345_1_0`、`/mnt/ds1/data/12345_0_0`  

## 2.5 代理路径的 FD 缓存  
代理 I/O 使用 per-(fileid, ds_id, stripe, mirror, flags) 的 FD 缓存（`fd_cache`），避免每次 I/O 都 open/close：  
```txt
hash(key) → fd_entry {
  fd, fileid, ds_id, stripe, mirror, flags,
  refcount,    // pin 计数，保护并发 I/O
  dead         // 标记失效，等 refcount=0 时真正 close
}
```
## 2.6 两条路径的对比  
| 维度      | Path A（客户端直接 DS I/O）                  | Path B（MDS 代理 I/O）           |
|---------|---------------------------------------|------------------------------|
| 控制面     | LAYOUTGET → 获取布局                      | 无布局交互                        |
| 数据面     | 客户端 → DS（NFS READ/WRITE）              | 客户端 → MDS → DS（MDS 转发）       |
| MDS 瓶颈  | 无（数据不经过 MDS）                          | 有（所有字节经 MDS）                 |
| 并行度     | 客户端并行向多个 DS 发 I/O                     | MDS 串行遍历各 stripe             |
| 条带寻址    | 客户端本地计算 dss_id = offset / stripe_unit | MDS 调用 compute_stripe_addr() |
| 寻址方式    | 稀疏（DS 文件偏移 = 逻辑偏移）                    | 稀疏（完全相同）                     |
| 镜像读     | 客户端选择任一 mirror                        | MDS 按序尝试，失败切换                |
| 镜像写     | 客户端写所有 mirror                         | MDS 写所有 mirror               |
| 元数据更新   | LAYOUTCOMMIT → MDS                    | MDS 在 WRITE 时直接更新            |
| DS 文件句柄 | LAYOUTGET 响应中携带 nfs_fh                | MDS 通过本地 mount 的 FD 缓存       |
| DS 地址获取 | GETDEVICEINFO → 缓存                    | MDS 内部 mounts[ds_id] 注册表     |
| 凭证模型    | ffl_user/ffl_group（合成 UID）            | MDS 的 root 权限（本地 mount）      |
| 适用客户端   | Linux 6.18+（flex-files）               | 任何 NFS 客户端                   |
| 性能      | 高（并行、无 MDS 瓶颈）                        | 低（串行、MDS 瓶颈）                 |
# 3 数据分布一致性保证  
两条路径使用完全相同的条带寻址公式，这是数据一致性的基石：  
```txt
// Path A: Linux 内核 flex-files 客户端
dss_id = nfs4_ff_layout_calc_dss_id(stripe_unit, dss_count, offset);
// 等价于: (offset / stripe_unit) % stripe_count

// Path B: MDS proxy_io.c
stripe_num = offset / stripe_unit;
stripe_idx = stripe_num % stripe_count;
local_offset = offset;   // 稀疏寻址
```

无论数据通过哪条路径写入 DS，另一条路径都能正确读取。这使得：
1. 客户端可以在 Path A 和 Path B 之间动态切换（例如 DS 故障时从 Path A 回退到 Path B）
2. LAYOUTERROR 后 MDS 会清除布局缓存（`compound_layout.c`），下次 LAYOUTGET 重新获取最新的 DS 文件句柄
3. 混合客户端场景（部分支持 pNFS，部分不支持）下数据不会损坏

# 4 客户端直接DS I/O的协议栈  
客户端挂载的v4的pnfs-mds,在路径A如何直接和v3通信呢？原因如下：  
pNFS-lattice 采用 **控制面 / 数据面协议分离** 架构：  
```txt
┌─────────────────────────────────────────────────────────────────┐
│                      Linux pNFS 客户端                           │
│                                                                 │
│  控制面: NFSv4.1 ─────────────────────────────► MDS (port 2050) │
│         CREATE_SESSION / OPEN / LAYOUTGET                      │
│         LAYOUTCOMMIT / LAYOUTRETURN / GETDEVICEINFO            │
│         CB_LAYOUTRECALL (反向)                                  │
│                                                                 │
│  数据面: NFSv3 ──────► DS0 (port 2049)                          │
│         NFSv3 ──────► DS1 (port 2049)     直接 READ/WRITE       │
│         NFSv3 ──────► DS2 (port 2049)                          │
└─────────────────────────────────────────────────────────────────┘
```

**户端用 NFSv4.1 挂载 MDS，但用 NFSv3 直连各个 DS 做数据 I/O。** 这 RFC 8435 Flex-Files 布局的标准设计。
## 4.1 使用NFSv3的原因
Flex-Files 布局（RFC 8435）选择 NFSv3 作为数据协议，原因是：
- NFSv3 是**无状态**的，不需要 SESSION/状态机，客户端可以并行向多个 DS 发起独立连接
- NFSv3 实现简单、部署广泛，所有 Linux 内核都支持
- 每个 DS 就是一个普通的 NFSv3 文件服务器（本项目中用 knfsd）
## 4.2 NFSv3 文件句柄如何传递到客户端
MDS 在 LAYOUTGET 响应中携带的不是 NFSv4 文件句柄，而是**每个 DS 上数据文件的 NFSv3 不透明文件句柄**。  
### 4.2.1 MDS 如何获取 DS 的 NFSv3 FH  
MDS 有两种方式获取 DS 上的 NFSv3 文件句柄：  
1. 主路径：`name_to_handle_at() ` 系统调用
MDS 在本地通过 NFSv3 挂载了每个 DS 的导出目录。当需要获取某个 DS 数据文件的句柄时：  
```txt
MDS 本地文件系统:
  /mnt/ds0/  → NFSv3 mount → DS0:/export/ds
  /mnt/ds1/  → NFSv3 mount → DS1:/export/ds

步骤:
  1. open("/mnt/ds0/data/12345_0_0", O_WRONLY|O_CREAT, 0666)  // 创建/打开 DS 文件
  2. name_to_handle_at(AT_FDCWD, path, &fh, &mount_id, 0)      // 提取 VFS 文件句柄
  3. 从 VFS 句柄中解析出 NFSv3 server FH:
     f_handle[0..11]  : VFS NFS wrapper
     f_handle[12..13] : uint16_t LE -- server FH size (e.g. 36)
     f_handle[14..]   : raw NFSv3 server FH bytes
```
2. 回退路径：NFSv3 RPC
当 syscall 路径失败时（DS 未挂载、内核不兼容等），MDS 直接向 DS 发起 NFSv3 RPC：  
```c
ds_nfs3_lookup_fh(host, nfs_port, export_path,
                  "data/12345_0_0",  // 相对路径
                  1,                  // create if absent
                  fh_out, &fh_len, 3000);
```
这个函数执行 NFSv3 MOUNT3 → 获取导出根 FH → NFSv3 LOOKUP 逐层解析 → 得到数据文件的 NFSv3 FH。  
### 4.2.2 FH 如何嵌入 LAYOUTGET 响应  
获取到的 NFSv3 FH 作为**不透明字节**嵌入到 flex-files 布局的 `ff_data_server4` 结构中  
```c
// compound_layout.c L2878-L2882
r->ff_mirrors[0].ds[k].nfs_fh_len = r->ds[k].nfs_fh_len;
memcpy(r->ff_mirrors[0].ds[k].nfs_fh,
       r->ds[k].nfs_fh,
       r->ds[k].nfs_fh_len);
```

XDR 编码器将其原样输出到线上 :  
```c
// fh_count = 1, then fh (opaque)
uint32_t fh_count = 1;
xdr_uint32_t(body_xdrs, &fh_count);
uint32_t fh_len = fds->nfs_fh_len;
xdr_uint32_t(body_xdrs, &fh_len);
xdr_opaque_encode(body_xdrs, fds->nfs_fh, fh_len);
```

### 4.2.3 客户端拿到 FH 后如何使用  
Linux 内核的 flex-files 客户端驱动收到 LAYOUTGET 响应后：
1. 解析出每个 DS 的 `deviceid` + `nfs_fh`（NFSv3 不透明句柄）
2. 对每个唯一的 `deviceid` 发送 GETDEVICEINFO → 获取 DS 的 IP/port
3. **建立 NFSv3 连接**到该 DS（独立于 NFSv4.1 MDS 连接）
4. 用拿到的 NFSv3 FH 直接向 DS 发送 NFSv3 READ/WRITE RPC
### 4.2.4 协议交互时序  
```txt
Client                    MDS (NFSv4.1)              DS (NFSv3)
  │                           │                          │
  │── CREATE_SESSION ────────►│                          │
  │◄── session+backchannel ───│                          │
  │                           │                          │
  │── OPEN(file) ────────────►│                          │
  │◄── stateid+attrs ─────────│                          │
  │                           │                          │
  │── LAYOUTGET ─────────────►│                          │
  │                           │── name_to_handle_at() ──►│
  │                           │   (通过本地 NFSv3 mount)   │
  │                           │◄── NFSv3 FH ─────────────│
  │                           │                          │
  │                           │── chown DS file ────────►│
  │                           │   (设置 ffl_user 所有权)   │
  │                           │                          │
  │◄── layout{stateid,        │                          │
  │    stripe_unit,           │                          │
  │    nfs_fh[0..N],  ←──────────────── NFSv3 文件句柄    │
  │    deviceid[0..N],        │                          │
  │    ffl_user, ffl_group} ──│                          │
  │                           │                          │
  │── GETDEVICEINFO ─────────►│                          │
  │◄── {addr="10.0.0.2",      │                          │
  │     port=2049,            │                          │
  │     rsize=1MiB, ──────────│                          │
  │     wsize=1MiB}           │                          │
  │                           │                          │
  │─── ── ─ ── ── ── ── ── ──│─ ── ── ── ── ── ── ── ─│
  │  现在建立 NFSv3 连接        │                          │
  │                           │                          │
  │── NFSv3 MOUNT ───────────────────────────────────────►│
  │◄── mount FH ─────────────────────────────────────────│
  │                           │                          │
  │══ NFSv3 WRITE(fh, offset, data) ════════════════════►│
  │══ NFSv3 WRITE(fh, offset, data) ════════════════════►│  并行!
  │══ NFSv3 READ(fh, offset)  ══════════════════════════►│
  │◄══ data ══════════════════════════════════════════════│
  │                           │                          │
  │── LAYOUTCOMMIT ──────────►│                          │
  │   (报告 new_size, mtime)  │── cat_setattr ──► NDB    │
  │◄── OK ────────────────────│                          │
```
### 4.2.5 凭证模型：AUTH_SYS 桥接

两条路径使用不同的凭证模型，这也是理解协议交互的关键：

| 路径           | 凭证                            | 说明                                  |
| ------------ | ----------------------------- | ----------------------------------- |
| Client → MDS | NFSv4.1 AUTH_SYS / RPCSEC_GSS | 标准 NFSv4 认证                         |
| Client → DS  | NFSv3 **AUTH_SYS**            | 用 `ffl_user` / `ffl_group` 作为 uid/gid |
MDS 在 LAYOUTGET 时告诉客户端"用哪个 uid/gid 去访问 DS"  
```c
// 合成 UID 模式（有 secret 配置时）
ffl_user = synth_uid_from_secret(secret, fileid, stripe, mirror);
ffl_group = inode.gid;

// 或真实 UID 模式（无 secret 时）
ffl_user = cd->cred_uid;   // 客户端的原始 uid
ffl_group = inode.gid;
```

同时 MDS 在授权布局前对 DS 文件执行 chown（`compound_layout.c`），使 DS 文件的 POSIX 所有权匹配 ffl_user/ffl_group，这样客户端用 AUTH_SYS 发送的 NFSv3 READ/WRITE 就能通过 DS 的权限检查。这就是 RFC 8435 所称的 **loosely coupled model（松耦合模型）**—— MDS 和 DS 不共享认证状态，靠文件所有权对齐实现权限传递。  
### 4.2.6 简要总结  
| 问题               | 答案                                                              |
|------------------|-----------------------------------------------------------------|
| 客户端挂载 MDS 用什么协议？ | NFSv4.1（pNFS 扩展）                                                |
| 客户端访问 DS 用什么协议？  | NFSv3（flex-files 布局的数据协议）                                       |
| 为什么能用 NFSv3？     | RFC 8435 Flex-Files 布局标准设计，MDS 在 LAYOUTGET 中下发 NFSv3 文件句柄       |
| NFSv3 FH 从哪来？    | MDS 通过本地 NFSv3 挂载用 name_to_handle_at() 提取，或回退到 NFSv3 RPC LOOKUP |
| 客户端怎么知道 DS 地址？   | GETDEVICEINFO 返回 IP + port                                      |
| 客户端怎么认证到 DS？     | AUTH_SYS，uid/gid 由 LAYOUTGET 响应中的 ffl_user/ffl_group 指定         |
| 两条路径的数据格式一致吗？    | 完全一致——相同的稀疏寻址、相同的条带公式、相同的 DS 文件布局                               |

# 5 MDS和DS的连接关系  
## 5.1 MDS挂载
客户端一般只访问挂载的那个 MDS, **但 multi-MDS 部署下有 referral 机制可以跨 MDS。**  
1. 单 MDS 场景
客户端挂载一个 MDS 后，所有**控制面操作**（OPEN、LAYOUTGET、LAYOUTCOMMIT、GETDEVICEINFO 等）都发往该 MDS。客户端**不会**主动联系其他 MDS 节点。
```txt
Client ──NFSv4.1──► MDS-1 (port 2050)    ← 唯一的控制面连接
         ╳ ──────► MDS-2 (不通信)
         ╳ ──────► MDS-3 (不通信)
```

2. 多 MDS 场景（referral 拓扑）
在 multi-MDS 部署中，命名空间根目录包含 referral 条目指向不同 MDS, 比如如下拓扑 :  

| Host | Mgmt IP | Internal IP | Role |
|------|---------|-------------|------|
| mds1 | 192.168.100.50 | 10.10.10.50 | `pnfs-mds` + RonDB data node 1 |
| mds2 | 192.168.100.51 | 10.10.10.51 | `pnfs-mds` + RonDB data node 2 |
| mds3 | 192.168.100.55 | 10.10.10.55 | `pnfs-mds` + RonDB management node |
| ds1  | 192.168.100.52 | 10.10.10.52 | NFS data server (`/export/ds`) |
| ds2  | 192.168.100.53 | 10.10.10.53 | NFS data server (`/export/ds`) |
| client1 | 192.168.100.54 | 10.10.10.54 | NFSv4.2 client (`/mnt/pnfs`) |
```txt
/mnt/pnfs/
  ├── shard1/   → referral → MDS-1 @ 10.10.10.50
  ├── shard2/   → referral → MDS-2 @ 10.10.10.51
  └── shard3/   → referral → MDS-3 @ 10.10.10.55
```
当客户端 `cd /mnt/pnfs/shard2/` 时，内核通过 `fs_locations` 发现 referral，**自动建立一条新的 NFSv4.1 连接到 MDS-2**。此后 shard2 下的文件操作都发往 MDS-2。  
但这是**按子树**的亲和关系，不是负载均衡：  
```txt
Client ──NFSv4.1──► MDS-1   (shard1 的文件)
       ──NFSv4.1──► MDS-2   (shard2 的文件)
       ──NFSv4.1──► MDS-3   (shard3 的文件)
```
每个 MDS 通过 referral_strict 配置（`config-keys.md`）控制：操作一个 FH 所属子树不在本 MDS 时，返回 NFS4ERR_MOVED，强制客户端重新走 referral。  
## 5.2 DS挂载  
客户端的 NFSv3 连接数取决于文件的条带集（stripe set）中有多少个不同的 DS，与 MDS 挂载了几个 DS 无关， 即， **MDS 的 DS 挂载 ≠ 客户端的 DS 连接**
```txt
┌─────────────────────────────────────────────┐
│                    MDS                      │
│                                             │
│  MDS 的 NFSv3 挂载（内部用）:                 │
│    /mnt/ds0 → DS0:/export/ds  ← 用于 FH 捕获 │
│    /mnt/ds1 → DS1:/export/ds  ← 用于代理 I/O │
│    /mnt/ds2 → DS2:/export/ds  ← 用于代理 I/O │
│    ... (所有注册的 DS)                        │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│                  Client                     │
│                                             │
│  Client 的 NFSv3 连接（按需建立）:             │
│    取决于文件的 stripe map 引用了哪些 DS        │
└─────────────────────────────────────────────┘
```
客户端 NFSv3 连接的建立过程：  
1. **LAYOUTGET 返回条带映射**：MDS 在响应中列出该文件用到的所有 DS（deviceid + nfs_fh）
2. **GETDEVICEINFO 获取 DS 地址**：客户端对每个唯一的 deviceid 查询 DS 的 IP/port
3. **按需建立 NFSv3 连接**：客户端对每个唯一的 DS 地址建立一条 NFSv3 TCP 连接

举例说明：  
假设集群有 4 个 DS（DS0-DS3），MDS 全部挂载了。  
1. 场景 A：单条带文件（stripe_count=1）
```txt
文件 A 的 stripe map: [DS0]

Client 建立 1 条 NFSv3 连接:
  Client ──NFSv3──► DS0
```
2. 场景 B：4 条带文件（stripe_count=4）
```txt
文件 B 的 stripe map: [DS0, DS1, DS2, DS3]

Client 建立 4 条 NFSv3 连接:
  Client ──NFSv3──► DS0
  Client ──NFSv3──► DS1
  Client ──NFSv3──► DS2
  Client ──NFSv3──► DS3
```

3. 场景 C：多个文件共享 DS
```txt
文件 C 的 stripe map: [DS0, DS1]
文件 D 的 stripe map: [DS0, DS2]

Client 总共建立 3 条 NFSv3 连接（DS0 复用）:
  Client ──NFSv3──► DS0  ← C 和 D 共享
  Client ──NFSv3──► DS1  ← 仅 C
  Client ──NFSv3──► DS2  ← 仅 D
```

客户端 NFSv3 连接数 = 该客户端访问的所有文件中，唯一 DS 地址的数量  
- 如果只有一个 `stripe_count=1` 的文件 → **1 条** NFSv3 连接
- 如果有一个 `stripe_count=N` 的文件 → **N 条** NFSv3 连接
- 连接在首次访问 DS 时建立，后续复用（Linux 内核 flex-files 驱动管理）
- `nconnect` 挂载选项不影响 NFSv3 DS 连接数（它只影响 NFSv4.1 MDS 连接）

总拓扑图：  
```txt
┌──────────┐
│  Client  │
└────┬─────┘
     │
     │ NFSv4.1 (控制面, 1 条 TCP 连接 + nconnect)
     │
     ▼
┌──────────┐
│  MDS-1   │──── 内部 NFSv3 mounts ────► DS0, DS1, DS2, DS3 (全部)
│ (port    │                              ↑
│  2050)   │                              │ 独立
     │                                     │
     │ LAYOUTGET 返回:                      │
     │ "文件X → DS1, DS2"                   │
     │                                     │
     ▼                                     │
┌──────────────────────────────────────────┘
│
│ NFSv3 (数据面, 按需建立, 只连文件用到的 DS)
│
├──► DS1  ← 因为文件 X 的 stripe 0 在 DS1
└──► DS2  ← 因为文件 X 的 stripe 1 在 DS2

注意: 客户端不连接 DS0 和 DS3（除非后续访问的文件用到了它们）
```

MDS 的 DS 挂载是**全量的**（所有注册的 DS），用于内部操作（FH 捕获、代理 I/O、健康探测等）。客户端的 DS 连接是**按需的**，只连 LAYOUTGET 响应中实际引用的 DS。两者互不干扰。  
