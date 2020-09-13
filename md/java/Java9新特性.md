# [Java 新特性](https://docs.oracle.com/en/java/javase/index.html)

## 不同版本，看 what‘s New



## [Java 9 新特性](https://docs.oracle.com/javase/9/whatsnew/toc.htm)

> jcmd，jps，jconsole，jdeps，Source，Compiler，Code Cache
>
> jshell，
>
> JavaScript on JVM
>
> JRuby
>
> SecurityManagement    Permission
>
> javac  java实现的  Source类   Compiler
>
> Guava 

* 重要新特性
  * 模块化
    module，指定依赖，指定可访问类。。。
  * 核心库
  * 安全
  * 部署
  * JVM
  * 调优


## 核心库



### 集合工厂方法（Factory Method of Collection）

> guava   Sets、Lists、List
>
> Apache   Lang
>
> Objects，Collections、Executors
>

### 进程（Process）

> ProcessHandle#descendants   所有子或孙 进程
>
> ProcessHandle#children   所有直接子进程
>
> ProcessHandle#current#allProcesses
>
> ProcessHandle#current#onExit#thenAccept(out::println)（报错）

* javap

```java
/** 获取 pid
Runtime#getRuntime#**addShutdownHook**(new Thread(){})
RuntimeMXBean
ManagementFactory#getRuntimeMXBean
jconsole 源码

[java 9前实现](https://github.com/mercyblitz/confucius-commons)

VMManagement
*/
```



### 堆栈 API（Stack - Walking API）

StackWalker#getInstance   Option

StackFrame（栈帧） 

官方 demo 有问题

fillInStackTrace



### 变量处理（Variable Handles）

[java.util.concurrent.atomic](https://docs.oracle.com/javase/9/docs/api/java/util/concurrent/atomic/package-summary.html) 

sun.misc.Unsafe

offset   用不好 jvm 崩溃

VarHandle

MethodHandles#lookup



### 统一日志 API（Platform Logging API）

System.Logger   java9

> Apache commons-logging：Facade API for Java Logging And Log4j
> slf4j-api : java logging + log4j + logback
>
> log4j2
>
> Java 9 : Logger
>
> Spring Framework < 5.0 - commons - logging
> Spring Framework >= 5.0 - slf4j-api
>
>
> 猜测：Spring Framework 6   Java 9  Logger



### 自旋等待提示（Spin - Wait Hints）

自旋：一段时间无意义的循环，避免频繁的线程上下文切换

### 压缩字符串（Compact String）



























































































