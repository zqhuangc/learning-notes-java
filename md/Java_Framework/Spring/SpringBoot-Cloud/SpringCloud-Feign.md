# Spring Cloud Feign

http 方式暴露服务？？？

#### 技术回顾

Netflix Eureka
RestTemplate
Netflix Ribbon：可以是服务端实现，也可以是客户端实现，类似于 AOP 封装，正常逻辑，容错处理



*APT*是 Annotation Processing Tool

#### 声明式 Web 服务客户端：Feign

声明式：接口声明、Annotation 驱动

Web 服务：HTTP 的方式作为通讯协议

客户端：用于服务调用的存根

Feign：原生并不是 Spring WebMVC 的实现，基于 JAX-RS（Java REST 的规范）实现。Spring Cloud 封装了 Feign，使其支持 Spring WebMVC、`RestTemplate`、`HttpMessageConverter`

>Spring WebMVC、`RestTemplate` 可以显式地自定义 `HttpMessageConverter `实现


Feign 可以将一个接口声明以 HTTP 方式调用



需要服务组件（SOA）：

1. 注册中心（Eureka Server）：服务发现和注册
   a. 应用名称、服务端口

2. Feign 客户端（服务消费）端：调用 Feign 声明接口
3. Feign 服务（服务提供）端：**不一定强制实现 Feign 声明接口**
4. Feign 声明接口（契约）：定义一种 Java 强类型接口

> Feign 客户端（服务消费）端、Feign 服务（服务提供）端、Feign 声明接口（契约）放在同一工程目录 module



增加 spring-cloud-starter-feign 依赖
@FeignClient(name = "${user.service.name}") // 服务提供应用名称，利用占位符避免未来整合硬编码

####  激活 FeignClient

@EnableFeigntClients



```java
// 引导类中
@SpringBootApplication
@RibbonClient(value="user-service-provider",configuration = BootstrapClass.class) // 指定目标应用名称
@EnableCircuitBreaker // 使用服务短路
@EnableFeignClients(clients = UserService.class) // 申明 UserService 接口作为 Feign Client 调用
```



@getmapping，@postmapping 的继承性（接口注解，实现类不用再），参数注解没有继承性，
契约为url（消费提供需要一样），接口为可选实现

### 整合Netflix Ribbon

**参考官方文档**

#### 关闭 Eureka 注册

调整客户端，关闭 Eureka

```properties
### ribbon 不使用 Eureka
ribbon.eureka.enabled = false
### 定义 ribbon 负载均衡服务列表  stores（应用名）
stores.ribbon.listOfServers:http://localhost:9090,http://.

```

**完全关闭**，则去掉对应注解@EnabledEurekaClient



#### 自定义 Ribbon 的规则

实现 IRule
暴露实现为 Spring Bean，在启动类中配置

```java
@Bean
public ClassName className(){
    return new ClassName();
}
```



### 整合 Netflix Hystrix





## 问答

1. FeignClient 类似 Dubbo，不过需要增加以下 @Annotation，和调用本地接口类似
2. Feign 通过注释驱动弱化了调用 Service 细节，Feign API 暴露 URI，比如：“/person/save”
3. Ribbon 对于 Eureka 不强依赖，不过也不排斥
4. 生产环境上也都是 Feign？ 不少公司在用，需要 Spring Cloud 更多整合：
   Feign作为客户端
   Ribbon 作为负载均衡
   Eureka 作为注册中心
   Zuul 作为网关
   Security 作为安全 OAuth 2 认证
5. Ribbon 是控制全局的负载均衡，主要作用于客户 Feign，Controller 是调用  Feign 接口，可能让人感觉直接作用于 Controller
6. Eureka 中也有Ribbon 的简单实现，可参考：`com.netflix.ribbon:ribbon-eureka`
7. 如果服务提供方，没有接口，客户端怎么处理？要根据服务信息，自建 Feign接口？
   可以，可是 Feign 的接口定义就是要求强制实现
8. 无法连接注册中心的老服务，如何调用 cloud 服务？
   可以通过域名的配置 Ribbon 服务白名单
9. eureka有时监控不到宕机的服务，正常的启动方式是什么？
   可以调整心跳检测的频率
10. eureka宕机怎么办？需要其他设备来监控。PING Eureka 服务器是否可用，或者高可用 Eureka。