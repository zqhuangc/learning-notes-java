## HandlerMethod

`HandlerMethod`封装了**很多属性**，在访问请求方法的时候可以**方便的访问到方法、方法参数、方法上的注解、所属类**等并且对方法参数封装处理，也可以方便的访问到方法参数的注解等信息。

> `HandlerMethod`它只负责准备数据，封装数据，而不提供具体使用的方式方法

```java
// @since 3.1
public class HandlerMethod {

	// Object类型，既可以是个Bean，也可以是个BeanName
	private final Object bean;
	// 如果是BeanName，拿就靠它拿出Bean实例了~
	@Nullable
	private final BeanFactory beanFactory;
	private final Class<?> beanType; // 该方法所属的类
	private final Method method; // 该方法本身
	private final Method bridgedMethod; // 被桥接的方法,如果method是原生的,它的值同method
	// 封装方法参数的类实例，**一个MethodParameter就是一个入参**
	// MethodParameter也是Spring抽象出来的一个非常重要的概念
	private final MethodParameter[] parameters;
	@Nullable
	private HttpStatus responseStatus; // http状态码（毕竟它要负责处理和返回）
	@Nullable
	private String responseStatusReason; // 如果状态码里还要复数原因，就是这个字段  可以为null


	// 通过createWithResolvedBean()解析此handlerMethod实例的handlerMethod。
	@Nullable
	private HandlerMethod resolvedFromHandlerMethod;
	// 标注在**接口入参**上的注解们（此处数据结构复杂，List+二维数组）
	@Nullable
	private volatile List<Annotation[][]> interfaceParameterAnnotations;

	// 它的构造方法众多  此处我只写出关键的步骤
	public HandlerMethod(Object bean, Method method) {
		...
		this.beanType = ClassUtils.getUserClass(bean);
		this.bridgedMethod = BridgeMethodResolver.findBridgedMethod(method);
		this.parameters = initMethodParameters();
		...
		evaluateResponseStatus();
	}
	// 这个构造方法抛出了一个异常NoSuchMethodException 
	public HandlerMethod(Object bean, String methodName, Class<?>... parameterTypes) throws NoSuchMethodException {
		...
		this.method = bean.getClass().getMethod(methodName, parameterTypes);
		this.parameters = initMethodParameters();
		...
		evaluateResponseStatus();
	}
	// 此处传的是BeanName
	public HandlerMethod(String beanName, BeanFactory beanFactory, Method method) {
		...
		// 这部判断：这个BeanName是必须存在的
		Class<?> beanType = beanFactory.getType(beanName);
		if (beanType == null) {
			throw new IllegalStateException("Cannot resolve bean type for bean with name '" + beanName + "'");
		}
		this.parameters = initMethodParameters();
		...
		evaluateResponseStatus();
	}

	// 供给子类copy使用的
	protected HandlerMethod(HandlerMethod handlerMethod) { ... }
	
	// 所有构造都执行了两个方法：initMethodParameters和evaluateResponseStatus

	// 初始化该方法所有的入参，此处使用的是内部类HandlerMethodParameter
	// 注意：处理了泛型的~~~
	private MethodParameter[] initMethodParameters() {
		int count = this.bridgedMethod.getParameterCount();
		MethodParameter[] result = new MethodParameter[count];
		for (int i = 0; i < count; i++) {
			HandlerMethodParameter parameter = new HandlerMethodParameter(i);
			GenericTypeResolver.resolveParameterType(parameter, this.beanType);
			result[i] = parameter;
		}
		return result;
	}

	// 看看方法上是否有标注了@ResponseStatus注解（接口上或者父类 组合注解上都行）
	// 若方法上没有，还会去所在的类上去看看有没有标注此注解
	// 主要只解析这个注解，把它的两个属性code和reason拿过来，最后就是返回它俩了~~~
	// code状态码默认是HttpStatus.INTERNAL_SERVER_ERROR-->(500, "Internal Server Error")
	private void evaluateResponseStatus() {
		ResponseStatus annotation = getMethodAnnotation(ResponseStatus.class);
		if (annotation == null) {
			annotation = AnnotatedElementUtils.findMergedAnnotation(getBeanType(), ResponseStatus.class);
		}
		if (annotation != null) {
			this.responseStatus = annotation.code();
			this.responseStatusReason = annotation.reason();
		}
	}
	... // 省略所有属性的get方法（无set方法）

	// 返回方法返回值的类型  此处也使用的MethodParameter 
	public MethodParameter getReturnType() {
		return new HandlerMethodParameter(-1);
	}
	// 注意和上面的区别。举个列子：比如方法返回的是Object，但实际return “fsx”字符串
	// 那么上面返回永远是Object.class，下面你实际的值是什么类型就是什么类型
	public MethodParameter getReturnValueType(@Nullable Object returnValue) {
		return new ReturnValueMethodParameter(returnValue);
	}

	// 该方法的返回值是否是void
	public boolean isVoid() {
		return Void.TYPE.equals(getReturnType().getParameterType());
	}
	// 返回标注在方法上的指定类型的注解   父方法也成
	// 子类ServletInvocableHandlerMethod对下面两个方法都有复写~~~
	@Nullable
	public <A extends Annotation> A getMethodAnnotation(Class<A> annotationType) {
		return AnnotatedElementUtils.findMergedAnnotation(this.method, annotationType);
	}
	public <A extends Annotation> boolean hasMethodAnnotation(Class<A> annotationType) {
		return AnnotatedElementUtils.hasAnnotation(this.method, annotationType);
	}


	// resolvedFromHandlerMethod虽然它只能被构造进来，但是它实际是铜鼓调用下面方法赋值
	@Nullable
	public HandlerMethod getResolvedFromHandlerMethod() {
		return this.resolvedFromHandlerMethod;
	}
	// 根据string类型的BeanName把Bean拿出来，再new一个HandlerMethod出来~~~这才靠谱嘛
	public HandlerMethod createWithResolvedBean() {
		Object handler = this.bean;
		if (this.bean instanceof String) {
			Assert.state(this.beanFactory != null, "Cannot resolve bean name without BeanFactory");
			String beanName = (String) this.bean;
			handler = this.beanFactory.getBean(beanName);
		}
		return new HandlerMethod(this, handler);
	}

	public String getShortLogMessage() {
		return getBeanType().getName() + "#" + this.method.getName() + "[" + this.method.getParameterCount() + " args]";
	}


	// 这个方法是提供给内部类HandlerMethodParameter来使用的~~ 它使用的数据结构还是蛮复杂的
	private List<Annotation[][]> getInterfaceParameterAnnotations() {
		List<Annotation[][]> parameterAnnotations = this.interfaceParameterAnnotations;
		if (parameterAnnotations == null) {
			parameterAnnotations = new ArrayList<>();

			// 遍历该方法所在的类所有的实现的接口们（可以实现N个接口嘛）
			for (Class<?> ifc : this.method.getDeclaringClass().getInterfaces()) {
			
				// getMethods：拿到所有的public的方法，包括父接口的  接口里的私有方法可不会获取来
				for (Method candidate : ifc.getMethods()) {
					// 判断这个接口方法是否正好是当前method复写的这个~~~
					// 刚好是复写的方法，那就添加进来，标记为接口上的注解们~~~
					if (isOverrideFor(candidate)) {
						// getParameterAnnotations返回的是个二维数组~~~~
						// 因为参数有多个，且每个参数前可以有多个注解
						parameterAnnotations.add(candidate.getParameterAnnotations());
					}
				}
			}
			this.interfaceParameterAnnotations = parameterAnnotations;
		}
		return parameterAnnotations;
	}

	
	// 看看内部类的关键步骤
	protected class HandlerMethodParameter extends SynthesizingMethodParameter {
		@Nullable
		private volatile Annotation[] combinedAnnotations;
		...

		// 父类只会在本方法拿，这里支持到了接口级别~~~
		@Override
		public Annotation[] getParameterAnnotations() {
			Annotation[] anns = this.combinedAnnotations;
			if (anns == null) { // 都只需要解析一次
				anns = super.getParameterAnnotations();
				int index = getParameterIndex();
				if (index >= 0) { // 有入参才需要去分析嘛
					for (Annotation[][] ifcAnns : getInterfaceParameterAnnotations()) {
						if (index < ifcAnns.length) {
							Annotation[] paramAnns = ifcAnns[index];
							if (paramAnns.length > 0) {
								List<Annotation> merged = new ArrayList<>(anns.length + paramAnns.length);
								merged.addAll(Arrays.asList(anns));
								for (Annotation paramAnn : paramAnns) {
									boolean existingType = false;
									for (Annotation ann : anns) {
										if (ann.annotationType() == paramAnn.annotationType()) {
											existingType = true;
											break;
										}
									}
									if (!existingType) {
										merged.add(adaptAnnotation(paramAnn));
									}
								}
								anns = merged.toArray(new Annotation[0]);
							}
						}
					}
				}
				this.combinedAnnotations = anns;
			}
			return anns;
		}
	}

	// 返回值的真正类型~~~
	private class ReturnValueMethodParameter extends HandlerMethodParameter {
		@Nullable
		private final Object returnValue;
		public ReturnValueMethodParameter(@Nullable Object returnValue) {
			super(-1); // 此处传的-1哦~~~~ 比0小是很有意义的
			this.returnValue = returnValue;
		}
		...
		// 返回值类型使用returnValue就行了~~~
		@Override
		public Class<?> getParameterType() {
			return (this.returnValue != null ? this.returnValue.getClass() : super.getParameterType());
		}
	}
}
```

@ResponseStatus



### InvocableHandlerMethod#invokeForRequest

它能够在调用的时候，把方法入参的参数都封装进来（从`HTTP request`里，借助HandlerMethodArgumentResolver）

```java
// @since 3.1
public class InvocableHandlerMethod extends HandlerMethod {
	private static final Object[] EMPTY_ARGS = new Object[0];

	// 它额外提供的几个属性，可以看到和数据绑定、数据校验就扯上关系了~~~

	// 用于产生数据绑定器、校验器
	@Nullable
	private WebDataBinderFactory dataBinderFactory;
	// HandlerMethodArgumentResolver用于入参的解析
	private HandlerMethodArgumentResolverComposite resolvers = new HandlerMethodArgumentResolverComposite();
	// 用于获取形参名
	private ParameterNameDiscoverer parameterNameDiscoverer = new DefaultParameterNameDiscoverer();
	
	... // 省略构造函数 全部使用super的
	// 它自己的三大属性都使用set方法设置进来~~~并且没有提供get方法
	// 也就是说：它自己内部使用就行了~~~

	// 在给定请求的上下文中解析方法的参数值后调用该方法。 也就是说：方法入参里就能够自动使用请求域（包括path里的，requestParam里的、以及常规对象如HttpSession这种）
	// 解释下providedArgs作用：调用者可以传进来，然后直接doInvoke()的时候原封不动的使用它
	//（弥补了请求域没有所有对象的不足，毕竟有些对象是用户自定义的嘛~）
	@Nullable
	public Object invokeForRequest(NativeWebRequest request, @Nullable ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
		// 虽然它是最重要的方法，但是此处不讲，因为核心原来还是`HandlerMethodArgumentResolver`
		// 它只是把解析好的放到对应位置里去~~~
		// 说明：这里传入了ParameterNameDiscoverer，它是能够获取到形参名的。
		// 这就是为何注解里我们不写value值，通过形参名字来匹配也是ok的核心原因~
		Object[] args = getMethodArgumentValues(request, mavContainer, providedArgs);
		if (logger.isTraceEnabled()) { // trace信息，否则日志也特多了~
			logger.trace("Arguments: " + Arrays.toString(args));
		}
		return doInvoke(args);
	}

	// doInvoke()方法就不说了，就是个普通的方法调用
	// ReflectionUtils.makeAccessible(getBridgedMethod());
	// return getBridgedMethod().invoke(getBean(), args); 
}
```



#### ServletInvocableHandlerMethod

增加了**返回值和响应状态码的处理**,内部类`ConcurrentResultHandlerMethod`继承于它，支持**异常调用结果**处理，`Servlet`容器下`Controller`在查找适配器时发起调用的最终就是`ServletInvocableHandlerMethod`。

```java
public class ServletInvocableHandlerMethod extends InvocableHandlerMethod {
	private static final Method CALLABLE_METHOD = ClassUtils.getMethod(Callable.class, "call");

	// 处理方法返回值
	@Nullable
	private HandlerMethodReturnValueHandlerComposite returnValueHandlers;

	// 构造函数略
	
	// 设置处理返回值的HandlerMethodReturnValueHandler
	public void setHandlerMethodReturnValueHandlers(HandlerMethodReturnValueHandlerComposite returnValueHandlers) {
		this.returnValueHandlers = returnValueHandlers;
	}


	// 它不是复写，但是是对invokeForRequest方法的进一步增强  因为调用目标方法还是靠invokeForRequest
	// 本处是把方法的返回值拿来进一步处理~~~比如状态码之类的
	public void invokeAndHandle(ServletWebRequest webRequest, ModelAndViewContainer mavContainer, Object... providedArgs) throws Exception {
		Object returnValue = invokeForRequest(webRequest, mavContainer, providedArgs);
		// 设置HttpServletResponse返回状态码 这里面还是有点意思的  因为@ResponseStatus#code()在父类已经解析了  但是子类才用
		setResponseStatus(webRequest);


		// 重点是这一句话：mavContainer.setRequestHandled(true); 表示该请求已经被处理过了
		if (returnValue == null) {

			// Request的NotModified为true 有@ResponseStatus注解标注 RequestHandled=true 三个条件有一个成立,则设置请求处理完成并返回
			if (isRequestNotModified(webRequest) || getResponseStatus() != null || mavContainer.isRequestHandled()) {
				mavContainer.setRequestHandled(true);
				return;
			}
		// 返回值不为null,@ResponseStatus存在reason 同样设置请求处理完成并返回
		} else if (StringUtils.hasText(getResponseStatusReason())) {
			mavContainer.setRequestHandled(true);
			return;
		}

		// 前边都不成立,则设置RequestHandled=false即请求未完成
		// 继续交给HandlerMethodReturnValueHandlerComposite处理
		// 可见@ResponseStatus的优先级还是蛮高的~~~~~
		mavContainer.setRequestHandled(false);
		Assert.state(this.returnValueHandlers != null, "No return value handlers");
		try {
		
			// 关于对方法返回值的处理，参见：https://blog.csdn.net/f641385712/article/details/90370542
			this.returnValueHandlers.handleReturnValue(returnValue, getReturnValueType(returnValue), mavContainer, webRequest);
		} catch (Exception ex) {
			if (logger.isTraceEnabled()) {
				logger.trace(formatErrorForReturnValue(returnValue), ex);
			}
			throw ex;
		}
	}

	// 设置返回的状态码到HttpServletResponse 里面去
	private void setResponseStatus(ServletWebRequest webRequest) throws IOException {
		HttpStatus status = getResponseStatus();
		if (status == null) { // 如果调用者没有标注ResponseStatus.code()此注解  此处就忽略它
			return;
		}

		HttpServletResponse response = webRequest.getResponse();
		if (response != null) {
			String reason = getResponseStatusReason();

			// 此处务必注意：若有reason，那就是sendError  哪怕你是200哦~
			if (StringUtils.hasText(reason)) {
				response.sendError(status.value(), reason);
			} else {
				response.setStatus(status.value());
			}
		}

		// 设置到request的属性，把响应码给过去。为了在redirect中使用
		// To be picked up by RedirectView
		webRequest.getRequest().setAttribute(View.RESPONSE_STATUS_ATTRIBUTE, status);
	}

	private boolean isRequestNotModified(ServletWebRequest webRequest) {
		return webRequest.isNotModified();
	}


	// 这个方法RequestMappingHandlerAdapter里有调用
	ServletInvocableHandlerMethod wrapConcurrentResult(Object result) {
		return new ConcurrentResultHandlerMethod(result, new ConcurrentResultMethodParameter(result));
	}

	// 内部类们
	private class ConcurrentResultMethodParameter extends HandlerMethodParameter {
		@Nullable
		private final Object returnValue;
		private final ResolvableType returnType;
		public ConcurrentResultMethodParameter(Object returnValue) {
			super(-1);
			this.returnValue = returnValue;
			// 主要是这个解析 兼容到了泛型类型 比如你的返回值是List<Person> 它也能把你的类型拿出来
			this.returnType = (returnValue instanceof ReactiveTypeHandler.CollectedValuesList ?
					((ReactiveTypeHandler.CollectedValuesList) returnValue).getReturnType() :
					ResolvableType.forType(super.getGenericParameterType()).getGeneric());
		}

		// 若返回的是List  这里就是List的类型哦  下面才是返回泛型类型
		@Override
		public Class<?> getParameterType() {
			if (this.returnValue != null) {
				return this.returnValue.getClass();
			}
			if (!ResolvableType.NONE.equals(this.returnType)) {
				return this.returnType.toClass();
			}
			return super.getParameterType();
		}

		// 返回泛型类型
		@Override
		public Type getGenericParameterType() {
			return this.returnType.getType();
		}


		// 即使实际返回类型为ResponseEntity<Flux<T>>，也要确保对@ResponseBody-style处理从reactive 类型中收集值
		// 是对reactive 的一种兼容
		@Override
		public <T extends Annotation> boolean hasMethodAnnotation(Class<T> annotationType) {
			// Ensure @ResponseBody-style handling for values collected from a reactive type
			// even if actual return type is ResponseEntity<Flux<T>>
			return (super.hasMethodAnnotation(annotationType) ||
					(annotationType == ResponseBody.class && this.returnValue instanceof ReactiveTypeHandler.CollectedValuesList));
		}
	}


	// 这个非常有意思   内部类继承了自己（外部类） 进行增强
	private class ConcurrentResultHandlerMethod extends ServletInvocableHandlerMethod {
		// 返回值
		private final MethodParameter returnType;

		// 此构造最终传入的handler是个Callable
		// result方法返回值 它支持支持异常调用结果处理
		public ConcurrentResultHandlerMethod(final Object result, ConcurrentResultMethodParameter returnType) {
			super((Callable<Object>) () -> {
				if (result instanceof Exception) {
					throw (Exception) result;
				} else if (result instanceof Throwable) {
					throw new NestedServletException("Async processing failed", (Throwable) result);
				}
				return result;
			}, CALLABLE_METHOD);


			// 给外部类把值设置上  因为wrapConcurrentResult一般都先调用，是对本类的一个增强
			if (ServletInvocableHandlerMethod.this.returnValueHandlers != null) {
				setHandlerMethodReturnValueHandlers(ServletInvocableHandlerMethod.this.returnValueHandlers);
			}
			this.returnType = returnType;
		}
		...
	}
}
```



## HandlerMethodArgumentResolver

策略接口：用于在**给定请求的上下文中**将方法参数解析为参数值。简单的理解为：它负责处理你`Handler`方法里的**所有入参**：包括自动封装、自动赋值、校验等等。有了它才能会让`Spring MVC`处理入参显得那么高级、那么自动化。

`HandlerMethodArgumentResolver = HandlerMethod + Argument(参数) + Resolver(解析器)`。

`HandlerMethod`方法的解析器，将`HttpServletRequest(header + body 中的内容)`解析为`HandlerMethod`方法的参数（method parameters）

```java
// @since 3.1   HandlerMethod 方法中 参数解析器
public interface HandlerMethodArgumentResolver {

	// 判断 HandlerMethodArgumentResolver 是否支持 MethodParameter
	// (PS: 一般都是通过 参数上面的注解|参数的类型)
	boolean supportsParameter(MethodParameter parameter);
	
	// 从NativeWebRequest中获取数据，ModelAndViewContainer用来提供访问Model
	// MethodParameter parameter：请求参数
	// WebDataBinderFactory用于创建一个WebDataBinder用于数据绑定、校验
	@Nullable
	Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception;
}
```



1. 基于`Name`
2. 数据类型是`Map`的
3. 固定参数类型
4. 基于`ContentType`的消息转换器



### AbstractNamedValueMethodArgumentResolver

> 基于MethodParameter构建NameValueInfo <-- 主要有name, defaultValue, required（其实主要是解析方法参数上标注的注解~）
> 通过BeanExpressionResolver(${}占位符以及SpEL) 解析name
> 通过模版方法resolveName从 HttpServletRequest, Http Headers, URI template variables 等等中获取对应的属性值（具体由子类去实现）
> 对 arg==null这种情况的处理, 要么使用默认值, 若 required = true && arg == null, 则一般报出异常（boolean类型除外~）
> 通过WebDataBinder将arg转换成Methodparameter.getParameterType()类型（注意：这里仅仅只是用了数据转换而已，并没有用bind()方法）
>

```java
// @since 3.1  负责从路径变量、请求、头等中拿到值。（都可以指定name、required、默认值等属性）
// 子类需要做如下事：获取方法参数的命名值信息、将名称解析为参数值
// 当需要参数值时处理缺少的参数值、可选地处理解析值

//特别注意的是：默认值可以使用${}占位符，或者SpEL语句#{}是木有问题的
public abstract class AbstractNamedValueMethodArgumentResolver implements HandlerMethodArgumentResolver {

	@Nullable
	private final ConfigurableBeanFactory configurableBeanFactory;
	@Nullable
	private final BeanExpressionContext expressionContext;
	private final Map<MethodParameter, NamedValueInfo> namedValueInfoCache = new ConcurrentHashMap<>(256);

	public AbstractNamedValueMethodArgumentResolver() {
		this.configurableBeanFactory = null;
		this.expressionContext = null;
	}
	public AbstractNamedValueMethodArgumentResolver(@Nullable ConfigurableBeanFactory beanFactory) {
		this.configurableBeanFactory = beanFactory;
		// 默认是RequestScope
		this.expressionContext = (beanFactory != null ? new BeanExpressionContext(beanFactory, new RequestScope()) : null);
	}

	// protected的内部类  所以所有子类（注解）都是用友这三个属性值的
	protected static class NamedValueInfo {
		private final String name;
		private final boolean required;
		@Nullable
		private final String defaultValue;
		public NamedValueInfo(String name, boolean required, @Nullable String defaultValue) {
			this.name = name;
			this.required = required;
			this.defaultValue = defaultValue;
		}
	}

	// 核心方法  注意此方法是final的，并不希望子类覆盖掉他~
	@Override
	@Nullable
	public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

		// 创建 MethodParameter 对应的 NamedValueInfo
		NamedValueInfo namedValueInfo = getNamedValueInfo(parameter);
		// 支持到了Java 8 中支持的 java.util.Optional
		MethodParameter nestedParameter = parameter.nestedIfOptional();

		// name属性（也就是注解标注的value/name属性）这里既会解析占位符，还会解析SpEL表达式，非常强大
		// 因为此时的 name 可能还是被 ${} 符号包裹, 则通过 BeanExpressionResolver 来进行解析
		Object resolvedName = resolveStringValue(namedValueInfo.name);
		if (resolvedName == null) {
			throw new IllegalArgumentException("Specified name must not resolve to null: [" + namedValueInfo.name + "]");
		}


		// 模版抽象方法：将给定的参数类型和值名称解析为参数值。  由子类去实现
		// @PathVariable     --> 通过对uri解析后得到的decodedUriVariables值(常用)
		// @RequestParam     --> 通过 HttpServletRequest.getParameterValues(name) 获取（常用）
		// @RequestAttribute --> 通过 HttpServletRequest.getAttribute(name) 获取   <-- 这里的 scope 是 request
		// @SessionAttribute --> 略
		// @RequestHeader    --> 通过 HttpServletRequest.getHeaderValues(name) 获取
		// @CookieValue      --> 通过 HttpServletRequest.getCookies() 获取
		Object arg = resolveName(resolvedName.toString(), nestedParameter, webRequest);

		// 若解析出来值仍旧为null，那就走defaultValue （若指定了的话）
		if (arg == null) {
			// 可以发现：defaultValue也是支持占位符和SpEL的~~~
			if (namedValueInfo.defaultValue != null) {
				arg = resolveStringValue(namedValueInfo.defaultValue);

			// 若 arg == null && defaultValue == null && 非 optional 类型的参数 则通过 handleMissingValue 来进行处理, 一般是报异常
			} else if (namedValueInfo.required && !nestedParameter.isOptional()) {
				
				// 它是个protected方法，默认抛出ServletRequestBindingException异常
				// 各子类都复写了此方法，转而抛出自己的异常（但都是ServletRequestBindingException的异常子类）
				handleMissingValue(namedValueInfo.name, nestedParameter, webRequest);
			}
	
			// handleNullValue是private方法，来处理null值
			// 针对Bool类型有这个判断：Boolean.TYPE.equals(paramType) 就return Boolean.FALSE;
			// 此处注意：Boolean.TYPE = Class.getPrimitiveClass("boolean") 它指的基本类型的boolean，而不是Boolean类型哦~~~
			// 如果到了这一步（value是null），但你还是基本类型，那就抛出异常了（只有boolean类型不会抛异常哦~）
			// 这里多嘴一句，即使请求传值为&bool=1，效果同bool=true的（1：true 0：false） 并且不区分大小写哦（TrUe效果同true）
			arg = handleNullValue(namedValueInfo.name, arg, nestedParameter.getNestedParameterType());
		}
		// 兼容空串，若传入的是空串，依旧还是使用默认值（默认值支持占位符和SpEL）
		else if ("".equals(arg) && namedValueInfo.defaultValue != null) {
			arg = resolveStringValue(namedValueInfo.defaultValue);
		}

		// 完成自动化的数据绑定~~~
		if (binderFactory != null) {
			WebDataBinder binder = binderFactory.createBinder(webRequest, null, namedValueInfo.name);
			try {
				// 通过数据绑定器里的Converter转换器把arg转换为指定类型的数值
				arg = binder.convertIfNecessary(arg, parameter.getParameterType(), parameter);
			} catch (ConversionNotSupportedException ex) { // 注意这个异常：MethodArgumentConversionNotSupportedException  类型不匹配的异常
				throw new MethodArgumentConversionNotSupportedException(arg, ex.getRequiredType(),
						namedValueInfo.name, parameter, ex.getCause());
			} catch (TypeMismatchException ex) { //MethodArgumentTypeMismatchException是TypeMismatchException 的子类
				throw new MethodArgumentTypeMismatchException(arg, ex.getRequiredType(),
						namedValueInfo.name, parameter, ex.getCause());

			}
		}

		// protected的方法，本类为空实现，交给子类去复写（并不是必须的）
		// 唯独只有PathVariableMethodArgumentResolver把解析处理啊的值存储一下数据到 
		// HttpServletRequest.setAttribute中（若key已经存在也不会存储了）
		handleResolvedValue(arg, namedValueInfo.name, parameter, mavContainer, webRequest);
		return arg;
	}


	// 此处有缓存，记录下每一个MethodParameter对象   value是NamedValueInfo值
	private NamedValueInfo getNamedValueInfo(MethodParameter parameter) {
		NamedValueInfo namedValueInfo = this.namedValueInfoCache.get(parameter);
		if (namedValueInfo == null) {
			// createNamedValueInfo是抽象方法，子类必须实现
			namedValueInfo = createNamedValueInfo(parameter);
			// updateNamedValueInfo：这一步就是我们之前说过的为何Spring MVC可以根据参数名封装的方法
			// 如果info.name.isEmpty()的话（注解里没指定名称），就通过`parameter.getParameterName()`去获取参数名~
			// 它还会处理注解指定的defaultValue：`\n\t\.....`等等都会被当作null处理
			// 都处理好后：new NamedValueInfo(name, info.required, defaultValue);（相当于吧注解解析成了此对象嘛~~）
			namedValueInfo = updateNamedValueInfo(parameter, namedValueInfo);
			this.namedValueInfoCache.put(parameter, namedValueInfo);
		}
		return namedValueInfo;
	}

	// 抽象方法 
	protected abstract NamedValueInfo createNamedValueInfo(MethodParameter parameter);
	// 由子类根据名称，去把值拿出来
	protected abstract Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception;
}
```

#### PathVariableMethodArgumentResolver

>**非RESTful接口的性能是RESTful接口的两倍，接口相应时间上更是达到10倍左右**

```java
// @since 3.1
public class PathVariableMethodArgumentResolver extends AbstractNamedValueMethodArgumentResolver implements UriComponentsContributor {
	private static final TypeDescriptor STRING_TYPE_DESCRIPTOR = TypeDescriptor.valueOf(String.class);


	// 简单一句话描述：@PathVariable是必须，不管你啥类型
	// 标注了注解，且是Map类型，
	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		if (!parameter.hasParameterAnnotation(PathVariable.class)) {
			return false;
		}
		if (Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType())) {
			PathVariable pathVariable = parameter.getParameterAnnotation(PathVariable.class);
			return (pathVariable != null && StringUtils.hasText(pathVariable.value()));
		}
		return true;
	}

	@Override
	protected NamedValueInfo createNamedValueInfo(MethodParameter parameter) {
		PathVariable ann = parameter.getParameterAnnotation(PathVariable.class);
		return new PathVariableNamedValueInfo(ann);
	}
	private static class PathVariableNamedValueInfo extends NamedValueInfo {
		public PathVariableNamedValueInfo(PathVariable annotation) {
			// 默认值使用的DEFAULT_NONE~~~
			super(annotation.name(), annotation.required(), ValueConstants.DEFAULT_NONE);
		}
	}

	// 根据name去拿值的过程非常之简单，但是它和前面的只知识是有关联的
	// 至于这个attr是什么时候放进去的，AbstractHandlerMethodMapping.handleMatch()匹配处理器方法上
	// 通过UrlPathHelper.decodePathVariables() 把参数提取出来了，然后放进request属性上暂存了~~~
	// 关于HandlerMapping内容，可来这里：https://blog.csdn.net/f641385712/article/details/89810020
	@Override
	@Nullable
	protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
		Map<String, String> uriTemplateVars = (Map<String, String>) request.getAttribute(HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE, RequestAttributes.SCOPE_REQUEST);
		return (uriTemplateVars != null ? uriTemplateVars.get(name) : null);
	}

	// MissingPathVariableException是ServletRequestBindingException的子类
	@Override
	protected void handleMissingValue(String name, MethodParameter parameter) throws ServletRequestBindingException {
		throw new MissingPathVariableException(name, parameter);
	}


	// 值完全处理结束后，把处理好的值放进请求域，方便view里渲染时候使用~
	// 抽象父类的handleResolvedValue方法，只有它复写了~
	@Override
	@SuppressWarnings("unchecked")
	protected void handleResolvedValue(@Nullable Object arg, String name, MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest request) {

		String key = View.PATH_VARIABLES;
		int scope = RequestAttributes.SCOPE_REQUEST;
		Map<String, Object> pathVars = (Map<String, Object>) request.getAttribute(key, scope);
		if (pathVars == null) {
			pathVars = new HashMap<>();
			request.setAttribute(key, pathVars, scope);
		}
		pathVars.put(name, arg);
	}
	...
}
```



#### RequestParamMethodArgumentResolver

```java
// @since 3.1
public class RequestParamMethodArgumentResolver extends AbstractNamedValueMethodArgumentResolver implements UriComponentsContributor {

	private static final TypeDescriptor STRING_TYPE_DESCRIPTOR = TypeDescriptor.valueOf(String.class);

	// 这个参数老重要了：
	// true：表示参数类型是基本类型 参考BeanUtils#isSimpleProperty(什么Enum、Number、Date、URL、包装类型、以上类型的数组类型等等)
	// 如果是基本类型，即使你不写@RequestParam注解，它也是会走进来处理的~~~(这个@PathVariable可不会哟~)
	// fasle：除上以外的。  要想它处理就必须标注注解才行哦，比如List等~
	// 默认值是false
	private final boolean useDefaultResolution;

	// 此构造只有`MvcUriComponentsBuilder`调用了  传入的false
	public RequestParamMethodArgumentResolver(boolean useDefaultResolution) {
		this.useDefaultResolution = useDefaultResolution;
	}
	// 传入了ConfigurableBeanFactory ，所以它支持处理占位符${...} 并且支持SpEL了
	// 此构造都在RequestMappingHandlerAdapter里调用，最后都会传入true来Catch-all Case  这种设计挺有意思的
	public RequestParamMethodArgumentResolver(@Nullable ConfigurableBeanFactory beanFactory, boolean useDefaultResolution) {
		super(beanFactory);
		this.useDefaultResolution = useDefaultResolution;
	}

	// 此处理器能处理如下Case：
	// 1、所有标注有@RequestParam注解的类型（非Map）/ 注解指定了value值的Map类型（自己提供转换器哦）
	// ======下面都表示没有标注@RequestParam注解了的=======
	// 1、不能标注有@RequestPart注解，否则直接不处理了
	// 2、是上传的request：isMultipartArgument() = true（MultipartFile类型或者对应的集合/数组类型  或者javax.servlet.http.Part对应结合/数组类型）
	// 3、useDefaultResolution=true情况下，"基本类型"也会处理
	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		if (parameter.hasParameterAnnotation(RequestParam.class)) {
			if (Map.class.isAssignableFrom(parameter.nestedIfOptional().getNestedParameterType())) {
				RequestParam requestParam = parameter.getParameterAnnotation(RequestParam.class);
				return (requestParam != null && StringUtils.hasText(requestParam.name()));
			} else {
				return true;
			}
		} else {
			if (parameter.hasParameterAnnotation(RequestPart.class)) {
				return false;
			}
			parameter = parameter.nestedIfOptional();
			if (MultipartResolutionDelegate.isMultipartArgument(parameter)) {
				return true;
			} else if (this.useDefaultResolution) {
				return BeanUtils.isSimpleProperty(parameter.getNestedParameterType());
			} else {
				return false;
			}
		}
	}


	// 从这也可以看出：即使木有@RequestParam注解，也是可以创建出一个NamedValueInfo来的
	@Override
	protected NamedValueInfo createNamedValueInfo(MethodParameter parameter) {
		RequestParam ann = parameter.getParameterAnnotation(RequestParam.class);
		return (ann != null ? new RequestParamNamedValueInfo(ann) : new RequestParamNamedValueInfo());
	}

	// 内部类
	private static class RequestParamNamedValueInfo extends NamedValueInfo {
		// 请注意这个默认值：如果你不写@RequestParam，那么就会用这个默认值
		// 注意：required = false的哟（若写了注解，required默认可是true，请务必注意区分）
		// 因为不写注解的情况下，若是简单类型参数都是交给此处理器处理的。所以这个机制需要明白
		// 复杂类型（非简单类型）默认是ModelAttributeMethodProcessor处理的
		public RequestParamNamedValueInfo() {
			super("", false, ValueConstants.DEFAULT_NONE);
		}
		public RequestParamNamedValueInfo(RequestParam annotation) {
			super(annotation.name(), annotation.required(), annotation.defaultValue());
		}
	}

	// 核心方法：根据Name 获取值（普通/文件上传）
	// 并且还有集合、数组等情况
	@Override
	@Nullable
	protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest request) throws Exception {
		HttpServletRequest servletRequest = request.getNativeRequest(HttpServletRequest.class);

		// 这块解析出来的是个MultipartFile或者其集合/数组
		if (servletRequest != null) {
			Object mpArg = MultipartResolutionDelegate.resolveMultipartArgument(name, parameter, servletRequest);
			if (mpArg != MultipartResolutionDelegate.UNRESOLVABLE) {
				return mpArg;
			}
		}

		Object arg = null;
		MultipartRequest multipartRequest = request.getNativeRequest(MultipartRequest.class);
		if (multipartRequest != null) {
			List<MultipartFile> files = multipartRequest.getFiles(name);
			if (!files.isEmpty()) {
				arg = (files.size() == 1 ? files.get(0) : files);
			}
		}

		// 若解析出来值仍旧为null，那处理完文件上传里木有，那就去参数里取吧
		// 由此可见：文件上传的优先级是高于请求参数的
		if (arg == null) {
		
			//小知识点：getParameter()其实本质是getParameterNames()[0]的效果
			// 强调一遍：?ids=1,2,3 结果是["1,2,3"]（兼容方式，不建议使用。注意：只能是逗号分隔）
			// ?ids=1&ids=2&ids=3  结果是[1,2,3]（标准的传值方式，建议使用）
			// 但是Spring MVC这两种都能用List接收  请务必注意他们的区别~~~
			String[] paramValues = request.getParameterValues(name);
			if (paramValues != null) {
				arg = (paramValues.length == 1 ? paramValues[0] : paramValues);
			}
		}
		return arg;
	}
	...
}
```



多个值**只能**使用`,`号分隔才行



#### RequestHeaderMethodArgumentResolver

#### AbstractCookieValueMethodArgumentResolver（抽象类）

#### MatrixVariableMethodArgumentResolver

/owners/42;q=11/pets/21;s=23;q=22



#### ExpressionValueMethodArgumentResolver（@Value）

```java
// @since 3.1
public class ExpressionValueMethodArgumentResolver extends AbstractNamedValueMethodArgumentResolver {
	// 唯一构造函数  支持占位符、SpEL
	public ExpressionValueMethodArgumentResolver(@Nullable ConfigurableBeanFactory beanFactory) {
		super(beanFactory);
	}

	//必须标注有@Value注解
	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		return parameter.hasParameterAnnotation(Value.class);
	}

	@Override
	protected NamedValueInfo createNamedValueInfo(MethodParameter parameter) {
		Value ann = parameter.getParameterAnnotation(Value.class);
		return new ExpressionValueNamedValueInfo(ann);
	}
	private static final class ExpressionValueNamedValueInfo extends NamedValueInfo {
		// 这里name传值为固定值  因为只要你的key不是这个就木有问题
		// required传固定值false
		// defaultValue：取值为annotation.value() --> 它天然支持占位符和SpEL嘛
		private ExpressionValueNamedValueInfo(Value annotation) {
			super("@Value", false, annotation.value());
		}
	}

	// 这里恒返回null，因此即使你的key是@Value，也是不会采纳你的传值的哟~
	@Override
	@Nullable
	protected Object resolveName(String name, MethodParameter parameter, NativeWebRequest webRequest) throws Exception {
		// No name to resolve
		return null;
	}
}
```



### Map

#### PathVariableMapMethodArgumentResolver

#### RequestParamMapMethodArgumentResolver

#### RequestHeaderMapMethodArgumentResolver



#### MapMethodProcessor



### 固定参数类型

#### ServletRequestMethodArgumentResolver

#### ServletResponseMethodArgumentResolver

#### SessionStatusMethodArgumentResolver

#### UriComponentsBuilderMethodArgumentResolver

#### RedirectAttributesMethodArgumentResolver

#### ModelMethodProcessor



### 基于`ContentType`消息转换器类型

#### AbstractMessageConverterMethodArgumentResolver

```java
// @since 3.1
public abstract class AbstractMessageConverterMethodArgumentResolver implements HandlerMethodArgumentResolver {

	// 默认支持的方法（没有Deleted方法）
	// httpMethod为null 或者方法不属于这集中 或者没有contendType且没有body 那就返回null
	// 也就是说如果是Deleted请求，即使body里有值也是返回null的。（因为它不是SUPPORTED_METHODS ）
	private static final Set<HttpMethod> SUPPORTED_METHODS = EnumSet.of(HttpMethod.POST, HttpMethod.PUT, HttpMethod.PATCH);
	private static final Object NO_VALUE = new Object();

	protected final List<HttpMessageConverter<?>> messageConverters;
	protected final List<MediaType> allSupportedMediaTypes;
	// 和RequestBodyAdvice和ResponseBodyAdvice有关的
	private final RequestResponseBodyAdviceChain advice;

	// 构造函数里指定HttpMessageConverter
	// 此一个参数的构造函数木人调用
	public AbstractMessageConverterMethodArgumentResolver(List<HttpMessageConverter<?>> converters) {
		this(converters, null);
	}

	// @since 4.2
	public AbstractMessageConverterMethodArgumentResolver(List<HttpMessageConverter<?>> converters, @Nullable List<Object> requestResponseBodyAdvice) {
		Assert.notEmpty(converters, "'messageConverters' must not be empty");
		this.messageConverters = converters;
		// 它会把所有的消息转换器里支持的MediaType都全部拿出来汇聚起来~
		this.allSupportedMediaTypes = getAllSupportedMediaTypes(converters);
		this.advice = new RequestResponseBodyAdviceChain(requestResponseBodyAdvice);
	}

	// 提供一个defualt方法访问
	RequestResponseBodyAdviceChain getAdvice() {
		return this.advice;
	}

	// 子类RequestResponseBodyMethodProcessor有复写此方法
	@Nullable
	protected <T> Object readWithMessageConverters(NativeWebRequest webRequest, MethodParameter parameter, Type paramType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {

		HttpInputMessage inputMessage = createInputMessage(webRequest);
		return readWithMessageConverters(inputMessage, parameter, paramType);
	}
	...
}

```



#### RequestPartMethodArgumentResolver

用于解析参数被`@RequestPart`修饰，或者参数类型是`MultipartFile | Servlet 3.0提供的javax.servlet.http.Part`类型（并且没有被`@RequestParam`修饰），数据通过 `HttpServletRequest`获取



#### AbstractMessageConverterMethodProcessor

```java
// @since 3.1
public abstract class AbstractMessageConverterMethodProcessor extends AbstractMessageConverterMethodArgumentResolver implements HandlerMethodReturnValueHandler {

	// 默认情况下：文件们后缀是这些就不弹窗下载
	private static final Set<String> WHITELISTED_EXTENSIONS = new HashSet<>(Arrays.asList("txt", "text", "yml", "properties", "csv",
			"json", "xml", "atom", "rss", "png", "jpe", "jpeg", "jpg", "gif", "wbmp", "bmp"));
	private static final Set<String> WHITELISTED_MEDIA_BASE_TYPES = new HashSet<>(Arrays.asList("audio", "image", "video"));
	private static final List<MediaType> ALL_APPLICATION_MEDIA_TYPES = Arrays.asList(MediaType.ALL, new MediaType("application"));
	private static final Type RESOURCE_REGION_LIST_TYPE = new ParameterizedTypeReference<List<ResourceRegion>>() { }.getType();
	
	// 用于给URL解码 decodingUrlPathHelper.decodeRequestString(servletRequest, filename);
	private static final UrlPathHelper decodingUrlPathHelper = new UrlPathHelper();
	// rawUrlPathHelper.getOriginatingRequestUri(servletRequest);
	private static final UrlPathHelper rawUrlPathHelper = new UrlPathHelper();
	static {
		rawUrlPathHelper.setRemoveSemicolonContent(false);
		rawUrlPathHelper.setUrlDecode(false);
	}

	// 内容协商管理器
	private final ContentNegotiationManager contentNegotiationManager;
	// 扩展名的内容协商策略
	private final PathExtensionContentNegotiationStrategy pathStrategy;
	private final Set<String> safeExtensions = new HashSet<>();

	protected AbstractMessageConverterMethodProcessor(List<HttpMessageConverter<?>> converters) {
		this(converters, null, null);
	}
	// 可以指定内容协商管理器ContentNegotiationManager 
	protected AbstractMessageConverterMethodProcessor(List<HttpMessageConverter<?>> converters, @Nullable ContentNegotiationManager contentNegotiationManager) {
		this(converters, contentNegotiationManager, null);
	}
	// 这个构造器才是重点
	protected AbstractMessageConverterMethodProcessor(List<HttpMessageConverter<?>> converters, @Nullable ContentNegotiationManager manager, @Nullable List<Object> requestResponseBodyAdvice) {
		super(converters, requestResponseBodyAdvice);

		// 可以看到：默认情况下会直接new一个
		this.contentNegotiationManager = (manager != null ? manager : new ContentNegotiationManager());
		// 若管理器里有就用管理器里的，否则new PathExtensionContentNegotiationStrategy()
		this.pathStrategy = initPathStrategy(this.contentNegotiationManager);

		// 用safeExtensions装上内容协商所支持的所有后缀
		// 并且把后缀白名单也加上去（表示是默认支持的后缀）
		this.safeExtensions.addAll(this.contentNegotiationManager.getAllFileExtensions());
		this.safeExtensions.addAll(WHITELISTED_EXTENSIONS);
	}

	// ServletServerHttpResponse是对HttpServletResponse的包装，主要是对响应头进行处理
	// 主要是处理：setContentType、setCharacterEncoding等等
	// 所以子类若要写数据，就调用此方法来向输出流里写吧~~~
	protected ServletServerHttpResponse createOutputMessage(NativeWebRequest webRequest) {
		HttpServletResponse response = webRequest.getNativeResponse(HttpServletResponse.class);
		Assert.state(response != null, "No HttpServletResponse");
		return new ServletServerHttpResponse(response);
	}

	// 注意：createInputMessage()方法是父类提供的，对HttpServletRequest的包装
	// 主要处理了：getURI()、getHeaders()等方法
	// getHeaders()方法主要是处理了：getContentType()...


	protected <T> void writeWithMessageConverters(T value, MethodParameter returnType, NativeWebRequest webRequest) throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
		ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
		ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);
		writeWithMessageConverters(value, returnType, inputMessage, outputMessage);
	}

	// 这个方法省略
	// 这个方法是消息处理的核心之核心：处理了contentType、消息转换、内容协商、下载等等
	// 注意：此处并且还会执行RequestResponseBodyAdviceChain，进行前后拦截
	protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
			ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException { ... }
}
```



##### RequestResponseBodyMethodProcessor

负责处理`@RequestBody`这个注解的参数

```java
public class RequestResponseBodyMethodProcessor extends AbstractMessageConverterMethodProcessor {
	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		return parameter.hasParameterAnnotation(RequestBody.class);
	}

	@Override
	public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

		parameter = parameter.nestedIfOptional();
		// 所以核心逻辑：读取流、消息换换等都在父类里已经完成。子类直接调用就可以拿到转换后的值arg 
		// arg 一般都是个类对象。比如Person实例
		Object arg = readWithMessageConverters(webRequest, parameter, parameter.getNestedGenericParameterType());
		// 若是POJO，就是类名首字母小写（并不是形参名）
		String name = Conventions.getVariableNameForParameter(parameter);

		// 进行数据校验（之前已经详细分析过，此处一笔带过）
		if (binderFactory != null) {
			WebDataBinder binder = binderFactory.createBinder(webRequest, arg, name);
			if (arg != null) {
				validateIfApplicable(binder, parameter);
				if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
					throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
				}
			}

			// 把校验结果放进Model里，方便页面里获取
			if (mavContainer != null) {
				mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());
			}
		}

		// 适配：支持到Optional类型的参数
		return adaptArgumentIfNecessary(arg, parameter);
	}
}
```



##### HttpEntityMethodProcessor

用于处理`HttpEntity`和`RequestEntity`类型的入参的。

```java
public class HttpEntityMethodProcessor extends AbstractMessageConverterMethodProcessor {
	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		return (HttpEntity.class == parameter.getParameterType() || RequestEntity.class == parameter.getParameterType());
	}

	@Override
	@Nullable
	public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws IOException, HttpMediaTypeNotSupportedException {

		ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
		// 拿到HttpEntity的泛型类型
		Type paramType = getHttpEntityType(parameter);
		if (paramType == null) {
			// 注意：这个泛型类型是必须指定的，必须的
			throw new IllegalArgumentException("HttpEntity parameter '" + parameter.getParameterName() + "' in method " + parameter.getMethod() + " is not parameterized");
		}

		// 调用父类方法拿到body的值(把泛型类型传进去了，所以返回的是个实例)
		Object body = readWithMessageConverters(webRequest, parameter, paramType);
		// 注意步操作：new了一个RequestEntity进去，持有实例即可
		if (RequestEntity.class == parameter.getParameterType()) {
			return new RequestEntity<>(body, inputMessage.getHeaders(), inputMessage.getMethod(), inputMessage.getURI());
		} else { // 用的父类HttpEntity，那就会丢失掉Method等信息（因此建议入参用RequestEntity类型，更加强大些）
			return new HttpEntity<>(body, inputMessage.getHeaders());
		}
	}
}
```

@RequestBody/HttpEntity它的参数（泛型）类型允许是Map
方法上的和类上的@ResponseBody都可以被继承，但@RequestBody不可以
@RequestBody它自带有Bean Validation校验能力（当然需要启用），HttpEntity更加的轻量和方便



#### ErrorsMethodArgumentResolver

用于在方法参数可以写`Errors`类型，来拿到数据校验结果。

```java
public class ErrorsMethodArgumentResolver implements HandlerMethodArgumentResolver {
	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		Class<?> paramType = parameter.getParameterType();
		return Errors.class.isAssignableFrom(paramType);
	}

	@Override
	@Nullable
	public Object resolveArgument(MethodParameter parameter,
			@Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest,
			@Nullable WebDataBinderFactory binderFactory) throws Exception {

		Assert.state(mavContainer != null,
				"Errors/BindingResult argument only supported on regular handler methods");

		ModelMap model = mavContainer.getModel();
		String lastKey = CollectionUtils.lastElement(model.keySet());
		
		// 只有@RequestBody/@RequestPart注解的  这里面才会有值
		if (lastKey != null && lastKey.startsWith(BindingResult.MODEL_KEY_PREFIX)) {
			return model.get(lastKey);
		}

		// 简单的说：必须有@RequestBody/@RequestPart这注解标注，Errors参数才有意义
		throw new IllegalStateException(
				"An Errors/BindingResult argument is expected to be declared immediately after " +
				"the model attribute, the @RequestBody or the @RequestPart arguments " +
				"to which they apply: " + parameter.getMethod());
	}
}
```











































