## JAVAEE 十三大规范

### 1. JDBC

（Java Data Base Connectivity,java数据库连接）是一种用于执行SQL语句的Java API，可以为多种关系数据库提供统一访问，它由一组用Java语言编写的类和接口组成。JDBC提供了一种基准，据此可以构建更高级的工具和接口，使数据库开发人员能够编写数据库应用程序。
与之相对应的是微软公司开发的ODBC(Open Database Connectivity）它建立了一组规范，并提供了一组对数据库访问的标准API（应用程序编程接口）。这些API利用SQL来完成其大部分任务。ODBC本身也提供了对SQL语言的支持，用户可以直接将SQL语句送给ODBC。

### 2. JNDI

(Java Naming and Directory Interface,Java命名和目录接口)是SUN公司提供的一种标准的Java命名系统接口，JNDI提供统一的客户端API，通过不同的访问提供者接口JNDI服务供应接口(SPI)的实现，由管理者将JNDI API映射为特定的命名服务和目录系统，使得Java应用程序可以和这些命名服务和目录服务之间进行交互。

在没有JNDI之前：
开发的时候，在连接数据库代码中需要对JDBC驱动程序类进行应用，通过一个URL连接到数据库。但是这样存在问题，比如我要改一种数据库，是不是要更换驱动，更换URL。每次都要进行这些配置和管理。

在有了JNDI之后：
可以在J2ee容器中配置JNDI参数，定义一个数据源，在程序中，通过数据源名称引用数据源从而访问后台数据库。在程序中定义一个上下文类，然后用content.lookup(“java:数据源名称”)就可以成功引入数据源了。

### 3. EJB

EJB(Enterprise JavaBean)是sun的JavaEE服务器端组件模型，设计目标与核心应用是部署分布式应用程序。用通俗的话来理解，就是把已经打包好的东西放到服务器中去执行，这样是凭借了java跨平台的优势，利用EJB技术部署分布式系统可以不限于特定的平台。EJB定义了服务器端组件是如何被编写以及提供了在组件和管理它们的服务器和组件间的标准架构协议.

### 4. RMI

RMI(Remote Method Invocation，远程方法调用）是Java的一组拥护开发分布式应用程序的API。RMI使用Java语言接口定义了远程对象，它集合了Java序列化和Java远程方法协议(Java Remote Method Protocol)。我的理解是：能够让在某个 Java 虚拟机上的对象调用另一个 Java 虚拟机中的对象上的方法。可以用此这个方法调用的任何对象必须实现该远程接口。

### 5. Java IDL/CORBA

Java IDL技术在Java平台上添加了CORBA(Common Object Request Broker Architecture)功能，提供了基于标准的互操作能力和连接性。Java IDL技术使得分布式的Java Web应用能够通过使用工业标准的IDL和IIOP(Internet Inter-ORB Protocol)来透明地调用远程网络服务的操作。运行时组件(Runtime Components)包括了一个用于分布式计算且使用IIOP通信的Java ORB.我对这个规范的理解，它也是借用了java的集成，让新旧系统集成，或是客户端跨平台的使用。

### 6. JSP

JSP全名为Java Server Pages，中文名叫java服务器页面，其根本是一个简化的Servlet设计，它是由Sun Microsystems公司倡导、许多公司参与一起建立的一种动态网页技术标准。JSP的定义让我想到做BS项目时候的ASP.NET技术。JSP页面也是用HTML和JS的交互，服务器在页面被客户端所请求以后对这些Java代码进行处理，然后将生成的HTML页面返回给客户端的浏览器。

### 7. Java Servlet

一种J2EE组件，servlet可被认为是运行在服务器端的applet，Servlets提供了基于组件、平台无关的方法用以构建基本Web的应用程序。Servlet必须部署在Java servlet容器才能使用，为了在web容器里注册上面的Servlet，为应用建一个web.xml入口文件。servlets全部由Java写成并且生成HTML。

### 8. XML

（Extensible Markup Language）可扩展标记语言，标准通用标记语言的子集，是一种用于标记电子文件使其具有结构性的标记语言。近年来，随着 Web的应用越来越广泛和深入，人们渐渐觉得HTML不够用了，HTML过于简单的语法严重地阻碍了用它来表现复杂的形式。尽管HTML推出了一个又一个新版本，已经有了脚本、表格、帧等表达功能，但始终满足不了不断增长的需求。
有人建议直接使用SGML 作为Web语言，这固然能解决HTML遇到的困难。但是SGML太庞大了，用户学习和使用不方便尚且不说，要全面实现SGML的浏览器就非常困难，于是自然会想到仅使用SGML的子集，使新的语言既方便使用又实现容易。正是在这种形势下，Web标准化组织W3C建议使用一种精简的SGML版本——XML应运而生了。

### 9. JMS

JMS即Java消息服务（Java Message Service）应用程序接口是一个Java平台中关于面向消息中间件（MOM）的API，用于在两个应用程序之间，或分布式系统中发送消息，进行异步通信。用一个很形象的例子，如果有人请我吃饭，她给我打电话占线，她决定先去占个位置，但是如果没有短信技术，那么是不是我就不知道她给我的消息了呢？为了保证这样的异步通信，我可以看到短信，准时去赴约。JMS就是提供了这样一个面向消息的中间件。它们提供了基于存储和转发的应用程序之间的异步数据发送，即应用程序彼此不直接通信，而是与作为中介的MOM 通信。MOM提供了有保证的消息发送，应用程序开发人员无需了解远程过程调用（PRC）和网络/通信协议的细节。

### 10. JTA

JTA，即Java Transaction API，JTA允许应用程序执行分布式事务处理——在两个或多个网络计算机资源上访问并且更新数据。JDBC驱动程序的JTA支持极大地增强了数据访问能力。我得理解是，利用了事务处理，可以让数据等到同步的更新，技术上可以支持多个服务器的分布式访问。

### 11. JTS

组件事务监视器（component transaction monitor）按照事务性对象的调用方法定义。这样可以使得资源透明被征用。

### 12. JavaMail

JavaMail API提供了一种独立于平台和独立于协议的框架来构建邮件和消息传递应用程序。JavaMail API可以为使用一个可选包Java SE平台也包括在Java EE平台.The JavaMail API provides a platform-independent and protocol-independent framework to build mail and messaging applications. The JavaMail API is available as an optional package for use with the Java SE platform and is also included in the Java EE platform.——Oracle官网。我的理解：是一个提供给使用java平台的开发者处理电子邮件有关的编程接口。

### 13. JAF

JAF 是一个平台，是基于java平台的一个扩展，它的好处是：让你利用标准的平台服务，决定一个任意类型的数据，封装并访问它。发现可用的操作，并适用于实体bean来执行操作。JavaBeans Activation Framework (JAF) is a standard extension to the Java platform that lets you take advantage of standard services to: determine the type of an arbitrary piece of data; encapsulate access to it; discover the operations available on it; and instantiate the appropriate bean to perform the operation(s).
我的理解：JavaMail利用JAF来处理MIME编码的邮件附件。MIME的字节流可以被转换成Java对象，或者转换自Java对象。大多数应用都可以不需要直接使用JAF。