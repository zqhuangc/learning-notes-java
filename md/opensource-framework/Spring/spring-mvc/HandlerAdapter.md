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





##HandlerMethodReturnValueHandler

```java
// @since 3.1  出现得相对还是比较晚的。因为`RequestMappingHandlerAdapter`也是这个时候才出来
// Strategy interface to handle the value returned from the invocation of a handler method
public interface HandlerMethodReturnValueHandler {

	// 每种处理器实现类，都对应着它能够处理的返回值类型~~~
	boolean supportsReturnType(MethodParameter returnType);

	// Handle the given return value by adding attributes to the model and setting a view or setting the
	// {@link ModelAndViewContainer#setRequestHandled} flag to {@code true} to indicate the response has been handled directly.
	// 简单的说就是处理返回值，可以处理着向Model里设置一个view
	// 或者ModelAndViewContainer#setRequestHandled设置true说我已经直接处理了，后续不要需要再继续渲染了~
	void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception;

}
```



#### MapMethodProcessor

```java
// @since 3.1
public class MapMethodProcessor implements HandlerMethodArgumentResolver, HandlerMethodReturnValueHandler {

	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		return Map.class.isAssignableFrom(parameter.getParameterType());
	}

	@Override
	@Nullable
	public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

		Assert.state(mavContainer != null, "ModelAndViewContainer is required for model exposure");
		return mavContainer.getModel();
	}

	// ==================上面是处理入参的，不是本文的重点~~~====================
	// 显然只有当你的返回值是个Map时候，此处理器才会生效
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		return Map.class.isAssignableFrom(returnType.getParameterType());
	}

	@Override
	@SuppressWarnings({"unchecked", "rawtypes"})
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
		
		// 做的事非常简单，仅仅只是把我们的Map放进Model里面~~~
		// 但是此处需要注意的是：它并没有setViewName，所以它此时是没有视图名称的~~~
		// ModelAndViewContainer#setRequestHandled(true) 所以后续若还有处理器可以继续处理
		if (returnValue instanceof Map){
			mavContainer.addAllAttributes((Map) returnValue);
		}
	}

}
```



#### ViewNameMethodReturnValueHandler

```java
// 可以直接返回一个视图名，最终会交给`RequestToViewNameTranslator`翻译一下~~~
public class ViewNameMethodReturnValueHandler implements HandlerMethodReturnValueHandler {

	// Spring4.1之后支持自定义重定向的匹配规则
	// Spring4.1之前还只能支持redirect:这个固定的前缀~~~~
	private String[] redirectPatterns;
	public void setRedirectPatterns(String... redirectPatterns) {
		this.redirectPatterns = redirectPatterns;
	}
	public String[] getRedirectPatterns() {
		return this.redirectPatterns;
	}
	protected boolean isRedirectViewName(String viewName) {
		return (PatternMatchUtils.simpleMatch(this.redirectPatterns, viewName) || viewName.startsWith("redirect:"));
	}


	// 支持void和CharSequence类型（子类型）
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		Class<?> paramType = returnType.getParameterType();
		return (void.class == paramType || CharSequence.class.isAssignableFrom(paramType));
	}

	// 注意：若返回值是void，此方法都不会进来
	@Override
	public void handleReturnValue(Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

		// 显然如果是void，returnValue 为null，不会走这里。
		// 也就是说viewName设置不了，所以会出现和上诉一样的循环报错~~~~~ 因此不要单独使用
		if (returnValue instanceof CharSequence) {
			String viewName = returnValue.toString();
			mavContainer.setViewName(viewName);
	
			// 做一个处理：如果是重定向的view，那就
			if (isRedirectViewName(viewName)) {
				mavContainer.setRedirectModelScenario(true);
			}
		}
		// 下面这都是不可达的~~~~~setRedirectModelScenario(true)标记一下
		// 此处不仅仅是else，而是还有个！=null的判断  
		// 那是因为如果是void的话这里返回值是null，属于正常的~~~~ 只是什么都不做而已~（viewName也没有设置哦~~~）
		else if (returnValue != null) {
			// should not happen
			throw new UnsupportedOperationException("Unexpected return type: " +
					returnType.getParameterType().getName() + " in method: " + returnType.getMethod());
		}
	}

}
```



#### ViewMethodReturnValueHandler

MappingJackson2JsonView、AbstractPdfView、MarshallingView、RedirectView、JstlView

```java
// javadoc上有说明：此处理器需要配置在支持`@ModelAttribute`或者`@ResponseBody`的处理器们前面。防止它被取代~~~~
public class ViewMethodReturnValueHandler implements HandlerMethodReturnValueHandler {
	
	// 处理素有的View.class类型
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		return View.class.isAssignableFrom(returnType.getParameterType());
	}

	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
			
		// 这个处理逻辑几乎完全同上
		// 最终也是为了mavContainer.setView(view);
		// 也会对重定向视图进行特殊的处理~~~~~~
		if (returnValue instanceof View) {
			View view = (View) returnValue;
			mavContainer.setView(view);
			if (view instanceof SmartView && ((SmartView) view).isRedirectView()) {
				mavContainer.setRedirectModelScenario(true);
			}
		}
		else if (returnValue != null) {
			// should not happen
			throw new UnsupportedOperationException("Unexpected return type: " +
					returnType.getParameterType().getName() + " in method: " + returnType.getMethod());
		}
	}
}
```



#### HttpHeadersReturnValueHandler

```java
public class HttpHeadersReturnValueHandler implements HandlerMethodReturnValueHandler {

	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		return HttpHeaders.class.isAssignableFrom(returnType.getParameterType());
	}

	@Override
	@SuppressWarnings("resource")
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

		// 请注意这里：已经标记该请求已经被处理过了~~~~~
		mavContainer.setRequestHandled(true);

		Assert.state(returnValue instanceof HttpHeaders, "HttpHeaders expected");
		HttpHeaders headers = (HttpHeaders) returnValue;

		// 返回值里自定义返回的响应头。这里会帮你设置到HttpServletResponse 里面去的~~~~
		if (!headers.isEmpty()) {
			HttpServletResponse servletResponse = webRequest.getNativeResponse(HttpServletResponse.class);
			Assert.state(servletResponse != null, "No HttpServletResponse");
			ServletServerHttpResponse outputMessage = new ServletServerHttpResponse(servletResponse);
			outputMessage.getHeaders().putAll(headers);
			outputMessage.getBody();  // flush headers
		}
	}

}

```



#### ModelMethodProcessor

#### ModelAndViewMethodReturnValueHandler

> **ModelAndView = model + view + HttpStatus**

```java
public class ModelAndViewMethodReturnValueHandler implements HandlerMethodReturnValueHandler {

	// Spring4.1后一样  增加自定义重定向前缀的支持
	@Nullable
	private String[] redirectPatterns;
	public void setRedirectPatterns(@Nullable String... redirectPatterns) {
		this.redirectPatterns = redirectPatterns;
	}
	@Nullable
	public String[] getRedirectPatterns() {
		return this.redirectPatterns;
	}
	protected boolean isRedirectViewName(String viewName) {
		return (PatternMatchUtils.simpleMatch(this.redirectPatterns, viewName) || viewName.startsWith("redirect:"));
	}


	// 显然它只处理ModelAndView这种类型~
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		return ModelAndView.class.isAssignableFrom(returnType.getParameterType());
	}

	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

		// 如果调用者返回null 那就标注此请求被处理过了~~~~ 不需要再渲染了
		// 浏览器的效果就是：一片空白
		if (returnValue == null) {
			mavContainer.setRequestHandled(true);
			return;
		}

		ModelAndView mav = (ModelAndView) returnValue;
	
		// isReference()方法为：(this.view instanceof String)
		// 这里专门处理视图就是一个字符串的情况，else是处理视图是个View对象的情况
		if (mav.isReference()) {
			String viewName = mav.getViewName();
			mavContainer.setViewName(viewName);
			if (viewName != null && isRedirectViewName(viewName)) {
				mavContainer.setRedirectModelScenario(true);
			}
		}
		// 处理view  顺便处理重定向
		else {
			View view = mav.getView();
			mavContainer.setView(view);
			
			// 此处所有的view，只有RedirectView的isRedirectView()才是返回true，其它都是false
			if (view instanceof SmartView && ((SmartView) view).isRedirectView()) {
				mavContainer.setRedirectModelScenario(true);
			}
		}
		// 把status和model都设置进去
		mavContainer.setStatus(mav.getStatus());
		mavContainer.addAllAttributes(mav.getModel());
	}
}
```

#### ModelAndViewResolverMethodReturnValueHandler

##### ModelAndViewResolver

#### ModelAttributeMethodProcessor

##### ServletModelAttributeMethodProcessor



#### AbstractMessageConverterMethodProcessor#writeWithMessageConverters

```java
// @since 3.1  会发现它也处理请求，但是不是本文讨论的重点
//return values by writing to the response with {@link HttpMessageConverter HttpMessageConverters}
public abstract class AbstractMessageConverterMethodProcessor extends AbstractMessageConverterMethodArgumentResolver
		implements HandlerMethodReturnValueHandler {
	...
	// 此处我们只关注它处理返回值的和信访方法
	// Writes the given return type to the given output message
	// 从JavaDoc解释可以看出，它的作用很“单一“：就是把返回值写进output message~~~
	@SuppressWarnings({"rawtypes", "unchecked"})
	protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
			ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

		Object body;
		Class<?> valueType;
		Type targetType;

		// 注意此处的特殊处理，相当于把所有的CharSequence类型的，都最终当作String类型处理的~
		if (value instanceof CharSequence) {
			body = value.toString();
			valueType = String.class;
			targetType = String.class;
		}
		// 我们本例；body为返回值对象  Person@5229
		// valueType为：class com.fsx.bean.Person
		// targetType：class com.fsx.bean.Person
		else {
			body = value;
			valueType = getReturnValueType(body, returnType);
	
			// 此处相当于兼容了泛型类型的处理
			targetType = GenericTypeResolver.resolveType(getGenericType(returnType), returnType.getContainingClass());
		}

		// 若返回值是个org.springframework.core.io.Resource  就走这里  此处忽略~~
		if (isResourceType(value, returnType)) {
			outputMessage.getHeaders().set(HttpHeaders.ACCEPT_RANGES, "bytes");
			if (value != null && inputMessage.getHeaders().getFirst(HttpHeaders.RANGE) != null &&
					outputMessage.getServletResponse().getStatus() == 200) {
				Resource resource = (Resource) value;
				try {
					List<HttpRange> httpRanges = inputMessage.getHeaders().getRange();
					outputMessage.getServletResponse().setStatus(HttpStatus.PARTIAL_CONTENT.value());
					body = HttpRange.toResourceRegions(httpRanges, resource);
					valueType = body.getClass();
					targetType = RESOURCE_REGION_LIST_TYPE;
				}
				catch (IllegalArgumentException ex) {
					outputMessage.getHeaders().set(HttpHeaders.CONTENT_RANGE, "bytes */" + resource.contentLength());
					outputMessage.getServletResponse().setStatus(HttpStatus.REQUESTED_RANGE_NOT_SATISFIABLE.value());
				}
			}
		}

		// selectedMediaType表示最终被选中的MediaType，毕竟请求放可能是接受N多种MediaType的~~~
		MediaType selectedMediaType = null;
		// 一般情况下 请求方很少会指定contentType的~~~
		// 如果请求方法指定了，就以它的为准，就相当于selectedMediaType 里面就被找打了
		// 否则就靠系统自己去寻找到一个最为合适的~~~
		MediaType contentType = outputMessage.getHeaders().getContentType();
		if (contentType != null && contentType.isConcrete()) {
			if (logger.isDebugEnabled()) {
				logger.debug("Found 'Content-Type:" + contentType + "' in response");
			}
			selectedMediaType = contentType;
		}
		else {
			HttpServletRequest request = inputMessage.getServletRequest();
			// 前面我们说了 若是谷歌浏览器  默认它的accept为：text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/s
			// 所以此处数组解析出来有7对
			List<MediaType> acceptableTypes = getAcceptableMediaTypes(request);

			// 这个方法就是从所有已经注册的转换器里面去找，看看哪些转换器.canWrite，然后把他们所支持的MediaType都加入进来~~~
			// 比如此例只能匹配到MappingJackson2HttpMessageConverter，所以匹配上的有application/json、application/*+json两个
			List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType, targetType);
			
			// 这个异常应该我们经常碰到：有body体，但是并没有能够支持的转换器，就是这额原因~~~
			if (body != null && producibleTypes.isEmpty()) {
				throw new HttpMessageNotWritableException("No converter found for return value of type: " + valueType);
			}

			// 下面相当于从浏览器可议接受的MediaType里面，最终抉择出N个来
			// 原理也非常简单：你能接受的isCompatibleWith上了我能处理的，那咱们就好说，处理就完了
			List<MediaType> mediaTypesToUse = new ArrayList<>();
			for (MediaType requestedType : acceptableTypes) {
				for (MediaType producibleType : producibleTypes) {
					if (requestedType.isCompatibleWith(producibleType)) {
					// 从两个中选择一个最匹配的  主要是根据q值来比较  排序
					// 比如此例，最终匹配上的有两个：application/json;q=0.8和application/*+json;q=0.8
						mediaTypesToUse.add(getMostSpecificMediaType(requestedType, producibleType));
					}
				}
			}
			
			// 这个异常也不少见，比如此处如果没有导入Jackson相关依赖包
			// 就会抛出这个异常了：HttpMediaTypeNotAcceptableException：Could not find acceptable representation
			if (mediaTypesToUse.isEmpty()) {
				if (body != null) {
					throw new HttpMediaTypeNotAcceptableException(producibleTypes);
				}
				if (logger.isDebugEnabled()) {
					logger.debug("No match for " + acceptableTypes + ", supported: " + producibleTypes);
				}
				return;
			}
			
			// 根据Q值进行排序：
			MediaType.sortBySpecificityAndQuality(mediaTypesToUse);
			// 因为已经排过
			for (MediaType mediaType : mediaTypesToUse) {
				if (mediaType.isConcrete()) {
					selectedMediaType = mediaType;
					break;
				}
				else if (mediaType.isPresentIn(ALL_APPLICATION_MEDIA_TYPES)) {
					selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
					break;
				}
			}

			if (logger.isDebugEnabled()) {
				logger.debug("Using '" + selectedMediaType + "', given " +
						acceptableTypes + " and supported " + producibleTypes);
			}
		}

		// 最终的最终 都会找到一个决定write的类型，必粗此处为：application/json;q=0.8
		//  因为最终会决策出来一个MediaType，所以此处就是要根据此MediaType找到一个合适的消息转换器，把body向outputstream写进去~~~
		// 注意此处：是RequestResponseBodyAdviceChain执行之处~~~~
		if (selectedMediaType != null) {
			selectedMediaType = selectedMediaType.removeQualityValue();
			for (HttpMessageConverter<?> converter : this.messageConverters) {

		
				// 从这个判断可以看出 ，处理body里面内容，GenericHttpMessageConverter类型的转换器是优先级更高，优先去处理的
				GenericHttpMessageConverter genericConverter = (converter instanceof GenericHttpMessageConverter ? (GenericHttpMessageConverter<?>) converter : null);
				if (genericConverter != null ? ((GenericHttpMessageConverter) converter).canWrite(targetType, valueType, selectedMediaType) :
						converter.canWrite(valueType, selectedMediaType)) {

					// 在写body之前执行~~~~  会调用我们注册的所有的合适的ResponseBodyAdvice#beforeBodyWrite方法
					// 相当于在写之前，我们可以介入对body体进行处理
					body = getAdvice().beforeBodyWrite(body, returnType, selectedMediaType,
							(Class<? extends HttpMessageConverter<?>>) converter.getClass(),
							inputMessage, outputMessage);
					if (body != null) {
						Object theBody = body;
						LogFormatUtils.traceDebug(logger, traceOn ->
								"Writing [" + LogFormatUtils.formatValue(theBody, !traceOn) + "]");

						// 给响应Response设置一个Content-Disposition的请求头（若需要的话）  若之前已经设置过了，此处将什么都不做
						// 比如我们常见的：response.setHeader("Content-Disposition", "attachment; filename=" + java.net.URLEncoder.encode(fileName, "UTF-8"));
						//Content-disposition 是 MIME 协议的扩展，MIME 协议指示 MIME 用户代理如何显示附加的文件。
						// 当 Internet Explorer 接收到头时，它会激活文件下载对话框，它的文件名框自动填充了头中指定的文件名
						addContentDispositionHeader(inputMessage, outputMessage);
						if (genericConverter != null) {
							genericConverter.write(body, targetType, selectedMediaType, outputMessage);
						}
						else {
							((HttpMessageConverter) converter).write(body, selectedMediaType, outputMessage);
						}
					}
					// 如果return null，body里面是null 那就啥都不写，输出一个debug日志即可~~~~
					else {
						if (logger.isDebugEnabled()) {
							logger.debug("Nothing to write: null body");
						}
					}

					// 这一句表示：只要一个一个消息转换器处理了，就立马停止~~~~
					return;
				}
			}
		}

		if (body != null) {
			throw new HttpMediaTypeNotAcceptableException(this.allSupportedMediaTypes);
		}
	}
	...
}
```



##### `RequestResponseBodyMethodProcessor`

@ResponseBody

```java
public class RequestResponseBodyMethodProcessor extends AbstractMessageConverterMethodProcessor {

	// 显然可以发现，方法上或者类上标注有@ResponseBody都是可以的~~~~
	// 这也就是为什么现在@RestController可以代替我们的的@Controller + @ResponseBody生效了
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		return (AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), ResponseBody.class) ||
				returnType.hasMethodAnnotation(ResponseBody.class));
	}

	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
		
		// 首先就标记：此请求已经被处理了~~~
		mavContainer.setRequestHandled(true);
		ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
		ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

		// Try even with null return value. ResponseBodyAdvice could get involved.
		// 这个方法是核心，也会处理null值~~~  这里面一些Advice会生效~~~~
		// 会选择到合适的HttpMessageConverter,然后进行消息转换~~~~（这里只指写~~~）  这个方法在父类上，是非常核心关键自然也是非常复杂的~~~
		writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
	}
}

// @since 3.1
public class RequestResponseBodyMethodProcessor extends AbstractMessageConverterMethodProcessor {

	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		return parameter.hasParameterAnnotation(RequestBody.class);
	}
	// 类上或者方法上标注了@ResponseBody注解都行
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		return (AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), ResponseBody.class) || returnType.hasMethodAnnotation(ResponseBody.class));
	}
	
	// 这是处理入参封装校验的入口，也是本文关注的焦点
	@Override
	public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
		
		// 它是支持`Optional`容器的
		parameter = parameter.nestedIfOptional();
		// 使用消息转换器HttpInputMessage把request请求转换出来，拿到值~~~
		// 此处注意：比如本例入参是Person类，所以经过这里处理会生成一个空的Person对象出来（反射）
		Object arg = readWithMessageConverters(webRequest, parameter, parameter.getNestedGenericParameterType());

		// 获取到入参的名称,其实不叫形参名字，应该叫objectName给校验时用的
		// 请注意：这里的名称是类名首字母小写，并不是你方法里写的名字。比如本利若形参名写为personAAA，但是name的值还是person
		// 但是注意：`parameter.getParameterName()`的值可是personAAA
		String name = Conventions.getVariableNameForParameter(parameter);

		// 只有存在binderFactory才会去完成自动的绑定、校验~
		// 此处web环境为：ServletRequestDataBinderFactory
		if (binderFactory != null) {
			WebDataBinder binder = binderFactory.createBinder(webRequest, arg, name);

			// 显然传了参数才需要去绑定校验嘛
			if (arg != null) {

				// 这里完成数据绑定+数据校验~~~~~（绑定的错误和校验的错误都会放进Errors里）
				// Applicable：适合
				validateIfApplicable(binder, parameter);

				// 若有错误消息hasErrors()，并且仅跟着的一个参数不是Errors类型，Spring MVC会主动给你抛出MethodArgumentNotValidException异常
				// 否则，调用者自行处理
				if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
					throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
				}
			}
		
			// 把错误消息放进去 证明已经校验出错误了~~~
			// 后续逻辑会判断MODEL_KEY_PREFIX这个key的~~~~
			if (mavContainer != null) {
				mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());
			}
		}

		return adaptArgumentIfNecessary(arg, parameter);
	}

	// 校验，如果合适的话。使用WebDataBinder，失败信息最终也都是放在它身上~  本方法是本文关注的焦点
	// 入参：MethodParameter parameter
	protected void validateIfApplicable(WebDataBinder binder, MethodParameter parameter) {
		// 拿到标注在此参数上的所有注解们（比如此处有@Valid和@RequestBody两个注解）
		Annotation[] annotations = parameter.getParameterAnnotations();
		for (Annotation ann : annotations) {
			// 先看看有木有@Validated
			Validated validatedAnn = AnnotationUtils.getAnnotation(ann, Validated.class);

			// 这个里的判断是关键：可以看到标注了@Validated注解 或者注解名是以Valid打头的 都会有效哦
			//注意：这里可没说必须是@Valid注解。实际上你自定义注解，名称只要一Valid开头都成~~~~~
			if (validatedAnn != null || ann.annotationType().getSimpleName().startsWith("Valid")) {
				// 拿到分组group后，调用binder的validate()进行校验~~~~
				// 可以看到：拿到一个合适的注解后，立马就break了~~~
				// 所以若你两个主机都标注@Validated和@Valid，效果是一样滴~
				Object hints = (validatedAnn != null ? validatedAnn.value() : AnnotationUtils.getValue(ann));
				Object[] validationHints = (hints instanceof Object[] ? (Object[]) hints : new Object[] {hints});
				binder.validate(validationHints);
				break;
			}
		}
	}
	...
}
```



#### AsyncHandlerMethodReturnValueHandler

```java
// @since 4.2
// 支持异步类型的返回值处理程序。此类返回值类型需要优先处理，以便异步值可以“展开”。
// 异步实现此接口并不是必须的，但是若你需要在处理程序之前执行，就需要实现这个接口了~~~
// 因为默认情况下：我们自定义的Handler它都是在内置的Handler后面去执行的~~~~
public interface AsyncHandlerMethodReturnValueHandler extends HandlerMethodReturnValueHandler {
	// 给定的返回值是否表示异步计算
	boolean isAsyncReturnValue(@Nullable Object returnValue, MethodParameter returnType);
}
```



#### StreamingResponseBodyReturnValueHandler

```java
public class StreamingResponseBodyReturnValueHandler implements HandlerMethodReturnValueHandler {

	// 显然这里支持返回值直接是StreamingResponseBody类型，也支持你用`ResponseEntity`在包一层
	// ResponseEntity的泛型类型必须是StreamingResponseBody类型~~~
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		if (StreamingResponseBody.class.isAssignableFrom(returnType.getParameterType())) {
			return true;
		} else if (ResponseEntity.class.isAssignableFrom(returnType.getParameterType())) {
			Class<?> bodyType = ResolvableType.forMethodParameter(returnType).getGeneric().resolve();
			return (bodyType != null && StreamingResponseBody.class.isAssignableFrom(bodyType));
		}
		return false;
	}

	@Override
	@SuppressWarnings("resource")
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

		// 从这句代码也可以看出，只有返回值为null了，它才关闭，否则可以持续不断的向response里面写东西
		if (returnValue == null) {
			mavContainer.setRequestHandled(true);
			return;
		}

		HttpServletResponse response = webRequest.getNativeResponse(HttpServletResponse.class);
		Assert.state(response != null, "No HttpServletResponse");
		ServerHttpResponse outputMessage = new ServletServerHttpResponse(response);

		// 从ResponseEntity里body提取出来~~~~~
		if (returnValue instanceof ResponseEntity) {
			ResponseEntity<?> responseEntity = (ResponseEntity<?>) returnValue;
			response.setStatus(responseEntity.getStatusCodeValue());
			outputMessage.getHeaders().putAll(responseEntity.getHeaders());
			returnValue = responseEntity.getBody();
			if (returnValue == null) {
				mavContainer.setRequestHandled(true);
				outputMessage.flush();
				return;
			}
		}

		ServletRequest request = webRequest.getNativeRequest(ServletRequest.class);
		Assert.state(request != null, "No ServletRequest");
		ShallowEtagHeaderFilter.disableContentCaching(request); // 禁用内容缓存

		Assert.isInstanceOf(StreamingResponseBody.class, returnValue, "StreamingResponseBody expected");
		StreamingResponseBody streamingBody = (StreamingResponseBody) returnValue;

		// 最终也是开启了一个Callable 任务，最后交给WebAsyncUtils去执行的~~~~
		Callable<Void> callable = new StreamingResponseBodyTask(outputMessage.getBody(), streamingBody);

		// WebAsyncUtils.getAsyncManager得到的是一个`WebAsyncManager`对象
		// startCallableProcessing会把callable任务都包装成一个`WebAsyncTask`,最终交给`AsyncTaskExecutor`执行
		// 至于异步的详细执行原理，请参考上面的相关博文，此处只点一下~~~~
		WebAsyncUtils.getAsyncManager(webRequest).startCallableProcessing(callable, mavContainer);
	}

	// 这个任务很简单，实现了Callable的call方法，它就是相当于启一个线程，把本次body里面的内容写进response输出流里面~~~
	// 但是此时输出流并不会关闭~~~~
	private static class StreamingResponseBodyTask implements Callable<Void> {

		private final OutputStream outputStream;

		private final StreamingResponseBody streamingBody;

		public StreamingResponseBodyTask(OutputStream outputStream, StreamingResponseBody streamingBody) {
			this.outputStream = outputStream;
			this.streamingBody = streamingBody;
		}

		@Override
		public Void call() throws Exception {
			this.streamingBody.writeTo(this.outputStream);
			return null;
		}
	}

}
```



#### DeferredResultMethodReturnValueHandler

```java
public class DeferredResultMethodReturnValueHandler implements HandlerMethodReturnValueHandler {

	// 它支持处理丰富的数据类型
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		Class<?> type = returnType.getParameterType();
		return (DeferredResult.class.isAssignableFrom(type) ||
				ListenableFuture.class.isAssignableFrom(type) ||
				CompletionStage.class.isAssignableFrom(type));
	}

	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

		// 一样的  只有返回null了才代表此请求处理完成了
		if (returnValue == null) {
			mavContainer.setRequestHandled(true);
			return;
		}

		DeferredResult<?> result;

		// 此处是适配器模式的使用，最终都适配成了一个DeferredResult（使用的内部类实现的~~~）
		if (returnValue instanceof DeferredResult) {
			result = (DeferredResult<?>) returnValue;
		} else if (returnValue instanceof ListenableFuture) {
			result = adaptListenableFuture((ListenableFuture<?>) returnValue);
		} else if (returnValue instanceof CompletionStage) {
			result = adaptCompletionStage((CompletionStage<?>) returnValue);
		} else {
			// Should not happen...
			throw new IllegalStateException("Unexpected return value type: " + returnValue);
		}
		// 此处调用的异步方法是：startDeferredResultProcessing
		WebAsyncUtils.getAsyncManager(webRequest).startDeferredResultProcessing(result, mavContainer);
	}

	// 下为两匿名内部实现类做的兼容适配、兼容处理~~~~~非常的简单~~~~
	private DeferredResult<Object> adaptListenableFuture(ListenableFuture<?> future) {
		DeferredResult<Object> result = new DeferredResult<>();
		future.addCallback(new ListenableFutureCallback<Object>() {
			@Override
			public void onSuccess(@Nullable Object value) {
				result.setResult(value);
			}
			@Override
			public void onFailure(Throwable ex) {
				result.setErrorResult(ex);
			}
		});
		return result;
	}

	private DeferredResult<Object> adaptCompletionStage(CompletionStage<?> future) {
		DeferredResult<Object> result = new DeferredResult<>();
		future.handle((BiFunction<Object, Throwable, Object>) (value, ex) -> {
			if (ex != null) {
				result.setErrorResult(ex);
			}
			else {
				result.setResult(value);
			}
			return null;
		});
		return result;
	}

}
```



#### CallableMethodReturnValueHandler

#### ResponseBodyEmitterReturnValueHandler

#### AsyncTaskMethodReturnValueHandler

#### HandlerMethodReturnValueHandlerComposite：处理器合成

```java
// 首先发现，它也实现了接口HandlerMethodReturnValueHandler 
// 它会缓存以前解析的返回类型以加快查找速度
public class HandlerMethodReturnValueHandlerComposite implements HandlerMethodReturnValueHandler {

	private final List<HandlerMethodReturnValueHandler> returnValueHandlers = new ArrayList<>();
	// 返回的是一个只读视图
	public List<HandlerMethodReturnValueHandler> getHandlers() {
		return Collections.unmodifiableList(this.returnValueHandlers);
	}
	public HandlerMethodReturnValueHandlerComposite addHandler(HandlerMethodReturnValueHandler handler) {
		this.returnValueHandlers.add(handler);
		return this;
	}
	public HandlerMethodReturnValueHandlerComposite addHandlers( @Nullable List<? extends HandlerMethodReturnValueHandler> handlers) {
		if (handlers != null) {
			this.returnValueHandlers.addAll(handlers);
		}
		return this;
	}
	
	// 由这两个可议看出，但凡有一个Handler支持处理这个返回值，就是支持的~~~
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		return getReturnValueHandler(returnType) != null;
	}
	@Nullable
	private HandlerMethodReturnValueHandler getReturnValueHandler(MethodParameter returnType) {
		for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
			if (handler.supportsReturnType(returnType)) {
				return handler;
			}
		}
		return null;
	}

	// 这里就是处理返回值的核心内容~~~~~
	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

		// selectHandler选择收个匹配的Handler来处理这个返回值~~~~ 若一个都木有找到  抛出异常吧~~~~
		// 所有很重要的一个方法是它：selectHandler()  它来匹配，以及确定优先级
		HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
		if (handler == null) {
			throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
		}
		handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
	}

	// 根据返回值，以及返回类型  来找到一个最为合适的HandlerMethodReturnValueHandler
	@Nullable
	private HandlerMethodReturnValueHandler selectHandler(@Nullable Object value, MethodParameter returnType) {
		// 这个和我们上面的就对应上了  第一步去判断这个返回值是不是一个异步的value（AsyncHandlerMethodReturnValueHandler实现类只能我们自己来写~）
		boolean isAsyncValue = isAsyncReturnValue(value, returnType);
		for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
			// 如果判断发现这个值是异步的value，那它显然就只能交给你自己定义的异步处理器处理了，别的处理器肯定就靠边站~~~~~
			if (isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler)) {
				continue;
			}
			if (handler.supportsReturnType(returnType)) {
				return handler;
			}
		}
		return null;
	}
	private boolean isAsyncReturnValue(@Nullable Object value, MethodParameter returnType) {
		for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
			if (handler instanceof AsyncHandlerMethodReturnValueHandler && ((AsyncHandlerMethodReturnValueHandler) handler).isAsyncReturnValue(value, returnType)) {
				return true;
			}
		}
		return false;
	}
}
```



# HttpMessageConverter

```java
// @since 3.0  Spring3.0后推出的   是个泛型接口
// 策略接口，指定可以从HTTP请求和响应转换为HTTP请求和响应的转换器
public interface HttpMessageConverter<T> {

	// 指定转换器可以读取的对象类型，即转换器可将请求信息转换为clazz类型的对象
	// 同时支持指定的MIME类型(text/html、application/json等)
	boolean canRead(Class<?> clazz, @Nullable MediaType mediaType);
	// 指定转换器可以将clazz类型的对象写到响应流当中，响应流支持的媒体类型在mediaType中定义
	boolean canWrite(Class<?> clazz, @Nullable MediaType mediaType);
	// 返回当前转换器支持的媒体类型~~
	List<MediaType> getSupportedMediaTypes();

	// 将请求信息转换为T类型的对象 流对象为：HttpInputMessage
	T read(Class<? extends T> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException;
	// 将T类型的对象写到响应流当中，同事指定响应的媒体类型为contentType 输出流为：HttpOutputMessage 
	void write(T t, @Nullable MediaType contentType, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException;
}
```



## HttpMessage

```java
public interface HttpMessage {
	// Return the headers of this message
	HttpHeaders getHeaders();
}

public interface HttpInputMessage extends HttpMessage {
	InputStream getBody() throws IOException;
}
public interface HttpOutputMessage extends HttpMessage {
	OutputStream getBody() throws IOException;
}
```



### FormHttpMessageConverter：form表单提交/文件下载

### AbstractHttpMessageConverter

```java
public abstract class AbstractHttpMessageConverter<T> implements HttpMessageConverter<T> {

	// 它主要内部维护了这两个属性，可议构造器赋值，也可以set方法赋值~~
	private List<MediaType> supportedMediaTypes = Collections.emptyList();
	@Nullable
	private Charset defaultCharset;

	// supports是个抽象方法，交给子类自己去决定自己支持的转换类型~~~~
	// 而canRead(mediaType)表示MediaType也得在我支持的范畴了才行（入参MediaType若没有指定，就返回true的）
	@Override
	public boolean canRead(Class<?> clazz, @Nullable MediaType mediaType) {
		return supports(clazz) && canRead(mediaType);
	}

	// 原理基本同上，supports和上面是同一个抽象方法  所以我们发现并不能入参处理Map，出餐处理List等等
	@Override
	public boolean canWrite(Class<?> clazz, @Nullable MediaType mediaType) {
		return supports(clazz) && canWrite(mediaType);
	}


	// 这是Spring的惯用套路:readInternal  虽然什么都没做，但我觉得还是挺有意义的。Spring后期也非常的好扩展了~~~~
	@Override
	public final T read(Class<? extends T> clazz, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException {
		return readInternal(clazz, inputMessage);
	}


	// 整体上就write方法做了一些事~~
	@Override
	public final void write(final T t, @Nullable MediaType contentType, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException {

		final HttpHeaders headers = outputMessage.getHeaders();
		// 设置一个headers.setContentType 和 headers.setContentLength
		addDefaultHeaders(headers, t, contentType);

		if (outputMessage instanceof StreamingHttpOutputMessage) {
			StreamingHttpOutputMessage streamingOutputMessage = (StreamingHttpOutputMessage) outputMessage;
			// StreamingHttpOutputMessage增加的setBody()方法，关于它下面会给一个使用案例~~~~
			streamingOutputMessage.setBody(outputStream -> writeInternal(t, new HttpOutputMessage() {
				// 注意此处复写：返回的是outputStream ，它也是靠我们的writeInternal对它进行写入的~~~~
				@Override
				public OutputStream getBody() {
					return outputStream;
				}
				@Override
				public HttpHeaders getHeaders() {
					return headers;
				}
			}));
		}
		// 最后它执行了flush，这也就是为何我们自己一般不需要flush的原因
		else {
			writeInternal(t, outputMessage);
			outputMessage.getBody().flush();
		}
	}
	
	// 三个抽象方法
	protected abstract boolean supports(Class<?> clazz);
	protected abstract T readInternal(Class<? extends T> clazz, HttpInputMessage inputMessage)
			throws IOException, HttpMessageNotReadableException;
	protected abstract void writeInternal(T t, HttpOutputMessage outputMessage)
			throws IOException, HttpMessageNotWritableException;
			
}
```



##### StringHttpMessageConverter

```java
// @since 3.0  出生非常早
public class StringHttpMessageConverter extends AbstractHttpMessageConverter<String> {
	// 这就是为何你return中文的时候会乱码的原因（若你不设置它的编码的话~）
	public static final Charset DEFAULT_CHARSET = StandardCharsets.ISO_8859_1;
	@Nullable
	private volatile List<Charset> availableCharsets;
	// 标识是否输出 Response Headers:Accept-Charset(默认true表示输出)
	private boolean writeAcceptCharset = true;

	public StringHttpMessageConverter() {
		this(DEFAULT_CHARSET);
	}
	public StringHttpMessageConverter(Charset defaultCharset) {
		super(defaultCharset, MediaType.TEXT_PLAIN, MediaType.ALL);
	}

	//Indicates whether the {@code Accept-Charset} should be written to any outgoing request.
	// Default is {@code true}.
	public void setWriteAcceptCharset(boolean writeAcceptCharset) {
		this.writeAcceptCharset = writeAcceptCharset;
	}

	// 只处理String类型~
	@Override
	public boolean supports(Class<?> clazz) {
		return String.class == clazz;
	}

	@Override
	protected String readInternal(Class<? extends String> clazz, HttpInputMessage inputMessage) throws IOException {
		// 哪编码的原则为：
		// 1、contentType自己指定了编码就以指定的为准
		// 2、没指定，但是类型是`application/json`，统一按照UTF_8处理
		// 3、否则使用默认编码：getDefaultCharset  ISO_8859_1
		Charset charset = getContentTypeCharset(inputMessage.getHeaders().getContentType());
		// 按照此编码，转换为字符串~~~
		return StreamUtils.copyToString(inputMessage.getBody(), charset);
	}


	// 显然，ContentLength和编码也是有关的~~~
	@Override
	protected Long getContentLength(String str, @Nullable MediaType contentType) {
		Charset charset = getContentTypeCharset(contentType);
		return (long) str.getBytes(charset).length;
	}


	@Override
	protected void writeInternal(String str, HttpOutputMessage outputMessage) throws IOException {
		// 默认会给请求设置一个接收的编码格式~~~（若用户不指定，是所有的编码都支持的）
		if (this.writeAcceptCharset) {
			outputMessage.getHeaders().setAcceptCharset(getAcceptedCharsets());
		}
		
		// 根据编码把字符串写进去~
		Charset charset = getContentTypeCharset(outputMessage.getHeaders().getContentType());
		StreamUtils.copy(str, charset, outputMessage.getBody());
	}
	...
}
```



### GenericHttpMessageConverter 子接口



















