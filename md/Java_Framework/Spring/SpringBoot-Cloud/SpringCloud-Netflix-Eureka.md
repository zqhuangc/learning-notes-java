# Spring Cloud Netflix Eureka



### 传统的服务治理



XML-RPC -> XML 方法描述、方法参数 -> WSDL（WebService 定义语言）
Webservise -> SOAP(HTTP、SMTP)  -> 文本协议（头部分、体部分）

REST -> JSON/XML(Schema：类型、结构)  -> 文本协议（HTTP Header、Body）
W3C Schema：xsd:string  原子类型，自定义自由组合原子类型
Java POJO：int、StringResponse Header -> Conten-Type:application/json;charset=UTF-8
Dubbo：Hession、Jaca Serialization（二进制），跨语言不变，一般通过 Client （Java、C++），

> 二进制的性能是非常好（字节流，免去字符流（字符编码），免去了字符解释，机器友好，对人不友好）
>
> 序列化：把编程语言的数据结构转换成字节流
>
> 反序列化：字节流转换成编程语言的数据结构（原生类型组合）



### 传统的服务发现

WebService
​     -- UDDI
REST
​     -- HATEOAS
Java
​     -- JMS
​     -- JNDI

### 高可用架构

#### 基本原则

1. 消除单点失败
2. 可靠性交迭
3. 故障探测

URI：统一资源定位符
URI 用于网络资源定位的描述
网络是通讯方式
资源是需要消费媒介
定位是路由 



Proxy：一般性代理，路由
​    Nginx：反向代理
Broker：包括路由，并且管理，老的称谓（MOM）
​    Message Broker：消息路由、消息管理（消息是否可达）



#### 可用性比率计算

可用性比率：通过时间来计算（一年或一个月）
比如：一年 99.99%
可用时间：365 * 24 * 3600 * 99.99%
不可用时间：365 * 24 * 3600 * 0.01% = 3153.6 秒  < 一个小时
不可用时间：1 个小时 推算一年 1 / 24 / 365 = 0.01%





单台机器 不可用比率： 1%

两台机器 不可用比率： 1%  *  1%

N 台机器 不可用比率： 1% ^ n



#### 可靠性

微服务里面的问题：
一次调用：
A       ->     B     ->   C
99%  ->   99%  ->  99%  =  97%

A       ->     B     ->   C       -> D
99%  ->   99%  ->  99%  ->  99%=  96%

结论： 增加机器可以提高可用性，增加服务调用会降低可靠性，同时降低了可用性



## 服务发现

##  Eureka 客户端

* Neflix Eureka Client
  依赖：spring-cloud-starter-eureka
  API:
  EurekaClient
  ServiceInstance
  Spring Cloud Commons

激活：@EnableEurekaClient
健康指标：HealthIndicator

项目创建：Eureka-Discovery Actuator Web

> RestTemplate   @LoadBalanced
>
> 前缀 http://user-service-provider/
>
> postForObject

@SpringCloudApplication

```properties
### 消费者配置类似
spring.applicaiton.name = user-service-provider
### Eureka 注册中心服务器端口
eureka.server.port = 9090
### 服务提供方端口
server.port = 7070
### Eureka Server 服务 URL,用于客户端注册
eureka.client.serviceUrl.defaultZone = \
http://localhost:${eureka.server.port}/eureka
### Management 安全失效
management.security.enabled = false
```



## Eureka 服务器

Neflix Eureka Server
激活：@EnableEurekaServer

```properties
### 取消服务器自我注册
eureka.client.register-with-eureka = false
### 注册中心的服务器，没有必要再去检索服务
eureka.client.fetch-registry = false
### Eureka Server 服务 URL,用于客户端注册
eureka.client.serviceUrl.defaultZone = \
http://localhost:${server.port}/eureka
```



**Eureka 服务器**一般不需要自我注册，也不需要注册其他服务器

Eureka 自我注册的问题，服务器本身没有启动

> Fast Fail：快速失败
>
> Fault-Torerance：容错



通常经验，Eureka 服务器不需要开启自动注册，也不需要检索服务。
但这两个设置并不影响作为服务器的使用，不过建议关闭，为了减少不必要的异常堆栈，减少错误的干扰（比如：系统异常和业务异常）

**Replicas**



###其他内容

dubbo 为 服务维度

spring cloud 为应用维度 

* 分布式系统基本组成
  - 服务提供方（Provider）
  - 服务消费方（Consumer）
  - 服务注册中心（Registry）（提供者和消费者都需注册）
  - 服务路由（Router）
  - 服务代理（Broker）
  - 通讯协议（Protocol）

服务管理、监控、健康检查

region  zone

##  问答

1. consul 和 eureka 一样吗？
   提供功能类似，consul 功能更强大，广播式服务发现/注册
2. 重启 eureka 服务器，客户端应用不需重启，客户端在不停上报信息，服务器重启阶段会大量报错
3. consumer 分别注册成多个服务，还是同意放在一起注册成一个服务？权限应该如何处理？
   consumer是否分成多个服务，要分情况，大多数情况是需要，根据应用职责划分。权限根据服务方法需要，比如有些敏感操作的话，可以根据不同用户做鉴权。
4. 客户上报信息存储在哪里？内存还是数据库？
   很多为集合的属性，都是在内存中缓存着，EurekaClient 并不是所有服务，需要的服务。比如：Eureka Server 管理了 200 个应用，每个应用存在 100 各实例，总体管理 20000 个实例， 客户端根据自己需要的应用实例。
5. 可用性保证？status，会自动切换，不过不一定及时，此时服务器可能存在脏数据，或者轮询更新时间未达
6. 多 service 如何保证事务？
   需要分布式事务实现（JTA），一般互联网项目，没有这种昂贵的操作
   XResource