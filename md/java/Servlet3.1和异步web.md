mutable



MIME、RFC2616

GenericServlet 和 HttpServlet

Servlet 默认是线程不安全的，需要开发人员处理多线程问题
通常 Web 容器对于并发请求将使用同一个 servlet 处理，并且在不同的线程中并发执行 service 方法
getLastModified

SingleThreadModel

>  Servlet 如何被加载、实例化、初始化、处理客户端请求，以及何时结束服务。

 ServletRequest 、 ServletResponse



 AsyncContext.dispatch     startAsyn



HttpUpgradeHandler



序监听器列表



ServletContainerInitializer 的 onStartup 方法

ServletContextListener 的 contextInitialized 方法

ClassLoader#getResource
ClassLoader#getResourceAsStream



HttpServletResponse#sendRedirect

setCharacterEncoding，
setContentType 和 setLocale



HttpSessionBindingListener

getMaxInactiveInterval



ServletContextEvent



如果 web.xml 描述符中的 metadata-complete 元素设置为 true，则存在于 class 文件和
绑定在 jar 包中的 web-fragments 中的指定部署信息的注解将不被处理。





当在 web.xml、web-fragment.xml 和 注解之间解析发生冲突时 web 应用的 web.xml 具有最高优先级。



RequestDispatcher#forward

getRequestDispatcher



### 共享库 /  运行时可插拔性

在 ServletContainerInitializer 实现上的
**HandlesTypes 注解**用于表示感兴趣的一些类，它们可能指定了 HandlesTypes 的 value 中的注解（类型、方法或自动级别的注解），或者是其类型的超类继承/实现了这些类之一。





### 异步

AsyncContext 

ServletRequest#startAsync



* 监听器类配置

与其他监听器不同，AsyncListener 类型的监听器可能仅通过编程式注册（使用一个 ServletRequest）。



#### 安全

ServletRegistration 接口的 setServletSecurity 



### 注解

@Resource 