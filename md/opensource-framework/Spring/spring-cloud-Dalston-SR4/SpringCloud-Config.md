# 基于 Spring Cloud Dalston.SR4 版本

**不同版本可能会有修改，具体以相应文档版本为主**

# Bootstrap 上下文

是 Spring Cloud 新引入的，与传统 Spring 上下文相同，系

ConfigurableApplicationContext 实例，由 BootstrapApplicationListener 在监听
ApplicationEnvironmentPreparedEvent 时创建。
Spring 事件/监听器模式
ApplicationEvent / ApplicationListenerotstrap 上下文

SpringApplication 是 Spring Boot 引导启动类，与 Spring 上下文、事件、监听器以及环境等组件
关系紧密，其中提供了了控制 Spring Boot 应用特征的行为方法。

 Spring Boot 应用运行监听器

SpringApplicationRunListener

理解 Spring Boot 事件

事件触发器：EventPublishingRunListener#environmentPrepared

ApplicationStartedEvent

ApplicationEnvironmentPreparedEvent

ApplicationPreparedEvent

ApplicationReadyEvent / ApplicationFailedEvent



Spring Boot 上下文
非 Web 应用：AnnotationConfigApplicationContext
Web 应用：AnnotationConfigEmbeddedWebApplicationContext

Spring Cloud 上下文：Bootstrap （父）





EnvironmentChangeEvent

spring-cloud-context

spring-cloud-commons：服务发现，负载平衡和熔断等模式的一个通用的抽象层

服务注册 `ServiceRegistry`

EnvironmentRepository

eureka.instance.metadataMap

# Environment

Environment 是一种在容器器内以 配置（Profile） 和 属性（Properties） 为模型的应用环境抽象整合。

一般应用 ： StandardEnvironment
Web 应用： StandardServletEnvironment



`Environment`: `PropertySources` =  1:1

`PropertySources`: `PropertySource` = 1:N

组合属性：PropertySources
单一属性：PropertySource

[0] `PropertySource` （Map）

​	spring.application.name = spring-cloud-config-client

[1] `PropertySource`（Map）

​	spring.application.name = spring-cloud-config-client-demo



### Spring Boot 配置文件



#### application.properties 或 application.xml



加载器：`PropertiesPropertySourceLoader`



#### application.yml 或者 application.yaml

加载器：`YamlPropertySourceLoader`



## Environment 端点



### 请求 URI ：`/env`

/env /xxx.xxx.xxx 操作某具体属性

数据来源：`EnvironmentEndpoint`

Controller 来源：`EnvironmentMvcEndpoint`



invoke() 调用

## Bootstrap 配置



参考`BootstrapApplicationListener` 实现

 

> 注：程序启动参数的加载逻辑：
>
> `SpringApplication#configurePropertySources()`



## Bootstrap 配置文件

* BootstrapApplicationListener

```java
String configName = environment.resolvePlaceholders("${spring.cloud.bootstrap.name:bootstrap}");
```

当 `spring.cloud.bootstrap.name` 存在时，使用该配置项，否则，使用 "bootstrap" 作为默认



```properties
## application.properties
## 通过调整 spring.cloud.bootstrap.enabled = false，尝试关闭 bootstrap 上下文
## 实际测试结果，没有效果，加载相比慢了
spring.cloud.bootstrap.enabled = false
```

> 注意：`BootstrapApplicationListener` 加载实际早于 `ConfigFileApplicationListener`
>
> 原因是：
>
>  `ConfigFileApplicationListener` 的 Order =  Ordered.HIGHEST_PRECEDENCE + 10（第十一位）
>
> `BootstrapApplicationListener`的 Order = Ordered.HIGHEST_PRECEDENCE + 5（第六位）
>
>
>
> `ConfigFileApplicationListener`在 Spring Boot 场景中，用于读取默认以及Profile 关联的配置文件（application.properties）

如果需要调整 控制 Bootstrap 上下文行为配置，需要更高优先级，也就是说 Order 需要  < Ordered.HIGHEST_PRECEDENCE + 5 (越小越优先)，比如使用程序启动参数：

```
--spring.cloud.bootstrap.enabld = true
```





### 调整 Bootstrap 配置



#### 调整 Bootstrap 配置文件名称



调整程序启动参数

```
--spring.cloud.bootstrap.name=spring-cloud
```

bootstrap 配置文件名称发生了改变"spring-cloud"，现有三个文件：

- `application.properties`
  - **spring.application.name = spring-cloud-config-client**
- `bootstrap.properties`
  - spring.application.name = spring-cloud-config-client-demo
- `spring-cloud.properties`
  - **spring.application.name = spring-cloud**



运行结果（部分）：

```json
"applicationConfig: [classpath:/application.properties]": {
    "spring.cloud.bootstrap.enabled": "false",
    "endpoints.env.sensitive": "false",
    "spring.application.name": "spring-cloud-config-client"
  },
  ...
  "applicationConfig: [classpath:/spring-cloud.properties]": {
    "spring.application.name": "spring-cloud-config-client"
  }
```



#### 调整 Bootstrap 配置文件路径



保留 **Bootstrap 配置文件名称**  程序启动参数：

```properties
--spring.cloud.bootstrap.name=spring-cloud
```



调整 **Bootstrap 配置文件路径** 程序启动参数：

```properties
--spring.cloud.bootstrap.location=config
```



现有四个文件：

- `application.properties`
  - **spring.application.name = spring-cloud-config-client**
- `bootstrap.properties`
  - spring.application.name = spring-cloud-config-client-demo
- `spring-cloud.properties`
  - **spring.application.name = spring-cloud**
- `config/spring-cloud.properties`
  - **spring.application.name = spring-cloud-2**



实际结果：

```json
  "applicationConfig: [classpath:/application.properties]": {
    "spring.cloud.bootstrap.enabled": "false",
    "endpoints.env.sensitive": "false",
    "spring.application.name": "spring-cloud-config-client"
  },
  ...
  "applicationConfig: [classpath:/config/spring-cloud.properties]": {
    "spring.application.name": "spring-cloud-config-client"
  },
  "applicationConfig: [classpath:/spring-cloud.properties]": {
    "spring.application.name": "spring-cloud-config-client"
  },
```



#### 覆盖远程配置属性



默认情况，Spring Cloud 是允许覆盖的，`spring.cloud.config.allowOverride=true`



通过程序启动参数，调整这个值为"**false**"

```properties
--spring.cloud.config.allowOverride=false
```



启动后，重新Postman 发送 POST 请求，调整`spring.application.name` 值为 "**spring-cloud-new**"

> 注意官方文档的说明：the remote property source has to grant it permission by setting `spring.cloud.config.allowOverride=true` (it doesn’t work to set this locally).
>
> 本地修改无效



#### 自定义 Bootstrap 配置

1. 创建`META-INF/spring.factories`文件（类似于 Spring Boot 自定义 Starter）

2. 自定义 Bootstrap 配置 Configuration

   ```java
   package com.segmentfault.springcloudlesson2.boostrap;
   
   import org.springframework.context.ApplicationContextInitializer;
   import org.springframework.context.ConfigurableApplicationContext;
   import org.springframework.context.annotation.Configuration;
   import org.springframework.core.env.ConfigurableEnvironment;
   import org.springframework.core.env.MapPropertySource;
   import org.springframework.core.env.MutablePropertySources;
   import org.springframework.core.env.PropertySource;
   
   import java.util.HashMap;
   import java.util.Map;
   
   /**
    * Bootstrap 配置 Bean
    *
    * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
    * @since Configuration
    */
   @Configuration
   public class MyConfiguration implements ApplicationContextInitializer {
   
       @Override
       public void initialize(ConfigurableApplicationContext applicationContext) {
           
           // 从 ConfigurableApplicationContext 获取 ConfigurableEnvironment 实例
           ConfigurableEnvironment environment = applicationContext.getEnvironment();
           // 获取 PropertySources
           MutablePropertySources propertySources = environment.getPropertySources();
           // 定义一个新的 PropertySource，并且放置在首位
           propertySources.addFirst(createPropertySource());
   
       }
   
       private PropertySource createPropertySource() {
   
           Map<String, Object> source = new HashMap<>();
   
           source.put("name", "小马哥");
   
           PropertySource propertySource = new MapPropertySource("my-property-source", source);
   
           return propertySource;
   
       }
   }
   ```

3. 配置`META-INF/spring.factories`文件，关联Key `org.springframework.cloud.bootstrap.BootstrapConfiguration`

   ```properties
   org.springframework.cloud.bootstrap.BootstrapConfiguration= \
   com.segmentfault.springcloudlesson2.boostrap.MyConfiguration
   ```

   ​


#### 自定义 Bootstrap 配置属性源



1. 实现`PropertySourceLocator`

   ```java
   package com.segmentfault.springcloudlesson2.boostrap;
   
   import org.springframework.cloud.bootstrap.config.PropertySourceLocator;
   import org.springframework.core.env.*;
   
   import java.util.HashMap;
   import java.util.Map;
   
   /**
    * 自定义 {@link PropertySourceLocator} 实现
    *
    * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
    * @since PropertySourceLocator
    */
   public class MyPropertySourceLocator implements PropertySourceLocator {
   
       @Override
       public PropertySource<?> locate(Environment environment) {
   
           if (environment instanceof ConfigurableEnvironment) {
   
               ConfigurableEnvironment configurableEnvironment = ConfigurableEnvironment.class.cast(environment);
   
               // 获取 PropertySources
               MutablePropertySources propertySources = configurableEnvironment.getPropertySources();
               // 定义一个新的 PropertySource，并且放置在首位
               propertySources.addFirst(createPropertySource());
   
           }
           return null;
       }
   
       private PropertySource createPropertySource() {
   
           Map<String, Object> source = new HashMap<>();
   
           source.put("spring.application.name", "小马哥的 Spring Cloud 程序");
           // 设置名称和来源
           PropertySource propertySource = new MapPropertySource("over-bootstrap-property-source", source);
   
           return propertySource;
   
       }
   }
   ```

2. 配置`META-INF/spring.factories`

   ```properties
   org.springframework.cloud.bootstrap.BootstrapConfiguration= \
   com.segmentfault.springcloudlesson2.boostrap.MyConfiguration,\
   com.segmentfault.springcloudlesson2.boostrap.MyPropertySourceLocator
   ```

## POST 方式修改配置

## 预备知识

发布者和订阅者：1:N

发布者和订阅者：N:M

#### 事件/监听模式

`java.util.EvenListener` 事件监听接口（标记）

`java.util.EventObject` 事件对象

    * 事件对象关联事件源（source）

#### 事件/监听模式

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

> 加载的优先级 高于` ConfigFileApplicaitonListener`，所以 application.properties 文件即使定义参数，也配置不到`bootstrap.properties`
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



Actuator 中文直译为“传动装置”，在 Spring Boot 使⽤用场景中表示为“生产而准备的特性”
（Production-ready features），这些特性通过 HTTP 端口的形式，帮助相关人员管理和监控应用。大致上可以归类为：
监控类：“端点信息”、“应用信息”、“外部化配置信息”、“指标信息”、“健康检查”、“Bean 管
理理”、“Web URL 映射管理理”、“Web URL 跟踪”
管理类：“外部化配置”、“日志配置”、“线程dump”、“堆dump”、“关闭应用”
注意：Spring Boot 1.5 开始 Actuator 增强了了安全能力



Spring Cloud 扩展 Actuator Endpoints
上下文重启：/restart
暂停：/pause
恢复：/resume

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
5. 如何防止 Order 一样，Spring Boot 和 Spring Cloud 没办法，Spring Security 通过异常实现
6. 服务监控跟鹰眼类似
7. BootstrapApplicationListener是引入 cloud组件来用
8.  pom.xml 引入 spring-boot-starter-config

##  书籍推荐

翟永超《Spring Cloud 微服务实战》

## 学习方向（个人看法）

配置文件加载顺序对配置内容加载的影响（有助于对执行顺序的了解）？

远程配置如何获取，获取到配置的执行顺序？

配置覆盖性？



# Spring Cloud 配置服务器



EnvironmentRepository
Spring Cloud 配置服务器管理多个客户端应用的配置信息，然而这些配置信息需要通过一定的规则获取。Spring Cloud Config Sever 提供 EnvironmentRepository 接口供客户端应用获取，其中获取维度有三：
• {application} : 配置客户端应用名称，即配置项：spring.application.name
• {profile}： 配置客户端应用当前激活的Profile，即配置项：spring.profiles.active
• {label}： 配置服务端标记的版本信息，如 Git 中的分支名



* 理解服务端配置映射
  /{application}/{profile}[/{label}]
  /{application}-{profile}.yml
  /{label}/{application}-{profile}.yml
  /{application}-{profile}.properties
  /{label}/{application}-{profile}.properties

## 搭建 Spring Cloud Config Server



### 基于文件系统（File System）



#### 创建本地仓库



1. 激活应用配置服务器

   在引导类上标注`@EnableConfigServer`

   ```java
   package com.segmentfault.springcloudlesson3configserver;
   
   import org.springframework.boot.SpringApplication;
   import org.springframework.boot.autoconfigure.SpringBootApplication;
   import org.springframework.cloud.config.server.EnableConfigServer;
   
   @SpringBootApplication
   @EnableConfigServer
   public class SpringCloudLesson3ConfigServerApplication {
   
   	public static void main(String[] args) {
   		SpringApplication.run(SpringCloudLesson3ConfigServerApplication.class, args);
   	}
   }
   ```

2. 创建本地目录

> 理解 Java 中的 ${user.dir}，在 IDE 中是指的当前项目物理路径

在 IDEA 中`src/main/resources`目录下，创建一个名为“configs”，它的绝对路径：

`${user.dir}/src/main/resources/config`

1. 配置 git 本地仓库 URI

   ```properties
   ## 配置服务器文件系统git 仓库
   ## ${user.dir} 减少平台文件系统的不一致
   spring.cloud.config.server.git.uri = ${user.dir}/src/main/resources/configs
   ```

2. 给应用名称为 "segmentfault" 创建三个环境的配置文件

   ```
   -rw-r--r--  1 segmentfault-prod.properties
   -rw-r--r--  1 segmentfault-test.properties
   -rw-r--r--  1 segmentfault.properties
   ```

   三个文件的环境 profile 分别（从上至下）是：`prod`、`test`、`default`

3. 初始化本地 git 仓库

   ```
   git init
   git add .
   git commit -m "First commit"
   [master (root-commit) 9bd81bd] First commit
    3 files changed, 9 insertions(+)
    create mode 100644 segmentfault-prod.properties
    create mode 100644 segmentfault-test.properties
    create mode 100644 segmentfault.properties
   ```



#### 测试配置服务器

通过浏览器测试应用为"segmentfault"，Profile为：“test”的配置内容 : http://localhost:9090/segmentfault/test

```json
{
  "name": "segmentfault",
  "profiles": [
    "test"
  ],
  "label": null,
  "version": "9bd81bdd024832a8bb207a8133ff15e7d2baabb6",
  "state": null,
  "propertySources": [
    {
      "name": "/Users/Mercy/Downloads/spring-cloud-lesson-3-config-server/src/main/resources/configs/segmentfault-test.properties",
      "source": {
        "name": "segumentfault-test"
      }
    },
    {
      "name": "/Users/Mercy/Downloads/spring-cloud-lesson-3-config-server/src/main/resources/configs/segmentfault.properties",
      "source": {
        "name": "segumentfault"
      }
    }
  ]
}
```

请注意：当指定了profile 时，默认的 profile（不指定）配置信息也会输出：

```json
 "propertySources": [
    {
      "name": "/Users/Mercy/Downloads/spring-cloud-lesson-3-config-server/src/main/resources/configs/segmentfault-test.properties",
      "source": {
        "name": "segumentfault-test"
      }
    },
    {
      "name": "/Users/Mercy/Downloads/spring-cloud-lesson-3-config-server/src/main/resources/configs/segmentfault.properties",
      "source": {
        "name": "segumentfault"
      }
    }
  ]
```





## 基于远程 Git 仓库



1. 激活应用配置服务器

在引导类上标注`@EnableConfigServer`

1. 配置远程 Git 仓库地址

   ```properties
   ## 配置服务器远程 Git 仓库（GitHub）
   spring.cloud.config.server.git.uri = https://github.com/mercyblitz/tmp
   ```

2. 本地 clone 远程Git 仓库

   ```
   git clone https://github.com/mercyblitz/tmp.git
   ```

   ```
   $ ls -als
   total 24
   0 drwxr-xr-x   6 Mercy  staff  192 11  3 21:16 .
   0 drwx------+ 12 Mercy  staff  384 11  3 21:16 ..
   0 drwxr-xr-x  12 Mercy  staff  384 11  3 21:16 .git
   8 -rw-r--r--   1 Mercy  staff   40 11  3 21:16 README.md
   8 -rw-r--r--   1 Mercy  staff   27 11  3 21:16 a.properties
   8 -rw-r--r--   1 Mercy  staff   35 11  3 21:16 tmp.properties
   ```

3. 给应用"segmentfault"创建三个环境的配置文件

   ```
   -rw-r--r--   1 segmentfault-prod.properties
   -rw-r--r--   1 segmentfault-test.properties
   -rw-r--r--   1 segmentfault.properties
   ```

4. 提交到 远程 Git 仓库

   ```
   $ git add segmentfault*.properties
   $ git commit -m "Add segmentfault config files"
   [master 297989f] Add segmentfault config files
    3 files changed, 9 insertions(+)
    create mode 100644 segmentfault-prod.properties
    create mode 100644 segmentfault-test.properties
    create mode 100644 segmentfault.properties
   $ git push
   Counting objects: 5, done.
   Delta compression using up to 8 threads.
   Compressing objects: 100% (5/5), done.
   Writing objects: 100% (5/5), 630 bytes | 630.00 KiB/s, done.
   Total 5 (delta 0), reused 0 (delta 0)
   To https://github.com/mercyblitz/tmp.git
      d2b742b..297989f  master -> master
   ```

5. 配置强制拉去内容

```properties
## 强制拉去 Git 内容
spring.cloud.config.server.git.force-pull = true
```



1. 重启应用



## 配置 Spring Cloud 配置客户端

Spring Cloud Config Client
Spring Cloud 配置客户端提供连接 Spring Cloud 服务端，并且**获取订阅的配置信息**。（文件直接加载到内存）
配置Spring Cloud 配置客户端
创建 bootstrap.yml 或者 bootstrap.properties
配置 spring.cloud.config.* 信息



1. 创建 Spring Cloud Config Client 应用

   创建一个名为 `spring-cloud-lesson-3-config-client` 应用

2. ClassPath 下面创建 bootstrap.properties

3. 配置  bootstrap.properties

   配置 以`spring.cloud.config.` 开头配置信息

   ```properties
   ## 配置客户端应用关联的应用
   ## spring.cloud.config.name 是可选的
   ## 如果没有配置，采用 ${spring.application.name}
   spring.cloud.config.name = segmentfault
   ## 关联 profile
   spring.cloud.config.profile = prod
   ## 关联 label
   spring.cloud.config.label = master
   ## 配置 配置服务器URI Http方式
   spring.cloud.config.uri = http://127.0.0.1:9090/
   
   
   ## eureka client 服务发现方式
   ## 激活 Config 服务器发现
   #spring.cloud.config.discovery.enabled = true
   ## 配置 Config 服务器的应用名称（Service ID）
   #spring.cloud.config.discovery.serviceId = spring-cloud-config-server
   ```

   application.properties 信息

   ```properties
   ## 配置客户端应用名称
   spring.application.name = spring-cloud-config-client
   
   ## 配置客户端应用服务端口
   server.port = 8080
   
   ## 关闭管理端actuator 的安全
   ## /env /health 端口完全开放
   management.security.enabled = false
   ```

4. 启动应用

```
[           main] c.c.c.ConfigServicePropertySourceLocator : Fetching config from server at: http://127.0.0.1:9090/
[           main] c.c.c.ConfigServicePropertySourceLocator : Located environment: name=segmentfault, profiles=[prod], label=master, version=15342a7ecdb59b691a8dd62d6331184cca3754f4, state=null
[           main] b.c.PropertySourceBootstrapConfiguration : Located property source: CompositePropertySource [name='configService', propertySources=[MapPropertySource {name='configClient'}, MapPropertySource {name='https://github.com/mercyblitz/tmp/segmentfault-prod.properties'}, MapPropertySource {name='https://github.com/mercyblitz/tmp/segmentfault.properties'}]]
```



### 测试 Spring Cloud 配置客户端

通过浏览器访问 http://localhost:8080/env



```json
"configService:configClient": {
  "config.client.version": "15342a7ecdb59b691a8dd62d6331184cca3754f4"
},
"configService:https://github.com/mercyblitz/tmp/segmentfault-prod.properties": {
  "name": "segumentfault.com"
},
"configService:https://github.com/mercyblitz/tmp/segmentfault.properties": {
  "name": "segumentfault.com"
}
```



通过具体的配置项`name`: http://localhost:8080/env/name

```json
{
  "name": "segumentfault.com"
}
```



## 动态配置属性Bean

#### @RefreshScope

/refresh Endpoint
ContextRefresher



通过调用`/refresh` Endpoint控制客户端配置更新

- `@RreshScope`关联配置更新时，相关应属性也会刷新，得先判断是否可变化  
  可用于标注 @configurationproperties 的类
- RefreshEndpoint
- ContextRefresher#refresh  拉取远程，刷新上下文，对比（修改内存的内容）

* 实现定时更新客户端 --> ContextRefresher#refresh



EndpointDiscoverer：

```
BeanFactoryUtils.beanNamesForAnnotationIncludingAncestors(this.applicationContext,
      Endpoint.class);// 获取所有标注了 @Endpoint 的 spring bean 名称
DefaultListableBeanFactory#findAnnotationOnBean
```



@WebEndpoint

EndpointWebMvcManagementContextConfiguration

EndpointHandlerMapping

#### 定义配置属性Bean `User`

```java
package com.segmentfault.springcloudlesson3configclient.domain;

import org.springframework.boot.context.properties.ConfigurationProperties;

/**
 * 用户模型
 *
 * @author <a href="mailto:mercyblitz@gmail.com">Mercy</a>
 * @since 1.0.0
 */
@ConfigurationProperties(prefix = "sf.user")
public class User {

    private Long id;

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



#### 将 `User` 关联配置项

```pr
## 用户的配置信息
## 用户 ID
sf.user.id = 1
## 用户名称
sf.user.name = xiaomage
```



> 通过浏览器访问
>
> - http://localhost:8080/env/sf.user.*
>
> ```json
> {
>  "sf.user.id": "1",
>  "sf.user.name": "xiaomage"
> }
> ```
>
> - http://localhost:8080/user
>
> ```json
> {
>  "id": 1,
>  "name": "xiaomage"
> }
> ```
>
>



### 通过 Postman 调整配置项

POST 方法提交参数 sf.user.id = 007 、sf.user.name = xiaomage 到 `/env`

```json
sf.user.id = 007
sf.user.name = xiaomage
```



调整后，本地的http://localhost:8080/user 的内容变化:

```json
{
  "id": 7,
  "name": "mercyblitz"
}
```



问题，如果`spring-cloud-config-client`需要调整所有机器的配置如何操作？

> 注意，配置客户端应用的所关联的分布式配置内容，优先于传统 `application.properties`（application.yml）或者 `bootstrap.properties`（bootstrap.yml）



#### 调整配置服务器配置信息：`segmentfault-prod.properties`



spring-cloud-config-monitor



### 健康检查

#### 意义

比如应用可以任意地输出业务健康、系统健康等指标

- 端点uri ：`/health`
- 实现类：`HealthEndpoint`
- 健康指示器：`HealthIndicator `

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

HealthBuilder 构建 Health

### 管理端点



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

花式修改配置文件，加载顺序问题，以及哪些配置文件被加载





### 构建 Spring Cloud 配置服务器

#### 实现步骤

1. 在 Configuration Class 标记 `@EnableConfigServer`

2. 配置文件目录（基于 git）

   1. melody(-default).properties（默认） //默认环境，跟着代码仓库
   2. melody-dev.properties(profile = "dev")
   3. melody-test.properties(profile = "test") // 测试环境
   4. melody-staging.properties(profile = "staging") // 预发布环境
   5. melody-prod.properties(profile = "prod") // 生产环境

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

2. `bootstrap.properties` 或者 `bootstrap.yaml`文件中配置客户端信息

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

3. 设置关键 Endpoint 的敏感性

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