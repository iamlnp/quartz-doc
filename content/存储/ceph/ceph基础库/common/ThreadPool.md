---
写作年份: 2023
tags: ceph基础库
---
# 代码路径

所在代码位置：`src/common/WorkQueue.cc`

# 线程池功能描述

## 线程池主要成员

```c++
class ThreadPool : public md_config_obs_t
{ 
    unsigned _num_threads;						     //构造线程池时指定的线程数量
    std::vector<WorkQueue_*> work_queues;  //待处理队列
    std::set<WorkThread*> _threads;  			 //线程对象集合
    std::list<WorkThread*> _old_threads;   //待join的线程集合
}
```

## 线程池函数逻辑

- start()

```c++
void ThreadPool::start_threads()
{
  ceph_assert(ceph_mutex_is_locked(_lock));
  while (_threads.size() < _num_threads) {  //1. 线程数量到达配置线程数量之前，不断创建新线程对象
    WorkThread *wt = new WorkThread(this); 
    ldout(cct, 10) << "start_threads creating and starting " << wt << dendl;
    _threads.insert(wt);										//2. 放入到线程池集合中

    wt->create(thread_name.c_str());			  //3. 创建pthread线程，并不断运行 WorkThread->pool->worker()
    																			  //   _threads中的线程从work_queues中竞争待处理队列
  }
}
```

- worker()

> 线程池核心运行函数

```c++
void ThreadPool::worker(WorkThread *wt)
{
  std::unique_lock ul(_lock);
  while (!_stop) {

    // manage dynamic thread pool
    join_old_threads();										    // 1. join _old_threads集合中的线程
    if (_threads.size() > _num_threads) {
      _threads.erase(wt);
      _old_threads.push_back(wt);
      break;
    }

    if (!_pause && !work_queues.empty()) {
      WorkQueue_* wq;
      int tries = 2 * work_queues.size();
      bool did = false;
      while (tries--) {
        next_work_queue %= work_queues.size();
        wq = work_queues[next_work_queue++];  // 2. 某个线程从work_queues中竞争到一个wq
        
        void *item = wq->_void_dequeue();     // 3. 从当前work_queue中取出一个item处理
        if (item) {
          processing++;
          ldout(cct,12) << "worker wq " << wq->name << " start processing " << item
            << " (" << processing << " active)" << dendl;
          ul.unlock();
					...
          wq->_void_process(item, tp_handle); // 4. 调用队列处理函数_void_process，放锁后执行，因此存在并发
          ul.lock();
          wq->_void_process_finish(item);     // 5. 调用队列处理finish函数_void_process_finish，不存在并发
          processing--;
          ldout(cct,15) << "worker wq " << wq->name << " done processing " << item
            << " (" << processing << " active)" << dendl;
          if (_pause || _draining)
            _wait_cond.notify_all();				  // 6. 唤醒等待的线程
          did = true;
          break;
        }
      }
      if (did)
        continue;
    }
  }
}
```

- pause(): 暂停线程池处理
- unpause()： 继续线程池处理
- drain():  排空work_queue
- stop(): join所有线程池中线程， 清空work_queue_队列（不执行，直接放弃）



## 队列结构

![[【01】技术系统/【01】通用技术/存储/ceph/ceph基础库/common/assets/IMG-2025-02-11-14.png]]

### 虚基类WorkQueue_

```c++
struct WorkQueue_ {
    std::string name;
    time_t timeout_interval, suicide_interval;
    WorkQueue_(std::string n, time_t ti, time_t sti)
      : name(std::move(n)), timeout_interval(ti), suicide_interval(sti)
    { }
    virtual ~WorkQueue_() {}
    /// Remove all work items from the queue.
    virtual void _clear() = 0;
    /// Check whether there is anything to do.
    virtual bool _empty() = 0;
    /// Get the next work item to process.
    virtual void *_void_dequeue() = 0;   //取出待处理的items
    /** @brief Process the work item.
     * This function will be called several times in parallel
     * and must therefore be thread-safe. */
    virtual void _void_process(void *item, TPHandle &handle) = 0;
    /** @brief Synchronously finish processing a work item.
     * This function is called after _void_process with the global thread pool lock held,
     * so at most one copy will execute simultaneously for a given thread pool.
     * It can be used for non-thread-safe finalization. */
    virtual void _void_process_finish(void *) = 0;
  };
```

### 四类work_queue

- BatchWorkQueue
  - 每次可以取出多个待处理item
  - 该WorkQueued的item存放容器需要自行定义
  - 需要自行实现如下接口（主要函数）：
    - `virtual void _dequeue(std::list<T*> *) = 0`: 如何从队列中work_queue中拿出items
    - `virtual bool _enqueue(T *) = 0`: 入队列接口
    - `virtual void _process(const std::list<T*> &items, TPHandle &handle) = 0`: 批处理接口
- WorkQueueVal
  - 适用于处理原始值类型或者小对象
  - 将T类型item的值存储队列
  - 存储T类型值的容器需要自行实现
  - 处理缓存容器已经实现，用于存在中间值：
    - std::list<U> to_process;  //待处理list， 从放入_void_dequeue()拿出的元素U，每次存入一个
    - std::list<U> to_finish;      //_void_process_finish会处理该list中的元素，每次处理一个
  - 需要自行实现如下接口：
    - `bool _empty() override = 0`:  判断容器非空
    - `virtual void _enqueue(T) = 0;`: 入队列接口
    - `virtual void _process(U u, TPHandle &) = 0;`: 处理元素U的函数
- WorkQueue
  - 适用于处理大对象或者动态申请内存的对象
  - 存储容器需要自行实现
  - 需要自行实现如下接口：
    - `virtual bool _enqueue(T *) = 0;`: 入workqueue接口
    - `virtual T *_dequeue() = 0;`： 取work_queue item接口
    - `virtual void _process(T *t, TPHandle &) = 0;` : item处理接口
- PointerWQ
  - 适用于处理大对象或者动态申请内存的对象，比WorkQueue更加方便，但是没有WorkQueue抽象
  - 存储容器已经实现：`std::list<T *> m_items`
  - 只需要实现`virtual void process(T *item) = 0;`, 用于item处理


### 可直接使用的两种实现

- **class GenContextWQ **: public ThreadPool::WorkQueueVal<GenContext<ThreadPool::TPHandle&>*>

- **class ContextWQ** : public ThreadPool::PointerWQ<Context>



# 应用举例

## 创建队列结构


```c++
class Rep_WQ : public ThreadPool::PointerWQ<Context>
{
    public:
        Rep_WQ(const std::string &name, time_t ti, ThreadPool *tp)
            : ThreadPool::PointerWQ<Context>(name, ti, 0, tp) {
                this->register_work_queue();
            }
        void process(Context *fin) override;
};

void Rep_WQ::process(Context *fin)
{
    fin->complete(0);
}
```



## 启动线程池

```c++
1. 创建ThreadPool
thread_pool = new ThreadPool(cct, "tfs_rep_thread_pool", "rep_daemon_tp", g_conf()->rep_thread_pool_nr);
thread_pool->start();

2. 创建队列
work_queue = new Rep_WQ("tfs_rep_daemon", 60, thread_pool);
```



## 投递任务

```c++
Context *ctx = new Test_TheadPool();
work_queue->queue(ctx);
```



## 销毁线程池

```c++
work_queue->drain();
delete work_queue;
work_queue = nullptr;

thread_pool->stop();
delete thread_pool;
thread_pool = nullptr;
```

