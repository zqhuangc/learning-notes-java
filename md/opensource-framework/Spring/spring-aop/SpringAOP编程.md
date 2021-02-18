## Aopalliance

* org.aopalliance.aop包
  * Advice：通知的标记接口。实现可以是任意类型，比如下面的Interceptor
  * AspectException：所有的AOP框架产生异常的父类。它是个RuntimeException
* org.aopalliance.intercept包
  * Interceptor：它继承自Advice，它通过拦截器得方式实现通知的效果(也属于标记接口)
  * MethodInterceptor：具体的接口。拦截方法 （Spring提供了非常多的具体实现类）
  * ConstructorInterceptor：具体接口。拦截构造器 （Spring并没有提供实现类）
  * Joinpoint：AOP运行时的连接点（顶层接口）
  * Invocation：继承自Joinpoint。 表示执行，提供了Object[] getArguments()来获取执行所需的参数
  * MethodInvocation：（和MethodInterceptor对应，它的invoke方法入参就是它）表示一个和方法有关的执行器。提供方法Method getMethod() （Spring提供了唯一（唯二）实现类：ProxyMethodInvocation）
  * ConstructorInvocation：和构造器有关。Constructor<?> getConstructor(); (Spring没有提供任何实现类)

#### Joinpoint

先看另一个`org.aspectj.lang.JoinPoint`：该对象封装了SpringAop中切面方法的信息,在切面方法中添加 JoinPoint 参数，可以很方便的获得更多信息。（一般用于@Aspect 标注的切面的方法入参里），它的API很多，常用的有下面几个：

1. Signature getSignature(); ：封装了署名信息的对象,在该对象中可以获取到目标方法名,所属类的Class等信息
2. Object[] getArgs();：传入目标方法的参数们
3. Object getTarget();：被代理的对象（目标对象）
4. Object getThis();：该代理对象

> 备注：`ProceedingJoinPoint`对象是`JoinPoint`的子接口,该对象只用在@Around的切面方法中

* `org.aopalliance.intercept.Joinpoint`

```java
// 此接口表示运行时的连接点（AOP术语）  （和aspectj里的连接点意思有点像）
public interface Joinpoint {

	// 执行此拦截点，并进入到下一个连接点
	Object proceed() throws Throwable;
	// 返回保存当前连接点静态部分【的对象】。  这里一般指的target
	Object getThis();
	// 返回此静态连接点  一般就为当前的Method(至少目前的唯一实现是MethodInvocation,所以连接点得静态部分肯定就是本方法喽)
	AccessibleObject getStaticPart();
}
```

#### Invocation

`org.aopalliance.intercept.Invocation`

```java
// 此接口表示程序中的调用~
// 该调用是一个可以被拦截器拦截的连接点
public interface Invocation extends Joinpoint {
	// 获得参数们。比如方法的入参们
	Object[] getArguments();
}
```

#### MethodInvocation

 `org.aopalliance.intercept.MethodInvocation`

`MethodInvocation`作为`aopalliance`里提供的最底层接口

```java
// 方法调用时，对这部分进行描述
public interface MethodInvocation extends Invocation {
	// 返回正在被调用得方法~~~  返回的是当前Method对象。
	// 此时，效果同父类的AccessibleObject getStaticPart() 这个方法
	Method getMethod();
}
```



##### ProxyMethodInvocation

`org.springframework.aop.ProxyMethodInvocation`

Spring自己也定义了一个接口，来进行扩展和统一管理：`ProxyMethodInvocation`

```java
// 这是Spring提供的对MethodInvocation 的一个扩展。
// 它允许访问  方法被调用的代理对象以及其它相关信息
public interface ProxyMethodInvocation extends MethodInvocation {
	// 返回代理对象
	Object getProxy();
	
	// 克隆一个，使用的Object得clone方法
	MethodInvocation invocableClone();
	MethodInvocation invocableClone(Object... arguments);

	// 设置参数  增强器、通知们执行的时候可能会用到
	void setArguments(Object... arguments);
	// 添加一些属性kv。这些kv并不会用于AOP框架内，而是保存下来给特殊的一些拦截器实用
	void setUserAttribute(String key, @Nullable Object value);
	@Nullable
	Object getUserAttribute(String key);
}
```



##### ReflectiveMethodInvocation

> InterceptorAndDynamicMethodMatcher
>
> 从这里我们需要注意到的是：`ProxyMethodInvocation`（`ReflectiveMethodInvocation`）是代理执行的入口。然后内部会把所有的 增强器 都拿出来 `递归执行`（比如前置通知，就在目标方法之前执行） `**这就实现了指定次序的链式调用**`

`JdkDynamicAopProxy`最终`执行时`候new出来的执行对象

```java
public class ReflectiveMethodInvocation implements ProxyMethodInvocation, Cloneable {

	protected final Object proxy; // 代理对象
	@Nullable
	protected final Object target; // 目标对象
	protected final Method method; // 被拦截的方法

	protected Object[] arguments = new Object[0];
	@Nullable
	private final Class<?> targetClass;

	@Nullable
	private Map<String, Object> userAttributes;
	protected final List<?> interceptorsAndDynamicMethodMatchers;
	
	// currentInterceptorIndex初始值为 -1 
	private int currentInterceptorIndex = -1;

	// 唯一的构造函数。注意是protected  相当于只能本包内、以及子类可以调用。外部是不能直接初始化的此对象的（显然就是Spring内部使用的类了嘛）
	//invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
	// proxy：代理对象
	// target：目标对象
	// method：被代理的方法
	// args：方法的参数们
	// targetClass：目标方法的Class (target != null ? target.getClass() : null)
	// interceptorsAndDynamicMethodMatchers：拦截链。  this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass)这个方法找出来的
	protected ReflectiveMethodInvocation(
			Object proxy, @Nullable Object target, Method method, @Nullable Object[] arguments,
			@Nullable Class<?> targetClass, List<Object> interceptorsAndDynamicMethodMatchers) {

		this.proxy = proxy;
		this.target = target;
		this.targetClass = targetClass;
		// 找到桥接方法，作为最后执行的方法。至于什么是桥接方法，自行百度关键字：bridge method
		// 桥接方法是 JDK 1.5 引入泛型后，为了使Java的泛型方法生成的字节码和 1.5 版本前的字节码相兼容，由编译器自动生成的方法（子类实现父类的泛型方法时会生成桥接方法）
		this.method = BridgeMethodResolver.findBridgedMethod(method);
		// 对参数进行适配
		this.arguments = AopProxyUtils.adaptArgumentsIfNecessary(method, arguments);
		this.interceptorsAndDynamicMethodMatchers = interceptorsAndDynamicMethodMatchers;
	}

	@Override
	public final Object getProxy() {
		return this.proxy;
	}
	@Override
	@Nullable
	public final Object getThis() {
		return this.target;
	}
	// 此处：getStaticPart返回的就是当前得method
	@Override
	public final AccessibleObject getStaticPart() {
		return this.method;
	}
	// 注意：这里返回的可能是桥接方法哦
	@Override
	public final Method getMethod() {
		return this.method;
	}
	@Override
	public final Object[] getArguments() {
		return this.arguments;
	}
	@Override
	public void setArguments(Object... arguments) {
		this.arguments = arguments;
	}


	// 这里就是核心了，要执行方法、执行通知、都是在此处搞定的
	// 这里面运用 递归调用 的方式，非常具有技巧性
	@Override
	@Nullable
	public Object proceed() throws Throwable {
		//	currentInterceptorIndex初始值为 -1  如果执行到链条的末尾 则直接调用连接点方法 即 直接调用目标方法
		if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
			// 这个方法相当于调用了目标方法~~~下面会分析
			return invokeJoinpoint();
		}

		// 获取集合中的 MethodInterceptor（并且currentInterceptorIndex + 1了哦）
		Object interceptorOrInterceptionAdvice =
				this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);

		//InterceptorAndDynamicMethodMatcher它是Spring内部使用的一个类。很简单，就是把MethodInterceptor实例和MethodMatcher放在了一起。看看在advisor chain里面是否能够匹配上
		if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
			InterceptorAndDynamicMethodMatcher dm =
					(InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
			
			// 去匹配这个拦截器是否适用于这个目标方法  试用就执行拦截器得invoke方法
			if (dm.methodMatcher.matches(this.method, this.targetClass, this.arguments)) {
				return dm.interceptor.invoke(this);
			}
			else {
				// 如果不匹配。就跳过此拦截器，而继续执行下一个拦截器
				// 注意：这里是递归调用  并不是循环调用
				return proceed();
			}
		}
		else {
			// 直接执行此拦截器。说明之前已经匹配好了，只有匹配上的方法才会被拦截进来的
			// 这里传入this就是传入了ReflectiveMethodInvocation，从而形成了一个链条了
			return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
		}
	}
	

	// 其实就是简单的一个：method.invoke(target, args);
	// 子类可以复写此方法，去执行。比如它的唯一子类CglibAopProxy内部类  CglibMethodInvocation就复写了这个方法  它对public的方法做了一个处理（public方法调用MethodProxy.invoke）
	@Nullable
	protected Object invokeJoinpoint() throws Throwable {
		// 此处传入的是target，而不能是proxy，否则进入死循环
		return AopUtils.invokeJoinpointUsingReflection(this.target, this.method, this.arguments);
	}

	@Override
	public MethodInvocation invocableClone() {
		Object[] cloneArguments = this.arguments;
		if (this.arguments.length > 0) {
			// Build an independent copy of the arguments array.
			cloneArguments = new Object[this.arguments.length];
			System.arraycopy(this.arguments, 0, cloneArguments, 0, this.arguments.length);
		}
		return invocableClone(cloneArguments);
	}
	@Override
	public MethodInvocation invocableClone(Object... arguments) {
		if (this.userAttributes == null) {
			this.userAttributes = new HashMap<>();
		}
		try {
			ReflectiveMethodInvocation clone = (ReflectiveMethodInvocation) clone();
			clone.arguments = arguments;
			return clone;
		} catch (CloneNotSupportedException ex) {
			throw new IllegalStateException(
					"Should be able to clone object of type [" + getClass() + "]: " + ex);
		}
	}
	@Override
	public void setUserAttribute(String key, @Nullable Object value) {
		if (value != null) {
			if (this.userAttributes == null) {
				this.userAttributes = new HashMap<>();
			}
			this.userAttributes.put(key, value);
		}
		else {
			if (this.userAttributes != null) {
				this.userAttributes.remove(key);
			}
		}
	}
	@Override
	@Nullable
	public Object getUserAttribute(String key) {
		return (this.userAttributes != null ? this.userAttributes.get(key) : null);
	}
	public Map<String, Object> getUserAttributes() {
		if (this.userAttributes == null) {
			this.userAttributes = new HashMap<>();
		}
		return this.userAttributes;
	}

	@Override
	public String toString() {
		// Don't do toString on target, it may be proxied.
		StringBuilder sb = new StringBuilder("ReflectiveMethodInvocation: ");
		sb.append(this.method).append("; ");
		if (this.target == null) {
			sb.append("target is null");
		}
		else {
			sb.append("target is of class [").append(this.target.getClass().getName()).append(']');
		}
		return sb.toString();
	}

}
```



##### CglibMethodInvocation

```java
private static class CglibMethodInvocation extends ReflectiveMethodInvocation {

		private final MethodProxy methodProxy;

		private final boolean publicMethod;

		public CglibMethodInvocation(Object proxy, @Nullable Object target, Method method,
				Object[] arguments, @Nullable Class<?> targetClass,
				List<Object> interceptorsAndDynamicMethodMatchers, MethodProxy methodProxy) {
			
			// 调用父类的构造  完成基本参数得初始化
			super(proxy, target, method, arguments, targetClass, interceptorsAndDynamicMethodMatchers);
			
			// 自己的个性化参数：
			
			// 这个参数是子类多传的，表示：它是CGLIb拦截的时候的类MethodProxy
			//MethodProxy为生成的代理类对方法的代理引用。cglib生成用来代替Method对象的一个对象，使用MethodProxy比调用JDK自身的Method直接执行方法效率会有提升
			// 它有两个重要的方法：invoke和invokeSuper
			this.methodProxy = methodProxy;
			// 方法是否是public的  对应下面的invoke方法的处理 见下面
			this.publicMethod = Modifier.isPublic(method.getModifiers());
		}

		@Override
		protected Object invokeJoinpoint() throws Throwable {
			// 如果是public的方法，调用methodProxy去执行目标方法
			// 否则直接执行method即可
			if (this.publicMethod) {
				// 此处务必注意的是，传入的是target，而不能是proxy，否则进入死循环
				return this.methodProxy.invoke(this.target, this.arguments);
			} else {
				return super.invokeJoinpoint();
			}
		}
	}
```

#### MethodInterceptor

 `org.aopalliance.intercept.MethodInterceptor`

>  注意：cglib包里也存在一个`MethodInterceptor`，它的主要作用是CGLIB内部使用，一般是和`Enhancer`一起来使用而创建一个动态代理对象。

@AspectJ定义的通知们（增强器们），或者是自己实现的MethodBeforeAdvice、AfterReturningAdvice…(总是都是org.aopalliance.aop.Advice一个通知器)，最终都会被包装成一个org.aopalliance.intercept.MethodInterceptor，最终交给MethodInvocation（其子类ReflectiveMethodInvocation）去执行，它会把你所有的增强器都给执行了，这就是我们面向切面编程的核心思路过程。

AspectJAfterAdvice

#### ConstructorInterceptor

Spring没有提供实现类

## `org.aspectj`包下的几个类（单独导入的Jar包）

#### `AjTypeSystem`：从@Aspect的Class到AjType的工具类

```JAVA
public class AjTypeSystem {
		
		// 每个切面都给缓存上   注意：此处使用的是WeakReference 一定程度上节约内存
		private static Map<Class, WeakReference<AjType>> ajTypes = 
			Collections.synchronizedMap(new WeakHashMap<Class,WeakReference<AjType>>());

		public static <T> AjType<T> getAjType(Class<T> fromClass) {
			WeakReference<AjType> weakRefToAjType =  ajTypes.get(fromClass);
			if (weakRefToAjType!=null) {
				AjType<T> theAjType = weakRefToAjType.get();
				if (theAjType != null) {
					return theAjType;
				} else {
					// 其实只有这一步操作：new AjTypeImpl~~~  AjTypeImpl就相当于代理了Class的很多事情~~~~
					theAjType = new AjTypeImpl<T>(fromClass);
					ajTypes.put(fromClass, new WeakReference<AjType>(theAjType));
					return theAjType;
				}
			}
			// neither key nor value was found
			AjType<T> theAjType =  new AjTypeImpl<T>(fromClass);
			ajTypes.put(fromClass, new WeakReference<AjType>(theAjType));
			return theAjType;
		}
}

```

#### AjType

```java
// 它继承自Java得Type和AnnotatedElement  它自己还提供了非常非常多的方法，基本都是获取元数据的一些方法，等到具体使用到的时候再来看也可以
public interface AjType<T> extends Type, AnnotatedElement {
	...
}
```

##### AjTypeImpl

```java
public class AjTypeImpl<T> implements AjType<T> {
	private static final String ajcMagic = "ajc$";
	// 它真正传进来的，只是这个class，它是一个标注了@Aspect注解的Class类
	private Class<T> clazz;
	
	private Pointcut[] declaredPointcuts = null;
	private Pointcut[] pointcuts = null;
	private Advice[] declaredAdvice = null;
	private Advice[] advice = null;
	private InterTypeMethodDeclaration[] declaredITDMethods = null;
	private InterTypeMethodDeclaration[] itdMethods = null;
	private InterTypeFieldDeclaration[] declaredITDFields = null;
	private InterTypeFieldDeclaration[] itdFields = null;
	private InterTypeConstructorDeclaration[] itdCons = null;
	private InterTypeConstructorDeclaration[] declaredITDCons = null;

	// 唯一的一个构造函数
	public AjTypeImpl(Class<T> fromClass) {
		this.clazz = fromClass;
	}

	// 这个方法有意思的地方在于：它把所有的接口类，都变成AjType类型了
	public AjType<?>[] getInterfaces() {
		Class<?>[] baseInterfaces = clazz.getInterfaces();
		return toAjTypeArray(baseInterfaces);
	}
	private AjType<?>[] toAjTypeArray(Class<?>[] classes) {
		AjType<?>[] ajtypes = new AjType<?>[classes.length];
		for (int i = 0; i < ajtypes.length; i++) {
			ajtypes[i] = AjTypeSystem.getAjType(classes[i]);
		}
		return ajtypes;
	}
	
	// 就是把clazz返回出去
	public Class<T> getJavaClass() {
		return clazz;
	}
	public AjType<? super T> getSupertype() {
		Class<? super T> superclass = clazz.getSuperclass();
		return superclass==null ? null : (AjType<? super T>) new AjTypeImpl(superclass);
	}
	// 判断是否是切面，就看是否有这个注解~~
	public boolean isAspect() {
		return clazz.getAnnotation(Aspect.class) != null;
	}
	
	// 这个方法很重要：PerClause AspectJ切面的表现形式
	// 备注：虽然有这么多(参考这个类PerClauseKind)，但是Spring AOP只支持前三种~~~
	public PerClause getPerClause() {
		if (isAspect()) {
			Aspect aspectAnn = clazz.getAnnotation(Aspect.class);
			String perClause = aspectAnn.value();
			if (perClause.equals("")) {
				// 如果自己没写，但是存在父类的话并且父类是切面，那就以父类的为准~~~~
				if (getSupertype().isAspect()) {
					return getSupertype().getPerClause();
				} 
				
				// 不写默认是单例的,下面的就不一一解释了
				return new PerClauseImpl(PerClauseKind.SINGLETON);
			} else if (perClause.startsWith("perthis(")) {
				return new PointcutBasedPerClauseImpl(PerClauseKind.PERTHIS,perClause.substring("perthis(".length(),perClause.length() - 1));
			} else if (perClause.startsWith("pertarget(")) {
				return new PointcutBasedPerClauseImpl(PerClauseKind.PERTARGET,perClause.substring("pertarget(".length(),perClause.length() - 1));				
			} else if (perClause.startsWith("percflow(")) {
				return new PointcutBasedPerClauseImpl(PerClauseKind.PERCFLOW,perClause.substring("percflow(".length(),perClause.length() - 1));								
			} else if (perClause.startsWith("percflowbelow(")) {
				return new PointcutBasedPerClauseImpl(PerClauseKind.PERCFLOWBELOW,perClause.substring("percflowbelow(".length(),perClause.length() - 1));
			} else if (perClause.startsWith("pertypewithin")) {
				return new TypePatternBasedPerClauseImpl(PerClauseKind.PERTYPEWITHIN,perClause.substring("pertypewithin(".length(),perClause.length() - 1));				
			} else {
				throw new IllegalStateException("Per-clause not recognized: " + perClause);
			}
		} else {
			return null;
		}
	}

	public AjType<?>[] getAjTypes() {
		Class[] classes = clazz.getClasses();
		return toAjTypeArray(classes);
	}
	
	public Field getDeclaredField(String name) throws NoSuchFieldException {
		Field f =  clazz.getDeclaredField(name);
		if (f.getName().startsWith(ajcMagic)) throw new NoSuchFieldException(name);
		return f;
	}

	// 这个有点意思：表示标注了@Before、@Around注解的并不算真的方法了，不会给与返回了
	public Method[] getMethods() {
		Method[] methods = clazz.getMethods();
		List<Method> filteredMethods = new ArrayList<Method>();
		for (Method method : methods) {
			if (isReallyAMethod(method)) filteredMethods.add(method);
		}
		Method[] ret = new Method[filteredMethods.size()];
		filteredMethods.toArray(ret);
		return ret;
	}
	private boolean isReallyAMethod(Method method) {
		if (method.getName().startsWith(ajcMagic)) return false;
		if (method.getAnnotations().length==0) return true;
		if (method.isAnnotationPresent(org.aspectj.lang.annotation.Pointcut.class)) return false;
		if (method.isAnnotationPresent(Before.class)) return false;
		if (method.isAnnotationPresent(After.class)) return false;
		if (method.isAnnotationPresent(AfterReturning.class)) return false;
		if (method.isAnnotationPresent(AfterThrowing.class)) return false;
		if (method.isAnnotationPresent(Around.class)) return false;
		return true;
	}

	// 拿到所有的Pointcut方法  并且保存缓存起来
	public Pointcut[] getDeclaredPointcuts() {
		if (declaredPointcuts != null) return declaredPointcuts;
		List<Pointcut> pointcuts = new ArrayList<Pointcut>();
		Method[] methods = clazz.getDeclaredMethods();
		for (Method method : methods) {
			Pointcut pc = asPointcut(method);
			if (pc != null) pointcuts.add(pc);
		}
		Pointcut[] ret = new Pointcut[pointcuts.size()];
		pointcuts.toArray(ret);
		declaredPointcuts = ret;
		return ret;
	}
	// 标注有org.aspectj.lang.annotation.Pointcut这个注解的方法。  相当于解析这个注解吧，最终包装成一个PointcutImpl
	// 主义：Spring-aop也有个接口Pointcut，这里也有一个Pointcut接口  注意别弄混了
	private Pointcut asPointcut(Method method) {
		org.aspectj.lang.annotation.Pointcut pcAnn = method.getAnnotation(org.aspectj.lang.annotation.Pointcut.class);
		if (pcAnn != null) {
			String name = method.getName();
			if (name.startsWith(ajcMagic)) {
				// extract real name
				int nameStart = name.indexOf("$$");
				name = name.substring(nameStart +2,name.length());
				int nextDollar = name.indexOf("$");
				if (nextDollar != -1) name = name.substring(0,nextDollar);
			}
			return new PointcutImpl(name,pcAnn.value(),method,AjTypeSystem.getAjType(method.getDeclaringClass()),pcAnn.argNames());
		} else {
			return null;
		}
	}

	// 最终返回的对象为AdviceImpl实现类
	public Advice[] getDeclaredAdvice(AdviceKind... ofType) { ... }
	public Advice[] getAdvice(AdviceKind... ofType) { ... }
	private void initDeclaredAdvice() {
		Method[] methods = clazz.getDeclaredMethods();
		List<Advice> adviceList = new ArrayList<Advice>();
		for (Method method : methods) {
			Advice advice = asAdvice(method);
			if (advice != null) adviceList.add(advice);
		}
		declaredAdvice = new Advice[adviceList.size()];
		adviceList.toArray(declaredAdvice);
	}
	// 标注了各个注解的 做对应的处理
	private Advice asAdvice(Method method) {
		if (method.getAnnotations().length == 0) return null;
		Before beforeAnn = method.getAnnotation(Before.class);
		if (beforeAnn != null) return new AdviceImpl(method,beforeAnn.value(),AdviceKind.BEFORE);
		After afterAnn = method.getAnnotation(After.class);
		if (afterAnn != null) return new AdviceImpl(method,afterAnn.value(),AdviceKind.AFTER);
		AfterReturning afterReturningAnn = method.getAnnotation(AfterReturning.class);
		if (afterReturningAnn != null) {
			// 如果没有自己指定注解pointcut()的值，那就取值为value的值吧~~~
			String pcExpr = afterReturningAnn.pointcut();
			if (pcExpr.equals("")) pcExpr = afterReturningAnn.value();
			
			// 会把方法的返回值放进去、下同。。。   这就是@After和@AfterReturning的区别的原理
			// 它可议自定义自己的切点表达式咯
			return new AdviceImpl(method,pcExpr,AdviceKind.AFTER_RETURNING,afterReturningAnn.returning());
		}
		AfterThrowing afterThrowingAnn = method.getAnnotation(AfterThrowing.class);
		if (afterThrowingAnn != null) {
			String pcExpr = afterThrowingAnn.pointcut();
			if (pcExpr == null) pcExpr = afterThrowingAnn.value();
			return new AdviceImpl(method,pcExpr,AdviceKind.AFTER_THROWING,afterThrowingAnn.throwing());
		}
		Around aroundAnn = method.getAnnotation(Around.class);
		if (aroundAnn != null) return new AdviceImpl(method,aroundAnn.value(),AdviceKind.AROUND);
		return null;
	}

	// 必须不是切面才行哦~~~~
	public boolean isLocalClass() {
		return clazz.isLocalClass() && !isAspect();
	}
	public boolean isMemberClass() {
		return clazz.isMemberClass() && !isAspect();
	}
	// 内部类也是能作为切面哒  哈哈
	public boolean isMemberAspect() {
		return clazz.isMemberClass() && isAspect();
	}

	public String toString() { return getName(); }
}
```







## AOP 基础类

| 类                            | 主要作用                                                     |
| ----------------------------- | ------------------------------------------------------------ |
| AdvisedSupport                | 注册被代理目标对象，通知，和需要代理的接口                   |
| ProxyCreatorSupport           | 注册和触发监听器，借助 DefaultAopProxyFactory 获取代理       |
| DefaultAdvisorAdapterRegistry | 将 Advice 包装成 Advisor（DefaultPointAdvisor）；借助 AdvisorAdapter，将 Advisor 包装成 MethodInterceptor |
| DefaultAdvisorChainFactory    | 借助  DefaultAdvisorAdapterRegistry 将 Advisor 集合转换成 MethodInterceptor 集合 |
| JdkDynamicAopProxy            | 生成 jdk 动态代理                                            |
| CglibAopProxy                 | 生成 cglib 动态代理                                          |

> 注意，Spring的AOP实现并不依赖于AspectJ任何类，它自己实现了一套AOP的。比如它Spring自己提供的BeforeAdvice和AfterAdvice都是对AOP联盟规范的标准实现。以及Spring自己抽象出来的对Advice的包装：org.springframework.aop.Advisor贯穿Spring AOP的始终
>
> **但是在当前注解驱动的流行下，基于POJO（xml方式）以及编程的方式去书写AOP代理，显得非常的繁琐。因此Spring提供了另外一种实现：基于AspectJ，到这才使用到了AspectJ的相关注解、以及类。**
>
> 但是还需要说明一点：哪怕使用到了AspectJ的相关注解和类，但核心的AOP织入的逻辑，还都是Spring自己用动态代理去实现的，没用AspectJ它

### `AopInfrastructureBean`：免被AOP代理的标记接口

### `ProxyConfig`：AOP配置类

```java
public class ProxyConfig implements Serializable {

	// 标记是否直接对目标类进行代理，而不是通过接口产生代理
	private boolean proxyTargetClass = false;
	// 标记是否对代理进行优化。true：那么在生成代理对象之后，如果对代理配置进行了修改，已经创建的代理对象也不会获取修改之后的代理配置。
	// 如果exposeProxy设置为true，即使optimize为true也会被忽略。
	private boolean optimize = false;
	// 标记是否需要阻止通过该配置创建的代理对象转换为Advised类型，默认值为false，表示代理对象可以被转换为Advised类型
	//Advised接口其实就代表了被代理的对象（此接口是Spring AOP提供，它提供了方法可以对代理进行操作，比如移除一个切面之类的），它持有了代理对象的一些属性，通过它可以对生成的代理对象的一些属性进行人为干预
	// 默认情况，我们可以这么完 Advised target = (Advised) context.getBean("opaqueTest"); 从而就可以对该代理持有的一些属性进行干预勒   若此值为true，就不能这么玩了
	boolean opaque = false;
	//标记代理对象是否应该被aop框架通过AopContext以ThreadLocal的形式暴露出去。
	//当一个代理对象需要调用它【自己】的另外一个代理方法时，这个属性将非常有用。默认是是false，以避免不必要的拦截。
	boolean exposeProxy = false;
	//标记是否需要冻结代理对象，即在代理对象生成之后，是否允许对其进行修改，默认为false.
	// 当我们不希望调用方修改转换成Advised对象之后的代理对象时，就可以设置为true 给冻结上即可
	private boolean frozen = false;
}
```

### `ProxyProcessorSupport#evaluateProxyInterfaces`

```java
ublic class ProxyProcessorSupport extends ProxyConfig implements Ordered, BeanClassLoaderAware, AopInfrastructureBean {
	/**
	 * This should run after all other processors, so that it can just add
	 * an advisor to existing proxies rather than double-proxy.
	 * 【AOP的自动代理创建器必须在所有的别的processors之后执行，以确保它可以代理到所有的小伙伴们，即使需要双重代理得那种】
	 */
	private int order = Ordered.LOWEST_PRECEDENCE;
	// 当然此处还是提供了方法，你可以自己set或者使用@Order来人为的改变这个顺序~~~
	public void setOrder(int order) {
		this.order = order;
	}
	@Override
	public int getOrder() {
		return this.order;
	}
	
	...
	// 这是它提供的一个最为核心的方法：这里决定了如果目标类没有实现接口直接就是Cglib代理
	// 检查给定beanClass上的接口们，并交给proxyFactory处理
	protected void evaluateProxyInterfaces(Class<?> beanClass, ProxyFactory proxyFactory) {
		// 找到该类实现的所有接口们~~~
		Class<?>[] targetInterfaces = ClassUtils.getAllInterfacesForClass(beanClass, getProxyClassLoader());
			
		// 标记：是否有存在【合理的】接口~~~
		boolean hasReasonableProxyInterface = false;
		for (Class<?> ifc : targetInterfaces) {
			if (!isConfigurationCallbackInterface(ifc) && !isInternalLanguageInterface(ifc) &&
					
					// 该接口必须还有方法才行，不要忘记了这步判断
					ifc.getMethods().length > 0) {
				hasReasonableProxyInterface = true;
				break;
			}
		}
		if (hasReasonableProxyInterface) {
			// Must allow for introductions; can't just set interfaces to the target's interfaces only.
			// 这里Spring的Doc特别强调了：不能值只把合理的接口设置进去，而是都得加入进去
			for (Class<?> ifc : targetInterfaces) {
				proxyFactory.addInterface(ifc);
			}
		}
		else {
			// 这个很明显设置true，表示使用CGLIB得方式去创建代理了~~~~
			proxyFactory.setProxyTargetClass(true);
		}
	}

	// 判断此接口类型是否属于：容器去回调的类型，这里例举处理一些接口 初始化、销毁、自动刷新、自动关闭、Aware感知等等
	protected boolean isConfigurationCallbackInterface(Class<?> ifc) {
		return (InitializingBean.class == ifc || DisposableBean.class == ifc || Closeable.class == ifc ||
				AutoCloseable.class == ifc || ObjectUtils.containsElement(ifc.getInterfaces(), Aware.class));
	}
	// 是否是如下通用的接口。若实现的是这些接口也会排除，不认为它是实现了接口的类
	protected boolean isInternalLanguageInterface(Class<?> ifc) {
		return (ifc.getName().equals("groovy.lang.GroovyObject") ||
				ifc.getName().endsWith(".cglib.proxy.Factory") ||
				ifc.getName().endsWith(".bytebuddy.MockAccess"));
	}
}
```

ProxyConfig：为上面三个类提供配置属性
AdvisedSupport:继承ProxyConfig，实现了Advised。封装了对通知（Advise）和通知器(Advisor)的操作
ProxyCreatorSupport:继承AdvisedSupport，其帮助子类（上面三个类）创建JDK或者cglib的代理对象
Advised：可以获取拦截器和其他 advice, Advisors和代理接口

### `ProxyCreatorSupport#createAopProxy`

* AopProxyFactory#createAopProxy

继承AdvisedSupport，其帮助子类创建JDK或者cglib的代理对象

```java
public class ProxyCreatorSupport extends AdvisedSupport {
	
	// new了一个aopProxyFactory 
	public ProxyCreatorSupport() {
		this.aopProxyFactory = new DefaultAopProxyFactory();
	}
	
	protected final synchronized AopProxy createAopProxy() {
		if (!this.active) {
			activate();
		}
		// 由此可议看出，它还是委托给了`AopProxyFactory`去做这件事~~~  它的实现类为：DefaultAopProxyFactory
		return getAopProxyFactory().createAopProxy(this);
	}
}

//DefaultAopProxyFactory#createAopProxy
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

	@Override
	public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
		// 对代理进行优化  或者  直接采用CGLIB动态代理  或者 
		//config.isOptimize()与config.isProxyTargetClass()默认返回都是false
		// 需要优化  强制cglib  没有实现接口等都会进入这里面来
		if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
			Class<?> targetClass = config.getTargetClass();
			if (targetClass == null) {
				throw new AopConfigException("TargetSource cannot determine target class: " +
						"Either an interface or a target is required for proxy creation.");
			}
			// 倘若目标Class本身就是个接口，或者它已经是个JDK得代理类（Proxy的子类。所有的JDK代理类都是此类的子类），那还是用JDK的动态代理吧
			if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
				return new JdkDynamicAopProxy(config);
			}
			// 实用CGLIB代理方式 ObjenesisCglibAopProxy是CglibAopProxy的子类。Spring4.0之后提供的
			// 
			return new ObjenesisCglibAopProxy(config);
		}
		// 否则（一般都是有实现接口） 都会采用JDK得动态代理
		else {
			return new JdkDynamicAopProxy(config);
		}
	}

	// 如果它没有实现过接口（ifcs.length == ）  或者 仅仅实现了一个接口，但是呢这个接口却是SpringProxy类型的   那就返回false
	// 总体来说，就是看看这个cofnig有没有实现过靠谱的、可以用的接口
	// SpringProxy:一个标记接口。Spring AOP产生的所有的代理类 都是它的子类~~
	private boolean hasNoUserSuppliedProxyInterfaces(AdvisedSupport config) {
		Class<?>[] ifcs = config.getProxiedInterfaces();
		return (ifcs.length == 0 || (ifcs.length == 1 && SpringProxy.class.isAssignableFrom(ifcs[0])));
	}
}
```

> Objenesis是专门用于实例化一些特殊java对象的一个工具，如私有构造方法。我们知道带参数的构造等不能通过class.newInstance()实例化的，通过它可以轻松完成
> 基于Objenesis的CglibAopProxy扩展，用于创建代理实例，没有调用类的构造器

#### ProxyFactoryBean

#### ProxyFactory

和Spring容器没啥关系，可以直接创建代理

```java
public class ProxyFactory extends ProxyCreatorSupport {
	// 它提供了丰富的构造函数~~~
	public ProxyFactory() {
	}
	public ProxyFactory(Object target) {
		setTarget(target);
		setInterfaces(ClassUtils.getAllInterfaces(target));
	}
	public ProxyFactory(Class<?>... proxyInterfaces) {
		setInterfaces(proxyInterfaces);
	}
	public ProxyFactory(Class<?> proxyInterface, Interceptor interceptor) {
		addInterface(proxyInterface);
		addAdvice(interceptor);
	}
	public ProxyFactory(Class<?> proxyInterface, TargetSource targetSource) {
		addInterface(proxyInterface);
		setTargetSource(targetSource);
	}
	
	// 创建代理的语句：调用父类ProxyCreatorSupport#createAopProxy 此处就不用再解释了
	public Object getProxy() {
		return createAopProxy().getProxy();
	}
	public Object getProxy(@Nullable ClassLoader classLoader) {
		return createAopProxy().getProxy(classLoader);
	}

	// 主义这是个静态方法，可以一步到位，代理指定的接口
	public static <T> T getProxy(Class<T> proxyInterface, Interceptor interceptor) {
		return (T) new ProxyFactory(proxyInterface, interceptor).getProxy();
	}
	public static <T> T getProxy(Class<T> proxyInterface, TargetSource targetSource) {
		return (T) new ProxyFactory(proxyInterface, targetSource).getProxy();
	}
	// 注意：若调用此方法生成代理，就直接使用的是CGLIB的方式的
	public static Object getProxy(TargetSource targetSource) {
		if (targetSource.getTargetClass() == null) {
			throw new IllegalArgumentException("Cannot create class proxy for TargetSource with null target class");
		}
		ProxyFactory proxyFactory = new ProxyFactory();
		proxyFactory.setTargetSource(targetSource);
		proxyFactory.setProxyTargetClass(true);
		return proxyFactory.getProxy();
	}
}
```



#### AspectJProxyFactory

```java
public class AspectJProxyFactory extends ProxyCreatorSupport {
	/** Cache for singleton aspect instances */
	private static final Map<Class<?>, Object> aspectCache = new ConcurrentHashMap<>();
	//基于AspectJ时,创建Spring AOP的Advice  下面详说
	private final AspectJAdvisorFactory aspectFactory = new ReflectiveAspectJAdvisorFactory();

	public AspectJProxyFactory() {
	}
	public AspectJProxyFactory(Object target) {
		Assert.notNull(target, "Target object must not be null");
		setInterfaces(ClassUtils.getAllInterfaces(target));
		setTarget(target);
	}
	public AspectJProxyFactory(Class<?>... interfaces) {
		setInterfaces(interfaces);
	}

	// 这两个addAspect方法是最重要的：我们可以把一个现有的aspectInstance传进去，当然也可以是一个Class（下面）======
	public void addAspect(Object aspectInstance) {
		Class<?> aspectClass = aspectInstance.getClass();
		String aspectName = aspectClass.getName();
		
		AspectMetadata am = createAspectMetadata(aspectClass, aspectName);
		// 显然这种直接传实例进来的，默认就是单例的。不是单例我们就报错了~~~~
		if (am.getAjType().getPerClause().getKind() != PerClauseKind.SINGLETON) {
			throw new IllegalArgumentException(
					"Aspect class [" + aspectClass.getName() + "] does not define a singleton aspect");
		}
		
		// 这个方法就非常的关键了~~~ Singleton...是它MetadataAwareAspectInstanceFactory的子类
		addAdvisorsFromAspectInstanceFactory(
				new SingletonMetadataAwareAspectInstanceFactory(aspectInstance, aspectName));
	}
	public void addAspect(Class<?> aspectClass) {
		String aspectName = aspectClass.getName();
		AspectMetadata am = createAspectMetadata(aspectClass, aspectName);
		MetadataAwareAspectInstanceFactory instanceFactory = createAspectInstanceFactory(am, aspectClass, aspectName);
		addAdvisorsFromAspectInstanceFactory(instanceFactory);
	}

	// 从切面工厂里，把对应切面实例里面的增强器（通知）都获取到~~~
	private void addAdvisorsFromAspectInstanceFactory(MetadataAwareAspectInstanceFactory instanceFactory) {
		// 从切面工厂里，先拿到所有的增强器们~~~
		List<Advisor> advisors = this.aspectFactory.getAdvisors(instanceFactory);
		Class<?> targetClass = getTargetClass();
		Assert.state(targetClass != null, "Unresolvable target class");
		advisors = AopUtils.findAdvisorsThatCanApply(advisors, targetClass);
		AspectJProxyUtils.makeAdvisorChainAspectJCapableIfNecessary(advisors);
		AnnotationAwareOrderComparator.sort(advisors);
		addAdvisors(advisors);
	}
}
```

AspectJProxyFactory,ProxyFactoryBean,ProxyFactory 大体逻辑都是：

1. 填充AdvisedSupport（ProxyCreatorSupport是其子类）的，然后交给父类ProxyCreatorSupport。
2. 得到JDK或者CGLIB的AopProxy
3. 代理调用时候被invoke或者intercept方法拦截 （分别在JdkDynamicAopProxy和ObjenesisCglibAopProxy（CglibAopProxy的子类）的中）
4. 并且在这两个方法中调用ProxyCreatorSupport的getInterceptorsAndDynamicInterceptionAdvice方法去初始化advice和各个方法直接映射关系并缓存



### Advised

> 不管是JDKproxy，还是cglib proxy，代理出来的对象都实现了org.springframework.aop.framework.Advised接口；

Advice: 通知拦截器
Advisor: 通知 + 切入点的适配器
Advised: 包含所有的Advised 和 Advice

该接口用于保存一个代理的相关配置。比如保存了这个代理相关的拦截器、通知、增强器等等。
所有的代理对象都实现了该接口（我们就能够通过一个代理对象获取这个代理对象怎么被代理出来的相关信息）

```java
// 这个 Advised 接口的实现着主要是代理生成的对象与AdvisedSupport (Advised的支持器)
public interface Advised extends TargetClassAware {
     // 这个 frozen 决定是否 AdvisedSupport 里面配置的信息是否改变
    boolean isFrozen();
     // 是否代理指定的类, 而不是一些 Interface
    boolean isProxyTargetClass();
     // 返回代理的接口
    Class<?>[] getProxiedInterfaces();
    // 判断这个接口是否是被代理的接口
    boolean isInterfaceProxied(Class<?> intf);
    // 设置代理的目标对象
    void setTargetSource(TargetSource targetSource);
    // 获取代理的对象
    TargetSource getTargetSource();
    // 判断是否需要将 代理的对象暴露到 ThreadLocal中, 而获取对应的代理对象则通过 AopContext 获取
    void setExposeProxy(boolean exposeProxy);
    // 返回是否应该暴露 代理对象
    boolean isExposeProxy();
     // 设置 Advisor 是否已经在前面过滤过是否匹配 Pointcut (极少用到)
    void setPreFiltered(boolean preFiltered);
    // 获取 Advisor 是否已经在前面过滤过是否匹配 Pointcut (极少用到)
    boolean isPreFiltered();
    // 获取所有的 Advisor
    Advisor[] getAdvisors();
    // 增加 Advisor 到链表的最后
    void addAdvisor(Advisor advisor) throws AopConfigException;
    // 在指定位置增加 Advisor
    void addAdvisor(int pos, Advisor advisor) throws AopConfigException;
    // 删除指定的 Advisor
    boolean removeAdvisor(Advisor advisor);
    // 删除指定位置的 Advisor
    void removeAdvisor(int index) throws AopConfigException;
    // 返回 Advisor 所在位置的 index
    int indexOf(Advisor advisor);
    // 将指定的两个 Advisor 进行替换
    boolean replaceAdvisor(Advisor a, Advisor b) throws AopConfigException;
     // 增加 Advice <- 这个Advice将会包裹成 DefaultPointcutAdvisor
    void addAdvice(Advice advice) throws AopConfigException;
    // 在指定 index 增加 Advice <- 这个Advice将会包裹成 DefaultPointcutAdvisor
    void addAdvice(int pos, Advice advice) throws AopConfigException;
    // 删除给定的 Advice
    boolean removeAdvice(Advice advice);
    // 获取 Advice 的索引位置
    int indexOf(Advice advice);
    // 将 ProxyConfig 通过 String 形式返回
    String toProxyConfigString();
}
```

#### AdvisedSupport#getInterceptorsAndDynamicInterceptionAdvice

* DefaultAdvisorChainFactory#getInterceptorsAndDynamicInterceptionAdvice

> `AdvisedSupport`本身不会提供创建代理的任何方法，专注于生成拦截器链。委托给`ProxyCreatorSupport`去创建代理对象

```java
public class AdvisedSupport extends ProxyConfig implements Advised {
	@Override
	public void addAdvisor(Advisor advisor) {
		int pos = this.advisors.size();
		addAdvisor(pos, advisor);
	}
	@Override
	public void addAdvisor(int pos, Advisor advisor) throws AopConfigException {
		if (advisor instanceof IntroductionAdvisor) {
			validateIntroductionAdvisor((IntroductionAdvisor) advisor);
		}
		addAdvisorInternal(pos, advisor);
	}	
	
	// advice最终都会备转换成一个`Advisor`（DefaultPointcutAdvisor  表示切面+通知），它使用的切面为Pointcut.TRUE
	// Pointcut.TRUE：表示啥都返回true，也就是说这个增强通知将作用于所有的方法上/所有的方法
	// 若要自己指定切面（比如切点表达式）,使用它的另一个构造函数：public DefaultPointcutAdvisor(Pointcut pointcut, Advice advice)
	@Override
	public void addAdvice(Advice advice) throws AopConfigException {
		int pos = this.advisors.size();
		addAdvice(pos, advice);
	}
	@Override
	public void addAdvice(int pos, Advice advice) throws AopConfigException {
		Assert.notNull(advice, "Advice must not be null");
		if (advice instanceof IntroductionInfo) {
			// We don't need an IntroductionAdvisor for this kind of introduction:
			// It's fully self-describing.
			addAdvisor(pos, new DefaultIntroductionAdvisor(advice, (IntroductionInfo) advice));
		}
		else if (advice instanceof DynamicIntroductionAdvice) {
			// We need an IntroductionAdvisor for this kind of introduction.
			throw new AopConfigException("DynamicIntroductionAdvice may only be added as part of IntroductionAdvisor");
		}
		else {
			addAdvisor(pos, new DefaultPointcutAdvisor(advice));
		}
	}

	// 这里需要注意的是：setTarget最终的效果其实也是转换成了TargetSource
	// 也就是说Spring最终代理的  是放进去TargetSource让它去处理
	public void setTarget(Object target) {
		setTargetSource(new SingletonTargetSource(target));
	}
	@Override
	public void setTargetSource(@Nullable TargetSource targetSource) {
		this.targetSource = (targetSource != null ? targetSource : EMPTY_TARGET_SOURCE);
	}

	
		... 其它实现略过，基本都是实现Advised接口的内容

	//将之前注入到advisorChain中的advisors转换为MethodInterceptor和InterceptorAndDynamicMethodMatcher集合（放置了这两种类型的数据）
	// 这些MethodInterceptor们最终在执行目标方法的时候  都是会执行的
	public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
		
		// 以这个Method生成一个key，准备缓存 
		// 此处小技巧：当你的key比较复杂事，可以用类来处理。然后重写它的equals、hashCode、toString、compare等方法
		MethodCacheKey cacheKey = new MethodCacheKey(method);
		List<Object> cached = this.methodCache.get(cacheKey);
		if (cached == null) {
			// 这个方法最终在这 DefaultAdvisorChainFactory#getInterceptorsAndDynamicInterceptionAdvice
			//DefaultAdvisorChainFactory：生成通知器链的工厂，实现了interceptor链的获取过程
			cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
					this, method, targetClass);
			
			// 此处为了提供效率，相当于把该方法对应的拦截器们都缓存起来，加速后续调用得速度
			this.methodCache.put(cacheKey, cached);
		}
		return cached;
	}
}

```

### AdvisorChainFactory

#### DefaultAdvisorChainFactory

```java
//DefaultAdvisorChainFactory：生成拦截器链
public class DefaultAdvisorChainFactory implements AdvisorChainFactory, Serializable {

// 获取匹配 targetClass 与 method 的所有切面的通知
@Override
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
        Advised config, Method method, Class<?> targetClass) {

    // This is somewhat tricky... We have to process introductions first,
    // but we need to preserve order in the ultimate list.
    List<Object> interceptorList = new ArrayList<Object>(config.getAdvisors().length);                                  // PS: 这里 config.getAdvisors 获取的是 advisors 是数组
    Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
    boolean hasIntroductions = hasMatchingIntroductions(config, actualClass);                                       // 判断是有 IntroductionAdvisor 匹配到
    // 下面这个适配器将通知 [Advice] 包装成拦截器 [MethodInterceptor]; 而 DefaultAdvisorAdapterRegistry则是适配器的默认实现
    AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();

    for (Advisor advisor : config.getAdvisors()) {              // 获取所有的 Advisor
        if (advisor instanceof PointcutAdvisor) {               // advisor 是 PointcutAdvisor 的子类
            // Add it conditionally.
            PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
                                                                // 判断此切面 [advisor] 是否匹配 targetClass (PS: 这里是类级别的匹配)
            if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
                /** 通过对适配器将通知 [Advice] 包装成 MethodInterceptor, 这里为什么是个数组? 因为一个通知类
                 *  可能同时实现了前置通知[MethodBeforeAdvice], 后置通知[AfterReturingAdvice], 异常通知接口[ThrowsAdvice]
                 *   环绕通知 [MethodInterceptor], 这里会将每个通知统一包装成 MethodInterceptor
                 */
                MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
                MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
                                                                // 是否匹配 targetClass 类的 method 方法     (PS: 这里是方法级别的匹配)
                if (MethodMatchers.matches(mm, method, actualClass, hasIntroductions)) {
                    if (mm.isRuntime()) {                       // 看了对应的所有实现类, 只有 ControlFlowPointcut 与 AspectJExpressionPointcut 有可能 返回 true
                        // Creating a new object instance in the getInterceptors() method
                        // isn't a problem as we normally cache created chains.
                        // 如果需要在运行时动态拦截方法的执行则创建一个简单的对象封装相关的数据, 它将延时
                        // 到方法执行的时候验证要不要执行此通知
                        for (MethodInterceptor interceptor : interceptors) {
                            interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));           // 里面装的是 Advise 与 MethodMatcher
                        }
                    }
                    else {
                        interceptorList.addAll(Arrays.asList(interceptors));
                    }
                }
            }
        }
        else if (advisor instanceof IntroductionAdvisor) {  // 这里是 IntroductionAdvisor
            // 如果是引入切面的话则判断它是否适用于目标类, Spring 中默认的引入切面实现是 DefaultIntroductionAdvisor 类
            // 默认的引入通知是 DelegatingIntroductionInterceptor 它实现了 MethodInterceptor 接口s
            IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
            if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
                Interceptor[] interceptors = registry.getInterceptors(advisor);
                interceptorList.addAll(Arrays.asList(interceptors));
            }
        }
        else {
            Interceptor[] interceptors = registry.getInterceptors(advisor);
            interceptorList.addAll(Arrays.asList(interceptors));
        }
    }
    return interceptorList;
}
```



### AdvisorAdapter

* AdvisorAdapter
  * MethodBeforeAdviceAdapter --- MethodBeforeAdvice
  * ThrowsAdviceAdapter ---  ThrowsAdvice
  * AfterReturningAdviceAdapter

spring aop 框架对BeforeAdvice、AfterAdvice、ThrowsAdvice三种通知类型的支持实际上是借助适配器模式来实现的，这样的好处是使得框架允许用户向框架中加入自己想要支持的任何一种通知类型

的AdvisorAdapter是一个适配器接口，它定义了自己支持的Advice类型，并且能把一个Advisor适配成MethodInterceptor，以下是它的定义



```java
public interface AdvisorAdapter {
    // 判断此适配器是否支持特定的Advice  
    boolean supportsAdvice(Advice advice);  
    // 将一个Advisor适配成MethodInterceptor  
    MethodInterceptor getInterceptor(Advisor advisor);  
}
```

> 如果我们想把自己定义的AdvisorAdapter注册到spring aop框架中，怎么办？
>
> 把我们自己写好得AdvisorAdapter放进Spring IoC容器中
> 配置一个AdvisorAdapterRegistrationManager，它是一个BeanPostProcessor，它会检测所有的Bean。若是AdvisorAdapter类型，就：
>
> this.advisorAdapterRegistry.registerAdvisorAdapter((AdvisorAdapter) bean);



### AdvisorAdapterRegistry

这里面存在着 MethodBeforeAdviceAdapter, AfterReturningAdviceAdapter, ThrowsAdviceAdapter 这三个类是将 Advice 适配成 MethodInterceptor 的适配类; 而其本身具有两个重要的功能：

1、将 Advice/MethodInterceptor 包裹成 DefaultPointcutAdvisor
2、通过上面的三个适配类将 Advisor 中的 Advice 适配成对应的 MethodInterceptor
(PS: 在代理对象执行时, 执行的都是MethodInterceptor, 当然在进行配置时既可以配置 Advice, MethodInterceptor, Advisor均可)

#### DefaultAdvisorAdapterRegistry

```java
public class DefaultAdvisorAdapterRegistry implements AdvisorAdapterRegistry, Serializable {
	////通知器适配器集合
	private final List<AdvisorAdapter> adapters = new ArrayList<>(3);
	
	// 默认就支持这几种类型的适配器
	public DefaultAdvisorAdapterRegistry() {
		registerAdvisorAdapter(new MethodBeforeAdviceAdapter());
		registerAdvisorAdapter(new AfterReturningAdviceAdapter());
		registerAdvisorAdapter(new ThrowsAdviceAdapter());
	}

	////将通知封装为通知器
	@Override
	public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {
		if (adviceObject instanceof Advisor) {
			return (Advisor) adviceObject;
		}
		if (!(adviceObject instanceof Advice)) {
			throw new UnknownAdviceTypeException(adviceObject);
		}

		Advice advice = (Advice) adviceObject;
		// 如果是MethodInterceptor类型，就根本不用适配器。DefaultPointcutAdvisor是天生处理这种有连接点得通知器的
		if (advice instanceof MethodInterceptor) {
			// So well-known it doesn't even need an adapter.
			return new DefaultPointcutAdvisor(advice);
		}
		
		// 这一步很显然了，就是校验看看这个advice是否是我们支持的这些类型（系统默认给出3中，但是我们也可以自己往里添加注册的）
		for (AdvisorAdapter adapter : this.adapters) {
			// Check that it is supported.
			if (adapter.supportsAdvice(advice)) {
				// 如果是支持的，也是被包装成了一个通用类型的DefaultPointcutAdvisor
				return new DefaultPointcutAdvisor(advice);
			}
		}
		throw new UnknownAdviceTypeException(advice);
	}

	////获得通知器的通知
	@Override
	public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
		List<MethodInterceptor> interceptors = new ArrayList<>(3);
		Advice advice = advisor.getAdvice();
		if (advice instanceof MethodInterceptor) {
			interceptors.add((MethodInterceptor) advice);
		}
		
		// 从所支持的适配器中拿到拦截器通知器
		for (AdvisorAdapter adapter : this.adapters) {
			if (adapter.supportsAdvice(advice)) {
				// 这一步需要注意：一定要把从适配器中拿到MethodInterceptor类型的通知器
				interceptors.add(adapter.getInterceptor(advisor));
			}
		}
		if (interceptors.isEmpty()) {
			throw new UnknownAdviceTypeException(advisor.getAdvice());
		}
		return interceptors.toArray(new MethodInterceptor[0]);
	}

	////注册通知适配器
	@Override
	public void registerAdvisorAdapter(AdvisorAdapter adapter) {
		this.adapters.add(adapter);
	}
}

```

- 生成代理的两个方法:getSingletonInstance和newPrototypeInstance

  ```
  private synchronized Object getSingletonInstance() {
  		// 如果是单例的，现在这里持有这个缓存  创建国就不会再创建了
  		if (this.singletonInstance == null) {
  			// 根据设置的targetName，去工厂里拿到这个bean对象（普通Bean被包装成SingletonTargetSource）
  			this.targetSource = freshTargetSource();
  			
  			// 这一步是如果你手动没有去设置需要被代理的接口，Spring还是会去帮你找看你有没有实现啥接口，然后全部给你代理上。可见Spring的容错性是很强的
  			if (this.autodetectInterfaces && getProxiedInterfaces().length == 0 && !isProxyTargetClass()) {
  				// Rely on AOP infrastructure to tell us what interfaces to proxy.
  				Class<?> targetClass = getTargetClass();
  				if (targetClass == null) {
  					throw new FactoryBeanNotInitializedException("Cannot determine target class for proxy");
  				}
  				setInterfaces(ClassUtils.getAllInterfacesForClass(targetClass, this.proxyClassLoader));
  			}
  			// Initialize the shared singleton instance.
  			super.setFrozen(this.freezeProxy);
  			// createAopProxy()方法就是父类ProxyCreatorSupport的方法，它的具体原理在推荐的博文里已经讲过了，这里就不鳌诉了
  			// 其中JdkDynamicAopProxy和CglibAopProxy对getProxy()方法的实现，也请参考前面分析
  			this.singletonInstance = getProxy(createAopProxy());
  		}
  		return this.singletonInstance;
  	}
  ```

### AopProxyFactory（实际创建代理）

这个接口中定义了根据 AdvisedSupport 中配置的信息来生成合适的AopProxy (主要分为 基于Java 动态代理的 JdkDynamicAopProxy 与基于 Cglib 的ObjenesisCglibAopProxy)

```java
public interface AopProxy {
	//Create a new proxy object. Uses the AopProxy's default class loader  ClassUtils.getDefaultClassLoader()
	Object getProxy();
	Object getProxy(@Nullable ClassLoader classLoader);
}
```



#### JdkDynamicAopProxy

1. setTarget和setInterfaces交给ProxyCreatorSupport
2. addAdvice：此处就是个Advice前置通知增强器。最终会被包装成
3. DefaultPointcutAdvisor交给ProxyCreatorSupport
4. getProxy()才是今天的关键，他内部会去new出来一个JdkDynamicAopProxy，然后调用其getProx()获取到动态代理出来的对象

```java
// 我们发现它自己就实现了了InvocationHandler，所以处理器就是它自己。会实现invoke方法
// 它还是个final类  默认是包的访问权限
final class JdkDynamicAopProxy implements AopProxy, InvocationHandler, Serializable {

	private static final Log logger = LogFactory.getLog(JdkDynamicAopProxy.class);

	/** 这里就保存这个AOP代理所有的配置信息  包括所有的增强器等等 */
	private final AdvisedSupport advised;

	// 标记equals方法和hashCode方法是否定义在了接口上=====
	private boolean equalsDefined;
	private boolean hashCodeDefined;

	public JdkDynamicAopProxy(AdvisedSupport config) throws AopConfigException {
		Assert.notNull(config, "AdvisedSupport must not be null");
		// 内部再校验一次：必须有至少一个增强器  和  目标实例才行
		if (config.getAdvisors().length == 0 && config.getTargetSource() == AdvisedSupport.EMPTY_TARGET_SOURCE) {
			throw new AopConfigException("No advisors and no TargetSource specified");
		}
		this.advised = config;
	}


	@Override
	public Object getProxy() {
		return getProxy(ClassUtils.getDefaultClassLoader());
	}

	// 真正创建JDK动态代理实例的地方
	@Override
	public Object getProxy(@Nullable ClassLoader classLoader) {
		if (logger.isDebugEnabled()) {
			logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
		}
		// 这部很重要，就是去找接口 我们看到最终代理的接口就是这里返回的所有接口们（除了我们自己的接口，还有Spring默认的一些接口）  大致过程如下：
		//1、获取目标对象自己实现的接口们(最终肯定都会被代理的)
		//2、是否添加`SpringProxy`这个接口：目标对象实现对就不添加了，没实现过就添加true
		//3、是否新增`Adviced`接口，注意不是Advice通知接口。 实现过就不实现了，没实现过并且advised.isOpaque()=false就添加（默认是会添加的）
		//4、是否新增DecoratingProxy接口。传入的参数decoratingProxy为true，并且没实现过就添加（显然这里，首次进来是会添加的）
		//5、代理类的接口一共是目标对象的接口+上面三个接口SpringProxy、Advised、DecoratingProxy（SpringProxy是个标记接口而已，其余的接口都有对应的方法的）
		//DecoratingProxy 这个接口Spring4.3后才提供
		Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
		findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
		// 第三个参数传的this，处理器就是自己嘛   到此一个代理对象就此new出来啦
		return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
	}

	// 找找看看接口里有没有自己定义equals方法和hashCode方法，这个很重要  然后标记一下
	// 注意此处用的是getDeclaredMethods，只会找自己的
	private void findDefinedEqualsAndHashCodeMethods(Class<?>[] proxiedInterfaces) {
		for (Class<?> proxiedInterface : proxiedInterfaces) {
			Method[] methods = proxiedInterface.getDeclaredMethods();
			for (Method method : methods) {
				if (AopUtils.isEqualsMethod(method)) {
					this.equalsDefined = true;
				}
				if (AopUtils.isHashCodeMethod(method)) {
					this.hashCodeDefined = true;
				}
				// 小技巧：两个都找到了 就没必要继续循环勒
				if (this.equalsDefined && this.hashCodeDefined) {
					return;
				}
			}
		}
	}

	 // 对于这部分代码和采用CGLIB的大部分逻辑都是一样的，Spring对此的解释很有意思：
	 // 本来是可以抽取出来的，使得代码看起来更优雅。但是因为此会带来10%得性能损耗，所以Spring最终采用了粘贴复制的方式各用一份
	 // Spring说它提供了基础的套件，来保证两个的执行行为是一致的。
	 //proxy:指的是我们所代理的那个真实的对象；method:指的是我们所代理的那个真实对象的某个方法的Method对象args:指的是调用那个真实对象方法的参数。

	// 此处重点分析一下此方法，这样在CGLIB的时候，就可以一带而过了~~~因为大致逻辑是一样的
	@Override
	@Nullable
	public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
		// 它是org.aopalliance.intercept这个包下的  AOP联盟得标准接口
		MethodInvocation invocation;
		Object oldProxy = null;
		boolean setProxyContext = false;

		// 进入invoke方法后，最终操作的是targetSource对象
		// 因为InvocationHandler持久的就是targetSource，最终通过getTarget拿到目标对象
		TargetSource targetSource = this.advised.targetSource;
		Object target = null;

		try {
			//“通常情况”Spring AOP不会对equals、hashCode方法进行拦截增强,所以此处做了处理
			// equalsDefined为false（表示自己没有定义过eequals方法）  那就交给代理去比较
			// hashCode同理，只要你自己没有实现过此方法，那就交给代理吧
			// 需要注意的是：这里统一指的是，如果接口上有此方法，但是你自己并没有实现equals和hashCode方法，那就走AOP这里的实现
			// 如国接口上没有定义此方法，只是实现类里自己@Override了HashCode，那是无效的，就是普通执行吧
			if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
				return equals(args[0]);
			}
			else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
				return hashCode();
			}


			// 下面两段做了很有意思的处理：DecoratingProxy的方法和Advised接口的方法  都是是最终调用了config，也就是this.advised去执行的~~~~
			else if (method.getDeclaringClass() == DecoratingProxy.class) {
				// There is only getDecoratedClass() declared -> dispatch to proxy config.
				return AopProxyUtils.ultimateTargetClass(this.advised);
			}
			else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
					method.getDeclaringClass().isAssignableFrom(Advised.class)) {
				// Service invocations on ProxyConfig with the proxy config...
				return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
			}
			
			// 这个是最终该方法的返回值~~~~
			Object retVal;

			//是否暴露代理对象，默认false可配置为true，如果暴露就意味着允许在线程内共享代理对象，
			//注意这是在线程内，也就是说同一线程的任意地方都能通过AopContext获取该代理对象，这应该算是比较高级一点的用法了。
			// 这里缓存一份代理对象在oldProxy里~~~后面有用
			if (this.advised.exposeProxy) {
				oldProxy = AopContext.setCurrentProxy(proxy);
				setProxyContext = true;
			}

			//通过目标源获取目标对象 (此处Spring建议获取目标对象靠后获取  而不是放在上面) 
			target = targetSource.getTarget();
			Class<?> targetClass = (target != null ? target.getClass() : null);

			// 获取作用在这个方法上的所有拦截器链~~~  参见DefaultAdvisorChainFactory#getInterceptorsAndDynamicInterceptionAdvice方法
			// 会根据切点表达式去匹配这个方法。因此其实每个方法都会进入这里，只是有很多方法得chain事Empty而已
			List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

		
			if (chain.isEmpty()) {
				// 若拦截器为空，那就直接调用目标方法了
				// 对参数进行适配：主要处理一些数组类型的参数，看是表示一个参数  还是表示多个参数（可变参数最终到此都是数组类型，所以最好是需要一次适配）
				Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
				// 这句代码的意思是直接调用目标方法~~~
				retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
			}
			else {
				// 创建一个invocation ，此处为ReflectiveMethodInvocation  最终是通过它，去执行前置加强、后置加强等等逻辑
				invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
				// 此处会执行所有的拦截器链  交给AOP联盟的MethodInvocation去处理。当然实现还是我们Spring得ReflectiveMethodInvocation
				retVal = invocation.proceed();
			}

			// 获取返回值的类型
			Class<?> returnType = method.getReturnType();
			if (retVal != null && retVal == target &&
					returnType != Object.class && returnType.isInstance(proxy) &&
					!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
				 // 一些列的判断条件，如果返回值不为空，且为目标对象的话，就直接将目标对象赋值给retVal
				retVal = proxy;
			}
			// 返回null，并且还不是Void类型。。。抛错
			else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
				throw new AopInvocationException(
						"Null return value from advice does not match primitive return type for: " + method);
			}
			return retVal;
		}
		finally {
			// 释放~~
			if (target != null && !targetSource.isStatic()) {
				targetSource.releaseTarget(target);
			}

			// 把老的代理对象重新set进去~~~
			if (setProxyContext) {
				AopContext.setCurrentProxy(oldProxy);
			}
		}
	}

	// AOP帮我们实现的CgLib方法
	@Override
	public boolean equals(@Nullable Object other) {
		if (other == this) {
			return true;
		}
		if (other == null) {
			return false;
		}

		JdkDynamicAopProxy otherProxy;
		if (other instanceof JdkDynamicAopProxy) {
			otherProxy = (JdkDynamicAopProxy) other;
		}
		else if (Proxy.isProxyClass(other.getClass())) {
			InvocationHandler ih = Proxy.getInvocationHandler(other);
			if (!(ih instanceof JdkDynamicAopProxy)) {
				return false;
			}
			otherProxy = (JdkDynamicAopProxy) ih;
		}
		else {
			// Not a valid comparison...
			return false;
		}

		// If we get here, otherProxy is the other AopProxy.
		return AopProxyUtils.equalsInProxy(this.advised, otherProxy.advised);
	}

	// AOP帮我们实现的HashCode方法
	@Override
	public int hashCode() {
		return JdkDynamicAopProxy.class.hashCode() * 13 + this.advised.getTargetSource().hashCode();
	}

}
```

细节：

除了实现类里自己写的方法（接口上没有的），其余方法统一都会进入代理得invoke()方法里面。只是invoke上做了很多特殊处理，比如DecoratingProxy和Advised等等的方法，都是直接执行了。
object的方法中，toString()方法会被增强（至于为何，我至今还没找到原因，麻烦的知道的给个答案） 因为我始终不知道AdvisedSupport#methodCache这个字段事什么把toString()方法缓存上的，打断点都没跟踪上
生成出来的代理对象，Spring默认都给你实现了接口：SpringProxy、DecoratingProxy、Advised
- 说明：CGLIB代理出来的对象没有实现接口DecoratingProxy。（多谢评论区【神的力量】小伙伴的指正）


#### CglibAopProxy

```java
class CglibAopProxy implements AopProxy, Serializable {

	// 它的两个getProxy()相对来说比较简单，就是使用CGLIB的方式，利用Enhancer创建了一个增强的实例
	// 这里面比较复杂的地方在：getCallbacks()这步是比较繁琐的
	// setCallbackFilter就是看看哪些方法需要拦截、哪些不需要~~~~
	@Override
	public Object getProxy() {
		return getProxy(null);
	}
	
	// CGLIB重写的这两个方法
	@Override
	public boolean equals(Object other) {
		return (this == other || (other instanceof CglibAopProxy &&
				AopProxyUtils.equalsInProxy(this.advised, ((CglibAopProxy) other).advised)));
	}
	@Override
	public int hashCode() {
		return CglibAopProxy.class.hashCode() * 13 + this.advised.getTargetSource().hashCode();
	}

	// 最后，所有的被代理得类的所有的方法调用，都会进入DynamicAdvisedInterceptor#intercept这个方法里面来（相当于JDK动态代理得invoke方法）
	// 它实现了MethodInterceptor接口
	private static class DynamicAdvisedInterceptor implements MethodInterceptor, Serializable {

		private final AdvisedSupport advised;

		public DynamicAdvisedInterceptor(AdvisedSupport advised) {
			this.advised = advised;
		}

		@Override
		@Nullable
		public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
			Object oldProxy = null;
			boolean setProxyContext = false;
			Object target = null;
			// 目标对象源
			TargetSource targetSource = this.advised.getTargetSource();
			try {
				if (this.advised.exposeProxy) {
					oldProxy = AopContext.setCurrentProxy(proxy);
					setProxyContext = true;
				}

				// 拿到目标对象   这里就是使用targetSource的意义，它提供多个实现类，从而实现了更多的可能性
				// 比如：SingletonTargetSource  HotSwappableTargetSource  PrototypeTargetSource  ThreadLocalTargetSource等等
				target = targetSource.getTarget();
				Class<?> targetClass = (target != null ? target.getClass() : null);
			
				// 一样的，也是拿到和这个方法匹配的 所有的增强器、通知们 和JDK Proxy中是一样的
				List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
				Object retVal;
				// 没有增强器，同时该方法是public得  就直接调用目标方法（不拦截）
				if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
					Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
					retVal = methodProxy.invoke(target, argsToUse);
				}
				else {
					// CglibMethodInvocation这里采用的是CglibMethodInvocation，它是`ReflectiveMethodInvocation`的子类   到这里就和JDK Proxy保持一致勒 
					retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
				}
				retVal = processReturnType(proxy, target, method, retVal);
				return retVal;
			}
			finally {
				if (target != null && !targetSource.isStatic()) {
					targetSource.releaseTarget(target);
				}
				if (setProxyContext) {
					AopContext.setCurrentProxy(oldProxy);
				}
			}
		}

		@Override
		public boolean equals(Object other) {
			return (this == other ||
					(other instanceof DynamicAdvisedInterceptor &&
							this.advised.equals(((DynamicAdvisedInterceptor) other).advised)));
		}

		/**
		 * CGLIB uses this to drive proxy creation.
		 */
		@Override
		public int hashCode() {
			return this.advised.hashCode();
		}
	}
}
```

细节：

和JDK的一样，Object的方法，只有toString()会被拦截（执行通知）
生成出来的代理对象，Spring默认都给你实现了接口：SpringProxy、DecoratingProxy、Advised
它和JDK不同的是，比如equals和hashCode等方法根本就不会进入intecept方法，而是在getCallbacks()那里就给特殊处理掉了

#### ObjenesisCglibAopProxy

```java
// 它是Spring4.0之后提供的
class ObjenesisCglibAopProxy extends CglibAopProxy {
	// 下面有解释，另外一种创建实例的方式（可议不用空的构造函数哟）
	private static final SpringObjenesis objenesis = new SpringObjenesis();
	
	public ObjenesisCglibAopProxy(AdvisedSupport config) {
		super(config);
	}

	// 创建一个代理得实例
	@Override
	protected Object createProxyClassAndInstance(Enhancer enhancer, Callback[] callbacks) {
		Class<?> proxyClass = enhancer.createClass();
		Object proxyInstance = null;
		
		// 如果为true，那我们就采用objenesis去new一个实例~~~
		if (objenesis.isWorthTrying()) {
			try {
				proxyInstance = objenesis.newInstance(proxyClass, enhancer.getUseCache());
			} catch (Throwable ex) {
				logger.debug("Unable to instantiate proxy using Objenesis, " +
						"falling back to regular proxy construction", ex);
			}
		}
		
		// 若果还为null，就再去拿到构造函数（指定参数的）
		if (proxyInstance == null) {
			// Regular instantiation via default constructor...
			try {
				Constructor<?> ctor = (this.constructorArgs != null ?
						proxyClass.getDeclaredConstructor(this.constructorArgTypes) :
						proxyClass.getDeclaredConstructor());
				
				// 通过此构造函数  去new一个实例
				ReflectionUtils.makeAccessible(ctor);
				proxyInstance = (this.constructorArgs != null ?
						ctor.newInstance(this.constructorArgs) : ctor.newInstance());
			} catch (Throwable ex) {
				throw new AopConfigException("Unable to instantiate proxy using Objenesis, " +
						"and regular proxy instantiation via default constructor fails as well", ex);
			}
		}

		((Factory) proxyInstance).setCallbacks(callbacks);
		return proxyInstance;
	}
}
```



### MethodInvocation

MethodInvocation 是进行 invoke 对应方法 / MethodInterceptor的类。
其主要分成用于 Proxy 的 ReflectiveMethodInvocation, 与用于 Cglib 的 CglibMethodInvocation
(其实就是就是递归的调用 MethodInterceptor, 当没有 MethodInterceptor可以调用时)

### AbstractAutoProxyCreator

这个类是声明式 Aop 编程中非常重要的一个角色：自动代理创建器
解读如下：

`AbstractAutoProxyCreator `继承了 `ProxyProcessorSupport`, 所以它具有了ProxyConfig中动态代理应该具有的配置属性

`AbstractAutoProxyCreator `实现了 `SmartInstantiationAwareBeanPostProcessor`(包括实例化的前后置函数, 初始化的前后置函数) 并进行了实现
实现了 创建代理类的主方法 createProxy 方法
定义了抽象方法 getAdvicesAndAdvisorsForBean**(获取 Bean对应的 Advisor)**
AbstractAutoProxyCreator 中等于是构建了创建 Aop 对象的主逻辑**（模版）**，而其子类 AbstractAdvisorAutoProxyCreator 实现了getAdvicesAndAdvisorsForBean 方法, 并且通过工具类 BeanFactoryAdvisorRetrievalHelper(PS: 它的方法findAdvisorBeans中实现类获取容器中所有 Advisor 的方法) 来获取其对应的 Advisor。它的主要子类如下：

#### AspectJAwareAdvisorAutoProxyCreator

通过解析 aop 命名空间的配置信息时生成的 AdvisorAutoProxyCreator, 主要通过ConfigBeanDefinitionParser.parse() -> ConfigBeanDefinitionParser.configureAutoProxyCreator() -> AopNamespaceUtils.registerAspectJAutoProxyCreatorIfNecessary() -> AopNamespaceUtils.registerAspectJAutoProxyCreatorIfNecessary(); 与之对应的 Pointcut 是AspectJExpressionPointcut, Advisor 是 AspectJPointcutAdvisor, Advice 则是 AspectJAfterAdvice, AspectJAfterReturningAdvice, AspectJAfterThrowingAdvice, AspectJAroundAdvice

#### AnnotationAwareAspectJAutoProxyCreator

这是基于 @AspectJ注解生成的 切面类的一个 AbstractAutoProxyCreator, 解析额工作交给了 AspectJAutoProxyBeanDefinitionParser, 步骤如下AspectJAutoProxyBeanDefinitionParser.parse() -> AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary() -> AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary()

#### DefaultAdvisorAutoProxyCreator

这个类也是 AbstractAutoProxyCreator的子类, 它帅选作用的类时主要是根据其中的 advisorBeanNamePrefix(类名前缀)配置进行判断

#### BeanNameAutoProxyCreator

通过类的名字来判断是否作用(正则匹配)

### TargetSource

该接口代表一个目标对象，在aop调用目标对象的时候，使用该接口返回真实的对象。
比如它有其中两个实现`SingletonTargetSource`和`PrototypeTargetSource`代表着每次调用返回同一个实例，和每次调用返回一个新的实例

TargetSource：其实是动态代理作用的对象

1. HotSwappableTargetSource: 进行线程安全的热切换到对另外一个对象实施动态代理操作
2. AbstractPoolingTargetSource: 每次进行生成动态代理对象时都返回一个新的对象（比如内部实现类CommonsPool2TargetSource就是例子，但它依赖于common-pool2包）
3. ThreadLocalTargetSource: 为每个进行请求的线程维护一个对象的 TargetSource
4. SingletonTargetSource: 最普遍最基本的单例 TargetSource, 在 Spring 中生成动态代理对象, 一般都是用这个 TargetSource

### TargetClassAware

`所有的`Aop代理对象或者代理工厂（proxy factory)都要实现的接口，该接口用于暴露出被代理目标对象类型；

```java
public interface TargetClassAware {
	// 返回被代理得目标类型  AopUtils#getTargetClass(Object)
	@Nullable
	Class<?> getTargetClass();
}
```



### BeanFactoryAspectJAdvisorsBuilder#buildAspectJAdvisors

注解驱动中 AnnotationAwareAspectJAutoProxyCreator 所继承的类中 重写了 setBeanFactory 创建

### AspectJAdvisorFactory（提供创建 Advisor 和 Advice）

#### AbstractAspectJAdvisorFactory

##### ReflectiveAspectJAdvisorFactory

getAdvisor，getAdvise

###### AbstractAspectJAdvice



### AspectMetadata

表示一个切面的元数据类

```java
public class AspectMetadata implements Serializable {
	private final String aspectName;
	private final Class<?> aspectClass;
	// AjType这个字段非常的关键，它表示有非常非常多得关于这个切面的一些数据、方法（位于org.aspectj下）
	private transient AjType<?> ajType;
	
	// 解析切入点表达式用的，但是真正的解析工作为委托给`org.aspectj.weaver.tools.PointcutExpression`来解析的
	//若是单例：则是Pointcut.TRUE  否则为AspectJExpressionPointcut
	private final Pointcut perClausePointcut;

	public AspectMetadata(Class<?> aspectClass, String aspectName) {
		this.aspectName = aspectName;

		Class<?> currClass = aspectClass;
		AjType<?> ajType = null;
		
		// 此处会一直遍历到顶层知道Object  直到找到有一个是Aspect切面就行，然后保存起来
		// 因此我们的切面写在父类上 也是欧克的
		while (currClass != Object.class) {
			AjType<?> ajTypeToCheck = AjTypeSystem.getAjType(currClass);
			if (ajTypeToCheck.isAspect()) {
				ajType = ajTypeToCheck;
				break;
			}
			currClass = currClass.getSuperclass();
		}
		
		// 由此可见，我们传进来的Class必须是个切面或者切面的子类的~~~
		if (ajType == null) {
			throw new IllegalArgumentException("Class '" + aspectClass.getName() + "' is not an @AspectJ aspect");
		}
		// 显然Spring AOP目前也不支持优先级的声明。。。
		if (ajType.getDeclarePrecedence().length > 0) {
			throw new IllegalArgumentException("DeclarePrecendence not presently supported in Spring AOP");
		}
		this.aspectClass = ajType.getJavaClass();
		this.ajType = ajType;

		// 切面的处在类型：PerClauseKind  由此可议看出，Spring的AOP目前只支持下面4种 
		switch (this.ajType.getPerClause().getKind()) {
			case SINGLETON:
				// 如国是单例，这个表达式返回这个常量
				this.perClausePointcut = Pointcut.TRUE;
				return;
			case PERTARGET:
			case PERTHIS:
				// PERTARGET和PERTHIS处理方式一样  返回的是AspectJExpressionPointcut
				AspectJExpressionPointcut ajexp = new AspectJExpressionPointcut();
				ajexp.setLocation(aspectClass.getName());
				//设置好 切点表达式
				ajexp.setExpression(findPerClause(aspectClass));
				ajexp.setPointcutDeclarationScope(aspectClass);
				this.perClausePointcut = ajexp;
				return;
			case PERTYPEWITHIN:
				// Works with a type pattern
				// 组成的、合成得切点表达式~~~
				this.perClausePointcut = new ComposablePointcut(new TypePatternClassFilter(findPerClause(aspectClass)));
				return;
			default:
				// 其余的Spring AOP暂时不支持
				throw new AopConfigException(
						"PerClause " + ajType.getPerClause().getKind() + " not supported by Spring AOP for " + aspectClass);
		}
	}

	private String findPerClause(Class<?> aspectClass) {
		String str = aspectClass.getAnnotation(Aspect.class).value();
		str = str.substring(str.indexOf('(') + 1);
		str = str.substring(0, str.length() - 1);
		return str;
	}
	...
	public Pointcut getPerClausePointcut() {
		return this.perClausePointcut;
	}
	// 判断perThis或者perTarger，最单实例、多实例处理
	public boolean isPerThisOrPerTarget() {
		PerClauseKind kind = getAjType().getPerClause().getKind();
		return (kind == PerClauseKind.PERTARGET || kind == PerClauseKind.PERTHIS);
	}
	// 是否是within的
	public boolean isPerTypeWithin() {
		PerClauseKind kind = getAjType().getPerClause().getKind();
		return (kind == PerClauseKind.PERTYPEWITHIN);
	}
	// 只要不是单例的，就都属于Lazy懒加载，延迟实例化的类型~~~~
	public boolean isLazilyInstantiated() {
		return (isPerThisOrPerTarget() || isPerTypeWithin());
	}
}
```



### AspectInstanceFactory：切面工厂（切面创建）

专门为切面创建实例的工厂

```java
// 它实现了Order接口哦~~~~支持排序的
public interface AspectInstanceFactory extends Ordered {
	//Create an instance of this factory's aspect.
	Object getAspectInstance();
	//Expose the aspect class loader that this factory uses.
	@Nullable
	ClassLoader getAspectClassLoader();
}
```

SimpleAspectInstanceFactory：根据切面的aspectClass，调用空构造函数反射.newInstance()创建一个实例（备注：构造函数private的也没有关系）

SingletonAspectInstanceFactory：因为已经持有aspectInstance得引用了，直接return即可



#### MetadataAwareAspectInstanceFactory

SimpleMetadataAwareAspectInstanceFactory和SingletonMetadataAwareAspectInstanceFactory已经直接关联到AspectMetadata，所以直接return即可。
LazySingletonAspectInstanceFactoryDecorator也只是个简单的装饰而已。

##### BeanFactoryAspectInstanceFactory

```java
public class BeanFactoryAspectInstanceFactory implements MetadataAwareAspectInstanceFactory, Serializable {
	
	// 持有对Bean工厂的引用
	private final BeanFactory beanFactory;
	// 需要处理的名称
	private final String name;
	private final AspectMetadata aspectMetadata;

	// 传了Name，type可议不传，内部判断出来
	public BeanFactoryAspectInstanceFactory(BeanFactory beanFactory, String name) {
		this(beanFactory, name, null);
	}
	public BeanFactoryAspectInstanceFactory(BeanFactory beanFactory, String name, @Nullable Class<?> type) {
		this.beanFactory = beanFactory;
		this.name = name;
		Class<?> resolvedType = type;
		// 若没传type，就去Bean工厂里看看它的Type是啥  type不能为null~~~~
		if (type == null) {
			resolvedType = beanFactory.getType(name);
			Assert.notNull(resolvedType, "Unresolvable bean type - explicitly specify the aspect class");
		}
		// 包装成切面元数据类
		this.aspectMetadata = new AspectMetadata(resolvedType, name);
	}

	// 此处：切面实例 是从Bean工厂里获取的  需要注意
	// 若是多例的，请注意Scope的值
	@Override
	public Object getAspectInstance() {
		return this.beanFactory.getBean(this.name);
	}

	@Override
	@Nullable
	public ClassLoader getAspectClassLoader() {
		return (this.beanFactory instanceof ConfigurableBeanFactory ?
				((ConfigurableBeanFactory) this.beanFactory).getBeanClassLoader() :
				ClassUtils.getDefaultClassLoader());
	}

	@Override
	public AspectMetadata getAspectMetadata() {
		return this.aspectMetadata;
	}

	@Override
	@Nullable
	public Object getAspectCreationMutex() {
		if (this.beanFactory.isSingleton(this.name)) {
			// Rely on singleton semantics provided by the factory -> no local lock.
			return null;
		}
		else if (this.beanFactory instanceof ConfigurableBeanFactory) {
			// No singleton guarantees from the factory -> let's lock locally but
			// reuse the factory's singleton lock, just in case a lazy dependency
			// of our advice bean happens to trigger the singleton lock implicitly...
			return ((ConfigurableBeanFactory) this.beanFactory).getSingletonMutex();
		}
		else {
			return this;
		}
	}

	@Override
	public int getOrder() {
		Class<?> type = this.beanFactory.getType(this.name);
		if (type != null) {
			if (Ordered.class.isAssignableFrom(type) && this.beanFactory.isSingleton(this.name)) {
				return ((Ordered) this.beanFactory.getBean(this.name)).getOrder();
			}
			// 若没实现接口，就拿注解的值
			return OrderUtils.getOrder(type, Ordered.LOWEST_PRECEDENCE);
		}
		return Ordered.LOWEST_PRECEDENCE;
	}
}
```

##### PrototypeAspectInstanceFactory



# AOP 核心组件

切面信息如何注册？

代理类创建的时机，什么情况下生成代理？如何生成代理？

怎么匹配符合条件的生成代理？

从激活 aop 到  Advisor bean 注册 关联到 pointcut 和 advice 注册，到 需要代理 bean 的代理创建

## Pointcut

```java
/ 由 ClassFilter 与 MethodMatcher 组成的 pointcut
public interface Pointcut {
    // 类过滤器, 可以知道哪些类需要拦截
    ClassFilter getClassFilter();
    // 方法匹配器, 可以知道哪些方法需要拦截
    MethodMatcher getMethodMatcher();
    // 匹配所有对象的 Pointcut
    Pointcut TRUE = TruePointcut.INSTANCE;
}

```

### ClassFilter

**ClassFilter限定在类级别上，MethodMatcher限定在方法级别上**

SpringAop主要支持在方法级别上的匹配，所以对类级别的匹配支持相对简单一些

```java
@FunctionalInterface
public interface ClassFilter {

	// true表示能够匹配。那就会进行织入的操作
	boolean matches(Class<?> clazz);
	// 常量 会匹配所有的类   TrueClassFilter不是public得class，所以只是Spring内部自己使用的
	ClassFilter TRUE = TrueClassFilter.INSTANCE;
}
```

#### RootClassFilter

```java
public class RootClassFilter implements ClassFilter, Serializable {

	private Class<?> clazz;
	public RootClassFilter(Class<?> clazz) {
		this.clazz = clazz;
	}
	// 显然，传进来的candidate必须是clazz的子类才行
	@Override
	public boolean matches(Class<?> candidate) {
		return clazz.isAssignableFrom(candidate);
	}
}
```

#### AnnotationClassFilter

```java
public class AnnotationClassFilter implements ClassFilter {
	...
	public AnnotationClassFilter(Class<? extends Annotation> annotationType) {
		// 默认情况下checkInherited给的false：不去看它继承过来的注解
		this(annotationType, false);
	}
	// checkInherited true：表示继承过来得注解也算
	public AnnotationClassFilter(Class<? extends Annotation> annotationType, boolean checkInherited) {
		Assert.notNull(annotationType, "Annotation type must not be null");
		this.annotationType = annotationType;
		this.checkInherited = checkInherited;
	}
	...
	@Override
	public boolean matches(Class<?> clazz) {
		return (this.checkInherited ?
				// 继承的注解也会找出来
				(AnnotationUtils.findAnnotation(clazz, this.annotationType) != null) :
				// 只会看自己本类的注解
				clazz.isAnnotationPresent(this.annotationType));
	}
}
```



### MethodMatcher

```java
public interface MethodMatcher {
	
	// 这个称为静态匹配：在匹配条件不是太严格时使用，可以满足大部分场景的使用
	boolean matches(Method method, @Nullable Class<?> targetClass);
	// 这个称为动态匹配（运行时匹配）: 它是严格的匹配。在运行时动态的对参数的类型进行匹配
	boolean matches(Method method, @Nullable Class<?> targetClass, Object... args);
	
	//两个方法的分界线就是boolean isRuntime()方法，步骤如下
	// 1、先调用静态匹配，若返回true。此时就会继续去检查isRuntime()的返回值
	// 2、若isRuntime()还返回true，那就继续调用动态匹配
	// (若静态匹配都匹配上，动态匹配那铁定更匹配不上得~~~~)

	// 是否需要执行动态匹配
	boolean isRuntime();
	MethodMatcher TRUE = TrueMethodMatcher.INSTANCE;

}
```

#### StaticMethodMatcher 静态匹配

作用：它表示不会考虑具体 方法参数。因为不用每次都检查参数，那么对于同样的类型的方法匹配结果，就可以在`框架内部缓存`以提高性能。比如常用的实现类：`AnnotationMethodMatcher`

```java
public abstract class StaticMethodMatcher implements MethodMatcher {
	// 永远返回false表示只会去静态匹配
	@Override
	public final boolean isRuntime() {
		return false;
	}
	// 参数matches抛出异常，使其不被调用
	@Override
	public final boolean matches(Method method, @Nullable Class<?> targetClass, Object... args) {
		// should never be invoked because isRuntime() returns false
		throw new UnsupportedOperationException("Illegal MethodMatcher usage");
	}

}
```

#### DynamicMethodMatcher 动态匹配

说明：因为每次都要对方法参数进行检查，无法对匹配结果进行缓存，所以，匹配效率相对 StatisMethodMatcher 来说要差，但匹配度更高。(`实际使用得其实较少`)

```java
public abstract class DynamicMethodMatcher implements MethodMatcher {
	
	// 永远返回true
	@Override
	public final boolean isRuntime() {
		return true;
	}
	// 永远返回true，去匹配动态匹配的方法即可
	@Override
	public boolean matches(Method method, @Nullable Class<?> targetClass) {
		return true;
	}
}
```



### Pointcut 子类

* NameMatchMethodPointcut：通过方法名进行精确匹配的。 (PS: 其中 ClassFilter = ClassFilter.TRUE)用于**只需要方法名字匹配**，无需理会方法的签名和返回类型
* ControlFlowPointcut：根据在当前线程的堆栈信息中的方法名来决定是否切入某个方法（效率较低）
* ComposablePointcut：组合模式的 Pointcut, 主要分成两种: 1.组合中所有都匹配算成功 2. 组合中都不匹配才算成功
* JdkRegexpMethodPointcut：通过 正则表达式来匹配方法（PS： ClassFilter.TRUE）
* AspectJExpressionPointcut：通过 AspectJ 包中的组件进行方法的匹配(切点表达式)
* TransactionAttributeSourcePointcut：通过 TransactionAttributeSource 在 类的方法上提取事务注解的属性 @Transactional 来判断是否匹配, 提取到则说明匹配, 提取不到则说明匹配不成功
* AnnotationJCacheOperationSource：支持JSR107的cache相关注解的支持
  `Advisor`是`Pointcut`以及`Advice`的一个结合

#### AspectJExpressionPointcut



```java
// 很容易发现，自己即是ClassFilter，也是MethodMatcher   
// 它是子接口:ExpressionPointcut的实现类
public class AspectJExpressionPointcut extends AbstractExpressionPointcut
		implements ClassFilter, IntroductionAwareMethodMatcher, BeanFactoryAware {
		...
		private static final Set<PointcutPrimitive> SUPPORTED_PRIMITIVES = new HashSet<>();
		
	// 从此处可以看出，Spring支持的AspectJ的切点语言表达式一共有10中（加上后面的自己的Bean方式一共11种）
	// AspectJ框架本省支持的非常非常多，详解枚举类：org.aspectj.weaver.tools.PointcutPrimitive
	static {
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.EXECUTION);
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.ARGS);
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.REFERENCE);
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.THIS);
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.TARGET);
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.WITHIN);
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.AT_ANNOTATION);
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.AT_WITHIN);
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.AT_ARGS);
		SUPPORTED_PRIMITIVES.add(PointcutPrimitive.AT_TARGET);
	}
	...
		
	// 它持有BeanFactory 的引用，但是是可以为null的，也就是说它脱离容器也能够正常work
	@Nullable
	private BeanFactory beanFactory;
	...
	// PointcutExpression是org.aspectj.weaver.tools.PointcutExpression是AspectJ的类
	// 它最终通过一系列操作，由org.aspectj.weaver.tools.PointcutParser#parsePointcutExpression从字符串表达式解析出来
	@Nullable
	private transient PointcutExpression pointcutExpression;
	
	...
	// 由此可见，我们不仅仅可议写&& ||  !这种。也支持 and or not这种哦~~~
	private String replaceBooleanOperators(String pcExpr) {
		String result = StringUtils.replace(pcExpr, " and ", " && ");
		result = StringUtils.replace(result, " or ", " || ");
		result = StringUtils.replace(result, " not ", " ! ");
		return result;
	}
	...
	// 这是ClassFilter 匹配类。借助的PointcutExpression#couldMatchJoinPointsInType 去匹配
	public boolean matches(Class<?> targetClass) { ... }
	// MethodMatcher 匹配方法，借助的PointcutExpression和ShadowMatch去匹配的
	public boolean matches(Method method, @Nullable Class<?> targetClass, boolean hasIntroductions) { ... }
	@Override
	public boolean isRuntime() {
		//mayNeedDynamicTest 相当于由AspectJ框架去判断的（是否有动态内容）
		return obtainPointcutExpression().mayNeedDynamicTest();
	}
	...

	// 初始化一个Pointcut的解析器。我们发现最后一行，新注册了一个BeanPointcutDesignatorHandler  它是准们处理Spring自己支持的bean() 的切点表达式的
	private PointcutParser initializePointcutParser(@Nullable ClassLoader classLoader) {
		PointcutParser parser = PointcutParser
				.getPointcutParserSupportingSpecifiedPrimitivesAndUsingSpecifiedClassLoaderForResolution(
						SUPPORTED_PRIMITIVES, classLoader);
		parser.registerPointcutDesignatorHandler(new BeanPointcutDesignatorHandler());
		return parser;
	}

	// 真正的解析，依赖于Spring自己实现的这个内部类（主要是ContextBasedMatcher 这个类，就会使用到BeanFactory了）
	private class BeanPointcutDesignatorHandler implements PointcutDesignatorHandler {

		private static final String BEAN_DESIGNATOR_NAME = "bean";
		@Override
		public String getDesignatorName() {
			return BEAN_DESIGNATOR_NAME;
		}
		// ContextBasedMatcher由Spring自己实现，对容器内Bean的匹配
		@Override
		public ContextBasedMatcher parse(String expression) {
			return new BeanContextMatcher(expression);
		}
	}

}
```



#### ControlFlowPointCut

#### ComposablePointcut

`ComposablePointcut`没有提供**直接对两个切点类型**并集交集的运算的方法。若需要，请参照`org.springframework.aop.support.Pointcuts`这个工具类里面有对两个Pointcut进行并集、交集的操作 

```java
public class ComposablePointcut implements Pointcut, Serializable {

	// 它持有ClassFilter 和 MethodMatcher ，最终通过它去组合匹配
	private ClassFilter classFilter;
	private MethodMatcher methodMatcher;

	// 构造函数一个共5个

	// 匹配所有类所有方法的复合切点
	public ComposablePointcut() {
		this.classFilter = ClassFilter.TRUE;
		this.methodMatcher = MethodMatcher.TRUE;
	}
	// 匹配特定切点的复合切点（相当于把这个节点包装了一下而已）
	public ComposablePointcut(Pointcut pointcut) {
		Assert.notNull(pointcut, "Pointcut must not be null");
		this.classFilter = pointcut.getClassFilter();
		this.methodMatcher = pointcut.getMethodMatcher();
	}
	// 匹配特定类**所有方法**的复合切点
	public ComposablePointcut(ClassFilter classFilter) {
		Assert.notNull(classFilter, "ClassFilter must not be null");
		this.classFilter = classFilter;
		this.methodMatcher = MethodMatcher.TRUE;
	}
	// 匹配**所有类**特定方法的复合切点
	public ComposablePointcut(MethodMatcher methodMatcher) {
		Assert.notNull(methodMatcher, "MethodMatcher must not be null");
		this.classFilter = ClassFilter.TRUE;
		this.methodMatcher = methodMatcher;
	}
	// 匹配特定类特定方法的复合切点（这个是最为强大的）
	public ComposablePointcut(ClassFilter classFilter, MethodMatcher methodMatcher) {
		Assert.notNull(classFilter, "ClassFilter must not be null");
		Assert.notNull(methodMatcher, "MethodMatcher must not be null");
		this.classFilter = classFilter;
		this.methodMatcher = methodMatcher;
	}
	
	// 匹配特定类特定方法的复合切点（这个是最为强大的）
	public ComposablePointcut union(ClassFilter other) {
		this.classFilter = ClassFilters.union(this.classFilter, other);
		return this;
	}

	// ==========3个并集(union) / 3个交集(intersection) 运算的方法========
	public ComposablePointcut intersection(ClassFilter other) {
		this.classFilter = ClassFilters.intersection(this.classFilter, other);
		return this;
	}
	public ComposablePointcut union(MethodMatcher other) {
		this.methodMatcher = MethodMatchers.union(this.methodMatcher, other);
		return this;
	}
	public ComposablePointcut intersection(MethodMatcher other) {
		this.methodMatcher = MethodMatchers.intersection(this.methodMatcher, other);
		return this;
	}
	public ComposablePointcut union(Pointcut other) {
		this.methodMatcher = MethodMatchers.union(
				this.methodMatcher, this.classFilter, other.getMethodMatcher(), other.getClassFilter());
		this.classFilter = ClassFilters.union(this.classFilter, other.getClassFilter());
		return this;
	}
	public ComposablePointcut intersection(Pointcut other) {
		this.classFilter = ClassFilters.intersection(this.classFilter, other.getClassFilter());
		this.methodMatcher = MethodMatchers.intersection(this.methodMatcher, other.getMethodMatcher());
		return this;
	}
	...
}
```



#### AnnotationMatchingPointcut

根据对象是否有指定类型的注解来匹配Pointcut
有两种注解，`类级别注解和方法级别注解`。

```java
/仅指定类级别的注解， 标注了 ClassLevelAnnotation 注解的类中的**所有方法**执行的时候，将全部匹配。  
AnnotationMatchingPointcut pointcut = new AnnotationMatchingPointcut(ClassLevelAnnotation.class);  
// === 还可以使用静态方法创建 pointcut 实例  
AnnotationMatchingPointcut pointcut = AnnotationMatchingPointcut.forClassAnnotation(ClassLevelAnnotation.class);  


//仅指定方法级别的注解，标注了 MethodLeavelAnnotaion 注解的**方法（忽略类匹配）都将匹配**  
AnnotationMatchingPointcut pointcut = AnnotationMatchingPointcut.forMethodAnnotation(MethodLevelAnnotation.class);  

==========这个是同时想限定：===============
//同时限定类级别和方法级别的注解，只有标注了 ClassLevelAnnotation 的类中 ***同时***标注了 MethodLevelAnnotation 的方法才会匹配  
AnnotationMatchingPointcut pointcut = new AnnotationMatchingPointcut(ClassLevelAnnotation.class, MethodLevelAnnotation.class);  
```



## Advice

### 普通advice 

MethodBeforeAdvice：在目标方法之前执行，主要实现有：
1. AspectJMethodBeoreAdvice：这是通过解析被 org.aspectj.lang.annotation.Before 注解注释的方法时解析成的Advice
2. AfterReturningAdvice：在切面方法执行后(这里的执行后指不向外抛异常, 否则的话就应该是 AspectJAfterThrowingAdvice 这种 Advice) 主要实现有：
3. AspectJAfterAdvice：解析 AspectJ 中的 @After 注解来生成的 Advice(PS: 在java中的实现其实就是在 finally 方法中调用以下对应要执行的方法)
4. AspectJAfterReturningAdvice：解析 AspectJ 中的@AfterReturning 属性来生成的 Advice(PS: 若切面方法抛出异常, 则这里的方法就将不执行)
5. AspectJAfterThrowingAdvice：解析 AspectJ 中的 @AfterThrowing 属性来生成的 Advice(PS: 若切面方法抛出异常, 则这里的方法就执行)
6. AspectJAroundAdvice：将执行类 MethodInvocation(MethodInvocation其实就是Pointcut) 进行包裹起来, 并控制其执行的 Advice (其中 Jdk中中 Proxy 使用ReflectiveMethodInvocation, 而 Cglib 则使用 CglibMethodInvocation)

> 注意：在Proxy中最终执行的其实都是`MethodInterceptor`，因此这些Advice最终都是交给 AdvisorAdapter -> 将 advice 适配成 MethodInterceptor

### Interceptor/MethodInterceptor

1. ExposeInvocationInterceptor：将当前 MethodInvocation 放到当前线程对应的 ThreadLoadMap里面的, 这是一个默认的 Interceptor, 在AspectJAwareAdvisorAutoProxy获取何时的 Advisor 时会调用自己的 extendAdvisors 方法, 从而将 ExposeInvocationInterceptor 方法执行链表的第一位。它的实现非常简单
2. SimpleTraceInterceptor：Spring内置了很多日志跟踪的拦截器，父类`AbstractTraceInterceptor`有多个日志实现：
3. AfterReturningAdviceInterceptor：这个类其实就是将 AfterReturningAdvice 包裹成 MethodInterceptor 的适配类, 而做对应适配工作的就是 AfterReturningAdviceAdapter
4. MethodBeforeAdviceInterceptor：这个类其实就是将 MethodBeforeAdvice 包裹成 MethodInterceptor 的适配类, 而做对应适配工作的就是 MethodBeforeAdviceAdapter
5. ThrowsAdviceInterceptor：这个类其实就是将 ThrowsAdvice 包裹成 MethodInterceptor 的适配类, 而做对应适配工作的就是 ThrowsAdviceAdapter
6. TransactionInterceptor：这个类就是大名鼎鼎的注解式事务的工具类, 这个类通过获取注解在方法上的 @Transactional 注解的信息来决定是否开启事务的 MethodInterceptor

> **1、无论通过aop命名空间/AspectJ注解注释的方法, 其最终都将解析成对应的 Advice**
> **2、所有解析的 Advice 最终都将适配成 MethodInterceptor, 并在 JdkDynamicAopProxy/CglibAopProxy中进行统一调用**

##　Advisor（关联 Pointcut 和 Advise 的创建）



**Advisor 其实它就是 Pointcut 与 Advice 的组合**, Advice 是执行的方法, 而要知道方法何时执行, 则 Advice 必需与 Pointcut 组合在一起

如何将代理类关联起来？Advisor 类型的 bean 注册时机？

> **Spring如何解析被 @AspectJ注解注释的类？**
>
> 首先标注 @Component 或其他方式将 标注了 @Aspect 的类 load 到 spring容器
>
>

打个比方：

1. Advice表示建议
2. Pointcut表示建议的地点
3. Advisor表示建议者（它知道去哪建议，且知道是什么建议）



1. `PointcutAdvisor`: Spring 中常用的 Advisor, 包含一个 Pointcut 与一个 advice
2. `AspectJPointcutAdvisor`: Spring 解析 aop 命名空间时生成的 Advisor(与之对应的 Advice 是 AspectJMethodBeforeAdvice, AspectJAfterAdvice, AspectJAfterReturningAdvice, AspectJAfterThrowingAdvice, AspectJAroundAdvice, Pointcut 则是AspectJExpressionPointcut), 对于这个类的解析是在 ConfigBeanDefinitionParser
3. `InstantiationModelAwarePointcutAdvisorImpl`: 这个Advisor是在Spring解析被 @AspectJ注解注释的类时生成的 Advisor, 而这个 Advisor中的 Pointcut与Advice都是由 ReflectiveAspectJAdvisorFactory 来解析生成的(与之对应的 Advice 是 AspectJMethodBeforeAdvice, AspectJAfterAdvice, AspectJAfterReturningAdvice, AspectJAfterThrowingAdvice, AspectJAroundAdvice, Pointcut 则是AspectJExpressionPointcut)， 解析的步骤是: AnnotationAwareAspectJAutoProxyCreator.findCandidateAdvisors() -> BeanFactoryAspectJAdvisorsBuilder.buildAspectJAdvisors() -> ReflectiveAspectJAdvisorFactory.getAdvisors() -> ReflectiveAspectJAdvisorFactory.getAdvisor() 最终生成了 InstantiationModelAwarePointcutAdvisorImpl (当然包括里面的 Pointcut 与 advice 也都是由 ReflectiveAspectJAdvisorFactory 解析生成的)
4. `TransactionAttributeSourceAdvisor`: 一个基于 MethodInterceptor(其实是 TransactionInterceptor)与 TransactionAttributeSourcePointcut 的Advisor, 而这个类最常与 TransactionProxyFactoryBean使用
5. `DefaultPointcutAdvisor`： 最常用的 Advisor, 在使用编程式aop时, 很多时候会将 Advice / MethodInterceptor 转换成 DefaultPointcutAdvisor
6. `NameMatchMethodPointcutAdvisor`: 这个是在使用 NameMatchPointcutAdvisor时创建的 Advisor, 主要是通过 方法名来匹配是否执行 Advice
7. `RegexpMethodPointcutAdvisor`: 基于正则表达式来匹配 Pointcut 的 Advisor, 其中的 Pointcut 默认是 JdkRegexpMethodPointcut
8. Spring 中解析 aop:advisor 时生成的 Advisor, 见 ConfigBeanDefinitionParser.parseAdvisor
9. `BeanFactoryTransactionAttributeSourceAdvisor`: 在注解式事务编程时, 主要是由 BeanFactoryTransactionAttributeSourceAdvisor, AnnotationTransactionAttributeSource, TransactionInterceptor 组合起来进行事务的操作(PS: AnnotationTransactionAttributeSource 主要是解析方法上的 @Transactional注解, TransactionInterceptor 是个 MethodInterceptor, 是正真操作事务的地方, 而BeanFactoryTransactionAttributeSourceAdvisor 其实起着组合它们的作用); <- 与之相似的还有 BeanFactoryCacheOperationSourceAdvisor



`Advisor`是Spring AOP的顶层抽象，用来管理`Advice`和`Pointcut`（`PointcutAdvisor和切点有关，但IntroductionAdvisor和切点无关`）



注意：Advice是`aopalliance`对通知（增强器）的顶层抽象，请注意区分~~
Pointcut是Spring AOP对切点的抽象。切点的实现方式有多种，其中一种就是AspectJ

> IntroductionAdvisor与PointcutAdvisor最本质上的区别就是，IntroductionAdvisor只能应用于类级别的拦截，只能使用Introduction型的Advice。而不能像PointcutAdvisor那样，可以使用任何类型的Pointcut，以及几乎任何类型的Advice。

```java
public interface Advisor {

	//@since 5.0 Spring5以后才有的  空通知  一般当作默认值
	Advice EMPTY_ADVICE = new Advice() {};
	
	// 该Advisor 持有的通知器
	Advice getAdvice();
	// 这个有点意思：Spring所有的实现类都是return true(官方说暂时还没有应用到)
	// 注意：生成的Advisor是单例还是多例不由isPerInstance()的返回结果决定，而由自己在定义bean的时候控制
	// 理解：和类共享（per-class）或基于实例（per-instance）相关  类共享：类比静态变量   实例共享：类比实例变量
	boolean isPerInstance();

}
```



### PointcutAdvisor

* AbstractPointcutAdvisor

  * DefaultPointcutAdvisor 通用的，最强大的Advisor，将任意的 Advice 和 Pointcut 封装，本身不做其他处理

* AbstractBeanFactoryPointcutAdvisor：和bean工厂有关的PointcutAdvisor

  * DefaultBeanFactoryPointcutAdvisor

  *  BeanFactoryCacheOperationSourceAdvisor：和Cache有关

    > Spring Cache的@Cachable等注解的拦截，就是采用了它。该类位于：`org.springframework.cache.interceptor`，显然它和cache相关了。Jar包属于：Spring-context.jar

  * AsyncAnnotationAdvisor：和@Async有关,位于包为：`org.springframework.scheduling.annotation`，所属jar包为spring-context.jar

    * buildAdvice：AnnotationAsyncExecutionInterceptor

* AbstractAspectJAdvice

* AspectJPointcutAdvisor

* AspectJExpressionPointcutAdvisor





#### InstantiationModelAwarePointcutAdvisor

```java
// 默认的访问权限，显然是Spring内部自己用的
class InstantiationModelAwarePointcutAdvisorImpl
		implements InstantiationModelAwarePointcutAdvisor, AspectJPrecedenceInformation, Serializable {
	private static final Advice EMPTY_ADVICE = new Advice() {};
	// 和AspectJExpression
	private final AspectJExpressionPointcut declaredPointcut;
	..
	
	// 通知方法
	private transient Method aspectJAdviceMethod;
	
	private final AspectJAdvisorFactory aspectJAdvisorFactory;
	private final MetadataAwareAspectInstanceFactory aspectInstanceFactory;

	@Nullable
	private Advice instantiatedAdvice;
	@Nullable
	private Boolean isBeforeAdvice;
	@Nullable
	private Boolean isAfterAdvice;
	
	...
	@Override
	public boolean isPerInstance() {
		return (getAspectMetadata().getAjType().getPerClause().getKind() != PerClauseKind.SINGLETON);
	}
	@Override
	public synchronized Advice getAdvice() {
		if (this.instantiatedAdvice == null) {
			this.instantiatedAdvice = instantiateAdvice(this.declaredPointcut);
		}
		return this.instantiatedAdvice;
	}
	// advice 由aspectJAdvisorFactory去生产  懒加载的效果
	private Advice instantiateAdvice(AspectJExpressionPointcut pointcut) {
		Advice advice = this.aspectJAdvisorFactory.getAdvice(this.aspectJAdviceMethod, pointcut,
				this.aspectInstanceFactory, this.declarationOrder, this.aspectName);
		return (advice != null ? advice : EMPTY_ADVICE);
	}

	@Override
	public boolean isBeforeAdvice() {
		if (this.isBeforeAdvice == null) {
			determineAdviceType();
		}
		return this.isBeforeAdvice;
	}
	@Override
	public boolean isAfterAdvice() {
		if (this.isAfterAdvice == null) {
			determineAdviceType();
		}
		return this.isAfterAdvice;
	}
	
	// 这里解释根据@Aspect方法上标注的注解，来区分这两个字段的值的
	private void determineAdviceType() {
		AspectJAnnotation<?> aspectJAnnotation =
				AbstractAspectJAdvisorFactory.findAspectJAnnotationOnMethod(this.aspectJAdviceMethod);
		if (aspectJAnnotation == null) {
			this.isBeforeAdvice = false;
			this.isAfterAdvice = false;
		}
		else {
			switch (aspectJAnnotation.getAnnotationType()) {
				case AtAfter:
				case AtAfterReturning:
				case AtAfterThrowing:
					this.isAfterAdvice = true;
					this.isBeforeAdvice = false;
					break;
				case AtAround:
				case AtPointcut:
					this.isAfterAdvice = false;
					this.isBeforeAdvice = false;
					break;
				case AtBefore:
					this.isAfterAdvice = false;
					this.isBeforeAdvice = true;
			}
		}
	}
}
```

> 这个Advisor是在Spring解析被 @AspectJ注解注释的类时生成的 Advisor,。
>
> 而这个 Advisor中的 Pointcut与Advice都是由ReflectiveAspectJAdvisorFactory 来解析生成的(与之对应的 Advice 是 AspectJMethodBeforeAdvice, AspectJAfterAdvice, AspectJAfterReturningAdvice, AspectJAfterThrowingAdvice, AspectJAroundAdvice,
> Pointcut 则是AspectJExpressionPointcut)
>
>
>
> 自动代理创建器：AnnotationAwareAspectJAutoProxyCreator.findCandidateAdvisors() ->
> Bean工厂相关的Advisor构建器：BeanFactoryAspectJAdvisorsBuilder.buildAspectJAdvisors() ->
> ReflectiveAspectJAdvisorFactory.getAdvisors() ->
> ReflectiveAspectJAdvisorFactory.getAdvisor() 最终生成了InstantiationModelAwarePointcutAdvisorImpl(当然包括里面的 Pointcut与 advice 也都是由 ReflectiveAspectJAdvisorFactory 解析生成的)

#### AbstractAspectJAdvice

### IntroductionAdvisor：引介切面

**引入增强（Introduction Advice）的概念：一个Java类，没有实现A接口，在不修改Java类的情况下，使其具备A接口的功能。引介增强平时使用得较少，但是在特殊的场景下，它能够解决某一类问题**

它仅有一个类过滤器`ClassFilter` 而没有 `MethodMatcher`，这是因为 **`引介切面** 的切点是类级别的，而 Pointcut 的切点是方法级别的（**细粒度更细，所以更加常用**）。

```java
// 它是一个Advisor，同时也是一个IntroductionInfo 
public interface IntroductionAdvisor extends Advisor, IntroductionInfo {
	
	// 它只有ClassFilter，因为它只能作用在类层面上
	ClassFilter getClassFilter();
	// 判断这些接口，是否真的能够增强。  DynamicIntroductionAdvice#implementsInterface()方法
	void validateInterfaces() throws IllegalArgumentException;

}

// 它直接事IntroductionAdvisor的实现类。同时也是一个ClassFilter
public class DefaultIntroductionAdvisor implements IntroductionAdvisor, ClassFilter, Ordered, Serializable {
	private final Advice advice;

	private final Set<Class<?>> interfaces = new LinkedHashSet<>();
	private int order = Ordered.LOWEST_PRECEDENCE;
    
	// 构造函数们
	public DefaultIntroductionAdvisor(Advice advice) {
		this(advice, (advice instanceof IntroductionInfo ? (IntroductionInfo) advice : null));
	}
	
	// 如果IntroductionInfo 不等于null，就会把接口都add进去/
	// IntroductionInfo 的实现类有常用的：DelegatingIntroductionInterceptor和DelegatePerTargetObjectIntroductionInterceptor
	public DefaultIntroductionAdvisor(Advice advice, @Nullable IntroductionInfo introductionInfo) {
		Assert.notNull(advice, "Advice must not be null");
		this.advice = advice;
		if (introductionInfo != null) {
			Class<?>[] introducedInterfaces = introductionInfo.getInterfaces();
			if (introducedInterfaces.length == 0) {
				throw new IllegalArgumentException("IntroductionAdviceSupport implements no interfaces");
			}
			for (Class<?> ifc : introducedInterfaces) {
				addInterface(ifc);
			}
		}
	}
	
	//当然你也可以不使用IntroductionInfo，而自己手动指定了这个接口
	public DefaultIntroductionAdvisor(DynamicIntroductionAdvice advice, Class<?> intf) {
		Assert.notNull(advice, "Advice must not be null");
		this.advice = advice;
		addInterface(intf);
	}
	...
	@Override
	public void validateInterfaces() throws IllegalArgumentException {
		for (Class<?> ifc : this.interfaces) {
			if (this.advice instanceof DynamicIntroductionAdvice &&
					!((DynamicIntroductionAdvice) this.advice).implementsInterface(ifc)) {
			 throw new IllegalArgumentException("DynamicIntroductionAdvice [" + this.advice + "] " +
					 "does not implement interface [" + ifc.getName() + "] specified for introduction");
			}
		}
	}
	...
	
}
```



#### IntroductionInfo：引介信息

#### IntroductionInterceptor：引介拦截器

```java
// IntroductionInterceptor它是对MethodInterceptor的一个扩展，同时他还继承了接口DynamicIntroductionAdvice
public interface IntroductionInterceptor extends MethodInterceptor, DynamicIntroductionAdvice {
	
}

public interface DynamicIntroductionAdvice extends Advice {
	boolean implementsInterface(Class<?> intf);
}
```

### DelegatingIntroductionInterceptor

```java
public class DelegatingIntroductionInterceptor extends IntroductionInfoSupport
		implements IntroductionInterceptor {
	
	// 需要被代理的那个对象。因为这个类需要子类继承使用，所以一般都是thid
	@Nullable
	private Object delegate;
	/**
	 * Construct a new DelegatingIntroductionInterceptor.
	 * The delegate will be the subclass, which must implement
	 * additional interfaces.
	 * 访问权限事protected，显然就是说子类必须去继承这个类，然后提供空构造函数。代理类就是this
	 */
	protected DelegatingIntroductionInterceptor() {
		init(this);
	}
	// 当然，你也可以手动指定delegate
	public DelegatingIntroductionInterceptor(Object delegate) {
		init(delegate);
	}
	private void init(Object delegate) {
		Assert.notNull(delegate, "Delegate must not be null");
		this.delegate = delegate;
		implementInterfacesOnObject(delegate);
		
		// 移除调这些内部标记的接口们
		// We don't want to expose the control interface
		suppressInterface(IntroductionInterceptor.class);
		suppressInterface(DynamicIntroductionAdvice.class);
	}
	
	// 如果你要自定义一些行为：比如环绕通知之类的，子类需要复写此方法（否则没有必要了）
	@Override
	@Nullable
	public Object invoke(MethodInvocation mi) throws Throwable {
		// 判断是否是引介增强
		if (isMethodOnIntroducedInterface(mi)) {
			Object retVal = AopUtils.invokeJoinpointUsingReflection(this.delegate, mi.getMethod(), mi.getArguments());

			// 如果返回值就是delegate 本身，那就把本身返回出去
			if (retVal == this.delegate && mi instanceof ProxyMethodInvocation) {
				Object proxy = ((ProxyMethodInvocation) mi).getProxy();
				if (mi.getMethod().getReturnType().isInstance(proxy)) {
					retVal = proxy;
				}
			}
			return retVal;
		}

		return doProceed(mi);
	}
	...
}
```

### DelegatePerTargetObjectIntroductionInterceptor



## 源码解析（AOP 工作流程）

SmartInstantiationAwareBeanPostProcessor 继承自InstantiationAwareBeanPostProcessor

在bean被create之前，先会执行所有的InstantiationAwareBeanPostProcessor#postProcessBeforeInstantiation，谁第一个返回了不为null的Bean，后面就都不会执行了 。然后会再执行BeanPostProcessor#postProcessAfterInitialization

#### 第一步（创建并缓存 Advisor，处理@Aspect）

从扫描注解 @EnableAspectJAutoProxy ，在 ConfigurationClassPostProcessor 加载 

@Import 中的 AspectJAutoProxyRegistrar  进而注册 

AnnotationAwareAspectJAutoProxyCreator 然后 在 

AbstractAutoProxyCreator#postProcessBeforeInstantiation

AspectJAwareAdvisorAutoProxyCreator#shouldSkip

AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors  

BeanFactoryAdvisorRetrievalHelper#findAdvisorBeans 拿到容器中的所有 Advisor

BeanFactoryAspectJAdvisorsBuilder#buildAspectJAdvisors 得到所有标注  @Aspect的类中非标注了 @Pointcut的方法

ReflectiveAspectJAdvisorFactory#getAdvisor

生成AspectJExpressionPointcut，在 new  InstantiationModelAwarePointcutAdvisorImpl 中 instantiateAdvice

ReflectiveAspectJAdvisorFactory#getAdvice 处理 

AbstractAspectJAdvisorFactory  处理 aop 相关注解

根据不同注解 生成相对应的 AbstractAspectJAdvice子类



（InstantiationModelAwarePointcutAdvisorImpl）AspectJAdvisor （注解切面中，不同方法，生成不同 advice，不同advisor，从方法注解得到 pointcut 或标注 @Pointcut 方法名）

（pointcut ，advice）

缓存在 BeanFactoryAspectJAdvisorsBuilder 中



#### 第二步（创建代理）

AbstractAutoProxyCreator#getCustomTargetSource 若自定义 TargetSource 则生成代理，

在 postProcessAfterInstantiation，postProcessBeforeInitialization不同额外处理

否则 在 AbstractAutoProxyCreator#postProcessAfterInitialization 时生成代理

``` 
AbstractAutoProxyCreator#wrapIfNecessary
AbstractAdvisorAutoProxyCreator#getAdvicesAndAdvisorsForBean
AbstractAdvisorAutoProxyCreator#findEligibleAdvisors 
AnnotationAwareAspectJAutoProxyCreator#findCandidateAdvisors得到前面缓存的 Advisor
AopUtils.findAdvisorsThatCanApply

首先通过 Pointcut#getClassFilter 判断是否为需要代理的类型
PointcutExpressionImpl#couldMatchJoinPointsInType
考虑 待代理 class 本身为代理类，得到原被代理类型
获取需代理类型中的所有方法
IntroductionAwareMethodMatcher#matches中进行匹配 得到合适的Advisor
extendAdvisor 针对 aspectJ ExposeInvocationInterceptor

若符合被代理的条件，继续执行，否则直接返回，所有容器中的 bean 都会被判断

AbstractAutoProxyCreator#createProxy 包装成 TargetSource 进行处理
ProxyFactory#copyfrom(this) 得到 ProxyConfig 信息
AbstractAutoProxyCreator#buildAdvisors
GlobalAdvisorAdapterRegistry#getInstance#wrap 实际调用
DefaultAdvisorAdapterRegistry#wrap
若本身是 Advisor 直接返回，否则将 Advice 封装到 DefaultPointcutAdvisor 

将 TargetSource 和 Advisors 设置到 ProxyFactory
proxyFactory#getProxy
DefaultAopProxyFactory#createAopProxy(ProxyCreatorSupport)
根据情况调用 JdkDynamicAopProxy、ObjenesisCglibAopProxy 的
getProxy 底层正常创建动态代理方式创建代理

生成的代理会缓存在 AbstractAutoProxyCreator#proxyTypes(rawType,ProxyType)
存储已经过处理的 bean

AbstractAutoProxyCreator#advisedBeans(rawType,true) 有代理
AbstractAutoProxyCreator#advisedBeans(rawType,false)没代理


在 方法调用时 ， 执行 invoke
```

```
SpringProxy
resolveInterceptorNames 设置的通用 InterceptorName，顺序的问题
customizeProxyFactory(proxyFactory) 扩展点
```

#### 第三步（调用）

动态代理如何拦截实例方法执行的？

```
针对 方法
JdkDynamicAopProxy#invoke
CglibAopProxy#DynamicAdvisedInterceptor#intercept

AdvisedSupport#getInterceptorsAndDynamicInterceptionAdvice
DefaultAdvisorChainFactory#getInterceptorsAndDynamicInterceptionAdvice
首先进行匹配
从 Advisor （@Aspect 或 封装到 DefaultPointcutAdvisor ）获取 Advice
若不是 MethodInterceptor ，则由 AdviseAdapter（默认的 3 个） 将适配的advice 适配为 MethodInterceptor
封装到InterceptorAndDynamicMethodMatcher(MethodInterceptor, IntroductionAwareMethodMatcher)

CglibMethodInvocation#proceed 实际调用
ReflectiveMethodInvocation#proceed
递归调用
intercept -> MethodInvocation#proceed() -->  MethodIntercrptor#invoke(this) --> MethodInvocation#proceed() --> .. --> invokeJoinPoint


mm#isRuntime（一般返回false，测试时用？）
递归调用 InterceptorAndDynamicMethodMatcher#methodMatcher#matches
若 match 则调用 InterceptorAndDynamicMethodMatcher#interceptor#invoke


IntroductionAdvisor 的处理不太一样
```



### 创建代理

#### 激活自动代理

##### xml 方式

\<aop:aspectj-autoproxy/>

##### 注解方式

@EnableAspectJAutoProxy

```java
//Enables support for handling components marked with AspectJ's {@code @Aspect} annotation,
//similar to functionality found in Spring's {@code <aop:aspectj-autoproxy>} XML element.

//Note: {@code @EnableAspectJAutoProxy} applies to its local application context only,（说明此注解只会作用于本容器，对子、父容器是无效得）
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class)
public @interface EnableAspectJAutoProxy {

	// 决定该类采用CGLIB代理还是使用JDK的动态代理（需要实现接口），默认为false，表示使用的是JDK得动态代理技术
	boolean proxyTargetClass() default false;
	
	// @since 4.3.1 代理的暴露方式：解决内部调用不能使用代理的场景  默认为false表示不处理
	// true：这个代理就可以通过AopContext.currentProxy()获得这个代理对象的一个副本（ThreadLocal里面）,从而我们可以很方便得在Spring框架上下文中拿到当前代理对象（处理事务时很方便）
	// 必须为true才能调用AopContext得方法，否则报错：Cannot find current proxy: Set 'exposeProxy' property on Advised to 'true' to make it available.
	boolean exposeProxy() default false;

}
```



#### AspectJAutoProxyRegistrar

为容器注册 自动代理创建器

```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {
	@Override
	public void registerBeanDefinitions(
			AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
			
	//这部非常重要，就是去注册了一个基于注解的自动代理创建器（如果需要的话）  当然，下面还会着重分析
		AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

		AnnotationAttributes enableAspectJAutoProxy =
				AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
		if (enableAspectJAutoProxy != null) {
			// 若为true，表示强制指定了要使用CGLIB，那就强制告知到时候使用CGLIB的动态代理方式
			if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
				AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
			}
			// 告知，强制暴露Bean的代理对象到AopContext
			if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
				AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
			}
		}
	}
}

```



#### 注册自动代理创建器AutoProxyCreator（`AnnotationAwareAspectJAutoProxyCreator`）

#### AbstractAutoProxyCreator

在内部，Spring使用 BeanPostProcessor 让自动生成代理。基于BeanPostProcessor的自动代理创建器的实现类，将根据一些规则在容器实例化Bean时为匹配的Bean生成代理实例。代理创建器可以分为三类：

基于Bean配置名规则的自动代理生成器：允许为一组特定配置名的Bean自动创建代理实例的代理创建器，实现类为BeanNameAutoProxyCreator
基于Advisor匹配机制的自动代理创建器它会对容器中的所有Advisor进行扫描，自动将这些切面应用到匹配的Bean中，实现类是DefaultAdvisorAutoProxyCreator（它也支持前缀匹配）
基于Bean中AspectJ注解的自动代理生成器：为包含AspectJ注解的切入的Bean自动创建代理实例

```java
public abstract class AbstractAutoProxyCreator extends ProxyProcessorSupport
		implements SmartInstantiationAwareBeanPostProcessor, BeanFactoryAware {
	
	// 实现类就是我们熟悉的它：	DefaultAdvisorAdapterRegistry
	private AdvisorAdapterRegistry advisorAdapterRegistry = GlobalAdvisorAdapterRegistry.getInstance();	

	// 目标源的创建器。它有一个方法getTargetSource(Class<?> beanClass, String beanName)
	// 两个实现类：QuickTargetSourceCreator和LazyInitTargetSourceCreator
	// 它的具体使用 后面有详解
	@Nullable
	private TargetSourceCreator[] customTargetSourceCreators;
	@Nullable
	private BeanFactory beanFactory;
	...
	private final Set<String> targetSourcedBeans = Collections.newSetFromMap(new ConcurrentHashMap<>(16));
	private final Set<Object> earlyProxyReferences = Collections.newSetFromMap(new ConcurrentHashMap<>(16));
	private final Map<Object, Class<?>> proxyTypes = new ConcurrentHashMap<>(16);
	private final Map<Object, Boolean> advisedBeans = new ConcurrentHashMap<>(256);
	...
	
	// 可以自己指定Registry 
	public void setAdvisorAdapterRegistry(AdvisorAdapterRegistry advisorAdapterRegistry) {
		this.advisorAdapterRegistry = advisorAdapterRegistry;
	}
	// 可议指定多个
	public void setCustomTargetSourceCreators(TargetSourceCreator... targetSourceCreators) {
		this.customTargetSourceCreators = targetSourceCreators;
	}
	// 通用拦截器得名字。These must be bean names in the current factory
	// 这些Bean必须在当前容器内存在的~~~
	public void setInterceptorNames(String... interceptorNames) {
		this.interceptorNames = interceptorNames;
	}
	//Set whether the common interceptors should be applied before bean-specific ones
	// 默认值是true
	public void setApplyCommonInterceptorsFirst(boolean applyCommonInterceptorsFirst) {
		this.applyCommonInterceptorsFirst = applyCommonInterceptorsFirst;
	}

	//===========下面是关于BeanPostProcessor的一些实现方法============
	
	// getBeanNamesForType()的时候会根据每个BeanName去匹配类型合适的Bean，这里不例外，也会帮忙在proxyTypes找一下
	@Override
	@Nullable
	public Class<?> predictBeanType(Class<?> beanClass, String beanName) {
		if (this.proxyTypes.isEmpty()) {
			return null;
		}
		Object cacheKey = getCacheKey(beanClass, beanName);
		return this.proxyTypes.get(cacheKey);
	}

	// getEarlyBeanReference()它是为了解决单例bean之间的循环依赖问题，提前将代理对象暴露出去
	@Override
	public Object getEarlyBeanReference(Object bean, String beanName) throws BeansException {
		Object cacheKey = getCacheKey(bean.getClass(), beanName);
		if (!this.earlyProxyReferences.contains(cacheKey)) {
			this.earlyProxyReferences.add(cacheKey);
		}
		return wrapIfNecessary(bean, beanName, cacheKey);
	}

	// 不做构造函数检测，返回null 让用空构造初始化吧
	@Override
	@Nullable
	public Constructor<?>[] determineCandidateConstructors(Class<?> beanClass, String beanName) throws BeansException {
		return null;
	}
	
	// 这个很重要，在Bean实例化之前，先给一个机会，看看缓存里有木有，有就直接返回得了
	// 简单的说：其主要目的在于如果用户使用了自定义的TargetSource对象，则直接使用该对象生成目标对象，而不会使用Spring的默认逻辑生成目标对象
	// (并且这里会判断各个切面逻辑是否可以应用到当前bean上)
	@Override
	public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
		Object cacheKey = getCacheKey(beanClass, beanName);
		
		// beanName无效或者targetSourcedBeans里不包含此Bean
		if (!StringUtils.hasLength(beanName) || !this.targetSourcedBeans.contains(beanName)) {
			
			//advisedBeans：已经被通知了的（被代理了的）Bean~~~~  如果在这里面  也返回null 
			if (this.advisedBeans.containsKey(cacheKey)) {
				return null;
			}
			// isInfrastructureClass:Advice、Pointcut、Advisor、AopInfrastructureBean的子类，表示是框架所属的Bean
			// shouldSkip:默认都是返回false的。AspectJAwareAdvisorAutoProxyCreator重写此方法：只要存在一个Advisor   ((AspectJPointcutAdvisor) advisor).getAspectName().equals(beanName)成立  就返回true
			if (isInfrastructureClass(beanClass) || shouldSkip(beanClass, beanName)) {	
				// 所以这里会把我们所有的Advice、Pointcut、Advisor、AopInfrastructureBean等Bean都装进来
				this.advisedBeans.put(cacheKey, Boolean.FALSE);
				return null;
			}
		}

		//到这，只有在TargetSource中没有进行缓存，并且应该被切面逻辑环绕，但是目前还未生成代理对象的bean才会通过此方法

		// Create proxy here if we have a custom TargetSource.
		// 如果我们有TargetSourceCreator，这里就会创建一个代理对象
		// getCustomTargetSource逻辑：存在TargetSourceCreator  并且 beanFactory.containsBean(beanName)  然后遍历所有的TargetSourceCreator，调用getTargetSource谁先创建不为null就终止
		TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
		// 若创建好了这个代理对象，继续进一步的操作：：：
		if (targetSource != null) {
			if (StringUtils.hasLength(beanName)) {
			  	// 缓存起来
				this.targetSourcedBeans.add(beanName);
			}
			
			//getAdvicesAndAdvisorsForBean：方法判断当前bean是否需要进行代理，若需要则返回满足条件的Advice或者Advisor集合
			// 这个方法由子类实现，AbstractAdvisorAutoProxyCreator和BeanNameAutoProxyCreator  代表中两种不同的代理方式
			Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
			
			// 顾名思义，就是根据目标对象创建代理对象的核心逻辑了 下面详解
			Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
			
			// 把创建好的代理  缓存~~~
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}

		return null;
	}

	// 这个方法也很重要，若我们自己要实现一个TargetSourceCreator ，就可议实现我们自定义的逻辑了
	// 这里条件苛刻：customTargetSourceCreators 必须不为null
	// 并且容器内还必须有这个Bean：beanFactory.containsBean(beanName)    备注：此BeanName指的即将需要被代理得BeanName，而不是TargetSourceCreator 的BeanName
	//下面会介绍我们自己自定义一个TargetSourceCreator 来实现我们自己的逻辑
	@Nullable
	protected TargetSource getCustomTargetSource(Class<?> beanClass, String beanName) {
		// We can't create fancy target sources for directly registered singletons.
		if (this.customTargetSourceCreators != null &&
				this.beanFactory != null && this.beanFactory.containsBean(beanName)) {
			for (TargetSourceCreator tsc : this.customTargetSourceCreators) {
				TargetSource ts = tsc.getTargetSource(beanClass, beanName);
				if (ts != null) {
					// Found a matching TargetSource.
					if (logger.isDebugEnabled()) {
						logger.debug("TargetSourceCreator [" + tsc +
								" found custom TargetSource for bean with name '" + beanName + "'");
					}
					return ts;
				}
			}
		}

		// No custom TargetSource found.
		return null;
	}

	// 这三个方法，没做什么动作~~
	@Override
	public boolean postProcessAfterInstantiation(Object bean, String beanName) {
		return true;
	}
	@Override
	public PropertyValues postProcessPropertyValues(
			PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) {
		return pvs;
	}
	@Override
	public Object postProcessBeforeInitialization(Object bean, String beanName) {
		return bean;
	}


	// 代理是通过AbstractAutoProxyCreator中的postProcessAfterInitialization()创建的
	// 因此这个方法是蛮重要的，主要是wrapIfNecessary()方法会特别的重要
	// earlyProxyReferences缓存：该缓存用于保存已经创建过代理对象的cachekey，**避免重复创建**
	@Override
	public Object postProcessAfterInitialization(@Nullable Object bean, String beanName) throws BeansException {
		if (bean != null) {
			Object cacheKey = getCacheKey(bean.getClass(), beanName);
			if (!this.earlyProxyReferences.contains(cacheKey)) {
				return wrapIfNecessary(bean, beanName, cacheKey);
			}
		}
		return bean;
	}

	// ============wrapIfNecessary方法==============
	protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
		// 若此Bean已经在targetSourcedBeans里，说明已经被代理过，那就直接返回即可
		// (postProcessBeforeInstantiation()中成功创建的代理对象都会将beanName加入到targetSourceBeans中)
		if (StringUtils.hasLength(beanName) && this.targetSourcedBeans.contains(beanName)) {
			return bean;
		}
		
		// 如果该Bean基础框架Bean或者免代理得Bean，那也不处理
		if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
			return bean;
		}
		// 逻辑同上，对于实现了Advice，Advisor，AopInfrastructureBean接口的bean，都认为是spring aop的基础框架类，不能对他们创建代理对象，
		// 同时子类也可以覆盖shouldSkip方法来指定不对哪些bean进行代理
		if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
			this.advisedBeans.put(cacheKey, Boolean.FALSE);
			return bean;
		}

		// Create proxy if we have advice.
		// getAdvicesAndAdvisorsForBean 该方法由子类实现，如国有Advice切面切进去了，我们就要给他代理
		//根据 getAdvicesAndAdvisorsForBean() 方法的具体实现的不同，AbstractAutoProxyCreator 又分成了两类自动代理机制
		Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
		
		// 需要代理，那就进来给它创建一个代理对象吧
		if (specificInterceptors != DO_NOT_PROXY) {
			// 缓存起来，赋值为true，说明此key是被代理了的
			this.advisedBeans.put(cacheKey, Boolean.TRUE);
			
			// 创建这个代理对象
			Object proxy = createProxy(bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
			// 创建好后缓存起来  避免重复创建
			this.proxyTypes.put(cacheKey, proxy.getClass());
			return proxy;
		}
		
		// 不需要代理，也把这种不需要代理的对象给与缓存起来  赋值为false
		this.advisedBeans.put(cacheKey, Boolean.FALSE);
		return bean;
	}

	// 创建代理对象  specificInterceptors：作用在这个Bean上的增强器们
	// 这里需要注意的地方：入参是 targetSource  而不是 target
	// 所以最终代理的是``每次AOP代理处理方法调用时，目标实例都会用到TargetSource实现``
	protected Object createProxy(Class<?> beanClass, @Nullable String beanName,
			@Nullable Object[] specificInterceptors, TargetSource targetSource) {

		if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
			AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
		}
		
		// 这个我们非常熟悉了ProxyFactory 创建代理对象的三大方式之一
		ProxyFactory proxyFactory = new ProxyFactory();
		// 复制当前类的相关配置，因为当前类它也是个ProxyConfig
		proxyFactory.copyFrom(this);

		// 看看是否是基于类的代理（CGLIB），若表面上是基于接口的代理  我们还需要进一步去检测
		if (!proxyFactory.isProxyTargetClass()) {
			// shouldProxyTargetClass方法用于判断是否应该使用targetClass类而不是接口来进行代理
			// 默认实现为和该bean定义是否属性值preserveTargetClass为true有关。默认情况下都不会有此属性值的~~~~~
			if (shouldProxyTargetClass(beanClass, beanName)) {
				proxyFactory.setProxyTargetClass(true);
			}
			
			// 到此处，上面说了，就是把这个类实现的接口们，都放进proxyFactory（当然也会处理一些特殊的接口~~~不算数的）
			else {
				evaluateProxyInterfaces(beanClass, proxyFactory);
			}
		}
		
		// buildAdvisors：整理合并得到最终的advisors （毕竟interceptorNames 还指定了一些拦截器的）
		// 至于调用的先后顺序，通过applyCommonInterceptorsFirst参数可以进行设置，若applyCommonInterceptorsFirst为true，interceptorNames属性指定的Advisor优先调用。默认为true
		Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
		// 添加进工厂里
		proxyFactory.addAdvisors(advisors);
		// 把targetSource放进去  TargetSource的实现方式有多种 后面会介绍
		proxyFactory.setTargetSource(targetSource);

		// 这个方法是交给子类的，子类可以继续去定制此proxyFactory（Spring内部并没有搭理它）
		customizeProxyFactory(proxyFactory);
		
		// 沿用this得freezeProxy的属性值
		proxyFactory.setFrozen(this.freezeProxy);
		
		// 设置preFiltered的属性值，默认是false。子类：AbstractAdvisorAutoProxyCreator修改为true
		// preFiltered：字段意思为：是否已为特定目标类筛选Advisor
		// 这个字段和DefaultAdvisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice获取所有的Advisor有关
		//CglibAopProxy和JdkDynamicAopProxy都会调用此方法，然后递归执行所有的Advisor的
		if (advisorsPreFiltered()) {
			proxyFactory.setPreFiltered(true);
		}
		
		// getProxyClassLoader():调用者可议指定  否则为：ClassUtils.getDefaultClassLoader()
		return proxyFactory.getProxy(getProxyClassLoader());
	}

	// 下面，只剩一个重要的方法：buildAdvisors()没有解释了
	protected Advisor[] buildAdvisors(@Nullable String beanName, @Nullable Object[] specificInterceptors) {
		// 解析interceptorNames而来得Advisor数组~~~
		Advisor[] commonInterceptors = resolveInterceptorNames();

		// 注意：此处用得事Object
		List<Object> allInterceptors = new ArrayList<>();

		if (specificInterceptors != null) {
			allInterceptors.addAll(Arrays.asList(specificInterceptors));
				
			// 若解析它来的有内容
			if (commonInterceptors.length > 0) {
				// 放在头部  也就是最上面
				if (this.applyCommonInterceptorsFirst) {
					allInterceptors.addAll(0, Arrays.asList(commonInterceptors));
				}
				// 放在末尾
				else {
					allInterceptors.addAll(Arrays.asList(commonInterceptors));
				}
			}
		}

		// 把每一个Advisor都用advisorAdapterRegistry.wrap()包装一下~~~~
		// 注意wrap方法，默认只支持那三种类型的Advice转换为Advisor的~~~
		Advisor[] advisors = new Advisor[allInterceptors.size()];
		for (int i = 0; i < allInterceptors.size(); i++) {
			advisors[i] = this.advisorAdapterRegistry.wrap(allInterceptors.get(i));
		}
		return advisors;
	}

	// 处理interceptorNames，去容器内找出来
	private Advisor[] resolveInterceptorNames() {
		BeanFactory bf = this.beanFactory;
		ConfigurableBeanFactory cbf = (bf instanceof ConfigurableBeanFactory ? (ConfigurableBeanFactory) bf : null);
		List<Advisor> advisors = new ArrayList<>();
		for (String beanName : this.interceptorNames) {
			// 排除一些情况：此工厂不是ConfigurableBeanFactory或者该Bean不在创建中
			if (cbf == null || !cbf.isCurrentlyInCreation(beanName)) {
				Assert.state(bf != null, "BeanFactory required for resolving interceptor names");
				
				// 拿到这个Bean，然后使用advisorAdapterRegistry把它适配一下即可~~~
				Object next = bf.getBean(beanName);
				advisors.add(this.advisorAdapterRegistry.wrap(next));
			}
		}
		return advisors.toArray(new Advisor[0]);
	}

}
```



##### AbstractAutoProxyCreator#getCustomTargetSource()

###### TargetSourceCreator

从Spring自带的实现中可议看出，都是和Spring容器相关的实现。LazyInitTargetSourceCreator：和@Lazy属性有关（LazyInitTargetSource）。QuickTargetSourceCreator：可能来自CommonsPool2TargetSource、可能来自ThreadLocalTargetSource、可能来自PrototypeTargetSource（和BeanName有关），可能是null。

> 绝大多数情况下调用者不会自己去实现TargetSourceCreator，而是Spring采用默认的SingletonTargetSource去生产AOP对象 `LazyInitTargetSource在JMX的MBean中使用较为广泛，所以是要了解的`

```java
public class QuickTargetSourceCreator extends AbstractBeanFactoryBasedTargetSourceCreator {

	// 可以看出，它的分类事根据BeanName以xxx开头来分辨的~~~这是一种约束
	public static final String PREFIX_COMMONS_POOL = ":";
	public static final String PREFIX_THREAD_LOCAL = "%";
	public static final String PREFIX_PROTOTYPE = "!";

	@Override
	@Nullable
	protected final AbstractBeanFactoryBasedTargetSource createBeanFactoryBasedTargetSource(
			Class<?> beanClass, String beanName) {

		if (beanName.startsWith(PREFIX_COMMONS_POOL)) {
			CommonsPool2TargetSource cpts = new CommonsPool2TargetSource();
			cpts.setMaxSize(25);
			return cpts;
		}
		else if (beanName.startsWith(PREFIX_THREAD_LOCAL)) {
			return new ThreadLocalTargetSource();
		}
		else if (beanName.startsWith(PREFIX_PROTOTYPE)) {
			return new PrototypeTargetSource();
		}
		else {
			// No match. Don't create a custom target source.
			return null;
		}
	}

}
```



##### AbstractAutoProxyCreator#getAdvicesAndAdvisorsForBean

###### BeanNameAutoProxyCreator

```java
//@since 10.10.2003  可以看出这个类出现得非常早
public class BeanNameAutoProxyCreator extends AbstractAutoProxyCreator {
	@Nullable
	private List<String> beanNames;

	public void setBeanNames(String... beanNames) {
		Assert.notEmpty(beanNames, "'beanNames' must not be empty");
		this.beanNames = new ArrayList<>(beanNames.length);
		for (String mappedName : beanNames) {
			// 对mappedName做取出空白处理
			this.beanNames.add(StringUtils.trimWhitespace(mappedName));
		}
	}
	// simpleMatch并不是完整的正则。但是支持*这种通配符，其余的不支持哦
	protected boolean isMatch(String beanName, String mappedName) {
		return PatternMatchUtils.simpleMatch(mappedName, beanName);
	}


	// 这里面注意一点：BeanNameAutoProxyCreator的此方法并没有去寻找Advisor，所以需要拦截的话
	// 只能依靠：setInterceptorNames()来指定拦截器。它是根据名字去Bean容器里取的
	@Override
	@Nullable
	protected Object[] getAdvicesAndAdvisorsForBean(
			Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {

		if (this.beanNames != null) {
			for (String mappedName : this.beanNames) {
				// 显然这里面，如果你针对的是FactoryBean,也是兼容的~~~
				if (FactoryBean.class.isAssignableFrom(beanClass)) {
					if (!mappedName.startsWith(BeanFactory.FACTORY_BEAN_PREFIX)) {
						continue;
					}
					// 对BeanName进行处理，去除掉第一个字符
					mappedName = mappedName.substring(BeanFactory.FACTORY_BEAN_PREFIX.length());
				}
				
				// 匹配就返回PROXY_WITHOUT_ADDITIONAL_INTERCEPTORS 而不是再返回null了
				if (isMatch(beanName, mappedName)) {
					return PROXY_WITHOUT_ADDITIONAL_INTERCEPTORS;
				}
				
				// 这里需要注意的是，如国存在Bean工厂，哪怕任意一个alias匹配都是可以的~~~
				BeanFactory beanFactory = getBeanFactory();
				if (beanFactory != null) {
					String[] aliases = beanFactory.getAliases(beanName);
					for (String alias : aliases) {
						if (isMatch(alias, mappedName)) {
							return PROXY_WITHOUT_ADDITIONAL_INTERCEPTORS;
						}
					}
				}
			}
		}
		return DO_NOT_PROXY;
	}
}
```

###### AbstractAdvisorAutoProxyCreator

它是Spring2.0提供的（2.0版本2006年才发布哦~~~）
**顾名思义，它和Advisor有关（只有被切入的类，才会给它创建一个代理类）**，它的核心方法是实现了父类的：`getAdvicesAndAdvisorsForBean`来获取`Advisor`们

```java
public abstract class AbstractAdvisorAutoProxyCreator extends AbstractAutoProxyCreator {

	// 这个类是重点，后面会详细介绍
	@Nullable
	private BeanFactoryAdvisorRetrievalHelper advisorRetrievalHelper;

	// 重写了setBeanFactory方法，事需要保证bean工厂必须是ConfigurableListableBeanFactory
	@Override
	public void setBeanFactory(BeanFactory beanFactory) {
		super.setBeanFactory(beanFactory);
		if (!(beanFactory instanceof ConfigurableListableBeanFactory)) {
			throw new IllegalArgumentException(
					"AdvisorAutoProxyCreator requires a ConfigurableListableBeanFactory: " + beanFactory);
		}
		// 就这一句话：this.advisorRetrievalHelper = new BeanFactoryAdvisorRetrievalHelperAdapter(beanFactory)
		// 对Helper进行初始化，找advisor最终事委托给他了的
		// BeanFactoryAdvisorRetrievalHelperAdapter继承自BeanFactoryAdvisorRetrievalHelper,为私有内部类，主要重写了isEligibleBean（）方法，调用.this.isEligibleAdvisorBean(beanName)方法
		initBeanFactory((ConfigurableListableBeanFactory) beanFactory);
	}

	// 这是复写父类的方法，也是实现代理方式。找到作用在这个Bean里面的切点方法
	// 当然 最终最终事委托给BeanFactoryAdvisorRetrievalHelper去做的
	@Override
	@Nullable
	protected Object[] getAdvicesAndAdvisorsForBean(
			Class<?> beanClass, String beanName, @Nullable TargetSource targetSource) {
		
		// findEligibleAdvisors：显然这个是具体的实现方法了。
		// eligible：合格的  合适的
		List<Advisor> advisors = findEligibleAdvisors(beanClass, beanName);
		if (advisors.isEmpty()) {
			return DO_NOT_PROXY;
		}
		return advisors.toArray();
	}
	
	// 找出合适的Advisor们~~~  主要分了下面几步
	protected List<Advisor> findEligibleAdvisors(Class<?> beanClass, String beanName) {
		
		// 首先找出所有的候选的Advisors，（根据名字判断）实现见下面~~~~
		List<Advisor> candidateAdvisors = findCandidateAdvisors();
		// 对上面找到的候选的Advisors们，进行过滤操作~~~  看看Advisor能否被用在Bean上（根据Advisor的PointCut判断）
		// 主要依赖于AopUtils.findAdvisorsThatCanApply()方法  在工具类讲解中有详细分析的
		// 逻辑简单概述为：看目标类是不是符合代理对象的条件，如果符合就把Advisor加到集合中，最后返回集合
		// 简单的说：它就是会根据ClassFilter和MethodMatcher等等各种匹配。（但凡只有有一个方法被匹配上了，就会给他创建代理类了）
		// 方法用的ReflectionUtils.getAllDeclaredMethods，**因此哪怕是私有方法，匹配上都会给创建的代理对象，这点务必要特别特别的注意**
		List<Advisor> eligibleAdvisors = findAdvisorsThatCanApply(candidateAdvisors, beanClass, beanName);
		
		//提供一个钩子。子类可以复写此方法  然后对eligibleAdvisors进行处理（增加/删除/修改等等）
		// AspectJAwareAdvisorAutoProxyCreator提供了实现
		extendAdvisors(eligibleAdvisors);
		
		// 如果最终还有，那就排序吧 
		if (!eligibleAdvisors.isEmpty()) {
			// 默认排序方式：AnnotationAwareOrderComparator.sort()排序  这个排序和Order接口有关~~~
			// 但是子类：AspectJAwareAdvisorAutoProxyCreator有复写此排序方法，需要特别注意~~~
			eligibleAdvisors = sortAdvisors(eligibleAdvisors);
		}
		return eligibleAdvisors;
	}


	// 找到候选的Advisor们~~~~   抽象类自己的实现，是直接把这件事委托给了advisorRetrievalHelper
	// 关于它的具体逻辑  后文问详细分析  毕竟属于核心逻辑
	// AnnotationAwareAspectJAutoProxyCreator对它有复写
	protected List<Advisor> findCandidateAdvisors() {
		Assert.state(this.advisorRetrievalHelper != null, "No BeanFactoryAdvisorRetrievalHelper available");
		return this.advisorRetrievalHelper.findAdvisorBeans();
	}

	// 判断给定的BeanName这个Bean，是否是合格的(BeanFactoryAdvisorRetrievalHelper里会用到这个属性)
	// 其中：DefaultAdvisorAutoProxyCreator和InfrastructureAdvisorAutoProxyCreator有复写
	protected boolean isEligibleAdvisorBean(String beanName) {
		return true;
	}

	// 此处复写了父类的方法，返回true了，表示
	@Override
	protected boolean advisorsPreFiltered() {
		return true;
	}
}
```

###### AopUtils#findAdvisorsThatCanApply

###### IntroductionAwareMethodMatcher#matches

###### ProxyMethodInvocation#getProxy

##### DefaultAdvisorAutoProxyCreator

```java
@Override
	protected boolean isEligibleAdvisorBean(String beanName) {
		return (this.beanFactory != null && this.beanFactory.containsBeanDefinition(beanName) &&
				this.beanFactory.getBeanDefinition(beanName).getRole() == BeanDefinition.ROLE_INFRASTRUCTURE);
	}
```



##### InfrastructureAdvisorAutoProxyCreator

Spring给自己内部使用的一个自动代理创建器。这个类在`@EnableTransactionManagement`事务相关里会再次提到（它的AutoProxyRegistrar就是向容器注册了它）

作用非常简单：`主要是读取Advisor类，并对符合的bean进行二次代理`

#### AspectJAwareAdvisorAutoProxyCreator

#### AnnotationAwareAspectJAutoProxyCreator

@EnableAspectJAutoProxy

> AspectJAwareAdvisorAutoProxyCreator它用于xml配置版的AspectJ切面自动代理创建（\<aop:config/>）
> AnnotationAwareAspectJAutoProxyCreator用于基于注解的自动代理创建(\<aop:aspectj-autoproxy/> 或 @EnableAspectJAutoProxy)

```java
// @since 2.0
public class AnnotationAwareAspectJAutoProxyCreator extends AspectJAwareAdvisorAutoProxyCreator {

	@Nullable
	private List<Pattern> includePatterns;
	//唯一实现类：ReflectiveAspectJAdvisorFactory
	// 作用：基于@Aspect时,创建Spring AOP的Advice
	// 里面会对标注这些注解Around.class, Before.class, After.class, AfterReturning.class, AfterThrowing.class的方法进行排序
	// 然后把他们都变成Advisor( getAdvisors()方法 )
	@Nullable
	private AspectJAdvisorFactory aspectJAdvisorFactory;
	//该工具类用来从bean容器，也就是BeanFactory中获取所有使用了@AspectJ注解的bean
	//就是这个方法：aspectJAdvisorsBuilder.buildAspectJAdvisors()
	@Nullable
	private BeanFactoryAspectJAdvisorsBuilder aspectJAdvisorsBuilder;


	// 很显然，它还支持我们自定义一个正则的模版
	// isEligibleAspectBean()该方法使用此模版，从而决定使用哪些Advisor
	public void setIncludePatterns(List<String> patterns) {
		this.includePatterns = new ArrayList<>(patterns.size());
		for (String patternText : patterns) {
			this.includePatterns.add(Pattern.compile(patternText));
		}
	}
	
	// 可以自己实现一个AspectJAdvisorFactory  否则用默认的ReflectiveAspectJAdvisorFactory
	public void setAspectJAdvisorFactory(AspectJAdvisorFactory aspectJAdvisorFactory) {
		Assert.notNull(aspectJAdvisorFactory, "AspectJAdvisorFactory must not be null");
		this.aspectJAdvisorFactory = aspectJAdvisorFactory;
	}

	// 此处一定要记得调用：super.initBeanFactory(beanFactory);
	@Override
	protected void initBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		super.initBeanFactory(beanFactory);
		if (this.aspectJAdvisorFactory == null) {
			this.aspectJAdvisorFactory = new ReflectiveAspectJAdvisorFactory(beanFactory);
		}
		this.aspectJAdvisorsBuilder = new BeanFactoryAspectJAdvisorsBuilderAdapter(beanFactory, this.aspectJAdvisorFactory);
	}


	// 拿到所有的候选的advisor们。请注意：这里没有先调用了父类的super.findCandidateAdvisors()  去容器里找出来一些
	// 然后，然后自己又通过aspectJAdvisorsBuilder.buildAspectJAdvisors()  解析@Aspect的方法得到一些Advisor
	@Override
	protected List<Advisor> findCandidateAdvisors() {
		List<Advisor> advisors = super.findCandidateAdvisors();
		if (this.aspectJAdvisorsBuilder != null) {
			advisors.addAll(this.aspectJAdvisorsBuilder.buildAspectJAdvisors());
		}
		return advisors;
	}
	
	// 加了中类型   如果该Bean自己本身就是一个@Aspect， 那也认为是基础主键，不要切了
	@Override
	protected boolean isInfrastructureClass(Class<?> beanClass) {
		return (super.isInfrastructureClass(beanClass) ||
				(this.aspectJAdvisorFactory != null && this.aspectJAdvisorFactory.isAspect(beanClass)));
	}

	// 拿传入的正则模版进行匹配（没传就返回true，所有的Advisor都会生效）
	protected boolean isEligibleAspectBean(String beanName) {
		if (this.includePatterns == null) {
			return true;
		}
		else {
			for (Pattern pattern : this.includePatterns) {
				if (pattern.matcher(beanName).matches()) {
					return true;
				}
			}
			return false;
		}
	}
	...
}
```

> ExposeInvocationInterceptor 的作用是用于暴露 MethodInvocation 对象到 ThreadLocal 中，其名字也体现出了这一点。如果其他地方需要当前的 MethodInvocation 对象，直接通过调用静态方法 ExposeInvocationInterceptor.currentInvocation 方法取出。那哪些地方会用到呢？？？？
> AspectJExpressionPointcut#matches就有用到





#### BeanFactoryAspectJAdvisorsBuilder#buildAspectAdvisor

BeanFactoryAdvisorRetrievalHelper：从Bean工厂检索出Advisor们

##### 重写 setBeanFactory

```java
// @since 2.0.2
public class BeanFactoryAdvisorRetrievalHelper {

	private final ConfigurableListableBeanFactory beanFactory;
	// 本地会做一个简单的字段缓存
	@Nullable
	private String[] cachedAdvisorBeanNames;
	...
	
	// 这里显然就是核心方法了
	public List<Advisor> findAdvisorBeans() {
		String[] advisorNames = null;
		
		synchronized (this) {
			advisorNames = this.cachedAdvisorBeanNames;
			if (advisorNames == null) {
				// 这里不会实例化FactoryBeans
				// 我们需要保留所有常规bean未初始化以允许自动代理创建者应用于它们
				// 注意此处：连祖先容器里面的Bean都会拿出来  (这个方法平时我们也可以使用)
				advisorNames = BeanFactoryUtils.beanNamesForTypeIncludingAncestors(
						this.beanFactory, Advisor.class, true, false);
				this.cachedAdvisorBeanNames = advisorNames;
			}
		}
		
		// 如果容器里面没有任何的advisor 那就拉倒吧
		if (advisorNames.length == 0) {
			return new LinkedList<>();
		}

		List<Advisor> advisors = new LinkedList<>();
		for (String name : advisorNames) {
			
			// isEligibleBean：表示这个bean是否是合格的，默认是true
			// 但上面书说了InfrastructureAdvisorAutoProxyCreator和DefaultAdvisorAutoProxyCreator都做了对应的复写
			if (isEligibleBean(name)) {
				// 如果当前Bean正在创建中  那好  就啥也不做吧
				if (this.beanFactory.isCurrentlyInCreation(name)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipping currently created advisor '" + name + "'");
					}
				}
				// 否则就把这个Advisor加入到List里面，是个合法的
				else {
					try {
						advisors.add(this.beanFactory.getBean(name, Advisor.class));
					}
					catch (BeanCreationException ex) {
						... 
						continue;
					}
				}
			}
		}
		return advisors;
	}
	
	protected boolean isEligibleBean(String beanName) {
		return true;
	}

}
```

1. `SpringAOP应尽量避免自己创建AutoProxyCreator`
2. `避免使用低级别的AOP API`





### 调用代理









































































































































































































