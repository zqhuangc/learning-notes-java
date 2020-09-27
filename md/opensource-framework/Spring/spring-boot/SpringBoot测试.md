# Spring 测试

行覆盖

## 单元测试
+  模拟对象
  - Environment
  - JNDI
  - Servlet API
  - Portlet API

+ 支持类
  - 通用工具类
    - 反射： ReflectionTestUtils
    - AOP：AopTestUtils 

+ Spring WebMVC



## 集成测试
+ Spring TestContext Framework
  - 上下文管理
  - 依赖注入
  - 事务管理
  - JDBC 测试支持

+ Spring WebMVC Test Framework
  - 服务端测试
  - HtmlUnit 集成
  - 客户端测试



> 从Spring Framework 5.0开始，其中的模拟对象`org.springframework.mock.web`基于Servlet 4.0 API。

ReflectionTestUtils

AopTestUtils

JdbcTestUtils

### 注解（Annotation）

### 上下文管理

* @ContextConfiguration：用于确定如何加载和配置`ApplicationContext`集成测试的类级元数据。具体来说， `@ContextConfiguration`声明应用程序上下文资源`locations`或`classes`用于加载上下文的组件。
* @ContextHierarchy：`@ContextHierarchy`是类级别的注释，用于定义`ApplicationContext`集成测试的实例层次结构 。`@ContextHierarchy`应该用一个或多个`@ContextConfiguration`实例的列表声明，每个实例定义上下文层次结构中的一个级别。
* @WebAppConfiguration：类级别的注释，您可以用来声明 `ApplicationContext`为集成测试加载的应该是`WebApplicationContext`
* @DirtiesContext：
* `@BootstrapWith`：是一个类级别的注释，您可以使用它来配置如何引导Spring TestContext Framework

#### 事务管理

@BeforeTransaction

@AfterTransaction

@Commit

@Rollback



#### 依赖注入

* @TestExecutionListeners

定义了用于配置`TestExecutionListener`应向进行注册的实现的 类级元数据 `TestContextManager`。通常，`@TestExecutionListeners`与结合使用 `@ContextConfiguration`。

默认情况下，`@TestExecutionListeners`支持继承的侦听器

Spring提供了以下`TestExecutionListener`实现，这些实现默认情况下按以下顺序注册：

- `ServletTestExecutionListener`：为配置Servlet API模拟 `WebApplicationContext`。
- `DirtiesContextBeforeModesTestExecutionListener`：处理`@DirtiesContext` “之前”模式的注释。
- `DependencyInjectionTestExecutionListener`：为测试实例提供依赖项注入。
- `DirtiesContextTestExecutionListener`：处理`@DirtiesContext`“之后”模式的注释。
- `TransactionalTestExecutionListener`：提供具有默认回滚语义的事务测试执行。
- `SqlScriptsTestExecutionListener`：运行通过`@Sql` 注释配置的SQL脚本。
- `EventPublishingTestExecutionListener`：将测试执行事件发布到测试 `ApplicationContext`



#### JDBC 测试支持

@Sql：用于注释测试类或测试方法，以配置在集成测试期间针对给定数据库运行的SQL脚本。

@SqlConfig：用于确定如何解析和运行使用`@Sql`注释配置的SQL脚本的元数据

@SqlGroup：一个聚合多个`@Sql`注释的容器注释

@SqlMergeMode:用于注释测试类或测试方法，以配置是否将方法级`@Sql`声明与类级`@Sql`声明合并



#### 配置相关
* @ActiveProfiles：类级别的注释，用于声明在`ApplicationContext`为集成测试加载时应激活哪些bean定义配置文件

* @TestPropertySource：类级别的注释，可用于配置属性文件和内联属性的位置，以将其添加到的集合中 `PropertySources`，以`Environment`进行`ApplicationContext`集成测试。

  @TestPropertySource("/test.properties")

* @DynamicPropertySource：方法级别的注释，可用于注册 要添加到的集合中以用于集成测试的*动态*属性。



## Junit 4 和 Junit 5

SpringRunner：以Spring 容器 执行



* Junit 5

@SpringJUnitConfig = @ExtendWith(SpringExtension.class)（JUnit Jupiter） + @ContextConfiguration



```
org.junit.Test;   // Junit 4
org.junit.jupiter.Test;  // Junit5
```

SmartContextLoader



# Spring Boot 测试

+ 注解（Annotation）
  - @SpringBootTest
    - 配置属性
    - Spring Bean配置
    - Web 环境
    - 自动装配
    - 测试JSON
  - Spring WebMVC
    - Data JPA
    - JDBC
    - RestClient

AssertJ
Mockito

