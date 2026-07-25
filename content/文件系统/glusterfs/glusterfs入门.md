---
写作年份:
  - "2025"
imageNameKey: glusterfs入门
tags:
  - glusterfs
  - 文件系统
  - 技术学习
---
官方入门文档：https://docs.gluster.org/en/latest/Install-Guide/Common-criteria/#getting-started
安装包：https://www.gluster.org/install/
源码：https://github.com/gluster/glusterfs

# 1 源码安装
1. 参考INSTALL文件：
    1. sh autogen.sh
    2. ./configure
    3. make
    4. make install
# 概念
brick: 指构成卷（volume）的基本存储单元（每个 brick 对应一个本地目录或块设备）
# 2 部署
配置文件路径：/var/lib/glusterd
## 2.1 单节点
### 格式化和挂载

### 错误处理
1. 格式化报错
```bash
[root@ceph-221 cluster]# mkfs.xfs -i size=512 /dev/sdd
mkfs.xfs: /dev/sdd appears to contain an existing filesystem (xfs).
mkfs.xfs: Use the -f option to force overwrite.
```


# 元数据
![[assets/2026-01-09-glusterfs入门-IMG.png|900]]