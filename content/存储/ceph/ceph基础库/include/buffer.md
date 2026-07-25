`ceph::buffer`是ceph非常底层的实现，负责管理ceph的内存。`ceph::buffer`主要包含`buffer::list`、`buffer::ptr`、`buffer::hash`，这三个类都定义在`src/include/buffer.h`和`src/common/buffer.cc`中。

文件路径： `include/buffer_fwd.h`中定义bufferlist： 
```c++
using bufferptr = buffer::ptr;
using bufferlist = buffer::list;
using bufferhash = buffer::hash;
```

`buffer::raw`：负责维护物理内存的引用计数nref和释放操作。  
`buffer::ptr`：指向buffer::raw的指针。  
`buffer::list`：表示一个ptr的列表（`std::list<bufferptr>`），相当于将N个ptr构成一个更大的虚拟的连续内存。
`buffer::hash`：一个或多个bufferlist的有效哈希。

# 1 buffer::raw
`bufferlist`基于bufferptr和buffer::raw实现的，因此优先分析buffer::raw。
`buffer::raw`位于文件`include/buffer_raw.h`中，
