# fuse简介

FUSE（用户空间文件系统）是这样一个框架：它使得FUSE用户在用户态下编写文件系统成为可能，而不必和内核打交道。

FUSE由三个部分组成：linux内核模块、FUSE库 以及mount 工具。

用户关心的只是FUSE库和mount工具，内核模块仅仅提供kernel的接入口，给了文件系统一个框架，而文件系统本身的主要实现代示位于用户空间中。FUSE库给用户提供了编程的接口，而mount工具则用于挂在用户编写的文件系统。

# 源代码目录

1. /doc 包含FUSE相关文档
2. /include 包含了FUSE API头，对创建文件系统有用，主要用fuse.h
3. /lib 存放FUSE库
4. /util 包含了FUSE工具库
5. /example 参考的例子
