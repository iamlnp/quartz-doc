---
状态:
  - 完成
年份:
  - "2025"
tags:
  - 技术学习
  - 存储调研
---
1. GPFS

一个GPFS集群支持多个fs, 一个fs只支持一个mds，当升级时其mds时要先用mmchmgr进行转移，其迁移时间和系统负载、硬件资源相关，对前台操作会有影响。
[https://www.ibm.com/docs/en/storage-scale/5.2.2?topic=upgrades-performing-rolling-upgrade](https://www.ibm.com/docs/en/storage-scale/5.2.2?topic=upgrades-performing-rolling-upgrade "https://www.ibm.com/docs/en/storage-scale/5.2.2?topic=upgrades-performing-rolling-upgrade")

2. LUSTRE
Lustre的rolling update依赖其failing over机制，其过程和我们大体相同。
[https://doc.lustre.org/lustre_manual.xhtml#configuringfailover](https://doc.lustre.org/lustre_manual.xhtml#configuringfailover "https://doc.lustre.org/lustre_manual.xhtml#configuringfailover")
[https://doc.lustre.org/lustre_manual.xhtml#Upgrading_2.x.x](https://doc.lustre.org/lustre_manual.xhtml#Upgrading_2.x.x "https://doc.lustre.org/lustre_manual.xhtml#Upgrading_2.x.x")

3. Daos
最新的daos 2.6不支持rolling update。
[https://docs.daos.io/v2.6/admin/administration/#software-upgrade](https://docs.daos.io/v2.6/admin/administration/#software-upgrade "https://docs.daos.io/v2.6/admin/administration/#software-upgrade")