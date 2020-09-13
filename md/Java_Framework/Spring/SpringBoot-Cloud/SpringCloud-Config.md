# Spring Cloud Config Client

## 预备知识

### Spring Boot 不同版本间的差异

#### 发布/订阅模式

`java.util.Observable`是一个发布者

`java.util.Observer`是订阅者

发布者和订阅者：1:N

发布者和订阅者：N:M

#### 事件/监听模式

`java.util.EvenListener` 事件监听接口（标记）

`java.util.EventObject` 事件对象

    * 事件对象关联事件源（source）

ApplicationListener

ApplicationEvent

```java
// Annocation 驱动 Spring上下文
// 注册监听器
AnnocationConfigApplicationContext#addApplicaitonListener
// 发布事件
AnnocationConfigApplicationContext#publishEvent
// 监听事件
AnnocationConfigApplicationContext#refresh
ApplicationListener#onApplicationEvent
```

* SpringBoot 核心事件 
  - ApplicationEnvironmentPreparedEvent
  - ApplicationPreparedEvent
  - ApplicationStartingEvent
  - ApplicationReadyEvent
  - ApplicationFailedEvent

## Spring 事件/监听

### Listener --》properties，Environment --》 profile

### Spring Boot 事件/监听器

#### ConfigFileApplicaitonListener

管理配置文件，比如：`application.properties`以及`application.yaml`  
`application-{profile}.properties`:  

profile = dev、test

1. `application-{profile}.properties`
2. `application.properties`



SpringBoot中相对于ClassPath：/MATA-INF/spring.fatories  

java SPI:`java.util.ServiceLoader`

Spring SPI  :  *Listener

##### 如何控制顺序

实现 Ordered 或标记`@Order`
在Spring里面，数值越小，越优先



```
ConfigFileApplicaitonListener#onApplicationEvent -> onApplicationEnvironmentPreparedEvent->postProcessEnvironment -> Loader  -> PropertySourceLoader


invokeBeanFactoryPostProcessors(beanFactory)  -> ... -> AutoConfigurationImportSelector#selectImports
```



### Spring Cloud 事件/监听器

#### BootstrapApplicationListener

> 加载的优先级 高于` ConfigFileApplicaitonListener`，所以 application.properties 文件即使定义参数，也配置不到`bootstrap.properties`！
>
> 原因在于：order
>
>  `BootstrapApplicationListener` 第 6 优先
> public static final int DEFAULT_ORDER = Ordered.HIGHEST_PRECEDENCE + 5;
>
> `ConfigFileApplicaitonListener` 第 11 优先
> public static final int DEFAULT_ORDER = Ordered.HIGHEST_PRECEDENCE + 10;

1. 负责加载`bootstrap.properties`或者`bootstrap.yaml`

2. 负责初始化 Bootstrap ApplicationContext ID=“boostrap”

Bootstrap 是一个根 Spring 上下文，parent = null 

> 联想ClassLoader：
>
> ExtClassLoader <-  AppClassLoader <- System ClassLoader <- Bootstrap ClassLoader(null)



post*方法，让子类实现
management.security.enabled = false

### 加载器

```poperties
org.springframework.boot.env.PropertySourceLoader=\
org.springframework.boot.env.PropertiesPropertySourceLoader,\
org.springframework.boot.env.YamlPropertySourceLoader
```

### ConfigurableApplicationContext

标准实现类：`AnnocationConfigApplicationContext`

#Actuator 监控

#### Env 端点：`EnvironmentEndpoint`

`Environment` 关联多个带名称的`PropertySource`
可参考 Spring 源码的#initPropertySource() 

`Environment` 有两种实现方式：
普通类型：`StandardEnvironment`

Web 类型：`StandardServletEnvironment`

`Environment`:

* ConfigurableEnvironment
  * `AbstractEnvironment`
    * StandardEnvironment

`Environment` 关联一个`PropertySources`实例
`PropertySources`关联多个 `PropertySource`,并且有优先级

常用的`PropertySource`：
Java System#getPropeties 实现：名称 “systemProperties”，对应的内容 `System.getPropeties ()`

Java System#getenv 实现（环境变量）：名称 “systemEnvironment”，对应的内容 `System.getPropeties ()`


关于Spring Boot 配置优先级顺序可查官网 Externalized Configuration

builder，MutablePropertySources，addLast

#### 实现自定义配置

实现步骤：

1. 实现`PropertySourceLocator#locale`
2. 暴露该实现作为一个 Spring Bean 
3. 实现`PropertySource`
4. 定义且配置/META-INF/spring.factories 

注意事项：

`Environment`允许出现同名配置，高优先级胜出
内部实现：`MutablePropertySources`关联的代码，采用写时复制List（CopyOnWriteList）
propertySourceList FIFO，说明有顺序

可通过 `MutablePropertySources#addFirst`提到 最优先，相当于`List#set(0,property)`



### 问题

1. .yml 和 .yaml 没什么区别，只是扩展名不同（不推荐用，可能方便但格式严格，若没配对会使用默认配置，容易出问题）
2. 自定义配置一般用于中间件开发
3. Spring 中 @EventListener 和 ApplicationListener 没什么区别，前者为 Annocation 编程模式，后者为 接口编程
4. /env 端点使用场景：排查问题，如分析` @Value("server.port") `里面占位符的具体值，查看配置文件加载顺序
5. 如何防止Order一样，Spring Boot和 Spring Cloud没办法，Spring Security 通过异常实现
6. 服务监控跟鹰眼类似
7. BootstrapApplicationListener是引入 cloud组件来用
8.  pom.xml 引入 spring-boot-starter-config

##  书籍推荐

翟永超《Spring Cloud 微服务实战》



# Spring Cloud Config Server



### 构建 Spring Cloud 配置服务器

#### 实现步骤

1. 在 Configuration Class 标记 `@EnableConfigServer`

2. 配置文件目录（基于 git）
   1.  melody(-default).properties（默认） //默认环境，跟着代码仓库
   2.  melody-dev.properties(profile = "dev")
   3.  melody-test.properties(profile = "test") // 测试环境
   4.  melody-staging.properties(profile = "staging") // 预发布环境
   5.  melody-prod.properties(profile = "prod") // 生产环境

   里面的内容，my.name = ... 如 melody

3. 服务端配置：配置版本仓库（本地）

   > 注意：放在存有`.git`的根目录

   ```properties
   spring.applicaiton.name = config-server
   ### 定义 HTTP 服务端口
   server.port = 9090
   ### 本地仓库的 Git URI 配置
   spring.cloud.server.git.uri = \
   file:/// ...
   ### https://
   ### 全局关闭 Actuator 安全
   # management.security.enabled = false
   ### 细粒度的开放 Actuator Endpoint
   ### sensitive 关注的是敏感性，安全
   endpoints.env.sensitive = false
   #endpoints.health.sensitive = false
   ```

### 构建 Spring Cloud 配置客户端



### 实现步骤

1. 创建 `bootstrap.properties` 或者 `bootstrap.yaml`

2.  `bootstrap.properties` 或者 `bootstrap.yaml`文件中配置客户端信息

   ```properties
   ### bootstrap.properties
   ### bootstrap 上下文配置
   ### 配置服务器 URI
   spring.cloud.config.uri = http://localhost:9090/
   ### 配置客户端应用名称？？前缀？{application}
   spring.cloud.config.name = melody
   ### profile 是激活的配置{profile}
   spring.cloud.config.profile = prod
   ### label 在 Git 中指的是分支的名称{label}
   spring.cloud.config.label = master
   ### URL: {uri}/{application}-{profile}.properties
   ### http://localhost:9090/melody-prod.properties
   ```

3.  设置关键 Endpoint 的敏感性

```properties
### application.properties
### 配置客户端
spring.application.name = config-client
### 全局关闭 Actuator 安全
# management.security.enabled = false
### 细粒度的开放 Actuator Endpoint
### sensitive 关注的是敏感性，安全
endpoints.env.sensitive = false
endpoints.refresh.sensitive = false
#endpoints.beans.sensitive = false
#endpoints.health.sensitive = false
### 
#endpoints.actuator.sensitive = false

```



version



### 动态配置属性 Bean

通过调用`/refresh` Endpoint控制客户端配置更新

* `@RreshScope`关联配置更新时，相关应属性也会刷新，得先判断是否可变化  
* RefreshEndpoint
* ContextRefresher#refresh

#### 实现定时更新客户端

ContextRefresher#refresh

### 健康检查

#### 意义

比如应用可以任意地输出业务健康、系统健康等指标

* 端点uri ：`/health`
* 实现类：`HealthEndpoint`
* 健康指示器：`HealthIndicator `

status：UP/DOWN

`HealthEndpoint`：`HealthIndicator `，1：N 一对多，一个DOWN全DOWN

HalJsonMvcEndpoint



#### 自定义 HealthIndicator 

1. 实现 AbstractHealthIndicator#doHealthCheck

2. 暴露 `MyHealthIndicator ` 为 Bean，用@Bean，最后会被聚合在一起

3. 关闭安全控制（权限问题）

   ```properties
   management.security.enabled = false
   ```


### 其他内容

**hareoas** 业界标准

REST API = /users，/withdraw

HATEOAS = REST 服务器发现的入口，UDDI

...



Spring Boot 激活 `actuator` 需要增加 Hateoas 的依赖：spring-hateoas



### 问题

1. Spring Boot 业界比较稳定的微服务中间件，不过易学难精
2. Git 文件存储方式、分布式的管理系统，Spring Cloud 官方实现基于 Git，达到的理念和 ZK（zookeeper） 一样
3. 若发生配置变更，解决方案可以是重启 Spring Context，考虑关联性问题，逐个服务重启更新。@RefreshScope 最佳实践应用于配置 Bean：开关、数据阈值、文案等等场景
4. 





