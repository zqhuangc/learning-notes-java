AnnotationConfigApplicationContext#register(ConfigurationClass)

AnnotationConfigApplicationContext#refresh

xml  标签元素   对应   标签注解

DispatcherServletConfiguration

元编程



### Servlet 3.0+ 自动装配

生命周期问题

ServletContext，ServletContextListener <ServletContainerInitializer

> ServletContainerInitializer#onStartup 当容器启动时
>
>  ServletContextListener#contexInitialized  当ServletContext 初始化时

### Spring Web 自动装配

@HandlesTypes(WebApplicationInitializer.class)

SpringServletContainerInitializer

默认扫描 WEB-INF/classes，WEB-INF/lib

onStartup  参数 Set<Class<?>> 关心的类对象(需要加载注册的)

WebApplicationInitializer 子类 

> Abstract ContextLoader Initializer  -> ContextLoader  Listener
>
> Abstract DispatcherServlet Initializer ->  DispatcherServlet 
>
> Abstract  Annotation Config DispatcherServlet  Initializer  ->  DispatcherServlet 
>
> onStartup 方法分析



mvn -Dmaven.test.skip -U clean package

Build   tomcat打包插件



### Spring web mvc 自动装配



### 条件装配

@Conditional

@ConditionalOnClass

可选装配

Qualifier

