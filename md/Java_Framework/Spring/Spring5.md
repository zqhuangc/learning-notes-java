HTTP/2

TestNG

Mono  ---- messageConverter

WebFlux



## 测试

### 单元测试



+ [Junit5](https://junit.org/junit5/docs/current/user-guide/)
  - JUnit Platform：启动测试框架的基础
  - JUnit Jupiter：测试框架的编程模型和扩展模型
    - @BeforeEach、@AfterEach
    - @BeforeAll、@AfterAll
    - @RepeatedTest
    - ParameterizedTest
  - JUnit Vintage：JUnit3、JUnit4 测试引擎
    - JUnit3：extends TestCase、setUp、tearDown
    - JUnit4 ：@Test、@Before、@After、@BeforeClass、@AfterClass



+ [Mock 对象](https://docs.spring.io/spring/docs/5.1.8.RELEASE/spring-framework-reference/testing.html#mock-objects)
  - Environment
    - MockEnvironment#getProperty
    - StandardEnvironment
  - Servlet API
    - 静态：MockHttpServletRequest
    - 动态代理：Mockito.mock(HttpServletRequest.class)
      - when(...).xxx
  - JNDI
  - Portlet API（removed）

## 题外话（spring 应用了许多 java 中的规范）

ioc 控制数据来源（规范）

依赖注入，依赖查找（实现）

Spring：ApplicationContext#getBean

JNDI：Context#lookup



### 集成测试

如何注入 Bean？

+ Spring TestContext Framework
  - @ContextConfiguration（since 2.5）
  - @ExtendWith(SpringExtension.class)（since 5.0）
  - @SpringJUnitConfig *(only supported on JUnit Jupiter)*
    - @ContextConfiguration + @ExtendWith(SpringExtension.class)

TestContext 关联 ApplicationContext 



+ Spring MVC Test Framework
  - @SpringJUnitWebConfig
  - @RestTemplate



+ Mockito 整合