VO参数

文档的顶层结构如下：

+ configuration 配置
  - properties 属性
  - settings 设置
  - typeAliases 类型别名
  - typeHandlers 类型处理器
    - set 存  get 取    jdbctype  convert with object type
  - objectFactory 对象工厂
  - plugins 插件
  - environments 环境
    - environment 环境变量
      - transactionManager 事务管理器
      - dataSource 数据源
  - databaseIdProvider 数据库厂商标识
  - mappers 映射器
## xml配置
```xml
<configuration>
<!--properties 通用配置文件加载：如数据库信息配置文件-->
<properties resource="org/mybatis/example/config.properties">
  <property name="username" value="dev_user"/>
  <property name="password" value="F2Fa3!33TYyg"/>
</properties>
<!-- 修改MyBatis在运行时的行为方式 -->
<settings>
  <!-- 缓存是否启用 -->
  <setting name="cacheEnabled" value="true"/>
  <!-- 延迟加载是否启用 -->
  <setting name="lazyLoadingEnabled" value="true"/>
  <setting name="multipleResultSetsEnabled" value="true"/>
  <setting name="useColumnLabel" value="true"/>
  <setting name="useGeneratedKeys" value="false"/>
  <setting name="autoMappingBehavior" value="PARTIAL"/>
  <setting name="autoMappingUnknownColumnBehavior" value="WARNING"/>
  <setting name="defaultExecutorType" value="SIMPLE"/>
  <setting name="defaultStatementTimeout" value="25"/>
  <setting name="defaultFetchSize" value="100"/>
  <setting name="safeRowBoundsEnabled" value="false"/>
  <setting name="mapUnderscoreToCamelCase" value="false"/>
  <setting name="localCacheScope" value="SESSION"/>
  <setting name="jdbcTypeForNull" value="OTHER"/>
  <setting name="lazyLoadTriggerMethods"
    value="equals,clone,hashCode,toString"/>
</settings>
<!--typeAliases 设置别名，类型别名只是Java类型的较短名称。它仅与XML配置相关，并且仅用于减少完全限定类名的冗余类型  @Alias-->
<typeAliases>
  <typeAlias alias="Author" type="domain.blog.Author"/>
  <typeAlias alias="Blog" type="domain.blog.Blog"/>
  <typeAlias alias="Comment" type="domain.blog.Comment"/>
  <typeAlias alias="Post" type="domain.blog.Post"/>
  <typeAlias alias="Section" type="domain.blog.Section"/>
  <typeAlias alias="Tag" type="domain.blog.Tag"/>
</typeAliases>

<!-- 
typeHandlers 类型处理器  
每当MyBatis在PreparedStatement上设置参数或从ResultSet中检索值时，都会使用TypeHandler以适合Java类型的方式检索值。

您可以覆盖类型处理程序或创建自己的处理程序来处理不支持或非标准类型。为此，请实现org.apache.ibatis.type.TypeHandler接口 或扩展便捷类org.apache.ibatis.type.BaseTypeHandler， 并可选择将其映射到JDBC类型。
-->

<!--
objectFactory 对象
每次MyBatis创建结果对象的新实例时，它都会使用ObjectFactory实例来执行此操作。默认的ObjectFactory只是使用默认构造函数实例化目标类，或者如果存在参数映射，则使用参数化构造函数。如果要覆盖ObjectFactory的默认行为，可以创建自己的行为。


ObjectFactory接口非常简单。它包含两个create方法，一个用于处理默认构造函数，另一个用于处理参数化构造函数。最后，setProperties方法可用于配置ObjectFactory。在ObjectFactory实例初始化之后，objectFactory元素主体中定义的属性将传递给setProperties方法。
-->

<!-- 
plugins 插件  
MyBatis允许您在映射语句的执行中拦截对某些点的调用。默认情况下，MyBatis允许插件拦截以下方法调用：
执行者（更新，查询，flushStatements，commit，rollback，getTransaction，close，isClosed）
ParameterHandler（getParameterObject，setParameters）
ResultSetHandler（handleResultSets，handleOutputParameters）
StatementHandler（准备，参数化，批处理，更新，查询）
-->

<!--environments 通用环境配置：如数据库相关信息配置-->
<!--要记住一件重要的事情：虽然您可以配置多个环境，但每个SqlSessionFactory实例只能选择一个。-->
<environments default="development">
  <environment id="development">
    <transactionManager type="JDBC">
      <property name="..." value="..."/>
    </transactionManager>
    <dataSource type="POOLED">
      <property name="driver" value="${driver}"/>
      <property name="url" value="${url}"/>
      <property name="username" value="${username}"/>
      <property name="password" value="${password}"/>
    </dataSource>
  </environment>
</environments>

<!--MyBatis能够执行不同的语句，具体取决于您的数据库供应商。多数据库供应商支持基于映射语句databaseId属性。MyBatis将加载所有没有databaseId属性或与当前匹配的databaseId的语句。如果在使用和不使用databaseId的情况下找到相同的语句，则后者将被丢弃。要启用多供应商支持，请将databaseIdProvider添加 到mybatis-config.xml文件-->
<databaseIdProvider type="DB_VENDOR">
  <property name="SQL Server" value="sqlserver"/>
  <property name="DB2" value="db2"/>
  <property name="Oracle" value="oracle" />
</databaseIdProvider>
<!--
mappers 引入mapper
使用类路径 相对资源
使用url 完全限定路径
使用mapper接口 类
将 包中 的所有接口注册为映射器
--> 


</configuration>
```
## 配置加载和方法执行方式
### 加载配置
* SqlSessionFactory创建
```java
String resource = "org/mybatis/example/mybatis-config.xml";
InputStream inputStream = Resources.getResourceAsStream(resource);
SqlSessionFactory sqlSessionFactory =
  new SqlSessionFactoryBuilder().build(inputStream);
```
### 硬编码执行方法
**session.(增删改查方法名)("命名空间.实际方法sql",参数)；**
```java
SqlSession session = sqlSessionFactory.openSession();
try {
  Blog blog = session.selectOne(
    "org.mybatis.example.BlogMapper.selectBlog", 101);
} finally {
  session.close();
}
```

### 硬编码配置
```java
DataSource dataSource = BlogDataSourceFactory.getBlogDataSource();
TransactionFactory transactionFactory =
  new JdbcTransactionFactory();
Environment environment =
  new Environment("development", transactionFactory, dataSource);
//....
Configuration configuration = new Configuration(environment);
// 加载与接口同名xml文件？？？？
configuration.addMapper(BlogMapper.class);
SqlSessionFactory sqlSessionFactory =
  new SqlSessionFactoryBuilder().build(configuration);
```

### 通过接口执行方法
* 使用正确描述给定语句的参数和返回值的接口
```java
SqlSession session = sqlSessionFactory.openSession();
try {
  BlogMapper mapper = session.getMapper(BlogMapper.class);
  Blog blog = mapper.selectBlog(101);
} finally {
  session.close();
}
```

### 小结
* 探索映射的SQL语句
此时您可能想知道SqlSession或Mapper类究竟执行了什么。

在上面的任何一个示例中，语句都可以由XML或Annotations定义。
XML：MyBatis提供的全套功能可以通过使用多年来使MyBatis流行的基于XML的映射语言来实现。如果您以前使用过MyBatis，那么您将很熟悉这个概念，但是对XML映射文档进行了大量改进，这些改进将在以后变得清晰。下面是一个基于XML的映射语句的示例，它将满足上述  
* SqlSession调用。
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
  PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="org.mybatis.example.BlogMapper">
  <select id="selectBlog" resultType="Blog">
    select * from Blog where id = #{id}
  </select>
</mapper>
```

虽然这看起来像这个简单示例的很多开销，但它实际上非常轻。您可以根据需要在单个映射器XML文件中定义任意数量的映射语句，因此您可以从XML标头和doctype声明中获得大量的里程数。文件的其余部分非常自我解释。它在名称空间“org.mybatis.example.BlogMapper”中定义了映射语句“selectBlog”的名称，该名称允许您通过指定“org.mybatis.example.BlogMapper.selectBlog”的**完全限定名称**来调用它(接口的完全限定名称)。
```java
Blog blog = session.selectOne(
  "org.mybatis.example.BlogMapper.selectBlog", 101);
```
注意这与在完全限定的Java类上调用方法有多相似，这是有原因的。此名称可以直接映射到与名称空间同名的Mapper类，并使用与name，parameter和return类型匹配的方法作为映射的select语句。这允许您非常简单地针对Mapper接口调用该方法，如上所示，但是在下面的示例中再次使用它：

```java
BlogMapper mapper = session.getMapper(BlogMapper.class);
Blog blog = mapper.selectBlog(101);
```
第二种方法有很多优点。首先，它不依赖于字符串文字，所以它更安全。其次，如果您的IDE已完成代码，则可以在导航映射的SQL语句时利用它。

* 注：**关于命名空间的注意事项**。

命名空间 在MyBatis的**早期版本**中是可选的，这令人困惑且无益。现在需要命名空间，其目的不仅仅是使用更长的完全限定名称隔离语句。

命名空间启用了界面绑定，如您所见，即使您认为今天不会使用它们，您也应该遵循这里列出的这些做法，以防您改变主意。使用命名空间一次，并将其放在适当的Java包命名空间中将清理代码并提高MyBatis的长期可用性。

名称解析：为了减少键入的数量，MyBatis对所有命名配置元素使用以下名称解析规则，包括语句，结果映射，缓存等。

* 完全限定名称（例如“com.mypackage.MyMapper.selectAllThings”）直接查找并在找到时使用。
* 短名称（例如“selectAllThings”）可用于引用任何明确的条目。但是，如果有两个或更多（例如“com.foo.selectAllThings和com.bar.selectAllThings”），那么您将收到一个错误，报告短名称不明确，因此必须是完全限定的。  

像BlogMapper这样的Mapper类还有一个技巧。它们的映射语句根本不需要与XML映射。相反，他们可以使用Java Annotations。例如，上面的XML可以被删除并替换为：
```java
package org.mybatis.example;
public interface BlogMapper {
  @Select("SELECT * FROM blog WHERE id = #{id}")
  Blog selectBlog(int id);
}

```


对于简单的语句，注释更加清晰，但是，对于更复杂的语句，最好使用XML映射语句。

由您和您的项目团队决定哪个适合您，以及以一致的方式定义映射语句对您来说有多重要。也就是说，你永远不会被锁定在一个单一的方法中。您可以非常轻松地将基于注释的基于映射的语句迁移到XML，反之亦然。

* 范围和生命周期
了解到目前为止我们讨论过的各种范围和生命周期类非常重要。错误地使用它们会导致严重的并发问题。

注意:对象生命周期和依赖注入框架

依赖注入框架可以创建线程安全的事务性SqlSession和映射器，并将它们直接注入到bean中，这样您就可以忘记它们的生命周期。您可能需要查看MyBatis-Spring或MyBatis-Guice子项目，以了解有关将MyBatis与DI框架一起使用的更多信息。

* SqlSessionFactoryBuilder对象
该类可以被实例化，使用和丢弃。一旦创建了SqlSessionFactory，就无需保留它。因此，SqlSessionFactoryBuilder实例的最佳范围是方法范围（即本地方法变量）。您可以重用SqlSessionFactoryBuilder来构建多个SqlSessionFactory实例，但最好不要保留它以确保释放所有XML解析资源以用于更重要的事情。

* SqlSessionFactory

一旦创建，SqlSessionFactory应该在应用程序执行期间存在。应该很少或没有理由处理它或重新创建它。在应用程序运行中不多次重建SqlSessionFactory是最佳做法。这样做应被视为“难闻的气味”。因此，SqlSessionFactory的最佳范围是应用程序范围。这可以通过多种方式实现。最简单的是使用Singleton模式或Static Singleton模式。

* SqlSession

每个线程都应该有自己的SqlSession实例。SqlSession的实例不是共享的，也不是线程安全的。因此，最佳范围是请求或方法范围。永远不要在静态字段甚至类的实例字段中保留对SqlSession实例的引用。永远不要在任何类型的托管范围内保留对SqlSession的引用，例如Servlet框架的HttpSession。如果您正在使用任何类型的Web框架，请考虑SqlSession遵循与HTTP请求类似的范围。换句话说，在收到HTTP请求后，您可以打开SqlSession，然后在返回响应时，您可以关闭它。结束会议非常重要。你应该总是确保它' 在最终区块内关闭。以下是确保关闭SqlSessions的标准模式：
```java
SqlSession session = sqlSessionFactory.openSession();
try {
  // do work
} finally {
  session.close();
}
```
在整个代码中始终如一地使用此模式将确保正确关闭所有数据库资源。

* Mapper 实例

映射器是您创建以绑定到映射语句的接口。映射器接口的实例是从SqlSession获取的。因此，从技术上讲，任何映射器实例的最广泛范围都与请求它们的SqlSession相同。但是，映射器实例的最佳范围是方法范围。也就是说，应该在使用它们的方法中请求它们，然后丢弃它们。它们不需要明确关闭。虽然在整个请求中保留它们并不是一个问题，类似于SqlSession，但您可能会发现在此级别管理太多资源很快就会失控。保持简单，将Mappers保留在方法范围内。
```java
SqlSession session = sqlSessionFactory.openSession();
try {
  BlogMapper mapper = session.getMapper(BlogMapper.class);
  // do work
} finally {
  session.close();
}
```

## setting标签
```
设置	描述	有效值	默认
cacheEnabled	全局启用或禁用此配置下任何映射器中配置的任何高速缓存。	是的| 假	真正
lazyLoadingEnabled	全局启用或禁用延迟加载。启用后，所有关系都将被延迟加载。通过在其上使用fetchType属性，可以为特定关系取代此值。	是的| 假	假
aggressiveLazyLoading	启用后，任何方法调用都将加载对象的所有惰性属性。否则，每个属性都按需加载（另请参阅lazyLoadTriggerMethods）。	是的| 假	false（真于≤3.4.1）
multipleResultSetsEnabled	允许或禁止从单个语句返回多个ResultSet（需要兼容的驱动程序）。	是的| 假	真正
useColumnLabel	使用列标签而不是列名称。在这方面，不同的司机表现不同。请参阅驱动程序文档，或测试两种模式以确定驱动程序的行为方式。	是的| 假	真正
useGeneratedKeys	允许JDBC支持生成的密钥。需要兼容的驱动程序。如果设置为true，此设置会强制使用生成的键，因为某些驱动程序拒绝兼容性但仍然有效（例如Derby）。	是的| 假	假
autoMappingBehavior	指定MyBatis是否以及如何自动将列映射到fields / properties。NONE禁用自动映射。PARTIAL将仅自动映射结果，而不在内部定义嵌套结果映射。FULL将自动映射任何复杂度的结果映射（包含嵌套或其他）。	没有，部分，全部	局部
autoMappingUnknownColumnBehavior	检测自动映射目标的未知列（或未知属性类型）时指定行为。
无：什么都不做
警告：输出警告日志（'org.apache.ibatis.session.AutoMappingUnknownColumnBehavior'的日志级别必须设置为WARN）
FAILING：失败映射（抛出SqlSessionException）
没有，警告，失败	没有
defaultExecutorType	配置默认执行程序。SIMPLE执行者没有什么特别的。REUSE执行程序重用预准备语句。BATCH执行程序重用语句和批处理更新。	简单重新使用批次	简单
defaultStatementTimeout	设置驱动程序等待数据库响应的秒数。	任何正整数	未设置（null）
defaultFetchSize	为驱动程序设置一个提示，以控制返回结果的提取大小。可以通过查询设置覆盖此参数值。	任何正整数	未设置（null）
safeRowBoundsEnabled	允许在嵌套语句上使用RowBounds。如果允许，请设置false。	是的| 假	假
safeResultHandlerEnabled	允许在嵌套语句上使用ResultHandler。如果允许，请设置false。	是的| 假	真正
mapUnderscoreToCamelCase	启用从经典数据库列名A_COLUMN到驼峰式经典Java属性名称aColumn的自动映射。	是的| 假	假
localCacheScope	MyBatis使用本地缓存来防止循环引用并加速重复的嵌套查询。默认情况下（SESSION）会话期间执行的所有查询都将被缓存。如果localCacheScope = STATEMENT本地会话将仅用于语句执行，则不会在对同一SqlSession的两个不同调用之间共享数据。	会话| 声明	SESSION
jdbcTypeForNull	如果未为参数提供特定的JDBC类型，则指定空值的JDBC类型。某些驱动程序需要指定列JDBC类型，但其他驱动程序使用泛型值，如NULL，VARCHAR或OTHER。	JdbcType枚举。最常见的是：NULL，VARCHAR和OTHER	其他
lazyLoadTriggerMethods	指定哪个Object的方法触发延迟加载	以逗号分隔的方法名称列表	等于，克隆，hashCode时的toString
defaultScriptingLanguage	指定默认用于动态SQL生成的语言。	类型别名或完全限定的类名。	org.apache.ibatis.scripting.xmltags.XMLLanguageDriver
defaultEnumTypeHandler	指定类型处理器默认为枚举使用。（自：3.4.5）	类型别名或完全限定的类名。	org.apache.ibatis.type.EnumTypeHandler
callSettersOnNulls	指定当检索的值为null时是否将调用setter或map的put方法。当您依赖Map.keySet（）或null值初始化时，它很有用。注意（int，boolean等）原语不会设置为null。	是的| 假	假
returnInstanceForEmptyRow	默认情况下，MyBatis 在返回行的所有列都为NULL时返回null。启用此设置后，MyBatis将返回一个空实例。请注意，它也适用于嵌套结果（即集合和关联）。自：3.4.2	是的| 假	假
logPrefix	指定MyBatis将添加到记录器名称的前缀字符串。	任何字符串	没有设置
logImpl	指定MyBatis应使用的日志记录实现。如果此设置不存在，则会自动发现日志记录实现。	SLF4J | LOG4J | LOG4J2 | JDK_LOGGING | COMMONS_LOGGING | STDOUT_LOGGING | NO_LOGGING	没有设置
ProxyFactory里	指定MyBatis将用于创建具有延迟加载功能的对象的代理工具。	CGLIB | 了Javassist	JAVASSIST（MyBatis 3.3或以上）
vfsImpl	指定VFS实现	由逗号分隔的自定义VFS实现的完全限定类名。	没有设置
useActualParamName	允许通过方法签名中声明的实际名称引用语句参数。要使用此功能，必须使用-parameters选项在Java 8中编译项目。（自：3.4.1）	是的| 假	真正
configurationFactory	指定提供Configuration实例的类。返回的Configuration实例用于加载反序列化对象的延迟属性。该类必须具有带签名静态配置getConfiguration（）的方法。
```