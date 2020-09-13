

# @EnableAsync

> 1. 异步方法建议尽量处理耗时的任务，或者是处理不希望阻断主流程的任务（比如异步记录操作日志）
> 2. 尽量为`@Async`准备一个专门的线程池来提高效率减少开销，而不要用Spring默认的`SimpleAsyncTaskExecutor`，它不是一个真正的线程池

@Async注解一般用在方法上，如果用在类上，那么这个类所有的方法都是异步执行的；
@Async可以放在任何方法上，哪怕你是private的（若是同类调用，请务必注意注解失效的情况~~~）
所使用的@Async注解方法的类对象应该是Spring容器管理的bean对象
@Async可以放在接口处（或者接口方法上）。但是只有使用的是JDK的动态代理时才有效，CGLIB会失效。因此建议：统一写在实现类的方法上
需要注解来开启异步注解的支持
若你希望得到异步调用的返回值，请你的返回值用Futrue变量包装起来

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AsyncConfigurationSelector.class)
public @interface EnableAsync {

	 //默认情况下，要开启异步操作，要在相应的方法或者类上加上@Async注解或者EJB3.1规范下的@Asynchronous注解。
	 //这个属性使得开发人员可以自己设置开启异步操作的注解(可谓非常的人性化了，但是大多情况下用Spring的就足够了)
	Class<? extends Annotation> annotation() default Annotation.class;
	// true表示启用CGLIB代理
	boolean proxyTargetClass() default false;
	
	// 代理方式：默认是PROXY  采用Spring的动态代理（含JDK动态代理和CGLIB）
	// 若改为：AdviceMode.ASPECTJ表示使用AspectJ静态代理方式。
	// 它能够解决同类内方法调用不走代理对象的问题，但是一般情况下都不建议这么去做，不要修改这个参数值
	AdviceMode mode() default AdviceMode.PROXY;
	// 直接定义：它的执行顺序（因为可能有多个@EnableXXX）
	int order() default Ordered.LOWEST_PRECEDENCE;
}

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Async {

	// May be used to determine the target executor to be used when executing this method
	// 意思是这个value值是用来指定执行器的（写入执行器BeanName即可采用特定的执行器去执行此方法）
	String value() default "";

}
```

### AsyncConfigurationSelector

#### AdviceModeImportSelector

```java
public class AsyncConfigurationSelector extends AdviceModeImportSelector<EnableAsync> {

	// 这类 我也不知道在哪？是用于支持AspectJ这种静态代理Mode的,忽略吧~~~~
	private static final String ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME =
			"org.springframework.scheduling.aspectj.AspectJAsyncConfiguration";

	@Override
	@Nullable
	public String[] selectImports(AdviceMode adviceMode) {
		// 这里AdviceMode 进行不同的处理，从而向Spring容器注入了不同的Bean~~~
		switch (adviceMode) {
			// 大多数情况下都走这里，ProxyAsyncConfiguration会被注入到Bean容器里面~~~
			case PROXY:
				return new String[] { ProxyAsyncConfiguration.class.getName() };
			case ASPECTJ:
				return new String[] { ASYNC_EXECUTION_ASPECT_CONFIGURATION_CLASS_NAME };
			default:
				return null;
		}
	}

}

//父类 AdviceModeImportSelector：抽象实现支持了AdviceMode，并且支持通用的@EnableXXX模式。
//@since 3.1  它是一个`ImportSelector`
public abstract class AdviceModeImportSelector<A extends Annotation> implements ImportSelector {

	// 默认都叫mode
	public static final String DEFAULT_ADVICE_MODE_ATTRIBUTE_NAME = "mode";
	// 显然也允许子类覆盖此方法
	protected String getAdviceModeAttributeName() {
		return DEFAULT_ADVICE_MODE_ATTRIBUTE_NAME;
	}
	
	// importingClassMetadata：注解的信息
	@Override
	public final String[] selectImports(AnnotationMetadata importingClassMetadata) {
		// 这里泛型，拿到泛型类型~~~
		Class<?> annType = GenericTypeResolver.resolveTypeArgument(getClass(), AdviceModeImportSelector.class);
		Assert.state(annType != null, "Unresolvable type argument for AdviceModeImportSelector");
		
		// 根据类型，拿到该类型的这个注解，然后转换为AnnotationAttributes
		AnnotationAttributes attributes = AnnotationConfigUtils.attributesFor(importingClassMetadata, annType);
		if (attributes == null) {
			throw new IllegalArgumentException(String.format( "@%s is not present annType.getSimpleName(), importingClassMetadata.getClassName()));
		}

		// 拿到AdviceMode，最终交给子类，让她自己去实现  决定导入哪个Bean吧
		AdviceMode adviceMode = attributes.getEnum(this.getAdviceModeAttributeName());
		String[] imports = selectImports(adviceMode);
		if (imports == null) {
			throw new IllegalArgumentException(String.format("Unknown AdviceMode: '%s'", adviceMode));
		}
		return imports;
	}

	// 子类去实现  具体导入哪个Bean
	@Nullable
	protected abstract String[] selectImports(AdviceMode adviceMode);
}
```

### ProxyAsyncConfiguration

```java
// 它是一个配置类，角色为ROLE_INFRASTRUCTURE  框架自用的Bean类型
@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyAsyncConfiguration extends AbstractAsyncConfiguration {

	// 注入AsyncAnnotationBeanPostProcessor，它是个BeanPostProcessor
	@Bean(name = TaskManagementConfigUtils.ASYNC_ANNOTATION_PROCESSOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public AsyncAnnotationBeanPostProcessor asyncAdvisor() {
		Assert.notNull(this.enableAsync, "@EnableAsync annotation metadata was not injected");
		AsyncAnnotationBeanPostProcessor bpp = new AsyncAnnotationBeanPostProcessor();
		
		// customAsyncAnnotation：自定义的注解类型
		// AnnotationUtils.getDefaultValue(EnableAsync.class, "annotation") 为拿到该注解该字段的默认值
		Class<? extends Annotation> customAsyncAnnotation = this.enableAsync.getClass("annotation");
	
		// 相当于如果你指定了AsyncAnnotationType,那就set进去吧
		if (customAsyncAnnotation != AnnotationUtils.getDefaultValue(EnableAsync.class, "annotation")) {
			bpp.setAsyncAnnotationType(customAsyncAnnotation);
		}
		
		// 只有自定义了AsyncConfigurer的实现类，自定义了一个线程执行器，这里才会有值
		if (this.executor != null) {
			bpp.setExecutor(this.executor);
		}
		// 同上，异步线程异常的处理器~~~~~
		if (this.exceptionHandler != null) {
			bpp.setExceptionHandler(this.exceptionHandler);
		}
		
		// 这两个参数，就不多说了。
		// 可以看到，order属性值，最终决定的是BeanPostProcessor的执行顺序的
		bpp.setProxyTargetClass(this.enableAsync.getBoolean("proxyTargetClass"));
		bpp.setOrder(this.enableAsync.<Integer>getNumber("order"));
		return bpp;
	}

}

// 它的父类：
@Configuration
public abstract class AbstractAsyncConfiguration implements ImportAware {

	// 此注解@EnableAsync的元信息
	protected AnnotationAttributes enableAsync;
	// 异步线程池
	protected Executor executor;
	// 异步异常的处理器
	protected AsyncUncaughtExceptionHandler exceptionHandler;

	@Override
	public void setImportMetadata(AnnotationMetadata importMetadata) {
		// 拿到@EnableAsync注解的元数据信息~~~
		this.enableAsync = AnnotationAttributes.fromMap(importMetadata.getAnnotationAttributes(EnableAsync.class.getName(), false));
		if (this.enableAsync == null) {
			throw new IllegalArgumentException("@EnableAsync is not present on importing class " + importMetadata.getClassName());
		}
	}

	/**
	 * Collect any {@link AsyncConfigurer} beans through autowiring.
	 */
	 // doc说得很明白。它会把所有的`AsyncConfigurer`的实现类都搜集进来，然后进行类似属性的合并
	 // 备注  虽然这里用的是Collection 但是AsyncConfigurer的实现类只允许有一个
	@Autowired(required = false)
	void setConfigurers(Collection<AsyncConfigurer> configurers) {
      
		if (CollectionUtils.isEmpty(configurers)) {
			return;
		}
      	//AsyncConfigurer用来配置线程池配置以及异常处理器，而且在Spring环境中最多只能有一个
      	//在这里我们知道了，如果想要自己去配置线程池，只需要实现AsyncConfigurer接口，并且不可以在Spring环境中有多个实现AsyncConfigurer的类。
		if (configurers.size() > 1) {
			throw new IllegalStateException("Only one AsyncConfigurer may exist");
		}
		// 拿到唯一的AsyncConfigurer ，然后赋值~~~~   默认的请参照这个类：AsyncConfigurerSupport（它并不会被加入进Spring容器里）
		AsyncConfigurer configurer = configurers.iterator().next();
		this.executor = configurer.getAsyncExecutor();
		this.exceptionHandler = configurer.getAsyncUncaughtExceptionHandler();
	}

}

```



### AbstractAdvisingBeanPostProcessor

主要处理`AdvisingBean`，也就是处理`Advisor`和Bean的关系

#### postProcessAfterInitialization (代理)

有代理添加 自己的 Advisor ，这里是 AsyncAnnotationAdvisor，未代理而需代理，则通过ProxyFactory 创建代理 并添加  自己的 Advisor

```java
// 它继承自，ProxyProcessorSupport，说明它也拥有Aop的通用配置
public abstract class AbstractAdvisingBeanPostProcessor extends ProxyProcessorSupport implements BeanPostProcessor {

	@Nullable
	protected Advisor advisor;
	protected boolean beforeExistingAdvisors = false;
	
	// 缓存合格的Bean们
	private final Map<Class<?>, Boolean> eligibleBeans = new ConcurrentHashMap<>(256);
  
	// 当遇到一个pre-object的时候，是否把该processor所持有得advisor放在现有的增强器们之前执行
	// 默认是false，会放在最后一个位置上的
	public void setBeforeExistingAdvisors(boolean beforeExistingAdvisors) {
		this.beforeExistingAdvisors = beforeExistingAdvisors;
	}
	
	// 不处理
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) {
		return bean;
	}

	// Bean已经实例化、初始化完成之后执行。
	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) {
		// 忽略AopInfrastructureBean的Bean，并且如果没有advisor也会忽略不处理~~~~~
		if (bean instanceof AopInfrastructureBean || this.advisor == null) {
			// Ignore AOP infrastructure such as scoped proxies.
			return bean;
		}
		
		// 如果这个Bean已经被代理过了（比如已经被AOP切中了），那本处就无需再重复创建代理了嘛
		// 直接向里面添加advisor就成了
		if (bean instanceof Advised) {
			Advised advised = (Advised) bean;
			// 注意此advised不能是已经被冻结了的。且源对象必须是Eligible合格的
			if (!advised.isFrozen() && isEligible(AopUtils.getTargetClass(bean))) {
				// Add our local Advisor to the existing proxy's Advisor chain...
				// 把自己持有的这个advisor放在首位（如果beforeExistingAdvisors=true）
				if (this.beforeExistingAdvisors) {
					advised.addAdvisor(0, this.advisor);
				}
				// 否则就是尾部位置
				else {
					advised.addAdvisor(this.advisor);
				}
				// 最终直接返回即可，因为已经没有必要再创建一次代理对象了
				return bean;
			}
		}

		// 如果这个Bean事合格的（此方法下面有解释）   这个时候是没有被代理过的
		if (isEligible(bean, beanName)) {
			// 以当前的配置，创建一个ProxyFactory 
			ProxyFactory proxyFactory = prepareProxyFactory(bean, beanName);
			// 如果不是使用CGLIB常见代理，那就去分析出它所实现的接口们  然后放进ProxyFactory 里去
			if (!proxyFactory.isProxyTargetClass()) {
				evaluateProxyInterfaces(bean.getClass(), proxyFactory);
			}
			
			// 切面就是当前持有得advisor
			proxyFactory.addAdvisor(this.advisor);
			// 留给子类，自己还可以对proxyFactory进行自定义~~~~~
			customizeProxyFactory(proxyFactory);
			// 最终返回这个代理对象~~~~~
			return proxyFactory.getProxy(getProxyClassLoader());
		}

		// No async proxy needed.（相当于没有做任何的代理处理,返回原对象）
		return bean;
	}
	
	// 检查这个Bean是否是合格的
	protected boolean isEligible(Object bean, String beanName) {
		return isEligible(bean.getClass());
	}
	protected boolean isEligible(Class<?> targetClass) {
		// 如果已经被缓存着了，那肯定靠谱啊
		Boolean eligible = this.eligibleBeans.get(targetClass);
		if (eligible != null) {
			return eligible;
		}
		// 如果没有切面（就相当于没有给配置增强器，那铁定是不合格的）
		if (this.advisor == null) {
			return false;
		}
	
		// 这个重要了：看看这个advisor是否能够切入进targetClass这个类，能够切入进取的也是合格的
		eligible = AopUtils.canApply(this.advisor, targetClass);
		this.eligibleBeans.put(targetClass, eligible);
		return eligible;
	}

	// 子类可以复写。比如`AbstractBeanFactoryAwareAdvisingPostProcessor`就复写了这个方法~~~
	protected ProxyFactory prepareProxyFactory(Object bean, String beanName) {
		ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.copyFrom(this);
		proxyFactory.setTarget(bean);
		return proxyFactory;
	}

	// 子类复写~
	protected void customizeProxyFactory(ProxyFactory proxyFactory) {
	}

}

```



### AbstractBeanFactoryAwareAdvisingPostProcessor

### AsyncAnnotationBeanPostProcessor

```java
public class AsyncAnnotationBeanPostProcessor extends AbstractBeanFactoryAwareAdvisingPostProcessor {

	// 建议换成AsyncExecutionAspectSupport.DEFAULT_TASK_EXECUTOR_BEAN_NAME  这样语意更加的清晰些
	public static final String DEFAULT_TASK_EXECUTOR_BEAN_NAME = AnnotationAsyncExecutionInterceptor.DEFAULT_TASK_EXECUTOR_BEAN_NAME;

	// 注解类型
	@Nullable
	private Class<? extends Annotation> asyncAnnotationType;
	// 异步的执行器
	@Nullable
	private Executor executor;
	// 异步异常处理器
	@Nullable
	private AsyncUncaughtExceptionHandler exceptionHandler;

	// 此处特别注意：这里设置为true，也就是说@Async的Advisor会放在首位
	public AsyncAnnotationBeanPostProcessor() {
		setBeforeExistingAdvisors(true);
	}

	// 可以设定需要扫描哪些注解类型。默认只扫描@Async以及`javax.ejb.Asynchronous`这个注解
	public void setAsyncAnnotationType(Class<? extends Annotation> asyncAnnotationType) {
		Assert.notNull(asyncAnnotationType, "'asyncAnnotationType' must not be null");
		this.asyncAnnotationType = asyncAnnotationType;
	}

	// 如果没有指定。那就将执行全局得默认查找。在上下文中查找唯一的`TaskExecutor`类型的Bean，或者一个名称为`taskExecutor`的Executor
	// 当然，如果上面途径都没找到。那就会采用一个默认的任务池
	public void setExecutor(Executor executor) {
		this.executor = executor;
	}
	public void setExceptionHandler(AsyncUncaughtExceptionHandler exceptionHandler) {
		this.exceptionHandler = exceptionHandler;
	}


	// 重写了父类的方法。然后下面：自己new了一个AsyncAnnotationAdvisor ，传入executor和exceptionHandler
	// 并且最终this.advisor = advisor 
	// 因此可议看出：AsyncAnnotationAdvisor 才是重点了。它定义了它的匹配情况~~~~
	@Override
	public void setBeanFactory(BeanFactory beanFactory) {
		super.setBeanFactory(beanFactory);

		AsyncAnnotationAdvisor advisor = new AsyncAnnotationAdvisor(this.executor, this.exceptionHandler);
		if (this.asyncAnnotationType != null) {
			advisor.setAsyncAnnotationType(this.asyncAnnotationType);
		}
		advisor.setBeanFactory(beanFactory);
		this.advisor = advisor;
	}

}

```

#### AsyncAnnotationAdvisor

它是一个`AnnotationMatchingPointcut`，因此是为注解来匹配的

```java
public class AsyncAnnotationAdvisor extends AbstractPointcutAdvisor implements BeanFactoryAware {

	private AsyncUncaughtExceptionHandler exceptionHandler;
	// 增强器
	private Advice advice;
	// 切点
	private Pointcut pointcut;

	// 两个都为null，那就是都会采用默认的方案
	public AsyncAnnotationAdvisor() {
		this(null, null);
	}

	// 创建一个AsyncAnnotationAdvisor实例，可以自己指定Executor 和 AsyncUncaughtExceptionHandler 
	@SuppressWarnings("unchecked")
	public AsyncAnnotationAdvisor(@Nullable Executor executor, @Nullable AsyncUncaughtExceptionHandler exceptionHandler) {
		// 这里List长度选择2，应为绝大部分情况下只会支持这两种@Async和@Asynchronous
		Set<Class<? extends Annotation>> asyncAnnotationTypes = new LinkedHashSet<>(2);
		asyncAnnotationTypes.add(Async.class);
		try {
			asyncAnnotationTypes.add((Class<? extends Annotation>)
					ClassUtils.forName("javax.ejb.Asynchronous", AsyncAnnotationAdvisor.class.getClassLoader()));
		}
		catch (ClassNotFoundException ex) {
			// If EJB 3.1 API not present, simply ignore.
		}

		if (exceptionHandler != null) {
			this.exceptionHandler = exceptionHandler;
		}
		// 若没指定，那就使用默认的SimpleAsyncUncaughtExceptionHandler（它仅仅是输出了一句日志而已）
		else {
			this.exceptionHandler = new SimpleAsyncUncaughtExceptionHandler();
		}
		
		// 这两个方法是重点，下面会重点介绍
		this.advice = buildAdvice(executor, this.exceptionHandler);
		this.pointcut = buildPointcut(asyncAnnotationTypes);
	}


	// 如果set了Executor,advice会重新构建。
	public void setTaskExecutor(Executor executor) {
		this.advice = buildAdvice(executor, this.exceptionHandler);
	}

	// 这里注意：如果你自己指定了注解类型。那么将不再扫描其余两个默认的注解，因此pointcut也就需要重新构建了
	public void setAsyncAnnotationType(Class<? extends Annotation> asyncAnnotationType) {
		Assert.notNull(asyncAnnotationType, "'asyncAnnotationType' must not be null");
		Set<Class<? extends Annotation>> asyncAnnotationTypes = new HashSet<>();
		asyncAnnotationTypes.add(asyncAnnotationType);
		this.pointcut = buildPointcut(asyncAnnotationTypes);
	}
	
	// 如果这个advice也实现了BeanFactoryAware，那就也把BeanFactory放进去
	@Override
	public void setBeanFactory(BeanFactory beanFactory) {
		if (this.advice instanceof BeanFactoryAware) {
			((BeanFactoryAware) this.advice).setBeanFactory(beanFactory);
		}
	}

	@Override
	public Advice getAdvice() {
		return this.advice;
	}
	@Override
	public Pointcut getPointcut() {
		return this.pointcut;
	}

	// 这个最终又是委托给`AnnotationAsyncExecutionInterceptor`，它是一个具体的增强器，有着核心内容
	protected Advice buildAdvice(@Nullable Executor executor, AsyncUncaughtExceptionHandler exceptionHandler) {
		return new AnnotationAsyncExecutionInterceptor(executor, exceptionHandler);
	}

	// Calculate a pointcut for the given async annotation types, if any
	protected Pointcut buildPointcut(Set<Class<? extends Annotation>> asyncAnnotationTypes) {
		// 采用一个组合切面：ComposablePointcut （因为可能需要支持多个注解嘛）
		ComposablePointcut result = null;
		for (Class<? extends Annotation> asyncAnnotationType : asyncAnnotationTypes) {
			// 这里为何new出来两个AnnotationMatchingPointcut？？？？？？
			// 第一个：类匹配（只需要类上面有这个注解，所有的方法都匹配）this.methodMatcher = MethodMatcher.TRUE;
			// 第二个：方法匹配。所有的类都可议。但是只有方法上有这个注解才会匹配上
			Pointcut cpc = new AnnotationMatchingPointcut(asyncAnnotationType, true);
			Pointcut mpc = new AnnotationMatchingPointcut(null, asyncAnnotationType, true);
			if (result == null) {
				result = new ComposablePointcut(cpc);
			}
			else {
				result.union(cpc);
			}
			// 最终的结果都是取值为并集的~~~~~~~
			result = result.union(mpc);
		}
		//  最后一个处理厉害了：也就是说你啥类型都木有的情况下，是匹配所有类的所有方法~~~
		return (result != null ? result : Pointcut.TRUE);
	}

}
```

##### buildPointcut



##### buildAdvice

默认是支持@Asycn以及EJB的那个异步注解的。但是最终的增强行为，委托给了`AnnotationAsyncExecutionInterceptor`



#### AsyncExecutionAspectSupport

用来支持处理异步线程执行器

```java
public abstract class AsyncExecutionAspectSupport implements BeanFactoryAware {

	// 这是备选的。如果找到多个类型为TaskExecutor的Bean，才会备选的再用这个名称去找的~~~
	public static final String DEFAULT_TASK_EXECUTOR_BEAN_NAME = "taskExecutor";
	// 缓存~~~AsyncTaskExecutor是TaskExecutor的子接口
	// 从这可以看出：不同的方法，对应的异步执行器还不一样咯~~~~~~
	private final Map<Method, AsyncTaskExecutor> executors = new ConcurrentHashMap<>(16);
	// 默认的线程执行器
	@Nullable
	private volatile Executor defaultExecutor;
	// 异步异常处理器
	private AsyncUncaughtExceptionHandler exceptionHandler;
	// Bean工厂
	@Nullable
	private BeanFactory beanFactory;

	public AsyncExecutionAspectSupport(@Nullable Executor defaultExecutor) {
		this(defaultExecutor, new SimpleAsyncUncaughtExceptionHandler());
	}
	public AsyncExecutionAspectSupport(@Nullable Executor defaultExecutor, AsyncUncaughtExceptionHandler exceptionHandler) {
		this.defaultExecutor = defaultExecutor;
		this.exceptionHandler = exceptionHandler;
	}
	public void setExecutor(Executor defaultExecutor) {
		this.defaultExecutor = defaultExecutor;
	}
	public void setExceptionHandler(AsyncUncaughtExceptionHandler exceptionHandler) {
		this.exceptionHandler = exceptionHandler;
	}
	@Override
	public void setBeanFactory(BeanFactory beanFactory) {
		this.beanFactory = beanFactory;
	}
	
	// 该方法是找到一个异步执行器，去执行这个方法~~~~~~
	@Nullable
	protected AsyncTaskExecutor determineAsyncExecutor(Method method) {
		// 如果缓存中能够找到该方法对应的执行器，就立马返回了
		AsyncTaskExecutor executor = this.executors.get(method);
		if (executor == null) {
			Executor targetExecutor;
			
			// 抽象方法：`AnnotationAsyncExecutionInterceptor`有实现。就是@Async注解的value值
			String qualifier = getExecutorQualifier(method);
			// 现在知道@Async直接的value值的作用了吧。就是制定执行此方法的执行器的（容器内执行器的Bean的名称）
			// 当然有可能为null。注意此处是支持@Qualified注解标注在类上来区分Bean的
			// 注意：此处targetExecutor仍然可能为null
			if (StringUtils.hasLength(qualifier)) {
				targetExecutor = findQualifiedExecutor(this.beanFactory, qualifier);
			}
			
			// 注解没有指定value值，那就去找默认的执行器
			else {
				targetExecutor = this.defaultExecutor;
				if (targetExecutor == null) {
					// 去找getDefaultExecutor()~~~
					synchronized (this.executors) {
						if (this.defaultExecutor == null) {
							this.defaultExecutor = getDefaultExecutor(this.beanFactory);
						}
						targetExecutor = this.defaultExecutor;
					}
				}
			}
			
			// 若还未null，那就返回null吧
			if (targetExecutor == null) {
				return null;
			}
			
			// 把targetExecutor 包装成一个AsyncTaskExecutor返回，并且缓存起来。
			// TaskExecutorAdapter就是AsyncListenableTaskExecutor的一个实现类
			executor = (targetExecutor instanceof AsyncListenableTaskExecutor ?
					(AsyncListenableTaskExecutor) targetExecutor : new TaskExecutorAdapter(targetExecutor));
			this.executors.put(method, executor);
		}
		return executor;
	}

	// 子类去复写此方法。也就是拿到对应的key，从而方便找bean吧（执行器）
	@Nullable
	protected abstract String getExecutorQualifier(Method method);

	// @since 4.2.6  也就是根据
	@Nullable
	protected Executor findQualifiedExecutor(@Nullable BeanFactory beanFactory, String qualifier) {
		if (beanFactory == null) {
			throw new IllegalStateException("BeanFactory must be set on " + getClass().getSimpleName() +
					" to access qualified executor '" + qualifier + "'");
		}
		return BeanFactoryAnnotationUtils.qualifiedBeanOfType(beanFactory, Executor.class, qualifier);
	}

	// @since 4.2.6 
	// Retrieve or build a default executor for this advice instance  检索或者创建一个默认的executor 
	@Nullable
	protected Executor getDefaultExecutor(@Nullable BeanFactory beanFactory) {
		if (beanFactory != null) {
			// 这个处理很有意思，它是用用的try catch的技巧去处理的
			try {
				// 如果容器内存在唯一的TaskExecutor（子类），就直接返回了
				return beanFactory.getBean(TaskExecutor.class);
			}
			catch (NoUniqueBeanDefinitionException ex) {
				// 这是出现了多个TaskExecutor类型的话，那就按照名字去拿  `taskExecutor`且是Executor类型
				try {
					return beanFactory.getBean(DEFAULT_TASK_EXECUTOR_BEAN_NAME, Executor.class);
				}
				// 如果再没有找到，也不要报错，而是接下来创建一个默认的处理器
				// 这里输出一个info信息
				catch (NoSuchBeanDefinitionException ex2) {
				}
			}
			catch (NoSuchBeanDefinitionException ex) {
				try {
					return beanFactory.getBean(DEFAULT_TASK_EXECUTOR_BEAN_NAME, Executor.class);
				}
				catch (NoSuchBeanDefinitionException ex2) {
				}
				// 这里还没有获取到，就放弃。 用本地默认的executor吧~~~
				// 子类可以去复写此方法，发现为null的话可议给一个默认值~~~~比如`AsyncExecutionInterceptor`默认给的就是`SimpleAsyncTaskExecutor`作为执行器的
				// Giving up -> either using local default executor or none at all...
			}
		}
		return null;
	}

	//Delegate for actually executing the given task with the chosen executor
	// 用选定的执行者实际执行给定任务的委托
	@Nullable
	protected Object doSubmit(Callable<Object> task, AsyncTaskExecutor executor, Class<?> returnType) {
		// 根据不同的返回值类型，来采用不同的方案去异步执行，但是执行器都是executor
		if (CompletableFuture.class.isAssignableFrom(returnType)) {
			return CompletableFuture.supplyAsync(() -> {
				try {
					return task.call();
				}
				catch (Throwable ex) {
					throw new CompletionException(ex);
				}
			}, executor);
		}
		// ListenableFuture接口继承自Future  是Spring自己扩展的一个接口。
		// 同样的AsyncListenableTaskExecutor也是Spring扩展自AsyncTaskExecutor的
		else if (ListenableFuture.class.isAssignableFrom(returnType)) {
			return ((AsyncListenableTaskExecutor) executor).submitListenable(task);
		}
		
		// 普通的submit
		else if (Future.class.isAssignableFrom(returnType)) {
			return executor.submit(task);
		}
		// 没有返回值的情况下  也用sumitt提交，按时返回null
		else {
			executor.submit(task);
			return null;
		}
	}

	// 处理错误
	protected void handleError(Throwable ex, Method method, Object... params) throws Exception {
	 	// 如果方法的返回值类型是Future,就rethrowException，表示直接throw出去
		if (Future.class.isAssignableFrom(method.getReturnType())) {
			ReflectionUtils.rethrowException(ex);
		} else {
			// Could not transmit the exception to the caller with default executor
			try {
				this.exceptionHandler.handleUncaughtException(ex, method, params);
			} catch (Throwable ex2) {
				logger.error("Exception handler for async method '" + method.toGenericString() +
						"' threw unexpected exception itself", ex2);
			}
		}
	}

}

```



##### （调用）AsyncExecutionInterceptor#invoke（MethodInterceptor）

处理异步的执行,可用于 仅基于xml方式

```java
// 他继承自AsyncExecutionAspectSupport 来处理异步方法的处理，同时是个MethodInterceptor，来增强复合条件的方法
public class AsyncExecutionInterceptor extends AsyncExecutionAspectSupport implements MethodInterceptor, Ordered {
	...
	// 显然这个方法它直接返回null，因为XML配置嘛~~~~
	@Override
	@Nullable
	protected String getExecutorQualifier(Method method) {
		return null;
	}

	// 这个厉害了。如果父类返回的defaultExecutor 为null，那就new一个SimpleAsyncTaskExecutor作为默认的执行器
	@Override
	@Nullable
	protected Executor getDefaultExecutor(@Nullable BeanFactory beanFactory) {
		Executor defaultExecutor = super.getDefaultExecutor(beanFactory);
		return (defaultExecutor != null ? defaultExecutor : new SimpleAsyncTaskExecutor());
	}
	// 最高优先级  希望的是最优先执行（XML配置就是这种优先级）
	@Override
	public int getOrder() {
		return Ordered.HIGHEST_PRECEDENCE;
	}

// 最重要的当然是这个invoke方法
	@Override
	@Nullable
	public Object invoke(final MethodInvocation invocation) throws Throwable {
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);
		// 注意：此处是getMostSpecificMethod  拿到最终要执行的那个方法
		Method specificMethod = ClassUtils.getMostSpecificMethod(invocation.getMethod(), targetClass);
		// 桥接方法~~~~~~~~~~~~~~
		final Method userDeclaredMethod = BridgeMethodResolver.findBridgedMethod(specificMethod);

		// determine一个用于执行此方法的异步执行器
		AsyncTaskExecutor executor = determineAsyncExecutor(userDeclaredMethod);
		if (executor == null) {
			throw new IllegalStateException(
					"No executor specified and no default executor set on AsyncExecutionInterceptor either");
		}

		// 构造一个任务，Callable(此处不采用Runable，因为需要返回值)
		Callable<Object> task = () -> {
			try {
				// result就是返回值
				Object result = invocation.proceed();
				// 注意此处的处理~~~~  相当于如果不是Future类型，就返回null了
				if (result instanceof Future) {
					return ((Future<?>) result).get();
				}
			}
			// 处理执行时可能产生的异常~~~~~~
			catch (ExecutionException ex) {
				handleError(ex.getCause(), userDeclaredMethod, invocation.getArguments());
			}
			catch (Throwable ex) {
				handleError(ex, userDeclaredMethod, invocation.getArguments());
			}
			return null;
		};
		
		// 提交任务~~~~invocation.getMethod().getReturnType()为返回值类型
		return doSubmit(task, executor, invocation.getMethod().getReturnType());
	}
	
}
```

> SimpleAsyncTaskExecutor：异步执行用户任务的SimpleAsyncTaskExecutor。每次执行客户提交给它的任务时，它会启动新的线程，并允许开发者控制并发线程的上限（concurrencyLimit），从而起到一定的资源节流作用。默认时，concurrencyLimit取值为-1，即**不启用**资源节流
> 所以它不是真的线程池，这个类不重用线程，每次调用都会创建一个新的线程（因此建议我们在使用@Aysnc的时候，自己配置一个线程池，节约资源）6

###### AnnotationAsyncExecutionInterceptor：@Async拦截器

```java
// 它继承自AsyncExecutionInterceptor ，只复写了一个方法
public class AnnotationAsyncExecutionInterceptor extends AsyncExecutionInterceptor {
	...
	// 由此可以见它就是去拿到@Async的value值。以方法的为准，其次是类上面的
	// 备注：发现这里是不支持EJB的@Asynchronous注解的，它是不能指定执行器的
	@Override
	@Nullable
	protected String getExecutorQualifier(Method method) {
		// Maintainer's note: changes made here should also be made in
		// AnnotationAsyncExecutionAspect#getExecutorQualifier
		Async async = AnnotatedElementUtils.findMergedAnnotation(method, Async.class);
		if (async == null) {
			async = AnnotatedElementUtils.findMergedAnnotation(method.getDeclaringClass(), Async.class);
		}
		return (async != null ? async.value() : null);
	}
}
```



#### @Async注解失效得原因

`代理失效`问题



#### AsyncConfigurer

```java
@EnableAsync //对应的@Enable注解，最好写在属于自己的配置文件上，保持内聚性
@Configuration
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10); //核心线程数
        executor.setMaxPoolSize(20);  //最大线程数
        executor.setQueueCapacity(1000); //队列大小
        executor.setKeepAliveSeconds(300); //线程最大空闲时间
        executor.setThreadNamePrefix("fsx-Executor-"); ////指定用于新创建的线程名称的前缀。
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy()); // 拒绝策略（一共四种，此处省略）

	    // 这一步千万不能忘了，否则报错： java.lang.IllegalStateException: ThreadPoolTaskExecutor not initialized
        executor.initialize();
        return executor;
    }

    // 异常处理器：当然你也可以自定义的，这里我就这么简单写了~~~
    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return new SimpleAsyncUncaughtExceptionHandler();
    }
}
```

