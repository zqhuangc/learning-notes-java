**《Spring 技术内幕》**
最核心两个 jar 包
spring-beans 定义的是规范，  
spring-context 工厂,DI的实现  
spring-core  顶层  
IOC(Inversion of Control)和 DI(Dependency Injection)的基本思想就是把类的依赖从类内部转化到外部以减少依赖。
**利用反射机制，依赖类为动态注入，类改变一般不会造成太大的影响**  
ListableBeanFactory
HierarchicalBeanFactory
DefaultListableBeanFactory

# IoC容器的使用

## DI实现方式

* **构造器注入**：(防修改)
```
通过构造器注入，能使当前实例作为不可变对象，并且能确保所有需要的依赖都是非空的.更进一步，构造器注入返回给客户代码的是一个完全初始化状态的对象.
```
* Setter方法注入：
```
Setter方法注入作为构造器注入的补充实现.能注入可选的有默认值的依赖.否则，会随处校验依赖的非空与否.
```
* 使用Field注入（用于注解方式）
## 手工装配依赖对象

在**xml配置文件**中，通过在bean节点下配置，如
//构造器注入
//属性setter方法注入

## 自动装配依赖对象
```
  @Autowired:即通过注解自动装配，默认方式是byType.，默认是通过类型匹配 具体的实现类 的
  @Resource:即通过注解自动装配，默认方式是byName.
  @Resource默认按名称装配，当找不到与名称匹配的bean才会按类型装配。
  @javax.inject.Inject:类似与@Autowired
  @Qualifier:指定实现不同的限定符，在具体注入时，通过该注解具体限定

注： @Resource注解在spring安装目录的lib\j2ee\common-annotations.jar

autodetect：通过bean类的自省机制（introspection）来决定是使用constructor还是byType方式进行自动装配。如果发现默认的构造器，那么将使用byType方式。
```

注解

通过配置自动注解扫描的根包，并且在bean上使用注解@Component(@Service,@Repositoty，@javax.inject.Named)等标示他是一个bean.

<context:annotation-config/>：  
隐式的向Spring容器注册
AutowiredAnnotationBeanPostProcessor,
CommonAnnotationBeanPostProcessor,
PersistenceAnnotationBeanPostProcessor,
RequiredAnnotationBeanPostProcessor

<context:component-scan/>：  
不但启用了对类包进行扫描以实施注释驱动 Bean 定义的功能，同时还启用了注释驱动自动注入的功能（即还隐式地在内部注册了AutowiredAnnotationBeanPostProcessor

和CommonAnnotationBeanPostProcessor），因此当使用<context:component-scan/>后，就可以将<context:annotation-config/>移除了。


* @Autowired：  

当通过@Autowired注入时，默认是通过类型匹配具体的实现类的，但是如果接口有多个实现类，Spring容器是没法做选择的，有两种方式解决这个问题：

1. @Primary注解，指定当有多个候选实现时，首选这个实现.
2. @Qualifier注解指定不同实现不同的限定符，在具体注入时，通过该注解具体限定.

可以对成员变量、方法和构造函数进行标注，来完成自动装配的工作。@Autowired的标注位置不同。

它们都会在Spring在初始化这个bean时，自动装配这个属性。注解之后就不需要**set/get**方法了。其中@Inject 和@Named是JSR 330 Standard Annotations

问题：当我们使用@Bean注解在例如@Component作用的class里面时，将会发生一种称之为注解@Bean的lite mode出现，这种不会使用CGLIB代理.所以只要我在@Bean修饰的方法之间不相互编码调用，代码将会很好的运作

bean的作用域
1. session
2. request
3. prototype
4. singleton
5. application  

其中singleton是容器级别的，即一个容器一个bean实例,spring的单例实例缓存在ConcurrentHashMap中;而GOF的单例模式是基于ClassLoader的，即一个类加载器只能有一个实例
## 通过Spring—Bean 后置处理器来增强功能

BeanPostProcessor 接口定义回调方法，你可以实现该方法来提供自己的实例化逻辑，依赖解析逻辑等。

BeanFactoryPostProcessor

* BeanPostProcessor
postProcessBeforeInitialization,
postProcessAfterInitialization

* BeanFactory 最顶层的一个接口类
  * ListableBeanFactory
  * HierarchicalBeanFactory 
  * AutowireCapableBeanFactory

# 源码分析



![](https://ws1.sinaimg.cn/large/006xzusPgy1g1nmc30grej315i11ek3v.jpg)

* 初始化入口：AbstractApplicationContext#refresh()
* 依赖注入入口：AbstractBeanFactory#getBean() 

# IoC容器的初始化



XmlBeanFactory

* 加载

> AbstractApplicationContext#refresh()     =>
>
> AbstractRefreshableApplicationContext#refreshBeanFactory()     =>
>
> AbstractXmlApplicationContext#loadBeanDefinitions(beanFactory)    => 
>
> XmlBeanDefinitionReader#loadBeanDefinitions    =>
>
> doLoadBeanDefinitions    =>    
>
> registerBeanDefinitions    =>
>
> BeanDefinitionDocumentReader  =>  
>
> DefaultBeanDefinitionDocumentReader#registerBeanDefinitions  =>
>
> doRegisterBeanDefinitions => 
>
> parseBeanDefinitions(root, this.delegate)  => 
>
> postProcessXml(root)   => 
>
> processBeanDefinition  =>
>
> BeanDefinitionReaderUtils#registerBeanDefinition

## FileSystemXmlApplicationContext的IOC容器流程（xml配置方式）

### Construction（构造函数）



```java
public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)  
            throws BeansException {
        //调用父类容器的构造方法(super(parent)方法)为容器设置好Bean资源加载器。   
        super(parent);  
        //设置资源加载器和资源定位
        setConfigLocations(configLocations);  
        if (refresh) {  
            refresh();  
        }  
    }
```
### AbstractApplicationContext#refresh()

```java
//容器初始化的过程，读入Bean定义资源，并解析注册
public void refresh() throws BeansException, IllegalStateException {
    synchronized (this.startupShutdownMonitor) {
        // Prepare this context for refreshing.
        //调用容器准备刷新的方法，获取容器的当时时间，同时给容器设置同步标识
        prepareRefresh();
        // Tell the subclass to refresh the internal bean factory.
        //告诉子类启动refreshBeanFactory()方法
        //Bean定义资源文件的载入从子类的refreshBeanFactory()方法启动
        //(加载)
        ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
        // Prepare the bean factory for use in this context.
        //（手动加载特定类）为 BeanFactory 配置容器特性，例如类加载器、事件处理器等
        prepareBeanFactory(beanFactory);
        try {
            // Allows post-processing of the bean factory in context subclasses.
            // 为容器的某些子类指定特殊的BeanPost事件处理器
            postProcessBeanFactory(beanFactory);
            // Invoke factory processors registered as beans in the context.
            // 调用所有注册的 BeanFactoryPostProcessor 的 Bean
            invokeBeanFactoryPostProcessors(beanFactory);
            // Register bean processors that intercept bean creation.
            // 为BeanFactory 注册 BeanPost 事件处理器.  
            // BeanPostProcessor 是 Bean 后置处理器，用于监听容器触发的事件 
            registerBeanPostProcessors(beanFactory);
            // Initialize message source for this context.
            // 初始化信息源，和国际化相关.
            initMessageSource();
            // Initialize event multicaster for this context.
            // 初始化容器事件传播器
            initApplicationEventMulticaster();
            // Initialize other special beans in specific context subclasses.
            // 调用子类的某些特殊Bean初始化方法
            onRefresh();
            // Check for listener beans and register them.
            // 为事件传播器注册事件监听器.
            registerListeners();
            // Instantiate all remaining (non-lazy-init) singletons.
            // 初始化所有剩余的单例 Bean.
            // 只要单例 bean 非抽象、非懒加载就会在这里进行依赖注入,getBean????
            finishBeanFactoryInitialization(beanFactory);
            // Last step: publish corresponding event.
            // 初始化容器的生命周期事件处理器，并发布容器的生命周期事件
            finishRefresh();
        }
        catch (BeansException ex) {
            // Destroy already created singletons to avoid dangling resources.
            // 销毁以创建的单态Bean
            destroyBeans();
            // Reset 'active' flag.
            // 取消refresh操作，重置容器的同步标识.
            cancelRefresh(ex);
            // Propagate exception to caller.
            throw ex;
        }
    }
}
```

### AbstractRefreshableApplicationContext#refreshBeanFactory()

```java
public abstract class AbstractRefreshableApplicationContext extends AbstractApplicationContext {
    ...
    @Override
	protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {//如果已经有容器，销毁容器中的bean，关闭容器
			destroyBeans();
			closeBeanFactory();
		}
		try {
			//创建IOC容器
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			//对IOC容器进行定制化，如设置启动参数，开启注解的自动装配等
			customizeBeanFactory(beanFactory);
			//调用载入Bean定义的方法，主要这里又使用了一个委派模式，在当前类中只定义了抽象的loadBeanDefinitions方法，具体的实现调用子类容器
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
    ...
}
```

### AbstractXmlApplicationContext#loadBeanDefinitions

```java
public abstract class AbstractXmlApplicationContext extends AbstractRefreshableConfigApplicationContext {  
    ……  
    //实现父类抽象的载入Bean定义方法  
    @Override  
    protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {  
        //创建XmlBeanDefinitionReader，即创建Bean读取器，并通过回调设置到容器中去，容  器使用该读取器读取Bean定义资源  
        XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);  
        //为Bean读取器设置Spring资源加载器，AbstractXmlApplicationContext的  
        //祖先父类AbstractApplicationContext继承DefaultResourceLoader，因此，容器本身也是一个资源加载器  
       beanDefinitionReader.setResourceLoader(this);  
       //为Bean读取器设置SAX xml解析器  
       beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));  
       //当Bean读取器读取Bean定义的Xml资源文件时，启用Xml的校验机制  
       initBeanDefinitionReader(beanDefinitionReader);  
       //Bean读取器真正实现加载的方法  
       loadBeanDefinitions(beanDefinitionReader);  
   }  
   //Xml Bean读取器加载Bean定义资源  
   protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {  
       //获取Bean定义资源的定位  
       Resource[] configResources = getConfigResources();  
       if (configResources != null) {  
           //Xml Bean读取器调用其父类AbstractBeanDefinitionReader读取定位  
           //的Bean定义资源  
           reader.loadBeanDefinitions(configResources);  
       }  
       //如果子类中获取的Bean定义资源定位为空，则获取FileSystemXmlApplicationContext构造方法中setConfigLocations方法设置的资源  
       String[] configLocations = getConfigLocations();  
       if (configLocations != null) {  
           //Xml Bean读取器调用其父类AbstractBeanDefinitionReader读取定位  
           //的Bean定义资源  
           reader.loadBeanDefinitions(configLocations);  
       }  
   }  
   //这里又使用了一个委托模式，调用子类的获取Bean定义资源定位的方法  
   //该方法在ClassPathXmlApplicationContext中进行实现，对于我们  
   //举例分析源码的FileSystemXmlApplicationContext没有使用该方法  
   protected Resource[] getConfigResources() {  
       return null;  
   } 
}
```

### AbstractBeanDefinitionReader读取Bean定义资源

```java
public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionStoreException {
		//获取在IoC容器初始化过程中设置的资源加载器 
		ResourceLoader resourceLoader = getResourceLoader();
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot import bean definitions from location [" + location + "]: no ResourceLoader available");
		}

		if (resourceLoader instanceof ResourcePatternResolver) {
			// Resource pattern matching available.
			try {
				//将指定位置的Bean定义资源文件解析为Spring IOC容器封装的资源  
	            //加载多个指定位置的Bean定义资源文件  
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
				//委派调用其子类XmlBeanDefinitionReader的方法，实现加载功能 
				int loadCount = loadBeanDefinitions(resources);
				if (actualResources != null) {
					for (Resource resource : resources) {
						actualResources.add(resource);
					}
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Loaded " + loadCount + " bean definitions from location pattern [" + location + "]");
				}
				return loadCount;
			}
			catch (IOException ex) {
				throw new BeanDefinitionStoreException(
						"Could not resolve bean definition resource pattern [" + location + "]", ex);
			}
		}
		else {
			// Can only load single resources by absolute URL.
			//将指定位置的Bean定义资源文件解析为Spring IOC容器封装的资源  
	        //加载单个指定位置的Bean定义资源文件
			Resource resource = resourceLoader.getResource(location);
			//委派调用其子类XmlBeanDefinitionReader的方法，实现加载功能
			int loadCount = loadBeanDefinitions(resource);
			if (actualResources != null) {
				actualResources.add(resource);
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + loadCount + " bean definitions from location [" + location + "]");
			}
			return loadCount;
		}
	}
```

#### 资源加载器 DefaultResourceLoader 获取要读入的资源

XmlBeanDefinitionReader通过调用其父类`DefaultResourceLoader`的getResource方法获取要加载的资源

```java
//获取Resource的具体实现方法
public Resource getResource(String location) {
    Assert.notNull(location, "Location must not be null");
    //如果是类路径的方式，那需要使用ClassPathResource 来得到bean 文件的资源对象
    if (location.startsWith(CLASSPATH_URL_PREFIX)) {
        return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
    }
    else {
        try {
            // Try to parse the location as a URL...
            // 如果是URL 方式，使用UrlResource 作为bean 文件的资源对象
            URL url = new URL(location);
            return new UrlResource(url);
        }
        catch (MalformedURLException ex) {
            // No URL -> resolve as resource path.
            //如果既不是classpath标识，又不是URL标识的Resource定位，则调用  
            //容器本身的getResourceByPath方法获取Resource 
            return getResourceByPath(location);
        }
    }
}
```

* FileSystemXmlApplicationContext容器提供了getResourceByPath方法的实现，就是为了处理既不是classpath标识，又不是URL标识的Resource定位这种情况。

```java
protected Resource getResourceByPath(String path) {    
   if (path != null && path.startsWith("/")) {    
        path = path.substring(1);    
    }  
    //这里使用文件系统资源对象来定义bean 文件
    return new FileSystemResource(path);  
}
```

### XmlBeanDefinitionReader 加载Bean定义资源

```java
public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {
    ...
    //XmlBeanDefinitionReader加载资源的入口方法
	public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
		//将读入的XML资源进行特殊编码处理
		return loadBeanDefinitions(new EncodedResource(resource));
	}
    ...
    //这里是载入XML形式Bean定义资源文件方法
	public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isInfoEnabled()) {
			logger.info("Loading XML bean definitions from " + encodedResource.getResource());
		}

		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<EncodedResource>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
			//将资源文件转为InputStream的IO流
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
				//从InputStream中得到XML的解析源
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
				//这里是具体的读取过程
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				//关闭从Resource中得到的IO流
				inputStream.close();
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}
    ...
    //从特定XML文件中实际载入Bean定义资源的方法
	protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
			int validationMode = getValidationModeForResource(resource);
			//将XML文件转换为DOM对象，解析过程由documentLoader实现
			Document doc = this.documentLoader.loadDocument(
					inputSource, getEntityResolver(), this.errorHandler, validationMode, isNamespaceAware());
			//这里是启动对Bean定义解析的详细过程，该解析过程会用到Spring的Bean配置规则
			return registerBeanDefinitions(doc, resource);
		}
        ...
	}
    ...
}
```

### DocumentLoader 将 Bean 定义资源转换为 Document 对象

* DefaultDocumentLoader#loadDocument

```java
//使用标准的JAXP将载入的Bean定义资源转换成document对象
public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
                             ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {

    //创建文件解析器工厂
    DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
    if (logger.isDebugEnabled()) {
        logger.debug("Using JAXP provider [" + factory.getClass().getName() + "]");
    }
    //创建文档解析器
    DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
    //解析Spring的Bean定义资源
    return builder.parse(inputSource);
}

//创建文档解析工厂
protected DocumentBuilderFactory createDocumentBuilderFactory(int validationMode, boolean namespaceAware)
    throws ParserConfigurationException {

    DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
    factory.setNamespaceAware(namespaceAware);

    //设置解析XML的校验
    if (validationMode != XmlValidationModeDetector.VALIDATION_NONE) {
        factory.setValidating(true);

        if (validationMode == XmlValidationModeDetector.VALIDATION_XSD) {
            // Enforce namespace aware for XSD...
            factory.setNamespaceAware(true);
            try {
                factory.setAttribute(SCHEMA_LANGUAGE_ATTRIBUTE, XSD_SCHEMA_LANGUAGE);
            }
            catch (IllegalArgumentException ex) {
                ParserConfigurationException pcex = new ParserConfigurationException(
                    "Unable to validate using XSD: Your JAXP provider [" + factory +
                    "] does not support XML Schema. Are you running on Java 1.4 with Apache Crimson? " +
                    "Upgrade to Apache Xerces (or Java 1.5) for full XSD support.");
                pcex.initCause(ex);
                throw pcex;
            }
        }
    }

    return factory;
}
```

### XmlBeanDefinitionReader解析载入的Bean定义资源文件

```java
//按照Spring的Bean语义要求将Bean定义资源解析并转换为容器内部数据结构
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    //得到BeanDefinitionDocumentReader来对xml格式的BeanDefinition解析
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    documentReader.setEnvironment(this.getEnvironment());
    //获得容器中注册的Bean数量
    int countBefore = getRegistry().getBeanDefinitionCount();
    //解析过程入口，这里使用了委派模式，BeanDefinitionDocumentReader只是个接口，
    //具体的解析实现过程有实现类DefaultBeanDefinitionDocumentReader完成
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    //统计解析的Bean数量
    return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

### DefaultBeanDefinitionDocumentReader对Bean定义的Document对象解析

```java
public class DefaultBeanDefinitionDocumentReader implements BeanDefinitionDocumentReader {
    ...
	//根据Spring DTD对Bean的定义规则解析Bean定义Document对象
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		//获得XML描述符
		this.readerContext = readerContext;
		logger.debug("Loading bean definitions");
		//获得Document的根元素
		Element root = doc.getDocumentElement();
		doRegisterBeanDefinitions(root);
	}
    
	protected void doRegisterBeanDefinitions(Element root) {
        ...
		//具体的解析过程由BeanDefinitionParserDelegate实现，  
	    //BeanDefinitionParserDelegate中定义了Spring Bean定义XML文件的各种元素 
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(this.readerContext, root, parent);

		//在解析Bean定义之前，进行自定义的解析，增强解析过程的可扩展性
		preProcessXml(root);
		//从Document的根元素开始进行Bean定义的Document对象
		parseBeanDefinitions(root, this.delegate);
		//在解析Bean定义之后，进行自定义的解析，增加解析过程的可扩展性
		postProcessXml(root);
		this.delegate = parent;
	}
    ...

	//使用Spring的Bean规则从Document的根元素开始进行Bean定义的Document对象
	protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		//Bean定义的Document对象使用了Spring默认的XML命名空间
		if (delegate.isDefaultNamespace(root)) {
			//获取Bean定义的Document对象根元素的所有子节点
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				//获得Document节点是XML元素节点
				if (node instanceof Element) {
					Element ele = (Element) node;
					//Bean定义的Document的元素节点使用的是Spring默认的XML命名空间 
					if (delegate.isDefaultNamespace(ele)) {
						//使用Spring的Bean规则解析元素节点
						parseDefaultElement(ele, delegate);
					}
					else {
						//没有使用Spring默认的XML命名空间，则使用用户自定义的解
						//析规则解析元素节点
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			//Document的根节点没有使用Spring默认的命名空间，则使用用户自定义的  
		    //解析规则解析Document根节点
			delegate.parseCustomElement(root);
		}
	}

	//使用Spring的Bean规则解析Document元素节点
	private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		//如果元素节点是<Import>导入元素，进行导入解析
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
		//如果元素节点是<Alias>别名元素，进行别名解析
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
		//元素节点既不是导入元素，也不是别名元素，即普通的<Bean>元素，  
		//按照Spring的Bean规则解析元素
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}

	//解析<Import>导入元素，从给定的导入路径加载Bean定义资源到Spring IoC容器中
	protected void importBeanDefinitionResource(Element ele) {
		//获取给定的导入元素的location属性
		String location = ele.getAttribute(RESOURCE_ATTRIBUTE);
		//如果导入元素的location属性值为空，则没有导入任何资源，直接返回
		if (!StringUtils.hasText(location)) {
			getReaderContext().error("Resource location must not be empty", ele);
			return;
		}

		// Resolve system properties: e.g. "${user.dir}"
		//使用系统变量值解析location属性值
		location = environment.resolveRequiredPlaceholders(location);

		Set<Resource> actualResources = new LinkedHashSet<Resource>(4);

		// Discover whether the location is an absolute or relative URI
		//标识给定的导入元素的location是否是绝对路径
		boolean absoluteLocation = false;
		try {
			absoluteLocation = ResourcePatternUtils.isUrl(location) || ResourceUtils.toURI(location).isAbsolute();
		}
		catch (URISyntaxException ex) {
			// cannot convert to an URI, considering the location relative
			// unless it is the well-known Spring prefix "classpath*:"
			//给定的导入元素的location不是绝对路径
		}

		// Absolute or relative?
		//给定的导入元素的location是绝对路径
		if (absoluteLocation) {
			try {
				//使用资源读入器加载给定路径的Bean定义资源
				int importCount = getReaderContext().getReader().loadBeanDefinitions(location, actualResources);
				if (logger.isDebugEnabled()) {
					logger.debug("Imported " + importCount + " bean definitions from URL location [" + location + "]");
				}
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error(
						"Failed to import bean definitions from URL location [" + location + "]", ele, ex);
			}
		}
		else {
			// No URL -> considering resource location as relative to the current file.
			//给定的导入元素的location是相对路径
			try {
				int importCount;
				//将给定导入元素的location封装为相对路径资源
				Resource relativeResource = getReaderContext().getResource().createRelative(location);
				//封装的相对路径资源存在
				if (relativeResource.exists()) {
					//使用资源读入器加载Bean定义资源
					importCount = getReaderContext().getReader().loadBeanDefinitions(relativeResource);
					actualResources.add(relativeResource);
				}
				//封装的相对路径资源不存在
				else {
					//获取Spring IOC容器资源读入器的基本路径 
					String baseLocation = getReaderContext().getResource().getURL().toString();
					//根据Spring IoC容器资源读入器的基本路径加载给定导入路径的资源
					importCount = getReaderContext().getReader().loadBeanDefinitions(
							StringUtils.applyRelativePath(baseLocation, location), actualResources);
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Imported " + importCount + " bean definitions from relative location [" + location + "]");
				}
			}
			catch (IOException ex) {
				getReaderContext().error("Failed to resolve current resource location", ele, ex);
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to import bean definitions from relative location [" + location + "]",
						ele, ex);
			}
		}
		Resource[] actResArray = actualResources.toArray(new Resource[actualResources.size()]);
		//在解析完<Import>元素之后，发送容器导入其他资源处理完成事件
		getReaderContext().fireImportProcessed(location, actResArray, extractSource(ele));
	}

	//解析<Alias>别名元素，为Bean向Spring IoC容器注册别名
	protected void processAliasRegistration(Element ele) {
		//获取<Alias>别名元素中name的属性值
		String name = ele.getAttribute(NAME_ATTRIBUTE);
		//获取<Alias>别名元素中alias的属性值
		String alias = ele.getAttribute(ALIAS_ATTRIBUTE);
		boolean valid = true;
		//<alias>别名元素的name属性值为空
		if (!StringUtils.hasText(name)) {
			getReaderContext().error("Name must not be empty", ele);
			valid = false;
		}
		//<alias>别名元素的alias属性值为空
		if (!StringUtils.hasText(alias)) {
			getReaderContext().error("Alias must not be empty", ele);
			valid = false;
		}
		if (valid) {
			try {
				//向容器的资源读入器注册别名
				getReaderContext().getRegistry().registerAlias(name, alias);
			}
			catch (Exception ex) {
				getReaderContext().error("Failed to register alias '" + alias +
						"' for bean with name '" + name + "'", ele, ex);
			}
			//在解析完<Alias>元素之后，发送容器别名处理完成事件
			getReaderContext().fireAliasRegistered(name, alias, extractSource(ele));
		}
	}


	//解析Bean定义资源Document对象的普通元素
	protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		// BeanDefinitionHolder是对BeanDefinition的封装，即Bean定义的封装类  
		// 对Document对象中<Bean>元素的解析由BeanDefinitionParserDelegate实现
		// BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// Register the final decorated instance.
				//向Spring IOC容器注册解析得到的Bean定义，这是Bean定义向IOC容器注册的入口
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			//在完成向Spring IOC容器注册解析得到的Bean定义之后，发送注册事件
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
    ...
}

```

### BeanDefinitionParserDelegate解析Bean定义资源文件中的 Bean 元素

```java
//解析Bean定义资源文件中的<Bean>元素，这个方法中主要处理<Bean>元素的id，name  
//和别名属性
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {
    //获取<Bean>元素中的id属性值
    String id = ele.getAttribute(ID_ATTRIBUTE);
    //获取<Bean>元素中的name属性值
    String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

    //获取<Bean>元素中的alias属性值
    List<String> aliases = new ArrayList<String>();
    //将<Bean>元素中的所有name属性值存放到别名中
    if (StringUtils.hasLength(nameAttr)) {
        String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
        aliases.addAll(Arrays.asList(nameArr));
    }

    String beanName = id;
    //如果<Bean>元素中没有配置id属性时，将别名中的第一个值赋值给beanName
    if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
        beanName = aliases.remove(0);
        if (logger.isDebugEnabled()) {
            logger.debug("No XML 'id' specified - using '" + beanName +
                         "' as bean name and " + aliases + " as aliases");
        }
    }

    //检查<Bean>元素所配置的id或者name的唯一性，containingBean标识<Bean>  
    //元素中是否包含子<Bean>元素
    if (containingBean == null) {
        //检查<Bean>元素所配置的id、name或者别名是否重复
        checkNameUniqueness(beanName, aliases, ele);
    }

    //详细对<Bean>元素中配置的Bean定义进行解析的地方
    AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
    if (beanDefinition != null) {
        if (!StringUtils.hasText(beanName)) {
            try {
                if (containingBean != null) {
                    //如果<Bean>元素中没有配置id、别名或者name，且没有包含子元素
                    //<Bean>元素，为解析的Bean生成一个唯一beanName并注册
                    beanName = BeanDefinitionReaderUtils.generateBeanName(
                        beanDefinition, this.readerContext.getRegistry(), true);
                }
                else {
                    //如果<Bean>元素中没有配置id、别名或者name，且包含了子元素
                    //<Bean>元素，为解析的Bean使用别名向IOC容器注册
                    beanName = this.readerContext.generateBeanName(beanDefinition);
                    // Register an alias for the plain bean class name, if still possible,
                    // if the generator returned the class name plus a suffix.
                    // This is expected for Spring 1.2/2.0 backwards compatibility.
                    //为解析的Bean使用别名注册时，为了向后兼容
                    //Spring1.2/2.0，给别名添加类名后缀
                    String beanClassName = beanDefinition.getBeanClassName();
                    if (beanClassName != null &&
                        beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
                        !this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
                        aliases.add(beanClassName);
                    }
                }
                if (logger.isDebugEnabled()) {
                    logger.debug("Neither XML 'id' nor 'name' specified - " +
                                 "using generated bean name [" + beanName + "]");
                }
            }
            catch (Exception ex) {
                error(ex.getMessage(), ele);
                return null;
            }
        }
        String[] aliasesArray = StringUtils.toStringArray(aliases);
        return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
    }
    //当解析出错时，返回null
    return null;
}

//详细对<Bean>元素中配置的Bean定义其他属性进行解析，由于上面的方法中已经对
//Bean的id、name和别名等属性进行了处理，该方法中主要处理除这三个以外的其他属性数据  
public AbstractBeanDefinition parseBeanDefinitionElement(
    Element ele, String beanName, BeanDefinition containingBean) {

    //记录解析的<Bean>
    this.parseState.push(new BeanEntry(beanName));

    //这里只读取<Bean>元素中配置的class名字，然后载入到BeanDefinition中去  
    //只是记录配置的class名字，不做实例化，对象的实例化在依赖注入时完成
    String className = null;
    if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
        className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
    }

    try {
        String parent = null;
        //如果<Bean>元素中配置了parent属性，则获取parent属性的值
        if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
            parent = ele.getAttribute(PARENT_ATTRIBUTE);
        }

        //根据<Bean>元素配置的class名称和parent属性值创建BeanDefinition  
        //为载入Bean定义信息做准备
        AbstractBeanDefinition bd = createBeanDefinition(className, parent);

        //对当前的<Bean>元素中配置的一些属性进行解析和设置，如配置的单态(singleton)属性等
        parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
        //为<Bean>元素解析的Bean设置description信息
        bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

        //对<Bean>元素的meta(元信息)属性解析
        parseMetaElements(ele, bd);
        //对<Bean>元素的lookup-method属性解析
        parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
        //对<Bean>元素的replaced-method属性解析
        parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

        //解析<Bean>元素的构造方法设置
        parseConstructorArgElements(ele, bd);
        //解析<Bean>元素的<property>设置
        parsePropertyElements(ele, bd);
        //解析<Bean>元素的qualifier属性
        parseQualifierElements(ele, bd);

        //为当前解析的Bean设置所需的资源和依赖对象
        bd.setResource(this.readerContext.getResource());
        bd.setSource(extractSource(ele));

        return bd;
    }
    catch (ClassNotFoundException ex) {
        error("Bean class [" + className + "] not found", ele, ex);
    }
    catch (NoClassDefFoundError err) {
        error("Class that bean class [" + className + "] depends on not found", ele, err);
    }
    catch (Throwable ex) {
        error("Unexpected failure during bean definition parsing", ele, ex);
    }
    finally {
        this.parseState.pop();
    }
    //解析<Bean>元素出错时，返回null
    return null;
}
```

### BeanDefinitionParserDelegate 解析 property 元素

```java
//解析<Bean>元素中的<property>子元素
public void parsePropertyElements(Element beanEle, BeanDefinition bd) {
    //获取<Bean>元素中所有的子元素
    NodeList nl = beanEle.getChildNodes();
    for (int i = 0; i < nl.getLength(); i++) {
        Node node = nl.item(i);
        //如果子元素是<property>子元素，则调用解析<property>子元素方法解析
        if (isCandidateElement(node) && nodeNameEquals(node, PROPERTY_ELEMENT)) {
            parsePropertyElement((Element) node, bd);
        }
    }
}

//解析<property>元素
public void parsePropertyElement(Element ele, BeanDefinition bd) {
    //获取<property>元素的名字
    String propertyName = ele.getAttribute(NAME_ATTRIBUTE);
    if (!StringUtils.hasLength(propertyName)) {
        error("Tag 'property' must have a 'name' attribute", ele);
        return;
    }
    this.parseState.push(new PropertyEntry(propertyName));
    try {
        //如果一个Bean中已经有同名的property存在，则不进行解析，直接返回。  
        //即如果在同一个Bean中配置同名的property，则只有第一个起作用
        if (bd.getPropertyValues().contains(propertyName)) {
            error("Multiple 'property' definitions for property '" + propertyName + "'", ele);
            return;
        }
        //解析获取property的值
        Object val = parsePropertyValue(ele, bd, propertyName);
        //根据property的名字和值创建property实例
        PropertyValue pv = new PropertyValue(propertyName, val);
        //解析<property>元素中的属性
        parseMetaElements(ele, pv);
        pv.setSource(extractSource(ele));
        bd.getPropertyValues().addPropertyValue(pv);
    }
    finally {
        this.parseState.pop();
    }
}

//解析获取property值
public Object parsePropertyValue(Element ele, BeanDefinition bd, String propertyName) {
    String elementName = (propertyName != null) ?
        "<property> element for property '" + propertyName + "'" :
    "<constructor-arg> element";

    // Should only have one child element: ref, value, list, etc.
    //获取<property>的所有子元素，只能是其中一种类型:ref,value,list等
    NodeList nl = ele.getChildNodes();
    Element subElement = null;
    for (int i = 0; i < nl.getLength(); i++) {
        Node node = nl.item(i);
        //子元素不是description和meta属性
        if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT) &&
            !nodeNameEquals(node, META_ELEMENT)) {
            // Child element is what we're looking for.
            if (subElement != null) {
                error(elementName + " must not contain more than one sub-element", ele);
            }
            else {//当前<property>元素包含有子元素
                subElement = (Element) node;
            }
        }
    }

    //判断property的属性值是ref还是value，不允许既是ref又是value 
    boolean hasRefAttribute = ele.hasAttribute(REF_ATTRIBUTE);
    boolean hasValueAttribute = ele.hasAttribute(VALUE_ATTRIBUTE);
    if ((hasRefAttribute && hasValueAttribute) ||
        ((hasRefAttribute || hasValueAttribute) && subElement != null)) {
        error(elementName +
              " is only allowed to contain either 'ref' attribute OR 'value' attribute OR sub-element", ele);
    }

    //如果属性是ref，创建一个ref的数据对象RuntimeBeanReference
    //这个对象封装了ref信息
    if (hasRefAttribute) {
        String refName = ele.getAttribute(REF_ATTRIBUTE);
        if (!StringUtils.hasText(refName)) {
            error(elementName + " contains empty 'ref' attribute", ele);
        }
        //一个指向运行时所依赖对象的引用
        RuntimeBeanReference ref = new RuntimeBeanReference(refName);
        //设置这个ref的数据对象是被当前的property对象所引用
        ref.setSource(extractSource(ele));
        return ref;
    }
    //如果属性是value，创建一个value的数据对象TypedStringValue
    //这个对象封装了value信息
    else if (hasValueAttribute) {
        //一个持有String类型值的对象
        TypedStringValue valueHolder = new TypedStringValue(ele.getAttribute(VALUE_ATTRIBUTE));
        //设置这个value数据对象是被当前的property对象所引用
        valueHolder.setSource(extractSource(ele));
        return valueHolder;
    }
    //如果当前<property>元素还有子元素
    else if (subElement != null) {
        //解析<property>的子元素
        return parsePropertySubElement(subElement, bd);
    }
    else {
        // Neither child element nor "ref" or "value" attribute found.
        //propery属性中既不是ref，也不是value属性，解析出错返回null
        error(elementName + " must specify a ref or value", ele);
        return null;
    }
}
```

#### 解析 property 元素的子元素

```java
//解析<property>元素中ref,value或者集合等子元素
public Object parsePropertySubElement(Element ele, BeanDefinition bd) {
    return parsePropertySubElement(ele, bd, null);
}

public Object parsePropertySubElement(Element ele, BeanDefinition bd, String defaultValueType) {
    //如果<property>没有使用Spring默认的命名空间，则使用用户自定义的规则解析
    //内嵌元素
    if (!isDefaultNamespace(ele)) {
        return parseNestedCustomElement(ele, bd);
    }
    //如果子元素是bean，则使用解析<Bean>元素的方法解析
    else if (nodeNameEquals(ele, BEAN_ELEMENT)) {
        BeanDefinitionHolder nestedBd = parseBeanDefinitionElement(ele, bd);
        if (nestedBd != null) {
            nestedBd = decorateBeanDefinitionIfRequired(ele, nestedBd, bd);
        }
        return nestedBd;
    }
    //如果子元素是ref，ref中只能有以下3个属性：bean、local、parent
    else if (nodeNameEquals(ele, REF_ELEMENT)) {
        // A generic reference to any name of any bean.
        //获取<property>元素中的bean属性值，引用其他解析的Bean的名称  
        //可以不再同一个Spring配置文件中，具体请参考Spring对ref的配置规则
        String refName = ele.getAttribute(BEAN_REF_ATTRIBUTE);
        boolean toParent = false;
        if (!StringUtils.hasLength(refName)) {
            // A reference to the id of another bean in the same XML file.
            //获取<property>元素中的local属性值，引用同一个Xml文件中配置  
            //的Bean的id，local和ref不同，local只能引用同一个配置文件中的Bean
            refName = ele.getAttribute(LOCAL_REF_ATTRIBUTE);
            if (!StringUtils.hasLength(refName)) {
                // A reference to the id of another bean in a parent context.
                //获取<property>元素中parent属性值，引用父级容器中的Bean
                refName = ele.getAttribute(PARENT_REF_ATTRIBUTE);
                toParent = true;

                if (!StringUtils.hasLength(refName)) {
                    error("'bean', 'local' or 'parent' is required for <ref> element", ele);
                    return null;
                }
            }
        }

        //没有配置ref的目标属性值 
        if (!StringUtils.hasText(refName)) {
            error("<ref> element contains empty target attribute", ele);
            return null;
        }
        //创建ref类型数据，指向被引用的对象
        RuntimeBeanReference ref = new RuntimeBeanReference(refName, toParent);
        //设置引用类型值是被当前子元素所引用
        ref.setSource(extractSource(ele));
        return ref;
    }
    //如果子元素是<idref>，使用解析ref元素的方法解析
    else if (nodeNameEquals(ele, IDREF_ELEMENT)) {
        return parseIdRefElement(ele);
    }
    //如果子元素是<value>，使用解析value元素的方法解析
    else if (nodeNameEquals(ele, VALUE_ELEMENT)) {
        return parseValueElement(ele, defaultValueType);
    }
    //如果子元素是null，为<property>设置一个封装null值的字符串数据
    else if (nodeNameEquals(ele, NULL_ELEMENT)) {
        // It's a distinguished null value. Let's wrap it in a TypedStringValue
        // object in order to preserve the source location.
        TypedStringValue nullHolder = new TypedStringValue(null);
        nullHolder.setSource(extractSource(ele));
        return nullHolder;
    }
    //如果子元素是<array>，使用解析array集合子元素的方法解析
    else if (nodeNameEquals(ele, ARRAY_ELEMENT)) {
        return parseArrayElement(ele, bd);
    }
    //如果子元素是<list>，使用解析list集合子元素的方法解析
    else if (nodeNameEquals(ele, LIST_ELEMENT)) {
        return parseListElement(ele, bd);
    }
    //如果子元素是<set>，使用解析set集合子元素的方法解析
    else if (nodeNameEquals(ele, SET_ELEMENT)) {
        return parseSetElement(ele, bd);
    }
    //如果子元素是<map>，使用解析map集合子元素的方法解析
    else if (nodeNameEquals(ele, MAP_ELEMENT)) {
        return parseMapElement(ele, bd);
    }
    //如果子元素是<props>，使用解析props集合子元素的方法解析
    else if (nodeNameEquals(ele, PROPS_ELEMENT)) {
        return parsePropsElement(ele);
    }
    //既不是ref，又不是value，也不是集合，则子元素配置错误，返回null
    else {
        error("Unknown property sub-element: [" + ele.getNodeName() + "]", ele);
        return null;
    }
}
```

#### 解析 list 子元素

```java
//解析<list>集合子元素
public List parseListElement(Element collectionEle, BeanDefinition bd) {
    //获取<list>元素中的value-type属性，即获取集合元素的数据类型
    String defaultElementType = collectionEle.getAttribute(VALUE_TYPE_ATTRIBUTE);
    //获取<list>集合元素中的所有子节点
    NodeList nl = collectionEle.getChildNodes();
    //Spring中将List封装为ManagedList
    ManagedList<Object> target = new ManagedList<Object>(nl.getLength());
    target.setSource(extractSource(collectionEle));
    //设置集合目标数据类型
    target.setElementTypeName(defaultElementType);
    target.setMergeEnabled(parseMergeAttribute(collectionEle));
    //具体的<list>元素解析
    parseCollectionElements(nl, target, bd, defaultElementType);
    return target;
}

//具体解析<list>集合元素，<array>、<list>和<set>都使用该方法解析
protected void parseCollectionElements(
    NodeList elementNodes, Collection<Object> target, BeanDefinition bd, String defaultElementType) {
    //遍历集合所有节点
    for (int i = 0; i < elementNodes.getLength(); i++) {
        Node node = elementNodes.item(i);
        //节点不是description节点
        if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT)) {
            //将解析的元素加入集合中，递归调用下一个子元素 
            target.add(parsePropertySubElement((Element) node, bd, defaultElementType));
        }
    }
}
```

### 解析过后的BeanDefinition在IoC容器中的注册
`DefaultBeanDefinitionDocumentReader`对Bean定义转换的Document对象解析的流程中，在其parseDefaultElement方法中完成对Document对象的解析后得到封装BeanDefinition的BeanDefinitionHold对象，然后调用BeanDefinitionReaderUtils的registerBeanDefinition方法向IoC容器注册解析的Bean

* BeanDefinitionReaderUtils#registerBeanDefinition

```java
//将解析的 BeanDefinitionHold 注册到容器中 
public static void registerBeanDefinition(BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)  
    throws BeanDefinitionStoreException {  
        //获取解析的 BeanDefinition 的名称
         String beanName = definitionHolder.getBeanName();  
        //向 IoC 容器注册 BeanDefinition 
        registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());  
        //如果解析的 BeanDefinition 有别名，向容器为其注册别名  
         String[] aliases = definitionHolder.getAliases();  
        if (aliases != null) {  
            for (String aliase : aliases) {  
                registry.registerAlias(beanName, aliase);  
            }  
        }  
}
```



### DefaultListableBeanFactory 向 IOC 容器注册解析后的 BeanDefinition

**HashMap**

```java
//存储注册的BeanDefinition  
private final Map<String, BeanDefinition> beanDefinitionMap = new ConcurrentHashMap<String, BeanDefinition>();  
//向IoC容器注册解析的BeanDefiniton  
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)  
    throws BeanDefinitionStoreException {  
    Assert.hasText(beanName, "Bean name must not be empty");  
    Assert.notNull(beanDefinition, "BeanDefinition must not be null");  
    //校验解析的BeanDefiniton  
    if (beanDefinition instanceof AbstractBeanDefinition) {  
        try {  
            ((AbstractBeanDefinition) beanDefinition).validate();  
        }  
        catch (BeanDefinitionValidationException ex) {  
            throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,  
                                                   "Validation of bean definition failed", ex);  
        }  
    }  
    //注册的过程中需要线程同步，以保证数据的一致性  
    synchronized (this.beanDefinitionMap) {  
        Object oldBeanDefinition = this.beanDefinitionMap.get(beanName);  
        //检查是否有同名的BeanDefinition已经在IoC容器中注册，如果已经注册，  
        //并且不允许覆盖已注册的Bean，则抛出注册失败异常  
        if (oldBeanDefinition != null) {  
            if (!this.allowBeanDefinitionOverriding) {  
                throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,  
                                                       "Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +  
                                                       "': There is already [" + oldBeanDefinition + "] bound.");  
            }  
            else {//如果允许覆盖，则同名的Bean，后注册的覆盖先注册的  
                if (this.logger.isInfoEnabled()) {  
                    this.logger.info("Overriding bean definition for bean '" + beanName +  
                                     "': replacing [" + oldBeanDefinition + "] with [" + beanDefinition + "]");  
                }  
            }  
        }  
        //IoC容器中没有已经注册同名的Bean，按正常注册流程注册  
        else {  
            this.beanDefinitionNames.add(beanName);  
            this.frozenBeanDefinitionNames = null;  
        }  
        this.beanDefinitionMap.put(beanName, beanDefinition);  
        //重置所有已经注册过的BeanDefinition的缓存  
        resetBeanDefinition(beanName);  
    }  
}
```



### 小结

ioc初始化步骤：
1. 初始化的入口在容器实现中的 refresh() 调用来完成
2. 对 bean 定义载入IOC容器使用的方法是 laodBeanDefinition，大致过程：通过 ResouceLoader 来完成资源文件位置的定位，DefaultResouceLoader是默认实现

工厂（标准化输出产品），容器（HashMap）

### Bean factory 和 Factory bean

BeanFactory 指的是 IOC 容器的编程抽象，比如 ApplicationContext， XmlBeanFactory 等，这些都是 IOC 容器的具体表现，需要使用什么样的容器由客户决定,但 Spring 为我们提供了丰富的选择。 FactoryBean 只是一个可以在 IOC而容器中被管理的一个 bean,是对各种处理过程和资源使用的抽象,Factory bean 在需要时产生另一个对象，而不返回 FactoryBean本身,我们可以把它看成是一个抽象工厂，对它的调用返回的是工厂生产的产品。所有的 Factory bean 都实现特殊的org.springframework.beans.factory.FactoryBean 接口，当使用容器中 factory bean 的时候，该容器不会返回 factory bean 本身,而是返回其生成的对象。Spring 包括了大部分的通用资源和服务访问抽象的 Factory bean 的实现，其中包括:对 JNDI 查询的处理，对代理对象的处理，对事务性代理的处理，对 RMI 代理的处理等，这些都可以看成是具体的工厂。也就是说 Spring 通过使用抽象工厂模式为我们准备了一系列工厂来生产一些特定的对象,免除我们手工重复的工作，我们要使用时只需要在 IOC 容器里配置好就能很方便的使用了

# IOC容器的依赖注入
### 1. 依赖注入发生的时间

(1).用户第一次通过getBean方法向IoC容索要Bean时，IoC容器触发依赖注入。  
(2).当用户在Bean定义资源中为<Bean>元素配置了lazy-init属性，即让容器在解析注册Bean定义时进行预实例化，触发依赖注入。

```java
doGetBean(  
final String name, 
final Class<T> requiredType, 
final Object[] args,
boolean typeCheckOnly)
//合并父类公共属性问题 
final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName); 
//创建一个指定Bean实例对象，如果有父级继承，则合并子类和父类的（ObejctFactory接口的匿名内部类的createBean方法）
prototypeInstance = createBean(beanName, mbd, args);-->doCreateBean(beanName, mbd, args);
//获取给定Bean的实例对象  
bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd); 

```

根据配置scope来创建，类属性，注册依赖类，父类，再创建实例

### 2. AbstractBeanFactory通过getBean向IoC容器获取被管理的Bean

```java
//获取IOC容器中指定名称的Bean 
public Object getBean(String name) throws BeansException {
    //doGetBean才是真正向IoC容器获取被管理Bean的过程
    return doGetBean(name, null, null, false);
}

//获取IOC容器中指定名称和类型的Bean
public <T> T getBean(String name, Class<T> requiredType) throws BeansException {
    //doGetBean才是真正向IoC容器获取被管理Bean的过程
    return doGetBean(name, requiredType, null, false);
}

//获取IOC容器中指定名称和参数的Bean
public Object getBean(String name, Object... args) throws BeansException {
    //doGetBean才是真正向IoC容器获取被管理Bean的过程 
    return doGetBean(name, null, args, false);
}

//获取IOC容器中指定名称、类型和参数的Bean
public <T> T getBean(String name, Class<T> requiredType, Object... args) throws BeansException {
    //doGetBean才是真正向IoC容器获取被管理Bean的过程
    return doGetBean(name, requiredType, args, false);
}

//真正实现向IOC容器获取Bean的功能，也是触发依赖注入功能的地方
@SuppressWarnings("unchecked")
protected <T> T doGetBean(
    final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
    throws BeansException {

    //根据指定的名称获取被管理Bean的名称，剥离指定名称中对容器的相关依赖  
    //如果指定的是别名，将别名转换为规范的Bean名称
    final String beanName = transformedBeanName(name);
    Object bean;

    // Eagerly check singleton cache for manually registered singletons.
    //先从缓存中取是否已经有被创建过的单态类型的Bean
    //对于单例模式的Bean整个IOC容器中只创建一次，不需要重复创建
    Object sharedInstance = getSingleton(beanName);
    //IOC容器创建单例模式Bean实例对象
    if (sharedInstance != null && args == null) {
        if (logger.isDebugEnabled()) {
            //如果指定名称的Bean在容器中已有单例模式的Bean被创建
            //直接返回已经创建的Bean
            if (isSingletonCurrentlyInCreation(beanName)) {
                logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
                             "' that is not fully initialized yet - a consequence of a circular reference");
            }
            else {
                logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
            }
        }
        //获取给定Bean的实例对象，主要是完成FactoryBean的相关处理  
        //注意：BeanFactory是管理容器中Bean的工厂，而FactoryBean是  
        //创建创建对象的工厂Bean，两者之间有区别
        bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
    }
    else {
        // Fail if we're already creating this bean instance:
        // We're assumably within a circular reference.
        //缓存没有正在创建的单例模式Bean  
        //缓存中已经有已经创建的原型模式Bean
        //但是由于循环引用的问题导致实 例化对象失败
        if (isPrototypeCurrentlyInCreation(beanName)) {
            throw new BeanCurrentlyInCreationException(beanName);
        }
        // Check if bean definition exists in this factory.
        //对IOC容器中是否存在指定名称的BeanDefinition进行检查，首先检查是否  
        //能在当前的BeanFactory中获取的所需要的Bean，如果不能则委托当前容器  
        //的父级容器去查找，如果还是找不到则沿着容器的继承体系向父级容器查找
        BeanFactory parentBeanFactory = getParentBeanFactory();
        //当前容器的父级容器存在，且当前容器中不存在指定名称的Bean
        if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
            // Not found -> check parent.
            //解析指定Bean名称的原始名称
            String nameToLookup = originalBeanName(name);
            if (args != null) {
                // Delegation to parent with explicit args.
                //委派父级容器根据指定名称和显式的参数查找
                return (T) parentBeanFactory.getBean(nameToLookup, args);
            }
            else {
                // No args -> delegate to standard getBean method.
                //委派父级容器根据指定名称和类型查找
                return parentBeanFactory.getBean(nameToLookup, requiredType);
            }
        }
        //创建的Bean是否需要进行类型验证，一般不需要
        if (!typeCheckOnly) {
            //向容器标记指定的Bean已经被创建
            markBeanAsCreated(beanName);
        }
        try {
            //根据指定Bean名称获取其父级的Bean定义
            //主要解决Bean继承时子类合并父类公共属性问题 
            final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
            checkMergedBeanDefinition(mbd, beanName, args);
            // Guarantee initialization of beans that the current bean depends on.
            //获取当前Bean所有依赖Bean的名称
            String[] dependsOn = mbd.getDependsOn();
            //如果当前Bean有依赖Bean
            if (dependsOn != null) {
                for (String dependsOnBean : dependsOn) {
                    //递归调用getBean方法，获取当前Bean的依赖Bean
                    getBean(dependsOnBean);
                    //把被依赖Bean注册给当前依赖的Bean
                    registerDependentBean(dependsOnBean, beanName);
                }
            }
            // Create bean instance.
            //创建单例模式Bean的实例对象
            if (mbd.isSingleton()) {
                //这里使用了一个匿名内部类，创建Bean实例对象，并且注册给所依赖的对象
                sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
                    public Object getObject() throws BeansException {
                        try {
                            //创建一个指定Bean实例对象，如果有父级继承，则合并子类和父类的定义
                            return createBean(beanName, mbd, args);
                        }
                        catch (BeansException ex) {
                            // Explicitly remove instance from singleton cache: It might have been put there
                            // eagerly by the creation process, to allow for circular reference resolution.
                            // Also remove any beans that received a temporary reference to the bean.
                            //显式地从容器单例模式Bean缓存中清除实例对象
                            destroySingleton(beanName);
                            throw ex;
                        }
                    }
                });
                //获取给定Bean的实例对象
                bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
            }

            //IOC容器创建原型模式Bean实例对象
            else if (mbd.isPrototype()) {
                // It's a prototype -> create a new instance.
                //原型模式(Prototype)是每次都会创建一个新的对象
                Object prototypeInstance = null;
                try {
                    //回调beforePrototypeCreation方法，默认的功能是注册当前创建的原型对象
                    beforePrototypeCreation(beanName);
                    //创建指定Bean对象实例
                    prototypeInstance = createBean(beanName, mbd, args);
                }
                finally {
                    //回调afterPrototypeCreation方法，默认的功能告诉IOC容器指定Bean的原型对象不再创建了
                    afterPrototypeCreation(beanName);
                }
                //获取给定Bean的实例对象
                bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
            }
            //要创建的Bean既不是单例模式，也不是原型模式，则根据Bean定义资源中  
            //配置的生命周期范围，选择实例化Bean的合适方法，这种在Web应用程序中  
            //比较常用，如：request、session、application等生命周期 
            else {
                String scopeName = mbd.getScope();
                final Scope scope = this.scopes.get(scopeName);
                //Bean定义资源中没有配置生命周期范围，则Bean定义不合法
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope '" + scopeName + "'");
                }
                try {
                    //这里又使用了一个匿名内部类，获取一个指定生命周期范围的实例
                    Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
                        public Object getObject() throws BeansException {
                            beforePrototypeCreation(beanName);
                            try {
                                return createBean(beanName, mbd, args);
                            }
                            finally {
                                afterPrototypeCreation(beanName);
                            }
                        }
                    });
                    //获取给定Bean的实例对象
                    bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
                }
                catch (IllegalStateException ex) {
                    throw new BeanCreationException(beanName,
                                                    "Scope '" + scopeName + "' is not active for the current thread; " +
                                                    "consider defining a scoped proxy for this bean if you intend to refer to it from a singleton",
                                                    ex);
                }
            }
        }
        catch (BeansException ex) {
            cleanupAfterBeanCreationFailure(beanName);
            throw ex;
        }
    }

    // Check if required type matches the type of the actual bean instance.
    //对创建的Bean实例对象进行类型检查
    if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
        try {
            return getTypeConverter().convertIfNecessary(bean, requiredType);
        }
        catch (TypeMismatchException ex) {
            if (logger.isDebugEnabled()) {
                logger.debug("Failed to convert bean '" + name + "' to required type [" +
                             ClassUtils.getQualifiedName(requiredType) + "]", ex);
            }
            throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
        }
    }
    return (T) bean;
}
```

### 3. AbstractAutowireCapableBeanFactory创建Bean实例对象

//将Bean实例对象封装，并且Bean定义中配置的属性值赋值给实例对象  
createBeanInstance  创建
populateBean(beanName, mbd, instanceWrapper) 注入
initializeBean(beanName, exposedObject, mbd) 无参构造  

```java
//创建Bean实例对象  
protected Object createBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)  
    throws BeanCreationException {  
    if (logger.isDebugEnabled()) {  
        logger.debug("Creating instance of bean '" + beanName + "'");  
    }  
    //判断需要创建的Bean是否可以实例化，即是否可以通过当前的类加载器加载  
    resolveBeanClass(mbd, beanName);  
    //校验和准备Bean中的方法覆盖  
    try {  
        mbd.prepareMethodOverrides();  
    }  
    catch (BeanDefinitionValidationException ex) {  
        throw new BeanDefinitionStoreException(mbd.getResourceDescription(),  
                                               beanName, "Validation of method overrides failed", ex);  
    }  
    try {  
        //如果Bean配置了初始化前和初始化后的处理器，则试图返回一个需要创建//Bean的代理对象  
        Object bean = resolveBeforeInstantiation(beanName, mbd);  
        if (bean != null) {  
            return bean;  
        }  
    }  
    catch (Throwable ex) {  
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,  
                                        "BeanPostProcessor before instantiation of bean failed", ex);  
    }  
    //创建Bean的入口  
    Object beanInstance = doCreateBean(beanName, mbd, args);  
    if (logger.isDebugEnabled()) {  
        logger.debug("Finished creating instance of bean '" + beanName + "'");  
    }  
    return beanInstance;  
}


//真正创建Bean的方法  
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args) {  
    //封装被创建的Bean对象  
    BeanWrapper instanceWrapper = null;  
    if (mbd.isSingleton()){//单态模式的Bean，先从容器中缓存中获取同名Bean  
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);  
    }  
    if (instanceWrapper == null) {  
        //创建实例对象  
        instanceWrapper = createBeanInstance(beanName, mbd, args);  
    }  
    final Object bean = (instanceWrapper != null ? instanceWrapper.getWrappedInstance() : null);  
    //获取实例化对象的类型  
    Class beanType = (instanceWrapper != null ? instanceWrapper.getWrappedClass() : null);  
    //调用PostProcessor后置处理器  
    synchronized (mbd.postProcessingLock) {  
        if (!mbd.postProcessed) {  
            applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);  
            mbd.postProcessed = true;  
        }  
    }  
    // Eagerly cache singletons to be able to resolve circular references  
    //向容器中缓存单态模式的Bean对象，以防循环引用  
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&  
                                      isSingletonCurrentlyInCreation(beanName));  
    if (earlySingletonExposure) {  
        if (logger.isDebugEnabled()) {  
            logger.debug("Eagerly caching bean '" + beanName +  
                         "' to allow for resolving potential circular references");  
        }  
        //这里是一个匿名内部类，为了防止循环引用，尽早持有对象的引用  
        addSingletonFactory(beanName, new ObjectFactory() {  
            public Object getObject() throws BeansException {  
                return getEarlyBeanReference(beanName, mbd, bean);  
            }  
        });  
    }  
    //Bean对象的初始化，依赖注入在此触发  
    //这个exposedObject在初始化完成之后返回作为依赖注入完成后的Bean  
    Object exposedObject = bean;  
    try {  
        //将Bean实例对象封装，并且Bean定义中配置的属性值赋值给实例对象  
        populateBean(beanName, mbd, instanceWrapper);  
        if (exposedObject != null) {  
            //初始化Bean对象  
            exposedObject = initializeBean(beanName, exposedObject, mbd);  
        }  
    }  
    catch (Throwable ex) {  
        if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {  
            throw (BeanCreationException) ex;  
        }  
        else {  
            throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);  
        }  
    }  
    if (earlySingletonExposure) {  
        //获取指定名称的已注册的单态模式Bean对象  
        Object earlySingletonReference = getSingleton(beanName, false);  
        if (earlySingletonReference != null) {  
            //根据名称获取的以注册的Bean和正在实例化的Bean是同一个  
            if (exposedObject == bean) {  
                //当前实例化的Bean初始化完成  
                exposedObject = earlySingletonReference;  
            }  
            //当前Bean依赖其他Bean，并且当发生循环引用时不允许新创建实例对象  
            else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {  
                String[] dependentBeans = getDependentBeans(beanName);  
                Set<String> actualDependentBeans = new LinkedHashSet<String>(dependentBeans.length);  
                //获取当前Bean所依赖的其他Bean  
                for (String dependentBean : dependentBeans) {  
                    //对依赖Bean进行类型检查  
                    if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {  
                        actualDependentBeans.add(dependentBean);  
                    }  
                }  
                if (!actualDependentBeans.isEmpty()) {  
                    throw new BeanCurrentlyInCreationException(beanName,  
                                                               "Bean with name '" + beanName + "' has been injected into other beans [" +  
                                                               StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +  
                                                               "] in its raw version as part of a circular reference, but has eventually been " +  
                                                               "wrapped. This means that said other beans do not use the final version of the " +  
                                                               "bean. This is often the result of over-eager type matching - consider using " +  
                                                               "'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");  
                }  
            }  
        }  
    }  
    //注册完成依赖注入的Bean  
    try {  
        registerDisposableBeanIfNecessary(beanName, bean, mbd);  
    }  
    catch (BeanDefinitionValidationException ex) {  
        throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);  
    }  
    return exposedObject;  
}
```

### 4. createBeanInstance方法创建Bean的java实例对象

在createBeanInstance方法中，根据指定的初始化策略，使用静态工厂、工厂方法或者容器的自动装配特性生成java实例对象

```java
//创建Bean的实例对象
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, Object[] args) {
    // Make sure bean class is actually resolved at this point.
    //检查确认Bean是可实例化的
    Class<?> beanClass = resolveBeanClass(mbd, beanName);

    //使用工厂方法对Bean进行实例化
    if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName,
                                        "Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
    }

    if (mbd.getFactoryMethodName() != null)  {
        //调用工厂方法实例化
        return instantiateUsingFactoryMethod(beanName, mbd, args);
    }

    // Shortcut when re-creating the same bean...
    //使用容器的自动装配方法进行实例化
    boolean resolved = false;
    boolean autowireNecessary = false;
    if (args == null) {
        synchronized (mbd.constructorArgumentLock) {
            if (mbd.resolvedConstructorOrFactoryMethod != null) {
                resolved = true;
                autowireNecessary = mbd.constructorArgumentsResolved;
            }
        }
    }
    if (resolved) {
        if (autowireNecessary) {
            //配置了自动装配属性，使用容器的自动装配实例化  
            //容器的自动装配是根据参数类型匹配Bean的构造方法
            return autowireConstructor(beanName, mbd, null, null);
        }
        else {
            //使用默认的无参构造方法实例化
            return instantiateBean(beanName, mbd);
        }
    }

    // Need to determine the constructor...
    //使用Bean的构造方法进行实例化
    Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    if (ctors != null ||
        mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_CONSTRUCTOR ||
        mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args))  {
        //使用容器的自动装配特性，调用匹配的构造方法实例化 
        return autowireConstructor(beanName, mbd, ctors, args);
    }

    // No special handling: simply use no-arg constructor.
    //使用默认的无参构造方法实例化
    return instantiateBean(beanName, mbd);
}


//使用默认的无参构造方法实例化Bean对象
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
    try {
        Object beanInstance;
        final BeanFactory parent = this;
        //获取系统的安全管理接口，JDK标准的安全管理AP
        if (System.getSecurityManager() != null) {
            //这里是一个匿名内置类，根据实例化策略创建实例对象
            beanInstance = AccessController.doPrivileged(new PrivilegedAction<Object>() {
                public Object run() {
                    return getInstantiationStrategy().instantiate(mbd, beanName, parent);
                }
            }, getAccessControlContext());
        }
        else {
            //将实例化的对象封装起来
            beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
        }
        BeanWrapper bw = new BeanWrapperImpl(beanInstance);
        initBeanWrapper(bw);
        return bw;
    }
    catch (Throwable ex) {
        throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
    }
}
```

### 5. SimpleInstantiationStrategy类使用默认的无参构造方法创建Bean实例化对象

**实例化策略**
在使用默认的无参构造方法创建Bean的实例化对象时，方法 `getInstantiationStrategy().instantiate`调用了`SimpleInstantiationStrategy`类中的实例化Bean的方法  
使用反射机制获取Bean的构造方法  
//使用BeanUtils实例化，通过反射机制调用”构造方法.newInstance(arg)”来进行实例化  
//使用CGLIB来实例化对象
如果Bean有方法被覆盖了，则使用JDK的反射机制进行实例化，否则，使用CGLIB进行实例化。

```java
//使用初始化策略实例化Bean对象
public Object instantiate(RootBeanDefinition beanDefinition, String beanName, BeanFactory owner) {
    // Don't override the class with CGLIB if no overrides.
    //如果Bean定义中没有方法覆盖，则就不需要CGLIB父类类的方法
    if (beanDefinition.getMethodOverrides().isEmpty()) {
        Constructor<?> constructorToUse;
        synchronized (beanDefinition.constructorArgumentLock) {
            //获取对象的构造方法或工厂方法
            constructorToUse = (Constructor<?>) beanDefinition.resolvedConstructorOrFactoryMethod;

            //如果没有构造方法且没有工厂方法 
            if (constructorToUse == null) {
                //使用JDK的反射机制，判断要实例化的Bean是否是接口
                final Class clazz = beanDefinition.getBeanClass();
                if (clazz.isInterface()) {
                    throw new BeanInstantiationException(clazz, "Specified class is an interface");
                }
                try {
                    if (System.getSecurityManager() != null) {
                        //这里是一个匿名内置类，使用反射机制获取Bean的构造方法
                        constructorToUse = AccessController.doPrivileged(new PrivilegedExceptionAction<Constructor>() {
                            public Constructor run() throws Exception {
                                return clazz.getDeclaredConstructor((Class[]) null);
                            }
                        });
                    }
                    else {
                        constructorToUse =	clazz.getDeclaredConstructor((Class[]) null);
                    }
                    beanDefinition.resolvedConstructorOrFactoryMethod = constructorToUse;
                }
                catch (Exception ex) {
                    throw new BeanInstantiationException(clazz, "No default constructor found", ex);
                }
            }
        }
        //使用BeanUtils实例化，通过反射机制调用”构造方法.newInstance(arg)”来进行实例化
        return BeanUtils.instantiateClass(constructorToUse);
    }
    else {
        // Must generate CGLIB subclass.
        //使用CGLIB来实例化对象
        return instantiateWithMethodInjection(beanDefinition, beanName, owner);
    }
}
```

通过上面的代码分析，我们看到了如果Bean有方法被覆盖了，则使用JDK的反射机制进行实例化，否则，使用CGLIB进行实例化。
instantiateWithMethodInjection方法调用SimpleInstantiationStrategy的子类CglibSubclassingInstantiationStrategy使用CGLIB来进行初始化

* CglibSubclassingInstantiationStrategy

```java
//使用CGLIB进行Bean对象实例化
public Object instantiate(Constructor ctor, Object[] args) {
    //CGLIB中的类
    Enhancer enhancer = new Enhancer();
    //将Bean本身作为其基类
    enhancer.setSuperclass(this.beanDefinition.getBeanClass());
    enhancer.setCallbackFilter(new CallbackFilterImpl());
    enhancer.setCallbacks(new Callback[] {
        NoOp.INSTANCE,
        new LookupOverrideMethodInterceptor(),
        new ReplaceOverrideMethodInterceptor()
    });

    //使用CGLIB的create方法生成实例对象
    return (ctor == null) ?
        enhancer.create() :
    enhancer.create(ctor.getParameterTypes(), args);
}
```

疑问：Spring配置文件中的lookup-method和replace-method

1. 如果需要替换的方法没有返回值，那么只能使用replace-method来替换，而不能用lookup-method来替换。 
2. replace-method必须实现MethodReplacer接口的Bean才能替换，而lookup-method则由BeanFactory自动为我们处理了

### 6. AbstractAutowireCapableBeanFactory#populateBean 方法对Bean属性的依赖注入：

(1).createBeanInstance：生成Bean所包含的java对象实例。
(2).populateBean ：对Bean属性的依赖注入进行处理。
//从实例对象中提取属性描述符...
//使用BeanPostProcessor处理器处理属性值  
pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);    
对属性的注入过程分以下两种情况：   
(1).属性值类型不需要转换时，不需要解析属性值，直接准备进行依赖注入。  
(2).属性值需要进行类型转换时，如对其他对象的引用等，首先需要解析属性值，然后对解析后的属性值进行依赖注入  

```java
//将Bean属性设置到生成的实例对象上
protected void populateBean(String beanName, RootBeanDefinition mbd, BeanWrapper bw) {
    //获取容器在解析Bean定义资源时为BeanDefiniton中设置的属性值
    PropertyValues pvs = mbd.getPropertyValues();

    //实例对象为null
    if (bw == null) {
        if (!pvs.isEmpty()) {
            throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
        }
        else {
            // Skip property population phase for null instance.
            //实例对象为null，属性值也为空，不需要设置属性值，直接返回 
            return;
        }
    }

    // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
    // state of the bean before properties are set. This can be used, for example,
    // to support styles of field injection.
    //在设置属性之前调用Bean的PostProcessor后置处理器
    boolean continueWithPropertyPopulation = true;

    if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
        for (BeanPostProcessor bp : getBeanPostProcessors()) {
            if (bp instanceof InstantiationAwareBeanPostProcessor) {
                InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                    continueWithPropertyPopulation = false;
                    break;
                }
            }
        }
    }

    if (!continueWithPropertyPopulation) {
        return;
    }

    //依赖注入开始，首先处理autowire自动装配的注入
    if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||
        mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
        MutablePropertyValues newPvs = new MutablePropertyValues(pvs);

        // Add property values based on autowire by name if applicable.
        //对autowire自动装配的处理，根据Bean名称自动装配注入
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {
            autowireByName(beanName, mbd, bw, newPvs);
        }

        // Add property values based on autowire by type if applicable.
        //根据Bean类型自动装配注入
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {
            autowireByType(beanName, mbd, bw, newPvs);
        }

        pvs = newPvs;
    }

    //检查容器是否持有用于处理单例模式Bean关闭时的后置处理器
    boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
    //Bean实例对象没有依赖，即没有继承基类
    boolean needsDepCheck = (mbd.getDependencyCheck() != RootBeanDefinition.DEPENDENCY_CHECK_NONE);

    if (hasInstAwareBpps || needsDepCheck) {
        //从实例对象中提取属性描述符
        PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
        if (hasInstAwareBpps) {
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                    //使用BeanPostProcessor处理器处理属性值
                    pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                    if (pvs == null) {
                        return;
                    }
                }
            }
        }
        if (needsDepCheck) {
            //为要设置的属性进行依赖检查
            checkDependencies(beanName, mbd, filteredPds, pvs);
        }
    }
    //对属性进行注入
    applyPropertyValues(beanName, mbd, bw, pvs);
}
```

* applyPropertyValues

```java
///解析并注入依赖属性的过程
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
    if (pvs == null || pvs.isEmpty()) {
        return;
    }

    //封装属性值
    MutablePropertyValues mpvs = null;
    List<PropertyValue> original;

    if (System.getSecurityManager()!= null) {
        if (bw instanceof BeanWrapperImpl) {
            //设置安全上下文，JDK安全机制
            ((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
        }
    }

    if (pvs instanceof MutablePropertyValues) {
        mpvs = (MutablePropertyValues) pvs;
        //属性值已经转换
        if (mpvs.isConverted()) {
            // Shortcut: use the pre-converted values as-is.
            try {
                //为实例化对象设置属性值
                bw.setPropertyValues(mpvs);
                return;
            }
            catch (BeansException ex) {
                throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Error setting property values", ex);
            }
        }
        //获取属性值对象的原始类型值
        original = mpvs.getPropertyValueList();
    }
    else {
        original = Arrays.asList(pvs.getPropertyValues());
    }

    //获取用户自定义的类型转换
    TypeConverter converter = getCustomTypeConverter();
    if (converter == null) {
        converter = bw;
    }
    //创建一个Bean定义属性值解析器，将Bean定义中的属性值解析为Bean实例对象的实际值
    BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

    // Create a deep copy, resolving any references for values.
    //为属性的解析值创建一个拷贝，将拷贝的数据注入到实例对象中
    List<PropertyValue> deepCopy = new ArrayList<PropertyValue>(original.size());
    boolean resolveNecessary = false;
    for (PropertyValue pv : original) {
        //属性值不需要转换
        if (pv.isConverted()) {
            deepCopy.add(pv);
        }
        //属性值需要转换
        else {
            String propertyName = pv.getName();
            //原始的属性值，即转换之前的属性值
            Object originalValue = pv.getValue();
            //转换属性值，例如将引用转换为IoC容器中实例化对象引用
            Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
            //转换之后的属性值
            Object convertedValue = resolvedValue;
            //属性值是否可以转换
            boolean convertible = bw.isWritableProperty(propertyName) &&
                !PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
            if (convertible) {
                //使用用户自定义的类型转换器转换属性值
                convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
            }
            // Possibly store converted value in merged bean definition,
            // in order to avoid re-conversion for every created bean instance.
            //存储转换后的属性值，避免每次属性注入时的转换工作
            if (resolvedValue == originalValue) {
                if (convertible) {
                    //设置属性转换之后的值
                    pv.setConvertedValue(convertedValue);
                }
                deepCopy.add(pv);
            }
            //属性是可转换的，且属性原始值是字符串类型，且属性的原始类型值不是  
            //动态生成的字符串，且属性的原始值不是集合或者数组类型
            else if (convertible && originalValue instanceof TypedStringValue &&
                     !((TypedStringValue) originalValue).isDynamic() &&
                     !(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
                pv.setConvertedValue(convertedValue);
                deepCopy.add(pv);
            }
            else {
                resolveNecessary = true;
                //重新封装属性的值
                deepCopy.add(new PropertyValue(pv, convertedValue));
            }
        }
    }
    if (mpvs != null && !resolveNecessary) {
        //标记属性值已经转换过
        mpvs.setConverted();
    }

    // Set our (possibly massaged) deep copy.
    //进行属性依赖注入
    try {
        bw.setPropertyValues(new MutablePropertyValues(deepCopy));
    }
    catch (BeansException ex) {
        throw new BeanCreationException(
            mbd.getResourceDescription(), beanName, "Error setting property values", ex);
    }
}
```

1. 属性值类型不需要转换时，不需要解析属性值，直接准备进行依赖注入。

2. 属性值需要进行类型转换时，如对其他对象的引用等，首先需要解析属性值，然后对解析后的属性值进行依赖注入。

对属性值的解析是在BeanDefinitionValueResolver类中的resolveValueIfNecessary方法中进行的，对属性值的依赖注入是通过bw.setPropertyValues方法实现的，在分析属性值的依赖注入之前，我们先分析一下对属性值的解析过程。

#### 7. BeanDefinitionValueResolver解析属性值

属性进行解析的由resolveValueIfNecessary方法实现，针对属性类型    
依赖注入是通过bw.setPropertyValues方法实现的，该方法也使用了委托模式，在BeanWrapper接口中至少定义了方法声明，依赖注入的具体实现交由其实现类BeanWrapperImpl来完成

```java
//解析属性值，对注入类型进行转换
public Object resolveValueIfNecessary(Object argName, Object value) {
    // We must check each value to see whether it requires a runtime reference
    // to another bean to be resolved.
    //对引用类型的属性进行解析
    if (value instanceof RuntimeBeanReference) {
        RuntimeBeanReference ref = (RuntimeBeanReference) value;
        //调用引用类型属性的解析方法
        return resolveReference(argName, ref);
    }
    //对属性值是引用容器中另一个Bean名称的解析
    else if (value instanceof RuntimeBeanNameReference) {
        String refName = ((RuntimeBeanNameReference) value).getBeanName();
        refName = String.valueOf(evaluate(refName));
        //从容器中获取指定名称的Bean
        if (!this.beanFactory.containsBean(refName)) {
            throw new BeanDefinitionStoreException(
                "Invalid bean name '" + refName + "' in bean reference for " + argName);
        }
        return refName;
    }
    //对Bean类型属性的解析，主要是Bean中的内部类
    else if (value instanceof BeanDefinitionHolder) {
        // Resolve BeanDefinitionHolder: contains BeanDefinition with name and aliases.
        BeanDefinitionHolder bdHolder = (BeanDefinitionHolder) value;
        return resolveInnerBean(argName, bdHolder.getBeanName(), bdHolder.getBeanDefinition());
    }
    else if (value instanceof BeanDefinition) {
        // Resolve plain BeanDefinition, without contained name: use dummy name.
        BeanDefinition bd = (BeanDefinition) value;
        return resolveInnerBean(argName, "(inner bean)", bd);
    }
    //对集合数组类型的属性解析
    else if (value instanceof ManagedArray) {
        // May need to resolve contained runtime references.
        ManagedArray array = (ManagedArray) value;
        //获取数组的类型
        Class<?> elementType = array.resolvedElementType;
        if (elementType == null) {
            //获取数组元素的类型
            String elementTypeName = array.getElementTypeName();
            if (StringUtils.hasText(elementTypeName)) {
                try {
                    //使用反射机制创建指定类型的对象
                    elementType = ClassUtils.forName(elementTypeName, this.beanFactory.getBeanClassLoader());
                    array.resolvedElementType = elementType;
                }
                catch (Throwable ex) {
                    // Improve the message by showing the context.
                    throw new BeanCreationException(
                        this.beanDefinition.getResourceDescription(), this.beanName,
                        "Error resolving array type for " + argName, ex);
                }
            }
            //没有获取到数组的类型，也没有获取到数组元素的类型 
            //则直接设置数组的类型为Object
            else {
                elementType = Object.class;
            }
        }
        //创建指定类型的数组
        return resolveManagedArray(argName, (List<?>) value, elementType);
    }
    //解析list类型的属性值
    else if (value instanceof ManagedList) {
        // May need to resolve contained runtime references.
        return resolveManagedList(argName, (List<?>) value);
    }
    //解析set类型的属性值
    else if (value instanceof ManagedSet) {
        // May need to resolve contained runtime references.
        return resolveManagedSet(argName, (Set<?>) value);
    }
    //解析map类型的属性值
    else if (value instanceof ManagedMap) {
        // May need to resolve contained runtime references.
        return resolveManagedMap(argName, (Map<?, ?>) value);
    }
    //解析props类型的属性值，props其实就是key和value均为字符串的map
    else if (value instanceof ManagedProperties) {
        Properties original = (Properties) value;
        //创建一个拷贝，用于作为解析后的返回值
        Properties copy = new Properties();
        for (Map.Entry propEntry : original.entrySet()) {
            Object propKey = propEntry.getKey();
            Object propValue = propEntry.getValue();
            if (propKey instanceof TypedStringValue) {
                propKey = evaluate((TypedStringValue) propKey);
            }
            if (propValue instanceof TypedStringValue) {
                propValue = evaluate((TypedStringValue) propValue);
            }
            copy.put(propKey, propValue);
        }
        return copy;
    }
    //解析字符串类型的属性值
    else if (value instanceof TypedStringValue) {
        // Convert value to target type here.
        TypedStringValue typedStringValue = (TypedStringValue) value;
        Object valueObject = evaluate(typedStringValue);
        try {
            //获取属性的目标类型
            Class<?> resolvedTargetType = resolveTargetType(typedStringValue);
            if (resolvedTargetType != null) {
                //对目标类型的属性进行解析，递归调用
                return this.typeConverter.convertIfNecessary(valueObject, resolvedTargetType);
            }
            //没有获取到属性的目标对象，则按Object类型返回
            else {
                return valueObject;
            }
        }
        catch (Throwable ex) {
            // Improve the message by showing the context.
            throw new BeanCreationException(
                this.beanDefinition.getResourceDescription(), this.beanName,
                "Error converting typed String value for " + argName, ex);
        }
    }
    else {
        return evaluate(value);
    }
}  

//解析引用类型的属性值
private Object resolveReference(Object argName, RuntimeBeanReference ref) {
    try {
        //获取引用的Bean名称
        String refName = ref.getBeanName();
        refName = String.valueOf(evaluate(refName));
        //如果引用的对象在父类容器中，则从父类容器中获取指定的引用对象
        if (ref.isToParent()) {
            if (this.beanFactory.getParentBeanFactory() == null) {
                throw new BeanCreationException(
                    this.beanDefinition.getResourceDescription(), this.beanName,
                    "Can't resolve reference to bean '" + refName +
                    "' in parent factory: no parent factory available");
            }
            return this.beanFactory.getParentBeanFactory().getBean(refName);
        }
        //从当前的容器中获取指定的引用Bean对象，如果指定的Bean没有被实例化  
        //则会递归触发引用Bean的初始化和依赖注入
        else {
            Object bean = this.beanFactory.getBean(refName);
            //将当前实例化对象的依赖引用对象
            this.beanFactory.registerDependentBean(refName, this.beanName);
            return bean;
        }
    }
    catch (BeansException ex) {
        throw new BeanCreationException(
            this.beanDefinition.getResourceDescription(), this.beanName,
            "Cannot resolve reference to bean '" + ref.getBeanName() + "' while setting " + argName, ex);
    }
}

//解析array类型的属性
private Object resolveManagedArray(Object argName, List<?> ml, Class<?> elementType) {
    //创建一个指定类型的数组，用于存放和返回解析后的数组
    Object resolved = Array.newInstance(elementType, ml.size());
    for (int i = 0; i < ml.size(); i++) {
        //递归解析array的每一个元素，并将解析后的值设置到resolved数组中，索引为i
        Array.set(resolved, i,
                  resolveValueIfNecessary(new KeyedArgName(argName, i), ml.get(i)));
    }
    return resolved;
}

//解析list类型的属性
private List resolveManagedList(Object argName, List<?> ml) {
    List<Object> resolved = new ArrayList<Object>(ml.size());
    for (int i = 0; i < ml.size(); i++) {
        //递归解析list的每一个元素
        resolved.add(
            resolveValueIfNecessary(new KeyedArgName(argName, i), ml.get(i)));
    }
    return resolved;
}


//解析set类型的属性
private Set resolveManagedSet(Object argName, Set<?> ms) {
    Set<Object> resolved = new LinkedHashSet<Object>(ms.size());
    int i = 0;
    //递归解析set的每一个元素
    for (Object m : ms) {
        resolved.add(resolveValueIfNecessary(new KeyedArgName(argName, i), m));
        i++;
    }
    return resolved;
}

//解析map类型的属性
private Map resolveManagedMap(Object argName, Map<?, ?> mm) {
    Map<Object, Object> resolved = new LinkedHashMap<Object, Object>(mm.size());
    //递归解析map中每一个元素的key和value
    for (Map.Entry entry : mm.entrySet()) {
        Object resolvedKey = resolveValueIfNecessary(argName, entry.getKey());
        Object resolvedValue = resolveValueIfNecessary(
            new KeyedArgName(argName, entry.getKey()), entry.getValue());
        resolved.put(resolvedKey, resolvedValue);
    }
    return resolved;
}
```

(1).对于集合类型的属性，将其属性值解析为目标类型的集合后直接赋值给属性。  
(2).对于非集合类型的属性，大量使用了JDK的反射和内省机制，通过属性的getter方法(reader method)获取指定属性注入以前的值，同时调用属性的setter方法(writer method)为属性设置注入后的值。看到这里相信很多人都明白了Spring的setter注入原理。

### 8. BeanWrapperImpl对Bean属性的依赖注入

setAccessible

BeanWrapperImpl类主要是对容器中完成初始化的Bean实例对象进行属性的依赖注入，即把Bean对象设置到它所依赖的另一个Bean的属性中去

```java
//实现属性依赖注入功能
@SuppressWarnings("unchecked")
private void setPropertyValue(PropertyTokenHolder tokens, PropertyValue pv) throws BeansException {
    //PropertyTokenHolder主要保存属性的名称、路径，以及集合的size等信息
    String propertyName = tokens.canonicalName;
    String actualName = tokens.actualName;

    //keys是用来保存集合类型属性的size
    if (tokens.keys != null) {
        // Apply indexes and map keys: fetch value for all keys but the last one.
        //将属性信息拷贝
        PropertyTokenHolder getterTokens = new PropertyTokenHolder();
        getterTokens.canonicalName = tokens.canonicalName;
        getterTokens.actualName = tokens.actualName;
        getterTokens.keys = new String[tokens.keys.length - 1];
        System.arraycopy(tokens.keys, 0, getterTokens.keys, 0, tokens.keys.length - 1);
        Object propValue;
        try {
            //获取属性值，该方法内部使用JDK的内省(Introspector)机制
            //调用属性的getter(readerMethod)方法，获取属性的值 
            propValue = getPropertyValue(getterTokens);
        }
        catch (NotReadablePropertyException ex) {
            throw new NotWritablePropertyException(getRootClass(), this.nestedPath + propertyName,
                                                   "Cannot access indexed value in property referenced " +
                                                   "in indexed property path '" + propertyName + "'", ex);
        }
        // Set value for last key.
        //获取集合类型属性的长度
        String key = tokens.keys[tokens.keys.length - 1];
        if (propValue == null) {
            // null map value case
            if (this.autoGrowNestedPaths) {
                // TODO: cleanup, this is pretty hacky
                int lastKeyIndex = tokens.canonicalName.lastIndexOf('[');
                getterTokens.canonicalName = tokens.canonicalName.substring(0, lastKeyIndex);
                propValue = setDefaultValue(getterTokens);
            }
            else {
                throw new NullValueInNestedPathException(getRootClass(), this.nestedPath + propertyName,
                                                         "Cannot access indexed value in property referenced " +
                                                         "in indexed property path '" + propertyName + "': returned null");
            }
        }
        //注入array类型的属性值
        if (propValue.getClass().isArray()) {
            //获取属性的描述符
            PropertyDescriptor pd = getCachedIntrospectionResults().getPropertyDescriptor(actualName);
            //获取数组的类型
            Class requiredType = propValue.getClass().getComponentType();
            //获取数组的长度
            int arrayIndex = Integer.parseInt(key);
            Object oldValue = null;
            try {
                //获取数组以前初始化的值
                if (isExtractOldValueForEditor() && arrayIndex < Array.getLength(propValue)) {
                    oldValue = Array.get(propValue, arrayIndex);
                }
                //将属性的值赋值给数组中的元素
                Object convertedValue = convertIfNecessary(propertyName, oldValue, pv.getValue(),
                                                           requiredType, TypeDescriptor.nested(property(pd), tokens.keys.length));
                Array.set(propValue, arrayIndex, convertedValue);
            }
            catch (IndexOutOfBoundsException ex) {
                throw new InvalidPropertyException(getRootClass(), this.nestedPath + propertyName,
                                                   "Invalid array index in property path '" + propertyName + "'", ex);
            }
        }
        //注入list类型的属性值
        else if (propValue instanceof List) {
            PropertyDescriptor pd = getCachedIntrospectionResults().getPropertyDescriptor(actualName);
            //获取list集合的类型
            Class requiredType = GenericCollectionTypeResolver.getCollectionReturnType(
                pd.getReadMethod(), tokens.keys.length);
            List list = (List) propValue;

            int index = Integer.parseInt(key);
            Object oldValue = null;
            if (isExtractOldValueForEditor() && index < list.size()) {
                oldValue = list.get(index);
            }
            //获取list解析后的属性值
            Object convertedValue = convertIfNecessary(propertyName, oldValue, pv.getValue(),
                                                       requiredType, TypeDescriptor.nested(property(pd), tokens.keys.length));
            //获取list集合的size
            int size = list.size();
            //如果list的长度大于属性值的长度，则多余的元素赋值为null 
            if (index >= size && index < this.autoGrowCollectionLimit) {
                for (int i = size; i < index; i++) {
                    try {
                        list.add(null);
                    }
                    catch (NullPointerException ex) {
                        throw new InvalidPropertyException(getRootClass(), this.nestedPath + propertyName,
                                                           "Cannot set element with index " + index + " in List of size " +
                                                           size + ", accessed using property path '" + propertyName +
                                                           "': List does not support filling up gaps with null elements");
                    }
                }
                list.add(convertedValue);
            }
            else {
                try {
                    //为list属性赋值
                    list.set(index, convertedValue);
                }
                catch (IndexOutOfBoundsException ex) {
                    throw new InvalidPropertyException(getRootClass(), this.nestedPath + propertyName,
                                                       "Invalid list index in property path '" + propertyName + "'", ex);
                }
            }
        }
        //注入map类型的属性值
        else if (propValue instanceof Map) {
            PropertyDescriptor pd = getCachedIntrospectionResults().getPropertyDescriptor(actualName);
            //获取map集合key的类型
            Class mapKeyType = GenericCollectionTypeResolver.getMapKeyReturnType(
                pd.getReadMethod(), tokens.keys.length);
            //获取map集合value的类型
            Class mapValueType = GenericCollectionTypeResolver.getMapValueReturnType(
                pd.getReadMethod(), tokens.keys.length);
            Map map = (Map) propValue;
            // IMPORTANT: Do not pass full property name in here - property editors
            // must not kick in for map keys but rather only for map values.
            TypeDescriptor typeDescriptor = (mapKeyType != null ?
                                             TypeDescriptor.valueOf(mapKeyType) : TypeDescriptor.valueOf(Object.class));
            //解析map类型属性key值
            Object convertedMapKey = convertIfNecessary(null, null, key, mapKeyType, typeDescriptor);
            Object oldValue = null;
            if (isExtractOldValueForEditor()) {
                oldValue = map.get(convertedMapKey);
            }
            // Pass full property name and old value in here, since we want full
            // conversion ability for map values.
            //解析map类型属性value值
            Object convertedMapValue = convertIfNecessary(propertyName, oldValue, pv.getValue(),
                                                          mapValueType, TypeDescriptor.nested(property(pd), tokens.keys.length));
            //将解析后的key和value值赋值给map集合属性
            map.put(convertedMapKey, convertedMapValue);
        }
        //对非集合类型的属性注入
        else {
            throw new InvalidPropertyException(getRootClass(), this.nestedPath + propertyName,
                                               "Property referenced in indexed property path '" + propertyName +
                                               "' is neither an array nor a List nor a Map; returned value was [" + pv.getValue() + "]");
        }
    }

    else {
        PropertyDescriptor pd = pv.resolvedDescriptor;
        if (pd == null || !pd.getWriteMethod().getDeclaringClass().isInstance(this.object)) {
            pd = getCachedIntrospectionResults().getPropertyDescriptor(actualName);
            //无法获取到属性名或者属性没有提供setter(写方法)方法
            if (pd == null || pd.getWriteMethod() == null) {
                //如果属性值是可选的，即不是必须的，则忽略该属性值
                if (pv.isOptional()) {
                    logger.debug("Ignoring optional value for property '" + actualName +
                                 "' - property not found on bean class [" + getRootClass().getName() + "]");
                    return;
                }
                //如果属性值是必须的，则抛出无法给属性赋值，因为没提供setter方法异常
                else {
                    PropertyMatches matches = PropertyMatches.forProperty(propertyName, getRootClass());
                    throw new NotWritablePropertyException(
                        getRootClass(), this.nestedPath + propertyName,
                        matches.buildErrorMessage(), matches.getPossibleMatches());
                }
            }
            pv.getOriginalPropertyValue().resolvedDescriptor = pd;
        }

        Object oldValue = null;
        try {
            Object originalValue = pv.getValue();
            Object valueToApply = originalValue;
            if (!Boolean.FALSE.equals(pv.conversionNecessary)) {
                if (pv.isConverted()) {
                    valueToApply = pv.getConvertedValue();
                }
                else {
                    if (isExtractOldValueForEditor() && pd.getReadMethod() != null) {
                        //获取属性的getter方法(读方法)，JDK内省机制
                        final Method readMethod = pd.getReadMethod();
                        //如果属性的getter方法不是public访问控制权限的，即访问控制权限比较严格，  
                        //则使用JDK的反射机制强行访问非public的方法(暴力读取属性值) 
                        if (!Modifier.isPublic(readMethod.getDeclaringClass().getModifiers()) &&
                            !readMethod.isAccessible()) {
                            if (System.getSecurityManager()!= null) {
                                //匿名内部类，根据权限修改属性的读取控制限制 
                                AccessController.doPrivileged(new PrivilegedAction<Object>() {
                                    public Object run() {
                                        readMethod.setAccessible(true);
                                        return null;
                                    }
                                });
                            }
                            else {
                                readMethod.setAccessible(true);
                            }
                        }
                        try {
                            //属性没有提供getter方法时，调用潜在的读取属性值的方法，获取属性值 
                            if (System.getSecurityManager() != null) {
                                oldValue = AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
                                    public Object run() throws Exception {
                                        return readMethod.invoke(object);
                                    }
                                }, acc);
                            }
                            else {
                                oldValue = readMethod.invoke(object);
                            }
                        }
                        catch (Exception ex) {
                            if (ex instanceof PrivilegedActionException) {
                                ex = ((PrivilegedActionException) ex).getException();
                            }
                            if (logger.isDebugEnabled()) {
                                logger.debug("Could not read previous value of property '" +
                                             this.nestedPath + propertyName + "'", ex);
                            }
                        }
                    }
                    //设置属性的注入值
                    valueToApply = convertForProperty(propertyName, oldValue, originalValue, pd);
                }
                pv.getOriginalPropertyValue().conversionNecessary = (valueToApply != originalValue);
            }
            //根据JDK的内省机制，获取属性的setter(写方法)方法
            final Method writeMethod = (pd instanceof GenericTypeAwarePropertyDescriptor ?
                                        ((GenericTypeAwarePropertyDescriptor) pd).getWriteMethodForActualAccess() :
                                        pd.getWriteMethod());
            //如果属性的setter方法是非public，即访问控制权限比较严格，则使用JDK的反射机制，  
            //强行设置setter方法可访问(暴力为属性赋值)  
            if (!Modifier.isPublic(writeMethod.getDeclaringClass().getModifiers()) && !writeMethod.isAccessible()) {
                if (System.getSecurityManager()!= null) {
                    AccessController.doPrivileged(new PrivilegedAction<Object>() {
                        public Object run() {
                            writeMethod.setAccessible(true);
                            return null;
                        }
                    });
                }
                else {
                    writeMethod.setAccessible(true);
                }
            }
            final Object value = valueToApply;
            //如果使用了JDK的安全机制，则需要权限验证 
            if (System.getSecurityManager() != null) {
                try {
                    //将属性值设置到属性上去 
                    AccessController.doPrivileged(new PrivilegedExceptionAction<Object>() {
                        public Object run() throws Exception {
                            writeMethod.invoke(object, value);
                            return null;
                        }
                    }, acc);
                }
                catch (PrivilegedActionException ex) {
                    throw ex.getException();
                }
            }
            else {
                writeMethod.invoke(this.object, value);
            }
        }
        catch (TypeMismatchException ex) {
            throw ex;
        }
        catch (InvocationTargetException ex) {
            PropertyChangeEvent propertyChangeEvent =
                new PropertyChangeEvent(this.rootObject, this.nestedPath + propertyName, oldValue, pv.getValue());
            if (ex.getTargetException() instanceof ClassCastException) {
                throw new TypeMismatchException(propertyChangeEvent, pd.getPropertyType(), ex.getTargetException());
            }
            else {
                throw new MethodInvocationException(propertyChangeEvent, ex.getTargetException());
            }
        }
        catch (Exception ex) {
            PropertyChangeEvent pce =
                new PropertyChangeEvent(this.rootObject, this.nestedPath + propertyName, oldValue, pv.getValue());
            throw new MethodInvocationException(pce, ex);
        }
    }
}
```



### 小结

第一步：读取BeanDefinition中信息，获取其依赖关系  
第二部：实例化（代理对象）
注入：设值

# IoC容器的高级特性

Spring IoC容器的lazy-init属性实现预实例化  
IoC容器的初始化过程就是对Bean定义资源的定位、载入和注册，此时容器对Bean的依赖注入并没有发生，依赖注入主要是在应用程序第一次向容器索取Bean时，通过getBean方法的调用完成。

## Spring IoC容器的lazy-init属性实现预实例化：

(1).refresh()  
(2).finishBeanFactoryInitialization处理预实例化Bean  
(3)DefaultListableBeanFactory对配置lazy-init属性单态Bean的预实例化

## FactoryBean的实现

(1).FactoryBean的源码
(2). AbstractBeanFactory的getBean方法调用FactoryBean：
(3).AbstractBeanFactory生产Bean实例对象：
(4).工厂Bean的实现类getObject方法创建Bean实例对象

## BeanPostProcessor后置处理器的实现

(1).BeanPostProcessor的源码  
(2).AbstractAutowireCapableBeanFactory类对容器生成的Bean添加后置处理器  
(3).initializeBean方法为容器产生的Bean实例对象添加BeanPostProcessor后置处理器  
(4).AdvisorAdapterRegistrationManager在Bean对象初始化后注册通知适配器：  

## AbstractAutoWireCapableBeanFactory类对容器生成的Bean 添加后置处理器

initializeBean   

BeanPostProcessor => AdvisorAdaptorRegistrationManager在Bean对象初始化后注册通知适配器
触发器 Trigger

```java
//调用BeanPostProcessor后置处理器实例对象初始化之后的处理方法  
public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
    throws BeansException {

    Object result = existingBean;
    //遍历容器为所创建的Bean添加的所有BeanPostProcessor后置处理器
    for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
        //调用Bean实例所有的后置处理中的初始化后处理方法，为Bean实例对象在  
        //初始化之后做一些自定义的处理操作
        result = beanProcessor.postProcessBeforeInitialization(result, beanName);
        if (result == null) {
            return result;
        }
    }
    return result;
}

//调用BeanPostProcessor后置处理器实例对象初始化之前的处理方法  
public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
    throws BeansException {

    Object result = existingBean;
    //遍历容器为所创建的Bean添加的所有BeanPostProcessor后置处理器
    for (BeanPostProcessor beanProcessor : getBeanPostProcessors()) {
        //调用Bean实例所有的后置处理中的初始化前处理方法，为Bean实例对象在  
        //初始化之前做一些自定义的处理操作
        result = beanProcessor.postProcessAfterInitialization(result, beanName);
        if (result == null) {
            return result;
        }
    }
    return result;
}
```



## Spring IoC容器autowiring实现原理：（自动注入）

* Spring IoC容器提供了两种管理Bean依赖关系的方式：  

a.  显式管理：通过BeanDefinition的属性值和构造方法实现Bean依赖关系管理。  
b． autowiring：Spring IoC容器的依赖自动装配功能，不需要对Bean属性的依赖关系做显式的声明，只需要在配置好autowiring属性，IoC容器会自动使用反射查找属性的类型和名称，然后基于属性的类型或者名称来自动匹配容器中管理的Bean，从而自动地完成依赖注入。  

容器对Bean实例对象的属性注入的处理发生在**AbstractAutoWireCapableBeanFactory**类中的populateBean方法中

### 1. AbstractAutoWireCapableBeanFactory对Bean实例进行属性依赖注入：

应用第一次通过 getBean 方法(配置了lazy-init预实例化属性的除外) 向IoC容器索取Bean时，容器创建 Bean 实例对象，并且对Bean实例对象进行属性依赖注入，AbstractAutoWireCapableBeanFactory的 populateBean 方法就是实现Bean属性依赖注入的功能

```java
protected void populateBean(String beanName, AbstractBeanDefinition mbd, BeanWrapper bw) {  
    //获取Bean定义的属性值，并对属性值进行处理  
    PropertyValues pvs = mbd.getPropertyValues();  
    ……  
        //对依赖注入处理，首先处理autowiring自动装配的依赖注入  
        if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME ||  
            mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {  
            MutablePropertyValues newPvs = new MutablePropertyValues(pvs);  
            //根据Bean名称进行autowiring自动装配处理  
            if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_NAME) {  
                autowireByName(beanName, mbd, bw, newPvs);  
            }  
            //根据Bean类型进行autowiring自动装配处理  
            if (mbd.getResolvedAutowireMode() == RootBeanDefinition.AUTOWIRE_BY_TYPE) {  
                autowireByType(beanName, mbd, bw, newPvs);  
            }  
        }  
    //对非autowiring的属性进行依赖注入处理  
    ……  
}
```



### 2. Spring IoC容器根据Bean名称或者类型进行autowiring自动依赖注入：(声明为接口，自动找到实现类，接口只有一个实现)

```java
//根据名称对属性进行自动依赖注入
protected void autowireByName(
    String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

    //对Bean对象中非简单属性(不是简单继承的对象，如8中原始类型，字符串，URL等都是简单属性)进行处理  
    String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
    for (String propertyName : propertyNames) {
        //如果Spring IOC容器中包含指定名称的Bean
        if (containsBean(propertyName)) {
            //调用getBean方法向IoC容器索取指定名称的Bean实例，迭代触发属性的初始化和依赖注入 
            Object bean = getBean(propertyName);
            //为指定名称的属性赋予属性值
            pvs.add(propertyName, bean);
            //指定名称属性注册依赖Bean名称，进行属性依赖注入
            registerDependentBean(propertyName, beanName);
            if (logger.isDebugEnabled()) {
                logger.debug("Added autowiring by name from bean name '" + beanName +
                             "' via property '" + propertyName + "' to bean named '" + propertyName + "'");
            }
        }
        else {
            if (logger.isTraceEnabled()) {
                logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
                             "' by name: no matching bean found");
            }
        }
    }
}


//根据类型对属性进行自动依赖注入
protected void autowireByType(
    String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

    //获取用户定义的类型转换器 
    TypeConverter converter = getCustomTypeConverter();
    if (converter == null) {
        converter = bw;
    }

    //存放解析的要注入的属性
    Set<String> autowiredBeanNames = new LinkedHashSet<String>(4);
    //对Bean对象中非简单属性(不是简单继承的对象，如8中原始类型，字符  
    //URL等都是简单属性)进行处理 
    String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
    for (String propertyName : propertyNames) {
        try {
            //获取指定属性名称的属性描述器 
            PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
            // Don't try autowiring by type for type Object: never makes sense,
            // even if it technically is a unsatisfied, non-simple property.
            //不对Object类型的属性进行autowiring自动依赖注入  
            if (!Object.class.equals(pd.getPropertyType())) {
                //获取属性的setter方法
                MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
                // Do not allow eager init for type matching in case of a prioritized post-processor.
                //检查指定类型是否可以被转换为目标对象的类型 
                boolean eager = !PriorityOrdered.class.isAssignableFrom(bw.getWrappedClass());
                //创建一个要被注入的依赖描述 
                DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
                //根据容器的Bean定义解析依赖关系，返回所有要被注入的Bean对象  
                Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
                if (autowiredArgument != null) {
                    //为属性赋值所引用的对象  
                    pvs.add(propertyName, autowiredArgument);
                }
                for (String autowiredBeanName : autowiredBeanNames) {
                    //指定名称属性注册依赖Bean名称，进行属性依赖注入 
                    registerDependentBean(autowiredBeanName, beanName);
                    if (logger.isDebugEnabled()) {
                        logger.debug("Autowiring by type from bean name '" + beanName + "' via property '" +
                                     propertyName + "' to bean named '" + autowiredBeanName + "'");
                    }
                }
                //释放已自动注入的属性 
                autowiredBeanNames.clear();
            }
        }
        catch (BeansException ex) {
            throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
        }
    }
}
```



### 3. DefaultSingletonBeanRegistry 的 registerDependentBean 方法对属性注入：

* autowiring的实现过程：  
  a.  对Bean的属性迭代调用getBean方法，完成依赖Bean的初始化和依赖注入。  
  b.  将依赖Bean的属性引用设置到被依赖的Bean属性上。  
  c.  将依赖Bean的名称和被依赖Bean的名称存储在IoC容器的集合中。  



```java
//为指定的Bean注入依赖的Bean  
public void registerDependentBean(String beanName, String dependentBeanName) {  
    //处理Bean名称，将别名转换为规范的Bean名称  
    String canonicalName = canonicalName(beanName);  
    //多线程同步，保证容器内数据的一致性  
    //先从容器中：bean名称-->全部依赖Bean名称集合找查找给定名称Bean的依赖Bean  
    synchronized (this.dependentBeanMap) {  
        //获取给定名称Bean的所有依赖Bean名称  
        Set<String> dependentBeans = this.dependentBeanMap.get(canonicalName);  
        if (dependentBeans == null) {  
            //为Bean设置依赖Bean信息  
            dependentBeans = new LinkedHashSet<String>(8);  
            this.dependentBeanMap.put(canonicalName, dependentBeans);  
        }  
        //向容器中：bean名称-->全部依赖Bean名称集合添加Bean的依赖信息  
        //即，将Bean所依赖的Bean添加到容器的集合中  
        dependentBeans.add(dependentBeanName);  
    }  
    //从容器中：bean名称-->指定名称Bean的依赖Bean集合找查找给定名称  
    //Bean的依赖Bean  
    synchronized (this.dependenciesForBeanMap) {  
        Set<String> dependenciesForBean = this.dependenciesForBeanMap.get(dependentBeanName);  
        if (dependenciesForBean == null) {  
            dependenciesForBean = new LinkedHashSet<String>(8);  
            this.dependenciesForBeanMap.put(dependentBeanName, dependenciesForBean);  
        }  
        //向容器中：bean名称-->指定Bean的依赖Bean名称集合添加Bean的依赖信息  
        //即，将Bean所依赖的Bean添加到容器的集合中  
        dependenciesForBean.add(canonicalName);  
    }  
}
```

