

## SpringMVC请求流程

一个典型的SpringMVC请求流程如图所示，详细分为12个步骤：  
* 用户发起请求，由前端控制器DispatcherServlet处理
* 前端控制器通过处理器映射器查找 handler，可以根据XML或者注解去找
(调用handlerMapping获取handlerChain)
* 处理器映射器返回执行链
(获取支持该handler解析的HandlerAdapter)
* 前端控制器请求处理器适配器来执行handler
(使用HandlerAdapter完成handler处理)
* 处理器适配器来执行handler
* 处理业务完成后，会给处理器适配器返回ModeAndView对象，其中有视图名称，模型数据
* 处理器适配器将视图名称和模型数据返回到前端控制器
* 前端控制器通过视图解析器来对视图进行解析
* 视图解析器返回真正的视图给前端控制器
* 前端控制器通过返回的视图和数据进行渲染
* 返回渲染完成的视图
* 将最终的视图返回给用户，产生响应
```
1. 用户发送请求至前端控制器DispatcherServlet
2. DispatcherServlet收到请求调用HandlerMapping处理器映射器。
3. 处理器映射器根据请求url找到具体的处理器，生成处理器对象及处理器拦截器(如果有则生成)一并返回给DispatcherServlet。
3. DispatcherServlet通过HandlerAdapter处理器适配器调用处理器
4. 执行处理器(Controller，也叫后端控制器)。
5. Controller执行完成返回ModelAndView
6. HandlerAdapter将controller执行结果ModelAndView返回给DispatcherServlet
7. DispatcherServlet将ModelAndView传给ViewReslover视图解析器
8. ViewReslover解析后返回具体View
9. DispatcherServlet对View进行渲染视图（即将模型数据填充至视图中）。
10 DispatcherServlet响应用户。
```

在给定的顺序中，DispatcherServlet将首先调用每个HandlerInterceptor的preHandle方法，如果所有的preHandle方法都返回true，那么最后调用handler本身。

#### 添加组件配置

```java
//添加处理器映射器
<bean class="org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping" />
<bean class="org.springframework.web.servlet.handler.RequestMappingHandlerMapping" />
// 添加处理器适配器
<bean class="org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter" />
<bean class="org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter" />
<bean class="org.springframework.web.servlet.mvc.RequestMappingHandlerAdapter" />
//添加视图解析器
<bean class="org.springframework.web.servlet.view.InternalResourceViewResolver" />
```
#### web.xml配置

```xml
 <!--springmvc前端控制器-->
    <servlet>
        <servlet-name>mvc-dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:mvc-dispatcher.xml</param-value>
        </init-param>
    </servlet>

    <servlet-mapping>
        <servlet-name>mvc-dispatcher</servlet-name>
        <url-pattern>*.action</url-pattern>
    </servlet-mapping>
```

ContextLoaderListener 实现了 ServletContextListener，由servlet标准可知 ServletContextListener 是容器的生命周期方法，springmvc就借助其启动与停止

ContextLoadListener调用了 initWebApplicationContext 方法,创建 WebApplicationContext 作为 spring 的容器上下文
DispatcherServlet 创建 WebApplicationContext 作为springmvc 的上下文 并将 ContextLoadListener 创建的上下文设置为自身的parent

### DispatcherServlet 请求处理

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	...
	try {
		ModelAndView mv = null;
		Exception dispatchException = null;
		try {
			processedRequest = checkMultipart(request);
			multipartRequestParsed = (processedRequest != request);
			//1.调用handlerMapping获取handlerChain
			mappedHandler = getHandler(processedRequest);
			if (mappedHandler == null || mappedHandler.getHandler() == null) {
				noHandlerFound(processedRequest, response);
				return;
			}
			// 2.获取支持该handler解析的HandlerAdapter
			HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
			...
			// 3.使用HandlerAdapter完成handler处理
			mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
			if (asyncManager.isConcurrentHandlingStarted()) {
				return;
			}
			// 4.视图处理(页面渲染)
			applyDefaultViewName(request, mv);
			mappedHandler.applyPostHandle(processedRequest, response, mv);
		}
		catch (Exception ex) {
			dispatchException = ex;
		}
		processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
	}
	...
}
```
调用HandlerMapping得到HandlerChain(Handler+Intercept)
调用HandlerAdapter执行handle过程(参数解析 过程调用)
异常处理(过程类似HanderAdapter)
调用ViewResolver进行视图解析
渲染视图

HandlerMapping下属子类可分为两个分支；

AbstractHandlerMethodMapping
AbstractUrlHandlerMapping

```java
//HandlerMapping
public interface HandlerMapping {
  //根据request获取处理链
   HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;
}
/**
获取所有 object 子类
根据条件过滤出 handle 处理类
解析 handle 类中定义的处理方法
保存解析得出的映射关系
*/

//HandlerAdapter
public interface HandlerAdapter {
   //判断是否支持该handler类型的解析
   boolean supports(Object handler);
   //参数解析 并调用handler完成过程调用 
   ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;
   //用于处理http请求头中的last-modified
   long getLastModified(HttpServletRequest request, Object handler);
}
/**
装载带有 ControllerAdvices 注解的对象
装载 ArgumentResolvers(默认+自定义)
装载 InitBinderArgumentResolvers(默认+自定义)
装载 ReturnValueHandlers(默认+自定义)
*/
```

## 九大组件

1. HandlerMapping

    是用来查找 Handler 的。在SpringMVC中会有很多请求，每个请求都需要一个 Handler 处理，具体接收到一个请求之后使用哪个 Handler 进行处理呢？这就是 HandlerMapping 需要做的事。
2. HandlerAdapter

    一个适配器。因为SpringMVC中的 Handler 可以是任意的形式，只要能处理请求就ok，但是 Servlet 需要的处理方法的结构却是固定的，都是以 request 和 response 为参数的方法。如何让固定的Servlet处理方法调用灵活的Handler来进行处理呢？这就是HandlerAdapter要做的事情。

    小结：Handler是用来干活的工具；HandlerMapping 用于根据需要干的活找到相应的工具；HandlerAdapter是使用工具干活的人，参数，类型的适配？？？
3. HandlerExceptionResolver

    专门对异常情况进行处理，在SpringMVC中就是HandlerExceptionResolver。具体来说，根据异常设置ModelAndView，之后再交给 render 方法进行渲染。
4. ViewResolver

    ViewResolver用来将String类型的视图名和Locale解析为View类型的视图。View是用来渲染页面的，也就是将程序返回的参数填入模板里，生成html（也可能是其它类型）文件。这里就有两个关键问题：使用哪个模板？用什么技术（规则）填入参数？这其实是ViewResolver主要要做的工作，ViewResolver需要找到渲染所用的模板和所用的技术（也就是视图的类型）进行渲染，具体的渲染过程则交由不同的视图自己完成。
5. RequestToViewNameTranslator

    ViewName是根据ViewName查找View，但有的Handler处理完后并没有设置View也没有设置ViewName，这时就需要从request获取ViewName了，如何从request中获取ViewName就是RequestToViewNameTranslator要做的事情了。RequestToViewNameTranslator在Spring MVC容器里只可以配置一个，所以所有request到ViewName的转换规则都要在一个Translator里面全部实现。
6. LocaleResolver

    解析视图需要两个参数：一是视图名，另一个是Locale。视图名是处理器返回的，Locale是从哪里来的？这就是LocaleResolver要做的事情。LocaleResolver用于从request解析出Locale，Locale就是zh-cn之类，表示一个区域，有了这个就可以对不同区域的用户显示不同的结果。SpringMVC主要有两个地方用到了Locale：一是ViewResolver视图解析的时候；二是用到国际化资源或者主题的时候。
7. ThemeResolver

    用于解析主题。SpringMVC中一个主题对应一个properties文件，里面存放着跟当前主题相关的所有资源、如图片、css样式等。SpringMVC的主题也支持国际化，同一个主题不同区域也可以显示不同的风格。SpringMVC中跟主题相关的类有 ThemeResolver、ThemeSource和Theme。主题是通过一系列资源来具体体现的，要得到一个主题的资源，首先要得到资源的名称，这是ThemeResolver的工作。然后通过主题名称找到对应的主题（可以理解为一个配置）文件，这是ThemeSource的工作。最后从主题中获取资源就可以了。
8. MultipartResolver

    用于处理上传请求。处理方法是将普通的request包装成MultipartHttpServletRequest，后者可以直接调用getFile方法获取File，如果上传多个文件，还可以调用getFileMap得到FileName->File结构的Map。此组件中一共有三个方法，作用分别是判断是不是上传请求，将request包装成MultipartHttpServletRequest、处理完后清理上传过程中产生的临时资源。
9. FlashMapManager

    用来管理FlashMap的，FlashMap主要用在redirect中传递参数。

### @RequestMapping的属性

RequestMapping是一个用来处理请求地址映射的注解，可用于类或方法上。用于类上，表示类中的所有响应请求的方法都是以该地址作为父路径。  
RequestMapping注解有六个属性，下面我们把她分成三类进行说明。
1. value， method；

  value： 指定请求的实际地址，指定的地址可以是URI Template 模式（后面将会说明）；

  method： 指定请求的method类型， GET、POST、PUT、DELETE等；

2. consumes，produces；

  consumes： 指定处理请求的提交内容类型（Content-Type），例如application/json, text/html;

  produces: 指定返回的内容类型，仅当request请求头中的(Accept)类型中包含该指定类型才返回；

3. params，headers；

  params： 指定request中必须包含某些参数值是，才让该方法处理。

  headers： 指定request中必须包含某些指定的header值，才能让该方法处理请求。



## 请求传参方式

SpringMVC 页面传递参数到controller的五种方式
一共是五种传参方式：

### 一：直接将请求参数名作为Controller中方法的形参

public  String login (String username,String password)   ：
解释：括号中的参数必须与页面Form 表单中的name 名字相同

### 二：使用@RequestParam 绑定请求参数参数值

自动调用request的getParament方法，而且能自动转型  
getParament获取结果为字符串=>(long,int)
举例：public String login(RequestParam ("username") String name) :
解释：双引号中的username 必须与页面vlaue名字相同，String name 中的name可以随便写

### 三：用注解@RequestMapping接收参数的方法

 @RequestMapping(value="/login/{username}/{password}")

public String login(@PathVariable("username") String name，@PathVariable("password") String name)   

解释:上面的 @RequestMapping(value="/login/{username}/{password}") 是以注解的方式写在方法上的。注解上的usernname和 password 必须好页面上value 相同

### 四：使用Pojo对象（就是封装的类，类中封装的字段作为参数）绑定请求参数值，原理是利用Set的页面反射机制找到User对象中的属性

举例：@ReauestMapping（value=/login”）
public String login(User user){
解释：就是把封装的一个类当成一个参数放在方法中，封装类中的属性就是就是参数。

### 五：使用原生的Servlet API 作为Controller 方法的参数

public String login(HttpServletRequest request){

String usernma=Request.getParameter("username");
}
解释：使用request 请求页面参数的方式获取从页面传过来的参数

### org.springframework.asm.ClassReader



### SpringMVC：前后端传值方式总结

```java
//@PathVarible
@RequestMapping(value="/findarticlesbyclassify/{classifyId}",method=RequestMethod.GET)
public String  findArticlesByClassify(@PathVariable String classifyId){
   ...
}
//前端： http://localhost:8080/article/findarticlesbyclassify/aaac4e63-da8b-4def-a86c-6543d80a8a1

//@PathParam
@RequestMapping(value = "/findarticlesbyclassify",method=RequestMethod.GET)
public String  findArticlesByClassify(@PathParam("classifyId") String classifyId){
    ...
}
//前端：http://localhost:8080/article/findarticlesbyclassify?classifyId=aaac4e63-da8b-4def-a86c-6543d80a8a1

//@RequestParam
//主要用来处理 Content-Type 为 application/x-www-form-urlencoded编码 的内容
@RequestMapping(value = "/findarticlesbyclassify",method=RequestMethod.GET)
public String  findArticlesByClassify(@RequestParam("classifyId") String classifyId){
    ...
}
```

@ RequestBody
@RequestHeader
Spring自动封装







