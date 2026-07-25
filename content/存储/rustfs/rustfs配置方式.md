---
写作年份:
  - "2025"
imageNameKey: rustfs建议配置和概述
tags:
  - rustfs
  - 存储调研
  - 对象存储
---
安装方式： https://github.com/rustfs/rustfs/blob/main/README_ZH.md

安装官方说明，一般有四种启动方法：
1. 一键脚本快速启动
2. Docker快速启动
3. 从源码构建
4. 使用Helm Chart部署

控制台可以通过浏览器导航到`http://localhost:9000`，默认用户名和密码是rustfsadmin，其中9000是默认的端口号

下面主要简要说明一键脚本快读启动的方式
# 1 一键脚本快速启动
```bash
curl -O  https://rustfs.com/install_rustfs.sh && bash install_rustfs.sh
```

脚本有三个主要功能：
1. 安装rustfs: `install_rustfs`
2. 卸载rustfs: `uninstall_rustfs`
3. 升级rustfs: `upgrade_rustfss`

下面主要简要介绍如何安装rustfs
## 1.1 安装rustfs
对应的处理 `install_rustfs`
1. 用户设置端口以及端口有效性检查
2. 用户设置console端口以及端口有效性检查
3. 用户设置数据目录
4. 下载所需的二进制文件并安装： `download_and_install_binary`
5. 设置systemd service问价
6. 设置rustfs配置文件
7. 通过systemd启动rustfs

# 2 默认全局配置
1. rustfs service文件： `/usr/lib/systemd/system/rustfs.service`
2. 默认配置文件(RUSTFS_CONFIG_FILE)： `/etc/default/rustfs`
```bash
RUSTFS_ACCESS_KEY=rustfsadmin
RUSTFS_SECRET_KEY=rustfsadmin
RUSTFS_VOLUMES="$RUSTFS_VOLUME" #卷路径
RUSTFS_ADDRESS=":$RUSTFS_PORT"  #服务端口号
RUSTFS_CONSOLE_ADDRESS=":$CONSOLE_PORT" #console端口号
RUSTFS_CONSOLE_ENABLE=true
RUST_LOG=warn
RUSTFS_OBS_LOG_DIRECTORY="$LOG_DIR/" #日志文件路径
```
3. 默认二进制文件路径(RUSTFS_BIN_PATH)：`/usr/local/bin/rustfs`
4. 默认日志路径（LOG_DIR）： `/var/logs/rustfs`
5. 默认端口号: 9000
6. 默认console端口号： 9001
7. 默认systemd service配置文件内容
```bash
[Unit]
Description=RustFS Object Storage Server
Documentation=https://rustfs.com/docs/
After=network-online.target
Wants=network-online.target
[Service]
Type=notify
NotifyAccess=main
User=root
Group=root
WorkingDirectory=/usr/local
EnvironmentFile=-$RUSTFS_CONFIG_FILE
ExecStart=$RUSTFS_BIN_PATH  \$RUSTFS_VOLUMES #RUSTFS_VOLUMES为存储数据的路径，一般是某个目录
LimitNOFILE=1048576
LimitNPROC=32768
TasksMax=infinity
Restart=always
RestartSec=10s
OOMScoreAdjust=-1000
SendSIGKILL=no
TimeoutStartSec=30s
TimeoutStopSec=30s
NoNewPrivileges=true
ProtectHome=true
PrivateTmp=true
PrivateDevices=true
ProtectClock=true
ProtectKernelTunables=true
ProtectKernelModules=true
ProtectControlGroups=true
RestrictSUIDSGID=true
RestrictRealtime=true
StandardOutput=append:$LOG_DIR/rustfs.log
StandardError=append:$LOG_DIR/rustfs-err.log
[Install]
WantedBy=multi-user.target
```
8. systemd启动rustfs的方式
```bash
systemctl daemon-reload || err "systemctl daemon-reload failed."
systemctl enable rustfs || err "systemctl enable rustfs failed."
systemctl start rustfs || err "systemctl start rustfs failed."
```


# 参考
1. https://docs.rustfs.com/installation/linux/single-node-single-disk.html
