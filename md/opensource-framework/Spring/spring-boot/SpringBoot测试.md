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



### 注解（Annotation）

### 上下文管理

@ContextConfiguration
@ContextHierarchy
@WebAppConfiguration
@DirtiesContext

#### 事务管理

@BeforeTransaction
@AfterTransaction
@Commit
@Rollback

#### 依赖注入
@TestExecutionListeners

#### JDBC 测试支持 
@Sql
@SqlConfig
@SqlGroup

#### 配置相关
@ActiveProfiles
@TestPropertySource



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

