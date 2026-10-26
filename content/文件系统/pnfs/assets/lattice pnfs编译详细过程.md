# pnfs-lattice 编译指南

> 基于 CentOS 8 环境，使用 GCC 14 + RonDB 26.02.4 从零构建 pnfs-lattice 的完整记录。

---

## 一、项目概况

| 属性 | 值 |
|------|-----|
| 项目 | pnfs-lattice — pNFS Flex Files 弹性元数据服务器 |
| 版本 | 0.1.0（commit `63b191a`） |
| 构建系统 | CMake 3.20+ |
| 语言 | C（主体）+ C++（RonDB NDB API 绑定） |
| 许可证 | MIT |
| 仓库 | https://github.com/PEAK-AIO/pnfs-lattice |

---

## 二、环境要求

### 2.1 核心依赖

| 依赖 | 最低版本 | 用途 |
|------|---------|------|
| Linux | — | 运行平台 |
| CMake | 3.20+ | 构建系统 |
| GCC | 11.1+（推荐 13+） | C/C++ 编译器 |
| OpenSSL | 1.1+ | HMAC 凭证派生 |
| libntirpc | 5.8 | NFS RPC 传输层 |
| pthreads | — | 线程支持 |

### 2.2 可选依赖

| 依赖 | 版本 | 构建选项 | 用途 |
|------|------|---------|------|
| RonDB | 26.02+ | `ENABLE_RONDB=ON` | 分布式元数据目录后端 |
| CMocka | — | `ENABLE_TESTS=ON` | 单元测试框架 |
| libbpf | — | `ENABLE_EBPF=ON` | eBPF 监控 |
| gRPC + protobuf | — | — | MDS 间 gRPC 通信 |
| FoundationDB | 7.3 | `ENABLE_FDB=ON` | FDB 目录后端 |
| DAOS | 2.6 | `ENABLE_DAOS=ON` | DAOS 目录后端 |

---

## 三、初始环境诊断

在开始编译前，先检查当前环境是否满足要求：

```bash
# 操作系统
cat /etc/os-release

# 编译器版本
gcc --version

# CMake
cmake --version

# 可用仓库
dnf repolist

# OpenSSL
ls /usr/lib64/libssl* /usr/lib64/libcrypto*
ls /usr/include/openssl/ 2>/dev/null

# libntirpc
find / -name "libntirpc.so*" 2>/dev/null

# RonDB
find / -name "NdbApi.hpp" -o -name "libndbclient*" 2>/dev/null
```

**本文实际环境**：

| 检查项 | 结果 | 是否满足 |
|--------|------|---------|
| OS | CentOS 8 (glibc 2.28) | 满足 |
| GCC | 8.4.1 | **不满足**（需 ≥ 11.1） |
| CMake | 未安装 | **不满足** |
| 仓库 | 仅 `local`（基础包） | 受限 |
| OpenSSL 库 | 1.1.1g 已安装 | 满足 |
| OpenSSL 头文件 | 缺失 | **不满足** |
| libntirpc | 未安装 | **不满足** |
| RonDB | 未安装 | **不满足** |

---

## 四、工具链安装

### 4.1 安装 CMake

系统 `dnf` 仓库无 cmake 包，通过 Python pip 安装：

```bash
pip3 install cmake
```

验证：

```bash
cmake --version
# cmake version 3.28.4
```

### 4.2 安装 GCC 14.2.1

#### 4.2.1 问题说明

系统 GCC 8.4.1 不满足项目 `CMakeLists.txt` 第 138 行的版本门控：

```cmake
if(CMAKE_C_COMPILER_VERSION VERSION_LESS "11.1")
    message(FATAL_ERROR
        "pnfs-mds requires GCC >= 11.1 (found ${CMAKE_C_COMPILER_VERSION}).")
endif()
```

#### 4.2.2 尝试 el9 包（失败）

```bash
# 下载 Rocky Linux 9 的 gcc-toolset-13 RPM
cd /tmp
BASE="https://dl.rockylinux.org/pub/rocky/9/AppStream/x86_64/os/Packages"
curl -sLO "$BASE/g/gcc-toolset-13-gcc-13.3.1-2.3.el9.x86_64.rpm"
curl -sLO "$BASE/g/gcc-toolset-13-gcc-c++-13.3.1-2.3.el9.x86_64.rpm"

mkdir -p /opt/gcc13
for rpm in gcc-toolset-13-*.rpm; do
    rpm2cpio "$rpm" | cpio -idm --directory=/opt/gcc13
done

# 测试
/opt/gcc13/opt/rh/gcc-toolset-13/root/usr/bin/gcc --version
# 错误: /lib64/libc.so.6: version `GLIBC_2.32' not found
# 原因: el9 二进制依赖 glibc 2.32+，CentOS 8 只有 glibc 2.28
```

#### 4.2.3 使用 el8 包（成功）

```bash
cd /tmp
BASE="https://dl.rockylinux.org/pub/rocky/8/AppStream/x86_64/os/Packages/g"

# 下载 5 个 RPM 包
curl -sLO "$BASE/gcc-toolset-14-gcc-14.2.1-11.el8_10.x86_64.rpm"              # 51M
curl -sLO "$BASE/gcc-toolset-14-gcc-c++-14.2.1-11.el8_10.x86_64.rpm"          # 16M
curl -sLO "$BASE/gcc-toolset-14-binutils-2.41-4.el8_10.x86_64.rpm"            # 7.6M
curl -sLO "$BASE/gcc-toolset-14-runtime-14.0-1.el8_10.x86_64.rpm"             # 27K
curl -sLO "$BASE/gcc-toolset-14-libstdc++-devel-14.2.1-11.el8_10.x86_64.rpm"  # 3.8M

# 解压到 /opt/gcc14
mkdir -p /opt/gcc14
for rpm in gcc-toolset-14-*.rpm; do
    rpm2cpio "$rpm" | cpio -idm --directory=/opt/gcc14
done
```

安装后的目录结构：

```
/opt/gcc14/opt/rh/gcc-toolset-14/root/usr/
├── bin/
│   ├── gcc              # GCC 14.2.1 主编译器
│   ├── g++              # G++ 14.2.1 C++ 编译器
│   ├── as               # 汇编器
│   └── ld               # 链接器
├── lib/
│   └── libstdc++.so*    # C++ 标准库
├── lib64/
│   ├── libbfd-2.41-4.el8_10.so   # binutils 库
│   └── libstdc++.so*
└── libexec/gcc/x86_64-redhat-linux/14/
    ├── cc1              # C 编译器后端
    └── as               # 汇编器
```

验证：

```bash
GCC14_ROOT=/opt/gcc14/opt/rh/gcc-toolset-14/root/usr
export PATH=$GCC14_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$GCC14_ROOT/lib64:$GCC14_ROOT/lib

gcc --version
# gcc (GCC) 14.2.1 20250110 (Red Hat 14.2.1-11)

echo 'int main() { return 0; }' > /tmp/test.c
gcc -o /tmp/test_bin /tmp/test.c && echo "GCC 14 编译测试通过"
```

### 4.3 安装开发依赖库

#### 4.3.1 OpenSSL 开发包

```bash
cd /tmp
curl -sLO "https://dl.rockylinux.org/pub/rocky/8/BaseOS/x86_64/os/Packages/o/openssl-devel-1.1.1k-18.el8_10.x86_64.rpm"
rpm -Uvh --nodeps openssl-devel-1.1.1k-18.el8_10.x86_64.rpm

# 修复符号链接（openssl-devel 创建了指向 1.1.1k 的链接，但实际库文件是 1.1.1g）
ln -sf libcrypto.so.1.1.1g /usr/lib64/libcrypto.so
ln -sf libssl.so.1.1.1g /usr/lib64/libssl.so
```

#### 4.3.2 Kerberos 开发包

```bash
cd /tmp
curl -sLO "https://dl.rockylinux.org/pub/rocky/8/BaseOS/x86_64/os/Packages/k/krb5-devel-1.18.2-34.el8_10.x86_64.rpm"
rpm -Uvh --nodeps krb5-devel-1.18.2-34.el8_10.x86_64.rpm
```

#### 4.3.3 libcom_err 开发包

提供 `et/com_err.h` 头文件（`krb5.h` 依赖它）：

```bash
cd /tmp
curl -sLO "https://dl.rockylinux.org/pub/rocky/8/BaseOS/x86_64/os/Packages/l/libcom_err-devel-1.45.6-7.el8_10.x86_64.rpm"
rpm -Uvh --nodeps libcom_err-devel-1.45.6-7.el8_10.x86_64.rpm
```

#### 4.3.4 libtirpc 开发包

```bash
cd /tmp
curl -sLO "https://dl.rockylinux.org/pub/rocky/8/BaseOS/x86_64/os/Packages/l/libtirpc-devel-1.1.4-12.el8_10.x86_64.rpm"
rpm -Uvh --nodeps libtirpc-devel-1.1.4-12.el8_10.x86_64.rpm
```

#### 4.3.5 手动创建缺失的链接文件

```bash
ln -sf libnsl.so.2 /usr/lib64/libnsl.so
ln -sf libcom_err.so.2 /usr/lib64/libcom_err.so
```

---

## 五、从源码构建 libntirpc v5.8

### 5.1 背景

libntirpc 是 NFS-Ganesha 项目的 TI-RPC 库，用于 NFSv4.1 RPC 处理。Debian/Ubuntu 提供 `libntirpc-dev` 包，但 RHEL/CentOS 不提供，需要从源码构建。

### 5.2 构建步骤

```bash
GCC14_ROOT=/opt/gcc14/opt/rh/gcc-toolset-14/root/usr
export PATH=$GCC14_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$GCC14_ROOT/lib64:$GCC14_ROOT/lib
export CC=$GCC14_ROOT/bin/gcc
export CXX=$GCC14_ROOT/bin/g++

# 克隆源码
rm -rf /tmp/ntirpc-build
git clone --depth 1 -b v5.8 https://github.com/nfs-ganesha/ntirpc.git /tmp/ntirpc-build

# 配置（关闭 GSS，启用 epoll）
cmake -S /tmp/ntirpc-build -B /tmp/ntirpc-build/build \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_INSTALL_PREFIX=/usr/local \
  -DUSE_GSS=OFF \
  -DTIRPC_EPOLL=ON

# 编译
cmake --build /tmp/ntirpc-build/build -j$(nproc)

# 安装
cmake --install /tmp/ntirpc-build/build
```

### 5.3 安装结果

```
/usr/local/lib64/libntirpc.so.5.8
/usr/local/lib64/libntirpc.so -> libntirpc.so.5.8
/usr/local/include/ntirpc/          （头文件目录）
/usr/local/lib64/pkgconfig/libntirpc.pc
```

### 5.4 更新动态链接器缓存

```bash
echo "/usr/local/lib64" > /etc/ld.so.conf.d/ntirpc.conf
echo "/usr/local/lib"  >> /etc/ld.so.conf.d/ntirpc.conf
ldconfig
```

---

## 六、安装 RonDB 26.02.4

### 6.1 背景

RonDB 是 Logical Clocks 基于 MySQL NDB Cluster 的分布式数据库，用作 pNFS 元数据目录后端。

### 6.2 下载与安装

```bash
# 下载（716MB）
wget -O /tmp/rondb-26.02.4.tar.gz \
  https://repo.hops.works/master/rondb-26.02.4-linux-glibc2.28-x86_64.tar.gz

# 解压到 /opt
tar xzf /tmp/rondb-26.02.4.tar.gz -C /opt/

# 创建符号链接
ln -sfn /opt/rondb-26.02.4-linux-glibc2.28-x86_64 /opt/rondb

# 配置动态链接器
echo "/opt/rondb/lib"       > /etc/ld.so.conf.d/rondb.conf
echo "/opt/rondb/lib/mysql" >> /etc/ld.so.conf.d/rondb.conf
ldconfig
```

### 6.3 安装目录结构

```
/opt/rondb/
├── bin/
│   ├── ndb_mgm              # 管理客户端
│   ├── ndb_mgmd             # 管理服务
│   ├── ndbmtd               # 数据节点
│   └── ...
├── include/
│   └── storage/ndb/
│       └── ndbapi/
│           ├── NdbApi.hpp   # C++ API 头文件（CMake 查找目标）
│           ├── Ndb.hpp
│           └── ...
├── lib/
│   ├── libndbclient.so      # NDB 客户端共享库（CMake 查找目标）
│   ├── libndbclient.so.6.1.0
│   ├── libmysqlclient.so
│   └── ...
└── lib64/
    └── ...
```

### 6.4 验证安装

```bash
/opt/rondb/bin/ndb_mgm --version
# 输出: RonDB distrib RonDB-26.02.4, for Linux (x86_64)
```

---

## 七、CMake 配置

### 7.1 配置命令

```bash
cd /home/code/pnfs-lattice

GCC14_ROOT=/opt/gcc14/opt/rh/gcc-toolset-14/root/usr
rm -rf build   # 清除旧的构建目录

CC=$GCC14_ROOT/bin/gcc \
CXX=$GCC14_ROOT/bin/g++ \
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

### 7.2 构建选项说明

| 选项 | 默认值 | 本文设置 | 说明 |
|------|--------|---------|------|
| `ENABLE_RONDB` | ON | ON | 启用 RonDB 目录后端 |
| `RonDB_ROOT` | — | /opt/rondb | RonDB 安装路径 |
| `ENABLE_MEMDB_BACKEND` | ON | ON | 同时编译内存后端 |
| `ENABLE_EBPF` | ON | OFF | eBPF 监控（需要 libbpf） |
| `ENABLE_ADMIN_CLI` | ON | ON | mds-admin 管理工具 |
| `ENABLE_TESTS` | ON | ON | 单元测试和一致性测试 |
| `ENABLE_FDB` | OFF | OFF | FoundationDB 后端 |
| `ENABLE_DAOS` | OFF | OFF | DAOS 后端 |
| `ENABLE_OP_METRICS` | ON | ON（默认） | 每操作延迟直方图 |
| `ENABLE_RESILVER` | OFF | OFF（默认） | 企业版模块 |
| `ENABLE_REBALANCE` | OFF | OFF（默认） | 企业版模块 |
| `ENABLE_TIERING` | OFF | OFF（默认） | 企业版模块 |
| `ENABLE_REPLICATION` | OFF | OFF（默认） | 企业版模块 |

### 7.3 预期配置输出

```
-- The C compiler identification is GNU 14.2.1
-- The CXX compiler identification is GNU 14.2.1
-- pnfs-mds git commit: 63b191a
-- Found Threads: TRUE
-- Found OpenSSL: /usr/lib64/libcrypto.so (found version "1.1.1k")
-- libntirpc: /usr/local/lib64/libntirpc.so
-- Found RonDB: /opt/rondb/lib/libndbclient.so
-- RonDB found -- RonDB catalogue backend enabled
-- resilver: disabled (community build) -- using stubs
-- rebalance: disabled (community build) -- using stubs
-- tiering: disabled (community build) -- using stubs
-- observability: community build -- using metrics_http_lite
-- replication: disabled (community build) -- using stubs
-- Configuring done
-- Generating done
-- Build files have been written to: /home/code/pnfs-lattice/build
```

---

## 八、编译

### 8.1 编译命令

```bash
GCC14_ROOT=/opt/gcc14/opt/rh/gcc-toolset-14/root/usr
CC=$GCC14_ROOT/bin/gcc \
CXX=$GCC14_ROOT/bin/g++ \
LD_LIBRARY_PATH=$GCC14_ROOT/lib64:$GCC14_ROOT/lib:/opt/rondb/lib \
cmake --build build -j$(nproc)
```

### 8.2 编译过程

```
[  0-10%]  公共库 libpnfs_common.a
           (metrics, config, log, tls, heartbeat, error, catalogue_backend_names...)

[ 10-17%]  RonDB schema、测试框架 (conformance_harness)

[ 17-36%]  FSAL 对象层 (layout, lease, placement, referral, ds_file...)
           集群层 (rename_2pc, hardlink_2pc, failover, migration, drain...)

[ 36-100%] MDS 核心 (compound, session, delegation, inode_cache, RPC server...)
           + RonDB 目录后端 (catalogue_rondb.c)
           + memdb 目录后端 (catalogue_memdb.c)
           + 各模块 stub 实现
           + 工具 (mds-admin, mds-apid, mds-find)
           + 80+ 测试二进制
           + 基准测试 (bench_create_layout_fusion, bench_mk_rm_scale)

[100%]     编译完成，无错误
```

### 8.3 编译警告（无害）

```
ld: /lib64/libjansson.so.4: no version information available
```

这是 GCC 14 链接器对系统 jansson 库版本的提示，不影响功能。

---

## 九、验证

### 9.1 验证 RonDB 安装

```bash
/opt/rondb/bin/ndb_mgm --version
# 预期: RonDB distrib RonDB-26.02.4, for Linux (x86_64)
```

### 9.2 验证二进制产物

```bash
ls -lh build/src/mds/pnfs-mds       # 1.3M（含 RonDB 后端）
ls -lh build/src/tools/mds-admin    # 339K
ls -lh build/src/tools/mds-apid     # 668K
ls -lh build/src/tools/mds-find     # 658K
```

> **对比**：不含 RonDB 的构建中 `pnfs-mds` 为 843K，含 RonDB 后增长到 1.3M，说明 RonDB 代码已编入。

### 9.3 验证动态链接

```bash
LD_LIBRARY_PATH=/opt/rondb/lib:/opt/rondb/lib/mysql:/usr/local/lib64 \
  ldd build/src/mds/pnfs-mds
```

预期输出中应包含：

```
libndbclient.so.6.1.0 => /opt/rondb/lib/libndbclient.so.6.1.0
libssl.so.3 => /opt/rondb/lib/private/libssl.so.3
libcrypto.so.3 => /opt/rondb/lib/private/libcrypto.so.3
libntirpc.so => /usr/local/lib64/libntirpc.so
libpthread.so.0 => /lib64/libpthread.so.0
libc.so.6 => /lib64/libc.so.6
```

关键确认：`libndbclient.so.6.1.0` 被正确解析到 `/opt/rondb/lib/`。

### 9.4 验证 RonDB 符号

```bash
nm build/src/mds/pnfs-mds | grep catalogue_rondb
```

预期输出（部分）：

```
00000000004731e0 T catalogue_rondb_alloc_fileid_range
0000000000473240 T catalogue_rondb_alloc_gc_seq_range
000000000046c040 t catalogue_rondb_backend_client_stats
000000000046bf60 t catalogue_rondb_backend_handle
000000000046e690 t catalogue_rondb_client_del
000000000046e6f0 t catalogue_rondb_client_get
000000000046e750 t catalogue_rondb_client_put
0000000000470450 t catalogue_rondb_close_backend
000000000046e870 t catalogue_rondb_deleg_del
000000000046e8d0 t catalogue_rondb_deleg_get
```

- `T`（大写）= 全局符号
- `t`（小写）= 局部符号

两者都存在说明 RonDB 后端完整编入。

### 9.5 验证 HAVE_RONDB 编译宏

```bash
grep "HAVE_RONDB" build/src/mds/CMakeFiles/pnfs_mds_core.dir/flags.make
```

应包含 `-DHAVE_RONDB=1`。

### 9.6 验证 memdb 后端

```bash
nm build/src/mds/pnfs-mds | grep catalogue_memdb
```

预期包含 `catalogue_memdb_*` 符号。运行时通过 `mds.conf` 选择后端：

```ini
catalogue_backend = rondb    # 或 memdb
```

### 9.7 运行单元测试

```bash
LD_LIBRARY_PATH=/opt/rondb/lib:/opt/rondb/lib/mysql:/usr/local/lib64 \
  ctest --test-dir build --output-on-failure
```

### 9.8 列出所有测试二进制

```bash
find build/tests -maxdepth 2 -type f -executable | sort
```

---

## 十、后续增量编译

代码修改后，只需重新编译变更部分：

```bash
GCC14_ROOT=/opt/gcc14/opt/rh/gcc-toolset-14/root/usr
export PATH=$GCC14_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$GCC14_ROOT/lib64:$GCC14_ROOT/lib:/opt/rondb/lib
export CC=$GCC14_ROOT/bin/gcc
export CXX=$GCC14_ROOT/bin/g++

cd /home/code/pnfs-lattice
cmake --build build -j$(nproc)
```

完全重新配置：

```bash
rm -rf build
# 然后重新执行第七节的 cmake -B build 命令
```

---

## 十一、一键编译脚本

将以下步骤保存为 `build.sh`：

```bash
#!/usr/bin/env bash
set -euo pipefail

GCC14_ROOT=/opt/gcc14/opt/rh/gcc-toolset-14/root/usr
export PATH="$GCC14_ROOT/bin:$PATH"
export LD_LIBRARY_PATH="$GCC14_ROOT/lib64:$GCC14_ROOT/lib:/opt/rondb/lib:/usr/local/lib64"
export CC="$GCC14_ROOT/bin/gcc"
export CXX="$GCC14_ROOT/bin/g++"

cd "$(dirname "$0")"

echo "==> Configuring..."
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

echo "==> Building ($(nproc) jobs)..."
cmake --build build -j"$(nproc)"

echo "==> Verifying..."
ls -lh build/src/mds/pnfs-mds
ldd build/src/mds/pnfs-mds | grep ndb
nm build/src/mds/pnfs-mds | grep -c catalogue_rondb | xargs -I{} echo "    RonDB symbols: {}"

echo "==> Done."
```

使用方法：

```bash
chmod +x build.sh
./build.sh
```

---

## 十二、问题排查

### 12.1 遇到的问题及解决方案

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| CMake 未安装 | CentOS 8 基础仓库无 cmake | `pip3 install cmake` |
| GCC 8.4.1 版本过低 | 项目要求 ≥ 11.1 | 从 Rocky 8 仓库下载 gcc-toolset-14 RPM |
| el9 的 GCC 13 不兼容 | 需要 glibc 2.32+，系统只有 2.28 | 改用 el8 的 GCC 14 包 |
| OpenSSL 库链接失败 | libcrypto.so 指向不存在的 1.1.1k | 手动修复符号链接指向 1.1.1g |
| `et/com_err.h` 缺失 | krb5-devel 不包含此头文件 | 安装 libcom_err-devel |
| `libnsl.so` 链接失败 | 只有 libnsl.so.2，无 .so 链接 | 手动创建符号链接 |
| libntirpc 不可用 | RHEL/CentOS 不提供此包 | 从 GitHub 源码构建 v5.8 |
| `libbfd-2.41.so` 找不到 | GCC 14 的 binutils 需要此库 | 设置 `LD_LIBRARY_PATH` 包含 gcc-toolset 的 lib64 |
| `OPENSSL_CRYPTO_LIBRARY` NOTFOUND | openssl-devel 与 openssl-libs 版本不匹配 | 修复 libcrypto.so 符号链接 |

### 12.2 常见运行时问题

**`pnfs-mds` 启动报 `failed to connect to RonDB`**：
```bash
/opt/rondb/bin/ndb_mgm --ndb-connectstring=<host>:1186 -e show
# 检查 RonDB 集群是否正常运行
```

**`ldd` 显示 `not found`**：
```bash
export LD_LIBRARY_PATH=/opt/rondb/lib:/opt/rondb/lib/mysql:/usr/local/lib64
ldd build/src/mds/pnfs-mds
```

---

## 附录 A：完整依赖安装清单

```bash
# 1. CMake
pip3 install cmake

# 2. GCC 14（从 Rocky Linux 8 仓库）
cd /tmp
BASE="https://dl.rockylinux.org/pub/rocky/8/AppStream/x86_64/os/Packages/g"
for pkg in \
  gcc-toolset-14-gcc-14.2.1-11.el8_10.x86_64.rpm \
  gcc-toolset-14-gcc-c++-14.2.1-11.el8_10.x86_64.rpm \
  gcc-toolset-14-binutils-2.41-4.el8_10.x86_64.rpm \
  gcc-toolset-14-runtime-14.0-1.el8_10.x86_64.rpm \
  gcc-toolset-14-libstdc++-devel-14.2.1-11.el8_10.x86_64.rpm
do
  curl -sLO "$BASE/$pkg"
done
mkdir -p /opt/gcc14
for rpm in gcc-toolset-14-*.rpm; do
  rpm2cpio "$rpm" | cpio -idm --directory=/opt/gcc14
done

# 3. 开发库（从 Rocky Linux 8 仓库）
BASE_OS="https://dl.rockylinux.org/pub/rocky/8/BaseOS/x86_64/os/Packages"
cd /tmp
curl -sLO "$BASE_OS/o/openssl-devel-1.1.1k-18.el8_10.x86_64.rpm"
curl -sLO "$BASE_OS/k/krb5-devel-1.18.2-34.el8_10.x86_64.rpm"
curl -sLO "$BASE_OS/l/libcom_err-devel-1.45.6-7.el8_10.x86_64.rpm"
curl -sLO "$BASE_OS/l/libtirpc-devel-1.1.4-12.el8_10.x86_64.rpm"
rpm -Uvh --nodeps openssl-devel-*.rpm krb5-devel-*.rpm \
  libcom_err-devel-*.rpm libtirpc-devel-*.rpm

# 4. 修复符号链接
ln -sf libcrypto.so.1.1.1g /usr/lib64/libcrypto.so
ln -sf libssl.so.1.1.1g    /usr/lib64/libssl.so
ln -sf libnsl.so.2         /usr/lib64/libnsl.so
ln -sf libcom_err.so.2     /usr/lib64/libcom_err.so

# 5. libntirpc（从源码）
GCC14_ROOT=/opt/gcc14/opt/rh/gcc-toolset-14/root/usr
export PATH=$GCC14_ROOT/bin:$PATH
export LD_LIBRARY_PATH=$GCC14_ROOT/lib64:$GCC14_ROOT/lib
export CC=$GCC14_ROOT/bin/gcc  CXX=$GCC14_ROOT/bin/g++
git clone --depth 1 -b v5.8 https://github.com/nfs-ganesha/ntirpc.git /tmp/ntirpc-build
cmake -S /tmp/ntirpc-build -B /tmp/ntirpc-build/build \
  -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=/usr/local \
  -DUSE_GSS=OFF -DTIRPC_EPOLL=ON
cmake --build /tmp/ntirpc-build/build -j$(nproc)
cmake --install /tmp/ntirpc-build/build
echo "/usr/local/lib64" > /etc/ld.so.conf.d/ntirpc.conf
echo "/usr/local/lib"  >> /etc/ld.so.conf.d/ntirpc.conf
ldconfig

# 6. RonDB 26.02.4
wget -O /tmp/rondb-26.02.4.tar.gz \
  https://repo.hops.works/master/rondb-26.02.4-linux-glibc2.28-x86_64.tar.gz
tar xzf /tmp/rondb-26.02.4.tar.gz -C /opt/
ln -sfn /opt/rondb-26.02.4-linux-glibc2.28-x86_64 /opt/rondb
echo "/opt/rondb/lib"       > /etc/ld.so.conf.d/rondb.conf
echo "/opt/rondb/lib/mysql" >> /etc/ld.so.conf.d/rondb.conf
ldconfig
```

## 附录 B：最终产物清单

| 二进制 | 大小 | 说明 |
|--------|------|------|
| `build/src/mds/pnfs-mds` | 1.3M | MDS 守护进程（含 RonDB + memdb 后端） |
| `build/src/tools/mds-admin` | 339K | 管理 CLI |
| `build/src/tools/mds-apid` | 668K | HTTP API 守护进程 |
| `build/src/tools/mds-find` | 658K | 查询工具 |
| `build/tests/test_*` | 80+ 个 | 单元/集成测试 |
| `build/tests/bench_*` | 2 个 | 基准测试 |
| `build/tests/catalogue_conformance/*` | 6 个 | 目录一致性测试 |
