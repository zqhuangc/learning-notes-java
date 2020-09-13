• Java 并发限制
• Java 线程池

ReentrantLock
知识点：
（1）重入锁的实现原理；
（2）公平锁的实现原理；
（3）非公平锁的实现原理；
（4）公平锁和非公平锁的比较

ReentrantReadWriteLock
知识点：
（1）如何表示读写状态；
（2）WriteLock的获取和释放；
（3）ReadLock的获取和释放；
（4）锁降级策略；
（5）生成Condition等待队列；
（6）应用场景

##  Java 并发锁

### 重进入锁 - ReentrantLock

openjdk-wiki

• 与 synchronized 的类似点
• 互斥（Mutual Exclusion）
• 重进入（Reentrancy）
• 隐性 Monitor （类似信号量）机制



 与 synchronized 的不同点
• 获得顺序（公平锁和非公平）
• 限时锁定（tryLock）
• 条件对象支持（Condition Support）
• 运维方法



setExclusiveOwnerThread

AQLS

Sync为ReentrantLock里面的一个内部类，它继承AQS（AbstractQueuedSynchronizer），它有两个子类：公平锁FairSync和非公平锁NonfairSync。

ReentrantLock里面大部分的功能都是委托给Sync来实现的，同时Sync内部定义了lock()抽象方法由其子类去实现，默认实现了nonfairTryAcquire(int acquires)方法，可以看出它是非公平锁的默认实现方式。



比较非公平锁和公平锁获取同步状态的过程，会发现两者唯一的区别就在于公平锁在获取同步状态时多了一个限制条件：hasQueuedPredecessors()，定义如下：

```
    public final boolean hasQueuedPredecessors() {
        Node t = tail;  //尾节点
        Node h = head;  //头节点
        Node s;

        //头节点 != 尾节点
        //同步队列第一个节点不为null
        //当前线程是同步队列第一个节点
        return h != t &&
                ((s = h.next) == null || s.thread != Thread.currentThread());
    }
```

该方法主要做一件事情：主要是判断当前线程是否位于CLH同步队列中的第一个。如果是则返回true，否则返回false。

### 重进入读写锁 - ReentrantReadWriteLock

int 高16位

• 继承 ReentrantLock 的特性
互斥（Mutual Exclusion）
• 重进入（Reentrancy）
• 获得顺序（公平和非公平）
• 中断（Interruption）
• 条件对象支持（Condition Support）

• 超越 ReentrantLock 的特性
• 共享-互斥模式（Shared - Exclusive）
• 读锁 - 共享
• 写锁 - 互斥
• 锁降级（Lock downgrading）



• 其他语⾔言实现
• POSIX - pthread_rwlock_t
• C++17 - std::shared_mutex

### 邮票锁 - StampedLock（since jdk1.8）

三种锁模式
• 写（Writing）
• 读（Reading）
• 优化读（Optimistic Reading）



类似实现
• Seqlock - https://en.wikipedia.org/wiki/Seqlock
• 参考资料料
• 《Effective synchronisation on Linux systems》
• 《Can Seqlocks Get Along With Programming Language Memory Models?》



### CAS

https://mp.weixin.qq.com/s/--AMdl0GZQkY1MWIWQ-HHA

**循环时间太长**

**只能保证一个共享变量原子操作**

在JUC中有些地方就限制了CAS自旋的次数，例如BlockingQueue的SynchronousQueue。

#### ABA 问题

预期值（原值）一样，不代表中间没有发生变化，A to B to A

版本号

AtomicStampedReference的compareAndSet()方法定义如下：

```
    public boolean compareAndSet(V   expectedReference,
                                 V   newReference,
                                 int expectedStamp,
                                 int newStamp) {
        Pair<V> current = pair;
        return
            expectedReference == current.reference &&
            expectedStamp == current.stamp &&
            ((newReference == current.reference &&
              newStamp == current.stamp) ||
             casPair(current, Pair.of(newReference, newStamp)));
    }
```



Pair记录着对象的引用和版本戳，版本戳为int型，保持自增。同时Pair是一个不可变对象，其所有属性全部定义为final，对外提供一个of方法，该方法返回一个新建的Pari对象。pair对象定义为volatile，保证多线程环境下的可见性。在AtomicStampedReference中，大多方法都是通过调用Pair的of方法来产生一个新的Pair对象，然后赋值给变量pair。如set方法：







## Java 原子操作

java.util.concurrent.atomic.Atomic* 类
• AtomicBoolean
• AtomicInteger 与 AtomicIntegerArray
• AtomicLong 与 AtomicLongArray
• AtomicReference 与 AtomicReferenceArray
• AtomicMarkableReference 与 AtomicStampedReference





• java.util.concurrent.atomic.*Adder 类
• java.util.concurrent.atomic.Striped64
• DoubleAccumulator
• DoubleAdder
• LongAccumulator
• LongAdder



### Java 并发限制

CountDownLatch
CyclicBarrier
Semaphore



## Java 线程池

• Executor 实现
• ThreadPoolExecutor
• ScheduledExecutorService 
• Runnable V.S. Callable
• Future







