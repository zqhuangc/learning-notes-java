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
