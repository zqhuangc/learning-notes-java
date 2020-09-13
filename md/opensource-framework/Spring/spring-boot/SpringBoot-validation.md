DataSourceBuilder#findType

@transactional
transactionInterceptor

Annocation驱动编程
异步事件驱动Web开发
API驱动


分析源码流程可以从报异常（）出发


### 基本
应用分为两个方面：功能性、非功能性  

* 功能性：系统设计的业务范畴
* 非功能性：安全、性能、监控、数据指标（CPU利用率、网卡使用率）

production-ready  非功能性范畴

外部配置 ：启动参数、配置文件、环境
外部应用 ：servlet、springmvc、web flux、websocket、webservice

#### 构建可执行jar或war

需springboot插件:
```xml
<build>plugins。。。 </build>pom.xml配置
```
>jar规范中，有一个 MANIFEST.MF，有一个Main-Class属性
>api：java.util.jar.Manifest#getAttributes  
>
>SpringBoot1.4才有 BOOT-INF

如果版本是**milestone**，看官网不同版本pom依赖，需添加：

SpringBootTest->webEnvironment
**熟悉实际配置类比熟悉怎样配置好**

--server.port 随即可用端口

* 自动装配模式  

  XXX-AutoConfiguration

### 验证

* Bean validation  
* Apache common validate
* Spring validate 

#### Spring Boot Bean Validator

* Maven 依赖

> 命名规则（since Spring Boot 1.4）SpringBoot 大多数情况采用``starter`(启动器，包含一些自动装配的Spring组件)，官方的命名规则：`spring-boot-starter-{name}`,业界或者民间：`{name}-spring-boot-starter`



> 可 null 则用包装类，否则为基本类型

@Valid

with PID  

apt 技术  jsr305

####  常用验证技术

Spring API方式：Assert 

JVM/Java 断言：assert 表达式

以上方式缺点，耦合了业务逻辑，虽可通过`HandetInterceptor`或者`Filter`做拦截，复杂麻烦，也可通过AOP提高代码可读性，

以上所有方式最大问题，不是统一标准

#### 自定义 Bean Validation

需求：员工卡号校验，由前缀和后缀验证

实现：

1. 复制成熟的Bean Validation Annocation的模式（由@NotNull查看）
2. 参考理解`@Constraint`
3. 实现`ConstraintValidator`接口
4. 将`ConstraintValidator`接口，@Constraint#validateby
5. 



StringUtils.delimitedListToStringArray

StringTokenizer类似枚举

String#split 使用正则，NPE保护不够，有时需转义

apache-commons-lang3  各种基础utils有处理null的情况

####  Apache common validator

#### 问题

Json校验，尝试转成bean  

动态表单校验，表单字段与Form对象绑定，一个接一个校验责任链模式





# 验证

## Apache commons-validator

功能特性	
可配置的校验引擎
可重用的原生校验手段
第三方依赖
commons-beanutils:1.9.2
commons-digester:1.8.1
commons-logging:1.2
commons-collections:3.2.2
设计模式
单例模式（GoF23）
校验器模式

### 验证器类型	

Date 与 Time 校验器
数值校验器
正则表达式校验器
ISBN校验器
IP 地址校验器
邮件地址校验器
URL 校验器
域名校验器

- Date 与 Time 校验器
  org.apache.commons.validator.routines.DateValidator
  org.apache.commons.validator.routines.CalendarValidator
  org.apache.commons.validator.routines.TimeValidator
- 使用场景
  校验
  格式化
  时区
  比较
- 数值校验器（Numberic）
  org.apache.commons.validator.routines.ByteValidator
  org.apache.commons.validator.routines.ShortValidator
  org.apache.commons.validator.routines.IntegerValidator
  org.apache.commons.validator.routines.LongValidator
  org.apache.commons.validator.routines.FloatValidator
  org.apache.commons.validator.routines.DoubleValidator
  org.apache.commons.validator.routines.BigIntegerValidator
  org.apache.commons.validator.routines.BigDecimalValidator
  org.apache.commons.validator.routines.CurrencyValidator
- 其他校验器
  org.apache.commons.validator.routines.RegexValidator
  org.apache.commons.validator.routines.CodeValidator
  org.apache.commons.validator.routines.ISBNValidator
  org.apache.commons.validator.routines.InetAddressValidator
  org.apache.commons.validator.routines.EmailValidator
  org.apache.commons.validator.routines.UrlValidator
  org.apache.commons.validator.routines.DomainNameValidator



## Spring Validator

Spring Framework 提供了用于校验对象的Validator 接口，在校验过程中，与 Errors 对象配合。校验器可以通过Errors 对象报告校验失败的信息。

### 实现模式

1. 实现 org.springframework.validation.Validator接口
2. 实现 supports 方法判断是否需要支持校验
3. 当方法返回 false时，视作不予校验
4. 实现 validate 方法，判断 target 参数是否校验合法
5. 当校验非法时，通过Errors 对象返回
6. 实现 MessageCodesResolver 或重用框架实现，完成错误信息编码化

 辅助
使用工具类 ValidationUtils ， 辅助通用校验逻辑

## Bean Validation 1.0（JSR-303）

Java API for JavaBean validation in Java EE and Java SE. The technical objective of this work is to provide a class level constraint declaration and validation facility for the Java application developer, as well as a constraint metadata repository and query API

规范版本
2009.10.12 Bean Validation 1.0（JSR-303）
2013.04.10 Bean Validation 1.1（JSR-349）
2017.06.21 Bean Validation 2.0.0.CR1（JSR-380）

常用注解
@Valid
@NotNull
@Null
@Size
@Min
@Max

## Validation Spring Boot 整合

ValidationAutoConfiguration





文本和时区 国际化

group 分组选择性校验

自定义 ConstraintValidator<自定义 annotation, 被注解属性类型>

参考已有实现

LocaleContextHolder   国际化