 

 

 

 

**Socket**

 

**socket** **选项 TCP NO DELAY 是指什么**

**Socket** **工作在 TCP/IP 协议栈是哪一层**

**TCP****、UDP 区别及 Java 实现方式**

 

  

 

**XML**

 

**XML****文档定义有几种形式？它们之间有何本质区别？解析XML文档有哪几种方式？DOM 和 SAX 解析器有什么不同？**

**Java****解析XML的方式**

**用 jdom 解析 xml 文件时如何解决中文问题？如何解析**

**你在项目中用到了 XML 技术的哪些方面？如何实现**



**请简述 Servlet 的生命周期及其相关的方法**

​     Servlet的生命周期一般分四步，

加载-->实例化-->服务-->销毁

加载：

​     加载一般是在运行tomcat容器时来完成，将servlet类加载到tomcat中，或者是客户端发来请求时也可以

实例化：

​     实例化一般是即读取配置信息、读取初始化参数等，这些基本上在整个生命周期中只需要执行一次。关于init（）方法已经在积累GenericServlet中提供缺省实现，如果不需特殊处理则没有必要再进行定义，否则要重写。

服务：

​     服务一般是当容器接收到客户端请求时，Servlet引擎将创建一个ServletRequest请求对象和一个ServletResponse响应对象，然后把这两个对象作为参数传递给对应Servlet对象的service方法。（该方法是一个重点实现的方法，ServletRequest对象可以获得客户端发出请求的相关信息，如请求参数等，ServletResponse对象可以使得Servlet建立响应头和状态代码，并可以写入响应内容返回给客户端。在此说明一点，当Servlet中有doGet（）或者doPost（）方法时，那么service方法就可以省略，默认为调用这两个方法）

销毁：

​    销毁一般是Servlet的卸载是由容器本身定义和实现，在卸载Servlet之前需要调用destroy（）方法，以让Servlet自行释放占用的系统资源。虽然Java虚拟机提供了垃圾自动回收处理机制，但是有一部分资源却是该机制不能处理或延迟很久才能处理的，如关闭文件，释放数据库连接等。一般tomcat关闭，servlet就会被销毁，如果想提前销毁，可以写一个监听



setInterval

### cookie 和 session

存储信息
客户端：cookie：JSESSIONID
服务端：ConcurrentHashMap

key、values、过期时间、路径、域



把用户信息存储到 session 中
客户端只要关闭浏览器，session就失效



session 共享、session 复制、jwt、cookie





**Cookie** **和 Session的区别**

​     cookie机制采用的是在客户端保持状态的方案，而session机制采用的是在服务器端保持状态的方案。

 

**get** **和 post请求的区别**

区别一

get是从服务器上获取的数据。

post则是向服务器传送数据。

区别二

get是把参数数据队列加到提交表单的ACTION属性所指的URL中，值和表单内各个字段一一对应，在URL中可以看到。

post是通过HTTP post机制，将表单内各个字段与其内容放置在HTML HEADER内一起传送到ACTION属性所指的URL地址。用户看不到这个过程。

区别三

get方式，服务器端用Request.QueryString获取变量的值。

post方式，服务器端用Request.Form获取提交的数据。

区别四

get传送的数据量较小，不能大于2KB。

post传送的数据量较大，一般被默认为不受限制。但理论上，IIS4中最大量为80KB，IIS5中为100KB。

 

区别五

get安全性比较低。

post安全性较高。

区别六

根据 HTTP 规范，GET 用于信息获取，而且应该是 安全的和幂等的。所谓安全的意味着该操作用于获取信息而非修改信息。换句话说，GET 请求一般不应产生副作用。幂等的意味着对同一 URL 的多个请求应该返回同样的结果。完整的定义并不像看起来那样严格。从根本上讲，其目标是当用户打开一个链接时，她可以确信从自身的角度来看没有改变资源。 比如，新闻站点的头版不断更新。虽然第二次请求会返回不同的一批新闻，该操作仍然被认为是安全的和幂等的，因为它总是返回当前的新闻。

POST 表示可能改变服务器上的资源的请求。仍然以新闻站点为例，读者对文章的注解应该通过 POST 请求实现，因为在注解提交之后站点已经不同了

区别七

在FORM提交的时候，如果不指定Method，则默认为GET请求，Form中提交的数据将会附加在url之后，以?分开与url分开。字母数字字符原 样发送，但空格转换为“+“号，其它符号转换为%XX,其中XX为该符号以16进制表示的ASCII（或ISO Latin-1）值。GET请求请提交的数据放置在HTTP请求协议头中。

而POST提交的数据则放在实体数据中；GET方式提交的数据最多只能有1024字节，而POST则没有此限制。

### Servlet的生命周期

分为5个阶段：加载、创建、初始化、处理客户请求、卸载。

(1)加载：容器通过类加载器使用servlet类对应的文件加载servlet

(2)创建：通过调用servlet构造函数创建一个servlet对象

(3)初始化：调用 init 方法初始化

(4)处理客户请求：每当有一个客户请求，容器会创建一个线程来处理客户请求 service()

(5)卸载：调用 destroy 方法让 servlet 自己释放其占用的资源



### JSP

JSP 四大作用域： page (作用范围最小)、request、session、application（作用范围最大）。

存储在application对象中的属性可以被同一个WEB应用程序中的所有Servlet和JSP页面访问。（属性作用范围最大）

存储在session对象中的属性可以被属于同一个会话（浏览器打开直到关闭称为一次会话，且在此期间会话不失效）的所有Servlet和JSP页面访问。

存储在request对象中的属性可以被属于同一个请求的所有Servlet和JSP页面访问（在有转发的情况下可以跨页面获取属性值），例如使用PageContext.forward和PageContext.include方法连接起来的多个Servlet和JSP页面。

存储在pageContext对象中的属性仅可以被当前JSP页面的当前响应过程中调用的各个组件访问，例如，正在响应当前请求的JSP页面和它调用的各个自定义标签类。



### XML解析方式

Java 内置解析 XML API: DOM、SAX
XML 解析框架 JDOM
XML 解析框架 DOM4J
XML 解析框架 XStream

### JavaWeb开发中跨域问题的解决方案
没有权限
1. 设置 document.domain（一级域名相同的情况之下）

2. HTML标签中 src 属性，只支持get请求，允许跨域

3. <script src=""> JSON格式 eval

4. iframe 之间交互 window.postMessage(字符串限长255个)

5. 服务器后台做文章：CORS（安全沙箱）
   Acess-Control-Allow-Origin:*;







 

**请简述一下 Ajax 的原理及实现步骤**

​     原理： HTTP协议的异步通信

​     简单来说通过XmlHttpRequest对象来向服务器发异步请求，从服务器获得数据，然后用javascript来操作DOM而更新页面。

**……****简单描述Struts的主要功能**

**……****什么是 N 层架构**

**什么是CORBA？用途是什么**

​    CORBA体系结构是对象管理组织（OMG）为解决分布式处理环境（DCE）中，硬件和软件系统的互连而提出的一种解决方案

​    CORBA是一种远程分布式方法调用，是服务器端和客户端传输数据的方式。

​    **CORBA****的好处在于IDL接口规范，于是这种传输方式可以跨平台、跨语言**

用途：

存取来自现行桌面应用程序的分布信息和资源；

使现有业务数据和系统成为可供利用的网络资源；

为某一特定业务用的定制的功能和能力来增强现行桌面工具和应用程序； 

改变和发展基于网络的系统以反映新的拓扑结构或新资源；