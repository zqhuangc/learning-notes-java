# Dubbo 源码解读要点

首先我们要关注的是服务的发布和服务的消费这两个主要的流程，那么就可以基于这个点去找到源码分析的突破口。那么自然而然我们就可以想到spring的配置

## Spring对外留出的扩展

dubbo是基于 spring 配置来实现服务的发布的，那么一定是基于spring的扩展来写了一套自己的标签，那么spring是如何解析这些配置呢？

* 在spring中定义了两个接口
  - NamespaceHandler: 注册一堆BeanDefinitionParser，利用他们来进行解析
  - BeanDefinitionParser:用于解析每个element的内容

 

Spring默认会加载jar包下的`META-INF/spring.handlers`文件寻找对应的`NamespaceHandler`。 

Dubbo-config 模块下的 dubbo-config-spring  

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24f5uo4t0j30ao07uglp.jpg)

## Dubbo的接入实现

Dubbo中spring扩展就是使用spring的自定义类型，所以同样也有NamespaceHandler、BeanDefinitionParser。而NamespaceHandler是DubboNamespaceHandler

```java
public class DubboNamespaceHandler extends NamespaceHandlerSupport {
   static {
      Version.checkDuplicate(DubboNamespaceHandler.class);
   }
   public void init() {
        registerBeanDefinitionParser("application", new DubboBeanDefinitionParser(ApplicationConfig.class, true));
        registerBeanDefinitionParser("module", new DubboBeanDefinitionParser(ModuleConfig.class, true));
        registerBeanDefinitionParser("registry", new DubboBeanDefinitionParser(RegistryConfig.class, true));
        registerBeanDefinitionParser("monitor", new DubboBeanDefinitionParser(MonitorConfig.class, true));
        registerBeanDefinitionParser("provider", new DubboBeanDefinitionParser(ProviderConfig.class, true));
        registerBeanDefinitionParser("consumer", new DubboBeanDefinitionParser(ConsumerConfig.class, true));
        registerBeanDefinitionParser("protocol", new DubboBeanDefinitionParser(ProtocolConfig.class, true));
        registerBeanDefinitionParser("service", new DubboBeanDefinitionParser(ServiceBean.class, true));
        registerBeanDefinitionParser("reference", new DubboBeanDefinitionParser(ReferenceBean.class, false));
        registerBeanDefinitionParser("annotation", new DubboBeanDefinitionParser(AnnotationBean.class, true));
    }
}

```



BeanDefinitionParser全部都使用了DubboBeanDefinitionParser，如果我们想看<dubbo:service>的配置，就直接看DubboBeanDefinitionParser中

这个里面主要做了一件事，把不同的配置分别转化成spring容器中的bean对象

application对应ApplicationConfig

registry对应RegistryConfig

monitor对应MonitorConfig

provider对应ProviderConfig

consumer对应ConsumerConfig

…

为了在spring启动的时候，也相应的启动provider发布服务注册服务的过程，而同时为了让客户端在启动的时候自动订阅发现服务，加入了两个bean

ServiceBean、ReferenceBean。

分别继承了ServiceConfig和ReferenceConfig

同时还分别实现了InitializingBean、DisposableBean, ApplicationContextAware, ApplicationListener, BeanNameAware

**InitializingBean**接口为bean提供了初始化方法的方式，它只包括afterPropertiesSet方法，凡是继承该接口的类，在初始化bean的时候会执行该方法。

**DisposableBean** bean被销毁的时候，spring容器会自动执行destory方法，比如释放资源

**ApplicationContextAware** 实现了这个接口的bean，当spring容器初始化的时候，会自动的将ApplicationContext注入进来

**ApplicationListener**  ApplicationEvent事件监听，spring容器启动后会发一个事件通知

**BeanNameAware** 获得自身初始化时，本身的bean的id属性

 

那么基本的实现思路可以整理出来了

1. 利用spring的解析收集xml中的配置信息，然后把这些配置信息存储到serviceConfig中

2. 调用ServiceConfig的export方法来进行服务的发布和注册

# 服务的发布

## ServiceBean

ServiceBean 是服务发布的切入点，通过 afterPropertiesSet 方法，调用export()方法进行发布。

export为父类 ServiceConfig 中的方法，所以跳转到 SeviceConfig 类中的 export 方法

* delay的使用

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24f6l7ek1j30bn0a2mxe.jpg)

我们发现，delay的作用就是延迟暴露，而延迟的方式也很直截了当，Thread.sleep(delay)

1. export 是 synchronized 修饰的方法。也就是说暴露的过程是原子操作，正常情况下不会出现锁竞争的问题，毕竟初始化过程大多数情况下都是单一线程操作，这里联想到了spring的初始化流程，也进行了加锁操作，这里也给我们平时设计一个不错的启示：初始化流程的性能调优优先级应该放的比较低，但是安全的优先级应该放的比较高！

2. 继续看 doExport() 方法。同样是一堆初始化代码

## export的过程

继续看 doExport()，最终会调用到 doExportUrls() 中：

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24f7ncne5j30j705f3yp.jpg)

最终实现逻辑

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24f7yyussj30nf03j0su.jpg)

在上面这段代码中可以看到Dubbo的比较核心的抽象：Invoker， Invoker是一个代理类，从ProxyFactory中生成。这个地方可以做一个小结

1. Invoker -执行具体的远程调用

2. Protocol – 服务地址的发布和订阅

3. Exporter – 暴露服务或取消暴露

## protocol发布服务

我们看一下dubboProtocol的 export方法：openServer(url）

接着调用 openServer， 继续 createServer 创建服务

继续看其中的 createServer方法：

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24f8b62u1j30i507mq36.jpg)

 

发现 ExchangeServer 是通过 Exchangers 创建的，直接看 Exchanger.bind 方法

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24f8yrst0j30s808j3z4.jpg)



getExchanger方法实际上调用的是 ExtensionLoader 的相关方法，这里的 ExtensionLoader 是dubbo插件化的核心，我们会在后面的插件化讲解中详细讲解，这里我们只需要知道Exchanger的默认实现只有一个：HeaderExchanger。上面一段代码最终调用的是：

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24f9a15dxj30xv02y74e.jpg)



可以看到Server与Client实例均是在这里创建的，HeaderExchangeServer需要一个Server类型的参数，来自Transporters.bind()：

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24f9jejq1j30nj043mx7.jpg)

getTransporter()获取的实例来源于配置，默认返回一个NettyTransporter：



# 服务消费

## ReferenceBean

和serviceBean发布一样，也是使用NamespaceHandler作为切入点，调用ReferenceBean里面的afterPropertiesSet方法

**方法调用顺序 afterPropertiesSet() -> getObject() -> get() -> init() -> createProxy()**

afterPropertiesSet方法中都是确认所有的组件是否都初始化好了，都准备好后我们进入生成Invoker的部分。这里的 getObject 会调用父类 ReferenceConfig 的 init 方法完成组装：

![ ](https://ws1.sinaimg.cn/large/006xzusPgy1g24f9r0a0aj30ni0of0v1.jpg)

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24fc6bhhtj30n60q50uy.jpg)

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24fcrp5mbj30if0dt75i.jpg)



## createProxy方法

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24fdbt4nij30pe0mw76c.jpg)

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24fdniszjj30pa0d9q3w.jpg)



## refprotocol.refer

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24fe1j1p1j30qo0dcjsk.jpg)

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24fe7y8g3j30q40dc75g.jpg)

 

至此 Reference 在关联了所有application、module、consumer、registry、monitor、service、protocol后调用对应 Protocol 类的 refer 方法生成 InvokerProxy。当用户调用service时dubbo会通过InvokerProxy 调用 Invoker 的 invoke 的方法向服务端发起请求。客户端就这样完成了自己的初始化。

 

这个代理实例中仅仅包含一个handler对象（InvokerInvocationHandler类的实例），handler中则包含了RPC调用中非常核心的一个接口Invoker<T>的实现，Invoker接口的的的定义如下：  

```java
public interface Invoker<T> extends Node {     
    Class<T> getInterface();  //调用过程的具体表示形式      
    Result invoke(Invocation invocation) throws RpcException;
} 
```



Invoker<T>接口的核心方法是invoke(Invocation invocation)，方法的参数Invocation是一个调用过程的抽象，也是Dubbo框架的核心接口，该接口中包含如何获取调用方法的名称、参数类型列表、参数列表以及绑定的数据，定义代码如下：  

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24fehzylsj30e009k74z.jpg)

代理中的 handler 实例中包含的 Invoker<T> 接口实现者是 MockClusterInvoker，其中MockClusterInvoker仅仅是一个 Invoker 的包装，并且也实现了接口Invoker<T>，其只是用于实现Dubbo框架中的mock功能，我们可以从他的 invoke 方法的实现中看出

 

# Dubbo插件化

Dubbo的插件化实现非常类似于原生的 JAVA 的 SPI ：它只是提供一种协议，并没有提供相关插件化实施的接口。用过的同学都知道，它有一种java原生的支持类：ServiceLoader，通过声明接口的实现类，在META-INF/services中注册一个实现类，然后通过ServiceLoader去生成一个接口实例，当更换插件的时候只需要把自己实现的插件替换到META-INF/services中即可。

 

## Dubbo的“SPI”

Dubbo 的 SPI 并非原生的 SPI，Dubbo 的规则是在`META-INF/dubbo`、`META-INF/dubbo/internal`或者`META-INF/services`下面以需要实现的接口去创建一个文件，并且在文件中以properties规则一样配置实现类的全面以及分配实现的一个名称。我们看一下dubbo-cluster模块的META-INF.dubbo.internal：

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24f3gpoyoj30sl07jdgt.jpg)

 

## 实现自己的扩展点

假如我们使用自己定义的协议 MyDefineProtocol

1. 在resources目录下新建META-INF/dubbo/com.alibaba.dubbo.rpc.Protocol文件，文件内容为com.***.MyDefineProtocol

2. 实现类的内容

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24ff074ltj30ej04jweg.jpg)

3. 最后在main方法中调用

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24ff8s60yj30wa0co75f.jpg)

4. 通过结果可以看到我们已经找到

 

## 源码分析

dubbo的扩展点框架主要位于这个包下：

​    com.alibaba.dubbo.common.extension

大概结构如下：

```
com.alibaba.dubbo.common.extension  
|  
|--factory  
|     |--AdaptiveExtensionFactory   #稍后解释  
|     |--SpiExtensionFactory        #稍后解释  
|  
|--support  
|     |--ActivateComparator  
|  
|--Activate  #自动激活加载扩展的注解  
|--Adaptive  #自适应扩展点的注解  
|--ExtensionFactory  #扩展点对象生成工厂接口  
|--ExtensionLoader   #扩展点加载器，扩展点的查找，校验，加载等核心逻辑的实现类  
|--@SPI   #扩展点注解 
```



### ExtensionLoader

#### getExtensionLoader

其中最核心的类就是`ExtensionLoader`，几乎所有特性都在这个类中实现。

ExtensionLoader没有提供public的构造方法，但是提供了一个public static的 getExtensionLoader，这个方法就是获取ExtensionLoader实例的工厂方法。其public成员方法中有三个比较重要的方法：

getActivateExtension ：根据条件获取当前扩展可自动激活的实现

getExtension ： 根据名称获取当前扩展的指定实现

getAdaptiveExtension : 获取当前扩展的自适应实现

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24ffiylr6j30j60baq45.jpg)

> 该方法需要一个Class类型的参数，该参数表示希望加载的扩展点类型，该参数必须是接口，且该接口必须被@SPI注解注释，否则拒绝处理。检查通过之后首先会检查ExtensionLoader缓存中是否已经存在该扩展对应的 ExtensionLoader，如果有则直接返回，否则创建一个新的ExtensionLoader负责加载该扩展实现，同时将其缓存起来。可以看到对于每一个扩展，dubbo中只会有一个对应的ExtensionLoader实例。



####ExtensionLoader的私有构造函数

接下来看下ExtensionLoader的私有构造函数：

 ![](https://ws1.sinaimg.cn/large/006xzusPgy1g24fhapcy1j30i104tmxh.jpg)

> 这里保存了对应的扩展类型，并且设置了一个额外的objectFactory属性，他是一个ExtensionFactory类型，ExtensionFactory主要用于加载扩展的实现：

### ExtensionFactory

ExtensionFactory主要用于加载扩展的实现：

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24fiyo1yuj30f906sq32.jpg)

> ExtensionFactory有@SPI注解，说明当前这个接口是一个扩展点。从extension包的结构图可以看到。Dubbo内部提供了两个实现类：SpiExtensionFactory和AdaptiveExtensionFactory。不同的实现可以以不同的方式来完成扩展点实现的加载。

### AdpativeExtensionFactory

默认的 ExtensionFactory 实现中，AdaptiveExtensionFactotry 被 @Adaptive注解 注释，也就是它就是ExtensionFactory对应的自适应扩展实现(每个扩展点最多只能有一个自适应实现，如果所有实现中没有被@Adaptive注释的，那么dubbo会动态生成一个自适应实现类)，也就是说，所有对ExtensionFactory调用的地方，实际上调用的都是AdpativeExtensionFactory，那么我们看下他的实现代码：

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24fjk3po1j30hu0epmyb.jpg)

> 这段代码，其实就相当于一个代理入口，它会遍历当前系统中所有的ExtensionFactory实现来获取指定的扩展实现，获取到扩展实现，遍历完所有ExtensionFactory实现，调用ExtensionLoader的getSupportedExtensions方法来获取ExtensionFactory的所有实现

#### getAdaptiveExtension

从前面ExtensionLoader的私有构造函数中可以看出，在选择ExtensionFactory的时候，并不是调用getExtension(name)来获取某个具体的实现类，而是调用getAdaptiveExtension来获取一个自适应的实现。那么首先我们就来分析一下getAdaptiveExtension这个方法的实现吧：.

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24fp497daj30hs0dqdgt.jpg)

> 首先检查缓存的adaptiveInstance是否存在，如果存在则直接使用，否则的话调用createAdaptiveExtension方法来创建新的adaptiveInstance并且缓存起来。也就是说对于某个扩展点，每次调用ExtensionLoader.getAdaptiveExtension获取到的都是同一个实例。



#### createAdaptiveExtension

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24fpffu09j30hu04o74j.jpg)

> 在createAdaptiveExtension方法中，首先通过getAdaptiveExtensionClass方法获取到最终的自适应实现类型，然后实例化一个自适应扩展实现的实例，最后进行扩展点注入操作

#### getAdaptiveExtensionClass

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24fpmwiwjj30hp04rt91.jpg)

> 他只是简单的调用了getExtensionClasses方法，然后在判adaptiveCalss缓存是否被设置，如果被设置那么直接返回，否则调用createAdaptiveExntesionClass方法动态生成一个自适应实现，关于动态生成自适应实现类然后编译加载并且实例化

#### getExtensionClasses

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24frf0l7pj30hw07xq3e.jpg)

> 在getExtensionClasses方法中，首先检查缓存的cachedClasses，如果没有再调用loadExtensionClasses 方法来加载，加载完成之后就会进行缓存。也就是说对于每个扩展点，其实现的加载只会执行一次。我们看下loadExtensionClasses方法：

#### loadExtensionClasses

![](https://ws1.sinaimg.cn/large/006xzusPgy1g24frnhttdj30i20bi75d.jpg)

> 从代码里面可以看到，在loadExtensionClasses中首先会检测扩展点在@SPI注解中配置的默认扩展实现的名称，并将其赋值给cachedDefaultName属性进行缓存，后面想要获取该扩展点的默认实现名称就可以直接通过访问cachedDefaultName字段来完成，比如getDefaultExtensionName方法就是这么实现的。从这里的代码中又可以看到，具体的扩展实现类型，是通过调用loadFile方法来加载，分别从一下三个地方加载：META-INF/dubbo/internal/、META-INF/dubbo/   META-INF/services/



* loadFile

调用loadFile方法，代码比较长，主要做了几个事情，有几个变量会赋值

cachedAdaptiveClass : 当前Extension类型对应的AdaptiveExtension类型(只能一个)

cachedWrapperClasses : 当前Extension类型对应的所有Wrapper实现类型(无顺序)

cachedActivates : 当前Extension实现自动激活实现缓存(map,无序)

cachedNames : 扩展点实现类对应的名称(如配置多个名称则值为第一个)

 

当 loadExtensionClasses 方法执行完成之后，还有以下变量被赋值：

cachedDefaultName : 当前扩展点的默认实现名称

 

当getExtensionClasses方法执行完成之后，除了上述变量被赋值之外，还有以下变量被赋值：

cachedClasses : 扩展点实现名称对应的实现类(一个实现类可能有多个名称)

其实也就是说，在调用了getExtensionClasses方法之后，当前扩展点对应的实现类的一些信息就已经加载进来了并且被缓存了。后面的许多操作都可以直接通过这些缓存数据来进行处理了。

 

回到createAdaptiveExtension方法，他调用了getExtesionClasses方法加载扩展点实现信息完成之后，就可以直接通过判断cachedAdaptiveClass缓存字段是否被赋值盘确定当前扩展点是否有默认的AdaptiveExtension实现。如果没有，那么就调用createAdaptiveExtensionClass方法来动态生成一个。在dubbo的扩展点框架中大量的使用了缓存技术。

 

创建自适应扩展点实现类型和实例化就已经完成了，下面就来看下扩展点自动注入的实现injectExtension

### injectExtension

 ![](https://ws1.sinaimg.cn/large/006xzusPgy1g24frxer7pj30ii0g9gmy.jpg)

> 这里可以看到，扩展点自动注入的一句就是根据setter方法对应的参数类型和property名称从ExtensionFactory中查询，如果有返回扩展点实例，那么就进行注入操作。到这里getAdaptiveExtension 方法就分析完毕了。 

### getExtension

这个方法的主要作用是用来获取 ExtensionLoader 实例代表的扩展的指定实现。已扩展实现的名字作为参数，结合前面学习 getAdaptiveExtension 的代码

 ![](https://ws1.sinaimg.cn/large/006xzusPgy1g24ftzqltuj30g40bimxy.jpg)

 ![](https://ws1.sinaimg.cn/large/006xzusPgy1g24fu5j8azj30ee0bcmyd.jpg)

# 总结

在整个过程中，最重要的两个方法 getExtensionClasses 和 createAdaptiveExtensionClass

 

**getExtensionClasses**

这个方法主要是读取 META-INF/services 、META-INF/dubbo、META-INF/internal 目录下的文件内容

分析每一行，如果发现其中有哪个类的annotation是@Adaptive，就找到对应的AdaptiveClass。如果没有的话，就动态创建一个

**createAdaptiveExtensionClass**

该方法是在getExtensionClasses方法找不到AdaptiveClass的情况下被调用，该方法主要是通过字节码的方式在内存中新生成一个类，它具有AdaptiveClass的功能，Protocol就是通过这种方式获得AdaptiveClass类的。