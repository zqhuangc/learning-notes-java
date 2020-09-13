mercyblitz/jsr

幂等

PUT 

初始状态：0

修改状态：1 * N

最终状态：1



DELETE

初始状态：1

修改状态：0 * N

最终状态：0



非幂等

POST

初始状态：1

修改状态：1 + 1 =2 

N次修改： 1+ N = N+1

最终状态：N+1



幂等/非幂等 依赖于服务端实现，这种方式是一种契约



#自描述消息（MessageConverter）

>  Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8


第一优先顺序：text/html -> application/xhtml+xml -> application/xml

第二优先顺序：image/webp -> image/apng



《Spring Boot 编程思想》



学习源码的路径：

@EnableWebMvc

​DelegatingWebMvcConfiguration

​		WebMvcConfigurationSupport#addDefaultHttpMessageConverters

所有的 HTTP 自描述消息处理器均在 messageConverters（类型：`HttpMessageConverter`)，这个集合会传递到 RequestMappingHandlerAdapter，最终控制写出。

application/json 在SpringBoot中默认使用jackson2


messageConverters，其中包含很多自描述消息类型的处理，比如 JSON、XML、TEXT等等



以 application/json 为例，Spring Boot 中默认使用 Jackson2 序列化方式，其中媒体类型：application/json，它的处理类 MappingJackson2HttpMessageConverter，提供两类方法：

1. 读read* ：通过 HTTP 请求内容转化成对应的 Bean
2. 写write*： 通过 Bean 序列化成对应文本内容作为响应内容



问题：为什么第一次是JSON，后来怎加了 XML 依赖，又变成了 XML 内用输出

回答：Spring Boot 应用默认没有增加XML 处理器（HttpMessageConverter）实现，所以最后采用轮询的方式去逐一尝试是否可以 canWrite(POJO) ,如果返回 true，说明可以序列化该 POJO 对象，那么 Jackson 2 恰好能处理，那么Jackson 输出了。

### 问答

问题：当 Accept 请求头未被制定时，为什么还是 JSON 来处理

回答：这个依赖于 messageConverters 的插入顺序。



问题：优先级是默认的是吧 可以修改吗

回答：是可以调整的，通过extendMessageConverters 方法调整



### 扩展自描述消息

Person

JSON 格式（application/json)

```json
{
	"id":1,
	"name":"小马哥"
}
```

XML 格式（application/xml）

```xml
<Person>
    <id>1</id>
    <name>小马哥</name>
</Person>
```

Properties 格式（application/properties+person)

（需要扩展）

```properties
person.id = 1
person.name = 小马哥
```



1. 实现 AbstractHttpMessageConverter 抽象类
   1. supports 方法：是否支持当前POJO类型
   2. readInternal 方法：读取 HTTP 请求中的内容，并且转化成相应的POJO对象（通过 Properties 内容转化成 JSON）
   3. writeInternal 方法：将 POJO 的内容序列化成文本内容（Properties格式），最终输出到 HTTP 响应中（通过 JSON 内容转化成 Properties ）

* @RequestMappng 中的 consumes 对应 请求头 “Content-Type”
* @RequestMappng 中的 produces   对应 请求头 “Accept”



HttpMessageConverter 执行逻辑：

 * 读操作：尝试是否能读取，canRead 方法去尝试，如果返回 true 下一步执行 read
* 写操作：尝试是否能写入，canWrite 方法去尝试，如果返回 true 下一步执行 write

如果请求中accept中和produces不一样，controller能执行吗？
不能执行 406 415

MediaType */*

为什么一定要经过 POJO 
JSON -> 反序列化 -> Map -> Properties  变复杂了



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