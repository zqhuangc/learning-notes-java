# Spring Boot Web

SpringBoot里面没有我们之前常规web开发的WebContent（WebApp），它只有src目录

在src/main/resources下面有两个文件夹，static和templates springboot默认 static中放静态页面，而templates中放动态页面

在不使用第三方jar包的情况下, Springboot不能直接访问templates下的静态页面, 需要加其他jar包依赖。

## 传统 Servlet 回顾
什么是Servlet？
​	Servlet 是一种基于 Java 技术的 Web 组件，用于生成动态内容，由容器管理。类似于其他 Java 技术组件，Servlet 是平台无关的 Java 类组成，并且由 Java Web 服务器加载执行。

什么是Servlet容器？
​	Servlet 容器，有时候也称作为 Servlet 引擎，作为Web服务器或应用服务器的一部分。通过请求和响应对话，提供 Web 客户端与 Servlets 交互的能力。容器管理Servlets实例以及它们的生命周期。

历史
​	1997年六月，Servlet 1.0 版本发行
### 核心接口
#### Servlet 3.0 前时代

##### 服务组件
javax.servlet.Servlet
javax.servlet.Filter（since Servlet 2.3）

##### 上下文组件
javax.servlet.ServletContext
javax.servlet.http.HttpSession
javax.servlet.http.HttpServletRequest
javax.servlet.http.HttpServletResponse
javax.servlet.http.Cookie（客户端）

##### 配置
javax.servlet.ServletConfig
javax.servlet.FilterConfig（since Servlet 2.3 ）

##### 输入输出
javax.servlet.ServletInputStream
javax.servlet.ServletOutputStream

##### 异常
javax.servlet.ServletException



##### 事件（since Servlet 2.3 ）

###### 生命周期类型

javax.servlet.ServletContextEvent
javax.servlet.http.HttpSessionEvent
java.servlet.ServletRequestEvent

###### 属性上下文类型
javax.servlet.ServletContextAttributeEvent
javax.servlet.http.HttpSessionBindingEvent
javax.servlet.ServletRequestAttributeEvent



##### 监听器（since Servlet 2.3）
生###### 命周期类型
javax.servlet.ServletContextListener
javax.servlet.http.HttpSessionListener
javax.servlet.http.HttpSessionActivationListener
javax.servlet.ServletRequestListener
###### 属性上下文类型
javax.servlet.ServletContextAttributeListener
javax.servlet.http.HttpSessionAttributeListener
javax.servlet.http.HttpSessionBindingListener
javax.servlet.ServletRequestAttributeListener

#### Servlet 3.0 后时代

##### 组件申明注解
@javax.servlet.annotation.WebServlet
@javax.servlet.annotation.WebFilter
@javax.servlet.annotation.WebListener
@javax.servlet.annotation.ServletSecurity
@javax.servlet.annotation.HttpMethodConstraint
@javax.servlet.annotation.HttpConstraint
##### 配置申明
@javax.servlet.annotation.WebInitParam

##### 上下文
javax.servlet.AsyncContext

##### 事件
javax.servlet.AsyncEvent

##### 监听器
javax.servlet.AsyncListener

##### Servlet 组件注册
javax.servlet.ServletContext#addServlet()
javax.servlet.ServletRegistration

##### Filter 组件注册
javax.servlet.ServletContext#addFilter()
javax.servlet.FilterRegistration

##### 监听器注册
javax.servlet.ServletContext#addListener()
javax.servlet.AsyncListener

##### 自动装配
###### 初始器
javax.servlet.ServletContainerInitializer

###### 类型过滤
@javax.servlet.annotation.HandlesTypes



### 生命周期

#### Servlet 生命周期

* 初始化
  当容器启动或者第一次执行时，Servlet#init(ServletConfig)方法被执行，初始化当前Servlet 。

* 处理请求
  当HTTP 请求到达容器时，Servlet#service(ServletRequest,ServletResponse) 方法被执行，来处理请求。

* 销毁
  当容器关闭时，容器将会调用Servlet#destroy 方法被执行，销毁当前Servlet。

#### Filter 生命周期
* 初始化
  当容器启动时，Filter#init(FilterConfig)方法被执行，初始化当前Filter。

* 处理请求
  当HTTP 请求到达容器时，Filter#doFilter(ServletRequest,ServletResponse,FilterChain) 方法被执行，来拦截请求，在Servlet#service(ServletRequest,ServletResponse) 方法调用前执行。

* 销毁
  当容器关闭时，容器将会调用Filter#destroy 方法被执行，销毁当前Filter。



##Servlet on Spring Boot 
### Servlet 组件扫描
@org.springframework.boot.web.servlet.ServletComponentScan

* 指定包路径扫描
String[] value() default {}
String[] basePackages() default {}

* 指定类扫描
Class<?>[] basePackageClasses() default {}

### 注解方式注册
#### Servlet 组件
* 扩展 javax.servlet.Servlet
javax.servlet.http.HttpServlet
org.springframework.web.servlet.FrameworkServlet

* 标记 @javax.servlet.annotation.WebServlet

#### Filter 组件
* 实现 javax.servlet.Filter
  org.springframework.web.filter.OncePerRequestFilter

* 标记 @javax.servlet.annotation.WebFilter

#### 监听器组件
* 实现Listener接口
  javax.servlet.ServletContextListener
  javax.servlet.http.HttpSessionListener
  javax.servlet.http.HttpSessionActivationListener
  javax.servlet.ServletRequestListener
  javax.servlet.ServletContextAttributeListener
  javax.servlet.http.HttpSessionAttributeListener
  javax.servlet.http.HttpSessionBindingListener
  javax.servlet.ServletRequestAttributeListener
  标记 @javax.servlet.annotation.WebListener


### Spring Boot API方式注册	
#### Servlet 组件
* 扩展 javax.servlet.Servlet
  javax.servlet.http.HttpServlet
  org.springframework.web.servlet.FrameworkServlet
* 组装 Servlet
  - Spring Boot 1.4.0 开始支持
    org.springframework.boot.web.servlet.ServletRegistrationBean
  - Spring Boot  1.4.0 之前
    org.springframework.boot.context.embedded.ServletRegistrationBean
* 暴露 Spring Bean
  @Bean

#### Filter 组件
* 实现 javax.servlet.Filter
  org.springframework.web.filter.OncePerRequestFilter

* 组装 Filter
  - Spring Boot 1.4.0 开始
    org.springframework.boot.web.servlet.FilterRegistrationBean
  - Spring Boot  1.4.0 之前
    org.springframework.boot.context.embedded.FilterRegistrationBean
* 暴露 Spring Bean
  @Bean

#### 监听器组件
* 实现 Listener

* 组装 Listener
  - Spring Boot 1.4.0 开始
    org.springframework.boot.web.servlet.ServletListenerRegistrationBean
  - Spring Boot  1.4.0 之前
    org.springframework.boot.context.embedded.ServletListenerRegistrationBean
* 暴露 Spring Bean
  @Bean



## JSP on Spring Boot
激活
激活 传统Servlet Web部署
Spring Boot 1.4.0 开始
org.springframework.boot.web.support.SpringBootServletInitializer

组装 org.springframework.boot.builder.SpringApplicationBuilder

配置JSP视图
org.springframework.boot.autoconfigure.web.WebMvcProperties
spring.mvc.view.prefix
spring.mvc.view.suffix



# REST

### 幂等与非幂等

幂等

PUT 

初始状态：0

修改状态：1 * N

最终状态：1



DELETE

初始状态：1

修改状态：0 * N

最终状态：0



非幂等

POST

初始状态：1

修改状态：1 + 1 =2 

N次修改： 1+ N = N+1

最终状态：N+1



幂等/非幂等 依赖于服务端实现，这种方式是一种契约



### 自描述消息（MessageConverter）

> Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8

第一优先顺序：text/html -> application/xhtml+xml -> application/xml

第二优先顺序：image/webp -> image/apng



《Spring Boot 编程思想》



# 学习源码的路径：

@EnableWebMvc

​DelegatingWebMvcConfiguration

​		WebMvcConfigurationSupport#addDefaultHttpMessageConverters

所有的 HTTP 自描述消息处理器均在 messageConverters（类型：`HttpMessageConverter`)，这个集合会传递到 RequestMappingHandlerAdapter，最终控制写出。

application/json 在SpringBoot中默认使用jackson2

messageConverters，其中包含很多自描述消息类型的处理，比如 JSON、XML、TEXT等等



以 application/json 为例，Spring Boot 中默认使用 Jackson2 序列化方式，其中媒体类型：application/json，它的处理类 MappingJackson2HttpMessageConverter，提供两类方法：

1. 读read* ：通过 HTTP 请求内容转化成对应的 Bean
2. 写write*： 通过 Bean 序列化成对应文本内容作为响应内容



问题：为什么第一次是JSON，后来怎加了 XML 依赖，又变成了 XML 内用输出

回答：Spring Boot 应用默认没有增加XML 处理器（HttpMessageConverter）实现，所以最后采用轮询的方式去逐一尝试是否可以 canWrite(POJO) ,如果返回 true，说明可以序列化该 POJO 对象，那么 Jackson 2 恰好能处理，那么Jackson 输出了。

### 问答

问题：当 Accept 请求头未被制定时，为什么还是 JSON 来处理

回答：这个依赖于 messageConverters 的插入顺序。



问题：优先级是默认的是吧 可以修改吗

回答：是可以调整的，通过extendMessageConverters 方法调整



### 扩展自描述消息

Person

JSON 格式（application/json)

```json
{
	"id":1,
	"name":"小马哥"
}
```

XML 格式（application/xml）

```xml
<Person>
    <id>1</id>
    <name>小马哥</name>
</Person>
```

Properties 格式（application/properties+person)

（需要扩展）

```properties
person.id = 1
person.name = 小马哥
```



1. 实现 AbstractHttpMessageConverter 抽象类
   1. supports 方法：是否支持当前POJO类型
   2. readInternal 方法：读取 HTTP 请求中的内容，并且转化成相应的POJO对象（通过 Properties 内容转化成 JSON）
   3. writeInternal 方法：将 POJO 的内容序列化成文本内容（Properties格式），最终输出到 HTTP 响应中（通过 JSON 内容转化成 Properties ）

- @RequestMappng 中的 consumes 对应 请求头 “Content-Type”
- @RequestMappng 中的 produces   对应 请求头 “Accept”



HttpMessageConverter 执行逻辑：

- 读操作：尝试是否能读取，canRead 方法去尝试，如果返回 true 下一步执行 read
- 写操作：尝试是否能写入，canWrite 方法去尝试，如果返回 true 下一步执行 write

如果请求中accept中和produces不一样，controller能执行吗？
不能执行 406 415

MediaType */*

为什么一定要经过 POJO 
JSON -> 反序列化 -> Map -> Properties  变复杂了



MVC

M : Model  
V : View  
C : Controller -> DispatcherServlet  
Front Controller = DispatcherServlet    
Application Controller = @Controller or Controller  

ServletContextListener -> ContextLoaderListener -> Root WebApplicationContext

DispatcherServlet -> Servlet WebApplicationContext

Services => @Service

Repositories => @Repository
​						  
​						  
2. 请求映射

Servlet / 和 /*

ServletContext path = /servlet-demo

URI : /servlet-demo/IndexServlet

DispatcherServlet < FrameworkServlet < HttpServletBean < HttpServlet

自动装配 : org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration

ServletContext path = "" or "/"

Request URI = ServletContex path + @RequestMapping("")/ @GetMapping()

当前例子：

Request URI = "" + "" = "" -> RestDemoController#index()

Request URI : "" 或者 "/"

HandlerMapping ，寻找Request URI，匹配的 Handler ：

	Handler：处理的方法，当然这是一种实例
	整体流程：Request -> Handler -> 执行结果 -> 返回（REST） -> 普通的文本
	
	请求处理映射：RequestMappingHandlerMapping -> @RequestMapping Handler Mapping
	
	拦截器：HandlerInterceptor 可以理解 Handler 到底是什么
	
			处理顺序：preHandle(true) -> Handler: HandlerMethod 执行(Method#invoke) -> postHandle -> afterCompletion
					  preHandle(false)


Spring Web MVC 的配置 Bean ：WebMvcProperties

Spring Boot 允许通过 application.properties 去定义一下配置，配置外部化

WebMvcProperties 配置前缀：spring.mvc      例如：spring.mvc.servlet 

## 异常处理

### 传统的Servlet web.xml 错误页面

* 优点：统一处理，业界标准
* 不足：灵活度不够，只能定义 web.xml文件里面

<error-page> 处理逻辑：

 * 处理状态码 <error-code>
 * 处理异常类型 <exception-type>
 * 处理服务：<location>

### Spring Web MVC 异常处理

 * @ExceptionHandler
    * 优点：易于理解，尤其是全局异常处理
    * 不足：很难完全掌握所有的异常类型
 * @RestControllerAdvice = @ControllerAdvice + @ResponseBody
 * @ControllerAdvice 专门拦截（AOP） @Controller

### Spring Boot 错误处理页面

 * 实现 ErrorPageRegistrar
    * 状态码：比较通用，不需要理解Spring WebMVC 异常体系
    * 不足：页面处理的路径必须固定
 * 注册 ErrorPage 对象
 * 实现 ErrorPage 对象中的Path 路径Web服务



## 视图技术
### View
#### render 方法

处理页面渲染的逻辑，例如：Velocity、JSP、Thymeleaf

### ViewResolver

View Resolver = 页面 + 解析器 -> resolveViewName 寻找合适/对应 View 对象

RequestURI-> RequestMappingHandlerMapping ->

HandleMethod -> return "viewName" ->

完整的页面名称 = prefix + "viewName" + suffix 

-> ViewResolver -> View -> render -> HTML

Spring Boot 解析完整的页面路径：

spring.view.prefix + HandlerMethod return + spring.view.suffix

#### ContentNegotiationViewResolver
用于处理多个ViewResolver：JSP、Velocity、Thymeleaf

当所有的ViewResover 配置完成时，他们的order 默认值一样，所以先来先服务（List）

当他们定义自己的order，通过order 来倒序排列

### Thymeleaf
#### 自动装配类：ThymeleafAutoConfiguration
配置项前缀：spring.thymeleaf

模板寻找前缀：spring.thymeleaf.prefix

模板寻找后缀：spring.thymeleaf.suffix

代码示例：/thymeleaf/index.htm

prefix: /thymeleaf/

return value : index

suffix: .htm


## 国际化（i18n）
Locale

```java
LocaleContextHolder
```
message

Locale/LocaleContext

LocaleContextHolder

LocaleResolver/LocaleContextResolver



## 嵌入式容器

org.springframework.boot.context.embedded.EmbeddedServletContainerCustomizer



org.springframework.boot.context.embedded.ConfigurableEmbeddedServletContainer




# 《阿里开源工程 Spring WebMVC 扩展之内容协商》

BasicErrorController

### 客户端请求

 https://developer.mozilla.org/en-US/docs/Web/HTTP/Content_negotiation 

```
accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3
```



### 服务端响应

```
content-type: text/html; charset=utf-8
```



```
content-type: application/xml; charset=utf-8
```





## Spring Web MVC 视图处理



### 视图模板

`@Controller` 

HandlerMethod 方法返回的内容视图的地址

### REST 处理

`@RestController` = `@Controller` + `@ResponseBody`

HandlerMethod 方法返回的 Body 内容

`ResponseEntity` = Body + Header





HandlerMethod = public 方法标注 `@RequestMapping`





### 架构特色

Spring Web MVC 允许多套视图处理器（ViewResolver）并存



#### 模板渲染

##### 媒体类型(Accept 请求)

text/html -> Velocity

> text/*
>
> */*

text/json -> themleaf

text/xml -> JSP

##### 请求后缀(URL)

xxx.html -> Velocity

xxx.json -> themleaf

xxx.xml -> JSP





URL : http://acme.com/abc.json?format=application/xml

Accept : text/html


#### 

​	








