# Spring Cloud Sleuth

微服务调用链路变长，监控存储要求更高，响应时间延长

## 理论基础

trace id

OpenTSDB（InfluenceDB）

MDC类



• Google Dapper
• 依赖
• spring-cloud-starter-sleuth
• 日志
• Logging MDC

Slf4jSpanLogger

**TraceFilter**   trace span id 请求头

maven  <optional></optional> 编译时依赖，不传递，兼容依赖者的版本，而不固定

###  Zipkin 服务器

• 依赖
• zipkin-server
• 激活
• @EnableZipServer



### Zipkin 客户端
• HTTP 收集方式
• 依赖
• spring-cloud-starter-zipkin
• 配置
• spring.zipkin.base-url = http://${zipkin.server.host}:${zipkin.server.port}



### Stream 收集方式

#### Zipkin 服务端（springboot 2.0以上不再支持自定义的zipkin服务器创建，需要自己下载一个以jar形式打开）

• 依赖
•spring-cloud-sleuth-zipkin-stream
•spring-cloud-binder-rabbit
• 激活
•@EnableZipkinStreamServer



#### Zipkin 客户端

• 依赖
• spring-cloud-sleuth-stream
• spring-cloud-binder-rabbit
• 移除 HTTP 收集方式配置



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
> **注意**：把前面 HTTP 上报 URL 配置注释或去掉

### 日志收集（文件系统）

sleuth 埋点  zipkin 收集

## 问答

1. 只有第一个被访问的应用的 pom 引用了 spring-cloud-starter-sleuth 包么？后面 span ID 是怎么生成的？上报是汇集到一起上报么？
   答：每个应用都需要依赖 spring-cloud-starter-sleuth 。spanid 是当前应用生成的，Trace ID 是由上个应用传递过来（Spring Cloud 通过请求头 Headers）

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



# Spring Cloud 分布式应用跟踪



SpanID : 阶段性 ID，比如一次 RPC 有可能多 Span



TraceID：一次 RPC 只有一个 TraceID





## 整合 Spring Cloud Sleuth



### 增加依赖

```xml
        <!-- 整合 Spring Cloud Sleuth -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-sleuth</artifactId>
        </dependency>
```





## Zipkin 整合



### 新增 Zipkin 服务器



#### 增加 Maven 依赖



```xml
        <!-- Zipkin Server 整合 -->
        <dependency>
            <groupId>io.zipkin.java</groupId>
            <artifactId>zipkin-server</artifactId>
        </dependency>

        <!-- Zipkin Server UI 整合 -->
        <dependency>
            <groupId>io.zipkin.java</groupId>
            <artifactId>zipkin-autoconfigure-ui</artifactId>
        </dependency>
```



#### 激活 Zipkin 服务器



HTTP 方式采集

```java
package com.segumentfault.spring.cloud.lesson15;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import zipkin.server.EnableZipkinServer;

/**
 * Zipkin 服务器应用
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
@SpringBootApplication
@EnableZipkinServer
public class ZipkinServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ZipkinServerApplication.class, args);
    }
}
```



#### 配置 Zipkin 服务器

```properties
## 应用元信息
## Zipkin 服务器应用名称
spring.application.name = zipkin-server

## Zipkin 服务器服务端口
server.port = 20000

## 管理端口安全失效
management.security.enabled = false
```



### 整合 ZipKin 客户端



#### 改造 `user-service-client`



HTTP 方式上报



##### 增加 Maven 依赖

```xml
        <!-- 整合 Spring Cloud Zipkin -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zipkin</artifactId>
        </dependency>
```





##### 配置：连接 zipkin 服务器

```properties
## Zipkin 配置
### 配置 Zipkin 服务器
zipkin.server.host = localhost
zipkin.server.port = 20000
spring.zipkin.base-url = http://${zipkin.server.host}:${zipkin.server.port}
```



#### 改造 `user-service-provider`



##### 增加 Maven 依赖

```xml
        <!-- 整合 Spring Cloud Zipkin -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zipkin</artifactId>
        </dependency>
```





##### 配置：连接 zipkin 服务器

```properties
## Zipkin 配置
### 配置 Zipkin 服务器
zipkin.server.host = localhost
zipkin.server.port = 20000
spring.zipkin.base-url = http://${zipkin.server.host}:${zipkin.server.port}
```



### 改造Zipkin 服务器：使用 Stream 方式订阅



#### 增加 Maven 依赖

```xml
        <!-- zipkin 整合 Spring Cloud Sleuth Stream  -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-sleuth-zipkin-stream</artifactId>
        </dependency>

        <!-- 整合 Spring Cloud Stream Binder Rabbit MQ -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
        </dependency>
```



#### 替换激活注解：`@EnableZipkinStreamServer`



Stream 方式采集

```java
package com.segumentfault.spring.cloud.lesson15.zipkin.server;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.sleuth.zipkin.stream.EnableZipkinStreamServer;

/**
 * Zipkin 服务器应用
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
@SpringBootApplication
//@EnableZipkinServer
@EnableZipkinStreamServer
public class ZipkinServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ZipkinServerApplication.class, args);
    }
}
```





### 改造 `user-service-client`



#### 增加 Maven 依赖

Stream 方式上报

```xml
	    <!-- 整合 Spring Cloud Sleuth -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-sleuth</artifactId>
        </dependency> 
	    <!-- 整合 Spring Cloud Sleuth Stream -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-sleuth-stream</artifactId>
        </dependency>
	    <!-- 整合 Spring Cloud Stream Binder Rabbit MQ -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
        </dependency>
```



> 注意应移除 `spring-cloud-starter-zipkin` 依赖



### 改造 `user-service-provider`



#### 增加 Maven 依赖

Stream 方式上报

```xml
	    <!-- 整合 Spring Cloud Sleuth -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-sleuth</artifactId>
        </dependency> 
	    <!-- 整合 Spring Cloud Sleuth Stream -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-sleuth-stream</artifactId>
        </dependency>
	    <!-- 整合 Spring Cloud Stream Binder Rabbit MQ -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
        </dependency>
```



> 注意应移除 `spring-cloud-starter-zipkin` 依赖