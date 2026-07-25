文件路径：`include/mempool.h`，位于mempool命名空间

MEMPOOL_CLASS_HELPERS
MEMPOOL_DEFINE_OBJECT_FACTORY(Tfs_bm_info, co_bm_info, mds_co);

# 1 简介

内存池是一种计算一套容器内存消耗的方法，内存池是静态声明的，详见pool_index_t，每个内存池跟踪其包含的字节数和项数。

可以声明分配器，并将其与一种类型关联，独立于总池之外独立追踪这种类型。



