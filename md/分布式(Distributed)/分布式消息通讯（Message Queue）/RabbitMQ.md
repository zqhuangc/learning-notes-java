## 工作模式
简单（一对一）单队列    通道
work（一对多）单队列    
订阅发布（一对多）
交换机，多队列（一对一）   队列绑定   
路由
topic

```xml
pom.xml

<dependency>
    <groupId>org.springframework.amqp</groupId>
    <artifactId>spring-rabbit</artifactId>
    <version>1.4.0.RELEASE</version>
</dependency>
<dependency>
    <groupId>com.rabbitmq</groupId>
    <artifactId>amqp-client</artifactId>
    <version>3.4.1</version>
</dependency>

<!--spring（发送与监听放在一起测试）
首先是命名空间 -->
<!-- 定义RabbitMQ的连接工厂 -->
<rabbit:connection-factory id="connectionFactory"
    host="127.0.0.1" port="5672" username="taotao" password="taotao"
    virtual-host="/taotao" />

<!-- 定义Rabbit模板，指定连接工厂以及定义exchange -->
<rabbit:template id="amqpTemplate" connection-factory="connectionFactory" exchange="fanoutExchange" />
<!-- <rabbit:template id="amqpTemplate" connection-factory="connectionFactory"
    exchange="fanoutExchange" routing-key="foo.bar" /> -->

<!-- MQ的管理，包括队列、交换器等 -->
<rabbit:admin connection-factory="connectionFactory" />

<!-- 定义交换器，自动声明 -->
<rabbit:fanout-exchange name="fanoutExchange" auto-declare="true">
    <rabbit:bindings>
        <rabbit:binding queue="myQueue"/>
    </rabbit:bindings>
</rabbit:fanout-exchange>

<!-- 	<rabbit:topic-exchange name="myExchange">
    <rabbit:bindings>
        <rabbit:binding queue="myQueue" pattern="foo.*" />
    </rabbit:bindings>
</rabbit:topic-exchange> -->

<!-- 定义队列，自动声明    还可通过图形管理界面绑定 -->
<rabbit:queue name="myQueue" auto-declare="true" durable="true"/>

<!-- 队列监听 -->
<rabbit:listener-container connection-factory="connectionFactory">
    <rabbit:listener ref="foo" method="listen" queue-names="myQueue" />
</rabbit:listener-container>
```





<http://localhost:15672/#/vhosts> 