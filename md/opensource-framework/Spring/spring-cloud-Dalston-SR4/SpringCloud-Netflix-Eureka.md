Spring Cloud 是一系列框架的有序集合。它利用 Spring Boot 的开发便利性巧妙地简化了分布式系统基础设施的开发，如服务发现注册、配置中心、消息总线、负载均衡、断路器、数据监控等，都可以用 Spring Boot 的开发风格做到一键启动和部署。

# Eureka

Eureka 是 Netflix 的子模块，它是一个基于 REST 的服务，用于定位服务，以实现云端中间层服务发现和故障转移。  

服务注册和发现对于微服务架构而言，是非常重要的。有了服务发现和注册，只需要使用服务的标识符就可以访问到服务，而不需要修改服务调用的配置文件。该功能类似于 Dubbo 的注册中心，比如 Zookeeper。

Eureka 采用了 CS 的设计架构。Eureka Server 作为服务注册功能的服务端，它是服务注册中心。而系统中其他微服务则使用 Eureka 的客户端连接到 Eureka Server 并维持心跳连接。

其运行原理如下图：
<img src="http://images.extlight.com/springcloud-eureka-01.jpg" />

*注：Spring Boot 与 SpringCloud 有版本兼容关系，如果引用版本不对应，项目启动会报错。*

## 搭建注册中心

### 1. 开启注册中心功能

在启动类上添加 @EnableEurekaServer 注解
http://localhost:9000 是 Eureka 监管界面访问地址，而 http://localhost:9000/eureka/ Eureka 第三方配置注册服务的地址。  

<b>添加依赖</b>
```
<dependencyManagement>
  <dependencies>
  	<dependency>
		<groupId>org.springframework.cloud</groupId>
		<artifactId>spring-cloud-dependencies</artifactId>
		<version>Dalston.SR1</version>
		<type>pom</type>
		<scope>import</scope>
	</dependency>
	
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>1.5.9.RELEASE</version>
		<type>pom</type>
		<scope>import</scope>
	</dependency>
  </dependencies>
</dependencyManagement>
  
<dependencies>  
    <!-- eureka 服务端 -->
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-eureka-server</artifactId>
    </dependency>
</dependencies>
```
#### 配置 application.yml 参数

```
server:
    port: 9000
    
eureka:
    instance:
        hostname: localhost   # eureka 实例名称
    client:
        register-with-eureka: false # 不向注册中心注册自己
        fetch-registry: false       # 是否检索服务
        service-url:
            defaultZone: http://${eureka.instance.hostname}:${server.port}/eureka/  # 注册中心访问地址
```
### 2. 开启服务注册功能

在启动类上添加 @EnableEurekaClient 注解。
### 3. 开启服务发现功能

在启动类上添加 @EnableDiscoveryClient 注解。

## Eureka 与 Zookeeper 的区别

两者都可以充当注册中心的角色，且可以集群实现高可用，相当于小型的分布式存储系统。
### 1. CAP 理论

CAP 分别为 consistency(强一致性)、availability(可用性) 和 partition toleranc(分区容错性)。
理论核心：一个分布式系统不可能同时很好的满足一致性、可用性和分区容错性这三个需求。因此，根据 CAP 原理将 NoSQL 数据库分成满足 CA 原则、满足 CP 原则和满足 AP 原则三大类：
* CA：单点集群，满足一致性，可用性的系统，通常在可扩展性上不高
* CP: 满足一致性，分区容错性的系统，通常性能不是特别高
* AP: 满足可用性，分区容错性的系统，通过对一致性要求较低
简单的说：CAP 理论描述在分布式存储系统中，最多只能满足两个需求。
### 2 Zookeeper 保证 CP

当向注册中心查询服务列表时，我们可以容忍注册中心返回的是几分钟前的注册信息，但不能接受服务直接挂掉不可用了。因此，服务注册中心对可用性的要求高于一致性。  
但是，zookeeper 会出现一种情况，当 master 节点因为网络故障与其他节点失去联系时，剩余节点会重新进行 leader 选举。问题在于，选举 leader 的时间较长，30 ~ 120 秒，且选举期间整个 zookeeper 集群是不可用的，这期间会导致注册服务瘫痪。在云部署的环境下，因网络问题导致 zookeeper 集群失去 master 节点的概率较大，虽然服务能最终恢复，但是漫长的选举时间导致注册服务长期不可用是不能容忍的。

### 3 Eureka 保证 AP

Eureka 在设计上优先保证了可用性。EurekaServer 各个节点都是平等的，几个节点挂掉不会影响正常节点的工作，剩余的节点依然可以提供注册和发现服务。  
而 Eureka 客户端在向某个 EurekaServer 注册或发现连接失败时，会自动切换到其他 EurekaServer 节点，只要有一台 EurekaServer 正常运行，就能保证注册服务可用，只不过查询到的信息可能不是最新的。
除此之外，EurekaServer 还有一种自我保护机制，如果在 15 分钟内超过 85% 的节点都没有正常的心跳，那么 EurekaServer 将认为客户端与注册中心出现网络故障，此时会出现一下几种情况：

* EurekaServer 不再从注册列表中移除因为长时间没有收到心跳而应该过期的服务
* EurekaServer 仍然能够接收新服务的注册和查询请求，但不会被同步到其他节点上
* 当网络稳定时，当前 EurekaServer 节点新的注册信息会同步到其他节点中  

因此，Eureka 可以很好的应对因网络故障导致部分节点失去联系的情况，而不会向 Zookeeper 那样是整个注册服务瘫痪。



## 配置源码位置（EurekaClientConfigBean）

## Eureka 高可用架构

### 高可用注册中心集群

只需要增加 Eureka 服务器注册 URL：

```properties
#启动参数 --server.port = 9091，${random.int[7070,7079]}
### 这方法并不好
eureka.client.serviceUrl.defaultZone = \
http://localhost:9090/eureka,http://localhost:9091/eureka
```

如果 Eureka 客户端应用配置多个 Eureka 注册服务器，那么默认情况只有第一台可用的服务器，存在注册信息

如果第一台可用的 Eureka 服务器 Down 掉了，那么 Eureka客户端应用将会选择下一台可用的 Eureka 服务器



#### 配置源码 （EurekaClientConfigBean）

配置项 `eureka.client.serviceUrl`实际映射的字段为 `serviceUrl`，它是 Map 类型，Key 为自定义，默认值 “defaultZone”，value是需要配置的 Eureka 注册服务器 URL。
value 可以是多值字段，通过 “,” 分割



### 获取注册信息时间间隔

Eureka 客户端需要获取 Eureka  服务器注册信息，这个方便服务调用
Eureka 客户端：`EurekaClient`，关联应用集合： `Applications`
单个应用信息：`Application`，关联多个应用实例
单个应用实例：`InstanceInfo`
当 Eureka 客户端需要调用某个服务时，比如`user-service-consumer`调用`user-service-provider`，`user-service-provider`实际对应对象是`Application`，关联了许多应用实例（`InstanceInfo`）。如果应用`user-service-provider`的应用实例发生变化时，那么`user-service-consumer`是需要感知的。比如：`user-service-provider`机器从 10 台降到 5 台，那么，作为调用方的`user-service-consumer`需要知道这个变化。变化过程可能存在一定的延迟，可以通过调整注册信息时间间隔来减少错误。

#### 具体配置项

```properties
### 调整注册信息的获取周期，默认值：30 秒
eureka.client.registryFetchIntervalSeconds = 5
```



### 实例信息复制信息时间

具体就是客户端信息的上报到 Eureka 服务器时间。当 Eureka 客户端应用上报的频率越频繁 ，那么 Eureka 服务器的应用状态管理一致性越高

#### 具体配置项

```properties
### 调整客户端应用状态信息上报的周期，默认值：30 
eureka.client.instanceInfoReplicationIntervalSeconds = 5
```

> Eureka 的应用信息同步方式：拉模式
>
> Eureka 的应用信息上报方式：推模式

### 实例 ID

从 Eureka Server Dashboard 里面可以看到具体某个应用中的实例信息

命名模式：`${hostname}：${spring.application.name}：${server.port}`

#### 配置项

EurekaInstanceConfig

```properties
### Eureka 应用实例 Id
eureka.instance.instanceId = ${spring.application.name}:${server.port}
```



### 实例端点映射

```properties
### Eureka 客户端应用实例状态 URL 默认 /info
eureka.instance.statusPageUrlPath = /health
```



## Eureka 服务端高可用

### 构建 Eureka 服务器相互注册

#### Eureka Server1 -> Profile:peer1

##### 配置项

```properties
### application-peer1.properties
spring.application.name = spring-cloud-eureka-server
server.port = 9090
### 取消服务器自我注册
eureka.client.register-with-eureka = true
### 注册中心的服务器，没有必要再去检索服务
eureka.client.fetch-registry = true
### Eureka Server 服务 URL,用于客户端注册
### 当前 Eureka 服务器向 9091（Eureka 服务器 复制数据）
eureka.client.serviceUrl.defaultZone = \
http://localhost:9091/eureka
```

#### Eureka Server2 -> Profile:peer2

##### 配置项

```properties
### Eureka Server2 profile
### application-peer2.properties
spring.application.name = spring-cloud-eureka-server
server.port = 9091
### 取消服务器自我注册
eureka.client.register-with-eureka = true
### 注册中心的服务器，没有必要再去检索服务
eureka.client.fetch-registry = true
### Eureka Server 服务 URL,用于客户端注册
### 当前 Eureka 服务器向 9090（Eureka 服务器 复制数据）
eureka.client.serviceUrl.defaultZone = \
http://localhost:9090/eureka
```



> 通过启动参数，分别激活： 
>
> --spring.profile.active = pee1
>
> --spring.profile.active = pee2
>
> dashboard：replicas信息







# Spring Cloud 服务发现/注册

- 服务注册：服务实例将自身服务信息注册到注册中心。这部分服务信息包括服务所在主机IP和提供服务的Port，以及暴露服务自身状态以及访问协议等信息。
- 服务发现：服务实例请求注册中心获取所依赖服务信息。服务实例通过注册中心，获取到注册到其中的服务实例的信息，通过这些信息去请求它们提供的服务。



客户端：Eureka Client
Eureka Client 为当前服务提供注册、同步、查找服务以及其实例信息或状态等能力。
• 运行 Eureka Client
• 依赖：org.springframework.cloud:spring-cloud-starter-eureka
• 激活：@EnableEurekaClient (不推荐，写定为eureka)或者 @EnableDiscoveryClient

调整状态页面
• 配置：eureka.instance.statusPageUrlPath
• 调整健康检查页面
• 配置：eureka.instance.healthCheckUrlPath
• 调整主页

Eureka 客户端配置 API
• EurekaClientConfigBean
• Eureka 实例例配置 API
• EurekaInstanceConfigBean

## Spring Cloud  Netflix Eureka



### Eureka 服务器



#### 引入 Maven 依赖



```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-eureka-server</artifactId>
</dependency>
```



#### 激活 Eureka 服务器



```java
package com.segumentfault.springcloudlesson4eurekaserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;

@SpringBootApplication
@EnableEurekaServer
public class SpringCloudLesson4EurekaServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringCloudLesson4EurekaServerApplication.class, args);
	}
}
```



#### 调整 Eureka 服务器配置

`application.properties`

```properties
## Spring Cloud Eureka 服务器应用名称
spring.application.name = spring-cloud-eureka-server

## Spring Cloud Eureka 服务器服务端口
server.port = 9090

## 管理端口安全失效
management.security.enabled = false
```



#### 检验 Eureka Server

异常堆栈信息：

```
[nfoReplicator-0] c.n.discovery.InstanceInfoReplicator     : There was a problem with the instance info replicator

com.netflix.discovery.shared.transport.TransportException: Cannot execute request on any known server
```



http://localhost:9090/

运行效果：

> # Instances currently registered with Eureka



问题原因：Eureka Server 既是注册服务器，也是客户端，默认情况，也需要配置注册中心地址。

> "description": "Spring Cloud Eureka Discovery Client"



#### 解决问题的方法

```properties
## Spring Cloud Eureka 服务器作为注册中心
## 通常情况下，不需要再注册到其他注册中心去
## 同时，它也不需要获取客户端信息
### 取消向注册中心注册
eureka.client.register-with-eureka = false
### 取消向注册中心获取注册信息（服务、实例信息）
eureka.client.fetch-registry = false
## 单机版能否关掉，而不需配为自己？？？（看源码，配自己会被移除）
## 解决 Peer / 集群 连接问题
eureka.instance.hostname = localhost
eureka.client.serviceUrl.defaultZone = http://${eureka.instance.hostname}:${server.port}/eureka
```





### Eureka 客户端



#### 引入 Maven 依赖

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```



#### 激活 Eureka 客户端



```java
package com.segumentfault.springcloudlesson4eurekaclient;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;

@SpringBootApplication
@EnableEurekaClient
//@EnableDiscoveryClient
public class SpringCloudLesson4EurekaClientApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringCloudLesson4EurekaClientApplication.class, args);
	}
}
```



#### 配置 Eureka 客户端

```properties
## Spring Cloud Eureka 客户端应用名称
spring.application.name = spring-cloud-eureka-client

## Spring Cloud Eureka 客户端服务端口
server.port = 8080

## 管理端口安全失效
management.security.enabled = false
```



#### 检验 Eureka 客户端

发现与 Eureka 服务器控制台出现相同异常：

```
[nfoReplicator-0] c.n.discovery.InstanceInfoReplicator     : There was a problem with the instance info replicator

com.netflix.discovery.shared.transport.TransportException: Cannot execute request on any known server
```



#### 需要再次调整 Eureka 客户端配置



```properties
## Spring Cloud Eureka 客户端应用名称
spring.application.name = spring-cloud-eureka-client

## Spring Cloud Eureka 客户端服务端口
server.port = 8080

## 管理端口安全失效
management.security.enabled = false

## Spring Cloud Eureka 客户端 注册到 Eureka 服务器
eureka.client.serviceUrl.defaultZone = http://localhost:9090/eureka
```



```
eureka:
  instance:
    statusPageUrlPath: ${server.servletPath}/info
    healthCheckUrlPath: ${server.servletPath}/health
```

eureka server --> eureka client /health -->  beans 



#### Jercy 框架

springmvc有着类似的实现





### Spring Cloud Config 与 Eureka 整合



#### 调整 `spring-cloud-config-server` 作为 Eureka 客户端



#### 引入 Maven 依赖

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-eureka</artifactId>
</dependency>
```



#### 激活 Eureka 客户端

```java
package com.segmentfault.springcloudlesson4configserver;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.config.server.EnableConfigServer;

@SpringBootApplication
@EnableConfigServer
@EnableDiscoveryClient
public class SpringCloudLesson4ConfigServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringCloudLesson4ConfigServerApplication.class, args);
	}
}
```



#### 调整 `spring-cloud-config-server` 配置



```properties
## 配置服务器应用名称
spring.application.name = spring-cloud-config-server

## 配置服务器端口
server.port = 7070

## 关闭管理端actuator 的安全
## /env /health 端口完全开放
management.security.enabled = false

## 配置服务器文件系统git 仓库
## ${user.dir} 减少平台文件系统的不一致
# spring.cloud.config.server.git.uri = ${user.dir}/src/main/resources/configs

## 配置服务器远程 Git 仓库（GitHub）
spring.cloud.config.server.git.uri = https://github.com/mercyblitz/tmp

## 强制拉去 Git 内容
spring.cloud.config.server.git.force-pull = true

## spring-cloud-config-server 注册到 Eureka 服务器
eureka.client.serviceUrl.defaultZone = http://localhost:9090/eureka
```



#### 调整 `spring-cloud-eureka-client` 称为 Config 客户端

##### 引入 `spring-cloud-starter-config` Maven 依赖

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

##### 创建 `bootstrap.properties`

##### 配置 Config 客户端配置

`bootstrap.properties`：

```properties
## 配置客户端应用关联的应用
## spring.cloud.config.name 是可选的
## 如果没有配置，采用 ${spring.application.name}
spring.cloud.config.name = segmentfault
## 关联 profile
spring.cloud.config.profile = prod
## 关联 label
spring.cloud.config.label = master
## 激活 Config 服务器发现
spring.cloud.config.discovery.enabled = true
## 配置 Config 服务器的应用名称（Service ID）
spring.cloud.config.discovery.serviceId = spring-cloud-config-server
```



##### 检验效果

启动发现，spring-cloud-config-server 服务无法找到，原因如下：

```
注意：当前应用需要提前获取应用信息，那么将 Eureka 的配置信息提前至 bootstrap.properties 文件
原因：bootstrap 上下文是 Spring Boot 上下文的 父 上下文，那么它最先加载，因此需要最优先加载 Eureka 注册信息
```



##### 调整后配置

`bootstrap.properties`：

```properties
## 注意：当前应用需要提前获取应用信息，那么将 Eureka 的配置信息提前至 bootstrap.properties 文件
## 原因：bootstrap 上下文是 Spring Boot 上下文的 父 上下文，那么它最先加载，因此需要最优先加载 Eureka 注册信息
## Spring Cloud Eureka 客户端 注册到 Eureka 服务器
eureka.client.serviceUrl.defaultZone = http://localhost:9090/eureka

## 配置客户端应用关联的应用
## spring.cloud.config.name 是可选的
## 如果没有配置，采用 ${spring.application.name}
spring.cloud.config.name = segmentfault
## 关联 profile
spring.cloud.config.profile = prod
## 关联 label
spring.cloud.config.label = master
## 激活 Config 服务器发现
spring.cloud.config.discovery.enabled = true
## 配置 Config 服务器的应用名称（Service ID）
spring.cloud.config.discovery.serviceId = spring-cloud-config-server
```



##### 再次检验效果

访问 http://localhost:8080/env：

```json
"configService:https://github.com/mercyblitz/tmp/segmentfault-prod.properties": {
    "sf.user.id": "001",
    "sf.user.name": "xiaomage",
    "name": "segumentfault.com"
  },
  "configService:https://github.com/mercyblitz/tmp/segmentfault.properties": {
    "name": "segumentfault.com"
  },
```

以上内容来自于`spring-cloud-config-server`:

http://localhost:7070/segmentfault-prod.properties

```properties
name: segumentfault.com
sf.user.id: 001
sf.user.name: xiaomage
```





# Spring Cloud 高可用服务治理

配置项：eureka.client.serviceUrl

思考
•当注册应用之间存在相互关联时，那么上层应用如何感知下层服务的存在？
•如果上层应用感知到下层服务，那么它是怎么同步下层服务信息？
•如果应用需要实时地同步信息，那么确保一致性



应用元信息
• 获取间隔
• 配置项：eureka.client.registryFetchIntervalSeconds
• 同步间隔
• 配置项：eureka.client.instanceInfoReplicationIntervalSeconds

## Spring Cloud Eureka



### Eureka 客户端

instanceid 区分

#### 配置多Eureka 注册中心



```properties
## 应用名称
spring.application.name = spring-cloud-eureka-client

## 客户端 端口随即可用
server.port = 0

## 配置连接 Eureka 服务器
## 配置多个 Eureka 注册中心，以"," 分割
eureka.client.serviceUrl.defaultZone = \
  http://localhost:9090/eureka,\
  http://localhost:9091/eureka
```



#### 激活 ：`@EnableEurekaClient` 或者 `@EnableDiscoveryClient`

```java
package com.segumentfault.springcloudlesson5eurekaclient;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.eureka.EnableEurekaClient;

@SpringBootApplication
@EnableDiscoveryClient
//@EnableEurekaClient
public class SpringCloudLesson5EurekaClientApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringCloudLesson5EurekaClientApplication.class, args);
	}
}
```





### Eureka 服务器



#### 配置高可用 Eureka 服务器



##### 设置公用 Eureka 服务器配置

`application.properties`:

```properties
## 定义 应用名称
spring.applicaiton.name = spring-cloud-eureka-server

## 管理端安全失效
management.security.enabled = false

## 公用 Eureka 配置
### 向注册中心注册
eureka.client.register-with-eureka = true
### 向获取注册信息（服务、实例信息）
eureka.client.fetch-registry = true
```



##### 配置 Peer 1 Eureka 服务器

`application-peer1.properties`:(单机情况相当于 profile = "peer1")

```properties
# peer 1 完整配置

## 配置 服务器端口
## peer 1 端口 9090
server.port = 9090

## peer 2 主机：localhost , 端口 9091
peer2.server.host = localhost
peer2.server.port = 9091

# Eureka 注册信息
eureka.client.serviceUrl.defaultZone = http://${peer2.server.host}:${peer2.server.port}/eureka
```



##### 启动 Peer 1 Eureka 服务器

通过启动参数 `—spring.profiles.active=peer1` ,相当于读取了 `application-peer1.properties `和 `application.properties`



##### 配置 Peer 2 Eureka 服务器

`application-peer2.properties`:(单机情况相当于 profile = "peer2")

```properties
# peer 2 完整配置

## 配置 服务器端口
## peer 2 端口 9091
server.port = 9091

## peer 1 主机：localhost , 端口 9090
peer1.server.host = localhost
peer1.server.port = 9090

# Eureka 注册信息
eureka.client.serviceUrl.defaultZone = http://${peer1.server.host}:${peer1.server.port}/eureka
```



##### 启动 Peer 2 Eureka 服务器

通过启动参数 `—spring.profiles.active=peer2` ,相当于读取了 `application-peer2.properties `和 `application.properties`



### Spring Cloud Consul

Consul是一种服务网格解决方案，提供具有服务发现，配置和分段功能的全功能控制平面。这些功能中的每一个都可以根据需要单独使用，也可以一起使用以构建全服务网格。Consul需要数据平面并支持代理和本机集成模型。Consul附带一个简单的内置代理，因此一切都可以开箱即用，但也支持第三方代理集成，如Envoy。

#### Consul 组件
• 服务发现（Service Discovery）
• 健康检查（Health Check）
• 键值存储（KV Store）
• 多数据中心（Multi Datacenter）
• 理解 Raft 协议：http://thesecretlivesofdata.com/raft/



#### 快速上手
• 安装
• Agent 启动
• 键值存储（KV Store）
• UI 控制台
• Spring Cloud 整合

#### 引入依赖

```xml
<dependency>
	<groupId>org.springframework.cloud</groupId>
	<artifactId>spring-cloud-starter-consul-discovery</artifactId>
</dependency>
```



#### 激活服务发现客户端

```java
package com.segumentfault.springcloudlesson5consulclient;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;

@SpringBootApplication
@EnableDiscoveryClient
public class SpringCloudLesson5ConsulClientApplication {

	public static void main(String[] args) {
		SpringApplication.run(SpringCloudLesson5ConsulClientApplication.class, args);
	}
}
```



#### 利用服务发现API 操作

##### 配置应用信息

```properties
## 应用名称
spring.application.name = spring-cloud-consul

## 服务端口
server.port = 8080

## 管理安全失效
management.security.enabled = false

## 连接 Consul 服务器的配置
### Consul 主机地址
spring.cloud.consul.host = localhost
### Consul 服务端口
spring.cloud.consul.port = 8500
```



##### 编写 `DiscoveryClient` Controller

```java
package com.segumentfault.springcloudlesson5consulclient.web.controller;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.client.ServiceInstance;
import org.springframework.cloud.client.discovery.DiscoveryClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.LinkedList;
import java.util.List;

/**
 * {@link DiscoveryClient} {@link RestController}
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 1.0.0
 */
@RestController
public class DiscoveryClientController {


    private final DiscoveryClient discoveryClient;

    private final String currentApplicationName;

    @Autowired
    public DiscoveryClientController(DiscoveryClient discoveryClient,
                                     @Value("${spring.application.name}") String currentApplicationName) {
        this.discoveryClient = discoveryClient;
        this.currentApplicationName = currentApplicationName;
    }

    /**
     * 获取当前应用信息
     *
     * @return
     */
    @GetMapping("/current/service-instance")
    public ServiceInstance getCurrentServiceInstance() {
//        return discoveryClient.getInstances(currentApplicationName).get(0);
        return discoveryClient.getLocalServiceInstance();

    }


    /**
     * 获取所有的服务名
     *
     * @return
     */
    @GetMapping("/list/services")
    public List<String> listServices() {
        return discoveryClient.getServices();
    }

    /**
     * 获取所有的服务实例信息
     *
     * @return
     */
    @GetMapping("/list/service-instances")
    public List<ServiceInstance> listServiceInstances() {
        List<String> services = listServices();
        List<ServiceInstance> serviceInstances = new LinkedList<>();

        services.forEach(serviceName -> {
            serviceInstances.addAll(discoveryClient.getInstances(serviceName));
        });

        return serviceInstances;
    }
}
```



#### health 问题

由于 healtbuilder withdetail  键值不能为空？？ consul 实现有部分为空













# Eureka note





### 传统的服务治理



XML-RPC -> XML 方法描述、方法参数 -> WSDL（WebService 定义语言）
Webservise -> SOAP(HTTP、SMTP)  -> 文本协议（头部分、体部分）

REST -> JSON/XML(Schema：类型、结构)  -> 文本协议（HTTP Header、Body）
W3C Schema：xsd:string  原子类型，自定义自由组合原子类型
Java POJO：int、StringResponse Header -> Conten-Type:application/json;charset=UTF-8
Dubbo：Hession、Jaca Serialization（二进制），跨语言不变，一般通过 Client （Java、C++），

> 二进制的性能是非常好（字节流，免去字符流（字符编码），免去了字符解释，机器友好，对人不友好）
>
> 序列化：把编程语言的数据结构转换成字节流
>
> 反序列化：字节流转换成编程语言的数据结构（原生类型组合）



### 传统的服务发现

WebService
​     -- UDDI
REST
​     -- HATEOAS
Java
​     -- JMS
​     -- JNDI

### 高可用架构

#### 基本原则

1. 消除单点失败
2. 可靠性交迭
3. 故障探测

URI：统一资源定位符
URI 用于网络资源定位的描述
网络是通讯方式
资源是需要消费媒介
定位是路由 



Proxy：一般性代理，路由
​    Nginx：反向代理
Broker：包括路由，并且管理，老的称谓（MOM）
​    Message Broker：消息路由、消息管理（消息是否可达）



#### 可用性比率计算

可用性比率：通过时间来计算（一年或一个月）
比如：一年 99.99%
可用时间：365 * 24 * 3600 * 99.99%
不可用时间：365 * 24 * 3600 * 0.01% = 3153.6 秒  < 一个小时
不可用时间：1 个小时 推算一年 1 / 24 / 365 = 0.01%





单台机器 不可用比率： 1%

两台机器 不可用比率： 1%  *  1%

N 台机器 不可用比率： 1% ^ n



#### 可靠性

微服务里面的问题：
一次调用：
A       ->     B     ->   C
99%  ->   99%  ->  99%  =  97%

A       ->     B     ->   C       -> D
99%  ->   99%  ->  99%  ->  99%=  96%

结论： 增加机器可以提高可用性，增加服务调用会降低可靠性，同时降低了可用性



## 服务发现

## Eureka 客户端

- Neflix Eureka Client
  依赖：spring-cloud-starter-eureka
  API:
  EurekaClient
  ServiceInstance
  Spring Cloud Commons

激活：@EnableEurekaClient 或 @EnableDiscoveryClient
健康指标：HealthIndicator

项目创建：Eureka-Discovery Actuator Web

> RestTemplate   @LoadBalanced
>
> 前缀 http://user-service-provider/
>
> postForObject

@SpringCloudApplication

```properties
### 消费者配置类似
spring.applicaiton.name = user-service-provider
### Eureka 注册中心服务器端口
eureka.server.port = 9090
### 服务提供方端口
server.port = 7070
### Eureka Server 服务 URL,用于客户端注册
eureka.client.serviceUrl.defaultZone = \
http://localhost:${eureka.server.port}/eureka
### Management 安全失效
management.security.enabled = false
```



## Eureka 服务器

Neflix Eureka Server
激活：@EnableEurekaServer

```properties
### 取消服务器自我注册
eureka.client.register-with-eureka = false
### 注册中心的服务器，没有必要再去检索服务
eureka.client.fetch-registry = false
### Eureka Server 服务 URL,用于客户端注册
eureka.client.serviceUrl.defaultZone = \
http://localhost:${server.port}/eureka
```



**Eureka 服务器**一般不需要自我注册，也不需要注册其他服务器

Eureka 自我注册的问题，服务器本身没有启动

> Fast Fail：快速失败
>
> Fault-Torerance：容错



通常经验，Eureka 服务器不需要开启自动注册，也不需要检索服务。
但这两个设置并不影响作为服务器的使用，不过建议关闭，为了减少不必要的异常堆栈，减少错误的干扰（比如：系统异常和业务异常）

**Replicas**



### 其他内容

dubbo 为 服务维度

spring cloud 为应用维度 

- 分布式系统基本组成
  - 服务提供方（Provider）
  - 服务消费方（Consumer）
  - 服务注册中心（Registry）（提供者和消费者都需注册）
  - 服务路由（Router）
  - 服务代理（Broker）
  - 通讯协议（Protocol）

服务管理、监控、健康检查

region  zone



## 问答

1. consul 和 eureka 一样吗？
   提供功能类似，consul 功能更强大，广播式服务发现/注册
2. 重启 eureka 服务器，客户端应用不需重启，客户端在不停上报信息，服务器重启阶段会大量报错
3. consumer 分别注册成多个服务，还是同意放在一起注册成一个服务？权限应该如何处理？
   consumer是否分成多个服务，要分情况，大多数情况是需要，根据应用职责划分。权限根据服务方法需要，比如有些敏感操作的话，可以根据不同用户做鉴权。
4. 客户上报信息存储在哪里？内存还是数据库？
   很多为集合的属性，都是在内存中缓存着，EurekaClient 并不是所有服务，需要的服务。比如：Eureka Server 管理了 200 个应用，每个应用存在 100 各实例，总体管理 20000 个实例， 客户端根据自己需要的应用实例。
5. 可用性保证？status，会自动切换，不过不一定及时，此时服务器可能存在脏数据，或者轮询更新时间未达
6. 多 service 如何保证事务？
   需要分布式事务实现（JTA），一般互联网项目，没有这种昂贵的操作
   XResource





