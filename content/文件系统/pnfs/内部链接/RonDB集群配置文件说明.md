```bash
[ndb_mgmd] # 管理节点，定义集群的管理服务器
NodeId=65  # 唯一id
HostName=137.33.24.134      # mgmd ip地址
PortNumber=1186             # 监听 1186 端口（RonDB 管理默认端口）
DataDir=/var/lib/rondb/mgm  # 管理节点存放集群日志和配置缓存的目录。

[ndbd default]  # 数据节点全局默认配置
NoOfReplicas=2  # 每份数据有 2 个副本（即容许 1 个数据节点故障）
DataDir=/var/lib/rondb/data
FileSystemPath=/var/lib/rondb/data
DataMemory=2G             # 每个数据节点用于存储数据的主内存为 2GB
SharedGlobalMemory=256M   # 共享全局内存 256MB，用于跨线程的共享资源
RedoBuffer=64M            # 重做日志缓冲区 64MB，用于崩溃恢复
TransactionMemory=128M    # 事务内存 128MB，用于事务执行过程中的中间数据
IndexMemory=64M           # 索引内存 64MB

[ndbd]    # 数据节点定义，配合 NoOfReplicas=2，构成一个完整的数据副本组。
NodeId=1
HostName=137.33.24.132

[ndbd]    # 数据节点定义
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
```