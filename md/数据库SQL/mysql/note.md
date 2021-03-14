



#字符集和校对顺序

> SHOW CHARACTER SET;
>
> SHOW COLLATION
>
> SHOW VALIABLES LIKE 'character'
>
> SHOW VALIABLES LIKE 'collations'




数据库表被用来存储和检索数据。不同的语言和字符集需要以不同
的方式存储和检索。因此，MySQL需要适应不同的字符集（不同的字母
和字符），适应不同的排序和检索数据的方法。
在讨论多种语言和字符集时，将会遇到以下重要术语：
  字符集为字母和符号的集合；
  编码为某个字符集成员的内部表示；
  校对为规定字符如何比较的指令。





实际上，字符集很少是服务器范围（甚至数据库范围）的设置。不同的表，甚至不同的列都可能需要不同的字符集，而且两者都可以在创建表时指定。为了给表指定字符集和校对，可使用带子句的 CREATE TABLE 

> CREATE TABLE xxx(...) DEFAULE CHARACTER SET char_set COLLATE char_collate



# 访问控制

MySQL服务器的安全基础是：用户应该对他们需要的数据具有适当
的访问权，既不能多也不能少。换句话说，用户不能对过多的数据具有
过多的访问权。
考虑以下内容：
  多数用户只需要对表进行读和写，但少数用户甚至需要能创建和
删除表；
  某些用户需要读表，但可能不需要更新表；
  你可能想允许用户添加数据，但不允许他们删除数据；
  某些用户（管理员）可能需要处理用户账号的权限，但多数用户
不需要；
  你可能想让用户通过存储过程访问数据，但不允许他们直接访问
数据；
  你可能想根据用户登录的地点限制对某些功能的访问。



使用MySQL Administrator MySQL Administrator 提供了一个图形用户界面，可用来管理用户及账号权限。

### 用户管理

mysql 数据库有一个名为 user 的表，它包含所有用户账号。 user表有一个名为 user 的列，它存储用户登录名。



#### 创建用户账号
为了创建一个新用户账号，使用 CREATE USER 语句，



> CREATE USER melody IDENTIFIED BY 'p@$$w0rd'

指定散列口令 IDENTIFIED BY 指定的口令为纯文本，MySQL
将在保存到 user 表之前对其进行加密。为了作为散列值指定口
令，使用 IDENTIFIED BY PASSWORD 。



> 使用 GRANT 或 INSERT GRANT 语句（稍后介绍）也可以创建用
> 户账号，但一般来说 CREATE USER 是最清楚和最简单的句子。
> 此外，也可以通过直接插入行到 user 表来增加用户，不过为安
> 全起见，一般不建议这样做。MySQL用来存储用户账号信息
> 的表（以及表模式等）极为重要，对它们的任何毁坏都可能严重地伤害到MySQL服务器。因此，相对于直接处理来说，最好是用标记和函数来处理这些表。

#### 重命名

RENAME USER xxx TO xxx_new

#### 删除

DROP USER xxx

#### 设置访问权限

mysql.user 表

角色 权限 用户

在创建用户账号后，必须接着分配访问权限。新创建的用户账号没有访
问权限。它们能登录MySQL，但不能看到数据，不能执行任何数据库操作。
为看到赋予用户账号的权限，使用 SHOW GRANTS FOR user_name

> GRANT USAGE ON *.* TO 'melody'@'%'
>
>  USAGE 表示根本没有权限
>
> 用户定义为 user@host MySQL的权限用用户名和主机名结
> 合定义。如果不指定主机名，则使用默认的主机名 % （授予用
> 户访问权限而不管主机名）。

##### 设置用户密码有效期

##### 锁定用户



#### GRANT  分配 和 REVOKE  撤销

为设置权限，使用 GRANT 语句。 GRANT 要求你至少给出以下信息：
  要授予的权限；
  被授予访问权限的数据库或表；
  用户名。



> GRANT SELECT  ON shiro.* TO melody
>
> 此 GRANT 允许用户在 shiro.* （ test数据库的所
> 有表）上使用 SELECT 
>
>
>
> GRANT 的反操作为 REVOKE ，用它来撤销特定的权限。
>
> REVOKE SELECT  ON shiro.* TO melody





> GRANT 和 REVOKE 可在几个层次上控制访问权限：
>   整个服务器，使用 GRANT ALL 和 REVOKE ALL；
>   整个数据库，使用 ON database.*；
>   特定的表，使用 ON database.table；
>   特定的列；
>   特定的存储过程。

##### 权限表

> 简化多次授权 可通过列出各权限并用逗号分隔，将多条
> GRANT 语句串在一起

```
表28-1 权限
权 限  说 明
ALL  除GRANT OPTION外的所有权限
ALTER  使用ALTER TABLE
ALTER ROUTINE  使用ALTER PROCEDURE和DROP PROCEDURE
CREATE  使用CREATE TABLE
CREATE ROUTINE  使用CREATE PROCEDURE
CREATE TEMPORARY
TABLES
使用CREATE TEMPORARY TABLE
CREATE USER  使用CREATE USER、DROP USER、RENAME USER和REVOKE
ALL PRIVILEGES
CREATE VIEW  使用CREATE VIEW
DELETE  使用DELETE
DROP  使用DROP TABLE
EXECUTE  使用CALL和存储过程
FILE  使用SELECT INTO OUTFILE和LOAD DATA INFILE
GRANT OPTION  使用GRANT和REVOKE
INDEX  使用CREATE INDEX和DROP INDEX
INSERT  使用INSERT
LOCK TABLES  使用LOCK TABLES
PROCESS  使用SHOW FULL PROCESSLIST
RELOAD  使用FLUSH
REPLICATION CLIENT  服务器位置的访问
REPLICATION SLAVE  由复制从属使用
SELECT  使用SELECT
SHOW DATABASES  使用SHOW DATABASES
SHOW VIEW  使用SHOW CREATE VIEW
SHUTDOWN  使用mysqladmin shutdown（用来关闭MySQL）
SUPER  使用CHANGE MASTER、KILL、LOGS、PURGE、MASTER
和SET GLOBAL。还允许mysqladmin调试登录
UPDATE  使用UPDATE
USAGE  无访问权限

```







### 更改口令(password)

为了更改用户口令，可使用 SET PASSWORD 语句。新口令必须如下加密：

> SET PASSWORD FOR melody = Password('new password');
>
> SET PASSWORD 更新用户口令。新口令必须传递到 Password() 函
> 数进行加密。
>
> SET PASSWORD = Password('new password');
>
> 在不指定用户名时， SET PASSWORD 更新当前登录用户的口令。





# 数据库维护 

  ANALYZE TABLE ，用来检查表键是否正确。 ANALYZE TABLE 返回如
下所示的状态信息：

  CHECK TABLE 用来针对许多问题对表进行检查。在 MyISAM 表上还对
索引进行检查。 CHECK TABLE 支持一系列的用于 MyISAM 表的方式。
CHANGED 检查自最后一次检查以来改动过的表。 EXTENDED 执行最
彻底的检查， FAST 只检查未正常关闭的表， MEDIUM 检查所有被删
除的链接并进行键检验， QUICK 只进行快速扫描。



查看日志文件

  错误日志。它包含启动和关闭问题以及任意关键错误的细节。此
日志通常名为 hostname.err ，位于 data 目录中。此日志名可用
--log-error 命令行选项更改。
  查询日志。它记录所有MySQL活动，在诊断问题时非常有用。此
日志文件可能会很快地变得非常大，因此不应该长期使用它。此
日志通常名为 hostname.log ，位于 data 目录中。此名字可以用
--log 命令行选项更改。
  二进制日志。它记录更新过数据（或者可能更新过数据）的所有
语句。此日志通常名为 hostname-bin ，位于 data 目录内。此名字
可以用 --log-bin 命令行选项更改。注意，这个日志文件是MySQL5中添加的，以前的MySQL版本中使用的是更新日志。
  缓慢查询日志。顾名思义，此日志记录执行缓慢的任何查询。这
个日志在确定数据库何处需要优化很有用。此日志通常名为
hostname-slow.log ， 位 于 data 目 录 中 。 此 名 字 可 以 用
--log-slow-queries 命令行选项更改。
在使用日志时，可用 FLUSH LOGS 语句来刷新和重新开始所有日志文
件。





提供进行性能优化探讨和分析的一个出
发点。
  首先，MySQL（与所有DBMS一样）具有特定的硬件建议。在学
习和研究MySQL时，使用任何旧的计算机作为服务器都可以。但
对用于生产的服务器来说，应该坚持遵循这些硬件建议。
  一般来说，关键的生产DBMS应该运行在自己的专用服务器上。
  MySQL是用一系列的默认设置预先配置的，从这些设置开始通常
是很好的。但过一段时间后你可能需要调整内存分配、缓冲区大
小等。（为查看当前设置，可使用 SHOW VARIABLES; 和 SHOW
STATUS; 。）
  MySQL一个多用户多线程的DBMS，换言之，它经常同时执行多
个任务。如果这些任务中的某一个执行缓慢，则所有请求都会执
行缓慢。如果你遇到显著的性能不良，可使用 SHOW PROCESSLIST
显示所有活动进程（以及它们的线程ID和执行时间）。你还可以用KILL 命令终结某个特定的进程（使用这个命令需要作为管理员登
录）。
  总是有不止一种方法编写同一条 SELECT 语句。应该试验联结、并、
子查询等，找出最佳的方法。
  使用 EXPLAIN 语句让MySQL解释它将如何执行一条 SELECT 语句。
  一般来说，存储过程执行得比一条一条地执行其中的各条MySQL
语句快。
  应该总是使用正确的数据类型。
  决不要检索比需求还要多的数据。换言之，不要用 SELECT * （除
非你真正需要每个列）。
  有的操作（包括 INSERT ）支持一个可选的 DELAYED 关键字，如果
使用它，将把控制立即返回给调用程序，并且一旦有可能就实际
执行该操作。
  在导入数据时，应该关闭自动提交。你可能还想删除索引（包括
FULLTEXT 索引），然后在导入完成后再重建它们。
  必须索引数据库表以改善数据检索的性能。确定索引什么不是一
件微不足道的任务，需要分析使用的 SELECT 语句以找出重复的
WHERE 和 ORDER BY 子句。如果一个简单的 WHERE 子句返回结果所花
的时间太长，则可以断定其中使用的列（或几个列）就是需要索
引的对象。
  你的 SELECT 语句中有一系列复杂的 OR 条件吗？通过使用多条
SELECT 语句和连接它们的 UNION 语句，你能看到极大的性能改
进。
  索引改善数据检索的性能，但损害数据插入、删除和更新的性能。
如果你有一些表，它们收集数据且不经常被搜索，则在有必要之
前不要索引它们。（索引可根据需要添加和删除。）
  LIKE 很慢。一般来说，最好是使用 FULLTEXT 而不是 LIKE 。
  数据库是不断变化的实体。一组优化良好的表一会儿后可能就面
目全非了。由于表的使用和内容的更改，理想的优化和配置也会
改变。
  最重要的规则就是，每条规则在某些条件下都会被打破。









# END