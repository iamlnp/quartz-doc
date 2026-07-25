---
写作年份: 2024
tags:
  - leveldb
  - 技术学习
imageNameKey: leveldb梳理
状态:
  - 取消
---

# 1 缓存系统
缓存对于一个数据库读性能的影响十分巨大，倘若leveldb的每一次读取都会发生一次磁盘的IO，那么其整体效率将会非常低下。

Leveldb中使用了一种基于LRUCache的缓存机制，用于缓存：

- 已打开的sstable文件对象和相关元数据；
- sstable中的dataBlock的内容；

使得在发生读取热数据时，尽量在cache中命中，避免IO读取。
## 1.1 Cache结构

leveldb中使用的cache是一种LRUcache，其结构由两部分内容组成：

- Hash table：用来存储数据；
- LRU：用来维护数据项的新旧信息；

![[assets/IMG-2025-07-16-11.png|475]]
其中Hash table是基于Yujie Liu等人的论文《Dynamic-Sized Nonblocking Hash Table》实现的，用来存储数据。由于hash表一般需要保证插入、删除、查找等操作的时间复杂度为 O(1)。

当hash表的数据量增大时，为了保证这些操作仍然保有较为理想的操作效率，需要对hash表进行resize，即改变hash表中bucket的个数，对所有的数据进行重散列。

基于该文章实现的hash table可以实现resize的过程中**不阻塞其他并发的读写请求**。

LRU中则根据Least Recently Used原则进行数据新旧信息的维护，当整个cache中存储的数据容量达到上限时，便会根据LRU算法自动删除最旧的数据，使得整个cache的存储容量保持一个常量。

## 1.2 Dynamic-sized NonBlocking Hash table
# 2 参考
https://leveldb-handbook.readthedocs.io/zh/latest/basic.html