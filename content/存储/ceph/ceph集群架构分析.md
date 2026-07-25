# ceph集群的架构组成
**三类守护进程**
- **ceph osd**：利用Ceph节点上的CPU、内存和网络进行数据复制、纠错、重新平衡、恢复、监控和报告等
- **ceph monitor**：维护Ceph集群的主副本映射、Ceph集群的当前状态以及处理各种与运行控制相关的工作
- **ceph manager**：维护Placement Group（放置组）有关的详细信息，代替Ceph Monitor处理元数据和主机元数据，能显著改善大规模集群的访问性能。

Ceph客户端接口负责和Ceph集群进行数据交互，包括数据的读写。客户端需要以下数据才能与Ceph集群进行通信。
- ceph配置文件或集群的名称、monitor地址
	- 配置文件路径写在代码中：`src/common/config.cc:static const char *CEPH_CONF_FILE_DEFAULT = "$data_dir/config, /etc/ceph/$cluster.conf, $home/.ceph/$cluster.conf, $cluster.conf"`
- 存储池名称
  - 客户端inode中的layout中有pool_id， mds带给的client
- 用户名和密钥路径
  - 此部分内容涉及到mon的用户校验，一般是用cephx验证，客户端需要持有ceph.conf和ceph.client.admin.keyring，一般也是在`/etc/ceph/`目录




## 文件系统

创建文件系统`Tfs_FSMap::create_filesystem`, 路径在`tfs_fsmap.cc`,会构造文件系统的各种参数， `Tfs_FsNewHandler`会传入datapool和metapool，这两个池子都是通过osdmap拿到，因此可以推断是osd从配置或者命令中创建datapool和metapool内存结构。



