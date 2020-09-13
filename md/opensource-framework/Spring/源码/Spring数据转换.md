## Converter<S, T>

是Spring中最为简单的一个接口。位于包：org.springframework.core.convert.converter。 相关的顶层接口（类）有：ConditionalConverter、GenericConverter、ConverterFactory、ConvertingComparator、ConverterRegistry

```java
// 实现此接口的 大都会实现ConditionalConverter
// 请保持线程安全~~
@FunctionalInterface
public interface Converter<S, T> {
	// 把S转成T
	@Nullable
	T convert(S source);
}

```

### Converter implement

```java
// StringToCharsetConverter  @since 4.2
	@Override
	public Charset convert(String source) {
		return Charset.forName(source);
	}
// StringToPropertiesConverter
	@Override
	public Properties convert(String source) {
		try {
			Properties props = new Properties();
			// Must use the ISO-8859-1 encoding because Properties.load(stream) expects it.
			props.load(new ByteArrayInputStream(source.getBytes(StandardCharsets.ISO_8859_1)));
			return props;
		}catch (Exception ex) {
			// Should never happen.
			throw new IllegalArgumentException("Failed to parse [" + source + "] into Properties", ex);
		}
	}
// StringToTimeZoneConverter @since 4.2
	@Override
	public TimeZone convert(String source) {
		return StringUtils.parseTimeZoneString(source);
	}
//ZoneIdToTimeZoneConverter @since 4.0
	@Override
	public TimeZone convert(ZoneId source) {
		return TimeZone.getTimeZone(source);
	}

// StringToBooleanConverter  这个转换器很有意思  哪些代表true，哪些代表fasle算是业界的一个规范了
// 这就是为什么，我们给传值1也会被当作true来封装进Boolean类型的根本原因所在~
	static {
		trueValues.add("true");
		trueValues.add("on");
		trueValues.add("yes");
		trueValues.add("1");

		falseValues.add("false");
		falseValues.add("off");
		falseValues.add("no");
		falseValues.add("0");
	}
// StringToUUIDConverter  @since 3.2
	@Override
	public UUID convert(String source) {
		return (StringUtils.hasLength(source) ? UUID.fromString(source.trim()) : null);
	}
// StringToLocaleConverter
	@Override
	@Nullable
	public Locale convert(String source) {
		return StringUtils.parseLocale(source);
	}

// SerializingConverter：把任意一个对象，转换成byte[]数组，唯独这一个是public的，其它的都是Spring内置的
public class SerializingConverter implements Converter<Object, byte[]> {
	// 序列化器：DefaultSerializer   就是new ObjectOutputStream(outputStream).writeObject(object)
	// 就是简单的把对象写到输出流里~~
	private final Serializer<Object> serializer;
	public SerializingConverter() {
		this.serializer = new DefaultSerializer();
	}
	public SerializingConverter(Serializer<Object> serializer) { // 自己亦可指定实现。
		Assert.notNull(serializer, "Serializer must not be null");
		this.serializer = serializer;
	}

	@Override
	public byte[] convert(Object source) {
		ByteArrayOutputStream byteStream = new ByteArrayOutputStream(1024);
		try  {
			this.serializer.serialize(source, byteStream);
			// 把此输出流转为byte[]数组~~~~~~
			return byteStream.toByteArray();
		} catch (Throwable ex) {
			throw new SerializationFailedException("Failed to serialize object using " +
					this.serializer.getClass().getSimpleName(), ex);
		}
	}

}
```

### `ConditionalConverter`

```java
// @since 3.2   出现稍微较晚
public interface ConditionalConverter {
	boolean matches(TypeDescriptor sourceType, TypeDescriptor targetType);
}
```

> org.springframework.core.convert.TypeDescriptor也是一个Spring的基础类（类似ResolvableType）这种，若有需要我们平时也可以使用它。 它能够把基础类型、MethodParameter、Field、org.springframework.core.convert.Property、Class等都描述进来。

#### TypeDescriptor 

```java
// @since 3.0
public class TypeDescriptor implements Serializable {
	public Class<?> getType() {
		return this.type;
	}
	public ResolvableType getResolvableType() {
		return this.resolvableType;
	}
	public Object getSource() {
		return this.resolvableType.getSource();
	}
	public String getName();
	public boolean isPrimitive();
	public Annotation[] getAnnotations();
	public boolean hasAnnotation(Class<? extends Annotation> annotationType);
	public <T extends Annotation> T getAnnotation(Class<T> annotationType);
	public boolean isAssignableTo(TypeDescriptor typeDescriptor);
	public boolean isCollection();
	public boolean isArray();
	public boolean isMap();
	public TypeDescriptor getMapKeyTypeDescriptor();
	public TypeDescriptor getMapValueTypeDescriptor()

	// 静态方法：可吧基础类型、任意一个class类型转为这个描述类型  依赖于下面的valueOf方法  source为null  返回null
	public static TypeDescriptor forObject(@Nullable Object source);
	public static TypeDescriptor valueOf(@Nullable Class<?> type);
	// 把集合转为描述类型~
	public static TypeDescriptor collection(Class<?> collectionType, @Nullable TypeDescriptor elementTypeDescriptor)
	public static TypeDescriptor map(Class<?> mapType, @Nullable TypeDescriptor keyTypeDescriptor, @Nullable TypeDescriptor valueTypeDescriptor);
	public static TypeDescriptor array(@Nullable TypeDescriptor elementTypeDescriptor);
	public static TypeDescriptor nested(MethodParameter methodParameter, int nestingLevel);
	public static TypeDescriptor nested(Field field, int nestingLevel);
	public static TypeDescriptor nested(Property property, int nestingLevel);
}
```



## `ConverterFactory`

range范围转换器的工厂：可以将对象从S转换为R的子类型（1:N）

```java
public interface ConverterFactory<S, R> {
	//Get the converter to convert from S to target type T, where T is also an instance of R
	<T extends R> Converter<S, T> getConverter(Class<T> targetType);
}
```



### `GenericConverter`

用于在两个或多个类型之间转换的通用转换器接口。**这是最灵活的转换器SPI接口，也是最复杂的**
灵活是因为它一个转换器就能转换多个s/t，**所以它是N->N的**。实现类们一般情况下也会实现接口：`ConditionalConverter`

```java
public interface GenericConverter {
	@Nullable
	Set<ConvertiblePair> getConvertibleTypes();
	@Nullable
	Object convert(@Nullable Object source, TypeDescriptor sourceType, TypeDescriptor targetType);

	/**
	 * Holder for a source-to-target class pair.
	 */
	 // 包含有一对  s和t
	final class ConvertiblePair {
		private final Class<?> sourceType;
		private final Class<?> targetType;

		public ConvertiblePair(Class<?> sourceType, Class<?> targetType) {
			Assert.notNull(sourceType, "Source type must not be null");
			Assert.notNull(targetType, "Target type must not be null");
			this.sourceType = sourceType;
			this.targetType = targetType;
		}
		... // 去掉get/set方法  以及toString equals等基础方法
	}
}
```



## `ConverterRegistry`



## ConversionService

用于类型转换的服务接口。这是进入转换系统的入口点。位于包：org.springframework.core.convert。相关的顶层接口（类）有：ConversionService、FormattingConversionService、DefaultConversionService、ConversionServiceFactoryBean、FormattingConversionServiceFactoryBean

用于类型转换的服务接口。这是转换系统的`入口点`。

```java
// @since 3.0
public interface ConversionService {
	
	// 特别说明：若是Map、集合、数组转换时。即使下面方法convert转换抛出了异常，这里也得返回true  因为Spring希望调用者处理这个异常：ConversionException
	boolean canConvert(@Nullable Class<?> sourceType, Class<?> targetType);
	boolean canConvert(@Nullable TypeDescriptor sourceType, TypeDescriptor targetType);

	// 注意此处：转换的source都是对象，target只需要类型即可~~~
	@Nullable
	<T> T convert(@Nullable Object source, Class<T> targetType);
	@Nullable
	Object convert(@Nullable Object source, @Nullable TypeDescriptor sourceType, TypeDescriptor targetType);

}
```

### `GenericConversionService`

```java
// @since 3.0  实现了接口ConversionService和ConverterRegistry  
public class GenericConversionService implements ConfigurableConversionService {
	
	// 啥都不做，但是呢conversion is not required，相当于占位的意思
	private static final GenericConverter NO_OP_CONVERTER = new NoOpConverter("NO_OP");
	// 当转换器缓存中没有任何匹配时，它上场
	// 请不要把它直接return，用null代替返回
	private static final GenericConverter NO_MATCH = new NoOpConverter("NO_MATCH");

	// 说明：Converter是一个静态内部类 它会Manages all converters registered with the service
	private final Converters converters = new Converters();
	// 缓存转换器。用的ConcurrentReferenceHashMap是Spring自己实现的一个软引用/弱引用的Map
	private final Map<ConverterCacheKey, GenericConverter> converterCache = new ConcurrentReferenceHashMap<>(64);
	
	// 仅有一个空构造函数,构造函数内啥都没做

	@Override
	public void addConverter(Converter<?, ?> converter) {
		// 这个处理很有意思：getRequiredTypeInfo    拿到两个泛型参数类型（若没有指定泛型  返回的是null）
		ResolvableType[] typeInfo = getRequiredTypeInfo(converter.getClass(), Converter.class);
		// Decorate和Proxy模式的区别。Decorate模式可用于函数防抖   Proxy模式就是我们常用的代理模式
		if (typeInfo == null && converter instanceof DecoratingProxy) {
			typeInfo = getRequiredTypeInfo(((DecoratingProxy) converter).getDecoratedClass(), Converter.class);
		}
		// 由此可见这个转换器的泛型类型是必须的~~~
		if (typeInfo == null) {
			throw new IllegalArgumentException("Unable to determine source type <S> and target type <T> for your " +
					"Converter [" + converter.getClass().getName() + "]; does the class parameterize those types?");
		}
	
		// ConverterAdapter是个GenericConverter。由此课件最终都是转换成了GenericConverter类型
		addConverter(new ConverterAdapter(converter, typeInfo[0], typeInfo[1]));
	}
	@Override
	public <S, T> void addConverter(Class<S> sourceType, Class<T> targetType, Converter<? super S, ? extends T> converter) {
		addConverter(new ConverterAdapter(converter, ResolvableType.forClass(sourceType), ResolvableType.forClass(targetType)));
	}
	// 最终都是转换成了GenericConverter  进行转换器的保存  全部放在Converters里保存着
	@Override
	public void addConverter(GenericConverter converter) {
		this.converters.add(converter);
		invalidateCache(); // 清空缓存
	}

	// 使用ConverterFactoryAdapter转换成GenericConverter
	@Override
	public void addConverterFactory(ConverterFactory<?, ?> factory) { ... }
	// 注意ConvertiblePair是重写了equals方法和hash方法的
	@Override
	public void removeConvertible(Class<?> sourceType, Class<?> targetType) {
		this.converters.remove(sourceType, targetType);
		invalidateCache();
	}

	// 主要是getConverter() 方法  相当于只有有转换器匹配，就是能够被转换的
	@Override
	public boolean canConvert(@Nullable TypeDescriptor sourceType, TypeDescriptor targetType) {
		Assert.notNull(targetType, "Target type to convert to cannot be null");
		if (sourceType == null) {
			return true;
		}
		GenericConverter converter = getConverter(sourceType, targetType);
		return (converter != null);
	}
	@Nullable
	protected GenericConverter getConverter(TypeDescriptor sourceType, TypeDescriptor targetType) {
		ConverterCacheKey key = new ConverterCacheKey(sourceType, targetType);
		GenericConverter converter = this.converterCache.get(key);
		// 这个处理：如果缓存有值   但是为NO_MATCH 那就返回null，而不是把No_Match直接return
		if (converter != null) {
			return (converter != NO_MATCH ? converter : null);
		}

		converter = this.converters.find(sourceType, targetType);
		if (converter == null) {
			converter = getDefaultConverter(sourceType, targetType);
		}
	
		// 如果默认的不为null 也可以return的
		// NO_OP_CONVERTER还是可以return的~~~
		if (converter != null) {
			this.converterCache.put(key, converter);
			return converter;
		}

		this.converterCache.put(key, NO_MATCH);
		return null;
	}
	@Nullable
	protected GenericConverter getDefaultConverter(TypeDescriptor sourceType, TypeDescriptor targetType) {
		return (sourceType.isAssignableTo(targetType) ? NO_OP_CONVERTER : null);
	}
	// 拿到泛型类型们
	@Nullable
	private ResolvableType[] getRequiredTypeInfo(Class<?> converterClass, Class<?> genericIfc) {
		ResolvableType resolvableType = ResolvableType.forClass(converterClass).as(genericIfc);
		ResolvableType[] generics = resolvableType.getGenerics();
		if (generics.length < 2) {
			return null;
		}
		Class<?> sourceType = generics[0].resolve();
		Class<?> targetType = generics[1].resolve();
		if (sourceType == null || targetType == null) {
			return null;
		}
		return generics;
	}
	...
}
```



#### `DefaultConversionService`

适用于大多数的场景

```java
// @since 3.1
public class DefaultConversionService extends GenericConversionService {
	// @since 4.3.5 改变量出现得还是比较晚的
	@Nullable
	private static volatile DefaultConversionService sharedInstance;
	// 空构造，那就注册到自己this身上~~~因为自己也是个ConverterRegistry
	public DefaultConversionService() {
		addDefaultConverters(this);
	}

	// 就是把sharedInstance返回出去~~~（永远不可能返回null）
	public static ConversionService getSharedInstance() { ... }

	// 默认情况下，这个ConversionService注册的转换器们~~~~  几乎涵盖了所有~~~~
	public static void addDefaultConverters(ConverterRegistry converterRegistry) {
		addScalarConverters(converterRegistry);
		addCollectionConverters(converterRegistry);

		converterRegistry.addConverter(new ByteBufferConverter((ConversionService) converterRegistry));
		converterRegistry.addConverter(new StringToTimeZoneConverter());
		converterRegistry.addConverter(new ZoneIdToTimeZoneConverter());
		converterRegistry.addConverter(new ZonedDateTimeToCalendarConverter());

		converterRegistry.addConverter(new ObjectToObjectConverter());
		converterRegistry.addConverter(new IdToEntityConverter((ConversionService) converterRegistry));
		converterRegistry.addConverter(new FallbackObjectToStringConverter());
		converterRegistry.addConverter(new ObjectToOptionalConverter((ConversionService) converterRegistry));
	}
	...
}
```

#### ConversionServiceFactoryBean

```java
public class ConversionServiceFactoryBean implements FactoryBean<ConversionService>, InitializingBean {
	// 保存着我们diy set捡来的转换器们
	@Nullable
	private Set<?> converters;
	// 最终是一个DefaultConversionService，然后向里添加自定义的转换器~
	@Nullable
	private GenericConversionService conversionService;
	
	// Bean初始化结束后，注册自定义的转换器进去~~
	@Override
	public void afterPropertiesSet() {
		this.conversionService = createConversionService();
		ConversionServiceFactory.registerConverters(this.converters, this.conversionService);
	}
	protected GenericConversionService createConversionService() {
		return new DefaultConversionService();
	}

	@Override
	@Nullable
	public ConversionService getObject() {
		return this.conversionService;
	}
	// 最终是个GenericConversionService,实际是个DefaultConversionService
	@Override
	public Class<? extends ConversionService> getObjectType() {
		return GenericConversionService.class;
	}
	@Override
	public boolean isSingleton() {
		return true;
	}

}
```



## PropertyEditor



## Converter or PropertyEditor？
Spring有两种自动类型转换器，一种是Converter,一种是PropertyEditor。

Converter是类型转换成类型，Editor:从string类型转换为其他类型。
从某种程度上，Converter包含Editor。如果出现需要从string转换到其他类型。首选Editor。



## `org.springframework.beans.TypeConverter`

```java
// @since 2.0
// 定义类型转换方法的接口。通常（但不一定）与PropertyEditorRegistry接口一起实现
// 通常接口TypeConverter的实现是基于非线程安全的PropertyEditors类，因此也不是线程安全的
public interface TypeConverter {

	// 将参数中的value转换成requiredType类型
	// 从String到任何类型的转换通常使用PropertyEditor类的setAsText方法或ConversionService中的Spring Converter
	@Nullable
	<T> T convertIfNecessary(@Nullable Object value, @Nullable Class<T> requiredType) throws TypeMismatchException;

	// 意义同上，增加了作为转换目标的方法参数，主要用于分析泛型类型，可能是null
	@Nullable
	<T> T convertIfNecessary(@Nullable Object value, @Nullable Class<T> requiredType, @Nullable MethodParameter methodParam) throws TypeMismatchException;
	// 意义同上，增加了转换目标的反射field
	@Nullable
	<T> T convertIfNecessary(@Nullable Object value, @Nullable Class<T> requiredType, @Nullable Field field) throws TypeMismatchException;

	// @since 5.1.4
	@Nullable
	default <T> T convertIfNecessary(@Nullable Object value, @Nullable Class<T> requiredType,
			@Nullable TypeDescriptor typeDescriptor) throws TypeMismatchException {

		throw new UnsupportedOperationException("TypeDescriptor resolution not supported");
	}

}
```

#### TypeConverterSupport

#### TypeConverterDelegate



PropertyEditor用于字符串到其它对象的转换，由于其局限性，spring提供了converter接口，由ConversionService来调用对外提供服务，而TypeConverter综合了上述两种转换方式，交由TypeConverterDelegate来进行转换。

TypeConverterDelegater先使用PropertyEditor转换器器转换，如果没找到对应的转换器器，会⽤ConversionService来进行对象转换















































