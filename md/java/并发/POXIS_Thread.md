



## POSIX 简介

• POSIX stands for Portable Operating System Interface for UNIX
• POSIX is thus not an implementation of UNIX but rather specifications of the APIs available
for UNIX implementations.
• The standards in POSIX are specified by the IEEE.

The POSIX thread libraries are a standards based thread API for C/C++. It allows one to spawn
a new concurrent process flow. It is most effective on multi-processor or multi-core systems
where the process flow can be scheduled to run on another processor thus gaining speed
through parallel or distributed processing.Threads require less overhead than "forking" or
spawning a new process because the system does not initialize a new system virtual memory
space and environment for the process. While most effective on a multiprocessor system,
gains are also found on uniprocessor systems which exploit latency in I/O and other system
functions which may halt process execution.



• POSIX Threads are described by three parts:
• Functions for runnning threads
• Functions for synchronizing threads
• Functions for scheduling threads



### 线程基础

• 同进程中的线程
• 进程指令（Process instructions）
• 打开的文件描述（FD）
• 信号和信号处理（signals and signal handlers）
• 当前工作目录（current working directory）



+ 线程特有的
  - 线程ID（Thread ID）
  - 寄存器集合、栈指针（set of registers, stack pointer）
  - 局部变量栈和返回地址（stack for local variables, return addresses）
  - 信号掩码（signal mask）
  - 优先级（priority）
  - 返回值（Return value: errno）



## POSIX Thread 基础 API

### 核心 API - 创建线程

* pthread_create
  - Java API - new Thread(Runnable)

源码。。。

void\* 类似 object

###核心 API - 线程属性（pthread_attr_t）
+ pthread_attr_*
  - 状态：是否可 JOIN，默认值 PTHREAD_CREATE_JOINABLE
  - 调度策略：是否实时，默认值
  - 范围：内核（PTHREAD_SCOPE_SYSTEM）、用户（PTHREAD_SCOPE_PROCESS）
  - 栈：地址和大小



### 核心 API - 线程执行结束
+ pthread_join
  - Java API - Thread#join()



### 核心 API - 终止线程
* 等待函数执行完毕

  - void pthread_exit(void* value)
    - Java API - Thread#exit()（非公开方法）

  - int pthread_cancel(pthread_t thread)
    -  Java API - Thread#stop()（不推荐方法）



## POSIX Thread 同步 API线程同步

+ 三大原语（primitives）
  - 互斥（Mutex）
  - 条件变量（Condition variable）
  - 屏障（Barriers）、



+ 互斥（Mutex）
  A POSIX Threads mutex is a lock with a sleep queue — and not a spin lock.
  - 类型：pthread_mutex_t



• 函数：
• pthread_mutex_init - used to create mutex attribute objects respectively.
• pthread_mutex_destroy - used to free a mutex object which is no longer needed.
• 函数（续）：
• pthread_mutex_lock - is used by a thread to acquire a lock on the
specified mutex variable. If the mutex is already locked by another thread, this call will
block the calling thread until the mutex is unlocked.
• pthread_mutex_trylock - will attempt to lock a mutex. However, if the mutex is already
locked, the routine will return immediately with a "busy" error code. This routine may be
useful in preventing deadlock conditions, as in a priority-inversion situation.



pthread_mutex_unlock - will unlock a mutex if called by the owning thread. Calling this
routine is required after a thread has completed its use of protected data if other threads
are to acquire the mutex for their work with the protected data. An error will be returned
if:
• If the mutex was already unlocked
• If the mutex is owned by another thread



* 条件变量（Condition variable）
  A condition variable lets a thread wait for something to happen in the future, and another
  thread to inform it that it has happened.
  - 类型：pthread_cond_t

* 主要函数：
  -  pthread_cond_wait — causes calling thread to wait
  -  pthread_cond_signal — wakes up one waiting thread
  -  pthread_cond_broadcast — wakes up all waiting threads