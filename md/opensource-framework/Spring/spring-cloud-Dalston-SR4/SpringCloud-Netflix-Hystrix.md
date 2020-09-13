* * 

# 预备知识

**martinfowler**
水桶效应，最小极限QPS，是否能降级（容忍服务不可用）

扩容  -> 数据源问题

QPS:Query Per Second，经过全链路压测，计算单机极限 QPS，（约等）集群 = 单机 QPS * 集群机器数量  **可靠性比率**

全链路压测 ，除了压 极限QPS，还有错误数量
全链路：一个完整的业务流程控制

Jmeter：可调整型，比较灵活

TPS:Transaction per second

reactor streams，reactivex，complete*

> Reactive Java 框架：
>
> - Java9 Flow API
> - Reactor
> - RxJava（Reactive X）

TSDB（时序数据库）   metric

## Spring Cloud Hystrix Client

官网：github托管

command，run
并发，调用失败达到设定阈值（进行短路处理）

### 激活 Hystrix

#### Annotation 方式

* 激活熔断保护 ：@EnableCircuitBreaker（通用） 或@EnableHystrix 

> @HystrixCommand(  fallbackMethod="methodName", commandProperties{  @HystrixProperty(name = "",value="")
>
> }) //调用超时或失败，替代返回
>
>
>
> javanica
>



#### 编码方式

用一个类继承 HystrixCommand，重写 run()，自定义一个方法来调用（超时或失败返回替代）方法

构造函数问题 HystrixCommandGroupKey.Factory.asKey("")，添加命令
通过execute() 执行



对比 Java 其他执行方式：

* RxJava

* Future





## 问答

1. ribbon 主要用于客户端负载均衡

2. kafka 与 Active mq？ActiveMQ 相对比较完善的中间件，kafak在能力上比较灵活，它放弃不少约束，性能相对比较好

3. 基本经典书籍要读

4. 注释{@link }，JavaDoc 一部分，通过 Java 注释生成 HTML。

   {@link }  引用到某一个类，比如{@link String}
   @since 从哪个版本开始
   @version 表示当前版本
   @author 作者
   <code></code> 里面嵌入 Java 代码

5. spring cloud 的 config 配置，获取到的 git 中 properties 文件的一些属性（比如，my.name），可以直接在其它 spring  的 xml 中使用吗？需要怎么配置？
   利用在启动类中加入注解 @ImportResource("xxx.xml")，xxx.xml 文件中引用方式相同 "${my.name}"

6. Spring Boot中用 
   `new SpringApplicationBuilder.sources(AppConfig.class) ` 方式启动，是先加载`AppConfig`还是先加载配置文件？
   AppConfig 是一个配置 @Configuration Class，那么配置文件是一个外部资源，其实不会相互影响，如果 AppConfig 增加了 `@PropertySource` 或者  `@PropertySources`  的话，会优先加载  @PropertySource 中的配置资源。

   @Repeatable







# Spring Cloud 服务短路（CircuitBreaker）

服务短路
• 名词由来
•https://martinfowler.com/bliki/CircuitBreaker.html
• 目的：系统整体性保护

熔断：错误达到一定比率断开，半开尝试，无误可重新访问



服务短路
• 近义词
• 服务容错（Fault tolerance）：强调容忍错误，不至于整体故障
• 服务降级（downgrade）：强调服务非强依赖，不影响主要流程



## Netflix Hystrix（覆盖复杂）

•Hystrix: Latency and Fault Tolerance for Distributed Systems
•https://github.com/Netflix/Hystrix
• Maven 依赖
•com.netflix.hystrix:hystrix-core



配置信息 wiki：

https://github.com/Netflix/Hystrix/wiki/Configuration



基于线程 异步触发

### 整合 Spring

• 激活：@EnableHystrix
• 编程模型
• 注解方式：@HystrixCommand
• 编程方式：HystrixCommand


### 整合 Spring Cloud
• 激活：@EnableCircuitBreaker
• 依赖：org.springframework.cloud:spring-cloud-starter-hystrix
• 端点
• Endpoint：/hystrix.stream



### 生产准备特性
Netflix Hystrix Dashboard
• 整合 Spring Cloud
• 激活：@EnableHystrixDashboard
• 依赖：org.springframework.cloud:spring-cloud-starter-hystrix-dashboard
• 收集



## 传统 Spring Web MVC



### 以 web 工程为例

#### 创建  DemoRestController:

```java
package com.segmentfault.spring.cloud.lesson8.web.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Random;
import java.util.concurrent.TimeoutException;

/**
 * Demo RestController
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
@RestController
public class DemoRestController {

    private final static Random random = new Random();

    /**
     * 当方法执行时间超过 100 ms 时，触发异常
     *
     * @return
     */
    @GetMapping("")
    public String index() throws Exception {

        long executeTime = random.nextInt(200);

        if (executeTime > 100) { // 执行时间超过了 100 ms
            throw new TimeoutException("Execution is timeout!");
        }

        return "Hello,World";
    }

}
```



#### 异常处理

##### 通过`@RestControllerAdvice` 实现

```java
package com.segmentfault.spring.cloud.lesson8.web.controller;

import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.concurrent.TimeoutException;

/**
 * {@link DemoRestController} 类似于AOP 拦截
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
@RestControllerAdvice(assignableTypes = DemoRestController.class)
public class DemoRestControllerAdvice {

    @ExceptionHandler(TimeoutException.class)
    public Object faultToleranceTimeout(Throwable throwable) {
        return throwable.getMessage();
    }
}
```



## Spring Cloud Netflix Hystrix



### 增加Maven依赖

```xml
	<dependencyManagement>
		<dependencies>

			<!-- Spring Boot 依赖 -->
			<dependency>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-dependencies</artifactId>
				<version>1.5.8.RELEASE</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>

			<!-- Spring Cloud 依赖 -->
			<dependency>
				<groupId>org.springframework.cloud</groupId>
				<artifactId>spring-cloud-dependencies</artifactId>
				<version>Dalston.SR4</version>
				<type>pom</type>
				<scope>import</scope>
			</dependency>

		</dependencies>
	</dependencyManagement>

    <dependencies>

        <!-- 其他依赖省略 -->

        <!-- 依赖 Spring Cloud Netflix Hystrix -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-hystrix</artifactId>
        </dependency>

    </dependencies>

```



### 使用`@EnableHystrix` 实现服务提供方短路

修改应用 `user-service-provider` 的引导类：

```java
package com.segumentfault.spring.cloud.lesson8.user.service;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;

/**
 * 引导类
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
@SpringBootApplication
@EnableHystrix
public class UserServiceProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServiceProviderApplication.class, args);
    }
}
```



#### 通过`@HystrixCommand`实现

增加 `getUsers()` 方法到 `UserServiceProviderController`：

```java
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
```



为 `getUsers()` 添加 fallback 方法：

```java
    /**
     * {@link #getUsers()} 的 fallback 方法
     *
     * @return 空集合
     */
    public Collection<User> fallbackForGetUsers() {
        return Collections.emptyList();
    }
```



### 使用`@EnableCircuitBreaker` 实现服务调用方短路

调整 `user-ribbon-client` ，为`UserRibbonController` 增加获取用户列表，实际调用`user-service-provider` "/user/list" REST 接口



#### 增加具备负载均衡 `RestTemplate`

在`UserRibbonClientApplication` 增加 `RestTemplate` 申明

```java
    /**
     * 申明 具有负载均衡能力 {@link RestTemplate}
     * @return
     */
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
```



#### 实现服务调用

```java
    /**
     * 调用 user-service-provider "/user/list" REST 接口，并且直接返回内容
     * 增加 短路功能
     */
    @GetMapping("/user-service-provider/user/list")
    public Collection<User> getUsersList() {
        return restTemplate.getForObject("http://" + providerServiceName + "/user/list", Collection.class);
    }
```



#### 激活 `@EnableCircuitBreaker`

调整 `UserRibbonClientApplication`:

```java
package com.segumentfault.spring.cloud.lesson8.user.ribbon.client;

import com.netflix.loadbalancer.IRule;
import com.segumentfault.spring.cloud.lesson8.user.ribbon.client.rule.MyRule;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
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
public class UserRibbonClientApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserRibbonClientApplication.class, args);
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
     * @return
     */
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

}
```



#### 增加编程方式的短路实现

```java
package com.segumentfault.spring.cloud.lesson8.user.ribbon.client.hystrix;

import com.netflix.hystrix.HystrixCommand;
import com.netflix.hystrix.HystrixCommandGroupKey;
import org.springframework.web.client.RestTemplate;

import java.util.Collection;
import java.util.Collections;

/**
 * User Ribbon Client HystrixCommand
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
public class UserRibbonClientHystrixCommand extends HystrixCommand<Collection> {

    private final String providerServiceName;

    private final RestTemplate restTemplate;

    public UserRibbonClientHystrixCommand(String providerServiceName, RestTemplate restTemplate) {
        super(HystrixCommandGroupKey.Factory.asKey(
                "User-Ribbon-Client"),
                100);
        this.providerServiceName = providerServiceName;
        this.restTemplate = restTemplate;
    }

    /**
     * 主逻辑实现
     *
     * @return
     * @throws Exception
     */
    @Override
    protected Collection run() throws Exception {
        return restTemplate.getForObject("http://" + providerServiceName + "/user/list", Collection.class);
    }

    /**
     * Fallback 实现
     *
     * @return 空集合
     */
    protected Collection getFallback() {
        return Collections.emptyList();
    }

}
```



#### 改造 `UserRibbonController#getUsersList()` 方法

```java
package com.segumentfault.spring.cloud.lesson8.user.ribbon.client.web.controller;

import com.segumentfault.spring.cloud.lesson8.domain.User;
import com.segumentfault.spring.cloud.lesson8.user.ribbon.client.hystrix.UserRibbonClientHystrixCommand;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.loadbalancer.LoadBalancerClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.client.RestTemplate;

import javax.annotation.PostConstruct;
import java.io.IOException;
import java.util.Collection;

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

    @Autowired
    private RestTemplate restTemplate;

    private UserRibbonClientHystrixCommand hystrixCommand;

    /**
     * 调用 user-service-provider "/user/list" REST 接口，并且直接返回内容
     * 增加 短路功能
     */
    @GetMapping("/user-service-provider/user/list")
    public Collection<User> getUsersList() {
        return new UserRibbonClientHystrixCommand(providerServiceName, restTemplate).execute();//每次执行都需要新的对象
    }
}
```



## 为生产为准备

技术独立性，不局限于springcloud，如springboot，springmvc都可

题外：@exceptionHandler   404   500    basicErrorController

### Netflix Hystrix Dashboard

`@EnableHystrixDashboard`

/hystrix

Hystrix Endpoint（“/hystrix.stream”）

metrics 指标信息

#### 整合 Netflix Turbine

生产准备特性：聚合数据指标 Turbine、Turbine Stream

spring-cloud-starter-turbine

@EnableTurbine

```properties
eureka.instance.metadata-map.management.port = ...
turbine.aggregator.clusterConfig = CUSTOMERS
turbine.appConfig = customers
```

/turbine.stream?cluster = CUSTOMERS



#### 创建 hystrix-dashboard 工程



#### 增加Maven 依赖

```xml
    <dependencies>

        <!-- 依赖 Hystrix Dashboard -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-hystrix-dashboard</artifactId>
        </dependency>

    </dependencies>
```



#### 增加引导类

```java
package com.segumentfault.spring.cloud.lesson8.hystrix.dashboard;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.hystrix.dashboard.EnableHystrixDashboard;

/**
 * Hystrix Dashboard 引导类
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
@SpringBootApplication
@EnableHystrixDashboard
public class HystrixDashboardApplication {

    public static void main(String[] args) {
        SpringApplication.run(HystrixDashboardApplication.class, args);
    }
    
}
```



#### 增加 application.properties

```properties
## Hystrix Dashboard 应用
spring.application.name = hystrix-dashboard

## 服务端口
server.port = 10000
```



# Spring Cloud Hystrix 源码分析

**ImportSelector  since spring3.1**

ribbon 只用于客户端 ， hystrix 都可以



## Spring Boot 自动装配

• Spring 模块装配
• 面向切面编程：AspectJ
• Reactive 编程：RxJava
• Java 并发编程：Java Utility Concurrent



## Spring Cloud Hystrix 源码解读



### `@EnableCircuitBreaker`

职责：

- 激活 Circuit Breaker（服务短路能力）
- 自动装配 HystrixCircuitBreakerConfiguration
- 导入选择器：EnableCircuitBreakerImportSelector
- 设计缺陷：覆盖默认实现 Hystrix 操作繁琐



#### 初始化顺序

- `@EnableCircuitBreaker `
  - `EnableCircuitBreakerImportSelector`
    - `HystrixCircuitBreakerConfiguration`



### HystrixCircuitBreakerConfiguration

主要职责：

* 自动装配 Hystrix 组件
* Hystrix 命令切面：HystrixCommandAspect
* Hystrix Endpoint：HystrixStreamEndpoint
* Hystrix 指标：HystrixMetricsPollerConfiguration

#### 初始化组件

- `HystrixCommandAspect`
- `HystrixShutdownHook`
- `HystrixStreamEndpoint` ： Servlet 
- `HystrixMetricsPollerConfiguration`



怎么覆盖切换，怎么去掉spring cloud 中的类定义

### SpringFactoryImportSelector

主要职责：

* 选择 /META-INF/spring.factories 中注解类型（泛型）所配置的 Spring Configuration 类
* 实现
  * EnableCircuitBreakerImportSelector
  * EnableDiscoveryClientImportSelector



### HystrixCommandAspect

主要职责：

* 拦截标注 @HystrixCommand 或 @HystrixCollapser 的方法（@Around）
* 生成拦截方法原信息（MetaHolderFactory）
* 生成 HystrixInvokable（HystrixCommandFactory）
* 选择执行模式（Observable 或非 Observable）



## Netflix Hystrix 源码解读

### `HystrixCommandAspect`

#### 依赖组件

- `MetaHolderFactory`
- `HystrixCommandFactory`: 生成`HystrixInvokable`
- `HystrixInvokable`
  - `CommandCollapser`  Hystrix-request collapsing(请求合并)
  - `GenericObservableCommand`
  - `GenericCommand`



### Future 实现 服务熔断

```java
package com.segumentfault.springcloudlesson9.future;

import java.util.Random;
import java.util.concurrent.*;

/**
 * 通过 {@link Future} 实现 服务熔断
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 */
public class FutureCircuitBreakerDemo {

    public static void main(String[] args) throws InterruptedException, ExecutionException {

        // 初始化线程池
        ExecutorService executorService = Executors.newSingleThreadExecutor();

        RandomCommand command = new RandomCommand();

        Future<String> future = executorService.submit(command::run);

        String result = null;
        // 100 毫秒超时时间
        try {
            result = future.get(100, TimeUnit.MILLISECONDS);
        } catch (TimeoutException e) {
            // fallback 方法调用
            result = command.fallback();
        }

        System.out.println(result);

        executorService.shutdown();

    }

    /**
     * 随机对象
     */
    private static final Random random = new Random();

    /**
     * 随机事件执行命令
     */
    public static class RandomCommand implements Command<String> {


        @Override
        public String run() throws InterruptedException {

            long executeTime = random.nextInt(200);

            // 通过休眠来模拟执行时间
            System.out.println("Execute Time : " + executeTime + " ms");

            Thread.sleep(executeTime);

            return "Hello,World";
        }

        @Override
        public String fallback() {
            return "Fallback";
        }
    }


    public interface Command<T> {

        /**
         * 正常执行，并且返回结果
         *
         * @return
         */
        T run() throws Exception;

        /**
         * 错误时，返回容错结果
         *
         * @return
         */
        T fallback();

    }
}
```



如何动态修改 hystrix 设置？

## 观察者模式



## RxJava 基础

* RxJava

Reactive Extensions for the JVM – a library for composing asynchronous and event-based programs using observable sequences for the Java VM.

* 官网：http://reactivex.io/
* 相关技术
  * Reactive Streams JVM
  * Java 9 Flow API

### 单数据：Single



```java
 Single.just("Hello,World") // 仅能发布单个数据
        .subscribeOn(Schedulers.io()) // 在 I/O 线程执行
        .subscribe(RxJavaDemo::println) // 订阅并且消费数据
;
```



### 多数据：Observable



```java
List<Integer> values = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9);

Observable.from(values) //发布多个数据
        .subscribeOn(Schedulers.computation()) // 在 I/O 线程执行
        .subscribe(RxJavaDemo::println) // 订阅并且消费数据
;

// 等待线程执行完毕
Thread.sleep(100);
```



### 使用标准 Reactive 模式



```java
List<Integer> values = Arrays.asList(1, 2, 3);

Observable.from(values) //发布多个数据
        .subscribeOn(Schedulers.newThread()) // 在 newThread 线程执行
        .subscribe(value -> {

            if (value > 2)
                throw new IllegalStateException("数据不应许大于 2");

            //消费数据
            println("消费数据：" + value);

        }, e -> {
            // 当异常情况，中断执行
            println("发生异常 , " + e.getMessage());
        }, () -> {
            // 当整体流程完成时
            println("流程执行完成");
        })

;

// 等待线程执行完毕
Thread.sleep(100);
```





## Java 9 Flow API

```java
package concurrent.java9;

import java.util.concurrent.Flow;
import java.util.concurrent.SubmissionPublisher;

/**
 * {@link SubmissionPublisher}
 *
 * @author mercyblitz
 **/
public class SubmissionPublisherDemo {

    public static void main(String[] args) throws InterruptedException {

        try (SubmissionPublisher<Integer> publisher =
                     new SubmissionPublisher<>()) {

            //Publisher(100) => A -> B -> C => Done
            publisher.subscribe(new IntegerSubscriber("A"));
            publisher.subscribe(new IntegerSubscriber("B"));
            publisher.subscribe(new IntegerSubscriber("C"));

            // 提交数据到各个订阅器
            publisher.submit(100);

        }


        Thread.currentThread().join(1000L);

    }

    private static class IntegerSubscriber implements
            Flow.Subscriber<Integer> {

        private final String name;

        private Flow.Subscription subscription;

        private IntegerSubscriber(String name) {
            this.name = name;
        }

        @Override
        public void onSubscribe(Flow.Subscription subscription) {
            System.out.printf(
                    "Thread[%s] Current Subscriber[%s] " +
                            "subscribes subscription[%s]\n",
                    Thread.currentThread().getName(),
                    name,
                    subscription);
            this.subscription = subscription;
            subscription.request(1);
        }

        @Override
        public void onNext(Integer item) {
            System.out.printf(
                    "Thread[%s] Current Subscriber[%s] " +
                            "receives item[%d]\n",
                    Thread.currentThread().getName(),
                    name,
                    item);
            subscription.request(1);
        }

        @Override
        public void onError(Throwable throwable) {
            throwable.printStackTrace();
        }

        @Override
        public void onComplete() {
            System.out.printf(
                    "Thread[%s] Current Subscriber[%s] " +
                            "is completed!\n",
                    Thread.currentThread().getName(),
                    name);
        }

    }

}
```

















