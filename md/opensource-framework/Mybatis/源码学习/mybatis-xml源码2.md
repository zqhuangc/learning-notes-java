## 源码：切入点

> 核心 SqlSession(configuration，executor)  利用构造函数关联起来
> ->  Xml 解析成 Configuration 
> executor 执行 sql           ->      CacheExecutor  
> transaction(容器管理)    ->      Connection
>
> Mapper 接口 与 mapper.xml 怎么映射 ？ 
> 完全限定名称.方法名；类名.方法名
> MapStatement(HashMap，每个方法两条记录)
> XmlConfigureBuilder
> MapperAnnotationBuilder

1. transaction 加在哪？
2. 缓存加在哪？用装饰器模式实现缓存  CacheExecutor

Configuration

## Mapper的动态代理

MapperProxyFactory
MapperProxy 代理类
不建议在Mapper接口中写**默认方法**

**MapperMethod**

## Sqlsession的执行流程
**Mybatis: Sqlsession -> Executor -> StatementHandler -> ResultHandler**

**JDBC: Connection -> Statement -> Result**

SqlCommand  识别 sql 的操作类型
MethodSignature 与 ParamNameResolver
rowCountResult()

* plugins插件

SqlSession下有如下四大对象.

* ParameterHandler: 处理参数设置问题
* ResultHandler: 结果处理
* StatementHandler: 负责连接数据库,执行sql
* Executor: 对上述过程的调度组织.

CachingExecutor 为二级缓存实现,当在 Mybatis Config 中配置了 cacheEnabled 才会用其包裹当前的Executor，利用类似 AOP 环绕通知的方式实现缓存.  
* BaseExecutor 为基本装饰器,其实现了 Executor 接口,并且内部也持有一个 protected Executor wrapper; 被包装对象.  
* SimpleExecutor 为最基本装饰器实现类,提供最基本的增删改查需求.  
* BatchExecutor 在原有基础上增加了对批量处理的支持. 
* ReuseExecutor 在原有基础上增加了复用SQL功能.  


SimpleExecutor 是真正从数据库查询的地方,查询是要经过StatementHandler组织ParameterHandler,ResultHandler 的处理过程   doQuery

## 动态Sql中的参数解析

* 参数输入解析  
  MapperMethod 中使用 ParamNameResolver 对输入参数解析  
  针对多参数的输入,最佳解决方案是用@Param注解,其次为使用Map集合包裹参数,这样的话method.convertArgsToSqlCommandParam(args)得到的则是该Map集合.
* 动态sql渲染解析  
  DefaultSqlSession 
  使用 DynamicContext 构造动态sql所需要的上下文
  ​    1. null，无任何参数传入
  ​    2. Map类型，对于多参数，或者参数本身就是 map再或者输入单一参数集合类型，数组类型都会转换为map
  ​    3. 单一POJO类型
  ​    SqlNode
  ​    ParameterHandler

### Mybatis的SQL解析总体流程如下:

1. 构造 ParamtersMap ，保存输入参数.
2. 构造 ContextMap，为OGNL解析提供数据.
3. 读取 xml，使用 SqlSource 与 SqlNode 解析 xml 中的  
4. sql 设置参数值到 boundSql 的  addtionParameters中，其为 ContextMap 的一个副本
5. 根据boundSql.parameterMappings获取到参数,从 addtionParameters 与 ParamtersMap 中读取参数设置到 PreparedStatement 中
6. 执行sql


TypeHandler 的作用就可以简单的理解为:

* 转换参数到sql中
* 转换查询结果到Java类中



## ResultSetHandler 分析

Mybatis 结果映射有哪几种形式?

简单来说有两种形式，resultMap 与 resultType
resultType 直接对应一个单一的Java类.
resultMap 则对应的是自定义映射规则,并且还可加入 association，collection 等标签完成其他操作.

结果映射是如何构造的  MappedStatement


使用resultMap配置结果映射
```
constructor - 类在实例化时,用来注入结果到构造方法中
idArg - ID 参数;标记结果作为 ID 可以帮助提高整体效能
arg - 注入到构造方法的一个普通结果
id – 一个 ID 结果;标记结果作为 ID 可以帮助提高整体效能
result – 注入到字段或 JavaBean 属性的普通结果
association – 一个复杂的类型关联;许多结果将包成这种类型
嵌入结果映射 – 结果映射自身的关联,或者参考一个
collection – 复杂类型的集
嵌入结果映射 – 结果映射自身的集,或者参考一个
discriminator – 使用结果值来决定使用哪个结果映射
case – 基于某些值的结果映射
嵌入结果映射 – 这种情形结果也映射它本身,因此可以包含很多相同的元素,或者它可以参照一个外部的结果映射。
```
buildResultMappingFromContext()
ResultMap 在 Executor 执行后是怎么对结果处理的?

## TypeHandler 类型转换器

* 通过调用 PrepareStatement不同的 set 方法实现 objectType convert to jdbcType

* 在获取结果返回之后,也需要将返回的结果转换成我们需要的java类型
  通过调用ResultSet对象不同类型的 get 方法实现 jdbcType to objectType

### TypeHandlerRegistry

 注册默认的 TypeHandler





## mybatis缓存机制详解
### 一级缓存
mybatis的一级缓存是SqlSession级别的缓存，在操作数据库的时候需要先创建SqlSession会话对象，在对象中有一个HashMap用于存储缓存数据，此HashMap是当前会话对象私有的，别的SqlSession会话对象无法访问。  
第一次执行select完毕会将查到的数据写入SqlSession内的HashMap中缓存起来

* 注意事项：
1.如果SqlSession执行了DML操作（insert、update、delete），并commit了，那么mybatis就会清空当前SqlSession缓存中的所有缓存数据，这样可以保证缓存中的存的数据永远和数据库中一致，避免出现脏读  
2.当一个SqlSession结束后那么他里面的一级缓存也就不存在了，mybatis默认是开启一级缓存，不需要配置  
3.mybatis的缓存是基于[namespace:sql语句:参数]来进行缓存的，意思就是，SqlSession的HashMap存储缓存数据时，是使用[namespace:sql:参数]作为key，查询返回的语句作为value保存的。例如：-1242243203:1146242777:winclpt.bean.userMapper.getUser:0:2147483647:select * from user where id=?:19

### 二级缓存：
Configuration
二级缓存是mapper级别的缓存，也就是同一个namespace的mappe.xml，当多个SqlSession使用同一个Mapper操作数据库的时候，得到的数据会缓存在同一个二级缓存区域  
二级缓存默认是没有开启的。需要在setting全局参数中配置开启二级缓存
```xml
conf.xml：
<settings>
    <setting name="cacheEnabled" value="true"/>默认是false：关闭二级缓存
<settings>
```
* 问题
1. 脏数据
2. 全部失效





## 其他

https://mp.weixin.qq.com/s/JOZeseLwZutVMocqJjFEZw

XPathParser 是 Mybatis 核心的 xml 解析器
SqlSessionFactoryBuilder 为 Mybatis 入口类

typeHandlers
BaseTypeHandler



### sql执行过程

MapperProxy

SqlSessionTemplate  
委托  
SqlSessionUtils => DefaultSqlSessionFactory.openSession(executorType);//simple 动态代理类DefaultSqlSession，实现SqlSession

CachingExecutor SimpleExecutor（有3种执行器，ReuseExecutor，SimpleExecutor，CachingExecutor）

RoutingStatementHandler => 委托给PreparedStatementHandler，DefaultParameterHandler初始化连接

DefaultResultSetHandler, 执行结果查询, 合并结果集等

handleLocallyCachedOutputParameters，只处理存储过程和函数调用的出参, 因为存储过程和函数的返回不是通过ResultMap而是ParameterMap来的，所以只要把缓存的非IN模式参数取出来设置到parameter对应的属性上即可

mybatis内置了 JNDI、POOLED、UNPOOLED 三种类型的数据源,其中 POOLED 对应的实现为org.apache.ibatis.datasource.pooled.PooledDataSource,它是mybatis自带实现的一个同步、线程安全的数据库连接池, 一般在生产中,我们会使用dbcp或者druid连接池

### spring管理 SqlSession和Mapper的生命周期

MapperScannerConfigurer -> xml  
MapperScannerRegistrar -> 注解
mapperscan

mapper 在Spring管理下为全局 singlton， 作用域原为 method 级  sqlsession 在transaction下为  transaction范围 ，原为request/method



