<jsp:include>标签  
<jsp:forward>标签  
<jsp:param>标签  
taglib指令
JSP指令的基本语法格式：

<%@ 指令 属性名="值" %>

<%= 变量或表达式 %>

<% 
​		多行java代码 
 %> 

JSP九大隐式对象

> request 		HttpServletRequest
> response 	HttpServletResponse
> session 		HttpSession
> application 	ServletcContext
> config   		ServletConfig
> exception		(特殊情况下使用)
> page    		this(本JSP页面)
> out      		JspWriter（带缓冲的PrintWriter）
> pageContext	(使普通Java类可访问WEB资源，自定义标签常用)

通过pageContext获得其他对象
getException方法返回exception隐式对象 
getPage方法返回page隐式对象
getRequest方法返回request隐式对象 
getResponse方法返回response隐式对象 
getServletConfig方法返回config隐式对象
getServletContext方法返回application隐式对象
getSession方法返回session隐式对象 
getOut方法返回out隐式对象
pageContext封装其它8大内置对象的意义

代表各个域的常量
PageContext.APPLICATION_SCOPE
PageContext.SESSION_SCOPE
PageContext.REQUEST_SCOPE
PageContext.PAGE_SCOPE 
findAttribute方法    （*重点，先后查找各个域中的属性）


## EL
### 获得web开发常用对象
${隐式对象名称}  ：获得对象的引用
pageContext  对应于JSP页面中的pageContext对象（注意：取的是pageContext对象。）

pageScope  代表page域中用于保存属性的Map对象
requestScope  代表request域中用于保存属性的Map对象
sessionScope  代表session域中用于保存属性的Map对象
applicationScope  代表application域中用于保存属性的Map对象
param  表示一个保存了所有请求参数的Map对象  
paramValues  表示一个保存了所有请求参数的Map对象，它对于某个请求参数，返回的是一个string[]  
header  表示一个保存了所有http请求头字段的Map对象
headerValues  同上，返回string[]数组。注意：如果头里面有“-” ，例Accept-Encoding，则要headerValues[“Accept-Encoding”]  
cookie  表示一个保存了所有cookie的Map对象  
initParam  表示一个保存了所有web应用初始化参数的map对象


注意：有些Tomcat服务器如不能使用EL表达式
​	（1）升级成tomcat6
​	（2）在JSP中加入<%@ page isELIgnored="false" %>












#### EL表达式保留关键字












