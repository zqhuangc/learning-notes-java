# Spring Cloud Sleuth

微服务调用链路变长，监控存储要求更高，响应时间延长

## 基础

trace id

OpenTSDB（InfluenceDB）

MDC类

### [Google Dapper]()



## Spring Cloud Sleuth 整合

1. 引入 Maven 依赖
2. 日志会发生调整（多了 traceId）

3. 引入 `spring-cloud-starter-sleuth` 会调整当前日志系统（slf4j）的 [MDC](https://www.slf4j.org/manual.html) ：`Slf4jSpanLogger#logContinuedSpan`（Span）



#### 整体流程（客户端）

`spring-cloud-starter-sleuth`会自动装配一个名为 `TraceFilter` 组件（在 Spring MVC  `DispatcherServlet` 之前），它会增加一些 slf4j MDC



## [Zipkin](zipkin.io) 整合

1.  引入 Maven 依赖

   ```xml
   <!-- Zipkin server-->
   <!-- Zipkin 服务器 UI 控制器 -->
   ```

2. 创建 spring cloud zipkin 服务器，默认端口 9411
   激活@EnableZipkinServer



### HTTP 收集（HTTP 调用）

#### 简单整合 `spring-cloud-sleuth`

#### 增加 Maven 依赖

```xml
<!-- Zipkin 客户端-->
```

#### 配置

```properties
## ZipKin 服务器配置
zipkin.server.host = localhost
zipkin.server.port = 23456
## 增加 ZipKin 服务器地址
spring.zipkin.base-url = \
http://${zipkin.server.host}:${zipkin.server.port}/
```



#### spring cloud 服务大整合

> * 端口信息
>
> > spring-cloud-zuul 端口：7070
> >
> > user-client 端口：8080
> >
> > user-service端口：9090
> >
> > Eureka Server 端口：12345
> >
> > Zipkin Server 端口：23456
>
> * 服务启动
>
> 1. Zipkin Server
> 2. Eureka Server
> 3. spring-cloud-config-server
> 4. user-service
> 5. user-client
> 6. spring-cloud-zuul

​	

##### spring-cloud-sleuth 改造

* 连接 Eureka 服务器
* 增加 Eureka 客户端依赖
* 激活 Eureka 客户端
* 调整配置
* 调整代码连接 zuul
* zuul 上报到 zipkin服务器
* user-client 上报到 zipkin服务器
* user-service 上报到 zipkin服务器

### Spring Cloud Stream 收集（消息）

> spring-cloud-sleuth-zipkin-stream
>
> 激活 ：@EnableZipkinServer 切换为 @EnableZipkinStreamServer
>
> 使用 Kafka 作为 Stream 服务器
>
> 启动 zookeeper、kafka 服务器
>
> 启动 spring-cloud-zipkin-server
>
> **注意**：把前面 HTTP 上报 URL 配置注释

### 日志收集（文件系统）



## 问答

1. 只有第一个被访问的应用的 pom 引用了 spring-cloud-starter-sleuth 包么？后面 span ID 是怎么生成的？上报是汇集到一起上报么？
   答：每个应用都需要依赖 spring-cloud-starter-sleuth 。sapnid 是当前应用生成的，Trace ID 是由上个应用传递过来（Spring Cloud 通过请求头 Headers）

2. sleuth 配合 zipkin 主要解决的是分布式系统下的请求的链路跟踪？
   答：可以这么理解，问题排查、性能、调用链路跟踪

3. sleuth 与 eureka 没关联吧？

   答：没有直接关系，Sleuth 可以利用 Eureka 做服务发现

4. 生产上类 Sleuth 的 log 日志应该看哪些资料？opentsdb，也有调用链路等相关信息么？
   答：OpenTsdb 主要是存储 JSON 格式，存放 Metrics 信息

5. 排查问题是根据 TranceId 来查找所有日志？
   答：可以通过 TraceId 查找调用链路，通过 Span Id 定位在什么环节

6. 整合显示是 sleuth 做的，zipkin 用来收集信息对吧？
   答：Zipkin 是一种整合方式

7. Map，JSON Path
