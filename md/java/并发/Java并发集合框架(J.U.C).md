## Java 并发集合框架

JDK升级    数组   String

高并发中  null是原来就没值，还是互斥看不见

接口契约编程

### CopyOnWrite* 实现
• 并发特征：
• 读：无锁（volatile）、快速（O(1) ）、
• 写：同步（synchronized）、复制（较慢、内存消耗）
• 代表实现：CopyOnWriteList、CopyOnWriteArraySet（底层实现为 CopyOnWriteList）



• 不同 List 实现比较

![](https://ws1.sinaimg.cn/large/006xzusPly1g5tpzin291j30l506f75c.jpg)

### ConcurrentSkipList* 实现

• 并发特征：无锁
• 数据结构：有序（ConcurrentNavigableMap 实现）、跳跃列表（Skip List）变种
• 时间复杂度：平均 log(n) - containsKey、get、put 和 remove 方法
• 代表实现：ConcurrentSkipListMap、ConcurrentSkipListSet（底层实现为
ConcurrentSkipListMap）
• 特征：空间换时间

（内存允许下，读多写多）

• 跳跃列表（Skip List）
• 时间复杂度：搜索 O(log(n))、插⼊入O(log(n))



注意事项：
• size() 方法非 O(C) 操作
• 批量操作无法保证原子执行，如 putAll、equals、toArray、containsValue 以及 clear 方法
• Iterator 和 Spliterators：弱一致性、并非 fail-fast
• 非 null 约束：keys 和 values 均不允许为 null



### ConcurrentHashMap 实现

读多写少

• 并发特征：（查看不同版本的源码）
• 1.5 - 1.6：读锁（部分）（null 造成的），写锁
• 1.7：读无锁，写锁
• 1.8：读无锁，写锁

数据结构：
• <1.8：桶（bucket）
• 1.8：桶（bucket）、平衡树（红黑树） 8

![](https://ws1.sinaimg.cn/large/006xzusPly1g5tpwolpgpj30jn04vta2.jpg)

### BlockingQueue 实现
+ 并发特征：读锁、写锁（共用一个锁对象）

* 典型实现：
  - ArrayBlockingQueue
  - LinkedBlockingQueue
  - SynchronousQueue：特殊的队列，即产即用，不做保存，只可用put，take

+ 子接口/实现：
  - BlockingDeque - LinkedBlockingDeque
  - TransferQueue - LinkedTransferQueue（无锁）

生产者--消费者

尽量用 put，少用 offer，别用 add

take，poll，remove

##  Java 7 Fork/Join 框架



Fork/Join 框架
• 编程模型
• ExecutorService 扩展 - ForkJoinPool
• Future 扩展 - ForkJoinTask、RecursiveTask、RecursiveAction

## Java 8 CompletableFuture

CompletableFuture 背景
• Future 的限制
• 阻塞式结果返回
• 无法链式多个Future
• 无法合并多个Future结果
• 缺少异常处理理



## Java 9 Flow 框架



Reactive Streams
 核心接口
Publisher
Subscriber
Subscription
Processor





# 发展历程

## Java1.4

- 并发实现
  - Java Green Thread
  - Java Native Thread
- 编程模型
  - Thread
  - Runnable

## Java5



## Java 7 时代

- 并行框架
  - Fork/Join
- 编程模型
  - ForkJoinPool，invoke，**invokeAll**
  - ForkJoinTask，
  - RecursiveAction，

Future 限制：

- 无法手动完成
- 阻塞性结果返回
- 无法链式多个 Future
- 无法合并多个 Future 结果
- 缺少异常处理



## Java 8 时代

- 异步并行框架
  - Fork/Join
- 编程模型
  - CompletionStage（interface）
  - CompletableFuture
    异步，流式（链式）处理结果合并，异常处理
- Lamdba(简写：异常问题)
  - supplier
  - comsumer
  - function
  - predicate





## Reactive Streams

实现：

- Java 9 Flow Api
- RxJava
- Reactor



- 核心组件：（推）
  - Publisher 发布者
  - Subscriber 订阅者
  - Subscription#request，cancel
  - Processor =  Publisher  +  Subscriber
- [Reactive Streams](https://github.com/reactive-streams/)
- [Reactive X](http://reactivex.io/intro.html)
  ReactiveX is a library for composing asynchronous and event-based programs by using observable sequences

Publisher -> (Subscriber)Processor (Publisher ) 





#### 前 Java 9 时代

- 观察者模式（拉）

  - Observable（java9淘汰）：使用Vector作为底层存储（全县城安全），无泛型，倒序遍历
    addObserver，setChanged
    方法级别提高（重载 public 调用 protected）
  - Observer

- 事件监听模式：Java Beans 规范 -> 内省

  - EventListener

    - PropertyChangeListener 注册器

  - EventObject

    - PropertyChangeEvent  广播源

      PropertyChangeSupport#addPropertyChangeListener，（触发）firePropertyChange

- Reactor 模式



### Java 9 时代

- Flow API
  - Publisher<T>
    SubmissionPublisher#submit 发布
  - Subscriber<T>
  - Subscription
    request(n)  //向服务器反向请求
  - Processor<T,R>





## [Reactor](https://projectreactor.io/docs/core/release/reference/)

WebFlux -> Reactor -> Reactive Stream API

Cpu密集型 Reactive  无意义

Reactive  观察者模式的扩展



Streams 流式

责任链、观察者、发布-订阅、迭代器





Iterator（拉） 与  Reactive Stream（推） 区别

- Iterator 命令式编程模式（ imperative）



#### 核心接口

Mono 异步 0-1 元素序列：Future<Optional<?>>

Flux     异步 0-N 元素序列：Future<Collection<?>>

- 编程方式

接口，函数式

- CallBack/Future

**Spring**

> ListenableFuture
>
> ListenableFutureCallBack
>
> AsyncListenableTaskExcutor

Future 不足：

1. 不知何时结束，CallBack帮助增加成功回调
2. Future 之间没有相互管理的方式



Flux（可异步）

> just(似of)
>
> subscribe
>
> generate
>
> BiFunction<S,T> 二元操作
>
> range
>
> Scheduler



- 场景



spring webflux  非阻塞io，函数式

反向压力（）



小马哥博文
[《Reactive programming ⼀一种技术 各自表述》](https://mercyblitz.github.io/2018/07/25/Reactive-Programming-一种技术-各自表述/)