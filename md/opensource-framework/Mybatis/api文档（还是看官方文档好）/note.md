## API文档
链接：
http://www.mybatis.org/mybatis-3/zh/sqlmap-xml.html#Parameters

关键参数：
new RowBounds(offset ，limit) 限制查询返回偏移量，行数    主要用于分页

> SqlSessionFactory#getConfigutration#set*

#### 弃用
曾经的 SQL Builder Class
new SQL()
.INSERT_INTO("PERSON")
.INTO_COLUMNS("ID", "FULL_NAME")
.INTO_VALUES("#{id}", "#{fullName}")
.toString();

#### 加载顺序.
如果某个属性存在于多个这些位置，MyBatis将按以下顺序加载它们。

首先读取properties元素主体中指定的属性，
从classpath资源加载的属性或properties元素的url属性将被读取，并覆盖已指定的任何重复属性，
作为方法参数传递的属性最后读取，并覆盖可能已从属性主体和资源/ url属性加载的任何重复属性。
因此，优先级最高的属性是作为方法参数传入的属性，后跟资源/ url属性，最后是属性元素主体中指定的属性。


#### 日志
MyBatis通过使用内部日志工厂提供日志记录信息。内部日志工厂将日志记录信息委派给以下日志实现之一：
SLF4J
Apache Commons Logging
Log4j2
Log4j
JDK日志记录  

选择的日志记录解决方案基于内部MyBatis日志工厂的运行时内省。MyBatis日志工厂将使用它找到的第一个日志记录实现（按上面的顺序搜索实现）。如果MyBatis没有找到上述任何实现，则将禁用日志记录。

许多环境将Commons Logging作为应用程序服务器类路径的一部分提供（很好的例子包括Tomcat和WebSphere）。重要的是要知道在这样的环境中，MyBatis将使用Commons Logging作为日志记录实现。在像WebSphere这样的环境中，这将意味着您的Log4J配置将被忽略，因为WebSphere提供了自己的Commons Logging专有实现。这可能非常令人沮丧，因为看起来MyBatis会忽略你的Log4J配置（实际上，MyBatis忽略了你的Log4J配置，因为MyBatis将在这样的环境中使用Commons Logging）。

```xml
<configuration> <settings> 
    ... <setting name = “logImpl” value = “LOG4J” /> 
    ... </ settings> </ configuration>
```

##### 日志配置
```shell
log4j.properties

# Global logging configuration
log4j.rootLogger=ERROR, stdout

# MyBatis logging configuration...
log4j.logger.org.mybatis.example.BlogMapper(.method_name)=TRACE  ?????#可指定记录范围

# Console output...
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%5p [%t] - %m%n
```





# Mybatis
全局XML配置文件
​	MyBatis 全局XML配置文件包含影响MyBatis行为的设置和属性。

SQL Mapper XML配置文件
​	SQL Mapper XML 配置用于映射SQL 模板语句与Java类型的配置。

SQL Mapper Annotation
SQL Mapper Annotation是Java Annotation的方式替代SQL Mapper XML配置文件。

全局XML配置文件
属性（properties）
设置（settings）
类型别名（typeAliases）
类型处理器（typeHanders）
对象工厂（objectFactory）
插件（plugins）
环境（environments）
数据库标识提供商（databaseIdProvider）
SQL映射文件（mappers）


类型处理器（typeHanders）
用于将预编译语句（PreparedStatement）或结果集（ResultSet）中的JDBC类型转化成Java 类型。
如：BooleanTypeHandler  将 JDBC类型中的BOOLEAN转化成Java类型中的java.lang.Boolean 或者 boolean。
若需要转换 Java 8 新增的Date与Time API，即JSR-310，需要再引入mybatis-typehandlers-jsr310：
<dependency>
 	<groupId>org.mybatis</groupId>
 	<artifactId>mybatis-typehandlers-jsr310</artifactId>
​	 <version>1.0.2</version>
</dependency>

插件（plugins）
Mybatis提供插件的方式来拦截映射语句（mapped statement）的执行，如以下方法：
Executor (update, query, flushStatements, commit, rollback, getTransaction, close, isClosed)
ParameterHandler (getParameterObject, setParameters)
ResultSetHandler (handleResultSets, handleOutputParameters)
StatementHandler (prepare, parameterize, batch, update, query)

MyBatis 配置
XML定义
文档类型约束方式
DTD：Document Type Definition
http://mybatis.org/dtd/mybatis-3-config.dtd
子元素
properties, settings, typeAliases, typeHandlers, objectFactory, objectWrapperFactory, reflectorFactory, plugins, environments, databaseIdProvider, mappers
API接口
org.apache.ibatis.session.Configuration

配置内容

组装API接口
org.apache.ibatis.session.Configuration#variables
填充配置
org.apache.ibatis.builder.xml.XMLConfigBuilder(XPathParser,String, Properties)

设置（settings）
XML声明
<settings> 元素
<setting> 元素
组装API接口
org.apache.ibatis.session.Configuration#setXXX(*)
填充配置
org.apache.ibatis.builder.xml. XMLConfigBuilder#settingsElement(Properties)


类型别名（typeAliases）
XML声明
<typeAliases> 元素
<typeAliase> 元素
组装API接口
org.apache.ibatis.session.Configuration#typeAliasRegistry
API定义
org.apache.ibatis.type.TypeAliasRegistry
填充配置
org.apache.ibatis.builder.xml.XMLConfigBuilder#typeAliasesElement(XNode)

类型处理器（typeHanders）
XML声明
<typeHanders> 元素
<typeHander> 元素
组装API接口
org.apache.ibatis.session.Configuration#typeHandlerRegistry
API定义
org.apache.ibatis.type.TypeHandlerRegistry
填充配置
org.apache.ibatis.builder.xml.XMLConfigBuilder#typeHandlerElement (XNode)