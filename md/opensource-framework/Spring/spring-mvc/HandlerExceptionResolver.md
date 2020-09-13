## Java异常

* Error：错误，对于所有的编译时期的错误以及系统错误都是通过Error抛出的，比如NoClassDefFoundError、Virtual MachineError、ZipError、硬件问题等等。
* Exception：异常，是更为重要的一个分支，是程序员经常打交道的。异常定义为是程序的问题，程序本身是可以处理的。


`Error`和`Exception`最大的区别是：异常是可以被程序处理的，而错误是没法处理的。



#### 为何需要`全局`异常处理？

1. `Controller`一般方法众多，那就需要写大量的`try-catch`代码，很难看也很难维护
2. 在此处`try-catch`也只能捕获住`Handler`的异常，万一是view抛出异常了呢？？？

即使你的程序出现了异常（因为避免不了），**你总不能把一些只有程序员才能看懂的错误代码抛给用户去看吧**，因此展现一个比较友好的错误页面就显得很有必要了，这就是全局异常处理。



## Spring MVC处理异常

Spring MVC提供处理异常的方式主要分为两种：

实现HandlerExceptionResolver方式
@ExceptionHandler注解方式。注解方式也有两种用法：
1. 使用在Controller内部
2. 配置@ControllerAdvice一起使用实现全局处理



## HandlerExceptionResolver

```java
// @since 22.11.2003
public interface HandlerExceptionResolver {
	// 注意：handler是有可能为null的，比如404
	@Nullable
	ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex);
}
```

处理方法返回一个`ModelAndView`视图：既可以是json，也可以是页面。从接口参数上可以发现的是：它只能处理`Exception`，因为`Error`是程序处理不了的（**注意：Error也是可以捕获的**），因此入参类型若写成`Throwable`是不合适的。



### HandlerExceptionResolverComposite

### AbstractHandlerExceptionResolver

主要是提供了对异常更细粒度的控制：此`Resolver`可只处理指定类型的异常。

```java
// @since 3.0
public abstract class AbstractHandlerExceptionResolver implements HandlerExceptionResolver, Ordered {
	...
	private int order = Ordered.LOWEST_PRECEDENCE;
	
	// 可以设置任何的handler，表示只作用于这些Handler们
	@Nullable
	private Set<?> mappedHandlers;
	// 表示只作用域这些Class类型的Handler们~~~
	@Nullable
	private Class<?>[] mappedHandlerClasses;
	// 以上两者若都为null，那就是匹配素有。但凡有一个有值，那就需要精确匹配（并集的关系）
	
	... // 省略所有的get/set方法

	@Override
	@Nullable
	public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {

		// 这个作用匹配逻辑很简答
		// 若mappedHandlers和mappedHandlerClasses都为null永远返回true
		// 但凡配置了一个就需要精确匹配（并集关系）
		// 需要注意的是：shouldApplyTo方法，子类AbstractHandlerMethodExceptionResolver是有复写的
		if (shouldApplyTo(request, handler)) {
			// 是否执行；response.addHeader(HEADER_CACHE_CONTROL, "no-store")  默认是不执行的
			prepareResponse(ex, response);
			// 此抽象方法留给子类去完成~~~~~
			ModelAndView result = doResolveException(request, response, handler, ex);
			return result;
		} else { // 若此处理器不处理，就返回null呗
			return null;
		}
	}
}
```

此抽象类主要是提供`setMappedHandlers`和`setMappedHandlerClasses`让此处理器可以作用在指定类型/处理器上，因此子类只要继承了它都将会有这种能力，这也是为何我推荐自定义实现也继承于它的原因。它提供了`shouldApplyTo()`方法用于匹配逻辑

#### doResolveException

#### SimpleMappingExceptionResolver

通过异常类型Properties exceptionMappings;映射。它的key可以是全类名、短名称，同时还有继承效果：比如key是Exception那将匹配所有的异常。value是view name视图名称

若有需要，可以配合Class<?>[] excludedExceptions来一起使用
通过状态码Map<String, Integer> statusCodes匹配。key是view name，value是http状态码

##### ResponseStatusExceptionResolver

若抛出的**异常类型**上有`@ResponseStatus`注解，那么此处理器就会处理，并且状态码会返给response。`Spring5.0`还能处理`ResponseStatusException`这个异常（此异常是5.0新增）



#### DefaultHandlerExceptionResolver

##### 异常类型

> 异常类型	 ----------------- >  状态码
> MissingPathVariableException	--> 500
> ConversionNotSupportedException	--> 	500
> HttpMessageNotWritableException	--> 	500
> AsyncRequestTimeoutException	--> 503
> MissingServletRequestParameterException	--> 	400
> ServletRequestBindingException	--> 	400
> TypeMismatchException	--> 	400
> HttpMessageNotReadableException	--> 	400
> MethodArgumentNotValidException	--> 	400
> MissingServletRequestPartException	--> 	400
> BindException	--> 	400
> NoHandlerFoundException	--> 	404
> HttpRequestMethodNotSupportedException	--> 	405
> HttpMediaTypeNotAcceptableException	--> 	406
> HttpMediaTypeNotSupportedException	--> 	415





## @ExceptionHandler

```java
// @since 3.0
@Target(ElementType.METHOD) // 只能标注在方法上
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ExceptionHandler {
	// 指定异常类型，可以多个
	Class<? extends Throwable>[] value() default {};
}
```



### AbstractHandlerMethodExceptionResolver

服务于处理器类型是`HandlerMethod`类型的抛出的异常，它并不规定实现方式必须是`@ExceptionHandler`。

```java
// @since 3.1 专门处理HandlerMethod类型是HandlerMethod类型的异常
public abstract class AbstractHandlerMethodExceptionResolver extends AbstractHandlerExceptionResolver {
	
	// 只处理HandlerMethod这种类型的处理器抛出的异常~~~~~~
	@Override
	protected boolean shouldApplyTo(HttpServletRequest request, @Nullable Object handler) {
		if (handler == null) {
			return super.shouldApplyTo(request, null);
		} else if (handler instanceof HandlerMethod) {
			HandlerMethod handlerMethod = (HandlerMethod) handler;
			// 可以看到最终getBean表示最终哪去验证的是它所在的Bean类，而不是方法本身
			// 所以异常的控制是针对于Controller这个类的~
			handler = handlerMethod.getBean(); 
			return super.shouldApplyTo(request, handler);
		} else {
			return false;
		}
	}

	@Override
	@Nullable
	protected final ModelAndView doResolveException(HttpServletRequest request, HttpServletResponse response, @Nullable Object handler, Exception ex) {
		return doResolveHandlerMethodException(request, response, (HandlerMethod) handler, ex);
	}

	@Nullable
	protected abstract ModelAndView doResolveHandlerMethodException(HttpServletRequest request, HttpServletResponse response, @Nullable HandlerMethod handlerMethod, Exception ex);
}
```



#### `ExceptionHandlerExceptionResolver`

用于处理标注有`@ExceptionHandler`注解的`HandlerMethod`方法

```java
// @since 3.1
public class ExceptionHandlerExceptionResolver extends AbstractHandlerMethodExceptionResolver implements ApplicationContextAware, InitializingBean {

	// 这个熟悉：用于处理方法入参的（比如支持入参里可写HttpServletRequest等等）
	@Nullable
	private List<HandlerMethodArgumentResolver> customArgumentResolvers;
	@Nullable
	private HandlerMethodArgumentResolverComposite argumentResolvers;
	
	// 用于处理方法返回值（ModelAndView、@ResponseBody、@ResponseStatus等）
	@Nullable
	private List<HandlerMethodReturnValueHandler> customReturnValueHandlers;
	@Nullable
	private HandlerMethodReturnValueHandlerComposite returnValueHandlers;

	// 消息处理器和内容协商管理器
	private List<HttpMessageConverter<?>> messageConverters;
	private ContentNegotiationManager contentNegotiationManager = new ContentNegotiationManager();

	// 通知（因为异常是可以做全局效果的）
	private final List<Object> responseBodyAdvice = new ArrayList<>();
	@Nullable
	private ApplicationContext applicationContext;

	// 缓存：异常类型对应的处理器
	// 它缓存着Controller本类，对应的异常处理器（多个@ExceptionHandler）~~~~
	private final Map<Class<?>, ExceptionHandlerMethodResolver> exceptionHandlerCache = new ConcurrentHashMap<>(64);
	// 它缓存ControllerAdviceBean对应的异常处理器（@ExceptionHandler）
	private final Map<ControllerAdviceBean, ExceptionHandlerMethodResolver> exceptionHandlerAdviceCache = new LinkedHashMap<>();

	// 唯一构造函数：注册上默认的消息转换器
	public ExceptionHandlerExceptionResolver() {
		StringHttpMessageConverter stringHttpMessageConverter = new StringHttpMessageConverter();
		...
		this.messageConverters.add(new AllEncompassingFormHttpMessageConverter());
	}
	... // 省略所有的get/set方法

	@Override
	public void afterPropertiesSet() {
		// Do this first, it may add ResponseBodyAdvice beans
		// 这一步骤同RequestMappingHandlerAdapter#initControllerAdviceCache
		// 目的是找到项目中所有的`ResponseBodyAdvice`，然后缓存起来。
		// 并且把它里面所有的标注有@ExceptionHandler的方法都解析保存起来
		// exceptionHandlerAdviceCache：每个advice切面对应哪个ExceptionHandlerMethodResolver（含多个@ExceptionHandler处理方法）
		
		//并且，并且若此Advice还实现了接口：ResponseBodyAdvice。那就还可干预到异常处理器的返回值处理上（基于body）
		//可见：若你想干预到异常处理器的返回值body上，可通过ResponseBodyAdvice来实现哟~~~~~~~~~ 
		// 可见ResponseBodyAdvice连异常处理方法也是生效的，但是`RequestBodyAdvice`可就木有啦。
		initExceptionHandlerAdviceCache();

		// 注册默认的参数处理器。支持到了@SessionAttribute、@RequestAttribute
		// ServletRequest/ServletResponse/RedirectAttributes/ModelMethod等等（当然你还可以自定义）
		if (this.argumentResolvers == null) {
			List<HandlerMethodArgumentResolver> resolvers = getDefaultArgumentResolvers();
			this.argumentResolvers = new HandlerMethodArgumentResolverComposite().addResolvers(resolvers);
		}
		// 支持到了：ModelAndView/Model/View/HttpEntity/ModelAttribute/RequestResponseBody
		// ViewName/Map等等这些返回值 当然还可以自定义
		if (this.returnValueHandlers == null) {
			List<HandlerMethodReturnValueHandler> handlers = getDefaultReturnValueHandlers();
			this.returnValueHandlers = new HandlerMethodReturnValueHandlerComposite().addHandlers(handlers);
		}
	}
	...

	// 处理HandlerMethod类型的异常。它的步骤是找到标注有@ExceptionHandler匹配的方法
	// 然后执行此方法来处理所抛出的异常
	@Override
	@Nullable
	protected ModelAndView doResolveHandlerMethodException(HttpServletRequest request, HttpServletResponse response, @Nullable HandlerMethod handlerMethod, Exception exception) {

		// 这个方法是精华，是关键。它最终返回的是一个ServletInvocableHandlerMethod可执行的方法处理器
		// 也就是说标注有@ExceptionHandler的方法最终会成为它

		// 1、本类能够找到处理方法，就在本类里找，找到就返回一个ServletInvocableHandlerMethod
		// 2、本类木有，就去ControllerAdviceBean切面里找，匹配上了也是欧克的
		//   显然此处会判断：advice.isApplicableToBeanType(handlerType) 看此advice是否匹配
		// 若两者都木有找到，那就返回null。这里的核心其实是ExceptionHandlerMethodResolver这个类
		ServletInvocableHandlerMethod exceptionHandlerMethod = getExceptionHandlerMethod(handlerMethod, exception);
		if (exceptionHandlerMethod == null) {
			return null;
		}

		
		// 给该执行器设置一些值，方便它的指定（封装参数和处理返回值）
		if (this.argumentResolvers != null) {
			exceptionHandlerMethod.setHandlerMethodArgumentResolvers(this.argumentResolvers);
		}
		if (this.returnValueHandlers != null) {
			exceptionHandlerMethod.setHandlerMethodReturnValueHandlers(this.returnValueHandlers);
		}
	}

	...
	// 执行此方法的调用（比couse也传入进去了）
	exceptionHandlerMethod.invokeAndHandle(webRequest, mavContainer, exception, cause, handlerMethod);
	... // 下面处理model、ModelAndView、view等等。最终返回一个ModelAndView
	// 这样异常梳理完成。
}
```

@ExceptionHandler的处理和执行是由本类完成的，同一个Class上的所有@ExceptionHandler方法对应着同一个ExceptionHandlerExceptionResolver，不同Class上的对应着不同的~
标注有@ExceptionHandler的方法入参上可写：具体异常类型、ServletRequest/ServletResponse/RedirectAttributes/ModelMethod等等
1. 注意：入参写具体异常类型时只能够写一个类型。（若有多种异常，请写公共父类，你再用instanceof来辨别，而不能直接写多个）
  返回值可写：ModelAndView/Model/View/HttpEntity/ModelAttribute/RequestResponseBody/@ResponseStatus等等
  @ExceptionHandler只能标注在方法上。既能标注在Controller本类内的方法上（只对本类生效），也可配合@ControllerAdvice一起使用（对全局生效）
  对步骤4的两种情况，执行时的匹配顺序如下：优先匹配本类（本Controller），再匹配全局的。
  有必要再强调一句：@ExceptionHandler方式并不是只能返回JSON串，步骤4也说了，它返回一个ModelAndView也是ok的



1. `@Controller + @ExceptionHandler`优先级最高
2. `@ControllerAdvice + @ExceptionHandler`次之
3. `HandlerExceptionResolver`最后（一般是`DefaultHandlerExceptionResolver`）



#### ResponseEntityExceptionHandler

`Spring 3.2`后对`REST`应用异常支持的一个暖心举动。它包装了各种`Spring MVC`在处理请求时**可能抛出的**异常的处理，处理结果都是封装成一个`ResponseEntity`对象。通过`ResponseEntity`我们可以指定需要响应的**状态码**、**header**和**body**等信息



### `ExceptionHandlerMethodResolver`

```java
// @since 3.1
public class ExceptionHandlerMethodResolver {

	// A filter for selecting {@code @ExceptionHandler} methods.
	public static final MethodFilter EXCEPTION_HANDLER_METHODS = method -> AnnotatedElementUtils.hasAnnotation(method, ExceptionHandler.class);

	// 两个缓存：key：异常类型   value：目标方法Method
	private final Map<Class<? extends Throwable>, Method> mappedMethods = new HashMap<>(16);
	private final Map<Class<? extends Throwable>, Method> exceptionLookupCache = new ConcurrentReferenceHashMap<>(16);

	// 唯一构造函数
	// detectExceptionMappings：传入method，找到这个Method可以处理的所有的异常类型们（注意此方法的逻辑）
	// addExceptionMapping：把异常类型和Method缓存进mappedMethods里
	public ExceptionHandlerMethodResolver(Class<?> handlerType) {
		for (Method method : MethodIntrospector.selectMethods(handlerType, EXCEPTION_HANDLER_METHODS)) {
			for (Class<? extends Throwable> exceptionType : detectExceptionMappings(method)) {
				addExceptionMapping(exceptionType, method);
			}
		}
	}

	// 找到此Method能够处理的所有的异常类型
	// 1、detectAnnotationExceptionMappings：本方法或者父类的方法上标注有ExceptionHandler注解，然后读取出其value值就是它能处理的异常们
	// 2、若value值木有指定，那所有的方法入参们的异常类型，就是此方法能够处理的所有异常们
	// 3、若最终还是空，那就抛出异常：No exception types mapped to " + method
	private List<Class<? extends Throwable>> detectExceptionMappings(Method method) {
		List<Class<? extends Throwable>> result = new ArrayList<>();
		detectAnnotationExceptionMappings(method, result);
		if (result.isEmpty()) {
			for (Class<?> paramType : method.getParameterTypes()) {
				if (Throwable.class.isAssignableFrom(paramType)) {
					result.add((Class<? extends Throwable>) paramType);
				}
			}
		}
		if (result.isEmpty()) {
			throw new IllegalStateException("No exception types mapped to " + method);
		}
		return result;
	}

	// 对于添加方法一样有一句值得说的：
	// 若不同的Method表示可以处理同一个异常，那是不行的："Ambiguous @ExceptionHandler method mapped for [" 
	// 注意：此处必须是同一个异常（比如Exception和RuntimeException不属于同一个...）
	private void addExceptionMapping(Class<? extends Throwable> exceptionType, Method method) {
		Method oldMethod = this.mappedMethods.put(exceptionType, method);
		if (oldMethod != null && !oldMethod.equals(method)) {
			throw new IllegalStateException("Ambiguous @ExceptionHandler method mapped for [" + exceptionType + "]: {" + oldMethod + ", " + method + "}");
		}
	}

	// 给指定的异常exception匹配上一个Method方法来处理
	// 若有多个匹配上的：使用ExceptionDepthComparator它来排序。若木有匹配的就返回null
	@Nullable
	public Method resolveMethod(Exception exception) {
		return resolveMethodByThrowable(exception);
	}
	// @since 5.0 递归到了couse异常类型 也会处理
	@Nullable
	public Method resolveMethodByThrowable(Throwable exception) {
		Method method = resolveMethodByExceptionType(exception.getClass());
		if (method == null) {
			Throwable cause = exception.getCause();
			if (cause != null) {
				method = resolveMethodByExceptionType(cause.getClass());
			}
		}
		return method;
	}

	//1、先去exceptionLookupCache找，若匹配上了直接返回
	// 2、再去mappedMethods这个缓存里找。很显然可能匹配上多个，那就用ExceptionDepthComparator排序匹配到一个最为合适的
	// 3、匹配上后放进缓存`exceptionLookupCache`，所以下次进来就不需要再次匹配了，这就是缓存的效果
	// ExceptionDepthComparator的基本理论上：精确匹配优先（按照深度比较）
	@Nullable
	public Method resolveMethodByExceptionType(Class<? extends Throwable> exceptionType) {
		Method method = this.exceptionLookupCache.get(exceptionType);
		if (method == null) {
			method = getMappedMethod(exceptionType);
			this.exceptionLookupCache.put(exceptionType, method);
		}
		return method;
	}
}
```

找到指定Class类（可能是Controller本身，也可能是@ControllerAdvice）里面所有标注有@ExceptionHandler的方法们
同一个Class内，不能出现同一个（注意理解同一个的含义）异常类型被多个Method处理的情况，否则抛出异常：Ambiguous @ExceptionHandler method mapped for ...
1. 相同异常类型处在不同的Class内的方法上是可以的，比如常见的一个在Controller内，一个在@ControllerAdvice内~
  提供缓存：
2. mappedMethods：每种异常对应的处理方法（直接映射代码上书写的异常-方法映射）
3. exceptionLookupCache：经过按照深度逻辑精确匹配上的Method方法
  既能处理本身的异常，也能够处理getCause()导致的异常
  ExceptionDepthComparator的匹配逻辑是按照深度匹配。比如发生的是NullPointerException，但是声明的异常有Throwable和Exception，这是它会根据异常的最近继承关系找到继承深度最浅的那个异常，即Exception。



1. 什么时候该抛异常，什么情况下它不是异常。（异常会使得JVM停顿，所以异常的使用请不要泛滥）
2. 对于异常的统一处理，请务必要分而治之。**不是所有异常都叫Exception~**
   \1. 合理的处理异常，这对于微服务架构在服务治理层面具有重要的意义，这也是对一个优秀架构师的考验之一

















