# JMS(Java Message Service)

指的是面向消息中间件（MOM），用于在两个应用程序之间，或分布式系统中发送消息，进行异步通信。  
JMS中定义了两种消息模型：点对点（point to point， queue）和发布/订阅（publish/subscribe，topic）。主要区别就是是否能重复消费。  
点对点：Queue，不可重复消费  
发布/订阅：Topic，可以重复消费(多消费者) 

**传统企业型消息队列ActiveMQ遵循了JMS规范**，实现了点对点和发布订阅模型，但其他流行的消息队列RabbitMQ、Kafka并没有遵循JMS规范。  

## ActiveMQ

## 安装ActiveMQ

1. 下载activeMq安装包

2. tar -zxvf **.tar.gz

3. sh bin/activemq start 启动activeMQ服务

 

## 什么是MOM

面向消息的中间件，使用消息传送提供者来协调消息传输操作。 MOM需要提供API和管理工具。 客户端调用api。 把消息发送到消息传送提供者指定的目的地

在消息发送之后，客户端会技术执行其他的工作。并且在接收方收到这个消息确认之前。提供者一直保留该消息

 

## JMS的概念和规范

 

![1553185897003](..\image\JMS.png)

 

# 消息传递域

## 点对点(p2p)

1. 每个消息只能有一个消费者(createConsumer)

2. 消息的生产者和消费者之间没有时间上的相关性。无论消费者在生产者发送消息的时候是否处于运行状态，都可以提取消息

## 发布订阅(pub/sub)

1. 每个消息可以有多个消费者

2. 消息的生产者和消费者之间存在时间上的相关性，订阅一个主题的消费者**只能消费自它订阅之后发布**的消息。**JMS规范允许提供客户端创建持久订阅**（createDurableSubscriber(topic)）clientid  消费端设置

## JMS API

ConnectionFactory  连接工厂

Connection            封装客户端与JMS provider之间的一个虚拟的连接

Session                    生产和消费消息的一个单线程上下文； 用于创建producer、consumer、message、queue..\

Destination              消息发送或者消息接收的目的地

MessageProducer/consumer 消息生产者/消费者

## 消息组成

### 消息头

包含消息的识别信息和路由信息

### 消息体

TextMessage

MapMessage

BytesMessage

StreamMessage   输入输出流

ObjectMessage  可序列化对象

### 属性

 XXXMessage.setxx

# JMS的可靠性机制

JMS消息之后被确认后，才会认为是被成功消费。消息的消费包含三个阶段： 客户端接收消息、客户端处理消息、消息被确认

## 事务性会话

![1553185997863](..\image\activemq-0.png)
  如上图，设置为true的时候，消息会在session.commit以后自动签收

## 非事务性会话

**Boolean.False**

在该模式下，消息何时被确认取决于创建会话时的应答模式

## 应答模式

### AUTO_ACKNOWLEDGE

当客户端成功从recive方法返回以后，或者[MessageListener.onMessage] 方法成功返回以后，会话会自动确认该消息

### CLIENT_ACKNOWLEDGE

```
客户端通过调用消息的textMessage.acknowledge();确认消息。
在这种模式中，如果一个消息消费者消费一共是10个消息，那么消费了5个消息，然后在第5个消息通过textMessage.acknowledge()，那么之前的所有消息都会被消确认
```

### DUPS_OK_ACKNOWLEDGE

延迟确认

 

## 本地事务

在一个JMS客户端，可以使用本地事务来组合消息的发送和接收。JMS Session 接口提供了commit和rollback方法。

JMS Provider会缓存每个生产者当前生产的所有消息，直到commit或者rollback，commit操作将会导致事务中所有的消息被持久存储；rollback意味着JMS Provider将会清除此事务下所有的消息记录。在事务未提交之前，消息是不会被持久化存储的，也不会被消费者消费

事务提交意味着生产的所有消息都被发送。消费的所有消息都被确认； 

事务回滚意味着生产的所有消息被销毁，消费的所有消息被恢复，也就是下次仍然能够接收到发送端的消息，除非消息已经过期了

 

## JMS （pub/sub）模型

1. 订阅可以分为非持久订阅和持久订阅

2. 当订阅之后所有的消息必须接收的时候，则需要用到持久订阅。反之，则用非持久订阅

## JMS  （P2P）模型

1. 如果session关闭时，有一些消息已经收到，但还没有被签收，那么当消费者下次连接到相同的队列时，消息还会被签收（重新消费?）

2. 如果用户在receive方法中设定了消息选择条件，那么不符合条件的消息会留在队列中不会被接收

3. 队列可以长久保存消息直到消息被消费者签收。消费者不需要担心因为消息丢失而时刻与jms provider保持连接状态

## Broker 

服务提供者实例

BrokerService#

useJMX,addConnection

KahaDB

# 答疑

## 消息的发送策略

持久化消息

默认情况下，生产者发送的消息是持久化的。消息发送到broker以后，producer会等待broker对这条消息的处理情况的反馈

可以设置消息发送端发送持久化消息的异步方式

ActiveMQConnectionFactory#setUseAsyncSend(**true**);

回执窗口大小设置    消息发送的没收到回执最大size？？

```
connectionFactory.setProducerWindowSize();
```

非持久化消息

```
textMessage.setJMSDeliveryMode(DeliveryMode.NON_PERSISTENCE);
```

非持久化消息模式下，默认就是异步发送过程，如果需要对非持久化消息的每次发送的消息都获得broker的回执的话

```
connectionFactory.setAlwaysSyncSend();
```

## consumer获取消息是pull还是（broker的主动 push）

默认情况下，mq服务器（broker）采用异步方式向客户端主动推送消息(push)。也就是说broker在向某个消费者会话推送消息后，不会等待消费者响应消息，直到消费者处理完消息以后，主动向broker返回处理结果



prefetchsize    “预取消息数量“

broker端一旦有消息，就主动按照默认设置的规则推送给当前活动的消费者。 每次推送都有一定的数量限制，而这个数量就是prefetchSize

Queue

持久化消息   prefetchSize=1000

非持久化消息 1000

topic

持久化消息        100

非持久化消息      32766

假如prefetchSize=0 . 此时对于consumer来说，就是一个pull模式

 

## 关于acknowledge为什么能够在第5次主动执行ack以后，把前面的消息都确认掉



ActiveMQSession#acknowledge

消表示已经被consumer接收但未确认的消息。

### 消息确认

ACK_TYPE，消费端和broker交换ack指令的时候，还需要告知broker  ACK_TYPE。 

ACK_TYPE 表示确认指令的类型，broker可以根据不同的ACK_TYPE去针对当前消息做不同的应对策略

 

REDELIVERED_ACK_TYPE (broker会重新发送该消息)  重发策略

DELIVERED_ACK_TYPE  消息已经接收，但是尚未处理结束

STANDARD_ACK_TYPE  表示消息处理成功

 

 

# ActiveMQ结合spring开发

Spring提供了对JMS的支持，需要添加Spring 支持JMS的包

## 添加jar依赖

```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jms</artifactId>
    <version>${spring.version}</version>
</dependency>   
<dependency>
    <groupId>org.apache.activemq</groupId>
    <artifactId>activemq-all</artifactId>
    <version>5.15.0</version>
</dependency>
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
    <version>2.4.2</version>
</dependency>
```



## 配置spring文件

```xml
<!--  spring-jms -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://code.alibabatech.com/schema/dubbo
        http://code.alibabatech.com/schema/dubbo/dubbo.xsd" default-autowire="byName">

    <bean id="connectionFactory" class="org.apache.activemq.pool.PooledConnectionFactory" destroy-method="stop">
        <property name="connectionFactory">
            <bean class="org.apache.activemq.ActiveMQConnectionFactory">
                <property name="brokerURL">
                    <value>tcp://192.168.11.129:61616</value>
                </property>
            </bean>
        </property>
        <property name="maxConnections" value="50"/>
    </bean>
    <bean id="destination" class="org.apache.activemq.command.ActiveMQQueue">
        <constructor-arg index="0" value="spring-queue"/>
    </bean>

    <bean id="destinationTopic" class="org.apache.activemq.command.ActiveMQTopic">
        <constructor-arg index="0" value="spring-topic"/>
    </bean>
    <bean id="jmsTemplate" class="org.springframework.jms.core.JmsTemplate">
        <property name="connectionFactory" ref="connectionFactory"/>
        <property name="defaultDestination" ref="destinationTopic"/>
        <property name="messageConverter">
            <bean class="org.springframework.jms.support.converter.SimpleMessageConverter"/>
        </property>
    </bean>
    
    <!-- 异步非阻塞 -->
    <bean id="jmsContainer" class="org.springframework.jms.listener.DefaultMessageListenerContainer">
        <property name="connectionFactory" ref="connectionFactory"/>
        <property name="destination" ref="destinationTopic"/>
        <property name="messageListener" ref="messageListener"/>
    </bean>

    <bean id="messageListener" class="com.gupao.vip.mic.dubbo.jms.FirstMessageListener"/>

</beans>
```



## 编写发送端代码

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://code.alibabatech.com/schema/dubbo
        http://code.alibabatech.com/schema/dubbo/dubbo.xsd" default-autowire="byName">

    <!--当前项目在整个分布式架构里面的唯一名称，计算依赖关系的标签-->
    <dubbo:application name="${application.name}" owner="${dubbo.application.owner}"/>
    <!--dubbo这个服务所要暴露的服务地址所对应的注册中心-->
    <dubbo:registry protocol="zookeeper"
                    address="${dubbo.zk.servers}"
                    group="${dubbo.zk.group}"
                    file="${dubbo.cache.dir}/dubbo-order.cache"/>

    <!--当前服务发布所依赖的协议；webserovice、Thrift、Hessain、http-->
    <dubbo:protocol
                    name="dubbo"
                    port="${dubbo.service.provider.port}"
                    threadpool="cached"
                    threads="${dubbo.service.provider.threads:200}"
                    accesslog="${dubbo.protocol.accesslog}"/>

 <!--   <import resource="classpath*:META-INF/cliuser-reference.xml.xml"/>-->

</beans>
```



## 配置接收端spring文件

直接copy发送端的文件

## 编写接收端代码



## spring的发布订阅模式配置

 

## 以事件通知方式来配置消费者

onMessage

###更改消费端的配置



### 增加FirstMessageListener监听类



### 启动spring容器



 

# ActiveMQ支持的传输协议

client端和broker端的通讯协议

**TCP**、UDP 、NIO、SSL、Http（s）、vm

```xml\
<transportConnectors>
            <!-- DOS protection, limit concurrent connections to 1000 and frame size to 100MB -->
            <transportConnector name="openwire" uri="tcp://0.0.0.0:61616?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
             <!-- 增加 nio 协议 -->
            <transportConnector name="nio" uri="nio://0.0.0.0:61618?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
            
            <transportConnector name="amqp" uri="amqp://0.0.0.0:5672?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
            
            <transportConnector name="stomp" uri="stomp://0.0.0.0:61613?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
            
            <transportConnector name="mqtt" uri="mqtt://0.0.0.0:1883?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
            
            <transportConnector name="ws" uri="ws://0.0.0.0:61614?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
        </transportConnectors>
```



# ActiveMQ持久化存储

![1553261975210](..\image\activemq-store.png)

 

\1.    kahaDB  默认的存储方式

```xml
<persistenceAdapter>
    <kahaDB directory="${activemq.data}/kahadb"/>
</persistenceAdapter>
```



\2.    AMQ 基于文件的存储方式

写入速度很快，容易恢复。

文件默认大小是32M

```xml
<persistenceAdapter>
    <!--5.3以前版本默认 -->
    <amqPersistenceAdapter directory="" maxFileLength="32m"/>
    <kahaDB directory="${activemq.data}/kahadb"/>
</persistenceAdapter>
```



\3.    JDBC 基于数据库的存储

ACTIVEMQ_ACKS ： 存储持久订阅的信息

ACTIVEMQ_LOCK ： 锁表（用来做集群的时候，实现master选举的表）

ACTIVEMQ_MSGS ： 消息表





## 第一步

```xml
<persistenceAdapter>
    <jdbcPersistenceAdapter dataSource="#myDataSource" createTableOnStart=""/>
</persistenceAdapter>

<bean id="myDataSource" class="xxx"/>
```



## 第二步

datasource  配置

## 第三步

添加jar包依赖

 

### JDBC Message store with activeMQ journal

1. 引入了快速缓存机制，缓存到Log文件中

2. 性能会比jdbc store要好

3. JDBC Message store with activeMQ journal 不能应用于master/slave模式

4. Memory 基于内存的存储

 

# LevelDB

5.8以后引入的持久化策略。通常用于集群配置

 

# ActiveMQ的网络连接

activeMQ如果要实现扩展性和高可用性的要求的话，就需要用用到网络连接模式

 

## NetworkConnector

主要用来配置broker与broker之间的通信连接



服务器S1（broker）和S2（broker）通过NewworkConnector相连，则生产者P1（连接 S1）发送消息，消费者C3和C4（连接 S2）都可以接收到，而生产者P3（连接 S2）发送的消息，消费者C1和C2（连接 S1）同样也可以接收到 

 

### 静态网络连接

修改activemq.xml，增加如下内容

```xml
<networkConnectors>
    <networkConnector uri="static://(tcp://0.0.0.0:61616,tcp://0.0.0.1:61616)"/>
</networkConnectors>
```



两个Brokers通过一个staic的协议来进行网络连接。一个Consumer连接到BrokerB的一个地址上，当Producer在BrokerA上以相同的地址发送消息是，此时消息会被转移到BrokerB上，也就是说BrokerA会转发消息到BrokerB上

## 丢失的消息-------消息回流

一些consumer连接到broker1、消费broker2上的消息。消息先被broker1从broker2消费掉，然后转发给这些consumers。假设，转发消息的时候broker1重启了，这些consumers发现brokers1连接失败，通过failover连接到broker2.但是因为有一部分没有消费的消息被broker2已经分发到broker1上去了，这些消息就好像消失了。除非有消费者重新连接到broker1上来消费

 failover:()

从5.6版本开始，在destinationPolicy上新增了一个选项replayWhenNoConsumers属性，这个属性可以用来解决当broker1上有需要转发的消息但是没有消费者时，把消息回流到它原始的broker。同时把enableAudit设置为false，为了防止消息回流后被当作重复消息而不被分发

通过如下配置，在activeMQ.xml中。 分别在两台服务器都配置。即可完成消息回流处理

```xml
<!-- 添加配置 -->
<policyEntry queue=">" enableAdit="false">
    <networkBridgeFilterFactory>
        <conditionalNetworkBridgeFilterFactory replayWhenNoConsumers="true"/>
    </networkBridgeFilterFactory>
</policyEntry>
```



### 动态网络连接

 vmware+ Centos 7 jdk8 

 

kafka -> redis -> nginx  -> mongoDB

 

# 回顾

1. activeMQ安装

2. activeMQ的应用场景

3. JMS的概念和模型

4. 通过JMS的api去实现了一个p2p的发送代码

5. JMS的消息结构组成：消息头、消息体、消息属性

6. JMS的域模型（点对点、pub/sub）

7. JMS的可靠性机制
   a)     事务型： session.commit 

   b)    非事务型： ack类型 ：AUTO_ACK / CLIENT_ACK /DUPS_ACK

8. 本地事务、消息的持久性

9. 轻量级的Broker。自己启动一个broker实例

10. spring+activeMQ整合

11. 持久化和非持久化发送策略
    默认持久化，如果是非持久化，需要设置deliverMode 
    如果在非事务模型下，使用持久化发送策略，该发送方式为同步

    主动设置当前的连接是同步

12. consumer消费消息是pull还是push （prefetchSize）

13. 传输协议（client-broker） tcp/nio/udp/http(s)/vm/ssl

14. 消息持久化策略
    a)     kahadb
    b)    AMQ
    c)     JDBC
    d)    内存
    e)     levelDB

15. activeMQ高性能策略(networkConnector)
    a)     静态网络连接  duplex=true  双向连接，或分别配 static 协议  ：消息回流
    b)    动态网络连接 

# 网络连接

#### 静态网络连接

 

#### 动态网络连接

multicast



networkConnector是一个高性能方案，并不是一个高可用方案

 

# 通过zookeeper+activemq实现高可用方案

（master/slave模型）

1.修改activeMQ

```xml
<persistenceAdapter>
    <!-- <kahaDB directory="${activemq.data}/kahadb"/> -->
    <replicatedLevelDB  directory="${activemq.data}/levelDB"
                       replicas="2" bind="tcp://0.0.0.0:61615"
                       zkAddress="" hostname=""
                       zkPath="/activemq/levelDB"
</persistenceAdapter>
```

 

\2. 启动zookeeper服务器

 

\3. 启动activeMQ

 

参数的意思

directory： levelDB数据文件存储的位置

replicas：计算公式（replicas/2）+1  ， 当replicas的值为2的时候， 最终的结果是2. 表示集群中至少有2台是启动的

bind:  用来负责slave和master的数据同步的端口和ip

zkAddress： 表示zk的服务端地址

hostname：本机ip

# jdbc存储的主从方案

基于LOCK锁表的操作来实现master/slave

 首位写表的为master

# 基于共享文件系统的主从方案

挂载网络磁盘，将数据文件保存到指定磁盘上即可完成master/slave模式

   文件锁

高可用+高性能方案



networkConnector  +  zookeeper

![img](file:///C:\Users\lenovo\AppData\Local\Temp\msohtmlclip1\01\clip_image002.png)

 

# 容错的链接

![img](file:///C:\Users\lenovo\AppData\Local\Temp\msohtmlclip1\01\clip_image003.png)

 failover:(tcp://xxx.xxx.xxx.xxx:61616,tcp://)?randomize=false（参数=xxx）

 

课后的作业1： ActiveMQ的重发机制？什么情况下会重发消息

课后作业2：   完善注册流程（发邮件） 

 

# ActiveMQ监控

ActiveMQ自带的管理界面的功能十分简单，只能查看ActiveMQ当前的Queue和Topics等简单信息，不能监控ActiveMQ自身运行的JMX信息等



durable  topic   持久化订阅   offline 未激活   active  激活

## hawtio

HawtIO 是一个新的可插入式 HTML5 面板，设计用来监控 ActiveMQ, Camel等系统；ActiveMQ在5.9.0版本曾将hawtio嵌入自身的管理界面，但是由于对hawtio的引入产生了争议，在5.9.1版本中又将其移除，但是开发者可以通过配置，使用hawtio对ActiveMQ进行监控。本文介绍了通过两种配置方式，使用hawtio对ActiveMQ进行监控。

\1.   从<http://hawt.io/getstarted/index.html> 下载hawtio的应用程序

\2.   下载好后拷贝到ActiveMQ安装目录的webapps目录下，改名为hawtio.war并解压到到hawtio目录下

\3.   编辑ActiveMQ安装目录下conf/jetty.xml文件,在第75行添加以下代码

<bean class="org.eclipse.jetty.webapp.WebAppContext">        

​        <property name="contextPath" value="/hawtio" />        

​        <property name="war" value="${activemq.home}/webapps/hawtio" />        

​        <property name="logUrlOnStart" value="true" />  

</bean>

\4.   修改bin/env文件

-Dhawtio.realm=activemq -Dhawtio.role=admins

-Dhawtio.rolePrincipalClasses=org.apache.activemq.jaas.GroupPrincipal

需要注意的是-Dhawtio的三个设定必须放在ACTIVEMQ_OPTS设置的最前面(在内存参数设置之后),否则会出现验证无法通过的错误(另外,ACTIVEMQ_OPTS的设置语句不要回车换行)

\5.   启动activeMQ服务。访问<http://ip:8161/hawtio>.  

 

# 源码

activemq-client     send()

从使用的过程找入口

ActiveMQMessageProducer#send

ActiveMQMessageConsumer#receive

transport   oneway  链





# RabbitMQ

RabbitMQ实现了AMQP协议，AMQP协议定义了消息路由规则和方式。

（更多AMQP内容，看这里：http://www.cnblogs.com/charlesblc/p/6058799.html）

生产端通过路由规则发送消息到不同queue，消费端根据queue名称消费消息。

RabbitMQ既支持内存队列也支持持久化队列，消费端为推模型，消费状态和订阅关系由服务端负责维护，消息消费完后立即删除，不保留历史消息。

# Kafka
Kafka只支持消息持久化，消费端为拉模型，消费状态和订阅关系由客户端端负责维护，消息消费完后不会立即删除，会保留历史消息。

因此支持多订阅时，消息只会存储一份就可以了。但是可能产生重复消费的情况。



对于消费者而言有两种方式从消息中间件**获取消息**：
1. Push方式：由消息中间件主动地将消息推送给消费者；
2. Pull方式：由消费者主动向消息中间件拉取消息。