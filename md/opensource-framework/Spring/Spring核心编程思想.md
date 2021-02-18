《小马哥讲Spring核心编程思想》课程目录

[TOC]



# 第一章：Spring Framework总览（Overview）

## 01 Spring 特性总览：核心特性、数据存储、Web技术、框架整合与测试

* 核心特性（Core）
  * IoC 容器（IoC Container）
  * Spring 事件（Events）
  * 资源管理（Resources）
  * 国际化（i18n）
  * 校验（Validation）
  * 数据绑定（Data Binding）
  * 类型装换（Type Conversion）
  * Spring 表达式（Spring Express Language）
  * 面向切面编程（AOP）



* 数据存储
  * JDBC
  * 事务抽象（Transactions）
  * DAO 支持（DAO Support）
  * O/R映射（O/R Mapping）
  * XML 编列（XML Marshalling）



* Web技术
  * Web Servlet 技术栈
    * Spring MVC
    * WebSocket
    * SockJS
  * Web Reactive 技术栈
    * Spring WebFlux
    * WebClient
    * WebSocket



* 框架整合
  * 远程调用（Remoting）
  * Java 消息服务（JMS）
  * Java 连接架构（ JCA）
  * Java 管理扩展（JMX）
  * Java 邮件客户端（Email）
  * 本地任务（Tasks）
  * 本地调度（Scheduling）
  * 缓存抽象（Caching）
  * Spring 测试（Testing）





* 测试

  * 模拟对象（Mock Objects）
  * TestContext 框架（TestContext Framework）
  * Spring MVC 测试（Spring MVC Test）
  * Web 测试客户端（WebTestClient）


## 02 Spring 版本特性：Spring 各个版本引入了哪些新特性？

| Spring Framework  版本 | Java  标准版 | Java  企业版          |
| ---------------------- | ------------ | --------------------- |
| 1.x                    | 1.3+         | J2EE 1.3 +            |
| 2.x                    | 1.4.2+       | J2EE 1.3 +            |
| 3.x                    | 5+           | J2EE 1.4 和 Java EE 5 |
| 4.x                    | 6+           | Java EE 6 和 7        |
| 5.x                    | 8+           | Java EE 7             |



## 03 Spring 模块化设计：Spring 功能特性如何在不同模块中组织？

• spring-aop
• spring-aspects
• spring-context-indexer
• spring-context-support
• spring-context
• spring-core
• spring-expression
• spring-instrument
• spring-jcl
• spring-jdbc
• spring-jms
• spring-messaging
• spring-orm
• spring-oxm
• spring-test
• spring-tx
• spring-web
• spring-webflux
• spring-webmvc
• spring-websocket



## 04 Java 语言特性运用：各种 Java 语法特性是怎样被 Spring 各种版本巧妙运用的？

![Java-version-feature](..\..\image\Java-version-feature-2018.PNG)

  

- Java 5 语法特性

| 语法特性               | Spring 支持版本 | 代表实现                   |
| ---------------------- | --------------- | -------------------------- |
| 注解（Annotation）     | 1.2 +           | @Transactional             |
| 枚举（Enumeration）    | 1.2 +           | Propagation                |
| for-each 语法          | 3.0 +           | AbstractApplicationContext |
| 自动装箱（AutoBoxing） | 3.0 +           |                            |
| 泛型（Generic）        | 3.0 +           | ApplicationListener        |



- Java 6 语法特性

| 语法特性         | Spring 支持版本 | 代表实现 |
| ---------------- | --------------- | -------- |
| 接口 @Override法 | 4.0 +           |          |



- Java 7 语法特性

| 语法特性                | Spring 支持版本 | 代表实现                    |
| ----------------------- | --------------- | --------------------------- |
| Diamond 语法            | 5.0 +           | DefaultListableBeanFactory  |
| try-with-resources 语法 | 5.0 +           | ResourceBundleMessageSource |



* Java 8 语法特性

| 语法特性    | Spring 支持版本 | 代表实现                      |
| ----------- | --------------- | ----------------------------- |
| Lambda 语法 | 5.0 +           | PropertyEditorRegistrySupport |
|             |                 |                               |



## 05 JDK API 实践：Spring 怎样取舍 Java I/O、集合、反射、动态代理等 API 的使用？

![Jdk](..\..\image\Java-jdk-api.PNG)

* < Java 5 API

| API 类型                  | Spring 支持版本 | 代表实现                   |
| ------------------------- | --------------- | -------------------------- |
| 反射（Reflection）        | 1.0 +           | MethodMatcher              |
| Java Beans                | 1.0 +           | CachedIntrospectionResults |
| 动态代理（Dynamic Proxy） | 1.0 +           | JdkDynamicAopProxy         |



* Java 5 API

| API 类型               | Spring 支持版本 | 代表实现                   |
| ---------------------- | --------------- | -------------------------- |
| XML 处理（DOM,SAX...） | 1.0 +           | XmlBeanDefinitionReader    |
| Java 管理扩展（JMX）   | 1.2 +           | @ManagedResource           |
| Instrumentation        | 2.0 +           | InstrumentationSavingAgent |
| 并发框架（J.U.C）      | 3.0 +           | ThreadPoolTaskScheduler    |
| 格式化（Formatter）    | 3.0 +           | DateFormatter              |



* Java 6 API

| API 类型                      | Spring 支持版本 | Spring 支持版本                   |
| ----------------------------- | --------------- | --------------------------------- |
| JDBC 4.0（JSR 221）           | 1.0 +           | JdbcTemplate                      |
| Common Annotations（JSR 250） | 2.5 +           | CommonAnnotationBeanPostProcessor |
| JAXB 2.0（JSR 222）           | 3.0 +           | Jaxb2Marshaller                   |
| Scripting in JVM（JSR 223）   | 4.2 +           | StandardScriptFactory             |
| 可插拔注解处理 API（JSR 269） | 5.0 +           | @Indexed                          |
| Java Compiler API（JSR 199）  | 5.0 +           | TestCompiler（单元测试）          |



* Java 7 API

| API 类型                  | Spring 支持版本 | 代表实现                |
| ------------------------- | --------------- | ----------------------- |
| Fork/Join 框架（JSR 166） | 3.1 +           | ForkJoinPoolFactoryBean |
| NIO 2（JSR 203）          | 4.0 +           | PathResource            |



* Java 8 API

| API 类型                      | Spring 支持版本 | 代表实现                             |
| ----------------------------- | --------------- | ------------------------------------ |
| Date and Time API（JSR 310）  | 4.0 +           | DateTimeContext                      |
| 可重复 Annotations（JSR 337） | 4.0 +           | @PropertySources                     |
| Stream API（JSR 335）         | 4.2 +           | StreamConverter                      |
| CompletableFuture（J.U.C）    | 4.2 +           | CompletableToListenableFutureAdapter |



## 06 Java EE API 整合：为什么 Spring 要与“笨重”的 Java EE 共舞？

* Java EE Web 技术相关

| JSR 规范                  | Spring 支持版本 | 代表实现                          |
| ------------------------- | --------------- | --------------------------------- |
| Servlet + JSP(JSR 035）   | 1.0 +           | DispatcherServlet                 |
| JSTL(JSR 052)             | 1.0 +           | JstlView                          |
| JavaServer Faces(JSR 127) | 1.1 +           | FacesContextUtils                 |
| Portlet(JSR 168)          | 2.0 - 4.2       | DispatcherPortlet                 |
| SOAP(JSR 067)             | 2.5 +           | SoapFaultException                |
| WebServices(JSR 109)      | 2.5 +           | CommonAnnotationBeanPostProcessor |
| WebSocket(JSR 356)        | 4.0 +           | WebSocketHandler                  |



* Java EE 数据存储相关

| JSR 规范                   | Spring 支持版本 | 代表实现              |
| -------------------------- | --------------- | --------------------- |
| JDO(JSR 12)                | 1.0 - 4.2       | JdoTemplate           |
| JTA(JSR 907)               | 1.0 +           | JtaTransactionManager |
| JPA(EJB 3.0 JSR 220的成员) | 2.0 +           | JpaTransactionManager |
| Java Caching API(JSR 107)  | 3.2 +           | JCacheCache           |



* Java EE Bean 技术相关

| JSR 规范                               | Spring 支持版本 | 代表实现                             |
| -------------------------------------- | --------------- | ------------------------------------ |
| JMS(JSR 914)                           | 1.1 +           | JmsTemplate                          |
| EJB 2.0 (JSR 19)                       | 1.0 +           | AbstractStatefulSessionBean          |
| Dependency Injection for Java(JSR 330) | 2.5 +           | AutowiredAnnotationBeanPostProcessor |
| Bean Validation(JSR 303)               | 3.0 +           | LocalValidatorFactoryBean            |



## 07 Spring 编程模型：Spring 实现了哪些编程模型？

![Jdk](..\..\image\Spring编程模型.PNG)

## 08 Spring 核心价值：我们能从 Spring Framework 中学到经验和教训呢？



## 09 面试题精选

沙雕面试题 - 什么是 Spring Framework？

答：Spring makes it easy to create Java enterprise applications.
It provides everything you need to embrace the Java language
in an enterprise environment, with support for Groovy and
Kotlin as alternative languages on the JVM, and with the
flexibility to create many kinds of architectures depending on
an application’s needs.



996 面试题 - Spring Framework 有哪些核心模块？

答：
spring-core：Spring 基础 API 模块，如资源管理，泛型处理
spring-beans：Spring Bean 相关，如依赖查找，依赖注入
spring-aop : Spring AOP 处理，如动态代理，AOP 字节码提升
spring-context : 事件驱动、注解驱动，模块驱动等
spring-expression：Spring 表达式语言模块



# 第二章：重新认识 loC

## 01 loC 发展简介：你可能对 loC 有些误会？



## 02 loC 主要实现策略：面试官总问 loC 和 DI 的区别，他真的理解吗？



## 03 loC 容器的职责：loC 除了依赖注入，还涵盖哪些职责呢？

> 通用职责
> • 依赖处理
> ​	• 依赖查找
> ​	• 依赖注入
> • 生命周期管理
> ​	• 容器
> ​	• 托管的资源（Java Beans 或其他资源）
> • 配置
> ​	• 容器
> ​	• 外部化配置
> ​	• 托管的资源（Java Beans 或其他资源）

## 04 loC 容器的实现：loC 容器是开源框架的专利吗？

> 主要实现
> • Java SE
> ​	• Java Beans
> ​	• Java ServiceLoader SPI
> ​	• JNDI（Java Naming and Directory Interface）
> • Java EE
> ​	• EJB（Enterprise Java Beans）
> ​	• Servlet
> • 开源
> ​	• Apache Avalon（http://avalon.apache.org/closed.html）
> ​	• PicoContainer（http://picocontainer.com/）
> ​	• Google Guice（https://github.com/google/guice）
> ​	• Spring Framework（https://spring.io/projects/spring-framework）

## 05 传统 loC 容器实现：JavaBeans 也是 loC 容器吗？

> Java Beans 作为 IoC 容器
> • 特性
> ​	• 依赖查找
> ​	• 生命周期管理
> ​	• 配置元信息
> ​	• 事件
> ​	• 自定义
> ​	• 资源管理
> ​	• 持久化
> • 规范
> ​	• JavaBeans：https://www.oracle.com/technetwork/java/javase/tech/index-jsp-138795.html
> ​	• BeanContext：https://docs.oracle.com/javase/8/docs/technotes/guides/beans/spec/beancontext.html
>
>
>
> 看了PropertyEditor相关资料以及Spring中的应用，其实这里相当于将一个字符串内容，通某些中间手段的转换，转换成你想要的类型，我的理解是，我们就不会直接调用某个贫血类的read / write方法，而是交给了某个PropertyEditorSupport子类完成
>
>
>
> Java beans 是一种综合需求的基础，简单地说，它包含 Bean 自省（Bean 内部描述），Bean 时间，Bean 的内容修改（编辑）等等，并且由 BeanContext 统一托管 Bean 示例，简单地说，Spring 有部分思想是借鉴了 Java Beans，故安排此节。
>
>
>
> PropertyDescriptor和PropertyEditor
>
> JavaBeans 基于反射，也使用了 Java Reference



## 06 轻量级 loC 容器：如何界定 loC 容器的“轻重”？

> Expert One-on-One™ J2EE™ Development without EJB™



## 07 依赖查找 VS. 依赖注入：为什么 Spring 总会强调后者，而选择性忽视前者？

> spring 中主要还是依赖注入,通过 setter 和构造方法两种方式注入;
>
> 依赖查找的应用是实现 ApplicationContextAware 获取 ApplicationContext 来查找自己想要的bean 对象也就是JNDI
>
> Auto-wiring: 自动植入、自动装配
>
> Java EE 之前，或者 EJB 3.0 或 JPA 1.0 之前通常是没有依赖注入实现，不过存在接口回调，来注入一些资源
>



## 08 构造器注入 VS. Setter 注入：为什么 Spring 官方文档的解读会与作者的初心出现偏差？

* ObjectProvider

> 构造器注入是绝对不允许循环依赖存在的，因为ta要求被注入的bean都是成熟态，而字段注入与setter注入没有这样的要求
>
>
>
> 如果使用了 @Autowired，那么就是非 auto-wiring 的模式，auto-wiring 是容器帮助 Bean 自动连接其他 Bean，其中 byName 和 byType 都是按照规则来连接。

## 09 面试题精选



# 第三章：Spring loC 容器概述



## 01 Spring loC 依赖查找：依赖注入还不够吗？依赖查找存在的价值几何？

- Spring IoC 依赖查找
  * 根据 Bean 名称查找
    * 实时查找
    * 延迟查找
  * 根据 Bean 类型查找
    * 单个 Bean 对象
    * 集合 Bean 对象
  * 根据 Bean 名称 + 类型查找
  * 根据 Java 注解查找
    * 单个 Bean 对象
    * 集合 Bean 对象



> ObjectFactoryCreatingFactoryBean ob = (ObjectFactoryCreatingFactoryBean) beanFactory.getBean("objectFactoryCreatingFactoryBean"); 这种根据名称查找编译直接报错，说类型转换错误。而根据类型查找确可以正确编译ObjectFactoryCreatingFactoryBean ob = (ObjectFactoryCreatingFactoryBean) beanFactory.getBean(ObjectFactoryCreatingFactoryBean.class); 
>
> 当 name 关联的 Bean 为 FactoryBean，实际返回的对象是 FactoryBean#getObject()，请参考org.springframework.beans.factory.support.AbstractBeanFactory#getObjectForBeanInstance 方法。
>
>
>
> ObjectFactory 或者 ObjectProvider 的延迟性
>
> 延迟依赖查找主要用于获取 BeanFactory 后，不马上获取相关的 Bean，比如在 BeanFactoryPostProcessor 接口中获取 ConfigurableListableBeanFactory 时，不马上获取，降低 Bean 过早初始化的情况。延迟的意思是指并非当时就查找出来，而需要二次获取
>
>
>
> ObjectFactoryCreatingFactoryBean 与ObjectFactory 没有父子关系，不过 ObjectFactoryCreatingFactoryBean 属于 FactoryBean 来创建 ObjectFactory，当依赖查找或依赖注入时，将返回 ObjectFactory 实例
>
> ObjectFactory 与 FactoryBean 是存在区别的，FactoryBean 的 getObject() 方法会框架调用，而 ObjectFactory 需要应用显示地调用

## 02 Spring loC 依赖注入：Spring 提供了哪些依赖注入模式和类型呢？

> Spring IoC 依赖注入
> ​	• 根据 Bean 名称注入
> ​	• 根据 Bean 类型注入
> ​		• 单个 Bean 对象
> ​		• 集合 Bean 对象
> ​	• 注入容器內建 Bean 对象
> ​	• 注入非 Bean 对象
> ​	• 注入类型
> ​		• 实时注入
> ​		• 延迟注入
>
> ObjectFactoryCreatingFactoryBean 是 ObjectFactory 和 FactoryBean 组合形式，通过 FactoryBean 注册 ObjectFactory





## 03 Spring loC 依赖来源：依赖注入和查找的对象来自于哪里？

> Spring IoC 依赖来源
> • 自定义 Bean
> • 容器內建 Bean 对象
> • 容器內建依赖
>
> 1. 自定义Bean(自己用xml配置或注解配置的bean)
> 2. 内部容器依赖的Bean(非自己定义的Bean,spring容器初始化的Bean)
> 3. 内部容器所构建的依赖(非Bean,不可通过获取依赖查找Bean的方法来获取(getBean(XXX)))
>
> 实际上，**内建的 Bean** 是普通的 Spring Bean，包括 BeanDefinitions 和 Singleton Objects，而**内建依赖**则是通过 AutowireCapableBeanFactory 中的 resolveDependency 方法来注册，这并非是一个 Spring Bean，无法通过依赖查找获取~
>
> 实际上，Spring IoC 底层容器就是指的 BeanFactory 的实现类，大多数情况是 DefaultListableBeanFactory 这个类，它来管理 Spring Beans，而 ApplicationContext 通常为开发人员接触到的 IoC 容器，它是一个 Facade，Wrap 了 BeanFactory 的实现。

## 04 Spring loC 配置元信息：Spring loC 有哪些配置元信息？它们的进化过程是怎样的？

> DefaultListableBeanFactory
>
> Spring IoC 配置元信息
> ​	• Bean 定义配置
> ​		• 基于 XML 文件
> ​		• 基于 Properties 文件
> ​		• 基于 Java 注解
> ​		• 基于 Java API（专题讨论）
> ​	• IoC 容器配置
> ​		• 基于 XML 文件
> ​		• 基于 Java 注解
> ​		• 基于 Java API （专题讨论）
> ​	• 外部化属性配置
> ​		• 基于 Java 注解



## 05 Spring loC容器：BeanFactory 和 ApplicationContext 谁才是 Spring loC 容器？

BeanFactory 是 Spring 底层 IoC 容器
ApplicationContext 是具备应用特性的 BeanFactory 超集



ApplicationContext  继承  BeanFactory  又组合 BeanFactory ，类似代理

>   org.springframework.context.support.AbstractApplicationContext#prepareBeanFactory 方法中，其中：
>
>   beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
>   beanFactory.registerResolvableDependency(ResourceLoader.class, this);
>   beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
>   beanFactory.registerResolvableDependency(ApplicationContext.class, this);
>
>   以上代码明确地指定了 BeanFactory 类型的对象是 ApplicationContext#getBeanFactory() 方法的内容，而非它自生。  
>
>
>
>   1.文中xml中自动注入的BeanFactory为DefaultListableBeanFactory,是因为在AbstractApplicationContext#prepareBeanFactory方法中beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory)中指定了接口的实现为自己bean对象也就是DefaultListableBeanFactory(最后是放在一个map中)
>   2.我们通过classPathXmlApplicationContext创建的bean对象为ApplicationContext类型,虽然继承了接口BeanFactory但是依赖注入的并不是它所以不相等
>   3.ApplicationContext中每次getBean()都是通过DefaultListableBeanFactory来查找bean对象的
>
>
>
>   AbstractRefreshableApplicationContext.getBeanFacory();获取的是组合的对象DefaultListableBeanFactory
>
>   1.xml中自定义bean中注入的BeanFactory是DefaultListableBeanFactory,是我们在创建ClassPathXmlApplicationContext时,构造函数再次创建的一个组合对象,即使他们的顶级接口都是BeanFactory,但是它们根本不是同一个对象
>   创建组合对象是在
>   org.springframework.context.support.AbstractRefreshableApplicationContext#refreshBeanFactory方法中的createBeanFactory()方法中明确创建了DefaultListableBeanFactory()
>   2.在我们操作Bean(getBean())时,其实调用的都是组合对象的getBean()方法org.springframework.context.support.AbstractApplicationContext#getBean(java.lang.String)



## 06 Spring 应用上下文：ApplicationContext 除了 loC 容器角色，还提供哪些特性？

> ApplicationContext 除了 IoC 容器角色，还有提供：
> ​	• 面向切面（AOP）
> ​	• 配置元信息（Configuration Metadata）
> ​	• 资源管理（Resources）
> ​	• 事件（Events）
> ​	• 国际化（i18n）
> ​	• 注解（Annotations）
> ​	• Environment 抽象（Environment Abstraction）
> https://docs.spring.io/spring/docs/5.2.2.RELEASE/spring-framework-reference/core.html#beans-introduction





## 07 使用 Spring loC 容器：选 BeanFactory 还是 ApplicationContext?

> • BeanFactory 是 Spring 底层 IoC 容器
> • ApplicationContext 是具备应用特性的 BeanFactory 超集
>
> DefaultListableBeanFactory 实现的设计模式有：
> ​	抽象工厂（BeanFactory 接口实现）
> ​	组合模式（组合 AutowireCandidateResolver 等实例）
> ​	单例模式（Bean Scope）
> ​	原型模式（Bean Scope）
> ​	模板模式（模板方法定义：AbstractBeanFactory）
> ​	适配器模式（适配 BeanDefinitionRegistry 接口）
> ​	策略模式（Bean 实例化）
> ​	代理模式（ObjectProvider 代理依赖查找）



## 08 Spring loC 容器生命周期：loC 容器启停过程中发生了什么？

启动 refresh

运行

停止 close

## 09 面试题精选

BeanFactory 与 FactoryBean？

答：BeanFactory 是 IoC 底层容器
FactoryBean 是 创建 Bean 的一种方式，帮助实现复杂的初始化逻辑



依赖查找和依赖注入的区别？

答：依赖查找是主动或手动的依赖查找方式，通常需要依赖容器或标准 API
实现。而依赖注入则是手动或自动依赖绑定的方式，无需依赖特定的容器和API



Spring IoC 容器启动时做了哪些准备？

答：IoC 配置元信息读取和解析、IoC 容器生命周期、Spring 事件发布、国
际化等，更多答案将在后续专题章节逐一讨论



# 第四章：Spring Bean 基础

## 01 定义 Bean：什么是 Bean Definition ?为什么说它是最底层的 Spring loC 配置元信息？

* 什么是 BeanDefinition？
  * BeanDefinition 是 Spring Framework 中定义 Bean 的配置元信息接口，包含：
    * Bean 的类名
    * Bean 行为配置元素，如作用域、自动绑定的模式，生命周期回调等
    * 其他 Bean 引用，又可称作合作者（collaborators）或者依赖（dependencies）
    * 配置设置，比如 Bean 属性（Properties）



## 02 BeanDefinition 元信息：除了 Bean 名称和类名，还有哪些 Bean 元信息值得关注？

| 属性（Property ）        | 说明                                          |
| ------------------------ | --------------------------------------------- |
| Class                    | Bean 全类名，必须是具体类，不能用抽象类或接口 |
| Name                     | Bean 的名称或者 ID ？                         |
| Scope                    | Bean 的作用域（如：singleton、prototype 等）  |
| Constructor arguments    | Bean 构造器参数（用于依赖注入）               |
| Properties               | Bean 属性设置（用于依赖注入）                 |
| Autowiring mode          | Bean 自动绑定模式（如：通过名称 byName）      |
| Lazy initialization mode | Bean 延迟初始化模式（延迟和非延迟）           |
| Initialization method    | Bean 初始化回调方法名称                       |
| Destruction method       | Bean 销毁回调方法名称                         |



* BeanDefinition 构建
  * 通过 BeanDefinitionBuilder
  * 通过 AbstractBeanDefinition 以及派生类

**注：BeanDefinition 本身不包含 beanName（非beanClassName）信息吗？**

注册时 ，将beanName 与 BeanDefinition 绑定，所以 beanName 在不可重写时唯一，BeanDefinition不唯一 

new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);



## 03 命名 Spring Bean： id 和 name 属性命名 Bean，哪个更好？

* Bean 的名称

  > 每个 Bean 拥有一个或多个标识符（identifiers），这些标识符在 Bean 所在的容器必须是唯一
  > 的。通常，一个 Bean 仅有一个标识符，如果需要额外的，可考虑使用别名（Alias）来扩充。
  > 在基于 XML 的配置元信息中，开发人员可用 id 或者 name 属性来规定 Bean 的 标识符。通常
  > Bean 的 标识符由字母组成，允许出现特殊字符。如果要想引入 Bean 的别名的话，可在
  > name 属性使用半角逗号（“,”）或分号（“;”) 来间隔。
  > Bean 的 id 或 name 属性并非必须制定，如果留空的话，容器会为 Bean 自动生成一个唯一的
  > 名称。Bean 的命名尽管没有限制，不过官方建议采用驼峰的方式，更符合 Java 的命名约定。

* Bean 名称生成器（BeanNameGenerator）

  * 由 Spring Framework 2.0.3 引入，框架內建两种实现：

    * DefaultBeanNameGenerator：默认通用 BeanNameGenerator 实现

    * AnnotationBeanNameGenerator：基于注解扫描的 BeanNameGenerator 实现，起始于
      Spring Framework 2.5，关联的官方文档：

      > With component scanning in the classpath, Spring generates bean names for unnamed components, following the rules described earlier: essentially, taking the simple class name and turning its initial character to lower-case. However, in the (unusual) special case when there is more than one character and both the first and second characters are upper case, the original casing gets preserved.
      > These are the same rules as defined by java.beans.Introspector.decapitalize (which Spring uses here).



## 04 Spring Bean 的别名：为什么命名 Bean 还需要别名？

* Bean 别名（Alias）的价值
  * 复用现有的 BeanDefinition
  * 更具有场景化的命名方法

> <alias name="myApp-dataSource" alias="subsystemA-dataSource"/>
> <alias name="myApp-dataSource" alias="subsystemB-dataSource"/>



## 05 注册 Spring Bean：如何将 BeanDefinition 注册到 loC 容器？

* BeanDefinition 注册
  * XML 配置元信息
    		• <bean name=”...” ... />

  * Java 注解配置元信息
    * @Bean
    * @Component
    * @Import
    * 是否会冲突？不会，只会 一个
  * Java API 配置元信息
    * 命名方式：BeanDefinitionRegistry#registerBeanDefinition(String,BeanDefinition)
    * 非命名方式：BeanDefinitionReaderUtils#registerWithGeneratedName
                                                    ​    ​    ​    ​    (AbstractBeanDefinition,BeanDefinitionRegistry)
    * 配置类方式：AnnotatedBeanDefinitionReader#register(Class...)



* 注册 Spring Bean
  * 外部单例对象注册
  * Java API 配置元信息
  * SingletonBeanRegistry#registerSingleton





注意：这个`DefaultSingletonBeanRegistry`不一样，这个类非常非常的重要，并且做的事情也很多。甚至认为是Spring容器 所谓的容器的核心内容。 他里面有非常多的缓存，需要解决Bean依赖问题、Bean循环引用问题、Bean正在创建中问题。


	//共享bean实例的通用注册表 实现了SingletonBeanRegistry. 允许注册表中注册的单例应该被所有调用者共享，通过bean名称获得。 
	// 可以注册bean之间的依赖关系，执行适当的注入、关闭顺序
	// 这个类主要用作基类的BeanFactory实现， 提供基本的管理 singleton bean 实例功能
	public class DefaultSingletonBeanRegistry extends SimpleAliasRegistry implements SingletonBeanRegistry {
		
		// ==========它里面维护了非常非常多的Map、List等  这些构成了我们所谓的容器==========
		
		// 很显然：所有的单例bean最终都会到这里来
		private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);
		//缓存了bean的name 和  ObjectFactory。  因为最终的Bean都是通过ObjectFactory的回调方法来创建的
		private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);
		// 缓存了已经存在单例，用于解决循环依赖的方法 循环依赖
		// 是存放singletonFactory 制造出来的 singleton 的缓存早期单例对象缓存
		private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
		// 已经注册好的单例bean的名称们  和 singletonObjects保持同步
	    private final Set<String> registeredSingletons = new LinkedHashSet<String>(256);
	
		// ===================以上四个缓存是这个类存放单例bean的主要Map  ===========================
	
		// 目前正在创建中的单例bean的名称的集合   存着正在初始化的Bean级不要再次发起初始化了 ===注意是正在===
		private final Set<String> singletonsCurrentlyInCreation = Collections.newSetFromMap(new ConcurrentHashMap<>(16));
		// 直接缓存当前不能加载的bean
		// 这个值是留给开发者自己set的，Spring自己不会往里面放值~~~~
		private final Set<String> inCreationCheckExclusions = Collections.newSetFromMap(new ConcurrentHashMap<>(16));
	
		//存放异常出现的相关的原因的集合
	    private Set<Exception> suppressedExceptions;  
		//标志，指示我们目前是否在销毁单例中</span><span>  
	    private boolean singletonsCurrentlyInDestruction = false;  
	
		// 一次性Bean  也就是说Bean是DisposableBean接口的实现
		// 实现DisposableBean接口的类，在类销毁时，会调用destroy()方法，开发人员可以重新该方法完成自己的工作
		// 目前像里添加的只有`AbstractBeanFactory#registerDisposableBeanIfNecessary`  其实还是来自于  doCreateBean方法
		private final Map<String, Object> disposableBeans = new LinkedHashMap<>();
		private final Map<String, Set<String>> containedBeanMap = new ConcurrentHashMap<>(16);
			
		// 查找依赖的类   我依赖了哪些们
		private final Map<String, Set<String>> dependentBeanMap = new ConcurrentHashMap<>(64);
		//被依赖的bean key为beanName    我被哪些们依赖了
		private final Map<String, Set<String>> dependenciesForBeanMap = new ConcurrentHashMap<>(64);
	
		// 添加一个单例对象
		@Override
		public void registerSingleton(String beanName, Object singletonObject) throws IllegalStateException {
			
			// 此处加锁
			synchronized (this.singletonObjects) {
				Object oldObject = this.singletonObjects.get(beanName);
				
				// 注意：此处是如果此单例bean已经存在了，直接抛出异常~~~~~
				if (oldObject != null) {
					throw new IllegalStateException("Could not register object [" + singletonObject +
							"] under bean name '" + beanName + "': there is already object [" + oldObject + "] bound");
				}
				addSingleton(beanName, singletonObject);
			}
		}
		// 添加进去一个实例，实际上它做了好几步操作呢
		// singletonObjects和singletonFactories是对立关系  不会同时存在
		protected void addSingleton(String beanName, Object singletonObject) {
			// 注意此处继续上锁
			synchronized (this.singletonObjects) {
				// 添加进map缓存起来
				this.singletonObjects.put(beanName, singletonObject);
				// 因为已经添加进去了，所以Factories就可议移除了~~~
				this.singletonFactories.remove(beanName);
				//已经存在单例（循环依赖）也可以移除了~~~
				this.earlySingletonObjects.remove(beanName);
				// beanName放进单例注册表中  
				this.registeredSingletons.add(beanName);
			}
		}
	
		// 注意：它是一个protected方法，并不是接口方法。子类会向这里添加工厂的~~~
		protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
			synchronized (this.singletonObjects) {
				// 首先判断一下：这个bean没有被产生才需要添加工厂，否则啥都不做~~~
				//  判断singletonObjects内名字为beanName是否被占用，若没有，进行注册操作  
				if (!this.singletonObjects.containsKey(beanName)) {
					this.singletonFactories.put(beanName, singletonFactory);
					//已经存在单例（循环依赖）也可以移除了~~~ 
					this.earlySingletonObjects.remove(beanName);
					// 注意注意注意：此处beanName也缓存了哦~~~一定要注意
					this.registeredSingletons.add(beanName);
				}
			}
		}
	
		// SingletonBeanRegistry接口的getSingleton方法的实现 
		@Override
		@Nullable
		public Object getSingleton(String beanName) {
			return getSingleton(beanName, true);
		}
		// allowEarlyReference：是否要创建早期引用
		@Nullable
		protected Object getSingleton(String beanName, boolean allowEarlyReference) {
			// 先根据这个beanName去查找，找到value值
			Object singletonObject = this.singletonObjects.get(beanName);
		
			// 此处：如果此单例不存在，也不要立马就返回null了  还有工作需要处理呢
			// 这里的条件是：如果单例不存在，并且并且这个bean正在chuangjianzhong~~~(在这个singletonsCurrentlyInCreation集合了，表示它正在创建中) 
			// 什么时候放进这个集合表示创建中呢？调用beforeSingletonCreation()方法，因为他是protected方法，所以只允许本类和子类调用~
			if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
				synchronized (this.singletonObjects) {
					// 这里再去earlySingletonObjects去看一下，看是否有呢   如果有直接返回即可
					singletonObject = this.earlySingletonObjects.get(beanName);
		
					// 如果还未null，并且 并且allowEarlyReference是允许的  也就是说是允许创建一个早期引用的（简单的说就是先可以把引用提供出去，但是并还没有完成真正的初始化~~~~）
					// 这里ObjectFactory就发挥很大的作用了~~~
					if (singletonObject == null && allowEarlyReference) {
						ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
						// 若有对应的ObjectFactory 那就可以继续处理
						// 备注：singletonFactories这个Map只有调用`addSingletonFactory()`方法的时候才往里添加
						// 它是一个protected方法，目前Spring还只有在`AbstractAutowireCapableBeanFactory#doCreateBean`里有过调用
						if (singletonFactory != null) {
							singletonObject = singletonFactory.getObject();
							// 注意此处：把生产出来的放进earlySingletonObjects里面去，表示生产出来了一个引用
							this.earlySingletonObjects.put(beanName, singletonObject);
							// 然后把nFactories可以移除了，因为引用已经产生了~~~
							this.singletonFactories.remove(beanName);
						}
					}
				}
			}
			return singletonObject;
		}
		
		// 这个方法蛮重要的。首先它不是接口方法，而是一个单纯的public方法~~~
		// 它的调用处只有一个地方：AbstractBeanFactory#doGetBean  在真正 `if (mbd.isSingleton()) { sharedInstance = getSingleton(...) }`
		// 它第二个参数传的是ObjectFactory~~~~~~~实现有创建Bean实例的逻辑~~~
		public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
			// 在锁内工作~~~~
			synchronized (this.singletonObjects) {
				Object singletonObject = this.singletonObjects.get(beanName);
				// 很显然如果都不为null了，那还做什么呢  直接返回吧
				if (singletonObject == null) {
					// 如果目前在销毁singellton 那就抛异常呗~~~~
					if (this.singletonsCurrentlyInDestruction) {
						//  抛异常
					}
					// 标记这个bean正在被创建~~~~
					beforeSingletonCreation(beanName);
					boolean newSingleton = false;
					
					try {
						singletonObject = singletonFactory.getObject();
						newSingleton = true;
					} catch (IllegalStateException ex) {
						... // 省略	
					} finally {
						// 释放这个状态  说明这个bean已经创建完成了
						afterSingletonCreation(beanName);
					}


					// 如果是新创建了  这里执行一下添加  缓存起来~~~~
					// 如果是旧的  是不用添加的~~~~
					if (newSingleton) {
						addSingleton(beanName, singletonObject);
					}
				}
				return singletonObject;
			}
		}
	
		// 这两个方法 一个在Bean创建开始之前还行。一个在创建完成后执行 finally里执行
	
		// 表示；beforeSingletonCreation()方法用于记录加载的状态  表示该Bean当前正在初始化中~~~
		// 调用this.singletonsCurrentlyInCreation.add(beanName)将当前正要创建的bean记录在缓存中，这样便可以对循环依赖进行检测啦
		// afterSingletonCreation显然就是
		protected void beforeSingletonCreation(String beanName) {
			if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}
		}
		protected void afterSingletonCreation(String beanName) {
			if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.remove(beanName)) {
				throw new IllegalStateException("Singleton '" + beanName + "' isn't currently in creation");
			}
		}
	
		// 直接检查 singleton object 存储器了，其他的存储器不做检查
		@Override
		public boolean containsSingleton(String beanName) {
			return this.singletonObjects.containsKey(beanName);
		}
		@Override
		public String[] getSingletonNames() {
			synchronized (this.singletonObjects) {
				return StringUtils.toStringArray(this.registeredSingletons);
			}
		}
		@Override
		public int getSingletonCount() {
			synchronized (this.singletonObjects) {
				return this.registeredSingletons.size();
			}
		}
		// 这个方法  只是简单的把这个Map返回出去了~~~~~
		@Override
		public final Object getSingletonMutex() {
			return this.singletonObjects;
		}
	}
	
	getSingleton的时候，spring的默认实现是，先从 singleton object 的存储器中去寻找，如果找不到，再从 early singleton object 存储器中寻找，再找不到，那就在寻找对应的 singleton factory，造出所需的 singleton object，然后返回
	
	Map<String, Set<String>> containedBeanMap // 依赖的bean name为key , 就是依赖类 -> 查找 被依赖的类
	Map<String, Set<String>> dependentBeanMap　// 依赖的原始bean name为key
	Map<String, Set<String>> dependenciesForBeanMap　// 被依赖的bean name为key



## 06 实例化 Spring Bean：Bean 实例化的姿势有多少种？

* Bean 实例化（Instantiation）

  * 常规方式

    * 通过构造器（配置元信息：XML、Java 注解和 Java API ）
    * 通过静态工厂方法（配置元信息：XML 和 Java API ）
    * 通过 Bean 工厂方法（配置元信息：XML和 Java API ）
    * 通过 FactoryBean（配置元信息：XML、Java 注解和 Java API ）

  * 特殊方式

    * 通过 ServiceLoaderFactoryBean（配置元信息：XML、Java 注解和 Java API ）
    * 通过 AutowireCapableBeanFactory#createBean(java.lang.Class, int, boolean)
    * 通过 BeanDefinitionRegistry#registerBeanDefinition(String,BeanDefinition)



## 07 初始化 Spring Bean：Bean 初始化有哪些方式？

* Bean 初始化（Initialization）
  * @PostConstruct 标注方法
  * 实现 InitializingBean 接口的 afterPropertiesSet() 方法
  * 自定义初始化方法
    * XML 配置：<bean init-method=”init” ... />
    * Java 注解：@Bean(initMethod=”init”)
    * Java API：AbstractBeanDefinition#setInitMethodName(String)

> 思考：假设以上三种方式均在同一 Bean 中定义，那么这些方法的执行顺序是怎样？
>
> 列举的



## 08 延迟初始化 Spring Bean：延迟初始化的 Bean 会影响依赖注入吗？

* Bean 延迟初始化（Lazy Initialization）
  * XML 配置：<bean lazy-init=”true” ... />
  * Java 注解：@Lazy(true)

> 思考：当某个 Bean 定义为延迟初始化，那么，Spring 容器返回的对象与非延迟的对象存在怎样的差异？
>
> 非延迟的单例对象 在 refresh() 完成时 被初始化
>
>  延迟初始化的Spring 对象 在初次调用时才会初始化
>
> init 方法执行的时机



处理@Lazy   getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
​      descriptor, requestingBeanName);

返回一个代理类

`ContextAnnotationAutowireCandidateResolver extends QualifierAnnotationAutowireCandidateResolver`



处理 @Qulifier

CustomAutowireConfigurer 

QualifierAnnotationAutowireCandidateResolver



##### Spring的Bean有序吗？试试用@DependsOn或static来提高优先级

@Bean方法上加static成为静态方法，并**不能**提升此Bean的优先级（@Bean解析的时机）



## 09 销毁 Spring Bean：销毁 Bean 的基本操作有哪些？

* Bean 销毁（Destroy）
  * @PreDestroy 标注方法
  * 实现 DisposableBean 接口的 destroy() 方法
  * 自定义销毁方法
    * XML 配置：<bean destroy=”destroy” ... />	
    * Java 注解：@Bean(destroy=”destroy”)
    * Java API：AbstractBeanDefinition#setDestroyMethodName(String)

> 思考：假设以上三种方式均在同一 Bean 中定义，那么这些方法的执行顺序是怎样？就这个
>
> applicationContext close()  时执行



## 10 回收 Spring Bean：Spring loC 容器管理的 Bean 能够被垃圾回收吗？

* Bean 垃圾回收（GC）
  1. 关闭 Spring 容器（应用上下文）  close()
  2. 执行 GC
  3. Spring Bean 覆盖的 finalize() 方法被回调



## 11 面试题精选

> 如何注册一个 Spring Bean？
>
> 答：通过 BeanDefinition 和外部单体对象来注册
>
> 什么是 Spring BeanDefinition？
>
> 答：回顾“定义 Spring Bean” 和 “BeanDefinition 元信息”
>

广义地来看，Java Bean 是一个表现形式，而 Spring Bean 是狭义地托管在 Spring 容器中的 Java Bean

用id和name有啥根本区别没？在早期的 Spring 有，说的是 ID 是全局唯一，而 name 是当前应用上下文唯一，而现在则没有这样的限制，id 和 name 相同了。



ServiceLoader，ServiceFactoryBean，ServiceListFactoryBean，ServiceLoaderFactoryBean具体关系和实现

FactoryBean 是用于 Spring Bean 对象创建过程较为复杂的 ，比如通过数据库加载等。

pring在应用运行过程中，因为Bean不会被回收，所有实例化的Bean最终都会进入老年代，对吗？还有，我记得有个说法Spring动态代理在1.7之前会使得Perm区溢出，因为动态的类创建过多。动态代理的Class对象会被存入Perm区，但实例化的对象还是在堆上，这么理解对吗？

作者回复: 差不多，Java Class 均存储在 ClassLoader 关联的空间，而这部分数据存放在 Perm 或 Metaspace 中，当动态代理，还是字节码提升还是会生成 Class 对象，这部分数据均在前面提到的空间类，均属于元信息。而普通 Java 对象包括数组均在 Heap 上。



# 第五章：Spring loC 依赖查找（Dependency Lookup）

## 01 依赖査找的今世前生：Spring loC 容器从 Java 标准中学到了什么？

* 单一类型依赖查找
  - JNDI - javax.naming.Context#lookup(javax.naming.Name)
  - JavaBeans - java.beans.beancontext.BeanContext
* 集合类型依赖查找
  - java.beans.beancontext.BeanContext
* 层次性依赖查找
  - java.beans.beancontext.BeanContext



## 02 单一类型依赖查找：如何查找已知名称或类型的 Bean 对象？

* 单一类型依赖查找接口 - BeanFactory
  * 根据 Bean 名称查找
    * getBean(String)
    * Spring 2.5 覆盖默认参数：getBean(String,Object...)
  * 根据 Bean 类型查找
    * Bean 实时查找
      * Spring 3.0 getBean(Class)
      * Spring 4.1 覆盖默认参数：getBean(Class,Object...)
    * Spring 5.1 Bean 延迟查找
      * getBeanProvider(Class)
      * getBeanProvider(ResolvableType)
  * 根据 Bean 名称 + 类型查找：getBean(String,Class)



## 03 集合类型依赖查找：如何查找已知类型多个 Bean 集合？

* 集合类型依赖查找接口 - ListableBeanFactory
  * 根据 Bean 类型查找
    * 获取同类型 Bean 名称列表
      * getBeanNamesForType(Class)
      * getBeanNamesForType(ResolvableType)（Spring 4.2）
    * 获取同类型 Bean 实例列表
      * getBeansOfType(Class) 以及重载方法
  * 通过注解类型查找
    * Spring 3.0 获取标注类型 Bean 名称列表
      * getBeanNamesForAnnotation(Class<? extends Annotation>)
    * Spring 3.0 获取标注类型 Bean 实例列表
      * getBeansWithAnnotation(Class<? extends Annotation>)
    * Spring 3.0 获取指定名称 + 标注类型 Bean 实例
      * findAnnotationOnBean(String,Class<? extends Annotation>)



## 04 层次性依赖查找：依赖查找也有双亲委派？

* 层次性依赖查找接口 - HierarchicalBeanFactory
  * 双亲 BeanFactory：getParentBeanFactory()
  * 层次性查找
    * 根据 Bean 名称查找
      * 基于 containsLocalBean 方法实现
    * 根据 Bean 类型查找实例列表
      * 单一类型：BeanFactoryUtils#beanOfType
      * 集合类型：BeanFactoryUtils#beansOfTypeIncludingAncestors
    * 根据 Bean 类型查找名称列表
      * BeanFactoryUtils#beanNamesForTypeIncludingAncestors
    * 根据 Java 注解查找名称列表（标注了 指定类型注解的 bean）
      * BeanFactoryUtils#beanNamesForAnnotationIncludingAncestors



## 05 延迟依赖查找：非延迟初始化 Bean 也能实现延迟查找？

* Bean 延迟依赖查找接口
  * org.springframework.beans.factory.ObjectFactory
  * org.springframework.beans.factory.ObjectProvider
    * Spring 5 对 Java 8 特性扩展
      * 函数式接口
        * getIfAvailable(Supplier)
        * ifAvailable(Consumer)
    * Stream 扩展 - stream()



## 06 安全依赖查找

• 依赖查找安全性对比

| 依赖查找类型 | 代表实现                           | 是否安全 |
| ------------ | ---------------------------------- | -------- |
| 单一类型查找 | BeanFactory#getBean                | 否       |
|              | ObjectFactory#getObject            | 否       |
|              | ObjectProvider#getIfAvailable      | 是       |
|              |                                    |          |
| 集合类型查找 | ListableBeanFactory#getBeansOfType | 是       |
|              | ObjectProvider#stream              | 是       |

注意：层次性依赖查找的安全性取决于其扩展的单一或集合类型的 BeanFactory 接口



## 07 内建可查找的依赖：哪些 Spring loC 容器内建依赖可供査找？

prepareBeanFactory

* ApplicationContextAwareProcessor

```
ignoreDependencyInterface
registerResolvableDependency
```

* AbstractApplicationContext 内建可查找的依赖

| Bean 名称                   | Bean 实例                        | 使用场景                |
| --------------------------- | -------------------------------- | ----------------------- |
| environment                 | Environment 对象                 | 外部化配置以及 Profiles |
| systemProperties            | java.util.Properties 对象        | Java 系统属性           |
| systemEnvironment           | java.util.Map 对象               | 操作系统环境变量        |
| messageSource               | MessageSource 对象               | 国际化文案              |
| lifecycleProcessor          | LifecycleProcessor 对象          | Lifecycle Bean 处理器   |
| applicationEventMulticaster | ApplicationEventMulticaster 对象 | Spring 事件广播器       |



* 注解驱动 Spring 应用上下文内建可查找的依赖

| Bean 名称                                                    | Bean 实例                                   | 使用场景                                              |
| ------------------------------------------------------------ | ------------------------------------------- | ----------------------------------------------------- |
| org.springframework.context.<br/>annotation.internalConfiguration<br/>AnnotationProcessor | ConfigurationClassPostProcessor 对象        | 处理 Spring 配置类                                    |
| org.springframework.context.<br/>annotation.internalAutowired<br/>AnnotationProcessor | AutowiredAnnotationBeanPostProcessor 对象   | 处理 @Autowired 以及 @Value                           |
| org.springframework.context.<br/>annotation.internalCommon<br/>AnnotationProcessor | CommonAnnotationBeanPostProcessor 对象      | （条件激活）处理 JSR-250 注解，                       |
| org.springframework.context.<br/>event.internalEventListener<br/>Processor | EventListenerMethodProcessor对象            | 处理标注 @EventListener 的                            |
| org.springframework.context.<br/>event.internalEventListener<br/>Factory | DefaultEventListenerFactory 对象            | @EventListener 事件监听方法适配为 ApplicationListener |
| org.springframework.context.<br/>annotation.internalPersistence
AnnotationProcessor | PersistenceAnnotationBeanPostProcessor 对象 | （条件激活）处理 JPA 注解场景                         |



## 07 依赖查找中的经典异常：Bean 找不到？ Bean 不是唯一的？Bean 创建失败？

| 异常类型                        | 触发条件（举例）                           | 场景举例                                     |
| ------------------------------- | ------------------------------------------ | -------------------------------------------- |
| NoSuchBeanDefinitionException   | 当查找 Bean 不存在于 IoC 容器时            | BeanFactory#getBean，ObjectFactory#getObject |
| NoUniqueBeanDefinitionException | 类型依赖查找时，IoC 容器存在多个 Bean 实例 | BeanFactory#getBean(Class)                   |
| BeanInstantiationException      | 当 Bean 所对应的类型非具体类时             |                                              |
| BeanCreationException           | 当 Bean 初始化过程中                       | Bean 初始化方法执行异常时                    |
| BeanDefinitionStoreException    | 当 BeanDefinition 配置元信息非法时         | XML 配置资源无法打开时                       |




## 08 面试题精选

ObjectFactory 与 BeanFactory 的区别？

答：ObjectFactory 与 BeanFactory 均提供依赖查找的能力。不过 ObjectFactory 仅关注一个或一种类型的 Bean 依赖查找，并且自身不具备依赖查找的能力，能力则由 BeanFactory 输出。BeanFactory 则提供了单一类型、集合类型以及层次性等多种依赖查找方式。

```
private <T> T resolveBean(ResolvableType requiredType, @Nullable Object[] args, boolean nonUniqueAsNull) 
```

BeanFactory.getBean 操作是否线程安全？

答：BeanFactory.getBean 方法的执行是线程安全的，操作过程中会增加互
斥锁



# 第六章：Spring loC 依赖注入（Dependency Injection）

## 01 依赖注入的模式和类型：Spring 提供了哪些依赖注入的模式和类型？

### 依赖注入的模式

• 手动模式 - 配置或者编程的方式，提前安排注入规则
​	• XML 资源配置元信息
​	• Java 注解配置元信息
​	• API 配置元信息
• 自动模式 - 实现方提供依赖自动关联的方式，按照內建的注入规则
​	• Autowiring（自动绑定）



### 依赖注入类型

| 依赖注入类型 | 配置元数据举例                                   |
| ------------ | ------------------------------------------------ |
| Setter 方法  | <proeprty name="user" ref="userBean" />          |
| 构造器       | <constructor-arg name="user" ref="userBean" />   |
| 字段         | @Autowired User user;                            |
| 方法         | @Autowired public void user(User user) { ... }   |
| 接口回调     | class MyBean implements BeanFactoryAware { ... } |



## 02 自动绑定（Autowiring）：为什么 Spring 会引入 Autowiring ?（不推荐）

> • 官方说明
> The Spring container can autowire relationships between collaborating beans. You can let Spring resolve collaborators (other beans) automatically for your bean by inspecting the contents of the ApplicationContext.
>
> • 优点
> ​	• Autowiring can significantly reduce the need to specify properties or constructor arguments.
> ​	• Autowiring can update a configuration as your objects evolve.

* **不推荐使用**

## 03 自动绑定（Autowiring）模式：各种自动绑定模式的使用场景是什么？

• Autowiring modes
参考枚举：org.springframework.beans.factory.annotation.Autowire

| 模式        | 说明                                                         |
| ----------- | ------------------------------------------------------------ |
| no          | 默认值，未激活 Autowiring，需要手动指定依赖注入对象          |
| byName      | 根据被注入属性的名称作为 Bean 名称进行依赖查找，并将对象设置到该<br/>属性 |
| byType      | 根据被注入属性的类型作为依赖类型进行查找，并将对象设置到该属性 |
| constructor | 特殊 byType 类型，用于构造器参数                             |

xml：BeanDefinitionParserDelegate#parseBeanDefinitionAttributes



## 04 自动绑定（Autowiring）限制和不足：如何理解和挖掘官方文档中深层次的含义？

> • 官方说明
> Limitations and Disadvantages of Autowiring 小节
> 链接：https://docs.spring.io/spring/docs/5.2.2.RELEASE/spring-framework-reference/core.html#beans-autowired-exceptions

不足：看官方文档

较低优先级，不够精确，不够灵活（若bean发生改变）



## 05 Setter 方法依赖注入：Setter 注入的原理是什么？

* 实现方法
  - 手动模式
    - XML 资源配置元信息
    - Java 注解配置元信息
    - API 配置元信息
  - 自动模式
    * byName
    * byType

Setter 注入是通过 Java Beans 来实现的，而方法注入则是直接通过 Java 反射来做的。当然底层都是 Java 反射~

PropertyDescriptor readMethodRef writeMethodRef



## 06 构造器依赖注入：官方为什么推荐使用构造器注入？

* 实现方法
  * 手动模式
    * XML 资源配置元信息
    * Java 注解配置元信息
    * API 配置元信息
  * 自动模式
  * constructor



Spring Framework 内建的实现仅在 XML 中体现，不过这是一个通用的功能，开发人员也可以通过 org.springframework.beans.factory.support.AbstractBeanDefinition#setAutowireMode 控制 Auto-wiring 的模式



## 07 字段注入：为什么 Spring 官方文档没有单独列举这种注入方式？

* 实现方法
  * 手动模式
    * Java 注解配置元信息
      * @Autowired
      * @Resource
      * @Inject（可选）



## 08 方法注入：方法注入是 @Autowired 专利吗？

* 实现方法
  - 手动模式
    - Java 注解配置元信息
      - @Autowired
      - @Resource
      - @Inject（可选）
      - @Bean



## 09 接口回调注入：回调注入的使用场景和限制有哪些？

xxxAware 

invokeAwareInterfaces

invokeAwareMethods



## 10 依赖注入类型选择：各种依赖注入有什么样的使用场景？



## 11 基础类型注入：String 和 Java 原生类型也能注入 Bean 的属性，它们算依赖注入吗？

* 类型转换
  * 基 础 类 型
    - 原生类型（Primitive) : boolean、byte、char、short、int、float、long、double
    - 标量类型（Scalar} : Number、Character、Boolean、Enum、Locale、Charset、Currency、 
      Properties、UUID
    - 常规类型（General）：Object、String、TimeZone、Calendar、Optional 等 
    - Spring 类型：Resource、InputSource、Formatter 等
  * string 转换成 Spring 类型 再注入
  * Enum



## 12 集合类型注入：注入 Collection 和 Map 类型的依赖区别？还支持哪些集合类型？



## 13 限定注入：如何限定 Bean名称注入？如何实现 Bean 逻辑分组注入？

* 使用注解@Qualifier限定
  * 通过Bean名称限定
  * 通过分组限定
* 基于注解@Qualifier扩展限定
  * 自定义注解 - 如 Spring Cloud @LoadBalanced



## 14 延迟依赖注入：如何实现延迟执行依赖注入？与延迟依赖查找是类似的吗？

循环依赖注入在3.0以前和在ObjectProvider出现后解决方案有所差异

* 使用 API **ObjectFactory** 延迟注入
  * 单一类型
  * 集合类型
* 使用 API **ObjectProvider** 延迟注人（推荐）
  * 单一类型
  * 集合类型



## 15 依赖处理过程：依赖处理时会发生什么？其中与依赖查找的差异在哪？（源码解析）

> DefaultListableBeanFactory#resolveDependency -->
>
> 1.判断是否懒加载 --> doResolveDependency() -->
>
> 2.判断是否是多类型的bean resolveMultipleBeans()-->
>
> 3.根据类型查找匹配到的bean findAutowireCandidates()-->
>
> 4.bean个数大于1,选择bean即@Primary修饰 determineAutowireCandidate()--->
>
> 返回结果

* 入口 - `DefaultListableBeanFactoryt#resolveDependency`   哪里调用的？
* 依赖描述符 - `DependencyDescriptor`
* 自定绑定候选对象处理器 - `AutowireCandidateResolver`



```java
	@Override
	@Nullable
	public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
			@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {
		
		// 把当前Bean工厂的名字发现器赋值给传进来DependencyDescriptor 类
		// 这里面注意了：有必要说说名字发现器这个东西，具体看下面吧==========还是比较重要的
		// Bean工厂的默认值为：private ParameterNameDiscoverer parameterNameDiscoverer = new DefaultParameterNameDiscoverer();
		descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());

		// 支持到Optional类型的注入，比如我们这样注入：private Optional<GenericBean<Object, Object>> objectGenericBean;
		// 也是能够注入进来的，只是类型变为，Optional[GenericBean(t=obj1, w=2)]
		// 对于Java8中Optional类的处理
		if (Optional.class == descriptor.getDependencyType()) {
			return createOptionalDependency(descriptor, requestingBeanName);
		}
		// 兼容ObjectFactory和ObjectProvider（Spring4.3提供的接口）
		// 关于ObjectFactory和ObjectProvider在依赖注入中的大作用，我觉得是非常有必要再撰文讲解的
		//对于前面讲到的提早曝光的ObjectFactory的特殊处理
		else if (ObjectFactory.class == descriptor.getDependencyType() ||
				ObjectProvider.class == descriptor.getDependencyType()) {
			return new DependencyObjectProvider(descriptor, requestingBeanName);
		}
		// 支持到了javax.inject.Provider这个类的实现
		else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
			return new Jsr330ProviderFactory().createDependencyProvider(descriptor, requestingBeanName);
		}
		// 这个应该是我们觉得部分触及到的，其实不管何种方式，最终都是交给doResolveDependency方法去处理了
		else {
			//getAutowireCandidateResolver()得到ContextAnnotationAutowireCandidateResolver 根据依赖注解信息，找到对应的Bean值信息
			//getLazyResolutionProxyIfNecessary方法，它也是唯一实现。
			//如果字段上带有@Lazy注解，表示进行懒加载 Spring不会立即创建注入属性的实例，而是生成代理对象，来代替实例
			Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
					descriptor, requestingBeanName);
			// 如果在@Autowired上面还有个注解@Lazy，那就是懒加载的，是另外一种处理方式（是一门学问）
			// 这里如果不是懒加载的（绝大部分情况都走这里） 就进入核心方法doResolveDependency 下面有分解
			if (result == null) {
				result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
			}
			return result;
		}
	}

```

#### doResolveDependency

```java
@Nullable
	public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
			@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {
	
		// 相当于打个点，记录下当前的步骤位置  返回值为当前的InjectionPoint 
 		InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
		try {
			// 简单的说就是去Bean工厂的缓存里去看看，有没有名称为此的Bean，有就直接返回，没必要继续往下走了
			// 比如此处的beanName为：objectGenericBean等等
			Object shortcut = descriptor.resolveShortcut(this);
			if (shortcut != null) {
				return shortcut;
			}

			// 此处为：class com.fsx.bean.GenericBean
			Class<?> type = descriptor.getDependencyType();
		
			// 看看ContextAnnotationAutowireCandidateResolver的getSuggestedValue方法,具体实现在父类 QualifierAnnotationAutowireCandidateResolver中
			//处理@Value注解-------------------------------------
			//获取@Value中的value属性
			Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
			// 若存在value值，那就去解析它。使用到了AbstractBeanFactory#resolveEmbeddedValue
			// 也就是使用StringValueResolver处理器去处理一些表达式~~
			if (value != null) {
				if (value instanceof String) {
					String strVal = resolveEmbeddedValue((String) value);
					BeanDefinition bd = (beanName != null && containsBean(beanName) ? getMergedBeanDefinition(beanName) : null);
					value = evaluateBeanDefinitionString(strVal, bd);
				}
				//如果需要会进行类型转换后返回结果
				TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
				return (descriptor.getField() != null ?
						converter.convertIfNecessary(value, type, descriptor.getField()) :
						converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
			}
			
			//对数组、Collection、Map等类型进行处理，也是支持自动注入的。
			//因为是数组或容器，Sprng可以直接把符合类型的bean都注入到数组或容器中，处理逻辑是：
			//1.确定容器或数组的组件类型 if else 分别对待，分别处理
			//2.调用findAutowireCandidates（核心方法）方法，获取与组件类型匹配的Map(beanName -> bean实例)
			//3.将符合beanNames添加到autowiredBeanNames中
			Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
			if (multipleBeans != null) {
				return multipleBeans;
			}
			
			// 获取所有【类型】匹配的Beans，形成一个Map（此处用Map装，是因为可能不止一个符合条件）
			// 该方法就特别重要了，对泛型类型的匹配、对@Qualifierd的解析都在这里面，下面详情分解
			Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
			// 若没有符合条件的Bean。。。
			if (matchingBeans.isEmpty()) {
			    // 并且是必须的，那就抛出没有找到合适的Bean的异常吧
			    // 我们非常熟悉的异常信息：expected at least 1 bean which qualifies as autowire candidate...
				if (isRequired(descriptor)) {
					raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
				}
				return null;
			}

			String autowiredBeanName;
			Object instanceCandidate;


			//如果类型匹配的bean不止一个，Spring需要进行筛选，筛选失败的话继续抛出异常
			// 如果只找到一个该类型的，就不用进这里面来帮忙筛选了~~~~~~~~~
			if (matchingBeans.size() > 1) {
				// 该方法作用：从给定的beans里面筛选出一个符合条件的bean，此筛选步骤还是比较重要的，因此看看可以看看下文解释吧
				autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
				if (autowiredBeanName == null) {
					// 如果此Bean是要求的，或者 不是Array、Collection、Map等类型，那就抛出异常NoUniqueBeanDefinitionException
					if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
						// 抛出此异常
						return descriptor.resolveNotUnique(type, matchingBeans);
					}
					// Spring4.3之后才有：表示如果是required=false，或者就是List Map类型之类的，即使没有找到Bean，也让它不抱错，因为最多注入的是空集合嘛
					else {
						// In case of an optional Collection/Map, silently ignore a non-unique case:
						// possibly it was meant to be an empty collection of multiple regular beans
						// (before 4.3 in particular when we didn't even look for collection beans).
						return null;
					}
				}
				
				instanceCandidate = matchingBeans.get(autowiredBeanName);
			}
			else {
				// We have exactly one match.
				// 仅仅只匹配上一个，走这里 很简单  直接拿出来即可
				// 注意这里直接拿出来的技巧：不用遍历，直接用iterator.next()即可
				Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
				autowiredBeanName = entry.getKey();
				instanceCandidate = entry.getValue();
			}
			
			// 把找到的autowiredBeanName 放进去
			if (autowiredBeanNames != null) {
				autowiredBeanNames.add(autowiredBeanName);
			}
			// 底层就是调用了beanFactory.getBean(beanName);  确保该实例肯定已经被实例化了的
			if (instanceCandidate instanceof Class) {
				instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
			}
			Object result = instanceCandidate;
			if (result instanceof NullBean) {
				if (isRequired(descriptor)) {
					raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
				}
				result = null;
			}
			// 再一次校验，type和result的type类型是否吻合=====
			if (!ClassUtils.isAssignableValue(type, result)) {
				throw new BeanNotOfRequiredTypeException(autowiredBeanName, type, instanceCandidate.getClass());
			}
			return result;
		}
		// 最终把节点归还回来
		finally {
			ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
		}
```

#### determineAutowireCandidate

> 将获取类型匹配的Bean工作交给BeanFactoryUtils.beanNamesForTypeIncludingAncestors。该方法除了当前beanFactory还会递归对父parentFactory进行查找
>
> 如果注入类型是特殊类型或其子类，会将特殊类型的实例添加到结果
> 对结果进行筛选
>
> BeanDefinition的autowireCandidate属性，表示是否允许该bena注入到其他bean中，默认为true
> 泛型类型的匹配，如果存在的话
>
> Qualifier注解。如果存在Qualifier注解的话，会直接比对Qualifier注解中指定的beanName。需要注意的是，Spring处理自己定义的Qualifier注解，还支持javax.inject.Qualifier注解
> 如果筛选后，结果为空，Spring会放宽筛选条件，再筛选一次

```java
//determineAutowireCandidate 从多个Bean中，筛选出一个符合条件的Bean
	@Nullable
	protected String determineAutowireCandidate(Map<String, Object> candidates, DependencyDescriptor descriptor) {
		Class<?> requiredType = descriptor.getDependencyType();
		// 看看传入的Bean中有没有标注了@Primary注解的
		String primaryCandidate = determinePrimaryCandidate(candidates, requiredType);
		// 如果找到了 就直接返回
		// 由此可见，@Primary的优先级还是非常的高的
		if (primaryCandidate != null) {
			return primaryCandidate;
		}
		//找到一个标注了javax.annotation.Priority注解的。（备注：优先级的值不能有相同的，否则报错）
		String priorityCandidate = determineHighestPriorityCandidate(candidates, requiredType);
		if (priorityCandidate != null) { 
			return priorityCandidate;
		}
		// Fallback
		// 这里是最终的处理（相信绝大部分情况下，都会走这里~~~~~~~~~~~~~~~~~~~~）
		// 此处就能看出resolvableDependencies它的效能了，他会把解析过的依赖们缓存起来，不用再重复解析了
		for (Map.Entry<String, Object> entry : candidates.entrySet()) {
			String candidateName = entry.getKey();
			Object beanInstance = entry.getValue();
			
			// 到这一步就比较简单了，matchesBeanName匹配上Map的key就行。
			// 需要注意的是，bean可能存在很多别名，所以只要有一个别名相同，就认为是能够匹配上的  具体参考AbstractBeanFactory#getAliases方法
			//descriptor.getDependencyName() 这个特别需要注意的是：如果是字段，这里调用的this.field.getName() 直接用的是字段的名称
			// 因此此处我们看到的情况是，我们采用@Autowired虽然匹配到两个类型的Bean了，即使我们没有使用@Qualifier注解，也会根据字段名找到一个合适的（若没找到，就抱错了）
			if ((beanInstance != null && this.resolvableDependencies.containsValue(beanInstance)) ||
					matchesBeanName(candidateName, descriptor.getDependencyName())) {
				return candidateName;
			}
		}
		return null;
	}

//determinePrimaryCandidate：顾名思义。它是从给定的Bean中看有木有标注了@Primary注解的Bean，有限选择它
	@Nullable
	protected String determinePrimaryCandidate(Map<String, Object> candidates, Class<?> requiredType) {
		String primaryBeanName = null;
		for (Map.Entry<String, Object> entry : candidates.entrySet()) {
			String candidateBeanName = entry.getKey();
			Object beanInstance = entry.getValue();
			// isPrimary就是去看看容器里（包含父容器）对应的Bean定义信息是否有@Primary标注
			if (isPrimary(candidateBeanName, beanInstance)) {
				if (primaryBeanName != null) {
					boolean candidateLocal = containsBeanDefinition(candidateBeanName);
					boolean primaryLocal = containsBeanDefinition(primaryBeanName);

					// 这个相当于如果已经找到了一个@Primary的，然后又找到了一个 那就抛出异常
					// @Primary只能标注到一个同类型的Bean上
					if (candidateLocal && primaryLocal) {
						throw new NoUniqueBeanDefinitionException(requiredType, candidates.size(),
								"more than one 'primary' bean found among candidates: " + candidates.keySet());
					}
					else if (candidateLocal) {
						primaryBeanName = candidateBeanName;
					}
				}
				// 把找出来的标注了@Primary的Bean的名称返回出去
				else {
					primaryBeanName = candidateBeanName;
				}
			}
		}
		return primaryBeanName;
	} 

// determineHighestPriorityCandidate：从给定的Bean里面筛选出一个优先级最高的
// 什么叫优先级最高呢？主要为了兼容JDK6提供的注解javax.annotation.Priority
	@Nullable
	protected String determineHighestPriorityCandidate(Map<String, Object> candidates, Class<?> requiredType) {
		String highestPriorityBeanName = null;
		Integer highestPriority = null;
		for (Map.Entry<String, Object> entry : candida         tes.entrySet()) {
			String candidateBeanName = entry.getKey();
			Object beanInstance = entry.getValue();
			if (beanInstance != null) {
				//AnnotationAwareOrderComparator#getPriority
				// 这里就是为了兼容JDK6提供的javax.annotation.Priority这个注解，然后做一个优先级排序
				// 注意注意注意：这里并不是@Order，和它木有任何关系~~~
				// 它有的作用像Spring提供的@Primary注解
				Integer candidatePriority = getPriority(beanInstance);
				// 大部分情况下，我们这里都是null，但是需要注意的是，@Primary只能标注一个，这个虽然可以标注多个，但是里面的优先级值，不能出现相同的（强烈建议不要使用~~~~而使用@Primary）
				if (candidatePriority != null) {
					if (highestPriorityBeanName != null) {
					
						// 如果优先级的值相等，是不允许的，这里需要引起注意，个人建议一般还是使用@Primary吧
						if (candidatePriority.equals(highestPriority)) {
							throw new NoUniqueBeanDefinitionException(requiredType, candidates.size(),
									"Multiple beans found with the same priority ('" + highestPriority +
									"') among candidates: " + candidates.keySet());
						}
						else if (candidatePriority < highestPriority) {
							highestPriorityBeanName = candidateBeanName;
							highestPriority = candidatePriority;
						}
					}
					else {
						highestPriorityBeanName = candidateBeanName;
						highestPriority = candidatePriority;
					}
				}
			}
		}
		return highestPriorityBeanName;
	}

protected Map<String, Object> findAutowireCandidates(
			@Nullable String beanName, Class<?> requiredType, DependencyDescriptor descriptor) {

		// 获取类型匹配的bean的beanName列表（包括父容器，但是此时还没有进行泛型的精确匹配）
		String[] candidateNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
				this, requiredType, true, descriptor.isEager());
		//存放结果的Map(beanName -> bena实例)  最终会return的
		Map<String, Object> result = new LinkedHashMap<>(candidateNames.length);
		
		//如果注入类型是特殊类型或其子类，会将特殊类型的实例添加到结果
		// 哪些特殊类型呢？上面截图有，比如你要注入ApplicationContext、BeanFactory等等
		for (Class<?> autowiringType : this.resolvableDependencies.keySet()) {
			if (autowiringType.isAssignableFrom(requiredType)) {
				Object autowiringValue = this.resolvableDependencies.get(autowiringType);
				autowiringValue = AutowireUtils.resolveAutowiringValue(autowiringValue, requiredType);
				if (requiredType.isInstance(autowiringValue)) {
					result.put(ObjectUtils.identityToString(autowiringValue), autowiringValue);
					break;
				}
			}
		}
			
		// candidateNames可能会有多个，这里就要开始过滤了，比如@Qualifier、泛型等等
		for (String candidate : candidateNames) {
			//不是自引用 && 符合注入条件
			// 自引用的判断：找到的候选的Bean的名称和当前Bean名称相等 或者 当前bean名称等于工厂bean的名称~~~~~~~
			// isAutowireCandidate：这个方法非常的关键，判断该bean是否允许注入进来。泛型的匹配就发生在这个方法里，下面会详解
			if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, descriptor)) {
				addCandidateEntry(result, candidate, descriptor, requiredType);
			}
		}
			
		////结果集为空 && 注入属性是非数组、容器类型  那么Spring就会放宽注入条件，然后继续寻找
		// 什么叫放宽：比如泛型不要求精确匹配了、比如自引用的注入等等
		if (result.isEmpty() && !indicatesMultipleBeans(requiredType)) {
			// Consider fallback matches if the first pass failed to find anything...
			//// FallbackMatch：放宽对泛型类型的验证  所以从这里用了一个新的fallbackDescriptor 对象   相当于放宽了泛型的匹配
			DependencyDescriptor fallbackDescriptor = descriptor.forFallbackMatch();
			for (String candidate : candidateNames) {
				if (!isSelfReference(beanName, candidate) && isAutowireCandidate(candidate, fallbackDescriptor)) {
					addCandidateEntry(result, candidate, descriptor, requiredType);
				}
			}
			if (result.isEmpty()) {
				// Consider self references as a final pass...
				// but in the case of a dependency collection, not the very same bean itself.
				//// 如果结果还是为空，Spring会将自引用添加到结果中  自引用是放在最后一步添加进去的
				for (String candidate : candidateNames) {
					if (isSelfReference(beanName, candidate) &&
							(!(descriptor instanceof MultiElementDescriptor) || !beanName.equals(candidate)) &&
							isAutowireCandidate(candidate, fallbackDescriptor)) {
						addCandidateEntry(result, candidate, descriptor, requiredType);
					}
				}
			}
		}
		return result;
	}

```

#### isAutowireCandidate(泛型和 @Qualifier)



resolveBeanClass 确保 bean 已被加载

GenericTypeAwareAutowireCandidateResolver 泛型匹配

checkGenericTypeMatch

ResolvableType#isAssignableFrom（自己的实现，泛型完整检查）

QualifierAnnotationAutowireCandidateResolver#isAutowireCandidate



## 16 @Autowired 注入：@Autowired 注入的规则和原理有哪些？

* @Autowired 注入过程
  * 元信息解析
  * 依赖查找
  * 依赖注入（字段、方法）
* AutowiredAnnotationBeanPostProcessor 

```
public AutowiredAnnotationBeanPostProcessor() {
		this.autowiredAnnotationTypes.add(Autowired.class);
		this.autowiredAnnotationTypes.add(Value.class);
		try {
			this.autowiredAnnotationTypes.add((Class<? extends Annotation>)
					ClassUtils.forName("javax.inject.Inject", AutowiredAnnotationBeanPostProcessor.class.getClassLoader()));
			logger.trace("JSR-330 'javax.inject.Inject' annotation found and supported for autowiring");
		}
		catch (ClassNotFoundException ex) {
			// JSR-330 API not available - simply skip.
		}
	}
```

> 1. 在`doCreateBean`中会先调用`applyMergedBeanDefinitionPostProcessors`，后执行`populateBean`所以会先调用`postProcessMergedBeanDefinition`后执行`InstantiationAwareBeanPostProcessor`的`postProcessProperties`。
> 2. `postProcessProperties`中有两个步骤：
>   （1）`findAutowiringMetadata `查找注入元数据，没有缓存就创建，具体是上一节内容。最终会返回 `InjectionMetadata`，里面包括待注入的 `InjectedElement `信息（field、method）等等
>   （2）执行 InjectionMetadata 的 inject 方法，具体为 AutowiredFieldElement 和AutowiredMethodElement 的 Inject 方法
>   （2.1）AutowiredFieldElement inject 具体流程：
>   （2.1.1）DependencyDescriptor的创建
>   （2.1.2）调用beanFactory的resolveDependency获取带注入的bean
>   （2.1.2.1）resolveDependency根据具体类型返回候选bean的集合或primary 的bean
>   （2.1.3）利用反射设置field
>
>
>
>   @Autowired注入过程（所有方法都在AutowiredAnnotationBeanPostProcessor#类里）
> （1）调用postProcessProperties()方法（spring 5.1之后才是这个方法名，5.1之前是postProcessPropertyValues）
> ​    该步骤信息点：
>
>        a. postProcessProperties 方法会比 bean 的 setXX(...) 方法先调用（元信息提前注入）
>        b.findAutowiringMetadata() 方法会找出一个 bean 加了 @Autowired 注解的字段（包括父类的），并且该方法做了缓存
>        c.xml配置的bean与bean之间是可以有继承关系的，有另一个周期（不是autowired的流程）是把配置super bean的属性合并到当前bean，之后会调用后置方法postProcessMergedBeanDefinition，该方法也会调用一次findAutowiringMetadata
>       d.经测试，postProcessMergedBeanDefinition会比postProcessProperties先执行，因此调用postProcessProperties时都是直接拿缓存
> （2）—>inject方法（获得对应的bean，然后通过反射注入到类的字段上）
> （3）inject方法会调用65讲的resolveDependency方法，这方法会根据@Autowired字段信息来匹配出符合条件的bean  

#### AutowireCandidateResolver



## 17 JSR-330 @lnject 注入：@lnject 与 @Autowired 的注入原理有怎样的联系？

* @lnject 注入过程
  * 如果 JSR-330 存在于 ClassPath 中 复用 AutowiredAnnotationBeanPostProcessor 实现



## 18 Java 通用注解注入原理：Spring 是如何实现 @Resource 和 @EJB 等注解注入的？

优先级越低（order数值越高），invoke方法会在后面调用

BeanPostProcessor 顺序

* CommonAnnotationBeanPostProcessor
  * 注入注解
    * javax.xml.ws.WebServiceRef
    *  javax.ejb.EJB
    * javax.annotation.Resource
  * 生命周期注解
    * javax.annotation.PostConstruct
    * javax.annotation.PreDestroy
    * 先子类后父类
* 先于 AutowiredAnnotationBeanPostProcessor  ，且多了生命周期处理



## 19 自定义依赖注入注解：如何最简化实现自定义依赖注入注解？

用@Autowired 元标注 注解

```
AutowiredAnnotationBeanPostProcessor#setAutowiredAnnotationTypes
覆盖或新增一个不同名BeanPostProcessor
注解处理顺序问题
linkedhashset
```



## 20 面试题精选

依赖查找(注入)的Bean会被缓存吗?

·单例Bean(Singleton) -会

·缓存位置：org.springframework.beans.factory.support.Default Singleton Bean Registry#singleton Objects

属性

·原型Bean(Prototype) -不会

·当依赖查询或依赖注入时， 根据Bean Definition每次创建

·其他Scope Bean

·request：每个ServletRequest内部缓存， 生命周期维持在每次HTTP请求

·session：每个HttpSession内部缓存， 生命周期维持在每个用户HTTP会话

·application：当前Servlet应用内部缓存



Bean的处理流程是怎样的?

·解析范围-Configuration Class中的@Bean方法

·方法类型-静态@Bean方法和实例@Bean方法



BeanFactory是如何处理循环依赖的?

·预备知识

·循环依赖开关(方法) -Abstract Auto wire Capable BeanFactory#set Allow Circular References

·单例工程(属性) -Default Singleton Bean Registry#singleton Factories

·获取早期未处理Bean(方法) -Abstract Auto wire Capable BeanFactory#get Early Bean Reference

·早期未处理Bean(属性) -Default Singleton Bean Registry#early Singleton Objects



BeanDefinitionReader

### AnnotatedBeanDefinitionReader

未实现 BeanDefinitionReader

### AnnotationConfigUtils

### AutowiredAnnotationBeanPostProcessor



postProcessMergedBeanDefinition#findAutowiringMetadata

* 非 Spring 托管

`AbstractAutowireCapableBeanFactory#autowireBean` -> `populateBean `-> `AutowiredAnnotationBeanPostProcessor#postProcessProperties `-> findAutowiringMetadata





PropertyValues   \<property>

InjectionMetadata

```
doCreateBean -> 实例化 -> 构建元信息 findAutowiringMetadata -> 初始化 populateBean

InstantiationAwareBeanPostProcessor#postProcessProperties
postProcessPropertyValues

applyPropertyValues
```

### ConfigurationClassPostProcessor（BeanFactoryPostProcessor）

ConfigurationClassUtils

```
invokeBeanFactoryPostProcessors
# 解析 
ConfigurationClassPostProcessor#processConfigBeanDefinitions
# 注解处理 加载为 class
ConfigurationClassParser#doProcessConfigurationClass
# 注册 bean
ConfigurationClassBeanDefinitionReader#loadBeanDefinitionsForConfigurationClass
```

* TrackedConditionEvaluator 过滤不满足条件Condition 的


```java
private void loadBeanDefinitionsForConfigurationClass(
			ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {
        
		if (trackedConditionEvaluator.shouldSkip(configClass)) {
			String beanName = configClass.getBeanName();
			if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
				this.registry.removeBeanDefinition(beanName);
			}
			this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
			return;
		}

		if (configClass.isImported()) {
			registerBeanDefinitionForImportedConfigurationClass(configClass);
		}
		for (BeanMethod beanMethod : configClass.getBeanMethods()) {
			loadBeanDefinitionsForBeanMethod(beanMethod);
		}

		loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
		loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
	}
```

> Component  PropertySource ComponentScan Import ImportResource Bean





####　doProcessConfigurationClass

```
@Nullable
	protected final SourceClass doProcessConfigurationClass(ConfigurationClass configClass, SourceClass sourceClass)
			throws IOException {
			//1、解析嵌套内部类
			//2、解析@PropertySource  === 这是下面的内容 ====
		// 相当于拿到所有的PropertySource注解，注意PropertySources属于重复注解的范畴~~~
		for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
				sourceClass.getMetadata(), PropertySources.class,
				org.springframework.context.annotation.PropertySource.class)) {
			
			// 这个判断目前来说是个恒等式~~~  所以的内置实现都是子接口ConfigurableEnvironment的实现类~~~~
			// processPropertySource：这个方法只真正解析这个注解的地方~~~
			if (this.environment instanceof ConfigurableEnvironment) {
				processPropertySource(propertySource);
			} else {
				logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
						"]. Reason: Environment must implement ConfigurableEnvironment");
			}
		}
			//3、解析@ComponentScan
			//4、解析@Import
			//5、解析@ImportResource 
		//拿到这个注解~~~~~~~~~~~
		AnnotationAttributes importResource = AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
		if (importResource != null) {
			String[] resources = importResource.getStringArray("locations");
			// readerClass 这个在自定义规则也是非常重要的一块内容~~~~~
			Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
			for (String resource : resources) {
				
				// 显然它还支持${}这种方式去环境变量里取值的~~~比如spring-beans-${profie}.xml等
				String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
				// 此处仅仅是吧注解解析掉，然后作为属性添加到configClass里面去，还并不是它真正的执行时机~~~~~
				configClass.addImportedResource(resolvedResource, readerClass);
			}
		}
			//6、解析@Bean
			//7、解析接口default方法~~~ 也可以用@Bean标注
			//8、解析super class父类
	}
}

```



### BeanDefinitionRegistryPostProcessor（BeanFactoryPostProcessor）

BeanDefinitionRegistry  早于一般 BeanFactoryPostProcessor

自定义  ->  PriorityOrdered -> Ordered -> the rest



### MergedBeanDefinitionPostProcessor（BeanPostProcessor）

```
MergedBeanDefinitionPostProcessor#applyMergedBeanDefinitionPostProcessors
```

处理执行早于 InstantiationAwareBeanPostProcessor

### InstantiationAwareBeanPostProcessor（BeanPostProcessor）

### 循环依赖

只支持 singlton   setter方式循环依赖

bean 实例后，判断是否提前暴露 exposedbean，再依赖注入

```
//循环依赖的处理
SmartInstantiationAwareBeanPostProcessor#getEarlyBeanReference
InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation

主要逻辑在AbstractAutoProxyCreator中
实现了getEarlyBeanReference,会在这里直接返回一个代理类  
然后放到 earlyProxyReferences, 标记这个 bean 已经被 early 处理过了  
然后在 postProcessAfterInitialization，会判定如果在 early 处理过了，就不需要在进行代理了


spring在初始化完之后，会check一遍
如果有循环依赖，即有early对象了

而且bean没有在beanPostProcessor被替换掉，那么就返回early对象到容器里
然后被修改过了，就会抛错处理
```

先从一级缓存singletonObjects中去获取。（如果获取到就直接return）
如果获取不到或者对象正在创建中（isSingletonCurrentlyInCreation()），那就再从二级缓存earlySingletonObjects中获取。（如果获取到就直接return）
如果还是获取不到，且允许singletonFactories（allowEarlyReference=true）通过getObject()获取。就从三级缓存singletonFactory.getObject()获取。（如果获取到了就从singletonFactories中移除，并且放进earlySingletonObjects。其实也就是从三级缓存移动（是剪切、不是复制哦~）到了二级缓存


# 第七章：Spring loC 依赖来源（Dependency Sources）

## 01 依赖查找的来源：除容器内建和自定义 Spring Bean 之外，还有其他来源提供依赖查找吗？

| 来源                  | 配置元数据                                   |
| --------------------- | -------------------------------------------- |
| Spring BeanDefinition | <bean id="user" class="org.geekbang...User"> |
|                       | @Bean public User user(){...}                |
|                       | BeanDefinitionBuilder                        |
| 单例对象              | API 实现                                     |

* Spring 內建 BeanDefintion

| Bean 名称                                                    | Bean 实例                                 | 使用场景                                              |
| ------------------------------------------------------------ | ----------------------------------------- | ----------------------------------------------------- |
| org.springframework.context. annotation.internalConfiguration AnnotationProcessor | ConfigurationClassPostProcessor 对象      | 处理 Spring 配置类                                    |
| org.springframework.context. annotation.internalAutowired AnnotationProcessor | AutowiredAnnotationBeanPostProcessor 对象 | 处理 @Autowired 以及 @Value                           |
| org.springframework.context. annotation.internalCommon AnnotationProcessor | CommonAnnotationBeanPostProcessor 对象    | （条件激活）处理 JSR-250 注解，                       |
| org.springframework.context. event.internalEventListener Processor | EventListenerMethodProcessor对象          | 处理标注 @EventListener 的                            |
| org.springframework.context. event.internalEventListener Factory | DefaultEventListenerFactory 对象          | @EventListener 事件监听方法适配为 ApplicationListener |

* Spring 內建单例对象

| Bean 名称                   | Bean 实例                        | 使用场景                |
| --------------------------- | -------------------------------- | ----------------------- |
| environment                 | Environment 对象                 | 外部化配置以及 Profiles |
| systemProperties            | java.util.Properties 对象        | Java 系统属性           |
| systemEnvironment           | java.util.Map 对象               | 操作系统环境变量        |
| messageSource               | MessageSource 对象               | 国际化文案              |
| lifecycleProcessor          | LifecycleProcessor 对象          | Lifecycle Bean 处理器   |
| applicationEventMulticaster | ApplicationEventMulticaster 对象 | Spring 事件广播器       |



## 02 依赖注入的来源：难道依赖注入的来源与依赖查找的不同吗？

| 来源                   | 配置元数据                                   |
| ---------------------- | -------------------------------------------- |
| Spring BeanDefinition  | <bean id="user" class="org.geekbang...User"> |
|                        | @Bean public User user(){...}                |
|                        | BeanDefinitionBuilder                        |
| 单例对象               | API 实现                                     |
| 非 Spring 容器管理对象 | Resolvable Dependency                        |

@Value 外部化配置



## 03 Spring 容器管理和游离对象：为什么会有管理对象和游离对象？

| 来源 Spring Bean      | 对象                        | 生命周期管理                | 配置元信息 | 使用场景           |
| --------------------- | --------------------------- | --------------------------- | ---------- | ------------------ |
| Spring BeanDefinition | 是                          | 是                          | 有         | 依赖查找、依赖注入 |
| 单体对象              | 是                          | <font color="red">否</font> | 无         | 依赖查找、依赖注入 |
| Resolvable Dependency | <font color="red">否</font> | <font color="red">否</font> | 无         | 依赖注入           |

单体对象就是已初始化的外部 Java 对象，在 Spring 容器中是唯一的



## 04 Spring Bean Definition 作为依赖来源：Spring Bean 的来源

* 要素
  * 元数据：BeanDefinition
  * 注册：BeanDefinitionRegistry#registerBeanDefinition
  * 类型：延迟和非延迟
  * 顺序：Bean 生命周期顺序按照注册顺序



## 05 单例对象作为依赖来源：单体对象与普通 Spring Bean 存在哪些差异？

* 要素
  * 来源：外部普通 Java 对象（不一定是 POJO）
  * 注册：SingletonBeanRegistry#registerSingleton
* 限制
  * 无生命周期管理
  * 无法实现延迟初始化 Bean



## 06 非 Spring 容器管理对象作为依赖来源：如何理解 ResolvableDependency?

* 要素
  * 注册：ConfigurableListableBeanFactory#registerResolvableDependency
* 限制
  * 无生命周期管理
  * 无法实现延迟初始化 Bean
  * 无法通过依赖查找



注册非spring管理的依赖对象时,可以通过两种方式实现
1.通过ApplicationContext.getBeanFactory()获取AnnotationConfigApplicationContext创建时初始化的DefaultListableBeanFactory对象,然后调用registryResolveableDependency来注册,为啥不能通过getAutowireCapableBeanFactory()来获取beanFactory对象,因为方法中需要beanFactory已被激活即执行了ApplicationContext的refresh()操作之后
2.通过addBeanFactoryPostProcessor回调方式实现,因为当refresh()方法执行invokeBeanFactoryPostProcessors()时会遍历已创建的beanFactoryPostProcessors集合对象来执行postProcessBeanFactory()方法



## 07 外部化配置作为依赖来源：@Value 是如何将外部化配置注入 Spring Bean 的？

* 要素
  * 类型：非常规 Spring 对象依赖来源
* 限制
  * 无生命周期管理
  * 无法实现延迟初始化 Bean
  * 无法通过依赖查找



类型转换，原型注入

EmbeddedValueResolver

## 08 面试题精选

注入和查找的依赖来源是否相同？

答：否，依赖查找的来源仅限于 Spring BeanDefinition 以及单例对象，而依赖注入的来源还包括 Resolvable Dependency 以及@Value 所标注的外部化配置

 单例对象能在 IoC 容器启动后注册吗？

答：可以的，单例对象的注册与 BeanDefinition 不同，BeanDefinition 会被 ConfigurableListableBeanFactory#freezeConfiguration() 方法影响，从而冻结注册，单例对象则没有这个限制。

Spring 依赖注入的来源有哪些？

答：
Spring BeanDefinition
单例对象
Resolvable Dependency
@Value 外部化配置





# 第八章：Spring Bean 作用域（Scopes）

## 01 Spring Bean 作用域：为什么 Spring Bean 需要多种作用域？



## 02 "singleton" Bean 作用域：单例 Bean 在当前 Spring 应用真是唯一的吗？

当前 bean factory是唯一的



## 03 "prototype" Bean 作用域：原型 Bean 在哪些场景下会创建新的实例？

Spring容器无法管理  "prototype" Bean 的完整生命周期，也没有办法记录实例的存在。销毁回调方法将不会执行，可以利用 BeanPostProcessor 进行清扫工作（不那么合理的）

用一个 disposebean 管理 ，prototype bean 的 destroy

```
// 结论一：
// Singleton Bean 无论依赖查找还是依赖注入，均为同一个对象
// Prototype Bean 无论依赖查找还是依赖注入，均为新生成的对象

// 结论二：
// 如果依赖注入集合类型的对象，Singleton Bean 和 Prototype Bean 均会存在一个
// Prototype Bean 有别于其他地方的依赖注入 Prototype Bean

// 结论三：
// 无论是 Singleton 还是 Prototype Bean 均会执行初始化方法回调
// 不过仅 Singleton Bean 会执行销毁方法回调
```



## 04 "request" Bean 作用域：request Bean 会在每次 HTTP 请求创建新的实例吗？

* AbstractRequestAttributesScope
* 生成代理，返回时再返回不同对象

> org.springframework.web.servlet.FrameworkServlet#initContextHolders 方法，其中会将 ServletRequestAttributes

```
public Object get(String name, ObjectFactory<?> objectFactory) {
   RequestAttributes attributes = RequestContextHolder.currentRequestAttributes();
   Object scopedObject = attributes.getAttribute(name, getScope());
   if (scopedObject == null) {
      scopedObject = objectFactory.getObject();
      attributes.setAttribute(name, scopedObject, getScope());
      // Retrieve object again, registering it for implicit session attribute updates.
      // As a bonus, we also allow for potential decoration at the getAttribute level.
      Object retrievedObject = attributes.getAttribute(name, getScope());
      if (retrievedObject != null) {
         // Only proceed with retrieved object if still present (the expected case).
         // If it disappeared concurrently, we return our locally created instance.
         scopedObject = retrievedObject;
      }
   }
   return scopedObject;
}
```



## 05 "session" Bean 作用域：session Bean 在 Spring MVC 场景下存在哪些局限性？

> Spring 将 Bean 的作用域分为三种，singleton、prototype、自定义 Scope，
>
> 在 AbstractBeanFactory#doGetBean 创建 bean 时根据三种情况分别创建对象。其中 singleton、prototype 是 Spring IoC 内置的，自定义 Scope 需要实现 Scope 接口，通过 get 方法创建对象。 
>
> request/session 这两种自定义 Scope 是为了解决 web 场景，RequestScope/SessionScope 会将创建的对象和 HttpRequest/HttpSession 绑定在一起。

```
public class SessionScope extends AbstractRequestAttributesScope {
   ...
   @Override
   public Object get(String name, ObjectFactory<?> objectFactory) {
      Object mutex = RequestContextHolder.currentRequestAttributes().getSessionMutex();
      synchronized (mutex) {
         return super.get(name, objectFactory);
      }
   }
   ...
}
```



## 06 "application" Bean 作用域：application Bean 是否真的有必要？

servletcontext

类似 singlton

```
// JSP EL 变量搜索路径 page -> request -> session -> application(ServletContext)
```

## 07 自定义 Bean 作用域：设计 Bean 作用域应该注意哪些原则？

> 扩展 spring 的 scope 参考spring3.0自带的org.springframework.context.support.SimpleThreadScope

实现 scope 中的方法



## 08 课外资料：Spring Cloud RefreshScope 是如何控制 Bean 的动态刷新？



## 09 面试题精选



# 第九章：Spring Bean 生命周期（Bean Lifecycle）

## 01 Spring Bean 元信息配置阶段：BeanDefinition 配置与扩展

* BeanDefinition 配置
  * 面向资源
    * XML 配置
    * Properties 资源配置
  * 面向注解
  * 面向 API

PropertiesBeanDefinitionReader

## 02 Spring Bean 元信息解析阶段：BeanDefinition 的解析

> “配置”是将元数据准备，比如 XML <bean> 等，“解析”是将元数据转化为运行时数据，如 <bean> 编程 BeanDefinition

* 面向资源 BeanDefinition 解析
  * BeanDefinitionReader
  * XML 解析器 - BeanDefinitionParser
* 面向注解 BeanDefinition 解析
  * AnnotatedBeanDefinitionReader



## 03 Spring Bean 注册阶段：BeanDefinition 与单体 Bean 注册

> updatedDefinitions 是非线程安全的，可以减少一些线程重进入的计算。

* BeanDefinition 注册接口
  * BeanDefinitionRegistry



## 04 Spring BeanDefinition 合并阶段：BeanDefinition 合并过程是怎样出现的？

* BeanDefinition 合并
  * 父子 BeanDefinition 合并
    * 当前 BeanFactory 查找
    * 层次性 BeanFactory 查找



## 05 Spring Bean Class 加载阶段：Bean ClassLoader 能够被替换吗？

* ClassLoader 类加载（传统的 java ClassLoader）
* Java Security 安全控制
* ConfigurableBeanFactory 临时 ClassLoader（loadtime weaver）

>  AbstractBeanFactory#resolveBeanClass中将string类型的beanClass 通过当前线程Thread.currentThread().getContextClassLoader(),Class.forName来获取class对象,将beanClass变为Class类型对象



## 06 Spring Bean 实例化前阶段：Bean 的实例化能否被绕开？

* 非主流生命周期 - Bean 实例化前阶段

  * `InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation`

    在 bean 实例化前回调，返回实例则不对bean实例化，返回null 则进行 spring bean 实例化 (doCreateBean);

这个阶段可以生成**代理对象**



## 07 Spring Bean 实例化阶段：Bean 实例是通过 Java 反射创建吗？

* 实例化方式
  * 传统实例化方式

    * 实例化策略 - InstantiationStrategy

  * 构造器依赖注入

    构造器参数 ，顺序，resolvedDependency





## 08 Spring Bean 实例化后阶段：Bean 实例化后是否一定会被使用吗？

* Bean 属性赋值（Populate）判断

  * `InstantiationAwareBeanPostProcessor#postProcessAfterInstantiation`

    在bean实例化后在填充 bean 属性之前回调，返回 true 则进行下一步的属性填充，返回 false 则不进行属性填充



## 09 Spring Bean 属性赋值前阶段：配置后的 PropertyValues 还有机会修改吗？

* Bean 属性值元信息

  * PropertyValues

* Bean 属性赋值前回调

  * Spring 1.2 - 5.0：InstantiationAwareBeanPostProcessor#postProcessPropertyValues

  * Spring 5.1：InstantiationAwareBeanPostProcessor#postProcessProperties

    postProcessProperties在属性赋值前的回调在 applyPropertyValues 之前操作可以对属性添加或修改等操作最后在通过applyPropertyValues应用bean对应的wapper对象


> **总结 InstantiationAwareBeanPostProcessor方法:**
> `postProcessBeforeInstantiation()` 在bean实例化前回调,返回实例则不对bean实例化,返回null 则进行 spring bean 实例化 (doCreateBean);
>
> `postProcessAfterInstantiation()`在bean实例化后在填充bean属性之前回调,返回true则进行下一步的属性填充,返回false:则不进行属性填充
>
> `postProcessProperties`在属性赋值前的回调在applyPropertyValues之前操作可以对属性添加或修改等操作最后在通过applyPropertyValues应用bean对应的wapper对象



## 10 Aware 接口回调阶段：众多 Aware 接口回调的顺序是安排的？

* Spring Aware 接口
  * `AbstractAutowireCapableBeanFactory#invokeAwareMethods`
    * BeanNameAware
    * BeanClassLoaderAware
    * BeanFactoryAware
  * BeanPostProcessors#applyBeanPostProcessorsBeforeInitialization
  * `ApplicationContextAwareProcessor#invokeAwareInterfaces`
    * EnvironmentAware
    * EmbeddedValueResolverAware
    * ResourceLoaderAware
    * ApplicationEventPublisherAware
    * MessageSourceAware
    * ApplicationContextAware





> 文中可以通过ConfigurableListableBeanFactory beanFactory = applicationContext.getBeanFactory()后给beanFactory.addBeanPostProcessor()添加自定义的InstantiationAwareBeanPostProcessor()处理器;因为在创建ClassPathXmlApplicationContext()对象时是默认调用了ApplicationContext.refresh()操作此时已经将beanFactory初始化;不过我们后面还要进行refresh()一次让beanPostProcessor加载到beanFactory中生效
>
>
>
> 因为 executeBeanFactory() 方法使用的是 DefaultListableBeanFactory，非 public 类 org.springframework.context.support.ApplicationContextAwareProcessor 无法被使用~
>
> ParserStrategyUtils#invokeAwareMethods



## 11 Spring Bean 初始化前阶段：BeanPostProcessor

`AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsBeforeInitialization`

* 已完成
  * Bean 实例化
  * Bean 属性赋值
  * Bean Aware 接口回调
* 方法回调
  * BeanPostProcessor#postProcessBeforeInitialization

> 通过`XmlBeanDefinitionReader#loadBeanDefinitions` 是将xml中bean对应的beanDefinition注册到 beanFactory 中,底层通过`BeanDefinitionReaderUtils#registerBeanDefinition()`方法实现,这个在BeanDefinition注册阶段讲过,这个时候只是注册beanDefinition没有像ApplicationContext.refresh()中的registerBeanPostProcessors()将bean post Processor添加到beanFactory的beanPostProcessors list集合中操作,
>
> 所以xml读取的时候需要手动的addBeanPostProcessor;如果通过ClassPathXmlApplicationContext创建ApplicationContext的方式xml中定义MyInstantiationAwareBeanPostProcessor是可以的,因为ClassPathXmlApplicationContext创建时会执行refresh()操作会从beanFactory中找到MyInstantiationAwareBeanPostProcessor bean后添加到beanPostProcessors的list集合中



## 12 Spring Bean 初始化阶段：@PostConstruct、InitializingBean 以及自定义方法

* 顺序

* @PostConstruct 标注方法

  * 处理类
    * InstantiationAwareBeanPostProcessor
    * InitDestroyAnnotationBeanPostProcessor
      * CommonAnnotationBeanPostProcessor

  * 处理流程

    > CommonAnnotationBeanPostProcessor#postProcessMergedBeanDefinition ->
    >
    > InitDestroyAnnotationBeanPostProcessor#buildLifecycleMetadata ->
    >
    >
    >
    > applyBeanPostProcessorsBeforeInitialization ->  
    >
    > InitDestroyAnnotationBeanPostProcessor#postProcessBeforeInitialization
    >
    > InitDestroyAnnotationBeanPostProcessor#findLifecycleMetadata
    >
    > metadata.invokeInitMethods

* AbstractAutowireCapableBeanFactory#invokeInitMethods

  * 实现 InitializingBean 接口的 afterPropertiesSet() 方法 
  * 自定义初始化方法：invokeCustomInitMethod(xml 配置)





## 13 Spring Bean 初始化后阶段：BeanPostProcessor

`AbstractAutowireCapableBeanFactory#applyBeanPostProcessorsAfterInitialization`

* 方法回调
  * BeanPostProcessor#postProcessAfterInitialization



## 14 Spring Bean 初始化完成阶段：SmartlnitializingSingleton

getBean 之后 调用    最后覆盖的机会

* 方法回调
  * Spring 4.1 +：SmartInitializingSingleton#afterSingletonsInstantiated

@EventListener 生成代理

ApplicationContext在refresh的操作里等beanFactory的一系列操作，messageSource，注册listener等操作都完毕之后通过finishBeanFactoryInitialization开始实例化所有非懒加载的单例bean，具体是在finishBeanFactoryInitialization调用beanFactory#preInstantiateSingletons进行的，preInstantiateSingletons里面就是通过beanDefinitionNames循环调用getBean来实例化bean的，这里有个细节，beanDefinitionNames是拷贝到一个副本中，循环副本，使得还能注册新的beanDefinition.
getBean的操作就是我们之前那么多节课分析的一顿操作的过程，最终得到一个完整的状态的bean。 然后所有的非延迟单例都加载完毕之后，再重新循环副本，判断bean是否是SmartInitializingSingleton，如果是的话执行SmartInitializingSingleton#afterSingletonsInstantiated。这保证执行afterSingletonsInstantiated的时候的bean一定是完整的

## 15 Spring Bean 销毁前阶段：DestructionAwareBeanPostProcessor 用在怎样的场景？

AbstractAutowireCapableBeanFactory#registerDisposableBeanIfNecessary

DisposableBeanAdapter#filterPostProcessors

* 方法回调
  * DestructionAwareBeanPostProcessor#postProcessBeforeDestruction

## 16 Spring Bean 销毁阶段：@PreDestroy、DisposableBean 以及自定义方法

DisposableBeanAdapter#destroy

* @PreDestroy 标注方法
  * DestructionAwareBeanPostProcessor#postProcessBeforeDestruction
* 实现 DisposableBean 接口的 destroy() 方法
* 自定义销毁方法：invokeCustomDestroyMethod



## 17 Spring Bean 垃圾收集（GC）：何时需要 GC Spring Bean?

* Bean 垃圾回收（GC）
  * 关闭 Spring 容器（应用上下文）
  * 执行 GC
  * Spring Bean 覆盖的 finalize() 方法被回调



## 18 面试题精选

沙雕面试题—BeanPostProcessor 的使用场景有哪些？

答：BeanPostProcessor 提供 Spring Bean 初始化前和初始化后的生命周期回调，分别对应 postProcessBeforeInitialization 以及postProcessAfterInitialization 方法，允许对关心的 Bean 进行扩展，甚至是替换。

加分项：其中，ApplicationContext 相关的 Aware 回调也是基于
BeanPostProcessor 实现，即 ApplicationContextAwareProcessor



996面试题 —BeanFactoryPostProcessor 与BeanPostProcessor 的区别?

答：BeanFactoryPostProcessor 是 Spring BeanFactory（实际为ConfigurableListableBeanFactory） 的后置处理器，用于扩展 BeanFactory，或通过 BeanFactory 进行依赖查找和依赖注入。

加分项：BeanFactoryPostProcessor 必须有 Spring ApplicationContext
执行，BeanFactory 无法与其直接交互。
而 BeanPostProcessor 则直接与BeanFactory 关联，属于 N 对 1 的关系。



劝退面试题 — BeanFactory 是怎样处理 Bean 生命周期？

• BeanDefinition 注册阶段 - registerBeanDefinition
• BeanDefinition 合并阶段 - getMergedBeanDefinition
• Bean 实例化前阶段 - resolveBeforeInstantiation
• Bean 实例化阶段 - createBeanInstance
• Bean 初始化后阶段 - populateBean
• Bean 属性赋值前阶段 - populateBean
• Bean 属性赋值阶段 - populateBean
• Bean Aware 接口回调阶段 - initializeBean
• Bean 初始化前阶段 - initializeBean
• Bean 初始化阶段 - initializeBean
• Bean 初始化后阶段 - initializeBean
• Bean 初始化完成阶段 - preInstantiateSingletons
• Bean 销毁前阶段 - destroyBean
• Bean 销毁阶段 - destroyBean

### 内建BeanPostProcessor

AnnotationConfigApplicationContext

1. org.springframework.context.support.ApplicationContextAwareProcessor
2. org.springframework.context.annotation.ConfigurationClassPostProcessor$ImportAwareBeanPostProcessor
3. org.springframework.context.support.PostProcessorRegistrationDelegate$BeanPostProcessorChecker
4. org.springframework.context.annotation.CommonAnnotationBeanPostProcessor
5. org.springframework.beans.factory.annotation.AutowiredAnnotationBeanPostProcessor
6. org.springframework.context.support.ApplicationListenerDetector





# 第十章：Spring 配置元信息（Configuration Metadata）

## 01 Spring 配置元信息：Spring 存在哪些配置元信息？它们分别用在什么场景？

* 配置元信息
  * Spring Bean 配置元信息 - BeanDefinition
  * Spring Bean 属性元信息 - PropertyValues
  * Spring 容器配置元信息
  * Spring 外部化配置元信息 - PropertySource
  * Spring Profile 元信息 - @Profile

## 02 Spring Bean 配置元信息：BeanDefinition

* Bean 配置元信息 - BeanDefinition
  * GenericBeanDefinition：通用型 BeanDefinition
  * RootBeanDefinition：无 Parent 的 BeanDefinition 或者合并后 BeanDefinition
  * AnnotatedBeanDefinition：注解标注的 BeanDefinition

## 03 Spring Bean 属性元信息：PropertyValues

* Bean 属性元信息 - PropertyValues
  * 可修改实现 - MutablePropertyValues
  * 元素成员 - PropertyValue
* Bean 属性上下文存储 - AttributeAccessor
* Bean 元信息元素 - BeanMetadataElement

## 04 Spring 容器配置元信息

* Spring XML 配置元信息 - beans 元素相关



* Spring XML 配置元信息 - 应用上下文相关



## 05 基于 XML 文件装载 Spring Bean 配置元信息

* Spring Bean 配置元信息

| XML 元素         | 使用场景                                      |
| ---------------- | --------------------------------------------- |
| <beans:beans />  | 单 XML 资源下的多个 Spring Beans 配置         |
| <beans:bean />   | 单个 Spring Bean 定义（BeanDefinition）配置   |
| <beans:alias />  | 为 Spring Bean 定义（BeanDefinition）映射别名 |
| <beans:import /> | 加载外部 Spring XML 配置资源                  |



## 06 基于 Properties 文件装载 SpringBean 配置元信息：为什么 Spring 官方不推荐？

* Spring Bean 配置元信息

| Properties 属性名 | 使用场景                        |
| ----------------- | ------------------------------- |
| (class)           | Bean 类全称限定名               |
| (abstract)        | 是否为抽象的 BeanDefinition     |
| (parent)          | 指定 parent BeanDefinition 名称 |
| (lazy-init)       | 是否为延迟初始化                |
| (ref)             | 引用其他 Bean 的名称            |
| (scope)           | 设置 Bean 的 scope 属性         |
| ${n}              | n 表示第 n+1 个构造器参数       |

> 底层实现 - PropertiesBeanDefinitionReader



## 07 基于 Java 注解装载 Spring Bean 配置元信息

*  Spring 模式注解

| Spring 注解    | 场景说明           | 起始版本 |
| -------------- | ------------------ | -------- |
| @Repository    | 数据仓储模式注解   | 2.0      |
| @Component     | 通用组件模式注解   | 2.5      |
| @Service       | 服务模式注解       | 2.5      |
| @Controller    | Web 控制器模式注解 | 2.5      |
| @Configuration | 配置类模式注解     | 3.0      |

> 注解元标注体现的“派生性”

*  Spring Bean 定义注解

| Spring 注解 | 场景说明                                        | 起始版本 |
| ----------- | ----------------------------------------------- | -------- |
| @Bean       | 替换 XML 元素 <bean>                            | 3.0      |
| @DependsOn  | 替代 XML 属性 <bean depends-on="..."/>          | 3.0      |
| @Lazy       | 替代 XML 属性 <bean lazy-init="true\|falses" /> | 3.0      |
| @Primary    | 替换 XML 元素 <bean primary="true\|falses" />   | 3.0      |
| @Role       | 替换 XML 元素 <bean role="..." />               | 3.1      |
| @Lookup     | 替代 XML 属性 <bean lookup-method="...">        | 4.1      |

* Spring Bean 依赖注入注解

| Spring 注解 | 场景说明                            | 起始版本 |
| ----------- | ----------------------------------- | -------- |
| @Autowired  | Bean 依赖注入，支持多种依赖查找方式 | 2.5      |
| @Qualifier  | 细粒度的 @Autowired 依赖查找        | 2.5      |

| Java 注解 | 场景说明          | 起始版本 |
| --------- | ----------------- | -------- |
| @Resource | 类似于 @Autowired | 2.5      |
| @Inject   | 类似于 @Autowired | 2.5      |

* Spring Bean 条件装配注解

| Spring 注解  | 场景说明       | 起始版本 |
| ------------ | -------------- | -------- |
| @Profile     | 配置化条件装配 | 3.1      |
| @Conditional | 编程条件装配   | 4.0      |

* Spring Bean 生命周期回调注解

| Spring 注解    | 场景说明                                                     | 起始版本 |
| -------------- | ------------------------------------------------------------ | -------- |
| @PostConstruct | 替换 XML 元素 <bean init-method="..." /> 或<br/>InitializingBean | 2.5      |
| @PreDestroy    | 替换 XML 元素 <bean destroy-method="..." /> 或<br/>DisposableBean | 2.5      |
> 内容来源于《Spring Boot 编程思想（核心篇）》 - “Spring 核心注解场景分类” 章节



## 08 Spring Bean 配置元信息底层实现

### Spring BeanDefinition 解析与注册

| 实现场景        | 实现类                         | 起始版本 |
| --------------- | ------------------------------ | -------- |
| XML 资源        | XmlBeanDefinitionReader        | 1.0      |
| Properties 资源 | PropertiesBeanDefinitionReader | 1.0      |
| Java 注解       | AnnotatedBeanDefinitionReader  | 3.0      |



#### Spring XML 资源 BeanDefinition 解析与注册

* 核心 API - XmlBeanDefinitionReader
  * 资源 - Resource
  * 底层 - BeanDefinitionDocumentReader
    * XML 解析 - Java DOM Level 3 API
    * BeanDefinition 解析 - BeanDefinitionParserDelegate
    * BeanDefinition 注册 - BeanDefinitionRegistry



#### Spring Properties 资源 BeanDefinition 解析与注册

- 核心 API - PropertiesBeanDefinitionReader
  - 资源
    - 字节流 - Resource
    - 字符流 - EncodedResouce
  - 底层
    - 存储 - java.util.Properties
    - BeanDefinition 解析 - API 内部实现
    - BeanDefinition 注册 - BeanDefinitionRegistry



#### Spring Java 注解 BeanDefinition 解析与注册

* 核心 API - AnnotatedBeanDefinitionReader
  * 资源
    * 类对象 - java.lang.Class
  * 底层
    * 条件评估 - ConditionEvaluator
    * Bean 范围解析 - ScopeMetadataResolver
    * BeanDefinition 解析 - 内部 API 实现
    * BeanDefinition 处理 -
      AnnotationConfigUtils.processCommonDefinitionAnnotations
    * BeanDefinition 注册 - BeanDefinitionRegistry



## 09 基于 XML 文件装载 Spring loC 容器配置元信息

* Spring IoC 容器相关 XML 配置

| 命名空间 | 所属模块       | Schema 资源 URL                                              |
| -------- | -------------- | ------------------------------------------------------------ |
| beans    | spring-beans   | https://www.springframework.org/schema/beans/spring-beans.xsd |
| context  | spring-context | https://www.springframework.org/schema/context/spring-context.xsd |
| aop      | spring-aop     | https://www.springframework.org/schema/aop/spring-aop.xsd    |
| tx       | spring-tx      | https://www.springframework.org/schema/tx/spring-tx.xsd      |
| util     | spring-beans   | https://www.springframework.org/schema/util/spring-util.xsd  |
| tool     | spring-beans   | https://www.springframework.org/schema/tool/spring-tool.xsd  |



## 10 基于 Java 注解装载 Spring loC 容器配置元信息

* Spring IoC 容器装配注解

| Spring 注解     | 场景说明                                    | 起始版本 |
| --------------- | ------------------------------------------- | -------- |
| @ImportResource | 替换 XML 元素 <import>                      | 3.1      |
| @Import         | 导入 Configuration Class                    | 4.0      |
| @ComponentScan  | 扫描指定 package 下标注 Spring 模式注解的类 | 3.1      |

* Spring IoC 配置属性注解

| Spring 注解      | 场景说明                         | 起始版本 |
| ---------------- | -------------------------------- | -------- |
| @PropertySource  | 配置属性抽象 PropertySource 注解 | 3.1      |
| @PropertySources | @PropertySource 集合注解         | 4.0      |

> 内容来源于《Spring Boot 编程思想（核心篇）》 - “Spring 核心注解场景分类” 章节



## 11 基于 Extensible XML authoring 扩展 Spring XML 元素

* Spring XML 扩展
  * 编写 XML Schema 文件：定义 XML 结构
  * 自定义 NamespaceHandler 实现：命名空间绑定
  * 自定义 BeanDefinitionParser 实现：XML 元素与 BeanDefinition 解析
  * 注册 XML 扩展：命名空间与 XML Schema 映射



* 触发时机
  * AbstractApplicationContext#obtainFreshBeanFactory
    * AbstractRefreshableApplicationContext#refreshBeanFactory
      * AbstractXmlApplicationContext#loadBeanDefinitions
        * ...
          * XmlBeanDefinitionReader#doLoadBeanDefinitions
            * ...
              * BeanDefinitionParserDelegate#parseCustomElement

> 如 DubboNamespaceHandler

## 12 Extensible XML authoring 扩展原理

* 核心流程
  * BeanDefinitionParserDelegate#parseCustomElement(org.w3c.dom.Element, BeanDefinition)
    * 获取 namespace
    * 通过 namespace 解析 NamespaceHandler
    * 构造 ParserContext
    * 解析元素，获取 BeanDefinintion



## 13 基于 Properties 文件装载外部化配置

* 注解驱动
  * @org.springframework.context.annotation.PropertySource
  * @org.springframework.context.annotation.PropertySources
* API 编程
  * org.springframework.core.env.PropertySource
  * org.springframework.core.env.PropertySources

> 详细讨论将在《Spring Environment 抽象》 中展开



## 14 基于 YAML 文件装载外部化配置

* API 编程
  * org.springframework.beans.factory.config.YamlProcessor
    * org.springframework.beans.factory.config.YamlMapFactoryBean
    * org.springframework.beans.factory.config.YamlPropertiesFactoryBean



## 15 面试题精选

* 沙雕面试题 - Spring 內建 XML Schema 常见有哪些？

| 命名空间 | 所属模块       | Schema 资源 URL                                              |
| -------- | -------------- | ------------------------------------------------------------ |
| beans    | spring-beans   | <https://www.springframework.org/schema/beans/spring-beans.xsd> |
| context  | spring-context | <https://www.springframework.org/schema/context/spring-context.xsd> |
| aop      | spring-aop     | <https://www.springframework.org/schema/aop/spring-aop.xsd>  |
| tx       | spring-tx      | <https://www.springframework.org/schema/tx/spring-tx.xsd>    |
| util     | spring-beans   | <https://www.springframework.org/schema/util/spring-util.xsd> |
| tool     | spring-beans   | <https://www.springframework.org/schema/tool/spring-tool.xsd> |

* 996 面试题 - Spring配置元信息具体有哪些？

答：
• Bean 配置元信息：通过媒介（如 XML、Proeprties 等），解析 BeanDefinition
• IoC 容器配置元信息：通过媒介（如 XML、Proeprties 等），控制 IoC 容器行为，
比如注解驱动、AOP 等
• 外部化配置：通过资源抽象（如 Proeprties、YAML 等），控制 PropertySource
• Spring Profile：通过外部化配置，提供条件分支流程



* 劝退面试题 - Extensible XML authoring 的缺点？

答：
• 高复杂度：开发人员需要熟悉 XML Schema，spring.handlers，spring.schemas
以及 Spring API 。
• 嵌套元素支持较弱：通常需要使用方法递归或者其嵌套解析的方式处理嵌套（子）元
素。
• XML 处理性能较差：Spring XML 基于 DOM Level 3 API 实现，该 API 便于理解，然
而性能较差。
• XML 框架移植性差：很难适配高性能和便利性的 XML 框架，如 JAXB。



# 第十一章：Spring 资源管理（Resources）

## 01 引入动机：为什么 Spring 不使用 Java 标准资源管理，而选择重新发明轮子？

* Java 标准资源管理强大，然而扩展复杂，资源存储方式并不统一
* Spring 要自立门户（重要的话，要讲三遍）
* Spring “抄”、“超” 和 “潮”



## 02 Java 标准资源管理：Java URL 资源管理存在哪些潜规则？

*  Java 标准资源定位

| 职责         | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| 面向资源     | 文件系统、artifact（jar、war、ear 文件）以及远程资源（HTTP、FTP 等） |
| API 整合     | java.lang.ClassLoader#getResource、java.io.File 或 java.net.URL |
| 资源定位     | java.net.URL 或 java.net.URI                                 |
| 面向流式存储 | java.net.URLConnection                                       |
| 协议扩展     | java.net.URLStreamHandler 或 java.net.URLStreamHandlerFactory |



* Java URL 协议扩展
  * 基于 java.net.URLStreamHandlerFactory
  * 基于 java.net.URLStreamHandler



* 基于 java.net.URLStreamHandlerFactory 扩展协议

  * ![URL 资源管理](..\..\image\JavaURL扩展.png)

  * JDK 1.8 內建协议实现

    | 协议   | 实现类                              |
    | ------ | ----------------------------------- |
    | file   | sun.net.www.protocol.file.Handler   |
    | ftp    | sun.net.www.protocol.ftp.Handler    |
    | http   | sun.net.www.protocol.http.Handler   |
    | https  | sun.net.www.protocol.https.Handler  |
    | jar    | sun.net.www.protocol.jar.Handler    |
    | mailto | sun.net.www.protocol.mailto.Handler |
    | netdoc | sun.net.www.protocol.netdoc.Handler |

  * 实现类名必须为 “Handler”

    | 实现类命名规则 | 说明                                                         |
    | -------------- | ------------------------------------------------------------ |
    | 默认           | sun.net.www.protocol.${protocol}.Handler                     |
    | 自定义         | 通过 Java Properties java.protocol.handler.pkgs 指定实现类包名，实<br/>现类名必须为“Handler”。如果存在多包名指定，通过分隔符 “ |



## 03 Spring 资源接口：Resource 接口有哪些语义？它是否“借鉴”了 SUN 的实现呢？

• 资源接口

| 类型       | 接口                                                  |
| ---------- | ----------------------------------------------------- |
| 输入流     | org.springframework.core.io.InputStreamSource（Root） |
| 只读资源   | org.springframework.core.io.Resource                  |
| 可写资源   | org.springframework.core.io.WritableResource          |
| 编码资源   | org.springframework.core.io.support.EncodedResource   |
| 上下文资源 | org.springframework.core.io.ContextResource           |

#### AbstractResource



## 04 Spring 内建 Resource 实现：Spring 框架提供了多少种内建的 Resource 实现呢？

• 內建实现

| 资源来源       | 资源协议       | 实现类                                                       |
| -------------- | -------------- | ------------------------------------------------------------ |
| Bean 定义      | 无             | org.springframework.beans.factory.support.BeanDefinitionResource |
| 数组           | 无             | org.springframework.core.io.ByteArrayResource                |
| 类路径         | classpath:/    | org.springframework.core.io.ClassPathResource                |
| 文件系统       | file:/         | org.springframework.core.io.FileSystemResource               |
| URL            | URL 支持的协议 | org.springframework.core.io.UrlResource                      |
| ServletContext | 无             | org.springframework.web.context.support.ServletContextResource |

> 整合  Url，Class ，ClassLoader 加载资源方式

## 05 Spring Resource 接口扩展：Resource 能否支持写入以及字符集编码？

* 可写资源接口
  * org.springframework.core.io.WritableResource
    * org.springframework.core.io.FileSystemResource

    * org.springframework.core.io.FileUrlResource（@since 5.0.2）

    * org.springframework.core.io.PathResource（@since 4.0 & @Deprecated）

      > java.nio.file.Path
* 编码资源接口

  * org.springframework.core.io.support.EncodedResource



## 06 Spring 资源加载器：为什么说 Spring 应用上下文也是一种 Spring 资源加载器？

* Resource 加载器
  * org.springframework.core.io.ResourceLoader
    * org.springframework.core.io.DefaultResourceLoader
      * org.springframework.core.io.FileSystemResourceLoader
      * org.springframework.core.io.ClassRelativeResourceLoader
      * **org.springframework.context.support.AbstractApplicationContext**
        * 关联 ResourcePatternResolver


```java

	// @since 10.03.2004  小细节：Resource接口是@since 28.12.2003   所以这个接口晚了大概4个月的样子
	public interface ResourceLoader {
		// 常量：classpath:
		// 其实ResourceLoader接口只提供了classpath前缀的支持。而classpath*的前缀支持是在它的子接口ResourcePatternResolver中
		String CLASSPATH_URL_PREFIX = ResourceUtils.CLASSPATH_URL_PREFIX;
		
		// 这个方法可谓是核心方法：返回Resource实例
		// 它的目的很简单：就是希望程序员在使用时，不需要太在意使用的是哪个实例。面向接口操作即可
		// 至于选取哪个实例去操作，交给框架来完成
		Resource getResource(String location);
		
		// Expose the ClassLoader used by this ResourceLoader.
		// 暴露出ResourceLoader使用的类加载器~~~
		@Nullable
		ClassLoader getClassLoader();
	}
	
	// @since 10.03.2004
	public class DefaultResourceLoader implements ResourceLoader {
		
		// 这里面classLoader是允许为null的
		@Nullable
		private ClassLoader classLoader;
		// 这个特别重要：ProtocolResolver这个接口事Spring为开发者提供了自定义扩展接口（允许我们自己去介入参与到具体的获取资源的处理上，后面getResouce方法可议看出来）
		// ProtocolResolver接口Spring没有提供任何实现，开发者可议自己实现，从而参与到资源获取的路子上去
		// 备注：这个接口@since 4.3  所以只有Spring4.3后才有这个能力哦~~~
		private final Set<ProtocolResolver> protocolResolvers = new LinkedHashSet<>(4);
		private final Map<Class<?>, Map<Resource, ?>> resourceCaches = new ConcurrentHashMap<>(4);
	
		// ClassLoader可以不指定（一般情况下也不需要指定）
		public DefaultResourceLoader() {
			this.classLoader = ClassUtils.getDefaultClassLoader();
		}
		public DefaultResourceLoader(@Nullable ClassLoader classLoader) {
			this.classLoader = classLoader;
		}
	
		// 我们可以自己实现一个ProtocolResolver ，然后实现我们自己的获取资源的逻辑~~~下面会有示例
		public void addProtocolResolver(ProtocolResolver resolver) {
			Assert.notNull(resolver, "ProtocolResolver must not be null");
			this.protocolResolvers.add(resolver);
		}
	
		// @since 5.0  和ASM有关
		public <T> Map<Resource, T> getResourceCache(Class<T> valueType) {
			return (Map<Resource, T>) this.resourceCaches.computeIfAbsent(valueType, key -> new ConcurrentHashMap<>());
		}
		...
		// 这个是核心方法~~~
		@Override
		public Resource getResource(String location) {
			Assert.notNull(location, "Location must not be null");
			
			// 首先，Spring会看我们自己有没有实现自己的ProtocolResolver 若有实现，会先以我们自己的为准
			// 备注：它会把ResourceLoader传给开发者，这点特别重要~~~
			for (ProtocolResolver protocolResolver : this.protocolResolvers) {
				Resource resource = protocolResolver.resolve(location, this);
				if (resource != null) {
					return resource;
				}
			}
			
			// 如果以/打头，就交给getResourceByPath()，注意，它是一个protected方法，子类是可议复写的
			if (location.startsWith("/")) {
				return getResourceByPath(location);
			}
			// 如果以classpath:打头，毫无疑问，交给ClassPathResource
			// 需要注意的是，此处必须`classpath:`  区分大小写的哦
			else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
				return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
			}
			// 最后的处理方式：就是当作一个URL来处理
			else {
				try {
					// Try to parse the location as a URL...
					// 如果是文件类型，交给FileUrlResource（@since 5.0.2）
					URL url = new URL(location);
					return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
				}
				catch (MalformedURLException ex) {
					// No URL -> resolve as resource path.
					// fallback  相当于还是交给子类去弄吧
					return getResourceByPath(location);
				}
			}
		}
		
		// 它是一个protected方法，很多子类都有复写它。可议看到，默认的处理方式是：去classpth里查找这个资源
		// ClassPathContextResource事一个protected内部类 ` extends ClassPathResource implements ContextResource`
		protected Resource getResourceByPath(String path) {
			return new ClassPathContextResource(path, getClassLoader());
		}
		
	}
```






## 07 Spring 通配路径资源加载器：如何理解路径通配 Ant 模式？

* 通配路径 ResourceLoader
  * org.springframework.core.io.support.ResourcePatternResolver
    * **org.springframework.core.io.support.PathMatchingResourcePatternResolver**
    * org.springframework.context.ApplicationContext

* 路径匹配器
  * org.springframework.util.PathMatcher
    * Ant 模式匹配实现 - org.springframework.util.AntPathMatcher

用于解析资源文件的`策略接口`，其特殊的地方在于，它应该提供带有*号这种通配符的资源路径。

模式匹配

```java
public class PathMatchingResourcePatternResolver implements ResourcePatternResolver {
	// 内部持有一个resourceLoader的引用
	private final ResourceLoader resourceLoader;
	// 我们发现，它内部使用是AntPathMatcher进行匹配的（Spring内部AntPathMatcher是PathMatcher接口的唯一实现。
	//如果你想改变此匹配规则，你可以自己实现（当然这是完全没必要的））
	private PathMatcher pathMatcher = new AntPathMatcher();
	
	// 默认使用的DefaultResourceLoader  当然也可以指定
	public PathMatchingResourcePatternResolver() {
		this.resourceLoader = new DefaultResourceLoader();
	}
	public PathMatchingResourcePatternResolver(ResourceLoader resourceLoader) {
		Assert.notNull(resourceLoader, "ResourceLoader must not be null");
		this.resourceLoader = resourceLoader;
	}
	public PathMatchingResourcePatternResolver(@Nullable ClassLoader classLoader) {
		this.resourceLoader = new DefaultResourceLoader(classLoader);
	}
	...
	
	// 显然，我们也可以自己指定解析器，默认使用AntPathMatcher
	public void setPathMatcher(PathMatcher pathMatcher) {
		Assert.notNull(pathMatcher, "PathMatcher must not be null");
		this.pathMatcher = pathMatcher;
	}
	// 最终是委托给ResourceLoader去做了
	@Override
	public Resource getResource(String location) {
		return getResourceLoader().getResource(location);
	}

	// 这个是核心方法~~~~~~~
	@Override
	public Resource[] getResources(String locationPattern) throws IOException {
		Assert.notNull(locationPattern, "Location pattern must not be null");

		// 以`classpath*:`打头~~~
		if (locationPattern.startsWith(CLASSPATH_ALL_URL_PREFIX)) {
			// 把前最后面那部分截取出来，看看是否是模版(包含*和?符号都属于模版，否则不是)
			if (getPathMatcher().isPattern(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()))) {
				// a class path resource pattern
				// 事patter，那就交给这个方法，这个是个核心方法   这里传入的是locationPattern
				return findPathMatchingResources(locationPattern);
			}
			else {
				// all class path resources with the given name
				// 如果不是pattern，那就完全匹配。去找所有的path下的匹配上的就成~~~
				return findAllClassPathResources(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()));
			}
		}
		
		// 不是以`classpath*:`打头的~~~~
		else {
			// 支持到tomcat的war:打头的方式~~~
			int prefixEnd = (locationPattern.startsWith("war:") ?  locationPattern.indexOf("*/") + 1 :
					locationPattern.indexOf(':') + 1);
			if (getPathMatcher().isPattern(locationPattern.substring(prefixEnd))) {
				return findPathMatchingResources(locationPattern);
			}
			
			// 如果啥都不打头，那就当作一个正常的处理,委托给ResourceLoader直接去处理
			else {
				// a single resource with the given name
				return new Resource[] {getResourceLoader().getResource(locationPattern)};
			}
		}
	}

	// 根据Pattern去匹配资源~~~~
	protected Resource[] findPathMatchingResources(String locationPattern) throws IOException {
		// 定位到跟文件夹地址。比如locationPattern=classpath:META-INF/spring.factories  得到classpath*:META-INF/
		// 若是classpath*:META-INF/ABC/*.factories 得到的就是 classpath*:META-INF/ABC/
		// 简单的说，就是截取第一个不是patter的地方的前半部分
		String rootDirPath = determineRootDir(locationPattern);
		// 后半部分  这里比如就是：*.factories
		String subPattern = locationPattern.substring(rootDirPath.length());
		
		// 这个递归就厉害了，继续调用了getResources("classpath*:META-INF/")方法
		// 相当于把该文件夹匹配的所有的资源（注意：可能会比较多的），最后在和patter匹配即可~~~~
		// 比如此处：只要jar里面有META-INF目录的  都会被匹配进来~~~~~~
		Resource[] rootDirResources = getResources(rootDirPath);
		Set<Resource> result = new LinkedHashSet<>(16);
		for (Resource rootDirResource : rootDirResources) {
			// resolveRootDirResource是留给子类去复写的。但是Spring没有子类复写此方法，默认实现是啥都没做~~~
			rootDirResource = resolveRootDirResource(rootDirResource);
			URL rootDirUrl = rootDirResource.getURL();
			
			// 这个if就一般不看了  是否为了做兼容~~~
			if (equinoxResolveMethod != null && rootDirUrl.getProtocol().startsWith("bundle")) {
				URL resolvedUrl = (URL) ReflectionUtils.invokeMethod(equinoxResolveMethod, null, rootDirUrl);
				if (resolvedUrl != null) {
					rootDirUrl = resolvedUrl;
				}
				rootDirResource = new UrlResource(rootDirUrl);
			}


			// 支持vfs协议（JBoss)~~~
			if (rootDirUrl.getProtocol().startsWith(ResourceUtils.URL_PROTOCOL_VFS)) {
				result.addAll(VfsResourceMatchingDelegate.findMatchingResources(rootDirUrl, subPattern, getPathMatcher()));
			}
		
			// 是否是jar文件或者是jar资源(显然大多数情况下都是此情况~~~)
			else if (ResourceUtils.isJarURL(rootDirUrl) || isJarResource(rootDirResource)) {
				// 把rootDirUrl, subPattern都交给doFindPathMatchingJarResources去处理
				result.addAll(doFindPathMatchingJarResources(rootDirResource, rootDirUrl, subPattern));
			}
			
			// 不是Jar文件（那就是本工程里字的META-INF目录~~~）
			// 那就没啥好说的，直接给个subPattern去匹配吧   注意这个方法名是File，上面是jar
			else {
				result.addAll(doFindPathMatchingFileResources(rootDirResource, subPattern));
			}
		}
		
		// 最终转换为数组返回。 注意此处的result是个set，是有去重的效果的~~~
		return result.toArray(new Resource[0]);
	}

	protected Set<Resource> doFindPathMatchingJarResources(Resource rootDirResource, URL rootDirURL, String subPattern) {
		... // 这个方法就源码不细说了，
		//Find all resources in jar files that match the given location pattern
		// 就是去这个jar里面去找所有的资源（默认利用Ant风格匹配~）
		//此处用到了`java.util.jar.JarFile`、`ZipFile`、`java.net.JarURLConnection`等等
		// 路径匹配：getPathMatcher().match(subPattern, relativePath) 使用的此方法去校对
	}

	// Find all resources in the file system that match the given location pattern
	// 简单的说，这个就是在我们自己的项目里找~~~~
	protected Set<Resource> doFindPathMatchingFileResources(Resource rootDirResource, String subPattern)
			throws IOException {

		// rootDir:最终是个绝对的路径地址，带盘符的。来代表META-INF这个文件夹~~~
		File rootDir;
		try {
			rootDir = rootDirResource.getFile().getAbsoluteFile();
		} catch (IOException ex) {
			return Collections.emptySet();
		}
		// FileSystem  最终是根据此绝对路径 去文件系统里找
		return doFindMatchingFileSystemResources(rootDir, subPattern);
	}

	// 这个比较简单：就是把该文件夹所有的文件都拿出来dir.listFiles()，然后一个个去匹配呗~~~~
	// 备注：子类`ServletContextResourcePatternResolver`复写了此方法~~~~~
	protected Set<Resource> doFindMatchingFileSystemResources(File rootDir, String subPattern) throws IOException {
		if (logger.isDebugEnabled()) {
			logger.debug("Looking for matching resources in directory tree [" + rootDir.getPath() + "]");
		}
		Set<File> matchingFiles = retrieveMatchingFiles(rootDir, subPattern);
		Set<Resource> result = new LinkedHashSet<>(matchingFiles.size());
		for (File file : matchingFiles) {]
			// 最终用FileSystemResource把File包装成一个Resource~
			result.add(new FileSystemResource(file));
		}
		return result;
	}
	...
}
```





## 08 Spring 通配路径模式扩展：如何扩展路径匹配的规则？

* 实现 org.springframework.util.PathMatcher



* 重置 PathMatcher
  - PathMatchingResourcePatternResolver#setPathMatcher



## 09 依赖注入 Spring Resource：如何在 XML 和 Java 注解场景注入 Resource 对象？

* 基于 @Value 实现
  * 如：
    @Value(“classpath:/...”)
    private Resource resource;



## 10 依赖注入 ResourceLoader：除了 ResourceLoaderAware 回调注入，还有哪些注入方法？

* 方法一：实现 ResourceLoaderAware 回调
* 方法二：@Autowired 注入 ResourceLoader
* 方法三：注入 ApplicationContext 作为 ResourceLoader



##11 面试题精选

 沙雕面试题 -Spring 配置资源中有哪些常见类型？

答：
• XML 资源
• Properties 资源
• YAML 资源

996 面试题 -请例举不同类型 Spring 配置资源？

答：

* XML 资源
  * 普通 Bean Definition XML 配置资源 - *.xml
  * Spring Schema 资源 - *.xsd
* Properties 资源
  * 普通 Properties 格式资源 - *.properties
  * Spring Handler 实现类映射文件 - META-INF/spring.handlers
  * Spring Schema 资源映射文件 - META-INF/spring.schemas
* YAML 资源
  * 普通 YAML 配置资源 - *.yaml 或 *.yml



劝退面试题 -  Java 标准资源管理扩展的步骤？

答：

* 简易实现
  * 实现 URLStreamHandler 并放置在 sun.net.www.protocol.${protocol}.Handler 包下
* 自定义实现
  * 实现 URLStreamHandler
  * 添加 -Djava.protocol.handler.pkgs 启动参数，指向 URLStreamHandler 实现类的包下
* 高级实现
  * 实现 URLStreamHandlerFactory 并传递到 URL 之中



ClassLoader#getSystemResources或者getResources

# 第十二章：Spring 国际化（i18n）

## 01 Spring 国际化使用场景

* 普通国际化文案
* Bean Validation 校验国际化文案
* Web 站点页面渲染
* Web MVC 错误消息提示



## 02 Spring 国际化接口： MessageSource 不是技术的创造者，只是技术的搬运工？

* 核心接口
  * org.springframework.context.MessageSource



* 主要概念
  * 文案模板编码（code）
  * 文案模板参数（args）
  * 区域（Locale）



## 03 层次性 MessageSource：双亲委派不是 ClassLoader 的专利吗？

* Spring 层次性接口回顾
  * org.springframework.beans.factory.HierarchicalBeanFactory
  * org.springframework.context.ApplicationContext
  * org.springframework.beans.factory.config.BeanDefinition



* Spring 层次性国际化接口
  * org.springframework.context.HierarchicalMessageSource



## 04 Java 国际化标准实现：ResourceBundle 潜规则多？

* 核心接口
  * 抽象实现 - java.util.ResourceBundle
  * Properties 资源实现 - java.util.PropertyResourceBundle
  * 例举实现 - java.util.ListResourceBundle



* ResourceBundle 核心特性
  * Key-Value 设计
  * 层次性设计
  *  缓存设计
  * 字符编码控制 - java.util.ResourceBundle.Control（@since 1.6）
  * Control SPI 扩展 - java.util.spi.ResourceBundleControlProvider（@since 1.8）



## 05 Java 文本格式化：MessageFormat 脱离 Spring 场景，能力更强大？

* 核心接口
  * java.text.MessageFormat



* 基本用法
  * 设置消息格式模式- new MessageFormat(...)
  * 格式化 - format(new Object[]{...})



* 消息格式模式
  * 格式元素：{ArgumentIndex (,FormatType,(FormatStyle))}
  * FormatType：消息格式类型，可选项，每种类型在 number、date、time 和 choice 类型选其一
  * FormatStyle：消息格式风格，可选项，包括：short、medium、long、full、integer、currency、percent



* 高级特性
  * 重置消息格式模式
  * 重置 java.util.Locale
  * 重置 java.text.Format



## 06 MessageSource 开箱即用实现：ResourceBundle+MessageFormat 组合拳？

* 基于 ResourceBundle + MessageFormat 组合 MessageSource 实现
  * org.springframework.context.support.ResourceBundleMessageSource

getMessage

code  --- key



> 无法重置 format ， locale，使用内建的 MessageFormat 

* 可重载 Properties + MessageFormat 组合 MessageSource 实现

  - org.springframework.context.support.ReloadableResourceBundleMessageSource

    （带ResourceBundle命名有问题 ）



文件不变，lastmodified不一定存在，读取远程 db的实现会更好

## 07 MessageSource 内建依赖：到底“我”是谁？

* MessageSource 內建 Bean 可能来源
  * 预注册 Bean 名称为：“messageSource”，类型为：MessageSource Bean
  * 默认內建实现 - DelegatingMessageSource
    * 层次性查找 MessageSource 对象



## 08 课外资料：SpringBoot 为什么要新建 MessageSource Bean?

* Spring Boot 为什么要新建 MessageSource Bean？
  * AbstractApplicationContext 的实现决定了 MessageSource 內建实现
  * Spring Boot 通过外部化配置简化 MessageSource Bean 构建
  * Spring Boot 基于 Bean Validation 校验非常普遍

basename

## 09 面试题精选

沙雕面试题 - Spring 国际化接口有哪些？

答：
• 核心接口 - MessageSource
• 层次性接口 - org.springframework.context.HierarchicalMessageSource



996 面试题 - Spring 有哪些 MessageSource 內建实现？

答：
• org.springframework.context.support.ResourceBundleMessageSource
• org.springframework.context.support.ReloadableResourceBundleMessageSource
• org.springframework.context.support.StaticMessageSource
• org.springframework.context.support.DelegatingMessageSource



劝退面试题 - 如何实现配置自动更新 MessageSource？

答：
• 主要技术
• Java NIO 2：java.nio.file.WatchService
• Java Concurrency : java.util.concurrent.ExecutorService
• Spring：org.springframework.context.support.AbstractMessageSource

KQueue



# 第十三章：Spring 校验（Validation）

## 01 Spring 校验使用场景：为什么 Validator 并不只是 Bean 的校验？

* Spring 常规校验（Validator）
* Spring 数据绑定（DataBinder）
* Spring Web 参数绑定（WebDataBinder）
* Spring Web MVC / Spring WebFlux 处理方法参数校验



## 02 Validator 接口设计：画虎不成反类犬？

职责不单一

* 接口职责
  * Spring 内部校验器接口，通过编程的方式校验目标对象



* 核心方法
  * supports(Class)：校验目标类能否校验
  *  validate(Object,Errors)：校验目标对象，并将校验失败的内容输出至 Errors 对象



* 配套组件
  * 错误收集器：org.springframework.validation.Errors
  * Validator 工具类：org.springframework.validation.ValidationUtils



## 03 Errors 接口设计：复杂得没有办法理解？

- 接口职责
  -  数据绑定和校验错误收集接口，与 Java Bean 和其属性有强关联性



- 核心方法
  - reject 方法（重载）：收集错误文案
  -  rejectValue 方法（重载）：收集对象字段中的错误文案



- 配套组件
  -  Java Bean 错误描述：org.springframework.validation.ObjectError
  -  Java Bean 属性错误描述：org.springframework.validation.FieldError



## 04 Errors 文案来源：Spring 国际化充当临时工？

*  Errors 文案生成步骤
  * 选择 Errors 实现（如：org.springframework.validation.BeanPropertyBindingResult）
  * 调用 reject 或 rejectValue 方法
  * 获取 Errors 对象中 ObjectError 或 FieldError
  * 将 ObjectError 或 FieldError 中的 code 和 args，关联 MessageSource 实现（如：
    ResourceBundleMessageSource）



## 05 自定义 Validator：为什么说 Validator 容易实现，却难以维护？

* 实现 org.springframework.validation.Validator 接口
  * 实现 supports 方法
  * 实现 validate 方法
    * 通过 Errors 对象收集错误
      * ObjectError：对象（Bean）错误：
      * FieldError：对象（Bean）属性（Property）错误
    * 通过 ObjectError 和 FieldError 关联 MessageSource 实现获取最终文案

Errors 的选型问题

## 06 Validator 的救赎：如果没有 Bean Validation, Validator 将会在哪里吗？

* Bean Validation 与 Validator 适配
  * 核心组件 - org.springframework.validation.beanvalidation.LocalValidatorFactoryBean
  * 依赖 Bean Validation - JSR-303 or JSR-349 provider
  * Bean 方法参数校验 - org.springframework.validation.beanvalidation.MethodValidationPostProcessor

推荐 Bean Validation 扩展



## 07 面试题精选

沙雕面试题 - Spring 校验接口是哪个？

答：org.springframework.validation.Validator



996 面试题 - Spring 有哪些校验核心组件？

答：
• 检验器：org.springframework.validation.Validator
• 错误收集器：org.springframework.validation.Errors
• Java Bean 错误描述：org.springframework.validation.ObjectError
• Java Bean 属性错误描述：org.springframework.validation.FieldError
• Bean Validation 适配：org.springframework.validation.beanvalidation.LocalValidatorFactoryBean



劝退面试题 - 请通过示例演示 Spring Bean 的校验？



# 第十四章：Spring 数据绑定（Data Binding）

## 01 Spring 数据绑定使用场景：为什么官方文档描述一笔带过？

* Spring BeanDefinition 到 Bean 实例创建
  * xml 配置 BeanDefinition 
  * 硬编码 BeanDefinition 
  * 注解 BeanDefinition 
* Spring 数据绑定（DataBinder）
* Spring Web 参数绑定（WebDataBinder）



## 02 Spring 数据绑定组件：DataBinder

MutablePropertyValues

* 标准组件
  * org.springframework.validation.DataBinder

* Web 组件
  * org.springframework.web.bind.WebDataBinder
  * org.springframework.web.bind.ServletRequestDataBinder
  * org.springframework.web.bind.support.WebRequestDataBinder
  * org.springframework.web.bind.support.WebExchangeDataBinder（since 5.0）



* DataBinder 核心属性

| 属性                 | 说明                           |
| -------------------- | ------------------------------ |
| target               | 关联目标 Bean                  |
| objectName           | 目标 Bean名称                  |
| bindingResult        | 属性绑定结果                   |
| typeConverter        | 类型转换器                     |
| conversionService    | 类型转换服务                   |
| messageCodesResolver | 校验错误文案 Code 处理器       |
| validators           | 关联的 Bean Validator 实例集合 |



* DataBinder 绑定方法
  * bind(PropertyValues)：将 PropertyValues Key-Value 内容映射到关联 Bean（target）中的属性上

    假设 PropertyValues 中包含“name = 小马哥”的键值对，同时 Bean 对象 User 中存在 name
    属性，当 bind 方法执行时，User 对象中的 name 属性值将被绑定为“小马哥”。



## 03 DataBinder 绑定元数据：PropertyValues 不是 Spring Bean 属性元信息吗？

| 参数名称            | 说明                               |
| ------------------- | ---------------------------------- |
| ignoreUnknownFields | 是否忽略未知字段，默认值：true     |
| ignoreInvalidFields | 是否忽略非法字段，默认值：false    |
| autoGrowNestedPaths | 是否自动增加嵌套路径，默认值：true |
| allowedFields       | 绑定字段白名单                     |
| disallowedFields    | 绑定字段黑名单                     |
| requiredFields      | 必须绑定字段                       |



## 04 DataBinder 绑定控制参数：ignoreUnknownFields 和 ignoreInvalidFields 有什么作用？

* DataBinder 绑定特殊场景分析

  * 当 PropertyValues 中包含名称 x 的 PropertyValue，目标对象 B 不存在 x 属性，当 bind 方法执行时，
    会发生什么？

  * 当 PropertyValues 中包含名称 x 的 PropertyValue，目标对象 B 中存在 x 属性，当 bind 方法执行时，
    如何避免 B 属性 x 不被绑定？

  * 当 PropertyValues 中包含名称 x.y 的 PropertyValue，目标对象 B 中存在 x 属性（嵌套 y 属性），当
    bind 方法执行时，会发生什么？



## 05 Spring 底层 JavaBeans 替换实现：BeanWrapper 源于JavaBeans 而高于 JavaBeans ?

* BeanWrapper
  * Spring 底层 JavaBeans 基础设施的中心化接口
  * 通常不会直接使用，间接用于 BeanFactory 和 DataBinder
  * 提供标准 JavaBeans 分析和操作，能够单独或批量存储 Java Bean 的属性（properties）
  * 支持嵌套属性路径（nested path）
  * 实现类 org.springframework.beans.BeanWrapperImpl



## 06 BeanWrapper 的使用场景：Spring 数据绑定只是副业？

* JavaBeans 核心实现 - java.beans.BeanInfo
  * 属性（Property）
    * java.beans.PropertyEditor
  * 方法（Method）
  * 事件（Event）
  * 表达式（Expression）



* Spring 替代实现 - org.springframework.beans.BeanWrapper
  * 属性（Property）
    * java.beans.PropertyEditor
  * 嵌套属性路径（nested path）



## 07 课外资料：标准 JavaBeans 是如何操作属性的？

| API                           | 说明                     |
| ----------------------------- | ------------------------ |
| java.beans.Introspector       | JavaBeans 内省 API       |
| java.beans.BeanInfo           | java.beans.BeanInfo      |
| java.beans.BeanDescriptor     | JavaBeans 信息描述符     |
| java.beans.PropertyDescriptor | JavaBeans 属性描述符     |
| java.beans.MethodDescriptor   | JavaBeans 方法描述符     |
| java.beans.EventSetDescriptor | JavaBeans 事件集合描述符 |

Introspector 反射的上层 api 

反射的整合

软弱引用

## 08 DataBinder 数据校验：又见 Validator

* DataBinder 与 BeanWrapper
  * bind 方法生成 BeanPropertyBindingResult
  * BeanPropertyBindingResult 关联 BeanWrapper



## 09 面试题精选

沙雕面试题 - Spring 数据绑定 API 是什么？

答：org.springframework.validation.DataBinder



996 面试题 - BeanWrapper 与 JavaBeans 之间关系是？

答：Spring 底层 JavaBeans 基础设施的中心化接口



劝退面试题 - DataBinder 是怎么完成属性类型转换的？



# 第十五章：Spring 类型转换（Type Conversion）

## 01 Spring 类型转换的实现：Spring 提供了哪几种类型转换的实现？

* 基于 JavaBeans 接口的类型转换实现
  * 基于 java.beans.PropertyEditor 接口扩展



* Spring 3.0+ 通用类型转换实现



## 02 使用场景：Spring 类型转换各自的使用场景以及发展脉络是怎样的？

| 场景               | 基于 JavaBeans 接口的类型转换实现 | Spring 3.0+ 通用类型转换实现 |
| ------------------ | --------------------------------- | ---------------------------- |
| 数据绑定           | YES                               | YES                          |
| BeanWrapper        | YES                               | YES                          |
| Bean 属性类型装换  | YES                               | YES                          |
| 外部化属性类型转换 | NO                                | YES                          |



## 03 基于 JavaBeans 接口的类型转换：Spring 是如何扩展 PropertyEditor 接口实现类型转换的？

* 核心职责
  * 将 String 类型的内容转化为目标类型的对象



* 扩展原理
  * Spring 框架将文本内容传递到 PropertyEditor 实现的 setAsText(String) 方法
  * PropertyEditor#setAsText(String) 方法实现将 String 类型转化为目标类型的对象
  * 将目标类型的对象传入 PropertyEditor#setValue(Object) 方法
  * PropertyEditor#setValue(Object) 方法实现需要临时存储传入对象
  * Spring 框架将通过 PropertyEditor#getValue() 获取类型转换后的对象



## 04 Spring 内建 PropertyEditor 扩展：哪些常见类型被 Spring 内建 PropertyEditor 实现？

* 內建扩展（org.springframework.beans.propertyeditors 包下）

| 转换场景            | 实现类                                                       |
| ------------------- | ------------------------------------------------------------ |
| String -> Byte 数组 | org.springframework.beans.propertyeditors.ByteArrayPropertyEditor |
| String -> Char      | org.springframework.beans.propertyeditors.CharacterEditor    |
| String -> Char 数组 | org.springframework.beans.propertyeditors.CharArrayPropertyEditor |
| String -> Charset   | org.springframework.beans.propertyeditors.CharsetEditor      |
| String -> Class     | org.springframework.beans.propertyeditors.ClassEditor        |
| String -> Currency  | org.springframework.beans.propertyeditors.CurrencyEditor     |
| ...                 | ...                                                          |



## 05 自定义 PropertyEditor 扩展：不尝试怎么知道它好不好用？

* 扩展模式
  * 扩展 java.beans.PropertyEditorSupport 类



* 实现 org.springframework.beans.PropertyEditorRegistrar
  * 实现 registerCustomEditors(org.springframework.beans.PropertyEditorRegistry) 方法
  * 将 PropertyEditorRegistrar 实现注册为 Spring Bean



* 向 org.springframework.beans.PropertyEditorRegistry 注册自定义 PropertyEditor 实现
  * 通用类型实现 registerCustomEditor(Class<?>, PropertyEditor)
  * Java Bean 属性类型实现：registerCustomEditor(Class<?>, String, PropertyEditor)



## 06 SpringPropertyEditor 的设计缺陷：为什么基于 PropertyEditor 扩展并不适合作为类型转换？

* 违反职责单一原则
  * java.beans.PropertyEditor 接口职责太多，除了类型转换，还包括 Java Beans 事件和 Java GUI 交互



* java.beans.PropertyEditor 实现类型局限
  * 来源类型只能为 java.lang.String 类型



* java.beans.PropertyEditor 实现缺少类型安全
  * 除了实现类命名可以表达语义，实现类无法感知目标转换类型



## 07 Spring 3 通用类型转换接口：为什么 Converter 接口设计比 PropertyEditor 更合理？

支持泛型，更抽象的方法

* 类型转换接口 - org.springframework.core.convert.converter.Converter<S,T>
  * 泛型参数 S：来源类型，参数 T：目标类型
  * 核心方法：T convert(S)



* 通用类型转换接口 - org.springframework.core.convert.converter.GenericConverter

  * 核心方法：convert(Object,TypeDescriptor,TypeDescriptor)
  * 配对类型：org.springframework.core.convert.converter.GenericConverter.ConvertiblePair
  * 类型描述：org.springframework.core.convert.TypeDescriptor


## 08 Spring 内建类型转换器：Spring 的内建类型转换器到底有多丰富？

| 转换场景             | 实现类所在包名（package）                    |
| -------------------- | -------------------------------------------- |
| 日期/时间相关        | org.springframework.format.datetime          |
| Java 8 日期/时间相关 | org.springframework.format.datetime.standard |
| 通用实现             | org.springframework.core.convert.support     |



## 09 Converter 接口的局限性：哪种类型转换场景 Converter 无法满足？有什么应对之策？

IntegerToEnum

* 局限一：缺少 Source Type 和 Target Type 前置判断
  * 应对：增加 org.springframework.core.convert.converter.ConditionalConverter 实现



* 局限二：仅能转换单一的 Source Type 和 Target Type
  * 应对：使用 org.springframework.core.convert.converter.GenericConverter 代替

不适用复合类型（array， collection， map），子参数问题



## 10 GenericConverter 接口：为什么 GenericConverter 比 Converter 更通用？

* org.springframework.core.convert.converter.GenericConverter

| 核心要素 | 说明                                                         |
| -------- | ------------------------------------------------------------ |
| 使用场景 | 用于“复合”类型转换场景，比如 Collection、Map、数组等         |
| 转换范围 | Set<ConvertiblePair> getConvertibleTypes()                   |
| 配对类型 | org.springframework.core.convert.converter.GenericConverter.ConvertiblePair |
| 转换方法 | convert(Object,TypeDescriptor,TypeDescriptor)                |
| 类型描述 | org.springframework.core.convert.TypeDescriptor              |



## 11 优化 GenericConverter 接口：为什么 GenericConverter 需要补充条件判断？

* GenericConverter 局限性
  * 缺少 Source Type 和 Target Type 前置判断
  * 单一类型转换实现复杂



* GenericConverter 优化接口 - ConditionalGenericConverter
  * 复合类型转换：org.springframework.core.convert.converter.GenericConverter
  * 类型条件判断：org.springframework.core.convert.converter.ConditionalConverter



## 12 扩展 Spring 类型转换器：为什么最终注册的都是 ConditionalGenericConverter?

* 实现转换器接口
  * org.springframework.core.convert.converter.Converter
  * org.springframework.core.convert.converter.ConverterFactory
  * org.springframework.core.convert.converter.GenericConverter



* 注册转换器实现
  * 通过 ConversionServiceFactoryBean  Spring Bean（`conversionService` spring bean固定名称）
  * 通过 org.springframework.core.convert.ConversionService API

typeconverter





## 13 统一类型转换服务：ConversionService 足够通用吗？

* org.springframework.core.convert.ConversionService

| 实现类型                           | 说明                                                         |
| ---------------------------------- | ------------------------------------------------------------ |
| GenericConversionService           | 通用 ConversionService 模板实现，不内置转化器实现            |
| DefaultConversionService           | 基础 ConversionService 实现，内置常用转化器实现              |
| FormattingConversionService        | 通用 Formatter + GenericConversionService 实现，不内置转化器和 Formatter 实现 |
| DefaultFormattingConversionService | DefaultConversionService + 格式化 实现（如：JSR-354 Money &<br/>Currency, JSR-310 Date-Time） |


## 14 ConversionService 作为依赖-能够同时作为依赖查找和依赖注入的来源吗？

* 类型转换器底层接口 - org.springframework.beans.TypeConverter
  * 起始版本：Spring 2.0
  * 核心方法 - convertIfNecessary 重载方法
  * 抽象实现 - org.springframework.beans.TypeConverterSupport
  * 简单实现 - org.springframework.beans.SimpleTypeConverter



* 类型转换器底层抽象实现 - org.springframework.beans.TypeConverterSupport
  * 实现接口 - org.springframework.beans.TypeConverter
  * 扩展实现 - org.springframework.beans.PropertyEditorRegistrySupport
  * 委派实现 - org.springframework.beans.TypeConverterDelegate



* 类型转换器底层委派实现 - org.springframework.beans.TypeConverterDelegate

  * 构造来源 - org.springframework.beans.AbstractNestablePropertyAccessor 实现

    * org.springframework.beans.BeanWrapperImpl

  * 依赖 - java.beans.PropertyEditor 实现

    * 默认內建实现 - PropertyEditorRegistrySupport#registerDefaultEditors

  * 可选依赖 - org.springframework.core.convert.ConversionService 实现



## 15 面试题精选

沙雕面试题 - Spring 类型转换实现有哪些？

答：
1. 基于 JavaBeans PropertyEditor 接口实现
2. Spring 3.0+ 通用类型转换实现



996 面试题 - Spring 类型转换器接口有哪些？

答：

* 类型转换接口 - org.springframework.core.convert.converter.Converter
* 通用类型转换接口 - org.springframework.core.convert.converter.GenericConverter
* 类型条件接口 - org.springframework.core.convert.converter.ConditionalConverter
* 综合类型转换接口 -
  org.springframework.core.convert.converter.ConditionalGenericConverter



劝退面试题 - TypeDescriptor 是如何处理泛型？



# 第十六章：Spring 泛型处理（Generic Resolution）

## 01 Java 泛型基础：泛型参数信息在擦写后还会存在吗？

泛型：声明中具有一个或者多个类型参数的类或者接口  ，如 List\<E>

每种泛型定义一组参数化的类型（parameterized type），格式为：先是类或者接口的名称，接着用尖括号（<>）把对应于泛型形式类型参数的实际类型参数列表括起来

参数化类型：List\<String> 是一个参数化的类型，类型参数 String，形式类型参数 E

原生态类型：List



* 泛型类型
  * 泛型类型是在类型上参数化的泛型类或接口



* 泛型使用场景
  * 编译时强类型检查
  * 避免类型强转
  * 实现通用算法



* 泛型类型擦写
  * 泛型被引入到 Java 语言中，以便在编译时提供更严格的类型检查并支持泛型编程。类型擦除确保不会为参数化类型创建新类；因此，泛型不会产生运行时开销。为了实现泛型，编译器将类型擦除应用于：
    * 将泛型类型中的所有类型参数替换为其边界，如果类型参数是无边界的，则将其替换为
      “Object”。因此，生成的字节码只包含普通类、接口和方法。
    * 必要时插入类型转换以保持类型安全。
    * 生成桥方法以保留扩展泛型类型中的多态性。



兼容以前版本



## 02 Java 5 类型接口-Type： Java 类型到底是 Type 还是 Class ?

* Java 5 类型接口 - java.lang.reflect.Type

| 派生类或接口                        | 说明                                  |
| ----------------------------------- | ------------------------------------- |
| java.lang.Class                     | Java 类 API，如 java.lang.String      |
| java.lang.reflect.GenericArrayType  | 泛型数组类型                          |
| java.lang.reflect.ParameterizedType | 泛型参数类型                          |
| java.lang.reflect.TypeVariable      | 泛型类型变量，如 Collection<E> 中的 E |
| java.lang.reflect.WildcardType      | 泛型通配类型                          |



- Java 泛型反射 API

| 类型                             | API                                    |
| -------------------------------- | -------------------------------------- |
| 泛型信息（Generics Info）        | java.lang.Class#getGenericInfo()       |
| 泛型参数（Parameters）           | java.lang.reflect.ParameterizedType    |
| 泛型父类（Super Classes）        | java.lang.Class#getGenericSuperclass() |
| 泛型接口（Interfaces）           | java.lang.Class#getGenericInterfaces() |
| 泛型声明（Generics Declaration） | java.lang.reflect.GenericDeclaration   |

泛型父类 带泛型的父类

泛型接口 带泛型的接口

## 03 Spring 泛型类型辅助类：GenericTypeResolver

* 核心 API - org.springframework.core.GenericTypeResolver

  * 版本支持：[2.5.2 , )
  * 处理类型相关（Type）相关方法
    * resolveReturnType
    * resolveType
  * 处理泛型参数类型（ParameterizedType）相关方法
    * resolveReturnTypeArgument
    * resolveTypeArgument
    * resolveTypeArguments
  * 处理泛型类型变量（TypeVariable）相关方法
    * getTypeVariableMap



## 04 Spring 泛型集合类型辅助类：GenericCollectionTypeResolver

* 核心 API - org.springframework.core.GenericCollectionTypeResolver

  * 版本支持：[2.0 , 4.3]

  * 替换实现：org.springframework.core.ResolvableType

  * 处理 Collection 相关

    * getCollection*Type

  * 处理 Map 相关

    * getMapKey*Type
    * *getMapValue*Type

泛型参数类型具体化



## 05 Spring 方法参数封装-MethodParameter：不仅仅是方法参数

* 核心 API - org.springframework.core.MethodParameter
  * 起始版本：[2.0 , )
  * 元信息
    * 关联的方法 - Method
    * 关联的构造器 - Constructor
    * 构造器或方法参数索引 - parameterIndex
    * 构造器或方法参数类型 - parameterType
    * 构造器或方法参数泛型类型 - genericParameterType
    * 构造器或方法参数参数名称 - parameterName
    * 所在的类 - containingClass



## 06 Spring 4.2 泛型优化实现-ResolvableType

* 核心 API - org.springframework.core.ResolvableType
  * 起始版本：[4.0 , )
  * 扮演角色：GenericTypeResolver 和 GenericCollectionTypeResolver 替代者
  * 工厂方法：for* 方法
  * 转换方法：as* 方法
  * 处理方法：resolve* 方法



## 07 ResolvableType 的局限性：形式比人强？

* 局限一：ResolvableType 无法处理泛型擦写
* 局限二：ResolvableType 无法处理非具体化的 ParameterizedType



## 08 面试题精选

沙雕面试题 - Java 泛型擦写发生在编译时还是运行时？

答： 运行时



996 面试题 - 请介绍 Java 5 Type 类型的派生类或接口？

答：

* java.lang.Class
* java.lang.reflect.GenericArrayType
* java.lang.reflect.ParameterizedType
* java.lang.reflect.TypeVariable
* java.lang.reflect.WildcardType



劝退面试题 - 请说明 ResolvableType 的设计优势？

答：

* 简化 Java 5 Type API 开发，屏蔽复杂 API 的运用，如 ParameterizedType
* 不变性设计（Immutability）
* Fluent API 设计（Builder 模式），链式（流式）编程



# 第十七章：Spring 事件（Events）

## 01 Java 事件/监听器编程模型：为什么 Java 中没有提供标淮实现？

* 设计模式 - 观察者模式扩展
  * 可观者对象（消息发送者）- java.util.Observable
  * 观察者 - java.util.Observer



* 标准化接口
  * 事件对象 - java.util.EvenObject
  * 事件监听器 - java.util.EvenListener 



## 02 面向接口的事件/监听器设计模式：单事件监听和多事件监听怎么选？





## 03 面向注解的事件/监听器设计模式：便利也会带来伤害？





## 04 Spring 标准事件-AppllcatlonEvent：为什么不用 EventObject ?





## 05 基于接口的 Spring 事件监听器：ApplicationListener 为什么选择单事件监听模式？





## 06 基于注解的 Spring事件监听器：@EventListener 有哪些潜在规则？





## 07 注册 Spring ApplicationListener：直接注册和间接注册有哪些差异？





## 08 Spring 事件发布器：Spring 4.2 给 ApplicationEventPublisher 带来哪些变化？





## 09 Spring 层次性上下文事件传播：这是一个 Feature 还是一个 Bug?





## 10 Spring 内建事件（Built-in Events）：为什么 ContextStartedEvent 和 ContextStoppedEvent 是鸡助事件？





## 11 Spring4.2 Payload事件：为什么说 PayloadApplicationEvent 并非一个良好的设计？





## 12 自定义 Spring 事件：自定义事件业务用得上吗？





## 13 依赖查找 ApplicationEventPublisher：ApplicationEventPublisher 从何而来？





## 14 依赖注入 ApplicatlonEventPublisher：事件推送还会引起 Bug?





## 15 ApplicationEventPublisher 底层实现：ApplicationEventMulticaster 也是 Java Observable 的延伸？



## 16 同步和异步 Spring 事件广播：Spring 对 J.U.C Executor接口的理解不够？





## 17 Spring 4.1 事件异常处理：ErrorHandler 使用有怎样旳限制？





## 18 Spring 事件/监听器实现原理：面向接口和注解旳事件/监听器实现有区别吗？





## 19 课外资料：Spring Boot 和 Spring Cloud 事件也是 Spring 事件？





## 20 面试题精选



# 第十八章：Spring 注解（Annotations）

## 01 Spring 注解驱动编程发展历程



## 02 Spring 核心注解场景分类



## 03 Spring 注解编程模型



## 04 Spring 元注解(Meta-Annotations)



## 05 Spring 模式注解(Stereotype Annotations)

```
ClassPathScanningCandidateComponentProvider
includeFilters
AnnotationTypeFilter
```

## 06 Spring 组合注解(Composed Annotations)【关键内容】



## 07 Spring 注解属性别名和覆盖(Attribute Aliases and Overrides)

```
isPresent 到底干了什么工作

AnnotationUtils#synthesizeAnnotation 作用：来获取一个动态代理注解（相当于调用者传进来的注解会被代理掉），该方法是别名注解@AliasFor的核心原理
AliasDescriptor

AnnotatedElementUtils：在AnnotatedElement finding annotations, meta-annotations, and repeatable annotations(@since 4.0)
```



## 08 Spring @Enable 模块驱动

@Enablexxx 都会标注 @import(xxx相关配置类.class)



## 09 Spring 条件装配

@Conditionxxx

```
ConditionEvaluator
ConditionContext
```

## 10 课外资料：Spring Boot 和 Spring Cloud 是怎样在 Spring注解内核上扩展旳？



## 11 面试题精选

```
AnnotationUtils
TypeMappedAnnotation
synthesizeAnnotation
```



# 第十九章：Spring Environment 抽象（Environment Abstraction）

## 01 理解 Spring  “外部化配置”



## 02 Spring Environment 接口使用场景



## 03 Environment 占位符处理



## 04 理解条件配置 Spring Profiles



## 05 Spring 4 重构@Profile



## 06 依赖注入 Environment



## 07 依赖注入 @Value



## 08 Spring 类型转换在 Environment 中的运用



## 09 Spring 类型转换在 @Value 中的运用



## 10 Spring 外部化配置属性源 PropertySource



## 11 Spring 内建的外部化配性源



## 12 基于 @PropertySource 扩展 Spring 外部化配置属性源



##13 基于API 扩展 Spring 外部化配置属性源：Propertysource



## 14 课外资料：Spring4.1 测试外部化配置属性源 @TestPropertySource



## 15 面试题精选



# 第二十章：Spring 应用上下文生命周期（Container Lifecycle）

## 01 Spring 应用上下文启动淮备阶段

```
prepareRefresh
// Initialize any placeholder property sources in the context environment.
->initPropertySources
// Validate that all properties marked as required are resolvable:
// see ConfigurablePropertyResolver#setRequiredProperties
->getEnvironment().validateRequiredProperties();
```

## 02 loC 底层容器（BeanFactory）创建阶段

```
obtainFreshBeanFactory
->
```

## 03 loC 底层容器（BeanFactory）初始化阶段

```
prepareBeanFactory
```

## 04 loC 底层容器（BeanFactory）自定义阶段

```
postProcessBeanFactory
invokeBeanFactoryPostProcessors
```

## 05 loC 底层容器（BeanFactory）注册 BeanProcessor 阶段

```
registerBeanPostProcessors
```

## 06 初始化内建 Bean：MessageSource



## 07 初始化内建 Bean：Spring 事件广播器



## 08 Spring 应用上下文刷新阶段

```
onRefresh
```

## 09 Spring 事件监听器注册阶段

```
registerListeners
```

## 10 loC 底层容器（BeanFactory）非延迟 Bean 初始化阶段

```
finishBeanFactoryInitialization
```

## 11 Spring 应用上下启动完成阶段

```
protected void finishRefresh() {
		// Clear context-level resource caches (such as ASM metadata from scanning).
		clearResourceCaches();

		// Initialize lifecycle processor for this context.
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		getLifecycleProcessor().onRefresh();

		// Publish the final event.
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		LiveBeansView.registerApplicationContext(this);
	}
```

## 12 Spring 生命周期 Bean 开始阶段



## 13 Spring 生命周期 Bean 停止阶段



## 14 Spring 应用上下文关闭阶段



## 15 面试题精选



## 16 结束语





# 工具类

## `AnnotationUtils`

## `AnnotatedElementUtils`：在`AnnotatedElement` finding annotations, meta-annotations, and repeatable annotations(`@since 4.0`)

## AnnotationBeanUtils：拷贝注解值到指定的Bean中

## AnnotationConfigUtils：（重要），和Config配置类有关的注解工具类

## BeanFactoryAnnotationUtils：提供与注解相关的查找Bean的方法（比如`@Qualifier`）



`ConfigurationClassUtils`：Spring内部处理`@Configuration`的工具类









# SpringMVC



### 初始化

```
HttpServletBean#init
```

#### ServletContainerInitializer

```
ServiceLoader#load(Class)
```

```
@HandlesTypes(WebApplicationInitializer.class)
```

#### SpringServletContainerInitializer



### 请求处理



```
interceptors + handler
HandlerExecutionChain#applyPreHandle    请求拦截，权限
handle
HandlerExecutionChain#applyPostHandle
```

