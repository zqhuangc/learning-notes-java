# Spring Cloud Zuul



## Zuul 基本使用

Nginx + Lua
Lua：控制规则（A/B Test）

Spring Cloud 学习技巧：

定位应用：Feign、Config Server、Ribbon、Zuul、Eureka定位应用，配置方式是不同的（配置对应的类）

### 增加 @EnableZuulProxy

基本模式：`zuul.routes.${app-name} = /${app-url-prefix}/**`

```properties
## Zuul 代理应用
spring.application.name = zuul-proxy

## 服务端口
server.port = 6060

## 管理安全失效
management.security.enabled = false

### 配置 Zuul 路由原则
## 指定应用 user-service-provider
zuul.routes.user-service-provider = /user-service-provider/**
###### /*当前目录下的所有 /**当前目录及其子目录下所有
## 配置 ribbon
user-service-provider.ribbon.listOfServers = http://localhost:9090/

## http://localhost:8080/user-service/* => http://localhost:9090/*
```



##整合 Ribbon

zuul -> person-service 

person-service 的 app-url-prefix：/person-service/

#### 配置

```properties
## 配置 ribbon
user-service-provider.ribbon.listOfServers = http://localhost:9090/
```



## 整合 Eureka

1. 引入依赖
2. 激活服务注册、发现客户端 EnableDiscoveryClient
3. 配置服务URL，客户端注册、zuul 路由配置



## 整合 Hystrix

1. 服务提供方，激活Hystrix @EnableHystrix 
2. 配置 Hystrix 规则



## 整合 Feign

1. 服务消费端，注册到 Eureka
   调用链路：spring-cloud-zuul(7070)  -> client(8080) -> service(9090)

2. zuul 路由和 Ribbon 配置修改

   ```properties
   ### 配置 Zuul 路由原则
   ## 指定应用 user-client
   zuul.routes.user-client= /user-client/**
   
   ```

   ```properties
   ## 配置 ribbon
   user-client-provider.ribbon.listOfServers = http://localhost:8080/
   ```






## 整合 Config Server

动态路由，动态配置

端口信息

> zuul-proxy : 7070
>
> config-server : 10000
>
> user-service-client: 8080
>
> user-service-provider : 9090
>
> eureka-server : 12345

1. 配置服务器：spring cloud config server

2. 为 spring-cloud-zuul 增加配置文件 
   profile：zuul.properties、zuul-test.properties、zuul-prod.properties

   > 注意：zuul 不为应用名，原因：避免应用名称中-

   * 创建本地仓库

   ```console
   $ git init
   $ git add *.properties
   $ git commit -m "test"
   $ git push
   ```

3. 增加 Eureka 客户端依赖，注册到 Eureka 服务器

4. 测试 /zuul/prod，/zuul/test，/zuul/default 是否实现

### Zuul 应用 配置为 config 客户端

bootstrap.properties

```properties
### bootstrap.properties
### bootstrap 上下文配置
### 配置服务器 URI  不使用 Eureka 的方式
###spring.cloud.config.uri = http://localhost:9090/
### 配置客户端应用名称？？前缀？{application}
spring.cloud.config.name = zuul
### profile 是激活的配置{profile}
spring.cloud.config.profile = prod
### label 在 Git 中指的是分支的名称{label}
spring.cloud.config.label = master
### 激活 discovery 连接配置项的方式
spring.cloud.config.discovery.enabled = true
### 配置 config server 应用名称
spring.cloud.config.discovery.serviceId = spring-cloud-config-server
### 由于配置优先问题，注册 Eureka 服务器应在这里
# ....
```

**push notification**  ` git push`自动刷新客户端获取的配置

## 问答和笔记

1. 不使用 ribbon，直接去 eureka server 找服务，使用 ribbon，从 listOfServers 中找一个，

2. zuul 更多的作为业务网关？很多企业内部使用 zuul 作为服务网关

3. zuul 中 `RequestContext `（继承 ConcurrentHashMap）已经存在 ThreadLocal 中了，为什么还要用 ConcurrentHashMap？
   答：`ThreadLocal `只能管理当前线程，不能够管理子线程，子线程管理需要是使用 `InheritableThreadLocal `。`ConcurrentHahMap` 实现一下，如果上下文处于多线程线程的环境，比如传递到子线程。假如多个子线程操作 RequestContext ，若其本身不能保证同步就会出问题。

   比如：T1 在管理 RequestContext ，但是 T1 又创建了多个线程（t1,t2），这个时候，把上下文传递到了子线程 t1,t2.

   Java 的进程所对应线程 main（group：main），main线程是所有子线程的福线程，main 线程 创建 T1，T2，T1 创建 t1，t2

4. ZuulServlet 已经管理了 RequestContext 的生命周期，为什么 ContextLifecycleFilter 还要再做一遍？

   答：都有 unset() 处理，`ZuulServletFilter`也有，`RequestContext `是任何 Servlet 和 Filter 都能处理，那么为了防止不正确的关闭，那么`ContextLifecycleFilter `相当于兜底操作，就是防止 ThreadLoacal 没有被 remove 掉。
   `ThreadLocal `对应一个 Thread，是不是意味着 Thread 处理完了，那么 `ThreadLocal`也随之GC？

   所有 Servlet 均采用线程池，因此，不清空的话，可能会出现意外的情况。除非，每次都异常！（这也要依赖于线程池的实现）



http   头部敏感信息

nginx 过滤   _xxx 头部

# Spring Cloud 服务网关

## 核心概念

### 网关
网关是程序或者系统之间的连接节点，扮演着程序或系统之间的⻔门户，允许它们之间通过通讯协议交换信息，它们可能是同构和异构的系统。
• 例例如
• REST API 网关
• WebServices 网关



### 使用场景
• 监控（Monitoring）
• 测试（Testing）
• 动态路路由（Dynamic Routing）
• 服务整合（Service Integration）
• 负荷减配（Load Shedding）
• 安全（Security）
• 静态资源处理理（Static Resources
handling）
• 活跃流量量管理理（Active traffic
management）



### 数据来源
• 服务发现
• 服务注册
• 通讯方式
• 协议：二进制、本文
• 方式：同步、异步



### 服务接口
• 平台无关
• XML、JSON、HTML
• 平台相关
• IDL、RMI、Hession

### Spring Cloud Zuul

#### 依赖
•org.springframework.cloud:spring-cloud-starter-zuul
• 激活
•@EnableZuulProxy
• 配置
•zuul.*

#### 路由设置
• 配置模式
• 服务-映射：zuul.routes.${service-id} = ${url-pattern}
• 路径模式
• 当前层级匹配：/*
• 递归层级匹配：/**



#### HTTP 客户端
• HttpClient（默认）
• 装配：HttpClientRibbonConfiguration
• OkHttp（条件）
• 激活配置：ribbon.okhttp.enabled = true
• 装配：OkHttpRibbonConfiguration



#### 端点（Endpoint）
• 实现：RoutesEndpoint
• 路路径：/routes
• 过滤器器：ZuulFilter

RouteLocator

## Spring Cloud Zuul



### 增加依赖

```xml
        <!-- 依赖 Spring Cloud Netflix Zuul -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-zuul</artifactId>
        </dependency>
```



### 创建 Zuul 代理应用



```java
package com.segumentfault.spring.cloud.lesson11.zuul.proxy;

import org.springframework.boot.SpringApplication;
import org.springframework.cloud.client.SpringCloudApplication;
import org.springframework.cloud.netflix.zuul.EnableZuulProxy;

/**
 * Zuul 代理引导类
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 1.0.0
 */
@EnableZuulProxy
@SpringCloudApplication
public class ZuulProxyApplication {

    public static void main(String[] args) {
        SpringApplication.run(ZuulProxyApplication.class, args);
    }

}
```



### 配置 Zuul 应用



`application.properties`

```properties
## Zuul 代理应用
spring.application.name = zuul-proxy

## 服务端口
server.port = 6060

## 管理安全失效
management.security.enabled = false
```



### 配置 zuul 路由规则

`application.properties`

```properties
## 指定 user-service-provider
zuul.routes.user-service-provider = /user-service/**

## 配置 ribbon
user-service-provider.ribbon.listOfServers = http://localhost:9090/

## http://localhost:8080/user-service/* => http://localhost:9090/*
```





### 配合 HTTP 客户端

> 注意：实际配置 Ribbon 底层 HTTP 调用客户端，并非 zuul 独享此功能



#### 默认客户端：HttpClient



装配类：`HttpClientRibbonConfiguration`



#### 配置客户端：OkHttpClient



装配类：`OkHttpRibbonConfiguration`

激活配置：`ribbon.okhttp.enabled=true`



## Spring Cloud 整合

### 服务端口信息

> 端口信息
>
> ​	zuul-proxy : 6060
>
> ​	config-server : 7070
>
> ​        user-service-client: 8080
>
> ​        user-service-provider : 9090
>
> ​        eureka-server : 10000



### 服务依赖关系



- eureka-server

  - user-service-provider (1) （数据源）

  - config-server (2) （配置源）

    - user-service-client
      - zuul-proxy

    ​

### config-server 配置 zuul-proxy 信息



`configs/zuul-config.properties`

```properties
## Zuul Proxy 配置内容

## 指定 user-service-provider
zuul.routes.user-service-provider = /user-service/**
## 指定 user-service-client
zuul.routes.user-service-client = /user-client/**
```



### zuul-proxy 作为配置客户端



#### 增加 config client 依赖

```xml
        <!-- 依赖 Spring Cloud Config Client -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
```



#### 配置 config client 信息



`bootstrap.properties`

```properties
## Zuul 代理应用
spring.application.name = zuul-proxy

## 配置客户端应用关联的应用
## spring.cloud.config.name 是可选的
## 如果没有配置，采用 ${spring.application.name}
spring.cloud.config.name = zuul-config
## 关联 profile
spring.cloud.config.profile = default
## 关联 label
spring.cloud.config.label = master
## 激活 Config Server 服务发现
spring.cloud.config.discovery.enabled = true
## Config Server 服务器应用名称
spring.cloud.config.discovery.serviceId = config-server
## Spring Cloud Eureka 客户端 注册到 Eureka 服务器
eureka.client.serviceUrl.defaultZone = http://localhost:10000/eureka
```



### zuul-proxy 激活服务发现



#### 增加 Eureka Client 依赖

```xml
        <!-- 依赖 Spring Cloud Netflix Eureka -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka</artifactId>
        </dependency>
```





ZuulFilter 调用链



`ZuulFilter#run()` <- `ZuulFilter#runFilter()` <- `FilterProcessor#runFilters`



`FilterProcessor#preRoute()`

`FilterProcessor#route()`

`FilterProcessor#postRoute()`

`ZuulServletFilter` `ZuulServlet`

```java
            RequestContext context = RequestContext.getCurrentContext();
            context.setZuulEngineRan();

            try {
                preRoute();
            } catch (ZuulException e) {
                error(e);
                postRoute();
                return;
            }
            try {
                route();
            } catch (ZuulException e) {
                error(e);
                postRoute();
                return;
            }
            try {
                postRoute();
            } catch (ZuulException e) {
                error(e);
                return;
            }
```



## Zuul 自动装配



`ZuulServletFilter` 适用范围更大，可以拦截所有的`Servlet`，包括 `ZuulServlet`



`ZuulServlet` 会有 URL 匹配的模式，url-pattern



Zuul 有两种的激活模式：

- `@EnableZuulProxy`

  导入`ZuulProxyMarkerConfiguration`，随后生成一个`ZuulProxyMarkerConfiguration.Marker()` Bean，这个Bean 作为`ZuulProxyAutoConfiguration` 的装配前置条件。

  请注意：`ZuulProxyMarkerConfiguration` 扩展了 `ZuulServerAutoConfiguration`，所以 `ZuulServlet` 和`ZuulController`会被自动装配



  `ZuulController` 有 `DispatcherServlet` 来在控制,它的映射地址是："/*"，

  `DispatcherServlet` 中注册了一个`ZuulHandlerMapping` ，它控制映射到`ZuulController`，可以参考`ZuulServerAutoConfiguration`:

  ```java
  	@Bean
  	public ZuulController zuulController() {
  		return new ZuulController();
  	}
  
  	@Bean
  	public ZuulHandlerMapping zuulHandlerMapping(RouteLocator routes) {
  		ZuulHandlerMapping mapping = new ZuulHandlerMapping(routes, zuulController());
  		mapping.setErrorController(this.errorController);
  		return mapping;
  	}
  ```

  通过源码分析，`ZuulController`  将请求委派给`ZuulServlet`，所以所有的`ZuulFilter` 实例都会被执行。

  > 因此，访问 http://localhost:6060/user-service-client/user/find/all ，实际将请求递交给 DispatcherServlet
  >
  > 发送请求"/user-service-client/user/find/all" 

  - `DispatcherServlet`
    - `ZuulHandlerMapping`
      - `ZuulController`
        - `ZuulServlet`
          - `RibbonRoutingFilter`


- `@EnableZuulServer`

  导入`ZuulServerMarkerConfiguration` ，随后生成一个 `ZuulServerMarkerConfiguration.Marker()` Bean ，主要用作引导装配`ZuulServerAutoConfiguration`



`ZuulServerAutoConfiguration`与 父类 `ZuulProxyAutoConfiguration` 区别：

父类`ZuulProxyAutoConfiguration` 提供了`RibbonRoutingFilter`



调用层次：

- `DispatcherServlet`
  - `ZuulHandlerMapping`
    - `ZuulController`
      - `ZuulServlet`
        - `ZuulFilter`





