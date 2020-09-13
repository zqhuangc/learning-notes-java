## JMX

jcp,jsr

JMX 全称 Java Management Extensions，技术提供构建分布式、Web、模块化的工具，以及管理和监控设备和应用的动态解决方案。从 Java 5 开始，JMX API 作为
Java 平台的一部分。



* 规范

  > JSR 3：JMX 1.0、JMX 1.1和 1.2（作为Java 5的一部分）
  > JMX 1.4：2006.11.09（作为Java 6的一部分）
  > JSR 255：JMX 2.0
  > JSR 160：JMX Remote API 1.0
  > JSR 262：JMX Remote API for Web Services


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

#### 开放 MBeans

动态 MBean，提供通用管理所依赖的基本数据类型以及用户友好的自描述信息

#### 模型 MBeans

同样也是动态MBean，在运行时能够完全可配置和自描述，为动态的设备资源提供带有默认行为的MBean泛型类

### 通知模型（Notification Model）

通知模型允许 MBean 广播管理事件，这种操作称之为通知。管理应用和其他对象注册成监听器。

```
NotificationBroadcaster
NotificationBroadcasterSupport# sendNotification
NotificationFilter
NotificationListener
```

### MBean 元数据类（MetaData Class）

元信息类包含描述所有MBean 管理接口的组件接口，其中包括：
属性（Attribute）
操作（Operation）
通知（Notification）
构造器（Constructor）



### 代理级别（Agent Level）
* MBean 服务器

MBean 服务器是一个在代理上的MBean的注册器。它仅用作暴露MBean 的管理接口，而非其引用对象。

* 代理服务

代理服务是在MBean服务器上能够执行已注册MBean的管理操作，其中包括一下代理服务：

> 动态类加载
> 监控
> 定时器
> 服务关系



## JMX 核心 API

### 标准 MBeans
MBean
接口的类名称必须以“MBean”为后缀，如MBean 定义为 “XXXMBean”,那么它的实现类名必须是“XXX”

### MXBean
接口的类名称必须以“MXBean”为后缀
举例：java.lang.management.MemoryManagerMXBean
或者接口标记@javax.management.MXBean注解

### 动态 MBeans
管理资源实现 javax.management.DynamicMBean接口
简化API：javax.management.StandardMBean

### MBean 元信息类（Meta Data Class）
属性：javax.management.MBeanAttributeInfo
操作：javax.management.MBeanOperationInfo
构造器：javax.management.MBeanConstructorInfo
参数：javax.management.MBeanParameterInfo
通知：javax.management.MBeanNotificationInfo
Bean：javax.management.MBeanInfo

### 开放MBean 元信息类（Meta Data Class）
属性：javax.management.openmbean.OpenMBeanAttributeInfo
操作：javax.management.openmbean.OpenMBeanOperationInfo
构造器：javax.management.openmbean.OpenMBeanConstructorInfo
参数：javax.management.openmbean.OpenMBeanParameterInfo
通知：javax.management.openmbean.OpenMBeanNotificationInfo
Bean：javax.management.openmbean.OpenMBeanInfo

### 模型 MBeans
参考 JMX 规范

### 代理相关（Agent）
MBean 服务器：javax.management.MBeanServer
管理工厂：java.lang.management.ManagementFactory

### JMX 客户端

jconsole    MBean

jvisualvm 性能分析

JMX Remote API（JSR-160）

### Spring Boot 整合

MBeanServerFactoryBean

```
@ManagedResource
```

server.port=0   随机端口



生产准备：管理和控制



* jolokia http方式的 jmx 
* jconsole GUI方式

#### 核心组件

* 管理资源
  组件：ManagedResource
  注解：@ManagedResource
* Spring JMX组件：MBeanExportor implement InitializingBean, SmartInitializingSingleton
* Spring Boot自动装配：JMXAutoConfiguration

```
@ManagedAttribute 标注 getter，setter方法 是否标注对应是否可读可写
```

#### Actuator

### 题外话

bootstrap classloader （加载 rt.jar 的 classloader ）为什么返回 null？

不希望你覆盖它的类





