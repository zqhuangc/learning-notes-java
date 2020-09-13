mutable



MIME、RFC2616

GenericServlet 和 HttpServlet

Servlet 默认是线程不安全的，需要开发人员处理多线程问题
通常 Web 容器对于并发请求将使用同一个 servlet 处理，并且在不同的线程中并发执行 service 方法

getLastModified

jsr 的 servlet 规范

### Servlet 生命周期

>  Servlet 如何被加载、实例化、初始化、处理客户端请求，以及何时结束服务。

每个 Servlet 对应一个 ServletConfig   ->  ServletContext

 ServletRequest 、 ServletResponse

SingleThreadModel接口（不推荐使用）



HttpUpgradeHandler



### ServletContext 接口

Servlet 可以使用 ServletContext 对象记录事件，获取 URL 引用的资源，
存取当前上下文的其他 Servlet 可以访问的属性。

#### 作用范围

每一个部署到容器的 Web 应用都有一个 Servlet 接口的实例与之关联。在容器分布在多台虚拟机的情况下，
每个 JVM 的每个 Web 应用将有一个 ServletContext 实例。





#### 扩展方式

编程方式定义 Servlet、Filter 和它们映射到的 url 模式（url pattern）

方案：

1. ServletContainerInitializer 的 onStartup 方法

2. ServletContextListener 的 contextInitialized,contextDestroyed 方法
   事件监听机制
   ServletContextEvent

> Spring MVC作为web的框架，提供由servlet容器解析的配置，有两种方案，一种是使用ServletContextListener的事件监听机制来完成框架的初始化，另一种方式是实现ServletContainerInitializer接口。第一种可以借助web.xml注册listener或者使用注解来注册listener，第二种与web.xml无关，是经由Jar service API来发现的。两种机制是不同的，后者的执行时机是在所有的Listener触发之前。这些都是servlet规范的实现，具体的逻辑都在内部的接口方法实现中

#### 资源

ClassLoader#getResource
ClassLoader#getResourceAsStream



HttpServletResponse#sendRedirect

setCharacterEncoding，setContentType 和 setLocale



ServletOutputStream.setWriteListener

HttpSessionBindingListener

getMaxInactiveInterval







如果 web.xml 描述符中的 metadata-complete 元素设置为 true，则存在于 class 文件和
绑定在 jar 包中的 web-fragments 中的指定部署信息的注解将不被处理。





当在 web.xml、web-fragment.xml 和 注解之间解析发生冲突时 web 应用的 web.xml 具有最高优先级。



RequestDispatcher#forward

getRequestDispatcher



### 共享库 /  运行时可插拔性(ServletContainerInitializer )

ServletContainerInitializer 类通过 jar services API 查找。对于每一个应用，应用启动时，由容器创建一个
ServletContainerInitializer 实 例 。 



#### HandlesTypes 注解

在 ServletContainerInitializer 实现上的
**HandlesTypes 注解**用于表示感兴趣的一些类，它们可能指定了 HandlesTypes 的 value 中的注解（类型、方法或自动级别的注解），或者是其类型的超类继承/实现了这些类之一。



ServletContainerInitializer’s 的 onStartup 得到一个类的 Set，其或者继承/实现 initializer 表示感兴趣的类，或
者它是使用指定在@HandlesTypes 注解中的任意类注解的



### 异步处理（Servlet3.0+）

AsyncContext #dispatch，#complete

ServletRequest#startAsync



 asyncSupported=true

* 监听器类配置

与其他监听器不同，AsyncListener 类型的监听器可能仅通过编程式注册（使用一个 ServletRequest）。



#### 安全

ServletRegistration 接口的 setServletSecurity 



### 注解

@Resource 