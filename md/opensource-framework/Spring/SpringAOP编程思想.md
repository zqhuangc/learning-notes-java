

[TOC]



# 小马哥讲 Spring AOP 编程思想课程大纲

## 第一章：Spring AOP总览

### 01 课程介绍



### 02 内容综述



### 03 知识储备：基础、基础，还是基础 !

知识储备
• Java 基础
• Java ClassLoading
• Java 动态代理
• Java 反射
• 字节码框架：ASM、CGLIB

• OOP 概念
• 封装
• 继承
• 多态



• Spring 核心基础
• Spring IoC 容器
• Spring Bean 生命周期（Bean Lifecycle）
• Spring 配置元信息（Configuration Metadata）
• Spring 事件（Events）
• Spring 注解（Annotations）



• GoF23 设计模式
• 创建模式（Creational patterns）
• 抽象工厂模式（Abstract factory）
• 构建器模式（Builder）
• 工厂方法模式（Factory method）
• 原型模式（Prototype）
• 单例模式（Singleton）

• 结构模式（Structural patterns）
• 适配器模式（Adapter）
• 桥接模式（Bridge）
• 组合模式（Composite）
• 装饰器模式（Decorator）
• 门面模式（Facade）
• 享元模式（Flyweight）
• 代理模式（Proxy）

• 行为模式（Behavioral patterns）
• 模板方法模式（Template Method）
• 中继器模式（Mediator）
• 责任链模式（Chain of Responsibility）
• 观察者模式（Observer）
• 策略模式（Strategy）
• 命令模式（Command）
• 状态模式（State）
• 访问者模式（Visitor）
• 解释器模式（Interpreter）、迭代器模式（Iterator）、备忘录模式（Memento）







### 04 AOP引入：OOP存在哪些局限性 ?

AOP 引入

• Java OOP 存在哪些局限性？
• 静态化语言：类结构一旦定义，不容易被修改
• 侵入性扩展：通过继承和组合组织新的类结构



### 05 AOP常见使用场景

AOP 常见使用场景

* 日志场景
  • 诊断上下文，如：log4j 或 logback 中的 _x0008_MDC
  • 辅助信息，如：方法执行时间



* 统计场景
  • 方法调用次数
  • 执行异常次数
  • 数据抽样
  • 数值累加



* 安防场景
  • 熔断，如：Netflix Hystrix
  • 限流和降级：如：Alibaba Sentinel
  • 认证和授权，如：Spring Security
  • 监控，如：JMX



* 性能场景
  • 缓存，如 Spring Cache
  • 超时控制



### 06 AOP概念：Aspect、Join Point 和 Advice 等术语应该如何理解?

• AOP 定义
• AspectJ：Aspect-oriented programming is a way of modularizing crosscutting
concerns much like object-oriented programming is a way of modularizing common
concerns.
• Spring：Aspect-oriented Programming (AOP) complements Object-oriented
Programming (OOP) by providing another way of thinking about program structure.
The key unit of modularity in OOP is the class, whereas in AOP the unit of modularity is
the aspect. Aspects enable the modularization of concerns (such as transaction
management) that cut across multiple types and objects.



• Aspect 概念
• AspectJ：aspect are the unit of modularity for crosscutting concerns. They behave
somewhat like Java classes, but may also include pointcuts, advice and inter-type
declarations.
• Spring：A modularization of a concern that cuts across multiple classes.



• Join point 概念
• AspectJ：A join point is a well-defined point in the program flow. A pointcut picks out
certain join points and values at those points.
• Spring：A point during the execution of a program, such as the execution of a method
or the handling of an exception. In Spring AOP, a join point always represents a
method execution.



• Pointcut 概念
• AspectJ：pointcuts pick out certain join points in the program flow.
• Spring：A predicate that matches join points.



• Advice 概念
• AspectJ：So pointcuts pick out join points. But they don't do anything apart from
picking out join points. To actually implement crosscutting behavior, we use advice.
Advice brings together a pointcut (to pick out join points) and a body of code (to run
at each of those join points).
• Spring：Action taken by an aspect at a particular join point. Different types of advice
include “around”, “before” and “after” advice. Many AOP frameworks,
including Spring, model an advice as an interceptor and maintain a chain of
interceptors around the join point.



• Introduction 概念
• AspectJ：Inter-type declarations in AspectJ are declarations that cut across classes
and their hierarchies. They may declare members that cut across multiple classes, or
change the inheritance relationship between classes.
• Spring：Declaring additional methods or fields on behalf of a type. Spring AOP lets
you introduce new interfaces (and a corresponding implementation) to any advised
object.



### 07 Java AOP 设计模式：代理、判断和拦截器模式

• 代理模式：静态和动态代理
• 判断模式：类、方法、注解、参数、异常...
• 拦截模式：前置、后置、返回、异常



### 08 Java AOP 代理模式 (Proxy) ：Java 静态代理和动态代理的区别是什么?

• Java 静态代理
• 常用 OOP 继承和组合相结合
• Java 动态代理
• JDK 动态代理
• 字节码提升，如 CGLIB



### 09 Java AOP 判断模式 (Predicate) ：如何筛选 Join Point?

• 判断来源
• 类型（Class）
• 方法（Method）
• 注解（Annotation）
• 参数（Parameter）
• 异常（Exception）



### 10 Java AOP 拦截器模式 (Interceptor) ：拦截执行分别代表什么?

• 拦截类型
• 前置拦截（Before）
• 后置拦截（After）
• 异常拦截（Exception）



### 11 Spring AOP 功能概述：核心特性、编程模型和使用限制

• 核心特性
• 纯 Java 实现、无编译时特殊处理、不修改和控制 ClassLoader
• 仅支持方法级别的 Join Points
• 非完整 AOP 实现框架
• Spring IoC 容器整合
• AspectJ 注解驱动整合（非竞争关系）





### 12 Spring AOP 编程模型：注解驱动、XML 配置驱动和底层 API

• 注解驱动
• 实现：Enable 模块驱动，@EnableAspectJAutoProxy
• 注解：
• 激活 AspectJ 自动代理：@EnableAspectJAutoProxy
• Aspect ： @Aspect
• Pointcut ：@Pointcut
• Advice ：@Before、@AfterReturning、@AfterThrowing、@After、@Around
• Introduction ：@DeclareParents



• XML 配置驱动
• 实现：Spring Extenable XML Authoring
• XML 元素
• 激活 AspectJ 自动代理：`<aop:aspectj-autoproxy/>`
• 配置：`<aop:config/>`
• Aspect ： `<aop:aspect/>`
• Pointcut ：`<aop:pointcut/>`
• Advice ：`<aop:around/>`、`<aop:before/>`、`<aop:after-returning/>`、`<aop:after-throwing/>` 和
`<aop:after/>`
• Introduction ：`<aop:declare-parents/>`
• 代理 Scope ： `<aop:scoped-proxy/>`



• 底层 API
• 实现：JDK 动态代理、CGLIB 以及 AspectJ
• API：
• 代理：AopProxy
• 配置：ProxyConfig
• Join Point：Pointcut
• Pointcut ：Pointcut
• Advice ：Advice、BeforeAdvice、AfterAdvice、AfterReturningAdvice、
ThrowsAdvice



### 13 Spring AOP 设计目标：Spring AOP 与 AOP框架之间的关系是竞争还是互补?

• 整体目标
Spring AOP’s approach to AOP differs from that of most other AOP frameworks. The aim is
not to provide the most complete AOP implementation (although Spring AOP is quite capable).
Rather, the aim is to provide a close integration between AOP implementation and Spring IoC,
to help solve common problems in enterprise applications.

Spring AOP never strives to compete with AspectJ to provide a comprehensive AOP solution.
We believe that both proxy-based frameworks such as Spring AOP and full-blown frameworks
such as AspectJ are valuable and that they are complementary, rather than in
competition.Spring seamlessly integrates Spring AOP and IoC with AspectJ, to enable all uses
of AOP within a consistent Spring-based application architecture. This integration does not
affect the Spring AOP API or the AOP Alliance API. Spring AOP remains backward-compatible.



### 14 Spring AOP Advice 类型：Spring AOP丰富了哪些 AOP Advice呢?

• Advice 类型
• 环绕（Around）
• 前置（Before）
• 后置（After）
• 方法执行
• finally 执行
• 异常（Exception）



### 15 Spring AOP代理实现：为什么 Spring Framework 选择三种不同 AOP 实现?

• JDK 动态代理实现 - 基于接口代理
• CGLIB 动态代理实现 - 基于类代理（字节码提升）
• AspectJ 适配实现



### 16 JDK 动态代理：为什么 Proxy.newProxyInstance 会生成新的字节码?



### 17 CGLIB 动态代理：为什么 Java 动态代理无法满足 AOP 的需要?



### 18 AspectJ 代理代理：为什么 Spring 推荐 AspectJ 注解?



### 19 AspectJ 基础：Aspect、Join Points、Pointcuts 和 Advice 语法和特性



### 20 AspectJ 注解驱动：注解能完全替代 AspectJ 语言吗?



### 21 面试题精选



## 第二章：Spring AOP 基础

### 01 Spring 核心基础：《小马哥讲Spring核心编程思想》还记得多少?

• 《小马哥讲 Spring核心编程思想》
• 第三章：Spring IoC 容器概述
• 第九章：Spring Bean 生命周期（Bean Lifecycle）
• 第十章：Spring 配置元信息（Configuration Metadata）
• 第十八章：Spring 注解（Annotations）
• 第二十章：Spring IoC 容器生命周期（Container Lifecycle）



### 02 @AspectJ 注解驱动

• 激活 @AspectJ 模块
• 注解激活 - @EnableAspectJAutoProxy
• XML 配置 - `<aop:aspectj-autoproxy/>`
• 声明 Aspect
• @Aspect



### 03 编程方式创建 @AspectJ 代理

04 XML配置驱动 — 创建AOP代理

### 05 标准代理工厂 API — ProxyFactory

### 06 @AspectJ Pointcut 指令与表达式：为什么 Spring 只能有限支持?



### 07 XML 配置 Pointcut



### 08 API 实现 Pointcut



### 09 @AspectJ 拦截动作：@Around 与 @Pointcut 有区别吗?



### 10 XML 配置 Around Advice



### 11 API 实现 Around Advice



### 12 @AspectJ 前置动作：@Before 与 @Around 谁优先级执行?



### 13 XML 配置 Before Advice



### 14 API 实现 Before Advice



### 15 @AspectJ 后置动作 — 三种 After Advice 之间的关系?



### 16 XML 配置三种 After Advice



### 17 API 实现三种 After Advice



### 18 自动动态代理



### 19 替换 TargetSource



### 20 面试题精选

沙雕面试题 - Spring AOP 支持哪些类型的 Advice？

答：
• Around Advice
• Before Advice
• After Advice
• After
• AfterReturning
• AfterThrowing



996 面试题 - Spring AOP 编程模型有哪些，代表组件有哪些？

答：
注解驱动：解释和整合 AspectJ 注解，如 @EnableAspectJAutoProxy
XML 配置：AOP 与 IoC Schema-Based 相结合
API 编程：如 Joinpoint、Pointcut、Advice 和 ProxyFactory 等



劝退面试题 - Spring AOP 三种实现方式是如何设计的?



## 第三章：Spring AOP API 设计与实现

### 01 Spring AOP API 整体设计

• Join point - Joinpoint
• Pointcut - Pointcut
• Advice 执行动作 - Advice
• Advice 容器 - Advisor
• Introduction - IntroductionInfo
• 代理对象创建基础类 - ProxyCreatorSupport
• 代理工厂 - ProxyFactory、ProxyFactoryBean
• AopProxyFactory 配置管理器 - AdvisedSupport
• IoC 容器自动代理抽象 - AbstractAutoProxyCreator



### 02 接入点接口 — Joinpoint

• Interceptor 执行上下文 - Invocation
• 方法拦截器执行上下文 - MethodInvocation
• ~~构造器拦截器执行上下文 - ConstructorInvocation~~
• MethodInvocation 实现
• 基于反射 - ReflectiveMethodInvocation
• 基于 CGLIB - CglibMethodInvocation



### 03 Joinpoint 条件接口 — Pointcut

• 核心组件
• 类过滤器 - ClassFilter
• 方法匹配器 - MethodMatcher



### 04 Pointcut 操作 — ComposablePointcut

• 组合实现 - org.springframework.aop.support.ComposablePointcut
• 工具类
• ClassFilter 工具类 - ClassFilters
• MethodMatcher 工具类 - MethodMatchers
• Pointcut 工具类 - Pointcuts



### 05 Pointcut 便利实现

• 静态 Pointcut - StaticMethodMatcherPointcut
• 正则表达式 Pointcut - JdkRegexpMethodPointcut
• 控制流 Pointcut - ControlFlowPointcut



### 06 Pointcut AspectJ 实现 — AspectJExpressionPointcut

• 实现类 - org.springframework.aop.aspectj.AspectJExpressionPointcut
• 指令支持 - SUPPORTED_PRIMITIVES 字段
• 表达式 - org.aspectj.weaver.tools.PointcutExpression



### 07 Join point 执行动作接口 — Advice

• Around Advice - Interceptor
• 方法拦截器 - MethodInterceptor
• 构造器拦截器 - ConstructorInterceptor
• 前置动作
• 标准接口 - org.springframework.aop.BeforeAdvice
• 方法级别 - org.springframework.aop.MethodBeforeAdvice
• 后置动作
• org.springframework.aop.AfterAdvice
• org.springframework.aop.AfterReturningAdvice
• org.springframework.aop.ThrowsAdvice



### 08 Join point Before Advice 标准实现

• 接口
• 标准接口 - org.springframework.aop.BeforeAdvice
• 方法级别 - org.springframework.aop.MethodBeforeAdvice
• 实现
• org.springframework.aop.framework.adapter.MethodBeforeAdviceInterceptor



### 09 Join point Before Advice AspectJ 实现

• 实现类 - org.springframework.aop.aspectj.AspectJMethodBeforeAdvice



### 10 Join point After Advice 标准实现

• 接口
• org.springframework.aop.AfterAdvice
• org.springframework.aop.AfterReturningAdvice
• org.springframework.aop.ThrowsAdvice
• 实现
• org.springframework.aop.framework.adapter.ThrowsAdviceInterceptor
• org.springframework.aop.framework.adapter.AfterReturningAdviceInterceptor



### 11 Join point After Advice AspectJ 实现

• 接口
• org.springframework.aop.AfterAdvice
• org.springframework.aop.AfterReturningAdvice
• org.springframework.aop.ThrowsAdvice
• 实现
• org.springframework.aop.aspectj.AspectJAfterAdvice
• org.springframework.aop.aspectj.AspectJAfterReturningAdvice
• org.springframework.aop.aspectj.AspectJAfterThrowingAdvice



### 12 Advice 容器接口 — Advisor

• 接口 - org.springframework.aop.Advisor
• 通用实现 - org.springframework.aop.support.DefaultPointcutAdvisor



### 13 Pointcut 与 Advice 连接器 — PointcutAdvisor

• 接口 - org.springframework.aop.PointcutAdvisor
• 通用实现
• org.springframework.aop.support.DefaultPointcutAdvisor
• AspectJ 实现
• org.springframework.aop.aspectj.AspectJExpressionPointcutAdvisor
• org.springframework.aop.aspectj.AspectJPointcutAdvisor
• 静态方法实现
• org.springframework.aop.support.StaticMethodMatcherPointcutAdvisor
• IoC 容器实现
• org.springframework.aop.support.AbstractBeanFactoryPointcutAdvisor



### 14 Introduction 与 Advice 连接器 — IntroductionAdvisor

• 接口 - org.springframework.aop.IntroductionAdvisor
• 元信息
• org.springframework.aop.IntroductionInfo
• 通用实现
• org.springframework.aop.support.DefaultIntroductionAdvisor
• AspectJ 实现
• org.springframework.aop.aspectj.DeclareParentsAdvisor



### 15 Advisor 的 Interceptor 适配器 — AdvisorAdapter

• 接口 - org.springframework.aop.framework.adapter.AdvisorAdapter
• MethodBeforeAdvice 实现
• org.springframework.aop.framework.adapter.MethodBeforeAdviceAdapter
• AfterReturningAdvice 实现
• org.springframework.aop.framework.adapter.AfterReturningAdviceAdapter
• ThrowsAdvice 实现
• org.springframework.aop.framework.adapter.ThrowsAdviceAdapter



### 16 AdvisorAdapter 实现

• MethodBeforeAdvice 实现
• org.springframework.aop.framework.adapter.MethodBeforeAdviceAdapter
• AfterReturningAdvice 实现
• org.springframework.aop.framework.adapter.AfterReturningAdviceAdapter
• ThrowsAdvice 实现
• org.springframework.aop.framework.adapter.ThrowsAdviceAdapter



### 17 AOP 代理接口 — AopProxy

• 接口 - org.springframework.aop.framework.AopProxy
• 实现
• JDK 动态代理
• org.springframework.aop.framework.JdkDynamicAopProxy
• CGLIB 字节码提升
• org.springframework.aop.framework.CglibAopProxy
• org.springframework.aop.framework.ObjenesisCglibAopProxy



### 18 AopProxy 工厂接口与实现

• 接口 - org.springframework.aop.framework.AopProxyFactory
• 默认实现：org.springframework.aop.framework.DefaultAopProxyFactory
• 返回类型
• org.springframework.aop.framework.JdkDynamicAopProxy
• org.springframework.aop.framework.CglibAopProxy
• org.springframework.aop.framework.ObjenesisCglibAopProxy



### 19 JDK AopProxy 实现 — JdkDynamicAopProxy

• 实现 - org.springframework.aop.framework.JdkDynamicAopProxy
• 配置 - org.springframework.aop.framework.AdvisedSupport
• 来源 - org.springframework.aop.framework.DefaultAopProxyFactory



### 20 CGLIB AopProxy 实现  — CglibAopProxy

• 实现 - org.springframework.aop.framework.CglibAopProxy
• 配置 - org.springframework.aop.framework.AdvisedSupport
• 来源 - org.springframework.aop.framework.DefaultAopProxyFactory



### 21 AopProxyFactory 配置管理器 — AdvisedSupport

• 核心 API - org.springframework.aop.framework.AdvisedSupport
• 语义 - 代理配置
• 基类 - org.springframework.aop.framework.ProxyConfig
• 实现接口 - org.springframework.aop.framework.Advised
• 使用场景 - org.springframework.aop.framework.AopProxy 实现



### 22 Advisor 链工厂接口与实现 — AdvisorChainFactory

• 核心 API - org.springframework.aop.framework.AdvisorChainFactory
• 特殊实现 -
org.springframework.aop.framework.InterceptorAndDynamicMethodMatcher
• 默认实现 - org.springframework.aop.framework.DefaultAdvisorChainFactory



### 23 目标对象来源接口与实现 — TargetSource

• 核心 API - org.springframework.aop.TargetSource
• 实现
• org.springframework.aop.target.HotSwappableTargetSource
• org.springframework.aop.target.AbstractPoolingTargetSource
• org.springframework.aop.target.PrototypeTargetSource
• org.springframework.aop.target.ThreadLocalTargetSource
• org.springframework.aop.target.SingletonTargetSource



### 24 代理对象创建基础类 — ProxyCreatorSupport

• 核心 API - org.springframework.aop.framework.ProxyCreatorSupport
• 语义 - 代理对象创建基类
• 基类 - org.springframework.aop.framework.AdvisedSupport



### 25 AdvisedSupport 事件监听器 — AdvisedSupportListener

• 核心 API - org.springframework.aop.framework.AdvisedSupportListener
• 事件对象 - org.springframework.aop.framework.AdvisedSupport
• 事件来源 - org.springframework.aop.framework.ProxyCreatorSupport
• 激活事件触发 - ProxyCreatorSupport#createAopProxy
• 变更事件触发 - 代理接口变化时、 Advisor 变化时、配置复制



### 26 ProxyCreatorSupport 标准实现 — ProxyFactory

• 核心 API - org.springframework.aop.framework.ProxyFactory
• 基类 - org.springframework.aop.framework.ProxyCreatorSupport
• 特性增强
• 提供一些便利操作



### 27 ProxyCreatorSupport loC 容器实现 — ProxyFactoryBean

• 核心 API - org.springframework.aop.framework.ProxyFactoryBean
• 基类 - org.springframework.aop.framework.ProxyCreatorSupport
• 特点
• Spring IoC 容器整合
• org.springframework.beans.factory.BeanClassLoaderAware
• org.springframework.beans.factory.BeanFactoryAware
• 特性增强
• 实现 org.springframework.beans.factory.FactoryBean



### 28 ProxyCreatorSupport AspectJ 实现 — AspectJ Proxy Factory

• 核心 API - org.springframework.aop.aspectj.annotation.AspectJProxyFactory
• 基类 - org.springframework.aop.framework.ProxyCreatorSupport
• 特点
• AspectJ 注解整合
• 相关 API
• AspectJ 元数据 - org.springframework.aop.aspectj.annotation.AspectMetadata
• AspectJ Advisor 工厂 - org.springframework.aop.aspectj.annotation.AspectJAdvisorFactory



### 29 loC 容器自动代理抽象 — AbstractAutoProxyCreator

• API - org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator
• 基类 - org.springframework.aop.framework.ProxyProcessorSupport
• 特点
• 与 Spring Bean 生命周期整合
• org.springframework.beans.factory.config.SmartInstantiationAwareBeanPostProcessor



### 30 loC 容器自动代理标准实现

• 基类 - org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator
• 默认实现 - DefaultAdvisorAutoProxyCreator
• Bean 名称匹配实现 - BeanNameAutoProxyCreator
• Infrastructure Bean 实现 - InfrastructureAdvisorAutoProxyCreator



### 31 loC 容器自动代理 AspectJ 实现 — AspectJAwareAdvisorAutoProxyCreator

• org.springframework.aop.aspectj.autoproxy.AspectJAwareAdvisorAutoProxyCreator
• 基类 - org.springframework.aop.framework.autoproxy.AbstractAdvisorAutoProxyCreator



### 32 AOP Infrastructure Bean接口 — AoplnfrastructureBean

• 接口 - org.springframework.aop.framework.AopInfrastructureBean
• 语义 - Spring AOP 基础 Bean 标记接口
• 实现
• org.springframework.aop.framework.autoproxy.AbstractAutoProxyCreator
• org.springframework.aop.scope.ScopedProxyFactoryBean
• 判断逻辑
• AbstractAutoProxyCreator#isInfrastructureClass
• ConfigurationClassUtils#checkConfigurationClassCandidate



### 33 AOP上下文辅助类 - AopContext

• API - org.springframework.aop.framework.AopContext
• 语义 - ThreadLocal 的扩展，临时存储 AOP 对象



### 34 代理工厂工具类 - AopProxyUtils

• API - org.springframework.aop.framework.AopProxyUtils
• 代表方法
• getSingletonTarget - 从实例中获取单例对象
• ultimateTargetClass - 从实例中获取最终目标类
• completeProxiedInterfaces - 计算 AdvisedSupport 配置中所有被代理的接口
• proxiedUserInterfaces - 从代理对象中获取代理接口



### 35 AOP 工具类 - AopUtils

• API - org.springframework.aop.support.AopUtils
• 代表方法
• isAopProxy- 判断对象是否为代理对象
• isJdkDynamicProxy - 判断对象是否为 JDK 动态代理对象
• isCglibProxy - 判断对象是否为 CGLIB 代理对象
• getTargetClass - 从对象中获取目标类型
• invokeJoinpointUsingReflection - 使用 Java 反射调用 Joinpoint（目标方法）



### 36 AspectJ Enable 模块驱动实现 一 @EnableAspectJAutoProxy

• 注解 - org.springframework.context.annotation.EnableAspectJAutoProxy
• 属性方法
• proxyTargetClass - 是否已类型代理
• exposeProxy - 是否将代理对象暴露在 AopContext 中
• 设计模式 - @Enable 模块驱动
• ImportBeanDefinitionRegistrar 实现 -
org.springframework.context.annotation.AspectJAutoProxyRegistrar
• 底层实现
• org.springframework.aop.aspectj.annotation.AnnotationAwareAspectJAutoProxyCreator



### 37 AspectJ XML配置驱动实现 — `<aop:aspectj-autoproxy/>`

• XML 元素 - `<aop:aspectj-autoproxy/>`
• 属性方法
• proxy-target-class - 是否已类型代理
• expose-proxy - 是否将代理对象暴露在 AopContext 中
• 设计模式 - Extensible XML Authoring
• 底层实现
• org.springframework.aop.config.AspectJAutoProxyBeanDefinitionParser



### 38 AOP配置 Schema-based 实现 — `<aop:config/>`

• XML 元素 - `<aop:config/>`
• 属性方法
• proxy-target-class - 是否已类型代理
• expose-proxy - 是否将代理对象暴露在 AopContext 中
• 嵌套元素
• pointcut
• advisor
• aspect
• 底层实现
• org.springframework.aop.config.ConfigBeanDefinitionParser



### 39 Aspect Schema-based 实现 — `<aop:aspect/>`



### 40 Pointcut Schema-based 实现 — `<aop:pointcut/>`



### 41 Around Advice Schema-based 实现 — `<aop:around/>`



### 42 Before Advice Schema-based 实现 — `<aop:before/>`

### 43 After Advice Schema-based 实现 — `<aop:after/>`

### 44 Introduction Schema-based 实现 — `<aop:declare-parents/>`

### 45 作用域代理 Schema-based 实现 — `<aop:scoped-proxy/>`

### 46 面试题精选



## 第四章：Spring AOP设计模式

### 01 抽象工厂模式 (Abstract factory) 实现

### 02 构建器模式 (Builder) 实现

### 03 工厂方法模式 (Factory method) 实现

### 04 原型模式 (Prototype) 实现

### 05 单例模式 (Singleton) 实现

### 06 适配器模式 (Adapter) 实现

### 07 组合模式 (Composite) 实现

### 08 装饰器模式 (Decorator) 实现

### 09 享元模式 (Flyweight) 实现

### 10 代理模式 (Proxy) 实现

### 11 模板方法模式 (Template Method) 实现

### 12 责任链模式 (Chain of Responsibility) 实现

### 13 观察者模式 (Observer) 实现

### 14 策略模式 (Strategy) 实现

### 15 命令模式 (Command) 实现

### 16 状态模式 (State) 实现

### 17 面试题精选





## 第五章：Spring AOP 在 SpringFramework 内部应用

### 01 Spring AOP 在 Spring 事件 (Events)

### 02 Spring AOP 在 Spring 事务 (Transactions)

### 03 Spring AOP 在 Spring 数据 (Data)

### 04 Spring AOP 在 Spring 缓存抽象 (Caching Abstract)

### 05 Spring AOP 在 Spring 本地调度 (Scheduling)

### 06 Spring AOP 在 Spring 整合 (Integration)

### 07 Spring AOP 在 Spring 远程调用 (Remoting)

### 08 面试题精选

### 09 结束语

 

   小马哥讲Spring AOP   

   编程思想   

   课程大纲   

   第一章：Spring AOP总览   

   课程介绍   

   01   

   02内容综述   

   03知识储备：基础、基础，   04AOP引入：OOP存在哪些局限性?   05AOP常见使用场景   

   还是基础!   

   AOP概念：Aspect、Join Point和Advice等术语应该   如何理解?   

   06   

   07 Java AOP设计模式：代理、判断和拦截器模式   

   Java AOP代理模式   

   (Proxy) ：Java 静态代理和动态代   

   08   

   理的区别是什么?   

   09 Java AOP判断模式   

   (Predicate)   ：如何筛选Join Point?   (Interceptor)   ：拦截执行分别代   

   Java AOP拦截器模式   

   10   

   表什么?   

   11 Spring AOP功能概述：核心特性、编程模型和使用限制   

 

 








   Spring AOP编程模型：注解驱动、XML配置驱动和底   

   12   13   14   

   层API   

   Spring AOP设计目标：Spring AOP   

   与AOP框架之间   

   的关系是竞争还是互补?   

   Spring AOP Advice类型：Spring AOP丰富了哪些   

   AOP Advice呢?   

   15 Spring AOP代理实现：为什么SpringFramework选   

   择三种不同AOP实现?   

   JDK动态代理：为什么Proxy.new Proxy   Instance   

   会生   

   16   

   成新的字节码?   

   17 C GLIB动态代理：为什么Java动态代理无法满足   AOP的需要?   

   AspectJ代理代理：为什么Spring推荐AspectJ注解?   

   18   

   19 AspectJ基础：Aspect、Join Points、Pointcuts和   Advice语法和特性   

   20 AspectJ注解驱动：   21 面试题精选   

   注解能完全替代AspectJ语言吗?   

   第二章：Spring AOP   

   基础   

   Spring核心基础：《小马哥讲Spring核心编程思想》还   

   记得多少?   

 

 








   @AspectJ注解驱动   

   2   

   编程方式创建@AspectJ代理   XML配置驱动-创建AOP代理   

   3   4   5   6   

   标准代理工厂 API-Proxy   Factory   

   指令与表达式：为什么Spring只   

   @AspectJ   Pointcut   能有限支持?   

   7 XML配置Pointcut   API实现Pointcut   

   8   

   @AspectJ拦截动作：@Around   XML配置Around Advice   

   与 @Pointcut有区别吗?   

   9   

   10   11   12   

   API实现Around Advice   

   @AspectJ前置动作：   执行?   

   @Before与   

   @Around谁优先级   

   XML配置Before Advice   

   13   

 






   API实现Before Advice   

   14   

   15 @AspectJ后置动作-   

   -三种After Advice之间的关系?   

   XML配置三种After Advice   API实现三种After Advice   自动动态代理   

   16   17   18   19   

   替换Target Source   

   面试题精选   

   20   

   第三章：Spring AOP API设计与实现   

   Spring AOP API整体设计   

   01   

   02接入点接口-Join point   

   03 Join point条件接口-Pointcut   

   04 Pointcut操作   

   Composable Pointcut   

   05 Pointcut便利实现   06 Pointcut AspectJ   

   实现-AspectJ Expression Pointcut   

   07 Join point执行动作接口-Advice   

 

 








   08 Join point Before Advice标准实现   09 Join point Before Advice AspectJ实现   Join point After Advice标准实现   

   10   11   12   13   14   

   Join point After Advice AspectJ实现   Advice容器接口-Advisor   

   Pointcut与Advice连接器-Pointcut Advisor   

   Introduction   

   Advice连接器-Introduction Advisor   

   与   

   Advisor的Interceptor适配器-Advisor Adapter   

   15   16   

   Advisor Adapter实现   

   17 AOP代理接口-Aop Proxy   

   18 Aop Proxy工厂接口与实现   

   19   

   JDK Aop Proxy实现-Jdk Dynamic Aop Proxy   

   20CGLIBAop Proxy实现-C glib Aop Proxy   

   配置管理器-Advised Support   

   21Aop Proxy Factory   

   22 Advisor链工厂接口与实现-Advisor Chain Factory   

   目标对象来源接口与实现-Target Source   

   23   

 

 








   24代理对象创建基础类-Proxy Creator Support   

   25 Advised Support事件监听器-Advised Support   Listener   26 Proxy Creator Support标准实现-Proxy Factory   27 Proxy Creator Support loC容器实现-Proxy Factory Bean   

   28 Proxy Creator Support AspectJ实现-   AspectJ Proxy Factory   

   29loC容器自动代理抽象-Abstract Auto Proxy Creator   30loC容器自动代理标准实现   

   loC容器自动代理AspectJ实现-   

   31   

   AspectJ Aware Advisor Auto Proxy Creator   

   32AOP Infrastructure Bean接口-A opl   nfrastructure Bean   33AOP上下文辅助类-Aop Context   

   34 代理工厂工具类-Aop Proxy Utils   35AOP工具类-Aop Utils   

 






   AspectJ Enable模块驱动实现   

   36   37   

   一   

   @Enable AspectJ Auto Proxy   

   AspectJ XML配置驱动实现-   

   <aop：aspectj-autoproxy/>   

   38AOP配置Schema-based实现   39 Aspect Schema-based   

   -<aop：config/>   

   实现-<aop：aspect/>   

   40 Pointcut Schema-based实现-<aop：pointcut/>   

   41   

   Around Advice Schema-based实现-<aop：around/>   

   42 Before Advice Schema-based实现-<aop：before/>   

   43 After Advice Schema-based实现-<aop：after/>   

   Introduction Schema-based实现-   

   45   

   <aop：declare-parents/>   

   44 作用域代理Schema-based实现-   

   <aop：scoped-proxy/>   

   45AOP配置Schema-based实现-<aop：config/>   46面试题精选   

   第四章：Spring AOP设计模式   

 

 








   02 构建器模式   03工厂方法模式   

   (Builder) 实现   (Factory method)   

   实现   

   04 原型模式 (Prototype) 实现   

   05单例模式   

   (Singleton)   (Adapter)   

   实现   

   06   

   适配器模式   

   实现   实现   

   07组合模式   

   (Composite)   

   08装饰器模式   09享元模式   

   (Decorator)   (Flyweight)   

   实现   

   实现   

   10 代理模式 (Proxy) 实现   

   11 模板方法模式   12 责任链模式   13 观察者模式   

   (Template Method)   

   实现   

   (Chain of Responsibility)   

   实现   

   (Observer)   

   实现   

   策略模式   命令模式   

   (Strategy) 实现   

   14   15   16   

   (Command)   

   实现   

   状态模式 (State) 实现   

 

 








   面试题精选   

   17   

   第五章：   

   Spring AOP在SpringFramework内部应用   

   01SpringAOP在Spring事件   

   (Events)   

   02SpringAOP在Spring事务   03SpringAOP在Spring数据   04SpringAOP在Spring缓存抽象   05SpringAOP在Spring本地调度   06SpringAOP在Spring整合   Spring AOP在Spring远程调用   

   (Transactions)   

   (Data)   

   (Caching Abstract)   

   (Scheduling)   

   (Integration)   

   (Remoting)   

   07   

   08面试题精选   

   结束语   

   09   

 

 







# END