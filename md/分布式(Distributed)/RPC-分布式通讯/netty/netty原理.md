# NIO(底层知识)



![工作过程](E:/Repository/git/learning-blog/md/image/Netty.png)

RPC：什么情况下才需要建立连接？服务调用时，代理对象获取（未缓存）

代理，调用RpcXXX接口中每一个方法的时候，实际上就是发起一次网络请求

一般想法：

请求信息：类名，方法名，参数类型，参数

响应信息：方法执行结果



# Netty

## 高性能的关键

* 协议：数据格式，编码格式
* IO 传输通道：
* 线程：Reactor 模型（单线程、多线程、主从多线程）



epoll模型设计思想（boss 线程 管理  N个 worker 线程）

线程池

handleAccept

handleRead



* 异步非阻塞通讯：基于 NIO 的  异步非阻塞通讯框架，加了 线程池

* 零拷贝：利用直接内存

* 内存池 ：ByteBuf，组合Buffer对象

* 无锁化的串行设计理念： pipeline#addlast(handler)
  线程Handler

![串行化处理](../../../image/netty-serial.png)

高效并发编程、

高性能序列化框架：protobuf

* 灵活 TCP 参数：SO_RCVBUF，SO_SNDBUF
  OPTION 参数

单个通道下所有的业务逻辑处理就是有序的，可控的（每一个逻辑就是一个线程）
多通道之间互不干扰，并行



文件传输 transferTo 方法，直接传避免 write 造成的内存拷贝



xxx0 实现类方法，而非接口方法





> * server 端 配置信息
>
> EventLoopGroup （NioEventLoopGroup）
>
> ServerBootstrap#group#
>
> channel(NioServerSocketChannel.class)#        // 类似 IOC 创建 channel
>
> option(ChannelOption.xxxx)#
>
> childhandler( new ChannelInitializer<SocketChannel>(){...})#
>
> childOption
>
>
>
> ServerBootstrap#bind
>
>
>
> * 客户端配置信息
>
> EventLoopGroup （NioEventLoopGroup）
>
> Bootstrap#group#
>
> channel(NioSocketChannel.class)#        // 类似 IOC 创建 channel
>
> option(ChannelOption.xxxx)#
>
> handler( new ChannelInitializer<SocketChannel>(){...})#
>
>
> Bootstrap#connect

ChannelPipeline  添加handler，编码，解码，自定义

## 使用 Netty

端点如何启动、handler 需要指定，不会自动加载？

手写Tomcat

不同的 handler：

* 实现对 HTTP 协议的解析

  > ```
  > /** 解析 HTTP 请求 */
  > pipeline.addLast(new HttpServerCodec());
  > //主要是将同一个http请求或响应的多个消息对象变成一个 fullHttpRequest完整的消息对象
  > pipeline.addLast(new HttpObjectAggregator(64 * 1024));
  > //主要用于处理大数据流,比如一个1G大小的文件如果你直接传输肯定会撑暴jvm内存的 ,使用该 handler我们就不用考虑这个问题了
  > pipeline.addLast(new ChunkedWriteHandler());
  > // 自定义处理(静态文件加载)
  > pipeline.addLast(new HttpHandler());
  > ```

* 实现对 WebSocket 的解析

  > ```
  > /** 解析 webSocket 请求 */
  > pipeline.addLast(new WebSocketServerProtocolHandler("/im"));
  > // 自定义处理
  > pipeline.addLast(new WebSocketHandler());
  > ```

* 实现对 自定义协议 的解析

  > ```
  > // 解析自定义协议
  > pipeline.addLast(new IMDecoder());
  > pipeline.addLast(new IMEncoder());
  > pipeline.addLast(new SocketHandler());
  > ```



## Netty的生命周期

新建连接：

> isSharable -> handlerAdded -> channelRegistered -> channelId（channelActive） 
>
> -> userEvenTriggered(channelRead) -> channelReadComplete

channelRead读事件，channel有数据可读时调用

channelReadComplete读完事件，channel读完之后调用

**1、如果在添加了Netty提供的对应协议的解码器后，那么channelRead回调时机有变化，Netty会在读取的数据被完整解码后才会继续传递回调它，这里不再是读到Channel的数据就立即开始传递回调channelRead，这个和业务上的完整数据包的设置有关系。**

**2、channelReadComplete方法的回调时机不受影响，即它的回调时机和是否使用了解码器没有一毛钱关系，它只认可JDK的Socketchannel是否读完了，两个判断条件：**

- **下次读到0个字节**
- **本次读到的字节数小于当前缓冲区的容量**

**满足一个，就会触发一次调用，和业务上的一个完整数据包的设置没有任何关系。**

关闭连接：

> channelReadComplete -> channelInactive  -> channelUnregistered ->  channel 删除成功

# Netty 源码分析

 channel() 创建后   -> 从底往上研究

![nio](../../../image/netty-NioSocketChannel.png)

## 客户端 Bootstrap 配置

```java
        EventLoopGroup group = new NioEventLoopGroup();

        final RpcProxyHandler handler = new RpcProxyHandler();

        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(group)
                    .channel(NioSocketChannel.class)
                    .option(ChannelOption.TCP_NODELAY, true)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) throws Exception {
                            ChannelPipeline pipeline = ch.pipeline();


                            pipeline.addLast("frameDecoder", new LengthFieldBasedFrameDecoder(Integer.MAX_VALUE, 0, 4, 0, 4));
                            pipeline.addLast("frameEncoder", new LengthFieldPrepender(4));

                            //处理序列化的解、编码器（JDK默认的序列化）
                            pipeline.addLast("encoder",new ObjectEncoder());
                            pipeline.addLast("decoder",new ObjectDecoder(Integer.MAX_VALUE, ClassResolvers.cacheDisabled(null)));

                            //自己的业务逻辑
                            pipeline.addLast("handler", handler);
                        }
                    });
            
            ChannelFuture f = bootstrap.connect("localhost", 8080).sync();
            f.channel().writeAndFlush(msg).sync();
            f.channel().closeFuture().sync();
            
        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            group.shutdownGracefully();
        }

        return handler.getResult();
    }
```



### EventLoopGroup（线程池）初始化

```java
public NioEventLoopGroup(int nThreads) {
        this(nThreads, (Executor) null);
    }

public NioEventLoopGroup(int nThreads, Executor executor) {
        this(nThreads, executor, SelectorProvider.provider());
    }

public NioEventLoopGroup(
            int nThreads, Executor executor, final SelectorProvider selectorProvider) {
        this(nThreads, executor, selectorProvider, DefaultSelectStrategyFactory.INSTANCE);
    }

public NioEventLoopGroup(int nThreads, Executor executor, final SelectorProvider selectorProvider,final SelectStrategyFactory selectStrategyFactory) {
        super(nThreads, executor, selectorProvider, selectStrategyFactory, RejectedExecutionHandlers.reject());
    }

private static final int DEFAULT_EVENT_LOOP_THREADS;

    static {
        DEFAULT_EVENT_LOOP_THREADS = Math.max(1, SystemPropertyUtil.getInt(
                "io.netty.eventLoopThreads", NettyRuntime.availableProcessors() * 2));

        if (logger.isDebugEnabled()) {
            logger.debug("-Dio.netty.eventLoopThreads: {}", DEFAULT_EVENT_LOOP_THREADS);
        }
    }
protected MultithreadEventLoopGroup(int nThreads, Executor executor, Object... args) {
        super(nThreads == 0 ? DEFAULT_EVENT_LOOP_THREADS : nThreads, executor, args);
    }

protected MultithreadEventExecutorGroup(int nThreads, Executor executor, Object... args) {
        this(nThreads, executor, DefaultEventExecutorChooserFactory.INSTANCE, args);
    }



protected MultithreadEventExecutorGroup(int nThreads, Executor executor,
                                            EventExecutorChooserFactory chooserFactory, Object... args) {
        if (nThreads <= 0) {
            throw new IllegalArgumentException(String.format("nThreads: %d (expected: > 0)", nThreads));
        }

        if (executor == null) {
            executor = new ThreadPerTaskExecutor(newDefaultThreadFactory());
        }

        children = new EventExecutor[nThreads];

        for (int i = 0; i < nThreads; i ++) {
            boolean success = false;
            try {
                children[i] = newChild(executor, args);
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



#### `NioEventLoop#run`

```java
        final int ioRatio = this.ioRatio;
                if (ioRatio == 100) {
                    try {
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        runAllTasks();
                    }
                } else {
                    final long ioStartTime = System.nanoTime();
                    try {
                        processSelectedKeys();
                    } finally {
                        // Ensure we always run tasks.
                        final long ioTime = System.nanoTime() - ioStartTime;
                        runAllTasks(ioTime * (100 - ioRatio) / ioRatio);
                    }
                }
```

> `processSelectionKeys()`要做的事情就是IO操作(select)，netty在处理io之前记录了一下io操作的开始时间，然后在io结束的时候计算了一下这段io操作花的总的时间 `ioTime`,然后`runAllTask`方法表示花多长时间来处理一下netty内部的任务队列(cpu计算为主)

`DefaultThreadFactory#newThread()`

MultithreadEventLoopGroup#newDefaultThreadFactory

SingleThreadEventExecutor#execute

```java
@Override
public void execute(Runnable task) {
    ...
    boolean inEventLoop = inEventLoop();
    if (inEventLoop) {
        addTask(task);
    } else {
        startThread();
        addTask(task);
        ...
    }
    ...
}

/**
外部线程在往任务队列里面添加任务的时候执行 startThread() ，netty会判断reactor线程有没有被启动，如果没有被启动，那就启动线程再往任务队列里面添加任务
SingleThreadEventExecutor 在执行doStartThread的时候，会调用内部执行器executor的execute方法，将调用NioEventLoop的run方法的过程封装成一个runnable塞到一个线程中去执行
该线程就是executor创建，对应netty的reactor线程实体。executor 默认是ThreadPerTaskExecutor

默认情况下，ThreadPerTaskExecutor 在每次执行execute 方法的时候都会通过DefaultThreadFactory创建一个FastThreadLocalThread线程，而这个线程就是netty中的reactor线程实体
*/
private void startThread() {
    if (STATE_UPDATER.get(this) == ST_NOT_STARTED) {
        if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
            doStartThread();
        }
    }
}

private void doStartThread() {
    ...
    executor.execute(new Runnable() {
        @Override
        public void run() {
            thread = Thread.currentThread();
            ...
                SingleThreadEventExecutor.this.run();
            ...
        }
    }
}
```



### Channel type 配置

> channel(xxxChannel.class)  -> clazz#getConstructor
>
> connect -> ReflectiveChannelFactory#newChannel

### Bootstrap#doResolveAndConnect

```java
private ChannelFuture doResolveAndConnect(final SocketAddress remoteAddress, final SocketAddress localAddress) {
        final ChannelFuture regFuture = initAndRegister();
        final Channel channel = regFuture.channel();
    
        // DefaultPromise resutlt==null时 返回 false
        if (regFuture.isDone()) {
            if (!regFuture.isSuccess()) {
                return regFuture;
            }
            return doResolveAndConnect0(channel, remoteAddress, localAddress, channel.newPromise());
        } else {
            // 一般走这里
            // Registration future is almost always fulfilled already, but just in case it's not.
            final PendingRegistrationPromise promise = new PendingRegistrationPromise(channel);
            regFuture.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    // Directly obtain the cause and do a null check so we only need one volatile read in case of a
                    // failure.
                    Throwable cause = future.cause();
                    if (cause != null) {
                        // Registration on the EventLoop failed so fail the ChannelPromise directly to not cause an
                        // IllegalStateException once we try to access the EventLoop of the Channel.
                        promise.setFailure(cause);
                    } else {
                        // Registration was successful, so set the correct executor to use.
                        // See https://github.com/netty/netty/issues/2586
                        promise.registered();
                        doResolveAndConnect0(channel, remoteAddress, localAddress, promise);
                    }
                }
            });
            return promise;
        }
    }
```



#### `AbstractBootstrap#initAndRegister`

```java
final ChannelFuture initAndRegister() {
        Channel channel = null;
        try {
            // 构造器反射创建 Channel
            channel = channelFactory.newChannel();
            init(channel);
        } catch (Throwable t) {
            if (channel != null) {
                // channel can be null if newChannel crashed (eg SocketException("too many open files"))
                channel.unsafe().closeForcibly();
                // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
                return new DefaultChannelPromise(channel, GlobalEventExecutor.INSTANCE).setFailure(t);
            }
            // as the Channel is not registered yet we need to force the usage of the GlobalEventExecutor
            return new DefaultChannelPromise(new FailedChannel(), GlobalEventExecutor.INSTANCE).setFailure(t);
        }

        ChannelFuture regFuture = config().group().register(channel);
        if (regFuture.cause() != null) {
            if (channel.isRegistered()) {
                channel.close();
            } else {
                channel.unsafe().closeForcibly();
            }
        }

        // If we are here and the promise is not failed, it's one of the following cases:
        // 1) If we attempted registration from the event loop, the registration has been completed at this point.
        //    i.e. It's safe to attempt bind() or connect() now because the channel has been registered.
        // 2) If we attempted registration from the other thread, the registration request has been successfully
        //    added to the event loop's task queue for later execution.
        //    i.e. It's safe to attempt bind() or connect() now:
        //         because bind() or connect() will be executed *after* the scheduled registration task is executed
        //         because register(), bind(), and connect() are all bound to the same thread.

        return regFuture;
    }
```

#### ReflectiveChannelFactory#newChannel

- Channel
- ChannelConfig
- ChannelId
- Unsafe
- Pipeline
- ChannelHander

##### NioSocketChannel

> 构造 NioSocketChannel 全过程：
>
> 1. 调用 NioSocketChannel#newSocket(DEFAULT_SELECTOR_PROVIDER)  来 open 一个新的 Java NIO SocketChannel
>
> 2. AbstractChannel(Channel parent) 初始化 AbstractChannel 属性：
>
>    `parent `= null；
>
>    DefaultChannelId 通过 newId
>
>    `unsafe `通过 newUnsafe() 实例化一个 unsafe 对象，类型为内部类
>
>    AbstractNioByteChannel#NioByteUnsafe
>
>    `pipeline `通过 newChannelPipeline() 新建一个 new DefaultChannelPipeline(this);
>
>    可知，每个 Channel 拥有自己的 ChannelPipeline，Channel 创建时会新建（自动绑定） ChannelPipeline
>
> 3. AbstractNioByteChannel 构造 AbstractNioChannel 属性:
>
>    SelectableChannel ch 被设为 Java SocketChannel， 即 NioSocketChannel#newSocket 返回  Java NIO SocketChannel 
>
>    readInterestOp 被设为 SelectionKey#OP_READ  默认注册事件
>
> 4. NioSocketChannel 属性：
>
>    NioSocketChannel  config = new NioSocketChannelConfig(this, socket.socket());



```java
// 私有静态内部类
private static SocketChannel newSocket(SelectorProvider provider) {
        try {
            /**
             *  Use the {@link SelectorProvider} to open {@link SocketChannel} and so remove condition in
             *  {@link SelectorProvider#provider()} which is called by each SocketChannel.open() otherwise.
             *
             *  See <a href="https://github.com/netty/netty/issues/2308">#2308</a>.
             */
            return provider.openSocketChannel();
        } catch (IOException e) {
            throw new ChannelException("Failed to open a socket.", e);
        }
    }

// NioSocketChannel 构造函数
// 中间调用
public NioSocketChannel(SelectorProvider provider) {
        this(newSocket(provider));
    }

public NioSocketChannel(SocketChannel socket) {
        this(null, socket);
    }
// 最终调用    
public NioSocketChannel(Channel parent, SocketChannel socket) {
        super(parent, socket);
        config = new NioSocketChannelConfig(this, socket.socket());
    }

protected AbstractChannel(Channel parent) {
        this.parent = parent;
        id = newId();
        unsafe = newUnsafe();
        pipeline = newChannelPipeline();
    }
```

##### AbstractChannel（finally）

```java
protected AbstractChannel(Channel parent) {
        this.parent = parent;
        id = newId();
        unsafe = newUnsafe();
        pipeline = newChannelPipeline();
    }
```

`ch.configureBlocking(false);`设置该channel为非阻塞模式

SelectionKey.OP_ACCEPT

##### \*Unsafe 类 初始化

Unsafe operations that should never be called from user-code. These methods  are only provided to implement the actual transport, and must be invoked from an I/O thread

Unsafe本身是个内部接口，聚合在Channel接口内部，作用是协助Channel进行网络I/O的操作。因为它的设计初衷就是Channel的内部辅助类，不应该被Netty的使用者调用，所以被命名为Unsafe

###### NioByteUnsafe

AbstractNioByteChannel#newUnsafe

```java
    @Override
    protected AbstractNioUnsafe newUnsafe() {
        return new NioByteUnsafe();
    }
```



##### `pipeline 初始化(管理 Handler)`

###### DefaultChannelPipeline

> AbstratChannel#newChannelPipeline()

```java
pipeline = newChannelPipeline();

protected DefaultChannelPipeline newChannelPipeline() {
    return new DefaultChannelPipeline(this);
}


protected DefaultChannelPipeline(Channel channel) {
    this.channel = ObjectUtil.checkNotNull(channel, "channel");
    succeededFuture = new SucceededChannelFuture(channel, null);
    voidPromise =  new VoidChannelPromise(channel, true);

    tail = new TailContext(this);
    head = new HeadContext(this);

    head.next = tail;
    tail.prev = head;
}
```



双向链表  HeadContext，TailContext

HeadContext，TailContext





#### Bootstrap#init

```java
    @Override
    @SuppressWarnings("unchecked")
    void init(Channel channel) {
        ChannelPipeline p = channel.pipeline();
        p.addLast(config.handler());
        // ChannelOptions
        setChannelOptions(channel, options0().entrySet().toArray(newOptionArray(0)), logger);
        // AttributeKey
        setAttributes(channel, attrs0().entrySet().toArray(newAttrArray(0)));
    }
```



#### `Channel 注册`

###### ChannelInitializer#initChannel

>  AbstractBootstrap.initAndRegister   -> 
>
>  ChannelFuture regFuture = config().group().register(channel); ->
>
>  MultithreadEventLoopGroup#register   ->
>
>  SingleThreadEventLoop#register   -> 
>
>  Channel 封装为 DefaultChannelPromise
>
>  promise.channel().unsafe().register(this, promise);
>
>  AbstractChannel$AbstractUnsafe#register   ->
>
>  eventloop#execute 
>
>  AbstractChannel#register0   ->
>
>  AbstractNioChannel#doRegister   ->
>
>  AbstractSelectableChannel#register   ->
>
>  Channel Initializer  只用于配置 （remove(this)），模板模式
>
>  fire* 方法
>
>  channelRegistered

EventExecutorChooserFactory#EventExecutorChooser

SelectionKey k = findKey(sel);

#####` AbstractChannel#register0`

> DefaultChannelPipeline#invokeHandlerAddedIfNeeded()
>
> DefaultChannelPipeline#callHandlerAddedForAllHandlers
>
> PendingHandlerAddedTask#execute() 
>
> DefaultChannelPipeline#callHandlerAdded0(ctx)
>
> AbstractChannelHandlerContext#callHandlerAdded
>
> handler().handlerAdded(this)
>
>
>
> pipeline.invokeHandlerAddedIfNeeded();
>
> pipeline.fireChannelRegistered();
>
>  pipeline.fireChannelActive();
>
>  beginRead();

```java
// AbstractChannel#register0  
private void register0(ChannelPromise promise) {
            try {
                // check if the channel is still open as it could be closed in the mean time when the register
                // call was outside of the eventLoop
                if (!promise.setUncancellable() || !ensureOpen(promise)) {
                    return;
                }
                boolean firstRegistration = neverRegistered;
                doRegister();
                neverRegistered = false;
                registered = true;

                // Ensure we call handlerAdded(...) before we actually notify the promise. This is needed as the
                // user may already fire events through the pipeline in the ChannelFutureListener.
                pipeline.invokeHandlerAddedIfNeeded();

                safeSetSuccess(promise);
                pipeline.fireChannelRegistered();
                // Only fire a channelActive if the channel has never been registered. This prevents firing
                // multiple channel actives if the channel is deregistered and re-registered.
                if (isActive()) {
                    if (firstRegistration) {
                        pipeline.fireChannelActive();
                    } else if (config().isAutoRead()) {
                        // This channel was registered before and autoRead() is set. This means we need to begin read
                        // again so that we process inbound data.
                        //
                        // See https://github.com/netty/netty/issues/4805
                        beginRead();
                    }
                }
            } catch (Throwable t) {
                // Close the channel directly to avoid FD leak.
                closeForcibly();
                closeFuture.setClosed();
                safeSetFailure(promise, t);
            }
        }
```





```java
    private void callHandlerAddedForAllHandlers() {
        final PendingHandlerCallback pendingHandlerCallbackHead;
        synchronized (this) {
            assert !registered;

            // This Channel itself was registered.
            registered = true;

            pendingHandlerCallbackHead = this.pendingHandlerCallbackHead;
            // Null out so it can be GC'ed.
            this.pendingHandlerCallbackHead = null;
        }

        // This must happen outside of the synchronized(...) block as otherwise handlerAdded(...) may be called while
        // holding the lock and so produce a deadlock if handlerAdded(...) will try to add another handler from outside
        // the EventLoop.
        PendingHandlerCallback task = pendingHandlerCallbackHead;
        while (task != null) {
            // 关键
            task.execute();
            task = task.next;
        }
    }
```





## 客户端 connect 分析

```
Bootstrap.connect -> 
Bootstrap.doResolveAndConnect -> 
AbstractBootstrap.initAndRegister ->
doResolveAndConnect0(实现类方法)  -> 
doConnect
-> ..... -> 
ChannelHandlerContext.connect
```

### Bootstrap#doResolveAndConnect0

```java

    private ChannelFuture doResolveAndConnect0(final Channel channel, SocketAddress remoteAddress,
                                               final SocketAddress localAddress, final ChannelPromise promise) {
        try {
            final EventLoop eventLoop = channel.eventLoop();
            final AddressResolver<SocketAddress> resolver = this.resolver.getResolver(eventLoop);

            if (!resolver.isSupported(remoteAddress) || resolver.isResolved(remoteAddress)) {
                // Resolver has no idea about what to do with the specified remote address or it's resolved already.
                doConnect(remoteAddress, localAddress, promise);
                return promise;
            }

            final Future<SocketAddress> resolveFuture = resolver.resolve(remoteAddress);

            if (resolveFuture.isDone()) {
                final Throwable resolveFailureCause = resolveFuture.cause();

                if (resolveFailureCause != null) {
                    // Failed to resolve immediately
                    channel.close();
                    promise.setFailure(resolveFailureCause);
                } else {
                    // 没问题走这里
                    // Succeeded to resolve immediately; cached? (or did a blocking lookup)
                    doConnect(resolveFuture.getNow(), localAddress, promise);
                }
                return promise;
            }

            // Wait until the name resolution is finished.
            resolveFuture.addListener(new FutureListener<SocketAddress>() {
                @Override
                public void operationComplete(Future<SocketAddress> future) throws Exception {
                    if (future.cause() != null) {
                        channel.close();
                        promise.setFailure(future.cause());
                    } else {
                        // 
                        doConnect(future.getNow(), localAddress, promise);
                    }
                }
            });
        } catch (Throwable cause) {
            promise.tryFailure(cause);
        }
        return promise;
    }



// AbstractNioChannel
@Override
public final void connect(
    final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {
    if (!promise.setUncancellable() || !ensureOpen(promise)) {
        return;
    }

    try {
        if (connectPromise != null) {
            // Already a connect in process.
            throw new ConnectionPendingException();
        }

        boolean wasActive = isActive();
        //
        if (doConnect(remoteAddress, localAddress)) {
            //
            fulfillConnectPromise(promise, wasActive);
        } else {
            connectPromise = promise;
            requestedRemoteAddress = remoteAddress;

            // Schedule connect timeout.
            int connectTimeoutMillis = config().getConnectTimeoutMillis();
            if (connectTimeoutMillis > 0) {
                connectTimeoutFuture = eventLoop().schedule(new Runnable() {
                    @Override
                    public void run() {
                        ChannelPromise connectPromise = AbstractNioChannel.this.connectPromise;
                        ConnectTimeoutException cause =
                            new ConnectTimeoutException("connection timed out: " + remoteAddress);
                        if (connectPromise != null && connectPromise.tryFailure(cause)) {
                            close(voidPromise());
                        }
                    }
                }, connectTimeoutMillis, TimeUnit.MILLISECONDS);
            }

            promise.addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    if (future.isCancelled()) {
                        if (connectTimeoutFuture != null) {
                            connectTimeoutFuture.cancel(false);
                        }
                        connectPromise = null;
                        close(voidPromise());
                    }
                }
            });
        }
    } catch (Throwable t) {
        promise.tryFailure(annotateConnectException(t, remoteAddress));
        closeIfClosed();
    }
}  


// NioSocketChannel
@Override
protected boolean doConnect(SocketAddress remoteAddress, SocketAddress localAddress) throws Exception {
    if (localAddress != null) {
        doBind0(localAddress);
    }

    boolean success = false;
    try {
        // 调用JDK的SocketChannel的connect API，发起远程连接操作
        boolean connected = SocketUtils.connect(javaChannel(), remoteAddress);
        if (!connected) {
            selectionKey().interestOps(SelectionKey.OP_CONNECT);
        }
        success = true;
        return connected;
    } finally {
        if (!success) {
            doClose();
        }
    }
}

/**
由于是在非阻塞模式下，故 connect 可以立即返回，必须判断返回结果。
如果没有立即连接成功，那么说明此时客户端已经发送了SYN包，但还没有收到服务端返回的ACK包，即TCP链路还没有完整建立，此时需要为已经绑定在客户端SocketChannel的Selector注册OP_CONNECT事件，后续客户端NIO线程的run方法（回忆事件循环机制）会轮询检测该事件，直到被触发——触发条件就是对端的ACK包到达，这部分检测和处理I/O事件的机制和前面分析其它事件一样，不再赘述。这样设计的目的在于即使当下连不上远端节点，Netty也不会阻塞主调线程，这里就是不阻塞NIO线程。本demo没有立即连接成功，connected值为false，故马上注册OP_CONNECT后，才立即返回
*/
```

#### Bootstrap#doConnect

> NioSocketChannel#connect
>
> DefaultChannelPipeline#connect
>
> AbstractChannelHandlerContext tail#connect
>
> AbstractNioChannel#AbstractNioUnsafe#connect  ->
>
> SocketUtils#connect  本地->
>
> NioSocketChannel#doConnect ->
>
> SocketChannel#connect

#### AbstractChannelHandlerContext#connect

```java

    @Override
    public ChannelFuture connect(
            final SocketAddress remoteAddress, final SocketAddress localAddress, final ChannelPromise promise) {

        if (remoteAddress == null) {
            throw new NullPointerException("remoteAddress");
        }
        if (isNotValidPromise(promise, false)) {
            // cancelled
            return promise;
        }

        final AbstractChannelHandlerContext next = findContextOutbound(MASK_CONNECT);
        EventExecutor executor = next.executor();
        if (executor.inEventLoop()) {
            next.invokeConnect(remoteAddress, localAddress, promise);
        } else {
            safeExecute(executor, new Runnable() {
                @Override
                public void run() {
                    next.invokeConnect(remoteAddress, localAddress, promise);
                }
            }, promise, null);
        }
        return promise;
    }

    // static final int MASK_CONNECT = 1 << 10;
    private AbstractChannelHandlerContext findContextOutbound(int mask) {
        AbstractChannelHandlerContext ctx = this;
        do {
            ctx = ctx.prev;
        } while ((ctx.executionMask & mask) == 0);
        return ctx;
    }
```



findContextOutbound()  ->  从 DefaultChannelPipeline 内 的 双向链表 的 tail 开始，不断向前寻找第一个 outbound 为 true 的 AbstractChannelHandlerContext，然后调用它的 invokeConnect 方法

NioByteUnsafe -> AbstractNioUnsafe.connect   （关键，正常点击找源码陷入死循环，debug方式）

#### NioSocketChannel#doConnect

```java
@Override
    protected boolean doConnect(SocketAddress remoteAddress, SocketAddress localAddress) throws Exception {
        if (localAddress != null) {
            doBind0(localAddress);
        }

        boolean success = false;
        try {
            // localAddress
            boolean connected = SocketUtils.connect(javaChannel(), remoteAddress);
            if (!connected) {
                // 
                selectionKey().interestOps(SelectionKey.OP_CONNECT);
            }
            success = true;
            return connected;
        } finally {
            if (!success) {
                doClose();
            }
        }
    }



SelectionKeyImpl
public SelectionKey nioInterestOps(int var1) {
        if ((var1 & ~this.channel().validOps()) != 0) {
            throw new IllegalArgumentException();
        } else {
            this.channel.translateAndSetInterestOps(var1, this);
            this.interestOps = var1;
            return this;
        }
    }

```



* 执行链路

> NioEventLoop#connect ->  
>
> NioEventLoop#doResolveAndConnect  ->  
>
> NioEventLoop#doResolveAndConnect0  -> 
>
> NioEventLoop#doConnect  ->  
>
> NioSocketChannel#connect -> 
>
> DefaultChannelPipeline#connect  -> 
>
> TailContext#connect ->
>
> AbstractNioUnsafe#connect  ->
>
> NioSocketChannel#doConnect ->
>
> SocketChannel#connect


1、在非阻塞模式下，Channel的connect方法不会阻塞，必须对返回值分策略处理

2、JDK规定，除了connect方法，还需要调用了finishConnect方法后，才能表明一个TCP连接完成建立

3、理解OP_CONNECT的本质是OP_WRITE，因此怎么处理写事件，就应该怎么处理连接事件，对于CONNECT，需要及时取消，以及只在没有连接的Channel上注册该事件?

4、Netty默认的连接超时时间是30s，业务层可以根据场景单独修改

5、处理OP_CONNECT就绪的Channel时，需要及时删除这个key，避免下次轮询继续被触发

6、结合TCP三次握手的发包过程，理解为什么非阻塞模式下，Channel的connect方法可能会一次连接不成功：

> NIO connect主要的“坑”点由：
>
> 1、在非阻塞模式下，Channel的connect方法不会阻塞，如果对TCP的三次握手过程理解不到位，那么可能不知道connect方法是可以一次连接不成功的，它会立即返回false，此时用户层不做处理，就可能导致bug
>
> 2、还有一个finishConnect方法，如果不调用，那么也会导致问题
>
> 3、不理解OP_CONNECT的本质，导致CPU空轮询的bug，这个很好说，前面总结过NIO的I/O事件在操作系统层面就两类，一个读一个写，OP_CONNECT本质就是OP_WRITE，因此怎么处理写事件，就应该怎么处理连接事件，处理I/O写事件，前面也总结了，对于CONNECT，也需要及时取消，以及只在没有连接的Channel上注册该事件。

## 服务端 ServerBootstrap

1. 设置启动类参数，最重要的就是设置 channel

2. 创建server对应的channel，创建各大组件，包括ChannelConfig,ChannelId,ChannelPipeline,ChannelHandler,Unsafe等

3. 初始化server对应的channel，设置一些attr，option，以及设置子channel的attr，option，给server的channel添加新channel接入器，并出发addHandler,register等事件

4. 调用到 jdk 底层做端口绑定，并触发active事件，active触发的时候，真正做服务端口绑定

![server](../../../image/netty-NioServerSocketChannel.PNG)

```java
        EventLoopGroup master = new NioEventLoopGroup();
        EventLoopGroup worker = new NioEventLoopGroup();

        try {
            ServerBootstrap bs = new ServerBootstrap();
            bs.group(master, worker)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {

                        @Override
                        protected void initChannel(SocketChannel socketChannel) throws Exception {

                            ChannelPipeline pipeline = socketChannel.pipeline();
                            // 处理粘包，拆包的解、编码器
                            pipeline.addLast(new LengthFieldBasedFrameDecoder(Integer.MAX_VALUE, 0, 4, 0, 4));
                            pipeline.addLast(new LengthFieldPrepender(4));

                            // 处理序列化的解、编码器（JDK默认序列化）
                            pipeline.addLast("encoder", new ObjectEncoder());
                            pipeline.addLast("decoder", new ObjectDecoder(Integer.MAX_VALUE, ClassResolvers.cacheDisabled(null)));

                            // 自己的业务逻辑
                            pipeline.addLast(new RegistryHandler());
                        }
                    })
                    .option(ChannelOption.SO_BACKLOG, 128)
                    .childOption(ChannelOption.SO_KEEPALIVE, true);

            ChannelFuture future = bs.bind(this.port).sync();

            System.out.println("Rpc server is start to listen at " + this.port);

            future.channel().closeFuture().sync();

        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {

            master.shutdownGracefully();
            worker.shutdownGracefully();

        }

```

### doBind

```java
private ChannelFuture doBind(final SocketAddress localAddress) {
    //...
    final ChannelFuture regFuture = initAndRegister();
    //...
    final Channel channel = regFuture.channel();
    //...
    doBind0(regFuture, channel, localAddress, promise);
    //...
    return promise;
}
```

#### initAndRegister()

```java
final ChannelFuture initAndRegister() {
    Channel channel = null;
    // ...
    channel = channelFactory.newChannel();
    //...
    init(channel);
    //...
    ChannelFuture regFuture = config().group().register(channel);
    //...
    return regFuture;
}
```

#### init(Channel channel)

```java
@Override
void init(Channel channel) throws Exception {
    // 1.设置option和attr
    final Map<ChannelOption<?>, Object> options = options0();
    synchronized (options) {
        channel.config().setOptions(options);
    }

    final Map<AttributeKey<?>, Object> attrs = attrs0();
    synchronized (attrs) {
        for (Entry<AttributeKey<?>, Object> e: attrs.entrySet()) {
            @SuppressWarnings("unchecked")
            AttributeKey<Object> key = (AttributeKey<Object>) e.getKey();
            channel.attr(key).set(e.getValue());
        }
    }

    ChannelPipeline p = channel.pipeline();
    
    // 2.设置新接入channel的option和attr
    final EventLoopGroup currentChildGroup = childGroup;
    final ChannelHandler currentChildHandler = childHandler;
    final Entry<ChannelOption<?>, Object>[] currentChildOptions;
    final Entry<AttributeKey<?>, Object>[] currentChildAttrs;
    synchronized (childOptions) {
        currentChildOptions = childOptions.entrySet().toArray(newOptionArray(childOptions.size()));
    }
    synchronized (childAttrs) {
        currentChildAttrs = childAttrs.entrySet().toArray(newAttrArray(childAttrs.size()));
    }

    // 3.加入新连接处理器
    p.addLast(new ChannelInitializer<Channel>() {
        @Override
        public void initChannel(Channel ch) throws Exception {
            final ChannelPipeline pipeline = ch.pipeline();
            ChannelHandler handler = config.handler();
            if (handler != null) {
                pipeline.addLast(handler);
            }

            ch.eventLoop().execute(new Runnable() {
                @Override
                public void run() {
                    pipeline.addLast(new ServerBootstrapAcceptor(
                            currentChildGroup, currentChildHandler, currentChildOptions, currentChildAttrs));
                }
            });
        }
    });
}
```

#### config().group().register(channel);

```java
// NioEventLoop 
Override
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}

@Override
public ChannelFuture register(final ChannelPromise promise) {
    ObjectUtil.checkNotNull(promise, "promise");
    promise.channel().unsafe().register(this, promise);
    return promise;
}

// AbstractUnsafe
/**
EventLoop事件循环器绑定到该 NioServerSocketChannel上，然后调用 register0()
*/
@Override
public final void register(EventLoop eventLoop, final ChannelPromise promise) {
    // ...
    AbstractChannel.this.eventLoop = eventLoop;
    // ...
    register0(promise);
}

private void register0(ChannelPromise promise) {
    try {
        boolean firstRegistration = neverRegistered;
        // 
        doRegister();
        neverRegistered = false;
        registered = true;
        
        // 
        pipeline.invokeHandlerAddedIfNeeded();

        safeSetSuccess(promise);
        //
        pipeline.fireChannelRegistered();
        if (isActive()) {
            if (firstRegistration) {
                //
                pipeline.fireChannelActive();
            } else if (config().isAutoRead()) {
                //
                beginRead();
            }
        }
    } catch (Throwable t) {
        closeForcibly();
        closeFuture.setClosed();
        safeSetFailure(promise, t);
    }
}
```

##### isActive

```java
/**
尚未将一个ServerSocket绑定到一个address，所以 isActive() 返回false，我们没有成功进入到pipeline.fireChannelActive();那么那里会调用呢？
因为netty里面很多任务执行都是异步线程即reactor线程调用的，如果我们要查看最先发起的方法调用，我们必须得查看Runnable被提交的地方，逐次递归下去，就能找到那行"消失的代码"
*/
if (isActive()) { // false
    if (firstRegistration) {
        // 调用时机在 doBind0
        pipeline.fireChannelActive();
    } else if (config().isAutoRead()) {
        beginRead();
    }
}

@Override
public boolean isActive() {
    return javaChannel().socket().isBound();
}

/**
* Returns the binding state of the ServerSocket.
*
* @return true if the ServerSocket succesfuly bound to an address
* @since 1.4
*/
// ServerSocket
public boolean isBound() {
    // Before 1.3 ServerSockets were always bound during creation
    return bound || oldImpl;
}


if (!wasActive && isActive()) {
    invokeLater(new Runnable() {
        @Override
        public void run() {
            pipeline.fireChannelActive();
        }
    });
}
```



### doBind0()

```java
private static void doBind0(
            final ChannelFuture regFuture, final Channel channel,
            final SocketAddress localAddress, final ChannelPromise promise) {
        channel.eventLoop().execute(new Runnable() {
            @Override
            public void run() {
                if (regFuture.isSuccess()) {
                    channel.bind(localAddress, promise).addListener(ChannelFutureListener.CLOSE_ON_FAILURE);
                } else {
                    promise.setFailure(regFuture.cause());
                }
            }
        });
    }

// AbstractChannel
@Override
public ChannelFuture bind(SocketAddress localAddress) {
    return pipeline.bind(localAddress);
}

@Override
public final ChannelFuture bind(SocketAddress localAddress) {
    return tail.bind(localAddress);
}

// HeadContext
@Override
public void bind(
        ChannelHandlerContext ctx, SocketAddress localAddress, ChannelPromise promise)
        throws Exception {
    unsafe.bind(localAddress, promise);
}

// NioMessageUnsafe
@Override
public final void bind(final SocketAddress localAddress, final ChannelPromise promise) {
    // ...
    boolean wasActive = isActive();
    // ...
    doBind(localAddress);

    if (!wasActive && isActive()) {
        invokeLater(new Runnable() {
            @Override
            public void run() {
                pipeline.fireChannelActive();
            }
        });
    }
    safeSetSuccess(promise);
}

protected void doBind(SocketAddress localAddress) throws Exception {
    if (PlatformDependent.javaVersion() >= 7) {
        //noinspection Since15
        javaChannel().bind(localAddress, config.getBacklog());
    } else {
        javaChannel().socket().bind(localAddress, config.getBacklog());
    }
}
```

#### channelActive

```java
// HeadContext
public void channelActive(ChannelHandlerContext ctx) throws Exception {
    ctx.fireChannelActive();

    readIfIsAutoRead();
}

private void readIfIsAutoRead() {
    // 默认返回 true
    if (channel.config().isAutoRead()) {
        channel.read();
    }
}

private volatile int autoRead = 1;
public boolean isAutoRead() {
    return autoRead == 1;
}

public Channel read() {
    pipeline.read();
    return this;
}

// 最终调用：AbstractNioUnsafe
protected void doBeginRead() throws Exception {
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }

    readPending = true;

    final int interestOps = selectionKey.interestOps();
    if ((interestOps & readInterestOp) == 0) {
        selectionKey.interestOps(interestOps | readInterestOp);
    }
}

// AbstractNioChannel
// selectionKey = javaChannel().register(eventLoop().selector, 0, this)
// selectionKey.interestOps(interestOps | readInterestOp);
```





> ServerBootstrap.handler(加权限控制用)

ChannelType

NioServerSocketChannel   

AbstractNioMessageChannel#NioMessageUnsafe

ServerBootstrapAcceptor



类似客户端逻辑 connect 对应的 bind

NioServerSocketChannel#doBind

ServerSocketChannelImpl#bind



## 新连接建立

1.检测到有新的连接
2.将新的连接注册到worker线程组
3.注册新连接的读事件



### 检测到有新连接进入

#### processSelectedKey

当服务端启动之后，服务端的channel已经注册到boos reactor线程中，reactor不断检测有新的事件，直到检测出有accept事件发生

所有的 channel 底层都会有一个与 unsafe 绑定，每种类型的channel实际的操作都由unsafe来实现

```java
// NioEventLoop
private void processSelectedKey(SelectionKey k, AbstractNioChannel ch) {
    final AbstractNioChannel.NioUnsafe unsafe = ch.unsafe();
    ...
    int readyOps = k.readyOps();
    if ((readyOps & (SelectionKey.OP_READ | SelectionKey.OP_ACCEPT)) != 0 || readyOps == 0) {
        unsafe.read();
    }
}
```

### 注册到 reactor 线程

用一条断言确定该read方法必须是reactor线程调用，然后拿到channel对应的pipeline和`RecvByteBufAllocator.Handle`(先不解释)

接下来，调用 `doReadMessages` 方法不断地读取消息，用 `readBuf` 作为容器，这里，其实可以猜到读取的是一个个连接，然后调用 `pipeline.fireChannelRead()`，将每条新连接经过一层服务端channel的处理之后清理容器，触发 `pipeline.fireChannelReadComplete()`

#### NioMessageUnsafe#read

```java
// NioMessageUnsafe
private final List<Object> readBuf = new ArrayList<Object>();

public void read() {
    assert eventLoop().inEventLoop();
    final ChannelPipeline pipeline = pipeline();
    // 1.
    final RecvByteBufAllocator.Handle allocHandle = unsafe().recvBufAllocHandle();
    do {
        // 2. 关键 readBuf 存储了什么？
        int localRead = doReadMessages(readBuf);
        if (localRead == 0) {
            break;
        }
        if (localRead < 0) {
            closed = true;
            break;
        }
    } while (allocHandle.continueReading());
    // 
    int size = readBuf.size();
    for (int i = 0; i < size; i ++) {
        // 3. 关键
        pipeline.fireChannelRead(readBuf.get(i));
    }
    // 4.
    readBuf.clear();
    // 5.
    pipeline.fireChannelReadComplete();
}
```

#### `doReadMessages`

源码分析：netty调用jdk底层nio的边界 `javaChannel().accept();`，由于netty中reactor线程第一步就扫描到有accept事件发生，因此，这里的`accept`方法是立即返回的，返回jdk底层nio`创建`的一条channel。

netty将jdk的 `SocketChannel` 封装成自定义的 `NioSocketChannel`，加入到list里面，这样外层就可以遍历该list，做后续处理

>  这里会新建 channel，关注到 channel 新建时 的处理，SelectionKey.OP_READ
>
> 1.channel 继承 Comparable 表示channel是一个可以比较的对象
> 2.channel 继承AttributeMap表示channel是可以绑定属性的对象，在用户代码中，我们经常使用channel.attr(...)方法就是来源于此
> 3.ChannelOutboundInvoker是4.1.x版本新加的抽象，表示一条channel可以进行的操作
> 4.DefaultAttributeMap用于AttributeMap抽象的默认方法,后面channel继承了直接使用
> 5.AbstractChannel用于实现channel的大部分方法，其中我们最熟悉的就是其构造函数中，创建出一条channel的基本组件
> 6.AbstractNioChannel基于AbstractChannel做了nio相关的一些操作，保存jdk底层的 `SelectableChannel`，并且在构造函数中设置channel为非阻塞
> 7.最后，就是两大channel，NioServerSocketChannel，NioSocketChannel对应着服务端接受新连接过程和新连接读写过程

```java
protected int doReadMessages(List<Object> buf) throws Exception {
    //
    SocketChannel ch = javaChannel().accept();

    try {
        if (ch != null) {
            // NioSocketChannel 的新建
            buf.add(new NioSocketChannel(this, ch));
            return 1;
        }
    } catch (Throwable t) {
        logger.warn("Failed to create a new channel from an accepted socket.", t);

        try {
            ch.close();
        } catch (Throwable t2) {
            logger.warn("Failed to close a socket.", t2);
        }
    }

    return 0;
}
```

> Netty封装客户端NioSocketChannel主要就是三件事：
>
> 1、配置SocketChannel为非阻塞——configureBlocking(false)
>
> 2、保存OP_READ事件，但是是延迟注册的，且会为Channel创建唯一id，unsafe组件（负责底层数据读写）和pipeline组件（业务数据流动的载体）
>
> 3、在NioSocketChannelConfig()中判断设置是否打开TCP的Nagle算法，即在非安卓环境下，执行setTcpNoDelay(true)，即禁止Nagle算法，希望把小数据包尽量发送出去，降低延迟，而Nagle算法会通过减少需要传输的数据包来优化网络。在Linux内核中，数据包的发送和接受会先做缓存，分别对应于写缓存和读缓存。目的是为了尽可能发送大块数据，避免网络中充斥着许多小数据块。

**Netty 充分利用了模板方法等设计模式，不论是命名上，还是具体实现上，都非常美观和流畅，可以学习这种组件分类设计的方式**

#### pipeline.fireChannelRead(NioSocketChannel)

```java
//AbstractNioMessageChannel
pipeline.fireChannelRead(NioSocketChannel);
```

 `pipeline.fireChannelRead(NioSocketChannel);` 最终通过 head-> unsafe -> ServerBootstrapAcceptor 的调用链，调用到这里的 `ServerBootstrapAcceptor`  的`channelRead`



#### ServerBootstrapAcceptor#channelRead

```java
 private static class ServerBootstrapAcceptor extends ChannelInboundHandlerAdapter {
    private final EventLoopGroup childGroup;
    private final ChannelHandler childHandler;
    private final Entry<ChannelOption<?>, Object>[] childOptions;
    private final Entry<AttributeKey<?>, Object>[] childAttrs;

    ServerBootstrapAcceptor(
            EventLoopGroup childGroup, ChannelHandler childHandler,
            Entry<ChannelOption<?>, Object>[] childOptions, Entry<AttributeKey<?>, Object>[] childAttrs) {
        this.childGroup = childGroup;
        this.childHandler = childHandler;
        this.childOptions = childOptions;
        this.childAttrs = childAttrs;
    }

    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        // msg 强制转换为 Channel, 为什么这里可以强制转换？
        // 这里的 msg 是 Netty 封装的新连接对象：NioSocketChannel
        final Channel child = (Channel) msg;
        //
        /**
        ServerBootstrap b = new ServerBootstrap();
            b.group(bossGroup, workerGroup)
             .channel(NioServerSocketChannel.class)
             .childHandler(new ChannelInitializer<SocketChannel>() {
                 @Override
                 public void initChannel(SocketChannel ch) throws Exception {
                     ChannelPipeline p = ch.pipeline();
                     p.addLast(new EchoServerHandler());
                 }
             });
        */
        // 之后NioSocketChannel中pipeline对应的处理器为 head->ChannelInitializer->tail
        child.pipeline().addLast(childHandler);

        for (Entry<ChannelOption<?>, Object> e: childOptions) {
            try {
                if (!child.config().setOption((ChannelOption<Object>) e.getKey(), e.getValue())) {
                    logger.warn("Unknown channel option: " + e);
                }
            } catch (Throwable t) {
                logger.warn("Failed to set a channel option: " + child, t);
            }
        }

        for (Entry<AttributeKey<?>, Object> e: childAttrs) {
            child.attr((AttributeKey<Object>) e.getKey()).set(e.getValue());
        }

        try {
            childGroup
                // worker线程池通过EventLoop的线程选择器——Chooser的next()方法选择一个
                // NioEventLoop线程和新连接绑定，和服务端线程池一样的逻辑
                // 注册客户端的新Channel到这个NioEventLoop的I/O多路复用器，
                // 并为其注册OP_READ事件
                .register(child)
                .addListener(new ChannelFutureListener() {
                @Override
                public void operationComplete(ChannelFuture future) throws Exception {
                    if (!future.isSuccess()) {
                        forceClose(child, future.cause());
                    }
                }
            });
        } catch (Throwable t) {
            forceClose(child, t);
        }
    }
```

##### next()

```java
// MultithreadEventLoopGroup
public ChannelFuture register(Channel channel) {
    return next().register(channel);
}

@Override
public EventLoop next() {
    return (EventLoop) super.next();
}

// MultithreadEventExecutorGroup
@Override
public EventExecutor next() {
    // 选择完一个reactor线程，即 NioEventLoop 之后
    return chooser.next();
}

// EventExecutorChooser
/**
默认情况下，chooser通过 DefaultEventExecutorChooserFactory被创建，在创建reactor线程选择器的时候，会判断reactor线程的个数，如果是2的幂，就创建PowerOfTowEventExecutorChooser，否则，创建GenericEventExecutorChooser

*/
public interface EventExecutorChooserFactory {

    /**
     * Returns a new {@link EventExecutorChooser}.
     */
    EventExecutorChooser newChooser(EventExecutor[] executors);

    /**
     * Chooses the next {@link EventExecutor} to use.
     */
    @UnstableApi
    interface EventExecutorChooser {

        /**
         * Returns the new {@link EventExecutor} to use.
         */
        EventExecutor next();
    }
}

public final class DefaultEventExecutorChooserFactory implements EventExecutorChooserFactory {

    public static final DefaultEventExecutorChooserFactory INSTANCE = new DefaultEventExecutorChooserFactory();

    private DefaultEventExecutorChooserFactory() { }

    @SuppressWarnings("unchecked")
    @Override
    public EventExecutorChooser newChooser(EventExecutor[] executors) {
        if (isPowerOfTwo(executors.length)) {
            return new PowerOfTowEventExecutorChooser(executors);
        } else {
            return new GenericEventExecutorChooser(executors);
        }
    }

    private static boolean isPowerOfTwo(int val) {
        return (val & -val) == val;
    }

    private static final class PowerOfTowEventExecutorChooser implements EventExecutorChooser {
        private final AtomicInteger idx = new AtomicInteger();
        private final EventExecutor[] executors;

        PowerOfTowEventExecutorChooser(EventExecutor[] executors) {
            this.executors = executors;
        }

        @Override
        public EventExecutor next() {
            return executors[idx.getAndIncrement() & executors.length - 1];
        }
    }

    private static final class GenericEventExecutorChooser implements EventExecutorChooser {
        private final AtomicInteger idx = new AtomicInteger();
        private final EventExecutor[] executors;

        GenericEventExecutorChooser(EventExecutor[] executors) {
            this.executors = executors;
        }

        @Override
        public EventExecutor next() {
            return executors[Math.abs(idx.getAndIncrement() % executors.length)];
        }
    }
}

// next() 和 register(channel)
// NioEventLoop
public ChannelFuture register(Channel channel) {
    return next().register(channel);
}

// SingleThreadEventLoop
@Override
public ChannelFuture register(Channel channel) {
    return register(new DefaultChannelPromise(channel, this));
}

// AbstractNioChannel
private void register0(ChannelPromise promise) {
    boolean firstRegistration = neverRegistered;
    doRegister();
    neverRegistered = false;
    registered = true;

    pipeline.invokeHandlerAddedIfNeeded();

    safeSetSuccess(promise);
    pipeline.fireChannelRegistered();
    if (isActive()) {
        if (firstRegistration) {
            pipeline.fireChannelActive();
        } else if (config().isAutoRead()) {
            beginRead();
        }
    }
}

protected void doRegister() throws Exception {
    boolean selected = false;
    for (;;) {
        try {
            selectionKey = javaChannel().register(eventLoop().selector, 0, this);
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                eventLoop().selectNow();
                selected = true;
            } else {
                throw e;
            }
        }
    }
}
```

将该条channel绑定到一个`selector`上去，一个selector被一个reactor线程使用，后续该channel的事件轮询，以及事件处理，异步task执行都是由此reactor线程来负责

绑定完reactor线程之后，调用 `pipeline.invokeHandlerAddedIfNeeded()`

```java
//AbstractNioChannel
private boolean initChannel(ChannelHandlerContext ctx) throws Exception {
    if (initMap.putIfAbsent(ctx, Boolean.TRUE) == null) { 
        try {
            initChannel((C) ctx.channel());
        } catch (Throwable cause) {
            exceptionCaught(ctx, cause);
        } finally {
            remove(ctx);
        }
        return true;
    }
    return false;
}
```

### 注册读事件

#### doBeginRead

```java
// AbstractNioChannel
private void register0(ChannelPromise promise) {
    // ..
    pipeline.fireChannelRegistered();
    if (isActive()) {
        if (firstRegistration) {
            pipeline.fireChannelActive();
        } else if (config().isAutoRead()) {
            beginRead();
        }
    }
}

// 这里其实就是将 SelectionKey.OP_READ事件注册到selector中去，表示这条通道已经可以开始处理read事件了
@Override
protected void doBeginRead() throws Exception {
    // Channel.read() or ChannelHandlerContext.read() was called
    final SelectionKey selectionKey = this.selectionKey;
    if (!selectionKey.isValid()) {
        return;
    }

    readPending = true;

    final int interestOps = selectionKey.interestOps();
    if ((interestOps & readInterestOp) == 0) {
        selectionKey.interestOps(interestOps | readInterestOp);
    }
}
```



 1. boos reactor线程轮询到有新的连接进入
 2. 通过封装jdk底层的channel创建 `NioSocketChannel`以及一系列的netty核心组件
 3. 将该条连接通过chooser，选择一条worker reactor线程绑定上去
 4. 注册读事件，开始新连接的读写





### 消息处理逻辑

数据从head节点流入，先拆包，然后解码成业务对象，最后经过业务Handler处理，调用write，将结果对象写出去。而写的过程先通过tail节点，然后通过encoder节点将对象编码成ByteBuf，最后将该ByteBuf对象传递到head节点，调用底层的Unsafe写到jdk底层管道



tail节点在发现字节数据(ByteBuf)或者decoder之后的业务对象在pipeline流转过程中没有被消费，落到tail节点，tail节点就会给你发出一个警告，告诉你，我已经将你未处理的数据给丢掉了



由于对端消费不及时导致writeAndFlush引发频繁Old GC的问题



#### MessageToByteEncoder：编码器

1.pipeline中的编码器原理是创建一个ByteBuf,将java对象转换为ByteBuf，然后再把ByteBuf继续向前传递
 2.调用write方法并没有将数据写到Socket缓冲区中，而是写到了一个单向链表的数据结构中，flush才是真正的写出
 3.writeAndFlush等价于先将数据写到netty的缓冲区，再将netty缓冲区中的数据写到Socket缓冲区中，写的过程与并发编程类似，用自旋锁保证写成功
 4.netty中的缓冲区中的ByteBuf为DirectByteBuf

```java
public abstract class MessageToByteEncoder<I> extends ChannelOutboundHandlerAdapter {

    private final TypeParameterMatcher matcher;
    private final boolean preferDirect;
    ...    
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
        ByteBuf buf = null;
        try {
            // 判断当前Handelr是否能处理写入的消息
            if (acceptOutboundMessage(msg)) {
                @SuppressWarnings("unchecked")
                // 强制转换
                I cast = (I) msg;
                // 分配一段ButeBuf
                buf = allocateBuffer(ctx, cast, preferDirect);
                try {
                // 调用encode，这里就调回到  `Encoder` 这个Handelr中    
                    encode(ctx, cast, buf);
                } finally {
                    // 既然自定义java对象转换成ByteBuf了，那么这个对象就已经无用了，释放掉
                    // (当传入的msg类型是ByteBuf的时候，就不需要自己手动释放了)
                    ReferenceCountUtil.release(cast);
                }
                // 如果buf中写入了数据，就把buf传到下一个节点
                if (buf.isReadable()) {
                    ctx.write(buf, promise);
                } else {
                // 否则，释放buf，将空数据传到下一个节点    
                    buf.release();
                    ctx.write(Unpooled.EMPTY_BUFFER, promise);
                }
                buf = null;
            } else {
                // 如果当前节点不能处理传入的对象，直接扔给下一个节点处理
                ctx.write(msg, promise);
            }
        } catch (EncoderException e) {
            throw e;
        } catch (Throwable e) {
            throw new EncoderException(e);
        } finally {
            // 当buf在pipeline中处理完之后，释放
            if (buf != null) {
                buf.release();
            }
        }
    }
    ...
}      
```

> 1.判断当前Handler是否能处理写入的消息，如果能处理，进入下面的流程，否则，直接扔给下一个节点处理
>  2.将对象强制转换成`Encoder`可以处理的 `Response`对象
>  3.分配一个ByteBuf
>  4.调用encoder，即进入到 `Encoder` 的 `encode`方法，该方法是用户代码，用户将数据写入ByteBuf
>  5.既然自定义java对象转换成ByteBuf了，那么这个对象就已经无用了，释放掉，(当传入的msg类型是ByteBuf的时候，就不需要自己手动释放了)
>  6.如果buf中写入了数据，就把buf传到下一个节点，否则，释放buf，将空数据传到下一个节点
>  7.最后，当buf在pipeline中处理完之后，释放节点
>
> `Encoder`节点分配一个ByteBuf，调用`encode`方法，将java对象根据自定义协议写入到ByteBuf，然后再把ByteBuf传入到下一个节点，

#### write：写队列

```java
// HeadContext
public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) throws Exception {
    unsafe.write(msg, promise);
}

// AbstractChannel
@Override
public final void write(Object msg, ChannelPromise promise) {
    // 确保该方法的调用是在reactor线程中
    assertEventLoop();
    
    ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;

    int size;
    try {
        // 将待写入的对象过滤，把非ByteBuf对象和FileRegion过滤，
        // 把所有的非直接内存转换成直接内存 DirectBuffer
        msg = filterOutboundMessage(msg);
        // 估算出需要写入的ByteBuf的size
        size = pipeline.estimatorHandle().size(msg);
        if (size < 0) {
            size = 0;
        }
    } catch (Throwable t) {
        safeSetFailure(promise, t);
        ReferenceCountUtil.release(msg);
        return;
    }
     
    outboundBuffer.addMessage(msg, size, promise);
}
```

##### filterOutboundMessage

```java
// AbstractNioByteChannel
@Override
protected final Object filterOutboundMessage(Object msg) {
    if (msg instanceof ByteBuf) {
        ByteBuf buf = (ByteBuf) msg;
        if (buf.isDirect()) {
            return msg;
        }

        return newDirectBuffer(buf);
    }

    if (msg instanceof FileRegion) {
        return msg;
    }

    throw new UnsupportedOperationException(
            "unsupported message type: " + StringUtil.simpleClassName(msg) + EXPECTED_TYPES);
}
```

##### ChannelOutboundBuffer#addMessage

```java
// ChannelOutboundBuffer
public void addMessage(Object msg, int size, ChannelPromise promise) {
    // 创建一个待写出的消息节点 , 封装promise
    Entry entry = Entry.newInstance(msg, size, total(msg), promise);
    if (tailEntry == null) {
        // 添加过程中一直为 null
        flushedEntry = null;
        tailEntry = entry;
    } else {
        Entry tail = tailEntry;
        tail.next = entry;
        tailEntry = entry;
    }
    if (unflushedEntry == null) {
        unflushedEntry = entry;
    }

    incrementPendingOutboundBytes(size, false);
}
```



ChannelOutboundBuffer 里面的数据结构是一个单链表结构，每个节点是一个 `Entry`，`Entry` 里面包含了待写出`ByteBuf` 以及消息回调 `promise`

1.flushedEntry 指针表示第一个被写到操作系统Socket缓冲区中的节点
 2.unFlushedEntry 指针表示第一个未被写入到操作系统Socket缓冲区中的节点
 3.tailEntry指针表示ChannelOutboundBuffer缓冲区的最后一个节点

初次调用 `addMessage` 之后，各个指针的情况为，`fushedEntry`指向空，`unFushedEntry`和 `tailEntry` 都指向新加入的节点

调用n次`addMessage`，flushedEntry指针一直指向NULL，表示现在还未有节点需要写出到Socket缓冲区，而`unFushedEntry`之后有n个节点，表示当前还有n个节点尚未写出到Socket缓冲区中去

#### flush：刷新写队列

不管调用`channel.flush()`，还是`ctx.flush()`，最终都会到pipeline中的head节点

```java
// HeadContext.
@Override
public void flush(ChannelHandlerContext ctx) throws Exception {
    unsafe.flush();
}

// AbstractChannel$AbstractUnsafe
public final void flush() {
   assertEventLoop();

   ChannelOutboundBuffer outboundBuffer = this.outboundBuffer;
   if (outboundBuffer == null) {
       return;
   }

   outboundBuffer.addFlush();
   flush0();
}

```

##### addFlush

```java
// ChannelOutboundBuffer
public void addFlush() {
    Entry entry = unflushedEntry;
    if (entry != null) {
        if (flushedEntry == null) {
            flushedEntry = entry;
        }
        // 统计需 flush 的 entry
        do {
            flushed ++;
            if (!entry.promise.setUncancellable()) {
                int pending = entry.cancel();
                decrementPendingOutboundBytes(pending, false, true);
            }
            entry = entry.next;
        } while (entry != null);
        // All flushed so reset unflushedEntry
        unflushedEntry = null;
    }
}

private void decrementPendingOutboundBytes(long size, boolean invokeLater, boolean notifyWritability) {
        if (size == 0) {
            return;
        }

        long newWriteBufferSize = TOTAL_PENDING_SIZE_UPDATER.addAndGet(this, -size);
        if (notifyWritability && newWriteBufferSize < channel.config().getWriteBufferLowWaterMark()) {
            setWritable(invokeLater);
        }
    }

// fireChannelWritabilityChanged
```

首先拿到 `unflushedEntry` 指针，然后将 `flushedEntry` 指向`unflushedEntry`所指向的节点,

##### `flush0`(核心)

```java
// AbstractUnsafe
protected void flush0() {
    doWrite(outboundBuffer);
}

// AbstractNioByteChannel
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    int writeSpinCount = -1;

    boolean setOpWrite = false;
    for (;;) {
        // 拿到第一个需要flush的节点的数据
        Object msg = in.current();

        if (msg instanceof ByteBuf) {
            // 强转为ByteBuf，若发现没有数据可读，直接删除该节点
            ByteBuf buf = (ByteBuf) msg;

            boolean done = false;
            long flushedAmount = 0;
            // 拿到自旋锁迭代次数
            if (writeSpinCount == -1) {
                writeSpinCount = config().getWriteSpinCount();
            }
            // 自旋，将当前节点写出
            // 自旋的方式将ByteBuf写出到jdk nio的Channel
            for (int i = writeSpinCount - 1; i >= 0; i --) {
                int localFlushedAmount = doWriteBytes(buf);
                if (localFlushedAmount == 0) {
                    setOpWrite = true;
                    break;
                }

                flushedAmount += localFlushedAmount;
                if (!buf.isReadable()) {
                    done = true;
                    break;
                }
            }

            in.progress(flushedAmount);

            // 写完之后，将当前节点删除
            if (done) {
                in.remove();
            } else {
                break;
            }
        } 
    }
}

// ChannelOutBoundBuffer
public Object current() {
    Entry entry = flushedEntry;
    if (entry == null) {
        return null;
    }

    return entry.msg;
}

protected int doWriteBytes(ByteBuf buf) throws Exception {
    final int expectedWrittenBytes = buf.readableBytes();
    // java nio 处理
    return buf.readBytes(javaChannel(), expectedWrittenBytes);
}

// ChannelOutBoundBuffer
public boolean remove() {
    Entry e = flushedEntry;
    Object msg = e.msg;

    ChannelPromise promise = e.promise;
    int size = e.pendingSize;
    // remove是逻辑移除，只是将flushedEntry指针移到下个节点
    removeEntry(e);

    if (!e.cancelled) {
        ReferenceCountUtil.safeRelease(msg);
        safeSuccess(promise);
        /**
        ctx.write(xx).addListener(new GenericFutureListener<Future<? super Void>>() {
        @Override
        public void operationComplete(Future<? super Void> future) throws Exception {
           // 回调 
        }
    })
        */
    }

    // recycle the entry
    e.recycle();

    return true;
}

private void removeEntry(Entry e) {
    if (-- flushed == 0) {
        flushedEntry = null;
        if (e == tailEntry) {
            tailEntry = null;
            unflushedEntry = null;
        }
    } else {
        flushedEntry = e.next;
    }
}
```



```java
ChannelConfig
/**
 * Returns the maximum loop count for a write operation until
 * {@link WritableByteChannel#write(ByteBuffer)} returns a non-zero value.
 * It is similar to what a spin lock is used for in concurrency programming.
 * It improves memory utilization and write throughput depending on
 * the platform that JVM runs on.  The default value is {@code 16}.
 */
int getWriteSpinCount();
```



#### writeAndFlush: 写队列并刷新

```java
// DefaultChannelPipeline
public final ChannelFuture writeAndFlush(Object msg) {
    return tail.writeAndFlush(msg);
}
// TailContext
public ChannelFuture writeAndFlush(Object msg) {
    // DefaultChannelPromise
    return writeAndFlush(msg, newPromise());
}

public ChannelFuture writeAndFlush(Object msg, ChannelPromise promise) {
    write(msg, true, promise);

    return promise;
}

// AbstractChannelHandlerContext
private void write(Object msg, boolean flush, ChannelPromise promise) {
    ...
    AbstractChannelHandlerContext next = findContextOutbound(...);
    EventExecutor executor = next.executor();
    // 判断当前驱动线程是不是当前Channel绑定的NIO线程
    if (executor.inEventLoop()) {
        if (flush) {
            next.invokeWriteAndFlush(m, promise);
        } else {
            next.invokeWrite(m, promise);
        }
    } else {
        // 外部线程驱动
        // 将异步API的底层逻辑封装为一个task扔到当前Channel绑定的NIO线程的MPSCQ里，
        // 排队等待被NIO线程执行
        final AbstractWriteTask task;
        if (flush) {
            task = WriteAndFlushTask.newInstance(next, m, promise);
        }  else {
            task = WriteTask.newInstance(next, m, promise);
        }
        if (!safeExecute(executor, task, promise, m)) {
            // We failed to submit the AbstractWriteTask. We need to cancel it so we decrement the pending bytes
            // and put it back in the Recycler for re-use later.
            //
            // See https://github.com/netty/netty/issues/8343.
            task.cancel();
        }
    }
}

private void invokeWriteAndFlush(Object msg, ChannelPromise promise) {
    // --> headcontext#write
    invokeWrite0(msg, promise);
    invokeFlush0();
}

private void invokeWrite(Object msg, ChannelPromise promise) {
        invokeWrite0(msg, promise);
}

private void invokeFlush(Object msg, ChannelPromise promise) {
        invokeFlush0(msg, promise);
}
```



#### 异步模型

Netty的异步模型本质就是观察者模式，其中：

1、观察者：GenericFutureListener

2、被观察者（主题）：ChannelFuture——ChannelPromise

3、注册观察，即向被观察者添加观察者的方式是：future.addListener(xxx)

4、主题通知观察者的流程：在异步任务完成后，会调用safeSuccess()（相应的异常情况下会调用safeSetFailure，只看正常情况即可），然后是—>trySuccess()—>notifyListener0()，最终回调了operationComplete方法。

5、一个细节，Netty的异步API的回调流程都是NIO线程驱动的，即如果外部调用该API的线程仍然是当前Channel绑定的NIO线程，那么底层其实是同步的，串行执行的，细品。。。



**如何正确使用writeAndFlush()**

Netty的所有的I/O的操作，如果是Netty调用的，或者直接使用的NIO线程驱动的，那么这些异步API本质还是同步的API，即串行执行，所以如果是耗时业务，那么一定不要在NIO线程中执行，因为Netty的API底层逻辑是当前Channel绑定的那个NIO线程来驱动的，比如回调operationComplete方法的流程会在NIO线程中执行，所以有耗时操作会增大异步API的返回时间，建议在自己的业务线程池，或者Netty提供的非I/O线程池中进行耗时业务处理。

**狭隘的说，API层面异步的本质是利用多线程（进程）提升性能**

在Java里，异步API往往是基于一个新开的线程实现的，从调用线程来看，该API是异步的，但是从新开的那个线程本身来看，这个过程仍然是同步（需要等待）的，只是对于调用方而言，这种同步是透明的

真正的异步可以搭配操作系统的异步I/O模型

管程机制

trySuccess--> notifyAll

Netty的异步API转同步，必须用非NIO线程调用，否则会死锁，Netty也会抛出异常

唤醒之前，先判断是否有阻塞的线程，有才唤醒，这里也有信号量的影子。需要掌握这种设计机制，尤其一些网络框架，比如dubbo，也是套的这个原理，只不过dubbo用的是JUC的Condition+Lock实现的，大同小异。

**不论“你”用什么骚操作，只要能让CPU可以立即返回做其他事情，这就是实现了异步**。

### Channel 和 ChannelPipeline

每个 channel 都有且仅有一个 ChannelPipeline  与之对应，一个 ChannelPipeline  对应多个context/handler

（handler 会封装成 handlerContext）

#### 关于 Pipeline 的事件传输机制

参照物 channel

Out  请求型动作 业务程序主动触发

In     响应型动作  Netty来主动触发，业务程序是被动接收（事件）



inbound （接收）        true/false         ChannelInboundHandler
outbound （发送）      false/true        ChannelOutboundHandler

尾进头出

Outbound 事件(请求发生)   tail -> customContext -> head



fireChannelRegistered ->  

U环

###  EventLoop



## Promise 和 Future（成对）

Promise    异步调用，可写Future

Future       异步调用

Future#addListener

UI层面



## 各种 Handler

### ChannelHandlerContext

Channel 加入 ChannelPipeline 创建 ChannelHandlerContext

AbstractChannelHandlerContext#firechannelregister()

### Channel 状态（方法 inbound）

ChannelUnregistered

ChannelRegistered

ChannelInactive

ChannelActive

## 设计模式

装饰器、责任链模式、模板模式、工厂模式、反应堆



大部分缓冲为堆外内存

MultithreadEventLoopGroup#newChild  -> 

SingleThreadEventLoop#register  ->  

 channel.unsafe().register -> 

AbstractUnsafe.register 中 AbstractChannel.this.eventLoop = eventLoop   -> eventLoop#execute

doStartThread

.











