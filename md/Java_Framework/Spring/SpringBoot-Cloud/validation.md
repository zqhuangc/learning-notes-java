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

