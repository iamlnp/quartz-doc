开源库： https://github.com/deepseek-ai/3FS

https://github.com/deepseek-ai/3FS/blob/main/README.md
https://arxiv.org/html/2408.14158v1#S6
https://mp.weixin.qq.com/s/X60PsEPeFsb-ZPKATMrWrA
https://zhuanlan.zhihu.com/p/512508588
https://mp.weixin.qq.com/s/YuDrT5Fn2kYnW-taC6OWOw

# 1 调研列表
1. [[3FS Meta Service调研报告]]

client
FoundationDB
mgmtd - 集群管理
Storage Service
FFRecord

# 2 进展

## 2.1 2025-3-19
志敏搭建完毕环境： 10.131.9.10
![[IMG-2025-03-19-11.png]]

## 2.2 2025-3-12
调研Meta Service

## 2.3 2025-3-11
![[IMG-2025-03-12-19.png]]
lenovo/Lenovo@123

## 2.4 2025-3-10
郝志敏：  
1. 主要负责研究hf3fuse & USRBio，重点是USRBio的实现方式以及是否3FS有工具直接测试该接口的性能，并看相关技术是否能在TDS中引入；
2. 联想研究院现有一套机群，了解相关安装流程，以便在后期借到新的机器时进行安装。
刘乃朋：
1. 研究3FS mds相关实现，重点是foundationDB，并对比4.X mds各方面的优劣。
孙艳强：
1. 研究3FS IO server相关技术，重点是CRAQ，数据条带的布局，加减节点数据如何负载在平衡，并对比TDS的副本实现。
韦新伟：
1. 研究FFRecord在3FS中的整个使用流程，3FS对其如何进行优化，4.X是否可以做相似的优化。