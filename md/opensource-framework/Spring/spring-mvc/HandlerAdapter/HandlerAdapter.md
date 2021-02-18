# HandlerAdapter

```java
public interface HandlerAdapter {

	// 判断当前的这个HandlerAdapter  是否 支持给与的handler
	// 因为一般来说：每个适配器只能作用于一种处理器（你总不能把手机适配器拿去用于电脑吧）
    // HandlerExecutionChain HandlerMapping.getHandler(HttpServletRequest)
	boolean supports(Object handler);
	
	// 核心方法：利用 Handler 处理请求，然后返回一个ModelAndView 
	// DispatcherServlet最终就是调用此方法，来返回一个ModelAndView的~
	@Nullable
	ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception;
	// 同HttpServlet 的 getLastModified方法
	// Can simply return -1 if there's no support in the handler class.
	long getLastModified(HttpServletRequest request, Object handler);

}
```



####　为何需要使用HandlerAdapter适配？

- `HandlerMapping`的作用主要是根据request请求匹配/映射上能够处理当前request的handler
- `HandlerAdapter`的作用在于将request中的各个属性，如`request param`适配为handler能够处理的形式，参数绑定、数据校验、内容协商…几乎所有的web层问题都在在这里完成的。

Spring MVC的Handler（Controller接口，HttpRequestHandler，Servlet、@RequestMapping）有四种表现形式，在Handler不确定是什么方式的时候（可能是方法、也可能是类），适配器这种设计模式就能模糊掉具体的实现，从而就能提供统一访问接口。

### SimpleControllerHandlerAdapter

```java
// 适配`org.springframework.web.servlet.mvc.Controller`这种Handler
public class SimpleControllerHandlerAdapter implements HandlerAdapter {

	@Override
	public boolean supports(Object handler) {
		return (handler instanceof Controller);
	}
	// 最终执行逻辑的还是Handler啊~~~~
	@Override
	@Nullable
	public ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler)
			throws Exception {
		return ((Controller) handler).handleRequest(request, response);
	}
	
	// 此处注意：若处理器实现了`LastModified`接口，那就委托给它了
	// 否则返回-1  表示不要缓存~
	@Override
	public long getLastModified(HttpServletRequest request, Object handler) {
		if (handler instanceof LastModified) {
			return ((LastModified) handler).getLastModified(request);
		}
		return -1L;
	}

}
```

### HttpRequestHandlerAdapter

适配`org.springframework.web.HttpRequestHandler`这种Handler。

### SimpleServletHandlerAdapter

适配`javax.servlet.Servlet`这种Handler。

## AbstractHandlerMethodAdapter

主要是支持到了`org.springframework.web.method.HandlerMethod`这种处理器

```java
// @since 3.1 @RequestMapping注解是Spring2.5出现的
// 注意：它实现了Ordered接口
public abstract class AbstractHandlerMethodAdapter extends WebContentGenerator implements HandlerAdapter, Ordered {
	
	// 唯一构造函数。传的false表示：忽略掉supportedMethods这个属性
	// 默认它的值是GET、POST、HEAD（见WebContentGenerator）
	public AbstractHandlerMethodAdapter() {
		// no restriction of HTTP methods by default
		super(false);
	}

	// 只处理HandlerMethod 类型的处理器。抽象方法supportsInternal默认返回true
	// 是流出的钩子可以给你自己扩展的
	@Override
	public final boolean supports(Object handler) {
		return (handler instanceof HandlerMethod && supportsInternal((HandlerMethod) handler));
	}

	// 抽象方法交给子类handleInternal去实现
	@Override
	@Nullable
	public final ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
		return handleInternal(request, response, (HandlerMethod) handler);
	}
	...
}
```





### `RequestMappingHandlerAdapter`

* HandlerMethodReturnValueHandler
* HttpMessageConverter



用于适配@RequestMapping注解标注的Handler（Handler类型为org.springframework.web.method.HandlerMethod），继承自父类AbstractHandlerMethodAdapter。
它是自Spring3.1新增的一个适配器类（HandlerMethod也是3.1后出现的），拥有数据绑定、数据转换、数据校验、内容协商…等一系列非常高级的功能。

```java
// @since 3.1 实现了InitializingBean接口和BeanFactoryAware
public class RequestMappingHandlerAdapter extends AbstractHandlerMethodAdapter implements BeanFactoryAware, InitializingBean {
	// 唯一构造方法：默认注册一些消息转换器。
	// 开启@EnableWebMvc后此默认行为会被setMessageConverters()方法覆盖
	public RequestMappingHandlerAdapter() {
		StringHttpMessageConverter stringHttpMessageConverter = new StringHttpMessageConverter();
		stringHttpMessageConverter.setWriteAcceptCharset(false);  // see SPR-7316

		this.messageConverters = new ArrayList<>(4);
		this.messageConverters.add(new ByteArrayHttpMessageConverter());
		this.messageConverters.add(stringHttpMessageConverter);
		try {
			this.messageConverters.add(new SourceHttpMessageConverter<>());
		} catch (Error err) {
			// Ignore when no TransformerFactory implementation is available
		}
		this.messageConverters.add(new AllEncompassingFormHttpMessageConverter());
	}
	
	// 此方法在此Bean初始化的时候会执行：扫描解析容器内的@ControllerAdvice...
	// 方法体看起来代码不多，但其实每个方法内部，都可谓是个庞然大物，请详细观察理解~~~~
	@Override
	public void afterPropertiesSet() {
		// Do this first, it may add ResponseBody advice beans
		// 详见下面的解释分析
		initControllerAdviceCache();

		// 这三大部分，可是 "参数自动组装" 相关的组件~~~~每一份都非常的重要
		if (this.argumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
			this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		if (this.initBinderArgumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
			this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		if (this.returnValueHandlers == null) {
			List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
			this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
		}
	}
	

}
```

#### initControllerAdviceCache()

此方法用于初始化`@ControllerAdvice`标注的Bean，并解析此Bean内部各部分（`@ModelAttribute、@InitBinder`、`RequestBodyAdvice`和`ResponseBodyAdvice`接口）然后缓存起来。

```java
RequestMappingHandlerAdapter：

// ======================相关成员变量======================
// 装载RequestBodyAdvice和ResponseBodyAdvice的实现类们~
private List<Object> requestResponseBodyAdvice = new ArrayList<>();
// MethodIntrospector.selectMethods的过滤器。
// 这里意思是：含有@ModelAttribute，但是但是但是不含有@RequestMapping注解的方法~~~~~
public static final MethodFilter MODEL_ATTRIBUTE_METHODS = method -> (!AnnotatedElementUtils.hasAnnotation(method, RequestMapping.class) && AnnotatedElementUtils.hasAnnotation(method, ModelAttribute.class));
// 标注了注解@InitBinder的方法~~~
public static final MethodFilter INIT_BINDER_METHODS = method -> AnnotatedElementUtils.hasAnnotation(method, InitBinder.class);
// 存储标注了@ModelAttribute注解的方法的缓存~~~~
private final Map<ControllerAdviceBean, Set<Method>> modelAttributeAdviceCache = new LinkedHashMap<>();
// 存储标注了@InitBinder注解的方法的缓存~~~~
private final Map<ControllerAdviceBean, Set<Method>> initBinderAdviceCache = new LinkedHashMap<>();


	private void initControllerAdviceCache() {
		if (getApplicationContext() == null) {
			return;
		}
		
		// 拿到容器内所有的标注有@ControllerAdvice的组件们
		// BeanFactoryUtils.beanNamesForTypeIncludingAncestors(context, Object.class)
		// .filter(name -> context.findAnnotationOnBean(name, ControllerAdvice.class) != null)
		// .map(name -> new ControllerAdviceBean(name, context)) // 使用ControllerAdviceBean包装起来，持有name的引用（还木实例化哟）
		// .collect(Collectors.toList());
		
		// 因为@ControllerAdvice注解可以指定包名等属性，具体可参见HandlerTypePredicate的判断逻辑，是否生效
		// 注意：@RestControllerAdvice是@ControllerAdvice和@ResponseBody的结合体，所以此处也会被找出来
		// 最后Ordered排序
		List<ControllerAdviceBean> adviceBeans = ControllerAdviceBean.findAnnotatedBeans(getApplicationContext());
		AnnotationAwareOrderComparator.sort(adviceBeans);

		// 临时存储RequestBodyAdvice和ResponseBodyAdvice的实现类
		// 它哥俩是必须配合@ControllerAdvice一起使用的~
		List<Object> requestResponseBodyAdviceBeans = new ArrayList<>();

		for (ControllerAdviceBean adviceBean : adviceBeans) {
			Class<?> beanType = adviceBean.getBeanType();
			if (beanType == null) {
				throw new IllegalStateException("Unresolvable type for ControllerAdviceBean: " + adviceBean);
			}

			// 又见到了这个熟悉的方法selectMethods~~~~过滤器请参照成员变量
			// 含有@ModelAttribute，但是但是但是不含有@RequestMapping注解的方法~~~~~  找到之后放在全局变量缓存起来
			// 简单的说就是找到@ControllerAdvice里面所有的@ModelAttribute方法们
			Set<Method> attrMethods = MethodIntrospector.selectMethods(beanType, MODEL_ATTRIBUTE_METHODS);
			if (!attrMethods.isEmpty()) {
				this.modelAttributeAdviceCache.put(adviceBean, attrMethods);
			}
		
			// 找标注了注解@InitBinder的方法~~~（和有没有@RequestMapping木有关系了~~~）
			// 找到@ControllerAdvice里面所有的@InitBinder方法们
			Set<Method> binderMethods = MethodIntrospector.selectMethods(beanType, INIT_BINDER_METHODS);
			if (!binderMethods.isEmpty()) {
				this.initBinderAdviceCache.put(adviceBean, binderMethods);
			}

			// 这两个接口是Spring4.1 4.2提供的，实现了这两个接口的 
			// 此处先放在requestResponseBodyAdviceBeans里面装着 最后放到全局缓存requestResponseBodyAdvice里面去
			if (RequestBodyAdvice.class.isAssignableFrom(beanType) || ResponseBodyAdvice.class.isAssignableFrom(beanType)) {
				requestResponseBodyAdviceBeans.add(adviceBean);
			}
		}

		// 这个意思是，放在该list的头部。
		// 因为requestResponseBodyAdvice有可能通过set方法进来已经有值了~~~所以此处放在头部
		if (!requestResponseBodyAdviceBeans.isEmpty()) {
			this.requestResponseBodyAdvice.addAll(0, requestResponseBodyAdviceBeans);
		}

		// 输出debug日志...略（debug日志哦~）
		if (logger.isDebugEnabled()) {
			...
		}
	}
```

> 所有的被标注有此注解的Bean最终都变成一个`org.springframework.web.method.ControllerAdviceBean`，它内部持有Bean本身，以及判断逻辑器（`HandlerTypePredicate`）的引用



```java
@Override
	public void afterPropertiesSet() {
		...
		// 初始化参数解析器
		if (this.argumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
			this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		// 初始化@InitBinder的参数解析器
		if (this.initBinderArgumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultInitBinderArgumentResolvers();
			this.initBinderArgumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		// 初始化返回值解析器
		if (this.returnValueHandlers == null) {
			List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
			this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
		}
	}
```

##### ControllerAdviceBean

```java
// 找到容器内（包括父容器）所有的标注有@ControllerAdvice的Bean们~~~
	public static List<ControllerAdviceBean> findAnnotatedBeans(ApplicationContext context) {
		return Arrays.stream(BeanFactoryUtils.beanNamesForTypeIncludingAncestors(context, Object.class))
				.filter(name -> context.findAnnotationOnBean(name, ControllerAdvice.class) != null)
				.map(name -> new ControllerAdviceBean(name, context))
				.collect(Collectors.toList());
	}
```

##### RequestBodyAdvice

**允许body体转换为对象之前进行自定义定制；也允许该对象作为实参传入方法之前对其处理**。

```java
public interface RequestBodyAdvice {

	// 第一个调用的。判断当前的拦截器（advice是否支持） 
	// 注意它的入参有：方法参数、目标类型、所使用的消息转换器等等
	boolean supports(MethodParameter methodParameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType);

	// 如果body体木有内容就执行这个方法（后面的就不会再执行喽）
	Object handleEmptyBody(Object body, HttpInputMessage inputMessage, MethodParameter parameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType);
	
	// 重点：它在body被read读/转换**之前**进行调用的
	HttpInputMessage beforeBodyRead(HttpInputMessage inputMessage, MethodParameter parameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType) throws IOException;

	// 它在body体已经转换为Object后执行。so此时都不抛出IOException了嘛~
	Object afterBodyRead(Object body, HttpInputMessage inputMessage, MethodParameter parameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType);

}
```

###### JsonViewRequestBodyAdvice

```java
// @since 4.2
public class JsonViewRequestBodyAdvice extends RequestBodyAdviceAdapter {

	// 处理使用的消息转换器是AbstractJackson2HttpMessageConverter类型
	// 并且入参上标注有@JsonView注解的
	@Override
	public boolean supports(MethodParameter methodParameter, Type targetType, Class<? extends HttpMessageConverter<?>> converterType) {
		return (AbstractJackson2HttpMessageConverter.class.isAssignableFrom(converterType) &&
				methodParameter.getParameterAnnotation(JsonView.class) != null);
	}

	// 显然这里实现的beforeBodyRead这个方法：
	// 它把body最终交给了MappingJacksonInputMessage来反序列处理消息体
	// 注意：@JsonView能处理这个注解。也就是说能指定把消息体转换成指定的类型，还是比较实用的
	// 可以看到当标注有@jsonView注解后 targetType就没啥卵用了
	@Override
	public HttpInputMessage beforeBodyRead(HttpInputMessage inputMessage, MethodParameter methodParameter, Type targetType, Class<? extends HttpMessageConverter<?>> selectedConverterType) throws IOException {
		JsonView ann = methodParameter.getParameterAnnotation(JsonView.class);
		Assert.state(ann != null, "No JsonView annotation");

		Class<?>[] classes = ann.value();
		// 必须指定class类型，并且有且只能指定一个类型
		if (classes.length != 1) {
			throw new IllegalArgumentException("@JsonView only supported for request body advice with exactly 1 class argument: " + methodParameter);
		}
		// 它是一个InputMessage的实现
		return new MappingJacksonInputMessage(inputMessage.getBody(), inputMessage.getHeaders(), classes[0]);
	}

}
```



##### ResponseBodyAdvice

它允许在`@ResponseBody/ResponseEntity`标注的处理方法上在用`HttpMessageConverter`在写数据**之前做些**什么。

```java
// @since 4.1 泛型T：body类型
public interface ResponseBodyAdvice<T> {
	boolean supports(MethodParameter returnType, Class<? extends HttpMessageConverter<?>> converterType);
	@Nullable
	T beforeBodyWrite(@Nullable T body, MethodParameter returnType, MediaType selectedContentType, Class<? extends HttpMessageConverter<?>> selectedConverterType, ServerHttpRequest request, ServerHttpResponse response);
}
```

##### AbstractMappingJacksonResponseBodyAdvice

JsonViewResponseBodyAdvice

##### RequestResponseBodyAdviceChain

RequestResponseBodyMethodProcessor



#### getDefaultArgumentResolvers()

初始化`HandlerMethodArgumentResolver`，提供对方法参数的支持。也就是`@RequestMapping`的handler上能写哪些注解自动封装参数

```java
RequestMappingHandlerAdapter：

	// Return the list of argument resolvers to use including built-in resolvers and custom resolvers provided via {@link #setCustomArgumentResolvers}.
	// 返回内建的参数处理器们，以及用户自定义的一些参数处理器（注意顺序）
	private List<HandlerMethodArgumentResolver> getDefaultArgumentResolvers() {
		List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>();

		// Annotation-based argument resolution
		// 基于注解的
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
		resolvers.add(new RequestParamMapMethodArgumentResolver());
		resolvers.add(new PathVariableMethodArgumentResolver());
		resolvers.add(new PathVariableMapMethodArgumentResolver());
		resolvers.add(new MatrixVariableMethodArgumentResolver());
		resolvers.add(new MatrixVariableMapMethodArgumentResolver());
		resolvers.add(new ServletModelAttributeMethodProcessor(false));
		resolvers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RequestPartMethodArgumentResolver(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RequestHeaderMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new RequestHeaderMapMethodArgumentResolver());
		resolvers.add(new ServletCookieValueMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new SessionAttributeMethodArgumentResolver());
		resolvers.add(new RequestAttributeMethodArgumentResolver());

		// Type-based argument resolution
		// 基于type类型的
		resolvers.add(new ServletRequestMethodArgumentResolver());
		resolvers.add(new ServletResponseMethodArgumentResolver());
		resolvers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.requestResponseBodyAdvice));
		resolvers.add(new RedirectAttributesMethodArgumentResolver());
		resolvers.add(new ModelMethodProcessor());
		resolvers.add(new MapMethodProcessor());
		resolvers.add(new ErrorsMethodArgumentResolver());
		resolvers.add(new SessionStatusMethodArgumentResolver());
		resolvers.add(new UriComponentsBuilderMethodArgumentResolver());

		// Custom arguments
		// 用户自定义的
		if (getCustomArgumentResolvers() != null) {
			resolvers.addAll(getCustomArgumentResolvers());
		}

		// Catch-all
		// 兜底方案：这就是为何很多时候不写注解参数也能够被自动封装的原因
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
		resolvers.add(new ServletModelAttributeMethodProcessor(true));

		return resolvers;
	}
```

#### getDefaultInitBinderArgumentResolvers()

```java
RequestMappingHandlerAdapter：

	private List<HandlerMethodArgumentResolver> getDefaultInitBinderArgumentResolvers() {
		List<HandlerMethodArgumentResolver> resolvers = new ArrayList<>();

		// Annotation-based argument resolution
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), false));
		resolvers.add(new RequestParamMapMethodArgumentResolver());
		resolvers.add(new PathVariableMethodArgumentResolver());
		resolvers.add(new PathVariableMapMethodArgumentResolver());
		resolvers.add(new MatrixVariableMethodArgumentResolver());
		resolvers.add(new MatrixVariableMapMethodArgumentResolver());
		resolvers.add(new ExpressionValueMethodArgumentResolver(getBeanFactory()));
		resolvers.add(new SessionAttributeMethodArgumentResolver());
		resolvers.add(new RequestAttributeMethodArgumentResolver());

		// Type-based argument resolution
		resolvers.add(new ServletRequestMethodArgumentResolver());
		resolvers.add(new ServletResponseMethodArgumentResolver());

		// Custom arguments
		if (getCustomArgumentResolvers() != null) {
			resolvers.addAll(getCustomArgumentResolvers());
		}

		// Catch-all
		resolvers.add(new RequestParamMethodArgumentResolver(getBeanFactory(), true));
		return resolvers;
	}
```

#### getDefaultReturnValueHandlers()

提供对`HandlerMethod`**返回值**的支持（比如`@ResponseBody、DeferredResult`等返回值类型）。多个返回值处理器最终使用的是`HandlerMethodReturnValueHandlerComposite`模式管理和使用。

```java
RequestMappingHandlerAdapter：

	private List<HandlerMethodReturnValueHandler> getDefaultReturnValueHandlers() {
		List<HandlerMethodReturnValueHandler> handlers = new ArrayList<>();

		// Single-purpose return value types
		handlers.add(new ModelAndViewMethodReturnValueHandler());
		handlers.add(new ModelMethodProcessor());
		handlers.add(new ViewMethodReturnValueHandler());
		// 返回值是ResponseBodyEmitter时候，得用reactiveAdapterRegistry看看是Reactive模式还是普通模式
		// taskExecutor：异步时使用的线程池，使用当前类的  contentNegotiationManager：内容协商管理器
		handlers.add(new ResponseBodyEmitterReturnValueHandler(getMessageConverters(), this.reactiveAdapterRegistry, this.taskExecutor, this.contentNegotiationManager));
		handlers.add(new StreamingResponseBodyReturnValueHandler());
		// 此处重要的是getMessageConverters()消息转换器，一般情况下Spring MVC默认会有8个，包括`MappingJackson2HttpMessageConverter`
		// 参见：WebMvcConfigurationSupport定的@Bean --> RequestMappingHandlerAdapter部分
		// 若不@EnableWebMvc默认是只有4个消息转换器的哦~（不支持json）
		// 此处的requestResponseBodyAdvice会介入到请求和响应的body里（消息转换期间）
		handlers.add(new HttpEntityMethodProcessor(getMessageConverters(), this.contentNegotiationManager, this.requestResponseBodyAdvice));
		handlers.add(new HttpHeadersReturnValueHandler());
		handlers.add(new CallableMethodReturnValueHandler());
		handlers.add(new DeferredResultMethodReturnValueHandler());
		handlers.add(new AsyncTaskMethodReturnValueHandler(this.beanFactory));

		// Annotation-based return value types
		// 当标注有@ModelAttribute或者@ResponseBody的时候  这里来处理。显然也用到了消息转换器~
		handlers.add(new ModelAttributeMethodProcessor(false));
		handlers.add(new RequestResponseBodyMethodProcessor(getMessageConverters(), this.contentNegotiationManager, this.requestResponseBodyAdvice));

		// Multi-purpose return value types
		// 当返回的是个字符串/Map时候，这时候就可能有多个目的了（Multi-purpose）
		// 比如字符串：可能重定向redirect、或者直接到某个view
		handlers.add(new ViewNameMethodReturnValueHandler());
		handlers.add(new MapMethodProcessor());

		// Custom return value types
		// 自定义的返回值处理器
		if (getCustomReturnValueHandlers() != null) {
			handlers.addAll(getCustomReturnValueHandlers());
		}

		// Catch-all
		// 兜底：ModelAndViewResolver是需要你自己实现然后set进来的（一般我们不会自定定义）
		// 所以绝大部分情况兜底使用的是ModelAttributeMethodProcessor表示，即使你的返回值里木有标注@ModelAttribute
		// 但你是非简单类型(比如对象类型)的话，返回值都会放进Model里
		if (!CollectionUtils.isEmpty(getModelAndViewResolvers())) {
			handlers.add(new ModelAndViewResolverMethodReturnValueHandler(getModelAndViewResolvers()));
		} else {
			handlers.add(new ModelAttributeMethodProcessor(true));
		}

		return handlers;
	}
```



#### 其它重要属性、方法

属性：

```java
RequestMappingHandlerAdapter：
	
	// ModelAndViewResolver木有内置实现，可自定义实现来参与到返回值到ModelAndView的过程（自定义返回值处理）
	// 一般不怎么使用，我个人也不太推荐使用
	@Nullable
	private List<ModelAndViewResolver> modelAndViewResolvers;
	// 内容协商管理器  默认就是它喽（使用的协商策略是HeaderContentNegotiationStrategy）
	private ContentNegotiationManager contentNegotiationManager = new ContentNegotiationManager();
	// 消息转换器。使用@Bean定义的时候，记得set进来，否则默认只会有4个（不支持json）
	// 若@EnableWebMvc后默认是有8个的，一般都够用了
	private List<HttpMessageConverter<?>> messageConverters;
	// 它在数据绑定初始化的时候会被使用到，调用其initBinder()方法
	// 只不过，现在一般都使用@InitBinder注解来处理了，所以使用较少
	// 说明：它作用域是全局的，对所有的HandlerMethod都生效~~~~~
	@Nullable
	private WebBindingInitializer webBindingInitializer;
	// 默认使用的SimpleAsyncTaskExecutor：每次执行客户提交给它的任务时，它会启动新的线程
	// 并允许开发者控制并发线程的上限（concurrencyLimit），从而起到一定的资源节流作用（默认值是-1，表示不限流）
	// @EnableWebMvc时可通过复写接口的WebMvcConfigurer.getTaskExecutor()自定义提供一个线程池
	private AsyncTaskExecutor taskExecutor = new SimpleAsyncTaskExecutor("MvcAsync");
	// invokeHandlerMethod()执行目标方法时若需要异步执行，超时时间可自定义（默认不超时）
	// 使用上面的taskExecutor以及下面的callableInterceptors/deferredResultInterceptors参与异步的执行
	@Nullable
	private Long asyncRequestTimeout;
	private CallableProcessingInterceptor[] callableInterceptors = new CallableProcessingInterceptor[0];
	private DeferredResultProcessingInterceptor[] deferredResultInterceptors = new DeferredResultProcessingInterceptor[0];

	// @Since 5.0
	private ReactiveAdapterRegistry reactiveAdapterRegistry = ReactiveAdapterRegistry.getSharedInstance();

	// 对应ModelAndViewContainer.setIgnoreDefaultModelOnRedirect()属性
	// redirect时,是否忽略defaultModel 默认值是false：不忽略
	private boolean ignoreDefaultModelOnRedirect = false;
	// 返回内容缓存多久（默认不缓存）  参考类：WebContentGenerator
	private int cacheSecondsForSessionAttributeHandlers = 0;
	// 执行目标方法HandlerMethod时是否要在同一个Session内同步执行？？？
	// 也就是同一个会话时，控制器方法全部同步执行（加互斥锁）
	// 使用场景：对同一用户同一Session的所有访问，必须串行化~~~~~~
	private boolean synchronizeOnSession = false;

	private SessionAttributeStore sessionAttributeStore = new DefaultSessionAttributeStore();
	private ParameterNameDiscoverer parameterNameDiscoverer = new DefaultParameterNameDiscoverer();
	@Nullable
	private ConfigurableBeanFactory beanFactory;

	// ====================下面是各种缓存们====================
	private final Map<Class<?>, SessionAttributesHandler> sessionAttributesHandlerCache = new ConcurrentHashMap<>(64);
	private final Map<Class<?>, Set<Method>> initBinderCache = new ConcurrentHashMap<>(64);
	private final Map<ControllerAdviceBean, Set<Method>> initBinderAdviceCache = new LinkedHashMap<>();
	private final Map<Class<?>, Set<Method>> modelAttributeCache = new ConcurrentHashMap<>(64);
	private final Map<ControllerAdviceBean, Set<Method>> modelAttributeAdviceCache = new LinkedHashMap<>();
```

方法：

```java
RequestMappingHandlerAdapter：

	... // 省略所有属性的get/set方法

	@Override
	protected long getLastModifiedInternal(HttpServletRequest request, HandlerMethod handlerMethod) {
		return -1;
	}
	// 因为它只需要处理HandlerMethod这样的Handler，所以这里恒返回true  请参照父类的supportsInternal()钩子方法
	@Override
	protected boolean supportsInternal(HandlerMethod handlerMethod) {
		return true;
	}

	// 可以认为这个就是`HandlerAdapter`的接口方法，是处理请求的入口 最终返回一个ModelAndView
	@Override
	protected ModelAndView handleInternal(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ModelAndView mav;
		checkRequest(request); // 检查方法

		// Execute invokeHandlerMethod in synchronized block if required.
		// 同一个Session下是否要串行，显然一般都是不需要的。直接看invokeHandlerMethod()方法吧
		if (this.synchronizeOnSession) {
			HttpSession session = request.getSession(false);
			if (session != null) {
				Object mutex = WebUtils.getSessionMutex(session);
				synchronized (mutex) {
					mav = invokeHandlerMethod(request, response, handlerMethod);
				}
			}
			else {
				// No HttpSession available -> no mutex necessary
				mav = invokeHandlerMethod(request, response, handlerMethod);
			}
		} else {
			// No synchronization on session demanded at all...
			mav = invokeHandlerMethod(request, response, handlerMethod);
		}
	
		// 处理Cache-Control这个请求头~~~~~~~~~（若你自己木有set的话）
		if (!response.containsHeader(HEADER_CACHE_CONTROL)) {
			if (getSessionAttributesHandler(handlerMethod).hasSessionAttributes()) {
				applyCacheSeconds(response, this.cacheSecondsForSessionAttributeHandlers);
			} else {
				prepareResponse(response);
			}
		}

		return mav;
	}
```



#### invokeHandlerMethod()

```java
// 它的作用就是执行目标的HandlerMethod，然后返回一个ModelAndView 
	@Nullable
	protected ModelAndView invokeHandlerMethod(HttpServletRequest request, HttpServletResponse response, HandlerMethod handlerMethod) throws Exception {

		ServletWebRequest webRequest = new ServletWebRequest(request, response);
		// 注意：此处只有try-finally 哦
		// 因为invocableMethod.invokeAndHandle(webRequest, mavContainer)是可能会抛出异常的（交给全局异常处理）
		try {
			// 最终创建的是一个ServletRequestDataBinderFactory，持有所有@InitBinder的method方法们
			WebDataBinderFactory binderFactory = getDataBinderFactory(handlerMethod);
			// 创建一个ModelFactory，@ModelAttribute啥的方法就会被引用进来
			ModelFactory modelFactory = getModelFactory(handlerMethod, binderFactory);

			// 把HandlerMethod包装为ServletInvocableHandlerMethod，具有invoke执行的能力喽
			// 下面这几部便是一直给invocableMethod的各大属性赋值~~~
			ServletInvocableHandlerMethod invocableMethod = createInvocableHandlerMethod(handlerMethod);
			if (this.argumentResolvers != null) {
				invocableMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
			}
			if (this.returnValueHandlers != null) {
				invocableMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
			}
			invocableMethod.setDataBinderFactory(binderFactory);
			invocableMethod.setParameterNameDiscoverer(this.parameterNameDiscoverer);


			ModelAndViewContainer mavContainer = new ModelAndViewContainer();
			// 把上个request里的值放进来到本request里
			mavContainer.addAllAttributes(RequestContextUtils.getInputFlashMap(request));
			// model工厂：把它里面的Model值放进mavContainer容器内（此处@ModelAttribute/@SessionAttribute啥的生效）
			modelFactory.initModel(webRequest, mavContainer, invocableMethod);
			mavContainer.setIgnoreDefaultModelOnRedirect(this.ignoreDefaultModelOnRedirect);

			AsyncWebRequest asyncWebRequest = WebAsyncUtils.createAsyncWebRequest(request, response);
			asyncWebRequest.setTimeout(this.asyncRequestTimeout);

			WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
			asyncManager.setTaskExecutor(this.taskExecutor);
			asyncManager.setAsyncWebRequest(asyncWebRequest);
			asyncManager.registerCallableInterceptors(this.callableInterceptors);
			asyncManager.registerDeferredResultInterceptors(this.deferredResultInterceptors);

			// 它不管是不是异步请求都先用AsyncWebRequest 包装了一下，但是若是同步请求
			// asyncManager.hasConcurrentResult()肯定是为false的~~~
			if (asyncManager.hasConcurrentResult()) {
				Object result = asyncManager.getConcurrentResult();
				mavContainer = (ModelAndViewContainer) asyncManager.getConcurrentResultContext()[0];
				asyncManager.clearConcurrentResult();
				LogFormatUtils.traceDebug(logger, traceOn -> {
					String formatted = LogFormatUtils.formatValue(result, !traceOn);
					return "Resume with async result [" + formatted + "]";
				});
				invocableMethod = invocableMethod.wrapConcurrentResult(result);
			}

			// 此处其实就是调用ServletInvocableHandlerMethod#invokeAndHandle()方法喽
			// 关于它你可以来这里：https://fangshixiang.blog.csdn.net/article/details/98385163
			// 注意哦：任何HandlerMethod执行完后都是把结果放在了mavContainer里（它可能有Model，可能有View，可能啥都木有~~）
			// 因此最后的getModelAndView()又得一看
			invocableMethod.invokeAndHandle(webRequest, mavContainer);
			if (asyncManager.isConcurrentHandlingStarted()) {
				return null;
			}

			return getModelAndView(mavContainer, modelFactory, webRequest);
		} finally {
			webRequest.requestCompleted();
		}
	}

// @Nullable：表示它返回的可以是个null哦~(若木有视图，就直接不会render啦~因为response已经写入过值了)
	@Nullable
	private ModelAndView getModelAndView(ModelAndViewContainer mavContainer, ModelFactory modelFactory, NativeWebRequest webRequest) throws Exception {
	
		// 把session里面的内容写入
		modelFactory.updateModel(webRequest, mavContainer);
		// Tips：若已经被处理过，那就返回null喽~~（比如若是@ResponseBody这种，这里就是true）
		if (mavContainer.isRequestHandled()) {
			return null;
		}
			
		// 通过View、Model、Status构造出一个ModelAndView，最终就可以完成渲染了
		ModelMap model = mavContainer.getModel();
		ModelAndView mav = new ModelAndView(mavContainer.getViewName(), model, mavContainer.getStatus());
		if (!mavContainer.isViewReference()) { // 是否是String类型
			mav.setView((View) mavContainer.getView());
		}

		// 对重定向RedirectAttributes参数的支持（两个请求之间传递参数，使用的是ATTRIBUTE）
		if (model instanceof RedirectAttributes) {
			Map<String, ?> flashAttributes = ((RedirectAttributes) model).getFlashAttributes();
			HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
			if (request != null) {
				RequestContextUtils.getOutputFlashMap(request).putAll(flashAttributes);
			}
		}
		return mav;
	}
```















