 线程的状态和基本操作
知识点：
（1）如何新建线程；
（2）线程状态的转换；
（3）线程的基本操作；
（4）守护线程Daemon；


知识点：
（1）Lock和synchronized的比较；
（2）AQS设计意图；
（3）如何使用AQS实现自定义同步组件；
（4）可重写的方法；
（5）AQS提供的模板方法；

知识点：
（1）AQS同步队列的数据结构；
（2）独占式锁；
（3）共享式锁；

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


并发容器
CopyOnWriteArrayList
知识点：
（1）实现原理；
（2）COW和ReentrantReadWriteLock的区别；
（3）应用场景；
（4）为什么具有弱一致性；
（5）COW的缺点；

并发容器之ConcurrentLinkedQueue
知识点：
（1）实现原理；
（2）数据结构；
（3）核心方法；
（4）HOPS延迟更新的设计意图

ThreadLocal
知识点：
（1）实现原理；
（2）set方法原理；
（3）get方法原理；
（4）remove方法原理；
（5）ThreadLocalMap

线程池（Executor体系）

读锁 如何获取、释放
写锁 如何获取、释放
如何阻塞
同步状态

concurrent包
AQS
同步队列：链式



重量级锁通过对象内部的监视器（monitor）实现，其中monitor的本质是依赖于底层操作系统的Mutex Lock实现，操作系统实现线程之间的切换需要从用户态到内核态的切换，切换成本非常高



## 头字

### final

内存屏障



### yield()

### join()

底层实现 用 wait()

### ThreadLocal

- InheritableThreadLocal





## 理论基础

线程安全（Thread Safety）
A computer programming concept applicable to multi-threaded code. Thread-safe code only
manipulates shared data structures in a manner that ensures that all threads behave properly
and fulfill their design specifications without unintended interaction. There are various
strategies for making thread-safe data structures.
A program may execute code in several threads simultaneously in a shared address space
where each of those threads has access to virtually all of the memory of every other thread.
Thread safety is a property that allows code to run in multithreaded environments by re-
establishing some of the correspondences between the actual flow of control and the text of
the program, by means of synchronization.



线程安全层次
•
Thread safe: Implementation is guaranteed to be free of race conditions when accessed by
multiple threads simultaneously.
•
Conditionally safe: Different threads can access different objects simultaneously, and
access to shared data is protected from race conditions.
•
Not thread safe: Data structures should not be accessed simultaneously by different
threads.



线程安全实现⼿手段
• Re-entrancy（重进入）
Writing code in such a way that it can be partially executed by a thread, reexecuted by the
same thread or simultaneously executed by another thread and still correctly complete the
original execution. This requires the saving of state information in variables local to each
execution, usually on a stack, instead of in static or global variables or other non-local state.
All non-local state must be accessed through atomic operations and the data-structures must
also be reentrant.



Thread-local storage（线程本地存储）
Variables are localized so that each thread has its own private copy. These variables retain
their values across subroutine and other code boundaries, and are thread-safe since they are
local to each thread, even though the code which accesses them might be executed
simultaneously by another thread.



Immutable objects（不可变对象）
The state of an object cannot be changed after construction. This implies both that only read-
only data is shared and that inherent thread safety is attained. Mutable (non-const) operations
can then be implemented in such a way that they create new objects instead of modifying
existing ones.



Mutual exclusion（互斥）
Access to shared data is serialized using mechanisms that ensure only one thread reads or
writes to the shared data at any time. Incorporation of mutual exclusion needs to be well
thought out, since improper usage can lead to side-effects like deadlocks, livelocks and
resource starvation.



Atomic operations（原子操作）
Shared data is accessed by using atomic operations which cannot be interrupted by other
threads. This usually requires using special machine language instructions, which might be
available in a runtime library. Since the operations are atomic, the shared data is always kept in
a valid state, no matter how other threads access it. Atomic operations form the basis of many
thread locking mechanisms, and are used to implement mutual exclusion primitives.



同步（Synchronization）
Synchronization refers to one of two distinct but related concepts: synchronization of
processes, and synchronization of data. Process synchronization refers to the idea that multiple
processes are to join up or handshake at a certain point, in order to reach an agreement or
commit to a certain sequence of action. Data synchronization refers to the idea of keeping
multiple copies of a dataset in coherence with one another, or to maintain data integrity.
Process synchronization primitives are commonly used to implement data synchronization.

同步引入的问题
• 死锁（Dead Lock）
• 饥饿（Starvation）
• 优先级倒转（Priority Inversion）
• 繁忙等待（Busy Waiting）



同步实现
• 信号量量（Semaphores）：Linux、Solaris
• 屏障（Barriers）：Linux、Pthreads
• 互斥（Mutex）：Linux、Pthreads
• 条件变量（Condition Variables）：Solaris、Pthreads

自旋锁（Spinlock）：Windows、Linux、Pthreads
• 读-写锁（Reader-Writer Lock）：Linux、Solaris、Pthreads



临界区（Critical Section）
Concurrent accesses to shared resources can lead to unexpected or erroneous behavior, so
parts of the program where the shared resource is accessed are protected. This protected
section is the critical section or critical region.
• 锁（Lock）
A lock or mutex (from mutual exclusion) is a synchronization mechanism for enforcing limits on
access to a resource in an environment where there are many threads of execution. A lock is
designed to enforce a mutual exclusion concurrency control policy.



## 同步原语

### 同步原语 - synchronized
• 锁定对象：对象（Object）和类（Class）（存的位置，object）
• 修饰范围：方法（Method）、代码块（Block）
• 特点：重进入（Reentrant）
• 方法 flags：ACC_SYNCHRONIZED
• 字节码：monitorenter 和 monitorexit
• 锁实现：Thin Lock、Inflated、HeavyWeight
进阶阅读：https://wiki.openjdk.java.net/display/HotSpot/Synchronization



重入是线程而言（线程ID），synchronized，  保护，防止死锁

偏向锁->轻量锁->重量锁（java 6）
monitorenter
monitorexit
对象头

单例模式

是否公平决定了 notify FIFO还是随机

Assert Fast-Fail



Monitor 是线程私有的数据结构，每一个线程都有一个可用monitor record列表，同时还有一个全局的可用列表。

每一个被锁住的对象都会和一个monitor关联（对象头的MarkWord中的LockWord指向monitor的起始地址），同时monitor中有一个Owner字段存放拥有该锁的线程的唯一标识，表示该锁被这个线程占用。

### 同步原语 - volatile

高级语言里一条语句往往需要多条CPU指令完成，例如count+=1;至少需要三条CPU指令：

指令1：把变量count从内存加载到CPU寄存器

指令2：在寄存器中执行+1操作

指令3：将结果写入内存，缓存机制导致可能写入的是CPU缓存而不是内存

Java内存模型只保证了基本读取和赋值是原子操作

1、禁止对被修饰的变量进行指令重排序——通过添加内存屏障指令实现，这是一汇编指令，可保证包裹的指令不被重排序

2、保证变量在多线程环境下的可见性——禁用缓存（缓存失效），强制刷新/读取

• 底层：内存屏障（Memory Barrier）
• 语义：可见性

直接操作主内存？？？

**不是副本而是构建新的**

可见性：保证
顺序性：禁止指令重排

> * 重排序
>
> 1. 编译器重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序；
>
> 2. 处理器重排序。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序；
>
> 3. 1、分配一块内存M
>
>    2、在M上初始化Singleton对象
>
>    3、M地址赋给instance变量

原子性：内存屏障（会优化掉多余的），缓存清除刷新，

Lock 前缀指令

> 线程通讯：
> 共享内存 --> 隐式通讯
> 消息传递 --> 显式通讯
>
> 线程同步：
> 在共享内存的并发模型中，同步是显式做的。
>
> 在消息传递的并发模型中，由于消息的发送必须在消息接受之前，所以同步是隐式
>
> 定位内存可见性问题：
> ​    什么对象是内存共享的，什么不是

Volatile 只对基本类型 (byte、char、short、int、long、float、double、boolean) 的赋值 操作和对象的引⽤赋值操作有效。

lock前缀的指令在多核处理器下会引发3件事：

- 将当前处理器缓存行的数据写回到系统内存
- 这个写回内存的操作会引起在其他CPU里缓存了该内存地址的数据无效
- 不是内存屏障却能完成类似内存屏障的功能，阻止屏障两边儿的指令重排序，内存屏障（Memory Barriers）是一组指令，用于实现对内存操作的顺序限制

在多处理器下，为了保证各个处理器的缓存一致，就会实现缓存一致性协议，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效态，当处理器要对这个数据进行操作时，会强制重新从内存里把数据读到缓存。

### 同步原语 - CAS（Compare And Swap）
• 底层：原子信号指令（Atomic Semaphore Instructions）
• 语义：原子性



## 线程 Liveness

Java 线程死锁（Dead Lock）
• Java 线程饥饿（Starvation）



## 问题

《java并发编程的艺术》
内存可见性
重排序
摘自https://juejin.im/post/5ae6c3ef6fb9a07ab508ac85
DCL（双重检验锁）
多线程开发时需要从原子性，有序性，可见性三个方面进行考虑
并发缺点

- 频繁的上下文切换  
  时间片是CPU分配给各个线程的时间，因为时间非常短，所以CPU不断通过切换线程，让我们觉得多个线程是同时执行的，时间片一般是几十毫秒。而每次切换时，需要保存当前的状态起来，以便能够进行恢复先前状态，而这个切换时非常损耗性能，过于频繁反而无法发挥出多线程编程的优势。通常减少上下文切换可以采用无锁并发编程，CAS算法，使用最少的线程和使用协程。

线程安全：死锁
避免死锁的情况：
避免一个线程同时获得多个锁；
避免一个线程在锁内部占有多个资源，尽量保证每个锁只占用一个资源；
尝试使用定时锁，使用lock.tryLock(timeOut)，当超时等待时当前线程不会阻塞；
对于数据库锁，加锁和解锁必须在一个数据库连接里，否则会出现解锁失败的情况

编译器和处理器在重排序时，会遵守数据依赖性，编译器和处理器不会改变存在数据依赖性关系的两个操作的执行顺序

- CAS的问题：  

1. ABA问题 ：添加一个版本号
2. 自旋时间过长
3. 只能保证一个共享变量的原子操作：利用对象整合多个共享变量

Java对象头里的Mark Word里默认的存放的对象的Hashcode,分代年龄和锁标记位。

Java SE 1.6中，锁一共有4种状态，级别从低到高依次是：无锁状态、偏向锁状态、轻量级锁状态和重量级锁状态

final重排序



如何避免线程安全问题？



- 保证共享资源在同一时间只能由一个线程进行操作(原子性，有序性)。
- 将线程操作的结果及时刷新，保证其他线程可以立即获取到修改后的最新数据（可见性）。



Loadstore 非所有操作系统支持

onSpinWait



Java 并发经典模型：Java 生产者和消费者模型



# 参考内容(摘自芋道源码)

happens-before即先行发生原则，这里的先行和时间上先行是两码事。

1、单线程规则：在一个线程内，写在前面的操作先行发生于写在后面的操作，就像刚刚说的一段代码的执行在单线程中看起来是有序的，即使进行了重排序，但最终结果是与程序书写顺序执行的结果一致的，这一点要理解。事实上这个规则是用来保证程序在单线程中执行的正确性，但无法保证程序在多线程中执行的正确性。



2、加锁规则：无论单线程还是多线程，同一个锁如果处于被锁定状态，那么必须先对锁进行释放，后面才能继续进行lock操作



3、volatile规则：这是一条比较重要的规则，volatile关键字会禁止指令重排，比如一个线程先去写一个volatile变量，然后另一个线程去读，那么写入操作肯定会先行发生于读操作



4、依赖传递规则：如果操作A先行发生于操作B，操作B又先行发生于操作C，那么可以得出操作A先行发生于操作C，实际上就是体现happens-before原则具备传递性（有人也叫数据依赖性），或者理解为编译器只会对不存在数据依赖的指令进行重排序



5、线程start规则：Thread对象的start()方法先行发生于此线程的每一个动作，常识



6、线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生，常识



7、线程join规则：这条是关于线程等待的。它是指线程A等待线程B完成（线程A调用线程B的join()方法），当线程B完成后，A线程能够看到B线程的操作结果。当然所谓的“看到”指的是对共享变量来说的。换句话说如果A调用B的join()并成功返回，那么B的任意操作先行发生于该join()操作的返回。常识



8、对象终结规则：对象的初始化完成（构造器执行结束）先行发生于他的finalize()方法的开始，常识



当数据从JVM主内存复制一份拷贝到Java线程的工作内存时，必须出现两个动作：

1、由JVM主内存执行的读（read）操作

2、由线程的工作内存执行相应的load操作

反过来，当数据从线程工作内存拷贝到JVM主内存时，也出现两个操作：

1、由线程的工作内存执行的存储（store）操作

2、由JVM主内存执行的写（write）操作

# sychronized

## 实现原理

> synchronized可以保证方法或者代码块在运行时，同一时刻只有一个方法可以进入到临界区，同时它还可以保证共享变量的内存可见性

Java中每一个对象都可以作为锁，这是synchronized实现同步的基础：

1. 普通同步方法，锁是当前实例对象
2. 静态同步方法，锁是当前类的class对象
3. 同步方法块，锁是括号里面的对象

当一个线程访问同步代码块时，它首先是需要得到锁才能执行同步代码，当退出或者抛出异常时必须要释放锁，那么它是如何来实现这个机制的呢？我们先看一段简单的代码：

```
public class SynchronizedTest {
    public synchronized void test1(){

    }

    public void test2(){
        synchronized (this){

        }
    }}
```

利用javap工具查看生成的class文件信息来分析Synchronize的实现  
![Synchronize-1](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfcZjfmG7iauo9157zkjDINRiaOEBzIFHt77oiaQ3aKhAibvdE7cxoJlVTaVbfkNYZAWHASo5q4hIic6auA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1) 
从上面可以看出，同步代码块是使用monitorenter和monitorexit指令实现的，同步方法（在这看不出来需要看JVM底层实现）依靠的是方法修饰符上的ACC_SYNCHRONIZED实现。 
同步代码块：monitorenter指令插入到同步代码块的开始位置，monitorexit指令插入到同步代码块的结束位置，JVM需要保证每一个monitorenter都有一个monitorexit与之相对应。任何对象都有一个monitor与之相关联，当且一个monitor被持有之后，他将处于锁定状态。线程执行到monitorenter指令时，将会尝试获取对象所对应的monitor所有权，即尝试获取对象的锁； 
同步方法：synchronized方法则会被翻译成普通的方法调用和返回指令如:invokevirtual、areturn指令，在VM字节码层面并没有任何特别的指令来实现被synchronized修饰的方法，而是在Class文件的方法表中将该方法的access_flags字段中的synchronized标志位置1，表示该方法是同步方法并使用调用该方法的对象或该方法所属的Class在JVM的内部对象表示Klass做为锁对象。(摘自：http://www.cnblogs.com/javaminer/p/3889023.html)

下面我们来继续分析，但是在深入之前我们需要了解两个重要的概念：Java对象头，Monitor。

## Java对象头、monitor

Java对象头和monitor是实现synchronized的基础！下面就这两个概念来做详细介绍。

### Java对象头

synchronized用的锁是存在Java对象头里的，那么什么是Java对象头呢？Hotspot虚拟机的对象头主要包括两部分数据：Mark Word（标记字段）、Klass Pointer（类型指针）。其中Klass Point是是对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例，Mark Word用于存储对象自身的运行时数据，它是实现轻量级锁和偏向锁的关键，所以下面将重点阐述

#### Mark Word

Mark Word用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等等。Java对象头一般占有两个机器码（在32位虚拟机中，1个机器码等于4字节，也就是32bit），但是如果对象是数组类型，则需要三个机器码，因为JVM虚拟机可以通过Java对象的元数据信息确定Java对象的大小，但是无法从数组的元数据来确认数组的大小，所以用一块来记录数组长度。下图是Java对象头的存储结构（32位虚拟机）： 
![222222_2](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfcZjfmG7iauo9157zkjDINRian4B6rG6NsQVrp0uYF1nDg1z5MtMTxpD6dkE1Jib8YX8FqBBF6oLh9KQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1) 
对象头信息是与对象自身定义的数据无关的额外存储成本，但是考虑到虚拟机的空间效率，Mark Word被设计成一个非固定的数据结构以便在极小的空间内存存储尽量多的数据，它会根据对象的状态复用自己的存储空间，也就是说，Mark Word会随着程序的运行发生变化，变化状态如下（32位虚拟机）： 
![11111111111_2](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfcZjfmG7iauo9157zkjDINRiadvA7W3qwCGjBrdhribJiawUS3zEJObOaNbyV5Qp4sj04QDduicMZCyXKQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

简单介绍了Java对象头，我们下面再看Monitor。

### Monitor

什么是Monitor？我们可以把它理解为一个同步工具，也可以描述为一种同步机制，它通常被描述为一个对象。 
与一切皆对象一样，所有的Java对象是天生的Monitor，每一个Java对象都有成为Monitor的潜质，因为在Java的设计中 ，每一个Java对象自打娘胎里出来就带了一把看不见的锁，它叫做内部锁或者Monitor锁。 
Monitor 是线程私有的数据结构，每一个线程都有一个可用monitor record列表，同时还有一个全局的可用列表。每一个被锁住的对象都会和一个monitor关联（对象头的MarkWord中的LockWord指向monitor的起始地址），同时monitor中有一个Owner字段存放拥有该锁的线程的唯一标识，表示该锁被这个线程占用。其结构如下：  
![44444](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfcZjfmG7iauo9157zkjDINRia9N9ib5X0CO7pvXOnpPjqqxYrPf0CospKXz3ffcWG55lpz1T4nenerrg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1) 
Owner：初始时为NULL表示当前没有任何线程拥有该monitor record，当线程成功拥有该锁后保存线程唯一标识，当锁被释放时又设置为NULL； 
EntryQ:关联一个系统互斥锁（semaphore），阻塞所有试图锁住monitor record失败的线程。 
RcThis:表示blocked或waiting在该monitor record上的所有线程的个数。 
Nest:用来实现重入锁的计数。 
HashCode:保存从对象头拷贝过来的HashCode值（可能还包含GC age）。 
Candidate:用来避免不必要的阻塞或等待线程唤醒，因为每一次只有一个线程能够成功拥有锁，如果每次前一个释放锁的线程唤醒所有正在阻塞或等待的线程，会引起不必要的上下文切换（从阻塞到就绪然后因为竞争锁失败又被阻塞）从而导致性能严重下降。Candidate只有两种可能的值0表示没有需要唤醒的线程1表示要唤醒一个继任线程来竞争锁。 
摘自：Java中synchronized的实现原理与应用） 
我们知道synchronized是重量级锁，效率不怎么滴，同时这个观念也一直存在我们脑海里，不过在jdk 1.6中对synchronize的实现进行了各种优化，使得它显得不是那么重了，那么JVM采用了那些优化手段呢？

## 锁优化

jdk1.6对锁的实现引入了大量的优化，如自旋锁、适应性自旋锁、锁消除、锁粗化、偏向锁、轻量级锁等技术来减少锁操作的开销。 
锁主要存在四中状态，依次是：无锁状态、偏向锁状态、轻量级锁状态、重量级锁状态，他们会随着竞争的激烈而逐渐升级。注意锁可以升级不可降级，这种策略是为了提高获得锁和释放锁的效率。

### 自旋锁

线程的阻塞和唤醒需要CPU从用户态转为核心态，频繁的阻塞和唤醒对CPU来说是一件负担很重的工作，势必会给系统的并发性能带来很大的压力。同时我们发现在许多应用上面，对象锁的锁状态只会持续很短一段时间，为了这一段很短的时间频繁地阻塞和唤醒线程是非常不值得的。所以引入自旋锁。 
何谓自旋锁？ 
所谓自旋锁，就是让该线程等待一段时间，不会被立即挂起，看持有锁的线程是否会很快释放锁。怎么等待呢？执行一段无意义的循环即可（自旋）。 
自旋等待不能替代阻塞，先不说对处理器数量的要求（多核，貌似现在没有单核的处理器了），虽然它可以避免线程切换带来的开销，但是它占用了处理器的时间。如果持有锁的线程很快就释放了锁，那么自旋的效率就非常好，反之，自旋的线程就会白白消耗掉处理的资源，它不会做任何有意义的工作，典型的占着茅坑不拉屎，这样反而会带来性能上的浪费。所以说，自旋等待的时间（自旋的次数）必须要有一个限度，如果自旋超过了定义的时间仍然没有获取到锁，则应该被挂起。 
自旋锁在JDK 1.4.2中引入，默认关闭，但是可以使用-XX:+UseSpinning开开启，在JDK1.6中默认开启。同时自旋的默认次数为10次，可以通过参数-XX:PreBlockSpin来调整； 
如果通过参数-XX:preBlockSpin来调整自旋锁的自旋次数，会带来诸多不便。假如我将参数调整为10，但是系统很多线程都是等你刚刚退出的时候就释放了锁（假如你多自旋一两次就可以获取锁），你是不是很尴尬。于是JDK1.6引入自适应的自旋锁，让虚拟机会变得越来越聪明。

### 适应自旋锁

JDK 1.6引入了更加聪明的自旋锁，即自适应自旋锁。所谓自适应就意味着自旋的次数不再是固定的，它是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。它怎么做呢？线程如果自旋成功了，那么下次自旋的次数会更加多，因为虚拟机认为既然上次成功了，那么此次自旋也很有可能会再次成功，那么它就会允许自旋等待持续的次数更多。反之，如果对于某个锁，很少有自旋能够成功的，那么在以后要或者这个锁的时候自旋的次数会减少甚至省略掉自旋过程，以免浪费处理器资源。 
有了自适应自旋锁，随着程序运行和性能监控信息的不断完善，虚拟机对程序锁的状况预测会越来越准确，虚拟机会变得越来越聪明。

### 锁消除

为了保证数据的完整性，我们在进行操作时需要对这部分操作进行同步控制，但是在有些情况下，JVM检测到不可能存在共享数据竞争，这是JVM会对这些同步锁进行锁消除。锁消除的依据是逃逸分析的数据支持。 
如果不存在竞争，为什么还需要加锁呢？所以锁消除可以节省毫无意义的请求锁的时间。变量是否逃逸，对于虚拟机来说需要使用数据流分析来确定，但是对于我们程序员来说这还不清楚么？我们会在明明知道不存在数据竞争的代码块前加上同步吗？但是有时候程序并不是我们所想的那样？我们虽然没有显示使用锁，但是我们在使用一些JDK的内置API时，如StringBuffer、Vector、HashTable等，这个时候会存在隐形的加锁操作。比如StringBuffer的append()方法，Vector的add()方法：

```
   public void vectorTest(){
        Vector<String> vector = new Vector<String>();
        for(int i = 0 ; i < 10 ; i++){
            vector.add(i + "");
        }

        System.out.println(vector);
    }
```

在运行这段代码时，JVM可以明显检测到变量vector没有逃逸出方法vectorTest()之外，所以JVM可以大胆地将vector内部的加锁操作消除。

### 锁粗化

我们知道在使用同步锁的时候，需要让同步块的作用范围尽可能小—仅在共享数据的实际作用域中才进行同步，这样做的目的是为了使需要同步的操作数量尽可能缩小，如果存在锁竞争，那么等待锁的线程也能尽快拿到锁。 
在大多数的情况下，上述观点是正确的，LZ也一直坚持着这个观点。但是如果一系列的连续加锁解锁操作，可能会导致不必要的性能损耗，所以引入锁粗话的概念。 
锁粗话概念比较好理解，就是将多个连续的加锁、解锁操作连接在一起，扩展成一个范围更大的锁。如上面实例：vector每次add的时候都需要加锁操作，JVM检测到对同一个对象（vector）连续加锁、解锁操作，会合并一个更大范围的加锁、解锁操作，即加锁解锁操作会移到for循环之外。

### 轻量级锁

引入轻量级锁的主要目的是在多没有多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。当关闭偏向锁功能或者多个线程竞争偏向锁导致偏向锁升级为轻量级锁，则会尝试获取轻量级锁，其步骤如下： 
获取锁

1. 判断当前对象是否处于无锁状态（hashcode、0、01），若是，则JVM首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝（官方把这份拷贝加了一个Displaced前缀，即Displaced Mark Word）；否则执行步骤（3）；
2. JVM利用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指正，如果成功表示竞争到锁，则将锁标志位变成00（表示此对象处于轻量级锁状态），执行同步操作；如果失败则执行步骤（3）；
3. 判断当前对象的Mark Word是否指向当前线程的栈帧，如果是则表示当前线程已经持有当前对象的锁，则直接执行同步代码块；否则只能说明该锁对象已经被其他线程抢占了，这时轻量级锁需要膨胀为重量级锁，锁标志位变成10，后面等待的线程将会进入阻塞状态；

释放锁 
轻量级锁的释放也是通过CAS操作来进行的，主要步骤如下：

1. 取出在获取轻量级锁保存在Displaced Mark Word中的数据；
2. 用CAS操作将取出的数据替换当前对象的Mark Word中，如果成功，则说明释放锁成功，否则执行（3）；
3. 如果CAS操作替换失败，说明有其他线程尝试获取该锁，则需要在释放锁的同时需要唤醒被挂起的线程。

对于轻量级锁，其性能提升的依据是“对于绝大部分的锁，在整个生命周期内都是不会存在竞争的”，如果打破这个依据则除了互斥的开销外，还有额外的CAS操作，因此在有多线程竞争的情况下，轻量级锁比重量级锁更慢；

------

下图是轻量级锁的获取和释放过程 
![22222222222222](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfcZjfmG7iauo9157zkjDINRia7UoKaZqSp5SibKstetMicp4WL1cJb1jyhYwZhHIbEPpTDseb6NibKUbhg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 偏向锁

引入偏向锁主要目的是：为了在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径。上面提到了轻量级锁的加锁解锁操作是需要依赖多次CAS原子指令的。那么偏向锁是如何来减少不必要的CAS操作呢？我们可以查看Mark work的结构就明白了。只需要检查是否为偏向锁、锁标识为以及ThreadID即可，处理流程如下： 
获取锁

1. 检测Mark Word是否为可偏向状态，即是否为偏向锁1，锁标识位为01；
2. 若为可偏向状态，则测试线程ID是否为当前线程ID，如果是，则执行步骤（5），否则执行步骤（3）；
3. 如果线程ID不为当前线程ID，则通过CAS操作竞争锁，竞争成功，则将Mark Word的线程ID替换为当前线程ID，否则执行线程（4）；
4. 通过CAS竞争锁失败，证明当前存在多线程竞争情况，当到达全局安全点，获得偏向锁的线程被挂起，偏向锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码块；
5. 执行同步代码块

释放锁 
偏向锁的释放采用了一种只有竞争才会释放锁的机制，线程是不会主动去释放偏向锁，需要等待其他线程来竞争。偏向锁的撤销需要等待全局安全点（这个时间点是上没有正在执行的代码）。其步骤如下：

1. 暂停拥有偏向锁的线程，判断锁对象石是否还处于被锁定状态；
2. 撤销偏向苏，恢复到无锁状态（01）或者轻量级锁的状态；

------

下图是偏向锁的获取和释放流程 
![image2](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfcZjfmG7iauo9157zkjDINRiayYZ3HE31H6RNA4Eqk0uWvgeGcmAic4kB7icHcIx4m7MhOLLuCyv39iaYg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

##### 重量级锁

重量级锁通过对象内部的监视器（monitor）实现，其中monitor的本质是依赖于底层操作系统的Mutex Lock实现，操作系统实现线程之间的切换需要从用户态到内核态的切换，切换成本非常高。

## 参考资料

1. 周志明：《深入理解Java虚拟机》
2. 方腾飞：《Java并发编程的艺术》
3. Java中synchronized的实现原理与应用）

# volatile

> Java编程语言允许线程访问共享变量，为了确保共享变量能被准确和一致地更新，线程应该确保通过排他锁单独获得这个变量。

上面比较绕口，通俗点讲就是说一个变量如果用volatile修饰了，则Java可以确保所有线程看到这个变量的值是一致的，如果某个线程对volatile修饰的共享变量进行更新，那么其他线程可以立马看到这个更新，这就是所谓的线程可见性。

volatile虽然看起来比较简单，使用起来无非就是在一个变量前面加上volatile即可，但是要用好并不容易（LZ承认我至今仍然使用不好，在使用时仍然是模棱两可）。

## 内存模型相关概念

### 操作系统语义

计算机在运行程序时，每条指令都是在CPU中执行的，在执行过程中势必会涉及到数据的读写。我们知道程序运行的数据是存储在主存中，这时就会有一个问题，读写主存中的数据没有CPU中执行指令的速度快，如果任何的交互都需要与主存打交道则会大大影响效率，所以就有了CPU高速缓存。CPU高速缓存为某个CPU独有，只与在该CPU运行的线程有关。

有了CPU高速缓存虽然解决了效率问题，但是它会带来一个新的问题：数据一致性。在程序运行中，会将运行所需要的数据复制一份到CPU高速缓存中，在进行运算时CPU不再也主存打交道，而是直接从高速缓存中读写数据，只有当运行结束后才会将数据刷新到主存中。举一个简单的例子：

```
i++i++
```

当线程运行这段代码时，首先会从主存中读取i( i = 1)，然后复制一份到CPU高速缓存中，然后CPU执行 + 1 （2）的操作，然后将数据（2）写入到告诉缓存中，最后刷新到主存中。其实这样做在单线程中是没有问题的，有问题的是在多线程中。如下：

假如有两个线程A、B都执行这个操作（i++），按照我们正常的逻辑思维主存中的i值应该=3，但事实是这样么？分析如下：

两个线程从主存中读取i的值（1）到各自的高速缓存中，然后线程A执行+1操作并将结果写入高速缓存中，最后写入主存中，此时主存i==2,线程B做同样的操作，主存中的i仍然=2。所以最终结果为2并不是3。这种现象就是缓存一致性问题。

解决缓存一致性方案有两种：

1. 通过在总线加LOCK#锁的方式
2. 通过缓存一致性协议

但是方案1存在一个问题，它是采用一种独占的方式来实现的，即总线加LOCK#锁的话，只能有一个CPU能够运行，其他CPU都得阻塞，效率较为低下。

第二种方案，缓存一致性协议（MESI协议）它确保每个缓存中使用的共享变量的副本是一致的。其核心思想如下：当某个CPU在写数据时，如果发现操作的变量是共享变量，则会通知其他CPU告知该变量的缓存行是无效的，因此其他CPU在读取该变量时，发现其无效会重新从主存中加载数据。

![212219343783699](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfdkd7jFDpZICoEesyxeSlB22C4bsThWSloEBosBTgbDfyxNwMIe8PkNF3BRNEXPRSYbutjpeD8YQg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### Java内存模型

上面从操作系统层次阐述了如何保证数据一致性，下面我们来看一下Java内存模型，稍微研究一下Java内存模型为我们提供了哪些保证以及在Java中提供了哪些方法和机制来让我们在进行多线程编程时能够保证程序执行的正确性。

在并发编程中我们一般都会遇到这三个基本概念：原子性、可见性、有序性。我们稍微看下volatile

#### 原子性

> 原子性：即一个操作或者多个操作 要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。

原子性就像数据库里面的事务一样，他们是一个团队，同生共死。其实理解原子性非常简单，我们看下面一个简单的例子即可：

```
i = 0;            
---1 j = i ;            
---2 i++;            
---3 i = j + 1;    
---4
```

上面四个操作，有哪个几个是原子操作，那几个不是？如果不是很理解，可能会认为都是原子性操作，其实只有1才是原子操作，其余均不是。

1—在Java中，对基本数据类型的变量和赋值操作都是原子性操作； 
2—包含了两个操作：读取i，将i值赋值给j 
3—包含了三个操作：读取i值、i + 1 、将+1结果赋值给i； 
4—同三一样

在单线程环境下我们可以认为整个步骤都是原子性操作，但是在多线程环境下则不同，Java只保证了基本数据类型的变量和赋值操作才是原子性的（注：在32位的JDK环境下，对64位数据的读取不是原子性操作*，如long、double）。要想在多线程环境下保证原子性，则可以通过锁、synchronized来确保。

> volatile是无法保证复合操作的原子性

#### 可见性

> 可见性是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

在上面已经分析了，在多线程环境下，一个线程对共享变量的操作对其他线程是不可见的。

Java提供了volatile来保证可见性。

当一个变量被volatile修饰后，表示着线程本地内存无效，当一个线程修改共享变量后他会立即被更新到主内存中，当其他线程读取共享变量时，它会直接从主内存中读取。 
当然，synchronize和锁都可以保证可见性。

#### 有序性

> 有序性：即程序执行的顺序按照代码的先后顺序执行。

在Java内存模型中，为了效率是允许编译器和处理器对指令进行重排序，当然重排序它不会影响单线程的运行结果，但是对多线程会有影响。

Java提供volatile来保证一定的有序性。最著名的例子就是单例模式里面的DCL（双重检查锁）。这里LZ就不再阐述了。

## 剖析volatile原理

JMM比较庞大，不是上面一点点就能够阐述的。上面简单地介绍都是为了volatile做铺垫的。

> volatile可以保证线程可见性且提供了一定的有序性，但是无法保证原子性。在JVM底层volatile是采用“内存屏障”来实现的。

上面那段话，有两层语义

1. 保证可见性、不保证原子性
2. 禁止指令重排序

第一层语义就不做介绍了，下面重点介绍指令重排序。

在执行程序时为了提高性能，编译器和处理器通常会对指令做重排序：

1. 编译器重排序。编译器在不改变单线程程序语义的前提下，可以重新安排语句的执行顺序；
2. 处理器重排序。如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序；

指令重排序对单线程没有什么影响，他不会影响程序的运行结果，但是会影响多线程的正确性。既然指令重排序会影响到多线程执行的正确性，那么我们就需要禁止重排序。那么JVM是如何禁止重排序的呢？这个问题稍后回答，我们先看另一个原则happens-before，happen-before原则保证了程序的“有序性”，它规定如果两个操作的执行顺序无法从happens-before原则中推到出来，那么他们就不能保证有序性，可以随意进行重排序。其定义如下：

1. 同一个线程中的，前面的操作 happen-before 后续的操作。（即单线程内按代码顺序执行。但是，在不影响在单线程环境执行结果的前提下，编译器和处理器可以进行重排序，这是合法的。换句话说，这一是规则无法保证编译重排和指令重排）。
2. 监视器上的解锁操作 happen-before 其后续的加锁操作。（Synchronized 规则）
3. 对volatile变量的写操作 happen-before 后续的读操作。（volatile 规则）
4. 线程的start() 方法 happen-before 该线程所有的后续操作。（线程启动规则）
5. 线程所有的操作 happen-before 其他线程在该线程上调用 join 返回成功后的操作。
6. 如果 a happen-before b，b happen-before c，则a happen-before c（传递性）。

我们着重看第三点volatile规则：对volatile变量的写操作 happen-before 后续的读操作。为了实现volatile内存语义，JMM会重排序，其规则如下：

对happen-before原则有了稍微的了解，我们再来回答这个问题JVM是如何禁止重排序的？

![20170104-volatile](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfdkd7jFDpZICoEesyxeSlB22cBHchxE3iaISGxqj5SRu5N9GjTkBNPaE78tzibUU7ylEdtZDaLWVdjw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

观察加入volatile关键字和没有加入volatile关键字时所生成的汇编代码发现，加入volatile关键字时，会多出一个lock前缀指令。lock前缀指令其实就相当于一个内存屏障。内存屏障是一组处理指令，用来实现对内存操作的顺序限制。volatile的底层就是通过内存屏障来实现的。下图是完成上述规则所需要的内存屏障：

volatile暂且下分析到这里，JMM体系较为庞大，不是三言两语能够说清楚的，后面会结合JMM再一次对volatile深入分析。

![20170104-volatile2](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfdkd7jFDpZICoEesyxeSlB2yfNKiaRsGwYRm5gHJxAws76veZpP55WB4zY7f9xgUgGMsOic1xj0rK8g/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 总结

volatile看起来简单，但是要想理解它还是比较难的，这里只是对其进行基本的了解。volatile相对于synchronized稍微轻量些，在某些场合它可以替代synchronized，但是又不能完全取代synchronized，只有在某些场合才能够使用volatile。使用它必须满足如下两个条件：

1. 对变量的写操作不依赖当前值；（类似 i++）
2. 该变量没有包含在具有其他变量的不变式中。（只能保证一个变量）

> volatile经常用于两个两个场景：状态标记两、double check

# CAS

CAS，Compare And Swap，即比较并交换。Doug lea大神在同步组件中大量使用CAS技术鬼斧神工地实现了Java多线程的并发操作。整个AQS同步组件、Atomic原子类操作等等都是以CAS实现的，甚至ConcurrentHashMap在1.8的版本中也调整为了CAS+Synchronized。可以说CAS是整个JUC的基石。

![这里写图片描述](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfe3YRrpblVRufjhKRBzknGY29OPsocbmt1YOgLJblPufqa3zxic3aQ6YnWZoHAVsXhdS10ibgqFKWmA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)这里写图片描述

## CAS分析

在CAS中有三个参数：内存值V、旧的预期值A、要更新的值B，当且仅当内存值V的值等于旧的预期值A时才会将内存值V的值修改为B，否则什么都不干。其伪代码如下：

```
if(this.value == A){
    this.value = B
    return true;
}else{
    return false;
}
```

JUC下的atomic类都是通过CAS来实现的，下面就以AtomicInteger为例来阐述CAS的实现。如下：

```
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;
```

Unsafe是CAS的核心类，Java无法直接访问底层操作系统，而是通过本地（native）方法来访问。不过尽管如此，JVM还是开了一个后门：Unsafe，它提供了硬件级别的原子操作。

valueOffset为变量值在内存中的偏移地址，unsafe就是通过偏移地址来得到数据的原值的。

value当前值，使用volatile修饰，保证多线程环境下看见的是同一个。

我们就以AtomicInteger的addAndGet()方法来做说明，先看源代码：

```
    public final int addAndGet(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
    }

    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
```

内部调用unsafe的getAndAddInt方法，在getAndAddInt方法中主要是看compareAndSwapInt方法：

```
    public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
```

该方法为本地方法，有四个参数，分别代表：对象、对象的地址、预期值、修改值（有位伙伴告诉我他面试的时候就问到这四个变量是啥意思…+_+）。该方法的实现这里就不做详细介绍了，有兴趣的伙伴可以看看openjdk的源码。

CAS可以保证一次的读-改-写操作是原子操作，在单处理器上该操作容易实现，但是在多处理器上实现就有点儿复杂了。

CPU提供了两种方法来实现多处理器的原子操作：总线加锁或者缓存加锁。

*总线加锁*：总线加锁就是就是使用处理器提供的一个LOCK#信号，当一个处理器在总线上输出此信号时，其他处理器的请求将被阻塞住,那么该处理器可以独占使用共享内存。但是这种处理方式显得有点儿霸道，不厚道，他把CPU和内存之间的通信锁住了，在锁定期间，其他处理器都不能其他内存地址的数据，其开销有点儿大。所以就有了缓存加锁。

**缓存加锁**：其实针对于上面那种情况我们只需要保证在同一时刻对某个内存地址的操作是原子性的即可。缓存加锁就是缓存在内存区域的数据如果在加锁期间，当它执行锁操作写回内存时，处理器不在输出LOCK#信号，而是修改内部的内存地址，利用缓存一致性协议来保证原子性。缓存一致性机制可以保证同一个内存区域的数据仅能被一个处理器修改，也就是说当CPU1修改缓存行中的i时使用缓存锁定，那么CPU2就不能同时缓存了i的缓存行。

## CAS缺陷

CAS虽然高效地解决了原子操作，但是还是存在一些缺陷的，主要表现在三个方法：循环时间太长、只能保证一个共享变量原子操作、ABA问题。

**循环时间太长**

如果CAS一直不成功呢？这种情况绝对有可能发生，如果自旋CAS长时间地不成功，则会给CPU带来非常大的开销。在JUC中有些地方就限制了CAS自旋的次数，例如BlockingQueue的SynchronousQueue。

**只能保证一个共享变量原子操作**

看了CAS的实现就知道这只能针对一个共享变量，如果是多个共享变量就只能使用锁了，当然如果你有办法把多个变量整成一个变量，利用CAS也不错。例如读写锁中state的高地位

**ABA问题**

CAS需要检查操作值有没有发生改变，如果没有发生改变则更新。但是存在这样一种情况：如果一个值原来是A，变成了B，然后又变成了A，那么在CAS检查的时候会发现没有改变，但是实质上它已经发生了改变，这就是所谓的ABA问题。对于ABA问题其解决方案是加上版本号，即在每个变量都加上一个版本号，每次改变时加1，即A —> B —> A，变成1A —> 2B —> 3A。

用一个例子来阐述ABA问题所带来的影响。

有如下链表

![这里写图片描述](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfe3YRrpblVRufjhKRBzknGYtbXtI8pggBo3ZW4oGnBAjHW2WzKxfqal2CjM0VdRnnh7Zea7iaPQ3bw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)这里写图片描述

假如我们想要把B替换为A，也就是compareAndSet(this,A,B)。线程1执行B替换A操作，线程2主要执行如下动作，A 、B出栈，然后C、A入栈，最终该链表如下：

![这里写图片描述](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZfe3YRrpblVRufjhKRBzknGYQ1mpMXgb0F3Bfz9YrBSswicC5br3oD4XJoZMq9VsyWa2MQsGx7ko16w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)这里写图片描述

完成后线程1发现仍然是A，那么compareAndSet(this,A,B)成功，但是这时会存在一个问题就是B.next = null,compareAndSet(this,A,B)后，会导致C丢失，改栈仅有一个B元素，平白无故把C给丢失了。

CAS的ABA隐患问题，解决方案则是版本号，Java提供了AtomicStampedReference来解决。AtomicStampedReference通过包装[E,Integer]的元组来对对象标记版本戳stamp，从而避免ABA问题。对于上面的案例应该线程1会失败。

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

compareAndSet有四个参数，分别表示：预期引用、更新后的引用、预期标志、更新后的标志。源码部门很好理解预期的引用 == 当前引用，预期的标识 == 当前标识，如果更新后的引用和标志和当前的引用和标志相等则直接返回true，否则通过Pair生成一个新的pair对象与当前pair CAS替换。Pair为AtomicStampedReference的内部类，主要用于记录引用和版本戳信息（标识），定义如下：

```
    private static class Pair<T> {
        final T reference;
        final int stamp;
        private Pair(T reference, int stamp) {
            this.reference = reference;
            this.stamp = stamp;
        }
        static <T> Pair<T> of(T reference, int stamp) {
            return new Pair<T>(reference, stamp);
        }
    }

    private volatile Pair<V> pair;
```

Pair记录着对象的引用和版本戳，版本戳为int型，保持自增。同时Pair是一个不可变对象，其所有属性全部定义为final，对外提供一个of方法，该方法返回一个新建的Pari对象。pair对象定义为volatile，保证多线程环境下的可见性。在AtomicStampedReference中，大多方法都是通过调用Pair的of方法来产生一个新的Pair对象，然后赋值给变量pair。如set方法：

```
    public void set(V newReference, int newStamp) {
        Pair<V> current = pair;
        if (newReference != current.reference || newStamp != current.stamp)
            this.pair = Pair.of(newReference, newStamp);
    }
```

下面我们将通过一个例子可以可以看到AtomicStampedReference和AtomicInteger的区别。我们定义两个线程，线程1负责将100 —> 110 —> 100，线程2执行 100 —>120，看两者之间的区别。

```
public class Test {
    private static AtomicInteger atomicInteger = new AtomicInteger(100);
    private static AtomicStampedReference atomicStampedReference = new AtomicStampedReference(100,1);

    public static void main(String[] args) throws InterruptedException {

        //AtomicInteger
        Thread at1 = new Thread(new Runnable() {
            @Override
            public void run() {
                atomicInteger.compareAndSet(100,110);
                atomicInteger.compareAndSet(110,100);
            }
        });

        Thread at2 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    TimeUnit.SECONDS.sleep(2);      // at1,执行完
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("AtomicInteger:" + atomicInteger.compareAndSet(100,120));
            }
        });

        at1.start();
        at2.start();

        at1.join();
        at2.join();

        //AtomicStampedReference

        Thread tsf1 = new Thread(new Runnable() {
            @Override
            public void run() {
                try {
                    //让 tsf2先获取stamp，导致预期时间戳不一致
                    TimeUnit.SECONDS.sleep(2);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                // 预期引用：100，更新后的引用：110，预期标识getStamp() 更新后的标识getStamp() + 1
                atomicStampedReference.compareAndSet(100,110,atomicStampedReference.getStamp(),atomicStampedReference.getStamp() + 1);
                atomicStampedReference.compareAndSet(110,100,atomicStampedReference.getStamp(),atomicStampedReference.getStamp() + 1);
            }
        });

        Thread tsf2 = new Thread(new Runnable() {
            @Override
            public void run() {
                int stamp = atomicStampedReference.getStamp();

                try {
                    TimeUnit.SECONDS.sleep(2);      //线程tsf1执行完
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                System.out.println("AtomicStampedReference:" +atomicStampedReference.compareAndSet(100,120,stamp,stamp + 1));
            }
        });

        tsf1.start();
        tsf2.start();
    }

}
```

运行结果：

![这里写图片描述](https://mmbiz.qpic.cn/mmbiz_jpg/JdLkEI9sZfe3YRrpblVRufjhKRBzknGYsOQOIib0pvia1YPLv0rYSk7bx2gr4sJVXcQEQ1jt0ECAcpW6ibAr198ow/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)这里写图片描述

运行结果充分展示了AtomicInteger的ABA问题和AtomicStampedReference解决ABA问题。



