# Spring Boot 配置

## DataBinder

## 外部配置

定义
​	Spring Boot 应用的外部配置资源，这些配置资源能够与代码相互配合，避免硬编码 方式，提供应用数据或行为变化的灵活性。

### 类型

- Properties 文件
- YAML 文件
- 环境变量
- Java 系统属性
- 命令行



### 加载顺序

配置源 PropertySource



- 热加载
- 测试
- 命令行
- Servlet 参数（ServletConfig、ServletContext）
- JNDI
- 先系统属性，再环境变量
- application-{profile}.properties（先外后内）
- application.properties（先外后内）
- @PropertySource
- SpringApplication.setDefaultProperties



激活顺序？？？

### 配置引用

- XML 文件
- Annotation
- Java Code



## Profiles 

定义
​	Spring Profiles 提供一种区隔应用配置的方式，让应用配置仅在某种特定的“环境”可用。

使用场景



在jar包用命令行运行时选择 profile
java -jar example.jar --spring.profiles.active=test

在application.properties这个全局配置中配置
在application.properties中添加spring.profiles.active=test

每个独立的profiles的配置方式为以"application-xxx.properties"格式，针对每个不同环境，例如：
application-pro.properties 表示预演环境
application-dev.properties 表示开发环境
application-test.properties 表示测试环境

## 核心API

### 环境相关

Environment

ConfigurableEnvironment

PropertySource

### 配置相关

### Profile 相关



## 装配原理

事件监听模式

PropertySourceLoader#load    find usage 

 event --> ApplicationListener

Event  --- Environment --- PropertySource

## 其他

@RestController（这个注解相当于同时添加@Controller和@ResponseBody注解。）
@EnableAutoConfiguration（@EnableAutoConfiguration作用：Spring Boot会自动根据你jar包的依赖来自动配置项目）



maven <optional>的合理利用，最少依赖

### Springboot加载配置文件的顺序

1. home目录下的devtools全局设置属性（ ~/.spring-boot-devtools.properties ，如果devtools激活）。
2. 测试用例上的@TestPropertySource注解。
3. 测试用例上的@SpringBootTest#properties注解。
4. 命令行参数
5. 来自 SPRING_APPLICATION_JSON 的属性（环境变量或系统属性中内嵌的内联JSON）。
6. ServletConfig 初始化参数。
7. ServletContext 初始化参数。
8. 来自于 java:comp/env 的 JNDI 属性。
9. Java系统属性（System.getProperties()）。
10. 操作系统环境变量。
11. RandomValuePropertySource，只包含 random.* 中的属性。
12. 没有打进 jar 包的 Profile-specific 应用属性（ application-{profile}.properties 和YAML变量）。
13. 打进 jar 包中的 Profile-specific 应用属性（ application-{profile}.properties 和YAML变量）。
14. 没有打进 jar 包的应用配置（ application.properties 和YAML变量）。
15. 打进 jar 包中的应用配置（ application.properties 和YAML变量）。
16. @Configuration 类上的 @PropertySource 注解。
17. 默认属性（使用 SpringApplication.setDefaultProperties 指定）。

全局变量从properties文件读入@Value()






Mybatis现在已经为Springboot提供了支持，我们只需要添加MyBatis-Spring-Boot-Starter这个依赖，它就会为我们去做好以下的事情：

自动检测已有的datasource
创建一个SqlSessionFactoryBean的实例SqlSessionFactory，并将datasource装配进去
创建一个SqlSessionTemplate的实例，并将SqlSessionFactory装配进去
自动扫描你的mapper，将它们连接到SqlSessionTemplate，并将它们注册到Spring的上下文，以便将它们注入到其他的bean中。

@EnableAutoConfiguration
@SpringBootApplication这个注解中已经包含了@EnableAutoCongiguration注解。

去掉多余的bean注入

一些特殊的配置，这时候就可以创建一个WebConfig去定制 （ WebConfig extends WebMvcConfigurerAdapter）



关于日志
logging.file : 输出到指定的文件，可以为相对路径或者绝对路径
logging.path : 与logging.file 是互斥的，指定文件的路径，默认的文件名为 spring.log

多数据源配置：
@Qualifier注解了，qualifier的意思是合格者，通过这个标示，表明了哪个实现类才是我们所需要的，我们修改调用代码，添加@Qualifier注解，需要注意的是@Qualifier的参数名称必须为我们之前定义@Service注解的名称之一！
@ConfigurationProperties(prefix="spring.datasource.primary")
Spring-data-jpa支持



加载时预设环境变量值问题
### 关于Environment的使用

覆盖加载的变量值的几种方法

* 通过增加EnvironmentPostProcessor的实现来实现该功能
* 通过增加PropertySourceLoader的实现来实现该功能
* 重写特定的使用实例
* 覆盖PropertyPlaceholderConfigurer的实现
  以上四种方式以第一种方式为佳,因为其不牵扯到正常的加载逻辑，算是在所有参数加载完后做的一些补充.



Conversion -------- getProperty(xxx, targetType)

**/env 可查看配置加载顺序**

ConfigutableEnvironment

MutablePropertySource

PropertySource 有顺序，所以加载有顺序

PropertySourcesPropertyResolver

### 题外

除数据库，其他大部分配置内容使用注解

controller需与application类同级或子级目录
@Configuration

## 自配置文件

@ImportResource(locations={"classpath:application-bean.xml"})
//@ImportResource(locations={"file:d:/test/application-bean1.xml"})注解 

该注解为springboot提供了springmvc中xml的配置方式。意味着我们完全可以把springMvc框架中xml配置文件中所需要的部分摘抄过来，最后通过@ImportResource注解引入实现。  

springboot它帮我们集成了很多东西，有很多的bean框架会帮我们自动导入，而我们只要直接引用就可以了，如上文的transactionManager对象。之前我是自定义了事务管理器txManager（如下）,导致事务无效，不知道是dataSource引用写错的原因还是事务管理器冲突的原因导致配置事务无效，无法启用。 

SpringBootApplication申明让spring boot自动给程序进行必要的配置，
等价于以默认属性使用@Configuration， @EnableAutoConfiguration和@ComponentScan

配置文件的生效顺序，会对值进行覆盖：

1. @TestPropertySource 注解
2. 命令行参数
3. Java系统属性（System.getProperties()）
4. 操作系统环境变量
5. 只有在random.*里包含的属性会产生一个RandomValuePropertySource
6. 在打包的jar外的应用程序配置文件（application.properties，包含YML和profile变量）
7. 在打包的jar内的应用程序配置文件（application.properties，包含YML和profile变量）
8. 在@Configuration类上的@PropertySource注解
9. 默认属性（使用SpringApplication.setDefaultProperties指定）

Application属性文件，按优先级排序，位置高的将覆盖位置低的

1. 当前目录下的一个/config子目录
2. 当前目录
3. 一个 classpath 下的 /config   包
4. classpath 根路径（root）



```
Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {
    String value() default "";
}

@Configuration 标记的类必须符合下面的要求：

配置类必须以类的形式提供（不能是工厂方法返回的实例），允许通过生成子类在运行时增强（cglib 动态代理）。
配置类不能是 final 类（没法动态代理）。
配置注解通常为了通过 @Bean 注解生成 Spring 容器管理的类，
配置类必须是非本地的（即不能在方法中声明，不能是 private）。
任何嵌套配置类都必须声明为static。
@Bean 方法可能不会反过来创建进一步的配置类（也就是返回的 bean 如果带有 @Configuration，也不会被特殊处理，只会作为普通的 bean）。

@Configuration相当于xml中的<beans>标签，@Bean相当于xml中的<bean>标签
@Bean通过注解一个方法来声明一个bean,默认的情况下，bean的注册名称是方法名称。


@RestController注解相当于@ResponseBody ＋ @Controller合在一起的作用。
是无法跳转页面的   页面中会显示json

@Controller  不加@ResponseBody 才可以跳转页面
跳转页面的前缀和后缀在application.properties配置
```



Docker化部署
编写Dockerfile
首先需要配置pom文件中Docker相关插件


