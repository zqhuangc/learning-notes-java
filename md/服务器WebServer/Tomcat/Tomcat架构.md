![](https://ws1.sinaimg.cn/large/006xzusPgy1g1zrqp2jpcj30iw0bwdhn.jpg)

## 目录结构

### conf 目录

valve  filter

 /../ 问题

`catalina.policy`：Tomcat 安全策略文件，控制 JVM 相关权限，具体可以参考`java.security.Permission`

`catalina.properties`：Tomcat Catalina行为控制配置文件，比如 Common ClassLoader

`logging.properties`:Tomcat 日志配置中件，JDK Logging   

* handlers ------ appender， LEVEL

`server.xml`：Tomcat Server 配置文件

* `GlobalNamingResources`：全局 JNDI 资源
* `Context`:

`context.xml`：全局 Context 配置

`tomcat-users.xml`: Tomcat 角色配置文件，（Realm 文件实现方式）

`web.xml`：Servlet 标准的 web 部署文件，Tomcat 默认实现部分配置入内

* `org.apache.catalina.servlets.DefaultServlet`
* `org.apache.jasper.servlet.JspServlet`







E-Tag，Last-Modified





#### lib 目录

Tomcat 存放公用类库

`ecj-4.4.2.jar`：Eclipse  Java 编译器

`jasper.jar`：JSP 编译器



#### logs 目录

`localhost.{data}.log`：当应用起不来，可查看该文件

* `NoClassDefFoundError`
* `ClassNotFoundException`

`catalina.{data}.log`： 控制台输出，System.out （setOut）可以外置



### webapps 目录

简化 web 应用部署的方式



## 部署 Web 应用

https://tomcat.apache.org/tomcat-7.0-doc/config/context.html

**Defining a context**

#### 方法一：放置在 webapps 目录

生产环境不推荐，如 docker 部署问题

####  方法二：修改 `conf/server.xml`(推荐)

* 配置 server.xml   

> <Context docBase="项目物理路径"  path="melody"/>  代表一个web项目
>
>
>
> 类 ContextConfig

Digester 解析xml 反射 pojo

熟悉配置元素，可以参考对应类型源码 setter方法

`Container`

* `Context`

**该方式不支持动态部署，建议在生产环境使用**



#### 方法三：独立 `context` xml 配置文件

首先注意 `conf\Catalina\localhost`



独立 `context` xml 配置文件路径：

`${Tomcat_Home}/conf/${Engine_Name}/${host_name}/${ContextPath}.xml`

如：`${Tomcat_Home}/conf/Catalina/localhost/abc.xml`

问题：watch_doc

**注：该方式可以实现热部署（reloadable = true），热加载，因此建议在开发环境使用**



####  方法四：host 中 appBase 能否修改？

#### 方法五：/META-INF/context.xml

自己测试，应该有问题





### [I/O连接器](https://tomcat.apache.org/tomcat-7.0-doc/config/http.html)

实现类 Connector

server.xml  doc解释

注意实现：setProtocol



NIO方式  仅读请求头、等待下一请求非阻塞，可参考连接方式比较



URIEncoding



## 问答：

### 问题一：如果配置 path，以文件名为主还是 以配置为主

独立 `context` xml 配置文件时 `path` 属性设置无效

### 问题二：根独立 `context` xml 配置文件路径

${Tomcat_Home}/conf/${Engine_Name}/${host_name}/ROOT.xml

### 问题三：热部署实现

调整`<context>`属性 reloadable = true

### 问题四：连接器里面的线程池用的是哪个

注意`server.xml`中的 connector 注释部分

StandardThreadExecutor 将连接处理委派给 Java 标准线程池

### 问题五：JNDI

java 的 Context.lookup          依赖查找

tomcat   的 JNDI Resource

```xml
<Context ...>
  ...
  <Resource name="mail/Session" auth="Container"
            type="javax.mail.Session"
            mail.smtp.host="localhost"/>
  ...
</Context>
```



```java
Context initCtx = new InitialContext();
Context envCtx = (Context) initCtx.lookup("java:comp/env");
Session session = (Session) envCtx.lookup("mail/Session");

Message message = new MimeMessage(session);
message.setFrom(new InternetAddress(request.getParameter("from")));
InternetAddress to[] = new InternetAddress[1];
to[0] = new InternetAddress(request.getParameter("to"));
message.setRecipients(Message.RecipientType.TO, to);
message.setSubject(request.getParameter("subject"));
message.setContent(request.getParameter("content"), "text/plain");
Transport.send(message);
```

