---
写作年份:
  - "2025"
imageNameKey: Kubernetes CSI Driver
tags:
  - 容器
  - kubernets
---

# 1 简介 
- CSI是一个为同期编排系统暴露任意的块存储或者文件存储系统接口的标准
- 第三方可以以插件的形式为K8s提供新的存储系统支持，而不用修改K8s的核心代码
- 详细定义：https://github.com/container-storage-interface/spec/blob/master/spec.md 

# 2 架构
![[assets/2025-07-16-Kubernetes CSI Driver-IMG.png|500]]