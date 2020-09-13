springMVC:HandlerInterceptor拦截器的使用

SessionAttributesHandler

```java
// @since 3.1
public class SessionAttributesHandler {

	private final Set<String> attributeNames = new HashSet<>();
	private final Set<Class<?>> attributeTypes = new HashSet<>();

	// 注意这个重要性：它是注解方式放入session和API方式放入session的关键（它只会记录注解方式放进去的session属性~~）
	private final Set<String> knownAttributeNames = Collections.newSetFromMap(new ConcurrentHashMap<>(4));
	// sessonAttr存储器：它最终存储到的是WebRequest的session域里面去（对httpSession是进行了包装的）
	// 因为有WebRequest的处理，所以达到我们上面看到的效果。complete只会清楚注解放进去的，并不清除API放进去的~~~
	// 它的唯一实现类DefaultSessionAttributeStore实现也简单。（特点：能够制定特殊的前缀，这个有时候还是有用的）
	// 前缀attributeNamePrefix在构造器里传入进来  默认是“”
	private final SessionAttributeStore sessionAttributeStore;

	// 唯一的构造器 handlerType：控制器类型  SessionAttributeStore 是由调用者上层传进来的
	public SessionAttributesHandler(Class<?> handlerType, SessionAttributeStore sessionAttributeStore) {
		Assert.notNull(sessionAttributeStore, "SessionAttributeStore may not be null");
		this.sessionAttributeStore = sessionAttributeStore;

		// 父类上、接口上、注解上的注解标注了这个注解都算
		SessionAttributes ann = AnnotatedElementUtils.findMergedAnnotation(handlerType, SessionAttributes.class);
		if (ann != null) {
			Collections.addAll(this.attributeNames, ann.names());
			Collections.addAll(this.attributeTypes, ann.types());
		}
		this.knownAttributeNames.addAll(this.attributeNames);
	}

	// 既没有指定Name 也没有指定type  这个注解标上了也没啥用
	public boolean hasSessionAttributes() {
		return (!this.attributeNames.isEmpty() || !this.attributeTypes.isEmpty());
	}

	// 看看指定的attributeName或者type是否在包含里面
	// 请注意：name和type都是或者的关系，只要有一个符合条件就成
	public boolean isHandlerSessionAttribute(String attributeName, Class<?> attributeType) {
		Assert.notNull(attributeName, "Attribute name must not be null");
		if (this.attributeNames.contains(attributeName) || this.attributeTypes.contains(attributeType)) {
			this.knownAttributeNames.add(attributeName);
			return true;
		} else {
			return false;
		}
	}

	// 把attributes属性们存储起来  进到WebRequest 里
	public void storeAttributes(WebRequest request, Map<String, ?> attributes) {
		attributes.forEach((name, value) -> {
			if (value != null && isHandlerSessionAttribute(name, value.getClass())) {
				this.sessionAttributeStore.storeAttribute(request, name, value);
			}
		});
	}

	// 检索所有的属性们  用的是knownAttributeNames哦~~~~
	// 也就是说手动API放进Session的 此处不会被检索出来的
	public Map<String, Object> retrieveAttributes(WebRequest request) {
		Map<String, Object> attributes = new HashMap<>();
		for (String name : this.knownAttributeNames) {
			Object value = this.sessionAttributeStore.retrieveAttribute(request, name);
			if (value != null) {
				attributes.put(name, value);
			}
		}
		return attributes;
	}

	// 同样的 只会清除knownAttributeNames
	public void cleanupAttributes(WebRequest request) {
		for (String attributeName : this.knownAttributeNames) {
			this.sessionAttributeStore.cleanupAttribute(request, attributeName);
		}
	}


	// 对底层sessionAttributeStore的一个传递调用~~~~~
	// 毕竟可以拼比一下sessionAttributeStore的实现~~~~
	@Nullable
	Object retrieveAttribute(WebRequest request, String attributeName) {
		return this.sessionAttributeStore.retrieveAttribute(request, attributeName);
	}
}
```

#### ModelFactory#initModel()

- 处理器执行前，初始化`Model`
- 处理器执行后，将`Model`中相应的参数同步更新到`SessionAttributes`中（不是全量，而是符合条件的那些）

```java
// @since 3.1
public final class ModelFactory {
	// ModelMethod它是一个私有内部类，持有InvocableHandlerMethod的引用  和方法的dependencies依赖们
	private final List<ModelMethod> modelMethods = new ArrayList<>();
	private final WebDataBinderFactory dataBinderFactory;
	private final SessionAttributesHandler sessionAttributesHandler;

	public ModelFactory(@Nullable List<InvocableHandlerMethod> handlerMethods, WebDataBinderFactory binderFactory, SessionAttributesHandler attributeHandler) {
	
		// 把InvocableHandlerMethod转为内部类ModelMethod
		if (handlerMethods != null) {
			for (InvocableHandlerMethod handlerMethod : handlerMethods) {
				this.modelMethods.add(new ModelMethod(handlerMethod));
			}
		}
		this.dataBinderFactory = binderFactory;
		this.sessionAttributesHandler = attributeHandler;
	}


	// 该方法完成Model的初始化
	public void initModel(NativeWebRequest request, ModelAndViewContainer container, HandlerMethod handlerMethod) throws Exception {
		// 先拿到sessionAttr里所有的属性们（首次进来肯定木有，但同一个session第二次进来就有了）
		Map<String, ?> sessionAttributes = this.sessionAttributesHandler.retrieveAttributes(request);
		// 和当前请求中 已经有的model合并属性信息
		// 注意：sessionAttributes中只有当前model不存在的属性，它才会放进去
		container.mergeAttributes(sessionAttributes);
		// 此方法重要：调用模型属性方法来填充模型  这里ModelAttribute会生效
		// 关于@ModelAttribute的内容  我放到了这里：https://blog.csdn.net/f641385712/article/details/98260361
		// 总之：完成这步之后 Model就有值了~~~~
		invokeModelAttributeMethods(request, container);

		// 最后，最后，最后还做了这么一步操作~~~
		// findSessionAttributeArguments的作用：把@ModelAttribute的入参也列入SessionAttributes（非常重要） 详细见下文
		// 这里一定要掌握：因为使用中的坑坑经常是因为没有理解到这块逻辑
		for (String name : findSessionAttributeArguments(handlerMethod)) {
		
			// 若ModelAndViewContainer不包含此name的属性   才会进来继续处理  这一点也要注意
			if (!container.containsAttribute(name)) {

				// 去请求域里检索为name的属性，若请求域里没有（也就是sessionAttr里没有），此处会抛出异常的~~~~
				Object value = this.sessionAttributesHandler.retrieveAttribute(request, name);
				if (value == null) {
					throw new HttpSessionRequiredException("Expected session attribute '" + name + "'", name);
				}
				// 把从sessionAttr里检索到的属性也向容器Model内放置一份~
				container.addAttribute(name, value);
			}
		}
	}


	// 把@ModelAttribute标注的入参也列入SessionAttributes 放进sesson里（非常重要）
	// 这个动作是很多开发者都忽略了的
	private List<String> findSessionAttributeArguments(HandlerMethod handlerMethod) {
		List<String> result = new ArrayList<>();
		// 遍历所有的方法参数
		for (MethodParameter parameter : handlerMethod.getMethodParameters()) {
			// 只有参数里标注了@ModelAttribute的才会进入继续解析~~~
			if (parameter.hasParameterAnnotation(ModelAttribute.class)) {
				// 关于getNameForParameter拿到modelKey的方法，这个策略是需要知晓的
				String name = getNameForParameter(parameter);
				Class<?> paramType = parameter.getParameterType();

				// 判断isHandlerSessionAttribute为true的  才会把此name合法的添加进来
				// （也就是符合@SessionAttribute标注的key或者type的）
				if (this.sessionAttributesHandler.isHandlerSessionAttribute(name, paramType)) {
					result.add(name);
				}
			}
		}
		return result;
	}

	// 静态方法：决定了parameter的名字  它是public的，因为ModelAttributeMethodProcessor里也有使用
	// 请注意：这里不是MethodParameter.getParameterName()获取到的形参名字，而是有自己的一套规则的

	// @ModelAttribute指定了value值就以它为准，否则就是类名的首字母小写（当然不同类型不一样，下面有给范例）
	public static String getNameForParameter(MethodParameter parameter) {
		ModelAttribute ann = parameter.getParameterAnnotation(ModelAttribute.class);
		String name = (ann != null ? ann.value() : null);
		return (StringUtils.hasText(name) ? name : Conventions.getVariableNameForParameter(parameter));
	}

	// 关于方法这块的处理逻辑，和上差不多，主要是返回类型和实际类型的区分
	// 比如List<String>它对应的名是：stringList。即使你的返回类型是Object~~~
	public static String getNameForReturnValue(@Nullable Object returnValue, MethodParameter returnType) {
		ModelAttribute ann = returnType.getMethodAnnotation(ModelAttribute.class);
		if (ann != null && StringUtils.hasText(ann.value())) {
			return ann.value();
		} else {
			Method method = returnType.getMethod();
			Assert.state(method != null, "No handler method");
			Class<?> containingClass = returnType.getContainingClass();
			Class<?> resolvedType = GenericTypeResolver.resolveReturnType(method, containingClass);
			return Conventions.getVariableNameForReturnType(method, resolvedType, returnValue);
		}
	}

	// 将列为@SessionAttributes的模型数据，提升到sessionAttr里
	public void updateModel(NativeWebRequest request, ModelAndViewContainer container) throws Exception {
		ModelMap defaultModel = container.getDefaultModel();
		if (container.getSessionStatus().isComplete()){
			this.sessionAttributesHandler.cleanupAttributes(request);
		} else { // 存储到sessionAttr里
			this.sessionAttributesHandler.storeAttributes(request, defaultModel);
		}

		// 若该request还没有被处理  并且 Model就是默认defaultModel
		if (!container.isRequestHandled() && container.getModel() == defaultModel) {
			updateBindingResult(request, defaultModel);
		}
	}

	// 将bindingResult属性添加到需要该属性的模型中。
	// isBindingCandidate：给定属性在Model模型中是否需要bindingResult。
	private void updateBindingResult(NativeWebRequest request, ModelMap model) throws Exception {
		List<String> keyNames = new ArrayList<>(model.keySet());
		for (String name : keyNames) {
			Object value = model.get(name);
			if (value != null && isBindingCandidate(name, value)) {
				String bindingResultKey = BindingResult.MODEL_KEY_PREFIX + name;
				if (!model.containsAttribute(bindingResultKey)) {
					WebDataBinder dataBinder = this.dataBinderFactory.createBinder(request, value, name);
					model.put(bindingResultKey, dataBinder.getBindingResult());
				}
			}
		}
	}

	// 看看这个静态内部类ModelMethod
	private static class ModelMethod {
		// 持有可调用的InvocableHandlerMethod 这个方法
		private final InvocableHandlerMethod handlerMethod;
		// 这字段是搜集该方法标注了@ModelAttribute注解的入参们
		private final Set<String> dependencies = new HashSet<>();

		public ModelMethod(InvocableHandlerMethod handlerMethod) {
			this.handlerMethod = handlerMethod;
			// 把方法入参中所有标注了@ModelAttribute了的Name都搜集进来
			for (MethodParameter parameter : handlerMethod.getMethodParameters()) {
				if (parameter.hasParameterAnnotation(ModelAttribute.class)) {
					this.dependencies.add(getNameForParameter(parameter));
				}
			}
		}
		...
	}
}
```



SessionAttributeStore



### @ModelAttribute

将方法参数/方法返回值绑定到`web view`的`Model`里面。只支持`@RequestMapping`这种类型的控制器。它既可以标注在方法入参上，也可以标注在方法（返回值）上。

```java
// @since 2.5  只能用在入参、方法上
@Target({ElementType.PARAMETER, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface ModelAttribute {

	@AliasFor("name")
	String value() default "";
	// The name of the model attribute to bind to. 注入如下默认规则
	// 比如person对应的类是：mypackage.Person（类名首字母小写）
	// personList对应的是：List<Person>  这些都是默认规则咯~~~ 数组、Map的省略
	// 具体可以参考方法：Conventions.getVariableNameForParameter(parameter)的处理规则
	@AliasFor("value")
	String name() default "";

	// 若是false表示禁用数据绑定。
	// @since 4.3
	boolean binding() default true;
}
```

#### `ModelFactory`

```java
// @since 3.1
public final class ModelFactory {

	// 初始化Model 这个时候`@ModelAttribute`有很大作用
	public void initModel(NativeWebRequest request, ModelAndViewContainer container, HandlerMethod handlerMethod) throws Exception {
		// 拿到sessionAttr的属性
		Map<String, ?> sessionAttributes = this.sessionAttributesHandler.retrieveAttributes(request);
		// 合并进容器内
		container.mergeAttributes(sessionAttributes);
		// 这个方法就是调用执行标注有@ModelAttribute的方法们~~~~
		invokeModelAttributeMethods(request, container);
		... 
	}

	//调用标注有注解的方法来填充Model
	private void invokeModelAttributeMethods(NativeWebRequest request, ModelAndViewContainer container) throws Exception {
		// modelMethods是构造函数进来的  一个个的处理吧
		while (!this.modelMethods.isEmpty()) {
			// getNextModelMethod：通过next其实能看出 执行是有顺序的  拿到一个可执行的InvocableHandlerMethod
			InvocableHandlerMethod modelMethod = getNextModelMethod(container).getHandlerMethod();

			// 拿到方法级别的标注的@ModelAttribute~~
			ModelAttribute ann = modelMethod.getMethodAnnotation(ModelAttribute.class);
			Assert.state(ann != null, "No ModelAttribute annotation");
			if (container.containsAttribute(ann.name())) {
				if (!ann.binding()) { // 若binding是false  就禁用掉此name的属性  让不支持绑定了  此方法也处理完成
					container.setBindingDisabled(ann.name());
				}
				continue;
			}

			// 调用目标的handler方法，拿到返回值returnValue 
			Object returnValue = modelMethod.invokeForRequest(request, container);
			// 方法返回值不是void才需要继续处理
			if (!modelMethod.isVoid()){

				// returnValueName的生成规则 上文有解释过  本处略
				String returnValueName = getNameForReturnValue(returnValue, modelMethod.getReturnType());
				if (!ann.binding()) { // 同样的 若禁用了绑定，此处也不会放进容器里
					container.setBindingDisabled(returnValueName);
				}
		
				//在个判断是个小细节：只有容器内不存在此属性，才会放进去   因此并不会有覆盖的效果哦~~~
				// 所以若出现同名的  请自己控制好顺序吧
				if (!container.containsAttribute(returnValueName)) {
					container.addAttribute(returnValueName, returnValue);
				}
			}
		}
	}

	// 拿到下一个标注有此注解方法~~~
	private ModelMethod getNextModelMethod(ModelAndViewContainer container) {
		
		// 每次都会遍历所有的构造进来的modelMethods
		for (ModelMethod modelMethod : this.modelMethods) {
			// dependencies：表示该方法的所有入参中 标注有@ModelAttribute的入参们
			// checkDependencies的作用是：所有的dependencies依赖们必须都是container已经存在的属性，才会进到这里来
			if (modelMethod.checkDependencies(container)) {
				// 找到一个 就移除一个
				// 这里使用的是List的remove方法，不用担心并发修改异常？？？ 哈哈其实不用担心的  小伙伴能知道为什么吗？？
				this.modelMethods.remove(modelMethod);
				return modelMethod;
			}
		}

		// 若并不是所有的依赖属性Model里都有，那就拿第一个吧~~~~
		ModelMethod modelMethod = this.modelMethods.get(0);
		this.modelMethods.remove(modelMethod);
		return modelMethod;
	}
	...
}
```



#### `ModelAttributeMethodProcessor`

`HandlerMethodArgumentResolver` + `HandlerMethodReturnValueHandler`

```java
// 这个处理器用于处理入参、方法返回值~~~~
// @since 3.1
public class ModelAttributeMethodProcessor implements HandlerMethodArgumentResolver, HandlerMethodReturnValueHandler {

	private static final ParameterNameDiscoverer parameterNameDiscoverer = new DefaultParameterNameDiscoverer();
	private final boolean annotationNotRequired;

	public ModelAttributeMethodProcessor(boolean annotationNotRequired) {
		this.annotationNotRequired = annotationNotRequired;
	}


	// 入参里标注了@ModelAttribute 或者（注意这个或者） annotationNotRequired = true并且不是isSimpleProperty()
	// isSimpleProperty()：八大基本类型/包装类型、Enum、Number等等 Date Class等等等等
	// 所以划重点：即使你没标注@ModelAttribute  单子还要不是基本类型等类型，都会进入到这里来处理
	// 当然这个行为是是收到annotationNotRequired属性影响的，具体的具体而论  它既有false的时候  也有true的时候
	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		return (parameter.hasParameterAnnotation(ModelAttribute.class) ||
				(this.annotationNotRequired && !BeanUtils.isSimpleProperty(parameter.getParameterType())));
	}

	// 说明：能进入到这里来的  证明入参里肯定是有对应注解的？？？
	// 显然不是，上面有说  这事和属性值annotationNotRequired有关的~~~
	@Override
	@Nullable
	public final Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer, NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
	
		// 拿到ModelKey名称~~~（注解里有写就以注解的为准）
		String name = ModelFactory.getNameForParameter(parameter);
		// 拿到参数的注解本身
		ModelAttribute ann = parameter.getParameterAnnotation(ModelAttribute.class);
		if (ann != null) {
			mavContainer.setBinding(name, ann.binding());
		}

		Object attribute = null;
		BindingResult bindingResult = null;

		// 如果model里有这个属性，那就好说，直接拿出来完事~
		if (mavContainer.containsAttribute(name)) {
			attribute = mavContainer.getModel().get(name);
		} else { // 若不存在，也不能让是null呀
			// Create attribute instance
			// 这是一个复杂的创建逻辑：
			// 1、如果是空构造，直接new一个实例出来
			// 2、若不是空构造，支持@ConstructorProperties解析给构造赋值
			//   注意:这里就支持fieldDefaultPrefix前缀、fieldMarkerPrefix分隔符等能力了 最终完成获取一个属性
			// 调用BeanUtils.instantiateClass(ctor, args)来创建实例
			// 注意：但若是非空构造出来，是立马会执行valid校验的，此步骤若是空构造生成的实例，此步不会进行valid的，但是下一步会哦~
			try {
				attribute = createAttribute(name, parameter, binderFactory, webRequest);
			} catch (BindException ex) {
				if (isBindExceptionRequired(parameter)) {
					// No BindingResult parameter -> fail with BindException
					throw ex;
				}
				// Otherwise, expose null/empty value and associated BindingResult
				if (parameter.getParameterType() == Optional.class) {
					attribute = Optional.empty();
				}
				bindingResult = ex.getBindingResult();
			}
		}

		// 若是空构造创建出来的实例，这里会进行数据校验  此处使用到了((WebRequestDataBinder) binder).bind(request);  bind()方法  唯一一处
		if (bindingResult == null) {
			// Bean property binding and validation;
			// skipped in case of binding failure on construction.
			WebDataBinder binder = binderFactory.createBinder(webRequest, attribute, name);
			if (binder.getTarget() != null) {
				// 绑定request请求数据
				if (!mavContainer.isBindingDisabled(name)) {
					bindRequestParameters(binder, webRequest);
				}
				// 执行valid校验~~~~
				validateIfApplicable(binder, parameter);
				//注意：此处抛出的异常是BindException
				//RequestResponseBodyMethodProcessor抛出的异常是：MethodArgumentNotValidException
				if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
					throw new BindException(binder.getBindingResult());
				}
			}
			// Value type adaptation, also covering java.util.Optional
			if (!parameter.getParameterType().isInstance(attribute)) {
				attribute = binder.convertIfNecessary(binder.getTarget(), parameter.getParameterType(), parameter);
			}
			bindingResult = binder.getBindingResult();
		}

		// Add resolved attribute and BindingResult at the end of the model
		// at the end of the model  把解决好的属性放到Model的末尾~~~
		// 可以即使是标注在入参上的@ModelAtrribute的属性值，最终也都是会放进Model里的~~~可怕吧
		Map<String, Object> bindingResultModel = bindingResult.getModel();
		mavContainer.removeAttributes(bindingResultModel);
		mavContainer.addAllAttributes(bindingResultModel);

		return attribute;
	}

	// 此方法`ServletModelAttributeMethodProcessor`子类是有复写的哦~~~~
	// 使用了更强大的：ServletRequestDataBinder.bind(ServletRequest request)方法
	protected void bindRequestParameters(WebDataBinder binder, NativeWebRequest request) {
		((WebRequestDataBinder) binder).bind(request);
	}
}
```

模型属性首先从Model中获取，若没有获取到，就使用默认构造函数（可能是有无参，也可能是有参）创建，然后会把ServletRequest请求的数据绑定上来， 然后进行@Valid校验（若添加有校验注解的话），最后会把属性添加到Model里面

最后加进去的代码是：mavContainer.addAllAttributes(bindingResultModel);

```java
public class ModelAttributeMethodProcessor implements HandlerMethodArgumentResolver, HandlerMethodReturnValueHandler {

	// 方法返回值上标注有@ModelAttribute注解（或者非简单类型）  默认都会放进Model内哦~~
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		return (returnType.hasMethodAnnotation(ModelAttribute.class) ||
				(this.annotationNotRequired && !BeanUtils.isSimpleProperty(returnType.getParameterType())));
	}

	// 这个处理就非常非常的简单了，注意：null值是不放的哦~~~~
	// 注意：void的话  returnValue也是null
	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

		if (returnValue != null) {
			String name = ModelFactory.getNameForReturnValue(returnValue, returnType);
			mavContainer.addAttribute(name, returnValue);
		}
	}
}
```

### @RequestAttribute

从request中取对应的**属性值**。

##### RequestAttributeMethodArgumentResolver



# Spring WebMVC 扩展之扩展 HandlerInterceptor





## Spring Web 



> Struts 1.x 
>
> ActionMapping
>
> ActionForward



### 处理器（Handler）映射 - HandlerMapping

绝大多数实现类继承 AbstractHandlerMapping

包含多个有序 interceptors

- HandlerInterceptor（不带）
- MappedInterceptor（自带映射规则）

> MappedInterceptor 会被适配成 HandlerInterceptor



### 处理器（Handler）

- @RequestMapping 标注方法
- handleRequest 实现方法



### 处理器（Handler）执行链 - HandleExecutionChain

- 包含多个  HandlerMapping 以及一个 Handler 对象
- 委派集合执行
  - HandlerExecutionChain#applyPreHandle
  - HandlerExecutionChain#applyPostHandle
  - HandlerExecutionChain#triggerAfterCompletion



### 处理器（Handler）拦截器 - HandlerInterceptor



#### HandlerInterceptor 与 Filter 的区别？

Filter 是拦截 Servlet，一旦被拦截（中断），后续 Servlet 不会被执行

Filter 前置和后置处理需要开发人员自己去管理（不自带，手动增加）

Filter 拦截请求形式 - DispatcherType：

- Request
- Forward
- Include
- Error
- Async

Filter 映射关系和 Servlet 无关，**单独**部署

> Filter1 -> FIlter2 -> Filter3 -> Servlet
>
> FilterChain -> N  *  Filter + 1 Servlet



HandlerExecutionChain -> N * HandlerInterceptor  + 1 Handler

HandlerInterceptor  它的映射关系与 DispatcherServlet 有关系（子关系）



```xml
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
</servlet>

<servlet-mapping>
    <servlet-name>dispatcher</servlet-name>
    <url-pattern>/abc</url-pattern>
</servlet-mapping>

<filter>
    <filter-name>myFilter</servlet-name>
    <filter-class>com.acme.MyFilter</filter-class>
</filter>


<filter-mapping>
    <filter-name>dispatcher</filter-name>
    <url-pattern>/myfilter/</url-pattern>
</filter-mapping>
<!-- SCWCD ->
```



#### HandlerInterceptor 与 MappedInterceptor 的区别

MappedInterceptor  是存在自己的 URL 判断映射关系的，URL Pattern 属于特殊映射逻辑，默认 - AntPathMatcher，可以自定义 PathMatcher



#### HandlerInterceptor   与  WebRequestInterceptor 的区别

WebRequestInterceptor 是通用拦截器，被拦截的请求对象是 WebRequest

Web 三种规范

- Servlet
- Portlet
- JSF

>  Web Reactive : 



### 处理器（Handler）拦截器注册中心 - InterceptorRegistry

注册一到多个有序 HandlerInterceptor，它为 AbstractHandlerMapping 和 HandleExecutionChain 提供数据来源





## Spring Web MVC

velocity-spring-boot

### Handler 到底是什么？

- HandlerMethod
- Controller 实现类



Handler 来源于 `HandlerMapping#getHandler(HttpServletRequest)` 方法实现。



### 如何区分 REST 处理和视图渲染处理？

ModelAndView 是否为 null

### 视图渲染



### REST 处理



### AnnotatedHandlerMethodHandlerInterceptorAdapter





interface Annotation

rawtype 泛型擦写后的（不带泛型）类型

静态aop



