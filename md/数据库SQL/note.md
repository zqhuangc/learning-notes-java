# 前言
**注意：MySQL命令终止符为分号 (;)**
**创建 MySql 的表时，表名和字段名外面的符号 ` 不是单引号，而是英文输入法状态下的反单引号，也就是键盘左上角 esc 按键下面的那一个 ~ 按键**
如果使用了表的别名，则不能再使用表的真名
连接 n个表,至少需要 n-1个连接条件。 例如：连接三个表，至少需要两个连接条件


mysql -u root -p password;
## 数据库

#### 创建

```sql
create database 数据库名;

CREATE DATABASE IF NOT EXISTS RUNOOB DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
创建数据库，该命令的作用：
 1. 如果数据库不存在则创建，存在则不创建。
 2. 创建RUNOOB数据库，并设定编码集为utf8

collate在sql中是用来定义排序规则的

BINARY  二进制比较，直接使用memcmp()比较
NOCASE  将26个大写字母转换为小写字母后进行与BINARY一样的比较
RTRIM   和BINARY一样，忽略结尾的空格
```

#### 删除
drop databse 数据库名;

#### 选择
use 数据库;

## 数据类型

```sql
# 数值类型
TINYINT   1 字节 (-128，127) (0，255) 小整数值
SMALLINT  2 字节  (-32 768，32 767) (0，65 535)   大整数值
MEDIUMINT 3 字节 (-8 388 608，8 388 607)	(0，16 777 215)	大整数值
INT或INTEGER 4 字节 (-2 147 483 648，2 147 483 647)  (0，4 294 967 295)	大整数值
BIGINT 8 字节 (-9 233 372 036 854 775 808，9 223 372 036 854 775 807)	(0，18 446 744 073 709 551 615)	极大整数值
FLOAT	4 字节	(-3.402 823 466 E+38，-1.175 494 351 E-38)，0，(1.175 494 351 E-38，3.402 823 466 351 E+38)	0，(1.175 494 351 E-38，3.402 823 466 E+38)	单精度
浮点数值
DOUBLE	8 字节	(-1.797 693 134 862 315 7 E+308，-2.225 073 858 507 201 4 E-308)，0，(2.225 073 858 507 201 4 E-308，1.797 693 134 862 315 7 E+308)	0，(2.225 073 858 507 201 4 E-308，1.797 693 134 862 315 7 E+308)	双精度
浮点数值
DECIMAL	对DECIMAL(M,D) ，如果M>D，为M+2否则为D+2	依赖于M和D的值	依赖于M和D的值	小数值


# 日期和时间类型
DATE	3	1000-01-01/9999-12-31	YYYY-MM-DD	日期值
TIME	3	'-838:59:59'/'838:59:59'	HH:MM:SS	时间值或持续时间
YEAR	1	1901/2155	YYYY	年份值
DATETIME	8	1000-01-01 00:00:00/9999-12-31 23:59:59	YYYY-MM-DD HH:MM:SS	混合日期和时间值
TIMESTAMP	4	
1970-01-01 00:00:00/2038
结束时间是第 2147483647 秒，北京时间 2038-1-19 11:14:07，格林尼治时间 2038年1月19日 凌晨 03:14:07
YYYYMMDD HHMMSS	混合日期和时间值，时间戳

# 字符串类型
CHAR	0-255字节	定长字符串
VARCHAR	0-65535 字节	变长字符串
TINYBLOB	0-255字节	不超过 255 个字符的二进制字符串
TINYTEXT	0-255字节	短文本字符串
BLOB	0-65 535字节	二进制形式的长文本数据
TEXT	0-65 535字节	长文本数据
MEDIUMBLOB	0-16 777 215字节	二进制形式的中等长度文本数据
MEDIUMTEXT	0-16 777 215字节	中等长度文本数据
LONGBLOB	0-4 294 967 295字节	二进制形式的极大文本数据
LONGTEXT	0-4 294 967 295字节	极大文本数据
```

## 表
#### 创建
```sql
CREATE TABLE table_name (column_name column_type);

CREATE TABLE IF NOT EXISTS `runoob_tbl`(
   `runoob_id` INT UNSIGNED AUTO_INCREMENT,
   `runoob_title` VARCHAR(100) NOT NULL,
   `runoob_author` VARCHAR(40) NOT NULL,
   `submission_date` DATE,
   PRIMARY KEY ( `runoob_id` )
)ENGINE=InnoDB DEFAULT CHARSET=utf8;
```
#### 删除
```sql
DROP TABLE table_name ;

删除表内数据，用 delete。格式为：
delete from 表名 where 删除条件;
delete from table_name where xxx : 带条件的删除，表结构不变，不管是 innodb 还是 MyISAM 都不会释放磁盘空间;
delete 操作以后，使用 optimize table table_name 会立刻释放磁盘空间

清除表内数据，保存表结构，用 truncate。格式为：
truncate table 表名;
```

### Alter命令
#### 删除，添加或修改表字段
SHOW COLUMNS from 表名;查看表结构
ALTER TABLE 表名  DROP column/ADD column INT [First / After column_c];
First(设定位第一列)， AFTER 字段名（设定位于某个字段之后
#### 修改字段类型及名称
ALTER TABLE 表名 MODIFY column CHAR(10);
ALTER TABLE 表名 CHANGE column_i column_j 类型;
#### 修改字段默认值
ALTER 来修改字段的默认值  
ALTER TABLE testalter_tbl ALTER i SET DEFAULT 1000;  
ALTER TABLE testalter_tbl ALTER i DROP DEFAULT;
#### 修改表名
ALTER TABLE testalter_tbl RENAME TO alter_tbl;
#### 其他修改
```
# 修改存储引擎
ALTER TABLE testalter_tbl ENGINE = MYISAM;
注意：查看数据表类型可以使用 SHOW TABLE STATUS 语句
# 删除外键约束：keyName是外键别名
alter table tableName drop foreign key keyName;
# 修改字段的相对位置：这里name1为想要修改的字段，type1为该字段原来类型，first和after二选一，这应该显而易见，first放在第一位，after放在name2字段后面
alter table tableName modify name1 type1 first|after name2;
```


## 数据处理
### 插入
```sql
INSERT INTO table_name ( field1, field2,...fieldN ) VALUES ( value1, value2,...valueN );
INSERT IGNORE INTO（忽略已存在数据，唯一数据重复不报错，但不新增）
```
### 更新
```sql
UPDATE table_name SET field1=new-value1, field2=new-value2 [WHERE Clause]

当我们需要将字段中的特定字符串批量修改为其他字符串时，可已使用以下操作：
UPDATE table_name SET field=REPLACE(field, 'old-string', 'new-string') 
[WHERE Clause]
```
### 查询
```sql
SELECT column_name,column_name
FROM table_name,table_name...
[WHERE Clause(分句)]
[LIMIT N][ OFFSET M]
```

### 删除
DELETE FROM table_name [WHERE Clause]

### Like和union
```sql
SELECT field1, field2,...fieldN 
FROM table_name
WHERE field1 LIKE condition1 [AND [OR]] filed2 = 'somevalue'
like 匹配/模糊匹配，会与 % 和 _ 结合使用。
'%a'     //以a结尾的数据
'a%'     //以a开头的数据
'%a%'    //含有a的数据
'_a_'    //三位且中间字母是a的
'_a'     //两位且结尾字母是a的
'a_'     //两位且开头字母是a的

# union
UNION 操作符用于连接两个以上的 SELECT 语句的结果组合到一个结果集合中

SELECT expression1, expression2, ... expression_n
FROM tables
[WHERE conditions]
UNION [ALL | DISTINCT]
SELECT expression1, expression2, ... expression_n
FROM tables
[WHERE conditions];

参数:
- expression1, expression2, ... expression_n: 要检索的列。
- tables: 要检索的数据表。
- WHERE conditions: 可选， 检索条件。
- *DISTINCT*: 可选，删除结果集中重复的数据。默认情况下 UNION 操作符已经删除了重复数据，所以 DISTINCT 修饰符对结果没啥影响。
- ALL: 可选，返回所有结果集，包含重复数据。
```

### 排序
```sql
SELECT field1, field2,...fieldN table_name1, table_name2...
ORDER BY field1, [field2...] [ASC [DESC]]
```
### 分组
```sql
SELECT column_name, function(column_name)
FROM table_name
WHERE column_name operator value
GROUP BY column_name;
```
WITH ROLLUP 可以实现在**分组**统计数据**基础上**再进行相同的统计（SUM,AVG,COUNT…）。
例如我们将以上的数据表按名字进行分组，再统计每个人登录的次数：
SELECT name, SUM(singin) as singin_count FROM  employee_tbl GROUP BY name WITH ROLLUP;

coalesce 来设置一个可以取代 NUll 的名称，coalesce 语法：
select coalesce(a,b,c);
参数说明：如果a==null,则选择b；如果b==null,则选择c；如果a!=null,则选择a；如果a b c 都为null ，则返回为null（没意义）。

**分组后**的条件使用 **HAVING** 来限定，WHERE 是对原始数据进行条件限制。几个关键字的使用顺序为 where 、group by 、having、order by 

SELECT name ,sum(*)  FROM employee_tbl WHERE id<>1 GROUP BY name  HAVING sum(*)>5 ORDER BY sum(*) DESC;

# Mysql 连接的使用
在 SELECT, UPDATE 和 DELETE 语句中使用 Mysql 的 JOIN 来联合多表查询。
JOIN (与 ON 一起使用)按照功能大致分为如下三类：
- INNER JOIN（内连接,或等值连接）：获取两个表中字段匹配关系的记录。
- LEFT JOIN（左连接）：获取左表**所有**记录，**即使**右表**没有**对应匹配的记录。
- RIGHT JOIN（右连接）： 与 LEFT JOIN 相反，用于获取右表所有记录，即使左表没有对应匹配的记录。

### MySQL NULL 值处理
MySQL提供了三大运算符:
- IS NULL: 当列的值是 NULL,此运算符返回 true。
- IS NOT NULL: 当列的值不为 NULL, 运算符返回 true。
- <=>: 比较操作符（不同于=运算符），当比较的的两个值为 NULL 时返回 true。

### MySQL 正则表达式
正则模式可应用于 REGEXP 操作符，匹配模式同一般正则表达式
例：
 SELECT name FROM person_tbl WHERE name REGEXP '^st';
```
^	匹配输入字符串的开始位置。如果设置了 RegExp 对象的 Multiline 属性，^ 也匹配 '\n' 或 '\r' 之后的位置。
$	匹配输入字符串的结束位置。如果设置了RegExp 对象的 Multiline 属性，$ 也匹配 '\n' 或 '\r' 之前的位置。
.	匹配除 "\n" 之外的任何单个字符。要匹配包括 '\n' 在内的任何字符，请使用象 '[.\n]' 的模式。
[...]	字符集合。匹配所包含的任意一个字符。例如， '[abc]' 可以匹配 "plain" 中的 'a'。
[^...]	负值字符集合。匹配未包含的任意字符。例如， '[^abc]' 可以匹配 "plain" 中的'p'。
p1|p2|p3	匹配 p1 或 p2 或 p3。例如，'z|food' 能匹配 "z" 或 "food"。'(z|f)ood' 则匹配 "zood" 或 "food"。
*	匹配前面的子表达式零次或多次。例如，zo* 能匹配 "z" 以及 "zoo"。* 等价于{0,}。
+	匹配前面的子表达式一次或多次。例如，'zo+' 能匹配 "zo" 以及 "zoo"，但不能匹配 "z"。+ 等价于 {1,}。
{n}	n 是一个非负整数。匹配确定的 n 次。例如，'o{2}' 不能匹配 "Bob" 中的 'o'，但是能匹配 "food" 中的两个 o。
{n,m}	m 和 n 均为非负整数，其中n <= m。最少匹配 n 次且最多匹配 m 次
```
## 事务
事务控制语句：
- BEGIN或START TRANSACTION；显式地开启一个事务；
- COMMIT；也可以使用COMMIT WORK，不过二者是等价的。COMMIT会提交事务，并使已对数据库进行的所有修改成为永久性的；
- ROLLBACK；有可以使用ROLLBACK WORK，不过二者是等价的。回滚会结束用户的事务，并撤销正在进行的所有未提交的修改；
- SAVEPOINT identifier；SAVEPOINT允许在事务中创建一个保存点，一个事务中可以有多个SAVEPOINT；
- RELEASE SAVEPOINT identifier；删除一个事务的保存点，当没有指定的保存点时，执行该语句会抛出一个异常；
- ROLLBACK TO identifier；把事务回滚到标记点；
- SET TRANSACTION；用来设置事务的隔离级别。InnoDB存储引擎提供事务的隔离级别有READ UNCOMMITTED、  READ COMMITTED、REPEATABLE READ和SERIALIZABLE。

MYSQL 事务处理主要有两种方法：
1. 用 BEGIN, ROLLBACK, COMMIT来实现
- BEGIN 开始一个事务
- ROLLBACK 事务回滚
- COMMIT 事务确认
2. 直接用 SET 来改变 MySQL 的自动提交模式:
- SET AUTOCOMMIT=0 禁止自动提交
- SET AUTOCOMMIT=1 开启自动提交


# 索引
### 普通索引
#### 创建索引
CREATE INDEX indexName ON mytable(username(length)); 
如果是CHAR，VARCHAR类型，length可以小于字段实际长度；如果是BLOB和TEXT类型，必须指定 length。

#### 修改表结构(添加索引)
ALTER table tableName ADD INDEX indexName(columnName)

#### 创建表的时候直接指定
 INDEX [indexName] (username(length))  

#### 删除索引的语法
DROP INDEX [indexName] ON mytable; 

### 唯一索引
创建索引
CREATE UNIQUE INDEX indexName ON mytable(username(length)) 
修改表结构
ALTER table mytable ADD UNIQUE [indexName] (username(length))
创建表的时候直接指定
UNIQUE [indexName] (username(length))  
```
### 使用ALTER 命令添加和删除索引
有四种方式来添加数据表的索引：
ALTER TABLE tbl_name ADD PRIMARY KEY (column_list): 该语句添加一个主键，这意味着索引值必须是唯一的，且不能为NULL。
ALTER TABLE tbl_name ADD UNIQUE index_name (column_list): 这条语句创建索引的值必须是唯一的（除了NULL外，NULL可能会出现多次）。
ALTER TABLE tbl_name ADD INDEX index_name (column_list): 添加普通索引，索引值可出现多次。
ALTER TABLE tbl_name ADD FULLTEXT index_name (column_list):该语句指定了索引为 FULLTEXT ，用于全文索引。
以下实例为在表中添加索引。
mysql> ALTER TABLE testalter_tbl ADD INDEX (c);
你还可以在 ALTER 命令中使用 DROP 子句来删除索引。尝试以下实例删除索引:
mysql> ALTER TABLE testalter_tbl DROP INDEX c;



### 使用 ALTER 命令添加和删除主键
主键只能作用于一个列上，添加主键索引时，你需要确保该主键默认不为空（NOT NULL）。实例如下：
mysql> ALTER TABLE testalter_tbl MODIFY i INT NOT NULL;
mysql> ALTER TABLE testalter_tbl ADD PRIMARY KEY (i);
你也可以使用 ALTER 命令删除主键：
mysql> ALTER TABLE testalter_tbl DROP PRIMARY KEY;
删除主键时只需指定PRIMARY KEY，但在删除索引时，你必须知道索引名。
显示索引信息
你可以使用 SHOW INDEX 命令来列出表中的相关的索引信息。可以通过添加 \G 来格式化输出信息。
尝试以下实例:
mysql> SHOW INDEX FROM table_name; \G
........
```
# 临时表和复制表
### 临时表
CREATE TEMPORARY TABLE 表名(...)
如果你退出当前MySQL会话，再使用 SELECT命令来读取原先创建的临时表数据，那你会发现数据库中没有该表的存在，因为在你退出时该临时表已经被销毁了
### 复制表
如何完整的复制MySQL数据表，步骤如下：
使用 SHOW CREATE TABLE 命令获取创建数据表(CREATE TABLE) 语句，该语句包含了原数据表的结构，索引等。
复制以下命令显示的SQL语句，修改数据表名，并执行SQL语句，通过以上命令 将完全的复制数据表结构。
如果你想复制表的内容，你就可以使用 INSERT INTO ... SELECT 语句来实现
```sql
另一种完整复制表的方法:
CREATE TABLE targetTable LIKE sourceTable;
INSERT INTO targetTable SELECT * FROM sourceTable;
其他:
可以拷贝一个表中其中的一些字段:
CREATE TABLE newadmin AS
(
    SELECT username, password FROM admin
)
可以将新建的表的字段改名:
CREATE TABLE newadmin AS
(  
    SELECT id, username AS uname, password AS pass FROM admin
)
可以拷贝一部分数据:
CREATE TABLE newadmin AS
(
    SELECT * FROM admin WHERE LEFT(username,1) = 's'
)
可以在创建表的同时定义表中的字段信息:
CREATE TABLE newadmin
(
    id INTEGER NOT NULL AUTO_INCREMENT PRIMARY KEY
)
AS
(
    SELECT * FROM admin
)  
Houses Chan
   Houses Chan
  149***684@qq.com
1年前 (2017-10-06)
   刘先生
  m13***663260@163.com
来给大家区分下mysql复制表的两种方式。
第一、只复制表结构到新表
create table 新表 select * from 旧表 where 1=2
或者
create table 新表 like 旧表 
第二、复制表结构及数据到新表
create table新表 select * from 旧表 
```

# MySQL元数据
       命令	        描述
SELECT VERSION( )	服务器版本信息
SELECT DATABASE( )	当前数据库名 (或者返回空)
SELECT USER( )	    当前用户名
SHOW   STATUS	    服务器状态
SHOW   VARIABLES	服务器配置变量
# MySQL 序列使用
AUTO_INCREMENT
#### 获取AUTO_INCREMENT值
在MySQL的客户端中你可以使用 SQL中的LAST_INSERT_ID( ) 函数来获取最后的插入表中的自增列的值。
#### 重置序列
如果你删除了数据表中的多条记录，并希望对剩下数据的AUTO_INCREMENT列进行重新排列，那么你可以通过删除自增的列，然后重新添加来实现。 不过该操作要非常小心，如果在删除的同时又有新记录添加，有可能会出现数据混乱。
#### 设置序列的开始值
ALTER TABLE t AUTO_INCREMENT = 100;

# MySQL重复数据处理
RIMARY KEY（主键） 或者 UNIQUE（唯一） 索引来保证数据的唯一性
统计重复数据COUNT(*)

可以在 SELECT 语句中使用 DISTINCT 关键字来过滤重复数据。

# 导入和导出
### 导出
```sql
SELECT a,b,a+b INTO OUTFILE '/tmp/result.text'
FIELDS TERMINATED BY ',' OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n'
FROM test_table;

SELECT ... INTO OUTFILE 语句有以下属性:
LOAD DATA INFILE是SELECT ... INTO OUTFILE的逆操作，SELECT句法。为了将一个数据库的数据写入一个文件，使用SELECT ... INTO OUTFILE，为了将文件读回数据库，使用LOAD DATA INFILE。
SELECT...INTO OUTFILE 'file_name'形式的SELECT可以把被选择的行写入一个文件中。该文件被创建到服务器主机上，因此您必须拥有FILE权限，才能使用此语法。
输出不能是一个已存在的文件。防止文件数据被篡改。
你需要有一个登陆服务器的账号来检索文件。否则 SELECT ... INTO OUTFILE 不会起任何作用。
在UNIX中，该文件被创建后是可读的，权限由MySQL服务器所拥有。这意味着，虽然你就可以读取该文件，但可能无法将其删除。
```
#### 导出表作为原始数据
mysqldump 是 mysql 用于转存储数据库的实用程序。它主要产生一个 SQL 脚本，其中包含从头重新创建数据库所必需的命令 CREATE TABLE INSERT 等。
使用 mysqldump 导出数据需要使用 --tab 选项来指定导出文件指定的目录，该目标必须是可写的。
mysqldump -u root -p --no-create-info \
--tab=/tmp RUNOOB runoob_tbl
password ******
#### 导出 SQL 格式的数据
mysqldump -u root -p RUNOOB runoob_tbl > dump.txt
password ******

如果你需要导出整个数据库的数据，可以使用以下命令：
mysqldump -u root -p RUNOOB > database_dump.txt
password ******

如果需要备份所有数据库，可以使用以下命令：
$ mysqldump -u root -p --all-databases > database_dump.txt
password ******
#### 将数据表及数据库拷贝至其他主机
 mysql -u root -p database_name < dump.txt
password *****
mysqldump -u root -p database_name \
| mysql -h other-host.com database_name

将指定主机的数据库拷贝到本地
如果你需要将远程服务器的数据拷贝到本地，你也可以在 mysqldump 命令中指定远程服务器的IP、端口及数据库名。
在源主机上执行以下命令，将数据备份到 dump.txt 文件中：
请确保两台服务器是相通的：
mysqldump -h other-host.com -P port -u root -p database_name > dump.txt
password ****

### 导出
#### mysql 命令导入
mysql -u用户名    -p密码    <  要导入的数据库数据(runoob.sql)
#### source 命令导入
mysql> create database abc;      # 创建数据库
mysql> use abc;                  # 使用已创建的数据库 
mysql> set names utf8;           # 设置编码
mysql> source /home/abc/abc.sql  # 导入备份数据库
#### 使用 LOAD DATA 导入数据
LOAD DATA LOCAL INFILE 'dump.txt' INTO TABLE mytbl;
LOCAL关键词，则表明从客户主机上按路径读取文件。如果没有指定，则文件在服务器上按路径读取文件。
#### 使用 mysqlimport 导入数据
mysqlimport客户端提供了LOAD DATA INFILEQL语句的一个命令行接口。mysqlimport的大多数选项直接对应LOAD DATA INFILE子句。
从文件 dump.txt 中将数据导入到 mytbl 数据表中, 可以使用以下命令：
$ mysqlimport -u root -p --local database_name dump.txt
password *****

mysqlimport命令可以指定选项来设置指定格式,命令语句格式如下：
$ mysqlimport -u root -p --local --fields-terminated-by=":" \
   --lines-terminated-by="\r\n"  database_name dump.txt
password *****

mysqlimport 语句中使用 --columns 选项来设置列的顺序：
$ mysqlimport -u root -p --local --columns=b,c,a \
    database_name dump.txt
password *****


# exist 和 not exist