# 概述
Spring Cloud 是一系列框架的有序集合。它利用 Spring Boot 的开发便利性巧妙地简化了分布式系统基础设施的开发，如服务发现注册、配置中心、消息总线、负载均衡、断路器、数据监控等，都可以用 Spring Boot 的开发风格做到一键启动和部署。

# Eureka
Eureka 是 Netflix 的子模块，它是一个基于 REST 的服务，用于定位服务，以实现云端中间层服务发现和故障转移。  

服务注册和发现对于微服务架构而言，是非常重要的。有了服务发现和注册，只需要使用服务的标识符就可以访问到服务，而不需要修改服务调用的配置文件。该功能类似于 Dubbo 的注册中心，比如 Zookeeper。

Eureka 采用了 CS 的设计架构。Eureka Server 作为服务注册功能的服务端，它是服务注册中心。而系统中其他微服务则使用 Eureka 的客户端连接到 Eureka Server 并维持心跳连接。

其运行原理如下图：
<img src="http://images.extlight.com/springcloud-eureka-01.jpg" />

*注：Spring Boot 与 SpringCloud 有版本兼容关系，如果引用版本不对应，项目启动会报错。*

# 搭建注册中心
## 1. 开启注册中心功能
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
## 2. 开启服务注册功能
在启动类上添加 @EnableEurekaClient 注解。
## 3. 开启服务发现功能
在启动类上添加 @EnableDiscoveryClient 注解。

# Eureka 与 Zookeeper 的区别
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