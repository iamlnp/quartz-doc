**时间：** 2026-10-23  
**环境：** RonDB 26.02.4 集群（pNFS Lab）  
**节点分布：**
- 管理节点（MGM）：137.33.24.134
- 数据节点（NDB）：137.33.24.132, 137.33.24.133
- MySQL API 节点：应分布在 132, 133, 134 三台机器

# 1 问题诊断

## 1.1 初始状态检查
执行命令：
```bash
/opt/rondb/bin/ndb_mgm --ndb-connectstring=137.33.24.134:1186 -e show
```

检查结果：
- ✅ 数据节点（NDB）：2 个节点正常运行
  - id=1 @137.33.24.132 (RonDB-26.02.4, Nodegroup: 0)
  - id=2 @137.33.24.133 (RonDB-26.02.4, Nodegroup: 0)
- ✅ 管理节点（MGM）：1 个节点正常运行
  - id=65 @137.33.24.134 (RonDB-26.02.4)
- ❌ **MySQL API 节点：全部 22 个节点未连接**

## 1.2 问题影响
- 集群无法提供 SQL 服务
- 应用程序无法通过 MySQL 协议访问数据
- 虽然数据节点正常，但缺少 SQL 接入层，集群不可用

## 1.3 根因分析
排查步骤：
```bash
# 检查 MySQL 进程
ps aux | grep mysqld

# 检查 MySQL 数据目录
ls -la /var/lib/rondb/

# 检查配置文件
find /etc -name "my*.cnf"

# 检查 systemd 服务
systemctl list-units --type=service | grep -i mysql
```

发现的问题：
1. MySQL 数据目录 `/var/lib/rondb/mysql` 不存在
2. 没有 RonDB MySQL 服务器的配置文件
3. 没有 MySQL API 节点的 systemd 服务
4. 只有 Bitnami MariaDB 在运行，与 RonDB 集群无关

**根本原因：** MySQL API 节点从未被初始化和启动。

# 2 解决方案

## 2.1 创建 MySQL 配置文件
文件路径：`/etc/rondb/my.cnf`

关键配置：
```ini
[mysqld]
# 基础配置
basedir=/opt/rondb
datadir=/var/lib/rondb/mysql
socket=/var/lib/rondb/mysql/mysql.sock
port=3306
bind-address=0.0.0.0

# NDB Cluster 配置（核心）
ndbcluster
ndb-connectstring=137.33.24.134:1186
ndb-nodeid=105
default-storage-engine=ndbcluster
default-tmp-storage-engine=ndbcluster

# 日志配置
log-error=/var/lib/rondb/mysql/error.log
slow-query-log=1
slow-query-log-file=/var/lib/rondb/mysql/slow.log

# 性能配置
max_connections=500
innodb_buffer_pool_size=512M
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci
```

配置说明：
- `ndb-nodeid=105`：对应 config.ini 中定义的 API 节点 ID（134 机器的第一个节点）
- `ndb-connectstring`：指向管理节点地址
- `default-storage-engine=ndbcluster`：默认使用 NDB 引擎

## 2.2 初始化 MySQL 数据目录
执行命令：
```bash
# 创建数据目录
mkdir -p /var/lib/rondb/mysql
chown -R root:root /var/lib/rondb/mysql

# 初始化数据目录（无密码模式）
/opt/rondb/bin/mysqld \
  --defaults-file=/etc/rondb/my.cnf \
  --initialize-insecure \
  --user=root
```

初始化结果：
- 创建了系统表空间（ibdata1）
- 创建了 InnoDB 重做日志
- 初始化了 mysql、performance_schema、sys 等系统数据库
- 创建了无密码的 root 用户

## 2.3 创建 systemd 服务
服务文件：`/etc/systemd/system/rondb-mysqld.service`
服务配置：
```ini
[Unit]
Description=RonDB MySQL Server (API Node)
Documentation=man:mysqld(8)
After=network.target rondb-mgmd.service
Wants=network.target

[Service]
Type=simple
User=root
Group=root
Environment=LD_LIBRARY_PATH=/opt/rondb/lib:/opt/rondb/lib/mysql
ExecStart=/opt/rondb/bin/mysqld --defaults-file=/etc/rondb/my.cnf --user=root
TimeoutSec=600
Restart=on-failure
RestartSec=5
StartLimitBurst=3
StartLimitInterval=60
LimitNOFILE=infinity
LimitMEMLOCK=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
```

启用服务：
```bash
systemctl daemon-reload
systemctl enable rondb-mysqld.service
systemctl start rondb-mysqld.service
```

# 3 验证方法

## 3.1 集群状态验证

检查 API 节点连接状态：
```bash
/opt/rondb/bin/ndb_mgm --ndb-connectstring=137.33.24.134:1186 -e show
```
预期输出：
```
[mysqld(API)]	22 node(s)
id=105	@137.33.24.134  (RonDB-26.02.4)  ← 已连接
```
验证要点：
- API 节点 105 显示 IP 地址和版本号（而非 "not connected"）
- 其他节点状态保持不变

## 3.2 服务状态验证
检查 systemd 服务：
```bash
systemctl status rondb-mysqld.service
```

预期输出：
```
Active: active (running) since ...
Main PID: xxxxx (mysqld)
```

检查进程：
```bash
ps aux | grep "mysqld --defaults-file=/etc/rondb/my.cnf"
```

## 3.3 MySQL 连接验证
通过 Unix Socket 连接：
```bash
/opt/rondb/bin/mysql -u root -S /var/lib/rondb/mysql/mysql.sock
```

通过 TCP 连接：
```bash
/opt/rondb/bin/mysql -u root -h 137.33.24.134 -P 3306
```

验证命令：
```sql
-- 检查 NDB 引擎
SHOW ENGINES;
-- 预期：ndbcluster 显示为 DEFAULT

-- 检查 NDB 连接
SHOW VARIABLES LIKE 'ndb_connect_string';
-- 预期：137.33.24.134:1186
```

## 3.4 功能验证
创建测试数据库和表：
```sql
CREATE DATABASE test_db;
USE test_db;

-- 创建 NDB 表
CREATE TABLE test_table (
    id INT PRIMARY KEY,
    name VARCHAR(50)
) ENGINE=NDB;

-- 插入数据
INSERT INTO test_table VALUES (1, 'test_data');

-- 查询数据
SELECT * FROM test_table;
```

预期结果：
- 表创建成功
- 数据插入成功
- 查询返回正确结果

验证 NDB 表属性：
```sql
SHOW CREATE TABLE test_table;
-- 预期：ENGINE=NDBcluster
```

## 3.5 日志验证
检查 MySQL 错误日志：
```bash
tail -f /var/lib/rondb/mysql/error.log
```

关键日志信息：
```
[NDB] Binlog: Startup and setup completed
[NDB] Binlog: NDB tables writable
[NDB] Completed metadata synchronization
```

检查系统日志：
```bash
journalctl -u rondb-mysqld.service -f
```

## 3.6 持久化验证
重启服务后验证：
```bash
systemctl restart rondb-mysqld.service
sleep 5

# 重新连接并查询
/opt/rondb/bin/mysql -u root -S /var/lib/rondb/mysql/mysql.sock \
  -e "USE test_db; SELECT * FROM test_table;"
```

预期结果：
- 服务重启成功
- 数据仍然存在
- NDB 连接自动恢复

# 4 完整验证脚本
```bash
#!/bin/bash
# RonDB MySQL API 节点验证脚本

echo "=== 1. 检查集群状态 ==="
/opt/rondb/bin/ndb_mgm --ndb-connectstring=137.33.24.134:1186 -e show | grep "id=105"

echo -e "\n=== 2. 检查服务状态 ==="
systemctl is-active rondb-mysqld.service

echo -e "\n=== 3. 测试 MySQL 连接 ==="
/opt/rondb/bin/mysql -u root -S /var/lib/rondb/mysql/mysql.sock \
  -e "SELECT 'Connection OK' AS status;"

echo -e "\n=== 4. 检查 NDB 引擎 ==="
/opt/rondb/bin/mysql -u root -S /var/lib/rondb/mysql/mysql.sock \
  -e "SHOW ENGINES;" | grep ndbcluster

echo -e "\n=== 5. 测试数据读写 ==="
/opt/rondb/bin/mysql -u root -S /var/lib/rondb/mysql/mysql.sock \
  -e "USE test_db; SELECT * FROM test_table;"

echo -e "\n=== 验证完成 ==="
```

# 5 故障排查命令
如果遇到问题，使用以下命令排查：

```bash
# 1. 检查 MySQL 是否启动
ps aux | grep mysqld

# 2. 查看 MySQL 错误日志
tail -100 /var/lib/rondb/mysql/error.log

# 3. 检查端口监听
netstat -tlnp | grep 3306

# 4. 检查 NDB 连接
/opt/rondb/bin/mysql -u root -S /var/lib/rondb/mysql/mysql.sock \
  -e "SHOW STATUS LIKE 'ndb%';"

# 5. 检查集群状态
/opt/rondb/bin/ndb_mgm --ndb-connectstring=137.33.24.134:1186 -e "show"

# 6. 查看服务日志
journalctl -u rondb-mysqld.service -n 50 --no-pager
```

# 6 当前状态总结
## 6.1 已完成的修复
1. ✅ 创建了 MySQL 配置文件 `/etc/rondb/my.cnf`
2. ✅ 初始化了 MySQL 数据目录 `/var/lib/rondb/mysql`
3. ✅ 创建了 systemd 服务 `rondb-mysqld.service`
4. ✅ 启动了 MySQL API 节点（id=105）
5. ✅ 验证了 NDB 表创建和数据读写功能

## 6.2 当前集群状态
```
数据节点：2/2 运行中 ✓
管理节点：1/1 运行中 ✓
MySQL API：1/22 连接（本机节点 105）✓
```

## 6.3 访问方式
```bash
# 通过 Unix Socket
/opt/rondb/bin/mysql -u root -S /var/lib/rondb/mysql/mysql.sock

# 通过 TCP
/opt/rondb/bin/mysql -u root -h 137.33.24.134 -P 3306
```

# 7 后续建议
## 7.1 其他节点配置

如需在 132 和 133 机器上启动 MySQL API 节点：

1. 复制配置文件 `/etc/rondb/my.cnf` 到对应机器
2. 修改 `ndb-nodeid`：
   - 132 机器：使用 101 或 102
   - 133 机器：使用 103 或 104
3. 修改 `ndb-connectstring` 指向管理节点
4. 初始化数据目录并创建 systemd 服务

## 7.2 安全加固
```sql
-- 设置 root 密码
ALTER USER 'root'@'localhost' IDENTIFIED BY 'your_password';

-- 创建专用用户
CREATE USER 'app_user'@'%' IDENTIFIED BY 'app_password';
GRANT ALL PRIVILEGES ON *.* TO 'app_user'@'%';
FLUSH PRIVILEGES;
```

## 7.3 监控建议
- 配置 Prometheus + Grafana 监控 MySQL 指标
- 监控 NDB 集群状态（使用 ndb_mgm）
- 设置告警：API 节点断开连接时通知

## 7.4 备份策略
- 定期使用 `ndb_mgm -e "START BACKUP"` 进行集群备份
- 配置 MySQL 逻辑备份（mysqldump）
- 测试恢复流程


# 8 配置文件清单

| 文件路径 | 用途 |
|---------|------|
| `/etc/rondb/config.ini` | NDB 集群配置（管理节点、数据节点、API 节点定义） |
| `/etc/rondb/my.cnf` | MySQL 服务器配置（API 节点 105） |
| `/etc/systemd/system/rondb-mgmd.service` | 管理节点 systemd 服务 |
| `/etc/systemd/system/rondb-mysqld.service` | MySQL 服务器 systemd 服务 |
| `/var/lib/rondb/mgm/` | 管理节点数据目录 |
| `/var/lib/rondb/mysql/` | MySQL 服务器数据目录 |
| `/var/lib/rondb/mysql/error.log` | MySQL 错误日志 |

