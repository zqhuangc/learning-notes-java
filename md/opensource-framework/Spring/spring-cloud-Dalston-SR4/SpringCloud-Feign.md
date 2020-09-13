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

Primary 的问题

####  激活 FeignClient

@EnableFeigntClients



```java
// 引导类中
@SpringBootApplication
@RibbonClient(value="user-service-provider",configuration = BootstrapClass.class) // 指定目标应用名称
@EnableCircuitBreaker // 使用服务短路
@EnableFeignClients(clients = UserService.class) // 申明 UserService 接口作为 Feign Client 调用
```

###  使用的关键

接口标注 @FeignClient 关联 服务提供应用名

服务调用方标注 @EnableFeignClients

服务提供  可选实现接口   url 相同的方法



Feign 的继承性，看官方文档，（接口方法标注注解，实现类不用再标注），参数注解没有继承性

契约为 url（消费提供需要一样），接口为可选实现

核心为 url ，调用服务时映射相同的 url



marshalling --- xml



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

   一般不希望协议暴露

5. Ribbon 是控制全局的负载均衡，主要作用于客户 Feign，Controller 是调用  Feign 接口，可能让人感觉直接作用于 Controller

6. Eureka 中也有Ribbon 的简单实现，可参考：`com.netflix.ribbon:ribbon-eureka`

7. 如果服务提供方，没有接口，客户端怎么处理？要根据服务信息，自建 Feign接口？
   可以，可是 Feign 的接口定义就是要求强制实现

8. 无法连接注册中心的老服务，如何调用 cloud 服务？
   可以通过域名的配置 Ribbon 服务白名单

9. eureka有时监控不到宕机的服务，正常的启动方式是什么？
   可以调整心跳检测的频率

10. eureka宕机怎么办？需要其他设备来监控。PING Eureka 服务器是否可用，或者高可用 Eureka。



config server 负载均衡？ 默认实现无法覆盖，默认取第一个，除非第一个出问题



# Spring Cloud 服务调用



## 核心概念

### 预备知识
• 远程过程调用（RPC）
• 接口定义语言（IDL）
• 通讯协议（Protocol）
• Netflix Feign
• Spring Cloud 技术回顾 （服务短路 、负载均衡、服务发现、分布式配置等）



#### 远程过程调用（RPC）
一个计算机通信协议。该协议允许运行于一台计算机的程序调用⼀一台计算机的子程序，而程序员无需额外地为这个交互作用编程。如果涉及的软件采用面向对象编程，那么远程过程调用亦可称作远程调用或远程方法调用
• 例如
• Java RMI（二进制协议）
• WebServices（文本协议）



#### 消息传递
RPC 是一种请求-响应协议，一次 RPC在客户端初始化，再由客户端将请求消息传递到远程的服务器，执行指定的带有参数的过程。经过远程服务器执行过程后，将结果作为响应内容返回到客户端。



#### 存根（Stub）

在一次分布式计算 RPC 中，客户端和服务器器转化参数的一段代码。 由于存根的参数转化，RPC执行过程如同本地执行函数调用。存根必须在客户端和服务器两端均装载，并且保持兼容。



## Spring Cloud Feign

### 依赖

•org.springframework.cloud:spring-cloud-starter-feign
• 激活
•@EnableFeignClients
• 申明
•@FeignClient



### 默认组件

•Decoder/Encoder : ResponseEntityDecoder / SpringEncoder
•Logger : Slf4jLogger
•Contract : SpringMvcContract
•Feign.Builder : HystrixFeign.Builder
• Client : LoadBalancerFeignClient（Ribbon 激活的话）



### 增加 spring-cloud-starter-feign 依赖

```xml
        <!-- 添加 Spring Cloud Feign 依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-feign</artifactId>
        </dependency>
```



### 申明 Feign 客户端



```java
package com.segumentfault.spring.cloud.lesson10.api;

import com.segumentfault.spring.cloud.lesson10.domain.User;
import org.springframework.cloud.netflix.feign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;

import java.util.List;

/**
 * 用户服务
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 1.0.0
 */
@FeignClient(name = "${user.service.name}") // 利用占位符避免未来整合硬编码
public interface UserService {

    /**
     * 保存用户
     *
     * @param user
     * @return
     */
    @PostMapping("/user/save")
    boolean saveUser(User user);


    /**
     * 查询所有的用户列表
     *
     * @return non-null
     */
    @GetMapping("/user/find/all")
    List<User> findAll();

}

```

> 注意，在使用`@FeignClient`  `name` 属性尽量使用占位符，避免硬编码。否则，未来升级时，不得不升级客户端版本。



### 激活 FeignClient



`@EnableFeigntClients`  代表可有多个

```java
package com.segumentfault.spring.cloud.lesson10.user.service.client;

import com.netflix.loadbalancer.IRule;
import com.segumentfault.spring.cloud.lesson10.api.UserService;
import com.segumentfault.spring.cloud.lesson10.user.service.client.rule.MyRule;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.feign.EnableFeignClients;
import org.springframework.cloud.netflix.ribbon.RibbonClient;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

/**
 * 引导类
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
@SpringBootApplication
@RibbonClient("user-service-provider") // 指定目标应用名称
@EnableCircuitBreaker // 使用服务短路
@EnableFeignClients(clients = UserService.class) // 申明 UserService 接口作为 Feign Client 调用
public class UserServiceClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServiceClientApplication.class, args);
    }

    /**
     * 将 {@link MyRule} 暴露成 {@link Bean}
     *
     * @return {@link MyRule}
     */
    @Bean
    public IRule myRule() {
        return new MyRule();
    }

    /**
     * 申明 具有负载均衡能力 {@link RestTemplate}
     *
     * @return
     */
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

}
```



## Spring Cloud 整合





### 整合负载均衡：Netflix Ribbon





### 客户端：激活`@FeignClient` `UserService`

```java
package com.segumentfault.spring.cloud.lesson10.user.service.client;

import com.netflix.loadbalancer.IRule;
import com.segumentfault.spring.cloud.lesson10.api.UserService;
import com.segumentfault.spring.cloud.lesson10.user.service.client.rule.MyRule;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.feign.EnableFeignClients;
import org.springframework.cloud.netflix.ribbon.RibbonClient;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

/**
 * 引导类
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
@SpringBootApplication
@RibbonClient("user-service-provider") // 指定目标应用名称
@EnableCircuitBreaker // 使用服务短路
@EnableFeignClients(clients = UserService.class) // 申明 UserService 接口作为 Feign Client 调用
public class UserServiceClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServiceClientApplication.class, args);
    }

    /**
     * 将 {@link MyRule} 暴露成 {@link Bean}
     *
     * @return {@link MyRule}
     */
    @Bean
    public IRule myRule() {
        return new MyRule();
    }

    /**
     * 申明 具有负载均衡能力 {@link RestTemplate}
     *
     * @return
     */
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

}
```



#### 客户端：配置@FeignClient(name = "${user.service.name}") 中的占位符

调整 `application.properties`

```properties
## 用户 Ribbon 客户端应用
spring.application.name = user-service-client

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

## 扩展 IPing 实现
user-service-provider.ribbon.NFLoadBalancerPingClassName = \
  com.segumentfault.spring.cloud.lesson10.user.service.client.ping.MyPing

## 配置 @FeignClient(name = "${user.service.name}") 中的占位符
## user.service.name 实际需要制定 UserService 接口的提供方
## 也就是 user-service-provider，可以使用 ${provider.service.name} 替代
user.service.name = ${provider.service.name}
```



### 服务端：实现`UserService` ，即暴露 HTTP REST 服务

调整应用：`user-service-provider`



#### 增加 `InMemoryUserService` 的Bean 名称

```java
package com.segumentfault.spring.cloud.lesson10.user.service.provider.service;

import com.segumentfault.spring.cloud.lesson10.api.UserService;
import com.segumentfault.spring.cloud.lesson10.domain.User;
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
@Service("inMemoryUserService") // Bean 名称
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



#### `UserServiceProviderController` 实现 Feign 客户端接口`UserService`

`UserServiceProviderController.java `

```java
package com.segumentfault.spring.cloud.lesson10.user.service.web.controller;

import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import com.netflix.hystrix.contrib.javanica.annotation.HystrixProperty;
import com.segumentfault.spring.cloud.lesson10.api.UserService;
import com.segumentfault.spring.cloud.lesson10.domain.User;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.Random;

/**
 * 用户服务提供方 Controller
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
@RestController
public class UserServiceProviderController implements UserService {

    @Autowired
    @Qualifier("inMemoryUserService") // 实现 Bean ： InMemoryUserService
    private UserService userService;

    private final static Random random = new Random();

    // 通过方法继承，URL 映射 ："/user/save"
    @Override
    public boolean saveUser(@RequestBody User user) {
        return userService.saveUser(user);
    }

    // 通过方法继承，URL 映射 ："/user/find/all"
    @Override
    public List<User> findAll() {
        return userService.findAll();
    }

    /**
     * 获取所有用户列表
     *
     * @return
     */
    @HystrixCommand(
            commandProperties = { // Command 配置
                    // 设置操作时间为 100 毫秒
                    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "100")
            },
            fallbackMethod = "fallbackForGetUsers" // 设置 fallback 方法
    )
    @GetMapping("/user/list")
    public Collection<User> getUsers() throws InterruptedException {

        long executeTime = random.nextInt(200);

        // 通过休眠来模拟执行时间
        System.out.println("Execute Time : " + executeTime + " ms");

        Thread.sleep(executeTime);

        return userService.findAll();
    }

    /**
     * {@link #getUsers()} 的 fallback 方法
     *
     * @return 空集合
     */
    public Collection<User> fallbackForGetUsers() {
        return Collections.emptyList();
    }

}
```



### 客户端：使用 `UserService` 直接调用远程 HTTP REST 服务



```java
package com.segumentfault.spring.cloud.lesson10.user.service.client.web.controller;

import com.segumentfault.spring.cloud.lesson10.api.UserService;
import com.segumentfault.spring.cloud.lesson10.domain.User;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

/**
 * {@link UserService} 客户端 {@link RestController}
 *
 * 注意：官方建议 客户端和服务端不要同时实现 Feign 接口
 * 这里的代码只是一个说明，实际情况最好使用组合的方式，而不是继承
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 1.0.0
 */
@RestController
public class UserServiceClientController implements UserService {

    @Autowired
    private UserService userService;

    // 通过方法继承，URL 映射 ："/user/save"
    @Override
    public boolean saveUser(@RequestBody User user) {
        return userService.saveUser(user);
    }

    // 通过方法继承，URL 映射 ："/user/find/all"
    @Override
    public List<User> findAll() {
        return userService.findAll();
    }

}
```





### 整合服务短路：Netflix Hystrix



#### API：调整`UserService` 并且实现 Fallback



`UserService` Fallback 实现

```java
package com.segumentfault.spring.cloud.lesson10.fallback;

import com.segumentfault.spring.cloud.lesson10.api.UserService;
import com.segumentfault.spring.cloud.lesson10.domain.User;

import java.util.Collections;
import java.util.List;

/**
 * {@link UserService} Fallback 实现
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 1.0.0
 */
public class UserServiceFallback implements UserService {

    @Override
    public boolean saveUser(User user) {
        return false;
    }

    @Override
    public List<User> findAll() {
        return Collections.emptyList();
    }
}
```



调整 `UserService` `@FeignClient` fallback 属性：

```java
package com.segumentfault.spring.cloud.lesson10.api;

import com.segumentfault.spring.cloud.lesson10.domain.User;
import com.segumentfault.spring.cloud.lesson10.fallback.UserServiceFallback;
import org.springframework.cloud.netflix.feign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;

import java.util.List;

/**
 * 用户服务
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 1.0.0
 */
@FeignClient(name = "${user.service.name}",fallback = UserServiceFallback.class) // 利用占位符避免未来整合硬编码
public interface UserService {

    /**
     * 保存用户
     *
     * @param user
     * @return
     */
    @PostMapping("/user/save")
    boolean saveUser(User user);


    /**
     * 查询所有的用户列表
     *
     * @return non-null
     */
    @GetMapping("/user/find/all")
    List<User> findAll();

}
```



#### 服务端： `UserServiceProviderController#findAll()` 方法整合 `@HystrixCommand`

```java
package com.segumentfault.spring.cloud.lesson10.user.service.web.controller;

import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
import com.netflix.hystrix.contrib.javanica.annotation.HystrixProperty;
import com.segumentfault.spring.cloud.lesson10.api.UserService;
import com.segumentfault.spring.cloud.lesson10.domain.User;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import java.util.Collection;
import java.util.Collections;
import java.util.List;
import java.util.Random;

/**
 * 用户服务提供方 Controller
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
@RestController
public class UserServiceProviderController implements UserService {

    @Autowired
    @Qualifier("inMemoryUserService") // 实现 Bean ： InMemoryUserService
    private UserService userService;

    private final static Random random = new Random();

    // 通过方法继承，URL 映射 ："/user/save"
    @Override
    public boolean saveUser(@RequestBody User user) {
        return userService.saveUser(user);
    }

    @HystrixCommand(
            commandProperties = { // Command 配置
                    // 设置操作时间为 100 毫秒
                    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "100")
            },
            fallbackMethod = "fallbackForGetUsers" // 设置 fallback 方法
    )
    // 通过方法继承，URL 映射 ："/user/find/all"
    @Override
    public List<User> findAll() {
        return userService.findAll();
    }

    /**
     * 获取所有用户列表
     *
     * @return
     */
    @HystrixCommand(
            commandProperties = { // Command 配置
                    // 设置操作时间为 100 毫秒
                    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "100")
            },
            fallbackMethod = "fallbackForGetUsers" // 设置 fallback 方法
    )
    @GetMapping("/user/list")
    public List<User> getUsers() throws InterruptedException {

        long executeTime = random.nextInt(200);

        // 通过休眠来模拟执行时间
        System.out.println("Execute Time : " + executeTime + " ms");

        Thread.sleep(executeTime);

        return userService.findAll();
    }

    /**
     * {@link #getUsers()} 的 fallback 方法
     *
     * @return 空集合
     */
    public List<User> fallbackForGetUsers() {
        return Collections.emptyList();
    }

}
```



### 整合服务发现：Netflix Eureka



#### 创建 Eureka Server



##### `pom.xml` 增加 Eureka Server 依赖

```xml
    <dependencies>

        <!-- Eureka Server 依赖 -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka-server</artifactId>
        </dependency>

    </dependencies>
```



##### 创建引导类：`EurekaServerApplication`

```java
package com.segumentfault.spring.cloud.lesson10.eureka.server;

/**
 * Eureka Server 引导类
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 1.0.0
 */

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }

}
```



##### 配置 Eureka Server

`application.properties`

```properties
## Spring Cloud Eureka 服务器应用名称
spring.application.name = eureka-server

## Spring Cloud Eureka 服务器服务端口
server.port = 10000

## 管理端口安全失效
management.security.enabled = false

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



> 端口信息
>
> ​	user-service-client : 8080
>
> ​	user-service-provider: 9090
>
> ​        eureka-server : 10000



#### 客户端：配置服务发现客户端



配置应用：`user-service-client`



##### `pom.xml` 增加 eureka-client 依赖

```xml
        <!-- 依赖 Spring Cloud Netflix Eureka -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka</artifactId>
        </dependency>
```



##### 激活服务发现客户端

`UserServiceClientApplication.java`

```java
package com.segumentfault.spring.cloud.lesson10.user.service.client;

import com.netflix.loadbalancer.IRule;
import com.segumentfault.spring.cloud.lesson10.api.UserService;
import com.segumentfault.spring.cloud.lesson10.user.service.client.rule.MyRule;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.feign.EnableFeignClients;
import org.springframework.cloud.netflix.ribbon.RibbonClient;
import org.springframework.context.annotation.Bean;
import org.springframework.web.client.RestTemplate;

/**
 * 引导类
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
@SpringBootApplication
@RibbonClient("user-service-provider") // 指定目标应用名称
@EnableCircuitBreaker // 使用服务短路
@EnableFeignClients(clients = UserService.class) // 申明 UserService 接口作为 Feign Client 调用
@EnableDiscoveryClient // 激活服务发现客户端
public class UserServiceClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServiceClientApplication.class, args);
    }

    /**
     * 将 {@link MyRule} 暴露成 {@link Bean}
     *
     * @return {@link MyRule}
     */
    @Bean
    public IRule myRule() {
        return new MyRule();
    }

    /**
     * 申明 具有负载均衡能力 {@link RestTemplate}
     *
     * @return
     */
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

}
```



##### 配置 Eureka 注册中心

`application.properties`

```properties
## 用户 Ribbon 客户端应用
spring.application.name = user-service-client

## 服务端口
server.port = 8080

## 提供方服务名称
provider.service.name = user-service-provider
## 提供方服务主机
provider.service.host = localhost
## 提供方服务端口
provider.service.port = 9090

## 激活 Eureka Client
eureka.client.enabled = true

## 扩展 IPing 实现
user-service-provider.ribbon.NFLoadBalancerPingClassName = \
  com.segumentfault.spring.cloud.lesson10.user.service.client.ping.MyPing

## 配置 @FeignClient(name = "${user.service.name}") 中的占位符
## user.service.name 实际需要制定 UserService 接口的提供方
## 也就是 user-service-provider，可以使用 ${provider.service.name} 替代
user.service.name = ${provider.service.name}

## Spring Cloud Eureka 客户端 注册到 Eureka 服务器
eureka.client.serviceUrl.defaultZone = http://localhost:10000/eureka
```



#### 服务端：配置服务发现客户端



配置应用：`user-service-provider`





##### `pom.xml` 增加 eureka-client 依赖

```xml
        <!-- 依赖 Spring Cloud Netflix Eureka -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka</artifactId>
        </dependency>
```



##### 激活服务发现客户端

`UserServiceProviderApplication.java`

```java
package com.segumentfault.spring.cloud.lesson10.user.service;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;

/**
 * 引导类
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
@SpringBootApplication
@EnableHystrix
@EnableDiscoveryClient // 激活服务发现客户端
public class UserServiceProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServiceProviderApplication.class, args);
    }
}
```



##### 配置 Eureka 注册中心

`application.properties`

```properties
## 用户服务提供方应用信息
spring.application.name = user-service-provider

## 服务端口
server.port = 9090

## Spring Cloud Eureka 客户端 注册到 Eureka 服务器
eureka.client.serviceUrl.defaultZone = http://localhost:10000/eureka
```





### 整合配置服务器：Config Server



#### 创建 Config Server



#### `pom.xml` 增加 Config Server 依赖

```xml
    <dependencies>

        <!-- 依赖 Spring Cloud Config Server -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-config-server</artifactId>
        </dependency>

    </dependencies>
```



#### 基于文件系统（File System）配置

> 注意：user-service-client  application.properties 中以下内容将会被配置服务器中的 user-service.properties 替代.

创建 `user-service.properties`

```properties
## 提供方服务名称
provider.service.name = user-service-provider
## 提供方服务主机
provider.service.host = localhost
## 提供方服务端口
provider.service.port = 9090
## 配置 @FeignClient(name = "${user.service.name}") 中的占位符
## user.service.name 实际需要制定 UserService 接口的提供方
## 也就是 user-service-provider，可以使用 ${provider.service.name} 替代
user.service.name = ${provider.service.name}
```



##### 初始化配置文件根路径

目前${user.dir} 指向: `/Users/Mercy/Downloads/spring-cloud-lesson-10/`

`user-service.properties` 相对于 `/config-server/src/main/resources/configs`

控制台执行 git 命令：

```
$ git init
Initialized empty Git repository in /Users/Mercy/Downloads/spring-cloud-lesson-10/config-server/src/main/resources/configs/.git/
$ git add user-service.properties
```



##### 设置配置文件根路径

`application.properties`

```properties
## Spring Cloud Config Server 应用名称
spring.application.name = config-server

## 服务器服务端口
server.port = 7070

## 管理端口安全失效
management.security.enabled = false

## Spring Cloud Eureka 客户端 注册到 Eureka 服务器
eureka.client.serviceUrl.defaultZone = http://localhost:10000/eureka

## 配置服务器文件系统git 仓库
## ${user.dir} 减少平台文件系统的不一致
## 目前 ${user.dir}/config-server/src/main/resources/configs
spring.cloud.config.server.git.uri = ${user.dir}/config-server/src/main/resources/configs
```



##### 激活服务发现客户端

`pom.xml`

```xml
        <!-- 依赖 Spring Cloud Netflix Eureka -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-eureka</artifactId>
        </dependency>
```



`application.properties`

```properties
## Spring Cloud Eureka 客户端 注册到 Eureka 服务器
eureka.client.serviceUrl.defaultZone = http://localhost:10000/eureka
```



激活服务发现

```java
package com.segumentfault.spring.cloud.lesson10.config.server;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.config.server.EnableConfigServer;

/**
 * 配置服务器应用
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 1.0.0
 */
@EnableConfigServer
@EnableDiscoveryClient
@SpringBootApplication
public class ConfigServerApplication {

    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```





### 整合配置客户端：Config Client



调整应用 `user-service-client` ，作为 config-client 应用



#### pom.xml 增加 config client 依赖

```xml
        <!-- 依赖 Spring Cloud Config Client -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-config</artifactId>
        </dependency>
```



#### 创建并且配置 `bootstrap.properties` 文件

`bootstrap.properties`

```properties
## 用户 Ribbon 客户端应用
spring.application.name = user-service-client

## 配置客户端应用关联的应用
## spring.cloud.config.name 是可选的
## 如果没有配置，采用 ${spring.application.name}
spring.cloud.config.name = user-service
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



`application.properties 也有调整`

```properties
## 服务端口
server.port = 8080

## 扩展 IPing 实现
user-service-provider.ribbon.NFLoadBalancerPingClassName = \
  com.segumentfault.spring.cloud.lesson10.user.service.client.ping.MyPing

## 以下内容有 Config Server 提供
### 提供方服务名称
#provider.service.name = user-service-provider
### 提供方服务主机
#provider.service.host = localhost
### 提供方服务端口
#provider.service.port = 9090
### 配置 @FeignClient(name = "${user.service.name}") 中的占位符
### user.service.name 实际需要制定 UserService 接口的提供方
### 也就是 user-service-provider，可以使用 ${provider.service.name} 替代
#user.service.name = ${provider.service.name}
```







