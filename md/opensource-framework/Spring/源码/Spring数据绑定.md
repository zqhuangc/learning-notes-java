# Spring中的数据绑定

## DataBinder（源码分析）

```
public class DataBinder implements PropertyEditorRegistry, TypeConverter {

	/** Default object name used for binding: "target". */
	public static final String DEFAULT_OBJECT_NAME = "target";
	/** Default limit for array and collection growing: 256. */
	public static final int DEFAULT_AUTO_GROW_COLLECTION_LIMIT = 256;

	@Nullable
	private final Object target;
	private final String objectName; // 默认值是target

	// BindingResult：绑定错误、失败的时候会放进这里来~
	@Nullable
	private AbstractPropertyBindingResult bindingResult;

	//类型转换器，会注册最为常用的那么多类型转换Map<Class<?>, PropertyEditor> defaultEditors
	@Nullable
	private SimpleTypeConverter typeConverter;

	// 默认忽略不能识别的字段~
	private boolean ignoreUnknownFields = true;
	// 不能忽略非法的字段（比如我要Integer，你给传aaa，那肯定就不让绑定了，抛错）
	private boolean ignoreInvalidFields = false;
	// 默认是支持级联的~~~
	private boolean autoGrowNestedPaths = true;

	private int autoGrowCollectionLimit = DEFAULT_AUTO_GROW_COLLECTION_LIMIT;

	// 这三个参数  都可以自己指定~~ 允许的字段、不允许的、必须的
	@Nullable
	private String[] allowedFields;
	@Nullable
	private String[] disallowedFields;
	@Nullable
	private String[] requiredFields;

	// 转换器ConversionService
	@Nullable
	private ConversionService conversionService;
	// 状态码处理器~
	@Nullable
	private MessageCodesResolver messageCodesResolver;
	// 绑定出现错误的处理器~
	private BindingErrorProcessor bindingErrorProcessor = new DefaultBindingErrorProcessor();
	// 校验器（这个非常重要）
	private final List<Validator> validators = new ArrayList<>();

	//  objectName没有指定，就用默认的
	public DataBinder(@Nullable Object target) {
		this(target, DEFAULT_OBJECT_NAME);
	}
	public DataBinder(@Nullable Object target, String objectName) {
		this.target = ObjectUtils.unwrapOptional(target);
		this.objectName = objectName;
	}
	... // 省略所有属性的get/set方法

	// 提供一些列的初始化方法，供给子类使用 或者外部使用  下面两个初始化方法是互斥的
	public void initBeanPropertyAccess() {
		Assert.state(this.bindingResult == null, "DataBinder is already initialized - call initBeanPropertyAccess before other configuration methods");
		this.bindingResult = createBeanPropertyBindingResult();
	}
	protected AbstractPropertyBindingResult createBeanPropertyBindingResult() {
		BeanPropertyBindingResult result = new BeanPropertyBindingResult(getTarget(), getObjectName(), isAutoGrowNestedPaths(), getAutoGrowCollectionLimit());
		if (this.conversionService != null) {
			result.initConversion(this.conversionService);
		}
		if (this.messageCodesResolver != null) {
			result.setMessageCodesResolver(this.messageCodesResolver);
		}
		return result;
	}
	// 你会发现，初始化DirectFieldAccess的时候，校验的也是bindingResult ~~~~
	public void initDirectFieldAccess() {
		Assert.state(this.bindingResult == null, "DataBinder is already initialized - call initDirectFieldAccess before other configuration methods");
		this.bindingResult = createDirectFieldBindingResult();
	}
	protected AbstractPropertyBindingResult createDirectFieldBindingResult() {
		DirectFieldBindingResult result = new DirectFieldBindingResult(getTarget(), getObjectName(), isAutoGrowNestedPaths());
		if (this.conversionService != null) {
			result.initConversion(this.conversionService);
		}
		if (this.messageCodesResolver != null) {
			result.setMessageCodesResolver(this.messageCodesResolver);
		}
		return result;
	}

	...
	// 把属性访问器返回，PropertyAccessor(默认直接从结果里拿)，子类MapDataBinder有复写
	protected ConfigurablePropertyAccessor getPropertyAccessor() {
		return getInternalBindingResult().getPropertyAccessor();
	}

	// 可以看到简单的转换器也是使用到了conversionService的，可见conversionService它的效用
	protected SimpleTypeConverter getSimpleTypeConverter() {
		if (this.typeConverter == null) {
			this.typeConverter = new SimpleTypeConverter();
			if (this.conversionService != null) {
				this.typeConverter.setConversionService(this.conversionService);
			}
		}
		return this.typeConverter;
	}

	... // 省略众多get方法
	
	// 设置指定的可以绑定的字段，默认是所有字段~~~
	// 例如，在绑定HTTP请求参数时，限制这一点以避免恶意用户进行不必要的修改。
	// 简单的说：我可以控制只有指定的一些属性才允许你修改~~~~
	// 注意：它支持xxx*,*xxx,*xxx*这样的通配符  支持[]这样子来写~
	public void setAllowedFields(@Nullable String... allowedFields) {
		this.allowedFields = PropertyAccessorUtils.canonicalPropertyNames(allowedFields);
	}
	public void setDisallowedFields(@Nullable String... disallowedFields) {
		this.disallowedFields = PropertyAccessorUtils.canonicalPropertyNames(disallowedFields);
	}

	// 注册每个绑定进程所必须的字段。
	public void setRequiredFields(@Nullable String... requiredFields) {
		this.requiredFields = PropertyAccessorUtils.canonicalPropertyNames(requiredFields);
		if (logger.isDebugEnabled()) {
			logger.debug("DataBinder requires binding of required fields [" + StringUtils.arrayToCommaDelimitedString(requiredFields) + "]");
		}
	}
	...
	// 注意：这个是set方法，后面是有add方法的~
	// 注意：虽然是set，但是引用是木有变的~~~~
	public void setValidator(@Nullable Validator validator) {
		// 判断逻辑在下面：你的validator至少得支持这种类型呀  哈哈
		assertValidators(validator);
		// 因为自己手动设置了，所以先清空  再加进来~~~
		// 这步你会发现，即使validator是null，也是会clear的哦~  符合语意
		this.validators.clear();
		if (validator != null) {
			this.validators.add(validator);
		}
	}
	private void assertValidators(Validator... validators) {
		Object target = getTarget();
		for (Validator validator : validators) {
			if (validator != null && (target != null && !validator.supports(target.getClass()))) {
				throw new IllegalStateException("Invalid target for Validator [" + validator + "]: " + target);
			}
		}
	}
	public void addValidators(Validator... validators) {
		assertValidators(validators);
		this.validators.addAll(Arrays.asList(validators));
	}
	// 效果同set
	public void replaceValidators(Validator... validators) {
		assertValidators(validators);
		this.validators.clear();
		this.validators.addAll(Arrays.asList(validators));
	}
	
	// 返回一个，也就是primary默认的校验器
	@Nullable
	public Validator getValidator() {
		return (!this.validators.isEmpty() ? this.validators.get(0) : null);
	}
	// 只读视图
	public List<Validator> getValidators() {
		return Collections.unmodifiableList(this.validators);
	}

	// since Spring 3.0
	public void setConversionService(@Nullable ConversionService conversionService) {
		Assert.state(this.conversionService == null, "DataBinder is already initialized with ConversionService");
		this.conversionService = conversionService;
		if (this.bindingResult != null && conversionService != null) {
			this.bindingResult.initConversion(conversionService);
		}
	}

	// =============下面它提供了非常多的addCustomFormatter()方法  注册进PropertyEditorRegistry里=====================
	public void addCustomFormatter(Formatter<?> formatter);
	public void addCustomFormatter(Formatter<?> formatter, String... fields);
	public void addCustomFormatter(Formatter<?> formatter, Class<?>... fieldTypes);

	// 实现接口方法
	public void registerCustomEditor(Class<?> requiredType, PropertyEditor propertyEditor);
	public void registerCustomEditor(@Nullable Class<?> requiredType, @Nullable String field, PropertyEditor propertyEditor);
	...
	// 实现接口方法
	// 统一委托给持有的TypeConverter~~或者是getInternalBindingResult().getPropertyAccessor();这里面的
	@Override
	@Nullable
	public <T> T convertIfNecessary(@Nullable Object value, @Nullable Class<T> requiredType,
			@Nullable MethodParameter methodParam) throws TypeMismatchException {

		return getTypeConverter().convertIfNecessary(value, requiredType, methodParam);
	}


	// ===========上面的方法都是开胃小菜，下面才是本类最重要的方法==============

	// 该方法就是把提供的属性值们，绑定到目标对象target里去~~~
	public void bind(PropertyValues pvs) {
		MutablePropertyValues mpvs = (pvs instanceof MutablePropertyValues ? (MutablePropertyValues) pvs : new MutablePropertyValues(pvs));
		doBind(mpvs);
	}
	// 此方法是protected的，子类WebDataBinder有复写~~~加强了一下
	protected void doBind(MutablePropertyValues mpvs) {
		// 前面两个check就不解释了，重点看看applyPropertyValues(mpvs)这个方法~
		checkAllowedFields(mpvs);
		checkRequiredFields(mpvs);
		applyPropertyValues(mpvs);
	}

	// allowe允许的 并且还是没有在disallowed里面的 这个字段就是被允许的
	protected boolean isAllowed(String field) {
		String[] allowed = getAllowedFields();
		String[] disallowed = getDisallowedFields();
		return ((ObjectUtils.isEmpty(allowed) || PatternMatchUtils.simpleMatch(allowed, field)) &&
				(ObjectUtils.isEmpty(disallowed) || !PatternMatchUtils.simpleMatch(disallowed, field)));
	}
	...
	// protected 方法，给target赋值~~~~
	protected void applyPropertyValues(MutablePropertyValues mpvs) {
		try {
			// 可以看到最终赋值 是委托给PropertyAccessor去完成的
			getPropertyAccessor().setPropertyValues(mpvs, isIgnoreUnknownFields(), isIgnoreInvalidFields());

		// 抛出异常，交给BindingErrorProcessor一个个处理~~~
		} catch (PropertyBatchUpdateException ex) {
			for (PropertyAccessException pae : ex.getPropertyAccessExceptions()) {
				getBindingErrorProcessor().processPropertyAccessException(pae, getInternalBindingResult());
			}
		}
	}

	// 执行校验，此处就和BindingResult 关联上了，校验失败的消息都会放进去（不是直接抛出异常哦~ ）
	public void validate() {
		Object target = getTarget();
		Assert.state(target != null, "No target to validate");
		BindingResult bindingResult = getBindingResult();
		// 每个Validator都会执行~~~~
		for (Validator validator : getValidators()) {
			validator.validate(target, bindingResult);
		}
	}

	// 带有校验提示的校验器。SmartValidator
	// @since 3.1
	public void validate(Object... validationHints) { ... }

	// 这一步也挺有意思：实际上就是若有错误，就抛出异常
	// 若没错误  就把绑定的Model返回~~~(可以看到BindingResult里也能拿到最终值哦~~~)
	// 此方法可以调用，但一般较少使用~
	public Map<?, ?> close() throws BindException {
		if (getBindingResult().hasErrors()) {
			throw new BindException(getBindingResult());
		}
		return getBindingResult().getModel();
	}
}
```



源码的分析中，大概能总结到DataBinder它提供了如下能力：

把属性值PropertyValues绑定到target上（bind()方法，依赖于PropertyAccessor实现~）
提供校验的能力：提供了public方法validate()对各个属性使用Validator执行校验~
提供了注册属性编辑器（PropertyEditor）和对类型进行转换的能力（TypeConverter）
还需要注意的是：

initBeanPropertyAccess和initDirectFieldAccess两个初始化PropertyAccessor方法是互斥的

1. initBeanPropertyAccess()创建的是BeanPropertyBindingResult，内部依赖BeanWrapper
2. initDirectFieldAccess创建的是DirectFieldBindingResult，内部依赖DirectFieldAccessor
   这两个方法内部没有显示的调用，但是Spring内部默认使用的是initBeanPropertyAccess()，具体可以参照getInternalBindingResult()方法~

总结
本文介绍了Spring用于数据绑定的最直接类DataBinder，它位于spring-context这个工程的org.springframework.validation包内，所以需要再次明确的是：它是Spring提供的能力而非web提供的



### WebDataBinder

从`web request`里（**注意：这里指的web请求，并不一定就是ServletRequest请求哟**）把web请求的`parameters`绑定到`JavaBean`上

`Controller`方法的参数类型可以是基本类型，也可以是封装后的普通Java类型。**若这个普通Java类型没有声明任何注解，则意味着它的每一个属性都需要到Request中去查找对应的请求参数。**

WebDataBinder在SpringMVC中使用，它不需要我们自己去创建，我们只需要向它注册参数类型对应的属性编辑器PropertyEditor。PropertyEditor可以将字符串转换成其真正的数据类型，它的void setAsText(String text)方法实现数据转换的过程。


```java
// @since 1.2
public class WebDataBinder extends DataBinder {

	// 此字段意思是：字段标记  比如name -> _name
	// 这对于HTML复选框和选择选项特别有用。
	public static final String DEFAULT_FIELD_MARKER_PREFIX = "_";
	// !符号是处理默认值的，提供一个默认值代替空值~~~
	public static final String DEFAULT_FIELD_DEFAULT_PREFIX = "!";
	
	@Nullable
	private String fieldMarkerPrefix = DEFAULT_FIELD_MARKER_PREFIX;
	@Nullable
	private String fieldDefaultPrefix = DEFAULT_FIELD_DEFAULT_PREFIX;
	// 默认也会绑定空的文件流~
	private boolean bindEmptyMultipartFiles = true;

	// 完全沿用父类的两个构造~~~
	public WebDataBinder(@Nullable Object target) {
		super(target);
	}
	public WebDataBinder(@Nullable Object target, String objectName) {
		super(target, objectName);
	}

	... //  省略get/set
	// 在父类的基础上，增加了对_和!的处理~~~
	@Override
	protected void doBind(MutablePropertyValues mpvs) {
		checkFieldDefaults(mpvs);
		checkFieldMarkers(mpvs);
		super.doBind(mpvs);
	}

	protected void checkFieldDefaults(MutablePropertyValues mpvs) {
		String fieldDefaultPrefix = getFieldDefaultPrefix();
		if (fieldDefaultPrefix != null) {
			PropertyValue[] pvArray = mpvs.getPropertyValues();
			for (PropertyValue pv : pvArray) {

				// 若你给定的PropertyValue的属性名确实是以!打头的  那就做处理如下：
				// 如果JavaBean的该属性可写 && mpvs不存在去掉!后的同名属性，那就添加进来表示后续可以使用了（毕竟是默认值，没有精确匹配的高的）
				// 然后把带!的给移除掉（因为默认值以已经转正了~~~）
				// 其实这里就是说你可以使用！来给个默认值。比如!name表示若找不到name这个属性的时，就取它的值~~~
				// 也就是说你request里若有穿!name保底，也就不怕出现null值啦~
				if (pv.getName().startsWith(fieldDefaultPrefix)) {
					String field = pv.getName().substring(fieldDefaultPrefix.length());
					if (getPropertyAccessor().isWritableProperty(field) && !mpvs.contains(field)) {
						mpvs.add(field, pv.getValue());
					}
					mpvs.removePropertyValue(pv);
				}
			}
		}
	}

	// 处理_的步骤
	// 若传入的字段以_打头
	// JavaBean的这个属性可写 && mpvs木有去掉_后的属性名字
	// getEmptyValue(field, fieldType)就是根据Type类型给定默认值。
	// 比如Boolean类型默认给false，数组给空数组[]，集合给空集合，Map给空map  可以参考此类：CollectionFactory
	// 当然，这一切都是建立在你传的属性值是以_打头的基础上的，Spring才会默认帮你处理这些默认值
	protected void checkFieldMarkers(MutablePropertyValues mpvs) {
		String fieldMarkerPrefix = getFieldMarkerPrefix();
		if (fieldMarkerPrefix != null) {
			PropertyValue[] pvArray = mpvs.getPropertyValues();
			for (PropertyValue pv : pvArray) {
				if (pv.getName().startsWith(fieldMarkerPrefix)) {
					String field = pv.getName().substring(fieldMarkerPrefix.length());
					if (getPropertyAccessor().isWritableProperty(field) && !mpvs.contains(field)) {
						Class<?> fieldType = getPropertyAccessor().getPropertyType(field);
						mpvs.add(field, getEmptyValue(field, fieldType));
					}
					mpvs.removePropertyValue(pv);
				}
			}
		}
	}

	// @since 5.0
	@Nullable
	public Object getEmptyValue(Class<?> fieldType) {
		try {
			if (boolean.class == fieldType || Boolean.class == fieldType) {
				// Special handling of boolean property.
				return Boolean.FALSE;
			} else if (fieldType.isArray()) {
				// Special handling of array property.
				return Array.newInstance(fieldType.getComponentType(), 0);
			} else if (Collection.class.isAssignableFrom(fieldType)) {
				return CollectionFactory.createCollection(fieldType, 0);
			} else if (Map.class.isAssignableFrom(fieldType)) {
				return CollectionFactory.createMap(fieldType, 0);
			}
		} catch (IllegalArgumentException ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Failed to create default value - falling back to null: " + ex.getMessage());
			}
		}
		// 若不在这几大类型内，就返回默认值null呗~~~
		// 但需要说明的是，若你是简单类型比如int，
		// Default value: null. 
		return null;
	}

	// 单独提供的方法，用于绑定org.springframework.web.multipart.MultipartFile类型的数据到JavaBean属性上~
	// 显然默认是允许MultipartFile作为Bean一个属性  参与绑定的
	// Map<String, List<MultipartFile>>它的key，一般来说就是文件们啦~
	protected void bindMultipart(Map<String, List<MultipartFile>> multipartFiles, MutablePropertyValues mpvs) {
		multipartFiles.forEach((key, values) -> {
			if (values.size() == 1) {
				MultipartFile value = values.get(0);
				if (isBindEmptyMultipartFiles() || !value.isEmpty()) {
					mpvs.add(key, value);
				}
			}
			else {
				mpvs.add(key, values);
			}
		});
	}
}
```

#### ServletRequestDataBinder

javax.servlet.ServletRequest

```java
public class ServletRequestDataBinder extends WebDataBinder {
	... // 沿用父类构造
	// 注意这个可不是父类的方法，是本类增强的~~~~意思就是kv都从request里来~~当然内部还是适配成了一个MutablePropertyValues
	public void bind(ServletRequest request) {
		// 内部最核心方法是它：WebUtils.getParametersStartingWith()  把request参数转换成一个Map
		// request.getParameterNames()
		MutablePropertyValues mpvs = new ServletRequestParameterPropertyValues(request);
		MultipartRequest multipartRequest = WebUtils.getNativeRequest(request, MultipartRequest.class);
	
		// 调用父类的bindMultipart方法，把MultipartFile都放进MutablePropertyValues里去~~~
		if (multipartRequest != null) {
			bindMultipart(multipartRequest.getMultiFileMap(), mpvs);
		}
		// 这个方法是本类流出来的一个扩展点~~~子类可以复写此方法自己往里继续添加
		// 比如ExtendedServletRequestDataBinder它就复写了这个方法，进行了增强（下面会说）  支持到了uriTemplateVariables的绑定
		addBindValues(mpvs, request);
		doBind(mpvs);
	}

	// 这个方法和父类的close方法类似，很少直接调用
	public void closeNoCatch() throws ServletRequestBindingException {
		if (getBindingResult().hasErrors()) {
			throw new ServletRequestBindingException("Errors binding onto object '" + getBindingResult().getObjectName() + "'", new BindException(getBindingResult()));
		}
	}
}
```

MockHttpServletRequest

#### ExtendedServletRequestDataBinder

用于把`URI template variables`参数添加进来用于绑定。

> AbstractUrlHandlerMapping.lookupHandler() --> 
>
> chain.addInterceptor(new UriTemplateVariablesHandlerInterceptor(uriTemplateVariables));  --> 
>
> preHandle()方法 -> 
>
> exposeUriTemplateVariables(this.uriTemplateVariables, request); ->
>
>  request.setAttribute(URI_TEMPLATE_VARIABLES_ATTRIBUTE, uriTemplateVariables);
>

```java
// @since 3.1
public class ExtendedServletRequestDataBinder extends ServletRequestDataBinder {
	... // 沿用父类构造

	//本类的唯一方法
	@Override
	@SuppressWarnings("unchecked")
	protected void addBindValues(MutablePropertyValues mpvs, ServletRequest request) {
		// 它的值是：HandlerMapping.class.getName() + ".uriTemplateVariables";
		String attr = HandlerMapping.URI_TEMPLATE_VARIABLES_ATTRIBUTE;

		// 注意：此处是attr，而不是parameter
		Map<String, String> uriVars = (Map<String, String>) request.getAttribute(attr);
		if (uriVars != null) {
			uriVars.forEach((name, value) -> {
				
				// 若已经存在确切的key了，不会覆盖~~~~
				if (mpvs.contains(name)) {
					if (logger.isWarnEnabled()) {
						logger.warn("Skipping URI variable '" + name + "' because request contains bind value with same name.");
					}
				} else {
					mpvs.addPropertyValue(name, value);
				}
			});
		}
	}
}
```

### WebExchangeDataBinder

### MapDataBinder

### WebRequestDataBinder

#### 如何注册自己的PropertyEditor来实现`自定义类型`数据绑定？

PropertyEditorRegistrySupport 

**Date类型**

> 处理步骤：
>
> 1. BeanWrapper调用setPropertyValue()给属性赋值，传入的value值都会交给convertForProperty()方法根据get方法的返回值类型进行转换~（比如此处为Date类型）
> 2. 委托给this.typeConverterDelegate.convertIfNecessary进行类型转换（比如此处为string->Date类型）
> 3. 先this.propertyEditorRegistry.findCustomEditor(requiredType, propertyName);找到一个合适的PropertyEditor（显然此处我们没有自定义Custom处理Date的PropertyEditor，返回null）
> 4. 回退到使用ConversionService，显然此处我们也没有设置，返回null
> 5. 回退到使用默认的editor = findDefaultEditor(requiredType);（注意：此处只根据类型去找了，因为上面说了默认不处理了Date，所以也是返回null）
> 6. 最终回退到Spring对Array、Collection、Map的默认值处理问题，最终若是String类型，都会调用BeanUtils.instantiateClass(strCtor, convertedValue)也就是有参构造进行初始化~~~(请注意这必须是String类型才有的权利)

### WebBindingInitializer

实现此接口重写initBinder方法注册的属性编辑器是**全局的**属性编辑器，对所有的Controller都有效。`WebBindingInitializer`为编码方式，`@InitBinder`为注解方式（当然注解方式还能控制到只对当前Controller有效，实现更细粒度的控制）

```java
// @since 2.5   Spring在初始化WebDataBinder时候的回调接口，给调用者自定义~
public interface WebBindingInitializer {

	// @since 5.0
	void initBinder(WebDataBinder binder);

	// @deprecated as of 5.0 in favor of {@link #initBinder(WebDataBinder)}
	@Deprecated
	default void initBinder(WebDataBinder binder, WebRequest request) {
		initBinder(binder);
	}

}
```

#### ConfigurableWebBindingInitializer

```java
public class ConfigurableWebBindingInitializer implements WebBindingInitializer {
	private boolean autoGrowNestedPaths = true;
	private boolean directFieldAccess = false; // 显然这里是false

	// 下面这些参数，不就是WebDataBinder那些可以配置的属性们吗？
	@Nullable
	private MessageCodesResolver messageCodesResolver;
	@Nullable
	private BindingErrorProcessor bindingErrorProcessor;
	@Nullable
	private Validator validator;
	@Nullable
	private ConversionService conversionService;
	// 此处使用的PropertyEditorRegistrar来管理的，最终都会被注册进PropertyEditorRegistry嘛
	@Nullable
	private PropertyEditorRegistrar[] propertyEditorRegistrars;

	... //  省略所有get/set
	
	// 它做的事无非就是把配置的值都放进去而已~~
	@Override
	public void initBinder(WebDataBinder binder) {
		binder.setAutoGrowNestedPaths(this.autoGrowNestedPaths);
		if (this.directFieldAccess) {
			binder.initDirectFieldAccess();
		}
		if (this.messageCodesResolver != null) {
			binder.setMessageCodesResolver(this.messageCodesResolver);
		}
		if (this.bindingErrorProcessor != null) {
			binder.setBindingErrorProcessor(this.bindingErrorProcessor);
		}
		// 可以看到对校验器这块  内部还是做了容错的
		if (this.validator != null && binder.getTarget() != null && this.validator.supports(binder.getTarget().getClass())) {
			binder.setValidator(this.validator);
		}
		if (this.conversionService != null) {
			binder.setConversionService(this.conversionService);
		}
		if (this.propertyEditorRegistrars != null) {
			for (PropertyEditorRegistrar propertyEditorRegistrar : this.propertyEditorRegistrars) {
				propertyEditorRegistrar.registerCustomEditors(binder);
			}
		}
	}
}
```

### WebDataBinderFactory

```java
// @since 3.1   注意：WebDataBinder 可是1.2就有了~
public interface WebDataBinderFactory {
	// 此处使用的是Spring自己的NativeWebRequest   后面两个参数就不解释了
	WebDataBinder createBinder(NativeWebRequest webRequest, @Nullable Object target, String objectName) throws Exception;
}
```

#### DefaultDataBinderFactory

```java
public class DefaultDataBinderFactory implements WebDataBinderFactory {
	@Nullable
	private final WebBindingInitializer initializer;
	// 注意：这是唯一构造函数
	public DefaultDataBinderFactory(@Nullable WebBindingInitializer initializer) {
		this.initializer = initializer;
	}

	// 实现接口的方法
	@Override
	@SuppressWarnings("deprecation")
	public final WebDataBinder createBinder(NativeWebRequest webRequest, @Nullable Object target, String objectName) throws Exception {

		WebDataBinder dataBinder = createBinderInstance(target, objectName, webRequest);
		
		// 可见WebDataBinder 创建好后，此处就会回调（只有一个）
        // WebBindingInitializer initializer在此处解析完成了 全局生效
		if (this.initializer != null) {
			this.initializer.initBinder(dataBinder, webRequest);
		}
		// 解析@InitBinder注解，它是个protected空方法，交给子类复写实现
		// InitBinderDataBinderFactory对它有复写
		initBinder(dataBinder, webRequest);
		return dataBinder;
	}

	//  子类可以复写，默认实现是WebRequestDataBinder
	// 比如子类ServletRequestDataBinderFactory就复写了，使用的new ExtendedServletRequestDataBinder(target, objectName)
	protected WebDataBinder createBinderInstance(@Nullable Object target, String objectName, NativeWebRequest webRequest) throws Exception 
		return new WebRequestDataBinder(target, objectName);
	}
}
```



##### InitBinderDataBinderFactory

```java
// @since 3.1
public class InitBinderDataBinderFactory extends DefaultDataBinderFactory {
	
	// 需要注意的是：`@InitBinder`可以标注N多个方法~  所以此处是List
	private final List<InvocableHandlerMethod> binderMethods;

	// 此子类的唯一构造函数
	public InitBinderDataBinderFactory(@Nullable List<InvocableHandlerMethod> binderMethods, @Nullable WebBindingInitializer initializer) {
		super(initializer);
		this.binderMethods = (binderMethods != null ? binderMethods : Collections.emptyList());
	}

	// 上面知道此方法的调用方法生initializer.initBinder之后
	// 所以使用注解它生效的时机是在直接实现接口的后面的~
	@Override
	public void initBinder(WebDataBinder dataBinder, NativeWebRequest request) throws Exception {
		for (InvocableHandlerMethod binderMethod : this.binderMethods) {
			// 判断@InitBinder是否对dataBinder持有的target对象生效~~~（根据name来匹配的）
			if (isBinderMethodApplicable(binderMethod, dataBinder)) {
				// invokeForRequest 和 调用普通控制器方法一样
				// 方法入参上也可以写格式各样的参数
				Object returnValue = binderMethod.invokeForRequest(request, null, dataBinder);

				// 标注@InitBinder的方法不能有返回值
				if (returnValue != null) {
					throw new IllegalStateException("@InitBinder methods must not return a value (should be void): " + binderMethod);
				}
			}
		}
	}

	//@InitBinder有个Value值，它是个数组。它是用来匹配dataBinder.getObjectName()是否匹配的   若匹配上了，现在此注解方法就会生效
    // 让@InitBinder注解只作用在指定的入参名字的数据绑定上
	// 若value为空，那就对所有生效~~~
	protected boolean isBinderMethodApplicable(HandlerMethod initBinderMethod, WebDataBinder dataBinder) {
		InitBinder ann = initBinderMethod.getMethodAnnotation(InitBinder.class);
		Assert.state(ann != null, "No InitBinder annotation");
		String[] names = ann.value();
		return (ObjectUtils.isEmpty(names) || ObjectUtils.containsElement(names, dataBinder.getObjectName()));
	}
}
```



###### ServletRequestDataBinderFactory

```java
// @since 3.1
public class ServletRequestDataBinderFactory extends InitBinderDataBinderFactory {
	public ServletRequestDataBinderFactory(@Nullable List<InvocableHandlerMethod> binderMethods, @Nullable WebBindingInitializer initializer) {
		super(binderMethods, initializer);
	}
	@Override
	protected ServletRequestDataBinder createBinderInstance(
			@Nullable Object target, String objectName, NativeWebRequest request) throws Exception  {
		return new ExtendedServletRequestDataBinder(target, objectName);
	}
}
```



### `@InitBinder`

#### AbstractNamedValueMethodArgumentResolver

RequestParamMethodArgumentResolver

```java
// RequestParamMethodArgumentResolver的父类就是它，resolveArgument方法在父类上
// 子类仅仅只需要实现抽象方法resolveName，即：从request里根据name拿值
AbstractNamedValueMethodArgumentResolver：

	@Override
	@Nullable
	public final Object resolveArgument( ... ) {
		...
		Object arg = resolveName(resolvedName.toString(), nestedParameter, webRequest);
		...
		if (binderFactory != null) {
			// 创建出一个WebDataBinder
			WebDataBinder binder = binderFactory.createBinder(webRequest, null, namedValueInfo.name);
			// 完成数据转换（比如String转Date、String转...等等）
			arg = binder.convertIfNecessary(arg, parameter.getParameterType(), parameter);
			...
		}
		...
		return arg;
	}
```



## 属性访问器 PropertyAccessor

注意此接口和属性解析器(`PropertyResolver`)是有本质区别的：属性解析器是用来获取**配置数据**的

**属性访问器PropertyAccessor**接口的作用是`存/取`Bean对象的属性。为了体现这个接口它的重要性，

- **所有Spring创建的Bean对象都使用该接口存取Bean属性值**（个人理解）

以访问命名属性`named properties`（例如对象的bean属性或对象中的字段）的类的公共接口。BeanWrapper`接口也继承自它，它所在包是`org.springframework.beans（`BeanWrapper`也在此包）

最终的实现类主要有`DirectFieldAccessor`和`BeanWrapperImpl`

> `DirectFieldAccessFallbackBeanWrapper`它在`spring-data-commons`这个jar里面

```java
// @since 1.1  出现得非常早
public interface PropertyAccessor {

	// 简单的说就是级联属性的分隔符。
	// 比如foo.bar最终会调用getFoo().getBar()两个方法
	String NESTED_PROPERTY_SEPARATOR = ".";
	char NESTED_PROPERTY_SEPARATOR_CHAR = '.';

	// 代表角标index的符号  如person.addresses[0]  这样就可以把值放进集合/数组/Map里了
	String PROPERTY_KEY_PREFIX = "[";
	char PROPERTY_KEY_PREFIX_CHAR = '[';
	String PROPERTY_KEY_SUFFIX = "]";
	char PROPERTY_KEY_SUFFIX_CHAR = ']';


	// 此属性是否可读。若属性不存在  返回false
	boolean isReadableProperty(String propertyName);
	// 此出行是否可写。若属性不存在，返回false
	boolean isWritableProperty(String propertyName);
	

	// 读方法
	@Nullable
	Class<?> getPropertyType(String propertyName) throws BeansException;
	@Nullable
	TypeDescriptor getPropertyTypeDescriptor(String propertyName) throws BeansException;
	@Nullable
	Object getPropertyValue(String propertyName) throws BeansException;

	// 写方法  
	void setPropertyValue(String propertyName, @Nullable Object value) throws BeansException;
	void setPropertyValue(PropertyValue pv) throws BeansException;
	// 批量设置值
	void setPropertyValues(Map<?, ?> map) throws BeansException;
	// 说明：PropertyValues和PropertyValue关系特别像PropertySources和PropertySource的关系
	void setPropertyValues(PropertyValues pvs) throws BeansException;

	// 可控制是否接受非法的字段、value值扽  ignoreUnknown/ignoreInvalid分别对应非法属性和非法value值的处理策略~
	void setPropertyValues(PropertyValues pvs, boolean ignoreUnknown) throws BeansException;
	void setPropertyValues(PropertyValues pvs, boolean ignoreUnknown, boolean ignoreInvalid) throws BeansException;
}
```

### ConfigurablePropertyAccessor

可配置的`PropertyAccessor`。它是一个子接口，提供了可配置的能力，并且它还继承了`PropertyEditorRegistry`、`TypeConverter`等接口

> 抽象实现：`AbstractPropertyAccessor`

```java
// @since 2.0
public interface ConfigurablePropertyAccessor extends PropertyAccessor, PropertyEditorRegistry, TypeConverter {

	// 设置一个ConversionService ，用于对value值进行转换
	// 它是Spring3.0后推出来替代属性编辑器PropertyEditors的方案~
	void setConversionService(@Nullable ConversionService conversionService);
	@Nullable
	ConversionService getConversionService();

	// 设置在将属性编辑器应用于属性的新值时是**否提取旧属性值**。
	void setExtractOldValueForEditor(boolean extractOldValueForEditor);
	boolean isExtractOldValueForEditor();

	// 设置此实例是否应尝试“自动增长”包含null的嵌套路径。
	// true：为null的值会自动被填充为一个默认的value值，而不是抛出异常NullValueInNestedPathException
	void setAutoGrowNestedPaths(boolean autoGrowNestedPaths);
	boolean isAutoGrowNestedPaths();
}
```

#### `AbstractPropertyAccessor`

```java
// @since 2.0  它继承自TypeConverterSupport 相当于实现了TypeConverter以及PropertyEditorRegistry的所有内容
public abstract class AbstractPropertyAccessor extends TypeConverterSupport implements ConfigurablePropertyAccessor {

	// 这两个属性上面已经解释了~~~
	private boolean extractOldValueForEditor = false;
	private boolean autoGrowNestedPaths = false;	

	... // 省略get/set方法
	// setPropertyValue是抽象方法~~~
	@Override
	public void setPropertyValue(PropertyValue pv) throws BeansException {
		setPropertyValue(pv.getName(), pv.getValue());
	}


	@Override
	public void setPropertyValues(Map<?, ?> map) throws BeansException {
		setPropertyValues(new MutablePropertyValues(map));
	}
	// MutablePropertyValues和MutablePropertySources特别像，此处就不再介绍了
	// 此方法把Map最终包装成了一个MutablePropertyValues，它还有个web子类：ServletRequestParameterPropertyValues
	@Override
	public void setPropertyValues(Map<?, ?> map) throws BeansException {
		setPropertyValues(new MutablePropertyValues(map));
	}
	// 当然也可以直接传入一个PropertyValues   这里传入fasle，表示默认要求属性和value值必须都合法否则抛出异常
	@Override
	public void setPropertyValues(PropertyValues pvs) throws BeansException {
		setPropertyValues(pvs, false, false);
	}
	@Override
	public void setPropertyValues(PropertyValues pvs, boolean ignoreUnknown) throws BeansException {
		setPropertyValues(pvs, ignoreUnknown, false);
	}

	// 此抽象类最重要的实现方法~~~
	@Override
	public void setPropertyValues(PropertyValues pvs, boolean ignoreUnknown, boolean ignoreInvalid) throws BeansException {
		List<PropertyAccessException> propertyAccessExceptions = null;
		// 显然绝大多数情况下，都是MutablePropertyValues~~~~ 直接拿即可
		List<PropertyValue> propertyValues = (pvs instanceof MutablePropertyValues ? ((MutablePropertyValues) pvs).getPropertyValueList() : Arrays.asList(pvs.getPropertyValues()));

		// 遍历一个一个执行，批量设置值最终也还是调用的单个的~~~~
		// 这里面是否要抛出异常，ignoreUnknown和ignoreInvalid就生效了。分别对应NotWritablePropertyException和NullValueInNestedPathException两个异常
		for (PropertyValue pv : propertyValues) {
			try {
				setPropertyValue(pv);
			} catch (NotWritablePropertyException ex) {
				if (!ignoreUnknown) {
					throw ex;
				}
				// Otherwise, just ignore it and continue...
			} catch (NullValueInNestedPathException ex) {
				if (!ignoreInvalid) {
					throw ex;
				}
				// Otherwise, just ignore it and continue...
			} catch (PropertyAccessException ex) {
				if (propertyAccessExceptions == null) {
					propertyAccessExceptions = new ArrayList<>();
				}
				// 把异常收集，因为是for循环，最终一次性抛出
				propertyAccessExceptions.add(ex);
			}
		}

		// If we encountered individual exceptions, throw the composite exception.
		if (propertyAccessExceptions != null) {
			PropertyAccessException[] paeArray = propertyAccessExceptions.toArray(new PropertyAccessException[0]);
			throw new PropertyBatchUpdateException(paeArray);
		}
	}

	// 子类AbstractNestablePropertyAccessor重写了此方法
	// Redefined with public visibility.
	@Override
	@Nullable
	public Class<?> getPropertyType(String propertyPath) {
		return null;
	}


	// 抽象方法  相当于具体的get/set方法由子类去实现的~~
	@Override
	@Nullable
	public abstract Object getPropertyValue(String propertyName) throws BeansException;
	@Override
	public abstract void setPropertyValue(String propertyName, @Nullable Object value) throws BeansException;
}
```

主要完成了对`PropertyEditorRegistry`和`TypeConverter`等接口的间接实现，然后完成了批量操作的模版操作，但是很明显最终的落地的get/set留给子类来实现

**getPropertyValue和setPropertyValue是分别用于获取和设置bean的属性值的**

#### `AbstractNestablePropertyAccessor`

一个典型的实现，为其它所有使用案例提供必要的基础设施。`nestable：可嵌套的，支持嵌套的`

此访问器将**集合和数组值**转换为相应的目标**集合或数组**，解决了级联属性（嵌套属性）的问题

> `AbstractNestablePropertyAccessor`这个抽象类在Spring4.2后才提供

```java
// @since 4.2
public abstract class AbstractNestablePropertyAccessor extends AbstractPropertyAccessor {
	
	private int autoGrowCollectionLimit = Integer.MAX_VALUE;
	@Nullable
	Object wrappedObject;
	private String nestedPath = "";
	@Nullable
	Object rootObject;
	/** Map with cached nested Accessors: nested path -> Accessor instance. */
	@Nullable
	private Map<String, AbstractNestablePropertyAccessor> nestedPropertyAccessors;

	// 默认是注册默认的属性编辑器的：defaultEditors  它几乎处理了所有的Java内置类型  包括基本类型、包装类型以及对应数组类型~~~
	protected AbstractNestablePropertyAccessor() {
		this(true);
	}
	protected AbstractNestablePropertyAccessor(boolean registerDefaultEditors) {
		if (registerDefaultEditors) {
			registerDefaultEditors();
		}
		this.typeConverterDelegate = new TypeConverterDelegate(this);
	}
	protected AbstractNestablePropertyAccessor(Object object) {
		registerDefaultEditors();
		setWrappedInstance(object);
	}
	protected AbstractNestablePropertyAccessor(Class<?> clazz) {
		registerDefaultEditors();
		// 传的Clazz 那就会反射先创建一个实例对象
		setWrappedInstance(BeanUtils.instantiateClass(clazz));
	}
	protected AbstractNestablePropertyAccessor(Object object, String nestedPath, Object rootObject) {
		registerDefaultEditors();
		setWrappedInstance(object, nestedPath, rootObject);
	}
	//  parent:不能为null
	protected AbstractNestablePropertyAccessor(Object object, String nestedPath, AbstractNestablePropertyAccessor parent) {
		setWrappedInstance(object, nestedPath, parent.getWrappedInstance());
		setExtractOldValueForEditor(parent.isExtractOldValueForEditor());
		setAutoGrowNestedPaths(parent.isAutoGrowNestedPaths());
		setAutoGrowCollectionLimit(parent.getAutoGrowCollectionLimit());
		setConversionService(parent.getConversionService());
	}

	// wrappedObject：目标对象
	public void setWrappedInstance(Object object, @Nullable String nestedPath, @Nullable Object rootObject) {
		this.wrappedObject = ObjectUtils.unwrapOptional(object);
		Assert.notNull(this.wrappedObject, "Target object must not be null");
		this.nestedPath = (nestedPath != null ? nestedPath : "");
		// 此处根对象，若nestedPath存在的话，是可以自定义一个rootObject的~~~
		this.rootObject = (!this.nestedPath.isEmpty() ? rootObject : this.wrappedObject);
		this.nestedPropertyAccessors = null;
		this.typeConverterDelegate = new TypeConverterDelegate(this, this.wrappedObject);
	}

	public final Object getWrappedInstance() {
		Assert.state(this.wrappedObject != null, "No wrapped object");
		return this.wrappedObject;
	}
	public final String getNestedPath() {
		return this.nestedPath;
	}
	// 显然rootObject和NestedPath相关，默认它就是wrappedObject
	public final Object getRootInstance() {
		Assert.state(this.rootObject != null, "No root object");
		return this.rootObject;
	}
	
	... // 简单的说，它会处理.逻辑以及[0]等逻辑  [0]对应着集合和数组都可
}
```



#### 实现类 DirectFieldAccessor

继承自`AbstractNestablePropertyAccessor`

功能是直接操作Bean的属性值，而代替使用get/set方法去操作Bean。它的实现原理就是简单的field.get(getWrappedInstance())和field.set(getWrappedInstance(), value)等。
它处理级联属性的大致步骤是：

1. 遇上级联属性，先找出canonicalName
2. 根据此canonicalName调用其field.get()拿到此字段的值~
3. 若不为null（有初始值），那就继续解析此类型，循而往复

> 1. 若是级联属性、集合数组等复杂属性，**初始值不能为null**
> 2. 使用它给属性赋值无序提供get、set方法（**侧面意思是：它不会走你的get/set方法逻辑**）
>
> 当然若你希望null值能够被自动初始化也是可以的，请设值：`accessor.setAutoGrowNestedPaths(true);`这样数组、集合、Map等都会为null时候给你初始化(其它Bean请保证有默认构造函数)

```java
// @since 2.0   出现得可比父类`AbstractNestablePropertyAccessor`要早哦~~~注意：父类的构造函数都是protected的
public class DirectFieldAccessor extends AbstractNestablePropertyAccessor {

	// 缓存着每个字段的处理器FieldPropertyHandler
	// ReflectionUtils.findField()根据String去找到Field对象的
	private final Map<String, FieldPropertyHandler> fieldMap = new HashMap<>();

	public DirectFieldAccessor(Object object) {
		super(object);
	}
	// 这个构造器也是protected 的  所以若你想自己指定nestedPath和parent，你可以继承此类~~~
	protected DirectFieldAccessor(Object object, String nestedPath, DirectFieldAccessor parent) {
		super(object, nestedPath, parent);
	}
	...

	// 实现父类的抽象方法，依旧使用DirectFieldAccessor去处理~~~
	@Override
	protected DirectFieldAccessor newNestedPropertyAccessor(Object object, String nestedPath) {
		return new DirectFieldAccessor(object, nestedPath, this);
	}

	// 字段field属性处理器，使用内部类实现PropertyHandler ~~~
	private class FieldPropertyHandler extends PropertyHandler {
		private final Field field;
		// 从此处可以看出`DirectFieldAccessor`里的field默认都是可读、可写的~~~~
		public FieldPropertyHandler(Field field) {
			super(field.getType(), true, true);
			this.field = field;
		}
		...
	}
}
```

##### PropertyHandler 



#### PropertyValue的作用什么？

当设置属性值时，少不了两样东西：

1. 属性访问表达式：如`listMap[0][0]`
2. 属性值：

`ProperyValue`对象就是用来封装这些信息的。如果某个值要给赋值给bean属性，Spring都会把这个值包装成`ProperyValue`对象。

PropertyTokenHolder的作用是什么？
这个类的作用是对属性访问表达式的细化和归类。比如这句代码：

> .setPropertyValue("listMap[0][0]", "listMapValue00"); 
>
> 这句代码的含义是：为Apple的成员变量listMap的第0个元素：即为Map。然后向该Map里放入键值对：0(key)和listMapValue00(value)。所以listMap[0][0]一个属性访问表达式，它在PropertyTokenHolder类里存储如下：

canonicalName:listMap[0][0]：代表整个属性访问表达式
actualName:listMap：仅包含最外层的属性名称
keys:[0, 0]：数组的长度代表索引深度，各元素代表索引值
由于每个部分各有各的作用，所以就事先分解好，包装成对象，避免重复分解

### PropertyAccessorFactory

`Spring2.5`后提供的快速获取`PropertyAccessor`两个重要实现类的工厂。

```java
public final class PropertyAccessorFactory {
	private PropertyAccessorFactory() {
	}
	// 生产一个BeanWrapperImpl（最为常用）
	public static BeanWrapper forBeanPropertyAccess(Object target) {
		return new BeanWrapperImpl(target);
	}
	// 生产一个DirectFieldAccessor
	public static ConfigurablePropertyAccessor forDirectFieldAccess(Object target) {
		return new DirectFieldAccessor(target);
	}

}
```



## BeanWrapper

方便开发人员**使用字符串**来对`Java Bean`的属性执行get、set操作的工具。关于它的数据转换使用了如下两种机制：

* PropertyEditor：隶属于Java Bean规范。PropertyEditor只提供了String <-> Object的转换。
* ConversionService：Spring自3.0之后提供的替代PropertyEditor的机制（BeanWrapper在Spring的第一个版本就存在了~）

> 按照Spring官方文档的说法，当容器内没有注册`ConversionService`的时候，会退回使用`PropertyEditor`机制。言外之意：首选方案是`ConversionService`

Spring低级JavaBeans基础设施的中央接口。通常来说并不直接使用`BeanWrapper`，而是借助`BeanFactory`或者`DataBinder`来一起使用

`BeanWrapper`相当于一个代理器，Spring委托`BeanWrapper`完成Bean属性的填充工作

```java
//@since 13 April 2001  很清晰的看到，它也是个`PropertyAccessor`属性访问器
public interface BeanWrapper extends ConfigurablePropertyAccessor {

	// @since 4.1
	void setAutoGrowCollectionLimit(int autoGrowCollectionLimit);
	int getAutoGrowCollectionLimit();


	Object getWrappedInstance();
	Class<?> getWrappedClass();

	// 获取属性们的PropertyDescriptor  获取属性们
	PropertyDescriptor[] getPropertyDescriptors();
	// 获取具体某一个属性~
	PropertyDescriptor getPropertyDescriptor(String propertyName) throws InvalidPropertyException;
}
```



### BeanWrapperImpl

作为`BeanWrapper`接口的默认实现，它足以满足所有的典型应用场景，它会缓存Bean的内省结果而提高效率

> 在Spring2.5之前，此实现类是非public的，但在2.5之后改为 public 并且还提供了工厂：`PropertyAccessorFactory`帮助第三方框架能快速获取到一个实例

BeanWrapperImpl的能力：

* Bean包裹器
* 属性访问器（PropertyAccessor）
* 属性编辑器注册表（PropertyEditorRegistry）

从源码中继续分析还能再得出如下两个结论：

* 给属性赋值调用的是Method方法，如readMethod.invoke和writeMethod.invoke
* 对Bean的操作，大都委托给CachedIntrospectionResults去完成



* 因此若想了解，必然主要是要先了解java.beans.PropertyDescriptor和org.springframework.beans.CachedIntrospectionResults
* Java内省


```java
public class BeanWrapperImpl extends AbstractNestablePropertyAccessor implements BeanWrapper {

	// 缓存内省结果~
	@Nullable
	private CachedIntrospectionResults cachedIntrospectionResults;
	// The security context used for invoking the property methods.
	@Nullable
	private AccessControlContext acc;

	// 构造方法都是沿用父类的~
	public BeanWrapperImpl() {
		this(true);
	}
	... 
	private BeanWrapperImpl(Object object, String nestedPath, BeanWrapperImpl parent) {
		super(object, nestedPath, parent);
		setSecurityContext(parent.acc);
	}


	// @since 4.3  设置目标对象~~~
	public void setBeanInstance(Object object) {
		this.wrappedObject = object;
		this.rootObject = object;
		this.typeConverterDelegate = new TypeConverterDelegate(this, this.wrappedObject);
		// 设置内省的clazz
		setIntrospectionClass(object.getClass());
	}

	// 复写父类的方法  增加内省逻辑
	@Override
	public void setWrappedInstance(Object object, @Nullable String nestedPath, @Nullable Object rootObject) {
		super.setWrappedInstance(object, nestedPath, rootObject);
		setIntrospectionClass(getWrappedClass());
	}

	// 如果cachedIntrospectionResults它持有的BeanClass并不是传入的clazz 那就清空缓存 重新来~~~
	protected void setIntrospectionClass(Class<?> clazz) {
		if (this.cachedIntrospectionResults != null && this.cachedIntrospectionResults.getBeanClass() != clazz) {
			this.cachedIntrospectionResults = null;
		}
	}
	private CachedIntrospectionResults getCachedIntrospectionResults() {
		if (this.cachedIntrospectionResults == null) {
			// forClass此方法：生成此clazz的类型结果，并且缓存了起来~~
			this.cachedIntrospectionResults = CachedIntrospectionResults.forClass(getWrappedClass());
		}
		return this.cachedIntrospectionResults;
	}
	...

	// 获取到此属性的处理器。此处是个BeanPropertyHandler 内部类~
	@Override
	@Nullable
	protected BeanPropertyHandler getLocalPropertyHandler(String propertyName) {
		PropertyDescriptor pd = getCachedIntrospectionResults().getPropertyDescriptor(propertyName);
		return (pd != null ? new BeanPropertyHandler(pd) : null);
	}
	@Override
	protected BeanWrapperImpl newNestedPropertyAccessor(Object object, String nestedPath) {
		return new BeanWrapperImpl(object, nestedPath, this);
	}
	@Override
	public PropertyDescriptor[] getPropertyDescriptors() {
		return getCachedIntrospectionResults().getPropertyDescriptors();
	}

	// 获取具体某一个属性的PropertyDescriptor 
	@Override
	public PropertyDescriptor getPropertyDescriptor(String propertyName) throws InvalidPropertyException {
		BeanWrapperImpl nestedBw = (BeanWrapperImpl) getPropertyAccessorForPropertyPath(propertyName);
		String finalPath = getFinalPath(nestedBw, propertyName);
		PropertyDescriptor pd = nestedBw.getCachedIntrospectionResults().getPropertyDescriptor(finalPath);
		if (pd == null) {
			throw new InvalidPropertyException(getRootClass(), getNestedPath() + propertyName, "No property '" + propertyName + "' found");
		}
		return pd;
	}
	...

	// 此处理器处理的是PropertyDescriptor 
	private class BeanPropertyHandler extends PropertyHandler {
		private final PropertyDescriptor pd;

		// 是否可读、可写  都是由PropertyDescriptor 去决定了~
		// java.beans.PropertyDescriptor~~
		public BeanPropertyHandler(PropertyDescriptor pd) {
			super(pd.getPropertyType(), pd.getReadMethod() != null, pd.getWriteMethod() != null);
			this.pd = pd;
		}
		...
		@Override
		@Nullable
		public Object getValue() throws Exception {
			...
			ReflectionUtils.makeAccessible(readMethod);
			return readMethod.invoke(getWrappedInstance(), (Object[]) null);
		}
		...
	}
}
```



## Java内省`Introspector`

### JavaBean

**JavaBean的概念**：一种特殊的类，主要用于传递数据信息。这种类中的方法主要用于访问私有的字段，且方法名**符合某种命名规则**。如果在两个模块之间传递信息，可以将信息封装进JavaBean中，这种对象称为“值对象”(`Value Object`)，或“`VO`”。

JavaBean都有如下几个特征：

* 属性都是私有的；
* 有无参的public构造方法；
* 对私有属性根据需要提供公有的getXxx方法以及setXxx方法；
* getters必须有返回值没有方法参数；setter值没有返回值，有方法参数；

符合这些特征的类，被称为JavaBean；JDK中提供了一套API用来访问某个属性的getter/setter方法，这些API存放在java.beans中，这就是内省(Introspector)。

**内省和反射的区别**

* 反射：Java反射机制是在运行中，对任意一个类，能够获取得到这个类的所有属性和方法；它针对的是任意类
* 内省（Introspector）：是Java语言对JavaBean类属性、事件的处理方法

反射可以操作各种类的属性，而内省只是通过反射来操作JavaBean的属性
内省设置属性值肯定会调用setter方法，反射可以不用（反射可直接操作属性Field）
反射就像照镜子，然后能看到.class的所有，是客观的事实。内省更像主观的判断：比如看到getName()内省就会认为这个类中有name字段，但事实上并不一定会有name；通过内省可以获取bean的getter/setter

### Introspector

内省的API主要有Introspector、BeanInfo、PropertyDescriptor等

> `getMethodDescriptors()`它把父类的`MethodDescriptor`也拿出来了。
> 而`PropertyDescriptor`中比较特殊的是因为有`getClass()`方法，因此class也算是一个`PropertyDescriptor`，但是它没有`writeMethod`

关于`BeanInfo`，Spring在3.1提供了一个类`ExtendedBeanInfo`继承自它实现了功能扩展，并且提供了`BeanInfoFactory`来专门生产它~~~（实现类为：`ExtendedBeanInfoFactory`）

### MethodIntrospector

```java
// @since 4.2.3
// 定义彻底搜索元数据关联方法的算法，包括接口和父类，同时还处理参数化方法，以及接口和基于类的代理遇到的常见场景
// 通常，但不一定，用于查找带注释的处理程序方法~~~~~~~~~~~~~~
// 也就是说倘若你的注解啥的，在接口上也是可以的
public abstract class MethodIntrospector {

	// 核心方法：就是从指定的targetType里找到合适的方法。metadataLookup：用于过滤
	public static <T> Map<Method, T> selectMethods(Class<?> targetType, final MetadataLookup<T> metadataLookup) {
		final Map<Method, T> methodMap = new LinkedHashMap<>();
		Set<Class<?>> handlerTypes = new LinkedHashSet<>();
		Class<?> specificHandlerType = null;
		
		// 如果该类型 不是JDK动态代理类型
		if (!Proxy.isProxyClass(targetType)) {
			specificHandlerType = ClassUtils.getUserClass(targetType);
			handlerTypes.add(specificHandlerType);
		}
		// 拿到该目标类型所有的实现的接口们~~~~~~~
		handlerTypes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetType));

		// 遍历所有的需要处理的handlerTypes  然后一个个的处理
		for (Class<?> currentHandlerType : handlerTypes) {
			// 如果找到了specificHandlerType ，那就用它，否则就是currentHandlerType
			final Class<?> targetClass = (specificHandlerType != null ? specificHandlerType : currentHandlerType);

		 	// 使用的是ReflectionUtils.USER_DECLARED_METHODS 方法过滤器。
			ReflectionUtils.doWithMethods(currentHandlerType, method -> {
				Method specificMethod = ClassUtils.getMostSpecificMethod(method, targetClass);
				// 对这个specificMethod 还会做进一步的处理~~~~
				T result = metadataLookup.inspect(specificMethod);
				if (result != null) {
					Method bridgedMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);
					//==================================
					if (bridgedMethod == specificMethod || metadataLookup.inspect(bridgedMethod) == null) {
						methodMap.put(specificMethod, result);
					}
				}
			}, ReflectionUtils.USER_DECLARED_METHODS);
		}

		return methodMap;
	}

	public static Set<Method> selectMethods(Class<?> targetType, final ReflectionUtils.MethodFilter methodFilter) {
		return selectMethods(targetType,
				(MetadataLookup<Boolean>) method -> (methodFilter.matches(method) ? Boolean.TRUE : null)).keySet();
	}

	public static Method selectInvocableMethod(Method method, Class<?> targetType) {
		if (method.getDeclaringClass().isAssignableFrom(targetType)) {
			return method;
		}
		try {
			String methodName = method.getName();
			Class<?>[] parameterTypes = method.getParameterTypes();
			for (Class<?> ifc : targetType.getInterfaces()) {
				try {
					return ifc.getMethod(methodName, parameterTypes);
				}
				catch (NoSuchMethodException ex) {
					// Alright, not on this interface then...
				}
			}
			// A final desperate attempt on the proxy class itself...
			return targetType.getMethod(methodName, parameterTypes);
		}
		catch (NoSuchMethodException ex) {
			throw new IllegalStateException(String.format(
					"Need to invoke method '%s' declared on target class '%s', " +
					"but not found in any interface(s) of the exposed proxy type. " +
					"Either pull the method up to an interface or switch to CGLIB " +
					"proxies by enforcing proxy-target-class mode in your configuration.",
					method.getName(), method.getDeclaringClass().getSimpleName()));
		}
	}

	@FunctionalInterface
	public interface MetadataLookup<T> {
		@Nullable
		T inspect(Method method);
	}

}
```



#### PropertyDescriptor 属性描述器

属性描述符描述了Java bean通过**一对访问器方法导出**的一个属性。

主要方法描述如下：

* getPropertyType()，获得属性的Class对象；
* getReadMethod()，获得用于读取属性值的方法；
* getWriteMethod()，获得用于写入属性值的方法；
* setReadMethod(Method readMethod)，设置用于读取属性值的方法；
* setWriteMethod(Method writeMethod)，设置用于写入属性值的方法。

#### CachedIntrospectionResults

Spring如果需要依赖注入那么就必须依靠Java内省这个特性了，说到Spring IOC与JDK内省的结合那么就不得不说一下Spring中的CachedIntrospectionResults这个类了。
它是Spring提供的专门用于缓存JavaBean的PropertyDescriptor描述信息的类，不能被应用代码直接使用。

它的缓存信息是被静态存储起来的（应用级别），因此对于同一个类型的被操作的 JavaBean并不会都创建一个新的CachedIntrospectionResults，因此，这个类使用了工厂模式，使用私有构造器和一个静态的forClass工厂方法来获取实例。

```java
public final class CachedIntrospectionResults {
	
	// 它可以通过在spring.properties里设置这个属性，来关闭内省的缓存~~~
	public static final String IGNORE_BEANINFO_PROPERTY_NAME = "spring.beaninfo.ignore";
	private static final boolean shouldIntrospectorIgnoreBeaninfoClasses = SpringProperties.getFlag(IGNORE_BEANINFO_PROPERTY_NAME);

	// 此处使用了SpringFactoriesLoader这个SPI来加载BeanInfoFactory,唯一实现类是ExtendedBeanInfoFactory
	/** Stores the BeanInfoFactory instances. */
	private static List<BeanInfoFactory> beanInfoFactories = SpringFactoriesLoader.loadFactories(
			BeanInfoFactory.class, CachedIntrospectionResults.class.getClassLoader());

	static final Set<ClassLoader> acceptedClassLoaders = Collections.newSetFromMap(new ConcurrentHashMap<>(16));
	static final ConcurrentMap<Class<?>, CachedIntrospectionResults> strongClassCache = new ConcurrentHashMap<>(64);
	static final ConcurrentMap<Class<?>, CachedIntrospectionResults> softClassCache = new ConcurrentReferenceHashMap<>(64);

	// 被包裹类的BeanInfo~~~也就是目标类
	private final BeanInfo beanInfo;
	// 它缓存了被包裹类的所有属性的属性描述器PropertyDescriptor。
	private final Map<String, PropertyDescriptor> propertyDescriptorCache;



	... // 其它的都是静态方法
	// 只有它会返回一个实例，此类是单例的设计~  它保证了每个beanClass都有一个CachedIntrospectionResults 对象，然后被缓存起来~
	static CachedIntrospectionResults forClass(Class<?> beanClass) throws BeansException { ... }
}
```

本处理类的核心内容是Java内省getBeanInfo()以及PropertyDescriptor~注意：为了使此内省缓存生效，有个前提条件请保证了：

* 确保将Spring框架的Jar包和你的应用类使用的是同一个ClassLoader加载的，这样在任何情况下会允许随着应用的生命周期来清楚缓存。

因此对于web应用来说，Spring建议给web容器注册一个IntrospectorCleanupListener监听器来防止多ClassLoader布局，这样也可以有效的利用caching从而提高效率

> 说明：请保证此监听器配置在第一个位置，比ContextLoaderListener还靠前~ 此监听器能有效的防止内存泄漏问题（因为内省的缓存是应用级别的全局缓存，很容易造成泄漏的）
> 其实流行框架比如struts, Quartz等在使用JDK的内省时，存在没有释的内存泄漏问题
>

#### `DirectFieldAccessFallbackBeanWrapper`

BeanWrapperImpl的子类 DirectFieldAccessFallbackBeanWrapper，如 BeanWrapperImpl 和 DirectFieldAccessor 的结合体。它先用 BeanWrapperImpl.getPropertyValue()，若抛出异常了，再用DirectFieldAccessor，此子类在 JedisClusterConnection 有被使用到过


<https://blog.csdn.net/zhuqiuhui/article/details/82391851>























