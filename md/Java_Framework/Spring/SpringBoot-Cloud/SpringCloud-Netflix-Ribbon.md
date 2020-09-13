# Spring Cloud Netflix Ribbon

## 配置源码位置（EurekaClientConfigBean）

## Eureka 高可用架构

### 高可用注册中心集群

只需要增加 Eureka 服务器注册 URL：

```properties
#启动参数 --server.port = 9091，${random.int[7070,7079]}
### 这方法并不好
eureka.client.serviceUrl.defaultZone = \
http://localhost:9090/eureka,http://localhost:9091/eureka
```

如果 Eureka 客户端应用配置多个 Eureka 注册服务器，那么默认情况只有第一台可用的服务器，存在注册信息

如果第一台可用的 Eureka 服务器 Down 掉了，那么 Eureka客户端应用将会选择下一台可用的 Eureka 服务器



#### 配置源码 （EurekaClientConfigBean）

配置项 `eureka.client.serviceUrl`实际映射的字段为 `serviceUrl`，它是 Map 类型，Key 为自定义，默认值 “defaultZone”，value是需要配置的 Eureka 注册服务器 URL。
value 可以是多值字段，通过 “,” 分割



### 获取注册信息时间间隔

Eureka 客户端需要获取 Eureka  服务器注册信息，这个方便服务调用
Eureka 客户端：`EurekaClient`，关联应用集合： `Applications`
单个应用信息：`Application`，关联多个应用实例
单个应用实例：`InstanceInfo`
当 Eureka 客户端需要调用某个服务时，比如`user-service-consumer`调用`user-service-provider`，`user-service-provider`实际对应对象是`Application`，关联了许多应用实例（`InstanceInfo`）。如果应用`user-service-provider`的应用实例发生变化时，那么`user-service-consumer`是需要感知的。比如：`user-service-provider`机器从 10 台降到 5 台，那么，作为调用方的`user-service-consumer`需要知道这个变化。变化过程可能存在一定的延迟，可以通过调整注册信息时间间隔来减少错误。

#### 具体配置项

```properties
### 调整注册信息的获取周期，默认值：30 秒
eureka.client.registryFetchIntervalSeconds = 5
```



### 实例信息复制信息时间

具体就是客户端信息的上报到 Eureka 服务器时间。当 Eureka 客户端应用上报的频率越频繁 ，那么 Eureka 服务器的应用状态管理一致性越高

#### 具体配置项

```properties
### 调整客户端应用状态信息上报的周期，默认值：30 
eureka.client.instanceInfoReplicationIntervalSeconds = 5
```

> Eureka 的应用信息同步方式：拉模式
>
> Eureka 的应用信息上报方式：推模式

### 实例 ID

从 Eureka Server Dashboard 里面可以看到具体某个应用中的实例信息

命名模式：`${hostname}：${spring.application.name}：${server.port}`

#### 配置项

EurekaInstanceConfig

```properties
### Eureka 应用实例 Id
eureka.instance.instanceId = ${spring.application.name}:${server.port}
```



### 实例端点映射

```properties
### Eureka 客户端应用实例状态 URL 默认 /info
eureka.instance.statusPageUrlPath = /health
```



## Eureka 服务端高可用

### 构建 Eureka 服务器相互注册

#### Eureka Server1 -> Profile:peer1

##### 配置项

```properties
### application-peer1.properties
spring.application.name = spring-cloud-eureka-server
server.port = 9090
### 取消服务器自我注册
eureka.client.register-with-eureka = true
### 注册中心的服务器，没有必要再去检索服务
eureka.client.fetch-registry = true
### Eureka Server 服务 URL,用于客户端注册
### 当前 Eureka 服务器向 9091（Eureka 服务器 复制数据）
eureka.client.serviceUrl.defaultZone = \
http://localhost:9091/eureka
```

#### Eureka Server2 -> Profile:peer2

##### 配置项

```properties
### Eureka Server2 profile
### application-peer2.properties
spring.application.name = spring-cloud-eureka-server
server.port = 9091
### 取消服务器自我注册
eureka.client.register-with-eureka = true
### 注册中心的服务器，没有必要再去检索服务
eureka.client.fetch-registry = true
### Eureka Server 服务 URL,用于客户端注册
### 当前 Eureka 服务器向 9090（Eureka 服务器 复制数据）
eureka.client.serviceUrl.defaultZone = \
http://localhost:9090/eureka
```



> 通过启动参数，分别激活： 
>
> --spring.profile.active = pee1
>
> --spring.profile.active = pee2
>
> dashboard：replicas信息



## Spring RestTemplate



### HTTP消息转换器：HttpMessageConvertor

自定义实现
编码问题
**切换序列化/反序列化的协议**

### HTTP Client 适配工厂：ClientHttpRequestFactory

这个方面主要考虑使用 HttpClient 的偏好：

* Spring 实现
  - SimpleClientHttpRequestFactory
* HttpClient
  - HttpComponentsClientHttpRequestFactory
* OkHttp
  - OkHttp3ClientHttpRequestFactory
  - OkHttpClientHttpRequestFactory

举例说明：RestTemplate 构造函数
**切换 HTTP 通讯实现，提升性能**

### HTTP 请求拦截器：ClientHttpRequestInterceptor

interceptor

**加深 RestTemplate 拦截过程的**

## 整合 Netflix Ribbon



`RestTemplate`增加了一个`LoadBalancerInterceptor`，调用 Netflix 中的 `LoadBalancer`实现，根据 Eureka 客户端应用获取目标应用 IP +Port 信息，轮询的方式调用。



### 实际请求客户端

* LoadBalancerClient
  * RibbonLoadBalancerClient

### 负载均衡上下文

* LoadBalancerContext
  * RibbonLoadBalancerContext

### 负载均衡器

* ILoadBalancer
  * BaseLoadBalancer
  * DynamicServerListLoadBalancer
  * ZoneAwareLoadBalancer
  * NoOpLoadBalancer



### 负载均衡规则
#### 核心规则接口

* IRule
  * 随机规则：RandomRule
  * 最可用规则：BestAvailableRule
  * 轮询规则：RoundRobinRule
  * 重试实现：RetryRule
  * 客户端配置：ClientConfigEnabledRoundRobinRule
  * 可用性过滤规则：AvailabilityFilteringRule
  * RT权重规则：WeightedResponseTimeRule
  * 规避区域规则：ZoneAvoidanceRule



### PING 策略

#### 核心策略接口

* IPingStrategy

#### PING 接口

* IPing
  * NoOpPing
  * DummyPing
  * PingConstant
  * PingUrl

##### Discovery Client 实现

*  NIWSDiscoveryPing



## 问答

1. 为什么要用 eureka？目前业界比较稳定云计算的开发员中间件，虽然有一些不足，基本上可用
2. eureka 主要功能为什么不能用浮动 IP 代替呢？ 如果要使用 浮动 IP，也可以，不过需要扩展
3. consul，zookeeeper，eureka 比较？
   https://www.consul.io/intro/vs/index.html
4. 通讯是指注册到 defaultZone 配置的？默认情况往 defaultZone 注册
5. 如果服务注册中心都挂了，服务还是能够运行吧？
   服务调用还是可以运行，有可能数据会不及时、不一致
6. spring cloud 日志收集方案？ 一般用 HBase、或者 TSDB、elk
7. spring cloud 链路跟踪？sleuth模块



**opentsdb**

### note

```properties
激活
    @RibbonClient
负载均衡器
    ILoadBalancer
        AbstarctLoadBalancer
            BaseLoadBalancer
                DynamicServerListLoadBalancer
                ZoneAwareLoadBalancer
负载均衡客户端
    LoadBalancerClient
自动装配
    LoadBalancerAutoConfiguration
源码分析
    RestTemplate 自定义器
        RestTemplateCustomizer
    RestTemplate 请求拦截器
        LoadBalancerInterceptor
    标记注解
        @LoadBalanced
             
```





























