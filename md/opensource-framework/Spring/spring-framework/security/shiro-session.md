```
javax.servlet.Filter ***

org.springframework.security.web.access.intercept.FilterSecurityInterceptor  filter interceptor 适配接口

- org.springframework.web.filter.GenericFilterBean ***

-- org.springframework.web.filter.OncePerRequestFilter

-- org.springframework.security.web
.authentication.AbstractAuthenticationProcessingFilter  权限认证相关

org.springframework.web.filter.DelegatingFilterProxy

FilterChainProxy
FilterChainProxy$VirtualFilterChain


StandardContext#startInternal
filterStart
ApplicationFilterConfig  constructor
filterDef
initFilter();

filter.init(ApplicationFilterConfig);

GenericFilterBean#init



```

#### spring boot 启动调用栈

```
org.apache.catalina.core.ApplicationFilterConfig.initFilter(ApplicationFilterConfig.java:270)
	  at org.apache.catalina.core.ApplicationFilterConfig.<init>(ApplicationFilterConfig.java:106)
	  at org.apache.catalina.core.StandardContext.filterStart(StandardContext.java:4528)
	  - locked <0x1e2d> (a java.util.HashMap)





  java.lang.Thread.State: RUNNABLE
	  at org.apache.catalina.core.ApplicationContext.addFilter(ApplicationContext.java:791)
	  at org.apache.catalina.core.ApplicationContext.addFilter(ApplicationContext.java:772)
	  at org.apache.catalina.core.ApplicationContextFacade.addFilter(ApplicationContextFacade.java:454)
	  at org.springframework.boot.web.servlet.AbstractFilterRegistrationBean.addRegistration(AbstractFilterRegistrationBean.java:210)
	  at org.springframework.boot.web.servlet.AbstractFilterRegistrationBean.addRegistration(AbstractFilterRegistrationBean.java:46)
	  at org.springframework.boot.web.servlet.DynamicRegistrationBean.register(DynamicRegistrationBean.java:108)
	  at org.springframework.boot.web.servlet.RegistrationBean.onStartup(RegistrationBean.java:53)
	  at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.selfInitialize(ServletWebServerApplicationContext.java:230)
	  at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext$$Lambda$528.88094983.onStartup(Unknown Source:-1)
	  at org.springframework.boot.web.embedded.tomcat.TomcatStarter.onStartup(TomcatStarter.java:53)


-------------- 	StandardContext.startInternal ------------------
	  at org.apache.catalina.core.StandardContext.startInternal(StandardContext.java:5165)
	  - locked <0x19bd> (a org.springframework.boot.web.embedded.tomcat.TomcatEmbeddedContext)
	  at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183)
	  at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1384)
	  at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1374)
	  at java.util.concurrent.FutureTask.run$$$capture(FutureTask.java:266)
	  at java.util.concurrent.FutureTask.run(FutureTask.java:-1)
	  at org.apache.tomcat.util.threads.InlineExecutorService.execute(InlineExecutorService.java:75)
	  at java.util.concurrent.AbstractExecutorService.submit(AbstractExecutorService.java:134)
	  at org.apache.catalina.core.ContainerBase.startInternal(ContainerBase.java:909)
	  - locked <0x1e74> (a org.apache.catalina.core.StandardHost)
	  at org.apache.catalina.core.StandardHost.startInternal(StandardHost.java:843)
	  at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183)
	  at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1384)
	  at org.apache.catalina.core.ContainerBase$StartChild.call(ContainerBase.java:1374)
	  at java.util.concurrent.FutureTask.run$$$capture(FutureTask.java:266)
	  at java.util.concurrent.FutureTask.run(FutureTask.java:-1)
	  at org.apache.tomcat.util.threads.InlineExecutorService.execute(InlineExecutorService.java:75)
	  at java.util.concurrent.AbstractExecutorService.submit(AbstractExecutorService.java:134)
	  at org.apache.catalina.core.ContainerBase.startInternal(ContainerBase.java:909)
	  - locked <0x1fb2> (a org.apache.catalina.core.StandardEngine)
	  at org.apache.catalina.core.StandardEngine.startInternal(StandardEngine.java:262)
	  at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183)
	  at org.apache.catalina.core.StandardService.startInternal(StandardService.java:421)
	  at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183)
	  - locked <0x1fb3> (a org.apache.catalina.core.StandardService)
	  at org.apache.catalina.core.StandardServer.startInternal(StandardServer.java:930)
	  - locked <0x1fb4> (a java.lang.Object)
	  at org.apache.catalina.util.LifecycleBase.start(LifecycleBase.java:183)
	  - locked <0x1fb5> (a org.apache.catalina.core.StandardServer)
	  at org.apache.catalina.startup.Tomcat.start(Tomcat.java:486)
	  at org.springframework.boot.web.embedded.tomcat.TomcatWebServer.initialize(TomcatWebServer.java:123)
	  - locked <0x1fb6> (a java.lang.Object)
	  at org.springframework.boot.web.embedded.tomcat.TomcatWebServer.<init>(TomcatWebServer.java:104)
	  at org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory.getTomcatWebServer(TomcatServletWebServerFactory.java:437)
	  at org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory.getWebServer(TomcatServletWebServerFactory.java:191)
	  at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.createWebServer(ServletWebServerApplicationContext.java:178)
	  at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.onRefresh(ServletWebServerApplicationContext.java:158)
	  at org.springframework.context.support.AbstractApplicationContext.refresh(AbstractApplicationContext.java:545)
	  - locked <0x1fb7> (a java.lang.Object)
	  at org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext.refresh(ServletWebServerApplicationContext.java:143)
	  at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:758)
	  at org.springframework.boot.SpringApplication.refresh(SpringApplication.java:750)
	  at org.springframework.boot.SpringApplication.refreshContext(SpringApplication.java:397)
	  at org.springframework.boot.SpringApplication.run(SpringApplication.java:315)
	  at org.springframework.boot.SpringApplication.run(SpringApplication.java:1237)
	  at org.springframework.boot.SpringApplication.run(SpringApplication.java:1226)
	  at com.zqh.security.SpringBootShiroMybatisApplication.main(SpringBootShiroMybatisApplication.java:10)
```



# Shiro企业级实战，统一的Session管理。

基础的什么配置这些都不说了，百度一下什么都有，直接上干货。

Shiro切入点是从web.xml文件，通过filter进行拦截。

![img](https://images2018.cnblogs.com/blog/1215497/201805/1215497-20180531153845630-1515499161.png)

 

直接看DelegatingFilterProxy这个类，很简单，父类就是一个filter，肯定会初始化filter，后面会调用这个方法：

```
@Override
    protected void initFilterBean() throws ServletException {
        synchronized (this.delegateMonitor) {
            if (this.delegate == null) {
                // If no target bean name specified, use filter name.
                if (this.targetBeanName == null) {
                    this.targetBeanName = getFilterName();
                }
                // Fetch Spring root application context and initialize the delegate early,
                // if possible. If the root application context will be started after this
                // filter proxy, we'll have to resort to lazy initialization.
                WebApplicationContext wac = findWebApplicationContext();
                if (wac != null) {
                    this.delegate = initDelegate(wac);
                }
            }
        }
    }
```

　　这个方法主要的作用就是获取filterName，通过filterName和类型从spring Bean容器中获取Bean然后赋予delegate。filtetName很显然是shiroFilter，然后这个类型就是Filter.class。

​     至于想要获取的这个Bean，也很简单。直接找spring中shiro的这块配置。

```
<bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">
        <property name="securityManager" ref="securityManager"/>
        <property name="loginUrl" value="${shiro.loginUrl}"/>
        <!--登录成功默认跳转页面，不配置则跳转至”/”。 如果登陆前点击的一个需要登录的页面，则在登录自动跳转到那个需要登录的页面。不跳转到此。 -->
        <property name="successUrl" value="${shiro.successUrl}"/>
        <!--没有权限跳转的链接 -->
        <property name="unauthorizedUrl" value="/login"/>
        <property name="filterChainDefinitions">
            <value>
                /favicon.ico =anon
                /css/**     = anon
                /js/**     = anon
                /druid/** = anon
                /ecoupon/info/detail = anon
                /** = authc
            </value>
        </property>
    </bean>
    
  <!-- 权限管理器 -->
<bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">
    <!-- 基于redis登录校验的实现 -->
    <property name="realm" ref="systemAuthorizingRealm"/>
    <!-- session 管理器 -->
    <property name="sessionManager" ref="sessionManager"/>
</bean>
```

#### ShiroFilterFactoryBean#getObject

```
ShiroFilterFactoryBean这个类实现FactoryBean,故在getBean时会执行

public Object getObject() throws Exception {
        if (instance == null) {
            instance = createInstance();
        }
        return instance;
    }
protected AbstractShiroFilter createInstance() throws Exception {
 
        log.debug("Creating Shiro Filter instance.");
 
        SecurityManager securityManager = getSecurityManager();
        if (securityManager == null) {
            String msg = "SecurityManager property must be set.";
            throw new BeanInitializationException(msg);
        }
 
        if (!(securityManager instanceof WebSecurityManager)) {
            String msg = "The security manager does not implement the WebSecurityManager interface.";
            throw new BeanInitializationException(msg);
        }
 
        FilterChainManager manager = createFilterChainManager();
 
        //Expose the constructed FilterChainManager by first wrapping it in a
        // FilterChainResolver implementation. The AbstractShiroFilter implementations
        // do not know about FilterChainManagers - only resolvers:
        PathMatchingFilterChainResolver chainResolver = new PathMatchingFilterChainResolver();
        chainResolver.setFilterChainManager(manager);
 
        //Now create a concrete ShiroFilter instance and apply the acquired SecurityManager and built
        //FilterChainResolver.  It doesn't matter that the instance is an anonymous inner class
        //here - we're just using it because it is a concrete AbstractShiroFilter instance that accepts
        //injection of the SecurityManager and FilterChainResolver:
        return new SpringShiroFilter((WebSecurityManager) securityManager, chainResolver);
    }
```

　　所以最后肯定就会获取到 SpringShiroFilter 对应的bean。其实就是最后new SpringShiroFilter((WebSecurityManager) securityManager, chainResolver)产生的对象，

这样整个入口算是清楚了。

　　比如现在正好有一个请求过来了，第一步，肯定是DelegatingFilterProxy中进行doFilter拦截住。

```
public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
        throws ServletException, IOException {
 
    // Lazily initialize the delegate if necessary.
    Filter delegateToUse = this.delegate;
    if (delegateToUse == null) {
        synchronized (this.delegateMonitor) {
            if (this.delegate == null) {
                WebApplicationContext wac = findWebApplicationContext();
                if (wac == null) {
                    throw new IllegalStateException("No WebApplicationContext found: no ContextLoaderListener registered?");
                }
                this.delegate = initDelegate(wac);
            }
            delegateToUse = this.delegate;
        }
    }
 
    // Let the delegate perform the actual doFilter operation.
    invokeDelegate(delegateToUse, request, response, filterChain);
}
```

　　

```
`delegateToUse这个我们也获取到了，后面最重要的是invokeDelegate(delegateToUse, request, response, filterChain)这个方法。后面会调用OncePerRequestFilter中的doFilter方法，很简单，这个是SpringShiroFilter的父类，然后就会执行doFilterInternal的实现，这个实现在AbstractShiroFilter<br>中。重点来了，`
```

#### AbstractShiroFilter#doFilterInternal

```
protected void doFilterInternal(ServletRequest servletRequest, ServletResponse servletResponse, final FilterChain chain)
           throws ServletException, IOException {
 
       Throwable t = null;
 
       try {<br>　　　　　　　这里就是对serverRequest和serverResponse包装
           final ServletRequest request = prepareServletRequest(servletRequest, servletResponse, chain);
           final ServletResponse response = prepareServletResponse(request, servletResponse, chain);
　　　　　　这个才是我们的重头戏
           final Subject subject = createSubject(request, response);
 
           //noinspection unchecked<br>　　　　　　　这个会执行SubjectCallAble中的call方法将生成的subject对象绑定到ThreadContext中，这个原理很简单，就是ThreadLocal对当前线程进行绑定。
           subject.execute(new Callable() {
               public Object call() throws Exception {<br>　　　　　　　　　　　　绑定到当前线程之后，更新session最后获取时间。
                   updateSessionLastAccessTime(request, response);
                   executeChain(request, response, chain);
                   return null;
               }
           });
       } catch (ExecutionException ex) {
           t = ex.getCause();
       } catch (Throwable throwable) {
           t = throwable;
       }
 
       if (t != null) {
           if (t instanceof ServletException) {
               throw (ServletException) t;
           }
           if (t instanceof IOException) {
               throw (IOException) t;
           }
           //otherwise it's not one of the two exceptions expected by the filter method signature - wrap it in one:
           String msg = "Filtered request failed.";
           throw new ServletException(msg, t);
       }
   }
```

后面会进入这个创建Subject的方法,看看具体干了什么

#### createSubject

```
public Subject createSubject(SubjectContext subjectContext) {
        //create a copy so we don't modify the argument's backing map:<br>　　　　　很简单，这个就是相当于包装，把subjectContext包装进去。
        SubjectContext context = copy(subjectContext);
 
        把secutityManager放进去。
        context = ensureSecurityManager(context);
 
        获取服务器session
        context = resolveSession(context);
　　　　　<br>　　　　　　<br>　　　　　这个其实没啥用，使用认证信息和权限信息缓存时才会有用，这个在多节点的服务来说，基本不会在服务器做这个，相当于把一个Map的key放进context中，第一次进来，肯定是空。<br>        等到认证完成后，才会有值，是从session中获取的。<br>　　　　
        context = resolvePrincipals(context);<br>
        这个很重要，就是将认证状态、host、request、response等信息放到subject中，
        Subject subject = doCreateSubject(context); 
        save(subject); // 将认证状态和上面说的那个key放到session中，刚开始肯定什么都不做，都是空。登陆后才会真正在session中写入。
        return subject; 
  }
        
```

　以上Filter走完后，中间其他过程不要理会，后面进行登陆操作。

　通过SecurityUtils.getSubject() 获取当前线程中的subject，也就是上面 filter 中创建的subject。

   调用subject.login()方法进行登陆认证。

#### subject#login

```
public void login(AuthenticationToken token) throws AuthenticationException {
        clearRunAsIdentitiesInternal();<br>　　　　　获取登陆中的subject，解释在下面方法中。
        Subject subject = securityManager.login(this, token);
 
        PrincipalCollection principals;
 
        String host = null;
 
        if (subject instanceof DelegatingSubject) {
            DelegatingSubject delegating = (DelegatingSubject) subject;
            //we have to do this in case there are assumed identities - we don't want to lose the 'real' principals:
            principals = delegating.principals;
            host = delegating.host;
        } else {
            principals = subject.getPrincipals();
        }
 
        if (principals == null || principals.isEmpty()) {
            String msg = "Principals returned from securityManager.login( token ) returned a null or " +
                    "empty value.  This value must be non null and populated with one or more elements.";
            throw new IllegalStateException(msg);
        }
        this.principals = principals;
        this.authenticated = true;
        if (token instanceof HostAuthenticationToken) {
            host = ((HostAuthenticationToken) token).getHost();
        }
        if (host != null) {
            this.host = host;
        }<br>　　　　　// 从登陆中的subject中获取session，并将session放入filter中生成的subject。
        Session session = subject.getSession(false);
        if (session != null) {
            this.session = decorate(session);
        } else {
            this.session = null;
        }
    }
```

#### securityManager#login

```
public Subject login(Subject subject, AuthenticationToken token) throws AuthenticationException {
        AuthenticationInfo info;
        try {<br>　　　　　// 在自己实现的Realm中进行认证并获取认证信息
            info = authenticate(token);
        } catch (AuthenticationException ae) {
            try {
                onFailedLogin(token, ae, subject);
            } catch (Exception e) {
                if (log.isInfoEnabled()) {
                    log.info("onFailedLogin method threw an " +
                            "exception.  Logging and propagating original AuthenticationException.", e);
                }
            }
            throw ae; //propagate
        }
　　　　　将token和info还有以前的subject来创建登陆中的subject，这是一个新的subject，此处会在服务端生成一个session，并将一些信息返回给写入<br>　　　　　如：认证状态和上面说的那个无用的key。
        Subject loggedIn = createSubject(token, info, subject);
 
        onSuccessfulLogin(token, info, loggedIn);
 
        return loggedIn;
    }
```

　　

　　以上shiro登陆流程算是走完了，如果使用默认的 sessionManager，也就是ServletContainerSessionManager 会在会话后在 cookie 生成一个 JsessionId

![img](https://images2018.cnblogs.com/blog/1215497/201805/1215497-20180531171932691-822135300.png)

 

 　　用户在第二次进来的时候服务器会根据JseeionId帮你找到服务器的上次登陆的session，里面包含登陆的信息。

```
save(subject);也就是这个方法里面做的。最简单的shiro登陆流程算是结束了。　　但是这种在服务器存放的session有很多问题，在多节点的项目中，必然是登陆功能要重新登陆好几次，原因很简单，session不共享。为了解决这个问题，可以不要使用默认的sessionManager（如果没有配置就是默认的ServletContainerSessionManager）配置使用DefaultWebSessionManager，然后在内部配置实现。

<!-- 权限管理器 -->
<bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">
    <!-- 基于redis登录校验的实现 -->
    <property name="realm" ref="systemAuthorizingRealm"/>
    <!-- session 管理器 -->
    <property name="sessionManager" ref="sessionManager"/>
</bean>
<bean id="sessionManager" class="org.apache.shiro.web.session.mgt.DefaultWebSessionManager">
    <!--session 超时时间：30分钟 -->
    <property name="globalSessionTimeout" value="${shiro.sessionTimeout}"/>
    <property name="sessionIdCookie" ref="sessionIdCookie"/>
    <!--持久化shiro session，以适应集群环境-->
    <property name="sessionDAO" ref="redisSessionDao"/>
</bean>
<!-- 指定本系统SESSIONID, 默认为: JSESSIONID 问题: 与SERVLET容器名冲突, 如JETTY, TOMCAT
    等默认JSESSIONID, 当跳出SHIRO SERVLET时如ERROR-PAGE容器会为JSESSIONID重新分配值导致登录会话丢失! -->
<bean id="sessionIdCookie" class="org.apache.shiro.web.servlet.SimpleCookie">
    <constructor-arg value="${shiro.sid}"/>
    <property name="httpOnly" value="true"/>
    <property name="domain" value="${shiro.domain}"/>
</bean>

<bean id="redisSessionDao" class="com.***.***.shiro.RedisSessionDAO">
    <property name="expire" value="2700"/>
    <property name="keyPrefix" value="${shiro.sessionkey}"/>
</bean>
　使用redis作为存储session的容器就可以完美的解决了，另外cookie也完全可以自定义，不必完全是JsessionId。
　中间过程很简单，就不一一叙述了。
```



#### mergePrincipals

```
`save(subject);也就是这个方法里面做的。最简单的shiro登陆流程算是结束了。<br><br>　　但是这种在服务器存放的session有很多问题，在多节点的项目中，必然是登陆功能要重新登陆好几次，原因很简单，session不共享。<br><br>　　为了解决这个问题，可以不要使用默认的sessionManager（如果没有配置就是默认的ServletContainerSessionManager）。<br>　　配置使用DefaultWebSessionManager，然后在内部配置实现。`
```

　

在这个getSession方法中会对根据不同的sessionManager获取sessionId。
redis的sessionDAO实现也发出来吧，都是可以直接用的。

#### RedisSessionDAO

```
public class RedisSessionDAO extends AbstractSessionDAO {
 
    private static Logger logger = LoggerFactory.getLogger(RedisSessionDAO.class);
 
    /**
     * The Redis key prefix for the sessions
     */
    private String keyPrefix = "shiro_redis_session:";
 
    /**
     * redis 缓存过期时间/秒
     */
    private int expire = 60 * 60;
 
    /**
     * save session
     *
     * @param session
     * @throws UnknownSessionException
     */
    private void saveSession(Session session) throws UnknownSessionException {
        logger.debug("saveSession");
        if (session == null || session.getId() == null) {
            logger.error("session or session id is null");
            return;
        }
        byte[] key = getByteKey(session.getId());
        byte[] value = SerializeUtils.serialize(session);
        session.setTimeout(expire * 1000);
        RedisUtils.set(key, value, expire);
    }
 
    @Override
    public void update(Session session) throws UnknownSessionException {
        logger.debug("update");
        this.saveSession(session);
    }
 
    @Override
    public void delete(Session session) {
        logger.debug("delete");
        if (session == null || session.getId() == null) {
            logger.error("session or session id is null");
            return;
        }
        RedisUtils.del(this.getByteKey(session.getId()));
    }
 
    @Override
    public Collection<Session> getActiveSessions() {
        logger.debug("getActiveSessions");
        Set<Session> sessions = new HashSet<>();
        Set<byte[]> keys = RedisUtils.keys(this.keyPrefix + "*");
        if (keys != null && keys.size() > 0) {
            for (byte[] key : keys) {
                Session s = (Session) SerializeUtils.deserialize(RedisUtils.get(key));
                sessions.add(s);
            }
        }
        return sessions;
    }
 
    @Override
    protected Serializable doCreate(Session session) {
        logger.debug("doCreate");
        Serializable sessionId = this.generateSessionId(session);
        this.assignSessionId(session, sessionId);
        this.saveSession(session);
        return sessionId;
    }
 
    @Override
    protected Session doReadSession(Serializable sessionId) {
        logger.debug("doReadSession,sessionId:{}", sessionId);
        if (sessionId == null) {
            logger.error("session id is null");
            return null;
        }
        try {
            return (Session) SerializeUtils.deserialize(RedisUtils.get(this.getByteKey(sessionId)));
        } catch (Exception e) {
            logger.error("Failed to deserialize", e);
            return null;
        }
    }
 
    /**
     * 获得byte[]型的key
     *
     * @param sessionId
     * @return
     */
    private byte[] getByteKey(Serializable sessionId) {
        String preKey = this.keyPrefix + sessionId;
        return preKey.getBytes(StandardCharsets.UTF_8);
    }
 
    /**
     * Returns the Redis session keys
     * prefix.
     *
     * @return The prefix
     */
    public String getKeyPrefix() {
        return keyPrefix;
    }
 
    /**
     * Sets the Redis sessions key
     * prefix.
     *
     * @param keyPrefix The prefix
     */
    public void setKeyPrefix(String keyPrefix) {
        this.keyPrefix = keyPrefix;
    }
 
    public int getExpire() {
        return expire;
    }
 
    public void setExpire(int expire) {
        this.expire = expire;
    }
}
```

　　

说的比较笼统，只要自己debug过的就会很清楚，弄懂之后shiro这块登陆和身份权限验证也就简单多了。自己也可以写一套了，不需要这么的复杂，

有其他的业务也可以直接扩展。