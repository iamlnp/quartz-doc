# 1 Service搭建

brpc是C/S架构，在通信建立之前首先需要搭建业务自己的service，Mds Daemon的服务搭建入口是` Tfs_MDSDaemon::start_rpc_server`，这个接口在rank `replay_done`和`creating_done`中调用。

`Tfs_MDSDaemon::start_rpc_server`调用brpc提供的`add_service`接口添加服务。目前mds一共添加以下几类服务：

- fs_service：用于client和mds的普通业务通信server
- tdl_server：client和mds中分布式锁的通信的server
- ioa_service：用于client/rank与mds/其他rank通信的ioa server（ioa主要处理配额和qos业务）
- snap_service： 用于client与mds通信的快照server
- rank_service：rank之前通信的server



client的搭建service的接口是`Tfs_Client::start_rpc_server`。目前client一共有以下几类服务：

- tdl_service： 分布式锁
- notify_service：接收mds发送的心跳请求



## 1. service添加方式
添加service的接口是：   
*server.cpp*
```c++
int Server::AddService(google::protobuf::Service* service,ServiceOwnership ownership);
```
若ownership参数为`SERVER_OWNS_SERVICE`，Server在析构时会一并删除Service，否则应设为`SERVER_DOESNT_OWN_SERVICE`。

Server启动后你无法再修改其中的Service。




