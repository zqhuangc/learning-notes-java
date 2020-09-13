# 2019.01.11「小马哥技术周报」- 第十六期《Apache Dubbo 微服务系列之 Dubbo Spring Cloud 实现》



## 预备知识

- Spring Cloud Commons 抽象
  - 服务注册和发现
    - 应用名称（服务名称）：`spring.aplication.name` 
    - 应用实例：`ServiceInstance`
      - 信息
        - 服务名：serviceId
        - 主机：host
        - 端口：port
        - 安全：secure（HTTPS / HTTP)
        - 元信息：`Map<String,String>`  metadata
      - `Registration`
    - 服务注册（到注册中心）
    - 服务发现（从注册中心）
    - 实现模块
      - 内建：`eureka`、`consul`、`zookeeper`
      - 孵化：`nacos`
  - Spring Cloud Environment（基于 Spring Framework Environment）
- Spring Cloud OpenFeign
  - 类型：客户端面向 Java 接口的 REST 框架
  - 协议：REST（HTTP）
  - 注解： `@FeignClient`
    - 服务接口绑定应用名称



## 场景分析



### 场景一：服务端（Dubbo + REST） 与客户端（Spring Cloud Feign）



Dubbo Spring Boot : 只会注册 Dubbo 服务接口相关的注册信息

Dubbo Spring Cloud：Dubbo Spring Boot  + 应用本身的信息



### 场景二：服务端（Dubbo + REST） 与客户端（Spring Cloud Feign底层使用 Dubbo 协议）



#### 代码分析猜测实现

- 示例代码 - 客户端接口声明

```java
    @FeignClient("spring-cloud-alibaba-dubbo")
    public interface EchoService {

        @GetMapping(value = "/echo")
        String echo(@RequestParam("message") String message);
    }
```

- 示例代码 - 客户端接口使用

```java
        @Autowired
        private EchoService echoService;

        @GetMapping("/call/echo")
        public String echo(@RequestParam("message") String message) {
            return echoService.echo(message);
        }
```



`EchoService` 是一个Bean（实现对象），猜测它是动态实现~~或者字节码提升~~

- 特征注解： `@FeignClient`
- 激活注解：`@EnableFeignClients `
- 协议：HTTP（REST）
- 路由规则：
  - URI：`/echo`
  - 请求参数：与服务提供方保持一致
  - 应用名称：`spring-cloud-alibaba-dubbo`
    - 服务实例：HOST+PORT
    - URL： http://${HOST}:${PORT}/${URI}



- 服务提供方
  - `DefaultEchoService`
    - Dubbo 唯一接口名称：`org.springframework.cloud.alibaba.dubbo.service.EchoService:1.0.0`
    - 接口
      - REST 接口
      - Dubbo Java 接口



- 疑问
  - 如何注册 REST 和 Dubbo 接口作为特殊服务到注册中心？
    - Dubbo Registry SPI + Spring Cloud Commons 抽象 API
      - Dubbo Registry  : `RegistryFactory` 以及 `Registry`
      - 注册实例数据：`Registration`（`ServiceInstance`）
      - 注册中心接口：`ServiceRegistry`
  - 如何获取当前应用暴露的 Dubbo 多协议的元信息











# 2019.01.18「小马哥技术周报」- 第十七期《Apache Dubbo 微服务系列之 Dubbo Spring Cloud 实现（下）》



### Feign Dubbo 服务接口匹配分析

服务端应用名称：`spring-cloud-alibaba-dubbo`

Dubbo 服务

- 接口：`org.springframework.cloud.alibaba.dubbo.service.EchoService`
- 版本：1.0.0
- 协议：dubbo 以及 rest

```java
@Service(version = "1.0.0", protocol = {"dubbo", "rest"})
@RestController
@Path("/")
public class DefaultEchoService implements EchoService {
    @Override
    @GetMapping(value = "/echo")
    @Path("/echo")
    @GET
    public String echo(@RequestParam @QueryParam("message") String message) {
        return RpcContext.getContext().getUrl() + " [echo] : " + message;
    }
}
```





客户端应用应用：spring-cloud-alibaba-dubbo

Feign 接口：

```java
@FeignClient("spring-cloud-alibaba-dubbo")
public interface FeignEchoService {

    @GetMapping(value = "/echo")
    String echo(@RequestParam("message") String message);
}
```

指定的 Spring Cloud 应用名称：`spring-cloud-alibaba-dubbo`

REST URI：`/echo`

Params ： `message`





`spring-cloud-alibaba-dubbo` -> 

配置信息 ID： `spring-cloud-alibaba-dubbo-rest-metadata.json`

内容：

```json
[
  {
    "name": "providers:org.springframework.cloud.alibaba.dubbo.service.EchoService:1.0.0",
    "meta": [
      {
        "configKey": "EchoService#echo(String)",
        "method": "GET",
        "url": "/echo?message={message}",
        "headers": {},
        "indexToName": {
          "0": [
            "message"
          ]
        }
      },
      ...
    ]
  }
]
```



Feign.Builder 

- 默认实现：`feign.Feign.Builder`
- Hystrix: `feign.hystrix.HystrixFeign.Builder`
- Sentinel:`org.springframework.cloud.alibaba.sentinel.feign.SentinelFeign`



`feign.Feign.Builder` -> Wrapper -> build()





`FactoryBean<FeignEchoService>` getType() == `FeignEchoService`







# 2019.03.29 「小马哥技术周报」- 第二十二期《Apache Dubbo 微服务系列之 ALL IN ONE》

## 项目地址

https://github.com/spring-cloud-incubator/spring-cloud-alibaba



## 场景解读



Spring Cloud 是 Cloud Native Java 技术栈的解决方案（框架）



## 注册中心

- Eureka
- Zookeeper
- Consul
- Nacos
- X

它们都是通过 Starter 来引导组件，如果同时存在

- 同时注册
- 唯一注册



Spring Cloud Commons 在注册中心设计原则是：互斥的 `ServiceRegistry` Bean 实例





dubbo://169.254.174.223:20881/org.springframework.cloud.alibaba.dubbo.service.RestService?anyhost=true&application=spring-cloud-alibaba-dubbo-web-provider&bean.name=providers:dubbo:org.springframework.cloud.alibaba.dubbo.service.RestService:1.0.0&default.deprecated=false&default.dynamic=false&default.register=true&deprecated=false&dubbo=2.0.2&dynamic=false&generic=false&interface=org.springframework.cloud.alibaba.dubbo.service.RestService&methods=headers,requestBodyUser,pathVariables,form,param,requestBodyMap,params&pid=12836&register=true&release=2.7.1&revision=1.0.0&side=provider&timestamp=1553863544857&version=1.0.0

application=spring-cloud-alibaba-dubbo-web-provider  -> spring.application.name 

side=provider

```java
[ "dubbo://169.254.174.223:20881/org.springframework.cloud.alibaba.dubbo.service.RestService?anyhost=true&application=spring-cloud-alibaba-dubbo-web-provider&bean.name=providers:dubbo:org.springframework.cloud.alibaba.dubbo.service.RestService:1.0.0&default.deprecated=false&default.dynamic=false&default.register=true&deprecated=false&dubbo=2.0.2&dynamic=false&generic=false&interface=org.springframework.cloud.alibaba.dubbo.service.RestService&methods=headers,requestBodyUser,pathVariables,form,param,requestBodyMap,params&pid=12836&register=true&release=2.7.1&revision=1.0.0&side=provider&timestamp=1553863544857&version=1.0.0", "dubbo://169.254.174.223:20881/org.springframework.cloud.alibaba.dubbo.service.DubboMetadataConfigService?anyhost=true&application=spring-cloud-alibaba-dubbo-web-provider&default.deprecated=false&default.dynamic=false&default.register=true&deprecated=false&dubbo=2.0.2&dynamic=false&generic=false&interface=org.springframework.cloud.alibaba.dubbo.service.DubboMetadataConfigService&methods=getServiceRestMetadata&pid=12836&register=true&release=2.7.1&revision=spring-cloud-alibaba-dubbo-web-provider&side=provider&timestamp=1553863547084&version=spring-cloud-alibaba-dubbo-web-provider" ]
```



`ServiceBean` -> Service Interface + Version + Group 

-> N URLs -> dubbo,rest

-> 1 Repository -> N URLs x ( M Services)



`ServiceInstance#getMetadata()`

Nacos  OK

Eureka OK

Consul 可以修改，但是无法同步状态（SNAPSHOT）

Zookeeper 当 payload 为 `null`，返回只读 Map，不为 `null` 时，返回有状态的可修改 Map









## 网关

- Spring Cloud Netflix Zuul
  - Servlet API
- Spring Cloud Gateway
  - Reactor + Netty



### Spring Cloud 服务调用（Service-To-Service calls）

客户端（REST）：Spring Cloud REST（HTTP）实现方式：

- `@LoadBalanced` RestTemplate
  - 面向 URL：${serviceName} + ${serviceURI}
- Spring Cloud OpenFeign : @FeignClient + Java Interfaces
  - ${serviceName} + 接口方法（绑定 ${serviceURI}）
  - Feign
    - 注解方面
      - feign：Feign 自定义注解
      - feign-jaxrs：Java REST 标准 API：JAX-RS 1 和 2 注解
      - Spring cloud openfeign：Spring Cloud OpenFeign -> Spring Web 注解 @RequestMapping
    - REST 支持的范围
      - 请求参数（Spring MVC：@RequestParam，JAX-RS：@QueryParam）
      - 请求头（Cookie 不要支持，Spring MVC：@CookieValue，JAX-RS：@CookieParam）
      - 请求主体（Spring MVC：@RequestBody，JAX-RS：@Bean）
      - 变量路径（Spring MVC：@PathVariable，JAX-RS：@PathParam）
    - REST 高级特性
      - 内容协商（Content Negotiation）
        - Accept 请求头：accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8
        - Content-Type 相应：content-type: text/html; charset=utf-8

> ${serviceName} : DNS ，使用应用为维度的 IP + Port 列表



## Spring Cloud Open Feign 设计与实现

`@EnableFeignClients`( Enable 模块驱动)

`@FeignClient`

- Java 动态代理
- Feign Core 库实现 HTTP 和 对象之间的转化
- Spring FactoryBean 以及 HttpMessageConverter



```java
    @FeignClient("spring-cloud-alibaba-dubbo")
    public interface FeignRestService {

        @GetMapping(value = "/param")
        String param(@RequestParam("param") String param);
        ...
    }
```



```json

```

http://spring-cloud-alibaba-dubbo/param?param=abc



Client： client

`@Reference(version="dubbo-spring-cloud-server")`

private DubboMetadataConfigService dubboMetadataConfigService;





Server: dubbo-spring-cloud-server



### `@LoadBalanced` RestTemplate 实现

http://spring-cloud-alibaba-dubbo/param?param=abc

http://${spring.application.name}/${uri}?${queryString}



`RestTemplate` 关联 N 个 `ClientHttpRequestInterceptor`，其中 `org.springframework.cloud.client.loadbalancer.LoadBalancerInterceptor` 即 `@LoadBalanced` RestTemplate 实现



HTTP请求 通过 Spring Cloud 负载均衡（数据源：注册中心）执行 HTTP 操作，将 HTTP 相应反序列化成需要的对象。



- 服务端
  - spring-cloud-alibaba-dubbo
  - Dubbo Service ：org.springframework.cloud.alibaba.dubbo.service.StandardRestService
    - /param
  - Spring MVC @RestController
    - /abc
- 客户端
  - Spring Cloud Open Feign Client
    - /param： 可以走 Dubbo 协议
    - /abc：不能走 Dubbo 协议
  - @LoadBalanced RestTemplate
    - /abc : HTTP 协议





服务端（REST）：Servlet 技术栈和 Reactive 技术栈

- Servlet：Spring WebMVC
- Reactive：
  - Spring WebFlux
  - Spring Web Reactive（Servlet 3.1）

#### 服务注册与发现

Spring Cloud Commons 抽象 API



### Dubbo 服务调用

#### 客户端 

Dubbo 二进制、本文协议

- 面向接口设计客户端调用
  - dubbo
  - REST
- ~~RestTemplate~~
- ~~OpenFeign~~

#### 服务端



#### 服务注册与发现

Dubbo Registry SPI



### Dubbo + Spring Cloud

客户端 Dubbo 二进制、本文协议

- 面向接口设计客户端调用
  - dubbo
  - REST
- RestTemplate
- OpenFeign



REST 协议暴露：

- JAX-RS（标准 Java REST API）
- Spring MVC 注解（Spring WebMVC 和 Spring WebFlux 公用，事实的标准）



#### 服务注册与发现

Dubbo Registry SPI + Spring Cloud Commons 抽象 API



- Dubbo Registry SPI
  - `org.apache.dubbo.registry.RegistryFactory`
  - `org.apache.dubbo.registry.Registry`
- Spring Cloud Commons API
  - Registration
    - DubboRegistration
    - NacosRegistration
    - EurekaRegistration
    - ZookeeperRegistration
    - ConsulRegistration



场景一： 传统 Dubbo  注册到 Nacos +Dubbo Spring Cloud 也注册到 Nacos

必须相同的 Nacos 服务命名规则：

`${CATEGORY}:${PROTCOL}:${INTERFACE_CLASS_NAME}:${VERSION}:${GROUP}`



Spring Cloud Commons 抽象 API

`${CATEGORY}:${PROTCOL}:${INTERFACE_CLASS_NAME}:${VERSION}:${GROUP}`



### Feign Contract 实现

- 默认 Feign Contract
  - Spring Cloud 不会运用
  - Dubbo 也不会运用
- JAX-RS (1 和 2 ) Feign Contract -> JAX-RS 请求映射元信息
- Spring Cloud OpenFeign Contract -> Spring WebMVC 请求映射元信息



#### 问题：

- Dubbo Registry API 如何适配不同的 Spring Cloud Registration 的实现
- Dubbo 服务接口作为 Spring Cloud 应用名称，与普通不同的应用名同级
  - Dubbo 服务接口：
    - providers:dubbo:org.springframework.cloud.alibaba.dubbo.service.EchoService:1.0.0
    - providers:rest:org.springframework.cloud.alibaba.dubbo.service.EchoService:1.0.0
    - consumers:dubbo:org.springframework.cloud.alibaba.dubbo.service.EchoService:1.0.0
  - Spring Cloud 应用：
    - spring-cloud-alibaba-dubbo
- Dubbo 服务端暴露 REST 和 Dubbo 协议
  - 已支持：客户端 Spring Cloud OpenFeign 或者 RestTemplate 直接调用
  - 待支持：客户端 Spring Cloud OpenFeign 走 Dubbo 协议
    - 服务端
      - 应用名称：`spring-cloud-alibaba-dubbo`
      - REST 服务
        - 请求映射：
          - URI: `/echo`
          - query string : `message=${message}`
      - Dubbo 服务
        - 协议：dubbo
        - 端口：12345
        - 接口：`org.springframework.cloud.alibaba.dubbo.service.EchoService#echo(String)`
      - 暴露动作
        - 当 Dubbo 服务暴露时，将 REST 请求映射信息推送到配置中心
    - 客户端
      - 调用应用：`spring-cloud-alibaba-dubbo`
      - OpenFeign 
        - 请求映射：
          - URI: `/echo`
          - query string : `message=${message}`
      - 订阅动作
        - 当应用启动时，去订阅调用应用的 Dubbo REST 请求映射信息
      - 匹配动作
        - 当调用应用的请求映射信息与 OpenFeign 接口定义的元信息如果匹配的话，可以替换为 dubbo 协议的调用
  - 待支持：客户端 Spring Cloud `@LoadBalanced` RestTemplate 走 Dubbo 协议
- Spring Cloud Feign 客户端如何控制调用非 REST 协议的Dubbo 服务端
  - dubbo
  - HTTP
  - Hession



#### TODO 

Dubbo 服务接口与 Spring Cloud 应用名称做隔离



## 功能演示



## 吐槽 Spring Cloud OpenFeign 实现设计

- `@FeignClient` 接口
  - 生成 Spring Bean
    - `org.springframework.cloud.openfeign.FeignClientFactoryBean`
  - spring-cloud-context 抽象：
    - 关联 `NamedContextFactory`
    - 生成一个 `ApplicationContext`
- `@EnableFeignClients`
  - `FeignClientsRegistrar`



## 介绍 Spring Cloud 官方实现中的特性缺失

- Spring Cloud 不支持配置推送
  - 间接方法：`spring-cloud-bus`
    - 依赖：
      - `spring-cloud-stream`
      - MQ 实现（RocketMQ）
  - 间接方式：`spring-cloud-config` + `spring-cloud-bus` + `spring-cloud-monitor`



## 为什么说 Spring Cloud 官方实现“能用但不成熟”

