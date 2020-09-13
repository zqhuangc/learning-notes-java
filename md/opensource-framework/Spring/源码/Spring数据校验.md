Spring Validation（Spring 扩展）

`Bean Validation`内的校验类大都是线程安全的，包括校验器`javax.validation.Validator`也是线程安全的

1. **3.0提供了Bean级别的校验**
2. **3.1提供了更加强大的方法级别的校验**

## BeanValidationPostProcessor（类）

去校验Spring容器中的Bean，从而决定允不允许它初始化完成。若校验不通过，在违反约束的情况下就会抛出异常，阻止容器的正常启动，对**所有的**Bean在初始化`前/后`进行校验。

> `BeanValidationPostProcessor`默认可是没有被装配进容器的

```java
public class BeanValidationPostProcessor implements BeanPostProcessor, InitializingBean {
	// 这就是我们熟悉的校验器
	// 请注意这里是javax.validation.Validator，而不是org.springframework.validation.Validator
	@Nullable
	private Validator validator;
	// true：表示在Bean初始化之后完成校验
	// false：表示在Bean初始化之前就校验
	private boolean afterInitialization = false;
	... // 省略get/set

	// 由此可见使用的是默认的校验器（当然还是Hibernate的）
	@Override
	public void afterPropertiesSet() {
		if (this.validator == null) {
			this.validator = Validation.buildDefaultValidatorFactory().getValidator();
		}
	}

	// 这个实现太简单了~~~
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if (!this.afterInitialization) {
			doValidate(bean);
		}
		return bean;
	}
	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		if (this.afterInitialization) {
			doValidate(bean);
		}
		return bean;
	}

	protected void doValidate(Object bean) {
		Assert.state(this.validator != null, "No Validator set");
		Object objectToValidate = AopProxyUtils.getSingletonTarget(bean);
		if (objectToValidate == null) {
			objectToValidate = bean;
		}
		Set<ConstraintViolation<Object>> result = this.validator.validate(objectToValidate);

		// 拼接错误消息最终抛出
		if (!result.isEmpty()) {
			StringBuilder sb = new StringBuilder("Bean state is invalid: ");
			for (Iterator<ConstraintViolation<Object>> it = result.iterator(); it.hasNext();) {
				ConstraintViolation<Object> violation = it.next();
				sb.append(violation.getPropertyPath()).append(" - ").append(violation.getMessage());
				if (it.hasNext()) {
					sb.append("; ");
				}
			}
			throw new BeanInitializationException(sb.toString());
		}
	}
}
```



### org.springframework.validation.Validator

**这个接口完全脱离了任何基础设施或上下文，它支持应用于程序内的任何层**

```java
// 注意：它可不是Spring3后才推出的  最初就有
public interface Validator {
	// 此clazz是否可以被validate
	boolean supports(Class<?> clazz);
	// 执行校验，错误消息放在Errors 装着
	// 可以参考ValidationUtils这个工具类，它能帮助你很多
	void validate(Object target, Errors errors);
}
```



#### SmartValidator

扩展增加了校验**分组**：hints。

```java
// @since 3.1  这个出现得比较晚
public interface SmartValidator extends Validator {
	
	// 注意：这里的Hints最终都会被转化到JSR的分组里去~~
	// 所以这个可变参数，传接口Class对象即可~
	void validate(Object target, Errors errors, Object... validationHints);

	// @since 5.1  简单的说，这个方法子类请复写 否则不能使用
	default void validateValue(Class<?> targetType, String fieldName, @Nullable Object value, Errors errors, Object... validationHints) {
		throw new IllegalArgumentException("Cannot validate individual value for " + targetType);
	}
}
```



##### `SpringValidatorAdapter`：验证适配器（重要）

`javax.validation.Validator`到Spring的`Validator`的适配，通过它可以**对接到JSR的校验器**来完成校验工作

```java
// @since 3.0
public class SpringValidatorAdapter implements SmartValidator, javax.validation.Validator {

	// 通用的三个约束注解都需要有的属性
	private static final Set<String> internalAnnotationAttributes = new HashSet<>(4);
	static {
		internalAnnotationAttributes.add("message");
		internalAnnotationAttributes.add("groups");
		internalAnnotationAttributes.add("payload");
	}

	// 最终都是委托给它来完成校验的
	@Nullable
	private javax.validation.Validator targetValidator;
    
	public SpringValidatorAdapter(javax.validation.Validator targetValidator) {
		Assert.notNull(targetValidator, "Target Validator must not be null");
		this.targetValidator = targetValidator;
	}

	// 简单的说：默认支持校验所有的Bean类型
	@Override
	public boolean supports(Class<?> clazz) {
		return (this.targetValidator != null);
	}
    
	// processConstraintViolations做的事一句话解释：
	// 把ConstraintViolations错误消息，全都适配放在Errors（BindingResult）里面存储着
	@Override
	public void validate(Object target, Errors errors) {
		if (this.targetValidator != null) {
			processConstraintViolations(this.targetValidator.validate(target), errors);
		}
	}

	@Override
	public void validate(Object target, Errors errors, Object... validationHints) {
		if (this.targetValidator != null) {
			processConstraintViolations(this.targetValidator.validate(target,  asValidationGroups(validationHints)), errors);
		}
	}

	@SuppressWarnings("unchecked")
	@Override
	public void validateValue(Class<?> targetType, String fieldName, @Nullable Object value, Errors errors, Object... validationHints) {
		if (this.targetValidator != null) {
			processConstraintViolations(this.targetValidator.validateValue(
					(Class) targetType, fieldName, value, asValidationGroups(validationHints)), errors);
		}
	}

	// 把validationHints都转换为group （支识别Class类型哦）
	private Class<?>[] asValidationGroups(Object... validationHints) {
		Set<Class<?>> groups = new LinkedHashSet<>(4);
		for (Object hint : validationHints) {
			if (hint instanceof Class) {
				groups.add((Class<?>) hint);
			}
		}
		return ClassUtils.toClassArray(groups);
	}

	// 关于Implementation of JSR-303 Validator interface  省略...
}
```

##### CustomValidatorBean

可配置（Custom）的Bean类，也同样的实现了`双接口`。

##### `LocalValidatorFactoryBean`：基石

Local `ValidatorFactory` Bean

`MethodValidationPostProcessor`依赖于它来提供验证器

```java
// @since 3.0  这个类非常的丰富  实现了接口javax.validation.ValidatorFactory
// 实现了ApplicationContextAware拿到Spring上下文...
// 但其实，它的实际工作都是委托式，自己只提供了各式各样的配置~~~（主要是配置JSR）
public class LocalValidatorFactoryBean extends SpringValidatorAdapter implements ValidatorFactory, ApplicationContextAware, InitializingBean, DisposableBean {
	private Class providerClass;
	private ValidationProviderResolver validationProviderResolver;
	private MessageInterpolator messageInterpolator;
	private TraversableResolver traversableResolver;
	private ConstraintValidatorFactory constraintValidatorFactory;
	private ParameterNameDiscoverer parameterNameDiscoverer = new DefaultParameterNameDiscoverer();
	private Resource[] mappingLocations;
	private final Map<String, String> validationPropertyMap = new HashMap<>();
	private ApplicationContext applicationContext;
	private ValidatorFactory validatorFactory;

	... // 省略所有的get/set
	... // 省略afterPropertiesSet()进行的默认配置初始化  最终调用setTargetValidator(this.validatorFactory.getValidator());

	// validator校验器可从上下文拿的，这里是从工厂拿的
	// 省略所有对ValidatorFactory接口的方法实现
}
```

#### OptionalValidatorFactoryBean

#### SpringConstraintValidatorFactory

```java
public class SpringConstraintValidatorFactory implements ConstraintValidatorFactory {

	private final AutowireCapableBeanFactory beanFactory;
	public SpringConstraintValidatorFactory(AutowireCapableBeanFactory beanFactory) {
		Assert.notNull(beanFactory, "BeanFactory must not be null");
		this.beanFactory = beanFactory;
	}

	// 注意：此处是直接调用了create方法，放进容器
	@Override
	public <T extends ConstraintValidator<?, ?>> T getInstance(Class<T> key) {
		return this.beanFactory.createBean(key);
	}
	// Bean Validation 1.1 releaseInstance method
	public void releaseInstance(ConstraintValidator<?, ?> instance) {
		this.beanFactory.destroyBean(instance);
	}

}
```



#### `MessageSourceResourceBundleLocator`

扩展了`Hibernate`包的`ResourceBundleLocator`国际化，而使用
Spring自己的国际化资源：`org.springframework.context.MessageSource`

##### MessageSourceResourceBundle

```java
public class MessageSourceResourceBundleLocator implements ResourceBundleLocator {

	private final MessageSource messageSource;
	public MessageSourceResourceBundleLocator(MessageSource messageSource) {
		Assert.notNull(messageSource, "MessageSource must not be null");
		this.messageSource = messageSource;
	}

	@Override
	public ResourceBundle getResourceBundle(Locale locale) {
		return new MessageSourceResourceBundle(this.messageSource, locale);
	}

}


//@since 27.02.2003 java.util.ResourceBundle  它是JDK提供来读取国际化的属性配置文件的  是个抽象类
public class MessageSourceResourceBundle extends ResourceBundle {
	private final MessageSource messageSource;
	private final Locale locale;

	public MessageSourceResourceBundle(MessageSource source, Locale locale) {
		Assert.notNull(source, "MessageSource must not be null");
		this.messageSource = source;
		this.locale = locale;
	}
	public MessageSourceResourceBundle(MessageSource source, Locale locale, ResourceBundle parent) {
		this(source, locale);
		setParent(parent);
	}

	@Override
	@Nullable
	protected Object handleGetObject(String key) {
		try {
			return this.messageSource.getMessage(key, null, this.locale);
		} catch (NoSuchMessageException ex) {
			return null;
		}
	}

	// @since 1.6
	@Override
	public boolean containsKey(String key) {
		try {
			this.messageSource.getMessage(key, null, this.locale);
			return true;
		}
		catch (NoSuchMessageException ex) {
			return false;
		}
	}
	@Override
	public Enumeration<String> getKeys() {
		throw new UnsupportedOperationException("MessageSourceResourceBundle does not support enumerating its keys");
	}
	@Override
	public Locale getLocale() {
		return this.locale;
	}
}
```



#### LocaleContextMessageInterpolator

```java
public abstract class AbstractMessageInterpolator implements MessageInterpolator {
	...
	private String interpolateMessage(String message, Context context, Locale locale) throws MessageDescriptorFormatException {
		// 如果message消息木有占位符，那就直接返回  不再处理了~
		// 这里自定义的优先级是最高的~~~
		if ( message.indexOf( '{' ) < 0 ) {
			return replaceEscapedLiterals( message );
		}

		// 调用resolveMessage方法处理message中的占位符和el表达式
		if ( cachingEnabled ) {
			resolvedMessage = resolvedMessages.computeIfAbsent( new LocalizedMessage( message, locale ), lm -> resolveMessage( message, locale ) );
		} else {
			resolvedMessage = resolveMessage( message, locale );
		}	
		...
	}

	private String resolveMessage(String message, Locale locale) {
		String resolvedMessage = message;

		// 获取资源ResourceBundle三部曲
		ResourceBundle userResourceBundle = userResourceBundleLocator.getResourceBundle( locale );
		ResourceBundle constraintContributorResourceBundle = contributorResourceBundleLocator.getResourceBundle( locale );
		ResourceBundle defaultResourceBundle = defaultResourceBundleLocator.getResourceBundle( locale );
		...
	}
}
```

> 若没占位符符号{需要处理，直接返回（比如我们自定义message属性值全是文字，就直接返回了）~
> 有占位符或者EL，交给resolveMessage()方法从资源文件里拿内容来处理~
> 拿取资源文件，按照如下三个步骤寻找：
> 1. userResourceBundleLocator：去用户自己的classpath里面去找资源文件（默认名字是ValidationMessages.properties，当然你也可以使用国际化名）
> 2. contributorResourceBundleLocator：加载贡献的资源包
> 3. defaultResourceBundle：默认的策略。去这里于/org/hibernate/validator加载ValidationMessages.properties
>   需要注意的是，如上是加载资源的顺序。无论怎么样，这三处的资源文件都会加载进内存的（并无短路逻辑）。进行占位符匹配的时候，依旧遵守这规律：
> 4. 最先用自己当前项目classpath下的资源去匹配资源占位符，若没匹配上再用下一级别的资源~~~
> 5. 规律同上，依次类推，递归的匹配所有的占位符（若占位符没匹配上，原样输出，并不是输出null哦~）

**MessageCodesResolver**

```java
public interface MessageCodesResolver {
	String[] resolveMessageCodes(String errorCode, String objectName);
	String[] resolveMessageCodes(String errorCode, String objectName, String field, @Nullable Class<?> fieldType);
}
```

#### 自定义约束

1. 自定义一个约束注解
2. 实现一个校验器(实现接口：`ConstraintValidator`)
3. 定义默认的校验错误信息



## MethodValidationPostProcessor（方法）

官方说明：方法里写有JSR校验注解要想其生效的话，**要求类型级别上必须使用@Validated标注（还能指定验证的Group）**

> AbstractBeanFactoryAwareAdvisingPostProcessor
>
> AsyncAnnotationBeanPostProcessor

```java
// @since 3.1
public class MethodValidationPostProcessor extends AbstractBeanFactoryAwareAdvisingPostProcessor implements InitializingBean {
	// 备注：此处你标注@Valid是无用的~~~Spring可不提供识别
	// 当然你也可以自定义注解（下面提供了set方法~~~）
	// 但是注意：若自定义注解的话，此注解只决定了是否要代理，并不能指定分组（别乱搞）
	private Class<? extends Annotation> validatedAnnotationType = Validated.class;
	// 这个是javax.validation.Validator
	@Nullable
	private Validator validator;

	// 可以自定义生效的注解
	public void setValidatedAnnotationType(Class<? extends Annotation> validatedAnnotationType) {
		Assert.notNull(validatedAnnotationType, "'validatedAnnotationType' must not be null");
		this.validatedAnnotationType = validatedAnnotationType;
	}

	// 注意：你可以自己传入一个Validator，并且可以是定制化的LocalValidatorFactoryBean(推荐)
	public void setValidator(Validator validator) {
		// 建议传入LocalValidatorFactoryBean功能强大，从它里面生成一个验证器出来靠谱
		if (validator instanceof LocalValidatorFactoryBean) {
			this.validator = ((LocalValidatorFactoryBean) validator).getValidator();
		} else if (validator instanceof SpringValidatorAdapter) {
			this.validator = validator.unwrap(Validator.class);
		} else {
			this.validator = validator;
		}
	}
	// 当然，你也可以简单粗暴的直接提供一个ValidatorFactory即可~
	public void setValidatorFactory(ValidatorFactory validatorFactory) {
		this.validator = validatorFactory.getValidator();
	}


	// Pointcut 使用 AnnotationMatchingPointcut，并且支持内部类
	// 说明：@Aysnc使用的也是 AnnotationMatchingPointcut，只不过因为它支持标注在类上和方法上，所以最终是组合的 ComposablePointcut
	
	// 至于Advice通知，此处是 MethodValidationInterceptor
	@Override
	public void afterPropertiesSet() {
		Pointcut pointcut = new AnnotationMatchingPointcut(this.validatedAnnotationType, true);
		this.advisor = new DefaultPointcutAdvisor(pointcut, createMethodValidationAdvice(this.validator));
	}
	
	// 这个advice 就是给 @Validated 的类进行增强的  
    // 说明：子类可以覆盖
	// @since 4.2
	protected Advice createMethodValidationAdvice(@Nullable Validator validator) {
		return (validator != null ? new MethodValidationInterceptor(validator) : new MethodValidationInterceptor());
	}
}
```



### `MethodValidationInterceptor#invoke`

方法调用时进行检验，参数以及返回值

> 注意理解方法级别：方法级别的入参有可能是各种平铺的参数、也可能是一个或者多个对象

> ExecutableValidator#validateParameters
>
> BeanMetaDataManager#getBeanMetaData ,
>
> BeanMetaDataManager#createBeanMetaData
>
> BeanMetaDataManager#getBeanConfigurationForHierarchy
>
> MetaDataProvider#getBeanConfiguration
>
> AnnotationMetaDataProvider#retrieveBeanConfiguration   --- getXXXMetaData --- ConstrainedElements 

```java
// @since 3.1  因为它校验Method  所以它使用的是javax.validation.executable.ExecutableValidator
public class MethodValidationInterceptor implements MethodInterceptor {

	// javax.validation.Validator
	private final Validator validator;

	// 如果没有指定校验器，那使用的就是默认的校验器
	public MethodValidationInterceptor() {
		this(Validation.buildDefaultValidatorFactory());
	}
	public MethodValidationInterceptor(ValidatorFactory validatorFactory) {
		this(validatorFactory.getValidator());
	}
	public MethodValidationInterceptor(Validator validator) {
		this.validator = validator;
	}


	@Override
	@SuppressWarnings("unchecked")
	public Object invoke(MethodInvocation invocation) throws Throwable {
		// Avoid Validator invocation on FactoryBean.getObjectType/isSingleton
		if (isFactoryBeanMetadataMethod(invocation.getMethod())) {
			return invocation.proceed();
		}

		Class<?>[] groups = determineValidationGroups(invocation);

		// Standard Bean Validation 1.1 API  ExecutableValidator是1.1提供的
		ExecutableValidator execVal = this.validator.forExecutables();
		Method methodToValidate = invocation.getMethod();
		Set<ConstraintViolation<Object>> result; // 错误消息result  若存在最终都会ConstraintViolationException异常形式抛出

		try {
			// 先校验方法入参
			result = execVal.validateParameters(invocation.getThis(), methodToValidate, invocation.getArguments(), groups);
		} catch (IllegalArgumentException ex) {
			// 此处回退一步：找到bridged method方法再来一次
			methodToValidate = BridgeMethodResolver.findBridgedMethod(ClassUtils.getMostSpecificMethod(invocation.getMethod(), invocation.getThis().getClass()));
			result = execVal.validateParameters(invocation.getThis(), methodToValidate, invocation.getArguments(), groups);
		}
		if (!result.isEmpty()) { // 有错误就抛异常抛出去
			throw new ConstraintViolationException(result);
		}
		// 执行目标方法  拿到返回值后  再去校验这个返回值
		Object returnValue = invocation.proceed();
		result = execVal.validateReturnValue(invocation.getThis(), methodToValidate, returnValue, groups);
		if (!result.isEmpty()) {
			throw new ConstraintViolationException(result);
		}

		return returnValue;
	}


	// 找到这个方法上面是否有标注 @Validated 注解  从里面拿到分组信息
    // 方法没有标注则从类上找
	// 备注：虽然代理只能标注在类上，但是分组可以标注在类上和方法上 
	protected Class<?>[] determineValidationGroups(MethodInvocation invocation) {
		Validated validatedAnn = AnnotationUtils.findAnnotation(invocation.getMethod(), Validated.class);
		if (validatedAnn == null) {
			validatedAnn = AnnotationUtils.findAnnotation(invocation.getThis().getClass(), Validated.class);
		}
		return (validatedAnn != null ? validatedAnn.value() : new Class<?>[0]);
	}
}
```

#### ExecutableValidator

```
public interface ExecutableValidator {

   //Validates all constraints placed on the parameters of the given method.
   
   <T> Set<ConstraintViolation<T>> validateParameters(T object,
                                          Method method,
                                          Object[] parameterValues,
                                          Class<?>... groups);

   //Validates all return value constraints of the given method.
   <T> Set<ConstraintViolation<T>> validateReturnValue(T object,
                                          Method method,
                                          Object returnValue,
                                          Class<?>... groups);

   //Validates all constraints placed on the parameters of the given constructor.
   <T> Set<ConstraintViolation<T>> validateConstructorParameters(Constructor<? extends T> constructor,
                                                  Object[] parameterValues,
                                                  Class<?>... groups);

   //Validates all return value constraints of the given constructor.
   <T> Set<ConstraintViolation<T>> validateConstructorReturnValue(Constructor<? extends T> constructor,
                                                   T createdObject,
                                                   Class<?>... groups);
}
```

#### `ValidatorImpl`

BridgeMethodResolver

OverridingMethodMustNotAlterParameterConstraints



### **@Validated**

JSR校验注解要想其生效的话，**要求类型级别上必须使用@Validated标注（还能指定验证的Group）**

### @Valid注解（级联校验）

该注解用于验证**级联的属性**、**方法参数**或**方法返回**类型。



#### BeanMetaData

##### BeanMetaDataManager 构造函数

##### BeanMetaDataImp

###### BeanMetaDataBuilder

#### MetaDataProvider

元数据提供者：约束相关元数据（如约束、**默认组序列**等）的`Provider`。它的作用和特点如下：

1. 基于不同的元数据：**如xml、注解。（还有个编程映射）** 这三种类型。对应的枚举类为：

```java
public enum ConfigurationSource {
	ANNOTATION( 0 ),
	XML( 1 ),
	API( 2 ); //programmatic API
}
```

2. `MetaDataProvider`只返回**直接为一个类配置的**元数据
3. 它不处理从超类、接口合并的元数据（`简单的说你@Valid放在接口处是无效的`）

###### BeanConfiguration

```java
public interface MetaDataProvider {

	// 将**注解处理选项**归还给此Provider配置。  它的唯一实现类为：AnnotationProcessingOptionsImpl
	// 它可以配置比如：areMemberConstraintsIgnoredFor  areReturnValueConstraintsIgnoredFor
	// 也就说可以配置：让免于被校验~~~~~~(开绿灯用的)
	AnnotationProcessingOptions getAnnotationProcessingOptions();
	// 返回作用在此Bean上面的`BeanConfiguration`   若没有就返回null了
	// BeanConfiguration持有ConfigurationSource的引用~
	<T> BeanConfiguration<? super T> getBeanConfiguration(Class<T> beanClass);
	
}

// 表示源于一个ConfigurationSource的一个Java类型的完整约束相关配置。  包含字段、方法、类级别上的元数据
// 当然还包含有默认组序列上的元数据（使用较少）
public class BeanConfiguration<T> {
	// 三种来源的枚举
	private final ConfigurationSource source;
	private final Class<T> beanClass;
	// ConstrainedElement表示待校验的元素，可以知道它会如下四个子类：
	// ConstrainedField/ConstrainedType/ConstrainedParameter/ConstrainedExecutable
	
	// 注意：ConstrainedExecutable持有的是java.lang.reflect.Executable对象
	//它的两个子类是java.lang.reflect.Method和Constructor
	private final Set<ConstrainedElement> constrainedElements;

	private final List<Class<?>> defaultGroupSequence;
	private final DefaultGroupSequenceProvider<? super T> defaultGroupSequenceProvider;
	... // 它自己并不处理什么逻辑，参数都是通过构造器传进来的
}
```

##### `AnnotationMetaDataProvider`

**元数据均来自于注解的标注**，然后它是`Hibernate Validation`的默认`configuration source`。它这里会处理标注有`@Valid`的元素

> ConstrainedElement.getConstraints()

##### retrieveBeanConfiguration

##### getCascadingMetaData

Cascading 级联

```java
public class AnnotationMetaDataProvider implements MetaDataProvider {

	private final ConstraintHelper constraintHelper;
	private final TypeResolutionHelper typeResolutionHelper;
	private final AnnotationProcessingOptions annotationProcessingOptions;
	private final ValueExtractorManager valueExtractorManager;

	// 这是一个非常重要的属性，它会记录着当前Bean  所有的待校验的Bean信息~~~
	private final BeanConfiguration<Object> objectBeanConfiguration;

	// 唯一构造函数
	public AnnotationMetaDataProvider(ConstraintHelper constraintHelper,
			TypeResolutionHelper typeResolutionHelper,
			ValueExtractorManager valueExtractorManager,
			AnnotationProcessingOptions annotationProcessingOptions) {
		this.constraintHelper = constraintHelper;
		this.typeResolutionHelper = typeResolutionHelper;
		this.valueExtractorManager = valueExtractorManager;
		this.annotationProcessingOptions = annotationProcessingOptions;

		// 默认情况下，它去把Object相关的所有的方法都retrieve:检索出来放着  我比较费解这件事~~~  
		// 后面才发现：一切为了效率
		this.objectBeanConfiguration = retrieveBeanConfiguration( Object.class );
	}

	// 实现接口方法
	@Override
	public AnnotationProcessingOptions getAnnotationProcessingOptions() {
		return new AnnotationProcessingOptionsImpl();
	}


	// 如果你的Bean是Object  就直接返回了~~~（大多数情况下  都是Object）
	@Override
	@SuppressWarnings("unchecked")
	public <T> BeanConfiguration<T> getBeanConfiguration(Class<T> beanClass) {
		if ( Object.class.equals( beanClass ) ) {
			return (BeanConfiguration<T>) objectBeanConfiguration;
		}
		return retrieveBeanConfiguration( beanClass );
	}
    
    private <T> BeanConfiguration<T> retrieveBeanConfiguration(Class<T> beanClass) {
		// 它检索的范围是：clazz.getDeclaredFields()  什么意思：就是搜集到本类所有的字段  包括private等等  但是不包括父类的所有字段
		Set<ConstrainedElement> constrainedElements = getFieldMetaData( beanClass );
		constrainedElements.addAll( getMethodMetaData( beanClass ) );
		constrainedElements.addAll( getConstructorMetaData( beanClass ) );

		//TODO GM: currently class level constraints are represented by a PropertyMetaData. This
		//works but seems somewhat unnatural
		// 这个TODO很有意思：当前，类级约束由PropertyMetadata表示。这是可行的，但似乎有点不自然
		// ReturnValueMetaData、ExecutableMetaData、ParameterMetaData、PropertyMetaData

		// 总之吧：此处就是把类级别的校验器放进来了（这个set大部分时候都是空的）
		Set<MetaConstraint<?>> classLevelConstraints = getClassLevelConstraints( beanClass );
		if (!classLevelConstraints.isEmpty()) {
			ConstrainedType classLevelMetaData = new ConstrainedType(ConfigurationSource.ANNOTATION, beanClass, classLevelConstraints);
			constrainedElements.add(classLevelMetaData);
		}
		
		// 组装成一个BeanConfiguration返回
		return new BeanConfiguration<>(ConfigurationSource.ANNOTATION, beanClass,
				constrainedElements, 
				getDefaultGroupSequence( beanClass ),  //此类上标注的所有@GroupSequence注解
				getDefaultGroupSequenceProvider( beanClass ) // 此类上标注的所有@GroupSequenceProvider注解
		);
	}
    
    private CascadingMetaDataBuilder getCascadingMetaData(Type type, AnnotatedElement annotatedElement,
			Map<TypeVariable<?>, CascadingMetaDataBuilder> containerElementTypesCascadingMetaData) {
		return CascadingMetaDataBuilder.annotatedObject( type, annotatedElement.isAnnotationPresent( Valid.class ), containerElementTypesCascadingMetaData,
						getGroupConversions( annotatedElement ) );
	}
}
```



##### 处理逻辑

核心解析逻辑在`retrieveBeanConfiguration()`这个私有方法上。总调用此方法的两个原始入口（一个构造器，一个接口方法）：

1. `ValidatorFactory.getValidator()`获取校验器的时候，初始化时会自己`new`一个

2. 调用 Validator.validate() 方法的时候，beanMetaDataManager.getBeanMetaData( rootBeanClass )它会遍历初始化时所有的 metaDataProviders (默认情况下两个，没有xml方式的)，拿出所有的 BeanConfiguration 交给 BeanMetaDataBuilder，最终构建出一个属于此Bean的 BeanMetaData。对此有一点注意事项描述如下：
   1. 处理MetaDataProvider 时会调用 ClassHierarchyHelper.getHierarchy( beanClass )方法，不仅仅处理本类。拿到本类自己和所有父类后，统一交给 provider.getBeanConfiguration( clazz )处理（也就是说任何一个类都会把Object类处理一遍）


### MetaConstraint

#### ConstraintTree#validateSingleConstraint

#### `ConstraintValidator`#isValid 实际验证逻辑

根据不同的 Constraint 调用不同的 ConstraintValidator

```java
success = metaConstraint.validateConstraint( validationContext, valueContext );
// ConstraintTree<A>
boolean validationResult = constraintTree.validateConstraints( executionContext, valueContext );

public abstract class ConstraintTree<A extends Annotation> {
	...
	protected final <T, V> Set<ConstraintViolation<T>> validateSingleConstraint(ValidationContext<T> executionContext,
			ValueContext<?, ?> valueContext,
			ConstraintValidatorContextImpl constraintValidatorContext,
			ConstraintValidator<A, V> validator) {
		...
		V validatedValue = (V) valueContext.getCurrentValidatedValue();
		isValid = validator.isValid( validatedValue, constraintValidatorContext );
		...
		// 显然校验不通过就返回错误消息  否则返回空集合
		if ( !isValid ) {
			return executionContext.createConstraintViolations(valueContext, constraintValidatorContext);
		}
		return Collections.emptySet();
	}
	...
}
```



# 使用

使用`@Validated`去校验方法`Method`，不管从使用上还是原理上，都是非常简单和简约的

#### 约束注解（如@NotNull）不能放在实体类上

若是`@Override`父类/接口的方法，**那么这个入参约束只能写在父类/接口上面**

约束：OverridingMethodMustNotAlterParameterConstraints

#### @NotEmpty/@NotBlank只能哪些类型上？

@NotEmpty：CharSequence、Collection、Map、Array

@NotBlank：CharSequence

#### 接口和实现类上都有注解，以谁为准？

隐含条件：只有校验方法返回值时才有这种可能性。

都有，接口优先

#### 如何校验`级联属性`？

#### 循环依赖问题（参考 Asycn）

平铺（水平排列）校验，1. 注册 MethodValidationPostProcessor 2. 标注@Validated，3. 拦截方法 Interceptor



#### 约束继承

如果子类继承自他的父类，除了校验子类，同时还会校验父类，这就是约束继承（同样适用于接口）。

#### 约束`失败消息message`自定义

每个约束定义中都包含有一个用于提示验证结果的消息模版`message`，并且在声明一个约束条件的时候,你可以通过这个约束注解中的**message属性**来重写默认的消息模版（**这是自定义message最简单的一种方式**）。

PlatformResourceBundleLocator

```java
private ConfigurationImpl() {
		this.validationBootstrapParameters = new ValidationBootstrapParameters();

		// 默认的国际化资源文件加载器USER_VALIDATION_MESSAGES值为：ValidationMessages
		// 这个值就是资源文件的文件名~~~~
		this.defaultResourceBundleLocator = new PlatformResourceBundleLocator(
				ResourceBundleMessageInterpolator.USER_VALIDATION_MESSAGES
		);
		this.defaultTraversableResolver = TraversableResolvers.getDefault();
		this.defaultConstraintValidatorFactory = new ConstraintValidatorFactoryImpl();
		this.defaultParameterNameProvider = new DefaultParameterNameProvider();
		this.defaultClockProvider = DefaultClockProvider.INSTANCE;
	}
```



```java
javax.validation.constraints.AssertFalse.message     = 只能为false
javax.validation.constraints.AssertTrue.message      = 只能为true
javax.validation.constraints.DecimalMax.message      = 必须小于或等于{value}
javax.validation.constraints.DecimalMin.message      = 必须大于或等于{value}
javax.validation.constraints.Digits.message          = 数字的值超出了允许范围(只允许在{integer}位整数和{fraction}位小数范围内)
javax.validation.constraints.Email.message           = 不是一个合法的电子邮件地址
javax.validation.constraints.Future.message          = 需要是一个将来的时间
javax.validation.constraints.FutureOrPresent.message = 需要是一个将来或现在的时间
javax.validation.constraints.Max.message             = 最大不能超过{value}
javax.validation.constraints.Min.message             = 最小不能小于{value}
javax.validation.constraints.Negative.message        = 必须是负数
javax.validation.constraints.NegativeOrZero.message  = 必须是负数或零
javax.validation.constraints.NotBlank.message        = 不能为空
javax.validation.constraints.NotEmpty.message        = 不能为空
javax.validation.constraints.NotNull.message         = 不能为null
javax.validation.constraints.Null.message            = 必须为null
javax.validation.constraints.Past.message            = 需要是一个过去的时间
javax.validation.constraints.PastOrPresent.message   = 需要是一个过去或现在的时间
javax.validation.constraints.Pattern.message         = 需要匹配正则表达式"{regexp}"
javax.validation.constraints.Positive.message        = 必须是正数
javax.validation.constraints.PositiveOrZero.message  = 必须是正数或零
javax.validation.constraints.Size.message            = 个数必须在{min}和{max}之间
```

####  @GroupSequenceProvider、@GroupSequence控制数据校验顺序，解决多字段联合逻辑校验问题

```java
// 该接口定义了：动态Group序列的协定
// 要想它生效，需要在T上标注@GroupSequenceProvider注解并且指定此类为处理类
// 如果`Default`组对T进行验证，则实际验证的实例将传递给此类以确定默认组序列（这句话特别重要  下面用例子解释）
public interface DefaultGroupSequenceProvider<T> {
	// 合格方法是给T返回默认的组（多个）。因为默认的组是Default嘛~~~通过它可以自定指定
	// 入参T object允许在验证值状态的函数中动态组合默认组序列。（非常强大）
	// object是待校验的Bean。它可以为null哦~（Validator#validateValue的时候可以为null）

	// 返回值表示默认组序列的List。它的效果同@GroupSequence定义组序列，尤其是列表List必须包含类型T
	List<Class<?>> getValidationGroups(T object);
}
```

##### 使用JSR提供的`@GroupSequence`注解控制校验顺序

> 说明：顺序只能控制在分组级别，无法控制在约束注解级别。因为一个类内的约束（同一分组内），它的顺序是`Set<MetaConstraint<?>> metaConstraints`来保证的，所以可以认为同一分组内的校验器是木有执行的先后顺序的（不管是类、属性、方法、构造器…）

# Java Validation

#### 为什么要有数据验证？

**在任何时候，当你要处理一个应用程序的业务逻辑，数据校验是你必须要考虑和面对的事情**。应用程序必须通过某种手段来确保输入进来的数据从语义上来讲是正确的



**不同的层出现了相同的验证代码**

1. 需要写**大量的**代码来进行参数验证。(这种代码多了就算垃圾代码)
2. **需要通过注释**来知道每个入参的约束是什么（否则别人咋看得懂）
3. 每个程序员做参数验证的方式不一样，参数验证不通过抛出的异常也不一样（后期几乎没法维护）

## Java Bean Validation

> Bean Validation 2.0 是JSR第380号标准。该标准连接如下：https://www.jcp.org/en/egc/view?id=380
> Bean Validation的主页：http://beanvalidation.org
> Bean Validation的参考实现：https://github.com/hibernate/hibernate-validator

`Bean Validation`是一个通过配置注解来验证参数的框架，它包含两部分`Bean Validation API`（规范）和`Hibernate Validator`（实现）。

`Bean Validation`是Java定义的一套**基于注解/xml**的数据校验规范

```java
<dependency>
    <groupId>javax.validation</groupId>
    <artifactId>validation-api</artifactId>
    <!-- <version>1.1.0.Final</version> -->
    <version>2.0.1.Final</version>
</dependency>
<!---- ---->
<dependency>
    <groupId>jakarta.validation</groupId>
    <artifactId>jakarta.validation-api</artifactId>
    <version>2.0.1</version>
</dependency>

<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>6.0.17.Final</version>
</dependency>
```

> #### Bean Validation 2.0的关注点（新特性）
>
> 对Java的最低版本要求是Java 8
> 支持容器的校验，通过TYPE_USE类型的注解实现对容器内容的约束：List<@Email String>
> 支持日期/时间的校验，@Past和@Future
> 拓展元数据（新增注解）：@Email，@NotEmpty，@NotBlank，@Positive， @PositiveOrZero，@Negative，@NegativeOrZero，@PastOrPresent和@FutureOrPresent
> 1. 像@Email、@NotEmpty、@NotBlank之前是Hibernate额外提供的，2.0标准后hibernate自动退位让贤并且标注为过期了
>   Bean Validation 2.0的唯一实现为Hibernate Validator。（其实还有Apache BVal，但是你懂的，forget it）
>   对于Hibernate Validator，它自己也扩展了一些注解支持。
> 2. 6.0以上版本新增(对应标准2.0版本)：@UniqueElements、@ISBN、@CodePointLength、、、、、、、、
> 3. 6.0以下版本可以使用的： @URL、@ScriptAssert、@SafeHtml、@Range、@ParameterScriptAssert、@Mod11Check、@Mod10Check、@LuhnCheck、@Length、@EAN、@Currency、@CreditCardNumber、@ConstraintComposition、@DurationMax、@DurationMin、**@REGON、@PESEL、@NIP、@TituloEleitoral、@CPF、@CNPJ**
> 4. Hibernate Validator默认会校验完所有的属性，然后返回所有的验证失败信息。开启fail fast mode后，只要有一个验证失败，则返回验证失败信息。



### Validation

fluent api

```java
public class Validation {

	// 方式一
	public static ValidatorFactory buildDefaultValidatorFactory() {
		return byDefaultProvider().configure().buildValidatorFactory();
	}
	// 方式二
	public static GenericBootstrap byDefaultProvider() {
		return new GenericBootstrapImpl();
	}
	// 方式三
	public static <T extends Configuration<T>, U extends ValidationProvider<T>> ProviderSpecificBootstrap<T> byProvider(Class<U> providerType) {
		return new ProviderSpecificBootstrapImpl<>( providerType );
	}
	...
}

//
HibernateValidatorConfiguration configuration = Validation.byProvider(HibernateValidator.class)
        // .providerResolver( ... ) // 因为制定了Provider，这个参数就可选了
        .configure()
        .failFast(false);
ValidatorFactory validatorFactory = configuration.buildValidatorFactory();
```

#### `HibernateValidatorConfiguration`

> META-INF/validation.xml 
>
> DefaultValidationProviderResolver

```java
public interface HibernateValidatorConfiguration extends Configuration<HibernateValidatorConfiguration> {

	// 这批属性，证明直接可以通过System属性值来控制，大大地方便~
	// 这个机制快速失败机制：true检查完一个有错误就返回，false全部检查完把错误消息一起返回   默认false
	String FAIL_FAST = "hibernate.validator.fail_fast"; 
	String ALLOW_PARAMETER_CONSTRAINT_OVERRIDE = "hibernate.validator.allow_parameter_constraint_override";
	String ALLOW_MULTIPLE_CASCADED_VALIDATION_ON_RESULT = "hibernate.validator.allow_multiple_cascaded_validation_on_result";
	String ALLOW_PARALLEL_METHODS_DEFINE_PARAMETER_CONSTRAINTS = "hibernate.validator.allow_parallel_method_parameter_constraint";
	// @since 5.2
	@Deprecated
	String CONSTRAINT_MAPPING_CONTRIBUTOR = "hibernate.validator.constraint_mapping_contributor";
	// @since 5.3
	String CONSTRAINT_MAPPING_CONTRIBUTORS = "hibernate.validator.constraint_mapping_contributors";
	// @since 6.0.3
	String ENABLE_TRAVERSABLE_RESOLVER_RESULT_CACHE = "hibernate.validator.enable_traversable_resolver_result_cache";
	// @since 6.0.3  ScriptEvaluatorFactory：执行脚本
	@Incubating
	String SCRIPT_EVALUATOR_FACTORY_CLASSNAME = "hibernate.validator.script_evaluator_factory";
	// @since 6.0.5 comparing date/time in temporal constraints. In milliseconds.
	@Incubating
	String TEMPORAL_VALIDATION_TOLERANCE = "hibernate.validator.temporal_validation_tolerance";

	// ResourceBundleMessageInterpolator用于 load resource bundles
	ResourceBundleLocator getDefaultResourceBundleLocator();
	// 创建一个ConstraintMapping：通过编程API配置的约束映射
	// 设置映射后，必须通过addMapping（constraintmapping）将其添加到此配置中。
	ConstraintMapping createConstraintMapping();
	// 拿到所有的值提取器  @since 6.0
	@Incubating
	Set<ValueExtractor<?>> getDefaultValueExtractors();

	// 往下就开始配置了~~~~~~~~~~
	HibernateValidatorConfiguration addMapping(ConstraintMapping mapping);
	HibernateValidatorConfiguration failFast(boolean failFast);
	// used for loading user-provided resources:
	HibernateValidatorConfiguration externalClassLoader(ClassLoader externalClassLoader);
	// true：表示允许覆盖约束的方法。false表示不予许（抛出异常）  默认值是false
	HibernateValidatorConfiguration allowOverridingMethodAlterParameterConstraint(boolean allow);
	// 定义是否允许对返回值标记多个约束以进行级联验证。  默认是false
	HibernateValidatorConfiguration allowMultipleCascadedValidationOnReturnValues(boolean allow);
	// 定义约束的**并行方法**是否应引发ConstraintDefinitionException
	HibernateValidatorConfiguration allowParallelMethodsDefineParameterConstraints(boolean allow);
	// 是否允许缓存TraversableResolver  默认值是true
	HibernateValidatorConfiguration enableTraversableResolverResultCache(boolean enabled);
	// 设置一个脚本执行器
	@Incubating
	HibernateValidatorConfiguration scriptEvaluatorFactory(ScriptEvaluatorFactory scriptEvaluatorFactory);
	// 允许在时间约束中比较日期/时间时设置可接受的误差范围
	// 比如@Past @PastOrPresent @Future @FutureOrPresent
	@Incubating
	HibernateValidatorConfiguration temporalValidationTolerance(Duration temporalValidationTolerance);
	// 允许设置将传递给约束验证器的有效负载。如果多次调用该方法，则只传播最后传递的有效负载。
	@Incubating
	HibernateValidatorConfiguration constraintValidatorPayload(Object constraintValidatorPayload);
}
```

#### Configuration

```java
public interface Configuration<T extends Configuration<T>> {
	// 该方法调用后就不会再去找META-INF/validation.xml了
	T ignoreXmlConfiguration();
	// 消息内插器  它是个狠角色，关于它的使用场景，后续会有详解（包括Spring都实现了它来做事）
	// 它的作用是：插入给定的约束冲突消息
	T messageInterpolator(MessageInterpolator interpolator);
	// 确定bean验证提供程序是否可以访问属性的协定。对每个正在验证或级联的属性调用此约定。（Spring木有实现它）
	// 对每个正在验证或级联的属性都会调用此约定
	// Traversable： 可移动的
	T traversableResolver(TraversableResolver resolver);
	// 创建ConstraintValidator的工厂
	// ConstraintValidator：定义逻辑以验证给定对象类型T的给定约束A。(A是个注解类型)
	T constraintValidatorFactory(ConstraintValidatorFactory constraintValidatorFactory);
	// ParameterNameProvider：提供Constructor/Method的方法名们
	T parameterNameProvider(ParameterNameProvider parameterNameProvider);
	// java.time.Clock 用作判定@Future和@Past（默认取值当前时间）
	// 若你希望他是个逻辑实现，提供一个它即可
	// @since 2.0
	T clockProvider(ClockProvider clockProvider);
	// 值提取器。这是add哦~ 负责从Optional、List等这种容器里提取值~
	// @since 2.0
	T addValueExtractor(ValueExtractor<?> extractor);
	// 加载xml文件
	T addMapping(InputStream stream);
	// 添加特定的属性给Provider用的。此属性等效于XML配置属性。
	// 此方法通常是框架自己分析xml文件得到属性值然后放进去，调用者一般不使用（当然也可以用）
	T addProperty(String name, String value);
	
	// 下面都是get方法喽
	MessageInterpolator getDefaultMessageInterpolator();
	TraversableResolver getDefaultTraversableResolver();
	ConstraintValidatorFactory getDefaultConstraintValidatorFactory();
	ParameterNameProvider getDefaultParameterNameProvider();
	ClockProvider getDefaultClockProvider();
	BootstrapConfiguration getBootstrapConfiguration(); // 整个配置也可返回出去

	// 上面都是工作，这个方法才是最终需要调用的：得到一个ValidatorFactory
	ValidatorFactory buildValidatorFactory();
}
```



### ValidationProviderResolver：验证提供程序处理器

> `javax.validation.ValidationProviderResolver`：确定运行时整个环境中可用的`ValidationProvider`列表。
>
> META-INF/services/javax.validation.spi.ValidationProvider

```java
public interface ValidationProviderResolver {
	// 返回所有可用的ValidationProvider
	List<ValidationProvider<?>> getValidationProviders();
}

public class Validation {
	private static class DefaultValidationProviderResolver implements ValidationProviderResolver {
		@Override
		public List<ValidationProvider<?>> getValidationProviders() {
			return GetValidationProviderListAction.getValidationProviderList();
		}
	}

	// 调用的工具方法里，最为核心的是这一句代码，其余的就不用多说了
	ServiceLoader<ValidationProvider> loader = ServiceLoader.load( ValidationProvider.class, classloader );
	...
}
```



### ValidationProvider：验证提供器

##### buildValidatorFactory()

```java
public interface ValidationProvider<T extends Configuration<T>> {
	// 这两个方法都是通过引导器BootstrapState 创建一个Configuration
	T createSpecializedConfiguration(BootstrapState state);
	Configuration<?> createGenericConfiguration(BootstrapState state);
	
	// 根据引导器，得到一个ValidatorFactory 
	// 请注意：Configuration也有同名方法：buildValidatorFactory()
	// Configuration的实现最终调用的是这个方法：getProvider().buildValidatorFactory(this)
	ValidatorFactory buildValidatorFactory(ConfigurationState configurationState);
}
```

##### HibernateValidator

```java
public class HibernateValidator implements ValidationProvider<HibernateValidatorConfiguration> {

	// 此处直接new ConfigurationImpl()  他是Hibernate校验的配置类
	// 请注意此两者的区别：一个传的是this，一个传的是入参state~~~
	@Override
	public HibernateValidatorConfiguration createSpecializedConfiguration(BootstrapState state) {
		return HibernateValidatorConfiguration.class.cast( new ConfigurationImpl( this ) );
	}
	@Override
	public Configuration<?> createGenericConfiguration(BootstrapState state) {
		return new ConfigurationImpl( state );
	}

	// ValidatorFactoryImpl是个ValidatorFactory ，也是最为重要的一个类之一
	@Override
	public ValidatorFactory buildValidatorFactory(ConfigurationState configurationState) {
		return new ValidatorFactoryImpl( configurationState );
	}
}
```



### ConstraintDescriptor：约束描述符

`javax.validation.metadata`

描述单个**约束或者组合（composing）约束**，这个描述非常的重要，到这里就把约束的注解、Groups、Payloads、getAttributes等等都关联起来了，它就是个`metadata`

```java
public interface ConstraintDescriptor<T extends Annotation> {
	// 返回此约束注解。  如果是组合约束（注解上面标注解），本注解的属性值覆盖组合进来的
	T getAnnotation();
	// 返回原本的message（还没有插值）
	String getMessageTemplate();
	// 获得该注解所属的分组  默认都属于javax.validation.groups.Default这个分组
	Set<Class<?>> getGroups();
	// 该约束持有的负载Payload。  Payload是个标记接口，木有任何方法  子接口有Unwrap和Skip接口
	Set<Class<? extends Payload>> getPayload();

	// 标注该约束作用在什么地方（入参or返回值？？？）  因为约束几乎可以标注在任何位	置，并且还可以标注在TYPE_USE上
	// TYPE_USE：java8新增的ElementType  可以写在字段上、类上面上。。。

	//ConstraintTarget 注解取值如下：
	//IMPLICIT：自动判断
			// 如果既不在方法上也不在构造函数上，则表示已注释的元素（类/字段）
			// 如果在没有参数的方法或构造函数上，那就作用在返回值上
			// 如果在没有返回值的方法上，那就作用在入参上
	// RETURN_VALUE：作用在方法/构造函数的返回值上
	// PARAMETERS：作用在方法/构造函数的入参上
	ConstraintTarget getValidationAppliesTo();

	// 获取需要作用在此约束上的所有校验器ConstraintValidator。（因为可能是组合，所以会有多个校验器）
	List<Class<? extends ConstraintValidator<T, ?>>> getConstraintValidatorClasses();

	// 就是此注解的属性-值的Map。包括那三大基础属性
	Map<String, Object> getAttributes();
	// 返回所遇的约束描述们~~~（毕竟可以标注多个注解  组合租借等等）
	Set<ConstraintDescriptor<?>> getComposingConstraints();

	// 如果约束注解上标注有 @ReportAsSingleViolation  此处就有返回值
	// 此注解作用：如果任何组合注解失败，承载此注解的约束注解将**返回组合注解错误报告**。
	// 它会忽略每个单独注解写的错误报告message~~~~**合成约束的计算将在第一个验证错误时停止**，也就是它有短路的效果
	boolean isReportAsSingleViolation();

	// @since 2.0 ValidateUnwrappedValue用于特定约束的展开行为（和ValueExtractor提取容器内的值有关）
	// DEFAULT：默认行为
	// UNWRAP：该值在校验前展开，既校验作用于容器内的值
	// SKIP：校验前不展开。相当于直接作用于本元素。比如作用在List上，而非List里面的元素
	ValidateUnwrappedValue getValueUnwrapping();
	<U> U unwrap(Class<U> type);
}
```



#### `ConstraintDescriptorImpl`

构造时准备

```java
public class ConstraintDescriptorImpl<T extends Annotation> implements ConstraintDescriptor<T>, Serializable {
	
	// 这些注解是会被忽略的，就是去注解上的注解时忽略这些注解
	private static final List<String> NON_COMPOSING_CONSTRAINT_ANNOTATIONS = Arrays.asList(
			Documented.class.getName(),
			Retention.class.getName(),
			Target.class.getName(),
			Constraint.class.getName(),
			ReportAsSingleViolation.class.getName(),
			Repeatable.class.getName(),
			Deprecated.class.getName()
	);
	// 该约束定义的ElementType~~~标注在字段上、方法上、构造器上？
	private final ElementType elementType;

	...
	// 校验源。注解定义实际位置，在根类或类层次结构中的某个地方
	// DEFINED_LOCALLY：约束定义在根类
	// DEFINED_IN_HIERARCHY：约束定义在父类、接口处等
	private final ConstraintOrigin definedOn;
	
	// 当前约束的类型
	// GENERIC：非**交叉参数**约束
	// CROSS_PARAMETER：交叉参数约束
	private final ConstraintType constraintType;
	
	// 上面已解释
	private final ConstraintTarget validationAppliesTo;
	
	// 多个约束的联合类型
	// OR：或者关系
	// AND：并且关系
	// ALL_FALSE：相当于必须所有条件都是false才行
	private final CompositionType compositionType;
	private final int hashCode;

	// 几乎所有的准备逻辑都在这个唯一的构造函数里
	public ConstraintDescriptorImpl(ConstraintHelper constraintHelper, Member member, ConstraintAnnotationDescriptor<T> annotationDescriptor,
			ElementType type, Class<?> implicitGroup, ConstraintOrigin definedOn, ConstraintType externalConstraintType) {
		this.annotationDescriptor = annotationDescriptor;
		this.elementType = type;
		this.definedOn = definedOn;
		// 约束上是否标注了@ReportAsSingleViolation注解~
		this.isReportAsSingleInvalidConstraint = annotationDescriptor.getType().isAnnotationPresent(ReportAsSingleViolation.class);
		
		// annotationDescriptor.getGroups()拿到所属分组
		this.groups = buildGroupSet( annotationDescriptor, implicitGroup );
		// 拿到负载annotationDescriptor.getPayload()
		this.payloads = buildPayloadSet( annotationDescriptor );
		// 对负载payloads类型进行区分  是否要提取值呢？？
		this.valueUnwrapping = determineValueUnwrapping( this.payloads, member, annotationDescriptor.getType() );
		// annotationDescriptor.getValidationAppliesTo()
		// 也就是说你自己自定义注解的时候，可以定义一个属性validationAppliesTo = ConstraintTarget.calss 哦~~~~
		this.validationAppliesTo = determineValidationAppliesTo( annotationDescriptor );

		// 委托constraintHelper帮助拿到此注解类型下所有的校验器们
		this.constraintValidatorClasses = constraintHelper.getAllValidatorDescriptors( annotationDescriptor.getType() )
				.stream()
				.map( ConstraintValidatorDescriptor::getValidatorClass )
				.collect( Collectors.collectingAndThen( Collectors.toList(), CollectionHelper::toImmutableList ) );

		// ValidationTarget配合注解`@SupportedValidationTarget(ValidationTarget.ANNOTATED_ELEMENT)`使用
		List<ConstraintValidatorDescriptor<T>> crossParameterValidatorDescriptors = CollectionHelper.toImmutableList( constraintHelper.findValidatorDescriptors(
				annotationDescriptor.getType(),
				ValidationTarget.PARAMETERS
		) );
		List<ConstraintValidatorDescriptor<T>> genericValidatorDescriptors = CollectionHelper.toImmutableList( constraintHelper.findValidatorDescriptors(
				annotationDescriptor.getType(),
				ValidationTarget.ANNOTATED_ELEMENT
		) );
		if ( crossParameterValidatorDescriptors.size() > 1 ) {
			throw LOG.getMultipleCrossParameterValidatorClassesException( annotationDescriptor.getType() );
		}

		// 判定是交叉参数约束  还是非交叉参数约束（这个决策非常的复杂）
		this.constraintType = determineConstraintType(
				annotationDescriptor.getType(), member, type,
				!genericValidatorDescriptors.isEmpty(),
				!crossParameterValidatorDescriptors.isEmpty(),
				externalConstraintType
		);
		// 这个方法比较复杂：解析出作用在此的约束们
		// member：比如Filed字段age  此处：annotationDescriptor.getType()为注解@Positive
		// type.getDeclaredAnnotations() 其实是上面会忽略掉的注解了。当然还可能有@SupportedValidationTarget等等@NotNull等注解
		this.composingConstraints = parseComposingConstraints( constraintHelper, member, constraintType );
		
		this.compositionType = parseCompositionType( constraintHelper );
		validateComposingConstraintTypes();
		if ( constraintType == ConstraintType.GENERIC ) {
			this.matchingConstraintValidatorDescriptors = CollectionHelper.toImmutableList( genericValidatorDescriptors );
		} else {
			this.matchingConstraintValidatorDescriptors = CollectionHelper.toImmutableList( crossParameterValidatorDescriptors );
		}

		this.hashCode = annotationDescriptor.hashCode();
	}
	...
}
```



### MessageInterpolator：message插值器

对message内容进行格式化，若有占位符{}或者el表达式就执行替换和计算。**对于语法错误应该尽量的宽容。**

```java
public interface MessageInterpolator {

	// 根据约束验证上下文格式化消息模板。(Locale对国际化提供了支持~)
	String interpolate(String messageTemplate, Context context);
	String interpolate(String messageTemplate, Context context,  Locale locale);

	// 与插值上下文相关的信息。
	interface Context {
		// ConstraintDescriptor对应于正在验证的约束，整体上进行了描述  上面已说明
		ConstraintDescriptor<?> getConstraintDescriptor();
		// 正在被校验的值
		Object getValidatedValue();
		// 返回允许访问特定于提供程序的API的指定类型的实例。如果bean验证提供程序实现不支持指定的类
		<T> T unwrap(Class<T> type);
	}
}
```

#### AbstractMessageInterpolator#interpolateMessage

```java
public abstract class AbstractMessageInterpolator implements MessageInterpolator {
	private static final int DEFAULT_INITIAL_CAPACITY = 100;
	private static final float DEFAULT_LOAD_FACTOR = 0.75f;
	private static final int DEFAULT_CONCURRENCY_LEVEL = 16;

	// 默认的国际化资源名称，支持多国语言，请参见下面截图
	private static final String DEFAULT_VALIDATION_MESSAGES = "org.hibernate.validator.ValidationMessages";
	// 规范中定义的用户提供的消息束的名称。
	public static final String USER_VALIDATION_MESSAGES = "ValidationMessages";
	// 由约束定义贡献者定义的消息束的默认名称。
	public static final String CONTRIBUTOR_VALIDATION_MESSAGES = "ContributorValidationMessages";

	// 当前JVM默认的Locale
	private final Locale defaultLocale;
	// 用户指定的国际资源文件  默认的  贡献者贡献的资源文件
	private final ResourceBundleLocator userResourceBundleLocator;
	private final ResourceBundleLocator defaultResourceBundleLocator;
	private final ResourceBundleLocator contributorResourceBundleLocator;

	// 这个Map缓存了1-3步插补文字
	private final ConcurrentReferenceHashMap<LocalizedMessage, String> resolvedMessages;
	// 步骤4
	private final ConcurrentReferenceHashMap<String, List<Token>> tokenizedParameterMessages;
	// 步骤5（El表达式~）
	private final ConcurrentReferenceHashMap<String, List<Token>> tokenizedELMessages;

	public AbstractMessageInterpolator(ResourceBundleLocator userResourceBundleLocator, ResourceBundleLocator contributorResourceBundleLocator, boolean cacheMessages) {
		defaultLocale = Locale.getDefault(); // 默认的Locale
		// 用户自定义的定位器
		if ( userResourceBundleLocator == null ) {
			this.userResourceBundleLocator = new PlatformResourceBundleLocator( USER_VALIDATION_MESSAGES );
		} else {
			this.userResourceBundleLocator = userResourceBundleLocator;
		}
		... // 其它Map的初始化
	}

	@Override
	public String interpolate(String message, Context context) {
		return interpolateMessage( message, context, defaultLocale);
	}
	// 此处就开始处理message消息了。比如本文可能的消息是：
	// name字段->名字不能为null  (若是自定义)
	// age字段->{javax.validation.constraints.Positive.message}
	private String interpolateMessage(String message, Context context, Locale locale) throws MessageDescriptorFormatException {
		// if the message does not contain any message parameter, we can ignore the next steps and just return
		// the unescaped message. It avoids storing the message in the cache and a cache lookup.
		if ( message.indexOf( '{' ) < 0 ) {
			return replaceEscapedLiterals( message );
		}

		String resolvedMessage = null;

		// either retrieve message from cache, or if message is not yet there or caching is disabled,
		// perform message resolution algorithm (step 1)
		if ( cachingEnabled ) {
			resolvedMessage = resolvedMessages.computeIfAbsent( new LocalizedMessage( message, locale ), lm -> resolveMessage( message, locale ) );
		} else {
			// 结合国际化资源文件处理~~
			resolvedMessage = resolveMessage( message, locale );
		}
		
		// 2-3步骤：若字符串里含有{param} / ${expr}这种 就进来解析
		// 	给占位符插值依赖于这个抽象方法public abstract String interpolate(Context context, Locale locale, String term);
		// 解析EL表达式也是依赖于这个方法~
			
		// 最后：处理转义字符
		...
		return resolvedMessage;
	}
	...
	//抽象方法，给你context，给你locale  给你term(字符串)，你帮我把这个字符串给我处理了
	public abstract String interpolate(Context context, Locale locale, String term);
}
```

##### ParameterMessageInterpolator#interpolate

资源束消息插值器，**不支持el表达式**，支持参数值表达式

```java
public class ParameterMessageInterpolator extends AbstractMessageInterpolator {
	@Override
	public String interpolate(Context context, Locale locale, String term) {
		// 简单的说就是以$打头，就认为是EL表达式  啥都不处理
		if ( InterpolationTerm.isElExpression( term ) ) {
			return term;
		} else {
			// 核心处理方法是context.getConstraintDescriptor().getAttributes().get( parameter ) 拿到对应的值
			ParameterTermResolver parameterTermResolver = new ParameterTermResolver();
			return parameterTermResolver.interpolate( context, term );
		}
	}
}
```

##### ResourceBundleMessageInterpolator

支持EL表达式

### TraversableResolver：可遍历处理器

> **确定某个属性是否能被ValidationProvider访问**
>
> 注意：访问每个属性的时候它都会被调用来判断

```java
public interface TraversableResolver {

	// 是否是可达的
	boolean isReachable(Object traversableObject,
						Node traversableProperty,
						Class<?> rootBeanType,
						Path pathToTraversableObject,
						ElementType elementType);
	// 是否是可级联的
	boolean isCascadable(Object traversableObject,
						 Node traversableProperty,
						 Class<?> rootBeanType,
						 Path pathToTraversableObject,
						 ElementType elementType);
}
```



### ConstraintValidatorFactory：约束验证器工厂

```java
public interface ConstraintValidatorFactory {
	<T extends ConstraintValidator<?, ?>> T getInstance(Class<T> key);
	// 释放方法是提供给Spring容器集成时 .destroyBean(instance);的
	void releaseInstance(ConstraintValidator<?, ?> instance);
}
```

#### ConstraintValidatorFactoryImpl

```java
public class ConstraintValidatorFactoryImpl implements ConstraintValidatorFactory {

	// NewInstance的run方法最终就是执行了这句话：clazz.getConstructor().newInstance()而已
	// 因此最终就是创建了一个key的实例而已~  Spring相关的会把它和Bean容器结合起来
	@Override
	public final <T extends ConstraintValidator<?, ?>> T getInstance(Class<T> key) {
		return run( NewInstance.action( key, "ConstraintValidator" ) );
	}
	@Override
	public void releaseInstance(ConstraintValidator<?, ?> instance) {
		// noop
	}

	// 入参是个函数式接口:java.security.PrivilegedAction
	private <T> T run(PrivilegedAction<T> action) {
		return System.getSecurityManager() != null ? AccessController.doPrivileged( action ) : action.run();
	}
}
```



### ParameterNameProvider：参数名提供

**提供方法、构造函数的入参names们**

```java
public interface ParameterNameProvider {
	List<String> getParameterNames(Constructor<?> constructor);
	List<String> getParameterNames(Method method);
}
```

#### DefaultParameterNameProvider

```java
public class DefaultParameterNameProvider implements ParameterNameProvider {
	@Override
	public List<String> getParameterNames(Constructor<?> constructor) {
		return doGetParameterNames( constructor );
	}
	@Override
	public List<String> getParameterNames(Method method) {
		return doGetParameterNames( method );
	}

	// Executable是1.8提供的抽象类。两个子类刚好就是：Method和Constructor
	// 因此在Java8后一个方法搞定：executable.getParameters();
	private List<String> doGetParameterNames(Executable executable) {
		Parameter[] parameters = executable.getParameters();
		List<String> parameterNames = new ArrayList<>( parameters.length );
		for ( Parameter parameter : parameters ) {
			parameterNames.add( parameter.getName() );
		}
		return Collections.unmodifiableList( parameterNames );
	}
}
```

#### ~~ReflectionParameterNameProvider~~

**@deprecated since 6.0**，因为通过反射去获取方法名字已经是默认的了



### ClockProvider 时钟提供

就是提供一个`Clock`，给`@Past`、`@Future`等判断作为参考

### ValueExtractor：值提取器

**从容器内把值提取处理**

> 注意：提取值的时候，会执行`doValidate`完成校验。

ValueExtractor<int @ExtractedValue[]>

```java
// @since 2.0
public interface ValueExtractor<T> {
	// 从原始值originalValue提取到receiver里
	void extractValues(T originalValue, ValueReceiver receiver);

	// 提供一组方法，用于接收ValueExtractor提取出来的值
	// 必须将该值传递给与原始值类型对应的最佳方法。
	interface ValueReceiver {
		// 接收从对象中提取的值。
		void value(String nodeName, Object object);
		// 接收从未编入索引的可ITerable对象中提取的值，如List、Map、Iterable等
		void iterableValue(String nodeName, Object object);
		// 处理List
		void indexedValue(String nodeName, int i, Object object);
		// 处理Map
		void keyedValue(String nodeName, Object key, Object object);
	}
}
```



### ValidatorContext：验证器上下文

创建`Validator`的上下文，例如，建立不同的消息插值器或可遍历分解器

```java
public interface ValidatorContext {
	ValidatorContext messageInterpolator(MessageInterpolator messageInterpolator);
	ValidatorContext traversableResolver(TraversableResolver traversableResolver);
	ValidatorContext constraintValidatorFactory(ConstraintValidatorFactory factory);
	ValidatorContext parameterNameProvider(ParameterNameProvider parameterNameProvider);
	// @since 2.0
	ValidatorContext clockProvider(ClockProvider clockProvider);
	ValidatorContext addValueExtractor(ValueExtractor<?> extractor);

	// 最终的方法
	Validator getValidator();
}
```

HibernateValidatorContext ~= HibernateValidatorConfiguration

#### ValidatorContextImpl

```java
public class ValidatorContextImpl implements HibernateValidatorContext {
	// 创建一个Validator
	private final ValidatorFactoryImpl validatorFactory;
	// 它加入了上面所有的内置的值提取器
	private final ValueExtractorManager valueExtractorManager;


	// 拿到一个校验器  使用ValidatorFactory  Validator就是最终对Bean进行校验的东西  
	// 它持有各种上下文，各种插值器、提取器等等
	@Override
	public Validator getValidator() {
		return validatorFactory.createValidator(
				constraintValidatorFactory,
				valueExtractorDescriptors.isEmpty() ? valueExtractorManager : new ValueExtractorManager( valueExtractorManager, valueExtractorDescriptors ),
				validatorFactoryScopedContextBuilder.build(),
				methodValidationConfigurationBuilder.build()
		);
	}
}
```



### ConstraintHelper：约束帮助类

@Constraint(validatedBy = { })，注册内置的`ConstraintValidator`

### ConstraintViolation：约束冲突

描述约束冲突。此对象公开约束冲突上下文以及描述冲突的消息。

```java
public interface ConstraintViolation<T> {
	// 返回已经插值过的错误消息
	String getMessage();
	// 未插值过的原始消息模版
	String getMessageTemplate();
	// 校验的Root Bean（若是校验方法那就是方法所在的Bean）
	T getRootBean();
	Class<T> getRootBeanClass();

	Object getLeafBean();
	Object[] getExecutableParameters();
	Object getExecutableReturnValue();
	// the property path to the value from {@code rootBean}
	Path getPropertyPath();
	// 验证木有通过的值
	Object getInvalidValue();
	// 这个很重要~~~信息量很大
	ConstraintDescriptor<?> getConstraintDescriptor();
	<U> U unwrap(Class<U> type);
}
```



#### ConstraintViolationException

### ConstraintValidatorContext：约束验证上下文

在应用给定的约束验证器(`ConstraintValidator`)时，提供上下文数据和操作。每一个约束（注解）都`至少`对应一个`ConstraintValidator`

```java
public interface ConstraintValidatorContext {
	// 禁用默认的错误时生成`ConstraintViolation`的方式（默认是使用message嘛）
	// 它的作用是：自己根据不同的message、或者不同的属性来生成不同的ConstraintViolation
	void disableDefaultConstraintViolation();
	// 未插值的message：constraintDescriptor.getMessageTemplate()
	String getDefaultConstraintMessageTemplate();
	ClockProvider getClockProvider();

	// 关于ConstraintViolationBuilder此处就不能在展开了，功能大强大  使用起来也太复杂了
	// 它的作用就是根据message模版，来添加和生成一个ConstraintViolation
	// ConstraintViolationBuilder 可以设置各种参数~~~
	ConstraintViolationBuilder buildConstraintViolationWithTemplate(String messageTemplate);
	<T> T unwrap(Class<T> type);
}
```

#### HibernateConstraintValidatorContext

#### ConstraintValidatorContextImpl

```java
public boolean isValid(String value, ConstraintValidatorContext constraintValidatorContext) {
        HibernateConstraintValidatorContext context = constraintValidatorContext.unwrap(HibernateConstraintValidatorContext.class);

        // 在addConstraintViolation之前把参数放进去，就可以创建出不同的ConstraintViolation了
        // 若不这么做，所有的ConstraintViolation取值都是一样的喽~~~
        context.addMessageParameter("foo", "bar");
        context.buildConstraintViolationWithTemplate("{foo}")
                .addConstraintViolation();

        context.addMessageParameter("foo", "snafu");
        context.buildConstraintViolationWithTemplate("{foo}")
                .addConstraintViolation();

        return false;
    }
```



### ValidatorFactory：校验器工厂

```java
public interface ValidatorFactory extends AutoCloseable {

	// 显然，这个接口是最为重要的
	Validator getValidator();
	// 定义一个新的ValidatorContext验证器上下文，并且和Validator关联上
	ValidatorContext usingContext();


	MessageInterpolator getMessageInterpolator();
	TraversableResolver getTraversableResolver();
	ConstraintValidatorFactory getConstraintValidatorFactory();
	ParameterNameProvider getParameterNameProvider();
	ClockProvider getClockProvider();

	public <T> T unwrap(Class<T> type);
	// 复写AutoCloseable的方法
	@Override
	public void close();

}
```

#### HibernateValidatorFactory

#### `ValidatorFactoryImpl`

```java
public class ValidatorFactoryImpl implements HibernateValidatorFactory {
	... // 省略非常多的成员变量
	// 唯一的构造函数，做了非常非常多初始化的事~~~
	public ValidatorFactoryImpl(ConfigurationState configurationState) {
		...
	}

	// 这个或许是最重要的一个方法
	@Override
	public Validator getValidator() {
		return createValidator(
				constraintValidatorManager.getDefaultConstraintValidatorFactory(),
				valueExtractorManager,
				validatorFactoryScopedContext,
				methodValidationConfiguration
		);
	}
	Validator createValidator(ConstraintValidatorFactory constraintValidatorFactory, ValueExtractorManager valueExtractorManager,
			ValidatorFactoryScopedContext validatorFactoryScopedContext, MethodValidationConfiguration methodValidationConfiguration) {
		BeanMetaDataManager beanMetaDataManager = beanMetaDataManagers.computeIfAbsent(
				new BeanMetaDataManagerKey( validatorFactoryScopedContext.getParameterNameProvider(), valueExtractorManager, methodValidationConfiguration ),
				key -> new BeanMetaDataManager(
						constraintHelper,
						executableHelper,
						typeResolutionHelper,
						validatorFactoryScopedContext.getParameterNameProvider(),
						valueExtractorManager,
						validationOrderGenerator,
						buildDataProviders(),
						methodValidationConfiguration
				)
		 );

		return new ValidatorImpl(
				constraintValidatorFactory,
				beanMetaDataManager,
				valueExtractorManager,
				constraintValidatorManager,
				validationOrderGenerator,
				validatorFactoryScopedContext
		);
	}

	@Override
	public MessageInterpolator getMessageInterpolator() {
		return validatorFactoryScopedContext.getMessageInterpolator();
	}
	@Override
	public TraversableResolver getTraversableResolver() {
		return validatorFactoryScopedContext.getTraversableResolver();
	}
	...
	@Override
	public <T> T unwrap(Class<T> type) {
		if ( type.isAssignableFrom( HibernateValidatorFactory.class ) ) {
			return type.cast( this );
		}
		throw LOG.getTypeNotSupportedForUnwrappingException( type );
	}

	@Override
	public HibernateValidatorContext usingContext() {
		return new ValidatorContextImpl( this );
	}
	...
}
```



## Validator：验证器

**验证Bean实例**

### ExecutableValidator

```java
public interface Validator {

	// 校验作用在此Bean上面的所有约束（所有属性、方法、构造器的所有约束）
	// groups可以指定只使用某个group，默认是Defualt的group嘛~
	<T> Set<ConstraintViolation<T>> validate(T object, Class<?>... groups);

	// 上面太过于粗暴。这里是校验这个Bean上 某个具体的属性~
	<T> Set<ConstraintViolation<T>> validateProperty(T object, String propertyName, Class<?>... groups);
	// 这个就更加精确了，具体的属性的具体value值都要校验
	<T> Set<ConstraintViolation<T>> validateValue(Class<T> beanType, String propertyName, Object value, Class<?>... groups);
	
	// 返回描述Bean约束的描述符对象，此对象和ConstraintDescriptor关联
	// 并且还持有PropertyDescriptor和ConstructorDescriptor等等~
	BeanDescriptor getConstraintsForClass(Class<?> clazz);
	<T> T unwrap(Class<T> type);

	// 返回用于验证方法和构造函数的参数和返回值的协定。
	// 不巧的是：ValidatorImpl实现了Validator, ExecutableValidator这两个接口
	ExecutableValidator forExecutables();
}

// ================关于ExecutableValidator接口================

// 它用于验证 方法 和 构造函数 的 **参数和返回值**。
public interface ExecutableValidator {

	// 验证方法的入参
	<T> Set<ConstraintViolation<T>> validateParameters(T object, Method method, Object[] parameterValues, Class<?>... groups);
	// 验证方法的返回值
	<T> Set<ConstraintViolation<T>> validateReturnValue(T object, Method method,Object returnValue, Class<?>... groups);

	// 不解释~~~~~~~~~~~~
	<T> Set<ConstraintViolation<T>> validateConstructorParameters(Constructor<? extends T> constructor,Object[] parameterValues,Class<?>... groups);
	<T> Set<ConstraintViolation<T>> validateConstructorReturnValue(Constructor<? extends T> constructor, T createdObject, Class<?>... groups);
}

```



### `ValidatorImpl`

```java
public class ValidatorImpl implements Validator, ExecutableValidator {
	private static final Collection<Class<?>> DEFAULT_GROUPS = Collections.<Class<?>>singletonList( Default.class );

	// 分组Group校验的顺序问题
	// 若依赖于校验顺序，可用使用@GroupSequence注解来控制Group顺序
	private final transient ValidationOrderGenerator validationOrderGenerator;
	private final ConstraintValidatorFactory constraintValidatorFactory;
	...

	// 唯一构造函数~
	public ValidatorImpl(ConstraintValidatorFactory constraintValidatorFactory, BeanMetaDataManager beanMetaDataManager,
			ValueExtractorManager valueExtractorManager, ConstraintValidatorManager constraintValidatorManager,
			ValidationOrderGenerator validationOrderGenerator, ValidatorFactoryScopedContext validatorFactoryScopedContext) {
		this.constraintValidatorFactory = constraintValidatorFactory;
		this.beanMetaDataManager = beanMetaDataManager;
		this.valueExtractorManager = valueExtractorManager;
		this.constraintValidatorManager = constraintValidatorManager;
		this.validationOrderGenerator = validationOrderGenerator;
		this.validatorScopedContext = new ValidatorScopedContext( validatorFactoryScopedContext );
		this.traversableResolver = validatorFactoryScopedContext.getTraversableResolver();
		this.constraintValidatorInitializationContext = validatorFactoryScopedContext.getConstraintValidatorInitializationContext();
	}


	@Override
	public final <T> Set<ConstraintViolation<T>> validate(T object, Class<?>... groups) {
		sanityCheckGroups( groups ); // groups里面的内容不能有null

		@SuppressWarnings("unchecked")
		Class<T> rootBeanClass = (Class<T>) object.getClass();
		BeanMetaData<T> rootBeanMetaData = beanMetaDataManager.getBeanMetaData( rootBeanClass );

		// 若没有约束存在，直接返回~
		if ( !rootBeanMetaData.hasConstraints() ) {
			return Collections.emptySet();
		}

		// ValidationContext这个实体类里面的属性极其多~  持有各种组件的引用
		ValidationContext<T> validationContext = getValidationContextBuilder().forValidate( rootBeanMetaData, object );
		// 找到这个分组们的（带有顺序）
		ValidationOrder validationOrder = determineGroupValidationOrder( groups );

		// ValueContext一个实例用于收集所有相关信息，以验证单个类、属性或方法调用。
		ValueContext<?, Object> valueContext = ValueContext.getLocalExecutionContext(validatorScopedContext.getParameterNameProvider(), object,
				validationContext.getRootBeanMetaData(), PathImpl.createRootPath()
		);

		// 此方法传入的信息量非常大~~ 验证上下文、值上下文，验证器
		// 返回的是失败的消息对象：ConstraintViolation  它是被存储在ValidationContext里的~~~~
		return validateInContext( validationContext, valueContext, validationOrder );
	}

	... // 省略validateProperty和validateValue
	// 下面实现`ExecutableValidator`的相关方法  它的四个接口方法无非就是两个方法：validateParameters和validateReturnValue 略

	//	beanMetaDataManager.getBeanMetaData( clazz )还是相对比较重要的
	@Override
	public final BeanDescriptor getConstraintsForClass(Class<?> clazz) {
		return beanMetaDataManager.getBeanMetaData( clazz ).getBeanDescriptor();
	}

	@Override
	public final <T> T unwrap(Class<T> type) {
		if ( type.isAssignableFrom( Validator.class ) ) {
			return type.cast( this );
		}
		throw LOG.getTypeNotSupportedForUnwrappingException( type );
	}

	// 返回自己即可。因为它是可以校验方法入参、返回值等等的
	@Override
	public ExecutableValidator forExecutables() {
		return this;
	}
	... // 省略所有的私有方法们~
}
```









































