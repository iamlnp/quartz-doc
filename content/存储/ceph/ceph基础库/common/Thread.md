代码路径： `src\common\Thread.h`

# 1 Thead类定义

```c++
class Thread {
private:
  	pthread_t thread_id; //线程id
  	pid_t pid;		//进程id
  	int cpuid;
  	const char *thread_name;
public:
  	Thread();
  	virtual ~Thread();
  
protected:
  	virtual void *entry() = 0;
public:
  	void create(const char *name, size_t statcksize = 0);
  	int try_create(size_t stacksize);
}
```



# 2 重点成员函数解析

## 2.1 create()

```c++
void Thread::create(const char *name, size_t stacksize)
{
  ceph_assert(strlen(name) < 16);
  thread_name = name;

  int ret = try_create(stacksize); 
	...
}
```

## 2.2 try_create()

```c++
int Thread::try_create(size_t stacksize)
{
  pthread_attr_t *thread_attr = NULL;
  pthread_attr_t thread_attr_loc;
  
  stacksize &= CEPH_PAGE_MASK;  // must be multiple of page
  if (stacksize) {
    thread_attr = &thread_attr_loc;
    pthread_attr_init(thread_attr);  //初始化线程属性，调用pthread_attr_init之后，pthread_t结构所包含的内容就是操作系统实现支持的线程所有属性的默认值
    pthread_attr_setstacksize(thread_attr, stacksize); //设置线程栈大小
  }

  int r;

  // The child thread will inherit our signal mask.  Set our signal mask to
  // the set of signals we want to block.  (It's ok to block signals more
  // signals than usual for a little while-- they will just be delivered to
  // another thread or delieverd to this thread later.)
  sigset_t old_sigset;
  if (g_code_env == CODE_ENVIRONMENT_LIBRARY) {
    block_signals(NULL, &old_sigset);
  }
  else {
    int to_block[] = { SIGPIPE , 0 };
    block_signals(to_block, &old_sigset);
  }
  
  //调用系统函数创建线程
  //thread_id: pthread库对线程的编号，而不是Linux系统对线程的编号
  r = pthread_create(&thread_id, thread_attr, _entry_func, (void*)this); 
  restore_sigset(&old_sigset);

  if (thread_attr) {
    pthread_attr_destroy(thread_attr);	//去除pthread_attr_init设置的初始化
  }

  return r;
}
```

## 2.3 join()

```c++
int Thread::join(void **prval)
{
  if (thread_id == 0) {
    ceph_abort_msg("join on thread that was never started");
    return -EINVAL;
  }

  int status = pthread_join(thread_id, prval);
	...

  thread_id = 0;
  return status;
}
```

## 2.4 detach()

```c++
int Thread::detach()
{
  return pthread_detach(thread_id);
}
```

## 2.5 kill()

```c++
1. 函数形式一：
void kill(std::thread& t, int signal)
{
  auto r = pthread_kill(t.native_handle(), signal);
  if (r != 0) {
    throw std::system_error(r, std::generic_category());
  }
}

2. 函数形式二：
int Thread::kill(int signal)
{
  if (thread_id)
    return pthread_kill(thread_id, signal);
  else
    return -EINVAL;
}
```



## 2.6 is_started()

```c++
bool Thread::is_started() const
{
  return thread_id != 0;
}
```

## 2.7 am_self() 

```c++
bool Thread::am_self() const
{
  return (pthread_self() == thread_id);
}
```



## 2.8 _entry_func()线程回调函数

```c++
void *Thread::_entry_func(void *arg) {
  void *r = ((Thread*)arg)->entry_wrapper();
  return r;
}

void *Thread::entry_wrapper()
{
  // may return -ENOSYS on other platforms, 使用syscall()直接用SYS_gettid的系统调用号去获取线程号,即linux线程号
  int p = ceph_gettid(); 
  if (p > 0)
    pid = p;
  if (pid && cpuid >= 0)
    _set_affinity(cpuid);

  //pthread_self()功能是获得线程自身的ID
  //对linux而言，ceph_pthread_setname就是pthread_setname_np，用于设置线程名称
  ceph_pthread_setname(pthread_self(), thread_name); 
  return entry();		// 最终是执行entry(), 因此entry是线程的继承类需要实现的接口
}
```





# 3 用法举例

1. 在类中声明thread类

```c++
// 1. 声明类型
class Tfs_ioa 
{
  	Tfs_Ioa::Tfs_Ioa(CephContext *cct, Tfs_IOADaemon *ioa_daemon, client_t whoami)
    : load_thread(this) { } // 初始化load_thread
  
  	class LoadThread : public Thread { 	// 继承Tread
    public:
        LoadThread() {}
        explicit LoadThread(Tfs_Ioa *_ioa) : ioa(_ioa) {}
        void *entry() override {  // 线程处理函数
            ioa->handle_load_thread();  
            return 0;
        }

    private:
        Tfs_Ioa *ioa;
    } load_thread;
}

// 2. 定义线程处理函数
void Tfs_Ioa::handle_load_thread() {
    int rc = 0;
    dout(5) << "handle_load_thread start" << dendl;

    while (1) {
			//do something
    }
}
```

2. 启动线程

```c++
void Tfs_Ioa::start(bool create)
{
    load_thread.create("ioa_load"); // 启动就执行线程处理函数
}
```

3. 销毁线程

```c++
void Tfs_Ioa::shutdown() 
{   
	if (load_thread.is_started()) {
        if (load_thread.am_self()) {  //本线程可以不需要做任何事情
            // do-nothing
        } else { 	//本进程的其他线程执行shutdown时，需要等待线程join
						...
            load_thread.join();
						...
        }
        dout(20) << "ioa load_thread shutdown OK " << dendl;
    }
}
```

