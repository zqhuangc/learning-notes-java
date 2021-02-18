Mybatis 相关 xml 文件解析器

* XPathParser
* BaseBuilder
  * XMLConfigBuilder
  * XMLMapperBuilder
  * MapperBuilderAssistant
  * ...
* MapperAnnotationBuilder



# SqlSessionFactory

MyBatis的前身叫iBatis，本是apache的一个开源项目, 2010年这个项目由apache software foundation 迁移到了google code，并且改名为MyBatis。MyBatis是支持普通SQL查询，存储过程和高级映射的优秀持久层框架。MyBatis消除了几乎所有的JDBC代码和参数的手工设置以及结果集的检索。MyBatis使用简单的XML或注解用于配置和原始映射，将接口和Java的POJOs（Plan Old Java Objects，普通的Java对象）映射成数据库中的记录。

## 使用

* configuration.xml 

```
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
  PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>

	<!-- 指定properties配置文件， 我这里面配置的是数据库相关 -->
	<properties resource="resource.properties"></properties>

	<!-- 指定Mybatis使用log4j -->
	<settings>
		<setting name="logImpl" value="LOG4J" />
	</settings>

	<environments default="development">
		<environment id="development">
			<transactionManager type="JDBC" />
			<dataSource type="POOLED">
				<!-- 上面指定了数据库配置文件， 配置文件里面也是对应的这四个属性 -->
				<property name="driver" value="${jdbc.driverClass}" />
				<property name="url" value="${jdbc.url}" />
				<property name="username" value="${jdbc.userName}" />
				<property name="password" value="${jdbc.password}" />
			</dataSource>
		</environment>
	</environments>

	<!-- 映射文件，mybatis精髓， 后面才会细讲 -->
	<mappers>
		<mapper resource="sqlmap/UserMapper.xml"/>
	</mappers>
</configuration>
```

resource.properties

```
jdbc.driverClass=com.mysql.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/test?useUnicode=true&characterEncoding=utf-8
jdbc.userName=root
jdbc.password=root
```



- 实验代码

```
public static void main(String[] args) throws Exception {
        SqlSessionFactory sessionFactory = null;
        String resource = "configuration.xml";
        sessionFactory = new SqlSessionFactoryBuilder().build(Resources.getResourceAsReader(resource));
        SqlSession sqlSession = sessionFactory.openSession();
        UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
        System.out.println(userMapper.findUserById(1));
    }
```



## new SqlSessionFactoryBuilder()#build

```
/**
 * Builds {@link SqlSession} instances.
 *
 * @author Clinton Begin
 */
public class SqlSessionFactoryBuilder {

  public SqlSessionFactory build(Reader reader) {  //Reader读取mybatis配置文件 "configuration.xml";
    return build(reader, null, null);
  }
  ...
  //通过XMLConfigBuilder解析mybatis配置，然后创建SqlSessionFactory对象
  public SqlSessionFactory build(Reader reader, String environment, Properties properties) {
    try {
      XMLConfigBuilder parser = new XMLConfigBuilder(reader, environment, properties);
      return build(parser.parse());
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
      ErrorContext.instance().reset();
      try {
        reader.close();
      } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
      }
    }
  }
  ...
  public SqlSessionFactory build(InputStream inputStream, String environment, Properties properties) {
    try {
      XMLConfigBuilder parser = new XMLConfigBuilder(inputStream, environment, properties);
      return build(parser.parse());
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error building SqlSession.", e);
    } finally {
      ErrorContext.instance().reset();
      try {
        inputStream.close();
      } catch (IOException e) {
        // Intentionally ignore. Prefer previous error.
      }
    }
  }
  
  /**
   *
   */
  public SqlSessionFactory build(Configuration config) {
    return new DefaultSqlSessionFactory(config);
  }
}
```

通过源码，我们可以看到SqlSessionFactoryBuilder有不同的参数的build，通过XMLConfigBuilder 去解析我们传入的mybatis的配置文件，转化成Configuration，通过SqlSessionFactory build(Configuration config) 最终创建DefaultSqlSessionFactory

## XMLConfigBuilder#parse

```
 // mybatis 配置文件解析
 public XMLConfigBuilder(Reader reader, String environment, Properties props) {
    this(new XPathParser(reader, true, props, new XMLMapperEntityResolver()), environment, props);
  }
  
  private XMLConfigBuilder(XPathParser parser, String environment, Properties props) {
    super(new Configuration());
    ErrorContext.instance().resource("SQL Mapper Configuration");
    this.configuration.setVariables(props);
    this.parsed = false;
    this.environment = environment;
    this.parser = parser;
  }
//读取XMLConfigBuilder返回一个Configuration对象
 public Configuration parse() {
    if (parsed) {
      throw new BuilderException("Each XMLConfigBuilder can only be used once.");
    }
    parsed = true;
    parseConfiguration(parser.evalNode("/configuration"));//xml的根节点
    return configuration;
  }
  //configuration下面能配置的节点为以下11个节点 reflectionFactory是后面版本新增的
  //不同子节点进行Element处理
  private void parseConfiguration(XNode root) {
    try {
      Properties settings = settingsAsPropertiess(root.evalNode("settings"));
      //issue #117 read properties first
      propertiesElement(root.evalNode("properties"));
      loadCustomVfs(settings);
      typeAliasesElement(root.evalNode("typeAliases"));
      pluginElement(root.evalNode("plugins"));
      objectFactoryElement(root.evalNode("objectFactory"));
      objectWrapperFactoryElement(root.evalNode("objectWrapperFactory"));
      reflectionFactoryElement(root.evalNode("reflectionFactory"));
      settingsElement(settings);
      // read it after objectFactory and objectWrapperFactory issue #631
      environmentsElement(root.evalNode("environments"));
      databaseIdProviderElement(root.evalNode("databaseIdProvider"));
      typeHandlerElement(root.evalNode("typeHandlers"));
      mapperElement(root.evalNode("mappers"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing SQL Mapper Configuration. Cause: " + e, e);
    }
  }
```

通过以上源码，我们就能看出，在mybatis的配置文件中：

\1. configuration节点为根节点。

\2. 在configuration节点之下，我们可以配置11个子节点， 分别为：properties、typeAliases、plugins、objectFactory、objectWrapperFactory、settings、environments、databaseIdProvider、typeHandlers、mappers、reflectionFactory（后面版本新增的）

如下图：

![img](https://mmbiz.qpic.cn/mmbiz_jpg/YUYc62VIvE3SWGhuFs2iagcuZ3CF83bA3miarVyowxxj7jnSp5fHC5hHsVFAkzgRriaI1D5Dg9dW9ECTVCEfUW3gw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



下面我们看下子节点的源码

### propertiesElement（相关配置读取）

```
 private void propertiesElement(XNode context) throws Exception {
    if (context != null) {
      Properties defaults = context.getChildrenAsProperties();
      String resource = context.getStringAttribute("resource");
      String url = context.getStringAttribute("url");
      if (resource != null && url != null) { //如果resource和url都为空就抛出异常
        throw new BuilderException("The properties element cannot specify both a URL and a resource based property file reference.  Please specify one or the other.");
      }
      if (resource != null) {
        defaults.putAll(Resources.getResourceAsProperties(resource));
      } else if (url != null) {
        defaults.putAll(Resources.getUrlAsProperties(url));
      }
      Properties vars = configuration.getVariables();
      if (vars != null) {
        defaults.putAll(vars);
      }
//把Properties defaults 设置到 parser.setVariables
      parser.setVariables(defaults);
//把Properties defaults 设置到 configuration.setVariables
      configuration.setVariables(defaults);
    }
  }
```

  

### settingsAsPropertiess全局性的配置

```
private Properties settingsAsPropertiess(XNode context) {
    if (context == null) {
      return new Properties();
    }
    Properties props = context.getChildrenAsProperties();
    // Check that all settings are known to the configuration class
    MetaClass metaConfig = MetaClass.forClass(Configuration.class, localReflectorFactory);
    for (Object key : props.keySet()) {
      if (!metaConfig.hasSetter(String.valueOf(key))) {
        throw new BuilderException("The setting " + key + " is not known.  Make sure you spelled it correctly (case sensitive).");
      }
    }
    return props;
  }
  
  //settings元素设置 元素比较多不懂的可以看下官方文档
  private void settingsElement(Properties props) throws Exception {
     configuration.setAutoMappingBehavior(AutoMappingBehavior.valueOf(props.getProperty("autoMappingBehavior", "PARTIAL")));
     configuration.setAutoMappingUnknownColumnBehavior(AutoMappingUnknownColumnBehavior.valueOf(props.getProperty("autoMappingUnknownColumnBehavior", "NONE")));
     configuration.setCacheEnabled(booleanValueOf(props.getProperty("cacheEnabled"), true));
     configuration.setProxyFactory((ProxyFactory) createInstance(props.getProperty("proxyFactory")));
     configuration.setLazyLoadingEnabled(booleanValueOf(props.getProperty("lazyLoadingEnabled"), false));
     configuration.setAggressiveLazyLoading(booleanValueOf(props.getProperty("aggressiveLazyLoading"), true));
     configuration.setMultipleResultSetsEnabled(booleanValueOf(props.getProperty("multipleResultSetsEnabled"), true));
     configuration.setUseColumnLabel(booleanValueOf(props.getProperty("useColumnLabel"), true));
     configuration.setUseGeneratedKeys(booleanValueOf(props.getProperty("useGeneratedKeys"), false));
     configuration.setDefaultExecutorType(ExecutorType.valueOf(props.getProperty("defaultExecutorType", "SIMPLE")));
     configuration.setDefaultStatementTimeout(integerValueOf(props.getProperty("defaultStatementTimeout"), null));
     configuration.setDefaultFetchSize(integerValueOf(props.getProperty("defaultFetchSize"), null));
     configuration.setMapUnderscoreToCamelCase(booleanValueOf(props.getProperty("mapUnderscoreToCamelCase"), false));
     configuration.setSafeRowBoundsEnabled(booleanValueOf(props.getProperty("safeRowBoundsEnabled"), false));
     configuration.setLocalCacheScope(LocalCacheScope.valueOf(props.getProperty("localCacheScope", "SESSION")));
     configuration.setJdbcTypeForNull(JdbcType.valueOf(props.getProperty("jdbcTypeForNull", "OTHER")));
     configuration.setLazyLoadTriggerMethods(stringSetValueOf(props.getProperty("lazyLoadTriggerMethods"), "equals,clone,hashCode,toString"));
     configuration.setSafeResultHandlerEnabled(booleanValueOf(props.getProperty("safeResultHandlerEnabled"), true));
     configuration.setDefaultScriptingLanguage(resolveClass(props.getProperty("defaultScriptingLanguage")));
     configuration.setCallSettersOnNulls(booleanValueOf(props.getProperty("callSettersOnNulls"), false));
     configuration.setLogPrefix(props.getProperty("logPrefix"));
     configuration.setLogImpl(resolveClass(props.getProperty("logImpl"))); //这个是我们的配置文件里面设置的 日志
     configuration.setConfigurationFactory(resolveClass(props.getProperty("configurationFactory")));
   }
```

有关全局设置说明

![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE3SWGhuFs2iagcuZ3CF83bA3ug17k7EyZGgpORJSnUibh8vKpuJZ2bYplX3Tmsw1rCuudthmv1JiaM9A/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### typeAliasesElement 为一些类定义别名

```
  private void typeAliasesElement(XNode parent) {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          String typeAliasPackage = child.getStringAttribute("name");
          configuration.getTypeAliasRegistry().registerAliases(typeAliasPackage);
        } else {
          String alias = child.getStringAttribute("alias");
          String type = child.getStringAttribute("type");
          try {
            Class clazz = Resources.classForName(type);
            if (alias == null) {
              typeAliasRegistry.registerAlias(clazz);
            } else {
              typeAliasRegistry.registerAlias(alias, clazz);
            }
          } catch (ClassNotFoundException e) {
            throw new BuilderException("Error registering typeAlias for '" + alias + "'. Cause: " + e, e);
          }
        }
      }
    }
  }
  
  public void registerAlias(String alias, Class value) {
     if (alias == null) {
       throw new TypeException("The parameter alias cannot be null");
     }
     // issue #748
     String key = alias.toLowerCase(Locale.ENGLISH);
     if (TYPE_ALIASES.containsKey(key) && TYPE_ALIASES.get(key) != null && !TYPE_ALIASES.get(key).equals(value)) {
       throw new TypeException("The alias '" + alias + "' is already mapped to the value '" + TYPE_ALIASES.get(key).getName() + "'.");
     }
     TYPE_ALIASES.put(key, value); //最终是放到map里面
   }
```

### environmentsElement Mybatis的环境

```
//数据源  事务管理器 相关配置
  private void environmentsElement(XNode context) throws Exception {
    if (context != null) {
      if (environment == null) {
        environment = context.getStringAttribute("default"); //默认值是default
      }
      for (XNode child : context.getChildren()) {
        String id = child.getStringAttribute("id");
        if (isSpecifiedEnvironment(id)) {
   // MyBatis 有两种事务管理类型（即type=”[JDBC|MANAGED]”） Configuration.java的构造函数
          TransactionFactory txFactory = transactionManagerElement(child.evalNode("transactionManager"));//事物
          DataSourceFactory dsFactory = dataSourceElement(child.evalNode("dataSource")); //数据源
          DataSource dataSource = dsFactory.getDataSource();
          Environment.Builder environmentBuilder = new Environment.Builder(id)
              .transactionFactory(txFactory)
              .dataSource(dataSource);
          configuration.setEnvironment(environmentBuilder.build());  //放到configuration里面
        }
      }
    }
  }
```

  

###  mappers 映射文件或映射类

 既然 MyBatis 的行为已经由上述元素配置完了,我们现在就要定义 SQL 映射语句了。但是, 首先我们需要告诉 MyBatis  到哪里去找到这些语句。Java 在这方面没有提供一个很好 的方法, 所以最佳的方式是告诉 MyBatis 到哪里去找映射文件。你可以使用相对于类路径的  资源引用,或者字符表示,或 url 引用的完全限定名(包括 file:///URLs) 

```
//读取配置文件将接口放到configuration.addMappers(mapperPackage) 
//四种方式 resource、class name url

 private void mapperElement(XNode parent) throws Exception {
   if (parent != null) {
     for (XNode child : parent.getChildren()) {
       if ("package".equals(child.getName())) {
         // annotaion 方式 配置mapper文件
         String mapperPackage = child.getStringAttribute("name");
         configuration.addMappers(mapperPackage);
       } else {
         // xml方式 配置 mapper文件
         String resource = child.getStringAttribute("resource");
         String url = child.getStringAttribute("url");
         String mapperClass = child.getStringAttribute("class");
         if (resource != null && url == null && mapperClass == null) {//demo是通过xml加载的
           ErrorContext.instance().resource(resource);
           InputStream inputStream = Resources.getResourceAsStream(resource);
           XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
           mapperParser.parse();
         } else if (resource == null && url != null && mapperClass == null) {
           ErrorContext.instance().resource(url);
           InputStream inputStream = Resources.getUrlAsStream(url);
           XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
           mapperParser.parse();
         } else if (resource == null && url == null && mapperClass != null) {
           Class mapperInterface = Resources.classForName(mapperClass);
           configuration.addMapper(mapperInterface);
         } else {
           throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
         }
       }
     }
   }
 }
```

#### 扫描 package 形式 + 指定 class（annotation + xml）

```java
if ("package".equals(child.getName())) {
    // annotaion 方式 + xml 方式 配置 mapper 
    String mapperPackage = child.getStringAttribute("name");
    configuration.addMappers(mapperPackage);
}
```

##### MapperRegistry#addMappers

```java
public <T> void addMapper(Class<T> type) {
    if (type.isInterface()) {
      if (hasMapper(type)) {
        throw new BindingException("Type " + type + " is already known to the MapperRegistry.");
      }
      boolean loadCompleted = false;
      try {
        knownMappers.put(type, new MapperProxyFactory<>(type));
        // It's important that the type is added before the parser is run
        // otherwise the binding may automatically be attempted by the
        // mapper parser. If the type is already known, it won't try.
        MapperAnnotationBuilder parser = new MapperAnnotationBuilder(config, type);
        parser.parse();
        loadCompleted = true;
      } finally {
        if (!loadCompleted) {
          knownMappers.remove(type);
        }
      }
    }
  }
```



* MapperProxyFactory



##### `MapperAnnotationBuilder#parse`

```java
public void parse() {
    String resource = type.toString();
    if (!configuration.isResourceLoaded(resource)) {
      // 当前类型路径下的 xml 文件解析
      loadXmlResource();
      configuration.addLoadedResource(resource);
      assistant.setCurrentNamespace(type.getName());
      parseCache();
      parseCacheRef();
      for (Method method : type.getMethods()) {
        if (!canHaveStatement(method)) {
          continue;
        }
        if (getAnnotationWrapper(method, false, Select.class, SelectProvider.class).isPresent()
            && method.getAnnotation(ResultMap.class) == null) {
          parseResultMap(method);
        }
        try {
          parseStatement(method);
        } catch (IncompleteElementException e) {
          configuration.addIncompleteMethod(new MethodResolver(this, method));
        }
      }
    }
    parsePendingMethods();
  }
```



assistant.addMappedStatement



#### 指定 resource + url 形式（仅 xml）

```java
 else {
     // xml方式 配置 mapper
     String resource = child.getStringAttribute("resource");
     String url = child.getStringAttribute("url");
     String mapperClass = child.getStringAttribute("class");
     ...
 }
```

##### `XMLMapperBuilder#parse`

```java
public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
      //   
      configurationElement(parser.evalNode("/mapper"));
        
      configuration.addLoadedResource(resource);
      // 绑定
      bindMapperForNamespace();
    }

    parsePendingResultMaps();
    parsePendingCacheRefs();
    parsePendingStatements();
  }


private void bindMapperForNamespace() {
    String namespace = builderAssistant.getCurrentNamespace();
    if (namespace != null) {
      Class<?> boundType = null;
      try {
        boundType = Resources.classForName(namespace);
      } catch (ClassNotFoundException e) {
        // ignore, bound type is not required
      }
      if (boundType != null && !configuration.hasMapper(boundType)) {
        // Spring may not know the real resource name so we set a flag
        // to prevent loading again this resource from the mapper interface
        // look at MapperAnnotationBuilder#loadXmlResource
        configuration.addLoadedResource("namespace:" + namespace);
        configuration.addMapper(boundType);
      }
    }
  }
```



##### `XMLStatementBuilder#parseStatementNode 解析 sql相关标签`

injectMappedStatement

resultMapElement

##### `MapperBuilderAssistant#addMappedStatement`（mapper信息构建）

##### MappedStatement$Builder

```
MappedStatement statement = statementBuilder.build();
configuration.addMappedStatement(statement);
```

#### MappedStatement



# SqlSession 创建 和 关联Executor

```
  public static void main(String[] args) throws Exception {
        SqlSessionFactory sessionFactory = null;
        String resource = "configuration.xml";
        sessionFactory = new SqlSessionFactoryBuilder().build(Resources.getResourceAsReader(resource));
        SqlSession sqlSession = sessionFactory.openSession();
        UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
        System.out.println(userMapper.findUserById(1));
  }
```



## Configuration构建

SqlSessionFactoryBuilder().build(Resources.getResourceAsReader("mybatis-config.xml"));

build  --> XMLConfigBuilder#parse (解析 mybatis-config.xml)--> Configuration

build  --> new DefaultSqlSessionFactory(config);



## DefaultSqlSessionFactory#openSession()

```java
configuration:protected ExecutorType defaultExecutorType = ExecutorType.SIMPLE;
configuration.getDefaultExecutorType()  
@Override
  @Override
  public SqlSession openSession() {
    return openSessionFromDataSource(configuration.getDefaultExecutorType(), null, false);
  }
```

通过 openSession() 最终调用的是 DefaultSqlSessionFactory#openSessionFromDataSource

```
  private SqlSession openSessionFromDataSource(ExecutorType execType, TransactionIsolationLevel level, boolean autoCommit) {
    Transaction tx = null;
    try {
      final Environment environment = configuration.getEnvironment();
      final TransactionFactory transactionFactory = getTransactionFactoryFromEnvironment(environment);
      tx = transactionFactory.newTransaction(environment.getDataSource(), level, autoCommit);
      final Executor executor = configuration.newExecutor(tx, execType);
      return new DefaultSqlSession(configuration, executor, autoCommit);
    } catch (Exception e) {
      closeTransaction(tx); // may have fetched a connection so lets call close()
      throw ExceptionFactory.wrapException("Error opening session.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

#### Environment --- \<environment> 标签

从configuration 获取 environment，通过 environment 获得 TransactionFactory（一般配置 JdbcTransactionFactory）， 接着创建 Transaction

**JdbcTransaction** 直接使用了 JDBC 的提交和回滚设置。

**ManagedTransaction** 几乎什么也不做，翻开源码你会看到提交和回滚的方法里只有一行注释代码 “Does 



TransactionIsolationLevel  隔离级别5种 

NONE(Connection.TRANSACTION_NONE),      READ_COMMITTED(Connection.TRANSACTION_READ_COMMITTED),  READ_UNCOMMITTED(Connection.TRANSACTION_READ_UNCOMMITTED),      REPEATABLE_READ(Connection.TRANSACTION_REPEATABLE_READ),

SERIALIZABLE(Connection.TRANSACTION_SERIALIZABLE);	



#### Configuration#newExecutor

ExecutorType（执行器类型）类型有 SIMPLE（默认）,REUSE,BATCH,



```
public Executor newExecutor(Transaction transaction, ExecutorType executorType) {
    executorType = executorType == null ? defaultExecutorType : executorType;
    executorType = executorType == null ? ExecutorType.SIMPLE : executorType;
    Executor executor;
    //根据executorType创建不同的Executor对象
    if (ExecutorType.BATCH == executorType) {
      executor = new BatchExecutor(this, transaction);
    } else if (ExecutorType.REUSE == executorType) {
      executor = new ReuseExecutor(this, transaction);
    } else {
      executor = new SimpleExecutor(this, transaction);
    }
    if (cacheEnabled) {
      executor = new CachingExecutor(executor);
    }
    executor = (Executor) interceptorChain.pluginAll(executor);
    return executor;
  }
```

根据executorType创建对应的Executor，从源码可以看出他有BatchExecutor、ReuseExecutor、CachingExecutor、SimpleExecutor



#####  InterceptorChain#pluginAll(executor)

```java
// 生成代理
public Object pluginAll(Object target) {
    for (Interceptor interceptor : interceptors) {
      target = interceptor.plugin(target);
    }
    return target;
  }
Interceptor:
public interface Interceptor {

  Object intercept(Invocation invocation) throws Throwable;

  default Object plugin(Object target) {
    return Plugin.wrap(target, this);
  }

  default void setProperties(Properties properties) {
    // NOP
  }
}


Plugin implements InvocationHandler:
@Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      if (methods != null && methods.contains(method)) {
        return interceptor.intercept(new Invocation(target, method, args));
      }
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }
```

##### org.apache.ibatis.plugin.Interceptor 扩展点

具体可以看下面 **Mybatis的插件原理**

```
Configuration#addInterceptor
```

## DefaultSqlSession

```
new DefaultSqlSession(configuration, executor, autoCommit（默认false）);
```

### DefaultSqlSession#selectList

statement  命名空间.methodName

```java
@Override
  public <E> List<E> selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

####  configuration.getMappedStatement(statement)



#### BaseExecutor#query

```java
public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    if (queryStack == 0 && ms.isFlushCacheRequired()) {
      clearLocalCache();
    }
    List<E> list;
    try {
      queryStack++;
      list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;
      if (list != null) {
        handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
      } else {
        list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);
      }
    } finally {
      queryStack--;
    }
    if (queryStack == 0) {
      for (DeferredLoad deferredLoad : deferredLoads) {
        deferredLoad.load();
      }
      // issue #601
      deferredLoads.clear();
      if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
        // issue #482
        clearLocalCache();
      }
    }
    return list;
  }
```



### DefaultSqlSession#getMapper

```java
MapperRegistry#getMapper
MapperProxyFactory#newInstance(SqlSession)

public T newInstance(SqlSession sqlSession) {
    final MapperProxy<T> mapperProxy = new MapperProxy<>(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
  
    


```



#### MapperProxy#invoke

```java
  MapperProxy#invoke
   @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      if (Object.class.equals(method.getDeclaringClass())) {
        return method.invoke(this, args);
      } else {
        return cachedInvoker(method).invoke(proxy, method, args, sqlSession);
      }
    } catch (Throwable t) {
      throw ExceptionUtil.unwrapThrowable(t);
    }
  }
  
  private MapperMethodInvoker cachedInvoker(Method method) throws Throwable {
    try {
      // A workaround for https://bugs.openjdk.java.net/browse/JDK-8161372
      // It should be removed once the fix is backported to Java 8 or
      // MyBatis drops Java 8 support. See gh-1929
      MapperMethodInvoker invoker = methodCache.get(method);
      if (invoker != null) {
        return invoker;
      }

      return methodCache.computeIfAbsent(method, m -> {
        if (m.isDefault()) {
          try {
            if (privateLookupInMethod == null) {
              return new DefaultMethodInvoker(getMethodHandleJava8(method));
            } else {
              return new DefaultMethodInvoker(getMethodHandleJava9(method));
            }
          } catch (IllegalAccessException | InstantiationException | InvocationTargetException
              | NoSuchMethodException e) {
            throw new RuntimeException(e);
          }
        } else {
          return new PlainMethodInvoker(new MapperMethod(mapperInterface, method, sqlSession.getConfiguration()));
        }
      });
    } catch (RuntimeException re) {
      Throwable cause = re.getCause();
      throw cause == null ? re : cause;
    }
  }
  


MapperMethodInvoker  

DefaultMethodInvoker // 处理 接口中的 default 方法
@Override
    public Object invoke(Object proxy, Method method, Object[] args, SqlSession sqlSession) throws Throwable {
      // java8+ 提供的 methodHandle 
      return methodHandle.bindTo(proxy).invokeWithArguments(args);
    }


PlainMethodInvoker // sql 方法
@Override
    public Object invoke(Object proxy, Method method, Object[] args, SqlSession sqlSession) throws Throwable {
      return mapperMethod.execute(sqlSession, args);
    }

```





## Executor

BaseExecutor： Configuration 和 transaction 关联





Executor是接口，是对于Statement的封装，我们看下Executor,他是真正执行sql的地方。

```
public interface Executor {
  ResultHandler NO_RESULT_HANDLER = null;
  int update(MappedStatement ms, Object parameter) throws SQLException;
   List query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey cacheKey, BoundSql boundSql) throws SQLException;
   List query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException;
   Cursor queryCursor(MappedStatement ms, Object parameter, RowBounds rowBounds) throws SQLException;
  List flushStatements() throws SQLException;
  void commit(boolean required) throws SQLException;
  void rollback(boolean required) throws SQLException;
  CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql);
  boolean isCached(MappedStatement ms, CacheKey key);
  void clearLocalCache();
  void deferLoad(MappedStatement ms, MetaObject resultObject, String property, CacheKey key, Class targetType);
  Transaction getTransaction();
  void close(boolean forceRollback);
  boolean isClosed();
  void setExecutorWrapper(Executor executor);
}
```

上面源码我可以看到Executor接口定义了update 、query、commit、rollback等方法，他的实现类如下图

* CachingExecutor(Executor)   cacheEnabled  装饰器
* BaseExecutor
  * SimpleExecutor
  * ResultLoaderMap$ClosedExecutor
  * BatchExecutor
  * ReuseExecutor



### SimpleExecutor#doQuery（执行 sql）

```
  @Override
  public  List doQuery(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) throws SQLException {
    Statement stmt = null;
    try {
      Configuration configuration = ms.getConfiguration();
      StatementHandler handler = configuration.newStatementHandler(wrapper, ms, parameter, rowBounds, resultHandler, boundSql);
      
      // StatementHandler封装了Statement, 让 StatementHandler 去处理
      // sql 的完整设置
      stmt = prepareStatement(handler, ms.getStatementLog());
      return handler.query(stmt, resultHandler);
    } finally {
      closeStatement(stmt);
    }
  }
 
```

#### SimpleExecutor#prepareStatement

```java
  private Statement prepareStatement(StatementHandler handler, Log statementLog) throws SQLException {
    Statement stmt;
    Connection connection = getConnection(statementLog);
    stmt = handler.prepare(connection, transaction.getTimeout());
    // 参数映射 typehandler
    handler.parameterize(stmt);
    return stmt;
  }
```



我们看看StatementHandler 的一个实现类 PreparedStatementHandler（这也是我们最常用的，封装的是PreparedStatement）, 看看它使怎么去处理的：



##### PreparedStatementHandler#query



```
  @Override
  public  List query(Statement statement, ResultHandler resultHandler) throws SQLException {
    String sql = boundSql.getSql();
    //这个和jdbc一样执行sql
    statement.execute(sql);
    //结果交给了ResultSetHandler 去处理
    return resultSetHandler.handleResultSets(statement);
  }
```

以上是sql底层执行的基本流程，说的直白一点就是所以sql底层都交给了Excutor，我们将在下一讲中分析上一层的调用，也就是Excutor的上层。

##### ResultSetHandler#handleResultSets

```
DefaultResultSetHandler
ResultSet


DefaultResultHandler
resultmap

DefaultResultSetHandler#handleResultSet
public void handleRowValues(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping) throws SQLException {
    if (resultMap.hasNestedResultMaps()) {
      ensureNoRowBounds();
      checkResultHandler();
      handleRowValuesForNestedResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    } else {
      handleRowValuesForSimpleResultMap(rsw, resultMap, resultHandler, rowBounds, parentMapping);
    }
  }

private void handleRowValuesForSimpleResultMap(ResultSetWrapper rsw, ResultMap resultMap, ResultHandler<?> resultHandler, RowBounds rowBounds, ResultMapping parentMapping)
      throws SQLException {
    DefaultResultContext<Object> resultContext = new DefaultResultContext<>();
    ResultSet resultSet = rsw.getResultSet();
    skipRows(resultSet, rowBounds);
    while (shouldProcessMoreRows(resultContext, rowBounds) && !resultSet.isClosed() && resultSet.next()) {
      ResultMap discriminatedResultMap = resolveDiscriminatedResultMap(resultSet, resultMap, null);
      Object rowValue = getRowValue(rsw, discriminatedResultMap, null);
      storeObject(resultHandler, resultContext, rowValue, parentMapping, resultSet);
    }
  }
  

hasTypeHandlerForResultObject(rsw, resultMap.getType())) 
applyPropertyMappings


private Object getPropertyMappingValue(ResultSet rs, MetaObject metaResultObject, ResultMapping propertyMapping, ResultLoaderMap lazyLoader, String columnPrefix)
      throws SQLException {
    if (propertyMapping.getNestedQueryId() != null) {
      return getNestedQueryMappingValue(rs, metaResultObject, propertyMapping, lazyLoader, columnPrefix);
    } else if (propertyMapping.getResultSet() != null) {
      addPendingChildRelation(rs, metaResultObject, propertyMapping);   // TODO is that OK?
      return DEFERRED;
    } else {
      final TypeHandler<?> typeHandler = propertyMapping.getTypeHandler();
      final String column = prependPrefix(propertyMapping.getColumn(), columnPrefix);
      return typeHandler.getResult(rs, column);
    }
  }
  
TypeHandler在 何时设到 resultMap
XMLMapperBuilder#resultMapElement

buildResultMappingFromContext
MapperBuilderAssistant#buildResultMapping
ResultMapping$Builder#build
ResultMapping$Builder#resolveTypeHandler()
private void resolveTypeHandler() {
      if (resultMapping.typeHandler == null && resultMapping.javaType != null) {
        Configuration configuration = resultMapping.configuration;
        TypeHandlerRegistry typeHandlerRegistry = configuration.getTypeHandlerRegistry();
        resultMapping.typeHandler = typeHandlerRegistry.getTypeHandler(resultMapping.javaType, resultMapping.jdbcType);
      }
    }
```

* BaseStatementHandler

```java
protected BaseStatementHandler(Executor executor, MappedStatement mappedStatement, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, BoundSql boundSql) {
    this.configuration = mappedStatement.getConfiguration();
    this.executor = executor;
    this.mappedStatement = mappedStatement;
    this.rowBounds = rowBounds;

    this.typeHandlerRegistry = configuration.getTypeHandlerRegistry();
    this.objectFactory = configuration.getObjectFactory();

    if (boundSql == null) { // issue #435, get the key before calculating the statement
      generateKeys(parameterObject);
      boundSql = mappedStatement.getBoundSql(parameterObject);
    }

    this.boundSql = boundSql;

    this.parameterHandler = configuration.newParameterHandler(mappedStatement, parameterObject, boundSql);
    this.resultSetHandler = configuration.newResultSetHandler(executor, mappedStatement, rowBounds, parameterHandler, resultHandler, boundSql);
  }
```









# Mapper 执行 Sql

我们想看下基本的时序图有个大致了解

![img](https://mmbiz.qpic.cn/mmbiz_jpg/YUYc62VIvE1VpByZTUX1PcyhdoBmwpweZhYV3YqEEBtC9rnmQSpXA8qGia4QUtNb2o2nOkEx0Y0PcEqgrAw9icCw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



RowBounds



## DefaultSqlSession 获取 getMapper

```
  @Override
  public  T getMapper(Class type) {
    return configuration.getMapper(type, this);
  }
```

 

## Configuration获取getMapper

```
 public  T getMapper(Class type, SqlSession sqlSession) {
   return mapperRegistry.getMapper(type, sqlSession);
 }
```



## MapperRegistry获取getMapper

```
 @SuppressWarnings("unchecked")
  public  T getMapper(Class type, SqlSession sqlSession) {
    final MapperProxyFactory mapperProxyFactory = (MapperProxyFactory) knownMappers.get(type);
    if (mapperProxyFactory == null) {
      throw new BindingException("Type " + type + " is not known to the MapperRegistry.");
    }
    try {
      return mapperProxyFactory.newInstance(sqlSession);
    } catch (Exception e) {
      throw new BindingException("Error getting mapper instance. Cause: " + e, e);
    }
  }
```

通过 knownMappers 获取一个MapperProxyFactory，后然newInstance了一下，那么newInstance得到了什么东西呢？



## MapperProxyFactory

```
  @SuppressWarnings("unchecked")
  protected T newInstance(MapperProxy mapperProxy) {
  //java代理
    return (T) Proxy.newProxyInstance(mapperInterface.getClassLoader(), new Class[] { mapperInterface }, mapperProxy);
  }
  public T newInstance(SqlSession sqlSession) {
    final MapperProxy mapperProxy = new MapperProxy(sqlSession, mapperInterface, methodCache);
    return newInstance(mapperProxy);
  }
```

  

通过以上的动态代理，咱们就可以方便地使用dao接口啦。到这里我们还没有看到任何执行sql有关的信息，或者说还没走到文章开始说的的Excutor， 我们看下MapperProxy代理类



## MapperProxy

```
public class MapperProxy implements InvocationHandler, Serializable {
  private static final long serialVersionUID = -6424540398559729838L;
  private final SqlSession sqlSession;
  private final Class mapperInterface;
  private final Map methodCache;
  public MapperProxy(SqlSession sqlSession, Class mapperInterface, Map methodCache) {
    this.sqlSession = sqlSession;
    this.mapperInterface = mapperInterface;
    this.methodCache = methodCache;
  }
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    if (Object.class.equals(method.getDeclaringClass())) {
      try {
        return method.invoke(this, args);
      } catch (Throwable t) {
        throw ExceptionUtil.unwrapThrowable(t);
      }
    }
    final MapperMethod mapperMethod = cachedMapperMethod(method);
    return mapperMethod.execute(sqlSession, args);
  }
  private MapperMethod cachedMapperMethod(Method method) {
    MapperMethod mapperMethod = methodCache.get(method);
    if (mapperMethod == null) {
      mapperMethod = new MapperMethod(mapperInterface, method, sqlSession.getConfiguration());
      methodCache.put(method, mapperMethod);
    }
    return mapperMethod;
  }
}
```



代理类交给了mapperMethod.execute进行处理

#### MapperMethod#execute

```
  public Object execute(SqlSession sqlSession, Object[] args) {
    Object result;
    if (SqlCommandType.INSERT == command.getType()) {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.insert(command.getName(), param));
    } else if (SqlCommandType.UPDATE == command.getType()) {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.update(command.getName(), param));
    } else if (SqlCommandType.DELETE == command.getType()) {
      Object param = method.convertArgsToSqlCommandParam(args);
      result = rowCountResult(sqlSession.delete(command.getName(), param));
    } else if (SqlCommandType.SELECT == command.getType()) {
      if (method.returnsVoid() && method.hasResultHandler()) {
        executeWithResultHandler(sqlSession, args);
        result = null;
      } else if (method.returnsMany()) {
        result = executeForMany(sqlSession, args);
      } else if (method.returnsMap()) {
        result = executeForMap(sqlSession, args);
      } else if (method.returnsCursor()) {
        result = executeForCursor(sqlSession, args);
      } else {
        Object param = method.convertArgsToSqlCommandParam(args);
        result = sqlSession.selectOne(command.getName(), param);
      }
    } else if (SqlCommandType.FLUSH == command.getType()) {
        result = sqlSession.flushStatements();
    } else {
      throw new BindingException("Unknown execution method for: " + command.getName());
    }
    if (result == null && method.getReturnType().isPrimitive() && !method.returnsVoid()) {
      throw new BindingException("Mapper method '" + command.getName() 
          + " attempted to return null from a method with a primitive return type (" + method.getReturnType() + ").");
    }
    return result;
  }
```

##### sqlSession#selectOne。

```
  @Override
  public  T selectOne(String statement, Object parameter) {
    // Popular vote was to return null on 0 results and throw exception on too many.
    List list = this.selectList(statement, parameter);
    if (list.size() == 1) {
      return list.get(0);
    } else if (list.size() > 1) {
      throw new TooManyResultsException("Expected one result (or null) to be returned by selectOne(), but found: " + list.size());
    } else {
      return null;
    }
  }
  
    @Override
  public  List selectList(String statement, Object parameter, RowBounds rowBounds) {
    try {
      MappedStatement ms = configuration.getMappedStatement(statement);
      //终于看到我们要找的executor接口了
      return executor.query(ms, wrapCollection(parameter), rowBounds, Executor.NO_RESULT_HANDLER);
    } catch (Exception e) {
      throw ExceptionFactory.wrapException("Error querying database.  Cause: " + e, e);
    } finally {
      ErrorContext.instance().reset();
    }
  }
```

  看到executor类了， 调用了query方法，接下来的事情全部交给了executor处理了，

我们代理执行sql的基本顺序是

MapperMethod.execute() --> DefaultSqlSession.selectOne  -->  BaseExecutor.query  -->  SimpleExecutor.doQuery  --> SimpleStatementHandler.query -->  DefaultResultSetHandler.handleResultSets(Statement stmt)  最终得到数据



# Cache 一级缓存

它的缓存分为一级缓存和二级缓存，本文我们主要分析下一级缓存。

```
    public static void main(String[] args) throws Exception {
        SqlSessionFactory sessionFactory = null;
        String resource = "configuration.xml";
        sessionFactory = new SqlSessionFactoryBuilder().build(Resources.getResourceAsReader(resource));
        SqlSession sqlSession = sessionFactory.openSession();
        UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
        System.out.println(userMapper.findUserById(1));
        System.out.println(userMapper.findUserById(1));
    }
```

上面代码我们执行了两次userMapper.findUserById(1)，结果如下图

![img](https://mmbiz.qpic.cn/mmbiz_jpg/YUYc62VIvE1URIWRibib5S3GqukiaBibXRd0ZT0HHjSdiaB8ZUYabdhTjz3kT215XmGEoF0t9OicEsIfcyzbIP2AXZicg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



从执行结果看，DB的查询只有1次，那么第二次的查询结果是在怎么来的呢？

## 一：什么是一级缓存



 	 每当我们使用MyBatis开启一次和数据库的会话，MyBatis会创建出一个SqlSession对象表示一次数据库会话。 在对数据库的一次会话中，我们有可能会反复地执行完全相同的查询语句，如果不采取一些措施的话，每一次查询都会查询一次数据库,而我们在极短的时间内做了完全相同的查询，那么它们的结果极有可能完全相同，由于查询一次数据库的代价很大，这有可能造成很大的资源浪费。      

​	为了解决这一问题，减少资源的浪费，MyBatis会在表示会话的SqlSession对象中建立一个简单的缓存，将每次查询到的结果结果缓存起来，当下次查询的时候，如果判断先前有个完全一样的查询，会直接从缓存中直接将结果取出，返回给用户，不需要再进行一次数据库查询了。

​     基本的流程示意图如下：

​     对于会话（Session）级别的数据缓存，我们称之为一级数据缓存，简称一级缓存。

​     

## 二：如何执行缓存

​    我们知道mybatis的SQL执行最后是交给了Executor执行器来完成的，我们看下BaseExecutor类的源码

### BaseExecutor#query

```
 @Override
     public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler) throws SQLException {
       BoundSql boundSql = ms.getBoundSql(parameter);
       CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
       return query(ms, parameter, rowBounds, resultHandler, key, boundSql);
    }
     
     @SuppressWarnings("unchecked")
     @Override
     public <E> List<E> query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
       ErrorContext.instance().resource(ms.getResource()).activity("executing a query").object(ms.getId());
       if (closed) {
         throw new ExecutorException("Executor was closed.");
       }
       if (queryStack == 0 && ms.isFlushCacheRequired()) {
         clearLocalCache();
       }
       List<E> list;
       try {
         queryStack++;
         list = resultHandler == null ? (List<E>) localCache.getObject(key) : null;//localCache 本地缓存
         if (list != null) {
           handleLocallyCachedOutputParameters(ms, key, parameter, boundSql);
         } else {
           list = queryFromDatabase(ms, parameter, rowBounds, resultHandler, key, boundSql);  //如果缓存没有就走DB
         }
       } finally {
         queryStack--;
       }
       if (queryStack == 0) {
         for (DeferredLoad deferredLoad : deferredLoads) {
           deferredLoad.load();
         }
         // issue #601
         deferredLoads.clear();
         if (configuration.getLocalCacheScope() == LocalCacheScope.STATEMENT) {
           // issue #482
           clearLocalCache();
         }
       }
       return list;
     }
     
     
     
  private <E> List<E> queryFromDatabase(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql) throws SQLException {
    List<E> list;
    localCache.putObject(key, EXECUTION_PLACEHOLDER);
    try {
      list = doQuery(ms, parameter, rowBounds, resultHandler, boundSql);
    } finally {
      localCache.removeObject(key);//清空现有缓存数据
    }
    localCache.putObject(key, list);//新的结果集存入缓存
    if (ms.getStatementType() == StatementType.CALLABLE) {
      localOutputParameterCache.putObject(key, parameter);
    }
    return list;
  }
```



通过上面的三个方法我们基本已经看明白了缓存的使用，它的本地缓存使用的是PerpetualCache类，内部其实还是一个HashMap，只是稍微做了封装而已。

我们再看下天的Key是如何生成的

```
 CacheKey key = createCacheKey(ms, parameter, rowBounds, boundSql);
```





```
  @Override
  public CacheKey createCacheKey(MappedStatement ms, Object parameterObject, RowBounds rowBounds, BoundSql boundSql) {
    if (closed) {
      throw new ExecutorException("Executor was closed.");
    }
    CacheKey cacheKey = new CacheKey();
    cacheKey.update(ms.getId());
    cacheKey.update(Integer.valueOf(rowBounds.getOffset()));
    cacheKey.update(Integer.valueOf(rowBounds.getLimit()));
    cacheKey.update(boundSql.getSql());
    List<ParameterMapping> parameterMappings = boundSql.getParameterMappings();
    TypeHandlerRegistry typeHandlerRegistry = ms.getConfiguration().getTypeHandlerRegistry();
    // mimic DefaultParameterHandler logic
    for (int i = 0; i < parameterMappings.size(); i++) {
      ParameterMapping parameterMapping = parameterMappings.get(i);
      if (parameterMapping.getMode() != ParameterMode.OUT) {
        Object value;
        String propertyName = parameterMapping.getProperty();
        if (boundSql.hasAdditionalParameter(propertyName)) {
          value = boundSql.getAdditionalParameter(propertyName);
        } else if (parameterObject == null) {
          value = null;
        } else if (typeHandlerRegistry.hasTypeHandler(parameterObject.getClass())) {
          value = parameterObject;
        } else {
          MetaObject metaObject = configuration.newMetaObject(parameterObject);
          value = metaObject.getValue(propertyName);
        }
        cacheKey.update(value);
      }
    }
    if (configuration.getEnvironment() != null) {
      // issue #176
      cacheKey.update(configuration.getEnvironment().getId());
    }
    return cacheKey;
  }
```

它是通过statementId,params,rowBounds，BoundSql来构建一个key值，根据这个key值去缓存Cache中取出对应的key值存储的缓存结果。



## 三：一级缓存生命周期



\1. MyBatis在开启一个数据库会话时，会创建一个新的SqlSession对象，SqlSession对象中会有一个新的Executor对象，

Executor对象中持有一个新的PerpetualCache对象；当会话结束时，SqlSession对象及其内部的Executor对象还有PerpetualCache对象也一并释放掉。

\2. 如果SqlSession调用了close()方法，会释放掉一级缓存PerpetualCache对象，一级缓存将不可用。

\3. 如果SqlSession调用了clearCache()，会清空PerpetualCache对象中的数据，但是该对象仍可使用。

4.SqlSession中执行了任何一个update操作(update()、delete()、insert()) ，都会清空PerpetualCache对象的数据，但是该对象可以继续使用。



# Cache 二级缓存

## 一：Cache类的介绍

讲解缓存之前我们需要先了解一下Cache接口以及实现MyBatis定义了一个org.apache.ibatis.cache.Cache接口作为其Cache提供者的SPI(ServiceProvider Interface) ，所有的MyBatis内部的Cache缓存，都应该实现这一接口

Cache的实现类中，Cache有不同的功能，每个功能独立，互不影响,则对于不同的Cache功能，这里使用了装饰者模式实现。

看下cache的实现类，如下图：

![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE3a6mCEzaMVNEoibcrku0NPpKSART5sBuyicOuQmF2BI8HplMbIpHYUDkTekzyq1JBrepHuhJolmWCA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

```
1.FIFOCache：先进先出算法 回收策略，装饰类，内部维护了一个队列，来保证FIFO，一旦超出指定的大小，
则从队列中获取Key并从被包装的Cache中移除该键值对。

2.LoggingCache：输出缓存命中的日志信息,如果开启了DEBUG模式，则会输出命中率日志。

3.LruCache：最近最少使用算法，缓存回收策略,在内部保存一个LinkedHashMap

4.ScheduledCache：定时清空Cache，但是并没有开始一个定时任务，而是在使用Cache的时候，才去检查时间是否到了。

5.SerializedCache：序列化功能，将值序列化后存到缓存中。该功能用于缓存返回一份实例的Copy，用于保存线程安全。

6.SoftCache：基于软引用实现的缓存管理策略,软引用回收策略，软引用只有当内存不足时才会被垃圾收集器回收

7.SynchronizedCache：同步的缓存装饰器，用于防止多线程并发访问

8.PerpetualCache 永久缓存，一旦存入就一直保持，内部就是一个HashMap

9.WeakCache：基于弱引用实现的缓存管理策略

10.TransactionalCache 事务缓存，一次性存入多个缓存，移除多个缓存

11.BlockingCache 可阻塞的缓存,内部实现是ConcurrentHashMap
```



## 二：二级缓存初始化

Mybatis默认对二级缓存是关闭的，一级缓存默认开启，如果需要开启只需在mapper上加入配置就好了

二级缓存是怎么初始化的呢？

 我们在之前的文章里面（Mybatis源码分析之SqlSessionFactory（一））分析了配置文件的加载，我们回到那边来找到二级缓存的加载地方，一开始我就说了“如果需要开启只需在mapper上加入配置就好了”那么加载一定是在mapper的节点

```
XMLConfigBuilder.parse()-->parseConfiguration(XNode root)-->mapperElement(root.evalNode("mappers"))-->
mapperElement(XNode parent)
```



看下mapperElement的方法

```
private void mapperElement(XNode parent) throws Exception {
    if (parent != null) {
      for (XNode child : parent.getChildren()) {
        if ("package".equals(child.getName())) {
          String mapperPackage = child.getStringAttribute("name");
          configuration.addMappers(mapperPackage);
        } else {
          String resource = child.getStringAttribute("resource");
          String url = child.getStringAttribute("url");
          String mapperClass = child.getStringAttribute("class");
          if (resource != null && url == null && mapperClass == null) {
            ErrorContext.instance().resource(resource);
            InputStream inputStream = Resources.getResourceAsStream(resource);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, resource, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url != null && mapperClass == null) {
            ErrorContext.instance().resource(url);
            InputStream inputStream = Resources.getUrlAsStream(url);
            XMLMapperBuilder mapperParser = new XMLMapperBuilder(inputStream, configuration, url, configuration.getSqlFragments());
            mapperParser.parse();
          } else if (resource == null && url == null && mapperClass != null) {
            Class mapperInterface = Resources.classForName(mapperClass);
            configuration.addMapper(mapperInterface);
          } else {
            throw new BuilderException("A mapper element may only specify a url, resource or class, but not more than one.");
          }
        }
      }
    }
  }
```



这个时候我已经找到了mapper节点了，我们在看向前走

```
XMLMapperBuilder.mapperParser.parse()
```

代码如下

```
public void parse() {
    if (!configuration.isResourceLoaded(resource)) {
      configurationElement(parser.evalNode("/mapper"));
      configuration.addLoadedResource(resource);
      bindMapperForNamespace();
    }
    parsePendingResultMaps();
    parsePendingChacheRefs();
    parsePendingStatements();
  }
```



看到configurationElement(parser.evalNode("/mapper"));看到mapper节点了继续走

```
 private void configurationElement(XNode context) {
    try {
      String namespace = context.getStringAttribute("namespace");
      if (namespace == null || namespace.equals("")) {
        throw new BuilderException("Mapper's namespace cannot be empty");
      }
      builderAssistant.setCurrentNamespace(namespace);
      cacheRefElement(context.evalNode("cache-ref"));
      cacheElement(context.evalNode("cache"));
      parameterMapElement(context.evalNodes("/mapper/parameterMap"));
      resultMapElements(context.evalNodes("/mapper/resultMap"));
      sqlElement(context.evalNodes("/mapper/sql"));
      buildStatementFromContext(context.evalNodes("select|insert|update|delete"));
    } catch (Exception e) {
      throw new BuilderException("Error parsing Mapper XML. Cause: " + e, e);
    }
  }
```



 到这里终于见到了他cacheElement(context.evalNode("cache"));

 看源码

```
 private void cacheElement(XNode context) throws Exception {
   if (context != null) {
     String type = context.getStringAttribute("type", "PERPETUAL");
     Class typeClass = typeAliasRegistry.resolveAlias(type);
     String eviction = context.getStringAttribute("eviction", "LRU");
     Class evictionClass = typeAliasRegistry.resolveAlias(eviction);
     Long flushInterval = context.getLongAttribute("flushInterval");
     Integer size = context.getIntAttribute("size");
     boolean readWrite = !context.getBooleanAttribute("readOnly", false);
     boolean blocking = context.getBooleanAttribute("blocking", false);
     Properties props = context.getChildrenAsProperties();
     builderAssistant.useNewCache(typeClass, evictionClass, flushInterval, size, readWrite, blocking, props);
   }
 }
```

终于找到了cacheElement读取，这里builderAssistant.useNewCache构建了一个二级缓存对象

```
public Cache useNewCache(Class typeClass,
      Class evictionClass,
      Long flushInterval,
      Integer size,
      boolean readWrite,
      boolean blocking,
      Properties props) {
    Cache cache = new CacheBuilder(currentNamespace)
        .implementation(valueOrDefault(typeClass, PerpetualCache.class))
        .addDecorator(valueOrDefault(evictionClass, LruCache.class))
        .clearInterval(flushInterval)
        .size(size)
        .readWrite(readWrite)
        .blocking(blocking)
        .properties(props)
        .build();
    configuration.addCache(cache);
    currentCache = cache;
    return cache;
  }
```



创建完成后放入configuration对象configuration.addCache(cache)，上面代码看到 addDecorator(valueOrDefault(evictionClass, LruCache.class))LruCache被装饰到里面了，

![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE3a6mCEzaMVNEoibcrku0NPpiczfrOgA2ert3ADmfLa1zaozPeU6HmjL4qtejWaNQdUh6Ip0jiafBKpA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

最后得到的对象是SynchronizedCache可以在.build()里面找到，他内部装饰设计模式。

## 三：缓存查数据  



通过之前的文章我们知道Executor是执行查询的最终接口，它有两个实现类一个是BaseExecutor另外一个是CachingExecutor。

我看下CachingExecutor实现类里面的query查询方法



```
 @Override
  public  List query(MappedStatement ms, Object parameterObject, RowBounds rowBounds, ResultHandler resultHandler, CacheKey key, BoundSql boundSql)
      throws SQLException {
    Cache cache = ms.getCache();//二级缓存对象
    if (cache != null) {
      flushCacheIfRequired(ms);
      if (ms.isUseCache() && resultHandler == null) {
        ensureNoOutParams(ms, parameterObject, boundSql);
        @SuppressWarnings("unchecked")
        List list = (List) tcm.getObject(cache, key);//从缓存中读取
        if (list == null) {
  //这段走到一级缓存或者DB
          list = delegate. query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
          tcm.putObject(cache, key, list); // issue #578 and #116  //数据放入缓存
        }
        return list;
      }
    }
    return delegate. query(ms, parameterObject, rowBounds, resultHandler, key, boundSql);
  }
```



在这里我们看一个实例如下图



![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE3a6mCEzaMVNEoibcrku0NPpb0uWTduibPKeMqXtWKwFw8ibmnwiayd7E9GX5yYVeNkVhjwkLn6vjEduQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

发现我们的二级缓存没有生效，这个是为什么呢？我们仔细分下源代码

仔细看CachingExecutor的query代码，在查询二级缓存的时候用的是tcm先去找TransactionalCache然后采取getObject。问题就在这里，但我们在一个事物里查询三次，第一次查数据库，这不必说，第二次以后会判断二级缓存时候有。第一次查询完了有这么一句。

tcm.putObject(cache, key, list); 

跟进去看：

```
getTransactionalCache(cache).putObject(key, value);
```

getTransactionalCache(cache)返回TransactionalCache对象，然后调用它的put，是什么呢

```
@Override  
  public void putObject(Object key, Object object) {  
    entriesToRemoveOnCommit.remove(key);  
    entriesToAddOnCommit.put(key, new AddEntry(delegate, key, object));  
  }
```

封装cache在一个addEntry对象中去了。

put方法不是保存数据到TransactionalCache，而是保存cache到entriesToAddOnCommit；那这个entriesToAddOnCommit干吗用的呢？



观察名字就知道是提交事务的时候需要用的。一个方法执行结束，事务提交，session提交，提交是层层调用的。最终调用到CachingExecutor的commit：



```
public void commit(boolean required) throws SQLException {  
    delegate.commit(required);  
    tcm.commit();  
  }
```

tcm的commit：

```
public void commit() {  
    for (TransactionalCache txCache : transactionalCaches.values()) {  
      txCache.commit();  
    }  
  }
```



把所有TransactionalCache提交，

```
public void commit() {  
    if (clearOnCommit) {  
      delegate.clear();  
    } else {  
      for (RemoveEntry entry : entriesToRemoveOnCommit.values()) {  
        entry.commit();  
      }  
    }  
    for (AddEntry entry : entriesToAddOnCommit.values()) {  
      entry.commit();  
    }  
    reset();  
  }
```



AddEntry的commit方法：

```
public void commit() {  
      cache.putObject(key, value);  
    }
```



就是把缓存数据放到二级缓存。

**总结就是：**

一个事务方法运行时，结果查询出来，缓存在一级缓存了，但是没有到二级缓存，事务cache只是保存了二级缓存的引用以及需要缓存的数据key和数据。当事务提交后，事务cache重置，之前保存的本该在二级缓存的数据在此刻真正放到二级缓存。



于是我们在这个方法中反复查询，二级缓存启用了却不能命中，只能返回一级缓存的数据。要想命中必须提交事务才行，第二个测试每次打开事务，查询，释放事务，在获得事务查询。所以二级缓存能命中。



我们调整下代码方法，把事物提交放到前面

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

这下正常了

## 四：一级和二级缓存的先后顺序



二级缓存———> 一级缓存——> 数据库 



## 五：使用二级缓存需要注意：



想要使用二级缓存时需要想好三个问题：

1）对该表的操作与查询都在同一个namespace下，其他的namespace如果有操作，就会发生数据过时。因为二级缓存是以namespace为单位的，不同namespace下的操作互不影响。

2）对关联表的查询，关联的所有表的操作都必须在同一个namespace。

3）不能直接操作数据库，否则数据查询结果会存在问题

总之，操作与查询在同一个namespace下的查询才能缓存，其他namespace下的查询都可能出现问题。



# 自定义缓存

## 一：自定义mybatis缓存

​    我们知道任何mybatis二级缓存都需要实现一个接口，这个接口就是org.apache.ibatis.cache.Cache,代码如下：

​	

```
	package com.demo.spring.mybatis.cache;

	import java.util.concurrent.locks.ReadWriteLock;
	import java.util.concurrent.locks.ReentrantReadWriteLock;

	import org.apache.ibatis.cache.Cache;
	import org.slf4j.Logger;
	import org.slf4j.LoggerFactory;

	import com.demo.spring.mybatis.util.SerializeUtil;

	import redis.clients.jedis.Jedis;
	import redis.clients.jedis.JedisPool;

	public class MybatisRedisCache implements Cache {
		private static Logger logger = LoggerFactory.getLogger(MybatisRedisCache.class);

		private Jedis redisClient = createReids();

		private final ReadWriteLock readWriteLock = new ReentrantReadWriteLock();

		private String id;

		public MybatisRedisCache(final String id) {
			if (id == null) {
				throw new IllegalArgumentException("Cache instances require an ID");
			}
			logger.debug(">>>>>>>>>>>>>>>>>>>>>>>>MybatisRedisCache:id=" + id);
			this.id = id;
		}

		@Override
		public String getId() {
			return this.id;
		}

		@Override
		public int getSize() {

			return Integer.valueOf(redisClient.dbSize().toString());
		}

		@Override
		public void putObject(Object key, Object value) {
			logger.debug(">>>>>>>>>>>>>>>>>>>>>>>>putObject:" + key + "=" + value);
			redisClient.set(SerializeUtil.serialize(key.toString()), SerializeUtil.serialize(value));
		}

		@Override
		public Object getObject(Object key) {
			Object value = SerializeUtil.unserialize(redisClient.get(SerializeUtil.serialize(key.toString())));
			logger.debug(">>>>>>>>>>>>>>>>>>>>>>>>getObject:" + key + "=" + value);
			return value;
		}

		@Override
		public Object removeObject(Object key) {
			return redisClient.expire(SerializeUtil.serialize(key.toString()), 0);
		}

		@Override
		public void clear() {
			redisClient.flushDB();
		}

		@Override
		public ReadWriteLock getReadWriteLock() {
			return readWriteLock;
		}

		protected static Jedis createReids() {
			JedisPool pool = new JedisPool("127.0.0.1", 6379);
			return pool.getResource();
		}
	}	
```

以上代码很简单就是基本的Cache实现，在定义一个序列化的工具类

```
  package com.demo.spring.mybatis.util;

import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;

public class SerializeUtil {
	public static byte[] serialize(Object object) {
		ObjectOutputStream oos = null;
		ByteArrayOutputStream baos = null;
		try {
			// 序列化
			baos = new ByteArrayOutputStream();
			oos = new ObjectOutputStream(baos);
			oos.writeObject(object);
			byte[] bytes = baos.toByteArray();
			return bytes;
		} catch (Exception e) {
			e.printStackTrace();
		}
		return null;
	}

	public static Object unserialize(byte[] bytes) {
		ByteArrayInputStream bais = null;
		try {
			// 反序列化
			bais = new ByteArrayInputStream(bytes);
			ObjectInputStream ois = new ObjectInputStream(bais);
			return ois.readObject();
		} catch (Exception e) {

		}
		return null;
	}
}
```

然后在mapper.xml配置

```
<cache eviction="LRU" type="com.demo.spring.mybatis.cache.MybatisRedisCache"/>
```

当然在主配置文件里面还需要配置如下代码，代表开启二级缓存，默认是不开启的

```
<setting name="cacheEnabled" value="true" />
```



所以得配置和代码都已经完成了运行结果如下：



![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)



为什么第二次走的是一级缓存呢？

这个在讲二级缓存源码的时候分析过，只有当执行commit的时候才把之前查询的结果放入缓存。

打开吗redis查看如下，因为存入的是序列化的结果，不过我们隐约还是能看到一些信息到下图

![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE1MAo4QJZaibujVS3kGTEAOwDZkkt6ZeXSPJarPmxiaQaeoduZicNjnSjITTwTNf9su4PdU0iclCVHhHg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 二：官方的实例

 地址: http://www.mybatis.org/redis-cache/

 其实和我们自定义的差不多的，不过使用的时候需要注意redis客户端版本要和它的版本兼容，否则或报错。

 mybatis-redis 1.0.0-beta2 对应负redis.clients » jedis 2.8.0  目前的最高版本2.9.0不支持



# Mybatis插件原理和PageHelper结合实战分页插件

## 一、Plugin接口



mybatis定义了一个插件接口org.apache.ibatis.plugin.Interceptor，任何自定义插件都需要实现这个接口PageHelper就实现了改接口



```
package org.apache.ibatis.plugin;

import java.util.Properties;

/**
 * @author Clinton Begin
 */
public interface Interceptor {

  Object intercept(Invocation invocation) throws Throwable;

  Object plugin(Object target);

  void setProperties(Properties properties);

}
```

1：intercept 拦截器，它将直接覆盖掉你真实拦截对象的方法。

2：plugin方法它是一个生成动态代理对象的方法

3：setProperties它是允许你在使用插件的时候设置参数值。

看下 com.github.pagehelper.PageHelper 分页的实现了那些





```
    /**
     * Mybatis拦截器方法
     *
     * @param invocation 拦截器入参
     * @return 返回执行结果
     * @throws Throwable 抛出异常
     */
    public Object intercept(Invocation invocation) throws Throwable {
        if (autoRuntimeDialect) {
            SqlUtil sqlUtil = getSqlUtil(invocation);
            return sqlUtil.processPage(invocation);
        } else {
            if (autoDialect) {
                initSqlUtil(invocation);
            }
            return sqlUtil.processPage(invocation);
        }
    }
```

这个方法获取了是分页核心代码，重新构建了BoundSql对象下面会详细分析

```
    /**
      * 只拦截Executor
      *
      * @param target
      * @return
      */
     public Object plugin(Object target) {
         if (target instanceof Executor) {
             return Plugin.wrap(target, this);
         } else {
             return target;
         }
     }
```



这个方法是正对Executor进行拦截

```
    /**
     * 设置属性值
     *
     * @param p 属性值
     */
    public void setProperties(Properties p) {
        checkVersion();
        //多数据源时，获取jdbcurl后是否关闭数据源
        String closeConn = p.getProperty("closeConn");
        //解决#97
        if(StringUtil.isNotEmpty(closeConn)){
            this.closeConn = Boolean.parseBoolean(closeConn);
        }
        //初始化SqlUtil的PARAMS
        SqlUtil.setParams(p.getProperty("params"));
        //数据库方言
        String dialect = p.getProperty("dialect");
        String runtimeDialect = p.getProperty("autoRuntimeDialect");
        if (StringUtil.isNotEmpty(runtimeDialect) && runtimeDialect.equalsIgnoreCase("TRUE")) {
            this.autoRuntimeDialect = true;
            this.autoDialect = false;
            this.properties = p;
        } else if (StringUtil.isEmpty(dialect)) {
            autoDialect = true;
            this.properties = p;
        } else {
            autoDialect = false;
            sqlUtil = new SqlUtil(dialect);
            sqlUtil.setProperties(p);
        }
    }
```



基本的属性设置

## 二、Plugin初始化

```
private void pluginElement(XNode parent) throws Exception {  
  if (parent != null) {  
    for (XNode child : parent.getChildren()) {  
      String interceptor = child.getStringAttribute("interceptor");  
      Properties properties = child.getChildrenAsProperties();  
      Interceptor interceptorInstance = (Interceptor) resolveClass(interceptor).newInstance();  
      interceptorInstance.setProperties(properties);  
      configuration.addInterceptor(interceptorInstance);  
    }  
  }  
}
```

这里是讲多个实例化的插件对象放入configuration,addInterceptor最终存放到一个list里面的，以为这可以同时存放多个Plugin



## 三、Plugin拦截

### 可拦截 ParameterHandler、ResultSetHandler、StatementHandler、Executor

插件可以拦截mybatis的4大对象 ParameterHandler、ResultSetHandler、StatementHandler、Executor，源码如下图

在Configuration类里面可以找到



newExecutor方法

```
executor = (Executor) interceptorChain.pluginAll(executor);
```

这个是生产一个代理对象，生产了代理对象就运行带invoke方法



## 四、Plugin运行

mybatis自己带了Plugin方法，源码如下

```
public class Plugin implements InvocationHandler {
  private Object target;
  private Interceptor interceptor;
  private Map<Class<?>, Set<Method>> signatureMap;
  private Plugin(Object target, Interceptor interceptor, Map<Class<?>, Set<Method>> signatureMap) {
    this.target = target;
    this.interceptor = interceptor;
    this.signatureMap = signatureMap;
  }
  public static Object wrap(Object target, Interceptor interceptor) {
    Map<Class<?>, Set<Method>> signatureMap = getSignatureMap(interceptor);
    Class<?> type = target.getClass();
    Class<?>[] interfaces = getAllInterfaces(type, signatureMap);
    if (interfaces.length > 0) {
      return Proxy.newProxyInstance(
          type.getClassLoader(),
          interfaces,
          new Plugin(target, interceptor, signatureMap));
    }
    return target;
  }
  @Override
  public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    try {
      Set<Method> methods = signatureMap.get(method.getDeclaringClass());
      if (methods != null && methods.contains(method)) {
        return interceptor.intercept(new Invocation(target, method, args));
      }
      return method.invoke(target, args);
    } catch (Exception e) {
      throw ExceptionUtil.unwrapThrowable(e);
    }
  }
  private static Map<Class<?>, Set<Method>> getSignatureMap(Interceptor interceptor) {
    Intercepts interceptsAnnotation = interceptor.getClass().getAnnotation(Intercepts.class);
    // issue #251
    if (interceptsAnnotation == null) {
      throw new PluginException("No @Intercepts annotation was found in interceptor " + interceptor.getClass().getName());      
    }
    Signature[] sigs = interceptsAnnotation.value();
    Map<Class<?>, Set<Method>> signatureMap = new HashMap<Class<?>, Set<Method>>();
    for (Signature sig : sigs) {
      Set<Method> methods = signatureMap.get(sig.type());
      if (methods == null) {
        methods = new HashSet<Method>();
        signatureMap.put(sig.type(), methods);
      }
      try {
        Method method = sig.type().getMethod(sig.method(), sig.args());
        methods.add(method);
      } catch (NoSuchMethodException e) {
        throw new PluginException("Could not find method on " + sig.type() + " named " + sig.method() + ". Cause: " + e, e);
      }
    }
    return signatureMap;
  }
  private static Class<?>[] getAllInterfaces(Class<?> type, Map<Class<?>, Set<Method>> signatureMap) {
    Set<Class<?>> interfaces = new HashSet<Class<?>>();
    while (type != null) {
      for (Class<?> c : type.getInterfaces()) {
        if (signatureMap.containsKey(c)) {
          interfaces.add(c);
        }
      }
      type = type.getSuperclass();
    }
    return interfaces.toArray(new Class<?>[interfaces.size()]);
  }
}
```

wrap方法是为了生成一个动态代理类。

invoke方法是代理绑定的方法，该方法首先判定签名类和方法是否存在，如果不存在则直接反射调度被拦截对象的方法，如果存在则调度插件的interceptor方法，这时候会初始化一个Invocation对象

我们在具体看下PageHelper，当执行到invoke后程序将跳转到PageHelper.intercept

 

```
 public Object intercept(Invocation invocation) throws Throwable {
        if (autoRuntimeDialect) {
            SqlUtil sqlUtil = getSqlUtil(invocation);
            return sqlUtil.processPage(invocation);
        } else {
            if (autoDialect) {
                initSqlUtil(invocation);
            }
            return sqlUtil.processPage(invocation);
        }
    }
```



我们在来看sqlUtil.processPage方法



```
 /**
     * Mybatis拦截器方法，这一步嵌套为了在出现异常时也可以清空Threadlocal
     *
     * @param invocation 拦截器入参
     * @return 返回执行结果
     * @throws Throwable 抛出异常
     */
    public Object processPage(Invocation invocation) throws Throwable {
        try {
            Object result = _processPage(invocation);
            return result;
        } finally {
            clearLocalPage();
        }
    }
```



继续跟进



```
   /**
     * Mybatis拦截器方法
     *
     * @param invocation 拦截器入参
     * @return 返回执行结果
     * @throws Throwable 抛出异常
     */
    private Object _processPage(Invocation invocation) throws Throwable {
        final Object[] args = invocation.getArgs();
        Page page = null;
        //支持方法参数时，会先尝试获取Page
        if (supportMethodsArguments) {
            page = getPage(args);
        }
        //分页信息
        RowBounds rowBounds = (RowBounds) args[2];
        //支持方法参数时，如果page == null就说明没有分页条件，不需要分页查询
        if ((supportMethodsArguments && page == null)
                //当不支持分页参数时，判断LocalPage和RowBounds判断是否需要分页
                || (!supportMethodsArguments && SqlUtil.getLocalPage() == null && rowBounds == RowBounds.DEFAULT)) {
            return invocation.proceed();
        } else {
            //不支持分页参数时，page==null，这里需要获取
            if (!supportMethodsArguments && page == null) {
                page = getPage(args);
            }
            return doProcessPage(invocation, page, args);
        }
    }
```

​	

这些都只是分装page方法，真正的核心是doProcessPage



```
  /**
     * Mybatis拦截器方法
     *
     * @param invocation 拦截器入参
     * @return 返回执行结果
     * @throws Throwable 抛出异常
     */
    private Page doProcessPage(Invocation invocation, Page page, Object[] args) throws Throwable {
        //保存RowBounds状态
        RowBounds rowBounds = (RowBounds) args[2];
        //获取原始的ms
        MappedStatement ms = (MappedStatement) args[0];
        //判断并处理为PageSqlSource
        if (!isPageSqlSource(ms)) {
            processMappedStatement(ms);
        }
        //设置当前的parser，后面每次使用前都会set，ThreadLocal的值不会产生不良影响
        ((PageSqlSource)ms.getSqlSource()).setParser(parser);
        try {
            //忽略RowBounds-否则会进行Mybatis自带的内存分页
            args[2] = RowBounds.DEFAULT;
            //如果只进行排序 或 pageSizeZero的判断
            if (isQueryOnly(page)) {
                return doQueryOnly(page, invocation);
            }
            //简单的通过total的值来判断是否进行count查询
            if (page.isCount()) {
                page.setCountSignal(Boolean.TRUE);
                //替换MS
                args[0] = msCountMap.get(ms.getId());
                //查询总数
                Object result = invocation.proceed();
                //还原ms
                args[0] = ms;
                //设置总数
                page.setTotal((Integer) ((List) result).get(0));
                if (page.getTotal() == 0) {
                    return page;
                }
            } else {
                page.setTotal(-1l);
            }
            //pageSize>0的时候执行分页查询，pageSize<=0的时候不执行相当于可能只返回了一个count
            if (page.getPageSize() > 0 &&
                    ((rowBounds == RowBounds.DEFAULT && page.getPageNum() > 0)
                            || rowBounds != RowBounds.DEFAULT)) {
                //将参数中的MappedStatement替换为新的qs
                page.setCountSignal(null);
                BoundSql boundSql = ms.getBoundSql(args[1]);
                args[1] = parser.setPageParameter(ms, args[1], boundSql, page);
                page.setCountSignal(Boolean.FALSE);
                //执行分页查询
                Object result = invocation.proceed();
                //得到处理结果
                page.addAll((List) result);
            }
        } finally {
            ((PageSqlSource)ms.getSqlSource()).removeParser();
        }
        //返回结果
        return page;
    }
```

​	

上面的有两个 Object result = invocation.proceed()执行，第一个是执行统计总条数，第二个是执行执行分页的查询的数据

里面用到了代理。最终第一回返回一个总条数，第二个把分页的数据得到。



## 五：PageHelper使用



以上讲解了Mybatis的插件原理和PageHelper相关的内部实现，下面具体讲讲PageHelper使用

1：先增加maven依赖:

```
<dependency>  
    <groupId>com.github.pagehelper</groupId>  
    <artifactId>pagehelper</artifactId>  
    <version>4.1.6</version>  
</dependency
```



2：配置configuration.xml文件加入如下配置（plugins应该在environments的上面 ）

```
<plugins>
         <!-- PageHelper4.1.6 --> 
        <plugin interceptor="com.github.pagehelper.PageHelper">
            <property name="dialect" value="mysql"/>
            <property name="offsetAsPageNum" value="false"/>
            <property name="rowBoundsWithCount" value="false"/>
            <property name="pageSizeZero" value="true"/>
            <property name="reasonable" value="false"/>
            <property name="supportMethodsArguments" value="false"/>
            <property name="returnPageInfo" value="none"/>
        </plugin>
    </plugins>
```

相关字段说明可以查看SqlUtilConfig源码里面都用说明



注意配置的时候顺序不能乱了否则报错

```
Caused by: org.apache.ibatis.builder.BuilderException: Error creating document instance.  Cause: org.xml.sax.SAXParseException; lineNumber: 57; columnNumber: 17; 元素类型为 "configuration" 的内容必须匹配 "(properties?,settings?,typeAliases?,typeHandlers?,objectFactory?,objectWrapperFactory?,plugins?,environments?,databaseIdProvider?,mappers?)"。
at org.apache.ibatis.parsing.XPathParser.createDocument(XPathParser.java:259)
at org.apache.ibatis.parsing.XPathParser.<init>(XPathParser.java:120)
at org.apache.ibatis.builder.xml.XMLConfigBuilder.<init>(XMLConfigBuilder.java:66)
at org.apache.ibatis.session.SqlSessionFactoryBuilder.build(SqlSessionFactoryBuilder.java:49)
... 2 more
```

意思是配置里面的节点顺序是properties->settings->typeAliases->typeHandlers->objectFactory->objectWrapperFactory->plugins->environments->databaseIdProvider->mappers   plugins应该在environments之前objectWrapperFactory之后  这个顺序不能乱了





3：具体使用

  1：分页

```
  SqlSession sqlSession = sessionFactory.openSession();
   UserMapper userMapper = sqlSession.getMapper(UserMapper.class);
   PageHelper.startPage(1,10,true);  //第一页 每页显示10条
   Page<User> page=userMapper.findUserAll();
```

  2：不分页

```
 PageHelper.startPage(1,-1,true);
```

  3:查询总条数

PageInfo\<User>  info=new PageInfo<>(userMapper.findUserAll());