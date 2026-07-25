---
状态:
  - 完成
年份:
  - "2025"
tags:
  - 技术学习
  - systemd
---
# 1 unit
配置文件路径：/lib/systemd/system/
## 1.1 service类型
可以**man  systemd.service**参考帮助
- **service**： 以.service结尾
```bash
[root@ceph-221 system]# cat tfs-mds@.service
[Unit]
Description=Tfs metadata server daemon
After=network-online.target local-fs.target time-sync.target
Wants=network-online.target local-fs.target time-sync.target
PartOf=tfs-mds.target

[Service]
LimitNOFILE=1048576
LimitNPROC=1048576
EnvironmentFile=-/etc/sysconfig/ceph
Environment=CLUSTER=ceph
ExecStart=/usr/bin/tfs-mds -f --cluster ${CLUSTER} --id %i --setuser ceph --setgroup ceph
ExecReload=/bin/kill -HUP $MAINPID
LockPersonality=true
MemoryDenyWriteExecute=true
NoNewPrivileges=true
PrivateDevices=yes
ProtectControlGroups=true
ProtectHome=true
ProtectKernelModules=true
ProtectKernelTunables=true
ProtectSystem=full
PrivateTmp=true
TasksMax=infinity
Restart=on-failure
RestartForceExitStatus=SIGHUP
StartLimitInterval=30min
StartLimitBurst=10
RestartSec=15

[Install]
WantedBy=tfs-mds.target
```
### 1.1.1 Unit节
1. **RefuseManualStop**=yes： disable the manual start, stop, and restart functions of a service. 

### 1.1.2 Service节
-  **Restart**=on-abort：Configures whether the service shall be restarted when the service process exits
![[IMG-2025-02-18-10.png|600]]
- **KillSignal**=SIGWINCH：changes the default kill action from SIGTERM to SIGWINCH.
- **Type**=oneshot：**oneshot** causes the service to just run as a normal script that performs some specified one-time task, instead of as a continuously running daemon. After the specified commands have run, the service shuts down.
### 1.1.3 Install节




## 1.2 socket
- **socket**：以.socket结尾， instead of having a server daemon run full-time, even when it isn't needed, we can leave it shut down most of the time, and only start it when the system detects an incoming network request for it
	- the default behavior for any socket file is to get its information from a service file that has the same prefix in the filename
```bash
[root@ceph-221 system]# cat sshd.socket
[Unit]
Description=OpenSSH Server Socket
Documentation=man:sshd(8) man:sshd_config(5)
Conflicts=sshd.service # This tells systemd to not allow the Secure Shell service to run normally if this socket is enabled. If you were to enable this socket, the normal SSH service would get shut down.

[Socket]
ListenStream=22
Accept=yes

[Install]
WantedBy=sockets.target
```
## 1.3 path
**path**：You can use a path unit to have systemd monitor a certain file or directory to see when it changes
```bash
# Common Unix Printing System（CUPS）是一个开源的打印系统
[root@ceph-221 system]# cat /lib/systemd/system/cups.path
[Unit]
Description=CUPS Scheduler
PartOf=cups.service

[Path]
PathExists=/var/cache/cups/org.cups.cupsd   # systemd to monitor a specific file for changes, which in this case is the /var/cache/cups/org.cups.cupsd file. If systemd detects any changes to this file, it will activate the printing service

[Install]
WantedBy=multi-user.target
```

# 2 target
In **systemd**, a target is a unit that groups together other **systemd** units for a particular purpose. The units that a target can group together include services, paths, mount points, sockets, and even other targets.

## 2.1 target文件的结构
举例：
```bash
[root@ceph-221 src]# systemctl cat sockets.target
[Unit]
Description=Sockets
Documentation=man:systemd.special(7)
```

this target is just a group of all the sockets that we need to have running：
```bash
[root@ceph-221 src]# ll /etc/systemd/system/sockets.target.wants
总用量 0
lrwxrwxrwx. 1 root root 43 8月  17 2021 avahi-daemon.socket -> /usr/lib/systemd/system/avahi-daemon.socket
lrwxrwxrwx. 1 root root 35 8月  17 2021 cups.socket -> /usr/lib/systemd/system/cups.socket
lrwxrwxrwx. 1 root root 39 8月  17 2021 dm-event.socket -> /usr/lib/systemd/system/dm-event.socket
lrwxrwxrwx. 1 root root 37 8月  17 2021 iscsid.socket -> /usr/lib/systemd/system/iscsid.socket
lrwxrwxrwx. 1 root root 39 8月  17 2021 iscsiuio.socket -> /usr/lib/systemd/system/iscsiuio.socket
lrwxrwxrwx. 1 root root 41 8月  17 2021 multipathd.socket -> /usr/lib/systemd/system/multipathd.socket
lrwxrwxrwx. 1 root root 38 1月  10 2022 rpcbind.socket -> /usr/lib/systemd/system/rpcbind.socket
lrwxrwxrwx. 1 root root 39 8月  17 2021 sssd-kcm.socket -> /usr/lib/systemd/system/sssd-kcm.socket
lrwxrwxrwx. 1 root root 40 8月  17 2021 virtlockd.socket -> /usr/lib/systemd/system/virtlockd.socket
lrwxrwxrwx. 1 root root 39 8月  17 2021 virtlogd.socket -> /usr/lib/systemd/system/virtlogd.socket
```
## 2.2 设置默认启动target
- **systemctl get-default**：show which target is set as the default  或者也可以查看 **/etc/systemd/system/default.target**
```bash
[root@ceph-221 nfs]# systemctl get-default
graphical.target
[root@ceph-221 nfs]# ll /etc/systemd/system/default.target
lrwxrwxrwx. 1 root root 36 8月  17 2021 /etc/systemd/system/default.target -> /lib/systemd/system/graphical.target
[root@ceph-221 nfs]#
```
- **systemctl set-default [target_name]**: 设置默认启动模式： `systemctl set-default multi-user`

### 2.2.1 临时改变target
- systemctl isolate multi-user： 切换至命令行模式
- systemctl isolate graphical： 切换至图形模式

## 2.3 查询命令
- **systemctl list-units -t target**：show all of the active targets on your system， Add the **--inactive** option to see the inactive targets。
- **systemctl list-dependencies [xxx.target]**: list dependencies of xxx.target， The **--after** and **--before** options show the dependencies that must start either _before_ or _after_ a target starts. (No, I didn't do that backward. The **--after** option indicates that the target must start after the listed dependencies, and the **--before** option indicates that the target must start before the listed dependencies.)
```bash
[root@ceph-221 nfs]# systemctl list-dependencies graphical.target
graphical.target
● ├─accounts-daemon.service
● ├─gdm.service
● ├─rtkit-daemon.service
● ├─systemd-update-utmp-runlevel.service
● ├─udisks2.service
● └─multi-user.target
●   ├─atd.service
●   ├─auditd.service
●   ├─auditlogd.service
●   ├─avahi-daemon.service
●   ├─chronyd.service
●   ├─crond.service
●   ├─cups.path
●   ├─cups.service
```

# 3 服务管理
## 3.1 Enabling and disabling services
- **`systemctl enable [servie_name]`**

```bash
[donnie@localhost ~]$ sudo systemctl enable httpd
			Created symlink /etc/systemd/system/multi-user.target.wants/httpd.service → /usr/lib/systemd/system/httpd.service.”

# When we enable the Apache service, we create a symbolic link in the /etc/systemd/system/multi-user.target.wants/ directory that points back to the httpd.service file
```
To determine where a symbolic link should be created, the systemctl enable command pulls in the setting from the [Install] section of the service file
```bash
. . .
. . .
[Install]
	WantedBy=multi-user.target”
```

Be aware that when you enable a service that isn't already running, the service doesn't automatically start until you reboot the machine
- **`systemctl enable --now [service_name]`**
You can issue a separate start command to start the service, or you can use the enable --now option to enable and start the service with just a single command

- **`systemctl disable [service_name]`**
When you disable a unit, the symbolic link for it gets removed. 
If the service is running, it will remain running after you issue the disable command. You can issue a separate stop command or use the disable --now option to disable and stop the service at the same time

## 3.2 mask a service
you have a service that you never want to start, either manually or automatically. You can accomplish this by masking the service， if you change your mind, just use the unmask option.

```bash
[root@ceph-221 ~]# systemctl mask tfs-mds@a
Created symlink /etc/systemd/system/tfs-mds@a.service → /dev/null.
[root@ceph-221 ~]# systemctl unmask tfs-mds@a
Removed /etc/systemd/system/tfs-mds@a.service.
```

# 4 systemd Timers
first have to create a **systemd** service, then call that service from the timer.

# shutdown和reboot
- **systemctl poweroff**：关机
- **systemctl halt**：挂起机器
- **systemctl reboot**：重启机器
# 5 依赖关系
Requires：
Wants：
After：
Before：
