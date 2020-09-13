### 新一代日志框架 - [logback](https://logback.qos.ch/manual/index.html)

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

#### 统一日志 API - slf4j

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

#### Spring Boot 日志



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