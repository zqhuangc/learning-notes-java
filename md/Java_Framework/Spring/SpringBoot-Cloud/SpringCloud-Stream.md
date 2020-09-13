# Spring Cloud Stream（上）

* kafka
* spring kafka
* spring boot kafka
* spring cloud stream（编程模型）
* spring cloud stream kafka binder

### Kafka

官方网页：http://kafka.apache.org

* 主要用途
  * 消息中间件
  * 流式计算处理
  * 日志

#### 同类产品比较

* ActiveMQ：JMS（Java Message Service）规范
* RabbitMQ：AMQP（Advanced Message Queue Protocol）规范实现
* Kafaka：并非某种规范实现，它灵活和性能相对是优势

执行脚本目录在 /bin

windows在其单独目录

### 官网-快速上手

以 Windows 为例，.bat 相当于 .sh

1. 启动` zookeeper`

> 复制zoo_sample.cfg， 为 zoo.cfg
>
> bin/zkServer.cmd

2. 启动 `Kafka`

   > bin/windows/kafka-server-start.bat

3.  ...

offset

#### Kafka API方式

properties

searializer

KafkaProducer#send  异步

ProducerRecord



### Spring Kafka

[官方文档](https://docs.spring.io/spring-kafka/docs/2.2.2.RELEASE/reference/html/)

Spring 社区对 data(`spring-data`) 操作，有一个基本模式，Template 模式：

* JDBC:`JdbcTemplate `
* Redis：`Redis`Template 
* Kafka：`KafkaTemplate`
* JMS：`JmsTemplate `
* Rest：Rest`Template `

> XXXTemplate 一定实现 XXXOperations
>
> 如 `KafkaTemplate`实现`KafkaOperations`

##### Maven 依赖

```properties
 <!-- 整合 Kafka -->
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>
```



### Spring Boot Kafka

Maven 依赖

```properties

```



自动装配器：`KafkaAutoConfiguration`

其中 `kafkatemplate`会被自动装配

```properties
## 具体配置类 kafkaProperties
### Spring Kafka 配置信息
### 全局配置
spring.kafka.boostrapServers = localhost:9092
kafka.topic = melody
### kafka 生产者，找具体实现
spring.kafka.producer.keySerializer = xxx
spring.kafka.producer.valueSerializer = xxx
### kafka 消费者
spring.kafka.comsumer.groupId = melody-1
spring.kafka.comsumer.keyDeserializer = xxx
spring.kafka.comsumer.keyDeserializer = xxx
```



##### 生产者

@Value("${kafka.topic}")

`KafkaTemplate#send`

##### 消费者

`@KafkaListener`



## 问答

1. 当使用 Future，异步调用都可以使用 get() 方式强制执行，get() 是等待当前线程执行完毕，并且获取返回接口

2. `@KafkaListener` 和 `KafkaConsumer` 没有实质区别，主要是编程模式：
   `KafkaConsumer` API 采用接口编程

   `@KafkaListener`采用注解驱动

3. 在生产环境配置多个生产者和消费者只需要定义不同的 group 就可以了吗？
   group 是一种，要看是不是相同 Topic

4. 为了不丢失数据，消息队列的容错，和排错后的处理，如何实现的？

   依赖于 zookeeper

5. kafka 适用场景，高性能 Stream 处理

6. kafka 消息不会一直存在

7. broker 不需要设置，单独启动

8. comsumer 为什么要分组？

   comsumer 需要定义不同逻辑分组，以便于管理？？？







# Spring Cloud Stream(下)（编程模型）

### [spring-integration](https://docs.spring.io/spring-integration/docs/5.1.1.RELEASE/reference/html/)

RabbitMQ：AMQP，JMS 规范

Kafka： 相对松散的消息队列协议

> Reactive Stream:
>
> * Publisher
> * Subscriber
> * Processor

## 基本概念

#### Source：来源，近义：Producer、Publisher

####  Sink：接收器，近义：Consumer、Subscriber

#### Processor：对于上流而言是 Sink，对于下流而言是 Source

《企业整合模式》 [Enterprise Integration Patterns](http://www.eaipatterns.com/)

> jps、jconsole

## Spring Cloud Stream Binder：[Kafka](https://segmentfault.com/l/1500000011386642)

#### 启动 zookeeper

#### 启动 kafka-server

#### 发送者

>@EnableBinding(Source.class)  // 代理 Source
>
>@Qualifier("xxx") // Bean 名称，接口有多个实现类，定位
>
>MessageChannel#send、MessageBuilder#withPayload
>
>Source#output#send
>
>@Output、@Input

>消息大致分为两个部分：
>
>* 消息头（Headers）
>* 消息体（Body/Payload）



```properties
### 定义 Spring Cloud Stream Source 消息去向
spring.cloud.stream.binding.output.destination = ${kafka.topic}
### 自定义 Source 
### spring.cloud.stream.binding.${output-name}.destination = ${kafka.topic}
```



#### 定义标准消息发送源



### 消费者

> @EnableBinding(Sink.class)  // 代理 Sink
>
> @Qualifier("xxx") // Bean 名称，接口有多个实现类，定位
>
> SubscribableChannel#subscribe、MessageHandler#handleMessage
>
> Message#getPayload
>
> @Input
>
> @PostConstruct // 当字段注入完成后的回调



```properties
### 定义 Spring Cloud Stream Source 
spring.cloud.stream.binding.input.destination = ${kafka.topic}
```



1. 实现标准 Sink 监听

2. 通过 SubscribableChannel 订阅消息

3. 通过 @ServiceActivator(inputChannel = Sink.INPUT) // 类似@KafkaListener
  订阅消息

4. 通过 @StreamListener(inputChannel = Sink.INPUT) 订阅消息

  > 上面3个订阅顺序，获取顺序 4 -> 3 -> 2 



## Spring Cloud Stream Binder：[RabbitMQ]()

>  spring-cloud-stream-binder-rabbitmq

API方式：

@RabbitMQListener



## 官方文档：Binder 

## 问答

* `@EnableBinding` 有什么用？
  答：`@EnableBinding`将` Source`、`Sink `以及 `Processor `提升成相应的代理
* @Autowired Source source 这种写法是默认用官方的实现

* 若消息中间件出问题，维护要怎么样？
  消息中间件无法保证不丢消息，多数高一致性的消息背后还是有持久化的

* @EnableBinder，@EnableZuulProxy，@EnableDiscoverClient 这些注解都是通过特定 BeanPostProcessor 实现的吗？

  答：不完全对，主要处理接口在` @Import`：

  * `ImportSelector `实现类
  * `ImportBeanDefinitionRegistrar `实现类
  * `@Configuration` 标注类
  * `BeanPostProcessor `实现类

* 流式处理是什么，一般应用在什么场景？

  答：Stream 处理简单地说，异步处理，消息是一种处理方式。
  提交申请，机器生成，对于高密度提交服务，多数场景采用异步处理，Stream、Event-Driven。举例说明：审核流程，鉴别黄图。

* 大量消息怎么快速消费，多线程？
  答：确实是使用多线程，不过不一定奏效。依赖于处理具体内容，比如：一个线程使用了 25% CPU，四个线程就将 CPU 耗尽。因此，并发 100 个处理，实际上 ，还是 4 个线程在处理。
  大多数是多线程，其实也是单线程，流式非阻塞







### [Spring Cloud 消息驱动整合]()


