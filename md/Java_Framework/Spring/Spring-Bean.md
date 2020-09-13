Spring 只帮我们管理单例模式 Bean 的完整生命周期，对于 prototype 的 bean ，Spring 在创建好交给使用者之后则不会再管理后续的生命周期。

### Spring中类名设计：  
* default* 都为默认实现  

* do*      都为具体做操作的方法  

* \*Wrapper 都为包装类  

* *Aware   监听  -> *Processor 处理类

  > bean需要知道容器的状态获取容器的信息直接使用容器那么就需要实现XXXAware来实现
  >
  > AnnotationConfigApplicationContext
  >
  > 一、资源文件的定位(xml,javaconfig)
  >
  > 二、BeanDefinition的载入
  >
  > 三、向IOC容器注册这些BeanDefinition
  >
  >
  >
  > finishBeanFactoryInitialization(beanFactory）。只要bean不是抽象，是单例，不是懒加载就会在这里进行依赖注入  -》 getBean
  >
  >
  >
  > * Bean实例生命周期的执行过程如下：
  >
  > Spring对bean进行实例化，默认bean是单例；
  >
  > Spring对bean进行依赖注入；
  >
  > 如果bean实现了BeanNameAware接口，spring将bean的id传给setBeanName()方法；
  >
  > 如果bean实现了BeanFactoryAware接口，spring将调用setBeanFactory方法，将BeanFactory实例传进来；
  >
  > 如果bean实现了ApplicationContextAware接口，它的setApplicationContext()方法将被调用，将应用上下文的引用传入到bean中；
  >
  >
  >
  > 如果bean实现了BeanPostProcessor接口，它的postProcessBeforeInitialization方法将被调用；
  >
  > 如果bean实现了InitializingBean接口，spring将调用它的afterPropertiesSet接口方法，类似的如果bean使用了init-method属性声明了初始化方法，该方法也会被调用；
  >
  >
  >
  > 如果bean实现了BeanPostProcessor接口，它的postProcessAfterInitialization接口方法将被调用；
  >
  > 此时bean已经准备就绪，可以被应用程序使用了，他们将一直驻留在应用上下文中，直到该应用上下文被销毁；
  >
  >
  >
  > 若bean实现了DisposableBean接口，spring将调用它的distroy()接口方法。同样的，如果bean使用了destroy-method属性声明了销毁方法，则该方法被调用；

* \*Support 实现功能优化扩展 

* \*Properties 对应配置信息类

## ClassPathXmlApplicationContext-IOC容器



![定位、载入、注册](../../../image/spring-ioc初始过程.PNG)
* BeanDefinition系列（配置信息）
* BeanDefinitionReader系列（载入、注册信息）
* beanFactory系列(初始化)

### 定位、载入、注册

![定位、载入、注册](../../../image/spring-ioc初始过程.PNG)
* 定位  得到Resource
* 加载  BeanDefinition系列（配置信息）
```
默认 bean 加载过程
http://www.cnblogs.com/wcj-java/p/9218239.html
SAX解析xml
自定义。。。
```
loadBeanDefinitions()
定位到具体"classpath:"Resource找到路径
doLoadBeanDefinitions(..) => 配置信息设到BeanDefinition记录

* BeanDefinitionReader系列（载入、注册信息）
* beanFactory系列(初始化)

## 实现 *Aware 接口（监听）
*Aware 接口可以用于在初始化 bean 时获得 Spring 中的一些对象，如获取 Spring 上下文等
ApplicationContextAware

## 自定义bean在创建和销毁执行的操作
### 注解方式

在 bean 初始化时会经历几个阶段，首先可以使用注解 @PostConstruct, @PreDestroy 来在 bean 的创建和销毁阶段进行调用:

InitializingBean, DisposableBean 接口

还可以实现 InitializingBean,DisposableBean 这两个接口，也是在初始化以及销毁阶段调用：

InitializingBean, DisposableBean 接口

还可以实现 InitializingBean,DisposableBean 这两个接口，也是在初始化以及销毁阶段调用：

### 自定义初始化和销毁方法

也可以自定义方法用于在初始化、销毁阶段调用:
@Bean(initMethod = "start", destroyMethod = "destroy")

### BeanPostProcessor 增强处理器

实现 BeanPostProcessor 接口，Spring 中所有 bean 在做初始化时都会调用该接口中的两个方法，可以用于对一些特殊的 bean 进行处理：

*Aware 接口可以用于在初始化 bean 时获得 Spring 中的一些对象，如获取 Spring 上下文等。

直到 Spring 上下文销毁时则会调用自定义的销毁方法以及实现了 DisposableBean 的 destroy() 方法。

@Bean
@Qualify
@







## 资源定位

refreshBeanFactory()

