## Nacos 配置 + 服务发现

Spring Cloud Alibaba Nacos Discovery ----- Eureka
Alibaba Nacos Config  -- 分布式配置
Nacos Spring Stack -- 超越外部化配置

refresh的粒度

日志问题

4大特性

加载时机  替换自己实现

模块

数据库

maven的插件

### 使用

* Nacos 服务器
  https://github.com/alibaba/nacos/releases
* 命令行启动
  sh startup.sh -m standalone
* org.springframework.cloud
  spring-cloud-starter-alibaba-nacos-discovery
* application.properties
  spring.cloud.nacos.discovery.server-addr=127.0.0.1:8848
* @EnableDiscoveryClient
  SpringApplication.run
* 已注册实例
  http://127.0.0.1:8848/nacos/v1/ns/instances?serviceName=



激活注册 @EnableDiscoveryClient

服务注册ServiceRegistry、Registration

负载均衡 Ribbon Server、ServerList

Actuator
HealthIndicator、@Endpoint

![nacosspringstack.png](http://ww1.sinaimg.cn/large/006xzusPgy1gbdtcaisn0j30yo0h0wn9.jpg)

Spring Cloud Context 抽象 - 配置相关

- Spring Environment

EnvironmentManager  ---- EnvironmentRepository

- Bean 动态刷新

@RefreshScope

- Spring Boot Actuator

EnvironmentEndpoint
WritableEnvironmentEndpoint
RefreshEndpoint

- Spring Cloud 事件

EnvironmentChangeEvent
RefreshEvent

- Spring 应用上下文

RefreshScope
ContextRefresher
scope - refresh

### 官方文档

##  Sentinel  ------- Hytrix

### Spring Cloud 服务熔断现状

熔断模式

服务超时（Timeout）
信号量（Semaphore）



编程模型

面向接口（Interface）
面向注解（Annotation）



策略规则

自定义实现
注解配置
多数据源适配



控制台

Hystrix Dashboard



生态整合

JVM 进程服务
RPC 服务
Spring Cloud Stack 等

![1580536444706.png](http://ww1.sinaimg.cn/large/006xzusPgy1gbguyjklylj30ox0gnn34.jpg)

Spring Cloud Hystrix 会切换线程池
ThreadLocal
Spring 事务
异常

### Spring Cloud Alibaba Sentinel
下载Dashboard

Alibaba Sentinel Dashboard
https://github.com/alibaba/Sentinel/releases/download/0.2.0/sentinel-dashboard.jar



Dashboard 命令行启动参数
java -Dserver.port=8080 -Dcsp.sentinel.dashboard.server=localhost:8080 -
Dproject.name=sentinel-dashboard -jar sentinel-dashboard.jar



Spring Cloud 应用 + Alibaba Sentinel（Client）
增加依赖 

org.springframework.cloud
spring-cloud-starter-alibaba-sentinel

增加外部化配置

application.properties
spring.cloud.sentinel.transport.dashboard=localhost:8080

配置应用名称 spring.application.name=sentinel-example

配置限流规则

Alibaba Sentinel Dashboard
http://localhost:8080/#/dashboard/flow/sentinel-dashboard

```<dependency>
 <dependency>
 <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-alibaba-sentinel</artifactId>
</dependency>```
```

AQS
J.U.C
Reactor
RxJava

阿里云新产品 AHAS（公测中）
https://www.aliyun.com/product/ahas?spm=5176.8142029.developerService.25.e9396d3ejVU79p








## Spring Cloud Stream Binder Alibaba

Spring Cloud Stream Binder RocketMQ

RocketMQ 服务器 https://github.com/apache/rocketmq

命令行启动

$ sh mqnamesrv

$ sh mqbroker -n 127.0.0.1:9876 autoCreateTopicEnable=true



org.springframework.cloud

spring-cloud-stream-binder-rocketmq



application.properties

spring.cloud.stream.rocketmq.binder.namesrv-addr=127.0.0.1:9876



@EnableBinding SpringApplication.run

@Source @Sink 底层：



## Spring Cloud Stream Bus Alibaba

技术基础


Spring Environment 抽象

Spring Event/Listener 机制

Spring Annotation-Driven

Spring Boot Externalized Configuration

Spring Boot Actuator

Spring Cloud Context 抽象

Spring Cloud Stream 抽象



自定义事件 RemoteApplicationEvent
激活自定义事件  @RemoteApplicationEventScan



Spring Cloud Alibaba RPC




配置中心：Config Server

注册中心：Eureka、consul、Nacos

服务短路：Hystrix、Sentinel

负载均衡：Ribbon

服务发现：Feign、Nacos

路由网关：Zuul

流式处理：Stream

服务监控跟踪：Sleuth

服务整合：Zipkin




