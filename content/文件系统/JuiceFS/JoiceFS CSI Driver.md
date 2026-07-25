---
写作年份:
  - "2025"
imageNameKey: JoiceFS CSI Driver
tags:
  - juicefs
  - 容器
---
# 1 简介
- juicefs csi driver为k8s提供[[Kubernetes CSI Driver|CSI]]插件
-  支持多读多写(ReadWriteMany)和只读(ReadOnlyMany)两种模式
- 部分特性：
    - 静态配置
    - 动态配置
    - 子目录挂载
    - 自动故障恢复


# 2 参考
1. https://juicefs.com/docs/zh/csi/introduction