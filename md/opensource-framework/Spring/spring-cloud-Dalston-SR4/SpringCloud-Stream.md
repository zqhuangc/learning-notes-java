#   Spring Cloud Stream（上）

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

其中 `kafkaTemplate`会被自动装配

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



异步 executorchannel     dispatch

directchannel 

AbstractBindingTargetFactory

哪里调用createinput



# Spring Cloud 消息驱动整合

## Spring Cloud Stream

### 使用场景

• 消息驱动的微服务应用
• 目的
• 简化编码
• 统一抽象



### 主要概念

• 应用模型
• Binder 抽象
• 持久化 发布/订阅支持
• 消费分组支持
• 分区支持



### 基本概念

+ Source：Stream 发送源
  - 近义词：Producer、Publisher
+ Sink：Stream 接收器
  - 近义词：Consumer、Subscriber、Processor



### 编程模型

• 激活 : @EnableBinding
•@Configuration
•@EnableIntegration

+ Source（Stream 发送源）
  - @Output
  - MessageChannel

+ Sink（Stream 接收器）
  - @Input
  - SubscribableChannel
  - @ServiceActivator
  - @StreamListener

## 整合 Kafka

• 依赖
•org.springframework.cloud:spring-cloud-stream-binder-kafka
• 配置
•Source : spring.cloud.stream.bindings.${source}.*
•Sink : spring.cloud.stream.bindings.${sink}.*



• 依赖
•org.springframework.cloud:spring-cloud-stream-binder-rabbit
• 配置
•Source : spring.cloud.stream.bindings.${source}.*
•Sink : spring.cloud.stream.bindings.${sink}.*

### 改造 user-service-client 消息发送源（Kafka 原生 API）



#### User 模型实现序列化接口



```java
package com.segumentfault.spring.cloud.lesson12.domain;

import java.io.Serializable;

/**
 * 用户模型
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 1.0.0
 */
public class User implements Serializable {

    private static final long serialVersionUID = -5688097732613347904L;

    /**
     * ID
     */
    private Long id;

    /**
     * 用户名称
     */
    private String name;

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }
}

```



#### 增加 kafka 依赖

```xml
        <!-- 整合 Kafka -->
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>
```



#### 利用 KafkaTemplate 实现消息发送



```java
package com.segumentfault.spring.cloud.lesson12.user.service.client.web.controller;

import com.segumentfault.spring.cloud.lesson12.api.UserService;
import com.segumentfault.spring.cloud.lesson12.domain.User;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.support.SendResult;
import org.springframework.util.concurrent.ListenableFuture;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

/**
 * {@link UserService} 客户端 {@link RestController}
 * <p>
 * 注意：官方建议 客户端和服务端不要同时实现 Feign 接口
 * 这里的代码只是一个说明，实际情况最好使用组合的方式，而不是继承
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 1.0.0
 */
@RestController
public class UserServiceClientController implements UserService {

    @Autowired
    private UserService userService;

    private final KafkaTemplate<String, Object> kafkaTemplate;

    @Autowired
    public UserServiceClientController(KafkaTemplate kafkaTemplate) {
        this.kafkaTemplate = kafkaTemplate;
    }

    @PostMapping("/user/save/message")
    public boolean saveUserByMessage(@RequestBody User user) {
        ListenableFuture<SendResult<String, Object>> future = kafkaTemplate.send("sf-users", 0, user);
        return future.isDone();
    }

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



#### 实现 Kafka 序列化器：Java 序列化协议



```java
package com.segumentfault.spring.cloud.lesson12.user.service.client.serializer;

import org.apache.kafka.common.serialization.Serializer;

import java.io.ByteArrayOutputStream;
import java.io.ObjectOutputStream;
import java.util.Map;

/**
 * Java 序列化协议
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
public class ObjectSerializer implements Serializer<Object> {


    @Override
    public void configure(Map<String, ?> configs, boolean isKey) {

    }

    @Override
    public byte[] serialize(String topic, Object object) {

        System.out.println("topic : " + topic + " , object : " + object);

        byte[] dataArray = null;

        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();

        try {
            ObjectOutputStream objectOutputStream = new ObjectOutputStream(outputStream);
            objectOutputStream.writeObject(object);

            dataArray = outputStream.toByteArray();

        } catch (Exception e) {
            throw new RuntimeException(e);
        }

        return dataArray;
    }

    @Override
    public void close() {

    }
}
```





## Spring Cloud Stream 整合



### 改造 user-service-provider 消息接收器（Sink）



#### 引入 spring-cloud-stream-binder-kafka

```xml
        <!-- 依赖 Spring Cloud Stream Binder Kafka -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-stream-binder-kafka</artifactId>
        </dependency>
```



#### 用户消息 Stream 接口定义



```java
package com.segumentfault.spring.cloud.lesson12.user.stream;

import com.segumentfault.spring.cloud.lesson12.domain.User;
import org.springframework.cloud.stream.annotation.Input;
import org.springframework.messaging.SubscribableChannel;

/**
 * {@link User 用户} 消息 Stream 接口定义
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
public interface UserMessage {

    String INPUT = "user-message";

    @Input(INPUT)
        // 管道名称
    SubscribableChannel input();

}
```



#### 激活用户消息 Stream 接口



```java
package com.segumentfault.spring.cloud.lesson12.user.service;

import com.segumentfault.spring.cloud.lesson12.user.stream.UserMessage;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.hystrix.EnableHystrix;
import org.springframework.cloud.stream.annotation.EnableBinding;

/**
 * 引导类
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 1.0.0
 */
@SpringBootApplication
@EnableHystrix
@EnableDiscoveryClient // 激活服务发现客户端
@EnableBinding(UserMessage.class) // 激活 Stream Binding 到 UserMessage
public class UserServiceProviderApplication {

    public static void main(String[] args) {
        SpringApplication.run(UserServiceProviderApplication.class, args);
    }
}
```



#### 配置 Kafka 以及 Stream Destination



```properties
## Spring Cloud Stream Binding 配置
### destination 指定 Kafka Topic
### userMessage 为输入管道名称
spring.cloud.stream.bindings.user-message.destination = sf-users

## Kafka 生产者配置

spring.kafka.BOOTSTRAP-SERVERS=localhost:9092
spring.kafka.consumer.group-id=sf-group
spring.kafka.consumer.clientId=user-service-provider
```



#### 添加 User 消息监听器



##### SubscribableChannel 实现

```java
package com.segumentfault.spring.cloud.lesson12.user.service.provider.service;

import com.segumentfault.spring.cloud.lesson12.api.UserService;
import com.segumentfault.spring.cloud.lesson12.domain.User;
import com.segumentfault.spring.cloud.lesson12.user.stream.UserMessage;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.messaging.SubscribableChannel;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import java.io.ByteArrayInputStream;
import java.io.ObjectInputStream;

/**
 * 用户 消息服务
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
@Service
public class UserMessageService {

    @Autowired
    private UserMessage userMessage;

    @Autowired
    @Qualifier("inMemoryUserService")
    private UserService userService;

    private void saveUser(byte[] data) {
        // message body 是字节流 byte[]
        ByteArrayInputStream inputStream = new ByteArrayInputStream(data);
        try {
            ObjectInputStream objectInputStream = new ObjectInputStream(inputStream);
            User user = (User) objectInputStream.readObject(); // 反序列化成 User 对象
            userService.saveUser(user);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
  
     @PostConstruct
    public void init() {

        SubscribableChannel subscribableChannel = userMessage.input();
        subscribableChannel.subscribe(message -> {
            System.out.println("Subscribe by SubscribableChannel");
            // message body 是字节流 byte[]
            byte[] body = (byte[]) message.getPayload();
            saveUser(body);

        });
    }

}
```



##### @ServiceActivator 实现

```java
    @ServiceActivator(inputChannel = INPUT)
    public void listen(byte[] data) {
        System.out.println("Subscribe by @ServiceActivator");
        saveUser(data);
    }
```



##### @StreamListener 实现



```java
    @StreamListener(INPUT)
    public void onMessage(byte[] data) {
        System.out.println("Subscribe by @StreamListener");
        saveUser(data);
    }
```

轮流接收



### 改造 user-service-client 消息发送源（ Stream Binder : Rabbit MQ）



#### 增加 spring-cloud-stream-binder-rabbitmq 依赖

```xml
        <!-- 整合 Spring Cloud Stream Binder Rabbit MQ -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
        </dependency>
```



#### 配置发送源管道



#### 添加用户消息接口

```java
package com.segumentfault.spring.cloud.lesson12.user.service.client.stream;

import org.springframework.cloud.stream.annotation.Output;
import org.springframework.messaging.MessageChannel;

/**
 * 用户消息(输出)
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
public interface UserMessage {

    @Output("user-message-out")
    MessageChannel output();

}
```



#### 激活用户消息接口

```java
package com.segumentfault.spring.cloud.lesson12.user.service.client;

import com.netflix.loadbalancer.IRule;
import com.segumentfault.spring.cloud.lesson12.api.UserService;
import com.segumentfault.spring.cloud.lesson12.user.service.client.rule.MyRule;
import com.segumentfault.spring.cloud.lesson12.user.service.client.stream.UserMessage;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
import org.springframework.cloud.netflix.feign.EnableFeignClients;
import org.springframework.cloud.netflix.ribbon.RibbonClient;
import org.springframework.cloud.stream.annotation.EnableBinding;
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
@EnableBinding(UserMessage.class)
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



#### 实现消息发送到 RabbitMQ

```java
    @PostMapping("/user/save/message/rabbit")
    public boolean saveUserByRabbitMessage(@RequestBody User user) throws JsonProcessingException {
        MessageChannel messageChannel = userMessage.output();
        // User 序列化成 JSON
        String payload = objectMapper.writeValueAsString(user);
        GenericMessage<String> message = new GenericMessage<String>(payload);
        // 发送消息
        return messageChannel.send(message);
    }
```



启动 Rabbit MQ



### 改造 user-service-provider 消息接收器（ Stream Binder : Rabbit MQ）



#### 替换依赖

```xml
        <!--&lt;!&ndash; 依赖 Spring Cloud Stream Binder Kafka &ndash;&gt;-->
        <!--<dependency>-->
            <!--<groupId>org.springframework.cloud</groupId>-->
            <!--<artifactId>spring-cloud-stream-binder-kafka</artifactId>-->
        <!--</dependency>-->

        <!-- 整合 Spring Cloud Stream Binder Rabbit MQ -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
        </dependency>
```







# Spring Cloud Stream Binder 实现





## JMS 实现 ActiveMQ



### 增加 Maven 依赖



```xml
        <!-- 整合 Sprig Boot Starter ActiveMQ -->
        <!-- 间接依赖：
            spring jms
            jms api
            activemq
            spring boot jms
        -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-activemq</artifactId>
        </dependency>
```



### 启动 ActiveMQ Broker



#### 安装

```
$ brew install apache-activemq
```



#### 启动

```
$ activemq console
```



### 原生API：生产消息



请注意启动后的控制台输出：

```
INFO | Listening for connections at: tcp://Mercy-MacBook-Pro.local:61616?maximumConnections=1000&wireFormat.maxFrameSize=104857600
```



其中 `tcp://Mercy-MacBook-Pro.local:61616` 就是 broker URL，请注意将主机名转换成 localhost：

`tcp://localhost:61616`



```java
private static void sendMessage() throws Exception {
        // 创建 ActiveMQ 链接，设置 Broker URL
        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory("tcp://localhost:61616");
        // 创造 JMS 链接
        Connection connection = connectionFactory.createConnection();
        // 创建会话 Session
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        // 创建消息目的 - Queue 名称为 "TEST"
        Destination destination = session.createQueue("TEST");
        // 创建消息生产者
        MessageProducer producer = session.createProducer(destination);
        // 创建消息 - 文本消息
        ActiveMQTextMessage message = new ActiveMQTextMessage();
        message.setText("Hello,World");
        // 发送文本消息
        producer.send(message);

        // 关闭消息生产者
        producer.close();
        // 关闭会话
        session.close();
        // 关闭连接
        connection.close();
    }
```



### 原生API：消费消息



```java
    private static void receiveMessage() throws Exception {

        // 创建 ActiveMQ 链接，设置 Broker URL
        ConnectionFactory connectionFactory = new ActiveMQConnectionFactory("tcp://localhost:61616");
        // 创造 JMS 链接
        Connection connection = connectionFactory.createConnection();
        // 启动连接
        connection.start();
        // 创建会话 Session
        Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
        // 创建消息目的 - Queue 名称为 "TEST"
        Destination destination = session.createQueue("TEST");
        // 创建消息消费者
        MessageConsumer messageConsumer = session.createConsumer(destination);
        // 获取消息
        Message message = messageConsumer.receive(100);

        if (message instanceof TextMessage) {
            TextMessage textMessage = (TextMessage) message;
            System.out.println("消息消费内容：" + textMessage.getText());
        }

        // 关闭消息消费者
        messageConsumer.close();
        // 关闭会话
        session.close();
        // 关闭连接
        connection.stop();
        connection.close();
    }
```



## Spring Boot JMS + ActiveMQ

ActiveMQ : JMS 实现
• 消息生产者 : MessageProducer
• 消息消费者 : MessageConsumer
• 消息连接 : Connection
• 消息会话 : Session
• 消息目的：Destination



•Spring JMS
•JmsTemplate
•Spring Boot JMS
•Spring Boot ActiveMQ
• 自动装配：ActiveMQAutoConfiguration
• 属性配置：ActiveMQProperties

### Maven 依赖

```xml
        <!-- 整合 Sprig Boot Starter ActiveMQ -->
        <!-- 间接依赖：
            spring jms
            jms api
            activemq
            spring boot jms
        -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-activemq</artifactId>
        </dependency>
```



### 配置 ActiveMQ 属性

`application.properties`

```properties
## ActiveMQ 配置
spring.activemq.brokerUrl = tcp://localhost:61616
```



### 配置 JMS 属性

`application.properties`

```properties
## JMS 配置
spring.jms.template.defaultDestination = sf-users-activemq
```



### 改造 user-service-client：实现 ActiveMQ  User 对象消息生产

JmsTemplate dosend     trustpackage

`UserServiceClientController.java`

```java
    @Autowired
    private JmsTemplate jmsTemplate;

    @PostMapping("/user/save/message/activemq")
    public boolean saveUserByActiveMQMessage(@RequestBody User user) throws Exception {
        jmsTemplate.convertAndSend(user);
        return true;
    }
```



### 启动 user-service-client



预先启动 "eureka-server" 以及 "config-server"



### 改造 user-service-provider : 实现 ActiveMQ  User 对象消息消费



> 提示：重复 ActiveMQ 配置 以及 JMS 配置



```java
    @Autowired
    private JmsTemplate jmsTemplate;

    @GetMapping("/user/poll")
    public Object pollUser() {
        // 获取消息队列中，默认 destination = sf-users-activemq
        return jmsTemplate.receiveAndConvert();
    }
```







## ActiveMQ Spring Cloud Sream Binder 实现

• 实现 Binder SPI
• 实现 Binder 接口
• 生产者配置 / 扩展配置
• 消费者配置 / 扩展配置
• @Configuration 标记 Binder 实现类
• 绑定实现到 META-INF/spring.binders



参考已有 binder 实现，了解内部 channel 的流向，源与接收端，管道同/异步调用

![](https://raw.githubusercontent.com/spring-cloud/spring-cloud-stream/master/docs/src/main/asciidoc/images/producers-consumers.png)

### 创建 spring-cloud-stream-binder-activemq 工程



### 引入 Maven 依赖





### 实现 Binder 接口 - 仅实现消息发送

```java
package com.segumentfault.spring.cloud.stream.binder.activemq;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cloud.stream.binder.Binder;
import org.springframework.cloud.stream.binder.Binding;
import org.springframework.cloud.stream.binder.ConsumerProperties;
import org.springframework.cloud.stream.binder.ProducerProperties;
import org.springframework.jms.core.JmsTemplate;
import org.springframework.messaging.MessageChannel;
import org.springframework.messaging.SubscribableChannel;
import org.springframework.util.Assert;

/**
 * Active MQ MessageChannel Binder 实现
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
public class ActiveMQMessageChannelBinder implements
        Binder<MessageChannel, ConsumerProperties, ProducerProperties> {

    @Autowired
    private JmsTemplate jmsTemplate;

    /**
     * 接受 ActiveMQ 消息
     *
     * @param name
     * @param group
     * @param inboundBindTarget
     * @param consumerProperties
     * @return
     */
    @Override
    public Binding<MessageChannel> bindConsumer(String name, String group, MessageChannel inboundBindTarget, ConsumerProperties consumerProperties) {
        // TODO: 实现消息消费
        return () -> {
        };
    }

    /**
     * 负责发送消息到 ActiveMQ
     *
     * @param name
     * @param outputChannel
     * @param producerProperties
     * @return
     */
    @Override
    public Binding<MessageChannel> bindProducer(String name, MessageChannel outputChannel, ProducerProperties producerProperties) {
        Assert.isInstanceOf(SubscribableChannel.class, outputChannel,
                "Binding is supported only for SubscribableChannel instances");

        SubscribableChannel subscribableChannel = (SubscribableChannel) outputChannel;

        subscribableChannel.subscribe(message -> {
            // 接受内部管道消息，来自于 MessageChannel#send(Message)
            // 实际并没有发送消息，而是此消息将要发送到 ActiveMQ Broker
            Object messageBody = message.getPayload();
            jmsTemplate.convertAndSend(name, messageBody);

        });

        return () -> {
            System.out.println("Unbinding");
        };
    }
}

```



### 实现 Spring Cloud Stream Binder 自动装配



```java
package com.segumentfault.spring.cloud.stream.binder.activemq;

import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.cloud.stream.binder.Binder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

/**
 * ActiveMQ Stream Binder 自动装配
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 0.0.1
 */
@Configuration
@ConditionalOnMissingBean(Binder.class)
public class ActiveMQStreamBinderAutoConfiguration {

    @Bean
    public ActiveMQMessageChannelBinder activeMQMessageChannelBinder() {
        return new ActiveMQMessageChannelBinder();
    }
}

```



### 配置 META-INF/spring.binders



```properties
activemq :\
com.segumentfault.spring.cloud.stream.binder.activemq.ActiveMQStreamBinderAutoConfiguration
```



### 整合消息生产者 user-service-client



#### 引入 ActiveMQ Spring Cloud Stream Binder Maven 依赖



```xml
        <!-- 引入 Active MQ Spring Cloud Stream Binder 实现 -->
        <dependency>
            <groupId>com.segumentfault</groupId>
            <artifactId>spring-cloud-stream-binder-activemq</artifactId>
            <version>${project.version}</version>
        </dependency>
```



#### 配置 ActiveMQ Spring Cloud Stream Binder 属性



```properties
## Spring Cloud Stream 默认 Binder
spring.cloud.stream.defaultBinder=rabbit

### 消息管道 activemq-out 配置
spring.cloud.stream.bindings.activemq-out.binder = activemq
spring.cloud.stream.bindings.activemq-out.destination = sf-users-activemq
```



### 实现 Binder 接口 - 实现消息消费

```java
    @Override
    public Binding<MessageChannel> bindConsumer(String name, String group, MessageChannel inputChannel, ConsumerProperties consumerProperties) {

        ConnectionFactory connectionFactory = jmsTemplate.getConnectionFactory();
        try {
            // 创造 JMS 链接
            Connection connection = connectionFactory.createConnection();
            // 启动连接
            connection.start();
            // 创建会话 Session
            Session session = connection.createSession(false, Session.AUTO_ACKNOWLEDGE);
            // 创建消息目的
            Destination destination = session.createQueue(name);
            // 创建消息消费者
            MessageConsumer messageConsumer = session.createConsumer(destination);

            messageConsumer.setMessageListener(message -> {
                // message 来自于 ActiveMQ
                if (message instanceof ObjectMessage) {
                    ObjectMessage objectMessage = (ObjectMessage) message;
                    try {
                        Object object = objectMessage.getObject();
                        inputChannel.send(new GenericMessage<Object>(object));
                    } catch (JMSException e) {
                        e.printStackTrace();
                    }
                }
            });
        } catch (JMSException e) {
            e.printStackTrace();
        }

        return () -> {
        };
    }
```





### 整合消息消费者 - user-service-provider



#### 引入 ActiveMQ Spring Cloud Stream Binder Maven 依赖



```xml
        <!-- 引入 Active MQ Spring Cloud Stream Binder 实现 -->
        <dependency>
            <groupId>com.segumentfault</groupId>
            <artifactId>spring-cloud-stream-binder-activemq</artifactId>
            <version>${project.version}</version>
        </dependency>
```



#### 配置 ActiveMQ Spring Cloud Stream Binder 属性



```properties
## Spring Cloud Stream 默认 Binder
spring.cloud.stream.defaultBinder=rabbit

### 消息管道 activemq-out 配置
spring.cloud.stream.bindings.activemq-in.binder = activemq
spring.cloud.stream.bindings.activemq-in.destination = sf-users-activemq
```



#### 实现 User 消息监听

```java
    @StreamListener("activemq-in")
    public void onUserMessage(User user) throws IOException {
        System.out.println("Subscribe by @StreamListener");
        userService.saveUser(user);
    }

    // 监听 ActiveMQ Stream
    userMessage.activeMQIn().subscribe(message -> {

        if (message instanceof GenericMessage) {
            GenericMessage genericMessage = (GenericMessage) message;
            User user = (User) genericMessage.getPayload();
            userService.saveUser(user);
        }
    });
```

