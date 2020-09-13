##  日志框架 Log4j

* 介绍

Log4j 是目前最为流行的Java 日志框架之一，虽然已经停滞发展，并逐步被Logback和Log4j 2 等日志框架所替代，可是无法掩饰光辉历程，以及优良的设计理念。

* 背景

Almost every large application includes its own logging or tracing API. In compliance with this rule, the E.U. SEMPER project decided to write its own tracing API. This was in early 1996. After countless enhancements, several incarnations and much work that API evolved to become log4j。

The Complete log4j Manual

* 结束

On August 5, 2015 the Logging Services Project Management Committee announced that Log4j 1.x had reached end of life. 

- http://logging.apache.org/log4j/1.2/


![图片1.png](http://ww1.sinaimg.cn/large/006xzusPgy1ga8sp10mwmj30qo0g20w5.jpg)



### Log4j API

#### 日志对象（org.apache.log4j.Logger）



#### 日志级别（org.apache.log4j.Level）

OFF
FATAL
ERROR
INFO
DEBUG
TRACE
ALL
API 层次
​	-org.apache.log4j.Priority
​			-org.apache.log4j.Level

#### 日志管理器（org.apache.log4j.LogManager）

* 主要职责

初始化默认log4j 配置
维护日志仓储（org.apache.log4j.spi.LoggerRepository）
获取日志对象（org.apache.log4j.Logger）

#### 日志仓储（org.apache.log4j.spi.LoggerRepository）

*  主要职责

管理日志级别阈值（org.apache.log4j.Level）
管理日志对象（org.apache.log4j.Logger）

#### 日志附加器（org.apache.log4j.Appender）

日志附加器是日志事件（org.apache.log4j.LoggingEvent）具体输出的介质，如：控制台、文件系统、网络套接字等。
日志附加器（org.apache.log4j.Appender）关联零个或多个**日志过滤器**（org.apache.log4j.Filter），这些过滤器形成过滤链。

* 主要职责

附加日志事件（org.apache.log4j.LoggingEvent）
关联日志布局（org.apache.log4j.Layout）
关联日志过滤器（org.apache.log4j.Filter）
关联错误处理器（org.apache.log4j.spi.ErrorHandler）

![图片1.png](http://ww1.sinaimg.cn/large/006xzusPly1ga8tim2146j30nu0ligp1.jpg)

日志附加器（org.apache.log4j.Appender）

+ 控制台实现：org.apache.log4j.ConsoleAppender
+ 文件实现
  - 普通方式：org.apache.log4j.FileAppender
  - 滚动方式：org.apache.log4j.RollingFileAppender
  - 每日规定方式：org.apache.log4j.DailyRollingFileAppender
+ 网络实现
  - Socket方式：org.apache.log4j.net.SocketAppender
  - JMS方式：org.apache.log4j.net.JMSAppender
  - SMTP方式：org.apache.log4j.net.SMTPAppender
+ 异步实现：org.apache.log4j.AsyncAppender

![图片1.png](http://ww1.sinaimg.cn/large/006xzusPly1ga8tj5hyicj30p70lnn09.jpg)

#### 日志过滤器（org.apache.log4j.spi.Filter）

​	日志过滤器用于决策当前日志事件（org.apache.log4j.spi.LoggingEvent）是否需要在执行所关联的日志附加器（org.apache.log4j.Appender）中执行。
​	决策结果有三种：
DENY：日志事件跳过日志附加器的执行
ACCEPT：日志附加器立即执行日志事件
NEUTRAL：跳过当前过滤器，让下一个过滤器决策

#### 日志格式布局（org.apache.log4j.Layout）

日志格式布局用于格式化日志事件（org.apache.log4j.spi.LoggingEvent）为可读性的文本内容。

* 内建实现
  简单格式：org.apache.log4j.SimpleLayout
  模式格式：org.apache.log4j.PatternLayout
  提升模式格式：org.apache.log4j.EnhancedPatternLayout
  HTML格式：org.apache.log4j.HTMLLayout
  XML格式：org.apache.log4j.xml.XMLLayout
  TTCC格式：org.apache.log4j.TTCCLayout
  TTCC – Time、Thread、Category、nested diagnostic Context information

#### 日志事件（org.apache.log4j.LoggingEvent）

日志事件是用于承载日志信息的对象，其中包括：

日志名称

日志内容

日志级别

异常信息（可选）

当前线程名称

时间戳

嵌套诊断上下文（NDC）

映射诊断上下文（MDC）

#### 日志配置器（org.apache.log4j.spi.Configurator）

日志配置器提供外部配置文件配置log4j行为的API，log4j 内建了两种实现：

* Properties 文件方式（org.apache.log4j.PropertyConfigurator）

```properties
log4j.rootLogger=DEBUG, com
log4j.appender.com=org.apache.log4j.ConsoleAppender
log4j.appender.com.layout=org.apache.log4j.PatternLayout
log4j.appender.com.layout.conversionPattern=[%t] %-5p %c - %m%n
```



* XML 文件方式（org.apache.log4j.xml.DOMConfigurator）

```xml

```



#### 日志诊断上下文（org.apache.log4j.NDC、org.apache.log4j.MDC）

日志诊断上下文作为日志内容的一部分，为其提供辅助性信息，如当前 HTTP 请求 URL。Neil Harrison described this method in the book “Patterns for Logging Diagnostic Messages,” in Pattern Languages of Program Design 3。





 log4j 有两种类型的日志诊断上下文，分别是映射诊断上下文和嵌套诊断上下文：
嵌套诊断上下文（NDC）：以Key-Value的形式存储诊断信息
映射诊断上下文（MDC）：以堆栈的形式存储诊断信息





Log4jConfigListener

## Java logging

Java Logging 是Java 标准的日志框架，也称为 Java Logging API，即 JSR 47。从
Java 1.4 版本开始，Java Logging 成为 Java SE的功能模块，其实现类存在“java.util.logging”包下。

Java Logging API
日志对象（java.util.logging.Logger）
日志级别（java.util.logging.Level）
日志管理器（java.util.logging.LogManager）
日志处理器（java.util.logging.Handler	） 
日志过滤器（java.util.logging.Filter）
日志格式器（java.util.logging.Formatter）
日志记录（java.util.logging.LogRecord）
日志权限（java.util.logging.LoggingPermission）
日志JMX接口（java.util.logging.LoggingMXBean）



## 新一代日志框架 - [logback](https://logback.qos.ch/manual/index.html)

Logback 是 Log4j 的替代者，在架构和特征上有着相当提升。
重要提升
执行速度更快，内存占用更小
Slf4 无缝整合
自动重载配置文件
自动移除老的归档日志
自动压缩归档日志文件
条件化配置文件
解读：https://logback.qos.ch/reasonsToSwitch.html

callAppender


+ 核心API

  - 日志对象（ch.qos.logback.classic.Logger）

  - 日志级别（ch.qos.logback.classic.Level）

  - 日志管理上下文（ch.qos.logback.classic.LoggerContext）

  - 日志附加器（ch.qos.logback.core.Appender）

    信息的显示，追加

  - 日志过滤器（ch.qos.logback.core.filter.Filter）

  - 日志格式布局（ch.qos.logback.core.Layout）

  - 日志事件（ch.qos.logback.classic.spi.LoggingEvent）

  - 日志配置器（ch.qos.logback.classic.spi.Configurator）

    - BasicConfigurator

  - 日志诊断上下文（org.slf4j.MDC）

```java
LoggerContext loggerContext = new LoggerContext();
Logger logger = loggerContext.getLogger(LogBackDemo.class);
//logger.addAppender();
BasicConfigurator config = new BasicConfigurator();
config.configure(loggerContext);
logger.info("log info ");
```



```xml
<configuration>

  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <!-- encoders are assigned the type
         ch.qos.logback.classic.encoder.PatternLayoutEncoder by default -->
    <encoder>
      <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
    </encoder>
  </appender>

  <root level="debug">
    <appender-ref ref="STDOUT" />
  </root>
</configuration>
```





additivity = false    不传播父logger

## 日志框架 – Log4j2

介绍
​	Log4j2 同样也是 Log4j 的替代者，在性能方面提升非常显著。
重要提升
执行速度更快，内存占用更小
避免锁
自动重载配置文件
高级过滤
插件架构

解读：https://logging.apache.org/log4j/2.x/


## 统一日志 API - slf4j

> 日志框架无论 Log4j 还是 Logback，虽然它们功能完备，但是各自API 相互独立，并且各自为政。当应用系统在团队协作开发时，由于工程人员可能有所偏好，因此，可能导致一套系统可能同时出现多套日志框架。
>
> 

>
> 其次，最流行的日志框架基本上基于实现类编程，而非接口编程。因此，暴露一些无关紧要的细节给用户，这种耦合性是没有必要的。

>
>

>   诸如此类的原因，开源社区提供统一日志的API框架，最为流行的是：

+ Apache commons-logging
  - 适配 log4j 和 Java Logging

+ slf4j
  - 适配 Log4j、Java Logging 和 Logback

* API：
  - slf4j-api
    - LoggerFactory.getLogger
    - StaticLoggerBinder
  - log4j-over-slf4j
  - jcl-over-slf4j
  - jul-to-slf4j

## Spring Boot 日志



* 401 问题

  springboot 1.5+  关闭了 Endpoint 管理，需要开启 

  ManagementServerProperties


配置文件 修改 level 级别

从配置入手

LoggingApplicationListener

LoggingSystem



initialize

\#cast()

### 题外

Reflection
