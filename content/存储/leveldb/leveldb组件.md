---
写作年份:
  - "2025"
imageNameKey: leveldb内存组件
tags:
  - leveldb
  - 数据库
---
# 1 内存组件
Area类简要说明：
![[assets/IMG-2025-07-10-09.png|600]]
基本原理图：
![[assets/IMG-2025-07-10-09-1.png|500]]


# 2 各种key
1. user key： 用户输入数据的key（slice格式）
2. InternalKey: 内部key，常用来key比较等场景，std::string rep_
3. ParsedInternalKey：对InternalKey的解析，因为internalKey是一个字符串，格式：
    - Slice user key
    - SequenceNumber Sequence
    - ValueType type
4. memtable key： 存储在memtable中的key，这个key比较特殊，他是包含value的
5. lookup key：用于DBimpl::Get中，成员变量如下：
    - char space_[200]
    - const char* start_
    - const char* kstart_
    - const char* end_


key的关系图：
![[assets/2025-07-10-leveldb内存组件-IMG.png|400]]

# 各类compare

![[assets/2025-07-10-leveldb内存组件-IMG-1.png|600]]

# WriteBatch
## 数据结构

![[assets/2025-07-10-leveldb内存组件-IMG-3.png|800]]

