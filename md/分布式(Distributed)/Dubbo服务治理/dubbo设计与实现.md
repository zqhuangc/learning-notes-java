> AnnotationAttributes  对比  Annotation
> AnnotatedTypeMetadata

## 注解驱动（处理 bean 注册）

### AnnotationAttributes 和 AnnotationMetadata

### AnnotatedElementUtils　和　AnnotationUtils

AnnotatedElement

> SimpleMetadataReader
>
> SimpleAnnotationMetadataReadingVisitor

### xml 装配

```
DubboNamespaceHandler
DubboBeanDefinitionParser
AnnotationBeanDefinitionParser
```

只能处理单层标签，处理不了嵌套    相比      spring  不超过 3层

serviceBean   初始化问题   initializeBean#afterpropertyset      重构解决：提前绑定





###　＠DubboComponentScan

> 注意：各核心类的生命周期
>
> 旧版本注解兼容怎么做？
>

@SevletComponentScan

```
...
@Import(DubboComponentScanRegistrar.class)
public @interface DubboComponentScan {

    /**
     * Alias for the {@link #basePackages()} attribute. Allows for more concise annotation
     * declarations e.g.: {@code @DubboComponentScan("org.my.pkg")} instead of
     * {@code @DubboComponentScan(basePackages="org.my.pkg")}.
     *
     * @return the base packages to scan
     */
    String[] value() default {};
    ...
}
```

> DubboBootstrapApplicationListener
> DubboLifecycleComponentApplicationListener

#### DubboComponentScanRegistrar（ImportBeanDefinitionRegistrar）

BeanRegistrar

```java
public class DubboComponentScanRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        Set<String> packagesToScan = getPackagesToScan(importingClassMetadata);

        registerServiceAnnotationBeanPostProcessor(packagesToScan, registry);

        registerReferenceAnnotationBeanPostProcessor(registry);

    }
    ...
```



#### ServiceAnnotationBeanPostProcessor（BeanDefinitionRegistryPostProcessor）服务 bean 注册

@Service 处理

**BeanNameGenerator**

##### postProcessBeanDefinitionRegistry

```java
public class ServiceAnnotationBeanPostProcessor implements BeanDefinitionRegistryPostProcessor, EnvironmentAware,
        ResourceLoaderAware, BeanClassLoaderAware {
     public ServiceAnnotationBeanPostProcessor(...packagesToScan){...}
     ...      
     @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) throws BeansException {

        // @since 2.7.5
        registerBeans(registry, DubboBootstrapApplicationListener.class);

        Set<String> resolvedPackagesToScan = resolvePackagesToScan(packagesToScan);

        if (!CollectionUtils.isEmpty(resolvedPackagesToScan)) {
            //(参考Spring @Component处理)
            registerServiceBeans(resolvedPackagesToScan, registry);
        } else {
            if (logger.isWarnEnabled()) {
                logger.warn("packagesToScan is empty , ServiceBean registry will be ignored!");
            }
        }

    }
    ...
        
}
```

##### registerServiceBeans 和 registerServiceBean

```java
// 注册标记了 @org.apache.dubbo.config.annotation.Service 的类
private void registerServiceBeans(Set<String> packagesToScan, BeanDefinitionRegistry registry) {

        DubboClassPathBeanDefinitionScanner scanner =
                new DubboClassPathBeanDefinitionScanner(registry, environment, resourceLoader);

        BeanNameGenerator beanNameGenerator = resolveBeanNameGenerator(registry);

        scanner.setBeanNameGenerator(beanNameGenerator);

        scanner.addIncludeFilter(new AnnotationTypeFilter(Service.class));

        /**
         * Add the compatibility for legacy Dubbo's @Service
         *
         * The issue : https://github.com/apache/dubbo/issues/4330
         * @since 2.7.3
         */
        scanner.addIncludeFilter(new AnnotationTypeFilter(com.alibaba.dubbo.config.annotation.Service.class));

        for (String packageToScan : packagesToScan) {

            // Registers @Service Bean first
            scanner.scan(packageToScan);

            // Finds all BeanDefinitionHolders of @Service whether @ComponentScan scans or not.
            Set<BeanDefinitionHolder> beanDefinitionHolders =
                    findServiceBeanDefinitionHolders(scanner, packageToScan, registry, beanNameGenerator);

            if (!CollectionUtils.isEmpty(beanDefinitionHolders)) {

                for (BeanDefinitionHolder beanDefinitionHolder : beanDefinitionHolders) {
                    registerServiceBean(beanDefinitionHolder, registry, scanner);
                }

                if (logger.isInfoEnabled()) {
                    logger.info(beanDefinitionHolders.size() + " annotated Dubbo's @Service Components { " +
                            beanDefinitionHolders +
                            " } were scanned under package[" + packageToScan + "]");
                }

            } else {

                if (logger.isWarnEnabled()) {
                    logger.warn("No Spring Bean annotating Dubbo's @Service was found under package["
                            + packageToScan + "]");
                }

            }

        }

    }
```



```
buildServiceBeanDefinition  使用时机？local @Service referenceBean注册时
ServiceBean 只用于local @Service 的
```

@Service

LinkedHashSet

##### AnnotationPropertyValuesAdapter

处理默认 annotation attributes 属性值



#### DubboClassPathBeanDefinitionScanner



为何不直接使用 ClassPathBeanDefinitionScanner？ exposes some methods to be public

dubbo中有体现吗？还是为了提供扩展？

```
/**
 * Dubbo {@link ClassPathBeanDefinitionScanner} that exposes some methods to be public.
 *
 * @see #doScan(String...) //方法暴露获取 BeanDefinitionHolder
 * @see #registerDefaultFilters() // useDefault=false
 * @since 2.5.7
 */
    public class DubboClassPathBeanDefinitionScanner extends ClassPathBeanDefinitionScanner {
 
    @Override
    public Set<BeanDefinitionHolder> doScan(String... basePackages) {
        return super.doScan(basePackages);
    }
    
  }

```

> doScan 方法暴露后好像没用到？
>
> findServiceBeanDefinitionHolders  算优化还是重新实现？

##### resolveBeanNameGenerator

```
//为何？ CONFIGURATION_BEAN_NAME_GENERATOR setBeanNameGenerator
if (registry instanceof SingletonBeanRegistry) {
    SingletonBeanRegistry singletonBeanRegistry = SingletonBeanRegistry.class.cast(registry);
    beanNameGenerator = (BeanNameGenerator) singletonBeanRegistry.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);
}
```

##### MetadataReader



#### ReferenceAnnotationBeanPostProcessor

##### AnnotationInjectedBeanPostProcessor（Deprecated）

##### AbstractAnnotationBeanPostProcessor



 ReferenceAnnotationBeanPostProcessor -> AbstractAnnotationBeanPostProcessor
(参考Spring AutowiredAnnotationBeanPostProcessor)

##### buildReferenceBeanIfAbsent 和 registerReferenceBean

怎么保证 DubboBootstrap 在  registerReference#get 前已经初始化化

referencebean   在本地服务未暴露时如何做？ 监听 ServiceBeanExportedEvent 

## 外部化配置

> ImportBeanDefinitionRegistrar#registerBeanDefinitions
>
> @EnableDubboConfig -> @Import DubboConfigConfigurationRegistrar
>
> @EnableConfigurationBeanBinding  -> @Import ConfigurationBeanBindingRegistrar
>
> DubboConfigAliasPostProcessor
>
> NamePropertyDefaultValueDubboConfigBeanCustomizer
>
> ConfigurationBindingBeanPostProcessor
>

### DubboConfigConfigurationRegistrar（ImportBeanDefinitionRegistrar）

```
@Override
public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

    AnnotationAttributes attributes = AnnotationAttributes.fromMap(
            importingClassMetadata.getAnnotationAttributes(EnableDubboConfig.class.getName()));

    boolean multiple = attributes.getBoolean("multiple");

    // Single Config Bindings
    registerBeans(registry, DubboConfigConfiguration.Single.class);

    if (multiple) { // Since 2.6.6 https://github.com/apache/dubbo/issues/3193
        registerBeans(registry, DubboConfigConfiguration.Multiple.class);
    }
    // @EnableConfigurationBeanBindings

    // Register DubboConfigAliasPostProcessor
    registerDubboConfigAliasPostProcessor(registry);

    // Register NamePropertyDefaultValueDubboConfigBeanCustomizer
    registerDubboConfigBeanCustomizers(registry);

}
```

### ConfigurationBeanBindingsRegister 

### ConfigurationBeanBindingRegistrar

配置信息与类型的绑定（映射）

#### registerConfigurationBeanDefinitions

BeanDefinition补充注解信息

```
PropertySourcesUtils.getSubProperties(environment.getPropertySources(), environment, prefix);
解析获取 如 dubbo.applicaiton. 为前缀的 子配置信息 
```

#### ConfigurationBindingBeanPostProcessor（BeanFactoryPostProcessor, BeanPostProcessor ）

**配置信息绑定的实际实现类**

```
 @Override
    public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {

        this.beanFactory = beanFactory;

        initConfigurationBeanBinder();

        initBindConfigurationBeanCustomizers();
    }
    
```



```
@Override
public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {

    BeanDefinition beanDefinition = getNullableBeanDefinition(beanName);

    if (isConfigurationBean(bean, beanDefinition)) {
        bindConfigurationBean(bean, beanDefinition);
        // 默认实现主要是关于 name 的
        customize(beanName, bean);
    }

    return bean;
}
```

### ConfigurationBeanBinder

绑定主要是对绑定配置信息的初始化

DataBinder

### 泛化服务（GernericService）





# DubboBootstrap

start()

init()



```
ExchangeServer
NettyServer
```









## dubbo native

如何解决或缓解注册中心压力过载？

* 注册中心内存压力
* 注册中心网络压力
* 注册中心通知压力



如何控制应用粒度的服务







