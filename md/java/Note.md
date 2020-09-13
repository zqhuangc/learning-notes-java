Parent的静态块
Child的静态块
Parent的构造块
Parent的构造方法
Child的构造块
Child的构造方法



属性不存在覆盖（即使同名类型不同），而方法是实实在在的覆盖（复写）

> Tips：构造代码块优先于构造方法执行，**且优先于属性初始化之前执行**



Try-With-Resource  AutoCloseable

https://mp.weixin.qq.com/s/o5VmolECLC27NmJyzB9CzA

- \1. 在Finally中清理资源或者使用Try-With-Resource语句

- - 使用Finally
  - Java 7的Try-With-Resource语句

- \2. 给出准确的异常处理信息

- \3. 记录你所指定的异常

  - 确保在Javadoc中添加一个@throws 声明，并描述可能导致的异常情况。

- \4. 使用描述性消息抛出异常

- \5. 最先捕获特定的异常

- \6. 不要在catch中使用Throwable

- \7. 不要忽略Exceptions

- \8. 不要记录和抛出一个异常

- \9. 包装异常

## 基础

primitive 原语

基本类型 （primitive type）

String Api 不同版本的优化

类：变量（全局，局部），方法，限定类型，

Socket 网络

I/O

包装类型（packaging type）

GUI   核心：事件监听机制





### Class

**a instanceof B** 

a是B的实例，B是类或者接口、父类或父接口，即B c = a成立。



**B.class.isInstance(a)**

这个叫动态等价，效果和上面等价，一般用于检查泛型，如jdk中CheckedMap里面用到这个检查Map里面的key、value类型是否和约定的一样。



**A.class.isAssignableFrom(B)**

两个class的类型关系判断，判断B是不是A的子类或子接口



## Web 规范

html5，css3

js

Servlet

Jsp







## 框架

首先学会用（官方参考文档，API）

底层架构

源码：找入口，追溯本源





## 系统

JVM

Linux

计算机系统原理，组成原理



## 网络协议



## 知识点

全局安全点（这个时间点是上没有正在执行的代码）



锁消除的依据是逃逸分析的数据支持。



## 死锁

产生条件：
互斥、持有、不可剥夺、环形等待。

## Jconsole查看死锁

进程->线程->查看各线程所持有情况
jps 

## jsatck来查看堆栈信息

避免死锁->加锁顺序





# Java SPI机制

SPI 全称为 (Service Provider Interface) ，是JDK内置的一种服务提供发现机制。SPI是一种动态替换发现的机制， 比如有个接口，想运行时动态的给它添加实现，你只需要添加一个实现。我们经常遇到的就是java.sql.Driver接口，其他不同厂商可以针对同一接口做出不同的实现，mysql和postgresql都有不同的实现提供给用户，而Java的SPI机制可以为某个接口寻找服务实现。



# java中什么是bridge method（桥接方法）

ACC_BRIDGE和ACC_SYNTHETIC


