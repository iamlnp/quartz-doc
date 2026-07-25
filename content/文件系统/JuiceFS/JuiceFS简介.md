---
状态:
  - 取消
年份:
  - "2025"
imageNameKey: JoiceFS简介
tags:
  - 技术学习
---

# 1 参考
- [JuiceFS 简介](https://juicefs.com/docs/zh/community/introduction/)
- [Gitlab源码](https://github.com/juicedata/juicefs)

# 2 配额

![[2026-02-04-JoiceFS简介-IMG.png|750]]

![[2026-02-04-JoiceFS简介-IMG-1.png|750]]

![[2026-02-04-JoiceFS简介-IMG-2.png|750]]

![[2026-02-04-JoiceFS简介-IMG-3.png|750]]

![[2026-02-04-JoiceFS简介-IMG-4.png|750]]

![[2026-02-04-JoiceFS简介-IMG-5.png|750]]

![[2026-02-04-JoiceFS简介-IMG-6.png|750]]
![[2026-02-04-JoiceFS简介-IMG-7.png|750]]

Q&A：
1. 异常场景下client挂掉了，缓存的quota的处理会损失掉，需要主动触发修复任务
2. 目录里创建硬链接时，子目录配额会增加inode和空间用量
3. 多客户端配额有写超出的情况
4. 配额会统计快照中clone的内容

# 3 随手记
