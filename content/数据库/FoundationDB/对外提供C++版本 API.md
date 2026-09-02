---
年份:
  - "2026"
月份:
  - 7月
imageNameKey: 对外提供C++版本 API
创建时间: 2026-07-29 14:09
修改时间: 2026-07-29 14:18
tags:
  - FoundationDB
---
开发版本：8.0.0  
# API层次关系图  
```bash
应用代码
  │
  ├── db.run(fun) ──────────────── 自动重试事务循环
  │     └── Transaction / ReadYourWritesTransaction
  │           ├── get / getRange / getKey ──── 读
  │           ├── set / clear / atomicOp ───── 写
  │           ├── commit ───────────────────── 提交
  │           ├── watch ────────────────────── 变更监控
  │           ├── onError ──────────────────── 重试判断
  │           └── setOption ────────────────── 精细控制
  │
  ├── ManagementAPI ────────────────────────── 集群运维
  │     ├── excludeServers / includeServers
  │     ├── lockDatabase / unlockDatabase
  │     ├── changeQuorum
  │     └── getDatabaseConfiguration
  │
  ├── BackupAgent ──────────────────────────── 备份/DR
  │     ├── FileBackupAgent (文件备份)
  │     └── DatabaseBackupAgent (跨集群DR)
  │
  └── 数据组织 ─────────────────────────────── 逻辑隔离
        ├── Tuple (编解码)
        ├── Subspace (前缀封装)
        └── Directory + DirectoryLayer (命名空间管理)
```

# 网络生命周期  
定义在 `NativeAPI.actor.h`  

| 接口                                    | 作用                           |
|---------------------------------------|------------------------------|
| setupNetwork(transportId, useMetrics) | 初始化网络系统（必须在任何 DB 操作前调用，仅调一次） |
| runNetwork()                          | 阻塞运行网络事件循环（放在专用网络线程）         |
| stopNetwork()                         | 从非网络线程调用，使 runNetwork() 返回   |
| setNetworkOption(option, value)       | 设置全局网络选项（TLS、trace、客户端线程数等）  |

# Database（数据库连接）  
