## Filter介绍

  Filter可以认为是Servlet的一种“加强版”，它主要用于对用户请求进行预处理，也可以对HttpServletResponse进行后处理，是个典型的处理链。Filter也可以对用户请求生成响应，这一点与Servlet相同，但实际上很少会使用Filter向用户请求生成响应。使用Filter完整的流程是：Filter对用户请求进行预处理，接着将请求交给Servlet进行预处理并生成响应，最后Filter再对服务器响应进行后处理。

  Filter有如下几个用处。



在HttpServletRequest到达Servlet之前，拦截客户的HttpServletRequest。
根据需要检查HttpServletRequest，也可以修改HttpServletRequest头和数据。
在HttpServletResponse到达客户端之前，拦截HttpServletResponse。
根据需要检查HttpServletResponse，也可以修改HttpServletResponse头和数据。
​     Filter有如下几个种类。

用户授权的Filter：Filter负责检查用户请求，根据请求过滤用户非法请求。
日志Filter：详细记录某些特殊的用户请求。
负责解码的Filter:包括对非标准编码的请求解码。
能改变XML内容的XSLT Filter等。
Filter可以负责拦截多个请求或响应；一个请求或响应也可以被多个Filter拦截。
​     创建一个Filter只需两个步骤

创建Filter处理类
web.xml文件中配置Filter
   创建Filter必须实现javax.servlet.Filter接口，在该接口中定义了如下三个方法。

void init(FilterConfig config):用于完成Filter的初始化。
void destory():用于Filter销毁前，完成某些资源的回收。
void doFilter(ServletRequest request,ServletResponse response,FilterChain chain):实现过滤功能，该方法就是对每个请求及响应增加的额外处理。该方法可以实现对用户请求进行预处理(ServletRequest request)，也可实现对服务器响应进行后处理(ServletResponse response)—它们的分界线为是否调用了chain.doFilter(),执行该方法之前，即对用户请求进行预处理；执行该方法之后，即对服务器响应进行后处理。

## Interceptor介绍

拦截器，在AOP(Aspect-Oriented Programming)中用于在某个方法或字段被访问之前，进行拦截，然后在之前或之后加入某些操作。拦截是AOP的一种实现策略。

在WebWork的中文文档的解释为—拦截器是动态拦截Action调用的对象。它提供了一种机制可以使开发者可以定义在一个Action执行的前后执行的代码，也可以在一个Action执行前阻止其执行。同时也提供了一种可以提取Action中可重用的部分的方式。

拦截器将Action共用的行为独立出来，在Action执行前后执行。这也就是我们所说的AOP，它是分散关注的编程方法，它将通用需求功能从不相关类之中分离出来；同时，能够共享一个行为，一旦行为发生变化，不必修改很多类，只要修改这个行为就可以。

拦截器将很多功能从我们的Action中独立出来，大量减少了我们Action的代码，独立出来的行为就有很好的重用性。

当你提交对Action(默认是.action结尾的url)的请求时，ServletDispatcher会根据你的请求，去调度并执行相应的Action。在Action执行之前，调用被Interceptor截取，Interceptor在Action执行前后执行。

创建Interceptor必须实现com.opensymphony.xwork2.interceptor.Interceptor接口，该接口定义了如下三个方法。

void init():在该拦截器被实例化之后，在该拦截器执行拦截之前，系统将回调该方法。对于每个拦截器而言，其init()方法只执行一次。因此，该方法的方法体主要用于初始化资源。

void destory():该方法与init()方法对应。在拦截器实例被销毁之前，系统将回调该拦截器的destory方法，该方法用于销毁在init方法里打开的资源。

String intercept(ActionInvocation invocation):该方法是用户需要实现的拦截动作。就像Action的execute方法一样。intercept方法会返回一个字符串作为逻辑视图。如果该方法直接返回了一个字符串，系统会将跳转到该逻辑视图对应的实际视图资源，不会调用被拦截的Action。该方法的ActionInvocation参数包含了被拦截的Action的引用，可以通过调用该参数的invoke方法，将控制权转给下一个拦截器，或者转给Action的execute方法(如果该拦截器后没有其他拦截器，则直接执行Action的execute方法)。

## 过滤器(Filter)和拦截器(Interceptor)的区别

Filter是基于函数回调的，而Interceptor则是基于Java反射的。
Filter依赖于Servlet容器，而Interceptor不依赖于Servlet容器。
Filter对几乎所有的请求起作用，而Interceptor只能对action请求起作用。
Interceptor可以访问Action的上下文，值栈里的对象，而Filter不能。
在action的生命周期里，Interceptor可以被多次调用，而Filter只能在容器初始化时调用一次。

## Filter和Interceptor的执行顺序

过滤前-拦截前-action执行(servlet)-拦截后-过滤后