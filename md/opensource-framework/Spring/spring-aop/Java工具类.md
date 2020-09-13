#### Assert

#### ClassUtils：常重要的一个工具类

```java
public abstract class ClassUtils {
	
	// 这些常量 挺重要的  都是public的  我们也可以使用   这些都遵循了Java的命名规范
	public static final String ARRAY_SUFFIX = "[]";
	private static final String INTERNAL_ARRAY_PREFIX = "[";
	// 这个非常有意思：如果你是八大基本类型(PRIMITIVE类型)的数组类型， toString()方法是"[A"/"[B"...等等
	// 但是如果你是new Person[0]或者new Object[0]这种数组类型，全都是"[L"打头得，这个非常有规律性
	private static final String NON_PRIMITIVE_ARRAY_PREFIX = "[L";
	private static final char PACKAGE_SEPARATOR = '.';
	private static final char PATH_SEPARATOR = '/';
	// 是否是内部类
	private static final char INNER_CLASS_SEPARATOR = '$';
	// 这个符号，用来判断是否是CGLIB的代理类
	public static final String CGLIB_CLASS_SEPARATOR = "$$";
	public static final String CLASS_FILE_SUFFIX = ".class";

	// 下面这个很重要，提高效率必备---缓存   他们都是private的
	// 缓存八大基本数据类型 --> 包装类型的映射 所有长度相同
	private static final Map<Class<?>, Class<?>> primitiveWrapperTypeMap = new IdentityHashMap<>(8);
	// 和上面区别：上面的key是这里的value
	private static final Map<Class<?>, Class<?>> primitiveTypeToWrapperMap = new IdentityHashMap<>(8);
	// 保存所遇的基础数据类型 8 + 8（对应数组类型） + void.class 一共会有17个元素
	private static final Map<String, Class<?>> primitiveTypeNameMap = new HashMap<>(32);
	
	// 顾名思义：它是缓存一些常用的Class类型。key为class.getName()  value为Class本身
	// 比如：上面所列的17个、 还有包装类型的数组（8个） + Number.class/Class.class... + Enum.class/Exception.class/ List.clss... 总之就是缓存比较常用的一些类型吧
	private static final Map<String, Class<?>> commonClassCache = new HashMap<>(64);
	// 缓存Java语言的一些常用接口：
	//Serializable.class, Externalizable.class, Closeable.class, AutoCloseable.class, Cloneable.class, Comparable.class
	private static final Set<Class<?>> javaLanguageInterfaces;

	// 这个方法非常常用：按照获取当前线程上下文类加载器-->获取当前类类加载器-->获取系统启动类加载器的顺序来获取
	@Nullable
	public static ClassLoader getDefaultClassLoader() {
		ClassLoader cl = null;
		try {
			cl = Thread.currentThread().getContextClassLoader();
		}
		catch (Throwable ex) {
			// Cannot access thread context ClassLoader - falling back...
		}
		if (cl == null) {
			// No thread context class loader -> use class loader of this class.
			cl = ClassUtils.class.getClassLoader();
			if (cl == null) {
				// getClassLoader() returning null indicates the bootstrap ClassLoader
				try {
					cl = ClassLoader.getSystemClassLoader();
				}
				catch (Throwable ex) {
					// Cannot access system ClassLoader - oh well, maybe the caller can live with null...
				}
			}
		}
		return cl;
	}

	//这个方法较为简单，使用传入的classloader替换线程的classloader；使用场景，比如一个线程的classloader和spring的classloader不一致的时候，就可以使用这个方法替换
	@Nullable
	public static ClassLoader overrideThreadContextClassLoader(@Nullable ClassLoader classLoaderToUse) {
		Thread currentThread = Thread.currentThread();
		ClassLoader threadContextClassLoader = currentThread.getContextClassLoader();
		if (classLoaderToUse != null && !classLoaderToUse.equals(threadContextClassLoader)) {
			currentThread.setContextClassLoader(classLoaderToUse);
			return threadContextClassLoader;
		}
		else {
			return null;
		}
	}

	// 看名字就知道，是Class.forName的一个增强版本；通过指定的classloader加载对应的类；除了能正常加载普通的类型，``还能加载简单类型，数组，或者内部类``
	// 数组是通过Array.newInstance创建出来的，然后.getClass()
	public static Class<?> forName(String name, @Nullable ClassLoader classLoader){
		...
	}
    
    //判定一个类是否是简单类型的包装类；
	boolean isPrimitiveWrapper(Class<?> clazz);
	// 基本类型或者包装类型
	boolean isPrimitiveOrWrapper(Class<?> clazz)
	// 基本类型的数组类型
	public static boolean isPrimitiveArray(Class<?> clazz);
	// 基本类型的包装类型的数组类型
	public static boolean isPrimitiveWrapperArray(Class<?> clazz);
	//如果传入的类型是一个简单类型，返回这个简单类型的包装类型
	public static Class<?> resolvePrimitiveIfNecessary(Class<?> clazz);

    public static boolean isAssignable(Class<?> lhsType, Class<?> rhsType) { ... }
	public static boolean isAssignableValue(Class<?> type, @Nullable Object value) { ... }

    public static String convertResourcePathToClassName(String resourcePath) {}
	public static String convertClassNameToResourcePath(String className) {}
	//在指定类的所属包下面，寻找一个资源文件，并返回该资源文件的文件路径
	public static String addResourcePathToPackagePath(Class<?> clazz, String resourceName) {}
	public static String classPackageAsResourcePath(@Nullable Class<?> clazz) {}
	public static String classNamesToString(Class<?>... classes) {}
	public static String classNamesToString(@Nullable Collection<Class<?>> classes) {}

    //将类集合变成类型数组
	public static Class<?>[] toClassArray(Collection<Class<?>> collection) {
	//获取一个对象的所有接口
	public static Class<?>[] getAllInterfaces(Object instance) {
	//获取一个类型的所有接口
	public static Class<?>[] getAllInterfacesForClass(Class<?> clazz) {
	//获取指定类加载器下的指定类型的所有接口
	public static Class<?>[] getAllInterfacesForClass(Class<?> clazz, @Nullable ClassLoader classLoader) {

	//获取一个对象的所有接口，返回Set；（基本同上）  只是最后用Set返回   toClassArray()就成了上面所需要的数组
	public static Set<Class<?>> getAllInterfacesAsSet(Object instance) {
	public static Set<Class<?>> getAllInterfacesForClassAsSet(Class<?> clazz) {
	public static Set<Class<?>> getAllInterfacesForClassAsSet(Class<?> clazz, @Nullable ClassLoader classLoader) 

    	// 这个方法很有意思：找到两个Class 的祖先类（也就是谁是父类就返回谁了）
	public static Class<?> determineCommonAncestor(@Nullable Class<?> clazz1, @Nullable Class<?> clazz2) {}

        
    // 是否是内部类
	public static boolean isInnerClass(Class<?> clazz) {
		return (clazz.isMemberClass() && !Modifier.isStatic(clazz.getModifiers()));
	}
	// 是否是CGLIB代理对象
	public static boolean isCglibProxy(Object object) {
		return isCglibProxyClass(object.getClass());
	}
	public static boolean isCglibProxyClass(@Nullable Class<?> clazz) {
		return (clazz != null && isCglibProxyClassName(clazz.getName()));
	}
	public static boolean isCglibProxyClassName(@Nullable String className) {
		return (className != null && className.contains(CGLIB_CLASS_SEPARATOR));
	}

	// 获取用户定义的本来的类型，大部分情况下就是类型本身，主要针对cglib做了额外的判断，获取cglib代理之后的父类；
	public static Class<?> getUserClass(Object instance) {}
	// 获取一个对象的描述类型；一般来说，就是类名，能够正确处理数组，如果是JDK代理对象，能够正确输出其接口类型：
	public static String getDescriptiveType(@Nullable Object value) { ... }
	
	public static String getShortName(String className) {
	public static String getShortName(Class<?> clazz) {
	public static String getShortNameAsProperty(Class<?> clazz) {
	public static String getClassFileName(Class<?> clazz) {
	public static String getPackageName(Class<?> clazz) {
	public static String getPackageName(String fqClassName) {
	public static String getQualifiedName(Class<?> clazz) {
   
    public static String getQualifiedMethodName(Method method) {
	public static String getQualifiedMethodName(Method method, @Nullable Class<?> clazz) {
	public static boolean hasConstructor(Class<?> clazz, Class<?>... paramTypes) {
	public static <T> Constructor<T> getConstructorIfAvailable(Class<T> clazz, Class<?>... paramTypes) {
	//判断类是否有指定的public方法
	public static boolean hasMethod(Class<?> clazz, String methodName, Class<?>... paramTypes) {...}
	public static Method getMethod(Class<?> clazz, String methodName, @Nullable Class<?>... paramTypes) {...}
	public static Method getMethodIfAvailable(Class<?> clazz, String methodName, @Nullable Class<?>... paramTypes) {...}
		
	//获取指定类中匹配该方法名称的方法个数，包括非public方法；
	public static int getMethodCountForName(Class<?> clazz, String methodName) {
	//判定指定的类及其父类中是否包含指定方法名称的方法，包括非public方法；
	public static boolean hasAtLeastOneMethodWithName(Class<?> clazz, String methodName) {
	//获得最匹配的一个可以执行的方法； 和Overrid有关
	public static Method getMostSpecificMethod(Method method, @Nullable Class<?> targetClass) {
	//该方法用于判定一个方法是否是用户可用的方法
	public static boolean isUserLevelMethod(Method method) {
		// 桥接方法，
		// method. isSynthetic方法：判定一个方法是否是虚构方法（synthetic method）；什么是synthetic方法？由编译器创建的，非默认构造方法
		//（我们知道，类都有默认构造方法，当然重载了默认构造方法的除外，编译器都会生成一个默认构造方法的实现）在源码中没有对应的方法实现的方法都是虚构方法。
		// 比如上面介绍的bridge方法就是一个典型的synthetic方法；
		// isGroovyObjectMethod：判定一个方法是否是Groovy的方法，因为Spring支持Groovy，而Groovy的类都实现了groovy.lang.GroovyObject类；
		return (method.isBridge() || (!method.isSynthetic() && !isGroovyObjectMethod(method)));
	}
        public static Method getStaticMethod(Class<?> clazz, String methodName, Class<?>... args) {
            
        }
}
```

#### SpringProperties

#### SpringVersion、SpringBootVersion

#### Constants：常量获取工具

#### ObjectUtils

#### CollectionUtils（spring-core包的）























































