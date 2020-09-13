--log-slow-queries     (查询日志)
--log-queries-not-using-indexes   (查询未使用索引日志)


## 一、普通索引
这是最基本的索引，它没有任何限制。它有以下几种创建方式：

1.创建索引

代码如下:

CREATE INDEX indexName ON mytable(username(length));

如果是CHAR，VARCHAR类型，length可以小于字段实际长度；如果是BLOB和TEXT类型，必须指定 length，下同。

2.修改表结构

代码如下:

ALTER mytable ADD INDEX [indexName] ON (username(length)) -- 创建表的时候直接指定

CREATE TABLE mytable(   ID INT NOT NULL,    username VARCHAR(16) NOT NULL,   INDEX [indexName] (username(length))   );

-- 删除索引的语法：

DROP INDEX [indexName] ON mytable;

## 二、唯一索引

它与前面的普通索引类似，不同的就是：索引列的值必须唯一，但允许有空值。如果是组合索引，则列值的组合必须唯一。它有以下几种创建方式：

代码如下:

CREATE UNIQUE INDEX indexName ON mytable(username(length)) 
-- 修改表结构 
ALTER mytable ADD UNIQUE [indexName] ON (username(length)) 
-- 创建表的时候直接指定 
CREATE TABLE mytable(   ID INT NOT NULL,    username VARCHAR(16) NOT NULL,   UNIQUE [indexName] (username(length))   );

## 三、主键索引

它是一种特殊的唯一索引，不允许有空值。一般是在建表的时候同时创建主键索引：

代码如下:

CREATE TABLE mytable(   ID INT NOT NULL,    username VARCHAR(16) NOT NULL,   PRIMARY KEY(ID)   );

当然也可以用 ALTER 命令。记住：一个表只能有一个主键。

## 组合索引

为了形象地对比单列索引和组合索引，为表添加多个字段：

代码如下:

CREATE TABLE mytable(   ID INT NOT NULL,    username VARCHAR(16) NOT NULL,   city VARCHAR(50) NOT NULL,   age INT NOT NULL  );

为了进一步榨取MySQL的效率，就要考虑建立组合索引。就是将 name, city, age建到一个索引里：

代码如下:

ALTER TABLE mytable ADD INDEX name_city_age (name(10),city,age);





* MySQL何时使用索引

对一个键码使用>, >=, =, <, <=, IF NULL和BETWEEN
```sql　　
　　SELECT * FROM table_name WHERE key_part1=1 and key_part2 > 5;
　　
　　SELECT * FROM table_name WHERE key_part1 IS NULL;
　　当使用不以通配符开始的LIKE
　　
　　SELECT * FROM table_name WHERE key_part1 LIKE 'jani%'
　　在进行联结时从另一个表中提取行时
　　
　　SELECT * from t1,t2 where t1.col=t2.key_part
　　找出指定索引的MAX()或MIN()值
　　
　　SELECT MIN(key_part2),MAX(key_part2) FROM table_name where key_part1=10
　　一个键码的前缀使用ORDER BY或GROUP BY
　　
　　SELECT * FROM foo ORDER BY key_part1,key_part2,key_part3
　　在所有用在查询中的列是键码的一部分时间
　　
　　SELECT key_part3 FROM table_name WHERE key_part1=1
```
* MySQL何时不使用索引  

如果MySQL能估计出它将可能比扫描整张表还要快时，则不使用索引。例如如果key_part1均匀分布在1和100之间，下列查询中使用索引就不是很好：
```sql
　　SELECT * FROM table_name where key_part1 > 1 and key_part1 < 90
　　如果使用HEAP表且不用=搜索所有键码部分。
　　在HEAP表上使用ORDER BY。
　　如果不是用键码第一部分
　　
　　SELECT * FROM table_name WHERE key_part2=1
　　如果使用以一个通配符开始的LIKE
　　
　　SELECT * FROM table_name WHERE key_part1 LIKE '%jani%'
　　搜索一个索引而在另一个索引上做ORDER BY
　　
　　SELECT * from table_name WHERE key_part1 = # ORDER BY key2
```


