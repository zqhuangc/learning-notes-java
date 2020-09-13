笛卡尔集会在下面条件下产生:
省略连接条件
所有表中的所有行互相连接
为了避免笛卡尔集， 可以在 WHERE 加入有效的连接条件。
在实际运行环境下，应避免使用笛卡尔集。


子查询要包含在括号内。 
将子查询放在比较条件的右侧。
单行操作符对应单行子查询，多行操作符对应多行子查询。
子查询可以出现在select，from，where，having子句中
子查询不可以出现在group by 子句中
主查询和子查询可以使用或不使用一张表示
在from子句后面的子查询最重要（例如oracle分页语句）

多行
IN
ANY
ALL

两个查询的结果，集合运算
UNION（不含重复）/UNION ALL 并集
INTERSECT 交集
MINUS 差集（前-前交后）
集合运算的注意事项
select语句中参数类型和个数要一致。
集合运算采用第一个语句列标题作为最终结果的列标题。


CREATE TABLE [schema.]table
	    (column datatype [DEFAULT expr][, ...]);

表名和列名:
必须以字母开头
必须在 1–30 个字符之间
必须只能包含 A–Z, a–z, 0–9, _, $, 和 #
必须不能和用户定义的其他对象重名
必须不能是Oracle 的保留字
Oracle默认存储是都存为大写

DESCRIBE
#### 使用子查询创建表
使用 AS subquery 选项，将创建表和插入数据结合起来
CREATE TABLE table
  	  [(column, column...)]
AS subquery;

指定的列和子查询中的列要一一对应
通过列名和默认值定义列

#### ALTER TABLE 语句
追加新的列
ALTER TABLE table ADD	(column datatype [DEFAULT expr]
[, column datatype]...);

修改现有的列
ALTER TABLE table MODIFY	   (column datatype [DEFAULT expr]
[, column datatype]...);

删除一个列
ALTER TABLE table DROP column     (column);
重命名
ALTER TABLE table_name rename column old_column_name 
to new_column_name



约束是表一级的限制
如果存在依赖关系，约束可以防止错误的删除数据
约束的类型：
NOT NULL
UNIQUE 
PRIMARY KEY
FOREIGN KEY
CHECK   定义每一行记录所必须满足的条件



FOREIGN KEY: 在子表中，定义了一个表级的约束
REFERENCES: 指定表和父表中的列
ON DELETE CASCADE: 当删除父表时，级联删除子表记录
ON DELETE SET NULL: 将子表的相关依赖记录的外键值置为null
例如：
constraint cid_FK foreign key(cid) references customers(id) 
on delete set null  


#### 从其它表中拷贝数据

在 INSERT 语句中加入子查询。
INSERT INTO sales_reps(id, name, salary, commission_pct)
SELECT employee_id, last_name, salary, commission_pct
FROM   employees
WHERE  job_id LIKE '%REP%';

不必书写 VALUES 子句。 
子查询中的值列表应与 INSERT 子句中的列名对应

#### Delete和Truncate
都是删除表中的数据
delete逐行删除，        truncate销毁表，再创建
delete不会释放空间， truncate会释放空间
delete会产生碎片，     truncate不会产生碎片
delete可回滚，            truncate不可回滚
delete项目中运用较多



数据库事务由以下的部分组成:
一个或多个DML 语句
一个 DDL(Data Definition Language – 数据定义语言) 语句
一个 DCL(Data Control Language – 数据控制语言) 语句

显示回滚 rollback
显示提交:commit
隐式提交(自动提交):DDL语言,DCL语言, exit(事务正常退出)
隐式回滚(系统异常终了):关闭窗口，死机，掉电

回滚到保留点
使用 SAVEPOINT 语句在当前事务中创建保存点。
使用 ROLLBACK TO SAVEPOINT 语句回滚到创建的保存点。

例如：
设置保留点：savepoint day01;
回滚到保留点day01：rollback to savepoint day01;

事务进程
自动提交在以下情况中执行:
DDL 语句。
DCL 语句。
正常结束会话exit。
会话异常结束或系统异常会导致自动回滚。

提交或回滚前的数据状态
改变前的数据状态是可以恢复的
其他用户不能看到当前用户所做的改变，直到当前用户结束事务。
DML语句所涉及到的行被锁定， 其他用户不能操作。


提交后的数据状态
数据的改变已经被保存到数据库中。
改变前的数据已经丢失。
所有用户可以看到结果。
锁被释放， 其他用户可以操作涉及到的数据。
所有保存点被释放。

回滚后的状态
使用 ROLLBACK 语句可使数据变化失效:
数据改变被取消。
修改前的数据状态被恢复。
锁被释放。

### 常见的数据库对象
```sql
对象  描述
表    基本的数据存储集合，由行和列组成。
视图  从表中抽出的逻辑上相关的数据集合。

序列  提供有规律的数值。
索引  提高查询的效率
同义词 	给对象起别名
```


#### 视图
创建视图
使用下面的语法格式创建视图
CREATE [OR REPLACE] VIEW view
[(alias[, alias]...)]
AS subquery
[WITH READ ONLY];

WITH READ ONLY:该视图只能做查询操作
子查询可以是简单或复杂的 SELECT 语句
···
CREATE VIEW 	empvu80
 AS SELECT  employee_id, last_name, salary
    FROM    employees
    WHERE   department_id = 80;

···
修改视图  
使用CREATE OR REPLACE VIEW 子句修改视图
CREATE VIEW 子句中各列的别名应和子查询中各列相对应

创建复杂视图
CREATE VIEW	dept_sum_vu
(name, minsal, maxsal, avgsal)
AS SELECT	 d.department_name, MIN(e.salary), 
MAX(e.salary),AVG(e.salary)
FROM      employees e, departments d
WHERE     e.department_id = d.department_id 
GROUP BY  d.department_name;


同义词
使用同义词访问相同的对象:
方便访问其它用户的对象
缩短对象名字的长度

CREATE SYNONYM synonym
FOR    object;

## PL/SQL程序结构
declare]
	说明部分;    （变量说明，光标说明，例外说明 〕
begin
	语句序列;   （DML语句/TCL事务控制语句〕… 
[exception]
	例外处理语句;  
end;
/

#### IF语句
IF   条件  THEN 语句;
   ELSIF  语句  THEN  语句;
  ELSE    语句;
 END   IF;
 #### 循环语句
 WHILE  total  <= 25000  LOOP
.. .
total : = total + salary;
END  LOOP;

Loop
   exit [when 条件成立];
   total:=total+salary;
end loop;

FOR   I   IN   1 . . 3    LOOP
语句序列 ;
END    LOOP ; 


### 光/游标(Cursor)类似于ResultSet
说明光标语法：
CURSOR  光标名  [ (参数名  数据类型[,参数名 数据类型]...)]
      IS  SELECT   语句；
用于存储一个查询返回的多行数据


打开光标：     open c1;    (打开光标执行查询)

取一行光标的值：fetch c1 into pjob; (取一行到变量中)

关闭光标：          close  c1;(关闭光标释放资源)

注意： 上面的pjob必须与emp表中的job列类型一致：
定义：pjob emp.empjob%type;
           exit when c1%notfound


带参数的光标
cursor c2(jobc varchar2)  
  is
   select ename, sal from emp 
   where job=jobc;
   执行语句:
open c2(‘clerk’);

### 例外==异常
例外是程序设计语言提供的一种功能，用来增强程序的健壮性和容错性。

系统定义例外
no_data_found    (没有找到数据)
too_many_rows          (select …into语句匹配多个行) 
zero_Divide   ( 被零除)
value_error     (算术或转换错误)
timeout_on_resource      (在等待资源时发生超时)

在declare节中定义例外   
out_of   exception ;

 在begin节中可行语句中抛出例外  
raise out_of ；

 在exception节处理例外
when out_of then …

## 存储过程
创建存储过程
用CREATE PROCEDURE命令建立存储过程。

语法：
create [or replace] procedure 过程名[(参数列表)]  
as
PLSQL程序体；【begin…end;/】无declare


存储函数
函数（Function）为一命名的存储程序，可带参数，并返回一个计算值。函数和过程的结构类似，但必须有一个RETURN子句，用于返回函数值。函数说明要指定函数名、结果值的类型，以及参数类型等。

建立存储函数的语法：

CREATE [OR REPLACE] FUNCTION 函数名【(参数列表) 】
 RETURN  返回值类型
AS
PLSQL子程序体；【begin…end;/】

函数的调用
declare
	v_sal number;
begin
	v_sal:=queryEmpSalary(7369);
	dbms_output.put_line('salary is:' || v_sal);
end;
/

#### 过程和函数中的in和out
## 触发器类
数据库触发器是一个与表相关联的、存储的PL/SQL程序。每当一个特定的数据操作语句(Insert,update,delete)在指定的表上发出时，Oracle自动地执行触发器中定义的语句序列。

触发器的类型
语句级触发器
在指定的操作语句操作之前或之后执行一次，不管这条语句影响了多少行 。

行级触发器（FOR EACH ROW）
触发语句作用的每一条记录都被触发。在行级触发器中使用:old和:new伪记录变量, 识别值的状态。
raise_application_error(‘-20000’,‘例外原因');

触发器可用于
数据确认（后）  
安全性检查（前）

### 创建触发器
CREATE  [or REPLACE] TRIGGER  触发器名
{BEFORE | AFTER}
{ INSERT | DELETE|-----语句级
	UPDATE OF 列名}----行级
ON  表名
[FOR EACH ROW]
PLSQL 块【declare…begin…end;/】

#### 触发语句与伪记录变量的值
