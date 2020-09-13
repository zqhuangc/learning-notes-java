# Spring Boot四大核心

[九九八八](https://blog.csdn.net/andy_zhang2007)

### CLI

### Starter

- 1、选择已有的starters，在此基础上进行扩展.
- 2、创建自动配置文件并设定META-INF/spring.factories里的内容.spi
- 3、发布你的starter

### Auto Configuration

### 扩展SPI机制

SpringApplication#run

SpringFactoriesLoader#loadFactoryNames 加载配置文件中`spring.factories`指定 key 配置的所有类(根据 ClassLoader，加载对应 jar 包下/META-INF/spring.factories)

* [各种PostProcessor](https://blog.csdn.net/andy_zhang2007/article/details/78595558)





#### ConfigurationClassPostProcessor

ConfigurationClassPostProcessor 是我们实现注解的核心类，改类在spring启动的时候，会去加载该类到spring容器当中，因为该类是BeanDefinitionRegistryPostProcessor的子类，在spring生命周期当中它会去执行其相应的方法。

大致处理流程就是： 
1、先去扫描已经被@Component所注释的类，当然会先判断有没有@Condition相关的注解。 
2、然后递归的取扫描该类中的@ImportResource，@PropertySource，@ComponentScan，@Bean，@Import。一直处理完。



Deferred 延迟

**DeferredImportSelector** 这个实现的类会执行当所有的 @Configuration 注解的 bean 被处理后执行。这个类主要是用来处理 @Conditional 的。

EnableAutoConfigurationImportSelector 实现了 DeferredImportSelector



ConfigurationClassPostProcessor在扫描@Component时候会去判断该类是否有@Import并且判断是不是实现了 **ImportSelector** 或者 **ImportBeanDefinitionRegistrar** 不是的话就按照普通的 @Configuration 放在spring容器当中。



@EnableAutoConfiguration 使用了 @Import，然后通过 selectImports 从 META-INF/spring.factories 中取出想要的配置，将这些类注册到 spring 容器当中

#### 配置的注解

ConfigurationClassPostProcessor 处理注解

@Configuration 

@ConfigurationProperties 

@EnableConfigurationProperties

@ImportResource 

@PropertySource        properties，yaml

@ComponentScan

@Bean

@Import

@Conditional

```
Condition#matches
```

*  @Value  与  @ConfigurableProperties 区别？

@Value 平铺式

@ConfigurableProperties("server.tomcat")  树形

@EnableConfigurableProperties

离应用越近的配置越优先 ？？？？



ConfigFileApplicationContextInitializer

PropertySourceLoader

Loader



#### ImportBeanDefinitionRegistrar

* 如果让你写一个新的jar，要求使用到了spring自动配置如何做？

  1. 我们只需要在自己的jar下面的配置文件INF/spring.factories写入

     ```
     org.springframework.boot.autoconfigure.EnableAutoConfiguration=这里是我们希望spring容器启动时希望自己加载配置的地方。
     ```

  2. 使用@EnableAutoConfiguration注解即可

#### Actuator





## 初识

@RestController（这个注解相当于同时添加@Controller和@ResponseBody注解。）
@EnableAutoConfiguration（@EnableAutoConfiguration作用：Spring Boot会自动根据你jar包的依赖来自动配置项目）


在jar包用命令行运行时选择profile
java -jar example.jar --spring.profiles.active=test

在application.properties这个全局配置中配置
在application.properties中添加spring.profiles.active=test

每个独立的profiles的配置方式为以"application-xxx.properties"格式，针对每个不同环境，例如：
application-pro.properties 表示预演环境
application-dev.properties 表示开发环境
application-test.properties 表示测试环境

### Springboot加载配置文件的顺序

1. home目录下的devtools全局设置属性（ ~/.spring-boot-devtools.properties ，如果devtools激活）。
2. 测试用例上的@TestPropertySource注解。
3. 测试用例上的@SpringBootTest#properties注解。
4. 命令行参数
5. 来自 SPRING_APPLICATION_JSON 的属性（环境变量或系统属性中内嵌的内联JSON）。
6. ServletConfig 初始化参数。
7. ServletContext 初始化参数。
8. 来自于 java:comp/env 的JNDI属性。
9. Java系统属性（System.getProperties()）。
10. 操作系统环境变量。
11. RandomValuePropertySource，只包含 random.* 中的属性。
12. 没有打进jar包的Profile-specific应用属性（ application-{profile}.properties 和YAML变量）。
13. 打进jar包中的Profile-specific应用属性（ application-{profile}.properties 和YAML变量）。
14. 没有打进jar包的应用配置（ application.properties 和YAML变量）。
15. 打进jar包中的应用配置（ application.properties 和YAML变量）。
16. @Configuration 类上的 @PropertySource 注解。
17. 默认属性（使用 SpringApplication.setDefaultProperties 指定）。


全局变量从properties文件读入@Value()


Mybatis现在已经为Springboot提供了支持，我们只需要添加MyBatis-Spring-Boot-Starter这个依赖，它就会为我们去做好以下的事情：

自动检测已有的datasource
创建一个SqlSessionFactoryBean的实例SqlSessionFactory，并将datasource装配进去
创建一个SqlSessionTemplate的实例，并将SqlSessionFactory装配进去
自动扫描你的mapper，将它们连接到SqlSessionTemplate，并将它们注册到Spring的上下文，以便将它们注入到其他的bean中。


@EnableAutoCongiguration
@SpringBootApplication这个注解中已经包含了@EnableAutoCongiguration注解。

去掉多余的bean注入

一些特殊的配置，这时候就可以创建一个WebConfig去定制 （ WebConfig extends WebMvcConfigurerAdapter）





关于启动时注解
@SpringBootApplication same as @Configuration @EnableAutoConfiguration @ComponentScan provides aliases to customize the attributes of @EnableAutoConfiguration and @ComponentScan

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



### 题外

除数据库，其他大部分配置内容使用注解

controller需与application类同级或子级目录
@Configuration

# 自配置文件

@ImportResource(locations={"classpath:application-bean.xml"})
//@ImportResource(locations={"file:d:/test/application-bean1.xml"})注解 
该注解为springboot提供了springMvc中xml的配置方式。意味着我们完全可以把springMvc框架中xml配置文件中所需要的部分摘抄过来，最后通过@ImportResource注解引入实现。  

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
3. 一个classpath下的/config包
4. classpath根路径（root）



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

