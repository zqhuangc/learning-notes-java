如何找看源码的入口？ 
IOC/DI/AOP/BOP 
ThreadLocal 多数据源切换
ListableBeanFactory  
HierarchicalBeanFactory
ResourceLoader

is a  父子关系

**Bean factory 的层次关系，父子容器，（类似双亲委派但先子后父）（像同时 annotation 和 xml 分别使用不同 bean factory）**

BeanDefinition  PostProcessor   BeanFactory



Spring初始化bean 的时候做一些事情该怎么做、想要在bean销毁的时候做一些事情该怎么做、MyBatis中$和#的区别等等，


spring中 ioc 只负责 new 并存对象实例，对象是否线程安全无关

DefaultBeanDefinitionDocumentReader 是 Spring 核心的解析器



### [事件驱动](..\..\..\md\Java\并发\Java并发集合框架(J.U.C).md)



Lombok就是一个实现了"JSR 269 API"的程序。在使用javac的过程中，它产生作用的具体流程如下：

1. javac对源代码进行分析，生成一棵抽象语法树(AST)
2. javac编译过程中调用实现了JSR 269的Lombok程序
3. 此时Lombok就对第一步骤得到的AST进行处理，找到Lombok注解所在类对应的语法树(AST)，然后修改该语法树(AST)，增加Lombok注解定义的相应树节点
4. javac使用修改后的抽象语法树(AST)生成字节码文件



## 提供扩展的方式







## 阿里开源项目系列之 Spring Context 扩展工程





## 使用场景



### 工具类 - `com.alibaba.spring.util`

为 `spring-core` 提供工具类的补充

>  `spring-core` 模块里面提供大量的工具类：
>
>  - StringUtils





### Spring Beans 相关扩展 - `com.alibaba.spring.beans`



#### Spring Bean 生命周期后置处理器 - `BeanPostProcessor`



##### 基本语义

- 处理 Bean 初始化生命周期，包括 before 和 after

- 在回调方法处理时，可能会对象变化，不过多数情况是不需要变化类型，即改变当前 Bean 参数对象中的状态即可

  - 比如：

  - ```java
    class MyBeanPostProcessor implements BeanPostProcessor {
    
        public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
            if (ClassUtils.isAssignable(bean.getClass(), CharSequence.class)) { // 凡是 Bean 类型为 CharSequence 的子类
                return bean.toString();                                         // 被转化 String 类型
            }
            return bean;
        }
    }
    ```


##### 实现细节

- 在 Spring Framework 5.0 之前，需要完全实现 `BeanPostProcessor` 接口定义的方法：
  - 初始化前 - postProcessBeforeInitialization
  - 初始化后 - postProcessAfterInitialization

- 从 Spring Framework 5.0 + 开始，`BeanPostProcessor` 接口提供了默认实现，即无操作返回

> 使用不便的地方：
>
> - 对 bean 类型通常需要自行判断



N Spring Beans -> M  `BeanPostProcessor`  -> Ops Count = N * M



##### 与 BeanFactory 关系

关联关系 - `BeanPostProcessor` 与 `BeanFactory ` 是 N 对 1 的的关系，即一个 Spring BeanFactory 实例可以关联 N 个 `BeanPostProcessor` ， `BeanPostProcessor`  来源：

- 显示地插入，即调用  `ConfigurableBeanFactory#addBeanPostProcessor` 方法
- `BeanPostProcessor` 定义成普通 Spring Bean，即 `AbstractApplicationContext#registerBeanPostProcessors`



#### Spring BeanFactory 生命周期后置处理器 - `BeanFactoryPostProcessor`

##### 基本语义

自定义处理 `ConfigurableListableBeanFactory`，当它已经被基本处理完成，即 `AbstractApplicationContext#prepareBeanFactory` 方法处理



##### 生命周期回调方法 - `postProcessBeanFactory`

- 方法参数 - `ConfigurableListableBeanFactory`

- 回调时机 - `AbstractApplicationContext#invokeBeanFactoryPostProcessors`

  - 回调 `BeanFactoryPostProcessor`
  - 回调 `BeanDefinitionRegistryPostProcessor` - 注册 `BeanDefinition`


##### 与 ApplicationContext 的关系

关联关系 - `BeanFactory` 与 `ApplicationContext` 是 1 对 1 的关系，绝大多数情况， `BeanFactory` 的实现是 `DefaultListableBeanFactory`，并且 `ApplicationContext` 实现类是 `AbstractApplicationContext` 的子类

`BeanFactoryPostProcessor` 与 `ApplicationContext` 是 N 对 1 的关系：

```java
public abstract class AbstractApplicationContext ... {
    ...
 	private final List<BeanFactoryPostProcessor> beanFactoryPostProcessors = new ArrayList<>(); // 有序       
    ...
}
```

`BeanFactoryPostProcessor` 的来源：

- 显示地插入，即调用 `AbstractApplicationContext#addBeanFactoryPostProcessor` 方法
- `BeanFactoryPostProcessor`  定义成普通 Spring Bean，即 `AbstractApplicationContext#invokeBeanFactoryPostProcessors` 方法处理



#### Spring Bean 定义 - `BeanDefinition`

##### 基本语义

`BeanDefinition` 是 Bean 声明元数据的一种描述接口

- 主要元数据指标：
  - Bean 类型 - `getBeanClassName()`
  - Parent Bean 名称 -`getParentName()`
  - 工厂 Bean 名称 - `getFactoryBeanName()`
  - 工厂方法名称 - `getFactoryMethodName()`
  - Bean 生命范围  - `getScope()`
- 次要元数据指标
  - Bean 是否懒加载 - `isLazyInit()`
  - Bean 是否为 Primary - `isPrimary()`



##### 黑科技

- 特殊元数据指标
  - Bean 定义的来源 - `setSource(Object)` - 区分不同 Bean 定义一种手段
  - Bean 定义属性上下文 - `setAttribute(String,Object)` - 临时存储于当前 Bean 定义相关的属性，它不影响 Bean 实例化/初始化，然而可以辅助 Bean 初始化





### BeanFactory 接口

子接口 

- ListableBeanFactory - 在 BeanFactory 基础上，主要职责是扩展对 BeanDefinition 管理，以及 Bean 对象集合返回
- HierarchicalBeanFactory - 层次性的 BeanFactory ，类似于 ClassLoader 双亲委派
- ConfigurableBeanFactory - 可配置（可写）BeanFactory，相对于其他 BeanFactory 只读的特性



## 常见问题



### BeanFactory 和 FactoryBean 的区别和联系？

`FactoryBean` -  创建特定 Bean 的工厂，其对象是 BeanFactory 一个普通 Bean

主要特性

- 指定 Bean 类型 - `getObjectType()` 方法
- 获取/创建 Bean - `getObject()` 方法
- 创建 Bean 是否为单例 - `isSingleton()` 方法



举例 - `UserFactoryBean` 创建 `User` Bean，通常应用关心 `User` Bean ，而非 `UserFactoryBean`  Bean



`BeanFactory` 是 Spring Bean 容器（接口），也管理 `FactoryBean` Bean 集合。









# 2019.04.26 「小马哥技术周报」- 第二十四期《Spring Core 面试题精选》



## 精选



### 为什么要使用 Spring？

### 

### 解释一下什么是 IoC？

好莱坞原则

BeanFactory



Spring -> Interface21

BeanFactory

1.0 ~ 2.0 XML 主导

1.2+ ~ 5.2（PRE） 注解

2.0 注解



<bean id="user" class="User" >



<bean class="XXX">

	<property bean-ref="user" />

</bean>

### Spring 中的 bean 是线程安全的吗？

Bean 对象的本身是否为是不确定的。



<bean id="map" class="java.util.concurrent.ConcurrentHashMap" />

>  获得 Bean 是线程安全



### Spring 常用的注入方式有哪些？

Setter（方法参数）

构造器（参数）

方法注入

```java
// @Autowired 可选
public User user(Money money){
    
}
```



### Spring 支持几种 bean 的作用域？

```xml
<bean id="map" class="java.util.concurrent.ConcurrentHashMap"  scope="prototype" />
```



request -> ServletRequest



Servlet 引擎的线程模型 1 Thread -> 1 ServletRequest

ServletRequest 是线程安全的



HTTP -> Tomcat(NIO) -> Thread -> Request( Bean)



```java
@RestController
public class MyController {
    
    @Autowired
    private User user;
    
    @GetMapping("/user/xxx")
    public void execute(){
        user.getName();
    }
}
```



JSP、JSTL、EL

Page - `PageContext`

Request - `ServletRequest`

Session - `HttpSession`

Application - `ServletContext`



### Spring 自动装配(Autowired) bean 有哪些方式？

Type

Name

Constructor

Auto detected

`AUTOWIRE_NO`

`AUTOWIRE_BY_NAME`

`AUTOWIRE_BY_TYPE`

`AUTOWIRE_CONSTRUCTOR`

`AUTOWIRE_AUTODETECT`



### 如何理解 Bean 的生命周期？



AbstractAutowireCapableBeanFactory

- invokeAwareMethods

ApplicationContextAwareProcessor

- ApplicationContextAware
- BeanClassLoaderAware
- ...

BeanPostProcessor

- InstantiationAwareBeanPostProcessor
  - SmartInstantiationAwareBeanPostProcessor

- postProcessBeforeInitialization()
- postProcessAfterInitialization()

InitializaingBean 

- afterPropertiesSet()

DisposableBean

- destroy()



### Spring 事务实现方式有哪些？



### 说一下 Spring 的事务隔离？



### @Autowired 的作用是什么？

@Autowired  用于依赖注入，Spring 官方不推荐使用，推荐使用构造器注入。不能修饰 static 字段



QQ 3群：98221500





### 依赖注入

#### 什么是 Spring 依赖注入？

#### Spring 有哪些依赖注入的类型？

