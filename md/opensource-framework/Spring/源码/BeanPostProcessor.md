## 五个接口十个扩展点
### BeanPostProcessor Bean后置处理器（和初始化相关）

1. `postProcessBeforeInitialization`：实例化、依赖注入完毕。在调用**显示的初始化之前（init-method、InitializingBean等之前）**完成一些定制的初始化任务。如：

   * BeanValidationPostProcessor 完成JSR-303 @Valid注解Bean验证

   * InitDestroyAnnotationBeanPostProcessor：完成@PostConstruct注解的初始化方法调用(所以它是在init显示调用之前执行的)

   * ApplicationContextAwareProcessor完成一些Aware接口的注入（如EnvironmentAware、ResourceLoaderAware、ApplicationContextAware）

2. `postProcessAfterInitialization`： 整个初始化都完全完毕了。做一些事情，如

   * AspectJAwareAdvisorAutoProxyCreator：完成 xml 风格的 AOP 配置(aop:config)的目标对象包装到AOP代理对象
   * AnnotationAwareAspectJAutoProxyCreator：完成 @Aspectj 注解风格（aop:aspectj-autoproxy @Aspect）的AOP配置的目标对象包装到AOP代理对象



### MergedBeanDefinitionPostProcessor 合并Bean定义（继承BeanPostProcessor）

* postProcessMergedBeanDefinition：执行Bean定义的合并



### InstantiationAwareBeanPostProcessor 实例化Bean（继承BeanPostProcessor）

1. postProcessBeforeInstantiation：实例化之前执行。给调用者一个机会，返回一个代理对象（相当于可以摆脱 Spring 的束缚，可以自定义实例化逻辑） 若返回null，继续后续Spring的逻辑。若返回不为null，就最后面都仅仅只执行` BeanPostProcessor#postProcessAfterInitialization`这一个回调方法了。如：
   * 当 `AbstractAutoProxyCreato `r的实现者注册了 `TargetSourceCreator`（创建自定义的`TargetSource`）将会按照这个流程去执行。绝大多数情况下调用者不会自己去实现`TargetSourceCreator`，而是Spring采用默认的 `SingletonTargetSource `去生产AOP对象。 当然除了 `SingletonTargetSource`，我们还可以使用`ThreadLocalTargetSource`（线程绑定的Bean）、`CommonsPoolTargetSource`（实例池的Bean）等等

2. `postProcessAfterInstantiation`：实例化完毕后、初始化之前执行。 若方法返回false，表示后续的InstantiationAwareBeanPostProcessor 都不用再执行了。(一般不建议去返回false，它的意义在于若返回fasle不仅后续的不执行了，就连自己个的且包括后续的处理器的 `postProcessPropertyValues `方法都将不会再执行了） `populateBean() `的时候调用，若有返回false，下面的 `postProcessPropertyValues `就都不会调用了
3. `postProcessPropertyValues`：紧接着上面 postProcessAfterInstantiation 执行的（false解释如上）。如：
   * AutowiredAnnotationBeanPostProcessor执行@Autowired注解注入
   * CommonAnnotationBeanPostProcessor执行@Resource等注解的注入，
   * PersistenceAnnotationBeanPostProcessor执行@ PersistenceContext等JPA注解的注入，
   * RequiredAnnotationBeanPostProcessor执行@ Required注解的检查等等



### SmartInstantiationAwareBeanPostProcessor 智能实例化Bean（继承InstantiationAwareBeanPostProcessor）

1. predictBeanType：预测Bean的类型，返回第一个预测成功的Class类型，如果不能预测返回null； 当你调用BeanFactory.getType(name)时当通过Bean定义无法得到Bean类型信息时就调用该回调方法来决定类型信息。方法：getBeanNamesForType(,)会循环调用此方法~~~ 如：
   * BeanFactory.isTypeMatch(name, targetType)用于检测给定名字的Bean是否匹配目标类型（在依赖注入时需要使用）
2. determineCandidateConstructors：检测Bean的构造器，可以检测出多个候选构造器，再有相应的策略决定使用哪一个。 在createBeanInstance的时候，会通过此方法尝试去找到一个合适的构造函数。若返回null，可能就直接使用空构造函数去实例化了 如：
   * AutowiredAnnotationBeanPostProcessor：它会扫描Bean中使用了@Autowired/@Value注解的构造器从而可以完成构造器注入
3. getEarlyBeanReference：和循环引用相关了。当正在创建A时，A依赖B。此时会：将A作为ObjectFactory放入单例工厂中进行early expose，此处又需要引用A，但A正在创建，从单例工厂拿到ObjectFactory**（其通过getEarlyBeanReference获取及早暴露Bean)**从而允许循环依赖。
   * AspectJAwareAdvisorAutoProxyCreator或AnnotationAwareAspectJAutoProxyCreator他们都有调用此方法，通过early reference能得到正确的代理对象。 有个小细节：这两个类中若执行了getEarlyBeanReference，那postProcessAfterInitialization就不会再执行了。

> 因为我们经常实现SmartInstantiationAwareBeanPostProcessor或者InstantiationAwareBeanPostProcessor只想实现其中的某个方法，因此本处介绍Spring为我们提供的一个适配器：InstantiationAwareBeanPostProcessorAdapter，它做了所有的空实现。这样我们需要哪个自己去实现对应方法即可，减少了干扰



### DestructionAwareBeanPostProcessor 销毁Bean（继承BeanPostProcessor）
postProcessBeforeDestruction：销毁后处理回调方法，该回调只能应用到单例Bean。如
1.InitDestroyAnnotationBeanPostProcessor完成@PreDestroy注解的销毁方法调用

## Spring内置的一些BeanPostProcessor
Spring内置了非常非常多的处理器，这里只举例子：

### ApplicationContextAwareProcessor
它的postProcessBeforeInitialization方法里处ApplicationContextAware、MessageSourceAwareResourceLoaderAware、EnvironmentAware、EmbeddedValueResolverAware、ApplicationEventPublisherAware

### CommonAnnotationBeanPostProcessor
继承InitDestroyAnnotationBeanPostProcessor。

提供对JSR-250规范注解的支持@javax.annotation.Resource、@javax.annotation.PostConstruct和@javax.annotation.PreDestroy等的支持

postProcessPropertyValues：通过此回调进行@Resource注解的依赖注入；（最重要）
postProcessBeforeInitialization()将会调用bean的@PostConstruct方法；（次重要）
postProcessBeforeDestruction()将会调用单例 Bean的@PreDestroy方法

### AutowiredAnnotationBeanPostProcessor
提供对JSR-330规范注解（@Inject）的支持和Spring自带注解的支持（@Autowired和@Value）

determineCandidateConstructors：决定候选构造器；
postProcessPropertyValues：进行依赖注入

### AbstractAutoProxyCreator AOP中对处理器的典型应用
继承自SmartInstantiationAwareBeanPostProcessor。AspectJAwareAdvisorAutoProxyCreator（提供对<aop:config>的支持）和AnnotationAwareAspectJAutoProxyCreator（提供对@AspectJ注解的支持）都是继承AbstractAutoProxyCreator

当使用aop:config配置时自动注册AspectJAwareAdvisorAutoProxyCreator，而使用aop:aspectj-autoproxy时会自动注册AnnotationAwareAspectJAutoProxyCreator。

predictBeanType：预测Bean的类型，如果目标对象被AOP代理对象包装，此处将返回AOP代理对象的类型（而不是目标对象的类型）
postProcessBeforeInstantiation：当我们配置TargetSourceCreator进行自定义TargetSource创建时，会创建代理对象并中断默认Spring创建流程
getEarlyBeanReference：获取early Bean引用（只有单例Bean才能回调该方法） wrapIfNecessary(bean, beanName, cacheKey);
postProcessAfterInitialization：如果之前调用过getEarlyBeanReference获取包装目标对象到AOP代理对象（如果需要），则不再执行 。否则包装目标对象到AOP代理对象（如果需要） wrapIfNecessary(bean, beanName, cacheKey);

getEarlyBeanReference和postProcessAfterInitialization是二者选一的，而且单例Bean目标对象只能被增强一次，而原型Bean目标对象可能被包装多次

### MethodValidationPostProcessor
基于Hibernate Validator的校验。

提供对方法参数/方法返回值的进行验证（即前置条件/后置条件的支持），通过JSR-303注解验证，使用方式如：

public @NotNull Object myValidMethod(@NotNull String arg1, @Max(10) int arg2)
1
默认只对@org.springframework.validation.annotation.Validated(它是jsr303注解javax.validation.Valid的变体)注解的Bean进行验证，我们可以修改validatedAnnotationType为其他注解类型来支持其他注解验证。而且目前只支持Hibernate Validator实现，在未来版本可能支持其他实现。

有了这东西之后我们就不需要在进行如Assert.assertNotNull（）这种前置条件/后置条件的判断了。

### ScheduledAnnotationBeanPostProcessor
当配置文件中有task:annotation-driven自动注册或@EnableScheduling自动注册。（提供对注解@Scheduled任务调度的支持）

postProcessAfterInitialization：通过查找Bean对象类上的@Scheduled注解来创建ScheduledMethodRunnable对象并注册任务调度方法（仅返回值为void且方法是无形式参数的才可以）。

### AsyncAnnotationBeanPostProcessor
当配置文件中有task:annotation-driven自动注册或@EnableAsync自动注册。

提供对@ Async和EJB3.1的@javax.ejb.Asynchronous注解的异步调用支持。
postProcessAfterInitialization：通过ProxyFactory创建目标对象的代理对象，默认使用AsyncAnnotationAdvisor（内部使用AsyncExecutionInterceptor 通过AsyncTaskExecutor（继承TaskExecutor）通过submit提交异步任务）。

### ServletContextAwareProcessor
在使用Web容器时自动注册。类似于ApplicationContextAwareProcessor，当你的Bean 实现了ServletContextAware/ ServletConfigAware会自动调用回调方法注入ServletContext/ ServletConfig。

### BeanPostProcessor的执行顺序
1. 如果使用BeanFactory实现，非ApplicationContext实现，BeanPostProcessor执行顺序就是添加顺序。
2. 如果使用的是AbstractApplicationContext（实现了ApplicationContext）的实现，则通过如下规则指定顺序。PriorityOrdered > Ordered > 无实现接口的 > 内部Bean后处理器（实现了MergedBeanDefinitionPostProcessor接口的是内部Bean PostProcessor，将在最后且无序注册）



## 总结
Spring提供了多种方式，让我们可以参与控制Bean的生命周期，从而达到我们的业务目的。相信有了这些功能之后，我们处理起来就会更加的游刃有余了。

备注：
1、实现InitializingBean接口是直接调用afterPropertiesSet方法，而init-method是通过反射来实现，效率较低，但是init-method方式消除了对spring的依赖。
2、InitializingBean接口实现先于init-method方法，如果调用afterPropertiesSet方法时出错，则不调用init-method指定的方法（画外音：若不出错，就都会调用）。



















