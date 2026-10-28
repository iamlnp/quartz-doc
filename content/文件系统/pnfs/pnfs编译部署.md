# 1 单节点编译部署
## 1.1 环境概览
### 1.1.1 系统环境

|项目|实际值|项目要求|说明|
|---|---|---|---|
|OS|CentOS 8.4|Ubuntu 22.04+ / Rocky 9+|低版本 CentOS 可用但需处理兼容性|
|GCC|8.4.1|>= 11.1|需临时降低门控，实际编译无错误|
|CMake|3.28.4（pip 安装）|>= 3.20|系统自带 3.18.2 不满足|
|libntirpc|3.2|需要|已安装|
|OpenSSL|1.1.1g|需要|已安装|
|nfs-utils|2.3.3|需要|已安装（DS 角色）|
### 1.1.2 部署拓扑

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
## 1.2 编译步骤

### 1.2.1 升级 CMake
系统自带 CMake 3.18.2 不满足 >= 3.20 的要求，通过 pip 升级：
```bash
pip3 install cmake
# 验证
cmake --version  # 输出: cmake version 3.28.4
```

### 1.2.2 处理 GCC 版本门控
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
### 1.2.3 配置和编译
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

### 1.2.4 编译产出

|二进制|大小|说明|
|---|---|---|
| `build/src/mds/pnfs-mds` |836 KB|MDS 守护进程（主程序）|
| `build/src/tools/mds-admin` |328 KB|管理 CLI 工具|
| `build/src/tools/mds-find` |170 KB|元数据搜索 CLI|
| `build/src/tools/mds-apid` |—|只读 HTTP 查询 API|

### 1.2.5 运行测试
```bash
ctest --test-dir build --output-on-failure -L unit
```

结果：**76 个单元测试全部通过 (100%)**，6 个 FDB/DAOS 测试正确跳过。  
## 1.3 部署步骤

### 1.3.1 设置数据服务器 (DS)
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
### 1.3.2 创建 MDS 配置

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

### 1.3.3 挂载 DS 导出
MDS 需要回环挂载 DS 导出以准备文件句柄：
```bash
mount -t nfs -o vers=3 127.0.0.1:/export/ds deploy/mnt/ds0
```

### 1.3.4 启动 MDS
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

### 1.3.5 客户端挂载
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

## 1.4 使用方法  
### 1.4.1 基本文件操作

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

### 1.4.2 扩展属性 (xattr)
```bash
# 设置
setfattr -n user.comment -v "important data" $MNT/mydata/renamed.txt

# 读取
getfattr -n user.comment --only-values $MNT/mydata/renamed.txt

# 列出所有
getfattr -d $MNT/mydata/renamed.txt
```

### 1.4.3 大文件写入
```bash
# 写入 100MB 文件
dd if=/dev/urandom of=$MNT/mydata/big.bin bs=1M count=100

# 验证 DS 上的数据文件
ls -la /export/ds/data/
# 输出示例:
# -rw-rw-rw- 1 root root 104857600 ... 5_0_0
# 文件命名: {inode}_{stripe}_{mirror}
```

### 1.4.4 可观测性
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

### 1.4.5 服务控制
```bash
# 停止 MDS
kill $(pgrep pnfs-mds)

# 卸载客户端
umount deploy/mnt/pnfs

# 停止 DS
systemctl stop nfs-server
umount deploy/mnt/ds0
```
## 1.5 端口分配  
| 端口    | 服务            | 用途            |
|-------|---------------|---------------|
| 2049  | knfsd         | DS NFS 服务     |
| 2050  | pnfs-mds      | MDS NFS 服务    |
| 50051 | pnfs-mds      | gRPC 集群通信     |
| 9090  | pnfs-mds      | Prometheus 指标 |
| 20048 | mountd_compat | showmount 兼容  |
## 1.6 文件目录结构  
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


# 2 多节点编译部署  
说明：基于V4.2的版本  
## 2.1 编译  

- [lattice pnfs编译详细过程](assets/lattice%20pnfs编译详细过程.md)  

### 2.1.1 安装工具链  

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
### 2.1.2 安装开发依赖  
从 Rocky Linux 8 仓库下载并安装以下 RPM:  
  
| 包名                               | 用途               |
| -------------------------------- | ---------------- |
| openssl-devel-1.1.1k-18.el8_10   | OpenSSL 头文件和库    |
| krb5-devel-1.18.2-34.el8_10      | Kerberos/GSSAPI  |
| libcom_err-devel-1.45.6-7.el8_10 | et/com_err.h 头文件 |
| libtirpc-devel-1.1.4-12.el8_10   | TI-RPC 开发库       |
### 2.1.3 从源码构建 libntirpc v5.8  
RHEL/CentOS 不提供 libntirpc 包，使用项目自带脚本 deps/build-ntirpc.sh：
```shell
git clone --depth 1 -b v5.8 https://github.com/nfs-ganesha/ntirpc.git
cmake -S ntirpc -B ntirpc/build -DCMAKE_BUILD_TYPE=Release \
      -DCMAKE_INSTALL_PREFIX=/usr/local -DUSE_GSS=OFF -DTIRPC_EPOLL=ON
cmake --build ntirpc/build -j$(nproc)
cmake --install ntirpc/build
```
   安装到 `/usr/local/lib64/libntirpc.so` 和 `/usr/local/include/ntirpc/`。  
### 2.1.4 安装 RonDB 26.02.4  
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
### 2.1.5 CMake 配置  
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
### 2.1.6 编译
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
## 2.2 部署  
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
### 2.2.1 在Node C配置和启动RonDB的mgmd(管理节点)  
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

### 2.2.2 在Node A和Node B配置和启动RonDB的data nodes
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

### 2.2.3 确认RonDB启动成功  
在Node C:  
```bash
/opt/rondb/bin/ndb_mgm --ndb-connectstring=137.33.24.134:1186 -e show
```

说明：  
mgmd进程： `/opt/rondb/bin/ndb_mgmd` , 检查方式： `systemctl status rondb-mgmd`
data nodes进程：  `/opt/rondb/bin/ndbmtd`, 检查方式： `systemctl status rondb-ndbmtd`
#### 2.2.3.1 问题1  - node A和node B连接失败
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

#### 2.2.3.2 mysqld连接失败  
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

#### 2.2.3.3 rondb连接不正常  
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
#### 2.2.3.4 问题2  - ndbmtd 内存占用过大
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

### 2.2.4 配置NFS data servers ds1和ds2  

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

检查导出是否生效：可以看到 `/export/ds 137.33.24.0/24` 和 `/export/ds 137.33.24.0/24 `  
```bash
[root@lzhnode1 ~]# showmount -e 137.33.24.132
Export list for 137.33.24.132:
/export/ds 137.33.24.0/24


[root@lzhnode2 system]# showmount -e 137.33.24.133
Export list for 137.33.24.133:
/export/ds 137.33.24.0/24
```

### 2.2.5 在所有MDS节点安装pnfs-mds  
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

### 2.2.6 在客户端上mount对应的命名空间，挂载完成后，可以看到
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
## 2.3 冒烟测试  

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

## 2.4 服务管理  
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