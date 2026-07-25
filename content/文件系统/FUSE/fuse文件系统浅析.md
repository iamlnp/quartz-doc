---
写作年份:
  - "2025"
imageNameKey: fuse文件系统浅析
tags:
  - fuse
  - 文件系统
---
# 1 整体设计
- 一种用户态文件系统架构
- 组成：
    1. 内核模块-fuse.ko
    2. 用户库-libfuse.*
    3. 挂载工具-fusermount
    4. 用户态守护进程-fuse daemon

![[assets/2025-09-02-fuse文件系统浅析-IMG.png]]

# 2 基本原理
1. fuse driver：加载时注册fuse文件系统到vfs，作为fuse daemon实现的特定文件系统代理
2. /dev/fuse: fuse driver还会注册一个/dev/fuse的misc或者block设备，作为daemon和kernel的接口
3. fuse queue：请求队列是daemon和kernel通信的数据结构，优先级由高到底共有5个
    - interrupt
    - forgets
    - processing
    - pending
    - background
![[assets/2025-09-02-fuse文件系统浅析-IMG-1.png|400]]

## fuse queue
1. 某个request在任何时刻只会属于一个queue
2. fuse daemon读/dev/fuse时，request传送给daemon的机制：
    - 