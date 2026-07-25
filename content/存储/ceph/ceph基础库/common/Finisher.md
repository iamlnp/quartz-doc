
# 1 Finisher类图

![](IMG-2025-02-11-14.drawio)

# 2 Finisher描述

Finisher的处理模型是生产者-消费者模型，内部只有一个处理队列，使用Finisher最主要的作用异步+保序，其模型会严格按照投递顺序调用回调函数，核心处理逻辑在于函数：

- queue
- finisher_thread_entry

1. queue接受待回调事件，尝试唤醒Finisher中finisher_thread_entry；
2.  finisher_thread_entry中检查到finisher_queue非空，则将finisher_queue swap到in_progress_queue中，然后释放锁，这样做queue流程可以继续接收context回调事件，减少锁竞争。
3. finisher_thread_entry遍历in_progress_queue，依次调用item的context中回调函数处理。
4. 如果finisher_queue非空则继续循环处理，否则wait等待重新唤醒

# 3 使用示例

1.  构造Context:  
```Context *onsafe = new C_Flush(this, flush_pos, now);```

2. 调用异步接口：

```c++
filer->write(ino, &layout, snapc,
      flush_pos, len, write_bl, ceph::real_clock::now(),
      0,
      wrap_finisher(onsafe), write_iohint);
```

3. 在write异步接口的wrap_finisher回调中，对之前的onsafe投递到Finisher队列中处理：
```c++
  void finish(int r) override {
    fin->queue(con, r);
    con = nullptr;
  }
```

#  附件

[Finisher线程visio图](C:\技术梳理\轩辕整理\ceph基础库\Finisher线程.vsdx)

