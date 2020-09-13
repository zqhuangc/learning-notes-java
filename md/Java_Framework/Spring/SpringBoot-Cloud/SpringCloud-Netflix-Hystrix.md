# Spring Cloud Netflix Hystrix



## 服务短路（CircuitBreaker）

Hystrix：Latency and Fault Tolerance for Distributed Systems

**martinfowler**
水桶效应，最小极限QPS，是否能降级（容忍服务不可用）

扩容  -> 数据源问题

QPS:Query Per Second，经过全链路压测，计算单机极限 QPS，（约等）集群 = 单机 QPS * 集群机器数量 * **可靠性比率**

全链路压测 ，除了压 极限QPS，还有错误数量
全链路：一个完整的业务流程控制

Jmeter：可调整型，比较灵活

TPS:Transaction per second

reactor streams，reactivex，comple*

> Reactive Java 框架：
>
> - Java9 Flow API
> - Reactor
> - RxJava（Reactive X）

## Spring Cloud Hystrix Client

官网：github托管

command，run
并发，达到设定阈值（进行短路处理）

### 激活 Hystrix

#### Annotation 方式

* @EnableHystrix，没有一些 Spring Cloud 功能
* 激活熔断保护 ：@EnableCircuitBreaker ：@EnableHystrix +   Spring Cloud 功能

> @HystrixCommand(
>
> fallbackMethod="methodName",
>
> commandProperties{
>
> @HystrixProperty(name = "",value="")
>
> }
>
> ) //调用超时或失败，替代返回
>
>  
>
> javanica
>
> 配置信息 wiki：
>
> https://github.com/Netflix/Hystrix/wiki/Configuration



#### 编程方式

用一个类继承 HystrixCommand，重写 run()，自定义一个方法来调用（超时或失败返回替代）方法
构造函数问题 HystrixCommandGroupKey.Factory.asKey("")，添加命令
通过execute() 执行



对比 Java 其他执行方式：

* RxJava

* Future



#### Health Endpoint

/health

#### Hystrix Endpoint（“/hystrix.stream”）

要使用 @EnableCircuitBreaker 

## Spring Cloud Hystrix Dashboard

### 激活

`@EnableHystrixDashboard`

/hystrix

 

## 整合 Netflix Turbine

生产准备特性：聚合数据指标 Turbine、Turbine Stream

spring-cloud-starter-turbine

@EnableTurbine

```properties
eureka.instance.metadata-map.management.port = ...
turbine.aggregator.clusterConfig = CUSTOMERS
turbine.appConfig = customers
```

/turbine.stream?cluster = CUSTOMERS



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

5.  spring cloud 的 config 配置，获取到的 git 中 properties 文件的一些属性（比如，my.name），可以直接在其它 spring  的 xml 中使用吗？需要怎么配置？
   利用在启动类中加入注解 @ImportResource("xxx.xml")，xxx.xml 文件中引用方式相同 "${my.name}"

6. Spring Boot中用 
   `new SpringApplicationBuilder.sources(AppConfig.class) ` 方式启动，是先加载`AppConfig`还是先加载配置文件？
   AppConfig 是一个配置 @Configuration Class，那么配置文件是一个外部资源，其实不会相互影响，如果 AppConfig 增加了 `@PropertySource` 或者  `@PropertySources`  的话，会优先加载  @PropertySource 中的配置资源。

   @Repeatable

















