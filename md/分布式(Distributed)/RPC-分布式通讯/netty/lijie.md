顺序问题 ，并发问题，

**抽象类SimpleChannelInboundHandle**

一般推荐继承该类去实现自己的入站处理器,泛型类,可以方便的处理字节消息，比如用户可以自定义要处理的消息的类型

SimpleChannelInboundHandler类能帮助用户自动释放堆外内存

c此时用户处理ByteBuf的业务逻辑应该写在channelRead0里，即使用户忘记继续传播channelRead事件，或者没有自行实现资源释放



Netty的设计理念原本是将异常处理器定义为既是出站handler，又是入站handler，但是这样和其传播顺序的设计有一些不和谐，后续索性就将其赶到了子类——ChannelInboundHandler接口，即自定义异常处理器的正确方法是继承ChannelInboundHandlerAdapter，将其看作一个入站handler。

######**模板方法和策略模式的对比**

两个模式都能改变类的行为，都是类的行为模式。但是策略模式改变类的行为，依靠的是委托机制，而模板方法模型改变类的行为，依靠的是继承，按照设计原则，应该是策略模式比较先进，因为它不会受制于父类的变化而被拖累。



**处理事件和传播事件是两个概念**

inbound事件从pipeline的head节点开始传播，直到tail结束。outbound事件传播恰恰相反，从tail节点开始，传播到head节点结束。

awaitTermination

register最后一个可选参数——附加的对象，即可将一个对象附着到SelectionKey，这样就能方便的安排一些自己的业务，Netty就是直接将自己封装的Channel对象，作为附加绑定到了I/O多路复用器上



业务线程池：DefaultEventExecutorGroup，



pipeline底层就是一个双向链表数据结构，链表的节点类型是聚合了ChannelHandler的一个上下文环境对象——ChannelHandlerContext，链表头为HeadContext，尾节点为TailContext（双向链表是Netty自己实现的，而不是使用JDK的链表，为了轻量)，并且每个ChannelHandlerContext中又关联着一个ChannelHandler，每一个Channel都关联一个独一无二的ChannelPipeline，因此各个ChannelPipeline就可以独立的拦截绑定的Channel上的入站事件和出站事件，这些事件本质都是ChannelHandler对象，这些事件的流动就是双向链表的遍历过程，前面说过入站事件从链表头部开始流动到尾部结束，出站事件相反，从链表尾部开始流动到头部结束，这两个过程本质就是正向和逆向遍历双向链表的过程，又因为是双向链表，所以用户可以非常快速的增加或删除ChannelHandler来实现对不同业务逻辑的灵活处理。



1、ChannelPipeline本质是一个双向链表，一个ChannelPipeline关联一个Channel，组成该链表的节点就是ChannelHandlerContext——一个将ChannelHandler和ChannelPipeline关联起来的上下文环境对象，其内部聚合了ChannelHandler和ChannelPipeline，用户（也包括Netty默认）每添加一个handler都会为其创建一个ChannelHandlerContext对象，然后将其插入pipeline



2、ChannelHandler是拥有独立的业务功能的逻辑处理单元，可以添加到多个ChannelPipeline上，也可以通过注解@Sharable标记其被多个ChannelPipeline（本质是网络Channel）共享



3、ChannelHandlerContext是关联ChannelHandler和ChannelPipeline的上下文环境，聚合了ChannelPipeline，也聚合了ChannelHandler，可以控制ChannelHandler在ChannelPipeline中的传播动作



- 某个客户端Channel成功连接到另一个服务器称为“连接就绪”，SelectionKey.OP_CONNECT事件
- 一个服务端Channel准备好接收新的连接称为“接收就绪”，SelectionKey.OP_ACCEPT事件
- 一个有数据可读的Channel可以说是“读就绪”，SelectionKey.OP_READ事件
- 等待写数据的通道可以说是“写就绪”——SelectionKey.OP_WRITE事件





Netty的服务端用法是实现两个线程池，一个是接入线程池，一个是I/O处理线程池，这个模式也叫Reactor模型



Netty在自己封装的buffer的基础上，实现了自己的内存管理体系，不再强依赖Java的GC。而是大胆的使用了堆外内存管理技术，并设计了很多工具类，使其尽最大可能复用内存（内存池化技术），提高GC性能。





如果 epoll_wait() 只在读/写事件发生时返回，就像前面举的经验例子，该触发叫做边缘触发——ET(edge-triggered)，也就是说如果事件处理函数只读取了该 fd 的缓冲区的部分内容就返回了，接下来再次调用 epoll_wait()，虽然此时该就绪的 fd 对应的缓冲区中还有数据，但 epoll_wait() 函数也不会返回。

　　相反，无论当前的 fd 中是否有读/写事件反生了，只要 fd 对应的缓冲区中有数据可读/写，epoll_wait() 就立即返回，这叫做水平触发——LT(level-triggered)。



epoll 由三个系统调用组成，分别是 epoll_create，epoll_ctl 和 epoll_wait，epoll_create 用于创建和初始化一些内部使用的数据结构，epoll_ctl 用于添加，删除或修改指定的 fd 及其期待的事件，epoll_wait 就是用于等待任何先前指定的fd事件就绪。

　　服务端使用 epoll 步骤如下：

- 调用 epoll_create 在 Linux 内核中创建一个事件表；
- 将 fd（监听套接字 listener）添加到所创建的事件表；
- 在主循环中，调用 epoll_wait 等待返回就绪的 fd 集合；



回调：A类调用B类的方法C，然后B类反过来调用A类的方法D，D方法就叫回调方法



1、如果都是直接拿到ctx就调用write方法，那么都是从当前handler开始传播write事件，通过pipeline的prev指针，从尾部到头部的方向，和inbound事件传播顺序相反

2、如果是先拿到当前Channel或者pipeline后在执行write方法，那么是从尾部开始传播



##### 信号量的使用

通俗的说就是信号量是记录一些信息(量)的信号，并根据记录的信息决定进程睡眠还是唤醒(信号)



1、同时允许多个线程进入临界区（这也是信号量的特有功能），但信号量不能同时唤醒多个线程，只能唤醒一个阻塞中的线程

2、信号量没有条件的概念——即JUC的Condition，换句话说，阻塞线程被信号量唤醒后，再次被调度时会直接运行而不会检查此时临界条件是否已经不满足，基于此考虑信号量模型才会设计出只能让一个线程被唤醒，否则就会出现因为缺少Condition的检查而带来的线程安全问题。正因为缺失了Condition的检查，所以用信号量来实现阻塞队列就很麻烦，因为要自己实现类似Condition的逻辑。

3、P、V操作（原语）的由来：信号量是一种功能较强的机制，表达了比信号更丰富的含义，可用来解决互斥与同步的问题。它只能被两个标准的原语wait和signal来访问，也可以记为“P操作”和“V操作”。

> P的名称来源于荷兰语的proberen，即test，测试看看是不是要阻塞。V的名称也来源于荷兰语verhogen(increment)，即增加的意思。
>
> 操作系统概念



管程能做的事情，信号量都能做，而管程不能做的事情，信号量也能做，比如Semaphore有一个功能是Lock不容易实现的，那就是信号量允许多个线程访问一个临界区。比较常见的需求就是工作中遇到的各种池化资源，例如各种连接池、对象池、线程池，甚至是限流器等。

比如数据库连接池同一时刻一定是允许多个线程同时使用连接池的，当然每个连接在被释放前，是不允许其他线程使用的。

所谓对象池指的是一次性创建出N个对象，之后所有的线程重复利用这N个对象，当然对象在被释放前，也是不允许其他线程使用的，通过信号量机制就能实现这样的要求。



##### 原子更新器

如果用原子类直接包装，那么后续每个BufferedInputStream对象都会额外创建一个原子类的对象，会消耗更多的内存，负担较重，因此JDK直接使用了原子更新器代替了原子类

原子更新器的注意事项：

1、包装的必须是被volatile修饰的共享变量

2、包装的必须是非静态的共享变量

3、必须搭配CAS的套路自行实现比较并交换的逻辑

4、自行实现比较并交换的逻辑时需要注意：如果是非一锤子买卖的原子更新操作，那么必须用局部变量缓存外部的共享变量的旧值，

多线程环境下，实例变量转为局部变量的程序设计技巧

```java
 /**
 * Abstract base class for {@link OrderedEventExecutor}'s that execute all its submitted tasks in a single thread.
 */
public abstract class SingleThreadEventExecutor extends AbstractScheduledEventExecutor implements OrderedEventExecutor {
    private static final int ST_NOT_STARTED = 1;
    private static final int ST_STARTED = 2;
    private static final int ST_SHUTTING_DOWN = 3;
    private static final int ST_SHUTDOWN = 4;
    private static final int ST_TERMINATED = 5;

    private static final AtomicIntegerFieldUpdater<SingleThreadEventExecutor> STATE_UPDATER;
    private static final AtomicReferenceFieldUpdater<SingleThreadEventExecutor, ThreadProperties> PROPERTIES_UPDATER;
    private static final long SCHEDULE_PURGE_INTERVAL = TimeUnit.SECONDS.toNanos(1);

    static {
        AtomicIntegerFieldUpdater<SingleThreadEventExecutor> updater =
                PlatformDependent.newAtomicIntegerFieldUpdater(SingleThreadEventExecutor.class, "state");
        if (updater == null) {
            updater = AtomicIntegerFieldUpdater.newUpdater(SingleThreadEventExecutor.class, "state");
        }
        STATE_UPDATER = updater;
    }

    private final Queue<Runnable> taskQueue;
    private final Executor executor;
    private volatile Thread thread;
    private volatile int state = ST_NOT_STARTED;


    /**
     * NioEventLoop线程启动方法, 这里会判断本NIO线程是否已经启动
     */
    private void startThread() {
        if (STATE_UPDATER.get(this) == ST_NOT_STARTED) {
            if (STATE_UPDATER.compareAndSet(this, ST_NOT_STARTED, ST_STARTED)) {
                doStartThread();
            }
        }
    }
```

























