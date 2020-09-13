OpenTSDB（时序数据库） 单元存放 metrics 信息

## 理论基础

### 客户端负载均衡

* 优势

  稳定性高

* 不足

  升级成本高

### 服务端负载均衡

* 优势

  统一维护，成本低

* 不足

  一旦故障，影响大



### 调度算法

* 先来先服务（First Come First Served）
* 最早截止时间优先（Earliest deadline first）
* 最短保留时间优先（Shortest remaining time first）
* 固定优先级（Fixed Priority）
* 轮训（Round-Robin）
* 多级别队列列（Multilevel Queue）



### 特性

* 非对称负载（Asymmetric load）
* 分布式拒绝服务攻击保护
* 直接服务返回
* 健康检查
* 优先级队列
* 其他



## Spring RestTemplate

序列化/反序列化：HttpMessageConvertor
实现适配：ClientHttpRequestFactory
请求拦截：ClientHttpRequestInterceptor



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



## Netflix Ribbon

* 依赖
  org.springframework.cloud:spring-cloud-starter-ribbon
* 激活
  @RibbonClient
* 配置
  ${serviceId}.ribbon.listOfServers = ${serviceUrl},${serviceUrl2},…
*  激活服务发现
  @EnableDiscoveryClient
* 触发负载均衡
  @LoadBalanced
* 客户端调用
  RestTemplate



### 源码核心

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



# Spring Cloud 负载均衡



## Netflix Ribbon



### 引入Maven 依赖

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>
```



### 激活 Ribbon 客户端

```java
package com.segumentfault.springcloudlesson6;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.ribbon.RibbonClient;
import org.springframework.cloud.netflix.ribbon.RibbonClients;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
// 多个 Ribbon 定义
@RibbonClients({
        @RibbonClient(name = "spring-cloud-service-provider")
})
public class SpringCloudLesson6Application {

	public static void main(String[] args) {
		SpringApplication.run(SpringCloudLesson6Application.class, args);
	}

	//声明 RestTemplate
	@Bean
	public RestTemplate restTemplate(){
		return new RestTemplate();
	}
}
```



### 配置 Ribbon 客户端

application.properties

```properties
### 配置ribbon 服务的提供方
spring-cloud-service-provider.ribbon.listOfServers = \
  http://${serivce-provider.host}:${serivce-provider.port}
```



### 调整 RestTemplate

```java
//声明 RestTemplate
@LoadBalanced // RestTemplate 的行为变化
@Bean
public RestTemplate restTemplate(){
    return new RestTemplate();
}
```





## Neflix Ribbon 整合 Eureka



### 激活服务发现的客户端

```java
package com.segumentfault.springcloudlesson6;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.ribbon.RibbonClient;
import org.springframework.cloud.netflix.ribbon.RibbonClients;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.ComponentScan;
import org.springframework.web.client.RestTemplate;

@SpringBootApplication
// 多个 Ribbon 定义
@RibbonClients({
        @RibbonClient(name = "spring-cloud-service-provider")
})
@EnableDiscoveryClient // 激活服务发现客户端
public class SpringCloudLesson6Application {

	public static void main(String[] args) {
		SpringApplication.run(SpringCloudLesson6Application.class, args);
	}

    //声明 RestTemplate
    @LoadBalanced // RestTemplate 的行为变化
    @Bean
    public RestTemplate restTemplate(){
        return new RestTemplate();
    }
}

```



### 创建并且启动 Eureka Server

以`spring-cloud-lesson6-eureka-server` 为例



#### 激活 Eureka Server

```java
package com.segumentfault.springcloudlesson6eurekaserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class SpringCloudLesson6EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringCloudLesson6EurekaServerApplication.class, args);
	}
}
```



#### 配置 Eureka 服务器

```properties
## Eureka Serer
spring.application.name = spring-cloud-eureka-server

## 服务端口
server.port = 10000

## Spring Cloud Eureka 服务器作为注册中心
## 通常情况下，不需要再注册到其他注册中心去
## 同时，它也不需要获取客户端信息
### 取消向注册中心注册
eureka.client.register-with-eureka = false
### 取消向注册中心获取注册信息（服务、实例信息）
eureka.client.fetch-registry = false
## 解决 Peer / 集群 连接问题
eureka.instance.hostname = localhost
eureka.client.serviceUrl.defaultZone = http://${eureka.instance.hostname}:${server.port}/eureka
```



#### 启动 Eureka Server



### 调整 Ribbon 客户端连接 Eureka Server

`applicaiont.properties`

```properties
## 服务提供方
spring.application.name = spring-cloud-ribbon-client

### 服务端口
server.port = 8080

### 管理安全失效
management.security.enabled = false

### 暂时性关闭 Eureka 注册
## 当使用 Eureka 服务发现时，请注释掉一下配置
# eureka.client.enabled = false

## 连接 Eureka Sever
eureka.client.serviceUrl.defaultZone = http://localhost:10000/eureka/

### 服务提供方主机
serivce-provider.host = localhost
### 服务提供方端口
serivce-provider.port = 9090

serivce-provider.name = spring-cloud-service-provider

### 配置ribbon 服务地提供方
## 当使用 Eureka 服务发现时，请注释掉一下配置
# spring-cloud-service-provider.ribbon.listOfServers = \
  http://${serivce-provider.host}:${serivce-provider.port}
```



#### 调整服务提供方并且连接 Eureka Server

```properties
## 服务提供方
spring.application.name = spring-cloud-service-provider

### 服务端口
server.port = 9090

### 管理安全失效
management.security.enabled = false

### 暂时性关闭 Eureka 注册
## 当使用 Eureka 服务发现时，请注释掉一下配置
# eureka.client.enabled = false

## 连接 Eureka Sever
eureka.client.serviceUrl.defaultZone = http://localhost:10000/eureka/
```



再启动两台服务提供方实例

--server.port=9091

--server.port=9092











实际请求客户端

- LoadBalancerClient
  - RibbonLoadBalancerClient

负载均衡上下文

- LoadBalancerContext
  - RibbonLoadBalancerContext

负载均衡器

- ILoadBalancer
  - BaseLoadBalancer
  - DynamicServerListLoadBalancer
  - ZoneAwareLoadBalancer
  - NoOpLoadBalancer

负载均衡规则

核心规则接口

- IRule
  - 随机规则：RandomRule
  - 最可用规则：BestAvailableRule
  - 轮训规则：RoundRobinRule
  - 重试实现：RetryRule
  - 客户端配置：ClientConfigEnabledRoundRobinRule
  - 可用性过滤规则：AvailabilityFilteringRule
  - RT权重规则：WeightedResponseTimeRule
  - 规避区域规则：ZoneAvoidanceRule

PING 策略

核心策略接口

- IPingStrategy

PING 接口

- IPing
  - NoOpPing
  - DummyPing
  - PingConstant
  - PingUrl

Discovery Client 实现

- NIWSDiscoveryPing









# Spring Cloud Netflix Ribbon 源码分析

## 学习方向（自己的看法）

@LoadBalanced 如何对 RestTemplate 默认实现类进行更换的？

Ribbon 选择 Server 的逻辑？

核心接口间如何联系？

## Netflix Ribbon 核心接口



`RestTemplate`增加了一个`LoadBalancerInterceptor`，调用 Netflix 中的 `LoadBalancer`实现，根据 Eureka 客户端**应用**获取目标应用 **IP +Port** 信息，轮询的方式调用，调整默认的 RestTemplate 实现。

转换

RestTemplate定制 ，添加 interceptor

### 负载均衡器客户端（实际请求客户端）

- LoadBalancerClient
  - RibbonLoadBalancerClient

+ 主要职责
  - 转化 URI：将含应用名称URI 转化成具体主机+端口的形式
  - 选择服务实例：通过负载算法，选择指定服务中的一台机器实例
  - 请求执行回调：针对选择后服务实例，执行具体的请求回调操作
  - 默认实现：RibbonLoadBalancerClient
  - 自动装配源：RibbonAutoConfiguration#loadBalancerClient()

### 负载均衡上下文

- LoadBalancerContext
  - RibbonLoadBalancerContext

+ 主要职责
  - 转化 URI：将含应用名称URI 转化成具体主机+端口的形式
  - 组件关联：关联 RetryHandler、ILoadBalancer 等
  - 记录服务统计信息：记录请求相应时间、错误数量等
  - 默认实现：RibbonLoadBalancerContext
  - 自动装配源：RibbonClientConfiguration#ribbonLoadBalancerContext(…)

### 负载均衡器

- ILoadBalancer
  - BaseLoadBalancer
  - DynamicServerListLoadBalancer
  - ZoneAwareLoadBalancer
  - NoOpLoadBalancer

+ 主要职责
  - 增加服务器
  - 获取服务器：通过关联 Key 获取、获取所有服务列表、获取可用服务器列表
  - 服务器状态：标记服务器宕机
  - 默认实现：ZoneAwareLoadBalancer
  - 自动装配源：RibbonClientConfiguration#ribbonLoadBalancer(…)

### 负载均衡规则

#### 核心规则接口

- IRule
  - 随机规则：RandomRule
  - 最可用规则：BestAvailableRule
  - 轮询规则：RoundRobinRule
  - 重试实现：RetryRule
  - 客户端配置：ClientConfigEnabledRoundRobinRule
  - 可用性过滤规则：AvailabilityFilteringRule
  - RT权重规则：WeightedResponseTimeRule
  - 规避区域规则：ZoneAvoidanceRule



+ 主要职责
  - 选择服务器：根据负载均衡器以及关联Key 获取候选的服务器
  - 默认实现：ZoneAvoidanceRule
  - 自动装配源：RibbonClientConfiguration#ribbonRule(…)



静态代理

### PING 策略

#### 核心策略接口

- IPingStrategy

#### PING 接口

- IPing
  - NoOpPing
  - DummyPing
  - PingConstant
  - PingUrl



+ 主要职责
  - 活动检测：根据指定的服务器，检测其是否活动
  - 默认实现：DummyPing
  - 自动装配源：RibbonClientConfiguration#ribbonPing(…)

isAlive

##### Discovery Client 实现

- NIWSDiscoveryPing



### 服务器列表 ServerList

+ 主要职责
  - 获取初始化服务器列表
  - 获取更新服务器列表
  - 默认实现：ConfigurationBasedServerList 或 DiscoveryEnabledNIWSServerList
  - 自动装配源：RibbonClientConfiguration



## Netflix Ribbon 自动装配

### RibbonAutoConfiguration

- LoadBalancerClient
- PropertiesFactory
- LoadBalancerAutoConfiguration
- @LoadBalanced
- RestTemplate



### RibbonClientConfiguration

- LoadBalancerContext
- IRule
- IPing
- ServerList
- ILoadBalancer
- IClientConfig



## Netflix Ribbon 配置化组件

### PropertiesFactory

- ILoadBalancer
- IPing
- IRule
- ServerList
- ServerListFilter





## 利用 RibbonLoadBalancerClient



### 新建一个工程

三个模块：

- user-api：公用 API
- user-robbon-client：客户端应用
- user-service-provider：服务端应用



#### 实现 user-robbon-client

##### 配置信息

`application.properties`:

```properties
## 用户 Ribbon 客户端应用
spring.application.name = user-ribbon-client

## 服务端口
server.port = 8080

## 提供方服务名称
provider.service.name = user-service-provider
## 提供方服务主机
provider.service.host = localhost
## 提供方服务端口
provider.service.port = 9090

## 关闭 Eureka Client，显示地通过配置方式注册 Ribbon 服务地址
eureka.client.enabled = false

## 定义 user-service-provider Ribbon 的服务器地址
## 为 RibbonLoadBalancerClient 提供服务列表
user-service-provider.ribbon.listOfServers = \
  http://${provider.service.host}:${provider.service.port}
```



##### 编写客户端调用

```java
package com.segumentfault.spring.cloud.lesson7.user.ribbon.client.web.controller;

import com.segumentfault.spring.cloud.lesson7.domain.User;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.loadbalancer.LoadBalancerClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

import javax.ws.rs.GET;
import java.io.IOException;

/**
 * 用户 Ribbon Controller
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
@RestController
public class UserRibbonController {

    /**
     * 负载均衡器客户端
     */
    @Autowired
    private LoadBalancerClient loadBalancerClient;

    @Value("${provider.service.name}")
    private String providerServiceName;


    @GetMapping("")
    public String index() throws IOException {

        User user = new User();
        user.setId(1L);
        user.setName("小马哥");

        // 选择指定的 service Id
        ServiceInstance serviceInstance = loadBalancerClient.choose(providerServiceName);

        return loadBalancerClient.execute(providerServiceName, serviceInstance, instance -> {

            //服务器实例，获取 主机名（IP） 和 端口
            String host = instance.getHost();
            int port = instance.getPort();
            String url = "http://" + host + ":" + port + "/user/save";
            RestTemplate restTemplate = new RestTemplate();

            return restTemplate.postForObject(url, user, String.class);

        });

    }

}
```



##### 编写引导类

```java
package com.segumentfault.spring.cloud.lesson7.user.ribbon.client;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.ribbon.RibbonClient;

/**
 * 引导类
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
@SpringBootApplication
@RibbonClient("user-service-provider") // 指定目标应用名称
public class UserRibbonClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserRibbonClientApplication.class, args);
    }
}
```



#### 实现 user-service-provider

##### 配置信息

`application.properties`:

```properties
## 用户服务提供方应用信息
spring.application.name = user-service-provider

## 服务端口
server.port = 9090

## 关闭 Eureka Client，显示地通过配置方式注册 Ribbon 服务地址
eureka.client.enabled = false
```



##### 实现 `UserService`

```java
package com.segumentfault.spring.cloud.lesson7.user.service.provider.service;

import com.segumentfault.spring.cloud.lesson7.api.UserService;
import com.segumentfault.spring.cloud.lesson7.domain.User;
import org.springframework.stereotype.Service;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * 内存实现{@link UserService}
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
@Service
public class InMemoryUserService implements UserService {

    private Map<Long, User> repository = new ConcurrentHashMap<>();

    @Override
    public boolean saveUser(User user) {
        return repository.put(user.getId(), user) == null;
    }

    @Override
    public List<User> findAll() {
        return new ArrayList(repository.values());
    }
}
```



##### 实现 Web 服务

```java
package com.segumentfault.spring.cloud.lesson7.user.service.web.controller;

import com.segumentfault.spring.cloud.lesson7.api.UserService;
import com.segumentfault.spring.cloud.lesson7.domain.User;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

/**
 * 用户服务提供方 Controller
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
@RestController
public class UserServiceProviderController {

    @Autowired
    private UserService userService;

    @PostMapping("/user/save")
    public boolean user(@RequestBody User user){
        return userService.saveUser(user);
    }

}
```



##### 编写引导类

```java
package com.segumentfault.spring.cloud.lesson7.user.service;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.ribbon.RibbonClient;

/**
 * 引导类
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
@SpringBootApplication
public class UserServiceProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServiceProviderApplication.class, args);
    }
}
```



#### 分析调用链路

##### 选择服务器逻辑

LoadBalancerClient（LoadBalancerClient） -> ILoadBalancer（ZoneAwareLoadBalancer） ->  IRule (ZoneAvoidanceRule)



#### 自定义实现 `IRule`



##### 扩展 `AbstractLoadBalancerRule` ： `MyRule`

```java
package com.segumentfault.spring.cloud.lesson7.user.ribbon.client.rule;

import com.netflix.client.config.IClientConfig;
import com.netflix.loadbalancer.AbstractLoadBalancerRule;
import com.netflix.loadbalancer.ILoadBalancer;
import com.netflix.loadbalancer.IRule;
import com.netflix.loadbalancer.Server;

import java.util.List;

/**
 * 自定义{@link IRule} 实现，永远选择最后一台可达服务器
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
public class MyRule extends AbstractLoadBalancerRule {

    @Override
    public void initWithNiwsConfig(IClientConfig clientConfig) {

    }

    @Override
    public Server choose(Object key) {

        ILoadBalancer loadBalancer = getLoadBalancer();

        //获取所有可达服务器列表
        List<Server> servers = loadBalancer.getReachableServers();
        if (servers.isEmpty()) {
            return null;
        }

        // 永远选择最后一台可达服务器
        Server targetServer = servers.get(servers.size() - 1);
        return targetServer;
    }

}
```



##### 将 `MyRule` 暴露成 `Bean`

> 通过`RibbonClientConfiguration`学习源码：
>
> ```java
> @Bean
> @ConditionalOnMissingBean
> public IRule ribbonRule(IClientConfig config) {
>    if (this.propertiesFactory.isSet(IRule.class, name)) {
>       return this.propertiesFactory.get(IRule.class, config, name);
>    }
>    ZoneAvoidanceRule rule = new ZoneAvoidanceRule();
>    rule.initWithNiwsConfig(config);
>    return rule;
> }
> ```



```java
/**
 * 将 {@link MyRule} 暴露成 {@link Bean}
 *
 * @return {@link MyRule}
 */
@Bean
public IRule myRule() {
    return new MyRule();
}
```



#### 配置化实现组件

> 通过学习`PropertiesFactory` 源码：
>
> ```java
> 	public PropertiesFactory() {
> 		classToProperty.put(ILoadBalancer.class, "NFLoadBalancerClassName");
> 		classToProperty.put(IPing.class, "NFLoadBalancerPingClassName");
> 		classToProperty.put(IRule.class, "NFLoadBalancerRuleClassName");
> 		classToProperty.put(ServerList.class, "NIWSServerListClassName");
> 		classToProperty.put(ServerListFilter.class, "NIWSServerListFilterClassName");
> 	}
> ```
>
> 可知`NFLoadBalancerClassName` 等是可以配置的

##### 实现 `IPing` : `MyPing`

```java
package com.segumentfault.spring.cloud.lesson7.user.ribbon.client.ping;

import com.netflix.loadbalancer.IPing;
import com.netflix.loadbalancer.Server;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.client.RestTemplate;
import org.springframework.web.util.UriComponentsBuilder;

import java.net.URI;

/**
 * 实现 {@link IPing} 接口：检查对象 /health 是否正常状态码:200
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
public class MyPing implements IPing {

    @Override
    public boolean isAlive(Server server) {

        String host = server.getHost();
        int port = server.getPort();
        // /health endpoint
        // 通过 Spring 组件来实现URL 拼装
        UriComponentsBuilder builder = UriComponentsBuilder.newInstance();
        builder.scheme("http");
        builder.host(host);
        builder.port(port);
        builder.path("/health");
        URI uri = builder.build().toUri();

        RestTemplate restTemplate = new RestTemplate();

        ResponseEntity responseEntity = restTemplate.getForEntity(uri, String.class);
        // 当响应状态等于 200 时，返回 true ，否则 false
        return HttpStatus.OK.equals(responseEntity.getStatusCode());
    }
    
}
```



##### 增加配置

`application.properties`:

```properties
## 扩展 IPing 实现
user-service-provider.ribbon.NFLoadBalancerPingClassName = \
  com.segumentfault.spring.cloud.lesson7.user.ribbon.client.ping.MyPing
```





