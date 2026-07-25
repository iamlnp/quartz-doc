官网： https://min.io/
文档： https://www.minio.org.cn/docs/minio/linux/index.html

# 1 下载rpm并安装
```bash
wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio-20241218131544.0.0-1.x86_64.rpm -O minio.rpm
sudo dnf install minio.rpm
```

# 2 启动minio服务器
```bash
mkdir ~/minio
minio server ~/minio --console-address :9090
```
启动时如下打印，可以根据提示登录对应的web页面
```bash
[root@ceph-221 tfs]# minio server /mnt/tfs/minio --console-address :9090
INFO: Formatting 1st pool, 1 set(s), 1 drives per set.
INFO: WARNING: Host local has more than 0 drives of set. A host failure will result in data becoming unavailable.
INFO:
┏━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┓
┃ You are running an older version of MinIO released 2 months before the latest release ┃
┃ Update: Run `mc admin update ALIAS`                                                   ┃
┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┛

MinIO Object Storage Server
Copyright: 2015-2025 MinIO, Inc.
License: GNU AGPLv3 - https://www.gnu.org/licenses/agpl-3.0.html
Version: RELEASE.2024-12-18T13-15-44Z (go1.23.4 linux/amd64)

API: http://10.131.1.221:9000  http://10.10.10.152:9000  http://192.168.122.1:9000  http://127.0.0.1:9000
   RootUser: minioadmin
   RootPass: minioadmin

WebUI: http://10.131.1.221:9090 http://10.10.10.152:9090 http://192.168.122.1:9090 http://127.0.0.1:9090
   RootUser: minioadmin
   RootPass: minioadmin

CLI: https://min.io/docs/minio/linux/reference/minio-mc.html#quickstart
   $ mc alias set 'myminio' 'http://10.131.1.221:9000' 'minioadmin' 'minioadmin'

Docs: https://docs.min.io
WARN: Detected default credentials 'minioadmin:minioadmin', we recommend that you change these values with 'MINIO_ROOT_USER' and 'MINIO_ROOT_PASSWORD' environment variables
```
# 3 按照MinIO客户端
下载 `mc`客户端安装，并通过命令行添加至系统 `PATH` 中，你就可以随时使用MinIO客户端进行管理了
```bash
wget https://dl.minio.org.cn/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/mc
```

使用 ``mc alias set`创建一个新的别名并将其关联到您的本地部署。 你可以运行 :mc-cmd:`mc`` 命令去管理指定别名的MinIO服务器:
```bash
mc alias set local http://10.10.10.22:12000 minioadmin minioadmin
mc admin info local

或者：
mc alias set local http://10.131.1.221:9000 minioadmin minioadmin
mc admin info local

```

输出
```bash
[root@ceph-221 ~]# mc alias set local http://10.131.1.221:9000 minioadmin minioadmin
mc: Configuration written to `/root/.mc/config.json`. Please update your access credentials.
mc: Successfully created `/root/.mc/share`.
mc: Initialized share uploads `/root/.mc/share/uploads.json` file.
mc: Initialized share downloads `/root/.mc/share/downloads.json` file.
Added `local` successfully.
[root@ceph-221 ~]# mc admin info local
●  10.131.1.221:9000
   Uptime: 3 minutes
   Version: 2024-12-18T13:15:44Z
   Network: 1/1 OK
   Drives: 1/1 OK
   Pool: 1

┌──────┬───────────────────────┬─────────────────────┬──────────────┐
│ Pool │ Drives Usage          │ Erasure stripe size │ Erasure sets │
│ 1st  │ 0.0% (total: 474 GiB) │ 1                   │ 1            │
└──────┴───────────────────────┴─────────────────────┴──────────────┘

0 B Used, 3 Buckets, 0 Objects
1 drive online, 0 drives offline, EC:0
```


# 4 数据格式
```bash
[root@ceph-221 1]# ll /mnt/nfs/minio/bucket1/2/2/3/4/5/1/1
总用量 4
-rw-r--r-- 1 root root 3692 2月  21 09:00 xl.meta
```

# 5 命令行
- mc rm --recursive local/bucket1 --force： 递归删除桶内元素

# 6 报错
1. 端口被占用
```bash
[root@node2 nfs]# minio server ~/minio --console-address :9090
FATAL Unable to start the server: Specified port is already in use
      > Please ensure no other program uses the same address/port
```