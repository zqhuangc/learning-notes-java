Java提供了`System`类的**静态方法**`getenv()`和`getProperty()`用于返回系统相关的`环境变量`与`系统属性`

- `getenv`方法返回的变量大多与**操作系统**相关
- `getProperty`方法返回的变量大多与**java程序**有关

#### 系统属性Property

环境属性：

```java
USERPROFILE        ：用户目录
USERDNSDOMAIN      ：用户域
PATHEXT            ：可执行后缀
JAVA_HOME          ：Java安装目录
TEMP               ：用户临时文件目录
SystemDrive        ：系统盘符
ProgramFiles       ：默认程序目录
USERDOMAIN         ：帐户的域的名称
ALLUSERSPROFILE    ：用户公共目录
SESSIONNAME        ：Session名称
TMP                ：临时目录
Path               ：path环境变量
CLASSPATH          ：classpath环境变量
PROCESSOR_ARCHITECTURE ：处理器体系结构
OS                     ：操作系统类型
PROCESSOR_LEVEL    ：处理级别
COMPUTERNAME       ：计算机名
Windir             ：系统安装目录
SystemRoot         ：系统启动目录
USERNAME           ：用户名
ComSpec            ：命令行解释器可执行程序的准确路径
APPDATA            ：应用程序数据目录
```

**系统属性：**

```java
java.version Java ：运行时环境版本
java.vendor Java ：运行时环境供应商
java.vendor.url ：Java供应商的 URL
java.home &nbsp;&nbsp;：Java安装目录
java.vm.specification.version： Java虚拟机规范版本
java.vm.specification.vendor ：Java虚拟机规范供应商
java.vm.specification.name &nbsp; ：Java虚拟机规范名称
java.vm.version ：Java虚拟机实现版本
java.vm.vendor ：Java虚拟机实现供应商
java.vm.name&nbsp; ：Java虚拟机实现名称
java.specification.version：Java运行时环境规范版本
java.specification.vendor：Java运行时环境规范供应商
java.specification.name ：Java运行时环境规范名称
java.class.version ：Java类格式版本号
java.class.path ：Java类路径
java.library.path  ：加载库时搜索的路径列表
java.io.tmpdir  ：默认的临时文件路径
java.compiler  ：要使用的 JIT编译器的名称
java.ext.dirs ：一个或多个扩展目录的路径
os.name ：操作系统的名称
os.arch  ：操作系统的架构
os.version  ：操作系统的版本
file.separator ：文件分隔符
path.separator ：路径分隔符
line.separator ：行分隔符
user.name ：用户的账户名称
user.home ：用户的主目录
user.dir：用户的当前工作目录
```





## Spring属性管理API

1. PropertySource：属性源。key-value属性对抽象
2. PropertyResolver：属性解析器。用于解析相应key的value
3. Profile：配置(资料里翻译为剖面，我实在不理解)。只有激活的配置profile的组件/配置才会注册到Spring容器，类似于maven中profile
4. Environment：环境，本身也是个属性解析器PropertyResolver。它在基础上还提供了Profile特性，能够很好的对多环境支持。因此我们一般使用它，而不是底层接口PropertyResolver。 可以简单粗暴的把它理解为Profile 和 PropertyResolver 的组合



ConfigurablePropertyResolver

AbstractPropertyResolver

Spring内建**唯一实现类**为：`PropertySourcesPropertyResolver`

```
public class PropertySourcesPropertyResolver extends AbstractPropertyResolver {
	...
	@Nullable
	private final PropertySources propertySources; //内部持有一组PropertySource

	// 由此可以看出propertySources的顺序很重要~~~
	// 并且还能处理占位符 resolveNestedPlaceholders支持内嵌、嵌套占位符
	@Nullable
	protected <T> T getProperty(String key, Class<T> targetValueType, boolean resolveNestedPlaceholders) {
		if (this.propertySources != null) {
			for (PropertySource<?> propertySource : this.propertySources) {
				Object value = propertySource.getProperty(key);
				if (value != null) {
					if (resolveNestedPlaceholders && value instanceof String) {
						value = resolveNestedPlaceholders((String) value);
					}
					logKeyFound(key, propertySource, value);
					return convertValueIfNecessary(value, targetValueType);
				}
			}
		}
		return null;
	}
	...
}

// @since 3.1
public interface Environment extends PropertyResolver {
	// 返回此环境下激活的配置文件集
	String[] getActiveProfiles();
	// 如果未设置激活配置文件，则返回默认的激活的配置文件集
	String[] getDefaultProfiles();

	// @since 5.1
	@Deprecated
	boolean acceptsProfiles(String... profiles);
	boolean acceptsProfiles(Profiles profiles);
}

//委托实现
public interface ConfigurableEnvironment extends Environment, ConfigurablePropertyResolver {
    // 指定该环境下的 profile 集
    void setActiveProfiles(String... profiles);
    // 增加此环境的 profile
    void addActiveProfile(String profile);
    // 设置默认的 profile
    void setDefaultProfiles(String... profiles);
    // 返回此环境的 PropertySources
    MutablePropertySources getPropertySources();
   // 尝试返回 System.getenv() 的值，若失败则返回通过 System.getenv(string) 的来访问各个键的映射
    Map<String, Object> getSystemEnvironment();
    // 尝试返回 System.getProperties() 的值，若失败则返回通过 System.getProperties(string) 的来访问各个键的映射
    Map<String, Object> getSystemProperties();
    void merge(ConfigurableEnvironment parent);
}

```



#### @Profile注解和`ProfileCondition`

```java
class ProfileCondition implements Condition {

	@Override
	public boolean matches(ConditionContext context, AnnotatedTypeMetadata metadata) {
		// 因为value值是个数组，所以此处有多个值 用的MultiValueMap
		MultiValueMap<String, Object> attrs = metadata.getAllAnnotationAttributes(Profile.class.getName());
		if (attrs != null) {
			for (Object value : attrs.get("value")) {
		
				// 多个值中，但凡只要有一个acceptsProfiles了，那就返回true~
				if (context.getEnvironment().acceptsProfiles(Profiles.of((String[]) value))) {
					return true;
				}
			}
			return false;
		}
		return true;
	}

}
```



# PropertyResolver

`org.springframework.core.env.PropertyResolver`此接口用于在**底层源**之上解析**一系列**的属性值,接口中定义了一系列读取，解析，判断是否包含指定属性的方法：

```java
// @since 3.1  出现得还是相对较晚的  毕竟SpEL也3.0之后才出来嘛~~~
public interface PropertyResolver {

	// 查看规定指定的key是否有对应的value   注意：若对应值是null的话 也是返回false
	boolean containsProperty(String key);
	// 如果没有则返回null
	@Nullable
	String getProperty(String key);
	// 如果没有则返回defaultValue
	String getProperty(String key, String defaultValue);
	// 返回指定key对应的value，会解析成指定类型。如果没有对应值则返回null（而不是抛错~）
	@Nullable
	<T> T getProperty(String key, Class<T> targetType);
	<T> T getProperty(String key, Class<T> targetType, T defaultValue);

	// 若不存在就不是返回null了  而是抛出异常~  所以不用担心返回值是null
	String getRequiredProperty(String key) throws IllegalStateException;
	<T> T getRequiredProperty(String key, Class<T> targetType) throws IllegalStateException;

	// 解析${...}这种类型的占位符，把他们替换为使用getProperty方法返回的结果，解析不了并且没有默认值的占位符会被忽略（原样输出）
	String resolvePlaceholders(String text);
	// 解析不了就抛出异常~
	String resolveRequiredPlaceholders(String text) throws IllegalArgumentException;

}
```



## ConfigurablePropertyResolver

扩展定义类型转换、属性校验、前缀、后缀、`分隔符`等

```java
public interface ConfigurablePropertyResolver extends PropertyResolver {

	// 返回在解析属性时使用的ConfigurableConversionService。此方法的返回值可被用户定制化set
	// 例如可以移除或者添加Converter  cs.addConverter(new FooConverter());等等
	ConfigurableConversionService getConversionService();
	// 全部替换ConfigurableConversionService的操作(不常用)  一般还是get出来操作它内部的东东
	void setConversionService(ConfigurableConversionService conversionService);

	// 设置占位符的前缀  后缀    默认是${}
	void setPlaceholderPrefix(String placeholderPrefix);
	void setPlaceholderSuffix(String placeholderSuffix);
	// 默认值的分隔符   默认为冒号:
	void setValueSeparator(@Nullable String valueSeparator);
	// 是否忽略解析不了的占位符，默认是false  表示不忽略~~~（解析不了就抛出异常）
	void setIgnoreUnresolvableNestedPlaceholders(boolean ignoreUnresolvableNestedPlaceholders);

	/**
	 * Specify which properties must be present, to be verified by
	 * {@link #validateRequiredProperties()}.
	 */
	void setRequiredProperties(String... requiredProperties);
	void validateRequiredProperties() throws MissingRequiredPropertiesException;
}
```



### AbstractPropertyResolver 

```java
// @since 3.1
public abstract class AbstractPropertyResolver implements ConfigurablePropertyResolver {
	@Nullable
	private volatile ConfigurableConversionService conversionService;

	// PropertyPlaceholderHelper是一个极其独立的类，专门用来解析占位符  我们自己项目中可议拿来使用  因为它不依赖于任何其他类
	@Nullable
	private PropertyPlaceholderHelper nonStrictHelper;
	@Nullable
	private PropertyPlaceholderHelper strictHelper;
	
	private boolean ignoreUnresolvableNestedPlaceholders = false;
	private String placeholderPrefix = SystemPropertyUtils.PLACEHOLDER_PREFIX;
	private String placeholderSuffix = SystemPropertyUtils.PLACEHOLDER_SUFFIX;
	@Nullable
	private String valueSeparator = SystemPropertyUtils.VALUE_SEPARATOR;
	private final Set<String> requiredProperties = new LinkedHashSet<>();

	// 默认值使用的DefaultConversionService
	@Override
	public ConfigurableConversionService getConversionService() {
		// Need to provide an independent DefaultConversionService, not the
		// shared DefaultConversionService used by PropertySourcesPropertyResolver.
		ConfigurableConversionService cs = this.conversionService;
		if (cs == null) {
			synchronized (this) {
				cs = this.conversionService;
				if (cs == null) {
					cs = new DefaultConversionService();
					this.conversionService = cs;
				}
			}
		}
		return cs;
	}
	... // 省略get/set
	@Override
	public void setRequiredProperties(String... requiredProperties) {
		for (String key : requiredProperties) {
			this.requiredProperties.add(key);
		}
	}
	// 校验这些key~
	@Override
	public void validateRequiredProperties() {
		MissingRequiredPropertiesException ex = new MissingRequiredPropertiesException();
		for (String key : this.requiredProperties) {
			if (this.getProperty(key) == null) {
				ex.addMissingRequiredProperty(key);
			}
		}
		if (!ex.getMissingRequiredProperties().isEmpty()) {
			throw ex;
		}
	}
	... //get/set property等方法省略  直接看处理占位符的方法即可
	@Override
	public String resolvePlaceholders(String text) {
		if (this.nonStrictHelper == null) {
			this.nonStrictHelper = createPlaceholderHelper(true);
		}
		return doResolvePlaceholders(text, this.nonStrictHelper);
	}
	private PropertyPlaceholderHelper createPlaceholderHelper(boolean ignoreUnresolvablePlaceholders) {
		return new PropertyPlaceholderHelper(this.placeholderPrefix, this.placeholderSuffix, this.valueSeparator, ignoreUnresolvablePlaceholders);
	}
	// 此处：最终都是委托给PropertyPlaceholderHelper去做  而getPropertyAsRawString是抽象方法  根据key返回一个字符串即可~
	private String doResolvePlaceholders(String text, PropertyPlaceholderHelper helper) {
		return helper.replacePlaceholders(text, this::getPropertyAsRawString);
	}
}
```



### `PropertyPlaceholderHelper`

将字符串里的占位符内容，用我们配置的properties里的替换。

```java
// @since 3.0  Utility class for working with Strings that have placeholder values in them
public class PropertyPlaceholderHelper {

	// 这里保存着  通用的熟悉的 开闭的符号们~~~
	private static final Map<String, String> wellKnownSimplePrefixes = new HashMap<>(4);
	static {
		wellKnownSimplePrefixes.put("}", "{");
		wellKnownSimplePrefixes.put("]", "[");
		wellKnownSimplePrefixes.put(")", "(");
	}

	private final String placeholderPrefix;
	private final String placeholderSuffix;
	private final String simplePrefix;
	@Nullable
	private final String valueSeparator;
	private final boolean ignoreUnresolvablePlaceholders; // 是否采用严格模式~~

	// 从properties里取值  若你有就直接从Properties里取值了~~~
	public String replacePlaceholders(String value, final Properties properties)  {
		Assert.notNull(properties, "'properties' must not be null");
		return replacePlaceholders(value, properties::getProperty);
	}

	// @since 4.3.5   抽象类提供这个类型转换的方法~ 需要类型转换的会调用它
	// 显然它是委托给了ConversionService，而这个类在前面文章已经都重点分析过了~
	@Nullable
	protected <T> T convertValueIfNecessary(Object value, @Nullable Class<T> targetType) {
		if (targetType == null) {
			return (T) value;
		}
		ConversionService conversionServiceToUse = this.conversionService;
		if (conversionServiceToUse == null) {
			// Avoid initialization of shared DefaultConversionService if
			// no standard type conversion is needed in the first place...
			if (ClassUtils.isAssignableValue(targetType, value)) {
				return (T) value;
			}
			conversionServiceToUse = DefaultConversionService.getSharedInstance();
		}
		return conversionServiceToUse.convert(value, targetType);
	}

	// 这里会使用递归，根据传入的符号，默认值等来处理~~~~
	protected String parseStringValue(String value, PlaceholderResolver placeholderResolver, Set<String> visitedPlaceholders) { ... }
}
```



#### PropertySourcesPropertyResolver

**主要是负责提供数据源。**

```java
// @since 3.1    PropertySource：就是我们所说的数据源，它是Spring一个非常重要的概念，比如可以来自Map，来自命令行、来自自定义等等~~~
public class PropertySourcesPropertyResolver extends AbstractPropertyResolver {

	// 数据源们~
	@Nullable
	private final PropertySources propertySources;

	// 唯一构造函数：必须制定数据源~
	public PropertySourcesPropertyResolver(@Nullable PropertySources propertySources) {
		this.propertySources = propertySources;
	}

	@Override
	public boolean containsProperty(String key) {
		if (this.propertySources != null) {
			for (PropertySource<?> propertySource : this.propertySources) {
				if (propertySource.containsProperty(key)) {
					return true;
				}
			}
		}
		return false;
	}
	...
	//最终依赖的都是propertySource.getProperty(key);方法拿到如果是字符串的话
	//就继续交给 value = resolveNestedPlaceholders((String) value);处理
	@Nullable
	protected <T> T getProperty(String key, Class<T> targetValueType, boolean resolveNestedPlaceholders) {
		if (this.propertySources != null) {
			for (PropertySource<?> propertySource : this.propertySources) {
				Object value = propertySource.getProperty(key);
				if (value != null) {
					// 若值是字符串，那就处理一下占位符~~~~~~  所以我们看到所有的PropertySource都是支持占位符的
					if (resolveNestedPlaceholders && value instanceof String) {
						value = resolveNestedPlaceholders((String) value);
					}
					logKeyFound(key, propertySource, value);
					return convertValueIfNecessary(value, targetValueType);
				}
			}
		}
		return null;
	}
	...
}
```



# Environment

这个接口代表了当前应用正在运行的环境，为应用的两个重要方面建立抽象模型 【`profiles`】和【`properties`】。关于属性访问的方法通过父接口`PropertyResolver`暴露给客户端使用，本接口主要是扩展出访问【`profiles`】相关的接口。

profiles：配置。它代表应用在一启动时注册到context中bean definitions的命名的逻辑分组。
properties：属性。几乎在所有应用中都扮演着重要角色，他可能源自多种源头。例如属性文件，JVM系统属性，系统环境变量，JNDI，servlet上下文参数，Map等等，Environment对象和其相关的对象一起提供给用户一个方便用来配置和解析属性的服务。

```java
// @since 3.1   可见Spring3.x版本是Spirng一次极其重要的跨越、升级
// 它继承自PropertyResolver，所以是对属性的一个扩展~
public interface Environment extends PropertyResolver {
	
	// 就算被激活  也是支持同时激活多个profiles的~
	// 设置的key是：spring.profiles.active
	String[] getActiveProfiles();
	// 默认的也可以有多个  key为：spring.profiles.default
	String[] getDefaultProfiles();

	// 看看传入的profiles是否是激活的~~~~  支持!表示不激活
	@Deprecated
	boolean acceptsProfiles(String... profiles);
	// Spring5.1后提供的  用于替代上面方法   Profiles是Spring5.1才有的一个函数式接口~
	boolean acceptsProfiles(Profiles profiles);

}
```



## ConfigurableEnvironment

扩展出了`修改`和配置profiles的一系列方法，包括用户自定义的和系统相关的属性。

```java
// @since 3.1
public interface ConfigurableEnvironment extends Environment, ConfigurablePropertyResolver {
	void setActiveProfiles(String... profiles);
	void addActiveProfile(String profile);
	void setDefaultProfiles(String... profiles);

	// 获取到所有的属性源~  	MutablePropertySources表示可变的属性源们~~~ 它是一个聚合的  持有List<PropertySource<?>>
	// 这样获取出来后，我们可以add或者remove我们自己自定义的属性源了~
	MutablePropertySources getPropertySources();

	// 这里两个哥们应该非常熟悉了吧~~~
	Map<String, Object> getSystemProperties();
	Map<String, Object> getSystemEnvironment();

	// 合并两个环境配置信息~  此方法唯一实现在AbstractEnvironment上
	void merge(ConfigurableEnvironment parent);
}
```



### `AbstractEnvironment`

该抽象类完成了对active、default等相关方法的复写处理。它内部持有一个`MutablePropertySources`引用来管理属性源。

```java
public abstract class AbstractEnvironment implements ConfigurableEnvironment {

	public static final String IGNORE_GETENV_PROPERTY_NAME = "spring.getenv.ignore";
	public static final String ACTIVE_PROFILES_PROPERTY_NAME = "spring.profiles.active";
	public static final String DEFAULT_PROFILES_PROPERTY_NAME = "spring.profiles.default";

	// 保留的默认的profile值   protected final属性，证明子类可以访问
	protected static final String RESERVED_DEFAULT_PROFILE_NAME = "default";


	private final Set<String> activeProfiles = new LinkedHashSet<>();
	// 显然这个里面的值 就是default这个profile了~~~~
	private final Set<String> defaultProfiles = new LinkedHashSet<>(getReservedDefaultProfiles());

	// 这个很关键，直接new了一个 MutablePropertySources来管理属性源们
	// 并且是用的PropertySourcesPropertyResolver来处理里面可能的占位符~~~~~
	private final MutablePropertySources propertySources = new MutablePropertySources();
	private final ConfigurablePropertyResolver propertyResolver = new PropertySourcesPropertyResolver(this.propertySources);

	// 唯一构造方法  customizePropertySources是空方法，交由子类去实现，对属性源进行定制~ 
	// Spring对属性配置分出这么多曾经，在SpringBoot中有着极其重要的意义~~~~
	public AbstractEnvironment() {
		customizePropertySources(this.propertySources);
	}
	// 该方法，StandardEnvironment实现类是有复写的~
	protected void customizePropertySources(MutablePropertySources propertySources) {
	}

	// 若你想改变默认default这个值，可以复写此方法~~~~
	protected Set<String> getReservedDefaultProfiles() {
		return Collections.singleton(RESERVED_DEFAULT_PROFILE_NAME);
	}
	
	//  下面开始实现接口的方法们~~~~~~~
	@Override
	public String[] getActiveProfiles() {
		return StringUtils.toStringArray(doGetActiveProfiles());
	}
	protected Set<String> doGetActiveProfiles() {
		synchronized (this.activeProfiles) {
			if (this.activeProfiles.isEmpty()) { 
		
				// 若目前是empty的，那就去获取：spring.profiles.active
				String profiles = getProperty(ACTIVE_PROFILES_PROPERTY_NAME);
				if (StringUtils.hasText(profiles)) {
					//支持,分隔表示多个~~~且空格啥的都无所谓
					setActiveProfiles(StringUtils.commaDelimitedListToStringArray(
							StringUtils.trimAllWhitespace(profiles)));
				}
			}
			return this.activeProfiles;
		}
	}


	@Override
	public void setActiveProfiles(String... profiles) {
		synchronized (this.activeProfiles) {
			this.activeProfiles.clear(); // 因为是set方法  所以情况已存在的吧
			for (String profile : profiles) {
				 // 简单的valid，不为空且不以!打头~~~~~~~~
				validateProfile(profile);
				this.activeProfiles.add(profile);
			}
		}
	}

	// default profiles逻辑类似，也是不能以!打头~
	@Override
	@Deprecated
	public boolean acceptsProfiles(String... profiles) {
		for (String profile : profiles) {

			// 此处：如果该profile以!开头，那就截断出来  把后半段拿出来看看   它是否在active行列里~~~ 
			// 此处稍微注意：若!表示一个相反的逻辑~~~~~请注意比如!dev表示若dev是active的，我反倒是不生效的
			if (StringUtils.hasLength(profile) && profile.charAt(0) == '!') {
				if (!isProfileActive(profile.substring(1))) {
					return true;
				}
			} else if (isProfileActive(profile)) {
				return true;
			}
		}
		return false;
	}

	// 采用函数式接口处理  就非常的优雅了~
	@Override
	public boolean acceptsProfiles(Profiles profiles) {
		Assert.notNull(profiles, "Profiles must not be null");
		return profiles.matches(this::isProfileActive);
	}

	// 简答的说要么active包含，要门是default  这个profile就被认为是激活的
	protected boolean isProfileActive(String profile) {
		validateProfile(profile);
		Set<String> currentActiveProfiles = doGetActiveProfiles();
		return (currentActiveProfiles.contains(profile) ||
				(currentActiveProfiles.isEmpty() && doGetDefaultProfiles().contains(profile)));
	}

	@Override
	public MutablePropertySources getPropertySources() {
		return this.propertySources;
	}

	public Map<String, Object> getSystemProperties() {
		return (Map) System.getProperties();
	}
	public Map<String, Object> getSystemEnvironment() {
		// 这个判断为：return SpringProperties.getFlag(IGNORE_GETENV_PROPERTY_NAME);
		// 所以我们是可以通过在`spring.properties`这个配置文件里spring.getenv.ignore=false关掉不暴露环境变量的~~~
		if (suppressGetenvAccess()) {
			return Collections.emptyMap();
		}
		return (Map) System.getenv();
	}

	// Append the given parent environment's active profiles, default profiles and property sources to this (child) environment's respective collections of each.
	// 把父环境的属性合并进来~~~~  
	// 在调用ApplicationContext.setParent方法时，会把父容器的环境合并进来  以保证父容器的属性对子容器都是可见的
	@Override
	public void merge(ConfigurableEnvironment parent) {
		for (PropertySource<?> ps : parent.getPropertySources()) {
			if (!this.propertySources.contains(ps.getName())) {
				this.propertySources.addLast(ps); // 父容器的属性都放在最末尾~~~~
			}
		}
		// 合并active
		String[] parentActiveProfiles = parent.getActiveProfiles();
		if (!ObjectUtils.isEmpty(parentActiveProfiles)) {
			synchronized (this.activeProfiles) {
				for (String profile : parentActiveProfiles) {
					this.activeProfiles.add(profile);
				}
			}
		}
		// 合并default
		String[] parentDefaultProfiles = parent.getDefaultProfiles();
		if (!ObjectUtils.isEmpty(parentDefaultProfiles)) {
			synchronized (this.defaultProfiles) {
				this.defaultProfiles.remove(RESERVED_DEFAULT_PROFILE_NAME);
				for (String profile : parentDefaultProfiles) {
					this.defaultProfiles.add(profile);
				}
			}
		}
	}

	// 其余方法全部委托给内置的propertyResolver属性，因为它就是个`PropertyResolver`
	...
}
```



#### StandardEnvironment

Spring应用在非web容器运行的环境

```java
public class StandardServletEnvironment extends StandardEnvironment implements ConfigurableWebEnvironment {
	public static final String SERVLET_CONTEXT_PROPERTY_SOURCE_NAME = "servletContextInitParams";
	public static final String SERVLET_CONFIG_PROPERTY_SOURCE_NAME = "servletConfigInitParams";
	public static final String JNDI_PROPERTY_SOURCE_NAME = "jndiProperties";

	// 放置三个web相关的配置源~  StubPropertySource是PropertySource的一个public静态内部类~~~
	@Override
	protected void customizePropertySources(MutablePropertySources propertySources) {
		propertySources.addLast(new StubPropertySource(SERVLET_CONFIG_PROPERTY_SOURCE_NAME));
		propertySources.addLast(new StubPropertySource(SERVLET_CONTEXT_PROPERTY_SOURCE_NAME));
		
		// 可以通过spring.properties配置文件里面的spring.jndi.ignore=true关闭对jndi的暴露   默认是开启的
		if (JndiLocatorDelegate.isDefaultJndiEnvironmentAvailable()) {
			propertySources.addLast(new JndiPropertySource(JNDI_PROPERTY_SOURCE_NAME));
		}
		super.customizePropertySources(propertySources);
	}

	// 注册servletContextInitParams和servletConfigInitParams到属性配置源头里
	@Override
	public void initPropertySources(@Nullable ServletContext servletContext, @Nullable ServletConfig servletConfig) {
		WebApplicationContextUtils.initServletPropertySources(getPropertySources(), servletContext, servletConfig);
	}
}
```





## EnvironmentCapable、EnvironmentAware

ClassPathScanningCandidateComponentProvider



# StringValueResolver

```java
// @since 2.5  该接口非常简单，就是个函数式接口~
@FunctionalInterface
public interface StringValueResolver {
	@Nullable
	String resolveStringValue(String strVal);
}
```

## EmbeddedValueResolver

帮助`ConfigurableBeanFactory`处理placeholders占位符的。

```java
// @since 4.3  这个类出现得还是蛮晚的   因为之前都是用内部类的方式实现的~~~~这个实现类是最为强大的  只是SpEL
public class EmbeddedValueResolver implements StringValueResolver {

	// BeanExpressionResolver之前有非常详细的讲解，简直不要太熟悉~  它支持的是SpEL  可以说非常的强大
	// 并且它有BeanExpressionContext就能拿到BeanFactory工厂，就能使用它的`resolveEmbeddedValue`来处理占位符~~~~
	// 双重功能都有了~~~拥有了和@Value一样的能力，非常强大~~~
	private final BeanExpressionContext exprContext;
	@Nullable
	private final BeanExpressionResolver exprResolver;

	public EmbeddedValueResolver(ConfigurableBeanFactory beanFactory) {
		this.exprContext = new BeanExpressionContext(beanFactory, null);
		this.exprResolver = beanFactory.getBeanExpressionResolver();
	}


	@Override
	@Nullable
	public String resolveStringValue(String strVal) {
		
		// 先使用Bean工厂处理占位符resolveEmbeddedValue
		String value = this.exprContext.getBeanFactory().resolveEmbeddedValue(strVal);
		// 再使用el表达式参与计算~~~~
		if (this.exprResolver != null && value != null) {
			Object evaluated = this.exprResolver.evaluate(value, this.exprContext);
			value = (evaluated != null ? evaluated.toString() : null);
		}
		return value;
	}

}
```



1. properties里面可以书写通配符如${app.name}，但需要注意：
   1. properties里的内容都原封不动的被放进了PropertySource里（或者说是环境里），而是只有在需要用的时候才会解析它
   2. 可以引用系统属性、环境变量等，设置引用的配置文件里都是ok的（只要保证在同一Environment就成）
2. resolvePlaceholders()它的入参是${}一起也包含进来的。它有如下特点：
   1. 若${}里面的key不存在，就原样输出，不报错。若存在就使用值替换
   2. key必须用${}包着，否则原样输出~~
   3. 若是resolveRequiredPlaceholders()方法，那key不存在就会抛错
3. getProperty()指定的是key本身，并不需要包含${}，
   1. 若key不存在返回null，但是若key的值里还有占位符，那就就继续解析。若出现占位符里的key不存在时，就抛错
   2. getRequiredProperty()方法若key不存在就直接报错了~



# BeanExpressionResolver

策略接口，用于通过将值作为表达式进行评估来解析值（如果适用）。

```java
// @since 3.0
public interface BeanExpressionResolver {
	// value此时还是复杂类型，比如本例的#{person.name}
	// BeanExpressionContext：持有beanFactory和scope的引用而已~
	@Nullable
	Object evaluate(@Nullable String value, BeanExpressionContext evalContext) throws BeansException;

}
```

## StandardBeanExpressionResolver

语言解析器的标准实现，支持解析SpEL语言。

```java
public class StandardBeanExpressionResolver implements BeanExpressionResolver {

	// 因为SpEL是支持自定义前缀、后缀的   此处保持了和SpEL默认值的统一
	// 它的属性值事public的   so你可以自定义~
	/** Default expression prefix: "#{". */
	public static final String DEFAULT_EXPRESSION_PREFIX = "#{";
	/** Default expression suffix: "}". */
	public static final String DEFAULT_EXPRESSION_SUFFIX = "}";
	private String expressionPrefix = DEFAULT_EXPRESSION_PREFIX;
	private String expressionSuffix = DEFAULT_EXPRESSION_SUFFIX;
	
	private ExpressionParser expressionParser; // 它的最终值是SpelExpressionParser

	// 每个表达式都对应一个Expression,这样可以不用重复解析了~~~
	private final Map<String, Expression> expressionCache = new ConcurrentHashMap<>(256);
	// 每个BeanExpressionContex都对应着一个取值上下文~~~
	private final Map<BeanExpressionContext, StandardEvaluationContext> evaluationCache = new ConcurrentHashMap<>(8);
	
	// 匿名内部类   解析上下文。  和TemplateParserContext的实现一样。个人觉得直接使用它更优雅
	// 和ParserContext.TEMPLATE_EXPRESSION 这个常量也一毛一样
	private final ParserContext beanExpressionParserContext = new ParserContext() {
		@Override
		public boolean isTemplate() {
			return true;
		}
		@Override
		public String getExpressionPrefix() {
			return expressionPrefix;
		}
		@Override
		public String getExpressionSuffix() {
			return expressionSuffix;
		}
	};
	

	// 空构造函数：默认就是使用的SpelExpressionParser  下面你也可以自己set你自己的实现~
	public StandardBeanExpressionResolver() {
		this.expressionParser = new SpelExpressionParser();
	}
	public void setExpressionParser(ExpressionParser expressionParser) {
		Assert.notNull(expressionParser, "ExpressionParser must not be null");
		this.expressionParser = expressionParser;
	}

	// 解析代码相对来说还是比较简单的，毕竟复杂的解析逻辑都是SpEL里边~  这里只是使用一下而已~
	@Override
	@Nullable
	public Object evaluate(@Nullable String value, BeanExpressionContext evalContext) throws BeansException {
		if (!StringUtils.hasLength(value)) {
			return value;
		}
		try {
			Expression expr = this.expressionCache.get(value);
			if (expr == null) {
				// 注意：此处isTemplte=true
				expr = this.expressionParser.parseExpression(value, this.beanExpressionParserContext);
				this.expressionCache.put(value, expr);
			}

			// 构建getValue计算时的执行上下文~~~
			// 做种解析BeanName的ast为；org.springframework.expression.spel.ast.PropertyOrFieldReference
			StandardEvaluationContext sec = this.evaluationCache.get(evalContext);
			if (sec == null) {
				// 此处指定的rootObject为：evalContext   --> BeanExpressionContext 
				sec = new StandardEvaluationContext(evalContext);
				// 此处新增了4个，加上一个默认的   所以一共就有5个属性访问器了
				// 这样我们的SpEL就能访问BeanFactory、Map、Enviroment等组件了~
				// BeanExpressionContextAccessor表示调用bean的方法~~~~(比如我们此处就是使用的它)  最终执行者为;BeanExpressionContext   它持有BeanFactory的引用嘛~
				// 如果是单村的Bean注入，最终使用的也是BeanExpressionContextAccessor 目前没有找到BeanFactoryAccessor的用于之地~~~
				// addPropertyAccessor只是：addBeforeDefault 所以只是把default的放在了最后，我们手动add的还是保持着顺序的~
				// 注意：这些属性访问器是有先后顺序的，具体看下面~~~
				sec.addPropertyAccessor(new BeanExpressionContextAccessor());
				sec.addPropertyAccessor(new BeanFactoryAccessor());
				sec.addPropertyAccessor(new MapAccessor());
				sec.addPropertyAccessor(new EnvironmentAccessor());

				// setBeanResolver不是接口方法，仅仅辅助StandardEvaluationContext 去获取Bean
				sec.setBeanResolver(new BeanFactoryResolver(evalContext.getBeanFactory()));
				sec.setTypeLocator(new StandardTypeLocator(evalContext.getBeanFactory().getBeanClassLoader()));

				// 若conversionService不为null，就使用工厂的。否则就使用SpEL里默认的DefaultConverterService那个  
				// 最后包装成TypeConverter给set进去~~~
				ConversionService conversionService = evalContext.getBeanFactory().getConversionService();
				if (conversionService != null) {
					sec.setTypeConverter(new StandardTypeConverter(conversionService));
				}

				// 这个很有意思，是一个protected的空方法，因此我们发现若我们自己要自定义BeanExpressionResolver，完全可以继承自StandardBeanExpressionResolver
				// 因为我们绝大多数情况下，只需要提供更多的计算环境即可~~~~~
				customizeEvaluationContext(sec);
				this.evaluationCache.put(evalContext, sec);
			}
			return expr.getValue(sec);
		} catch (Throwable ex) {
			throw new BeanExpressionException("Expression parsing failed", ex);
		}
	}
	
	//Spring留给我们扩展的SPI	
	protected void customizeEvaluationContext(StandardEvaluationContext evalContext) {
	}
}
```

