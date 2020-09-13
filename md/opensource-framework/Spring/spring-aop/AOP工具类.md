

## AOP 工具类

### AopUtils

#### findAdvisorsThatCanApply

#### canApply

#### invokeJoinpointUsingReflection

```java
// 关于Pointcut、Advisor的
public abstract class AopUtils {
	...
	// 判断一个切入点能否匹配一个指定的类型   显然默认是不支持引介匹配
	public static boolean canApply(Pointcut pc, Class<?> targetClass) {
		return canApply(pc, targetClass, false);
	}

	//判断一个切入点能否匹配一个指定的类型，是否支持引介匹配；
	public static boolean canApply(Pointcut pc, Class<?> targetClass, boolean hasIntroductions) {
		Assert.notNull(pc, "Pointcut must not be null");
		if (!pc.getClassFilter().matches(targetClass)) {
			return false;
		}

		MethodMatcher methodMatcher = pc.getMethodMatcher();
		// 如果是恒等式true，那就毫无疑问全匹配楼
		if (methodMatcher == MethodMatcher.TRUE) {
			return true;
		}
		//IntroductionAwareMethodMatcher 是MethodMatcher的子类  增加了匹配方法：
		// boolean matches(Method method, @Nullable Class<?> targetClass, boolean hasIntroductions);
		IntroductionAwareMethodMatcher introductionAwareMethodMatcher = null;
		if (methodMatcher instanceof IntroductionAwareMethodMatcher) {
			introductionAwareMethodMatcher = (IntroductionAwareMethodMatcher) methodMatcher;
		}

		Set<Class<?>> classes = new LinkedHashSet<>();
		if (!Proxy.isProxyClass(targetClass)) {
			classes.add(ClassUtils.getUserClass(targetClass));
		}
		classes.addAll(ClassUtils.getAllInterfacesForClassAsSet(targetClass));

		for (Class<?> clazz : classes) {
			Method[] methods = ReflectionUtils.getAllDeclaredMethods(clazz);
			for (Method method : methods) {
				if (introductionAwareMethodMatcher != null ?
						introductionAwareMethodMatcher.matches(method, targetClass, hasIntroductions) :
						methodMatcher.matches(method, targetClass)) {
					return true;
				}
			}
		}

		return false;
	}

	//判断一个建议(advisor)能否匹配一个指定的类型
	public static boolean canApply(Advisor advisor, Class<?> targetClass) {
		return canApply(advisor, targetClass, false);
	}
	// 同上，是否包含引介匹配
	// 这个原理很简单，判断是否是IntroductionAdvisor还是PointcutAdvisor，然后做对应判断即可
	public static boolean canApply(Advisor advisor, Class<?> targetClass, boolean hasIntroductions) {
		if (advisor instanceof IntroductionAdvisor) {
			return ((IntroductionAdvisor) advisor).getClassFilter().matches(targetClass);
		}
		else if (advisor instanceof PointcutAdvisor) {
			PointcutAdvisor pca = (PointcutAdvisor) advisor;
			return canApply(pca.getPointcut(), targetClass, hasIntroductions);
		}
		else {
			// It doesn't have a pointcut so we assume it applies.
			// 默认情况都是匹配的
			return true;
		}
	}

	//在给定的一组建议(advisor)中，返回能够匹配指定类型的建议者列表
	// 这个方法很总要：在自动代理创建器AbstractAdvisorAutoProxyCreator中，都是这样筛选的能够匹配上此类型的Advisor们~~~
	public static List<Advisor> findAdvisorsThatCanApply(List<Advisor> candidateAdvisors, Class<?> clazz) {
		if (candidateAdvisors.isEmpty()) {
			return candidateAdvisors;
		}
		List<Advisor> eligibleAdvisors = new LinkedList<>();
	
		// 这里遍历了candidateAdvisors两次   注意这个技巧
		// 第一次遍历：找出所有的IntroductionAdvisor 类型的，并且canApply的Advisor们
		for (Advisor candidate : candidateAdvisors) {
			if (candidate instanceof IntroductionAdvisor && canApply(candidate, clazz)) {
				eligibleAdvisors.add(candidate);
			}
		}
		
		// hasIntroductions：如果上面不为空，这里就是true  然后继续便利  吧canApply的找出来
		boolean hasIntroductions = !eligibleAdvisors.isEmpty();
		for (Advisor candidate : candidateAdvisors) {
			if (candidate instanceof IntroductionAdvisor) {
				// already processed
				continue;
			}
			if (canApply(candidate, clazz, hasIntroductions)) {
				eligibleAdvisors.add(candidate);
			}
		}
		return eligibleAdvisors;
	}

	//通过反射，执行一个目标方法；这个方法其实就是method.invoke方法的更完善的方法，指在target对象上，使用args参数列表执行method
	@Nullable
	public static Object invokeJoinpointUsingReflection(@Nullable Object target, Method method, Object[] args)
			throws Throwable {
		try {
			ReflectionUtils.makeAccessible(method);
			return method.invoke(target, args);
		} catch (InvocationTargetException ex) { 
			throw ex.getTargetException();
		} catch (IllegalArgumentException ex) {
			throw new AopInvocationException("AOP configuration seems to be invalid: tried calling method [" + method + "] on target [" + target + "]", ex);
		} catch (IllegalAccessException ex) {
			throw new AopInvocationException("Could not access method [" + method + "]", ex);
		}
	}
}
```



### AopConfigUtils

#### registerAutoProxyCreatorIfNecessary

#### registerAspectJAnnotationAutoProxyCreatorIfNecessary

#### registerOrEscalateApcAsRequired

```java
public abstract class AopConfigUtils {

	// 这是注册自动代理创建器，默认的BeanName（若想覆盖，需要使用这个BeanName）
	public static final String AUTO_PROXY_CREATOR_BEAN_NAME = "org.springframework.aop.config.internalAutoProxyCreator";

	// 按照升级顺序 存储自动代理创建器（注意这里是升级的顺序 一个比一个强的）
	private static final List<Class<?>> APC_PRIORITY_LIST = new ArrayList<>();
	static {
		APC_PRIORITY_LIST.add(InfrastructureAdvisorAutoProxyCreator.class);
		APC_PRIORITY_LIST.add(AspectJAwareAdvisorAutoProxyCreator.class);
		APC_PRIORITY_LIST.add(AnnotationAwareAspectJAutoProxyCreator.class);
	}

	// 这两个：注册的是`InfrastructureAdvisorAutoProxyCreator`  
	// 调用处为：AutoProxyRegistrar#registerBeanDefinitions（它是一个ImportBeanDefinitionRegistrar实现类） 
	// 而AutoProxyRegistrar使用处为CachingConfigurationSelector，和`@EnableCaching`注解有关
	// 其次就是AopNamespaceUtils有点用，这个下面再分析
	@Nullable
	public static BeanDefinition registerAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry,
			@Nullable Object source) {
		return registerOrEscalateApcAsRequired(InfrastructureAdvisorAutoProxyCreator.class, registry, source);
	}
	@Nullable
	public static BeanDefinition registerAspectJAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
		return registerAspectJAutoProxyCreatorIfNecessary(registry, null);
	}
	
	// 下面这两个是注入：AspectJAwareAdvisorAutoProxyCreator
	// 目前没有地方默认调用~~~~和Aop的xml配置方案有关的
	@Nullable
	public static BeanDefinition registerAspectJAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
		return registerAspectJAutoProxyCreatorIfNecessary(registry, null);
	}
	@Nullable
	public static BeanDefinition registerAspectJAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry,
			@Nullable Object source) {
		return registerOrEscalateApcAsRequired(AspectJAwareAdvisorAutoProxyCreator.class, registry, source);
	}

	// 这个就是最常用的，注入的是：AnnotationAwareAspectJAutoProxyCreator  注解驱动的自动代理创建器
	// `@EnableAspectJAutoProxy`注入进来的就是它了
	@Nullable
	public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
		return registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry, null);
	}
	@Nullable
	public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry,
			@Nullable Object source) {
		return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
	}


	// 这两个方法，很显然，就是处理注解的两个属性值
	// proxyTargetClass：true表示强制使用CGLIB的动态代理
	// exposeProxy：true暴露当前代理对象到线程上绑定
	// 最终都会放到自动代理创建器得BeanDefinition 里面去~~~创建代理的时候会用到此属性值
	public static void forceAutoProxyCreatorToUseClassProxying(BeanDefinitionRegistry registry) {
		if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
			BeanDefinition definition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
			definition.getPropertyValues().add("proxyTargetClass", Boolean.TRUE);
		}
	}
	public static void forceAutoProxyCreatorToExposeProxy(BeanDefinitionRegistry registry) {
		if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
			BeanDefinition definition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
			definition.getPropertyValues().add("exposeProxy", Boolean.TRUE);
		}
	}

	//========上面的注册自动代理创建器IfNecessary之类的方法，最终都是调用了这里========
	@Nullable
	private static BeanDefinition registerOrEscalateApcAsRequired(Class<?> cls, BeanDefinitionRegistry registry,
			@Nullable Object source) {
		
		// 这里相当于，如果你自己定义了一个名称为这个的自动代理创建器，那也是ok的（需要注意的是使用工厂方法@Bean的方式定义，这里是会报错的）
		if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
			BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
			
			// 若使用@Bean的方法定义，这里apcDefinition.getBeanClassName()就是null，导致后面的findPriorityForClass(apcDefinition.getBeanClassName())就会报错~~~~~~~  需要特别的注意哦~~~~~
			if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
				int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
				int requiredPriority = findPriorityForClass(cls);
				if (currentPriority < requiredPriority) {
					apcDefinition.setBeanClassName(cls.getName());
				}
			}
			return null;
		}

		// 绝大部分情况下都会走这里，new一个Bean定义信息出来，然后order属性值为HIGHEST_PRECEDENCE
		// role是：ROLE_INFRASTRUCTURE属于Spring框架自己使用的Bean
		// BeanName为：AUTO_PROXY_CREATOR_BEAN_NAME
		RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
		beanDefinition.setSource(source);
		beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
		beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
		return beanDefinition;
	}

	...findPriorityForClass的逻辑省略
}
```



### AopNamespaceUtils:处理Spring AOP命名空间的工具类

### AopProxyUtils#completeProxiedInterfaces

```java
public abstract class AopProxyUtils {
	...
	//很牛逼的方法来了，判断一个advised真正需要代理的目标接口列表 
	// 这个方法在getProxy()上都有应用，非常的重要。核心实现事下面的这个私有方法
	public static Class<?>[] completeProxiedInterfaces(AdvisedSupport advised) {
		return completeProxiedInterfaces(advised, false);
	}
	// 很显然发现，最终代理出来的对象，除了实现了自己的接口外，还实现了额外的接口，如：
	// SpringProxy、Advised、DecoratingProxy等三个接口（备注：CGLIB代理未实现DecoratingProxy接口）
	static Class<?>[] completeProxiedInterfaces(AdvisedSupport advised, boolean decoratingProxy) {
		Class<?>[] specifiedInterfaces = advised.getProxiedInterfaces();
		if (specifiedInterfaces.length == 0) {
			// No user-specified interfaces: check whether target class is an interface.
			Class<?> targetClass = advised.getTargetClass();
			if (targetClass != null) {
				if (targetClass.isInterface()) {
					advised.setInterfaces(targetClass);
				}
				else if (Proxy.isProxyClass(targetClass)) {
					advised.setInterfaces(targetClass.getInterfaces());
				}
				specifiedInterfaces = advised.getProxiedInterfaces();
			}
		}
		boolean addSpringProxy = !advised.isInterfaceProxied(SpringProxy.class);
		boolean addAdvised = !advised.isOpaque() && !advised.isInterfaceProxied(Advised.class);
		boolean addDecoratingProxy = (decoratingProxy && !advised.isInterfaceProxied(DecoratingProxy.class));
		int nonUserIfcCount = 0;
		if (addSpringProxy) {
			nonUserIfcCount++;
		}
		if (addAdvised) {
			nonUserIfcCount++;
		}
		if (addDecoratingProxy) {
			nonUserIfcCount++;
		}
		Class<?>[] proxiedInterfaces = new Class<?>[specifiedInterfaces.length + nonUserIfcCount];
		System.arraycopy(specifiedInterfaces, 0, proxiedInterfaces, 0, specifiedInterfaces.length);
		int index = specifiedInterfaces.length;
		if (addSpringProxy) {
			proxiedInterfaces[index] = SpringProxy.class;
			index++;
		}
		if (addAdvised) {
			proxiedInterfaces[index] = Advised.class;
			index++;
		}
		if (addDecoratingProxy) {
			proxiedInterfaces[index] = DecoratingProxy.class;
		}
		return proxiedInterfaces;
	}

	//该方法用于获取一个代理对象中的用户定义的接口，即非（Advised接口体系）之外的其他接口
	public static Class<?>[] proxiedUserInterfaces(Object proxy) {
		Class<?>[] proxyInterfaces = proxy.getClass().getInterfaces();
		int nonUserIfcCount = 0;
		if (proxy instanceof SpringProxy) {
			nonUserIfcCount++;
		}
		if (proxy instanceof Advised) {
			nonUserIfcCount++;
		}
		if (proxy instanceof DecoratingProxy) {
			nonUserIfcCount++;
		}
		Class<?>[] userInterfaces = new Class<?>[proxyInterfaces.length - nonUserIfcCount];
		System.arraycopy(proxyInterfaces, 0, userInterfaces, 0, userInterfaces.length);
		Assert.notEmpty(userInterfaces, "JDK proxy must implement one or more interfaces");
		return userInterfaces;
	}

	//判断两个（即将）代理出来的对象是否相同
	public static boolean equalsInProxy(AdvisedSupport a, AdvisedSupport b) {
		return (a == b ||
				(equalsProxiedInterfaces(a, b) && equalsAdvisors(a, b) && a.getTargetSource().equals(b.getTargetSource())));
	}
	// 判断它哥俩的接口是否相同
	public static boolean equalsProxiedInterfaces(AdvisedSupport a, AdvisedSupport b) {
		return Arrays.equals(a.getProxiedInterfaces(), b.getProxiedInterfaces());
	}
	// 判断它哥俩的增强器（Advisor）是否相同
	public static boolean equalsAdvisors(AdvisedSupport a, AdvisedSupport b) {
		return Arrays.equals(a.getAdvisors(), b.getAdvisors());
	}
}

```

### AopContext

这个工具类就更简单，它代表AOP的上下文。主要是提供我们访问上下文中当前AOP对象的快速方法。



### AutoProxyUtils

为自动代理组件准备的工具类。主要用于框架内部使用（`AbstractAutoProxyCreator`）

```java
public abstract class AutoProxyUtils {

	// org.springframework.aop.framework.autoproxy.AutoProxyUtils.preserveTargetClass
	// preserve：保护的  保留的
	// determine：查明  测定
	public static final String PRESERVE_TARGET_CLASS_ATTRIBUTE =
			Conventions.getQualifiedAttributeName(AutoProxyUtils.class, "preserveTargetClass");
	// org.springframework.aop.framework.autoproxy.AutoProxyUtils.originalTargetClass
	public static final String ORIGINAL_TARGET_CLASS_ATTRIBUTE =
			Conventions.getQualifiedAttributeName(AutoProxyUtils.class, "originalTargetClass");


	// 判断该beanName是否应该被代理
	// `AbstractAutoProxyCreator`里就有判断是否能够被代理。  如果能够被代理，那就采用CGLIB的代理方式了
	// 往里setAttr,目前只有`ConfigurationClassPostProcessor`对config配置类进行增强的时候
	// 
	public static boolean shouldProxyTargetClass(ConfigurableListableBeanFactory beanFactory, @Nullable String beanName) {
		// 容器内存在这个Bean，并且这个Bean的定义信息里面属性值`PRESERVE_TARGET_CLASS_ATTRIBUTE`必须是true才行  说明才能被代理
		if (beanName != null && beanFactory.containsBeanDefinition(beanName)) {
			BeanDefinition bd = beanFactory.getBeanDefinition(beanName);
			return Boolean.TRUE.equals(bd.getAttribute(PRESERVE_TARGET_CLASS_ATTRIBUTE));
		}
		return false;
	}

	// 看看这个Bean定义的TargetClass
	// 如果Bean定义信息里面有ORIGINAL_TARGET_CLASS_ATTRIBUTE这个字段，那就不用getType()了
	// 以及ScopedProxyUtils创建和Scope有关的代理类的时候，其余地方都不会设置此属性
	@Nullable
	public static Class<?> determineTargetClass(ConfigurableListableBeanFactory beanFactory, @Nullable String beanName) {
		if (beanName == null) {
			return null;
		}
		if (beanFactory.containsBeanDefinition(beanName)) {
			BeanDefinition bd = beanFactory.getMergedBeanDefinition(beanName);
			Class<?> targetClass = (Class<?>) bd.getAttribute(ORIGINAL_TARGET_CLASS_ATTRIBUTE);
			if (targetClass != null) {
				return targetClass;
			}
		}
		return beanFactory.getType(beanName);
	}

	//Expose the given target class for the specified bean, if possible
	// 就是在它的Bean定义信息里面，设置一个ORIGINAL_TARGET_CLASS_ATTRIBUTE属性，然后吧targetClass类型放进去
	// 也是子啊`AbstractAutoProxyCreator`创建代理的时候会这只进去的
	static void exposeTargetClass(ConfigurableListableBeanFactory beanFactory, @Nullable String beanName,
			Class<?> targetClass) {

		if (beanName != null && beanFactory.containsBeanDefinition(beanName)) {
			beanFactory.getMergedBeanDefinition(beanName).setAttribute(ORIGINAL_TARGET_CLASS_ATTRIBUTE, targetClass);
		}
	}

}
```



###  AutoProxyUtils

```java
public abstract class AutoProxyUtils {

	// org.springframework.aop.framework.autoproxy.AutoProxyUtils.preserveTargetClass
	// preserve：保护的  保留的
	// determine：查明  测定
	public static final String PRESERVE_TARGET_CLASS_ATTRIBUTE =
			Conventions.getQualifiedAttributeName(AutoProxyUtils.class, "preserveTargetClass");
	// org.springframework.aop.framework.autoproxy.AutoProxyUtils.originalTargetClass
	public static final String ORIGINAL_TARGET_CLASS_ATTRIBUTE =
			Conventions.getQualifiedAttributeName(AutoProxyUtils.class, "originalTargetClass");


	// 判断该beanName是否应该被代理
	// `AbstractAutoProxyCreator`里就有判断是否能够被代理。  如果能够被代理，那就采用CGLIB的代理方式了
	// 往里setAttr,目前只有`ConfigurationClassPostProcessor`对config配置类进行增强的时候
	// 
	public static boolean shouldProxyTargetClass(ConfigurableListableBeanFactory beanFactory, @Nullable String beanName) {
		// 容器内存在这个Bean，并且这个Bean的定义信息里面属性值`PRESERVE_TARGET_CLASS_ATTRIBUTE`必须是true才行  说明才能被代理
		if (beanName != null && beanFactory.containsBeanDefinition(beanName)) {
			BeanDefinition bd = beanFactory.getBeanDefinition(beanName);
			return Boolean.TRUE.equals(bd.getAttribute(PRESERVE_TARGET_CLASS_ATTRIBUTE));
		}
		return false;
	}

	// 看看这个Bean定义的TargetClass
	// 如果Bean定义信息里面有ORIGINAL_TARGET_CLASS_ATTRIBUTE这个字段，那就不用getType()了
	// 以及ScopedProxyUtils创建和Scope有关的代理类的时候，其余地方都不会设置此属性
	@Nullable
	public static Class<?> determineTargetClass(ConfigurableListableBeanFactory beanFactory, @Nullable String beanName) {
		if (beanName == null) {
			return null;
		}
		if (beanFactory.containsBeanDefinition(beanName)) {
			BeanDefinition bd = beanFactory.getMergedBeanDefinition(beanName);
			Class<?> targetClass = (Class<?>) bd.getAttribute(ORIGINAL_TARGET_CLASS_ATTRIBUTE);
			if (targetClass != null) {
				return targetClass;
			}
		}
		return beanFactory.getType(beanName);
	}

	//Expose the given target class for the specified bean, if possible
	// 就是在它的Bean定义信息里面，设置一个ORIGINAL_TARGET_CLASS_ATTRIBUTE属性，然后吧targetClass类型放进去
	// 也是子啊`AbstractAutoProxyCreator`创建代理的时候会这只进去的
	static void exposeTargetClass(ConfigurableListableBeanFactory beanFactory, @Nullable String beanName,
			Class<?> targetClass) {

		if (beanName != null && beanFactory.containsBeanDefinition(beanName)) {
			beanFactory.getMergedBeanDefinition(beanName).setAttribute(ORIGINAL_TARGET_CLASS_ATTRIBUTE, targetClass);
		}
	}

}
```



### AspectJProxyUtils

```java
public abstract class AspectJProxyUtils {

	// 判断，该Advisor是否是AspectJ的的增强器
	private static boolean isAspectJAdvice(Advisor advisor) {
		return (advisor instanceof InstantiationModelAwarePointcutAdvisor ||
				advisor.getAdvice() instanceof AbstractAspectJAdvice ||
				(advisor instanceof PointcutAdvisor &&
						 ((PointcutAdvisor) advisor).getPointcut() instanceof AspectJExpressionPointcut));
	}

	// 只提供这么一个公共方法，但是这个方法都还是非常重要的。 Capable：有能力的
	// 它在自动代理创建器`AspectJAwareAdvisorAutoProxyCreator#extendAdvisors`方法中有调用（重要~~~）
	// 在AspectJProxyFactory#addAdvisorsFromAspectInstanceFactory方法中也有调用
	// 它的作用：只要发现有AspectJ的Advisor存在，并且advisors还不包含有ExposeInvocationInterceptor.ADVISOR  那就在第一个位置上调添加一个ExposeInvocationInterceptor.ADVISOR
	// 这个`ExposeInvocationInterceptor.ADVISOR`的作用：就是获取到当前的currentInvocation，也是使用的ThreadLocal
	public static boolean makeAdvisorChainAspectJCapableIfNecessary(List<Advisor> advisors) {
		// Don't add advisors to an empty list; may indicate that proxying is just not required
		if (!advisors.isEmpty()) {
			boolean foundAspectJAdvice = false;
			for (Advisor advisor : advisors) {
				// Be careful not to get the Advice without a guard, as
				// this might eagerly instantiate a non-singleton AspectJ aspect
				if (isAspectJAdvice(advisor)) {
					foundAspectJAdvice = true;
				}
			}
			if (foundAspectJAdvice && !advisors.contains(ExposeInvocationInterceptor.ADVISOR)) {
				advisors.add(0, ExposeInvocationInterceptor.ADVISOR);
				return true;
			}
		}
		return false;
	}

}
```















