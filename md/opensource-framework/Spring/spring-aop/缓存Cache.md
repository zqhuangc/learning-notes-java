

# **Spring 注解并不支持设置过期时间**



CachingProvider：创建、配置、获取、管理和控制多个CacheManager
CacheManager：创建、配置、获取、管理和控制多个唯一命名的Cache。（一个CacheManager仅被一个CachingProvider所拥有）
Cache：一个类似Map的数据结构。（一个Cache仅被一个CacheManager所拥有）
Entry：一个存储在Cache中的key-value对
Expiry：每一个存储在Cache中的条目有一个定义的有效期，过期后不可访问、更新、删除。缓存有效期可以通过ExpiryPolicy设置

### `CacheManager`

**CacheManager简单描述就是用来存放Cache，Cache用于存放具体的key-value值**

```java
// pring's central cache manager SPI.  它是个SPI接口
// @since 3.1
public interface CacheManager {
	@Nullable
	Cache getCache(String name);
	// 管理的所有的Cache的names~
	Collection<String> getCacheNames();
}
```

##### AbstractCacheManager

```java
// @since 3.1 实现了InitializingBean接口~~
public abstract class AbstractCacheManager implements CacheManager, InitializingBean {
	
	// 保存着所有的Cache对象~~key为名字
	private final ConcurrentMap<String, Cache> cacheMap = new ConcurrentHashMap<>(16);	
	//  此处使用了volatile 关键字
	private volatile Set<String> cacheNames = Collections.emptySet();

	@Override
	public void afterPropertiesSet() {
		initializeCaches();
	}

	// @since 4.2.2  模版方法模式。   abstract方法loadCaches()交给子类实现~~~
	public void initializeCaches() {
		Collection<? extends Cache> caches = loadCaches();

		synchronized (this.cacheMap) {
			this.cacheNames = Collections.emptySet();
			this.cacheMap.clear();
			Set<String> cacheNames = new LinkedHashSet<>(caches.size());
			for (Cache cache : caches) {
				String name = cache.getName();
				// decorateCache是protected方法，交给子类  不复写也无所谓~~~
				this.cacheMap.put(name, decorateCache(cache));
				cacheNames.add(name);
			}
			// cacheNames是个只读视图~（框架设计中考虑读写特性~）
			this.cacheNames = Collections.unmodifiableSet(cacheNames);
		}
	}

	protected abstract Collection<? extends Cache> loadCaches();

	// 根据名称 获取Cache对象。若没有，就返回null
	@Override
	@Nullable
	public Cache getCache(String name) {
		Cache cache = this.cacheMap.get(name);
		if (cache != null) {
			return cache;
		} else {
			// Fully synchronize now for missing cache creation...
			synchronized (this.cacheMap) {
				cache = this.cacheMap.get(name);
				if (cache == null) {
					// getMissingCache默认直接返回null，交给子类复写~~~~
					// 将决定权交给实现者，你可以创建一个Cache，或者记录日志
					cache = getMissingCache(name);
					// cache  != null，主要靠getMissingCache这个方法了~~~向一个工厂一样创建一个新的~~~
					if (cache != null) {
						cache = decorateCache(cache);
						this.cacheMap.put(name, cache);
						// 向全局缓存里面再添加进去一个Cache~~~~
						updateCacheNames(name);
					}
				}
				return cache;
			}
		}
	}

	@Override
	public Collection<String> getCacheNames() {
		return this.cacheNames;
	}

	// @since 4.1
	protected final Cache lookupCache(String name) {
		return this.cacheMap.get(name);
	}
	...
}
```

##### SimpleCacheManager

```java
// @since 3.1
public class SimpleCacheManager extends AbstractCacheManager {
	private Collection<? extends Cache> caches = Collections.emptySet();
	public void setCaches(Collection<? extends Cache> caches) {
		this.caches = caches;
	}
	@Override
	protected Collection<? extends Cache> loadCaches() {
		return this.caches;
	}
}
```

##### NoOpCacheManager

一种基本的、无操作的`CacheManager`实现，适用于**禁用缓存**，通常用于在没有实际存储的情况下作为缓存声明。

##### `ConcurrentMapCacheManager`(重要)

```java
// @since 3.1
public class ConcurrentMapCacheManager implements CacheManager, BeanClassLoaderAware {

	private final ConcurrentMap<String, Cache> cacheMap = new ConcurrentHashMap<>(16);
	// true表示：若为null，就新创建一个缓存添加进去
	private boolean dynamic = true;
	// 是否允许value为null
	private boolean allowNullValues = true;
	// 指定此缓存管理器是存储每个条目的副本，还是存储其所有缓存的引用
	// 影响到ConcurrentMapCache对value值的存储~~~
	// false表示：存储它自己（引用）
	// true表示：存储一个副本（对序列化有要求~~~） 很少这么使用~~~
	private boolean storeByValue = false;
	

	public ConcurrentMapCacheManager() {
	}
	public ConcurrentMapCacheManager(String... cacheNames) {
		setCacheNames(Arrays.asList(cacheNames));
	}
	public void setCacheNames(@Nullable Collection<String> cacheNames) {
		if (cacheNames != null) {
			for (String name : cacheNames) {
				this.cacheMap.put(name, createConcurrentMapCache(name));
			}
			this.dynamic = false; // 手动设置了，就不允许在动态创建了
		} else {
			this.dynamic = true;
		}
	}
	protected Cache createConcurrentMapCache(String name) {
		// isStoreByValue=true需要存储值的副本的时候，才对序列化有要求~~~否则直接存引用即可
		SerializationDelegate actualSerialization = (isStoreByValue() ? this.serialization : null);
		return new ConcurrentMapCache(name, new ConcurrentHashMap<>(256), isAllowNullValues(), actualSerialization);

	}

	@Override
	public void setBeanClassLoader(ClassLoader classLoader) {
		this.serialization = new SerializationDelegate(classLoader);
		if (isStoreByValue()) {
			// 重新创建Cache 因为要副本嘛
			recreateCaches();
		}
	}
	private void recreateCaches() {
		for (Map.Entry<String, Cache> entry : this.cacheMap.entrySet()) {
			entry.setValue(createConcurrentMapCache(entry.getKey()));
		}
	}

	@Override
	@Nullable
	public Cache getCache(String name) {
		Cache cache = this.cacheMap.get(name);
		// dynamic=true  为null的时候会动态创建一个
		if (cache == null && this.dynamic) {
			synchronized (this.cacheMap) {
				cache = this.cacheMap.get(name);
				if (cache == null) {
					cache = createConcurrentMapCache(name);
					this.cacheMap.put(name, cache);
				}
			}
		}
		return cache;
	}
	@Override
	public Collection<String> getCacheNames() {
		return Collections.unmodifiableSet(this.cacheMap.keySet());
	}
	...
}
```

它的缓存存储是**基于内存**的，所以它的生命周期是与应用关联的

##### CompositeCacheManager

```ja
// @since 3.1  
public class CompositeCacheManager implements CacheManager, InitializingBean {
	// 内部聚合管理着一批CacheManager
	private final List<CacheManager> cacheManagers = new ArrayList<>();
	// 若这个为true，则可以结合NoOpCacheManager实现效果~~~
	private boolean fallbackToNoOpCache = false;

	public CompositeCacheManager() {
	}
	public CompositeCacheManager(CacheManager... cacheManagers) {
		setCacheManagers(Arrays.asList(cacheManagers));
	}
	// 也可以调用此方法，来自己往里面添加（注意是添加）CacheManager们
	public void setCacheManagers(Collection<CacheManager> cacheManagers) {
		this.cacheManagers.addAll(cacheManagers);
	}

	public void setFallbackToNoOpCache(boolean fallbackToNoOpCache) {
		this.fallbackToNoOpCache = fallbackToNoOpCache;
	}

	// 如果fallbackToNoOpCache=true，那么在这个Bean初始化完成后，也就是在末尾添加一个NoOpCacheManager
	// 当然fallbackToNoOpCache默认值是false
	@Override
	public void afterPropertiesSet() {
		if (this.fallbackToNoOpCache) {
			this.cacheManagers.add(new NoOpCacheManager());
		}
	}


	// 找到一个cache不为null的就return了~
	@Override
	@Nullable
	public Cache getCache(String name) {
		for (CacheManager cacheManager : this.cacheManagers) {
			Cache cache = cacheManager.getCache(name);
			if (cache != null) {
				return cache;
			}
		}
		return null;
	}

	// 可以看到返回的是所有names，并且用的set
	@Override
	public Collection<String> getCacheNames() {
		Set<String> names = new LinkedHashSet<>();
		for (CacheManager manager : this.cacheManagers) {
			names.addAll(manager.getCacheNames());
		}
		return Collections.unmodifiableSet(names);
	}
}
```

### Cache

```java
public interface Cache {
	String getName();
	// 返回本地存储的那个。比如ConcurrentMapCache本地就是用的一个ConcurrentMap
	Object getNativeCache();
	
	// 就是用下面的ValueWrapper把值包装了一下而已~
	@Nullable
	ValueWrapper get(Object key);
	@Nullable
	<T> T get(Object key, @Nullable Class<T> type);
	@Nullable
	<T> T get(Object key, Callable<T> valueLoader);
	
	void put(Object key, @Nullable Object value);
	// @since 4.1
	// 不存在旧值直接put就先去了返回null，否则返回旧值（并且不会把新值put进去）
	@Nullable
	ValueWrapper putIfAbsent(Object key, @Nullable Object value);
	// 删除
	void evict(Object key);
	// 清空
	void clear();


	@FunctionalInterface
	interface ValueWrapper {
		@Nullable
		Object get();
	}
}
```

##### AbstractValueAdaptingCache

```java
// @since 4.2.2 出现得还是挺晚的~~~
public abstract class AbstractValueAdaptingCache implements Cache {
	private final boolean allowNullValues;
	protected AbstractValueAdaptingCache(boolean allowNullValues) {
		this.allowNullValues = allowNullValues;
	}
	// lookup为抽象方法
	@Override
	@Nullable
	public ValueWrapper get(Object key) {
		Object value = lookup(key);
		return toValueWrapper(value);
	}
	@Nullable
	protected abstract Object lookup(Object key);

	// lookup出来的value继续交给fromStoreValue()处理~  其实就是对null值进行了处理
	// 若是null值就返回null,而不是具体的值了~~~
	@Override
	@SuppressWarnings("unchecked")
	@Nullable
	public <T> T get(Object key, @Nullable Class<T> type) {
		Object value = fromStoreValue(lookup(key));
		if (value != null && type != null && !type.isInstance(value)) {
			throw new IllegalStateException("Cached value is not of required type [" + type.getName() + "]: " + value);
		}
		return (T) value;
	}

	// 它是protected 方法  子类有复写
	@Nullable
	protected Object fromStoreValue(@Nullable Object storeValue) {
		if (this.allowNullValues && storeValue == NullValue.INSTANCE) {
			return null;
		}
		return storeValue;
	}

	// 提供给子类使用的方法，对null值进行转换~  子类有复写
	protected Object toStoreValue(@Nullable Object userValue) {
		if (userValue == null) {
			if (this.allowNullValues) {
				return NullValue.INSTANCE;
			}
			throw new IllegalArgumentException("Cache '" + getName() + "' is configured to not allow null values but null was provided");
		}
		return userValue;
	}

	// 把value进行了一层包装为SimpleValueWrapper
	@Nullable
	protected Cache.ValueWrapper toValueWrapper(@Nullable Object storeValue) {
		return (storeValue != null ? new SimpleValueWrapper(fromStoreValue(storeValue)) : null);
	}
}
```

##### ConcurrentMapCache

```java
}

	@Override
	public void put(Object key, @Nullable Object value) {
		this.store.put(key, toStoreValue(value));
	}
	@Override
	@Nullable
	public ValueWrapper putIfAbsent(Object key, @Nullable Object value) {
		Object existing = this.store.putIfAbsent(key, toStoreValue(value));
		return toValueWrapper(existing);
	}
	@Override
	public void evict(Object key) {
		this.store.remove(key);
	}
	@Override
	public void clear() {
		this.store.clear();
	}
	...
}
```





### CacheOperation：缓存操作

缓存操作的基类。我们知道不同的缓存注解，都有不同的**缓存操作**并且注解内的属性也挺多，此类就是对缓存操作的抽象

```java
// @since 3.1  三大缓存注解属性的基类~~~
public abstract class CacheOperation implements BasicOperation {

	private final String name;
	private final Set<String> cacheNames;
	private final String key;
	private final String keyGenerator;
	private final String cacheManager;
	private final String cacheResolver;
	private final String condition;

	// 把toString()作为一个成员变量记着了   因为调用的次数太多
	private final String toString;

	// 构造方法是protected 的~~~  入参一个Builder来对各个属性赋值
	// builder方式是@since 4.3提供的，显然Spring4.3对这部分进行了改造~
	protected CacheOperation(Builder b) {
		this.name = b.name;
		this.cacheNames = b.cacheNames;
		this.key = b.key;
		this.keyGenerator = b.keyGenerator;
		this.cacheManager = b.cacheManager;
		this.cacheResolver = b.cacheResolver;
		this.condition = b.condition;
		this.toString = b.getOperationDescription().toString();
	}
	... // 省略所有的get/set（其实builder里都只有set方法~~~）

	// 它的public静态抽象内部类  Builder  简单的说，它是构建一个CacheOperation的构建器
	public abstract static class Builder {
		private String name = "";
		private Set<String> cacheNames = Collections.emptySet();
		private String key = "";
		private String keyGenerator = "";
		private String cacheManager = "";
		private String cacheResolver = "";
		private String condition = "";
		// 省略所有的get/set

		// 抽象方法  自行实现~
		public abstract CacheOperation build();
	}

}

// @since 4.1 
public interface BasicOperation {
	Set<String> getCacheNames();
}
```



#### CacheableOperation

```java
// @since 3.1
public class CacheableOperation extends CacheOperation {
	@Nullable
	private final String unless;
	private final boolean sync;

	public CacheableOperation(CacheableOperation.Builder b) {
		super(b);
		this.unless = b.unless;
		this.sync = b.sync;
	}
	... // 省略get方法（无set方法哦）

	// @since 4.3
	public static class Builder extends CacheOperation.Builder {
		@Nullable
		private String unless;
		private boolean sync;
		... // 生路set方法（没有get方法哦~）
	
		// Spring4.3抽象的这个技巧还是不错的，此处传this进去即可
		@Override
		public CacheableOperation build() {
			return new CacheableOperation(this);
		}

		@Override
		protected StringBuilder getOperationDescription() {
			StringBuilder sb = super.getOperationDescription();
			sb.append(" | unless='");
			sb.append(this.unless);
			sb.append("'");
			sb.append(" | sync='");
			sb.append(this.sync);
			sb.append("'");
			return sb;
		}

	}
}
```



#### CacheEvictOperation

#### CachePutOperation



### CacheOperationSource：缓存属性源

`缓存属性源`。该接口被`CacheInterceptor`它使用。它能够获取到`Method`上所有的缓存操作集合：

```java
/ @since 3.1
public interface CacheOperationSource {

	// 返回此Method方法上面所有的缓存操作~~~~CacheOperation 集合
	// 显然一个Method上可以对应有多个缓存操作~~~~
	@Nullable
	Collection<CacheOperation> getCacheOperations(Method method, @Nullable Class<?> targetClass);

}
```

AnnotationCacheOperationSource

#### NameMatchCacheOperationSource

根据**方法名称**来匹配看看作用在此方法上的缓存操作有哪些（不需要注解了）

```java
// @since 3.1
public class NameMatchCacheOperationSource implements CacheOperationSource, Serializable {

	/** Keys are method names; values are TransactionAttributes. */
	private Map<String, Collection<CacheOperation>> nameMap = new LinkedHashMap<>();

	// 你配置的时候，可以调用此方法。这里使用的也是add方法
	// 先拦截这个方法，然后看看这个方法有木有匹配的缓存操作，有点想AOP的配置
	public void setNameMap(Map<String, Collection<CacheOperation>> nameMap) {
		nameMap.forEach(this::addCacheMethod);
	}
	public void addCacheMethod(String methodName, Collection<CacheOperation> ops) {
		// 输出debug日志
		if (logger.isDebugEnabled()) {
			logger.debug("Adding method [" + methodName + "] with cache operations [" + ops + "]");
		}
		this.nameMap.put(methodName, ops);
	}


	// 这部分逻辑挺简单，就是根据方法去Map类里匹配到一个最为合适的Collection<CacheOperation>
	@Override
	@Nullable
	public Collection<CacheOperation> getCacheOperations(Method method, @Nullable Class<?> targetClass) {
		// look for direct name match
		String methodName = method.getName();
		Collection<CacheOperation> ops = this.nameMap.get(methodName);

		// 若不是null就直接return了~  首次进来，都会进入到这个逻辑~~~~
		if (ops == null) {
			// Look for most specific name match. // 找打一个最适合的  最匹配的
			String bestNameMatch = null;
			// 遍历所有外部已经制定进来了的方法名们~~~~
			for (String mappedName : this.nameMap.keySet()) {
				// isMatch就是Ant风格匹配~~(第一步)  光Ant匹配上了还不算
				// 第二步：bestNameMatch=null或者
				if (isMatch(methodName, mappedName)
						&& (bestNameMatch == null || bestNameMatch.length() <= mappedName.length())) {
					// mappedName为匹配上了的名称~~~~ 给bestNameMatch 赋值  
					// 继续循环，最终找到一个最佳的匹配~~~~
					ops = this.nameMap.get(mappedName);
					bestNameMatch = mappedName;
				}
			}
		}

		return ops;
	}

	protected boolean isMatch(String methodName, String mappedName) {
		return PatternMatchUtils.simpleMatch(mappedName, methodName);
	}
	...
}
```

#### AbstractFallbackCacheOperationSource

这个抽象方法主要目的是：让`缓存注解（当然此抽象类并不要求一定是注解，别的方式也成）`既能使用在类上，也能使用在方法上。方法上没找到，就`Fallback`到类上去找。

> 并且它还支持把注解写在接口上，哪怕你只是一个JDK动态代理的实现而已。**比如我们的MyBatis Mapper接口上也是可以直接使用缓存注解的**

```java
// @since 3.1
public abstract class AbstractFallbackCacheOperationSource implements CacheOperationSource {

	private static final Collection<CacheOperation> NULL_CACHING_ATTRIBUTE = Collections.emptyList();

	// 这个Map初始值可不小，因为它要缓存所有的Method
	private final Map<Object, Collection<CacheOperation>> attributeCache = new ConcurrentHashMap<>(1024);


	@Override
	@Nullable
	public Collection<CacheOperation> getCacheOperations(Method method, @Nullable Class<?> targetClass) {
		// 如果这个方法是Object的方法，那就不考虑缓存操作~~~
		if (method.getDeclaringClass() == Object.class) {
			return null;
		}
		
		// 以method和Class作为上面缓存Map的key
		Object cacheKey = getCacheKey(method, targetClass);
		Collection<CacheOperation> cached = this.attributeCache.get(cacheKey);
		if (cached != null) {
			return (cached != NULL_CACHING_ATTRIBUTE ? cached : null);
		}
		
		// 核心处理逻辑：包括AnnotationCacheOperationSource的主要逻辑也是沿用的这个模版
		else {

			// computeCacheOperations计算缓存属性，这个方法是本类的灵魂，见下面
			Collection<CacheOperation> cacheOps = computeCacheOperations(method, targetClass);
			if (cacheOps != null) {
				if (logger.isTraceEnabled()) {
					logger.trace("Adding cacheable method '" + method.getName() + "' with attribute: " + cacheOps);
				}
				this.attributeCache.put(cacheKey, cacheOps);
			} else { // 若没有标注属性的方法，用NULL_CACHING_ATTRIBUTE占位~ 不用null值哦~~~~ Spring内部大都不直接使用null
				this.attributeCache.put(cacheKey, NULL_CACHING_ATTRIBUTE);
			}
			return cacheOps;
		}
	}

	@Nullable
	private Collection<CacheOperation> computeCacheOperations(Method method, @Nullable Class<?> targetClass) {
		// Don't allow no-public methods as required.
		// allowPublicMethodsOnly()默认是false（子类复写后的默认值已经写为true了）
		// 也就是说：缓存注解只能标注在public方法上~~~~不接收别非public方法~~~
		if (allowPublicMethodsOnly() && !Modifier.isPublic(method.getModifiers())) {
			return null;
		}

		// The method may be on an interface, but we need attributes from the target class.
		// If the target class is null, the method will be unchanged.
		Method specificMethod = AopUtils.getMostSpecificMethod(method, targetClass);

		// First try is the method in the target class.
		// 第一步：先去该方法上找，看看有木有啥缓存属性  有就返回
		Collection<CacheOperation> opDef = findCacheOperations(specificMethod);
		if (opDef != null) {
			return opDef;
		}

		// Second try is the caching operation on the target class.
		// 第二步：方法上没有，就再去方法所在的类上去找。
		// isUserLevelMethod：我们自己书写的方法（非自动生成的） 才直接return，否则继续处理
		opDef = findCacheOperations(specificMethod.getDeclaringClass());
		if (opDef != null && ClassUtils.isUserLevelMethod(method)) {
			return opDef;
		}

		// 他俩不相等，说明method这个方法它是标注在接口上的，这里也给与了支持
		// 此处透露的性息：我们的缓存注解也可以标注在接口方法上，比如MyBatis的接口上都是ok的~~~~
		if (specificMethod != method) {
			// Fallback is to look at the original method.
			opDef = findCacheOperations(method);
			if (opDef != null) {
				return opDef;
			}
			// Last fallback is the class of the original method.
			opDef = findCacheOperations(method.getDeclaringClass());
			if (opDef != null && ClassUtils.isUserLevelMethod(method)) {
				return opDef;
			}
		}

		return null;
	}

	@Nullable
	protected abstract Collection<CacheOperation> findCacheOperations(Class<?> clazz);
	@Nullable
	protected abstract Collection<CacheOperation> findCacheOperations(Method method);
	protected boolean allowPublicMethodsOnly() {
		return false;
	}
}
```

> 抽象类提供的能力是：你的缓存属性可以放在方法上，方法上没有的话会去类上找
>
> 实现类：`AnnotationCacheOperationSource`

##### AnnotationCacheOperationSource

```java
// @since 3.1
public class AnnotationCacheOperationSource extends AbstractFallbackCacheOperationSource implements Serializable {

	// 是否只允许此注解标注在public方法上？？？下面有设置值为true
	// 该属性只能通过构造函数赋值
	private final boolean publicMethodsOnly;
	private final Set<CacheAnnotationParser> annotationParsers;

	// 默认就设置了publicMethodsOnly=true
	public AnnotationCacheOperationSource() {
		this(true);
	}
	public AnnotationCacheOperationSource(boolean publicMethodsOnly) {
		this.publicMethodsOnly = publicMethodsOnly;
		this.annotationParsers = Collections.singleton(new SpringCacheAnnotationParser());
	}
	...
	// 也可以自己来实现注解的解析器。（比如我们要做自定义注解的话，可以这么搞~~~）
	public AnnotationCacheOperationSource(CacheAnnotationParser... annotationParsers) {
		this.publicMethodsOnly = true;
		this.annotationParsers = new LinkedHashSet<>(Arrays.asList(annotationParsers));
	}

	// 这两个方法：核心事件都交给了CacheAnnotationParser.parseCacheAnnotations方法
	@Override
	@Nullable
	protected Collection<CacheOperation> findCacheOperations(Class<?> clazz) {
		return determineCacheOperations(parser -> parser.parseCacheAnnotations(clazz));
	}
	@Override
	@Nullable
	protected Collection<CacheOperation> findCacheOperations(Method method) {
		return determineCacheOperations(parser -> parser.parseCacheAnnotations(method));
	}

	// CacheOperationProvider就是一个函数式接口而已~~~类似Function~
	// 这块determineCacheOperations() + CacheOperationProvider接口的设计还是很巧妙的  可以学习一下
	@Nullable
	protected Collection<CacheOperation> determineCacheOperations(CacheOperationProvider provider) {
		Collection<CacheOperation> ops = null;
		
		// 调用我们设置进来的所有的CacheAnnotationParser一个一个的处理~~~
		for (CacheAnnotationParser annotationParser : this.annotationParsers) {
			Collection<CacheOperation> annOps = provider.getCacheOperations(annotationParser);
	
			// 理解这一块：说明我们方法上、类上是可以标注N个注解的，都会同时生效~~~最后都会combined 
			if (annOps != null) {
				if (ops == null) {
					ops = annOps;
				} else {
					Collection<CacheOperation> combined = new ArrayList<>(ops.size() + annOps.size());
					combined.addAll(ops);
					combined.addAll(annOps);
					ops = combined;
				}
			}
		}
		return ops;
	}
}
```

> 专用于处理三大缓存注解，就是获取标注在方法上的缓存注解们。
> 另外需要说明的是：若你想扩展你自己的缓存注解（比如加上超时时间TTL），你的处理器可以继承自`AnnotationCacheOperationSource`，然后进行自己的扩展
>
> CacheAnnotationParser

#### CompositeCacheOperationSource

```java
public class CompositeCacheOperationSource implements CacheOperationSource, Serializable {
	// 这里用的数组，表面只能赋值一次  并且只能通过构造函数赋值
	private final CacheOperationSource[] cacheOperationSources;
	public CompositeCacheOperationSource(CacheOperationSource... cacheOperationSources) {
		this.cacheOperationSources = cacheOperationSources;
	}

	public final CacheOperationSource[] getCacheOperationSources() {
		return this.cacheOperationSources;
	}

	@Override
	@Nullable
	public Collection<CacheOperation> getCacheOperations(Method method, @Nullable Class<?> targetClass) {
		Collection<CacheOperation> ops = null;
		for (CacheOperationSource source : this.cacheOperationSources) {
			Collection<CacheOperation> cacheOperations = source.getCacheOperations(method, targetClass);
			if (cacheOperations != null) {
				if (ops == null) {
					ops = new ArrayList<>();
				}
				ops.addAll(cacheOperations);
			}
		}
		return ops;
	}

}
```

所以组合进取的属性源，最终都会**叠加生效**。

##### BeanFactoryCacheOperationSourceAdvisor 

```java
public class BeanFactoryCacheOperationSourceAdvisor extends AbstractBeanFactoryPointcutAdvisor {

	@Nullable
	private CacheOperationSource cacheOperationSource;

	private final CacheOperationSourcePointcut pointcut = new CacheOperationSourcePointcut() {
		@Override
		@Nullable
		protected CacheOperationSource getCacheOperationSource() {
			return cacheOperationSource;
		}
	};
	...
}
```

#### `CacheAnnotationParser`：缓存注解解析器

```java
// @since 3.1
public interface CacheAnnotationParser {
	@Nullable
	Collection<CacheOperation> parseCacheAnnotations(Class<?> type);
	@Nullable
	Collection<CacheOperation> parseCacheAnnotations(Method method);
}
```

##### SpringCacheAnnotationParser

```java
// @since 3.1
public class SpringCacheAnnotationParser implements CacheAnnotationParser, Serializable {
	private static final Set<Class<? extends Annotation>> CACHE_OPERATION_ANNOTATIONS = new LinkedHashSet<>(8);
	// 它能处理的注解类型，一目了然
	static {
		CACHE_OPERATION_ANNOTATIONS.add(Cacheable.class);
		CACHE_OPERATION_ANNOTATIONS.add(CacheEvict.class);
		CACHE_OPERATION_ANNOTATIONS.add(CachePut.class);
		CACHE_OPERATION_ANNOTATIONS.add(Caching.class);
	}

	// 处理class和Method
	// 使用DefaultCacheConfig，把它传给parseCacheAnnotations()  来给注解属性搞定默认值
	// 至于为何自己要新new一个呢？？？其实是因为@CacheConfig它只能放在类上呀~~~（传Method也能获取到类）
	// 返回值都可以为null（没有标注此注解方法、类  那肯定返回null啊）
	@Override
	@Nullable
	public Collection<CacheOperation> parseCacheAnnotations(Class<?> type) {
		DefaultCacheConfig defaultConfig = new DefaultCacheConfig(type);
		return parseCacheAnnotations(defaultConfig, type);
	}
	@Override
	@Nullable
	public Collection<CacheOperation> parseCacheAnnotations(Method method) {
		DefaultCacheConfig defaultConfig = new DefaultCacheConfig(method.getDeclaringClass());
		return parseCacheAnnotations(defaultConfig, method);
	}

	// 找到方法/类上的注解们~~~
	private Collection<CacheOperation> parseCacheAnnotations(DefaultCacheConfig cachingConfig, AnnotatedElement ae) {
		// 第三个参数传的false：表示接口的注解它也会看一下~~~
		Collection<CacheOperation> ops = parseCacheAnnotations(cachingConfig, ae, false);
		// 若出现接口方法里也标了，实例方法里也标了，那就继续处理。让以实例方法里标注的为准~~~
		if (ops != null && ops.size() > 1) {
			// More than one operation found -> local declarations override interface-declared ones...
			Collection<CacheOperation> localOps = parseCacheAnnotations(cachingConfig, ae, true);
			if (localOps != null) {
				return localOps;
			}
		}
		return ops;
	}
	private Collection<CacheOperation> parseCacheAnnotations(DefaultCacheConfig cachingConfig, AnnotatedElement ae, boolean localOnly) {

		// localOnly=true，只看自己的不看接口的。false表示接口的也得看~
		Collection<? extends Annotation> anns = (localOnly ? AnnotatedElementUtils.getAllMergedAnnotations(ae, CACHE_OPERATION_ANNOTATIONS) : AnnotatedElementUtils.findAllMergedAnnotations(ae, CACHE_OPERATION_ANNOTATIONS));
		if (anns.isEmpty()) {
			return null;
		}

		// 这里为和写个1？？？因为绝大多数情况下，我们都只会标注一个注解~~
		final Collection<CacheOperation> ops = new ArrayList<>(1);

		// 这里的方法parseCacheableAnnotation/parsePutAnnotation等 说白了  就是把注解属性，转换封装成为`CacheOperation`对象~~
		// 注意parseCachingAnnotation方法，相当于~把注解属性转换成了CacheOperation对象  下面以它为例介绍
		anns.stream().filter(ann -> ann instanceof Cacheable).forEach(
				ann -> ops.add(parseCacheableAnnotation(ae, cachingConfig, (Cacheable) ann)));
		anns.stream().filter(ann -> ann instanceof CacheEvict).forEach(
				ann -> ops.add(parseEvictAnnotation(ae, cachingConfig, (CacheEvict) ann)));
		anns.stream().filter(ann -> ann instanceof CachePut).forEach(
				ann -> ops.add(parsePutAnnotation(ae, cachingConfig, (CachePut) ann)));
		anns.stream().filter(ann -> ann instanceof Caching).forEach(
				ann -> parseCachingAnnotation(ae, cachingConfig, (Caching) ann, ops));
		return ops;
	}

	// CacheableOperation是抽象类CacheOperation的子类~
	private CacheableOperation parseCacheableAnnotation(
			AnnotatedElement ae, DefaultCacheConfig defaultConfig, Cacheable cacheable) {

		// 这个builder是CacheOperation.Builder的子类，父类规定了所有注解通用的一些属性~~~
		CacheableOperation.Builder builder = new CacheableOperation.Builder();

		builder.setName(ae.toString());
		builder.setCacheNames(cacheable.cacheNames());
		builder.setCondition(cacheable.condition());
		builder.setUnless(cacheable.unless());
		builder.setKey(cacheable.key());
		builder.setKeyGenerator(cacheable.keyGenerator());
		builder.setCacheManager(cacheable.cacheManager());
		builder.setCacheResolver(cacheable.cacheResolver());
		builder.setSync(cacheable.sync());

		// DefaultCacheConfig是本类的一个内部类，来处理buider，给他赋值默认值~~~  比如默认的keyGenerator等等
		defaultConfig.applyDefault(builder);
		CacheableOperation op = builder.build();
		validateCacheOperation(ae, op); // 校验。key和KeyGenerator至少得有一个   CacheManager和CacheResolver至少得配置一个

		return op;
	}
	... // 解析其余注解略，一样的。

	// 它其实就是把三三者聚合了，一个一个的遍历。所以它最后一个参数传的ops，在内部进行add
	private void parseCachingAnnotation(AnnotatedElement ae, DefaultCacheConfig defaultConfig, Caching caching, Collection<CacheOperation> ops) {

		Cacheable[] cacheables = caching.cacheable();
		for (Cacheable cacheable : cacheables) {
			ops.add(parseCacheableAnnotation(ae, defaultConfig, cacheable));
		}
		CacheEvict[] cacheEvicts = caching.evict();
		for (CacheEvict cacheEvict : cacheEvicts) {
			ops.add(parseEvictAnnotation(ae, defaultConfig, cacheEvict));
		}
		CachePut[] cachePuts = caching.put();
		for (CachePut cachePut : cachePuts) {
			ops.add(parsePutAnnotation(ae, defaultConfig, cachePut));
		}
	}

	// 设置默认值的私有静态内部类
	private static class DefaultCacheConfig {

		private final Class<?> target;
		@Nullable
		private String[] cacheNames;
		@Nullable
		private String keyGenerator;
		@Nullable
		private String cacheManager;
		@Nullable
		private String cacheResolver;
		private boolean initialized = false;

		// 唯一构造函数~
		public DefaultCacheConfig(Class<?> target) {
			this.target = target;
		}

		public void applyDefault(CacheOperation.Builder builder) {
			// 如果还没初始化过，就进行初始化（毕竟一个类上有多个方法，这种默认通用设置只需要来一次即可）
			if (!this.initialized) {
				// 找到类上、接口上的@CacheConfig注解  它可以指定本类使用的cacheNames和keyGenerator和cacheManager和cacheResolver
				CacheConfig annotation = AnnotatedElementUtils.findMergedAnnotation(this.target, CacheConfig.class);
				if (annotation != null) {
					this.cacheNames = annotation.cacheNames();
					this.keyGenerator = annotation.keyGenerator();
					this.cacheManager = annotation.cacheManager();
					this.cacheResolver = annotation.cacheResolver();
				}
				this.initialized = true;
			}

			// 下面方法一句话总结：方法上没有指定的话，就用类上面的CacheConfig配置
			if (builder.getCacheNames().isEmpty() && this.cacheNames != null) {
				builder.setCacheNames(this.cacheNames);
			}
			if (!StringUtils.hasText(builder.getKey()) && !StringUtils.hasText(builder.getKeyGenerator()) && StringUtils.hasText(this.keyGenerator)) {
				builder.setKeyGenerator(this.keyGenerator);
			}

			if (StringUtils.hasText(builder.getCacheManager()) || StringUtils.hasText(builder.getCacheResolver())) {
				// One of these is set so we should not inherit anything
			} else if (StringUtils.hasText(this.cacheResolver)) {
				builder.setCacheResolver(this.cacheResolver);
			} else if (StringUtils.hasText(this.cacheManager)) {
				builder.setCacheManager(this.cacheManager);
			}
		}
	}

}
```

> **三大缓存注解**，最终都被收集到`CacheOperation`里来了，这也就和`CacheOperationSource`缓存属性源接口的功能照应

#### CacheOperationInvocationContext：执行上下文

代表缓存执行时的上下文

```java
//@since 4.1  注意泛型O extends BasicOperation
public interface CacheOperationInvocationContext<O extends BasicOperation> {

	// 缓存操作属性CacheOperation
	O getOperation();

	// 目标类、目标方法
	Object getTarget();
	Method getMethod();

	// 方法入参们
	Object[] getArgs();
}
```



##### CacheOperationContext

只有一个`CacheAspectSupport`内部类实现`CacheOperationContext`

```java
protected class CacheOperationContext implements CacheOperationInvocationContext<CacheOperation> {

	// 它里面包含了CacheOperation、Method、Class、Method targetMethod;(注意有两个Method)、AnnotatedElementKey、KeyGenerator、CacheResolver等属性
	// this.method = BridgeMethodResolver.findBridgedMethod(method);
	// this.targetMethod = (!Proxy.isProxyClass(targetClass) ? AopUtils.getMostSpecificMethod(method, targetClass)  : this.method);
	// this.methodKey = new AnnotatedElementKey(this.targetMethod, targetClass);
    private final CacheOperationMetadata metadata;
    private final Object[] args;
    private final Object target;
    private final Collection<? extends Cache> caches;
    private final Collection<String> cacheNames;
    
    @Nullable
    private Boolean conditionPassing; // 标志CacheOperation.conditon是否是true：表示通过  false表示未通过

    public CacheOperationContext(CacheOperationMetadata metadata, Object[] args, Object target) {
        this.metadata = metadata;
        this.args = extractArgs(metadata.method, args);
        this.target = target;
        // 这里方法里调用了cacheResolver.resolveCaches(context)方法来得到缓存们~~~~  CacheResolver
        this.caches = CacheAspectSupport.this.getCaches(this, metadata.cacheResolver);
        // 从caches拿到他们的names们
        this.cacheNames = createCacheNames(this.caches);
    }
	... // 省略get/set

    protected boolean isConditionPassing(@Nullable Object result) {
        if (this.conditionPassing == null) {
			// 如果配置了并且还没被解析过，此处就解析condition条件~~~
            if (StringUtils.hasText(this.metadata.operation.getCondition())) {

				// 执行上下文：此处就不得不提一个非常重要的它了：CacheOperationExpressionEvaluator
				// 它代表着缓存操作中SpEL的执行上下文~~~  具体可以先参与下面的对它的介绍
				// 解析conditon~
                EvaluationContext evaluationContext = createEvaluationContext(result);
                this.conditionPassing = evaluator.condition(this.metadata.operation.getCondition(),
                        this.metadata.methodKey, evaluationContext);
            } else {
                this.conditionPassing = true;
            }
        }
        return this.conditionPassing;
    }

	// 解析CacheableOperation和CachePutOperation好的unless
    protected boolean canPutToCache(@Nullable Object value) {
        String unless = "";
        if (this.metadata.operation instanceof CacheableOperation) {
            unless = ((CacheableOperation) this.metadata.operation).getUnless();
        } else if (this.metadata.operation instanceof CachePutOperation) {
            unless = ((CachePutOperation) this.metadata.operation).getUnless();
        }
        if (StringUtils.hasText(unless)) {
            EvaluationContext evaluationContext = createEvaluationContext(value);
            return !evaluator.unless(unless, this.metadata.methodKey, evaluationContext);
        }
        return true;
    }

	
	// 这里注意：生成key  需要注意步骤。
	// 若配置了key（非空串）：那就作为SpEL解析出来
	// 否则走keyGenerator去生成~~~（所以你会发现，即使咱们没有配置key和keyGenerator，程序依旧能正常work,只是生成的key很长而已~~~）
	// （keyGenerator你可以能没配置？？？？）
	// 若你自己没有手动指定过KeyGenerator，那会使用默认的SimpleKeyGenerator 它的实现比较简单
	// 其实若我们自定义KeyGenerator，我觉得可以继承自`SimpleKeyGenerator `，而不是直接实现接口~~~
    @Nullable
    protected Object generateKey(@Nullable Object result) {
        if (StringUtils.hasText(this.metadata.operation.getKey())) {
            EvaluationContext evaluationContext = createEvaluationContext(result);
            return evaluator.key(this.metadata.operation.getKey(), this.metadata.methodKey, evaluationContext);
        }
        // key的优先级第一位，没有指定采用生成器去生成~
        return this.metadata.keyGenerator.generate(this.target, this.metadata.method, this.args);
    }

    private EvaluationContext createEvaluationContext(@Nullable Object result) {
        return evaluator.createEvaluationContext(this.caches, this.metadata.method, this.args,
                this.target, this.metadata.targetClass, this.metadata.targetMethod, result, beanFactory);
    }
	...
}
```



### CacheOperationExpressionEvaluator：缓存操作执行器

`EventExpressionEvaluator`，它在解析`@EventListener`注解的`condition`属性的时候被使用到，它也继承自抽象父类`CachedExpressionEvaluator`

```java
// @since 3.1   注意抽象父类CachedExpressionEvaluator在Spring4.2才有
// CachedExpressionEvaluator里默认使用的解析器是：SpelExpressionParser  以及
// 还准备了一个ParameterNameDiscoverer  可以交给执行上文CacheEvaluationContext使用
class CacheOperationExpressionEvaluator extends CachedExpressionEvaluator {

	// 注意这两个属性是public的  在CacheAspectSupport里都有使用~~~
	public static final Object NO_RESULT = new Object();
	public static final Object RESULT_UNAVAILABLE = new Object();


	public static final String RESULT_VARIABLE = "result";

	private final Map<ExpressionKey, Expression> keyCache = new ConcurrentHashMap<>(64);
	private final Map<ExpressionKey, Expression> conditionCache = new ConcurrentHashMap<>(64);
	private final Map<ExpressionKey, Expression> unlessCache = new ConcurrentHashMap<>(64);

	// 这个方法是创建执行上下文。能给解释：为何可以使用#result这个key的原因
	// 此方法的入参确实不少：8个
	public EvaluationContext createEvaluationContext(Collection<? extends Cache> caches,
			Method method, Object[] args, Object target, Class<?> targetClass, Method targetMethod,
			@Nullable Object result, @Nullable BeanFactory beanFactory) {

		// root对象，此对象的属性决定了你的#root能够取值的范围，具体下面有贴出此类~
		CacheExpressionRootObject rootObject = new CacheExpressionRootObject(caches, method, args, target, targetClass);

		// 它继承自MethodBasedEvaluationContext，已经讲解过了，本文就不继续深究了~
		CacheEvaluationContext evaluationContext = new CacheEvaluationContext(rootObject, targetMethod, args, getParameterNameDiscoverer());
		if (result == RESULT_UNAVAILABLE) {
			evaluationContext.addUnavailableVariable(RESULT_VARIABLE);
		} else if (result != NO_RESULT) {
			// 这一句话就是为何我们可以使用#result的核心原因~
			evaluationContext.setVariable(RESULT_VARIABLE, result);
		}
		
		// 从这里可知，缓存注解里也是可以使用容器内的Bean的~
		if (beanFactory != null) {
			evaluationContext.setBeanResolver(new BeanFactoryResolver(beanFactory));
		}
		return evaluationContext;
	}

	// 都有缓存效果哦，因为都继承自抽象父类CachedExpressionEvaluator嘛
	// 抽象父类@since 4.2才出来，就是给解析过程提供了缓存的效果~~~~（注意此缓存非彼缓存）

	// 解析key的SpEL表达式
	@Nullable
	public Object key(String keyExpression, AnnotatedElementKey methodKey, EvaluationContext evalContext) {
		return getExpression(this.keyCache, methodKey, keyExpression).getValue(evalContext);
	}

	// condition和unless的解析结果若不是bool类型，统一按照false处理~~~~
	public boolean condition(String conditionExpression, AnnotatedElementKey methodKey, EvaluationContext evalContext) {
		return (Boolean.TRUE.equals(getExpression(this.conditionCache, methodKey, conditionExpression).getValue(evalContext, Boolean.class)));
	}
	public boolean unless(String unlessExpression, AnnotatedElementKey methodKey, EvaluationContext evalContext) {
		return (Boolean.TRUE.equals(getExpression(this.unlessCache, methodKey, unlessExpression).getValue(evalContext, Boolean.class)));
	}

	/**
	 * Clear all caches.
	 */
	void clear() {
		this.keyCache.clear();
		this.conditionCache.clear();
		this.unlessCache.clear();
	}
}

// #root可取的值，参考CacheExpressionRootObject这个对象的属性
// @since 3.1
class CacheExpressionRootObject {

	// 可见#root.caches竟然都是阔仪的~~~
	private final Collection<? extends Cache> caches;
	private final Method method;
	private final Object[] args;
	private final Object target;
	private final Class<?> targetClass;
	// 省略所有的get/set（其实只有get方法）
}
```

### CacheResolver

根据实际情况来动态解析使用哪个`Cache`，它是`Spring4.1`提供的新特性

```java
// @since 4.1
@FunctionalInterface
public interface CacheResolver {

	// 根据执行上下文，拿到最终的缓存Cache集合
	// CacheOperationInvocationContext：执行上下文
	Collection<? extends Cache> resolveCaches(CacheOperationInvocationContext<?> context);

}
```

#### AbstractCacheResolver

具体实现根据**调用上下文**提供缓存名称集合

```java
// @since 4.1  实现了InitializingBean
public abstract class AbstractCacheResolver implements CacheResolver, InitializingBean {
	
	// 课件它还是依赖于CacheManager的
	@Nullable
	private CacheManager cacheManager;

	// 构造函数统一protected
	protected AbstractCacheResolver() {
	}
	protected AbstractCacheResolver(CacheManager cacheManager) {
		this.cacheManager = cacheManager;
	}

	// 设置，指定一个CacheManager
	public void setCacheManager(CacheManager cacheManager) {
		this.cacheManager = cacheManager;
	}
	public CacheManager getCacheManager() {
		Assert.state(this.cacheManager != null, "No CacheManager set");
		return this.cacheManager;
	}
	
	
	// 做了一步校验而已~~~CacheManager 必须存在
	// 这是一个使用技巧哦   自己的在设计框架的框架的时候可以使用~
	@Override
	public void afterPropertiesSet()  {
		Assert.notNull(this.cacheManager, "CacheManager is required");
	}

	@Override
	public Collection<? extends Cache> resolveCaches(CacheOperationInvocationContext<?> context) {
		// getCacheNames抽象方法，子类去实现~~~~
		Collection<String> cacheNames = getCacheNames(context);
		if (cacheNames == null) {
			return Collections.emptyList();
		}


		// 根据cacheNames  去CacheManager里面拿到Cache对象， 作为最终的返回
		Collection<Cache> result = new ArrayList<>(cacheNames.size());
		for (String cacheName : cacheNames) {
			Cache cache = getCacheManager().getCache(cacheName);
			if (cache == null) {
				throw new IllegalArgumentException("Cannot find cache named '" +
						cacheName + "' for " + context.getOperation());
			}
			result.add(cache);
		}
		return result;
	}

	// 子类需要实现此抽象方法
	@Nullable
	protected abstract Collection<String> getCacheNames(CacheOperationInvocationContext<?> context);
}
```

实现了`resolveCaches()`方法。抽象方法的实现交给了实现类



### CacheOperationSourcePointcut

```java
// @since 3.1 它是个StaticMethodMatcherPointcut  所以方法入参它不关心
abstract class CacheOperationSourcePointcut extends StaticMethodMatcherPointcut implements Serializable {


	// 如果你这个类就是一个CacheManager，不切入
	@Override
	public boolean matches(Method method, Class<?> targetClass) {
		if (CacheManager.class.isAssignableFrom(targetClass)) {
			return false;
		}

		// 获取到当前的缓存属性源~~~getCacheOperationSource()是个抽象方法
		CacheOperationSource cas = getCacheOperationSource();

		// 下面一句话解释为：如果方法/类上标注有缓存相关的注解，就切入进取~~
		// 具体逻辑请参见方法：cas.getCacheOperations();
		return (cas != null && !CollectionUtils.isEmpty(cas.getCacheOperations(method, targetClass)));
	}

	@Override
	public boolean equals(Object other) {
		if (this == other) {
			return true;
		}
		if (!(other instanceof CacheOperationSourcePointcut)) {
			return false;
		}
		CacheOperationSourcePointcut otherPc = (CacheOperationSourcePointcut) other;
		return ObjectUtils.nullSafeEquals(getCacheOperationSource(), otherPc.getCacheOperationSource());
	}

	@Override
	public int hashCode() {
		return CacheOperationSourcePointcut.class.hashCode();
	}

	@Override
	public String toString() {
		return getClass().getName() + ": " + getCacheOperationSource();
	}


	/**
	 * Obtain the underlying {@link CacheOperationSource} (may be {@code null}).
	 * To be implemented by subclasses.
	 */
	@Nullable
	protected abstract CacheOperationSource getCacheOperationSource();

}
```

### CacheErrorHandler

处理缓存发生错误时的策略接口。在大多数情况下，提供者抛出的任何异常都应该简单地被抛出到客户机上，**但是在某些情况下，基础结构可能需要以不同的方式处理缓存提供者异常。这个时候可以使用此接口来处理**

```java
public interface CacheErrorHandler {
	void handleCacheGetError(RuntimeException exception, Cache cache, Object key);
	void handleCachePutError(RuntimeException exception, Cache cache, Object key, @Nullable Object value);
	void handleCacheEvictError(RuntimeException exception, Cache cache, Object key);
	void handleCacheClearError(RuntimeException exception, Cache cache);	
}
```

### AbstractCacheInvoker

在进行缓存操作时发生异常时，调用组件`CacheErrorHandler`来进行处理

```java
// @since 4.1
public abstract class AbstractCacheInvoker {
	//protected属性~
	protected SingletonSupplier<CacheErrorHandler> errorHandler;

	protected AbstractCacheInvoker() {
		this.errorHandler = SingletonSupplier.of(SimpleCacheErrorHandler::new);
	}
	protected AbstractCacheInvoker(CacheErrorHandler errorHandler) {
		this.errorHandler = SingletonSupplier.of(errorHandler);
	}

	// 此处用的obtain方法  它能保证不为null
	public CacheErrorHandler getErrorHandler() {
		return this.errorHandler.obtain();
	}

	@Nullable
	protected Cache.ValueWrapper doGet(Cache cache, Object key) {
		try {
			return cache.get(key);
		} catch (RuntimeException ex) {
			getErrorHandler().handleCacheGetError(ex, cache, key);
			// 只有它有返回值哦  返回null
			return null;  // If the exception is handled, return a cache miss
		}
	}
	... // 省略doPut、doEvict、doClear  处理方式同上
}
```





## 常用注解

方法参数与返回结果作为键值对

方法---方法参数----返回结果

### `@Cacheable`：缓存

```java
// @since 3.1  可以标注在方法上、类上  下同
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Cacheable {
	// 缓存名称  可以写多个~
	@AliasFor("cacheNames")
	String[] value() default {};
	@AliasFor("value")
	String[] cacheNames() default {};

	// 支持写SpEL，切可以使用#root
	String key() default "";
	// Mutually exclusive：它和key属性互相排斥。请只使用一个
	String keyGenerator() default "";

	String cacheManager() default "";
	String cacheResolver() default "";

	// SpEL，可以使用#root。  只有true时，才会作用在这个方法上
	String condition() default "";
	// 可以写SpEL #root，并且可以使用#result拿到方法返回值~~~
	String unless() default "";
	
	// true：表示强制同步执行。（若多个线程试图为**同一个键**加载值，以同步的方式来进行目标方法的调用）
	// 同步的好处是：后一个线程会读取到前一个缓存的缓存数据，不用再查库了~~~ 
	// 默认是false，不开启同步one by one的
	// @since 4.3  注意是sync而不是Async
	// 它的解析依赖于Spring4.3提供的Cache.get(Object key, Callable<T> valueLoader);方法
	boolean sync() default false;
}
```

### `@CachePut`：缓存更新

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Cacheable {

	@AliasFor("cacheNames")
	String[] value() default {};
	@AliasFor("value")
	String[] cacheNames() default {};

	// 注意：它和上面区别是。此处key它还能使用#result
	String key() default "";
	String keyGenerator() default "";

	String cacheManager() default "";
	String cacheResolver() default "";

	String condition() default "";
	String unless() default "";
}
```

### `@CacheEvict`：缓存删除

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Cacheable {

	@AliasFor("cacheNames")
	String[] value() default {};
	@AliasFor("value")
	String[] cacheNames() default {};

	// 它也能使用#result
	String key() default "";
	String keyGenerator() default "";

	String cacheManager() default "";
	String cacheResolver() default "";
	String condition() default "";

	// 是否把上面cacheNames指定的所有的缓存都清除掉，默认false
	boolean allEntries() default false;
	// 是否让清理缓存动作在目标方法之前执行，默认是false（在目标方法之后执行）
	// 注意：若在之后执行的话，目标方法一旦抛出异常了，那缓存就清理不掉了~~~~
	boolean beforeInvocation() default false;

}
```

### `@Caching`：用于处理复杂的缓存情况。比如用户既要根据id缓存一份，也要根据电话缓存一份，还要根据电子邮箱缓存一份，就可以使用它

```java
@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Inherited
@Documented
public @interface Caching {
	Cacheable[] cacheable() default {};
	CachePut[] put() default {};
	CacheEvict[] evict() default {};
}
```

### `@CacheConfig`：可以在类级别上标注一些共用的缓存属性。（所有方法共享，@since 4.1）

```java
// @since 4.1 出现得还是比较晚的
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface CacheConfig {
	String[] cacheNames() default {};
	String keyGenerator() default "";
	String cacheManager() default "";
	String cacheResolver() default "";
}
```

属性说明表格：

| 属性名           | 解释                                                         |
| ---------------- | ------------------------------------------------------------ |
| value            | 缓存的名称。可定义多个（至少需要定义一个）                   |
| cacheNames       | 同value属性                                                  |
| keyGenerator     | key生成器。字符串为：beanName                                |
| key              | 缓存的 key。可使用SpEL。优先级大于keyGenerator               |
| cacheManager     | 缓存管理器。填写beanName                                     |
| cacheResolver    | 缓存处理器。填写beanName                                     |
| condition        | 缓存条件。若填写了，返回true才会执行此缓存。可使用SpEL       |
| unless           | 否定缓存。false就生效。可以写SpEL                            |
| sync             | true：所有相同key的同线程顺序执行。默认值是false             |
| allEntries       | 是否清空所有缓存内容，缺省为 false，如果指定为 true，则方法调用后将立即清空所有缓存 |
| beforeInvocation | 是否在方法执行前就清空，缺省为 false，如果指定为 true        |



## EnableCaching

### 开启缓存注解的步骤

1. 配置类上开启缓存注解支持：`@EnableCaching`
2. `@EnableCaching`这注解用于开启Spring的缓存注解功能，它是一个模块注解，功能类似于xml时代的：`<cache:annotation-driven>`配置项。向容器内至少放置一个`CacheManager`类型的Bean

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(CachingConfigurationSelector.class)
public @interface EnableCaching {
	boolean proxyTargetClass() default false;
	AdviceMode mode() default AdviceMode.PROXY;
	int order() default Ordered.LOWEST_PRECEDENCE;
}
```



### CachingConfigurationSelector

```java
public class CachingConfigurationSelector extends AdviceModeImportSelector<EnableCaching> {
	
	...

	static {
		ClassLoader classLoader = CachingConfigurationSelector.class.getClassLoader();
		jsr107Present = ClassUtils.isPresent("javax.cache.Cache", classLoader);
		jcacheImplPresent = ClassUtils.isPresent(PROXY_JCACHE_CONFIGURATION_CLASS, classLoader);
	}

	// 绝大多数情况下我们不会使用ASPECTJ，若要使用它还得额外导包~
	// getAspectJImports()这个方法略
	@Override
	public String[] selectImports(AdviceMode adviceMode) {
		switch (adviceMode) {
			case PROXY:
				return getProxyImports();
			case ASPECTJ:
				return getAspectJImports();
			default:
				return null;
		}
	}

	// 向容器导入了AutoProxyRegistrar和ProxyCachingConfiguration
	// 若JSR107的包存在(导入了javax.cache:cache-api这个包)，并且并且存在ProxyJCacheConfiguration这个类
	// 显然ProxyJCacheConfiguration这个类我们一般都不会导进来~~~~  所以JSR107是不生效的。   但是但是Spring是支持的，非常良心
	private String[] getProxyImports() {
		List<String> result = new ArrayList<>(3);
		result.add(AutoProxyRegistrar.class.getName());
		result.add(ProxyCachingConfiguration.class.getName());
		if (jsr107Present && jcacheImplPresent) {
			result.add(PROXY_JCACHE_CONFIGURATION_CLASS);
		}
		return StringUtils.toStringArray(result);
	}
	...
}
```



### ProxyCachingConfiguration

```java
// @since 3.1  内部注入的Bean角色都是ROLE_INFRASTRUCTURE  建议先看下面的父类
@Configuration
@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
public class ProxyCachingConfiguration extends AbstractCachingConfiguration {

	// 缓存注解的增强器：重点在CacheOperationSource和CacheInterceptor
	@Bean(name = CacheManagementConfigUtils.CACHE_ADVISOR_BEAN_NAME)
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public BeanFactoryCacheOperationSourceAdvisor cacheAdvisor() {
		BeanFactoryCacheOperationSourceAdvisor advisor = new BeanFactoryCacheOperationSourceAdvisor();
		advisor.setCacheOperationSource(cacheOperationSource());
		advisor.setAdvice(cacheInterceptor());
		if (this.enableCaching != null) {
			advisor.setOrder(this.enableCaching.<Integer>getNumber("order"));
		}
		return advisor;
	}

	// 关于CacheOperationSource和CacheInterceptor是理解注解原理重要的两个类，放在下一章节讲解更妥
	// 注意CacheOperationSource是给CacheInterceptor用的
	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public CacheOperationSource cacheOperationSource() {
		return new AnnotationCacheOperationSource();
	}
	@Bean
	@Role(BeanDefinition.ROLE_INFRASTRUCTURE)
	public CacheInterceptor cacheInterceptor() {
		CacheInterceptor interceptor = new CacheInterceptor();
		interceptor.configure(this.errorHandler, this.keyGenerator, this.cacheResolver, this.cacheManager);
		interceptor.setCacheOperationSource(cacheOperationSource());
		return interceptor;
	}
}

// 抽象父类：定义和管理了Caching相关的API、类们
@Configuration
public abstract class AbstractCachingConfiguration implements ImportAware {

	// 这些属性都是protected的，子类可以直接访问哦~~~
	@Nullable
	protected AnnotationAttributes enableCaching;
	@Nullable
	protected Supplier<CacheManager> cacheManager;
	@Nullable
	protected Supplier<CacheResolver> cacheResolver; // 缓存注解的处理器
	@Nullable
	protected Supplier<KeyGenerator> keyGenerator; // key的生成器
	@Nullable
	protected Supplier<CacheErrorHandler> errorHandler; // 错误处理器


	// 继承此抽象类前提条件：必须标注@EnableCaching注解~
	@Override
	public void setImportMetadata(AnnotationMetadata importMetadata) {
		this.enableCaching = AnnotationAttributes.fromMap(importMetadata.getAnnotationAttributes(EnableCaching.class.getName(), false));
		if (this.enableCaching == null) {
			throw new IllegalArgumentException("@EnableCaching is not present on importing class " + importMetadata.getClassName());
		}
	}

	// 可以通过实现CachingConfigurer接口来**指定缓存使用的默认**的：
	// 缓存管理器
	// 缓存解析器
	// key生成器
	// 错误处理器
	@Autowired(required = false)
	void setConfigurers(Collection<CachingConfigurer> configurers) {
		if (CollectionUtils.isEmpty(configurers)) {
			return;
		}
		// CachingConfigurer的实现类，最多只能有一个
		if (configurers.size() > 1) {
			throw new IllegalStateException(configurers.size() + " implementations of " +
					"CachingConfigurer were found when only 1 was expected. " +
					"Refactor the configuration such that CachingConfigurer is " +
					"implemented only once or not at all.");
		}
		CachingConfigurer configurer = configurers.iterator().next();
		useCachingConfigurer(configurer);
	}

	/**
	 * Extract the configuration from the nominated {@link CachingConfigurer}.
	 */
	protected void useCachingConfigurer(CachingConfigurer config) {
		this.cacheManager = config::cacheManager;
		this.cacheResolver = config::cacheResolver;
		this.keyGenerator = config::keyGenerator;
		this.errorHandler = config::errorHandler;
	}

}
```



### CacheOperationSource



###  BeanFactoryCacheOperationSourceAdvisor

```java
public class BeanFactoryCacheOperationSourceAdvisor extends AbstractBeanFactoryPointcutAdvisor {
	@Nullable
	private CacheOperationSource cacheOperationSource;

	// 切面Pointcut
	private final CacheOperationSourcePointcut pointcut = new CacheOperationSourcePointcut() {
		@Override
		@Nullable
		protected CacheOperationSource getCacheOperationSource() {
			return cacheOperationSource;
		}
	};

	public void setCacheOperationSource(CacheOperationSource cacheOperationSource) {
		this.cacheOperationSource = cacheOperationSource;
	}

	// 注意：此处你可以自定义一个ClassFilter，过滤掉你想忽略的类
	public void setClassFilter(ClassFilter classFilter) {
		this.pointcut.setClassFilter(classFilter);
	}

	@Override
	public Pointcut getPointcut() {
		return this.pointcut;
	}
}
```



### CacheAspectSupport(类似 TransactionAspectSupport)

```java
// @since 3.1  它相较于TransactionAspectSupport额外实现了SmartInitializingSingleton接口
// SmartInitializingSingleton应该也不会陌生。它在初始化完所有的单例Bean后会执行这个接口的`afterSingletonsInstantiated()`方法
// 比如我们熟悉的ScheduledAnnotationBeanPostProcessor、EventListenerMethodProcessor都是这么来处理的

// 另外还需要注意，它还继承自AbstractCacheInvoker：主要对异常情况用CacheErrorHandler处理
public abstract class CacheAspectSupport extends AbstractCacheInvoker implements BeanFactoryAware, InitializingBean, SmartInitializingSingleton {

	// CacheOperationCacheKey：缓存的key  CacheOperationMetadata就是持有一些基础属性的性息
	// 这个缓存挺大，相当于每一个类、方法都有气对应的**缓存属性元数据**
	
	private final Map<CacheOperationCacheKey, CacheOperationMetadata> metadataCache = new ConcurrentHashMap<>(1024);
	// 解析一些condition、key、unless等可以写el表达式的处理器~~~
	// 之前讲过的熟悉的有：EventExpressionEvaluator
	private final CacheOperationExpressionEvaluator evaluator = new CacheOperationExpressionEvaluator();
	// 属性源，默认情况下是基于注解的`AnnotationCacheOperationSource`
	@Nullable
	private CacheOperationSource cacheOperationSource;
	// 看到了吧  key生成器默认使用的SimpleKeyGenerator
	// 注意SingletonSupplier是Spring5.1的新类，实现了接口java.util.function.Supplier  主要是对null值进行了容错
	private SingletonSupplier<KeyGenerator> keyGenerator = SingletonSupplier.of(SimpleKeyGenerator::new);

	@Nullable
	private SingletonSupplier<CacheResolver> cacheResolver;
	@Nullable
	private BeanFactory beanFactory;
	private boolean initialized = false;

	// @since 5.1
	public void configure(@Nullable Supplier<CacheErrorHandler> errorHandler, @Nullable Supplier<KeyGenerator> keyGenerator,@Nullable Supplier<CacheResolver> cacheResolver, @Nullable Supplier<CacheManager> cacheManager) {
		// 第二个参数都是默认值，若调用者没传的话
		this.errorHandler = new SingletonSupplier<>(errorHandler, SimpleCacheErrorHandler::new);
		this.keyGenerator = new SingletonSupplier<>(keyGenerator, SimpleKeyGenerator::new);
		this.cacheResolver = new SingletonSupplier<>(cacheResolver, () -> SimpleCacheResolver.of(SupplierUtils.resolve(cacheManager)));
	}

	// 此处：若传入了多个cacheOperationSources，那最终使用的就是CompositeCacheOperationSource包装起来
	// 所以发现，Spring是支持我们多种 缓存属性源的
	public void setCacheOperationSources(CacheOperationSource... cacheOperationSources) {
		Assert.notEmpty(cacheOperationSources, "At least 1 CacheOperationSource needs to be specified");
		this.cacheOperationSource = (cacheOperationSources.length > 1 ? new CompositeCacheOperationSource(cacheOperationSources) : cacheOperationSources[0]);
	}
	// @since 5.1 单数形式的设置
	public void setCacheOperationSource(@Nullable CacheOperationSource cacheOperationSource) {
		this.cacheOperationSource = cacheOperationSource;
	}
	
	... // 省略各种get/set方法~~~

	// CacheOperationSource必须不为null，因为一切依托于它
	@Override
	public void afterPropertiesSet() {
		Assert.state(getCacheOperationSource() != null, "The 'cacheOperationSources' property is required: " + "If there are no cacheable methods, then don't use a cache aspect.");
	}

	// 这个来自于接口：SmartInitializingSingleton  在实例化完所有单例Bean后调用
	@Override
	public void afterSingletonsInstantiated() {
		// 若没有给这个切面手动设置cacheResolver  那就去拿CacheManager吧
		// 这就是为何我们只需要把CacheManager配进容器里即可  就自动会设置在切面里了
		if (getCacheResolver() == null) {
			// Lazily initialize cache resolver via default cache manager...
			Assert.state(this.beanFactory != null, "CacheResolver or BeanFactory must be set on cache aspect");
			try {
				// 请注意：这个方法实际上是把CacheManager包装成了一个SimpleCacheResolver
				// 所以最终还是给SimpleCacheResolver赋值
				setCacheManager(this.beanFactory.getBean(CacheManager.class));
			} ...
		}
		this.initialized = true;
	}

	// 主要为了输出日志，子类可复写
	protected String methodIdentification(Method method, Class<?> targetClass) {
		Method specificMethod = ClassUtils.getMostSpecificMethod(method, targetClass);
		return ClassUtils.getQualifiedMethodName(specificMethod);
	}

	// 从这里也能看出，至少要指定一个Cache才行（也就是cacheNames）
	protected Collection<? extends Cache> getCaches(CacheOperationInvocationContext<CacheOperation> context, CacheResolver cacheResolver) {

		Collection<? extends Cache> caches = cacheResolver.resolveCaches(context);
		if (caches.isEmpty()) {
			throw new IllegalStateException("No cache could be resolved for '" +
					context.getOperation() + "' using resolver '" + cacheResolver +
					"'. At least one cache should be provided per cache operation.");
		}
		return caches;
	}


	// 这个根据CacheOperation 这部分还是比较重要的
	protected CacheOperationMetadata getCacheOperationMetadata(CacheOperation operation, Method method, Class<?> targetClass) {
		CacheOperationCacheKey cacheKey = new CacheOperationCacheKey(operation, method, targetClass);
		CacheOperationMetadata metadata = this.metadataCache.get(cacheKey);


		if (metadata == null) {
			// 1、指定了KeyGenerator就去拿这个Bean（没有就报错，所以key不要写错了）
			// 没有指定就用默认的
			KeyGenerator operationKeyGenerator;
			if (StringUtils.hasText(operation.getKeyGenerator())) {
				operationKeyGenerator = getBean(operation.getKeyGenerator(), KeyGenerator.class);
			} else {
				operationKeyGenerator = getKeyGenerator();
			}

			// 1、自己指定的CacheResolver
			// 2、再看指定的的CacheManager,包装成一个SimpleCacheResolver
			// 3、
			CacheResolver operationCacheResolver;
			if (StringUtils.hasText(operation.getCacheResolver())) {
				operationCacheResolver = getBean(operation.getCacheResolver(), CacheResolver.class);
			} else if (StringUtils.hasText(operation.getCacheManager())) {
				CacheManager cacheManager = getBean(operation.getCacheManager(), CacheManager.class);
				operationCacheResolver = new SimpleCacheResolver(cacheManager);
			} else { //最终都没配置的话，取本切面默认的
				operationCacheResolver = getCacheResolver();
				Assert.state(operationCacheResolver != null, "No CacheResolver/CacheManager set");
			}

			// 封装成Metadata
			metadata = new CacheOperationMetadata(operation, method, targetClass, operationKeyGenerator, operationCacheResolver);
			this.metadataCache.put(cacheKey, metadata);
		}
		return metadata;
	}

	// qualifiedBeanOfType的意思是，@Bean类上面标注@Qualifier注解也生效
	protected <T> T getBean(String beanName, Class<T> expectedType) {
		if (this.beanFactory == null) {
			throw new IllegalStateException(
					"BeanFactory must be set on cache aspect for " + expectedType.getSimpleName() + " retrieval");
		}
		return BeanFactoryAnnotationUtils.qualifiedBeanOfType(this.beanFactory, expectedType, beanName);
	}

	// 请Meta数据的缓存
	protected void clearMetadataCache() {
		this.metadataCache.clear();
		this.evaluator.clear();
	}


	// 父类最为核心的方法，真正执行目标方法 + 缓存操作
	@Nullable
	protected Object execute(CacheOperationInvoker invoker, Object target, Method method, Object[] args) {
		// Check whether aspect is enabled (to cope with cases where the AJ is pulled in automatically)
		// 如果已经表示初始化过了(有CacheManager，CacheResolver了)，执行这里
		if (this.initialized) {
			// getTargetClass拿到原始Class  解剖代理（N层都能解开）
			Class<?> targetClass = getTargetClass(target);
			CacheOperationSource cacheOperationSource = getCacheOperationSource();
			
			if (cacheOperationSource != null) {
				// 简单的说就是拿到该方法上所有的CacheOperation缓存操作，最终一个一个的执行~~~~
				Collection<CacheOperation> operations = cacheOperationSource.getCacheOperations(method, targetClass);
				if (!CollectionUtils.isEmpty(operations)) {

					// CacheOperationContexts是非常重要的一个私有内部类
					// 注意它是复数哦~不是CacheOperationContext单数  所以它就像持有多个注解上下文一样  一个个执行吧
					// 所以我建议先看看此类的描述，再继续往下看~~~
					return execute(invoker, method, new CacheOperationContexts(operations, method, args, target, targetClass));
				}
			}
		}

		// 若还没初始化  直接执行目标方法即可
		return invoker.invoke();
	}



	@Nullable
	private Object execute(final CacheOperationInvoker invoker, Method method, CacheOperationContexts contexts) {
		// Special handling of synchronized invocation
		// 如果是需要同步执行的话，这块还是
		if (contexts.isSynchronized()) {
			CacheOperationContext context = contexts.get(CacheableOperation.class).iterator().next();
			if (isConditionPassing(context, CacheOperationExpressionEvaluator.NO_RESULT)) {
				Object key = generateKey(context, CacheOperationExpressionEvaluator.NO_RESULT);
				Cache cache = context.getCaches().iterator().next();
				try {
					return wrapCacheValue(method, cache.get(key, () -> unwrapReturnValue(invokeOperation(invoker))));
				}
				catch (Cache.ValueRetrievalException ex) {
					// The invoker wraps any Throwable in a ThrowableWrapper instance so we
					// can just make sure that one bubbles up the stack.
					throw (CacheOperationInvoker.ThrowableWrapper) ex.getCause();
				}
			}
			else {
				// No caching required, only call the underlying method
				return invokeOperation(invoker);
			}
		}

		// sync=false的情况，走这里~~~

		
		// Process any early evictions  beforeInvocation=true的会在此处最先执行~~~
		
		// 最先处理@CacheEvict注解~~~真正执行的方法请参见：performCacheEvict
		// context.getCaches()拿出所有的caches，看看是执行cache.evict(key);方法还是cache.clear();而已
		// 需要注意的的是context.isConditionPassing(result); condition条件此处生效，并且可以使用#result
		// context.generateKey(result)也能使用#result
		// @CacheEvict没有unless属性
		processCacheEvicts(contexts.get(CacheEvictOperation.class), true, CacheOperationExpressionEvaluator.NO_RESULT);

	
		// 执行@Cacheable  看看缓存是否能够命中
		Cache.ValueWrapper cacheHit = findCachedItem(contexts.get(CacheableOperation.class));

		// Collect puts from any @Cacheable miss, if no cached item is found
		List<CachePutRequest> cachePutRequests = new LinkedList<>();
		// 如果缓存没有命中，那就准备一个cachePutRequest
		// 因为@Cacheable首次进来肯定命中不了，最终肯定是需要执行一次put操作的~~~这样下次进来就能命中了呀
		if (cacheHit == null) {
			collectPutRequests(contexts.get(CacheableOperation.class), CacheOperationExpressionEvaluator.NO_RESULT, cachePutRequests);
		}

		Object cacheValue;
		Object returnValue;

		// 如果缓存命中了，并且并且没有@CachePut的话，也就直接返回了~~
		if (cacheHit != null && !hasCachePut(contexts)) {
			// If there are no put requests, just use the cache hit
			cacheValue = cacheHit.get();
			// wrapCacheValue主要是支持到了Optional
			returnValue = wrapCacheValue(method, cacheValue);
		} else { //到此处，目标方法就肯定是需要执行了的~~~~~
			// Invoke the method if we don't have a cache hit
			// 啥都不说，先invokeOperation执行目标方法，拿到方法的的返回值  后续在处理put啥的
			returnValue = invokeOperation(invoker);
			cacheValue = unwrapReturnValue(returnValue);
		}

		// Collect any explicit @CachePuts   explicit：明确的
		collectPutRequests(contexts.get(CachePutOperation.class), cacheValue, cachePutRequests);

		// Process any collected put requests, either from @CachePut or a @Cacheable miss
		for (CachePutRequest cachePutRequest : cachePutRequests) {
			// 注意：此处unless啥的生效~~~~
			// 最终执行cache.put(key, result);方法
			cachePutRequest.apply(cacheValue);
		}

		// Process any late evictions beforeInvocation=true的会在此处最先执行~~~  beforeInvocation=false的会在此处最后执行~~~
		// 所以中途若抛出异常，此部分就不会执行了~~~~
		processCacheEvicts(contexts.get(CacheEvictOperation.class), false, cacheValue);
		return returnValue;
	}


	// 缓存属性的上下文们。每个方法可以对应多个上下文~~~
	private class CacheOperationContexts {

		// 因为方法上可以标注多个注解  
		// 需要注意的是它的key是Class，而CacheOperation的子类也就那三个哥们而已~
		private final MultiValueMap<Class<? extends CacheOperation>, CacheOperationContext> contexts;
		// 是否要求同步执行，默认值是false
		private final boolean sync;

		public CacheOperationContexts(Collection<? extends CacheOperation> operations, Method method, Object[] args, Object target, Class<?> targetClass) {

			this.contexts = new LinkedMultiValueMap<>(operations.size());
			for (CacheOperation op : operations) {
				this.contexts.add(op.getClass(), getOperationContext(op, method, args, target, targetClass));
			}

			// sync这个属性虽然不怎么使用，但determineSyncFlag这个方法可以看一下
			this.sync = determineSyncFlag(method);
		}

		public Collection<CacheOperationContext> get(Class<? extends CacheOperation> operationClass) {
			Collection<CacheOperationContext> result = this.contexts.get(operationClass);
			return (result != null ? result : Collections.emptyList());
		}
		public boolean isSynchronized() {
			return this.sync;
		}


		// 因为只有@Cacheable有sync属性，所以只需要看CacheableOperation即可
		private boolean determineSyncFlag(Method method) {
			List<CacheOperationContext> cacheOperationContexts = this.contexts.get(CacheableOperation.class);
			if (cacheOperationContexts == null) {  // no @Cacheable operation at all
				return false;
			}
			
			boolean syncEnabled = false;
			// 单反只要有一个@Cacheable的sync=true了，那就为true  并且下面还有检查逻辑
			for (CacheOperationContext cacheOperationContext : cacheOperationContexts) {
				if (((CacheableOperation) cacheOperationContext.getOperation()).isSync()) {
					syncEnabled = true;
					break;
				}
			}

			// 执行sync=true的检查逻辑
			if (syncEnabled) {
				// 人话解释：sync=true时候，不能还有其它的缓存操作 也就是说@Cacheable(sync=true)的时候只能单独使用
				if (this.contexts.size() > 1) {
					throw new IllegalStateException("@Cacheable(sync=true) cannot be combined with other cache operations on '" + method + "'");
				}
				// 人话解释：@Cacheable(sync=true)时，多个@Cacheable也是不允许的
				if (cacheOperationContexts.size() > 1) {
					throw new IllegalStateException("Only one @Cacheable(sync=true) entry is allowed on '" + method + "'");
				}

				// 拿到唯一的一个@Cacheable
				CacheOperationContext cacheOperationContext = cacheOperationContexts.iterator().next();
				CacheableOperation operation = (CacheableOperation) cacheOperationContext.getOperation();

				// 人话解释：@Cacheable(sync=true)时，cacheName只能使用一个
				if (cacheOperationContext.getCaches().size() > 1) {
					throw new IllegalStateException("@Cacheable(sync=true) only allows a single cache on '" + operation + "'");
				}
				// 人话解释：sync=true时，unless属性是不支持的~~~并且是不能写的
				if (StringUtils.hasText(operation.getUnless())) {
					throw new IllegalStateException("@Cacheable(sync=true) does not support unless attribute on '" + operation + "'");
				}
				return true; // 只有校验都通过后，才返回true
			}
			return false;
		}
	}
	...
}
```

Spring处理的步骤总结如下：

CacheOperation封装了@CachePut、@Cacheable、@CacheEvict（下称三大缓存注解）的属性信息，以便于拦截的时候能直接操作此对象来执行逻辑。
1. 解析三大注解到CacheOperation的过程是由CacheAnnotationParser完成的
2. CacheAnnotationSource代表缓存属性源，非常非常重要的一个概念。它提供接口方法来获取目标方法的CacheOperation集合。由上可知，这个具体工作是委托给CacheAnnotationParser去完成的
3. BeanFactoryCacheOperationSourceAdvisor它代表增强器，至于需要增强哪些类呢？？？就是看有没有存在CacheOperation属性的方法
4. CacheInterceptor实现了MethodInterceptor接口，在Spring AOP中实现对执行方法的拦截。在调用invoke执行目标方法前后，通过CacheAnnotationSource获取到方法所有的缓存操作属性，从而一个个的执行
  执行的时候，每一个CacheOperation最后被封装成了CacheOperationContext，而
5. CacheOperationContext最终通过CacheResolver解析出缓存对象Cache（可能是多个）
6. 最后最后最后，CacheInterceptor调用其父类AbstractCacheInvoker执行对应的doPut / doGet / doEvict / doClear 等等。（可以处理执行异常）


#### CacheInterceptor(类似 TransactionInterceptor)

```java
// @since 3.1  它是个MethodInterceptor环绕增强器~~~
public class CacheInterceptor extends CacheAspectSupport implements MethodInterceptor, Serializable {

	@Override
	@Nullable
	public Object invoke(final MethodInvocation invocation) throws Throwable {
		Method method = invocation.getMethod();

		// 采用函数的形式，最终把此函数传交给父类的execute()去执行
		// 但是很显然，最终**执行目标方法**的是invocation.proceed();它
		
		//这里就是对执行方法调用的一次封装，主要是为了处理对异常的包装。
		CacheOperationInvoker aopAllianceInvoker = () -> {
			try {
				return invocation.proceed();
			}
			catch (Throwable ex) {
				throw new CacheOperationInvoker.ThrowableWrapper(ex);
			}
		};

		try {
			// //真正地去处理缓存操作的执行，很显然这是父类的方法，所以我们要到父类CacheAspectSupport中去看看。
			return execute(aopAllianceInvoker, invocation.getThis(), method, invocation.getArguments());
		} catch (CacheOperationInvoker.ThrowableWrapper th) {
			throw th.getOriginal();
		}
	}

}
```





### CacheProxyFactoryBean：手动实现Cache功能



不建议这么设置，因为一般都是允许缓存null值的。

@Cacheable注解sync=true的效果
在多线程环境下，某些操作可能使用相同参数同步调用（相同的key）。默认情况下，缓存不锁定任何资源，可能导致多次计算，而违反了缓存的目的。对于这些特定的情况，属性 sync 可以指示底层将缓存锁住，使只有一个线程可以进入计算，而其他线程堵塞，直到返回结果更新到缓存中（Spring4.3提供的）



# 扩展缓存注解支持失效时间TTL

`Spring Cache`抽象本省是并**不支持**`Expire`失效时间的设定的

若想在缓存注解上指定失效时间，必须具备如下两个基本条件：

* 缓存实现产品支持Expire失效时间（Ehcache、Redis等几乎所有第三方实现都支持）
* xxxCacheManager管理的xxxCache必须扩展了Expire的实现

因为缓存的k-v键值对具有自动失效的特性实在太重要和太实用了，所以虽然org.springframework.cache.Cache它没有实现Expire，但好在第三方产品对Spring缓存标准实现的时候，大都实现了这个重要的失效策略，比如典型例子：RedisCache。



#### 方式一：使用源生的`RedisCacheManager`进行集中式控制

```jav
@EnableCaching // 使用了CacheManager，别忘了开启它  否则无效
@Configuration
public class CacheConfig extends CachingConfigurerSupport {

    @Bean
    public CacheManager cacheManager() {
        RedisCacheConfiguration defaultCacheConfig = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofDays(1)) // 默认没有特殊指定的
                .computePrefixWith(cacheName -> "caching:" + cacheName);

        // 针对不同cacheName，设置不同的过期时间
        Map<String, RedisCacheConfiguration> initialCacheConfiguration = new HashMap<String, RedisCacheConfiguration>() {{
            put("demoCache", RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofHours(1))); //1小时
            put("demoCar", RedisCacheConfiguration.defaultCacheConfig().entryTtl(Duration.ofMinutes(10))); // 10分钟
            // ...
        }};

        RedisCacheManager redisCacheManager = RedisCacheManager.builder(redisConnectionFactory())
                .cacheDefaults(defaultCacheConfig) // 默认配置（强烈建议配置上）。  比如动态创建出来的都会走此默认配置
                .withInitialCacheConfigurations(initialCacheConfiguration) // 不同cache的个性化配置
                .build();
        return redisCacheManager;
    }

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration configuration = new RedisStandaloneConfiguration();
        configuration.setHostName("10.102.132.150");
        configuration.setPort(6379);
        configuration.setDatabase(0);
        LettuceConnectionFactory factory = new LettuceConnectionFactory(configuration);
        return factory;
    }

    @Bean
    public RedisTemplate<String, String> stringRedisTemplate() {
        RedisTemplate<String, String> redisTemplate = new StringRedisTemplate();
        redisTemplate.setConnectionFactory(redisConnectionFactory());
        return redisTemplate;
    }

}
```

> 即使禁用前缀disableKeyPrefix()，也是不会影响对应CacheName的TTL（因为TTL针对的是Cache，而不是key）
> 每个CacheName都可以对应一个RedisCacheConfiguration（它里面有众多属性都可以个性化），若没配置的（比如动态生成的）都走默认配置
> Spring提供的在`RedisCacheManager`来统一管理Cache的TTL，不让他分散在各个缓存注解上



#### 方式二：自定义cacheNames方式

> @Cacheable(cacheNames = "demoCache#3600", key = "#id + 0")

```java
public class MyRedisCacheManager extends RedisCacheManager {

    public MyRedisCacheManager(RedisCacheWriter cacheWriter, RedisCacheConfiguration defaultCacheConfiguration) {
        super(cacheWriter, defaultCacheConfiguration);
    }

    @Override
    protected RedisCache createRedisCache(String name, RedisCacheConfiguration cacheConfig) {
        String[] array = StringUtils.delimitedListToStringArray(name, "#");
        name = array[0];
        if (array.length > 1) { // 解析TTL
            long ttl = Long.parseLong(array[1]);
            cacheConfig = cacheConfig.entryTtl(Duration.ofSeconds(ttl)); // 注意单位我此处用的是秒，而非毫秒
        }
        return super.createRedisCache(name, cacheConfig);
    }

}
```































































