### JMX

jcp,jsr

JMX 全称 Java Management Extensions，技术提供构建分布式、Web、模块化的工具，以及管理和监控设备和应用的动态解决方案。从 Java 5 开始，JMX API 作为
Java 平台的一部分。

![](https://ws1.sinaimg.cn/large/006xzusPly1g5ffheylekj30f60agtas.jpg)

元信息：描述信息的信息（Class，reflect）

java 监控和管理控制台    所以应该要安全控制

javap 反编译

> The management interface of an MBean is represented as:
> ■ Valued attributes that can be accessed
> ■ Operations that can be invoked
> ■ Notifications that can be emitted (see “Notification Model” on page 29)
> ■ The constructors for the MBean’s Java clas

### 管理Bean（MBeans）

#### 标准 MBeans

  设计和实现最为简单，Bean的管理**通过接口方法来描述**。MXBean 是一种特殊标准MBean，它使用开放MBean的概念，允许通用管理，同时简化编码

```
javax.management.xxx
```

实现类必须为前缀

```java
//接口：com.example.demo.mbean.TestMBean   nnnMBean
//实现：com.example.demo.mbean.Test        nnn
MBeanServer mBeanServer = ManagementFactory.getPlatformMBeanServer();
ObjectName objectName = new ObjectName("com.example.demo.mbean:type=Test");
mBeanServer.registerMBean(new Test(), objectName);

```



#### 动态 MBeans

  必须实现指定的接口，不过它在运行时能让管理接口发挥最大弹性

```java
/**
 * 接口：DynamicMBean
 * 标准实现：StandardMBean
 * 接口:Data
 * 实现：xxx
 */

MBeanServer mBeanServer = ManagementFactory.getPlatformMBeanServer();

ObjectName objectName = new ObjectName("com.example.demo.mbean:type=Data");

Data data = new DataMBean();

DynamicMBean dynamicMBean = new StandardMBean(data, Data.class);

mBeanServer.registerMBean(dynamicMBean, objectName);
```



## 通知 Notificatoin



### JMX 客户端

jconsole    MBean



jvisualvm 性能分析



### Spring Boot 整合

MBeanServerFactoryBean

```
@ManagedResource
```

server.port=0   随机端口



生产准备：管理和控制



* jolokia http方式的 jmx 
* jconsole GUI方式

### 题外话

bootstrap classloader （加载 rt.jar 的 classloader ）为什么返回 null？

不希望你覆盖它的类





