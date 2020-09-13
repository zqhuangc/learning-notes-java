## Servlet

![](https://ws1.sinaimg.cn/large/006xzusPly1g5gpywlye0j30jf07mtbz.jpg)

![](https://ws1.sinaimg.cn/large/006xzusPly1g5h5zxx1roj30qz0b6tcs.jpg)

![](https://ws1.sinaimg.cn/large/006xzusPly1g5h6269wkjj30xi0aywnj.jpg)

> 2.3    事件监听
>
> 3.0   异步            security 难用 
>
> 3.1   非阻塞

payload    body

主机地址，

解析请求头、请求体     输入 



![](https://ws1.sinaimg.cn/large/006xzusPly1g5gs5wsfkhj30qs08fq5h.jpg)



![](https://ws1.sinaimg.cn/large/006xzusPly1g5guwqxmibj30fc0f8dit.jpg)

看版本，看方法

编译时解释，运行时解释

Play 框架 

ServletContextListener

ContextLoaderListener

ContextLoader  WebApplicationContext



ServletConfig

GenericServlet

HttpServletBean



init()

FilterConfig      跟 servlet 方式 差不多

编码过滤问题：filter 顺序

filter场景（servlet 规范）

**生命周期 Lifecycle**

初始化  启动

看启动日志，看初始化信息

http://www.corej2eepatterns.com/

jsp 对象

#### 主要内容（核心：servlet 规范）

servlet 在 springmvc 是如何加载，如何初始化的



规范：编程模式/模型，API



## JSP

![](https://ws1.sinaimg.cn/large/006xzusPly1g5h6vv137oj30m60ci77v.jpg)

Tomcat  

development =false，jsp不会编译

Ant 脚本 编译



#### 前端控制器模式

JspServlet

![](https://ws1.sinaimg.cn/large/006xzusPly1g5hqgilqiwj30g10cgwha.jpg)

### spring mvc

Handler 方法

注解  派生



InternalViewResolver  怎么关联  view

resolveViewName    setViewClass

viewClass ---- JstlView

set

ModelAndView

双返回值

web.xml     dispatcherServlet    namespace

![](https://ws1.sinaimg.cn/large/006xzusPly1g5hwm6u55zj30pt04tdhe.jpg)



#### 手动装配

@EnableWebMVC

AbatractAnnotationConfigDispacherServletInitializer

#### Spring Boot 运用

debug 查看是否有自动装配类，如：

dispatcherServlet    

InternalViewResolver  



资源存放位置



#### @Value  与  @ConfigurableProperties 区别？

@Value 平铺式

@ConfigurableProperties  树形

@EnableConfigurableProperties

离应用越近的配置越优先 ？？？？



ConfigFileApplicationContextInitializer

PropertySourceLoader

Loader