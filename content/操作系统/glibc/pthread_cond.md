---
写作年份:
  - "2025"
tags:
  - glibc
  - pthread
---

# 1 pthread支持操作一览图
```text
pthread 操作
├── 线程创建与结束
│   ├── pthread_create        : 创建新线程
│   ├── pthread_exit          : 线程结束，返回退出状态
│   └── pthread_cancel        : 请求取消指定线程
│
├── 线程属性
│   ├── pthread_attr_init     : 初始化线程属性
│   ├── pthread_attr_destroy  : 销毁线程属性
│   ├── pthread_attr_setdetachstate : 设置线程是否可分离（detached 或 joinable）
│   └── pthread_attr_getdetachstate : 获取线程的分离状态
│
├── 线程同步与等待
│   ├── pthread_join          : 等待线程结束，获取返回值
│   ├── pthread_detach        : 分离线程，使其结束时自动回收资源
│   └── pthread_barrier_wait  : 阻塞线程直到所有线程到达同步点
│
├── 互斥锁（用于线程同步）
│   ├── pthread_mutex_init    : 初始化互斥锁
│   ├── pthread_mutex_lock    : 锁定互斥锁，防止其他线程进入临界区
│   ├── pthread_mutex_trylock : 尝试锁定互斥锁（不阻塞）
│   ├── pthread_mutex_unlock  : 解锁互斥锁
│   ├── pthread_mutex_destroy : 销毁互斥锁
│   └── pthread_mutexattr_*   : 设置互斥锁属性（如递归锁等）
│
├── 条件变量（用于线程同步）
│   ├── pthread_cond_init     : 初始化条件变量
│   ├── pthread_cond_wait     : 等待条件变量（释放互斥锁并阻塞等待）
│   ├── pthread_cond_signal   : 发送信号唤醒一个等待条件变量的线程
│   ├── pthread_cond_broadcast: 唤醒所有等待条件变量的线程
│   ├── pthread_cond_destroy  : 销毁条件变量
│
├── 读写锁（用于同步多个读者/单个写者）
│   ├── pthread_rwlock_init   : 初始化读写锁
│   ├── pthread_rwlock_rdlock : 锁定读模式
│   ├── pthread_rwlock_wrlock : 锁定写模式
│   ├── pthread_rwlock_unlock : 解锁读写锁
│   └── pthread_rwlock_destroy: 销毁读写锁
│
├── 屏障（线程同步机制）
│   ├── pthread_barrier_init  : 初始化屏障
│   ├── pthread_barrier_wait  : 等待所有线程到达屏障
│   └── pthread_barrier_destroy: 销毁屏障
│
└── 线程特定数据（线程局部存储）
    ├── pthread_key_create    : 创建线程特定数据的键
    ├── pthread_key_delete    : 删除键
    ├── pthread_setspecific   : 为特定线程设置线程局部数据
    └── pthread_getspecific   : 获取线程局部数据
```

# 2 接口说明和举例


# 3 源码解析
## 3.1 公共结构
### 3.1.1 信号量
```c
// sysdeps/nptl/bits/thread-shared-types.h
/*
1. __extension__ union: 匿名联合体是 GCC 支持的一种扩展特性，通过__extension__可以在代码中使用
2. unsigned long long int是 C99 标准引入的整数类型，它表示无符号的长整型，占用至少 64 位存储空间
3. __wseq: Waiter sequence counter，步长是2
	1. LSB（最低有效位（Least Significant Bit））是当前G2的索引
	2. 当waiters尝试获取某个条件变量的mutex锁时会对__wseq执行fetch-add，signlers则同时对__wseq读取并执行fetch-xor（原子的按位异或）
4. __g1_start: Starting position of G1 (inclusive)
	1. LSB是当前G2的索引
5. __g1_orig_size: Initial size of G1, 步长是4（4 = 100）， 
     * The two least-significant bits represent the condvar-internal lock - 两个最低有效位有特殊含义
     * Only accessed while having acquired the condvar-internal lock.
5. __wrefs：G1和G2所有等待的线程数，是按照8的倍数来的，1个线程为8，2个线程是16，以此类推。 8 = 1000（二进制）
	1. Bit 2 is true if waiters should run futex_wake when they remove the last reference.  pthread_cond_destroy uses this as futex word.
	2. Bit 1 is the clock ID (0 == CLOCK_REALTIME, 1 == CLOCK_MONOTONIC).
	3. Bit 0 is true iff this is a process-shared condvar.
6. For each of the two groups：
	1. __g_refs：表示G1和G2futex waiter的引用计数，例如{2，2}表示G1和G2各有一个waiter，每次递增/递减的步长为2
		1. LSB is true if waiters should run futex_wake when they remove the last reference.
		2. Reference count used by waiters concurrently with signalers that have acquired the condvar-internal lock.
	2. __g_signals：可以被消费的信号数， 步长是2（2=10）， 最低有效位有特殊用途
		1. Used as a futex word by waiters.  Used concurrently by waiters and signalers.
		2. LSB is true iff this group has been completely signaled (i.e., it is
       closed).
	3. __g_size：Waiters remaining in this group - G1和G2在切换之后，G1里面剩余的waiter数量，步长是1
**/
struct __pthread_cond_s
{
	__extension__ union
	{
        __extension__ unsigned long long int __wseq;
        struct
        {
            unsigned int __low;
            unsigned int __high;
        } __wseq32;
    };
    __extension__ union
    {
        __extension__ unsigned long long int __g1_start;
        struct
        {
            unsigned int __low;
            unsigned int __high;
        } __g1_start32;
    };
    unsigned int __g_refs[2] __LOCK_ALIGNMENT;
    unsigned int __g_size[2];
    unsigned int __g1_orig_size;
    unsigned int __wrefs;
    unsigned int __g_signals[2];
};

// sysdeps/nptl/bits/pthreadtypes.h
typedef union
{
	struct __pthread_cond_s __data;
	char __size[__SIZEOF_PTHREAD_COND_T];
	__extension__ long long int __align;
} pthread_cond_t;
```

### 3.1.2 mutex
```c
// sysdeps/nptl/bits/thread-shared-types.h
struct __pthread_mutex_s
{
	int __lock __LOCK_ALIGNMENT;
	unsigned int __count;
	int __owner;
#if !__PTHREAD_MUTEX_NUSERS_AFTER_KIND
	unsigned int __nusers;
#endif
  /* KIND must stay at this position in the structure to maintain
     binary compatibility with static initializers.  */
	int __kind;
	__PTHREAD_COMPAT_PADDING_MID
#if __PTHREAD_MUTEX_NUSERS_AFTER_KIND
	unsigned int __nusers;
#endif
#if !__PTHREAD_MUTEX_USE_UNION
	__PTHREAD_SPINS_DATA;
	__pthread_list_t __list;
# define __PTHREAD_MUTEX_HAVE_PREV      1
#else
	__extension__ union
	{
	    __PTHREAD_SPINS_DATA;
	    __pthread_slist_t __list;
	};
# define __PTHREAD_MUTEX_HAVE_PREV      0
#endif
	__PTHREAD_COMPAT_PADDING_END
};

// sysdeps/nptl/bits/pthreadtypes.h
typedef union
{
  struct __pthread_mutex_s __data;
  char __size[__SIZEOF_PTHREAD_MUTEX_T];
  long int __align;
} pthread_mutex_t;
```

## 3.2 函数解析

### 3.2.1 接口函数
#### 3.2.1.1 pthread_cond_signal
概念解释：
- G1和G2：即新的waiter将加入G2,signal将从G1中取waiter进行唤醒，如果G1没有waiter，再从G2中取waiter唤醒
```c
int
__pthread_cond_signal (pthread_cond_t *cond)
{
	unsigned int wrefs = atomic_load_relaxed (&cond->__data.__wrefs);
	if (wrefs >> 3 == 0)  //没有waiters等待（当cond wait时，wrefs每次是按照8递增的），则直接返回
	    return 0;

	/*
	* #define __PTHREAD_COND_SHARED_MASK 1   -> bit 0用于判断私有还是共享信号量
	* if ((wrefs & __PTHREAD_COND_SHARED_MASK) == 0)
	*   return FUTEX_PRIVATE;
	* else
	*   return FUTEX_SHARED;
	**/
	int private = __condvar_get_private (wrefs); 
	__condvar_acquire_lock (cond, private);  // 调用futex_wait_simple实现线程同步，可以理解为加锁

	unsigned long long int wseq = __condvar_load_wseq_relaxed (cond); //读取cond的__wseq

	/*获取wseq最低位，即最右侧的bit，然后与1进行异或操作，计算g1的位置，即提取 wseq 的最低位，并将其值反转
	**1. 如果 wseq 的最低位是 0，g1 取 1。
	**2. 如果 wseq 的最低位是 1，g1 取 0。
	**/
	unsigned int g1 = (wseq & 1) ^ 1; 
	wseq >>= 1; //wseq右移1位，即除以2

	bool do_futex_wake = false;

	//如果G1已经没有剩余的waiter，那么就需要从G2中取waiter, 通过__condvar_quiesce_and_switch_g1实现
	if ((cond->__data.__g_size[g1] != 0)
      || __condvar_quiesce_and_switch_g1 (cond, wseq, &g1, private)) //交换G1和G2
    {
	    //新G1产生一个可供消费的信号量，之所以递增2，是因为二进制为10，最低位0表示当前group没有全部处理
	    atomic_fetch_add_relaxed (cond->__data.__g_signals + g1, 2);  
		cond->__data.__g_size[g1]--;  //当前group的waiter数量递减
	    do_futex_wake = true;         //需要唤醒一个waiter
    }

	//放锁
	__condvar_release_lock (cond, private);

	if (do_futex_wake)
	    futex_wake (cond->__data.__g_signals + g1, 1, private); //唤醒G1中的一个waiter
	
}
```
#### 3.2.1.2 pthread_cond_wait
内部实现函数是`__pthread_cond_wait_common`

```c
static __always_inline int
__pthread_cond_wait_common (pthread_cond_t *cond, pthread_mutex_t *mutex,
    const struct timespec *abstime)
{
	const int maxspin = 0;
	int err;
	int result = 0;

	LIBC_PROBE (cond_wait, 2, cond, mutex);
	//__wseq递增，步长为2
	uint64_t wseq = __condvar_fetch_add_wseq_acquire (cond, 2);
	//wseq初始值为0。而wseq每次原子地递增2，因此当前wseq是一个偶数。wseq的奇偶性不是一成不变的，当g1和g2发生切换时，wseq会发生变化。
	//Find our group's index.  We always go into what was G2 when we acquired our position  ??
	unsigned int g = wseq & 1; //取G2组
	uint64_t seq = wseq >> 1; //忽略最低位

	//__wrefs递增，步长为8
	unsigned int flags = atomic_fetch_add_relaxed (&cond->__data.__wrefs, 8);
	int private = __condvar_get_private (flags);

	err = __pthread_mutex_unlock_usercnt (mutex, 0);
	if (__glibc_unlikely (err != 0))
    {
	    __condvar_cancel_waiting (cond, seq, g, private);
	    __condvar_confirm_wakeup (cond, private);
	    return err;
    }

	//自旋检查cond->__data.__g_signals+ g 这个group中的信号数量，如果有信号，意味着不用进入内核态，而直接唤醒
	unsigned int signals = atomic_load_acquire (cond->__data.__g_signals + g);
	do
    {
	    while (1)
		{
			unsigned int spin = maxspin;
			while (signals == 0 && spin > 0)
			{
			    /* Check that we are not spinning on a group that's already closed.  */
			    if (seq < (__condvar_load_g1_start_relaxed (cond) >> 1))
					goto done;
			    /* TODO Back off.  */
				/* Reload signals.  See above for MO.  */
				signals = atomic_load_acquire (cond->__data.__g_signals + g);
			    spin--;
			}
		
			//signals的最低位表示是否关闭，1表示group关闭
			if (signals & 1)
			    goto done;

			/* If there is an available signal, don't block. 
			** 如果signals的值低位不是1，并且大于0，则认为获取到了有效的信号。 */
			if (signals != 0)
			    break;

			//如果逻辑没有走到这里，意味着自旋过程中，没有收到信号，于是尝试开始进行阻塞的动作。首先将引用计数增加2，意味着将要进入内核wait。
			//__g_refs递增2（二进制10）
			atomic_fetch_add_acquire (cond->__data.__g_refs + g, 2);
			//cond->__data.__g_signals + g对应的group关闭，或者seq在__g1_start指向的位置之前，则唤醒所有的waiters
			if (((atomic_load_acquire (cond->__data.__g_signals + g) & 1) != 0)
			      || (seq < (__condvar_load_g1_start_relaxed (cond) >> 1)))
		    {
			    /* Our group is closed.  Wake up any signalers that might be waiting.  */
			    __condvar_dec_grefs (cond, g, private);
			    goto done;
			}


			//调用futex_wait进行等待
			// Now block.
			...

			/* Reload signals.  See above for MO.  */
			signals = atomic_load_acquire (cond->__data.__g_signals + g);
		}
	} while (!atomic_compare_exchange_weak_acquire (cond->__data.__g_signals + g, &signals, signals - 2));

	
```
### 3.2.2 内部函数
#### 3.2.2.1 `__condvar_quiesce_and_switch_g1`

检查G2是否有waiter，如果没有waiter，则不进行调整，如果产生新的waiter，则仍然记录在G2中
```c
unsigned int old_orig_size = __condvar_get_orig_size (cond);
uint64_t old_g1_start = __condvar_load_g1_start_relaxed (cond) >> 1;
if (((unsigned) (wseq - old_g1_start - old_orig_size) + cond->__data.__g_size[g1 ^ 1]) == 0)
	return false;
```

下面将G1的__g_signals最低位设置为1，表示关闭G1（atomic_fetch_or_relaxed：**放松内存顺序**执行按位“或”，仅保证操作的原子性，不保证内存操作的顺序）
```c
atomic_fetch_or_relaxed (cond->__data.__g_signals + g1, 1);
```

将G1中剩下的waiter全部唤醒，实际上进入__condvar_quiesce_and_switch_g1方法时，G1的长度已经为0，这里G1又出现了waiter就是由于程序的并发生可能导致的问题，因此这里将G1剩下的waiter进行唤醒，这里__g_refs和已经调用futex_wait进行睡眠的waiter数量相关。
```c
//获取G1的waiters，使用释放内存顺序，保证在该操作之前的所有写入在当前线程中已完成，并且这些写入对其他线程可见
unsigned r = atomic_fetch_or_release (cond->__data.__g_refs + g1, 0);
while ((r >> 1) > 0)
{
    for (unsigned int spin = maxspin; ((r >> 1) > 0) && (spin > 0); spin--)
	{
		/* TODO Back off.  */
		r = atomic_load_relaxed (cond->__data.__g_refs + g1);
	}
    if ((r >> 1) > 0)
	{
		/* There is still a waiter after spinning.  Set the wake-request
	     flag and block.  Relaxed MO is fine because this is just about
	     this futex word.  */
		r = atomic_fetch_or_relaxed (cond->__data.__g_refs + g1, 1);

		if ((r >> 1) > 0)
		    futex_wait_simple (cond->__data.__g_refs + g1, r, private);
		/* Reload here so we eventually see the most recent value even if we do not spin.   */
		r = atomic_load_relaxed (cond->__data.__g_refs + g1);
	}
}
```

设置内存屏障
```c
/*
**atomic_thread_fence_acquire 通常与 atomic_thread_fence_release 搭配使用：
**1. atomic_thread_fence_release：确保发布的数据在屏障之前完成，之后的操作不会被重排到屏障之前。
**2. atomic_thread_fence_acquire：确保读取的数据在屏障之后完成，之前的操作不会被重排到屏障之后。
**/
atomic_thread_fence_acquire()
```

更新__g1_start的新值
```c
__condvar_add_g1_start_relaxed (cond,  (old_orig_size << 1) + (g1 == 1 ? 1 : - 1));

// reopen G1
atomic_store_release (cond->__data.__g_signals + g1, 0);
```

更换G1和G2对应的数组位置，即交换G1和G2
```c
// 获取wseq，__wseq最低位是当前G2的索引，

wseq = __condvar_fetch_xor_wseq_release (cond, 1) >> 1;

//交换G1和G2对应的数组位置
g1 ^= 1;
*g1index ^= 1;
```
更新G2的__g_size
```c
unsigned int orig_size = wseq - (old_g1_start + old_orig_size);
__condvar_set_orig_size (cond, orig_size);
cond->__data.__g_size[g1] += orig_size;
```
# 4 BUG
https://www.sourceware.org/bugzilla/show_bug.cgi?id=25847


