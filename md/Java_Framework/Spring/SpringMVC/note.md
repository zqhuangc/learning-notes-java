springmvc private和public方法
关键底层是否setAccessable(true)

可认为能100%零配置（无xml，property）



@RequestMapping 请求路径
@ResponseBody  
@RequestBody    （请求参数格式转化） pojo
@RequestParam  （参数名不一样）必须有值
@PathVariable    （参数名一样） 取值

### 父子容器问题
```java
//（1）在spring-mvc.xml中有以下配置：
<!-- 只扫描 @Controller注解-->
<context:component-scanbase-package="xin.sun.blog.controlller">
    <context:include-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>

//（2）在spring-config.xml中有如下配置：
<!-- 配置扫描注解,不扫描 @Controller注解-->
<context:component-scan base-package="xin.sun.blog">
    <context:exclude-filter type="annotation" expression="org.springframework.stereotype.Controller"/>
</context:component-scan>
```
* 注意：  
Spring为了保证注入类的一致性，采用了双亲委托的机制，如果父容器中存在该类的实例那么优先使用父容器中的实例，如果父容器中没有该实例才会用子容器中的实例

### springmvc静态资源加载不了
点击登陆或退出时，页面失去样式.......
如果你的DispatcherServlet拦截 *.do或者*.action这样的URL，就不存在访问不到静态资源的问题。如果你的DispatcherServlet拦截“/”，拦截了所有的请求，同时对*.js,*.jpg的访问也就被拦截了。


法一：
在SpringMVC3.0之后推荐使用一： 
<!-- 静态资源访问 -->
  <mvc:default-servlet-handler/>
以下两种在SpringMVC3.0之前可以使用

法二：
也可以使用二：
  <!-- 静态资源访问
  <mvc:resources location="/img/" mapping="/img/**"/> 
  <mvc:resources location="/js/" mapping="/js/**"/>  
  <mvc:resources location="/css/" mapping="/css/**"/>
 -->
法三：
也可以使用三：
web.xml里添加如下的配置
<servlet-mapping>
​     <servlet-name>default</servlet-name>
​     <url-pattern>*.css</url-pattern>
</servlet-mapping>
<servlet-mapping>
​    <servlet-name>default</servlet-name>
​    <url-pattern>*.gif</url-pattern>
</servlet-mapping>
<servlet-mapping>
​     <servlet-name>default</servlet-name>
​     <url-pattern>*.jpg</url-pattern>
</servlet-mapping>
<servlet-mapping>
​     <servlet-name>default</servlet-name>
​     <url-pattern>*.js</url-pattern>
</servlet-mapping>
即可解决
但是我的又出现了这种怪异的情况
2、直接访问jsp页面可以加载页面样式，再通过controller访问同一个页面则加载不了？
刚开始想的原因有以下：
一种是路径问题：
request是会话请求，仅仅和这一次的请求相关，所以很容易被各种框架重写，比如需要指定跳转到其他地方。。
所以出现静态文件访问不到的情况，要么是路径问题，要么是使用了spring，但是没有指定过滤css，导致这些文件被spring直接拦截了
另一种是：在springmvc.xml中没有配置加载静态资源
我的就是路径问题。
我的访问路径是:
http://localhost:8080/maven_show/user/toLogin加载不出来样式，
加载不出来的看到图片的路径（通过控制台看到的404路径找不到）
http://localhost:8080/maven_show/user/images/p1.jpg 
直接访问http://localhost:8080/maven_show/show/index.jsp 就有样式
直接这样访问图片也能访问http://localhost:8080/maven_show/show/images/p1.jpg

解决办法：在jsp页面引入<% String path = request.getContextPath(); %>
访问页面的时候
例如：
<script src="<%=path %>/resources/js/jquery-1.8.3.min.js"></script>
<script type="text/javascript" src="<%=path %>/resources/js/move-top.js"></script>
<script type="text/javascript" src="<%=path %>/resources/js/easing.js"></script>
访问相对路径下的静态资源。


