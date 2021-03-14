

## TransactionStatus

它表示`事务的状态`，若有需要，事务代码可以使用它来检索状态信息，以编程方式请求回滚（而不是抛出导致隐式回滚的异常）

> The `TransactionStatus` interface provides a simple way for transactional code to control transaction execution and query transaction status. The concepts should be familiar, as they are common to all transaction APIs
>
> TransactionStatus 接口为事务代码提供了一种控制事务执行和查询事务状态的简单方法

```java
// 可以看到它继承自SavepointManager，所以它也会处理还原点
public interface TransactionStatus extends SavepointManager, Flushable {

	// 判断当前的事务是否是新事务
	boolean isNewTransaction();
	// 判断该事务里面是否含有还原点~
	boolean hasSavepoint();
	
	// Set the transaction rollback-only
	// 这是了这个，事务的唯一结果是进行回滚。因此如果你在外层给try catche住不让事务回滚，就会抛出你可能常见的异常：
	// Transaction rolled back because it has been marked as rollback-only
	void setRollbackOnly();
	boolean isRollbackOnly();
	
	//将基础会话刷新到数据存储 for example, all affected Hibernate/JPA sessions
	@Override 
	void flush();
	// Return whether this transaction is completed
	// 不管是commit或者rollback了都算结束了~~~
	boolean isCompleted();
}
```

### AbstractTransactionStatus

```java
// @since 1.2.3 
public abstract class AbstractTransactionStatus implements TransactionStatus {
	// 两个标志位
	private boolean rollbackOnly = false;
	private boolean completed = false;

	// 一个还原点
	@Nullable
	private Object savepoint;

	// 把该属性值保存为true
	@Override
	public void setRollbackOnly() {
		this.rollbackOnly = true;
	}
	// 注意此处并不是直接读取
	@Override
	public boolean isRollbackOnly() {
		return (isLocalRollbackOnly() || isGlobalRollbackOnly());
	}
	public boolean isLocalRollbackOnly() {
		return this.rollbackOnly;
	}
	// Global这里返回false 但是子类DefaultTransactionStatus复写了此方法
	public boolean isGlobalRollbackOnly() {
		return false;
	}
	// 啥都没做  也是由子类去实现
	@Override
	public void flush() {
	}
	...
}
```

#### SimpleTransactionStatus

```java
public class SimpleTransactionStatus extends AbstractTransactionStatus {
	private final boolean newTransaction;

	// 构造函数
	public SimpleTransactionStatus() {
		this(true);
	}
	public SimpleTransactionStatus(boolean newTransaction) {
		this.newTransaction = newTransaction;
	}
	@Override
	public boolean isNewTransaction() {
		return this.newTransaction;
	}
}
```

#### DefaultTransactionStatus(Spring默认使用的事务状态)

```java
public class DefaultTransactionStatus extends AbstractTransactionStatus {
	// 它有很多的标志位，成员变量  
	@Nullable
	private final Object transaction;
	// 是否是新事务
	private final boolean newTransaction;
	// 如果为给定事务打开了新的事务同步  该值为true
	private final boolean newSynchronization;
	// 该事务是否标记为了只读
	private final boolean readOnly;
	private final boolean debug;


	@Nullable
	private final Object suspendedResources;

	// 它的唯一构造函数如下：
	public DefaultTransactionStatus(@Nullable Object transaction, boolean newTransaction, boolean newSynchronization,
			boolean readOnly, boolean debug, @Nullable Object suspendedResources) {
		this.transaction = transaction;
		this.newTransaction = newTransaction;
		this.newSynchronization = newSynchronization;
		this.readOnly = readOnly;
		this.debug = debug;
		this.suspendedResources = suspendedResources;
	}
	
	// 直接把底层事务返回
	public Object getTransaction() {
		Assert.state(this.transaction != null, "No transaction active");
		return this.transaction;
	}
	public boolean hasTransaction() {
		return (this.transaction != null);
	}
	
	// 首先
	@Override
	public boolean isNewTransaction() {
		return (hasTransaction() && this.newTransaction);
	}
	...
	// 都由SmartTransactionObject去处理  该接口的实现类有：
	// JdbcTransactionObjectSupport和JtaTransactionObject（分布式事务）
	public boolean isGlobalRollbackOnly() {
		return ((this.transaction instanceof SmartTransactionObject) &&
				((SmartTransactionObject) this.transaction).isRollbackOnly());
	}
	@Override
	public void flush() {
		if (this.transaction instanceof SmartTransactionObject) {
			((SmartTransactionObject) this.transaction).flush();
		}
	}

}
```







## `TransactionAttribute(extends TransactionDefinition)`

```java
// 它继承自TransactionDefinition ，所有可以定义事务的基础属性
public interface TransactionAttribute extends TransactionDefinition {
	 // 返回与此事务属性关联的限定符值
	//@since 3.0
	@Nullable
	String getQualifier();
	// Should we roll back on the given exception?
	boolean rollbackOn(Throwable ex);
}
```

### DefaultTransactionAttribute

```java
// 它继承自DefaultTransactionDefinition 
public class DefaultTransactionAttribute extends DefaultTransactionDefinition implements TransactionAttribute {

	@Nullable
	private String qualifier;
	@Nullable
	private String descriptor;

	// 你自己也可以自定义一个TransactionAttribute other 来替换掉一些默认行为
	public DefaultTransactionAttribute() {
		super();
	}
    
	public DefaultTransactionAttribute(TransactionAttribute other) {
		super(other);
	}

	// @since 3.0
	public void setQualifier(@Nullable String qualifier) {
		this.qualifier = qualifier;
	}
    
	@Override
	@Nullable
	public String getQualifier() {
		return this.qualifier;
	}
	...
        
	// 可以清晰的看到：默认只回滚RuntimeException 或者 Error(比如OOM这种)
	@Override
	public boolean rollbackOn(Throwable ex) {
		return (ex instanceof RuntimeException || ex instanceof Error);
	}
	
}
```

#### RuleBasedTransactionAttribute

```java
public class RuleBasedTransactionAttribute extends DefaultTransactionAttribute implements Serializable {
	/** Prefix for rollback-on-exception rules in description strings. */
	public static final String PREFIX_ROLLBACK_RULE = "-";
	/** Prefix for commit-on-exception rules in description strings. */
	public static final String PREFIX_COMMIT_RULE = "+";

	// RollbackRuleAttribute：它是个实体类，确定给定异常是否应导致回滚的规则
	// 相当于封装了这个规则的一个实体，内部封装一个异常  提供一个实例变量： 这个变量相当于回滚规则为只回滚RuntimeException
	// public static final RollbackRuleAttribute ROLLBACK_ON_RUNTIME_EXCEPTIONS = new RollbackRuleAttribute(RuntimeException.class);
	// 所以此类最重要的一个属性，就是这个，它能维护多种回滚的规则~~~~
	@Nullable
	private List<RollbackRuleAttribute> rollbackRules;
	...
	public List<RollbackRuleAttribute> getRollbackRules() {
		if (this.rollbackRules == null) {
			this.rollbackRules = new LinkedList<>();
		}
		return this.rollbackRules;
	}

	// 核心逻辑输入，复写了父类的rollbackOn方法。也就是看看当前异常是否需要回滚呢？？？
	@Override
	public boolean rollbackOn(Throwable ex) {
		RollbackRuleAttribute winner = null;
		int deepest = Integer.MAX_VALUE;

		// 这里getDepth()就是去看看异常栈里面  该类型的异常处于啥位置。
		// 这里用了Integer的最大值，基本相当于不管异常有多深，遇上此异常都应该回滚喽，也就是找到这个winnner了~~~~~
		if (this.rollbackRules != null) {
			for (RollbackRuleAttribute rule : this.rollbackRules) {
				int depth = rule.getDepth(ex);
				if (depth >= 0 && depth < deepest) {
					deepest = depth;
					winner = rule;
				}
			}
		}

		// User superclass behavior (rollback on unchecked) if no rule matches.
		// 这句相当于：如果你没有指定回滚规则，那就交给父类吧（只回滚RuntimeException和Error类型）
		if (winner == null) {
			return super.rollbackOn(ex);
		}
		
		// 最终只要找到了，但是不是NoRollbackRuleAttribute类型就成`~~~~
		return !(winner instanceof NoRollbackRuleAttribute);
	}
}
```

#### DelegatingTransactionAttribute



## `TransactionDefinition`

> The `TransactionDefinition` interface specifies:
>
> TransactionDefinition 接口指定:
>
> - Propagation: Typically, all code within a transaction scope runs in that transaction. However, you can specify the behavior if a transactional method is run when a transaction context already exists. For example, code can continue running in the existing transaction (the common case), or the existing transaction can be suspended and a new transaction created. Spring offers all of the transaction propagation options familiar from EJB CMT. To read about the semantics of transaction propagation in Spring, see [Transaction Propagation](https://docs.spring.io/spring-framework/docs/current/reference/html/data-access.html#tx-propagation).
>
>   传播: 通常，事务范围内的所有代码都在该事务中运行。但是，如果事务方法在事务上下文已经存在时运行，则可以指定该行为。例如，代码可以在现有事务中继续运行(常见情况) ，或者可以挂起现有事务并创建新事务。Spring 提供了 EJB CMT 中熟悉的所有事务传播选项。要了解 Spring 中事务传播的语义，请参阅事务传播。
>
> - Isolation: The degree to which this transaction is isolated from the work of other transactions. For example, can this transaction see uncommitted writes from other transactions?
>
>   隔离性: 此事务与其他事务工作隔离的程度。例如，此事务能否看到来自其他事务的未提交写操作？
>
> - Timeout: How long this transaction runs before timing out and being automatically rolled back by the underlying transaction infrastructure.
>
>   Timeout: 此事务在超时并被基础事务基础结构自动回滚之前运行多长时间。
>
> - Read-only status: You can use a read-only transaction when your code reads but does not modify data. Read-only transactions can be a useful optimization in some cases, such as when you use Hibernate.
>
>   只读状态: 当代码读取但不修改数据时，可以使用只读事务。在某些情况下，只读事务可能是一种有用的优化，例如在使用 Hibernate 时。

#### DelegatingTransactionDefinition

代理抽象类，啥都木有做。内部持有一个`TransactionDefinition targetDefinition`的引用而已，所有方法都是委托给`targetDefinition`去做的

#### `ResourceTransactionDefinition`

```java
// @since 5.1
// 指示资源事务，尤其是事务性资源是否准备好进行本地优化
public interface ResourceTransactionDefinition extends TransactionDefinition {
	// 确定事务性资源是否准备好进行本地优化
	// @see #isReadOnly()
	boolean isLocalResource();
}
```

它和 ResourceTransactionManager 的使用相关联。ResourceTransactionManager 是PlatformTransactionManager 的一个子接口。
我们最常用的事务管理器 DataSourceTransactionManager 也实现了这个接口

> 目前Spring还未提供任何`ResourceTransactionDefinition`它的具体实现



## TransactionAttributeSource：事务属性源（提供获取 @Transaction 的TransactionAttribute）

位于包：`org.springframework.transaction.interceptor`
它有点类似于之前讲过的`TargetSource`，它也是对`TransactionAttribute`进行了一层包装

```java
public interface TransactionAttributeSource {
	// 通过Method和目标类，拿到事务属性
	// 比如我们的 @Transaction 是标注在方法上的，可以自定义方法级别的事务属性，用它就特别的方便~
	@Nullable
	TransactionAttribute getTransactionAttribute(Method method, @Nullable Class<?> targetClass);

}
```

1. `method` – 目前正在进行的方法调用
2. `targetClass` – `真正`要调用的方法所在的类

- `method`的所属类不一样是`targetClass`。比如：method是代理对象的方法，它的所属类是代理出来的类
- 但是：`targetClass`一定会有一个方法和`method`的方法签名一样



> TransactionAttributeSource一般都是作为 TransactionInterceptor 的一个属性被set进去，然后看看这个事务属性可以作用在不同的方法上面，实现不同方法的个性化定制
> （实际真正处理它的是父类 TransactionAspectSupport，它会做匹配~~~~） 具体的在详解TransactionInterceptor的时候会讲述到

#### NameMatchTransactionAttributeSource



```java
// 自定义配置一个事务拦截器（@Transaction注解也会使用此拦截器进行拦截）
    @Bean
    public TransactionInterceptor transactionInterceptor(PlatformTransactionManager transactionManager) {
        Map<String, TransactionAttribute> txMap = new HashMap<>();
        // required事务  适用于觉得部分场景~
        RuleBasedTransactionAttribute requiredTx = new RuleBasedTransactionAttribute();
        requiredTx.setRollbackRules(Collections.singletonList(new RollbackRuleAttribute(RuntimeException.class)));
        requiredTx.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
        txMap.put("add*", requiredTx);
        txMap.put("save*", requiredTx);
        txMap.put("insert*", requiredTx);
        txMap.put("update*", requiredTx);
        txMap.put("delete*", requiredTx);

        // 查询 使用只读事务
        RuleBasedTransactionAttribute readOnlyTx = new RuleBasedTransactionAttribute();
        readOnlyTx.setReadOnly(true);
        readOnlyTx.setPropagationBehavior(TransactionDefinition.PROPAGATION_NOT_SUPPORTED);
        txMap.put("get*", readOnlyTx);
        txMap.put("query*", readOnlyTx);

        // 定义事务属性的source~~~ 此处使用它  也就是根据方法名进行匹配的~~~
        NameMatchTransactionAttributeSource source = new NameMatchTransactionAttributeSource();
        source.setNameMap(txMap);
        return new TransactionInterceptor(transactionManager, source);
    }

```

> 基于XML的配置事务的时候，原理就是这样的
>
> > 注意此处的匹配模式也是基于简单匹配的：`PatternMatchUtils.simpleMatch`。而非强大的正则匹配。底层`getTransactionAttribute()`时会根据不同的方法名，来返回不同的事务属性



#### MethodMapTransactionAttributeSource

> 使用方式和NameMatchTransactionAttributeSource基本相同，但是有一个不同在于：
>
> 如果使用 NameMatchTransactionAttributeSource 配置属性源，比如 get* 配置为执行事务，那么所有的bean的get方法都会被加上事务，这可能不是我们想要的，因此对于自动代理，我们更好的选择是 MethodMapTransactionAttributeSource，它需要指定需要事务化的完整类名和方法名



#### CompositeTransactionAttributeSource

```java
// @since 2.0 它是Spring2.0后才推出来的
public class CompositeTransactionAttributeSource implements TransactionAttributeSource, Serializable {
	private final TransactionAttributeSource[] transactionAttributeSources;
	public CompositeTransactionAttributeSource(TransactionAttributeSource... transactionAttributeSources) {
		Assert.notNull(transactionAttributeSources, "TransactionAttributeSource array must not be null");
		this.transactionAttributeSources = transactionAttributeSources;
	}
		
	// 这个实现方法也很容易。多个TransactionAttributeSource放在一起，只要任意一个匹配上就成
	// 备注：若匹配上多个，请注意先后顺序就成   这里面是数组  会保持和你放入的顺序一样~~~
	@Override
	@Nullable
	public TransactionAttribute getTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
		for (TransactionAttributeSource source : this.transactionAttributeSources) {
			TransactionAttribute attr = source.getTransactionAttribute(method, targetClass);
			if (attr != null) {
				return attr;
			}
		}
		return null;
	}
}
```

#### MatchAlwaysTransactionAttributeSource

它是`TransactionAttributeSource`的一个最简单的实现，每次调用，都是返回相同的`TransactionAttribute`

> 当然它是可议set一个`TransactionAttribute`作为通用的事务属性的实现的



#### `AnnotationTransactionAttributeSource`

基于注解驱动的事务管理的事务属性源，和`@Transaction`相关，也是现在使用得最最多的方式。

它的基本作用为：它遇上比如`@Transaction`标注的方法时，此类会分析此事务注解，最终组织形成一个`TransactionAttribute`供随后的调用。

```java
public class AnnotationTransactionAttributeSource extends AbstractFallbackTransactionAttributeSource implements Serializable {

	// 这个是“向下兼容”，JavaEE提供的其余两种注解~~
	private static final boolean jta12Present; //JTA 1.2事务注解
	private static final boolean ejb3Present; //EJB 3 事务注解是
	static {
		ClassLoader classLoader = AnnotationTransactionAttributeSource.class.getClassLoader();
		jta12Present = ClassUtils.isPresent("javax.transaction.Transactional", classLoader);
		ejb3Present = ClassUtils.isPresent("javax.ejb.TransactionAttribute", classLoader);
	}
	
	// true：只处理public方法（基于JDK的代理  显然就只会处理这种方法）
	// false：private/protected等方法都会处理。   基于AspectJ代理得方式可议设置为false
	// 默认情况下：会被赋值为true，表示只处理public的方法
	private final boolean publicMethodsOnly;
	// 保存用于分析事务注解的事务注解分析器   这个注解分析的解析器是重点
	private final Set<TransactionAnnotationParser> annotationParsers;

	// 构造函数, publicMethodsOnly 缺省使用 true
	public AnnotationTransactionAttributeSource() {
		this(true);
	}
	public AnnotationTransactionAttributeSource(boolean publicMethodsOnly) {
		this.publicMethodsOnly = publicMethodsOnly;
		if (jta12Present || ejb3Present) {
			this.annotationParsers = new LinkedHashSet<>(4);
			this.annotationParsers.add(new SpringTransactionAnnotationParser());
			if (jta12Present) {
				this.annotationParsers.add(new JtaTransactionAnnotationParser());
			}
			if (ejb3Present) {
				this.annotationParsers.add(new Ejb3TransactionAnnotationParser());
			}
		} 
		// 默认情况下，只添加Spring自己的注解解析器（绝大部分情况都是这里）
		else {
			this.annotationParsers = Collections.singleton(new SpringTransactionAnnotationParser());
		}
	}
	// 自己也可以指定一个TransactionAnnotationParser   或者多个也成
	public AnnotationTransactionAttributeSource(TransactionAnnotationParser annotationParser) {	... }
	public AnnotationTransactionAttributeSource(TransactionAnnotationParser... annotationParsers) { ... }
	public AnnotationTransactionAttributeSource(Set<TransactionAnnotationParser> annotationParsers) { ... }

	// 获取某个类/方法上的事务注解属性（属于 父类的抽象方法）
	@Override
	@Nullable
	protected TransactionAttribute findTransactionAttribute(Class<?> clazz) {
		return determineTransactionAttribute(clazz);
	}
	@Override
	@Nullable
	protected TransactionAttribute findTransactionAttribute(Method method) {
		return determineTransactionAttribute(method);
	}

	// 具体实现如下：
	// 分析获取某个被注解的元素（AnnotatedElement ），具体的来讲，指的是一个类或者一个方法上的事务注解属性。
	// 实现会遍历自己属性annotationParsers中所包含的事务注解属性分析器试图获取事务注解属性  所以主要还是依赖于TransactionAnnotationParser 去解析的
	@Nullable
	protected TransactionAttribute determineTransactionAttribute(AnnotatedElement element) {
		for (TransactionAnnotationParser annotationParser : this.annotationParsers) {
			TransactionAttribute attr = annotationParser.parseTransactionAnnotation(element);
			if (attr != null) {
				return attr;
			}
		}
		return null;
	}

	/**
	 * By default, only public methods can be made transactional.
	 */
	@Override
	protected boolean allowPublicMethodsOnly() {
		return this.publicMethodsOnly;
	}
	...
}
```

> 真正提供给调用的`getTransactionAttribute`在父类中实现的

#### AbstractFallbackTransactionAttributeSource（上面类的父类）

> `AbstractFallbackTransactionAttributeSource`是接口`TransactionAttributeSource`的抽象实现，也是上面提到的工具类`AnnotationTransactionAttributeSource`的父类

```java
public abstract class AbstractFallbackTransactionAttributeSource implements TransactionAttributeSource {

	// 针对没有事务注解属性的方法进行事务注解属性缓存时使用的特殊值，用于标记该方法没有事务注解属性
	// 从而不用在首次缓存在信息后，不用再次重复执行真正的分析  来提高查找的效率
	// 标注了@Transaction注解的表示有事务属性的，才会最终加入事务。但是，但是此处需要注意的是，只要被事务的Advisor切中的，都会缓存起来  放置过度的查找~~~~ 因此才有这个常量的出现
	private static final TransactionAttribute NULL_TRANSACTION_ATTRIBUTE = new DefaultTransactionAttribute() {
		@Override
		public String toString() {
			return "null";
		}
	};

	// 方法上的事务注解属性缓存，key使用目标类上的方法，使用类型MethodClassKey来表示
	// 这个Map会比较大，会被事务相关的Advisor拦截下来的方法，最终都会缓存下来。关于事务相关的Advisor，后续也是会着重讲解的~~~
	// 因为会有很多，所以我们才需要一个NULL_TRANSACTION_ATTRIBUTE常量来提高查找的效率~~~
	private final Map<Object, TransactionAttribute> attributeCache = new ConcurrentHashMap<>(1024);

	// 获取指定方法上的注解事务属性   如果方法上没有注解事务属性，则使用目标方法所属类上的注解事务属性
	@Override
	@Nullable
	public TransactionAttribute getTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
		// 如果目标方法是内置类Object上的方法，总是返回null，这些方法上不应用事务
		if (method.getDeclaringClass() == Object.class) {
			return null;
		}

		// 先看缓存里有木有，此处使用的非常经典的MethodClassKey作为Map的key
		Object cacheKey = getCacheKey(method, targetClass);
		TransactionAttribute cached = this.attributeCache.get(cacheKey);
		if (cached != null) {
			//目标方法上上并没有事务注解属性，但是已经被尝试分析过并且已经被缓存，
			// 使用的值是 NULL_TRANSACTION_ATTRIBUTE,所以这里再次尝试获取其注解事务属性时，直接返回 null
			if (cached == NULL_TRANSACTION_ATTRIBUTE) {
				return null;
			} else {
				return cached;
			}
		}
		// 缓存没有命中~~~~
		else {
			// 通过方法、目标Class 分析出此方法上的事务属性~~~~~
			TransactionAttribute txAttr = computeTransactionAttribute(method, targetClass);
			// 如果目标方法上并没有使用注解事务属性，也缓存该信息，只不过使用的值是一个特殊值:
			if (txAttr == null) {
				this.attributeCache.put(cacheKey, NULL_TRANSACTION_ATTRIBUTE);
			}
			// 存在目标属性~ 就put到里面去。
			// 获取到methodIdentification  基本只为了输出日志~~~
			else {
				String methodIdentification = ClassUtils.getQualifiedMethodName(method, targetClass);
				if (txAttr instanceof DefaultTransactionAttribute) {
					((DefaultTransactionAttribute) txAttr).setDescriptor(methodIdentification);
				}
				if (logger.isTraceEnabled()) {
					logger.trace("Adding transactional method '" + methodIdentification + "' with attribute: " + txAttr);
				}
				this.attributeCache.put(cacheKey, txAttr);
			}
			return txAttr;
		}
	}

	//查找目标方法上的事务注解属性 也是上面的核心方法
	@Nullable
	protected TransactionAttribute computeTransactionAttribute(Method method, @Nullable Class<?> targetClass) {
		// 如果事务注解属性分析仅仅针对public方法，而当前方法不是public，则直接返回null
		// 如果是private，AOP是能切入，代理对象也会生成的  但就是事务不回生效的~~~~
		if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
			return null;
		}

		// 上面说了，因为Method并不一样属于目标类。所以这个方法就是获取targetClass上的那个和method对应的方法  也就是最终要执行的方法
		Method specificMethod = AopUtils.getMostSpecificMethod(method, targetClass);

		// 第一步：去找直接标记在方法上的事务属性~~~ 如果方法上有就直接返回（不用再看类上的了）
		// findTransactionAttribute这个方法其实就是子类去实现的
		TransactionAttribute txAttr = findTransactionAttribute(specificMethod);
		if (txAttr != null) {
			return txAttr;
		}

		// 然后尝试检查事务注解属性是否标记在目标方法 specificMethod（注意此处用不是Method） 所属类上
		txAttr = findTransactionAttribute(specificMethod.getDeclaringClass());
		if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
			return txAttr;
		}

		// 程序走到这里说明目标方法specificMethod，也就是实现类上的目标方法上没有标记事务注解属性（否则直接返回了嘛）
		
		// 如果 specificMethod 和 method 不同，则说明 specificMethod 是具体实现类的方法method 是实现类所实现接口的方法
		// 因此再次尝试从 method 上获取事务注解属性
		// 这也就是为何我们的@Transaction标注在接口上或者接口的方法上都是好使的原因~~~~~~~
		if (specificMethod != method) {
			// Fallback is to look at the original method.
			txAttr = findTransactionAttribute(method);
			if (txAttr != null) {
				return txAttr;
			}
			txAttr = findTransactionAttribute(method.getDeclaringClass());
			if (txAttr != null && ClassUtils.isUserLevelMethod(method)) {
				return txAttr;
			}
		}
		return null;
	}

	// 可议看到默认值是false  表示private的也是ok的
	// 但是`AnnotationTransactionAttributeSource`复写了它  可以由开发者指定（默认是true了）
	protected boolean allowPublicMethodsOnly() {
		return false;
	}

}
```



## TransactionAnnotationParser

```java
// @since 2.5
public interface TransactionAnnotationParser {
	@Nullable
	TransactionAttribute parseTransactionAnnotation(AnnotatedElement element);

}
```



#### JtaTransactionAnnotationParser

#### Ejb3TransactionAnnotationParser



#### SpringTransactionAnnotationParser

专门用于解析Class或者Method上的`org.springframework.transaction.annotation.Transactional`注解的

```java
// @since 2.5 此类的实现相对来说还是比较简单的
public class SpringTransactionAnnotationParser implements TransactionAnnotationParser, Serializable {
	
	// 此方法对外暴露，表示获取该方法/类上面的TransactionAttribute 
	@Override
	@Nullable
	public TransactionAttribute parseTransactionAnnotation(AnnotatedElement element) {
		AnnotationAttributes attributes = AnnotatedElementUtils.findMergedAnnotationAttributes(element, Transactional.class, false, false);
		if (attributes != null) {
			// 此处注意，把这个注解的属性交给它，最终转换为事务的属性类~~~~
			return parseTransactionAnnotation(attributes);
		}
		// 注解都木有，那就返回null
		else {
			return null;
		}
	}

	// 顺便提供的一个重载方法，可以让你直接传入一个注解
	public TransactionAttribute parseTransactionAnnotation(Transactional ann) {
		return parseTransactionAnnotation(AnnotationUtils.getAnnotationAttributes(ann, false, false));
	}

	// 这个简单的说：就是把注解的属性们 专门为事务属性们~~~~
	protected TransactionAttribute parseTransactionAnnotation(AnnotationAttributes attributes) {
		// 此处用的 RuleBasedTransactionAttribute  因为它可议指定不需要回滚的类~~~~
		RuleBasedTransactionAttribute rbta = new RuleBasedTransactionAttribute();
		
		// 事务的传播属性枚举：内部定义了7种事务传播行为~~~~~
		Propagation propagation = attributes.getEnum("propagation");
		rbta.setPropagationBehavior(propagation.value());
		
		// 事务的隔离级别枚举。一共是4中，枚举里提供一个默认值: 也就是上面我们说的TransactionDefinition.ISOLATION_DEFAULT
		// 至于默认值是哪种隔离界别：这个具体的数据库有关~~~
		Isolation isolation = attributes.getEnum("isolation");
		rbta.setIsolationLevel(isolation.value());
	
		// 设置事务的超时时间
		rbta.setTimeout(attributes.getNumber("timeout").intValue());
		// 是否是只读事务
		rbta.setReadOnly(attributes.getBoolean("readOnly"));
		// 这个属性，是指定事务管理器PlatformTransactionManager的BeanName的，若不指定，那就按照类型找了
		// 若容器中存在多个事务管理器，但又没指定名字  那就报错啦~~~
		rbta.setQualifier(attributes.getString("value"));

		// rollbackFor可以指定需要回滚的异常，可议指定多个  若不指定默认为RuntimeException
		// 此处使用的RollbackRuleAttribute包装~~~~  它就是个POJO没有实现其余接口
		List<RollbackRuleAttribute> rollbackRules = new ArrayList<>();
		for (Class<?> rbRule : attributes.getClassArray("rollbackFor")) {
			rollbackRules.add(new RollbackRuleAttribute(rbRule));
		}
		// 全类名的方式~~
		for (String rbRule : attributes.getStringArray("rollbackForClassName")) {
			rollbackRules.add(new RollbackRuleAttribute(rbRule));
		}


		// 指定不需要回滚的异常类型们~~~
		// 此处使用的NoRollbackRuleAttribute包装  它是RollbackRuleAttribute的子类
		for (Class<?> rbRule : attributes.getClassArray("noRollbackFor")) {
			rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
		}
		for (String rbRule : attributes.getStringArray("noRollbackForClassName")) {
			rollbackRules.add(new NoRollbackRuleAttribute(rbRule));
		}
		// 最后别忘了set进去
		rbta.setRollbackRules(rollbackRules);

		return rbta;
	}
}
```

> 通过这个parser就可以把方法/类上的注解，转换为事务属性，然后缓存起来。
> 这样方法在调用的时候，直接根据`Method`就能取到事务属性，从而执行不同的事务策略

## （**）SavepointManager

管理事务 savepoint 的编程式 API 接口。

JDBC定义了SavePoint接口，提供在一个更细粒度的事务控制机制。当设置了一个保存点后，可以 rollback 到该保存点处的状态，而不是 rollback 整个事务。Connection 接口的setSavepoint 和 releaseSavepoint 方法可以设置和释放保存点。

```java
// @since 1.1
public interface SavepointManager {
	Object createSavepoint() throws TransactionException;
	void rollbackToSavepoint(Object savepoint) throws TransactionException;
	void releaseSavepoint(Object savepoint) throws TransactionException;
}
```

> `TransactionStatus`这个分支很重要。

#### JdbcTransactionObjectSupport

```java
// @since 1.1  继承自SmartTransactionObject 
public abstract class JdbcTransactionObjectSupport implements SavepointManager, SmartTransactionObject {
	
	// 这是Spring定义的类，持有java.sql.Connection
	// 所以最支不支持还原点、创建还原点其实都是委托给它来的~
	@Nullable
	private ConnectionHolder connectionHolder;
	...
}
```

> 它也只是个抽象类，`SmartTransactionObject`接口相关的方法都没有去实现，
>
> 而由它的子类`DataSourceTransactionObject`去实现的
>
> 关于还原点的实现，整体上还是比较简单的，就是委托给Connection去做

##### DataSourceTransactionObject

## CallbackPreferringPlatformTransactionManager(编程式事务)

WebSphereUowTransactionManager

joinpointIdentification



## `TransactionInterceptor`：事务拦截器

真正做事情的其实还是在父类，它有一个执行事务的模版

`MethodInterceptor`，被事务拦截的方法最终都会执行到此增强器身上。
`MethodInterceptor`是个环绕通知，恰好符合我们的开启、提交、回滚事务等操作

```java
public class TransactionInterceptor extends TransactionAspectSupport implements MethodInterceptor, Serializable {
	// 构造函数：
	// 可议不用特殊的指定PlatformTransactionManager 事务管理器，后面会讲解自定义去获取
	// 可议自己指定Properties 以及 TransactionAttributeSource 
	public TransactionInterceptor() {
	}
    
	public TransactionInterceptor(PlatformTransactionManager ptm, Properties attributes) {
		setTransactionManager(ptm);
		setTransactionAttributes(attributes);
	}
    
	public TransactionInterceptor(PlatformTransactionManager ptm, TransactionAttributeSource tas) {
		setTransactionManager(ptm);
		setTransactionAttributeSource(tas);
	}

	// 接下来就是这个invoke方法：就是拦截的入口~
	@Override
	@Nullable
	public Object invoke(MethodInvocation invocation) throws Throwable {
		// 获取目标类
		Class<?> targetClass = (invocation.getThis() != null ? AopUtils.getTargetClass(invocation.getThis()) : null);

		// invokeWithinTransaction：父类TransactionAspectSupport的模板方法
		// invocation::proceed本处执行完成  执行目标方法（当然可能还有其余增强器）
		return invokeWithinTransaction(invocation.getMethod(), targetClass, invocation::proceed);
	}
	...
}

```



## `TransactionAspectSupport`

```java
// 通过BeanFactoryAware获取到BeanFactory
// InitializingBean的afterPropertiesSet是对Bean做一些验证（经常会借助它这么来校验Bean~~~）
public abstract class TransactionAspectSupport implements BeanFactoryAware, InitializingBean {

	// 这个类不允许实现Serializable接口，请注意~~
	// NOTE: This class must not implement Serializable because it serves as base
	// class for AspectJ aspects (which are not allowed to implement Serializable)!

	/**
	 * Key to use to store the default transaction manager.
	 */
	private static final Object DEFAULT_TRANSACTION_MANAGER_KEY = new Object();
	// currentTransactionStatus() 方法依托于它
	private static final ThreadLocal<TransactionInfo> transactionInfoHolder = new NamedThreadLocal<>("Current aspect-driven transaction");

	//Subclasses can use this to return the current TransactionInfo.
	// Only subclasses that cannot handle all operations in one method
	// 注意此方法是个静态方法  并且是protected的  说明只有子类能够调用，外部并不可以~~~
	@Nullable
	protected static TransactionInfo currentTransactionInfo() throws NoTransactionException {
		return transactionInfoHolder.get();
	}
	
	// 外部调用此Static方法，可议获取到当前事务的状态  从而甚至可议手动来提交、回滚事务
	public static TransactionStatus currentTransactionStatus() throws NoTransactionException {
		TransactionInfo info = currentTransactionInfo();
		if (info == null || info.transactionStatus == null) {
			throw new NoTransactionException("No transaction aspect-managed TransactionStatus in scope");
		}
		return info.transactionStatus;
	}

	//==========================================
	// 事务管理器的名称（若设置，会根据此名称去找到事务管理器~~~~）
	@Nullable
	private String transactionManagerBeanName;
	@Nullable
	private PlatformTransactionManager transactionManager;
	@Nullable
	private TransactionAttributeSource transactionAttributeSource;
	@Nullable
	private BeanFactory beanFactory;

	// 因为事务管理器可能也会有多个  所以此处做了一个简单的缓存~
	private final ConcurrentMap<Object, PlatformTransactionManager> transactionManagerCache = new ConcurrentReferenceHashMap<>(4);
	
	public void setTransactionAttributeSource(@Nullable TransactionAttributeSource transactionAttributeSource) {
		this.transactionAttributeSource = transactionAttributeSource;
	}
	// 这部操作发现，若传入的为Properties  内部是实际使用的是NameMatchTransactionAttributeSource 去匹配的
	// 备注：若调用了此方法   transactionAttributeSource就会被覆盖的哟
	public void setTransactionAttributes(Properties transactionAttributes) {
		NameMatchTransactionAttributeSource tas = new NameMatchTransactionAttributeSource();
		tas.setProperties(transactionAttributes);
		this.transactionAttributeSource = tas;
	}
    
	// 若你有多种匹配策略，这也是支持的  可谓非常强大有木有~~~
	public void setTransactionAttributeSources(TransactionAttributeSource... transactionAttributeSources) {
		this.transactionAttributeSource = new CompositeTransactionAttributeSource(transactionAttributeSources);
	}
	...
	// 接下来就只剩我们最为核心的处理事务的模版方法了：
	//protected修饰，不允许其他包和无关类调用
	@Nullable
	protected Object invokeWithinTransaction(Method method, @Nullable Class<?> targetClass, final InvocationCallback invocation) throws Throwable {

		// 获取事务属性源~
		TransactionAttributeSource tas = getTransactionAttributeSource();
		// 获取该方法对应的事务属性（这个特别重要）
		final TransactionAttribute txAttr = (tas != null ? tas.getTransactionAttribute(method, targetClass) : null);

		// 这个厉害了：就是去找到一个合适的事务管理器（具体策略详见方法~~~)
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		// 拿到目标方法唯一标识（类.方法，如service.UserServiceImpl.save）
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

		// 如果txAttr为空或者tm 属于非CallbackPreferringPlatformTransactionManager，执行目标增强
		// 在TransactionManager上，CallbackPreferringPlatformTransactionManager实现PlatformTransactionManager接口，暴露出一个方法用于执行事务处理中的回调
		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
		
			// 看是否有必要创建一个事务，根据`事务传播行为`，做出相应的判断
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			
			Object retVal = null;
			try {
				//回调方法执行，执行目标方法（原有的业务逻辑）
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// 出现异常了，进行回滚（注意：并不是所有异常都会rollback的）
				// 备注：此处若没有事务属性   会commit 兼容编程式事务吧
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
				//清除信息
				cleanupTransactionInfo(txInfo);
			}
	
			// 目标方法完全执行完成后，提交事务~~~
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}

		//编程式事务处理(CallbackPreferringPlatformTransactionManager) 会走这里 
		// 原理也差不太多，这里不做详解~~~~
		else {
			final ThrowableHolder throwableHolder = new ThrowableHolder();

			// It's a CallbackPreferringPlatformTransactionManager: pass a TransactionCallback in.
			try {
				Object result = ((CallbackPreferringPlatformTransactionManager) tm).execute(txAttr, status -> {
					TransactionInfo txInfo = prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
					try {
						return invocation.proceedWithInvocation();
					}
					catch (Throwable ex) {
						if (txAttr.rollbackOn(ex)) {
							// A RuntimeException: will lead to a rollback.
							if (ex instanceof RuntimeException) {
								throw (RuntimeException) ex;
							}
							else {
								throw new ThrowableHolderException(ex);
							}
						}
						else {
							// A normal return value: will lead to a commit.
							throwableHolder.throwable = ex;
							return null;
						}
					}
					finally {
						cleanupTransactionInfo(txInfo);
					}
				});

				// Check result state: It might indicate a Throwable to rethrow.
				if (throwableHolder.throwable != null) {
					throw throwableHolder.throwable;
				}
				return result;
			}
			catch (ThrowableHolderException ex) {
				throw ex.getCause();
			}
			catch (TransactionSystemException ex2) {
				if (throwableHolder.throwable != null) {
					logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
					ex2.initApplicationException(throwableHolder.throwable);
				}
				throw ex2;
			}
			catch (Throwable ex2) {
				if (throwableHolder.throwable != null) {
					logger.error("Application exception overridden by commit exception", throwableHolder.throwable);
				}
				throw ex2;
			}
		}
	}

	// 从容器中找到一个事务管理器
	@Nullable
	protected PlatformTransactionManager determineTransactionManager(@Nullable TransactionAttribute txAttr) {
		// 如果这两个都没配置，所以肯定是手动设置了PlatformTransactionManager的，那就直接返回即可
		if (txAttr == null || this.beanFactory == null) {
			return getTransactionManager();
		}

		// qualifier 就在此处发挥作用了，他就相当于BeanName
		String qualifier = txAttr.getQualifier();
		if (StringUtils.hasText(qualifier)) {
			// 根据此名称 以及PlatformTransactionManager.class 去容器内招
			return determineQualifiedTransactionManager(this.beanFactory, qualifier);
		}
		// 若没有指定qualifier   那再看看是否指定了 transactionManagerBeanName
		else if (StringUtils.hasText(this.transactionManagerBeanName)) {
			return determineQualifiedTransactionManager(this.beanFactory, this.transactionManagerBeanName);
		}
		// 若都没指定，那就不管了。直接根据类型去容器里找 getBean(Class)
		// 此处：若容器内有两个PlatformTransactionManager ，那就铁定会报错啦~~~
		else {
			PlatformTransactionManager defaultTransactionManager = getTransactionManager();
			if (defaultTransactionManager == null) {
				defaultTransactionManager = this.transactionManagerCache.get(DEFAULT_TRANSACTION_MANAGER_KEY);
				if (defaultTransactionManager == null) {
					defaultTransactionManager = this.beanFactory.getBean(PlatformTransactionManager.class);
					this.transactionManagerCache.putIfAbsent(
							DEFAULT_TRANSACTION_MANAGER_KEY, defaultTransactionManager);
				}
			}
			return defaultTransactionManager;
		}
	}
	// ======================================
}
```

> 不同的事务处理方式使用不同的逻辑。对于`声明式事务`的处理与`编程式事务`的处理，重要区别在于事务属性上，因为编程式的事务处理是不需要有事务属性的

##### TransactionAspectSupport#TransactionInfo：内部类 事务Info

它是`TransactionAspectSupport`的一个`protected`内部类

```java
protected final class TransactionInfo {
		// 当前事务  的事务管理器
		@Nullable
		private final PlatformTransactionManager transactionManager;
		// 当前事务  的事务属性
		@Nullable
		private final TransactionAttribute transactionAttribute;
		// joinpoint标识
		private final String joinpointIdentification;
		// 当前事务 	的TransactionStatus 
		@Nullable
		private TransactionStatus transactionStatus;

		// 重点就是这个oldTransactionInfo字段
		// 这个字段保存了当前事务所在的`父事务`上下文的引用，构成了一个链，准确的说是一个有向无环图
		@Nullable
		private TransactionInfo oldTransactionInfo;
	
		// 唯一的构造函数~~~~
		public TransactionInfo(@Nullable PlatformTransactionManager transactionManager, @Nullable TransactionAttribute transactionAttribute, String joinpointIdentification) {
			this.transactionManager = transactionManager;
			this.transactionAttribute = transactionAttribute;
			this.joinpointIdentification = joinpointIdentification;
		}

		public PlatformTransactionManager getTransactionManager() {
			Assert.state(this.transactionManager != null, "No PlatformTransactionManager set");
			return this.transactionManager;
		}
		
		// 注意这个方法名，新的一个事务status
		public void newTransactionStatus(@Nullable TransactionStatus status) {
			this.transactionStatus = status;
		}
		public boolean hasTransaction() {
			return (this.transactionStatus != null);
		}

		//绑定当前正在处理的事务的所有信息到ThreadLocal
		private void bindToThread() {
			// Expose current TransactionStatus
			// 老的事务  先从线程中拿出来，再把新的（也就是当前）绑定进去~~~~~~
			this.oldTransactionInfo = transactionInfoHolder.get();
			transactionInfoHolder.set(this);
		}

		// 当前事务处理完之后，恢复父事务上下文
		private void restoreThreadLocalStatus() {
			transactionInfoHolder.set(this.oldTransactionInfo);
		}

		@Override
		public String toString() {
			return (this.transactionAttribute != null ? this.transactionAttribute.toString() : "No transaction");
		}
	}
```

##### TransactionAspectSupport#createTransactionIfNecessary

创建事务的一个重要方法，它会判断是否存在事务，根据事务的传播属性。做出不同的处理，也是做了一层包装，核心是通过TransactionStatus来判断事务的属性

```java
// 若有需要 创建一个TransactionInfo (具体的事务从事务管理器里面getTransaction()出来~)
	protected TransactionInfo createTransactionIfNecessary(@Nullable PlatformTransactionManager tm,
			@Nullable TransactionAttribute txAttr, final String joinpointIdentification) {

		// If no name specified, apply method identification as transaction name.
		// 这个简单的说，就是给Name赋值~~~~~
		if (txAttr != null && txAttr.getName() == null) {
			txAttr = new DelegatingTransactionAttribute(txAttr) {
				@Override
				public String getName() {
					return joinpointIdentification;
				}
			};
		}

		// 从事务管理器里，通过txAttr拿出来一个TransactionStatus
		TransactionStatus status = null;
		if (txAttr != null) {
			if (tm != null) {
				status = tm.getTransaction(txAttr);
			}
			...
		}
		// 通过TransactionStatus 等，转换成一个通用的TransactionInfo
		return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
	}
```

##### 准备事务：prepareTransactionInfo()

```java
protected TransactionInfo prepareTransactionInfo(@Nullable PlatformTransactionManager tm,
			@Nullable TransactionAttribute txAttr, String joinpointIdentification,
			@Nullable TransactionStatus status) {

		// 构造一个TransactionInfo 
		TransactionInfo txInfo = new TransactionInfo(tm, txAttr, joinpointIdentification);
		if (txAttr != null) {
			// 如果已存在不兼容的Tx，事务管理器将标记错误
			txInfo.newTransactionStatus(status);
		}
		,..

		// We always bind the TransactionInfo to the thread, even if we didn't create
		// a new transaction here. This guarantees that the TransactionInfo stack
		// will be managed correctly even if no transaction was created by this aspect.
		// 这句话是最重要的：把生成的TransactionInfo并绑定到当前线程的ThreadLocal
		txInfo.bindToThread();
		return txInfo;
	}
```

##### 提交事务：commitTransactionAfterReturning()

```java
//比较简单  只用用事务管理器提交事务即可~~~  具体的实现逻辑在事务管理器的commit实现里~~~
	protected void commitTransactionAfterReturning(@Nullable TransactionInfo txInfo) {
		if (txInfo != null && txInfo.getTransactionStatus() != null) {
			txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
		}
	}
```

##### 回滚事务：completeTransactionAfterThrowing()

```java
protected void completeTransactionAfterThrowing(@Nullable TransactionInfo txInfo, Throwable ex) {

		if (txInfo != null && txInfo.getTransactionStatus() != null) {
			// 如果有事务属性了，那就调用rollbackOn看看这个异常需不需要回滚
			if (txInfo.transactionAttribute != null && txInfo.transactionAttribute.rollbackOn(ex)) {
				try {
					txInfo.getTransactionManager().rollback(txInfo.getTransactionStatus());
				}
				...
			}
			// 编程式事务没有事务属性，那就commit吧
			else {
				try {
					txInfo.getTransactionManager().commit(txInfo.getTransactionStatus());
				}
				...
			}
		}
	}
```

##### 清除（解绑）事务：cleanupTransactionInfo()

```java
protected void cleanupTransactionInfo(@Nullable TransactionInfo txInfo) {
		if (txInfo != null) {
			txInfo.restoreThreadLocalStatus();
		}
	}
```

## PlatformTransactionManager：事务管理器

**关于事务管理器，不管是JPA（JpaTransactionManager ）还是JDBC（DataSourceTransactionManager）甚至是JTA（JtaTransactionManager）等都实现自接口 PlatformTransactionManager**

```java
public interface PlatformTransactionManager {
	TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException;
	void commit(TransactionStatus status) throws TransactionException;
	void rollback(TransactionStatus status) throws TransactionException;
｝
```

`AbstractPlatformTransactionManager`，这个抽象类提供了处理的模版（其实Spring的设计模式中，很多抽象类都提供了实现模版），然后提供开口给子类去各自实现

最常用的实现：`DataSourceTransactionManager`

### （模板）`AbstractPlatformTransactionManager`

实现Spring的标准事务工作流
这个基类提供了以下工作流程处理：

* 确定如果有现有的事务;
* 应用适当的传播行为;
* 如果有必要暂停和恢复事务;
* 提交时检查rollback-only标记;
* 应用适当的修改当回滚(实际回滚或设置rollback-only);
* 触发同步回调注册(如果事务同步是激活的)

```java
public abstract class AbstractPlatformTransactionManager implements PlatformTransactionManager, Serializable {

	//始终激活事务同步（请参阅事务的传播属性~）
	public static final int SYNCHRONIZATION_ALWAYS = 0;
	//仅对实际事务（即，不针对由传播导致的空事务）激活事务同步\不支持现有后端事务
	public static final int SYNCHRONIZATION_ON_ACTUAL_TRANSACTION = 1;
	//永远不激活事务同步
	public static final int SYNCHRONIZATION_NEVER = 2;

	// 相当于把本类的所有的public static final的变量都收集到此处~~~~
	private static final Constants constants = new Constants(AbstractPlatformTransactionManager.class);

	// ===========默认值
	private int transactionSynchronization = SYNCHRONIZATION_ALWAYS;
	// 事务默认的超时时间  为-1表示不超时
	private int defaultTimeout = TransactionDefinition.TIMEOUT_DEFAULT;
	//Set whether nested transactions are allowed. Default is "false".
	private boolean nestedTransactionAllowed = false;
	// Set whether existing transactions should be validated before participating（参与、加入）
	private boolean validateExistingTransaction = false;
	
	//设置是否仅在参与事务`失败后`将 现有事务`全局`标记为回滚  默认值是true 需要注意~~~
	// 表示只要你的事务失败了，就标记此事务为rollback-only 表示它只能给与回滚  而不能再commit或者正常结束了
	// 这个调用者经常会犯的一个错误就是：上层事务service抛出异常了，自己把它给try住，并且并且还不throw，那就肯定会报错的：
	// 报错信息：Transaction rolled back because it has been marked as rollback-only
	// 当然喽，这个属性强制不建议设置为false~~~~~~
	private boolean globalRollbackOnParticipationFailure = true;
	// 如果事务被全局标记为仅回滚，则设置是否及早失败~~~~
	private boolean failEarlyOnGlobalRollbackOnly = false;
	// 设置在@code docommit调用失败时是否应执行@code dorollback 通常不需要，因此应避免
	private boolean rollbackOnCommitFailure = false;
	
	// 此处我们直接可以通过属性们来社会，语意思更清晰些了
	// 我们发现使用起来有点枚举的意思了，特别是用XML配置的时候  非常像枚举的使用~~~~~~~
	// 这也是Constants的重要意义~~~~
	public final void setTransactionSynchronizationName(String constantName) {
		setTransactionSynchronization(constants.asNumber(constantName).intValue());
	}
	public final void setTransactionSynchronization(int transactionSynchronization) {
		this.transactionSynchronization = transactionSynchronization;
	}
	//... 省略上面所有字段的一些get/set方法~~~

	// 最为重要的一个方法，根据实物定义，获取到一个事务TransactionStatus 
	@Override
	public final TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException {
		//doGetTransaction()方法是抽象方法，具体的实现由具体的事务处理器提供（下面会以DataSourceTransactionManager为例子）
		Object transaction = doGetTransaction();

		//如果没有配置事务属性，则使用默认的事务属性
		if (definition == null) {
			definition = new DefaultTransactionDefinition();
		}

		//检查当前线程是否存在事务  isExistingTransaction此方法默认返回false  但子类都复写了此方法
		if (isExistingTransaction(transaction)) {
			// handleExistingTransaction方法为处理已经存在事务的情况
			// 这个方法的实现也很复杂，总之还是对一些传播属性进行解析，各种情况的考虑~~~~~ 如果有新事务产生 doBegin()就会被调用~~~~
			return handleExistingTransaction(definition, transaction, debugEnabled);
		}

		// 超时时间的简单校验~~~~
		if (definition.getTimeout() < TransactionDefinition.TIMEOUT_DEFAULT) {
			throw new InvalidTimeoutException("Invalid transaction timeout", definition.getTimeout());
		}

		// 处理事务属性中配置的事务传播特性==============
	
		// PROPAGATION_MANDATORY 如果已经存在一个事务，支持当前事务。如果没有一个活动的事务，则抛出异常
		if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_MANDATORY) {
			throw new IllegalTransactionStateException("No existing transaction found for transaction marked with propagation 'mandatory'");
		}
	
		//如果事务传播特性为required、required_new或nested
		else if (definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRED ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_REQUIRES_NEW ||
				definition.getPropagationBehavior() == TransactionDefinition.PROPAGATION_NESTED) {
				
			// 挂起，但是doSuspend()由子类去实现~~~
			// 挂起操作，触发相关的挂起注册的事件，把当前线程事物的所有属性都封装好，放到一个SuspendedResourcesHolder
			// 然后清空清空一下`当前线程事务`
			SuspendedResourcesHolder suspendedResources = suspend(null);

			// 此处，开始创建事务~~~~~
			try {
				boolean newSynchronization = (getTransactionSynchronization() != SYNCHRONIZATION_NEVER);

				// //创建一个新的事务状态  就是new DefaultTransactionStatus()  把个属性都赋值上
				DefaultTransactionStatus status = newTransactionStatus(
						definition, transaction, true, newSynchronization, debugEnabled, suspendedResources);
				// 开始事务，抽象方法，由子类去实现~
				doBegin(transaction, definition);
				//初始化和同步事务状态    是TransactionSynchronizationManager这个类  它内部维护了很多的ThreadLocal
				prepareSynchronization(status, definition);
				return status;
			}
			catch (RuntimeException | Error ex) {
				//重新开始 doResume由子类去实现
				resume(null, suspendedResources);
				throw ex;
			}
		}
		// 走到这里  传播属性就是不需要事务的  那就直接创建一个
		else {
			boolean newSynchronization = (getTransactionSynchronization() == SYNCHRONIZATION_ALWAYS);
			// 这个方法相当于先newTransactionStatus,再prepareSynchronization这两步~~~
			// 显然和上面的区别是：中间不回插入调用doBegin()方法，因为没有事务  begin个啥~~
			return prepareTransactionStatus(definition, null, true, newSynchronization, debugEnabled, null);
		}
	}


	// 再看看commit方法
	@Override
	public final void commit(TransactionStatus status) throws TransactionException {
		//如果是一个已经完成的事物，不可重复提交
		if (status.isCompleted()) {
			throw new IllegalTransactionStateException("Transaction is already completed - do not call commit or rollback more than once per transaction");
		}

		DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
		// 如果已经标记为了需要回滚，那就执行回滚吧
		if (defStatus.isLocalRollbackOnly()) {
			processRollback(defStatus, false);
			return;
		}

		//  shouldCommitOnGlobalRollbackOnly这个默认值是false，目前只有JTA事务复写成true了
		// isGlobalRollbackOnly：是否标记为了全局的RollbackOnly
		if (!shouldCommitOnGlobalRollbackOnly() && defStatus.isGlobalRollbackOnly()) {
			processRollback(defStatus, true);
			return;
		}
		// 提交事务   这里面还是挺复杂的，会考虑到还原点、新事务、事务是否是rollback-only之类的~~
		processCommit(defStatus);
	}

	// rollback方法  里面doRollback方法交给子类去实现~~~
	@Override
	public final void rollback(TransactionStatus status) throws TransactionException {
		DefaultTransactionStatus defStatus = (DefaultTransactionStatus) status;
		processRollback(defStatus, false);
	}
}
```

非常非常典型的模版实现，各个方法实现都是这样。自己先提供实现模版，很多具体的实现方案都开放给子类，比如begin,suspend, resume, commit, rollback等，相当于留好了众多的连接点

这个类的抽象程度非常的高，逻辑也非常的复杂。要想绝对的理解到位，必须要对JDBC的事务非常了解，而且还对这些代码逻辑必须进行精读

#### `（参考）DataSourceTransactionManager`

```java
// 它还实现了ResourceTransactionManager接口，提供了getResourceFactory()方法
public class DataSourceTransactionManager extends AbstractPlatformTransactionManager implements ResourceTransactionManager, InitializingBean {
	// 显然它管理的就是DataSource  而JTA分布式事务管理可能就是各种各样的数据源了
	@Nullable
	private DataSource dataSource;
	// 不要强制标记为ReadOnly
	private boolean enforceReadOnly = false;

	// JDBC默认是允许内嵌的事务的
	public DataSourceTransactionManager() {
		setNestedTransactionAllowed(true);
	}
	public DataSourceTransactionManager(DataSource dataSource) {
		this();
		setDataSource(dataSource);
		// 它自己的InitializingBean也是做了一个简单的校验而已~~~
		afterPropertiesSet();
	}

	// 手动设置数据源
	public void setDataSource(@Nullable DataSource dataSource) {
		// 这步处理有必要
		// TransactionAwareDataSourceProxy是对dataSource 的包装
		if (dataSource instanceof TransactionAwareDataSourceProxy) {
			this.dataSource = ((TransactionAwareDataSourceProxy) dataSource).getTargetDataSource();
		} else {
			this.dataSource = dataSource;
		}
	}

	//Return the JDBC DataSource
	@Nullable
	public DataSource getDataSource() {
		return this.dataSource;
	}
	// @since 5.0 Spring5.0提供的方法   其实还是调用的getDataSource()  判空了而已
	protected DataSource obtainDataSource() {
		DataSource dataSource = getDataSource();
		Assert.state(dataSource != null, "No DataSource set");
		return dataSource;
	}
	// 直接返回的数据源~~~~
	@Override
	public Object getResourceFactory() {
		return obtainDataSource();
	}
	...
	// 这里返回的是一个`DataSourceTransactionObject`
	// 它是一个`JdbcTransactionObjectSupport`，所以它是SavepointManager、实现了SmartTransactionObject接口
	@Override
	protected Object doGetTransaction() {
		DataSourceTransactionObject txObject = new DataSourceTransactionObject();
		txObject.setSavepointAllowed(isNestedTransactionAllowed());
		// 这个获取有意思~~~~相当于按照线程来的~~~
		ConnectionHolder conHolder = (ConnectionHolder) TransactionSynchronizationManager.getResource(obtainDataSource());
		txObject.setConnectionHolder(conHolder, false);
		return txObject;
	}

	// 检查当前事务是否active
	@Override
	protected boolean isExistingTransaction(Object transaction) {
		DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
		return (txObject.hasConnectionHolder() && txObject.getConnectionHolder().isTransactionActive());
	}


	// 这是一个核心内容了，里面逻辑需要分析分析~~~
	@Override
	protected void doBegin(Object transaction, TransactionDefinition definition) {
		DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
		Connection con = null;

		try {
			if (!txObject.hasConnectionHolder() || txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
				// 从DataSource里获取一个连接（这个DataSource一般是有连接池的~~~）
				Connection newCon = obtainDataSource().getConnection();
				// 把这个链接用ConnectionHolder包装一下~~~
				txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
			}

			txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
			con = txObject.getConnectionHolder().getConnection();
			
			// 设置isReadOnly、设置隔离界别等~
			Integer previousIsolationLevel = DataSourceUtils.prepareConnectionForTransaction(con, definition);
			txObject.setPreviousIsolationLevel(previousIsolationLevel);

			// 这里非常的关键，先看看Connection 是否是自动提交的
			// 如果是 就con.setAutoCommit(false)  要不然数据库默认没执行一条SQL都是一个事务，就没法进行事务的管理了
			if (con.getAutoCommit()) {
				txObject.setMustRestoreAutoCommit(true);
				con.setAutoCommit(false);
			}
			// ====因此从这后面，通过此Connection执行的所有SQL语句只要没有commit就都不会提交给数据库的=====
			
			// 这个方法特别特别有意思   它自己`Statement stmt = con.createStatement()`拿到一个Statement
			// 然后执行了一句SQL：`stmt.executeUpdate("SET TRANSACTION READ ONLY");`
			// 所以，所以：如果你仅仅只是查询。把事务的属性设置为readonly=true  Spring对帮你对SQl进行优化的
			// 需要注意的是：readonly=true 后，只能读，不能进行dml操作）（只能看到设置事物前数据的变化，看不到设置事物后数据的改变）
			prepareTransactionalConnection(con, definition);
			txObject.getConnectionHolder().setTransactionActive(true);

			int timeout = determineTimeout(definition);
			if (timeout != TransactionDefinition.TIMEOUT_DEFAULT) {
				txObject.getConnectionHolder().setTimeoutInSeconds(timeout);
			}

			// Bind the connection holder to the thread.
			// 这一步：就是把当前的链接 和当前的线程进行绑定~~~~
			if (txObject.isNewConnectionHolder()) {
				TransactionSynchronizationManager.bindResource(obtainDataSource(), txObject.getConnectionHolder());
			}
		} catch (Throwable ex) {
			// 如果是新创建的链接，那就释放~~~~
			if (txObject.isNewConnectionHolder()) {
				DataSourceUtils.releaseConnection(con, obtainDataSource());
				txObject.setConnectionHolder(null, false);
			}
			throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
		}
	}

	// 真正提交事务
	@Override
	protected void doCommit(DefaultTransactionStatus status) { DataSourceTransactionObject txObject = (DataSourceTransactionObject) status.getTransaction();
		// 拿到链接  然后直接就commit了   
		Connection con = txObject.getConnectionHolder().getConnection();
		try {
			con.commit();
		} catch (SQLException ex) {
			throw new TransactionSystemException("Could not commit JDBC transaction", ex);
		}
	}
	//doRollback()方法也类似  这里不再细说
}
```

> 事务属性readonly=true 后，只能读，不能进行dml操作）（只能看到设置事物前数据的变化，看不到设置事物后数据的改变） 但是但是但是通过源码我发现，你光@Transactional(readOnly = true)这样是不够的，还必须在配置DataSourceTransactionManager的时候，来这么一句dataSourceTransactionManager.setEnforceReadOnly(true)，最终才会对你的只读事务进行优化~

仅仅只是来了这么一句@Transactional(readOnly = true)而已，最终会把这个Connection设置为只读：con.setReadOnly(true); 它表示将此**连接**设置为只读模式，作为驱动程序启用数据库优化的提示。 将链接设置为只读模式通知数据库后，数据库会对做自己的只读优化。

但是但是但是，这对数据库而言不一定对于数据库而言这就是readonly事务，这点是非常重要的。（因为毕竟一个事务内可能有多个链接~~~~）
因此若想它变成只读性事务，进行最大程度上的优化，那么请你配置上的时候加上这一句：

若想它变成`只读性事务`，进行最大程度上的优化，那么请你配置上的时候加上这一句

```java
    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        DataSourceTransactionManager dataSourceTransactionManager = new DataSourceTransactionManager(dataSource);
        dataSourceTransactionManager.setEnforceReadOnly(true); // 让事务管理器进行只读事务层面上的优化  建议开启
        return dataSourceTransactionManager;
    }
```

> 基于这个特性，我在我的工作中强烈建议Controller层、Service层甚至Dao层各单元都进行`读写分离`，这样对读这一层能进行很好的统一优化，提升统一管控的效率



JDBC几个重要的API

* 关闭自动提交：java.sql.Connection.setAutoCommit(false) 若是true，每次操作都被认为是一次提交
* 手动提交事务：con.commit();
* 出现异常时回滚，不一定在catch语句中，只要在con.commit()前需要回滚时执行都可：con.rollback();
* 关闭连接：con.close();
* 设置事务隔离级别: java.sql.Connection#setTransactionIsolation()


> 说归说，实现快速开发，高可维护性又必然是永远的趋势。所以`会用、熟练的使用永远摆在第一位`
>
> 个人建议看任何问题都需要有辩证性的思维。肯定不是不推荐使用工具（**毕竟我认为重复造轮子是在浪费生命**），相反反而是更加的推崇。但是我们需要知道它背后得设计思路、设计原理，才能做到心中有数，更加运用自如。更重要的是扩展起来也是才能胸有成足，做到有把握：`稳`



# 声明式事务管理

## xml schema

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        https://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <aop:config>

        <aop:pointcut id="defaultServiceOperation"
                expression="execution(* x.y.service.*Service.*(..))"/>

        <aop:pointcut id="noTxServiceOperation"
                expression="execution(* x.y.service.ddl.DefaultDdlManager.*(..))"/>

        <aop:advisor pointcut-ref="defaultServiceOperation" advice-ref="defaultTxAdvice"/>

        <aop:advisor pointcut-ref="noTxServiceOperation" advice-ref="noTxAdvice"/>

    </aop:config>

    <!-- this bean will be transactional (see the 'defaultServiceOperation' pointcut) -->
    <bean id="fooService" class="x.y.service.DefaultFooService"/>

    <!-- this bean will also be transactional, but with totally different transactional settings -->
    <bean id="anotherFooService" class="x.y.service.ddl.DefaultDdlManager"/>

    <tx:advice id="defaultTxAdvice">
        <tx:attributes>
            <tx:method name="get*" read-only="true"/>
            <tx:method name="*"/>
        </tx:attributes>
    </tx:advice>

    <tx:advice id="noTxAdvice">
        <tx:attributes>
            <tx:method name="*" propagation="NEVER"/>
        </tx:attributes>
    </tx:advice>

    <!-- other transaction infrastructure beans such as a TransactionManager omitted... -->

</beans>



<!-- from the file 'context.xml' -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xmlns:aop="http://www.springframework.org/schema/aop"
    xmlns:tx="http://www.springframework.org/schema/tx"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/tx
        https://www.springframework.org/schema/tx/spring-tx.xsd
        http://www.springframework.org/schema/aop
        https://www.springframework.org/schema/aop/spring-aop.xsd">

    <!-- this is the service object that we want to make transactional -->
    <bean id="fooService" class="x.y.service.DefaultFooService"/>

    <!-- enable the configuration of transactional behavior based on annotations -->
    <!-- @Transactional -->
    <tx:annotation-driven transaction-manager="txManager"/><!-- a TransactionManager is still required --> 

    <bean id="txManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <!-- (this dependency is defined somewhere else) -->
        <property name="dataSource" ref="dataSource"/>
    </bean>

    <!-- other <bean/> definitions here -->

</beans>



<!-- 多个事务管理器 qualifier  -->
<tx:annotation-driven/>

    <bean id="transactionManager1" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        ...
        <qualifier value="order"/>
    </bean>

    <bean id="transactionManager2" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        ...
        <qualifier value="account"/>
    </bean>

    <bean id="transactionManager3" class="org.springframework.data.r2dbc.connectionfactory.R2dbcTransactionManager">
        ...
        <qualifier value="reactive-account"/>
    </bean>
```



## @Transactional 的使用 类比 @Async

开启注解驱动

```java
@EnableTransactionManagement // 开启注解驱动
@Configuration
public class JdbcConfig { ... }
```

> 提示：使用@EnableTransactionManagement注解前，请务必保证你已经配置了至少一个PlatformTransactionManager的Bean，否则会报错。（当然你也可以实现TransactionManagementConfigurer来提供一个专属的，只是我们一般都不这么去做



在你想要加入事务的方法上(或者类（接口）上)标注`@Transactional`注解

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Transactional {

	// value 和 transactionManager 属性 它们两个是一样的意思。当配置了多个事务管理器时，可以使用该属性指定选择哪个事务管理器。
	@AliasFor("transactionManager")
	String value() default "";
	 // @since 4.2
	@AliasFor("value")
	String transactionManager() default "";

	Propagation propagation() default Propagation.REQUIRED; //事务的传播行为
	Isolation isolation() default Isolation.DEFAULT; // 事务的隔离级别
	int timeout() default TransactionDefinition.TIMEOUT_DEFAULT; //事务的超时时间，默认值为-1
	boolean readOnly() default false; //是否只读 默认是false
	
	// 需要回滚的异常 可以指定多个异常类型   不指定默认只回滚RuntimeException和Error
	Class<? extends Throwable>[] rollbackFor() default {};
	String[] rollbackForClassName() default {};
	
	// 不需要回滚的异常们~~~
	Class<? extends Throwable>[] noRollbackFor() default {};
	String[] noRollbackForClassName() default {};
}

```



## @EnableTransactionManagement

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(TransactionManagementConfigurationSelector.class)
public @interface EnableTransactionManagement {
	boolean proxyTargetClass() default false;
	AdviceMode mode() default AdviceMode.PROXY;
	int order() default Ordered.LOWEST_PRECEDENCE;
}

```

属性和`@EnableAsync`注解的一毛一样。不同之处只在于`@Import`导入器导入的这个类



### TransactionManagementConfigurationSelector

所在的包为`org.springframework.transaction.annotation`，jar属于：spring-tx（**若引入了spring-jdbc，这个jar会自动导入**）

```java
public class TransactionManagementConfigurationSelector extends AdviceModeImportSelector<EnableTransactionManagement> {

	@Override
	protected String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
			// 很显然，绝大部分情况下，我们都不会使用AspectJ的静态代理的~~~~~~~~
			// 这里面会导入两个类~~~
			case PROXY:
				return new String[] {AutoProxyRegistrar.class.getName(),
						ProxyTransactionManagementConfiguration.class.getName()};
			case ASPECTJ:
				return new String[] {determineTransactionAspectClass()};
			default:
				return null;
		}
	}
	private String determineTransactionAspectClass() {
		return (ClassUtils.isPresent("javax.transaction.Transactional", getClass().getClassLoader()) ?
				TransactionManagementConfigUtils.JTA_TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME :
				TransactionManagementConfigUtils.TRANSACTION_ASPECT_CONFIGURATION_CLASS_NAME);
	}

}
```

> AdviceModeImportSelector目前所知的三个子类是：AsyncConfigurationSelector、TransactionManagementConfigurationSelector、CachingConfigurationSelector。由此可见后面还会着重分析的Spring的缓存体系@EnableCaching



### AutoProxyRegistrar

```java
// @since 3.1
public class AutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

	@Override
	public void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
		boolean candidateFound = false;
		
		// 这里面需要特别注意的是：这里是拿到所有的注解类型~~~而不是只拿@EnableAspectJAutoProxy这个类型的
		// 原因：因为mode、proxyTargetClass等属性会直接影响到代理得方式，而拥有这些属性的注解至少有：
		// @EnableTransactionManagement、@EnableAsync、@EnableCaching等~~~~
		// 甚至还有启用AOP的注解：@EnableAspectJAutoProxy它也能设置`proxyTargetClass`这个属性的值，因此也会产生关联影响~
		Set<String> annoTypes = importingClassMetadata.getAnnotationTypes();
		for (String annoType : annoTypes) {
			AnnotationAttributes candidate = AnnotationConfigUtils.attributesFor(importingClassMetadata, annoType);
			if (candidate == null) {
				continue;
			}
			// 拿到注解里的这两个属性
			// 说明：如果你是比如@Configuration或者别的注解的话  他们就是null了
			Object mode = candidate.get("mode");
			Object proxyTargetClass = candidate.get("proxyTargetClass");

			// 如果存在mode且存在proxyTargetClass 属性
			// 并且两个属性的class类型也是对的，才会进来此处（因此其余注解相当于都挡外面了~）
			if (mode != null && proxyTargetClass != null && AdviceMode.class == mode.getClass() &&
					Boolean.class == proxyTargetClass.getClass()) {
		
				// 标志：找到了候选的注解~~~~
				candidateFound = true;
				if (mode == AdviceMode.PROXY) {
					// 这一部是非常重要的~~~~又到了我们熟悉的AopConfigUtils工具类，且是熟悉的registerAutoProxyCreatorIfNecessary方法
					// 它主要是注册了一个`internalAutoProxyCreator`，但是若出现多次的话，这里不是覆盖的形式，而是以第一次的为主
					// 当然它内部有做等级的提升之类的，这个之前也有分析过~~~~
					AopConfigUtils.registerAutoProxyCreatorIfNecessary(registry);
					
					// 看要不要强制使用CGLIB的方式(由此可以发现  这个属性若出现多次，是会是覆盖的形式)
					if ((Boolean) proxyTargetClass) {
						AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
						return;
					}
				}
			}
		}
		
		// 如果一个都没有找到（我在想，肿么可能呢？）
		// 其实有可能：那就是自己注入这个类，而不是使用注解去注入（但并不建议这么去做）
		if (!candidateFound && logger.isInfoEnabled()) {
			// 输出info日志（注意并不是error日志）
		}
	}

}

```



> 跟踪`AopConfigUtils`的源码你会发现，事务这块向容器注入的是一个`InfrastructureAdvisorAutoProxyCreator`，它主要是读取`Advisor`类，并对符合的bean进行二次代理。

#### AopConfigUtils#registerAutoProxyCreatorIfNecessary

```java
@Nullable
	public static BeanDefinition registerAutoProxyCreatorIfNecessary(
			BeanDefinitionRegistry registry, @Nullable Object source) {

		return registerOrEscalateApcAsRequired(InfrastructureAdvisorAutoProxyCreator.class, registry, source);
	}


@Nullable
	private static BeanDefinition registerOrEscalateApcAsRequired(
			Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {

		Assert.notNull(registry, "BeanDefinitionRegistry must not be null");

		if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
			BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
			if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
				int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
				int requiredPriority = findPriorityForClass(cls);
				if (currentPriority < requiredPriority) {
					apcDefinition.setBeanClassName(cls.getName());
				}
			}
			return null;
		}
```

#### InfrastructureAdvisorAutoProxyCreator

>  相比  AnnotationAwareAspectJAutoProxyCreator 只是没有 @Aspect处理

BeanFactoryAdvisorRetrievalHelperAdapter

BeanFactoryAdvisorRetrievalHelper

主要是读取`Advisor`类，并对符合的bean进行二次代理。



### ProxyTransactionManagementConfiguration

```java
@Configuration
public class ProxyTransactionManagementConfiguration extends AbstractTransactionManagementConfiguration {

	// 这个Advisor可是事务的核心内容。。。。。也是本文重点分析的对象
	@Bean(name = TransactionManagementConfigUtils.TRANSACTION_ADVISOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public BeanFactoryTransactionAttributeSourceAdvisor transactionAdvisor() {
		BeanFactoryTransactionAttributeSourceAdvisor advisor = new BeanFactoryTransactionAttributeSourceAdvisor();
		advisor.setTransactionAttributeSource(transactionAttributeSource());
		advisor.setAdvice(transactionInterceptor());
		// 顺序由@EnableTransactionManagement注解的Order属性来指定 默认值为：Ordered.LOWEST_PRECEDENCE
		if (this.enableTx != null) {
			advisor.setOrder(this.enableTx.<Integer>getNumber("order"));
		}
		return advisor;
	}

	// TransactionAttributeSource 这种类特别像 `TargetSource`这种类的设计模式
	// 这里直接使用的是AnnotationTransactionAttributeSource  基于注解的事务属性源~~~
	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionAttributeSource transactionAttributeSource() {
		return new AnnotationTransactionAttributeSource();
	}

	// 事务拦截器，它是个`MethodInterceptor`，它也是Spring处理事务最为核心的部分
	// 请注意：你可以自己定义一个TransactionInterceptor（同名的），来覆盖此Bean（注意是覆盖）
	// 另外请注意：你自定义的BeanName必须同名，也就是必须名为：transactionInterceptor  否则两个都会注册进容器里面去~~~~~~
	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public TransactionInterceptor transactionInterceptor() {
		TransactionInterceptor interceptor = new TransactionInterceptor();
		// 事务的属性
		interceptor.setTransactionAttributeSource(transactionAttributeSource());
		// 事务管理器（也就是注解最终需要使用的事务管理器,父类已经处理好了）
		// 此处注意：我们是可议不用特殊指定的，最终它自己会去容器匹配一个适合的~~~~
		if (this.txManager != null) {
			interceptor.setTransactionManager(this.txManager);
		}
		return interceptor;
	}

}

// 父类（抽象类）  它实现了ImportAware接口  所以拿到@Import所在类的所有注解信息
@Configuration
public abstract class AbstractTransactionManagementConfiguration implements ImportAware {

	@Nullable
	protected AnnotationAttributes enableTx;
	/**
	 * Default transaction manager, as configured through a {@link TransactionManagementConfigurer}.
	 */
	// 此处：注解的默认的事务处理器（可议通过实现接口TransactionManagementConfigurer来自定义配置）
	// 因为事务管理器这个东西，一般来说全局一个就行，但是Spring也提供了定制化的能力~~~
	@Nullable
	protected PlatformTransactionManager txManager;

	@Override
	public void setImportMetadata(AnnotationMetadata importMetadata) {
		// 此处：只拿到@EnableTransactionManagement这个注解的就成~~~~~ 作为AnnotationAttributes保存起来
		this.enableTx = AnnotationAttributes.fromMap(importMetadata.getAnnotationAttributes(EnableTransactionManagement.class.getName(), false));
		// 这个注解是必须的~~~~~~~~~~~~~~~~
		if (this.enableTx == null) {
			throw new IllegalArgumentException("@EnableTransactionManagement is not present on importing class " + importMetadata.getClassName());
		}
	}

	// 这里和@Async的处理一样，配置文件可以实现这个接口。然后给注解驱动的给一个默认的事务管理器~~~~
	// 设计模式都是想通的~~~
	@Autowired(required = false)
	void setConfigurers(Collection<TransactionManagementConfigurer> configurers) {
		if (CollectionUtils.isEmpty(configurers)) {
			return;
		}
		// 同样的，最多也只允许你去配置一个~~~
		if (configurers.size() > 1) {
			throw new IllegalStateException("Only one TransactionManagementConfigurer may exist");
		}
		TransactionManagementConfigurer configurer = configurers.iterator().next();
		this.txManager = configurer.annotationDrivenTransactionManager();
	}


	// 注册一个监听器工厂，用以支持@TransactionalEventListener注解标注的方法，来监听事务相关的事件
	// 后面会专门讨论，通过事件监听模式来实现事务的监控~~~~
	@Bean(name = TransactionManagementConfigUtils.TRANSACTIONAL_EVENT_LISTENER_FACTORY_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public static TransactionalEventListenerFactory transactionalEventListenerFactory() {
		return new TransactionalEventListenerFactory();
	}

}
```



### `BeanFactoryTransactionAttributeSourceAdvisor`

**Bean工厂和事务都有关系的Advisor**

**父类**：`AbstractBeanFactoryPointcutAdvisor`

```java
// @since 2.5.5
// 它是一个AbstractBeanFactoryPointcutAdvisor ，关于这个Advisor 请参阅之前的博文讲解~~~
public class BeanFactoryTransactionAttributeSourceAdvisor extends AbstractBeanFactoryPointcutAdvisor {
	
	@Nullable
	private TransactionAttributeSource transactionAttributeSource;
	
	// 这个很重要，就是切面。它决定了哪些类会被切入，从而生成的代理对象~ 
	// 关于：TransactionAttributeSourcePointcut 下面有说~
	private final TransactionAttributeSourcePointcut pointcut = new TransactionAttributeSourcePointcut() {
		// 注意此处`getTransactionAttributeSource`就是它的一个抽象方法~~~
		@Override
		@Nullable
		protected TransactionAttributeSource getTransactionAttributeSource() {
			return transactionAttributeSource;
		}
	};
	
	// 可议手动设置一个事务属性源~
	public void setTransactionAttributeSource(TransactionAttributeSource transactionAttributeSource) {
		this.transactionAttributeSource = transactionAttributeSource;
	}

	// 当然我们可以指定ClassFilter  默认情况下：ClassFilter classFilter = ClassFilter.TRUE;  匹配所有的类的
	public void setClassFilter(ClassFilter classFilter) {
		this.pointcut.setClassFilter(classFilter);
	}

	// 此处pointcut就是使用自己的这个pointcut去切入~~~
	@Override
	public Pointcut getPointcut() {
		return this.pointcut;
	}

}

```

### TransactionAttributeSourcePointcut

事务的匹配Pointcut切面，决定了哪些类需要生成代理对象从而应用事务

```java
// 首先它的访问权限事default 显示是给内部使用的
// 首先它继承自StaticMethodMatcherPointcut   所以`ClassFilter classFilter = ClassFilter.TRUE;` 匹配所有的类
// 并且isRuntime=false  表示只需要对方法进行静态匹配即可~~~~
abstract class TransactionAttributeSourcePointcut extends StaticMethodMatcherPointcut implements Serializable {

	// 方法的匹配  静态匹配即可（因为事务无需要动态匹配这么细粒度~~~）
	@Override
	public boolean matches(Method method, Class<?> targetClass) {
		// 实现了如下三个接口的子类，就不需要被代理了  直接放行
		// TransactionalProxy它是SpringProxy的子类。  如果是被TransactionProxyFactoryBean生产出来的Bean，就会自动实现此接口，那么就不会被这里再次代理了
		// PlatformTransactionManager：spring抽象的事务管理器~~~
		// PersistenceExceptionTranslator对RuntimeException转换成DataAccessException的转换接口
		if (TransactionalProxy.class.isAssignableFrom(targetClass) ||
				PlatformTransactionManager.class.isAssignableFrom(targetClass) ||
				PersistenceExceptionTranslator.class.isAssignableFrom(targetClass)) {
			return false;
		}
		
		// 重要：拿到事务属性源~~~~~~
		// 如果tas == null表示没有配置事务属性源，那是全部匹配的  也就是说所有的方法都匹配~~~~（这个处理还是比较让我诧异的~~~）
		// 或者 标注了@Transaction这样的注解的方法才会给与匹配~~~
		TransactionAttributeSource tas = getTransactionAttributeSource();
		return (tas == null || tas.getTransactionAttribute(method, targetClass) != null);
	}	
	...
	// 由子类提供给我，告诉事务属性源~~~~ 我才好知道哪些方法我需要切嘛~~~
	@Nullable
	protected abstract TransactionAttributeSource getTransactionAttributeSource();
}

```

> 关于matches方法的调用时机：只要是容器内的每个Bean，都会经过AbstractAutoProxyCreator#postProcessAfterInitialization从而会调用wrapIfNecessary方法，因此容器内所有的Bean的所有方法在容器启动时候都会执行此matche方法，因此请注意缓存的使用~~~~~
> 真正的执行事务方法，还是在`TransactionInterceptor`这个增强器里





注意 : 如果某个方法和该方法所属类上都有事务注解属性，优先使用方法上的事务注解属性。

#### TransactionAttributeSourceClassFilter

TransactionalProxy

#### StaticMethodMatcherPointcut

### 自定义注解驱动的事务管理器 TransactionManagementConfigurer

注解的事务管理器会有一个默认的，然后@Transaction里也可以通过`value`属性进行制定~~。
改变默认的有一个非常优雅的方式，那就是使用`TransactionManagementConfigurer`接口来提供



```java
EnableTransactionManagement
@Configuration
public class JdbcConfig implements TransactionManagementConfigurer {
	
    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        DataSourceTransactionManager dataSourceTransactionManager = new DataSourceTransactionManager(dataSource);
        return dataSourceTransactionManager;
    }

	// 复写提供一个即可。可以自己new一个，当然也可议向着一样从容器中获取~~~
    @Override
    public PlatformTransactionManager annotationDrivenTransactionManager() {
    	// 直接使用容器内的事务管理器~~~
        return transactionManager(dataSource());
    }
	
}

```



#### 思考：若既有`@EnableAspectJAutoProxy`又有`@EnableTransactionManagement`，那么自动代理创建器怎么注入谁呢？和注解的标注的先后顺序有关吗？

@EnableAspectJAutoProxy会像容器注入AnnotationAwareAspectJAutoProxyCreator
@EnableTransactionManagement会像容器注入InfrastructureAdvisorAutoProxyCreator
那么它俩同时使用时



核心代码在这：`AopConfigUtils#registerOrEscalateApcAsRequired`方法

> 从源码分析可以知道：无论你的这些注解有多少个，无论他们的先后顺序如何，它内部都有咯`优先级提升`的机制来保证向下的覆盖兼容。因此一般情况下，我们使用的都是最高级的`AnnotationAwareAspectJAutoProxyCreator`这个自动代理创建器

```java
public abstract class AopConfigUtils {
	...
	@Nullable
	private static BeanDefinition registerOrEscalateApcAsRequired(
			Class<?> cls, BeanDefinitionRegistry registry, @Nullable Object source) {
		
		// 可以发现这里有一个很巧妙的处理：会对自动代理创建器进行升级~~~~
		// 所以如果你第一次进来的是`InfrastructureAdvisorAutoProxyCreator`，第二次进来的是`AnnotationAwareAspectJAutoProxyCreator`，那就会取第二次进来的这个Class
		// 反之则不行。这里面是维护的一个优先级顺序的，具体参看本类的static代码块，就是顺序  最后一个`AnnotationAwareAspectJAutoProxyCreator`才是最为强大的
		if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
			BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
			if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
				int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
				int requiredPriority = findPriorityForClass(cls);
				if (currentPriority < requiredPriority) {
					apcDefinition.setBeanClassName(cls.getName());
				}
			}
			return null;
		}

		RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
		beanDefinition.setSource(source);
		beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
		beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
		registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
		return beanDefinition;
	}
	...
}
```









SmartTransactionObject