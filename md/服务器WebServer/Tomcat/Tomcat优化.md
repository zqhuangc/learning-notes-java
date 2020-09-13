# 源码分析

* 寻找加载 servlet 源码

> Context -> StandardContext.loadOnStartup() 
>
> wrapper   servlet?



* 如何找到监控端口

Connector   xxxmethod

> Connector#initInternal->   protocolHandler#init   -> AbstractProtocol#init  --->  endpoint#init   --->AbstractEndpoint#bind
>
> 默认 NIO

endpoint



## Tomcat 架构

![](https://ws1.sinaimg.cn/large/006xzusPgy1g1zrqp2jpcj30iw0bwdhn.jpg)

* server.xml    ---->   源码   ----->    架构



## 源码分析，从 author 角度考虑

1. load -----> conf/server.xml      important   file

   start a new server instance      找到 server.xml   解析 dom4j   apache --->digester
   Bootstrap.main()  ------> Catalina.load()   --->

2. start()   ----



init/destroy  生命周期管理  lifecycle  

sever



实现领域驱动



优雅上下线，优雅关闭



# Tomcat 性能优化

## Tomcat 配置调优

![](https://ws1.sinaimg.cn/large/006xzusPly1g5dhqrjsyoj30jx0d5jt9.jpg)

### 减少配置优化



* 场景一：假设当前 REST 应用（微服务）

  分析：它不需要静态资源，Tomcat 容器静态和动态

  - 静态处理：`DefaultServlet`
    - 优化方案：通过移除 `conf/web.xml `中的 ` DefaultServlet`

  - 动态：应用 `Servlet`，` JspServlet`
    - 优化方案：通过移除 `conf/web.xml `中的 ` JspServlet`

  > DispatcherServlet：Spring Web MVC 应用 Servlet
  >
  > JspServlet：编译并且执行 Jsp 页面
  >
  > DefaultServlet：Tomcat 处理静态资源的 Servlet

  * 移除 `conf/web.xml`中 welcome-file-list

  * 如果程序是 REST  JSON  Content-Type 或者 MIME Type：application/json

  * 移除 Session 设置

    对于微服务 REST 应用，不需要 Session，因为不需要状态

    Spring Security OAuth 2.0、JWT

    Session 通过 jsessionid 进行用户跟踪， HTTP 无状态，需要一个 ID 与当前用户会话联系。Spring Session HttpSession jsessionId 作为 Redis Key，实现多机器登录，用户会话不丢失

    存储方法：Cookie、URL 重写、SSL。

  * 移除 Valve

    `Valve `类似于 `Filter`

    移除`AccessLogValve`，可以通过 Nginx 的 Access Log 替代，`Valve`实现都需要消耗 Java 应用的计算时间。

* 场景二：需要 JSP 的情况（要去了解 web.xml）

  分析：`JspServlet`无法移除，了解 `JspServlet`处理原理

  > Servlet 周期：
  >
  > * 实例化：Servlet 和 Filter 实现类必须包含默认构造器。反射的方式，进行实例化
  > * 初始化：Servlet 容器调用 Servlet 或 Filter init() 方法
  > * 销毁：Servlet 容器关闭时，Servlet 或者 Filter destroy() 方法调用

  Servlet 或 Filter  在一个容器中，一般情况在一个 Web App 中是一个单例，不排除应用定义多个（不同名）

JspServlet 相关的优化 `ServletConfig `参数：

preCompile

- 需要编译

  - compiler
  - modificationTestInterval

- 不需要编译

  - development 设置 false

  development 设置 false，Jsp 怎么编译？
  优化方法：

  - Ant Task 执行 JSP 编译

  - Maven 插件：org.codehaus.mojo:jspc-maven-plugin

    - goals：jspc，脚本目录 scripts

    > JSP -> 翻译 .jsp 或者 .jspx 文件成 .java -> 编译 .class

* 总结

`conf/web.xml ` 作为 Servlet 应用的默认 `web.xml`，实际上应用程序存在两份`web.xml` ,其中包括应用的 `web.xml`，最终将两者合并。



JspServlet 如果 development 设置 true，它会自定义检查文件是否修改，如果修改，重新翻译，在编译（加载和执行）。言外之意，JspServlet 开发模式可能会导致 OOM，卸载Class 不及时会导致 Perm 区域不够

* 如何卸载一个 Class？

> Parent ClassLoader -> 1.class   2.class   3.class 
>
> Child ClassLoader  -> 4.class     5.class
>
> Child ClassLoader  load  1 - 5.class
>
> 1.class 需要卸载，需要将 ParenClassLoader 设置为 null， 当 ClassLoader 被 GC 后， 1 - 3 class 全部会被卸载
>
> 1.class 它是文件，文件被 JVM 加载，二进制 -> Verify -> 解析

JspRuntimeContext

JspServletWrapper



## 配置调整

### 关闭自动重载

（ reloadable=false）

`context.xml`

```xml
<Context docBase="项目物理路径"  path="melody" reloadable="false"/>
```



### 修改连接线程池数量

#### 通过 server.xml

```xml
<Executor name="tomcatThreadPool" namePrefix="catalina-exec-" maxThreads="150" minSpareThreads="4"/>

<Connector executor="tomcatThreadPool" port="8080" protocol="HTTP/1.1" connectionTimeout="20000" redirectPort="8443" />
```

通过程序来理解，`<Executor>`  实际的 Tomcat 接口：  

* `org.apache.catalina.Executor`

  * 扩展： J.U.C 标准接口`java.util.concurrent.Executor`

  * 实现： `org.apache.catalina.core.StandardThreadExecutor`

    * 线程数量

      ```java
          /**
           * max number of threads
           */
          protected int maxThreads = 200;
          
          /**
           * min number of threads
           */
          protected int minSpareThreads = 25;
      
          public void setMaxThreads(int maxThreads) {
                  this.maxThreads = maxThreads;
                  if (executor != null) {
                      executor.setMaximumPoolSize(maxThreads);
                  }
              }
      
          public void setMinSpareThreads(int minSpareThreads) {
              this.minSpareThreads = minSpareThreads;
              if (executor != null) {
                  executor.setCorePoolSize(minSpareThreads);
              }
          }
      
      ```

    * 线程池：`org.apache.tomcat.util.threads.ThreadPoolExecutor`(java.util.concurrent.ThreadPoolExecutor)
      afterExecute

    * 总结：Tomcat IO连接器使用的线程池实际标准的 Java 线程池的扩展，最大线程数量和最小线程数量实际上分别是 `MaximumPoolSize `和 `CorePoolSize`


#### 通过  JMX 调整

> jconsole + jmeter
>
> throughput 吞吐量
>
> nameprefix
>
> MBean 找到对应类直接设置（部分属性修改可能无效，因为本身已初始化，或状态可能未改）
>
> ThreadFactory

观察 `StandardThreadExecutor` 是否存在调整线程池数量的 API



评估一些参考：

1. 正确率
2. Load（CPU -> JVM GC）
3. TPS/QPS（越大越好）
4. CPU 密集型（加密/解密、算法）
5. I/O密集型，网络、文件读写等



问题：到底设置多少的线程数量才是最优？

首先，评估整体的情况量，假设 100W QPS，有机器数量 100 台， 每台支撑 1w QPS。

第二，进行压力测试，需要一些测试样本， JMeter 来实现，假设一次请求需要 RT 10ms，1 秒可以同时完成 100 个请求。10000/100 = 100 线程。

一般而言，确保 Load 不要太高。减少 Full GC，GC 取决于 JVM 堆的大小。执行一次操作需要 5MB 内存，50GB。

20 GB 内存，必然执行 GC。 要不调优程序，最好对象存储外化，比如 Redis，同时又需要评估 Redis 网络开销。又要评估网卡的接受能力。

第三，常规性压测，由于业务变更，会导致底层性能变化



## 程序调优

* 调整堆大小
  - 评估内存使用
* 调整 GC算法



## JVM 调优

### 调整 GC 算法

如果 Java 版本小于 9，默认 `PS MarkSweep`，可选设置 CMS、G1

如果 Java 9 的话，默认 `G1`



#### 默认算法

```txt
java -jar -server -XX:-PrintGCDetails -Xloggc:./lg/gc.log -XX:+HeapDumpOnOutOfMemoryError -Xmslg -Xmxlg -XX:MaxGCPauseMillis=250 -Djava.awt.headless=true stress-test-demo-0.0.1-SNATSHOP.jar
```



![](https://ws1.sinaimg.cn/large/006xzusPly1g5ehw0ak6cj30pz0e9jzt.jpg)



> `-server ` 主要提高吞吐量，在有限的资源，实现最大化利用
>
> `-client` 主要提高响应时间，主要是提高用户体验



#### G1 算法

![](https://ws1.sinaimg.cn/large/006xzusPly1g5ehwm1jmvj30px09dtek.jpg)

```txt
java -jar -server -XX:-PrintGCDetails -Xloggc:./lg/g1-gc.log 
-XX:+HeapDumpOnOutOfMemoryError -Xmslg -Xmxlg -XX:+UseG1GC -XX:+UseNUMA -XX:MaxGCPauseMillis=250 -Djava.awt.headless=true stress-test-demo-0.0.1-SNATSHOP.jar
```



![](https://ws1.sinaimg.cn/large/006xzusPly1g5ehurbh6yj30qb0evnea.jpg)



## Spring Boot 配置调优

查看官方文档 Tomcat相关

> @ConfigurationProperties
>
> ServerProperties
>
> 配置 server.tomcat.xxx = 

* 线程池数量
* JspServlet
  - tomcat-embedded-Jasper

ResourceProperties

* `application.properties`

```properties
## 线程池
## 取消 JspServlet
server.jspServlet.registered = false
## 取消 TomcatAccessLogValve
server.tomcat.accesslog.enabled = false
## 取消静态资源处理
## spring.resources.chain.enabled = false ????
```





## 问题

Tomcat.getServer().await() 和  Object.wait() 区别？

Tomcat.getServer().await()  利用 `Thread.sleep(long)` 实现：

```java
        if( port==-1 ) {
            try {
                awaitThread = Thread.currentThread();
                while(!stopAwait) {
                    try {
                        //
                        Thread.sleep( 10000 );
                    } catch( InterruptedException ex ) {
                        // continue and check the flag
                    }
                }
            } finally {
                awaitThread = null;
            }
            return;
        }
```



Object#wait()   方法也是 Native 方法    park











