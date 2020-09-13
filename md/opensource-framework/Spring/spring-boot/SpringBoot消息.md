# 消息

## Java Message Service（JSR-914）

面向消息中间件	
​	面向消息中间件（Message Oriented Middleware）是一种支持在分布式系统中发送和接受消息软件或者硬件的基础设施。通过非对称平台，MOM让应用模块成为分布式，同时减少了开发跨操作系统和网络接口应用的复杂度。

- 优势
  异步
  路由
  解耦
- 不足
  性能
  可靠性
  复杂

### JMS 元素

JMS 提供方（Provider）：实现JMS 接口的MOM
JMS 客户端（Client）：生产或消费消息的应用或进程
JMS 生产者（Producer）：创建和发送消息的JMS客户端
JMS 消费者（Consumer）：接收消息的JMS客户端
JMS 消息    （Message）：JMS客户端之间的传输数据对象
JMS 队列    （Queue）：包含待读取消息的准备区域
JMS 主题    （Topic）：发布消息的分布机制

### JMS 消息

消息头（Header）
所有消息支持相同的头字段集合，头字段包含客户端和提供方识别和路由消息的数据。

消息属性（Properties）
除标准的头字段以外，提供一种内建机制来添加可选的消息头字段。

应用特殊属性
标准属性
提供方特殊属性
消息主体（Body）

JMS定义了多种消息主题类型，覆盖了主要的消息风格

JMS 消息头字段（Header Fields）
JMSDestination：消息发送目的地
JMSDeliveryMode：消息传递模式
JMSMessageID：消息ID
JMSTimestamp：消息发送时间戳
JMSCorrelationID：提供方特殊ID、应用特殊ID和提供方本地字节数组值
JMSReplyTo：消息回复地址，说明消息期待回复，可选值
JMSRedelivered：消息重投递标识
JMSType：消息客户端发送消息时的类型标识
JMSExpiration：消息过期
JMSPriority：消息优先级

### JMS 消息主体（Body）

​	JMS提供五种消息主体的形式，每种形式通过消息接口定义：
StreamMessage
消息整体主体包含流式Java原生值，它是连续地被填充和读取的。
MapMessage
消息整体主体包含键值对集合，其中键为字符串，值为Java原生类型。条目访问可被计算器连续地或者名称随机地访问，它的顺序并不一定。
TextMessage
​	消息整体主体包含一个Java String 对象。
ObjectMessage
消息整体主体包含一个Serializable 对象，如果需要使用集合对象，确保JDK 1.2或更高。
BytesMessage

JMS 消息确认（Acknowledgment）
​	所有JMS消息支持acknowledge方法的使用，当客户端已规定JMS消息将明确地收到。如果客户端使用了消息自动确认的话，调用acknowledge方法将被忽略。

JMS 消息模型
点对点模型（Point-To-Point Model ）

发布/订阅模型（Publish/subscribe Model）

## 高级消息队列协议（AMQP）

介绍
​	高级消息队列协议（AMQP），全称Advanced Message Queuing Protocol，是一种针对面向消息中间件的开放标准应用层协议，定义了面向消息、队列、路由、可靠性和安全等特性。

历史
1.0 协议：2011年10月30日

实现
Apache ActiveMQ
Pivotal RabbitMQ

## Apache Kafka

介绍
​	Kafka 是一种分布式流式计算平台，用于构建实时的数据流水线以及流式计算应用，它是水平伸缩的、容错的、极其快速，并且运行在成千上万的公司的生产环境。

三大关键能力
发使用容错的方式来存储流式记录
发布和订阅流式记录，类似于消息队列或企业消息系统
处理流式记录

优势
构建实时的流式计算数据流水线
构建实时的流式计算应用

基本概念
Kafka是集群式运行
Kafka集群分类存储流式记录，这种分类称为主题（Topic）
每条记录包含键（Key）、值（Value）、以及时间戳（Timestamp）

四类核心API
生产者 API
能够让应用发布流式记录到一个或多个主题
消费者 API
能够让应用订阅流式记录到一个或多个主题，并且处理他们
流式 API
能够让应用充当流式处理器，消费一个或多个主题的输入流，生产一个或多个主题的输出流，并且高效地将输入流转化成输出流
连接器 API
构建生产者和消费者之间连接

操作演示
启动服务
zookeeper
kafka
创建主题
生产消息
消费消息
搭建集群
编码演示

编码演示

建立连接

生产消息

消费消息

消息主体序列化/序列化

自定义 serializer deserializer