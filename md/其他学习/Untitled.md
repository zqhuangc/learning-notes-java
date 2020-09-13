传统MVC：模型器，控制器，视图器

映射器，  BeanNameUrlHanderMapping(可省)，SimpleUrlHanderMapping
适配器（实现接口），SimpleController(可省)
控制器（继承类），AbstractCommandController
视图器    InternalResourceViewResolver  (若为真实路径可省)
Action  需注册到SpringIOC容器中，不可省 ,
​      bean在符合访问路径时一次性全创建 

ParameterizableViewController

struts2框架有如下特点：
​    每次请求action时，都创建action实例
​    action类一成不变的直接或间接继续ActionSupport类
 action类中的业务控制方法总是相类似的签名且无参
​    action类中，接收参数要用实例变量和对应的set方法或set/get方法
struts.xml配置文件，必须以struts.xml命名，且放在src目录下


留心图                   所有框架都只提供对POST请求中文乱码的解决

springmvc与struts2的区别
1）springmvc的入口是一个servlet，即前端控制器，例如：*.action
   struts2入口是一个filter过虑器，即前端过滤器，例如：/*
2）springmvc是基于方法开发，传递参数是通过方法形参，可以设计为单例
   struts2是基于类开发，传递参数是通过类的属性，只能设计为多例
3）springmvc通过参数解析器是将request对象内容进行解析成方法形参，将响应数据和页面封装成
   ModelAndView对象，最后又将模型数据通过request对象传输到页面
struts采用值栈存储请求和响应的数据，通过OGNL存取数据











Spring           classpath:appcontext.xml
Spring框架
2.1 专业术语了解
组件/框架设计
侵入式设计
​		引入了框架，对现有的类的结构有影响；即需要实现或继承某些特定类。
​		例如：	Struts框架
非侵入式设计
​	引入了框架，对现有的类结构没有影响。
​	例如：Hibernate框架 / Spring框架
控制反转:
​	Inversion on Control , 控制反转 IOC
​	对象的创建交给外部容器完成，这个就做控制反转.
​	
	依赖注入，  dependency injection 
		处理对象的依赖关系
	
	区别：
 控制反转， 解决对象创建的问题 【对象创建交给别人】
​	依赖注入，
​		在创建完对象后， 对象的关系的处理就是依赖注入 【通过set方法依赖注入】
AOP
​	面向切面编程。切面，简单来说来可以理解为一个类，由很多重复代码形成的类。
​	切面举例：事务、日志、权限;


2.2 Spring框架
a. 概述
Spring框架，可以解决对象创建以及对象之间依赖关系的一种框架。
​			且可以和其他框架一起使用；Spring与Struts,  Spring与hibernate
​			(起到整合（粘合）作用的一个框架)
Spring提供了一站式解决方案：
​	1） Spring Core  spring的核心功能： IOC容器, 解决对象创建及依赖关系
​	2） Spring Web  Spring对web模块的支持。
​						- 可以与struts整合,让struts的action创建交给spring
​					    - spring mvc模式
​	3） Spring DAO  Spring 对jdbc操作的支持  【JdbcTemplate模板工具类】
​	4） Spring ORM  spring对orm的支持： 
​						 既可以与hibernate整合，【session】
​						 也可以使用spring的对hibernate操作的封装
​	5）Spring AOP  切面编程
​	6）SpringEE   spring 对javaEE其他模块的支持
b. 开发步骤
spring各个版本中：
​	在3.0以下的版本，源码有spring中相关的所有包【spring功能 + 依赖包】
​		如2.5版本；
​	在3.0以上的版本，源码中只有spring的核心功能包【没有依赖包】
​		(如果要用依赖包，需要单独下载！)


1） 源码, jar文件：spring-framework-3.2.5.RELEASE
​	commons-logging-1.1.3.jar           日志
spring-beans-3.2.5.RELEASE.jar        bean节点
spring-context-3.2.5.RELEASE.jar       spring上下文节点
spring-core-3.2.5.RELEASE.jar         spring核心功能
spring-expression-3.2.5.RELEASE.jar    spring表达式相关表

以上是必须引入的5个jar文件，在项目中可以用户库管理！

2） 核心配置文件: applicationContext.xml  
​	Spring配置文件：applicationContext.xml / bean.xml
​	
	约束参考：
spring-framework-3.2.5.RELEASE\docs\spring-framework-reference\htmlsingle\index.html

参考：
id 和 name 的命名规范不是很严格。

2.id的时候用分号（“；”）、空格（“ ”）或逗号（“，”）分隔开就只能当成一个标识，
name的时候用分号（“；”）、空格（“ ”）或逗号（“，”）分隔开就要当成分开来的多个标识
（相当于别名 alias 的作用）。

对象依赖关系
Spring中，如何给对象的属性赋值?  【DI, 依赖注入】
​	1) 通过构造函数
​	2) 通过set方法给属性注入值
​	3) p名称空间
​	4) 自动装配(了解)
​	5) 注解





```sql
CREATE DATABASE webdb;
CREATE TABLE emp(
	eid INT(10) PRIMARY KEY AUTO_INCREMENT,
	ename VARCHAR(20),
	esal DOUBLE(8,2)
)
DESC emp;
SELECT eid,ename,esal FROM emp WHERE eid=1;


-- 创建存储过程
DELIMITER $       -- 声明存储过程的结束符
CREATE PROCEDURE pro_test()   -- 存储过程名称(参数列表)         
BEGIN             -- 开始
	-- 可以写多个sql语句;          -- sql语句+流程控制
	SELECT * FROM emp;
END $            -- 结束 结束符

-- 执行存储过程
CALL pro_test();          -- CALL 存储过程名称(参数);

-- 创建存储过程
DELIMITER $       -- 声明存储过程的结束符
CREATE PROCEDURE get_id(IN id VARCHAR(20),OUT sel VARCHAR(20))   -- 存储过程名称(参数列表)         
BEGIN             -- 开始
	-- 可以写多个sql语句          -- sql语句+流程控制
	SET sel='1';
	INSERT INTO emp(ename) VALUES(id);	
END $            -- 结束 结束符
DROP PROCEDURE get_id;
```

