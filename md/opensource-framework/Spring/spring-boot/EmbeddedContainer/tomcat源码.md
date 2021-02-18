```




SocketProcessorBase
NioEndpoint$SocketProcessor

AbstractProtocol$ConnectionHandler#process
-- AbstractHttp11Protocol#createProcessor
-- AbstractProcessor 构造函数 new org.apache.coyote.Request(), new Response()
AbstractProcessorLight#process
Http11Processor#service
CoyoteAdapter#service
-- org.apache.catalina.connector.Request
-- org.apache.catalina.connector.Response
StandardEngineValve
StandardHostValve#invoke
-- request.getRequest() 包装为  Request
StandardContextValve
AuthenticatorBase#invoke
StandardWrapperValve#invoke
-- ApplicationFilterChain filterChain =ApplicationFilterFactory
   .createFilterChain(request, wrapper, servlet);






```

#### CoyoteAdapter#service



```
    @Override
    public void service(org.apache.coyote.Request req, org.apache.coyote.Response res)
            throws Exception {

        Request request = (Request) req.getNote(ADAPTER_NOTES);
        Response response = (Response) res.getNote(ADAPTER_NOTES);

        // 创建 connector包下的 request，response并关联 coyote包下
        if (request == null) {
            // Create objects
            request = connector.createRequest();
            request.setCoyoteRequest(req);
            response = connector.createResponse();
            response.setCoyoteResponse(res);

            // Link objects
            request.setResponse(response);
            response.setRequest(request);

            // Set as notes
            req.setNote(ADAPTER_NOTES, request);
            res.setNote(ADAPTER_NOTES, response);

            // Set query string encoding
            req.getParameters().setQueryStringCharset(connector.getURICharset());
        }
```





# tomcat session 设计分析

------

tomcat session 组件，其中`Context`对应一个`webapp`应用，每个`webapp`有多个`HttpSessionListener`， 并且每个应用的`session`是独立管理的，而`session`的创建、销毁由`Manager`组件完成，它内部维护了 N 个`Session`实例对象。在前面的文章中，我们分析了`Context`组件，它的默认实现是`StandardContext`，它与`Manager`是一对一的关系，`Manager`创建、销毁会话时，需要借助`StandardContext`获取 `HttpSessionListener`列表并进行事件通知，而`StandardContext`的后台线程会对`Manager`进行过期` Session 的清理工作

```
org.apache.catalina
```

`org.apache.catalina.Manager`接口的主要方法如下所示，它提供了 `Context`、`org.apache.catalina.SessionIdGenerator`的`getter/setter`接口，以及创建、添加、移除、查找、遍历`Session`的 API 接口，此外还提供了`Session`持久化的接口（load/unload） 用于加载/卸载会话信息，当然持久化要看不同的实现类

#### Manager

```
ManagerBase
PersistentManagerBase
StandardManager 默认
```

```javascript
public interface Manager {
    public Context getContext();
    public void setContext(Context context);
    public SessionIdGenerator getSessionIdGenerator();
    public void setSessionIdGenerator(SessionIdGenerator sessionIdGenerator);
    public void add(Session session);
    public void addPropertyChangeListener(PropertyChangeListener listener);
    public void changeSessionId(Session session);
    public void changeSessionId(Session session, String newId);
    public Session createEmptySession();
    public Session createSession(String sessionId);
    public Session findSession(String id) throws IOException;
    public Session[] findSessions();
    public void remove(Session session);
    public void remove(Session session, Boolean update);
    public void removePropertyChangeListener(PropertyChangeListener listener);
    public void unload() throws IOException;
    public void backgroundProcess();
    public Boolean willAttributeDistribute(String name, Object value);
}
```

tomcat8.5 提供了 4 种实现，默认使用 StandardManager，tomcat 还提供了集群会话的解决方案，但是在实际项目中很少运用，关于 Manager 的详细配置信息请参考 tomcat 官方文档

- **StandardManager**：Manager 默认实现，在内存中管理 session，宕机将导致 session 丢失；但是当调用 Lifecycle 的 start/stop 接口时，将采用 jdk 序列化保存 Session 信息，因此当 tomcat 发现某个应用的文件有变更进行 reload 操作时，这种情况下不会丢失 Session 信息
- **DeltaManager**：增量 Session 管理器，用于Tomcat集群的会话管理器，某个节点变更 Session 信息都会同步到集群中的所有节点，这样可以保证 Session 信息的实时性，但是这样会带来较大的网络开销
- **BackupManager**：用于 Tomcat 集群的会话管理器，与DeltaManager不同的是，某个节点变更 Session 信息的改变只会同步给集群中的另一个 backup 节点
- **PersistentManager**：当会话长时间空闲时，将会把 Session 信息写入磁盘，从而限制内存中的活动会话数量；此外，它还支持容错，会定期将内存中的 Session 信息备份到磁盘

Session 相关的类图如下所示，StandardSession 同时实现了 javax.servlet.http.HttpSession、org.apache.catalina.Session 接口，并且对外提供的是 StandardSessionFacade 外观类，保证了 StandardSession 的安全，避免开发人员调用其内部方法进行不当操作。而 org.apache.catalina.connector.Request 实现了 javax.servlet.http.HttpServletRequest 接口，它持有 StandardSession 的引用，对外也是暴露 RequestFacade 外观类。而 StandardManager 内部维护了其创建的 StandardSession，是一对多的关系，并且持有 StandardContext 的引用，而 StandardContext 内部注册了 webapp 所有的 HttpSessionListener 实例。





# 创建Session

------

我们以 HttpServletRequest#getSession() 作为切入点，对 Session 的创建过程进行分析

```javascript
public class SessionExample extends HttpServlet {
    public void doGet(HttpServletRequest request, HttpServletResponse response)
            throws IOException, ServletException  {
        HttpSession session = request.getSession();
        // other code......
    }
}
```

整个流程图如下图所示：

tomcat 创建 session 的流程如上图所示，我们的应用程序拿到的 HttpServletRequest 是 org.apache.catalina.connector.RequestFacade(除非某些 Filter 进行了特殊处理)，它是 org.apache.catalina.connector.Request 的门面模式。首先，会判断 Request 对象中是否存在 Session，如果存在并且未失效则直接返回，因为在 tomcat 中 Request 对象是被重复利用的，只会替换部分组件，所以会进行这步判断。此时，如果不存在 Session，则尝试根据 requestedSessionId 查找 Session，而该 requestedSessionId 会在 HTTP Connector 中进行赋值（如果存在的话），如果存在 Session 的话则直接返回，如果不存在的话，则创建新的 Session，并且把 sessionId 添加到 Cookie 中，后续的请求便会携带该 Cookie，这样便可以根据 Cookie 中的sessionId 找到原来创建的 Session 了

在上面的过程中，Session 的查找、创建都是由 Manager 完成的，下面我们分析下 StandardManager 创建 Session 的具体逻辑。首先，我们来看下 StandardManager 的类图，它也是个 Lifecycle 组件，并且 ManagerBase 实现了主要的逻辑。



整个创建 Session 的过程比较简单，就是实例化 StandardSession 对象并设置其基本属性，以及生成唯一的 sessionId，其次就是记录创建时间，关键代码如下所示：

```javascript
public Session createSession(String sessionId) {
    // 限制 session 数量，默认不做限制，maxActiveSessions = -1
    if ((maxActiveSessions >= 0) &&
                (getActiveSessions() >= maxActiveSessions)) {
        rejectedSessions++;
        throw new TooManyActiveSessionsException(sm.getString("managerBase.createSession.ise"), maxActiveSessions);
    }
    // 创建 StandardSession 实例，子类可以重写该方法
    Session session = createEmptySession();
    // 设置属性，包括创建时间，最大失效时间
    session.setNew(true);
    session.setValid(true);
    session.setCreationTime(System.currentTimeMillis());
    // 设置最大不活跃时间(单位s)，如果超过这个时间，仍然没有请求的话该Session将会失效
    session.setMaxInactiveInterval(getContext().getSessionTimeout() * 60);
    String id = sessionId;
    if (id == null) {
        id = generateSessionId();
    }
    session.setId(id);
    sessionCounter++;
    // 这个地方不是线程安全的，可能当时开发人员认为计数器不要求那么准确
    // 将创建时间添加到LinkedList中，并且把最先添加的时间移除，主要还是方便清理过期session
    SessionTiming timing = new SessionTiming(session.getCreationTime(), 0);
    synchronized (sessionCreationTiming) {
        sessionCreationTiming.add(timing);
        sessionCreationTiming.poll();
    }
    return (session);
}
```

在 tomcat 中是可以限制 session 数量的，如果需要限制，请指定 Manager 的 maxActiveSessions 参数，默认不做限制，不建议进行设置，但是如果存在恶意攻击，每次请求不携带 Cookie 就有可能会频繁创建 Session，导致 Session 对象爆满最终出现 OOM。另外 sessionId 采用随机算法生成，并且每次生成都会判断当前是否已经存在该 id，从而避免 sessionId 重复。而 StandardManager 是使用 ConcurrentHashMap 存储 session 对象的，sessionId 作为 key，org.apache.catalina.Session 作为 value。此外，值得注意的是 StandardManager 创建的是 tomcat 的 org.apache.catalina.session.StandardSession，同时他也实现了 servlet 的 HttpSession，但是为了安全起见，tomcat 并不会把这个 StandardSession 直接交给应用程序，因此需要调用 org.apache.catalina.Session#getSession() 获取 HttpSession。

**我们再来看看 StandardSession 的内部结构**

- **attributes**：使用 ConcurrentHashMap 解决多线程读写的并发问题
- **creationTime**：Session 的创建时间
- **expiring**：用于标识 Session 是否过期
- **expiring**：用于标识 Session 是否过期
- **lastAccessedTime**：上一次访问的时间，用于计算 Session 的过期时间
- **maxInactiveInterval**：Session 的最大存活时间，如果超过这个时间没有请求，Session 就会被清理、
- **listeners**：这是 tomcat 的 SessionListener，并不是 servlet 的 HttpSessionListener
- **facade**：HttpSession 的外观模式，应用程序拿到的是该对象

```javascript
public class StandardSession implements HttpSession, Session, Serializable {
    protected ConcurrentMap<String, Object> attributes = new ConcurrentHashMap<>();
    protected long creationTime = 0L;
    protected transient volatile boolean expiring = false;
    protected transient StandardSessionFacade facade = null;
    protected String id = null;
    protected volatile long lastAccessedTime = creationTime;
    protected transient ArrayList<SessionListener> listeners = new ArrayList<>();
    protected transient Manager manager = null;
    protected volatile int maxInactiveInterval = -1;
    protected volatile boolean isNew = false;
    protected volatile boolean isValid = false;
    protected transient Map<String, Object> notes = new Hashtable<>();
    protected transient Principal principal = null;
}
```

# Session清理

------

## Background 线程

前面我们分析了 Session 的创建过程，而 Session 会话是有时效性的，下面我们来看下 tomcat 是如何进行失效检查的。在分析之前，我们先回顾下 Container 容器的 Background 线程。

tomcat 所有容器组件，都是继承至 ContainerBase 的，包括 StandardEngine、StandardHost、StandardContext、StandardWrapper，而 ContainerBase 在启动的时候，如果 backgroundProcessorDelay 参数大于 0 则会开启 ContainerBackgroundProcessor 后台线程，调用自己以及子容器的 backgroundProcess 进行一些后台逻辑的处理，和 Lifecycle 一样，这个动作是具有传递性的，也就是说子容器还会把这个动作传递给自己的子容器，如下图所示，其中父容器会遍历所有的子容器并调用其 backgroundProcess 方法，而 StandardContext 重写了该方法，它会调用 StandardManager#backgroundProcess() 进而完成 Session 的清理工作。看到这里，不得不感慨 tomcat 的责任

![img](https://ask.qcloudimg.com/http-save/5261283/6i77ab41t5.png?imageView2/2/w/1620)

关键代码如下所示：

```javascript
ContainerBase.java(省略了异常处理代码)

protected synchronized void startInternal() throws LifecycleException {
    // other code......
    // 开启ContainerBackgroundProcessor线程用于处理子容器，默认情况下backgroundProcessorDelay=-1，不会启用该线程
    threadStart();
}

protected class ContainerBackgroundProcessor implements Runnable {
    public void run() {
        // threadDone 是 volatile 变量，由外面的容器控制
        while (!threadDone) {
            try {
                Thread.sleep(backgroundProcessorDelay * 1000L);
            } catch (InterruptedException e) {
                // Ignore
            }
            if (!threadDone) {
                processChildren(ContainerBase.this);
            }
        }
    }

    protected void processChildren(Container container) {
        container.backgroundProcess();
        Container[] children = container.findChildren();
        for (int i = 0; i < children.length; i++) {
            // 如果子容器的 backgroundProcessorDelay 参数小于0，则递归处理子容器
            // 因为如果该值大于0，说明子容器自己开启了线程处理，因此父容器不需要再做处理
            if (children[i].getBackgroundProcessorDelay() <= 0) {
                processChildren(children[i]);
            }
        }
    }
}
```

## Session 检查

backgroundProcessorDelay 参数默认值为 -1，单位为秒，即默认不启用后台线程，而 tomcat 的 Container 容器需要开启线程处理一些后台任务，比如监听 jsp 变更、tomcat 配置变动、Session 过期等等，因此 StandardEngine 在构造方法中便将 backgroundProcessorDelay 参数设为 10（当然可以在 server.xml 中指定该参数），即每隔 10s 执行一次。那么这个线程怎么控制生命周期呢？我们注意到 ContainerBase 有个 threadDone 变量，用 volatile 修饰，如果调用 Container 容器的 stop 方法该值便会赋值为 false，那么该后台线程也会退出循环，从而结束生命周期。另外，有个地方需要注意下，父容器在处理子容器的后台任务时，需要判断子容器的 backgroundProcessorDelay 值，只有当其小于等于 0 才进行处理，因为如果该值大于0，子容器自己会开启线程自行处理，这时候父容器就不需要再做处理了

前面分析了容器的后台线程是如何调度的，下面我们重点来看看 webapp 这一层，以及 StandardManager 是如何清理过期会话的。StandardContext 重写了 backgroundProcess 方法，除了对子容器进行处理之外，还会对一些缓存信息进行清理，关键代码如下所示：

```javascript
StandardContext.java

@Override
public void backgroundProcess() {
    if (!getState().isAvailable())
        return;
    // 热加载 class，或者 jsp
    Loader loader = getLoader();
    if (loader != null) {
        loader.backgroundProcess();
    }
    // 清理过期Session
    Manager manager = getManager();
    if (manager != null) {
        manager.backgroundProcess();
    }
    // 清理资源文件的缓存
    WebResourceRoot resources = getResources();
    if (resources != null) {
        resources.backgroundProcess();
    }
    // 清理对象或class信息缓存
    InstanceManager instanceManager = getInstanceManager();
    if (instanceManager instanceof DefaultInstanceManager) {
        ((DefaultInstanceManager)instanceManager).backgroundProcess();
    }
    // 调用子容器的 backgroundProcess 任务
    super.backgroundProcess();
}
```

StandardContext 重写了 backgroundProcess 方法，在调用子容器的后台任务之前，还会调用 Loader、Manager、WebResourceRoot、InstanceManager 的后台任务，这里我们只关心 Manager 的后台任务。弄清楚了 StandardManager 的来龙去脉之后，我们接下来分析下具体的逻辑。

StandardManager 继承至 ManagerBase，它实现了主要的逻辑，关于 Session 清理的代码如下所示。backgroundProcess 默认是每隔10s调用一次，但是在 ManagerBase 做了取模处理，默认情况下是 60s 进行一次 Session 清理。tomcat 对 Session 的清理并没有引入时间轮，因为对 Session 的时效性要求没有那么精确，而且除了通知 SessionListener。

```javascript
ManagerBase.java

public void backgroundProcess() {
    // processExpiresFrequency 默认值为 6，而backgroundProcess默认每隔10s调用一次，也就是说除了任务执行的耗时，每隔 60s 执行一次
    count = (count + 1) % processExpiresFrequency;
    if (count == 0) // 默认每隔 60s 执行一次 Session 清理
        processExpires();
}

/**
 * 单线程处理，不存在线程安全问题
 */
public void processExpires() {
    long timeNow = System.currentTimeMillis();
    Session sessions[] = findSessions();    // 获取所有的 Session
    int expireHere = 0 ;
    for (int i = 0; i < sessions.length; i++) {
        // Session 的过期是在 isValid() 里面处理的
        if (sessions[i]!=null && !sessions[i].isValid()) {
            expireHere++;
        }
    }
    long timeEnd = System.currentTimeMillis();
    // 记录下处理时间
    processingTime += ( timeEnd - timeNow );
}
```

## 清理过期 Session

在上面的代码，我们并没有看到太多的过期处理，只是调用了 sessions[i].isValid()，原来清理动作都在这个方法里面处理的，相当的隐晦。在 StandardSession#isValid() 方法中，如果 now - thisAccessedTime >= maxInactiveInterval则判定当前 Session 过期了，而这个 thisAccessedTime 参数在每次访问都会进行更新

```javascript
public boolean isValid() {
    // other code......
    // 如果指定了最大不活跃时间，才会进行清理，这个时间是 Context.getSessionTimeout()，默认是30分钟
    if (maxInactiveInterval > 0) {
        int timeIdle = (int) (getIdleTimeInternal() / 1000L);
        if (timeIdle >= maxInactiveInterval) {
            expire(true);
        }
    }
    return this.isValid;
}
```

而 expire 方法处理的逻辑较繁锁，下面我用伪代码简单地描述下核心的逻辑，由于这个步骤可能会有多线程进行操作，因此使用 synchronized 对当前 Session 对象加锁，还做了双重校验，避免重复处理过期 Session。它还会向 Container 容器发出事件通知，还会调用 HttpSessionListener 进行事件通知，这个也就是我们 web 应用开发的 HttpSessionListener 了。由于 Manager 中维护了 Session 对象，因此还要将其从 Manager 移除。Session 最重要的功能就是存储数据了，可能存在强引用，而导致 Session 无法被 gc 回收，因此还要移除内部的 key/value 数据。由此可见，tomcat 编码的严谨性了，稍有不慎将可能出现并发问题，以及出现内存泄露

```javascript
public void expire(boolean notify) {
    1、校验 isValid 值，如果为 false 直接返回，说明已经被销毁了
    synchronized (this) {   // 加锁
        2、双重校验 isValid 值，避免并发问题
        Context context = manager.getContext();
        if (notify) {   
            Object listeners[] = context.getApplicationLifecycleListeners();
            HttpSessionEvent event = new HttpSessionEvent(getSession());
            for (int i = 0; i < listeners.length; i++) {
            3、判断是否为 HttpSessionListener，不是则继续循环
            4、向容器发出Destory事件，并调用 HttpSessionListener.sessionDestroyed() 进行通知
            context.fireContainerEvent("beforeSessionDestroyed", listener);
            listener.sessionDestroyed(event);
            context.fireContainerEvent("afterSessionDestroyed", listener);
        }
        5、从 manager 中移除该  session
        6、向 tomcat 的 SessionListener 发出事件通知，非 HttpSessionListener
        7、清除内部的 key/value，避免因为强引用而导致无法回收 Session 对象
    }
}
```

由前面的分析可知，tomcat 会根据时间戳清理过期 Session，那么 tomcat 又是如何更新这个时间戳呢？我们在 StandardSession#thisAccessedTime 的属性上面打个断点，看下调用栈。原来 tomcat 在处理完请求之后，会对 Request 对象进行回收，并且会对 Session 信息进行清理，而这个时候会更新 thisAccessedTime、lastAccessedTime 时间戳。此外，我们通过调用 request.getSession() 这个 API 时，在返回 Session 时会调用 Session#access() 方法，也会更新 thisAccessedTime 时间戳。这样一来，每次请求都会更新时间戳，可以保证 Session 的鲜活时间

方法调用栈如下所示：

![img](https://ask.qcloudimg.com/http-save/5261283/9gtrpuhg40.png?imageView2/2/w/1620)

关键代码如下所示：

```javascript
org.apache.catalina.connector.Request.java

protected void recycleSessionInfo() {
    if (session != null) {  
        session.endAccess();    // 更新时间戳
    }
    // 回收 Request 对象的内部信息
    session = null;
    requestedSessionCookie = false;
    requestedSessionId = null;
    requestedSessionURL = false;
    requestedSessionSSL = false;
}
```

org.apache.catalina.session.StandardSession.java

```javascript
public void endAccess() {
    isNew = false;
    if (LAST_ACCESS_AT_START) {     // 可以通过系统参数改变该值，默认为false
        this.lastAccessedTime = this.thisAccessedTime;
        this.thisAccessedTime = System.currentTimeMillis();
    } else {
        this.thisAccessedTime = System.currentTimeMillis();
        this.lastAccessedTime = this.thisAccessedTime;
    }
}

public void access() {
    this.thisAccessedTime = System.currentTimeMillis();
}
```

# HttpSessionListener

------

## 创建通知

前面我们分析了 Session 的创建过程，但是在整个创建流程中，似乎没有看到关于 HttpSessionListener 的创建通知。原来，在给 Session 设置 id 的时候会进行事件通知，和 Session 的销毁一样，也是非常的隐晦，个人感觉这一块设计得不是很合理。

创建通知这块的逻辑很简单，首先创建 HttpSessionEvent 对象，然后遍历 Context 内部的 LifecycleListener，并且判断是否为 HttpSessionListener 实例，如果是的话则调用 HttpSessionListener#sessionCreated() 方法进行事件通知。

```javascript
public void setId(String id, boolean notify) {
    // 省略部分代码
    if (notify) {
        tellNew();
    }
}

public void tellNew() {

    // 通知 org.apache.catalina.SessionListener
    fireSessionEvent(Session.SESSION_CREATED_EVENT, null);

    // 获取 Context 内部的 LifecycleListener，并判断是否为 HttpSessionListener
    Context context = manager.getContext();
    Object listeners[] = context.getApplicationLifecycleListeners();
    if (listeners != null && listeners.length > 0) {
        HttpSessionEvent event = new HttpSessionEvent(getSession());
        for (int i = 0; i < listeners.length; i++) {
            if (!(listeners[i] instanceof HttpSessionListener))
                continue;
            HttpSessionListener listener = (HttpSessionListener) listeners[i];
            context.fireContainerEvent("beforeSessionCreated", listener);   // 通知 Container 容器
            listener.sessionCreated(event);
            context.fireContainerEvent("afterSessionCreated", listener);
        }
    }
}
```

## 销毁通知

我们在前面分析清理过期 Session时大致分析了 Session 销毁时会触发 HttpSessionListener 的销毁通知，这里不再重复了。





创建 session和销毁session的时机



1、创建session的时候会附带着创建一个cookie，它的MaxAge为-1，也就是说只能存在于内存中。当浏览器端禁用cookie时，这个cookie依然会被创建。

2、当浏览器提交的请求中有jsessionid参数或cookie报头时，容器不再新建session，而只是找到先前的session进行关联。这里又分为两种情况：
​    1）使用jsessionid。该值若能与现有的session对应，就不创建新的session，否则，仍然创建新的session。
​    2）使用cookie。该值若能与现有的session对应，也不创建新的session；但若没有session与之对应（就如上面的重启服务器之后）容器会根据cookie信息恢复这个与之对应的session，就好像是以前有过一样。

3、session何时被销毁？
​         当我们关闭浏览器，再打开它，连接服务器时，服务器端会分配一个新的session，也就是说会启动一个新的会话。那么原来的session是不是被销毁了呢？
​         通过实现一个SessionListener可以发现，当浏览器关闭时，原session并没有被销毁（destory方法没有执行），而是等到timeout到期，才销毁这个session。关闭浏览器只是在客户端的内存中清除了与原会话相关的cookie，再次打开浏览器进行连接时，浏览器无法发送cookie信息，所以服务器会认为是一个新的会话。因此，如果有某些与session关联的资源想在关闭浏览器时就进行清理（如临时文件等），那么应该发送特定的请求到服务器端，而不是等到session的自动清理。

# Tomcat源码学习--Session创建销毁



 每个web后台开发人员肯定对Session有所接触和了解，简单来说就是后台服务器维护了一个存在有效时间和范围限制的缓存数据。

 接下来我们通过这篇博客来分析一下tomcat创建、使用和销毁Session的相关过程。首先我们需要看一下tomcat中与session相关的所有类，如下：

HttpSession：是javax.servlet包中提供的session接口

StandardSessionFacade：HttpSession的实现类，StandardSession的外观类

Session：tomcat内部session接口

StandardSession：实现了Session和HttpSession，是tomcat维护session的载体

##### 一、创建获取session

1、通过request获取session

```java
HttpSession session  = request.getSession();
```

2、当session不存在是时创建新的session

tomcat实现了接口HttpServletRequest得的RequestFacade（）

```java
@Override
    public HttpSession getSession() {

        if (request == null) {
          throw new IllegalStateException(                           sm.getString("requestFacade.nullRequest"));

        }
   return getSession(true);



    }
@Override
   public HttpSession getSession(boolean create) {
     if (request == null) {
         throw new IllegalStateException(
sm.getString("requestFacade.nullRequest"));
       }
     if (SecurityUtil.isPackageProtectionEnabled()){
        return AccessController.
             doPrivileged(new GetSessionPrivilegedAction(create));

        } else {
         return request.getSession(create);

        }
    }
```

在tomcat的Request类中获取session



在doGetSession中获取session，doGetSession会包括session的查找创建等操作。

```java
protected Session doGetSession(boolean create) {

        // There cannot be a session if no context has been assigned yet
		//session的作用域范围是上下午
       Context context = getContext();



        if (context == null) {



            return (null);



        }



 



        // Return the current session if it exists and is valid



        if ((session != null) && !session.isValid()) {



            session = null;



        }



        if (session != null) {



            return (session);



        }



 



        // Return the requested session if it exists and is valid



		//获取管理器



        Manager manager = context.getManager();



        if (manager == null) {



            return (null);      // Sessions are not supported



        }



		//requestedSessionId是用来区分session的key值，如果存在则直接获取



        if (requestedSessionId != null) {



            try {



                session = manager.findSession(requestedSessionId);



            } catch (IOException e) {



                session = null;



            }



            if ((session != null) && !session.isValid()) {



                session = null;



            }



            if (session != null) {



                session.access();



                return (session);



            }



        }



 



        // Create a new session if requested and the response is not committed



        if (!create) {



            return (null);



        }



        if (response != null



                && context.getServletContext()



                        .getEffectiveSessionTrackingModes()



                        .contains(SessionTrackingMode.COOKIE)



                && response.getResponse().isCommitted()) {



            throw new IllegalStateException(



                    sm.getString("coyoteRequest.sessionCreateCommitted"));



        }



 



        // Re-use session IDs provided by the client in very limited



        // circumstances.



		//如果session为空或者失效就创建新的session id



        String sessionId = getRequestedSessionId();



        if (requestedSessionSSL) {



            // If the session ID has been obtained from the SSL handshake then



            // use it.



        } else if (("/".equals(context.getSessionCookiePath())



                && isRequestedSessionIdFromCookie())) {



            if (context.getValidateClientProvidedNewSessionId()) {



                boolean found = false;



                for (Container container : getHost().findChildren()) {



                    Manager m = ((Context) container).getManager();



                    if (m != null) {



                        try {



                            if (m.findSession(sessionId) != null) {



                                found = true;



                                break;



                            }



                        } catch (IOException e) {



                            // Ignore. Problems with this manager will be



                            // handled elsewhere.



                        }



                    }



                }



                if (!found) {



                    sessionId = null;



                }



            }



        } else {



            sessionId = null;



        }



		//通过sessionid创建新的session



        session = manager.createSession(sessionId);



 



        // Creating a new session cookie based on that session



		//将sessionid添加到cookie中，这样可以通过cookie获取到sessionid并根据sessionid来获取session



        if (session != null



                && context.getServletContext()



                        .getEffectiveSessionTrackingModes()



                        .contains(SessionTrackingMode.COOKIE)) {



            Cookie cookie =



                ApplicationSessionCookieConfig.createSessionCookie(



                        context, session.getIdInternal(), isSecure());



 



            response.addSessionCookieInternal(cookie);



        }



 



        if (session == null) {



            return null;



        }



		//返回session



        session.access();



        return session;



    }
```

session查找，当requestedSessionId不为空时会在管理器中查找session

在ManagerBase基类中提供了findSession方法

session对象最终是保存在一个ConcurrentHashMap中

```java
protected Map<String, Session> sessions = new ConcurrentHashMap<>();
```

当查找不到session存在时会创建新的session

```java
session = manager.createSession(sessionId);
```

createSession中会根据sessionid创建一个新的

创建新的session就是一个StandardSession对象

session id会通过cookie保存到浏览器中，名称就是我们经常见到的是JSESSIONID了。



![img](data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAABAIAAADJCAYAAAC5QJDRAAAgAElEQVR4Ae3dC7hVVb338bHL5wgqysUbkBYX8RAXMSjICjmpBzH1TRAjRE3icso4Xt6E8OQlLAn0kczMEOKY4uW0AxVDINGDmggISUKSCmIakCmgoYL1vO73+Q37T8eee6611957rb3nXOs7nmfvNde8jDnGZ8695hr/OcbcVTU1NTUuhWn79u2uU6dOKSxZtouEa/GPH6bFN1WOuOJaGoHS5Mr5WnxXTItvqhxxxbU0AsXPlXO1+KbKEVdcTeAjNsErAggggAACCCCAAAIIIIAAAgiUvwCBgPI/xtQQAQQQQAABBBBAAAEEEEAAgUiAQEBEwQQCCCCAAAIIIIAAAggggAAC5S9AIKD8jzE1RAABBBBAAAEEEEAAAQQQQCASqFq3bl0qHxYYlZAJBBBAAAEEEEAAAQQQQAABBBAomkBVc/3XgPnz57sxY8YUreBkhAACCCCAAAIIIIAAAggggAACDRdgaEDDzdgCAQQQQAABBBBAAAEEEEAAgcwK7JfZklNwBFIosG7duhSWqvFF6t+/f+M3ZksEEEAAAQQQQAABBBBIpQCBgFQeFgqVZYFyaTyXW1Ajy+cUZUcAAQQQQAABBBBAoJgCDA0opiZ5IYAAAggggAACCCCAAAIIIJByAQIBKT9AFA8BBBBAAAEEEEAAAQQQQACBYgoQCCimJnkhgAACCCCAAAIIIIAAAgggkHIBAgEpP0AUDwEEEEAAAQQQQAABBBBAAIFiChAIKKYmeSGAAAIIIIAAAggggAACCCCQcoEWDQS89tprbubMme5vf/tbxKRpzdMyEgIIIIAAAggggAACCCCAAAIIFFegRf994D/+8Q/3xhtv+Ib/5MmTfc0UBPj73//utKyh6b333nNz5851Z5xxhjv66KOjzZctW+Y6duzo+vbtG81r7ESufTQ2v0K20z5//OMfu/Xr1/vV27Rp46644opadSwkn6R13nrrLTdr1iw3duzYouSXtI9wXrwuWjZlypSiHJtwP2mc3rp1q/vP//xPt2PHDl+82267zX3qU59qdFEXLVrktz3zzDNr5aH5f/rTn9ykSZNqzS/Wm+uvvz46Fy1POyeXL1/uOnfu7IYOHWqL6n3V3+e2bdv8OVjvyqyAAAIIIIAAAggggAACTRZo0UDAxz72MTdu3DjfeFcAQElBAM3TMtKHAgcccICbPn26b6y/8sorbt68ee7SSy91hxxyyIcrNWJK219zzTWN2LLxmyTV5eMf/3iT69L4EpV+SzXOFy5c6O68807Xtm1bv0M1mvft2+datWpV1ALEAwNFzdw5d/nll/ssn332Wbd27dpaDXgFlBqaGhI0aGjerI8AAggggAACCCCAAAJ1BVp0aICK06NHD/eVr3zF9wxQ7wBNax4pt8ARRxzhDjvsMKe7+VlP5VSXXMfizTffdGvWrHE/+tGPoiCA1j355JOLHgTIVQbmI4AAAggggAACCCCAAAIm0KI9AlQIPRPgvvvucwcffLAvk6aPOeaY6L0VtJivupv+yCOP+CzPP/98341Zjervf//7bvv27bXm643ufM6YMcPPP+GEE/yr/dLd+euuu87t2bPHderUyX33u9/1d7a1j/bt27ulS5e6s846yw9NsDz69evnu4jvv//+lk2jX7X/Bx980G//8ssv+/2rLvEyrVq1qlb3a9vunHPOcbfffrubMGGCL3dYVyvnXXfd5QYMGOC778d7IzR12IU9C0IBAaX40AEbNpBUrhUrVri9e/e6F1980XdVD/2VV3icrS4yt7qrC3t1dbWzbu0aTpK0H20T5mXnjC9wAb9eeuklp/pZT4CkTW6++Wb3i1/8wi869dRT/XG0ngL5llle6nEwZ84cP4Rkw4YNfrZ6BqjHgc5rnYdKNhxBwYlLLrnEbdy40Z+bGnrSpUsXy67Rr3Kyc0XT3bt39+en/q50LDUs4o477qhlrnNIST0DNB0e0/C4hX+jOtYaVqG/MXoUNPpwsSECCCCAAAIIIIBAhQq0aCDAHgxowwF0DDTGX8ME9MwACw405Ni8++67burUqXU2USNEyRoq6sJs4/179uzpu9xr7LOSGhxqMA0aNMhPq+Fi3fLVUPzpT3/qn0NgDcqbbrrJqbGoZQsWLIi6SquBqmVKqpfl4Wc04dfzzz/vVE81LtWQVsPvm9/8prvooouiRm68TCNGjPB1Ut00HGDTpk2+kfYv//IvUUlUfnX1VsNfSY0yNbbVYFYDTs9Y0HY1NTV+DPqxxx7rNm/e7OTXkBQeo7ChZ8dDvULU/dyOg57v8MQTTyT6qYGr5yVofZXX/HWclcK6aNq6rstMjVTN03bqpn/uuecm7iffOVNIvf/85z87DX3IldTQV5K9kjXqNcY/3zK/snPud7/7ne9xoKCGggcWCFAQQL0Qvv71r/tggBr/N9xwg+vatat7/PHH3fDhw30QyPIpxauCVAqO6VgqOKWgWGhuxyPctx1TPU9BAQqd7zrX9DepXhRq+Cs/BTj0noQAAggggAACCCCAAAINE2jRQMBf//pX99GPftQ/E8CGA+j5AGooaFljAgHh+HOjUENPSQ3NnTt3Rnf3bfkXvvAFHwjQemr0K+kusRobajydcsop0YP01CDp06ePX0fLVq5c6X8sLzVstR8lNVjsrn/r1q19o856DNj6hb6GjWe78215qzwql1KuMqmB2KFDh6gBr4ezKTgQJjX21VPCekto2UknneQbW2rQKTCi7TRP66pxq3rZ3fwwr3zTdowUkNCDChXM0N14NVzVqyEM5Og46A5xLj81LO3BkCqfGouvv/66331YP1umY6oksyFDhvhpBTIU0FCK76e+c8ZvVM8vPe/iySefTFxLdVY677zzouWDBw/2DXbrLZG0TMdZSQEM/Z3ovLIeBJaR8v7jH//oRo4cabP83f/du3f7Z3CoF4jK1pQHFkYZ55hQ2e05Ft26dfPnkFZVcEfnUlIKj6mCT3auaV0dRyXlSRAgSY95CCCAAAIIIIAAAgjUL9CigQDdkb3sssuihoKKq4BAfF791Sh8DWuEWuPRtlQQQA0TBSHszrSWqRGSL6lRnHRXM76N1rG7mIcffniDhwbkKnd8P3qfq0zqsm13nbWeGlPWMLZ8krq9W2BDT6JXMOG4447zd951p1a9BSwgYXkU+qr96660GrNmKJukYEmSX6H7ach68f1MnDjRNcQ+aV/t2rXzgQY13vMND0jatr55On5VVVU5Hzqohn782QTKU8MAdC6ox8HVV1/t77wXY2hAfeVt7HLVU8chHuxobH5shwACCCCAAAIIIIBAJQu0+MMC7W5heBCS5oXLGzutBqvu+KrhGU8KAqhRq6Q7sfpXb0q6W/zUU09FDWY1fq3rtRp1zz33nO+O71eu55fqpe7MatDY3d56Nmnw4nxlsl4D6mafdDdVd2llEw8OyE1BGzUc1VhUPeS4ePHiBg8LiFdIZVIvDQ1LsEaenmeQlJL8tJ0FKrSdAhV6kKKShglYsmWFnFvhftR4z3XOWN71vcpMfuqWbz0A9HrvvfdGm+q/CVhSt331srCeFknLLKCgXg8XXHCBH+9vvQQsH/NUfrmShh+MHz8+OqdzrdfS82WhXjH6+1PSOZr0d9zS5WT/CCCAAAIIIIAAAghkQaBFewS0BJDGgWvcsV6VrJu9GsYaw6yhAXpYoTXC1HPgs5/9rB+Dr/X1sEAbGqBluoMedmXPdUdd+1y/fr3fp9aJ90jwC4rwK1+Z1KBXsEONZ6tfuEt1w5aDnjdgyR7WZwER627fq1cvp7HvSfnYtoW8qkwaQqHGrnoCqLu6giU2RENDLXRXfvbs2XX89LwCBVUuvvhi/7BGe96A9hs/zuGyXOVSQCHpOMXzsnOmkKCC7UsNbo39//znP+9nKeiifamxroa46qweG0r2sEBN51vmV3bOd+3XeHodOw2NsKS8v/3tb/sgwbRp0/xsy/s3v/mNi8+z7dL4qvNEz44IH4JZyiENaTSgTAgggAACCCCAAAIIFEugqkZPfmuGNH/+fDdmzJhm2BO7qBQBe/ZDmp4av27dOte/f/+yOARpr4s9xFEBLBICCCCAAAIIIIAAAggULtDiQwMKLyprIoAAAh8I6D926Cfff2PACgEEEEAAAQQQQAABBJIFKm5oQDIDcxFAIO0C6gFg/9FC/01C/zayIcMz0l4/yocAAggggAACCCCAQHMJEAhoLmn2U3SBNA0JKHrlyLCOgP6jg/13iToLmYEAAggggAACCCCAAAIFCzA0oGAqVkQAAQQQQAABBBBAAAEEEEAg+wIEArJ/DKkBAggggAACCCCAAAIIIIAAAgULEAgomIoVEUAAAQQQQAABBBBAAAEEEMi+QNW6deua5d8HPvfcc07/i56EAAIIIIAAAggggAACCCCAAAItJ7Df/vvv32x7L5f/r95sYOwIAQQQQAABBBBAAAEEEEAAgSILMDSgyKBkhwACCCCAAAIIIIAAAggggECaBQgEpPnoUDYEEEAAAQQQQAABBBBAAAEEiixAIKDIoGSHAAIIIIAAAggggAACCCCAQJoFCASk+ehQNgQQQAABBBBAAAEEEEAAAQSKLEAgoMigZIcAAggggAACCCCAAAIIIIBAmgUIBKT56FA2BBBAAAEEEEAAAQQQQAABBIosQCCgyKBkhwACCCCAAAIIIIAAAggggECaBQgEpPnoUDYEEEAAAQQQQAABBBBAAAEEiixAIKDIoGSHAAIIIIAAAggggAACCCCAQJoFCASk+ehQNgQQQAABBBBAAAEEEEAAAQSKLEAgoMigZIcAAggggAACCCCAAAIIIIBAmgX2S3Phqqur01w8yoYAAggggAACCCCAAAIIIIBA5gRSHQiQ5uc///nMoVJgBBBAAIHKEvjtb3/rhg0bVlmVprYIIIBAhQosWbKkoj/zK73+pTjtW8KUoQGlOJLkiQACCCCAAAIIIIAAAggggEBKBQgEpPTAUCwEEEAAAQQQQAABBBBAAAEESiFAIKAUquSJAAIIIIAAAggggAACCCCAQEoFCASk9MBQLAQQQAABBBBAAAEEEEAAAQRKIUAgoBSq5IkAAggggAACCCCAAAIIIIBASgUyFwjYt2+f+973vuduvfXWWqRvvfWWn6flJAQQQAABBFpa4Ec/+pFbu3ZtrWLs3bvX/eAHP3AvvfRSrfnhm/vvv9/ph4QAAgggkB0Bfb5PnTrVHXfccdGP3mt+c6Q333zTfetb38p7fSllOeL113/SyXetSyqL6iAzvZKcP3fi51T8e0WhTjoWOj9C28wFAlTZgw46yD3zzDP+p9DKsx4CCCCAAALNKaB/f6t/KximHTt2+LcdO3YMZzONAAIIIFAGAmqj3Hfffe73v/+9/xk4cKAbP358rcZXqarZtm1b95Of/MR17dq1VLuoN9+w/rfccou78cYb89a9kOB4vTst8xVCU51XAwYMaFSNdV7o/NB5Ymm/Xr162XRJX9VwL2YaNWqUe/TRR13Pnj1dq1atipk1eSGAAAIIINBkge7du7sFCxb4L0F24X322WedrrutW7ducv5kgAACCCCQboEvf/nLvoArVqxwNp3uEhevdAp462fXrl21Gp/F2wM5NVUgkz0CVGlFNQ444AD3yCOP1DGw4QO6G6MfG0ag4QMaVqAAguZPnDjRvfbaa36e3mtZOLRA21keixcvrrMfZiCAAAIIIJBLQI3/I444wm3evNmvojsff/jDH1zfvn39e3Xvsy6kubpQxocXhO/j3TAb210wV/mZjwACCCDQdAF95uuzX5/ZSvoct8/+cOiAhoTdfffd0fACvQ+vE/YZr67dY8aMifKwoWRht3p1A9cwNOWnfeW6xjS9doXlkFRmeUybNs398pe/dGeddZZ3sdzWrFkT1c/qbct4dX74oM4jJdnakJD4OaTzRMuV7JyQu9abO3euy2wgQBUaPXq0U4Tt5Zdf9hW0X+ohcPXVV/sumcuXL3fvvvtutM727dvd888/75edeeaZ7qKLLnIXXHCB03pKmzZt8q8KAgwaNCjK449//GOUh+2HVwQQQAABBPIJKJhswwPiwwLUvc+6j1577bVu0aJF+bKqtUwXcnW5VJdT5fHYY49FvQ9qrcgbBBBAAIHUCFjjzT77NXRg9uzZUfnuvPNO/7muz/Rf/epX/vqhdX/+85/7z3h99ivIPH/+/Oizf/Xq1VFjL8rIOb+tbppqe938bMg1JsynsdMKfrz99tu+V0BSmd977z131VVXuXPOOccPp7jkkkv8rl599VX33HPP+XKH9W5sObK+nQwVKAkDOkOHDnXvvPOOb9zffvvt7mtf+1o0JMTOIR33s88+22l5UlIber+kBVmZd8ghh/hgwJIlS/xrWG4NRZg0aVI064tf/KJr166d69SpU7Supk866ST3iU98wq93+OGH+1f1CvjrX/9aa3stUB62bpQxEwgggAACCOQQCIcHxIcF2N2Qhx56yG992mmnRXeMcmQXzdYXKAWu9eXAkq5pdME0DV4RQACB9Aiod5glNdosDRkyxM2YMSNqyKvBrl7Puj5o+LNuWiq1b9/ePyPNttMdXd30VLLPfq0TJgWi1WBUCnsllHJomjVatc8+ffr4Mem2v6QyJz0v56ijjvINW+Wha6iSrnmWj59RQb/sGQHxZz989atf9Te0dYzD5wbYOSSi+PkVsilIkOlAgCqjPxJFuNatWxfVTUEAzdNdfvUOsKEB0QoFTAhd0TYa/gVgsQoCCCCAQKKADQ/YsGGD71VmXwAtCDBixAg3ffp0H9W/5557EvPINVNfluIP/sm1LvMRQAABBFpGQG2SYrYn1KBWb2jd8dW1RL3D0pJyNVrTXOa02DW0HAr8dOjQoaGb1Vo/00MDVBM19NW1X2P4NQRASd3/dXdfy/RcgPXr19eqdH1vtJ260qinAQkBBBBAAIGmCOiuzM9+9jN/N0eBASXd3dizZ4+/y6P36i2gOynxpC+PNrRA4/xsrOT+++/vV1XXPhICCCCAQDoFNBRAzyPTXVu7ox121dZnuHoL2LWhkFooCGCBBQ05Uzf6tKcsljnNpgoA6b8y6BkLSvbdQNMaKqLlSvWdX5kPBKiS+mNQ1xH7EqUvXWr863Xy5Mm+y4zXaMCvCy+80A8PUB76UTcLBRVICCCAAAIINERA16eqqirXr1+/aDN96Tv55JOjcX8aS6k7KfGkbn26wGtsoB4GpF4ASvpCOWXKFD+GNOmhU/F8eI8AAgggUHoB6xpvn8vao3p9WRBA7QkFBmy5Gm2a15Ck4QJ6roDyUG+Ao48+uiGbt8i6ucosF/0nnfjDAlukkCndafycUu8KHf8DDzzQDyOR7axZs3zPQlVB3yWGDx/uz4/6zq+qmpqamuaot7rZ68mFDUnV1dW+Ed6QbVgXAQQQQACB5hbQXXs9lZmEAAIIIFD+Auo1XMmf+ZVe/1Kc4cUwVZBAqdB/VVkWPQJKcTDIEwEEEEAAAQQQQAABBBBAAIFyFCAQUI5HlTohgAACCCCAAAIIIIAAAgggkEMg8/81IEe9mI0AAggggAACCCCAAAIIIIBARQgUOiTAMOgRYBK8IoAAAggggAACCCCAAAIIIFABAql/WGAFHAOqiAACCCCAAAIIIIAAAggggECzCaR+aMDIkSObDYMdIYAAAggg0BgB/ZcbrleNkWMbBBBAIHsClf6ZX+n1L8UZ2xKmDA0oxZEkTwQQQAABBBBAAAEEEEAAAQRSKkAgIKUHhmIhgAACCCCAAAIIIIAAAgggUAoBAgGlUCVPBBBAAAEEEEAAAQQQQAABBFIqQCAgpQeGYiGAAAIIIIAAAggggAACCCBQCgECAaVQJU8EEEAAAQQQQAABBBBAAAEEUiqQuUDA7t273YgRI1y3bt2in8GDB7vNmzenlDh3sfR0yLAel156qdu7d2/uDepZIgNZzJw5s86aq1ev9vtKWlZnZWYggAACCCCAAAIIIIAAAgiUrUDmAgF2JM4880y3ceNGt2zZMj/rlltuaVIj2vIt1qsa3ApYKHCRlLT8O9/5jrv77rvdli1b/M8pp5yStGpR53Xp0qWo+ZEZAggggAACCCCAAAIIIIBAtgT2y1Zx65a2c+fOrn///u6VV15x+/btc61bt667UjPPUeNfd+BzJS2bPXu2++EPf+gGDhwYrXbaaadF08We0H4UcCAhgAACCCCAAAIIIIAAAghUtkBmewTYYdu2bZtbt26db1C3a9fOz7Zu8Nbt3hrl1nXe5uvV7tpbN31bN/4+vq2WK8X3tXz5cjdu3Di3fv16/zNgwIA6QYHHHnvMKYBx/PHHWzVqvcaHP1gZtZKGDmgIgdUh17CIcD2V1cpp5S60PuZRq4C8QQABBBBAAAEEEEAAAQQQyKxAZgMBixYtcr1793Zjx4518+bNc5MnT/YHQQ3X0aNH+y73GjqgIQTqhr9jxw6n4QOHHXaYW7t2rR9SoMZ4IUmNZu1n0qRJ/q76xIkT3c033+weffRRd/nll/t9aF+6437yySe7uXPnun79+vkf7Su861/f/tSAnzZtml9N2+pHScEFBQi0XwU+NCRC+1RvCJVNZQzTr3/9aycj9ToYOXJkuMivW2h9GlL2WjvhDQIIIIAAAggggAACCCCAQCoFMhsIUAP/iSee8A37sCGsu+1KCgYoUKDG8Ouvv+42bdpUq+eADSko5Kg888wzTj0PNKZfd+LVrV/v33//fb9/7WPo0KF1GuOF5B1fJ97DQb0c1BhXHbZu3erv7Kvxr/JrGMQJJ5zgy7Jz584oq1WrVvmAgYxOP/30aL5NNGd9bJ+8IoAAAggggAACCCCAAAIIpEMgs4EA8amRrJ4AajwvXLgwElUjWXfM7SF8jz/+uDv66KOj5Y2dCB/sZ3f/FyxY4NRDQGVQMMC63ufbhx7Yp/XVIC9lsucm5NpHseqTK3/mI4AAAggggAACCCCAAAIIpE8g04EAcfbt29d3zVdXeHWPt0a2BQbU1f6hhx5yHTp08HfvNXRAXeztzrsdEgsUqPGspLvvlmzZvffeG/1nAuWpvJUUjFCjWknbtWrVKm/gQXfpdbdePQzCMfjKs3379r67v5XTHjyoXgA9e/b0vQM0NEDl1/5XrlzphyD06NHD71+/Bg0a5K6//nr/jII5c+ZE822iofWx7XhFAAEEEEAAAQQQQAABBBDIvkDmAwFh93g9A0CNbI2LV/d9dePX8ACNpbfeA3qInx7gp+EEhx56aHQELaBg3f81lMCSuuaroW/PJVC+Dz/8sHvppZecHtan9xqKoMa9niOgMo0aNSrnwwK1fNasWb4ngbbT9panll111VV+1yqnfpQ0T8uUv4IC6n2guikoMGPGDF8/K69eVWb1VJCDnpEQpobWJ9yWaQQQQAABBBBAAAEEEEAAgWwLVNXU1NQ0RxXmz5/vxowZ06BdqZt9/EF3Dcogz8q6m37FFVf4fzuoh/vZfxzIswmLEEAAAQQQSBQo5fUqcYfMRAABBBBoMYFK/8yv9PqX4sRrCdPM9wgoxYEgTwQQQAABBBBAAAEEEEAAAQTKVYBAQLkeWeqFAAIIIIAAAggggAACCCCAQILAfgnzKmKWjdOviMpSSQQQQAABBBBAAAEEEEAAAQT+KUCPAE4FBBBAAAEEEEAAAQQQQAABBCpIIPU9Avbse7+CDgdVRQABBBDIqgDXq6weOcqNAAIINFyg0j/zK73+DT9j6t+iuU3pEVD/MWENBBBAAAEEEEAAAQQQQAABBMpGIPU9AnJJt2n90VyLmI8AAggggECzCoxs1r2xMwQQKERgz97/V8hqrIMAAghUpEBmAwE6WnzAV+Q5S6URQACB1AksfXCBO/WMEakrFwVCoFIFuGFUqUeeeiOAQKECDA0oVIr1EEAAAQQQQAABBBBAAAEEECgDAQIBZXAQqQICCCCAAAIIIIAAAggggAAChQpkOhCw9unVvp5vvrnbnffVka5fr2P8T675w04Z4l7astlvc//CX0XrT518mdu3d6+fr20tH+WpvMN5lrdWvunG66N1tU24zMqk/YRJ+597260+Xyuz8tH+VQ7bt17DcoV5MI0AAgggkC0BXQvsmmIlt+tEeO3QMl0ndL2KXwfCa5FdK+LbantdU/QTpvCap21tea752jZcFl6PwvlhXmG5Nd/KZvW0MofX4nCbuI9dF7W/MKnslpe9Wn1sG80P92Pbh4a2PNzG8rOyh3WNl095ar2k+WGeth+tH87Xvqxu4X6sDGYellnLbL7yC/3C+VoWOll94vtXfrZM2yhpuzCvsGxJddX65v/PLGqdO9pHaGDrhGUP861vf7Y9rwikTcD+vuJ/D/oM1DwtD5P+9sJ17bMy/jcZbpPm6UqvfymOTalNMx0IELiAZlx3rbv4sm+79X940a1a+6zbtWtXdCw+dtRRbsWTa/yyJQ+vcF27dfcXvdWrVvp1tc15F1zotm/f5i+ot8+bE63/ve9Pd9u3/dkN+PRAd8210/2PppXsD1fb234XVP9PdFHXOtr3r355bxR8iArlnGvbtp1T/ueMGu0mfuNbflGbNm3cwkVLfH7Kc+CgE9z4sef7oEG4LdMIIIAAAtkR0Jc7XXMOaXtIVGg1diZ9Y4I78d++GM3ThNa98YYfult+NtdfC448sqNbumRxtM6FX58QXSN0bbvpxhui6441rA4//Ah34EEHRdvYhK5jds26+LLLbba/tsXnK6/lDy+Nrocqx8aNz+bdZs2qp9w91ff5fehapuup6qOUdC2O1/XOe6r9tdF2ouuyrouys3y0TGW38upVJp/7wmC/2exbf+KvnZp/7XUz3JzZP42+fOu6reu0vido+X2LlrjNm1/028Wvv7rWa5/6jmH7GjHyK+4X8+Za8fzr+md+53p+8pNuxaOP1Jqfqxw6lroNC5gAACAASURBVLJUnjp+VrcvDz872o+W6VjpO0Cr1q19vuFxnz7zRj8/7heeKzq/lJRX/Fgk1dUKr+P+/PObvLvNGzjos1HZQoOGnG86ny/6j3FR0CFedjv2hZhbuXhFII0C+vt6es3q6FzPVUZ9tvxm6UPujP9zVrTK5hdf8J8nmq/lWUyVXv9SHLNSmmY/EPDePm/e/Zge/lUXzX8fOizvcfjzq6/WusB+slcfHyDYtWunO+aYY6MvIgoaaFk8KVL3l7/siBrwWq79Trniyuiibtucf+HX3YMP3Odqamr8rKoq56r0y+nVuY985MNDoOl/LvLL9cXg7HNGucf+91H/nl+lEbBjU5rcyRUBBCpdQAFhNaDatDk4otDnuxo/hx56WDRPE/oiqOuQrj9K+pL43B82JH4pVED55ltvi6472kYB74GfPaFWno1906lTZ9dq/1Z+849/oku92Yw697zo+qlt9bNr586c26nxfPIpp0Z1ja/47O/Xu38/9TTfcJZLUlLDUddjfQew6SFfPMmv2rt3X9+gtUD/O++87a7+3g+ixnV93xfkqzpZ6ntcP6c87Au69vfO2287HcvwGOUrh/IyS9mq8R5Pyl/5aX/5Uq5z5c3du/z21sDQeaFzKpeh7UP71feVseMm2iz/2rFT5+i9AjqWGnK+aV0FA6yBk+vY12du++YVgTQL6Lu/neu5yqnPpcOPOLLW558Ci/o8UdLyrKZKr38pjlupTD9shZai1M2Qpy4aupCqV4BdnOvbrS6uitbHu97oi8Tap9dEd1dy5RMPJNh6uqgrahN+8enb9zj/JeV365621Rr02ve449ym5zYWXLcGZc7KXkCBGYIBnAwIIFAKAV1n1Fi0YHV9+9D1xRqKWrd9hw5uz549bt8/g97x7e0aWF8jL75d+P6aK6f6rvZh92w13A488CDfG0F10J1rNawLTSqvtlcwIClZY/eAAw+MhkHYXWytr+V/enmrd9Pd/iefeDwpG9+41Z1zOejaq2uwBS/U0FcZFORXUEGBfc1rbFI+YR4y17Hq2vWDoI19cc9XDgUp1NNCd9P1PUTbq+xhSmog/PfPb/PHKOxin+tceetvb/lzRueOJe1H6+dL6q2g9dq3/3C7+Po6Dtb7Ir6svvd2LmzdusUHKnId+zCfuHm4jGkE0irQvfsx0ednrjLqM6nf8Z+KFltgUZ8n+pzR8qymSq9/KY5bqUwzHwgQtroJ6ovAoAF9oy77dhB04Rvyuc/U+pKjLzjqvqguleFFVRfjOfPu8F94ksbNWZ76chJGxW1++KUjnDduwjfcL/57bq2ujbacVwQQQACB8hRQY1YNpwvGjktFBa3Br+ubNbp19yns+h4G1dXg0zbjvjamVi86VSYpr7CS6kKv7a3hXeda/NabvrH6v4887Lvoq5u8NZCVjw1D0HVZQRTd9dcX5TDJV3fd6rtzrm1yXbctPwVbhp85rE5j25Zr3xpWYL0Nwn2rjoV+cVd9dHde+1LDO6nsuisfNhA0TMGOkYY7XP3dqXUsrJyFvCbVVfWT0anDvlQnCy1TkEjnjY6pDZGss2IDZqgMuY69ZRM3t/m8IpAFAX3uh59pYZl1biu4GgaI1UtGgTh9nuhzIexlFG6blelKr38pjlMpTMsiECBs+zKjaRu/r+lwXKKNQdN8XYz1Xl3Vrvqv70Tj+PUHqPF3NvYy3mtA2+aLrGtcZjyarsDDSScPbXQX/8MOPyL6MqX9kxBAAAEE0i+ghp4aTrrelDIlXXeS9hc+I8C6n4brRY3c9/b5HnPheHo1Ei14oG3y5aVrsK6TYYOxzrX4kLb+7v34id/01zcZaZiA3QUL7zxrmXr+xXs92B14u9Mc1iU+ne+6rXXVk8Ce0WPPE7I89KX9u1MvdyqrHcv4vgv94m42atjrhoSeB6HeAZa0LxvqYPPCV/XK0DMJwp6H4fJCppPqquErGkpggZswH9VZ35dUZh2X8EGC4XqFTKuHRrv2HfIee+WTZF5I/qyDQFoE9HfztbHj/XCbeJn0WWY9mbRMgcVwOJB9ptnnTHz7LLyv9PqX4hiVwjTTgYB4g1voipYk3TnIdUDUSNc4fPvyYesJWw8g1EUvnvSFRpE8/eGGSReuLZtf9F05w/ma1gOhHlm+zG3e/OEFP75O0vsHH7i/zhjSpPWYhwACCCCQHgFdD9SQ1t103UlVz7Qlix90Xx15Vq2GX7zEur6o0W0p3s3c5turGpHrn1mXeN2xdRrzqjv44cPq1Eh8443X683KGrpJgYZwYzU4NT42KclOw/TMTn7qGh+/Huu6rWcIWOM1PoxC12iN6dd3hVzX7aT9h/OsQXrZt79Tayyv9v3Le+/2PRFVPt3hf+Lxx/y43lzlOPDAA315rBeAvmco+KHu75biDQSbn/Sa61w55OBDfEM7DBbk6xGhc0jnqupgdVHdkh5WrO9YSrmGqiSV0+apl4e+n7U9pG3OY691c5lbPrwikBUBBe50zq9ZvSoqctibyGaqwa/PD/sbVA9n/Q3G2ya2flZeK73+pThOxTbNZCBAFwklXWw1fe9dd0bWuvDt2fO36H3SxG+WLan1Rcy+dOnuf9gDINd4Oh0EJY3vs6Ry6AnQYYTPlulVF/wLLhzn7rx9Xjg77/SPZ93gXnvtL27oqaflXY+FCCCAAALpEtBnvt1F1Z1U9TIb9qUz/F1gBaBzJXUVffHF56NrlLqJh43dcDs14PQkdt110v6amtQ1VXfelVe84axy6FkH+VKhQQDLQ93fla+SrqHqRqtGshrDAz79mag7vPmFLlo/3rVW5Vb5VQ8lNTx1B1reua7b4fcHv1HwS/tQT4B4EMDu3lkPApVPP/oPQKpPrnL0/GRvXx77cq/81QC3lNRAsGX2qjqpW73uGOY6V9q2a++HKpitzhMFRMzA8rJX+agXhNVD9dJ/NNJQST1vQeW0ZL0y7DkMNr++V323uvKKKb5XhQI3uY59LvP68mc5AmkU0LmunkT3L6z2f4Mqoxr98YcE6jNBnx/2N6hX/R3qMzH8+0tjHfOVqdLrn8+mscuKbZqpQIAukuqSpjsrSrrY6sdfVHod4yPZ+lKki7bmK4XjEhXp1sXoMwMH+TF2eq8fJd290EVSF2Wbry8Z+sPUNhoPqR9Na38aPqBk66pM6kGQ7y5Ir1593FFHH+2/TOkP+5or/8tV/8897raf3eLz0sX97C+f7vr3/Vf/o5k/+OEN0d0OvxK/EEAAAQQyL6Bu9rp+2Bh8Tev6omuXGvZ2Z0gVDbvY20PjtL6udxreZsvV4NNzb7TtT2660eevxrmlcFy/de/WcruO2TXP9qlGte5Mabnuatm/urX8wlftW0Mhwn2EDx8M17VpK7fy1zVU9VajNBwWYOvKRWPrrRGtBqkFLWwdveqOtV3H9Rwgu4OddN1W8P7UL50ebl5rWgGF3wZ36VROeVl3Xeu+axspiKFgha7vucoRzled9d8kzCFXvnaufLD/G/x/KFJ98p0rGuuvY6ZtdJ6MGn1eo75L6M6/nJSPfmRr/3mhIeebjoWGQlgQzOqsPMNjn8vcjHlFIGsCOuePPbanD+Cp7PoMC58BYoFF6ylk9dPni/7TjAXfbH7WXiu9/qU4XsU0rapppselz58/340ZM6ZBHtXV1e7UM0YkbtOm9Ufdnr3/L3FZ2maK2P5lYNrKRnk+EOAYcSYggEBTBJY+uCDn9aop+bItAgg0TiBL3xMbV0O2akmBxnzmK0ioB6kqqKpgXpZTpde/FMeuJUz3K0VFyLNxAmqMhnGZj3wkUx023Pvvvx9VvKllN4d4ACXX/GjHTCCAAAIIIIAAAgggkDIB9eLRfzqr1FTp9S/FcW+qaaYDAYr2khBAAAEEEGhpgZEtXQD2jwACCCCAAAIINEAgs4GArAwL0LHQXez4ne2kYxRfL/4+aZvmmpevLPmWNbZ8ylOpELfG7iPcrhR1CPNnGgEEylugMV36yluE2iGAAAIIIIBAmgUyGwhIM2qpyqbGqjWQw6734XxrOOs13rhNep+Un3XxVx6Wj+3D5jWkjrattolvn2+Z7cPWUZ01bfno1ZbZPOWvZHXQdGjlF/ILAQQQQAABBBBAAAEEEKhggdQHAtq0ytY4+aRzSY1Va6AmLbd5dRu1H9b9gzzi7z9o7Dv3QYNd+Wg9Je0vvt/wfb78qqrqnhbhtlbeD/f34f7DZUnLlY+SlS9f2cP1zC++fVVVLpOPFmTuC9OAXhu2Pq8IIIBAXKAcrlfxOvEeAQQQQCBZoNI/8yu9/slnRdPmNrdp3RZf08rP1k0UsLvXuqNtjV9lqQawNYJtF7bcXjVf0/H1bP3wNV9+2reVI9wmabqQ/eUrX75lVo9wnXgZwjv/WqZ19RP3i2/HewQQQAABBBBAAAEEEECgUgUIBKT0yFs3eGsEWwO3WMXNlZ/Nt4a07T/fftVgL2S9fHnkWlZf3rkCFuan7XOtk2ufzEcAAQQQQAABBBBAAAEEylngw37V5VzLMqmb3SGPVyecH05rvfB9OB1fFs+zIY1nBQHy5R0uC6fjZYgvU74qR/yuf1jW+DbhMm2fVLZwHaYRQAABBBBAAAEEEEAAgUoToEdAio+4GrFqBKsxbNPW8LVGbtJ8q1JDl2l95R/uw/LSa65eArZd2GC3QEJ8md7rRynfsnC/FgywPG1ZfHvN1zrxOtj+bDteEUAAAQQQQAABBBBAAIFKFiAQkKKjn9RgDRu/4XRY7Ph8a8hrnfiyfNtpmcqQVI5c8y2/fMsbsyxeBqtHfH5S3knzrJy8IoAAAggggAACCCCAAAKVLsDQgEo/A6g/AggggAACCCCAAAIINElg79697tJLL3UzZ86slc/u3bv9PC0P0+rVq2utq/VGjBjhND+LqdLrX4pjVmrTTAYC9AfW2D+SzZs3u6uuusrF/xhLcfDIEwEEEECgsgW4XlX28af2CCBQWQJt2rRxq1atqredonbI4sWL3fDhwyOgF154wfXq1cvPz2o7pdLrHx3MIk6U0jSTgYAi2pZlVvHu82VZSSqFAAIIIIAAAggggEDKBMaNG1dvY37btm3uyCOPdN27d49Kv27dOjdy5Ej/Xsuzmiq9/qU4bqUyzVwgQHdXZs+e7UaPHu273yhiprv8gwcPdt26dfNdatS1Rqm6utrP03xtp/XGjh3r7rrrLte7d+96o3WlOJDkiQACCCBQGQJcryrjOFNLBBBAIBTo0aOHO+igg9yvf/3rcHat6Weeecb1798/mqe2y9tvv+0DA3369HFantVU6fUvxXErlWnmAgGTJ092EydOdHfffbebNWuWU8TsjjvucMuWLXNbtmxxWj5nzhynP6iVK1e6tWvXRvMVdZs3b54799xz3caNG93AgQNLcazIEwEEEEAAAX894nrFiYAAAghUnsD48ePd0qVL/U3IeO2tjaLGnaXly5e7Ll26uNatW7vjjz/ebdiwIdPDmCu9/nZci/laCtPMBQLioDt37ozu8OvOv3oK7NixI/oXeNOmTUvFH1L4JP94HXjfsgIcm5b1Z+8IVIpAVq5XlXI8qCcCCCBQKoF27dq5CRMmuIULF9bZhZ4FcMIJJzito6TezWr4KwCg1LlzZ/+a5eEBlV5/fwCL/KsUpmXx7wN1x0U9AeJJPQY0HGDo0KHu9NNPT1wnvk0p3mvMfvi/7UuxD/JsmgDPVWiaH1sjgEBhAmm/XhVWC9ZCAAEEEKhPoG/fvu7ee+91Tz31VLSqPSTw/PPPj+apwb9ixQp/YzOa6ZzTEIHwGQLhsixMV3r9S3GMim2a+R4BHTp08E/nVIM/KekP6IEHHvC9BOzZAUnrlXqe/W97XqtcGg1KffzJHwEEEMjK9YojhQACCCDQdAF187/ooov8M8s0/l9Jjf74QwL1PIBJkyb5ocwa5qwfDXnW0IKWbLs0VaDS699Uv6Tti22ayUDAiSeeGD0sUN1npkyZ4u/6a2iAfvSQQP3h6H9x6v2AAQPcqFGjfBccrb9nzx4eFph0djEPAQQQQKCoAlyvispJZggggECmBHRDsmfPnr7toYLHHxIYHxZglVN75eCDD3YaRpDlVOn1L8WxK6ZpVU0zDZCeP3++GzNmTIM81KC3f6PRoA1ZGQEEEEAAgWYU4HrVjNjsCgEEEGhhgcZ85usmpR5orrv/urOb5VTp9S/FsWsJ07J4RkApDgZ5IoAAAggggAACCCCAAALFENDD3pKeaVaMvLOQR6XXvxTHqKmmmRwaUApI8kQAAQQQQAABBBBAAAEEEECgEgQIBFTCUaaOCCCAAAIIIIAAAggggAACCPxTgEAApwICCCCAAAIIIIAAAggggAACFSSQ+mcE7Nn3fgUdDqqKAAIIIJBVAa5XWT1ylBsBBBBouEClf+ZXev0bfsbUv0Vzm9IjoP5jwhoIIIAAAggggAACCCCAAAIIlI0AgYCyOZRUBAEEEEAAAQQQQAABBBBAAIH6BQgE1G/EGggggAACCCCAAAIIIIAAAgiUjQCBgLI5lFQEAQQQQAABBBBAAAEEEEAAgfoFCATUb8QaCCCAAAIIIIAAAggggAACCJSNQOYCAW++udud99WRrl+vY6KfYacMcS9t2Vw2B8Uqcv/CX0V1VH2nTr7M7du71xY3+FVGsrrpxuvrbKt52sfap1fXWcYMBBBAAAEEEEAAAQQQQACB8hHIXCDA6Id96Qy3au2zbuGiJX7WnNk/bVIj2fJNy6sa5tdcOdXNvX2+W/+HF/3Pv510SkmL17FTZ9e+fYeS7oPMEUAAAQQQQAABBBBAAAEEWlYgs4EAY+vUqbPrd/yn3J9ffdXte2+fzc70q+7K//fPb3PXXDvdDfj0wKgu/z50mGvVunX0vpgTF192uVvy8ArXtVv3YmZLXggggAACCCCAAAIIIIAAAikTyHwgYPv2bW79M79zAz79Gde2bTvPq4Z0OHTAuruHXe1tOIHuvGt6zKizo23CrvPxvMJlSdtqH0rxIQy2nXXPt/LZ+uF58eQTjzvdne97XL9wdjQdz1tDJTRPSUMHNITA8rd6Rhv/cyJcT2WwutgQi1zljHuYbTx/3iOAAAIIIIAAAggggAACCKRTILOBgCWLH3SDBvR1F/3HOHfLz+Y63dFWUsN03NfG+C71GjqgIQQ33XiDW/G/j/iu9hd+fYLvZh/e/d6xfZs7+5xRfr6W62688lFj+MorpjjbRt30tcwa9dqftr3k/17uVjy5xvXp288tqP4f95e/7HAzrrvWvfHG637ogrr2q3zKT+Wd+I1vRfuafetPGvR8AzXglbeS9qkfpUnfmOCDAcpPgRENmVD91VtC+7QGvl/ZObd0yWInQ/U6+PLws222f81VzsdXPOo9bFiG6hX2WKiVCW8QQAABBBBAAAEEEEAAAQRSKZDZQIAao0uXP+YOPfSwWg1d3U1XUjBAgQI1dtUg/0hVlb/LroZ8eAdd64Z33z/3hcF+ew01ePb3631D3+Z1P6aHb+yroW8P7bNx9a32b+U+dtRRfl/vvvOOO/LIjn7b4WcOc3bX3/LT2H/dsVdZFEjYtWun32chv+I9INQLQr0hVMc/bd3q1j69xjf+NWRCwwgGDjqhzj6eXrPaKWAgw1OHfanObnOV8/2a9723TM86c1id4EKdjJiBAAIIIIAAAggggAACCCCQOoHMBgIkqUbwxZd92zd0H3zgvghXjXPdEbeH7Onu/+AhX3T3LVriG78bnl3vhnzuMyV9Qr56AKgHgZIa/uqu/49//N2/Dx8AmHRX/eOf6OLrpAZ5KVN9z1WIl3PIv53s7ryn2veQUAAjDHKUspzkjQACCCCAAAIIIIAAAgggUDyBTAcCxNC7d1/fuFdXd3Vpt0a0BQZ05/43yz74zwK6Qz595o2+O7y2VUNYSY1aNbq1rrr2Ww8BjdHXtPUy2PziC05BBN1lL+Shfeo2b0MGtK/DDj/C70/7sB4FKptN+4XO+bv0uluvAEI4Bl/rtm/f3t/x151/PRdAP9YL4Nh/7el7B2hogHoOKN/Vq1b6XgzqzWDp058Z6K69boavyy/mzbXZ0at6NijlKmcY5PjTy1uj7ZhAAAEEEEAAAQQQQAABBBBIv8B+6S9i/hJa93d1V9e/ELz6ez/wG6gRra73Shrjrwa0hgtY0jyNjbfx/lpfP0q6E25Pz1eDWduFecXH1Fue9vree/t8DwCVSUnBBD3HQHkqb+Vny9TgHzx4iG3qXy1goeEFYZlt3SlXXOmfCaBeDUp6NoHmaTs9f0BDF3S3Xsn2rd4Tu3Z+OARBQQp7HoJfMfilZUnlPOqoo91lF1/kAydaXeXR/kgIIIAAAggggAACCCCAAALZEaiqqampaY7izp8/340Z82FDvJB9VldXu1PPGFHIqo1eR4EA9SawhnqjM2JDBBBAAIGKFVj64IKSX68qFpeKI4AAAikTqPTP/EqvfylOx5YwzfzQgFIcCPJEAAEEEEAAAQQQQAABBBBAoFwFCASU65GlXggggAACCCCAAAIIIIAAAggkCFR8IEAPvtN/FbBnAiQYMQsBBBBAAAEEEEAAAQQQQACBshGo+EBA2RxJKoIAAggggAACCCCAAAIIIIBAAQKp/68BbVoRqyjgOLIKAggggEALC3C9auEDwO4RQACBZhSo9M/8Sq9/KU615jallV2Ko0ieCCCAAAIIIIAAAggggAACCKRUgEBASg8MxUIAAQQQQAABBBBAAAEEEECgFAIEAkqhSp4IIIAAAggggAACCCCAAAIIpFSAQEBKDwzFQgABBBBAAAEEEEAAAQQQQKAUAgQCSqFKnggggAACCCCAAAIIIIAAAgikVIBAQEoPDMVCAAEEEEAAAQQQQAABBBCoTIG9e/e6Sy+91HXr1s3/zJw5sxaE3tuy1atX11pmb3Kts3nzZpfJQIAqlKuyVulcr6r0VVdd5QRLQgABBBBAoJQCXK9KqUveCCCAAAIIlLeA2q1btmxxGzdudDt27IjawNXV1b7iWrZs2TJ32223ud27d9fCyLWO1ps+fXo2AwG1asgbBBBAAAEEEEAAAQQQQAABBMpIoHXr1q5du3a+Rpru2LGjn9YN7Q0bNrjhw4f79927d3fHHnuse+GFF6La51tH62n9zPUI0N2V2bNnu9GjR/uuEqqk7vIPHjzYd40YMWJEFA1RFMS6S2g7rTd27Fh31113ud69e0cRlUiMCQQQQAABBIokwPWqSJBkgwACCCCAQIUL6C6+egT06NHD7du3z+3Zs8d16NAhUunSpYt75ZVXovf51tF6Wn+/aO2MTEyePNmX9MQTT3QDBw70jfs77rjDd4lQpERDBubMmePGjx/vVq5c6dauXRtFUrThvHnznNafOnWq0/okBBBAAAEESiHA9aoUquSJAAIIIIBA5Qiobasb4P369XNz58717dr4EIDGamSuR0C8ojt37ozu8Ovuv6AULampqfGrTps2jecBxNF4jwACCCDQ7AJcr5qdnB0igAACCCCQaQHd+NZzABQEGDdunLNx/8WoVOZ6BCRVeuLEic7uvITLZ82a5XsMDB061J1++umJ64TrM40AAggggEApBbhelVKXvBFAAAEEEChPAT0rQO3dxx57zLVq1cq1adPG6QaDPUNg69atTj3mLdW3jvLJfI8AjY1YtWqVb/BbxcNXPTzhgQce8L0EitWNIsyfaQQQQAABBAoR4HpViBLrIIAAAggggIAE1HYN/9OdGu8a26/h7X369HELFy70UHoO3ttvv+369u3r28T6TwNKudbRcwaef/75bAYCFO2whwV27tzZTZkyxemuvz0YUF0mBKcHB2regAED3KhRo3zEROvr4Qo8LNCfH/xCAAEEECihANerEuKSNQIIIIAAAmUsoKf7q81qbVwFAUaOHOlrrN7uGg6vZXoY/vnnn1/n+Xe51lEvggkTJriqGhtMX2LE+fPnuzFjxjRoL2rQW2UbtCErI4AAAggg0IwCXK+aEZtdIYAAAi0sUOmf+ZVe/1Kcfi1hmvmhAaU4EOSJAAIIIIAAAggggAACCCCAQLkKEAgo1yNLvRBAAAEEEEAAAQQQQAABBBBIECAQkIDCLAQQQAABBBBAAAEEEEAAAQTKVYBAQLkeWeqFAAIIIIAAAggggAACCCCAQIIAgYAEFGYhgAACCCCAAAIIIIAAAgggUK4CBALK9chSLwQQQAABBBBAAAEEEEAAAQQSBAgEJKAwCwEEEEAAAQQQQAABBBBAAIFyFSAQUK5HlnohgAACCCCAAAIIIIAAAgggkCBAICABhVkIIIAAAggggAACCCCAAAIIlKsAgYByPbLUCwEEEEAAAQQQQAABBBBAAIEEAQIBCSjMQgABBBBAAAEEEEAAAQQQQKBcBQgElOuRpV4IIIAAAggggAACCCCAAAIIJAgQCEhAYRYCCCCAAAIIIIAAAggggAAC5SpAIKBcjyz1QgABBBBAAAEEEEAAAQQQQCBBgEBAAgqzEEAAAQQQQAABBBBAAAEEEChXAQIB5XpkqRcCCCCAAAIIIIAAAggggAACCQIEAhJQmIUAAggggAACCCCAAAIIIIBAuQoQCCjXI0u9EEAAAQQQQAABBBBAAAEEEEgQIBCQgMIsBBBAAAEEEEAAAQQQQAABBMpVgEBAuR5Z6oUAAggggAACCCCAAAIIIIBAggCBgAQUZiGAAAIIIIAAAggggAACCCBQrgIEAsr1yFIvBBBAAAEEEEAAAQQQQAABBBIECAQkoDALAQQQQAABBBBAAAEEEEAAgXIVIBBQrkeWeiGAAAIIIIAAAggggAACCCCQIEAgIAGFWQgggAACCCCAAAIIIIAAAgiUqwCBgHI9stQLAQQQQAABBBBAAAEEEEAAgQQBAgEJKMxCAAEEEEAAAQQQQAABBBBAoFwFCASU65GlXggggAACCCCAQ/ouwQAAARVJREFUAAIIIIAAAggkCBAISEBhFgIIIIAAAggggAACCCCAAALlKkAgoFyPLPVCAAEEEEAAAQQQQAABBBBAIEGAQEACCrMQQAABBBBAAAEEEEAAAQQQKFcBAgHlemSpFwIIIIAAAggggAACCCCAAAIJAgQCElCYhQACCCCAAAIIIIAAAggggEC5ChAIKNcjS70QQAABBBBAAAEEEEAAAQQQSBAgEJCAwiwEEEAAAQQQQAABBBBAAAEEylWAQEC5HlnqhQACCCCAAAIIIIAAAggggECCwH7r1q1LmM0sBBBAAAEEEEAAAQQQQAABBBAoR4H9unbt2iz12rRpU7Psh50ggAACCCCAAAIIIIAAAggggEBugf8Piu0JlqhFlqsAAAAASUVORK5CYII=)



##### 二、Session清理

在tomcat运行中会起一个守护线程ContainerBackgroundProcessor来进行session的清理工作





守护线程的通过循环执行，每清理一次的时间间隔默认值是10秒，当然这个线程不仅仅会清理session，不同的Container容器类型所做的清理工作是不一样的。



```java
protected class ContainerBackgroundProcessor implements Runnable {



 



        @Override



        public void run() {



            Throwable t = null;



            String unexpectedDeathMessage = sm.getString(



                    "containerBase.backgroundProcess.unexpectedThreadDeath",



                    Thread.currentThread().getName());



            try {



                while (!threadDone) {



                    try {



						//backgroundProcessorDelay默认值是10，每10秒清理一下session



                        Thread.sleep(backgroundProcessorDelay * 1000L);



                    } catch (InterruptedException e) {



                        // Ignore



                    }



                    if (!threadDone) {



                        processChildren(ContainerBase.this);



                    }



                }



            } catch (RuntimeException|Error e) {



                t = e;



                throw e;



            } finally {



                if (!threadDone) {



                    log.error(unexpectedDeathMessage, t);



                }



            }



        }



 



        protected void processChildren(Container container) {



            ClassLoader originalClassLoader = null;



 



            try {



                if (container instanceof Context) {



                    Loader loader = ((Context) container).getLoader();



                    // Loader will be null for FailedContext instances



                    if (loader == null) {



                        return;



                    }



                    originalClassLoader = ((Context) container).bind(false, null);



                }



				//执行后台处理操作



                container.backgroundProcess();



                Container[] children = container.findChildren();



                for (int i = 0; i < children.length; i++) {



                    if (children[i].getBackgroundProcessorDelay() <= 0) {



                        processChildren(children[i]);



                    }



                }



            } catch (Throwable t) {



                ExceptionUtils.handleThrowable(t);



                log.error("Exception invoking periodic operation: ", t);



            } finally {



                if (container instanceof Context) {



                    ((Context) container).unbind(false, originalClassLoader);



               }



            }



        }



    }
```

会执行子容器的backgroudProcess()方法，最终会调用ManagerBase的backgroudProcess()方法。





```java
 @Override



    public void backgroundProcess() {



        count = (count + 1) % processExpiresFrequency;//执行清理的时间间隔是6*10秒即1分钟



        if (count == 0)



            processExpires();



    }
```

在processExpires()中会获取Context下的所有session，并依次调用session.isValid方法来清理自身





```java
public void processExpires() {



 



        long timeNow = System.currentTimeMillis();



        Session sessions[] = findSessions();



        int expireHere = 0 ;



 



        if(log.isDebugEnabled())



            log.debug("Start expire sessions " + getName() + " at " + timeNow + " sessioncount " + sessions.length);



		//调用session的isValid方法来清理自身



        for (int i = 0; i < sessions.length; i++) {



            if (sessions[i]!=null && !sessions[i].isValid()) {



                expireHere++;



            }



        }



        long timeEnd = System.currentTimeMillis();



        if(log.isDebugEnabled())



             log.debug("End expire sessions " + getName() + " processingTime " + (timeEnd - timeNow) + " expired sessions: " + expireHere);



        processingTime += ( timeEnd - timeNow );



 



    }
```

session.isValid方法会判断自身是否有效





```java
	@Override



    public boolean isValid() {



 



        if (!this.isValid) {



            return false;



        }



 



        if (this.expiring) {



            return true;



        }



 



        if (ACTIVITY_CHECK && accessCount.get() > 0) {



            return true;



        }



 



        if (maxInactiveInterval > 0) {



            int timeIdle = (int) (getIdleTimeInternal() / 1000L);



            if (timeIdle >= maxInactiveInterval) {



                expire(true);



            }



        }



 



        return this.isValid;



    }
```

session.expire(boolean notify)会将自身设置为无效并从管理器Manager中删除





```java
public void expire(boolean notify) {



 



        // Check to see if session has already been invalidated.



        // Do not check expiring at this point as expire should not return until



        // isValid is false



        if (!isValid)



            return;



 



        synchronized (this) {



            // Check again, now we are inside the sync so this code only runs once



            // Double check locking - isValid needs to be volatile



            // The check of expiring is to ensure that an infinite loop is not



            // entered as per bug 56339



            if (expiring || !isValid)



                return;



 



            if (manager == null)



                return;



 



            // Mark this session as "being expired"



            expiring = true;



 



            // Notify interested application event listeners



            // FIXME - Assumes we call listeners in reverse order



            Context context = manager.getContext();



 



            // The call to expire() may not have been triggered by the webapp.



            // Make sure the webapp's class loader is set when calling the



            // listeners



            if (notify) {



                ClassLoader oldContextClassLoader = null;



                try {



                    oldContextClassLoader = context.bind(Globals.IS_SECURITY_ENABLED, null);



                    Object listeners[] = context.getApplicationLifecycleListeners();



                    if (listeners != null && listeners.length > 0) {



                        HttpSessionEvent event =



                            new HttpSessionEvent(getSession());



                        for (int i = 0; i < listeners.length; i++) {



                            int j = (listeners.length - 1) - i;



                            if (!(listeners[j] instanceof HttpSessionListener))



                                continue;



                            HttpSessionListener listener =



                                (HttpSessionListener) listeners[j];



                            try {



                                context.fireContainerEvent("beforeSessionDestroyed",



                                        listener);



                                listener.sessionDestroyed(event);



                                context.fireContainerEvent("afterSessionDestroyed",



                                        listener);



                            } catch (Throwable t) {



                                ExceptionUtils.handleThrowable(t);



                                try {



                                    context.fireContainerEvent(



                                            "afterSessionDestroyed", listener);



                                } catch (Exception e) {



                                    // Ignore



                                }



                                manager.getContext().getLogger().error



                                    (sm.getString("standardSession.sessionEvent"), t);



                            }



                        }



                    }



                } finally {



                    context.unbind(Globals.IS_SECURITY_ENABLED, oldContextClassLoader);



                }



            }



 



            if (ACTIVITY_CHECK) {



                accessCount.set(0);



            }



 



            // Remove this session from our manager's active sessions



            manager.remove(this, true);



 



            // Notify interested session event listeners



            if (notify) {



                fireSessionEvent(Session.SESSION_DESTROYED_EVENT, null);



            }



 



            // Call the logout method



            if (principal instanceof TomcatPrincipal) {



                TomcatPrincipal gp = (TomcatPrincipal) principal;



                try {



                    gp.logout();



                } catch (Exception e) {



                    manager.getContext().getLogger().error(



                            sm.getString("standardSession.logoutfail"),



                            e);



                }



            }



 



            // We have completed expire of this session



            setValid(false);



            expiring = false;



 



            // Unbind any objects associated with this session



            String keys[] = keys();



            ClassLoader oldContextClassLoader = null;



            try {



                oldContextClassLoader = context.bind(Globals.IS_SECURITY_ENABLED, null);



                for (int i = 0; i < keys.length; i++) {



                    removeAttributeInternal(keys[i], notify);



                }



            } finally {



                context.unbind(Globals.IS_SECURITY_ENABLED, oldContextClassLoader);



            }



        }



 



    }
```