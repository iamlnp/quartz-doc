---
年份:
  - "2026"
月份:
  - 7月
imageNameKey: FoundationDB
创建时间: 2026-07-29 12:12
修改时间: 2026-07-29 12:12
tags:
  - FoundationDB
---
开发版本：8.0.0
FoundationDB 对外提供**六大类接口**，从底层 C API 到上层语言绑定、命令行工具、管理接口，层次分明。  

# 1 接口层次总览

```
┌──────────────────────────────────────────────────────────────┐
│                    应用层接口                                 │
│  fdbcli (交互式管理)    fdbbackup/fdbrestore (备份CLI)        │
├──────────────────────────────────────────────────────────────┤
│                    语言绑定层                                 │
│  Python │ Java │ Go │ Ruby │ Flow/C++                        │
├──────────────────────────────────────────────────────────────┤
│                    C API (fdb_c.h)                            │
│  Network → Database → Transaction → Future                   │
│  + CDC API (流注册/消费/确认)                                  │
│  + Special Key Space (\xff\xff/... 管理虚拟化)                │
├──────────────────────────────────────────────────────────────┤
│                    内部 C++ API                               │
│  NativeAPI.actor.h (DatabaseContext/Transaction/RYW)          │
│  ManagementAPI.cpp (配置/exclude/include)                     │
│  BackupAgent.h (DatabaseBackupAgent/DR)                       │
└──────────────────────────────────────────────────────────────┘
```
# 2 C API —— 核心公共接口

定义在 `fdb_c.h`，是所有语言绑定的基础。

## 2.1 网络管理

| 函数 | 作用 |
|------|------|
| `fdb_select_api_version(v)` | 选择 API 版本 |
| `fdb_get_max_api_version()` | 获取最大支持版本 |
| `fdb_get_client_version()` | 获取客户端库版本 |
| `fdb_network_set_option(option, value, len)` | 设置网络选项 |
| `fdb_setup_network()` | 初始化网络 |
| `fdb_run_network()` | 运行网络事件循环（阻塞） |
| `fdb_stop_network()` | 停止网络 |
| `fdb_add_network_thread_completion_hook()` | 注册网络线程回调 |

## 2.2 数据库操作

| 函数 | 作用 |
|------|------|
| `fdb_create_database(cluster_file, &db)` | 创建数据库连接 |
| `fdb_create_database_from_connection_string()` | 通过连接字符串创建 |
| `fdb_database_destroy(db)` | 销毁数据库连接 |
| `fdb_database_set_option(db, option, value, len)` | 设置数据库选项 |
| `fdb_database_create_transaction(db, &tr)` | 创建事务 |
| `fdb_database_get_main_thread_busyness(db)` | 获取主线程繁忙程度 |
| `fdb_database_get_server_protocol(db, version)` | 获取服务端协议版本 |
| `fdb_database_get_client_status(db)` | 获取客户端状态 |
| `fdb_database_reboot_worker(db, addr, check, duration)` | 重启工作进程 |
| `fdb_database_force_recovery_with_data_loss(db, dcid)` | 强制恢复（有数据丢失） |
| `fdb_database_create_snapshot(db, uid, cmd)` | 创建快照 |

## 2.3 事务操作

| 函数 | 作用 |
|------|------|
| **读操作** | |
| `fdb_transaction_get(tr, key, snapshot)` | 读取单个 key |
| `fdb_transaction_get_key(tr, key, orEqual, offset, snapshot)` | 按 KeySelector 读取 |
| `fdb_transaction_get_range(tr, begin, end, limit, ...)` | 范围读取 |
| `fdb_transaction_get_mapped_range(tr, ...)` | 映射范围读取（关联查询） |
| `fdb_transaction_get_read_version(tr)` | 获取读版本 |
| `fdb_transaction_get_addresses_for_key(tr, key)` | 获取 key 所在服务器地址 |
| `fdb_transaction_get_estimated_range_size_bytes(tr, begin, end)` | 估算范围大小 |
| `fdb_transaction_get_range_split_points(tr, begin, end, chunk_size)` | 获取范围分裂点 |
| **写操作** | |
| `fdb_transaction_set(tr, key, value)` | 写入 key-value |
| `fdb_transaction_clear(tr, key)` | 删除单个 key |
| `fdb_transaction_clear_range(tr, begin, end)` | 删除范围 |
| `fdb_transaction_atomic_op(tr, key, param, op_type)` | 原子操作（ADD/AND/OR/XOR/MIN/MAX 等） |
| **事务控制** | |
| `fdb_transaction_commit(tr)` | 提交事务 |
| `fdb_transaction_on_error(tr, error)` | 错误处理/重试 |
| `fdb_transaction_reset(tr)` | 重置事务 |
| `fdb_transaction_cancel(tr)` | 取消事务 |
| `fdb_transaction_set_option(tr, option, value, len)` | 设置事务选项 |
| `fdb_transaction_set_read_version(tr, version)` | 手动设置读版本 |
| `fdb_transaction_get_committed_version(tr, &version)` | 获取已提交版本 |
| `fdb_transaction_get_versionstamp(tr)` | 获取 versionstamp |
| `fdb_transaction_watch(tr, key)` | 监控 key 变化 |
| `fdb_transaction_add_conflict_range(tr, begin, end, type)` | 手动添加冲突范围 |
| **统计** | |
| `fdb_transaction_get_tag_throttled_duration(tr)` | 获取限流等待时间 |
| `fdb_transaction_get_total_cost(tr)` | 获取事务总开销 |
| `fdb_transaction_get_approximate_size(tr)` | 获取事务近似大小 |

## 2.4 Future 异步模型

| 函数 | 作用 |
|------|------|
| `fdb_future_destroy(f)` | 释放 Future |
| `fdb_future_cancel(f)` | 取消操作 |
| `fdb_future_block_until_ready(f)` | 阻塞等待完成 |
| `fdb_future_is_ready(f)` | 检查是否完成 |
| `fdb_future_set_callback(f, cb, param)` | 注册回调 |
| `fdb_future_get_error(f)` | 获取错误 |
| `fdb_future_get_bool/int64/uint64/double(f, &out)` | 获取标量结果 |
| `fdb_future_get_key(f, &key, &len)` | 获取 key |
| `fdb_future_get_value(f, &present, &value, &len)` | 获取 value |
| `fdb_future_get_keyvalue_array(f, &kv, &count, &more)` | 获取 KV 数组 |
| `fdb_future_get_mappedkeyvalue_array(f, &kv, &count, &more)` | 获取映射 KV 数组 |
| `fdb_future_get_string_array(f, &strings, &count)` | 获取字符串数组 |
| `fdb_future_get_key_array(f, &keys, &count)` | 获取 key 数组 |
| `fdb_future_get_keyrange_array(f, &ranges, &count)` | 获取 KeyRange 数组 |

## 2.5 错误处理

| 函数 | 作用 |
|------|------|
| `fdb_get_error(code)` | 错误码转描述字符串 |
| `fdb_error_predicate(predicate, code)` | 错误码分类判断（如 `retryable`、`already_committed`） |

## 2.6 数据类型  
定义在 `fdb_c_types.h`：

```c
typedef struct FDB_future FDBFuture;       // 异步结果
typedef struct FDB_result FDBResult;       // 同步结果
typedef struct FDB_database FDBDatabase;   // 数据库连接
typedef struct FDB_transaction FDBTransaction; // 事务
typedef struct FDB_cdc_consumer FDBCdcConsumer; // CDC 消费者

typedef struct { const uint8_t* key; int key_length; } FDBKey;
typedef struct { const uint8_t* key; int key_length; const uint8_t* value; int value_length; } FDBKeyValue;
typedef struct { FDBKey key; fdb_bool_t orEqual; int offset; } FDBKeySelector;
typedef struct { const uint8_t* begin_key; ...; const uint8_t* end_key; ...; } FDBKeyRange;
```


# 3 CDC API（变更数据捕获）

同样定义在 `fdb_c.h`，用于实时订阅数据变更：

| 函数 | 作用 |
|------|------|
| `fdb_database_register_cdc_stream(db, name, begin, end)` | 注册 CDC 流（绑定 key 范围） |
| `fdb_database_remove_cdc_stream(db, name)` | 移除 CDC 流 |
| `fdb_database_list_cdc_streams(db)` | 列出所有 CDC 流 |
| `fdb_database_create_cdc_consumer(db, name)` | 创建消费者 |
| `fdb_database_resume_cdc_consumer(db, stream_id, version)` | 恢复消费 |
| `fdb_cdc_consumer_consume(consumer)` | 消费变更 |
| `fdb_cdc_consumer_acknowledge(consumer)` | 确认消费进度 |
| `fdb_cdc_consumer_get_position(consumer, &stream_id, &version)` | 获取消费位置 |
# 4 语言绑定

基于 C API 封装，位于 `bindings/` 目录：

| 语言 | 目录 | 特点 |
|------|------|------|
| **Python** | [`bindings/python/`](file:///Users/liunaipeng/Documents/code/foundationdb/bindings/python) | `import fdb`，装饰器风格，最常用 |
| **Java** | [`bindings/java/`](file:///Users/liunaipeng/Documents/code/foundationdb/bindings/java) | Maven 包，CompletableFuture 风格 |
| **Go** | [`bindings/go/`](file:///Users/liunaipeng/Documents/code/foundationdb/bindings/go) | goroutine 友好 |
| **Ruby** | [`bindings/ruby/`](file:///Users/liunaipeng/Documents/code/foundationdb/bindings/ruby) | block 风格 |
| **Flow/C++** | [`bindings/flow/`](file:///Users/liunaipeng/Documents/code/foundationdb/bindings/flow) | 服务端内部使用，含 Subspace/Directory/Tuple |
# 5 CLI 命令行接口（fdbcli）

定义在 `fdbcli/`，提供交互式管理：

## 5.1 数据操作

| 命令 | 作用 |
|------|------|
| `get <KEY>` | 读取 key |
| `set <KEY> <VALUE>` | 写入 key（需先 `writemode on`） |
| `clear <KEY>` | 删除 key |
| `clearrange <BEGIN> <END>` | 删除范围 |
| `getrange <BEGIN> <END> [LIMIT]` | 范围读取 |

## 5.2 集群管理

| 命令                           | 作用               |
| ---------------------------- | ---------------- |
| `status [details]`           | 集群状态             |
| `configure [options]`        | 修改集群配置（副本模式、区域等） |
| `fileconfigure <file>`       | 从文件加载配置          |
| `coordinators [addresses]`   | 管理协调者            |
| `exclude [addresses]`        | 排除服务器（下线维护）      |
| `include [addresses]`        | 重新包含服务器          |
| `setclass [address] [class]` | 设置进程类别           |
| `lock` / `unlock`            | 锁定/解锁数据库         |
| `maintenance [on/off]`       | 维护模式             |
| `kill <address>`             | 杀掉进程             |
| `suspend <address>`          | 挂起进程             |
## 5.3 运维诊断

| 命令                                      | 作用       |
| --------------------------------------- | -------- |
| `datadistribution [on\off] `            | 数据分发控制   |
| `consistencycheck`                      | 一致性检查    |
| `consistencyscan`                       | 一致性扫描    |
| `hotrange`                              | 热点范围查询   |
| `profile [client\list\start\stop]`      | 性能分析     |
| `throttle [on\off\list\enable\disable]` | 限流控制     |
| `advanceversion <version>`              | 推进版本     |
| `force_recovery_with_data_loss <dcid>`  | 强制恢复     |
| `snapshot <command>`                    | 快照管理     |
| `rangeconfig <range> [options]`         | 范围配置     |
| `rangelock <range> [options]`           | 范围锁      |
| `locationmetadata`                      | 位置元数据    |
| `checkmetadataencoding`                 | 元数据编码检查  |
| `bulkload` / `bulkdump`                 | 批量加载/导出  |
| `auditstorage` / `getauditstatus`       | 存储审计     |
| `idempotencyids`                        | 幂等 ID 管理 |
| `versionepoch`                          | 版本纪元     |
| `debug` / `expensive_data_check`        | 调试/深度检查  |
# 6 Special Key Space —— 管理 API 虚拟化

定义在 `SpecialKeySpace.h`，将管理操作映射为 `\xff\xff` 前缀下的键值读写：

| 模块 | 前缀 | 作用 |
|------|------|------|
| `MANAGEMENT` | `\xff\xff/management/` | exclude/include、锁定、维护等 |
| `CONFIGURATION` | `\xff\xff/configuration/` | 集群配置读写 |
| `STATUSJSON` | `\xff\xff/status/` | 集群状态 JSON |
| `CLUSTERFILEPATH` | `\xff\xff/cluster_file_path/` | 集群文件路径 |
| `CLUSTERID` | `\xff\xff/cluster_id/` | 集群不可变 ID |
| `CONNECTIONSTRING` | `\xff\xff/connection_string/` | 连接字符串 |
| `GLOBALCONFIG` | `\xff\xff/global_config/` | 全局配置 |
| `METRICS` | `\xff\xff/metrics/` | 数据分布指标 |
| `TRANSACTION` | `\xff\xff/transaction/` | 事务冲突范围信息 |
| `TRACING` | `\xff\xff/tracing/` | 分布式追踪配置 |
| `ACTORLINEAGE` | `\xff\xff/actor_lineage/` | Actor 采样数据 |
| `ACTOR_PROFILER_CONF` | `\xff\xff/actor_profiler_conf/` | 分析器配置 |
| `WORKERINTERFACE` | `\xff\xff/worker_interfaces/` | Worker 接口信息 |
| `ERRORMSG` | `\xff\xff/error_message/` | 最近错误信息 |

**使用方式**：普通事务读写这些 key 即可触发管理操作，无需额外 API。  

# 7 备份/恢复 CLI 工具

| 工具           | 作用                                   |
| ------------ | ------------------------------------ |
| `fdbbackup`  | 备份管理（start/status/abort/discontinue） |
| `fdbrestore` | 恢复管理（start/status/abort）             |
| `fdbdecode`  | 解码备份文件                               |

