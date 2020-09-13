

# Spring Boot四大核心

![1.png](http://ww1.sinaimg.cn/large/006xzusPly1g9wc9ph2ktj30mt0gcq6e.jpg)

[九九八八](https://blog.csdn.net/andy_zhang2007)

正确理解 @springbootapplicaiton 注解的作用

### CLI

### Starter

SpringFactoriesLoader

- 1、选择已有的starters，在此基础上进行扩展.
- 2、创建自动配置文件并设定META-INF/spring.factories里的内容.spi
- 3、发布你的starter

### Auto Configuration（自动装配）

实现：@configuration + @condition 系列注解

location：META-INF/spring.factories

@ConfigurationProperties 并不会视作 标注bean实例化

@EnableConfigurationProperties

#### 注解

@ConditionalOnBean（仅仅在当前上下文中存在某个对象时，才会实例化一个Bean）
@ConditionalOnClass（某个class位于类路径上，才会实例化一个Bean）
@ConditionalOnExpression（当表达式为true的时候，才会实例化一个Bean）
@ConditionalOnMissingBean（仅仅在当前上下文中不存在某个对象时，才会实例化一个Bean）
@ConditionalOnMissingClass（某个class类路径上不存在的时候，才会实例化一个Bean）
@ConditionalOnNotWebApplication（不是web应用）



@ConditionalOnClass：该注解的参数对应的类必须存在，否则不解析该注解修饰的配置类；
@ConditionalOnMissingBean：该注解表示，如果存在它修饰的类的bean，则不需要再创建这个bean；可以给该注解传入参数例如@ConditionOnMissingBean(name = "example")，这个表示如果name为“example”的bean存在，这该注解修饰的代码块不执行。



了解源码中  @ConditionalOnXXX 的使用方式





#### 自动装配顺序

- 在特定自动装配Class之前 
  - @AutoConfigureBefore
- 在特定自动装配Class之后 
  - @AutoConfigureAfter
- 指定顺序 
  - @AutoConfigureOrder



/autoconfig 匹配原则

### 扩展SPI机制

SpringApplication#run

SpringFactoriesLoader#loadFactoryNames 加载配置文件中`spring.factories`指定 key 配置的所有类(根据 ClassLoader，加载对应 jar 包下/META-INF/spring.factories)

- [各种PostProcessor](https://blog.csdn.net/andy_zhang2007/article/details/78595558)





#### ConfigurationClassPostProcessor

ConfigurationClassPostProcessor 是我们实现注解的核心类，改类在spring启动的时候，会去加载该类到spring容器当中，因为该类是BeanDefinitionRegistryPostProcessor的子类，在spring生命周期当中它会去执行其相应的方法。

大致处理流程就是： 
1、先去扫描已经被@Component所注释的类，当然会先判断有没有@Condition相关的注解。 
2、然后递归的取扫描该类中的@ImportResource，@PropertySource，@ComponentScan，@Bean，@Import。一直处理完。



Deferred 延迟

**DeferredImportSelector** 这个实现的类会执行当所有的 @Configuration 注解的 bean 被处理后执行。这个类主要是用来处理 @Conditional 的。

EnableAutoConfigurationImportSelector 实现了 DeferredImportSelector



ConfigurationClassPostProcessor在扫描@Component时候会去判断该类是否有@Import并且判断是不是实现了 **ImportSelector** 或者 **ImportBeanDefinitionRegistrar** 不是的话就按照普通的 @Configuration 放在spring容器当中。



@EnableAutoConfiguration 使用了 @Import，然后通过 selectImports 从 META-INF/spring.factories 中取出想要的配置，将这些类注册到 spring 容器当中

#### 配置的注解

ConfigurationClassPostProcessor 处理注解

@Configuration 

@ConfigurationProperties 

@EnableConfigurationProperties

@ImportResource 

@PropertySource        properties，yaml

@ComponentScan

@Bean

@Import

@Conditional

```
Condition#matches
```

- @Value  与  @ConfigurableProperties 区别？

@Value 平铺式

@ConfigurableProperties("server.tomcat")  树形

@EnableConfigurableProperties

离应用越近的配置越优先 ？？？？



ConfigFileApplicationContextInitializer

PropertySourceLoader

Loader



#### ImportBeanDefinitionRegistrar

- 如果让你写一个新的jar，要求使用到了spring自动配置如何做？

  1. 我们只需要在自己的jar下面的配置文件INF/spring.factories写入

     ```
     org.springframework.boot.autoconfigure.EnableAutoConfiguration=这里是我们希望spring容器启动时希望自己加载配置的地方。
     ```

  2. 使用@EnableAutoConfiguration注解即可

#### Actuator





# SpringBoot 执行流程

> * 自动加载 META-INF/spring.factories 中特定的类
>
> @SpringBootApplication -> 
>
> @EnableAutoConfiguration -> 
>
> AutoConfigurationImportSelector -> 
>
> selectImport
>
>
>
> xxxPostProcessor  -> @import -> xxxImportSelector -> selectImport

### SpringApplication#run

* SpringApplication

> getSpringFactoriesInstances(ApplicationContextInitializer.class))
>
> getSpringFactoriesInstances(ApplicationListener.class) 加载并实例化
>
> META-INF/spring.factories 中 key 为  ApplicationContextInitializer、ApplicationListener 所有类

* \#run

事件 Event 匹配 Listener

事件监听机制：自动注册

```
        StopWatch stopWatch = new StopWatch();
		stopWatch.start();
		ConfigurableApplicationContext context = null;
		Collection<SpringBootExceptionReporter> exceptionReporters = new ArrayList<>();

		configureHeadlessProperty();
		// META-INF/spring.factories 中 key 为 SpringApplicationRunListeners
		SpringApplicationRunListeners listeners = getRunListeners(args);
		// ApplicationListener -> ApplicationStartingEvent
		listeners.starting();
		try {
			ApplicationArguments applicationArguments = new DefaultApplicationArguments(args);
			// 加载主要配置文件
			// ConfigFileApplicationListener -> ApplicationEnvironmentPreparedEvent
			ConfigurableEnvironment environment = prepareEnvironment(listeners,
					applicationArguments);
			configureIgnoreBeanInfo(environment);
			Banner printedBanner = printBanner(environment);
			context = createApplicationContext();
			exceptionReporters = getSpringFactoriesInstances(
					SpringBootExceptionReporter.class,
					new Class[] { ConfigurableApplicationContext.class }, context);
			prepareContext(context, environment, listeners, applicationArguments,
					printedBanner);
			// 刷新容器
			refreshContext(context);
			afterRefresh(context, applicationArguments);
			stopWatch.stop();
			if (this.logStartupInfo) {
				new StartupInfoLogger(this.mainApplicationClass)
						.logStarted(getApplicationLog(), stopWatch);
			}
			listeners.started(context);
			callRunners(context, applicationArguments);
		}
```

#### ConfigFileApplicationListener

```
private void onApplicationEnvironmentPreparedEvent(
			ApplicationEnvironmentPreparedEvent event) {
		List<EnvironmentPostProcessor> postProcessors = loadPostProcessors();
		// ConfigFileApplicationListener 加入处理器列表
		postProcessors.add(this);
		
		AnnotationAwareOrderComparator.sort(postProcessors);
		// 环境设置
		for (EnvironmentPostProcessor postProcessor : postProcessors) {
			postProcessor.postProcessEnvironment(event.getEnvironment(),
					event.getSpringApplication());
		}
	}
	

protected void addPropertySources(ConfigurableEnvironment environment,
			ResourceLoader resourceLoader) {
		RandomValuePropertySource.addToEnvironment(environment);
		new Loader(environment, resourceLoader).load();
}

// PropertySourceLoader对应类 加载实例化
Loader(ConfigurableEnvironment environment, ResourceLoader resourceLoader) {
			this.environment = environment;
			this.resourceLoader = (resourceLoader != null) ? resourceLoader
					: new DefaultResourceLoader();
			this.propertySourceLoaders = SpringFactoriesLoader.loadFactories(
					PropertySourceLoader.class, getClass().getClassLoader());
		}
		
// application.properties		
load(null, this::getNegativeProfileFilter,
					addToLoaded(MutablePropertySources::addFirst, true));
```



### AnnotationConfigServletWebServerApplicationContext