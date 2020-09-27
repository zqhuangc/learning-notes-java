## [API文档](https://mybatis.org/mybatis-3/zh/sqlmap-xml.html)

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

