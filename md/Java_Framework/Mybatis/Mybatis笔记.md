https://mp.weixin.qq.com/s?src=11&timestamp=1543481088&ver=1273&signature=5XlJMSusYFcDNLxnun3hkB910oJD2qXIOdtxJ16Eq2l3DLCNo-pNgIQDIUDgjAeXA2mViQEuqGQi6h6DTeCmOl3j-k6RsEoQ*S9CPD6K*xNt2E0uCepo95GvLfXppKIj&new=1
## $ 和 \#
**$符是直接拼成sql的 ，#符则会以字符串的形式 与sql进行拼接**
目前来看，能用#就不要用$,  

使用#传入参数是，sql语句解析是会加上""  
\#{}传参能防止sql注入，如果你传入的参数为 单引号'，那么如果使用${},这种方式 那么是会报错的。
anything' OR 'x'='x ????

### mybatis中的#和$的区别
1. \#将传入的数据都当成一个字符串，会对自动传入的数据加一个双引号。(使用了#即可在Mybatis中对参数进行转义)
2. $将传入的数据直接显示生成在sql中。
3. \#方式能够很大程度防止sql注入。
4. $方式无法防止Sql注入。
5. $方式一般用于传入数据库对象，例如传入表名.　
6. 一般能用#的就别用$。MyBatis排序时使用order by 动态参数时需要注意，用$而不是#


字符串替换
默认情况下，使用#{}格式的语法会导致MyBatis创建预处理语句属性并以它为背景设置安全的值（比如?）。这样做很安全，很迅速也是首选做法，有时你只是想直接在SQL语句中插入一个不改变的字符串。比如，像ORDER BY，你可以这样来使用：
ORDER BY ${columnName}
这里MyBatis不会修改或转义字符串。
重要：接受从用户输出的内容并提供给语句中不变的字符串，这样做是不安全的。这会导致潜在的SQL注入攻击，因此你不应该允许用户输入这些字段，或者通常自行转义并检查。


Mybatis的执行流程主要部件，SqlSession 提供给用户操作的Api，Executor 具体执行对数据库的操作，但其实在Executor内部还会再委托给StatementHandler这个接口。


### mybatis代理
接口   invocationhandler  mapperProxy

与动态代理相比少了实现类

## 使用笔记
example 的使用方便，索引建立问题？？
JdbcType的作用

```xml
<!-- 一对一 -->
<!--  嵌套查询   
resultMap  association property column select，两次查询 -->
<mapper nasmespace= "com.melody.dao">
  <resultMap id="" type="">
    <id column="" jdbcType="" property=""/>
    <result column="" jdbcType="" property="">
    <association property="" column="" select="">
    </association>
  </resultMap>
      
<!-- 嵌套结果 resultMap  
association jdbcType  -->
  <resultMap id="" type="">
    <id column="" jdbcType="" property=""/>
    <result column="" jdbcType="" property=""/>
    <association property="" jdbcType="" >
       <id column="" jdbcType="" property=""/>
       <result column="" jdbcType="" property="">  
    </association>
  </resultMap>

<!-- 一对多 -->
<!--  嵌套查询   
resultMap  association property column select，两次查询 -->
<mapper nasmespace= "com.melody.dao">
  <resultMap id="" type="">
    <id column="" jdbcType="" property=""/>
    <result column="" jdbcType="" property="">
    <collection property="" column="" select="" ofType="">
    </collection>
  </resultMap>
</mapper>
```
@Data

N+1问题：只想查询一次，但关联查询多了N次
@ignore

* 懒加载  
factory.getConfiguration().setLazyLoadingEnable(true);
factory.getConfiguration().setAggressiveLazyLoading(false);
cglib
NestedQuery
分页插件pagehelper count(0)问题







