# Netty线程池



Netty自己实现了一套异步的，局部串行的，无锁化的，高性能线程池，它不同于JDK的work-thread模式的线程池，它扩展了JDK的线程池接口ExecutorService，看类图还知道Netty还扩展了定时线程池的接口，即Netty同时支持定时任务和普通任务的执行，



1.对于boss NioEventLoop来说，轮询到的是基本上就是连接事件，后续的事情就通过他的pipeline将连接扔给一个worker NioEventLoop处理
 2.对于worker NioEventLoop来说，轮询到的基本上都是io读写事件，后续的事情就是通过他的pipeline将读取到的字节流传递给每个channelHandler来处理

1、实例化NioEventLoopGroup时，如果指定线程池大小，那么nThreads就是指定的值，反之是机器的处理器核心数*2个

2、boss线程池没必要设置多个线程数，因为针对一个服务端端口，实际起作用的I/O线程只有一个。关于线程池线程数目的配置，需要理解I/O密集型任务和CPU密集型任务的区别

3、每个NioEventLoopGroup都对应了一个线程执行器，默认是Netty提供的ThreadPerTaskExcutor，作用是创建NioEventLoopGroup底层JDK的线程对象——Thread，用户也可以自己实现并当做参数传入

4、EventLoopGroup（其实是MultithreadEventExecutorGroup类）内部，维护了一个类型为EventExecutor[]的children数组，大小是nThreads，这样就构成了一个线程池，也就是所谓的Netty线程池的本来面目

5、MultithreadEventExecutorGroup构造器会调用newChild抽象方法（模板方法模式的实现），并结合线程执行器来赋值这个children数组，该方法会返回NioEventLoop实例，同时给每个NIO线程都附加一个任务队列，实现局部串行无锁化线程池。

6、提供了分配线程的组件，主要是为了给新来的Channel分配线程，也叫线程选择器，我更细化叫线程轮询器，它也是NioEventLoopGroup构造器内创建

#### MultithreadEventExecutorGroup

构造器主要做了三件事：

1、创建NioEventLoopGroup的线程执行器——ThreadPerTaskExcutor

它是对Java的线程池接口的实现，负责创建NioEventLoopGroup的底层Java线程Thread



EventLoopGroup实现的线程池其实就是类MultithreadEventExecutorGroup内部维护的类型为EventExecutor[]的数组，其大小是nThreads，而EventExecutor本身是一个Netty线程相关的接口，这样就构成了一个Netty线程池

```java
protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
        if (nThreads <= 0) {
            throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
        }

        if (executor == null) {
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }

        // 
        children = new EventExecutor[nThreads];
        // for循环+newChild(executor,args)方法，去构造nThreads个NioEventLoop对象
        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                // 
                children[i] = newChild(executor, args);
                /** NioEventLoopGroup
                @Override
                protected EventLoop newChild(Executor executor, Object... args) throws Exception {
                    EventLoopTaskQueueFactory queueFactory = args.length == 4 ? (EventLoopTaskQueueFactory) args[3] : null;
                    return new NioEventLoop(this, executor, (SelectorProvider) args[0],
                        ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2], queueFactory);
                }
                */
                success = true;
            } catch (Exception e) {
                // TODO: Think about if this is a good exception type
                throw new IllegalStateException("failed to create a child event loop", e);
            } finally {
                if (!success) {
                    for (int j = 0; j < i; j ++) {
                        children[j].shutdownGracefully();
                    }

                    for (int j = 0; j < i; j ++) {
                        EventExecutor e = children[j];
                        try {
                            while (!e.isTerminated()) {
                                e.awaitTermination(Integer.MAX_VALUE, TimeUnit.SECONDS);
                            }
                        } catch (InterruptedException interrupted) {
                            // Let the caller handle the interruption.
                            Thread.currentThread().interrupt();
                            break;
                        }
                    }
                }
            }
        }

        // EvenLoopGroup 创建线程选择器，为新连接 Channel 分配 NioEventLoop 线程
        chooser = chooserFactory.newChooser(children);

        final FutureListener<Object> terminationListener = new FutureListener<Object>() {
            @Override
            public void operationComplete(Future<Object> future) throws Exception {
                if (terminatedChildren.incrementAndGet() == children.length) {
                    terminationFuture.setSuccess(null);
                }
            }
        };

        for (EventExecutor e: children) {
            e.terminationFuture().addListener(terminationListener);
        }

        Set<EventExecutor> childrenSet = new LinkedHashSet<EventExecutor>(children.length);
        Collections.addAll(childrenSet, children);
        readonlyChildren = Collections.unmodifiableSet(childrenSet);
    }
```

#### ThreadPerTaskExcutor

```java

public final class ThreadPerTaskExecutor implements Executor {
    private final ThreadFactory threadFactory;

    public ThreadPerTaskExecutor(ThreadFactory threadFactory) {
        if (threadFactory == null) {
            throw new NullPointerException("threadFactory");
        }
        this.threadFactory = threadFactory;
    }

    @Override
    public void execute(Runnable command) {
        threadFactory.newThread(command).start();
    }
}

public interface ThreadFactory {

    /**
     * Constructs a new {@code Thread}.  Implementations may also initialize
     * priority, name, daemon status, {@code ThreadGroup}, etc.
     *
     * @param r a runnable to be executed by new thread instance
     * @return constructed thread, or {@code null} if the request to
     *         create a thread is rejected
     */
    Thread newThread(Runnable r);
}

// DefaultThreadFactory
```



#### NioEvenLoop

Netty在实例化NioEventLoopGroup时，会在其父类——MultithreadEventExecutorGroup里创建NioEventLoopGroup的线程执行器——ThreadPerTaskExcutor，也就是JDK的Excuter接口的一个实例

创建一个EventExecutor类型的数组——children，它就是Netty线程池的样子

在一个for循环里创建[children数组.length]个NioEventLoop实例，也就是为children数组赋值。

`io.netty.util.concurrent.MultithreadEventExecutorGroup#newChild`

线程选择器工厂，它的作用是给新客户端连接分配线程，实现负载均衡的效果

Netty的线程池的线程，不是直接就是Thread对象，而是聚合了Thread以及线程执行器的事件循环组件——EventLoop

```
private final IntSupplier selectNowSupplier = new IntSupplier() {
    public int get() throws Exception {
        return NioEventLoop.this.selectNow();
    }
};
```

```java
// NioEventLoopGroup
@Override
    protected EventLoop newChild(Executor executor, Object... args) throws Exception {
        EventLoopTaskQueueFactory queueFactory = args.length == 4 ? (EventLoopTaskQueueFactory) args[3] : null;
        return new NioEventLoop(this, executor, (SelectorProvider) args[0],
            ((SelectStrategyFactory) args[1]).newSelectStrategy(), (RejectedExecutionHandler) args[2], queueFactory);
    }

// NioEventLoop
NioEventLoop(NioEventLoopGroup parent, Executor executor, SelectorProvider selectorProvider,
                 SelectStrategy strategy, RejectedExecutionHandler rejectedExecutionHandler,
                 EventLoopTaskQueueFactory queueFactory) {
        // DefaultEventExecutorChooserFactory 标注了 unstable
        //
        super(parent, executor, false, newTaskQueue(queueFactory), newTaskQueue(queueFactory),
                rejectedExecutionHandler);
        if (selectorProvider == null) {
            throw new NullPointerException("selectorProvider");
        }
        if (strategy == null) {
            throw new NullPointerException("selectStrategy");
        }
        provider = selectorProvider;
        final SelectorTuple selectorTuple = openSelector();
        selector = selectorTuple.selector;
        unwrappedSelector = selectorTuple.unwrappedSelector;
        selectStrategy = strategy;
    }

protected SingleThreadEventLoop(EventLoopGroup parent, Executor executor,
                                    boolean addTaskWakesUp, Queue<Runnable> taskQueue, Queue<Runnable> tailTaskQueue,
                                    RejectedExecutionHandler rejectedExecutionHandler) {
        super(parent, executor, addTaskWakesUp, taskQueue, rejectedExecutionHandler);
        tailTasks = ObjectUtil.checkNotNull(tailTaskQueue, "tailTaskQueue");
    }

protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor,
                                        boolean addTaskWakesUp, Queue<Runnable> taskQueue,
                                        RejectedExecutionHandler rejectedHandler) {
        super(parent);
        this.addTaskWakesUp = addTaskWakesUp;
        this.maxPendingTasks = DEFAULT_MAX_PENDING_EXECUTOR_TASKS;
        this.executor = ThreadExecutorMap.apply(executor, this);
        this.taskQueue = ObjectUtil.checkNotNull(taskQueue, "taskQueue");
        rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
    }

// NioEvenLoop   jctools
private static Queue<Runnable> newTaskQueue0(int maxPendingTasks) {
        // This event loop never calls takeTask()
        return maxPendingTasks == Integer.MAX_VALUE ? PlatformDependent.<Runnable>newMpscQueue()
                : PlatformDependent.<Runnable>newMpscQueue(maxPendingTasks);
    }


protected SingleThreadEventExecutor(EventExecutorGroup parent, Executor executor,
                                        boolean addTaskWakesUp, Queue<Runnable> taskQueue,
                                        RejectedExecutionHandler rejectedHandler) {
        super(parent);
        this.addTaskWakesUp = addTaskWakesUp;
        this.maxPendingTasks = DEFAULT_MAX_PENDING_EXECUTOR_TASKS;
        this.executor = ThreadExecutorMap.apply(executor, this);
        this.taskQueue = ObjectUtil.checkNotNull(taskQueue, "taskQueue");
        rejectedExecutionHandler = ObjectUtil.checkNotNull(rejectedHandler, "rejectedHandler");
    }
```





### MultithreadEventLoopGroup



```java
public abstract class MultithreadEventLoopGroup extends MultithreadEventExecutorGroup implements EventLoopGroup {

    private static final InternalLogger logger = InternalLoggerFactory.getInstance(MultithreadEventLoopGroup.class);

    private static final int DEFAULT_EVENT_LOOP_THREADS;

    static {
        DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
                "io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));

        if (logger.isDebugEnabled()) {
            logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
        }
    }

    /**
     * @see MultithreadEventExecutorGroup#MultithreadEventExecutorGroup(int, Executor, Object...)
     */
    protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
        super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
    }

    /**
     * @see MultithreadEventExecutorGroup#MultithreadEventExecutorGroup(int, ThreadFactory, Object...)
     */
    protected MultithreadEventLoopGroup(int nThreads, ThreadFactory threadFactory, Object... args) {
        super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, threadFactory, args);
    }

    /**
     * @see MultithreadEventExecutorGroup#MultithreadEventExecutorGroup(int, Executor,
     * EventExecutorChooserFactory, Object...)
     */
    protected MultithreadEventLoopGroup(int nThreads, Executor executor, EventExecutorChooserFactory chooserFactory,
                                     Object... args) {
        super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, chooserFactory, args);
    }

    @Override
    protected ThreadFactory newDefaultThreadFactory() {
        return new DefaultThreadFactory(getClass(), Thread.MAX_PRIORITY);
    }

    @Override
    public EventLoop next() {
        return (EventLoop) super.next();
    }

    @Override
    protected abstract EventLoop newChild(Executor executor, Object... args) throws Exception;

    // next() 在bossGroup里挑选一根线程去驱动任务
    @Override
    public ChannelFuture register(Channel channel) {
        return next().register(channel);
    }

    @Override
    public ChannelFuture register(ChannelPromise promise) {
        return next().register(promise);
    }

    @Deprecated
    @Override
    public ChannelFuture register(Channel channel, ChannelPromise promise) {
        return next().register(channel, promise);
    }

}

```

### AbstractBootstrap#initAndRegister

事件循环线程的启动关键（`对socket连接进行监听的轮询线程启动`？），看下面分析

### AbstractChannel$AbstractUnsafe#register

`SingleThreadEventExecutor`

NioEvenLoop 的运行关键

```java
initAndRegister--->...
@Override
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}

@Override
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    promise.channel().unsafe().register(this, promise);
    return promise;
}

// AbstractChannel$AbstractUnsafe
        @Override
        public final void register(EventLoop eventLoop, final ChannelPromise promise) {
            if (eventLoop == null) {
                throw new NullPointerException("eventLoop");
            }
            if (isRegistered()) {
                promise.setFailure(new IllegalStateException("registered to an event loop already"));
                return;
            }
            if (!isCompatible(eventLoop)) {
                promise.setFailure(
                        new IllegalStateException("incompatible event loop type: " + eventLoop.getClass().getName()));
                return;
            }
            //
            AbstractChannel.this.eventLoop = eventLoop;

            // 
            if (eventLoop.inEventLoop()) {
                register0(promise);
            } else {
                try {
                    // 程序启动时的线程不是 reactor 线程
                    // SingleThreadEventLoop entends SingleThreadEventExecutor
                    // SingleThreadEventExecutor#execute ->...
                    // NioEventLoop#run
                    eventLoop.execute(new Runnable() {
                        @Override
                        public void run() {
                            register0(promise);
                        }
                    });
                } catch (Throwable t) {
                    logger.warn(
                            "Force-closing a channel whose registration task was not accepted by an event loop: {}",
                            AbstractChannel.this, t);
                    closeForcibly();
                    closeFuture.setClosed();
                    safeSetFailure(promise, t);
                }
            }
        }
```



> selectionKey=javaChannel().register(eventLoop().selector,0,this);



#### DefaultThreadProperties

netty的 reactor 线程在添加一个任务的时候被创建，该线程实体为 `FastThreadLocalThread`

**最后线程执行主体为`NioEventLoop`的`run`方法。**



# 事件循环

处理已经就绪的I/O事件，另一方面也会处理MPSCQ里的异步任务

1、执行Netty自己封装的select()方法——负责检查是否有I/O事件已经就绪的Channel，并在循环里轮询出注册到当前Selector上的已经就绪的Channel

2、处理I/O任务，核心就是processSelectedKeys()方法——负责处理I/O事件，比如在第1步轮询出了新连接接入的OP_ACCEPT事件，下一步就是在该方法里处理该事件

3、并发的处理异步任务，核心是调用runAllTasks()方法——它负责处理MPSCQ的task，该队列里有普通任务，也可能有定时任务（看用户的使用方式），这些任务按照来源又可以分为外部线程扔到MPSCQ里的任务和Netty自己封装的底层的任务

CPU任务执行时间的比例=(100-ioRatio）/ ioRatio

NIO线程执行CPU任务和I/O任务的耗时比例，默认是1:1，靠参数ioRatio调节，这个参数提供了一个粗略的时间分配机制，用来大致控制处理I/O相关(Socket的读、写、连接、关闭等)任务和CPU任务的时间分配比。用户可以自己调用NioEventLoop类的setIoRatio方法配置，注意最大不能为100，且是正数

## SingleThreadEventExecutor#doStartThread()

```java
 /**
     * NioEventLoop 线程启动的核心方法
     */
    private void doStartThread() {
        assert thread == null;
        executor.execute(new Runnable() {
            @Override
            public void run() {
               // 前面省略
                try {
                    // NioEventLoop 线程的事件循环核心代码, 在run 方法里
                    SingleThreadEventExecutor.this.run();
                    success = true;
                } catch (Throwable t) {
                    logger.warn("Unexpected exception from an event executor: ", t);
                } finally {
                  // 省略
                    try {
                       // 省略
                    } finally {
                        try {
                            cleanup();
                        } finally {
                            STATE_UPDATER.set(SingleThreadEventExecutor.this, ST_TERMINATED);
                            threadLock.release();
                            if (!taskQueue.isEmpty()) {
                                logger.warn(
                                        "An event executor terminated with " +
                                                "non-empty task queue (" + taskQueue.size() + ')');
                            }

                            terminationFuture.setSuccess(null);
                        }
                    }
                }
            }
        });
    }
```



## `NioEventLoop#run`

处理已经就绪的I/O事件，另一方面也会处理MPSCQ里的异步任务

NioEventLoop的父类——SingleThreadEventLoop类的构造器

newTaskQueue MpscQueue的队列，该队列的全称是Multi procedure single consumer queue

1、multi procedure对应的是Netty外部线程，外部线程可能会有多个

2、single consumer是Netty的NioEventLoop线程（即当前这个I/O线程）

3、知道Netty封装的每个NIO线程都会依赖一个MpscQueue

```java
Netty要执行某任务时，如果该任务是被外部线程驱动，即不是被NioEventLoop（I/O线程）线程驱动，那么Netty就将其放到NioEventLoop持有的MpscQueue里排队，等待让NioEventLoop线程执行。
```



> 线程可以驱动任务，两者不一样。即Java提供了一种描述任务的方式，可以由Runnable接口实现，覆写run方法即可创建一个任务，该任务可执行自定义的逻辑。当从Runnable接口导出一个类时，它必须实现run方法，但是这个run方法并无特殊之处——它不会产生任何线程的能力，要实现线程的行为，就必须显式地将一个任务附着到Thread上，在驱动这个任务，才能让该任务实现线程的行为。
>
> 《Java编程思想》
>
> Executor在客户端和任务执行之间提供了一个间接层，与客户端直接执行任务不同，这个中介将执行任务。Executor允许管理异步任务的执行，而无须显式地管理线程的生命周期。Executor在Java中是启动任务的优选方法。
>
>
>
> 在Java中，Thread类自身不执行任何操作，它只是驱动附着在它上面的任务，不要将线程和任务混为一谈。
>
> 《Java编程思想》
>
>
>
> NioEventLoopGroup创建NioEventLoop的关键方法是newChild(......),主要做了三件事：
>
> - 保存NioEventLoopGroup的线程执行器ThreadPerTaskExecutor
> - 为NioEventLoop创建MpscQueue，用于外部线程执行Netty任务时，如判断该任务不在NioEventLoop的线程里执行，那么就直接将其塞到该队列，由NioEventLoop对应的线程执行，保证线程安全，深层次的考虑还有规避JDK NIO的一些坑
> - 新建一个I/O多路复用器绑定到NioEventLoop，以便未来能让绑定的线程执行器去轮询Channel上的I/O事件

register0

```java
@Override
protected void run() {
    // 一般会一直轮询
    for (;;) {// hasTasks 判断当前异步任务队列有没有任务需要执行
        try {
            // 计算轮询策略
            switch (selectStrategy.calculateStrategy(selectNowSupplier, hasTasks())) {
                case SelectStrategy.CONTINUE:
                    continue;
                case SelectStrategy.SELECT:
                    // 1.首先轮询注册到reactor线程对用的selector上的所有的channel的IO事件
                    // 标识 selector 上阻塞的线程是否应该被唤醒
                    select(wakenUp.getAndSet(false));
                    // 减少 wakeup 开销
                    if (wakenUp.get()) {
                        // 唤醒 I/O 多路复用器上的阻塞线程
                        // 注册 channel，改变 interestOp，判断超时
                        selector.wakeup();
                    }
                default:
                    // fallthrough
            }
            //2.处理产生网络IO事件的channel
            // (100-ioRatio）/ioRatio
            processSelectedKeys();
            // 3.处理任务队列          
            runAllTasks(...);
            }
        } catch (Throwable t) {
            handleLoopException(t);
        }
        ...
    }
```



### 一、select操作

> reactor线程select步骤做的事情：不断地轮询是否有IO事件发生，并且在轮询的过程中不断检查是否有定时任务和普通任务，保证了netty的任务队列中的任务得到有效执行，轮询过程顺带用一个计数器避开了了jdk空轮询的bug
>
>
>
> 误区：不要认为Selector的select返回值是已准备就绪的Channel的总数，其实它返回的是从上一个select()调用后进入就绪状态的Channel的数量

```java
int selectCnt = 0;
long currentTimeNanos = System.nanoTime();
long selectDeadLineNanos = currentTimeNanos + delayNanos(currentTimeNanos);

for (;;) {
    // 1.定时任务截至事时间快到了，中断本次轮询
    long timeoutMillis = (selectDeadLineNanos - currentTimeNanos + 500000L) / 1000000L;
    if (timeoutMillis <= 0) {
        if (selectCnt == 0) {
            selector.selectNow();
            selectCnt = 1;
        }
        break;
    }
    ....
    // 2.轮询过程中发现有任务加入，中断本次轮询
    if (hasTasks() && wakenUp.compareAndSet(false, true)) {
        selector.selectNow();
        selectCnt = 1;
        break;
    }
    ...
    // 3.阻塞式select操作，select()方法会返回注册的interest的I/O事件已经就绪的那些通道的数目
    int selectedKeys = selector.select(timeoutMillis);
    selectCnt ++;
    /**判断是否中断本次轮询
    轮询到IO事件 （`selectedKeys != 0`）
    oldWakenUp 参数为true
    任务队列里面有任务（`hasTasks`）
    第一个定时任务即将要被执行 （`hasScheduledTasks（）`）
    用户主动唤醒（`wakenUp.get()`）
    */
    if (selectedKeys != 0 || oldWakenUp || wakenUp.get() || hasTasks() || hasScheduledTasks()) {
        break;
    }
    ...
    // Java nio在Linux系统下的epoll空轮询问题
    // 4.解决jdk的nio bug
    long time = System.nanoTime();
    if (time - TimeUnit.MILLISECONDS.toNanos(timeoutMillis) >= currentTimeNanos) {
        selectCnt = 1;
    } else if (SELECTOR_AUTO_REBUILD_THRESHOLD > 0 &&
            selectCnt >= SELECTOR_AUTO_REBUILD_THRESHOLD) {

        rebuildSelector();
        selector = this.selector;
        selector.selectNow();
        selectCnt = 1;
        break;
    }
    currentTimeNanos = time; 
    /**
    该bug会导致Selector一直空轮询，最终导致cpu 100%，nio server不可用，严格意义上来说，netty没有解决jdk的bug，而是通过一种方式来巧妙地避开了这个bug
    */
    ...
}
```

`wakenUp` 表示是否应该唤醒正在阻塞的select操作，可以看到netty在进行一次新的loop之前，都会将`wakeUp` 被设置成false，标志新的一轮loop的开始



netty里面定时任务队列是按照延迟时间从小到大进行排序， `delayNanos(currentTimeNanos)`方法即取出第一个定时任务的延迟时间

```java
protected long delayNanos(long currentTimeNanos) {
    ScheduledFutureTask<?> scheduledTask = peekScheduledTask();
    if (scheduledTask == null) {
        return SCHEDULE_PURGE_INTERVAL;
    }
    return scheduledTask.delayNanos(currentTimeNanos);
 }



@Override
public void execute(Runnable task) { 
    ...
    wakeup(inEventLoop); // inEventLoop为false
    ...
}

protected void wakeup(boolean inEventLoop) {
    if (!inEventLoop && wakenUp.compareAndSet(false, true)) {
        selector.wakeup();
    }
}
```





连接出现了RST，因为poll和epoll对于突然中断的连接socket会对返回的eventSet事件集合置为POLLHUP或者POLLERR，eventSet事件集合发生了变化，这就导致Selector会被唤醒，进而导致CPU 100%问题。根本原因就是JDK没有处理好这种情况，比如SelectionKey中就没定义有异常事件的类型。

> ```dart
> /**
> * 通常是阻塞的，但是在epoll空轮询的bug中，
> * 之前处于连接状态突然被断开，select()的
> * 返回值noOfKeys应该等于0，也就是阻塞状态
> * 但是，在此bug中，select()被唤醒，而又
> * 没有数据传入，导致while (itr.hasNext())
> * 根本不会执行，而后就进入for (;;) {的死循环
> * 但是，正常状态下应该阻塞，也就是只输出一个waiting...
> * 而此时进入死循环，不断的输出waiting...，程序死循环
> * cpu自然很快飙升到100%状态。
> */
> int noOfKeys = selector.select();
> ```



如果持续的时间大于等于timeoutMillis，说明就是一次有效的轮询，重置`selectCnt`标志，否则，表明该阻塞方法并没有阻塞这么长时间，可能触发了jdk的空轮询bug，当空轮询的次数超过一个阀值的时候，默认是512，就开始重建selector

```java
int selectorAutoRebuildThreshold = SystemPropertyUtil.getInt("io.netty.selectorAutoRebuildThreshold", 512);
if (selectorAutoRebuildThreshold < MIN_PREMATURE_SELECTOR_RETURNS) {
    selectorAutoRebuildThreshold = 0;
}
SELECTOR_AUTO_REBUILD_THRESHOLD = selectorAutoRebuildThreshold;

public void rebuildSelector() {
    final Selector oldSelector = selector;
    final Selector newSelector;
    newSelector = openSelector();

    int nChannels = 0;
     try {
        for (;;) {
                for (SelectionKey key: oldSelector.keys()) {
                    Object a = key.attachment();
                     if (!key.isValid() || key.channel().keyFor(newSelector) != null) {
                         continue;
                     }
                     int interestOps = key.interestOps();
                     key.cancel();
                     SelectionKey newKey = key.channel().register(newSelector, interestOps, a);
                     if (a instanceof AbstractNioChannel) {
                         ((AbstractNioChannel) a).selectionKey = newKey;
                      }
                     nChannels ++;
                }
                break;
        }
    } catch (ConcurrentModificationException e) {
        // Probably due to concurrent modification of the key set.
        continue;
    }
    selector = newSelector;
    oldSelector.close();
}
```



```java
private SelectedSelectionKeySet selectedKeys;

private Selector NioEventLoop.openSelector() {
    //...
    final SelectedSelectionKeySet selectedKeySet = new SelectedSelectionKeySet();
    // selectorImplClass -> sun.nio.ch.SelectorImpl
    Field selectedKeysField = selectorImplClass.getDeclaredField("selectedKeys");
    Field publicSelectedKeysField = selectorImplClass.getDeclaredField("publicSelectedKeys");
    selectedKeysField.setAccessible(true);
    publicSelectedKeysField.setAccessible(true);
    selectedKeysField.set(selector, selectedKeySet);
    publicSelectedKeysField.set(selector, selectedKeySet);
    //...
    selectedKeys = selectedKeySet;
}
```



### 二、processSelectedKeys()

> 基于 4.1.6 版本，4.1.9后会不同
>
> netty的reactor线程第二步做的事情为处理IO事件，netty使用数组替换掉jdk原生的HashSet来保证IO事件的高效处理，每个SelectionKey上绑定了netty类`AbstractChannel`对象作为attachment，在处理每个SelectionKey的时候，就可以找到`AbstractChannel`，然后通过pipeline的方式将处理串行到ChannelHandler，回调到用户方法

```java
// netty有两种选择，从名字上看，一种是处理优化过的selectedKeys，一种是正常的处理
private void processSelectedKeys() {
    if (selectedKeys != null) {
        processSelectedKeysOptimized(selectedKeys.flip());
    } else {
        processSelectedKeysPlain(selector.selectedKeys());
    }
}

private void processSelectedKeys() {
        if (selectedKeys != null) {
            processSelectedKeysOptimized();
        } else {
            processSelectedKeysPlain(selector.selectedKeys());
        }
    }
```

SelectedKeys是一个 `SelectedSelectionKeySet` 类对象，在`NioEventLoop` 的 `openSelector` 方法中创建，之后就通过反射将selectedKeys与 `sun.nio.ch.SelectorImpl` 中的两个field绑定

> `sun.nio.ch.SelectorImpl` 中我们可以看到，这两个field其实是两个HashSet

```java
// Public views of the key sets
private Set<SelectionKey> publicKeys;             // Immutable
private Set<SelectionKey> publicSelectedKeys;     // Removal allowed, but not addition
protected SelectorImpl(SelectorProvider sp) {
    super(sp);
    keys = new HashSet<SelectionKey>();
    selectedKeys = new HashSet<SelectionKey>();
    if (Util.atBugLevel("1.4")) {
        publicKeys = keys;
        publicSelectedKeys = selectedKeys;
    } else {
        publicKeys = Collections.unmodifiableSet(keys);
        publicSelectedKeys = Util.ungrowableSet(selectedKeys);
    }
}  
```



#### SelectedSelectionKeySet

> 在4.1.9.Final 版本中，netty已经将`SelectedSelectionKeySet.java` 底层使用一个数组了

```java
// 
final class SelectedSelectionKeySet extends AbstractSet<SelectionKey> {

    private SelectionKey[] keysA;
    private int keysASize;
    private SelectionKey[] keysB;
    private int keysBSize;
    private boolean isA = true;

    SelectedSelectionKeySet() {
        keysA = new SelectionKey[1024];
        keysB = keysA.clone();
    }

    @Override
    public boolean add(SelectionKey o) {
        if (o == null) {
            return false;
        }

        if (isA) {
            int size = keysASize;
            keysA[size ++] = o;
            keysASize = size;
            if (size == keysA.length) {
                doubleCapacityA();
            }
        } else {
            int size = keysBSize;
            keysB[size ++] = o;
            keysBSize = size;
            if (size == keysB.length) {
                doubleCapacityB();
            }
        }

        return true;
    }

    private void doubleCapacityA() {
        SelectionKey[] newKeysA = new SelectionKey[keysA.length << 1];
        System.arraycopy(keysA, 0, newKeysA, 0, keysASize);
        keysA = newKeysA;
    }

    private void doubleCapacityB() {
        SelectionKey[] newKeysB = new SelectionKey[keysB.length << 1];
        System.arraycopy(keysB, 0, newKeysB, 0, keysBSize);
        keysB = newKeysB;
    }

    SelectionKey[] flip() {
        if (isA) {
            isA = false;
            keysA[keysASize] = null;
            keysBSize = 0;
            return keysA;
        } else {
            isA = true;
            keysB[keysBSize] = null;
            keysASize = 0;
            return keysB;
        }
    }

    @Override
    public int size() {
        if (isA) {
            return keysASize;
        } else {
            return keysBSize;
        }
    }

    @Override
    public boolean remove(Object o) {
        return false;
    }

    @Override
    public boolean contains(Object o) {
        return false;
    }

    @Override
    public Iterator<SelectionKey> iterator() {
        throw new UnsupportedOperationException();
    }
}


// 
final class SelectedSelectionKeySet extends AbstractSet<SelectionKey> {

    SelectionKey[] keys;
    int size;

    SelectedSelectionKeySet() {
        keys = new SelectionKey[1024];
    }

    @Override
    public boolean add(SelectionKey o) {
        if (o == null) {
            return false;
        }

        keys[size++] = o;
        if (size == keys.length) {
            increaseCapacity();
        }

        return true;
    }

    @Override
    public boolean remove(Object o) {
        return false;
    }

    @Override
    public boolean contains(Object o) {
        return false;
    }

    @Override
    public int size() {
        return size;
    }

    @Override
    public Iterator<SelectionKey> iterator() {
        return new Iterator<SelectionKey>() {
            private int idx;

            @Override
            public boolean hasNext() {
                return idx < size;
            }

            @Override
            public SelectionKey next() {
                if (!hasNext()) {
                    throw new NoSuchElementException();
                }
                return keys[idx++];
            }

            @Override
            public void remove() {
                throw new UnsupportedOperationException();
            }
        };
    }

    void reset() {
        reset(0);
    }

    void reset(int start) {
        Arrays.fill(keys, start, size, null);
        size = 0;
    }

    private void increaseCapacity() {
        SelectionKey[] newKeys = new SelectionKey[keys.length << 1];
        System.arraycopy(keys, 0, newKeys, 0, size);
        keys = newKeys;
    }
}

```



#### processSelectedKeysOptimized

```java
 private void processSelectedKeysOptimized(SelectionKey[] selectedKeys) {
     for (int i = 0;; i ++) {
         // 1.取出IO事件以及对应的channel
         final SelectionKey k = selectedKeys[i];
         if (k == null) {
             break;
         }
         // 尽快 gc 回收，避免内存泄漏
         selectedKeys[i] = null;
         // attachment 是什么？
         final Object a = k.attachment();
         // 2.处理该channel
         if (a instanceof AbstractNioChannel) {
             processSelectedKey(k, (AbstractNioChannel) a);
         } else {
             NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
             processSelectedKey(k, task);
         }
         // 3.判断是否该再来次轮询
         if (needsToSelectAgain) {
             for (;;) {
                 i++;
                 if (selectedKeys[i] == null) {
                     break;
                 }
                 selectedKeys[i] = null;
             }
             selectAgain();
             selectedKeys = this.selectedKeys.flip();
             i = -1;
         }
     }
 }
// 4.1.9+
  private void processSelectedKeysOptimized() {
        for (int i = 0; i < selectedKeys.size; ++i) {
            final SelectionKey k = selectedKeys.keys[i];
            // null out entry in the array to allow to have it GC'ed once the Channel close
            // See https://github.com/netty/netty/issues/2363
            selectedKeys.keys[i] = null;

            final Object a = k.attachment();

            if (a instanceof AbstractNioChannel) {
                processSelectedKey(k, (AbstractNioChannel) a);
            } else {
                @SuppressWarnings("unchecked")
                NioTask<SelectableChannel> task = (NioTask<SelectableChannel>) a;
                processSelectedKey(k, task);
            }
            
            // 什么时候 needsToSelectAgain == true？默认 false
            if (needsToSelectAgain) {
                // null out entries in the array to allow to have it GC'ed once the Channel close
                // See https://github.com/netty/netty/issues/2363
                selectedKeys.reset(i + 1);

                selectAgain();
                i = -1;
            }
            /**
            void cancel(SelectionKey key) {
                key.cancel();
                cancelledKeys ++;
                if (cancelledKeys >= CLEANUP_INTERVAL) {
                    cancelledKeys = 0;
                    needsToSelectAgain = true;
                }
            }
            对于每个NioEventLoop而言，每隔256个channel从selector上移除的时候，就标记 needsToSelectAgain 为true，我们还是跳回到上面这段代码
            */
        }
    }

private void selectAgain() {
    needsToSelectAgain = false;
    try {
        selector.selectNow();
    } catch (Throwable t) {
        logger.warn("Failed to update SelectionKeys.", t);
    }
}

```

遍历保存客户端新Channel的集合——readBuf，然后将每个新连接传播出去——调用pipeline.fireChannelRead()，将每条新连接沿着服务端Channel的pipeline传递，交给Channel后续的入站handler

###### processSelectedKey

```java

// reactor 线程驱动
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
        final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
        if (!k.isValid()) {
            final EventLoop eventLoop;
            try {
                eventLoop = ch.eventLoop();
            } catch (Throwable ignored) {
                // If the channel implementation throws an exception because there is no event loop, we ignore this
                // because we are only trying to determine if ch is registered to this event loop and thus has authority
                // to close ch.
                return;
            }
            // Only close ch if ch is still registered to this EventLoop. ch could have deregistered from the event loop
            // and thus the SelectionKey could be cancelled as part of the deregistration process, but the channel is
            // still healthy and should not be closed.
            // See https://github.com/netty/netty/issues/5125
            if (eventLoop != this || eventLoop == null) {
                return;
            }
            // close the channel if the key is not valid anymore
            unsafe.close(unsafe.voidPromise());
            return;
        }

        try {
            // 通道已准备就绪的操作的集合
            int readyOps = k.readyOps();
            // We first need to call finishConnect() before try to trigger a read(...) or write(...) as otherwise
            // the NIO JDK channel implementation may throw a NotYetConnectedException.
            // OP_CONNECT事件的处理
            // 轮询到OP_CONNECT后，及时删除该key，避免该Channel一直处于connect就绪，导致CPU空轮询
            if ((readyOps & SelectionKey.OP_CONNECT) != 0) {
                // remove OP_CONNECT as otherwise Selector.select(..) will always return without blocking
                // See https://github.com/netty/netty/issues/924
                int ops = k.interestOps();
                ops &= ~SelectionKey.OP_CONNECT; // 取反 相与 =0
                k.interestOps(ops);
                // 调用了该finishConnect方法才意味着一条TCP连接彻底建立完成了。
                unsafe.finishConnect();
            }

            // Process OP_WRITE first as we may be able to write some queued buffers and so free memory.
            if ((readyOps & SelectionKey.OP_WRITE) != 0) {
                // Call forceFlush which will also take care of clear the OP_WRITE once there is nothing left to write
                ch.unsafe().forceFlush();
            }

            // Also check for readOps of 0 to workaround possible JDK bug which may otherwise lead
            // to a spin loop
            if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
                // 若是服务端，accept后 就会读操作
                unsafe.read();
            }
        } catch (CancelledKeyException ignored) {
            unsafe.close(unsafe.voidPromise());
        }
    }
```



##### AbstractNioChannel

> netty的轮询注册机制其实是将`AbstractNioChannel`内部的jdk类`SelectableChannel`对象注册到jdk类`Selctor`对象上去，并且将`AbstractNioChannel`作为`SelectableChannel`对象的一个attachment附属上，这样再jdk轮询出某条`SelectableChannel`有IO事件发生时，就可以直接取出`AbstractNioChannel`进行后续操作

```java
protected void doRegister() throws Exception {
    // ...
    selectionKey = javaChannel().register(eventLoop().selector, 0, this);
    // ...
}
protected SelectableChannel javaChannel() {
    return ch;
}
```

##### NioTask

```java
public interface NioTask<C extends SelectableChannel> {
    void channelReady(C ch, SelectionKey key) throws Exception;
    void channelUnregistered(C ch, Throwable cause) throws Exception;
}
```



### 三、reactor线程task的调度

#####  用户自定义普通任务

```java

ctx.channel().eventLoop().execute(new Runnable() {
    @Override
    public void run() {
        //...
    }
});

@Override
public void execute(Runnable task) {
    //...
    addTask(task);
    //...
}

protected void addTask(Runnable task) {
    // ...
    if (!offerTask(task)) {
        // RejectedExecutionHandler 
        reject(task);
    }
}

final boolean offerTask(Runnable task) {
    // ...
    return taskQueue.offer(task);
}

//taskQueue 定义的地方和被初始化的地方
private final Queue<Runnable> taskQueue;


taskQueue = newTaskQueue(this.maxPendingTasks);

@Override
protected Queue<Runnable> newTaskQueue(int maxPendingTasks) {
    // This event loop never calls takeTask()
    return PlatformDependent.newMpscQueue(maxPendingTasks);
}
/**
taskQueue在NioEventLoop中默认是mpsc队列，mpsc队列，即多生产者单消费者队列，netty使用mpsc，方便的将外部线程的task聚集，在reactor线程内部用单线程来串行执行，我们可以借鉴netty的任务执行模式来处理类似多线程数据上报，定时聚合的应用
*/
```

##### 非当前reactor线程调用channel的各种方法

```java
// non reactor thread
channel.write(...)
```

###### AbstractChannelHandlerContext

```java
private void write(Object msg, boolean flush, ChannelPromise promise) {
    // ...
    EventExecutor executor = next.executor();
    /**
    外部线程在调用write的时候，executor.inEventLoop()会返回false，直接进入到else分支，
    将write封装成一个WriteTask（这里仅仅是write而没有flush，因此flush参数为false）, 
    然后调用 safeExecute方法
    */
    if (executor.inEventLoop()) {
        if (flush) {
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise);
        }
    } else {
        AbstractWriteTask task;
        if (flush) {
            task = WriteAndFlushTask.newInstance(next, m, promise);
        }  else {
            task = WriteTask.newInstance(next, m, promise);
        }
        safeExecute(executor, task, promise, m);
    }
}

private static void safeExecute(EventExecutor executor, Runnable runnable, ChannelPromise promise, Object msg) {
    // ...
    executor.execute(runnable);
    // ...
}
```

接下来的调用链就进入到第一种场景了，但是和第一种场景有个明显的区别就是，第一种场景的调用链的发起线程是reactor线程，第二种场景的调用链的发起线程是用户线程，用户线程可能会有很多个，显然多个线程并发写`taskQueue`可能出现线程同步问题，于是，这种场景下，netty的mpsc queue就有了用武之地



##### 用户自定义定时任务

```java
ctx.channel().eventLoop().schedule(new Runnable() {
    @Override
    public void run() {

    }
}, 60, TimeUnit.SECONDS);


public ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit) {
//...
    return schedule(new ScheduledFutureTask<Void>(
            this, command, null, ScheduledFutureTask.deadlineNanos(unit.toNanos(delay))));
} 
// 通过 ScheduledFutureTask, 将用户自定义任务再次包装成一个netty内部的任务
<V> ScheduledFuture<V> schedule(final ScheduledFutureTask<V> task) {
    // ...
    scheduledTaskQueue().add(task);
    // ...
    return task;
}

Queue<ScheduledFutureTask<?>> scheduledTaskQueue() {
    if (scheduledTaskQueue == null) {
        scheduledTaskQueue = new PriorityQueue<ScheduledFutureTask<?>>();
    }
    return scheduledTaskQueue;
}
```

在非定时任务的处理中，netty通过一个mpsc队列将任务落地

果不其然，`scheduledTaskQueue()` 方法，会返回一个优先级队列，然后调用 `add` 方法将定时任务加入到队列中去，但是，这里为什么要使用优先级队列，而不需要考虑多线程的并发？

因为我们现在讨论的场景，调用链的发起方是reactor线程，不会存在多线程并发这些问题

但是，万一有的用户在reactor之外执行定时任务呢？虽然这类场景很少见，但是netty作为一个无比健壮的高性能io框架，必须要考虑到这种情况。

对此，netty的处理是，如果是在外部线程调用schedule，netty将添加定时任务的逻辑封装成一个普通的task，这个task的任务是添加[添加定时任务]的任务，而不是添加定时任务，其实也就是第二种场景，这样，对 `PriorityQueue`的访问就变成单线程，即只有reactor线程

> 完整的schedule方法

```java
<V> ScheduledFuture<V> schedule(final ScheduledFutureTask<V> task) {
    if (inEventLoop()) {
        scheduledTaskQueue().add(task);
    } else {
        // 进入到场景二，进一步封装任务
        execute(new Runnable() {
            @Override
            public void run() {
                scheduledTaskQueue().add(task);
            }
        });
    }
    return task;
}
```

在阅读源码细节的过程中，我们应该多问几个为什么？这样会有利于看源码的时候不至于犯困！比如这里，为什么定时任务要保存在优先级队列中，我们可以先不看源码，来思考一下优先级对列的特性

优先级队列按一定的顺序来排列内部元素，内部元素必须是可以比较的，联系到这里每个元素都是定时任务，那就说明定时任务是可以比较的，那么到底有哪些地方可以比较？

每个任务都有一个下一次执行的截止时间，截止时间是可以比较的，截止时间相同的情况下，任务添加的顺序也是可以比较的，就像这样，阅读源码的过程中，一定要多和自己对话，多问几个为什么

```java
final class ScheduledFutureTask<V> extends PromiseTask<V> implements ScheduledFuture<V> {
    private static final AtomicLong nextTaskId = new AtomicLong();
    private static final long START_TIME = System.nanoTime();

    static long nanoTime() {
        return System.nanoTime() - START_TIME;
    }

    private final long id = nextTaskId.getAndIncrement();
    /* 0 - no repeat, >0 - repeat at fixed rate, <0 - repeat with fixed delay */
    private final long periodNanos;

    @Override
    public int compareTo(Delayed o) {
        //...
    }

    // 精简过的代码
    @Override
    public void run() {
    }
    

    
public int compareTo(Delayed o) {
    if (this == o) {
        return 0;
    }

    ScheduledFutureTask<?> that = (ScheduledFutureTask<?>) o;
    long d = deadlineNanos() - that.deadlineNanos();
    if (d < 0) {
        return -1;
    } else if (d > 0) {
        return 1;
    } else if (id < that.id) {
        return -1;
    } else if (id == that.id) {
        throw new Error();
    } else {
        return 1;
    }
}
```

先比较任务的截止时间，截止时间相同的情况下，再比较id，即任务添加的顺序，如果id再相同的话，就抛Error

这样，在执行定时任务的时候，就能保证最近截止时间的任务先执行

下面，我们再来看下netty是如何来保证各种定时任务的执行的，netty里面的定时任务分以下三种

1.若干时间后执行一次
2.每隔一段时间执行一次
3.每次执行结束，隔一定时间再执行一次

netty使用一个 `periodNanos` 来区分这三种情况

```java
/* 0 - no repeat, >0 - repeat at fixed rate, <0 - repeat with fixed delay */
private final long periodNanos;


public void run() {
    if (periodNanos == 0) {
        V result = task.call();
        setSuccessInternal(result);
    } else { 
        task.call();
        long p = periodNanos;
        if (p > 0) {
            deadlineNanos += p;
        } else {
            deadlineNanos = nanoTime() - p;
        }
            scheduledTaskQueue.add(this);
        }
    }
}
```

`if (periodNanos == 0)` 对应 `若干时间后执行一次` 的定时任务类型，执行完了该任务就结束了。

否则，进入到else代码块，先执行任务，然后再区分是哪种类型的任务，`periodNanos`大于0，表示是以固定频率执行某个任务，和任务的持续时间无关，然后，设置该任务的下一次截止时间为本次的截止时间加上间隔时间`periodNanos`，否则，就是每次任务执行完毕之后，间隔多长时间之后再次执行，截止时间为当前时间加上间隔时间，`-p`就表示加上一个正的间隔时间，最后，将当前任务对象再次加入到队列，实现任务的定时执行



#### runAllTasks(long timeoutNanos)

如何添加，如何执行？时机？

runAllTasks(long timeoutNanos)

尽量在一定的时间内，将所有的任务都取出来run一遍。`timeoutNanos` 表示该方法最多执行这么长时间，netty为什么要这么做？我们可以想一想，reactor线程如果在此停留的时间过长，那么将积攒许多的IO事件无法处理(见reactor线程的前面两个步骤)，最终导致大量客户端请求阻塞，因此，默认情况下，netty将控制内部队列的执行时间



```java
    // SingleThreadEventExecutor#runAllTasks
    protected boolean runAllTasks(long timeoutNanos) {
        // 从scheduledTaskQueue转移定时任务到taskQueue(mpsc queue)  
        fetchFromScheduledTaskQueue();
        Runnable task = pollTask();
        if (task == null) {
            afterRunningAllTasks();
            return false;
        }
        /**
        这一步将取出第一个任务，用reactor线程传入的超时时间 timeoutNanos 
        来计算出当前任务循环的deadline，并且使用了runTasks，lastExecutionTime
        来时刻记录任务的状态
        */
        // 计算本次任务循环的截止时间
        final long deadline = ScheduledFutureTask.nanoTime() + timeoutNanos;
        long runTasks = 0;
        long lastExecutionTime;
        
        // 循环执行任务
        /**
        调用`safeExecute`来确保任务安全执行，`忽略任何异常`,然后将已运行任务 runTasks 加一，
        每隔0x3F任务，即每执行完64个任务之后，判断当前时间是否超过本次reactor任务循环的截止时间
        了，如果超过，那就break掉，如果没有超过，那就继续执行。可以看到，netty对性能的优化考虑地相
        当的周到，假设netty任务队列里面如果有海量小任务，如果每次都要执行完任务都要判断一下是否到截
        止时间，那么效率是比较低下的
        */
        for (;;) {
            safeExecute(task);

            runTasks ++;

            // Check timeout every 64 tasks because nanoTime() is relatively expensive.
            // XXX: Hard-coded value - will make it configurable if it is really a problem.
            if ((runTasks & 0x3F) == 0) {
                lastExecutionTime = ScheduledFutureTask.nanoTime();
                if (lastExecutionTime >= deadline) {
                    break;
                }
            }

            task = pollTask();
            if (task == null) {
                lastExecutionTime = ScheduledFutureTask.nanoTime();
                break;
            }
        }
        // 收尾工作
        afterRunningAllTasks();
        this.lastExecutionTime = lastExecutionTime;
        return true;
    }
```



##### fetchFromScheduledTaskQueue

netty在把任务从scheduledTaskQueue转移到taskQueue的时候还是非常小心的，当taskQueue无法offer的时候，需要把从scheduledTaskQueue里面取出来的任务重新添加回去

从scheduledTaskQueue从拉取一个定时任务的逻辑如下，传入的参数`nanoTime`为当前时间(其实是当前纳秒减去`ScheduledFutureTask`类被加载的纳秒个数)



```java
private boolean fetchFromScheduledTaskQueue() {
    long nanoTime = AbstractScheduledEventExecutor.nanoTime();
    Runnable scheduledTask  = pollScheduledTask(nanoTime);
    while (scheduledTask != null) {
        if (!taskQueue.offer(scheduledTask)) {
            // No space left in the task queue add it back to the scheduledTaskQueue so we pick it up again.
            scheduledTaskQueue().add((ScheduledFutureTask<?>) scheduledTask);
            return false;
        }
        scheduledTask  = pollScheduledTask(nanoTime);
    }
    return true;
}


protected final Runnable pollScheduledTask(long nanoTime) {
    assert inEventLoop();

    Queue<ScheduledFutureTask<?>> scheduledTaskQueue = this.scheduledTaskQueue;
    ScheduledFutureTask<?> scheduledTask = scheduledTaskQueue == null ? null : scheduledTaskQueue.peek();
    if (scheduledTask == null) {
        return null;
    }

    if (scheduledTask.deadlineNanos() <= nanoTime) {
        scheduledTaskQueue.remove();
        return scheduledTask;
    }
    return null;
}
```



##### safeExecute

调用`safeExecute`来确保任务安全执行，`忽略任何异常`

```java
// AbstractEventExecutor
protected static void safeExecute(Runnable task) {
    try {
        task.run();
    } catch (Throwable t) {
        logger.warn("A task raised an exception. Task: {}", task, t);
    }
}
```



##### afterRunningAllTasks

```java
@Override
protected void afterRunningAllTasks() {
        runAllTasksFrom(tailTasks);
}
```

`NioEventLoop`可以通过父类`SingleTheadEventLoop`的`executeAfterEventLoopIteration`方法向`tailTasks`中添加收尾任务，比如，你想统计一下一次执行一次任务循环花了多长时间就可以调用此方法

```java
public final void executeAfterEventLoopIteration(Runnable task) {
        // ...
        if (!tailTasks.offer(task)) {
            reject(task);
        }
        //...
}
```



### Summary

* 当前reactor线程调用当前eventLoop执行任务，直接执行，否则，添加到任务队列稍后执行
* netty内部的任务分为普通任务和定时任务，分别落地到MpscQueue和PriorityQueue
* netty每次执行任务循环之前，会将已经到期的定时任务从PriorityQueue转移到MpscQueue
* netty每隔64个任务检查一下是否该退出任务循环

NioEventLoopGroup+NioEventLoop

> isSharable -> handlerAdded -> channelRegistered -> 
>
> userEventTriggered -> channelActive -> channelReadComplete

第一步是轮询Channel，一次轮询完毕，要么对轮询出的I/O事件进行处理；要么对MPSCQ里的任务进行处理，针对前者其处理方法是processSelectedKeys()，后者的方法是runAllTasks()



Selector的selectedKeys和publicSelectedKeys的数据结构是HashSet



Netty的入站和出站事件,入站事件从其头部节点（本质是出站handler）开始传播，直到尾部节点这个入站handler结束，反之，出站事件从尾部开始传播，直到头部这个出站handler结束

- 入站事件指的是用户被动感知的事件，一般由Netty的I/O线程发起，在Netty框架外面看，就是Netty被动响应了一些回调方法，是无需用户主动调用的，因此可以让业务层实现一些拦截操作，可以通过继承入站handler来自定义自己的入站处理器

  > ```
  > channelRegistered：注册I/O多路复用器成功的事件，channel注册到EventLoop上后调用，例如服务开始启动时会自动调用pipeline.fireChannelRegistered();
  > channelUnregistered：注销注册事件，channel从EventLoop上注销后调用，例如关闭连接成功后自动调用pipeline.fireChannelUnregistered();
  > channelActive：连接激活事件，表明当前Channel已经被打开且已经连接到对端
  > channelInactive：连接非激活事件，连接关闭后调用，pipeline.fireChannelInactive();
  > channelRead：连接中有数据可读的事件，channel有数据可读时调用，pipeline.fireChannelRead();
  > channelReadComplete：读完事件，channel读完之后调用，pipeline.fireChannelReadComplete();注意该事件的特殊性，后续专门总结
  > channelWritabilityChanged：连接可写状态变更事件，当一个Channel的可写状态发生改变时执行，可以保证写的操作不要太快，防止OOM，pipeline.fireChannelWritabilityChanged();
  > userEventTriggered：用户自定义的事件触发，例如心跳检测，ctx.fireUserEventTriggered(evt);
  > exceptionCaught：用于出、入站内发生的异常事件的捕获，方便业务层面处理异常
  > ```

- 出站事件一般指的是用户主动发起的事件，由用户线程（或者用户主动驱动I/O线程）主动发起，在Netty框架外面看，就是用户主动执行一些方法，需要用户主动配置或者使用，可以通过继承出站handler实现自己的出站处理器

  > ```
  > bind事件,服务端主动绑定端口
  > close事件，主动关闭channel自身
  > connect事件，用于客户端主动连接一个远程机器
  > disconnect事件，用于客户端主动关闭远程连接
  > deregister事件，用于客户端，在执行断开连接disconnect操作后调用，将channel从EventLoop中注销
  > read事件，用于新接入连接时，注册I/O多路复用器后，修改监听为OP_READ
  > write事件，向通道写数据，但是并不刷新缓存
  > flush事件，将通道排队的数据刷新到连接上
  > ```

- 所谓的出、入，不要和方向上的出入强关联，这容易混乱，只需知道入站事件代表I/O线程执行的被动触发的回调方法，即被动响应，出站事件代表用户主动调用的方法，其命名和什么方向无关





#### writeAndFlush

```java
    // AbstractChannel
    @Override
    public boolean isWritable() {
        ChannelOutboundBuffer buf = unsafe.outboundBuffer();
        return buf != null && buf.isWritable();
    }
// ChannelOutboundBuffer
private void incrementPendingOutboundBytes(long size, boolean invokeLater) {
        if (size == 0) {
            return;
        }

        long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, size);
        if (newWriteBufferSize >= channel.config().getWriteBufferHighWaterMark()) {
            setUnwritable(invokeLater);
        }
    }
/**
channel.config().getWriteBufferHighWaterMark()返回的field是ChannelConfig里面对应的writeBufferHighWaterMark，可以看到，默认值为64K, 表示如果你在写之前调用调用isWriteable方法，netty最多给你缓存64K的数据, 否则，缓存就一直膨胀
*/
// DefaultChannelConfig.java
private volatile int writeBufferHighWaterMark = 64 * 1024;

// AbstractChannel
    public final void close(final ChannelPromise promise) {
            if (!promise.setUncancellable()) {
                return;
            }

            if (outboundBuffer == null) {
                // Only needed if no VoidChannelPromise.
                if (!(promise instanceof VoidChannelPromise)) {
                    // This means close() was called before so we just register a listener and return
                    closeFuture.addListener(new ChannelFutureListener() {
                        @Override
                        public void operationComplete(ChannelFuture future) throws Exception {
                            promise.setSuccess();
                        }
                    });
                }
                return;
            }

            if (closeFuture.isDone()) {
                // Closed already.
                safeSetSuccess(promise);
                return;
            }

            final boolean wasActive = isActive();
            final ChannelOutboundBuffer buffer = outboundBuffer;
            outboundBuffer = null; // Disallow adding any messages and flushes to outboundBuffer.
            Executor closeExecutor = closeExecutor();
            if (closeExecutor != null) {
                closeExecutor.execute(new OneTimeTask() {
                    @Override
                    public void run() {
                        try {
                            // Execute the close.
                            doClose0(promise);
                        } finally {
                            // Call invokeLater so closeAndDeregister is executed in the EventLoop again!
                            invokeLater(new OneTimeTask() {
                                @Override
                                public void run() {
                                    // Fail all the queued messages
                                    buffer.failFlushed(CLOSED_CHANNEL_EXCEPTION, false);
                                    buffer.close(CLOSED_CHANNEL_EXCEPTION);
                                    fireChannelInactiveAndDeregister(wasActive);
                                }
                            });
                        }
                    }
                });
            } else {
                try {
                    // Close the channel and fail the queued messages in all cases.
                    doClose0(promise);
                } finally {
                    // Fail all the queued messages.
                    buffer.failFlushed(CLOSED_CHANNEL_EXCEPTION, false);
                    buffer.close(CLOSED_CHANNEL_EXCEPTION);
                }
                if (inFlush0) {
                    invokeLater(new OneTimeTask() {
                        @Override
                        public void run() {
                            fireChannelInactiveAndDeregister(wasActive);
                        }
                    });
                } else {
                    fireChannelInactiveAndDeregister(wasActive);
                }
            }
        }

void failFlushed(Throwable cause, boolean notify) {
        // Make sure that this method does not reenter.  A listener added to the current promise can be notified by the
        // current thread in the tryFailure() call of the loop below, and the listener can trigger another fail() call
        // indirectly (usually by closing the channel.)
        //
        // See https://github.com/netty/netty/issues/1501
        if (inFail) {
            return;
        }

        try {
            inFail = true;
            for (;;) {
                if (!remove0(cause, notify)) {
                    break;
                }
            }
        } finally {
            inFail = false;
        }
    }

private boolean remove0(Throwable cause, boolean notifyWritability) {
        Entry e = flushedEntry;
        if (e == null) {
            clearNioBuffers();
            return false;
        }
        Object msg = e.msg;

        ChannelPromise promise = e.promise;
        int size = e.pendingSize;

        removeEntry(e);

        if (!e.cancelled) {
            // only release message, fail and decrement if it was not canceled before.
            ReferenceCountUtil.safeRelease(msg);

            safeFail(promise, cause);
            decrementPendingOutboundBytes(size, false, notifyWritability);
        }

        // recycle the entry
        e.recycle();

        return true;
    }

private void removeEntry(Entry e) {
        if (-- flushed == 0) {
            // processed everything
            flushedEntry = null;
            if (e == tailEntry) {
                tailEntry = null;
                unflushedEntry = null;
            }
        } else {
            // 指针指向下一个待删除的缓存
            flushedEntry = e.next;
        }
    }

/**
所有的缓存对象都被remove掉,remove0 每调用一次都会调用一次safeFail(promise, cause)方法，
*/
private static void safeFail(ChannelPromise promise, Throwable cause) {
        if (!(promise instanceof VoidChannelPromise) && !promise.tryFailure(cause)) {
            logger.warn("Failed to mark a promise as failure because it's done already: {}", promise, cause);
        }
    }

// DefaultPromise
@Override
    public boolean tryFailure(Throwable cause) {
        if (setFailure0(cause)) {
            notifyListeners();
            return true;
        }
        return false;
    }

static void notifyListener0(Future future, GenericFutureListener l) {
        try {
            l.operationComplete(future);
        } catch (Throwable t) {
            if (logger.isWarnEnabled()) {
                logger.warn("An exception was thrown by " + l.getClass().getName() + ".operationComplete()", t);
            }
        }
    }

@Override
    protected void channelRead0(ChannelHandlerContext ctx, TransferFromSdkDataPacket msg) throws Exception {
        TransferToPushServerDataPacket dataPacket = new TransferToPushServerDataPacket();

        dataPacket.setVersion(Constants.PUSH_SERVER_VERSION);
        dataPacket.setData(msg.getData());
        dataPacket.setConnectionId(ctx.channel().attr(AttributeKeys.CONNECTION_ID).get());

        final long startTime = System.nanoTime();

        try {
            ChannelGroupFuture channelFutures = pushServerChannels.writeAndFlushRandom(dataPacket);
            channelFutures.addListener(new GenericFutureListener<Future<? super Void>>() {
                @Override
                public void operationComplete(Future<? super Void> future) throws Exception {
                    if (future.isSuccess()) {
                        CatUtil.logTransaction(startTime, null, CatTransactions.TransferToPushServer, CatTransactions.TransferToPushServer);
                    } else {
                        Channel channel = (Channel) ((ChannelGroupFuture) future).group().toArray()[0];
                        final String pushServer = channel.remoteAddress().toString();
                        TransferToPushServerException e = new TransferToPushServerException(String.format("pushServer: %s", pushServer), future.cause());

                        CatUtil.logTransaction(new CatUtil.CatTransactionCallBack() {
                            @Override
                            protected void beforeComplete() {
                                Cat.logEvent(CatEvents.WriteToPushServerError, pushServer);
                            }
                        }, startTime, e, CatTransactions.TransferToPushServer, CatTransactions.TransferToPushServer);
                    }
                }

            });
        } catch (Exception e) {
            CatUtil.logTransaction(startTime, e, CatTransactions.TransferToPushServer, CatTransactions.TransferToPushServer);
        }
    }
```

