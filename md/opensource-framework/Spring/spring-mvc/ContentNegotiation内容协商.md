## ContentNegotiationStrategy

```java
// A strategy for resolving the requested media types for a request.
// @since 3.2
@FunctionalInterface
public interface ContentNegotiationStrategy {
	// @since 5.0.5
	List<MediaType> MEDIA_TYPE_ALL_LIST = Collections.singletonList(MediaType.ALL);

	// 将给定的请求解析为媒体类型列表
	// 返回的 List 首先按照 specificity 参数排序，其次按照 quality 参数排序
	// 如果请求的媒体类型不能被解析则抛出 HttpMediaTypeNotAcceptableException 异常
	List<MediaType> resolveMediaTypes(NativeWebRequest webRequest) throws HttpMediaTypeNotAcceptableException;
}
```

Spring MVC默认加载两个该策略接口的实现类：
ServletPathExtensionContentNegotiationStrategy–>根据文件扩展名（支持RESTful）。
HeaderContentNegotiationStrategy–>根据HTTP Header里的Accept字段（支持Http）。



### HeaderContentNegotiationStrategy

`Accept Header`解析：它根据请求头`Accept`来协商。

```java
public class HeaderContentNegotiationStrategy implements ContentNegotiationStrategy {
	@Override
	public List<MediaType> resolveMediaTypes(NativeWebRequest request) throws HttpMediaTypeNotAcceptableException {
	
		// 我的Chrome浏览器值是：[text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3]
		// postman的值是：[*/*]
		String[] headerValueArray = request.getHeaderValues(HttpHeaders.ACCEPT);
		if (headerValueArray == null) {
			return MEDIA_TYPE_ALL_LIST;
		}

		List<String> headerValues = Arrays.asList(headerValueArray);
		try {
			List<MediaType> mediaTypes = MediaType.parseMediaTypes(headerValues);
			// 排序
			MediaType.sortBySpecificityAndQuality(mediaTypes);
			// 最后Chrome浏览器的List如下：
			// 0 = {MediaType@6205} "text/html"
			// 1 = {MediaType@6206} "application/xhtml+xml"
			// 2 = {MediaType@6207} "image/webp"
			// 3 = {MediaType@6208} "image/apng"
			// 4 = {MediaType@6209} "application/signed-exchange;v=b3"
			// 5 = {MediaType@6210} "application/xml;q=0.9"
			// 6 = {MediaType@6211} "*/*;q=0.8"
			return !CollectionUtils.isEmpty(mediaTypes) ? mediaTypes : MEDIA_TYPE_ALL_LIST;
		} catch (InvalidMediaTypeException ex) {
			throw new HttpMediaTypeNotAcceptableException("Could not parse 'Accept' header " + headerValues + ": " + ex.getMessage());
		}
	}
}
```

**如果没有传递Accept，则默认使用MediaType.ALL 也就是\*/***



### MediaTypeFileExtensionResolver

`MediaType`和路径扩展名解析策略的接口，例如将 `.json` 解析成 `application/json` 或者反向解析

```java
// @since 3.2
public interface MediaTypeFileExtensionResolver {

	// 根据指定的mediaType返回一组文件扩展名
	List<String> resolveFileExtensions(MediaType mediaType);
	// 返回该接口注册进来的所有的扩展名
	List<String> getAllFileExtensions();
}
```



#### MappingMediaTypeFileExtensionResolver

```java
public class MappingMediaTypeFileExtensionResolver implements MediaTypeFileExtensionResolver {

	// key是lowerCaseExtension，value是对应的mediaType
	private final ConcurrentMap<String, MediaType> mediaTypes = new ConcurrentHashMap<>(64);
	// 和上面相反，key是mediaType，value是lowerCaseExtension（显然用的是多值map）
	private final MultiValueMap<MediaType, String> fileExtensions = new LinkedMultiValueMap<>();
	// 所有的扩展名（List非set哦~）
	private final List<String> allFileExtensions = new ArrayList<>();

	...
	public Map<String, MediaType> getMediaTypes() {
		return this.mediaTypes;
	}
	// protected 方法
	protected List<MediaType> getAllMediaTypes() {
		return new ArrayList<>(this.mediaTypes.values());
	}
	// 给extension添加一个对应的mediaType
	// 采用ConcurrentMap是为了避免出现并发情况下导致的一致性问题
	protected void addMapping(String extension, MediaType mediaType) {
		MediaType previous = this.mediaTypes.putIfAbsent(extension, mediaType);
		if (previous == null) {
			this.fileExtensions.add(mediaType, extension);
			this.allFileExtensions.add(extension);
		}
	}

	// 接口方法：拿到指定的mediaType对应的扩展名们~
	@Override
	public List<String> resolveFileExtensions(MediaType mediaType) {
		List<String> fileExtensions = this.fileExtensions.get(mediaType);
		return (fileExtensions != null ? fileExtensions : Collections.emptyList());
	}
	@Override
	public List<String> getAllFileExtensions() {
		return Collections.unmodifiableList(this.allFileExtensions);
	}

	// protected 方法：根据扩展名找到一个MediaType~（当然可能是找不到的）
	@Nullable
	protected MediaType lookupMediaType(String extension) {
		return this.mediaTypes.get(extension.toLowerCase(Locale.ENGLISH));
	}
}
```

维护了一个文件扩展名和`MediaType`的双向查找表



### AbstractMappingContentNegotiationStrategy

模版处理流程

```java
// @since 3.2 它是个协商策略抽象实现，同时也有了扩展名+MediaType对应关系的能力
public abstract class AbstractMappingContentNegotiationStrategy extends MappingMediaTypeFileExtensionResolver implements ContentNegotiationStrategy {

	// Whether to only use the registered mappings to look up file extensions,
	// or also to use dynamic resolution (e.g. via {@link MediaTypeFactory}.
	// org.springframework.http.MediaTypeFactory是Spring5.0提供的一个工厂类
	// 它会读取/org/springframework/http/mime.types这个文件，里面有记录着对应关系
	private boolean useRegisteredExtensionsOnly = false;
	// Whether to ignore requests with unknown file extension. Setting this to
	// 默认false：若认识不认识的扩展名，抛出异常：HttpMediaTypeNotAcceptableException
	private boolean ignoreUnknownExtensions = false;

	// 唯一构造函数
	public AbstractMappingContentNegotiationStrategy(@Nullable Map<String, MediaType> mediaTypes) {
		super(mediaTypes);
	}

	// 实现策略接口方法
	@Override
	public List<MediaType> resolveMediaTypes(NativeWebRequest webRequest) throws HttpMediaTypeNotAcceptableException {
		// getMediaTypeKey：抽象方法(让子类把扩展名这个key提供出来)
		return resolveMediaTypeKey(webRequest, getMediaTypeKey(webRequest));
	}

	public List<MediaType> resolveMediaTypeKey(NativeWebRequest webRequest, @Nullable String key) throws HttpMediaTypeNotAcceptableException {
		if (StringUtils.hasText(key)) {
			// 调用父类方法：根据key去查找出一个MediaType出来
			MediaType mediaType = lookupMediaType(key); 
			// 找到了就return就成（handleMatch是protected的空方法~~~  子类目前没有实现的）
			if (mediaType != null) {
				handleMatch(key, mediaType); // 回调
				return Collections.singletonList(mediaType);
			}

			// 若没有对应的MediaType，交给handleNoMatch处理（默认是抛出异常，见下面）
			// 注意：handleNoMatch如果通过工厂找到了，那就addMapping()保存起来（相当于注册上去）
			mediaType = handleNoMatch(webRequest, key);
			if (mediaType != null) {
				addMapping(key, mediaType);
				return Collections.singletonList(mediaType);
			}
		}
		return MEDIA_TYPE_ALL_LIST; // 默认值：所有
	}

	// 此方法子类ServletPathExtensionContentNegotiationStrategy有复写
	@Nullable
	protected MediaType handleNoMatch(NativeWebRequest request, String key) throws HttpMediaTypeNotAcceptableException {

		// 若不是仅仅从注册里的拿，那就再去MediaTypeFactory里看看~~~  找到了就返回
		if (!isUseRegisteredExtensionsOnly()) {
			Optional<MediaType> mediaType = MediaTypeFactory.getMediaType("file." + key);
			if (mediaType.isPresent()) {
				return mediaType.get();
			}
		}

		// 忽略找不到，返回null吧  否则抛出异常：HttpMediaTypeNotAcceptableException
		if (isIgnoreUnknownExtensions()) {
			return null;
		}
		throw new HttpMediaTypeNotAcceptableException(getAllMediaTypes());
	}
}
```



#### ParameterContentNegotiationStrategy

```java
public class ParameterContentNegotiationStrategy extends AbstractMappingContentNegotiationStrategy {
	// 请求参数默认的key是format，你是可以设置和更改的。(set方法)
	private String parameterName = "format";

	// 唯一构造
	public ParameterContentNegotiationStrategy(Map<String, MediaType> mediaTypes) {
		super(mediaTypes);
	}
	... // 生路get/set

	// 小Tips：这里调用的是getParameterName()而不是直接用属性名，以后建议大家设计框架也都这么使用 虽然很多时候效果是一样的，但更符合使用规范
	@Override
	@Nullable
	protected String getMediaTypeKey(NativeWebRequest request) {
		return request.getParameter(getParameterName());
	}
}
```



##### PathExtensionContentNegotiationStrategy

根据请求`URL`路径中所请求的**文件资源的扩展名**部分判断请求的`MediaType`（借助`UrlPathHelper`和`UriUtils`解析`URL`）

```java
public class PathExtensionContentNegotiationStrategy extends AbstractMappingContentNegotiationStrategy {

	private UrlPathHelper urlPathHelper = new UrlPathHelper();

	// 它额外提供了一个空构造
	public PathExtensionContentNegotiationStrategy() {
		this(null);
	}
	// 有参构造
	public PathExtensionContentNegotiationStrategy(@Nullable Map<String, MediaType> mediaTypes) {
		super(mediaTypes);
		setUseRegisteredExtensionsOnly(false);
		setIgnoreUnknownExtensions(true); // 注意：这个值设置为了true
		this.urlPathHelper.setUrlDecode(false); // 不需要解码（url请勿有中文）
	}

	// @since 4.2.8  可见Spring MVC允许你自己定义解析的逻辑
	public void setUrlPathHelper(UrlPathHelper urlPathHelper) {
		this.urlPathHelper = urlPathHelper;
	}


	@Override
	@Nullable
	protected String getMediaTypeKey(NativeWebRequest webRequest) {
		HttpServletRequest request = webRequest.getNativeRequest(HttpServletRequest.class);
		if (request == null) {
			return null;
		}

		// 借助urlPathHelper、UriUtils从URL中把扩展名解析出来
		String path = this.urlPathHelper.getLookupPathForRequest(request);
		String extension = UriUtils.extractFileExtension(path);
		return (StringUtils.hasText(extension) ? extension.toLowerCase(Locale.ENGLISH) : null);
	}

	// 子类ServletPathExtensionContentNegotiationStrategy有使用和复写
	// 它的作用是面向Resource找到这个资源对应的MediaType ~
	@Nullable
	public MediaType getMediaTypeForResource(Resource resource) { ... }
}
```



#### ServletPathExtensionContentNegotiationStrategy

```java
public class ServletPathExtensionContentNegotiationStrategy extends PathExtensionContentNegotiationStrategy {
	private final ServletContext servletContext;
	... // 省略构造函数

	// 一句话：在去工厂找之前，先去this.servletContext.getMimeType("file." + extension)这里找一下，找到就直接返回。否则再进工厂
	@Override
	@Nullable
	protected MediaType handleNoMatch(NativeWebRequest webRequest, String extension) throws HttpMediaTypeNotAcceptableException { ... }

	//  一样的：先this.servletContext.getMimeType(resource.getFilename()) 再交给父类处理
	@Override
	public MediaType getMediaTypeForResource(Resource resource) { ... }

	// 两者调用父类的条件都是：mediaType == null || MediaType.APPLICATION_OCTET_STREAM.equals(mediaType)
}
```

#### FixedContentNegotiationStrategy



## ContentNegotiationManager

```java
//  它不仅管理一堆strategies（List），还管理一堆resolvers（Set）
public class ContentNegotiationManager implements ContentNegotiationStrategy, MediaTypeFileExtensionResolver {
	private final List<ContentNegotiationStrategy> strategies = new ArrayList<>();
	private final Set<MediaTypeFileExtensionResolver> resolvers = new LinkedHashSet<>();
	
	...
	// 若没特殊指定，至少是包含了这一种的策略的：HeaderContentNegotiationStrategy
	public ContentNegotiationManager() {
		this(new HeaderContentNegotiationStrategy());
	}
	... // 因为比较简单，所以省略其它代码
}
```



### ContentNegotiationManagerFactoryBean

后缀 > 请求参数 > HTTP首部Accept

```java
// @since 3.2  还实现了ServletContextAware，可以得到当前servlet容器上下文
public class ContentNegotiationManagerFactoryBean implements FactoryBean<ContentNegotiationManager>, ServletContextAware, InitializingBean {
	
	// 默认就是开启了对后缀的支持的
	private boolean favorPathExtension = true;
	// 默认没有开启对param的支持
	private boolean favorParameter = false;
	// 默认也是开启了对Accept的支持的
	private boolean ignoreAcceptHeader = false;

	private Map<String, MediaType> mediaTypes = new HashMap<String, MediaType>();
	private boolean ignoreUnknownPathExtensions = true;
	// Jaf是一个数据处理框架，可忽略
	private Boolean useJaf;
	private String parameterName = "format";
	private ContentNegotiationStrategy defaultNegotiationStrategy;
	private ContentNegotiationManager contentNegotiationManager;
	private ServletContext servletContext;
	... // 省略普通的get/set

	// 注意这里传入的是：Properties  表示后缀和MediaType的对应关系
	public void setMediaTypes(Properties mediaTypes) {
		if (!CollectionUtils.isEmpty(mediaTypes)) {
			for (Entry<Object, Object> entry : mediaTypes.entrySet()) {
				String extension = ((String)entry.getKey()).toLowerCase(Locale.ENGLISH);
				MediaType mediaType = MediaType.valueOf((String) entry.getValue());
				this.mediaTypes.put(extension, mediaType);
			}
		}
	}
	public void addMediaType(String fileExtension, MediaType mediaType) {
		this.mediaTypes.put(fileExtension, mediaType);
	}
	...
	
	// 这里面处理了很多默认逻辑
	@Override
	public void afterPropertiesSet() {
		List<ContentNegotiationStrategy> strategies = new ArrayList<ContentNegotiationStrategy>();

		// 默认favorPathExtension=true，所以是支持path后缀模式的
		// servlet环境使用的是ServletPathExtensionContentNegotiationStrategy，否则使用的是PathExtensionContentNegotiationStrategy
		// 
		if (this.favorPathExtension) {
			PathExtensionContentNegotiationStrategy strategy;
			if (this.servletContext != null && !isUseJafTurnedOff()) {
				strategy = new ServletPathExtensionContentNegotiationStrategy(this.servletContext, this.mediaTypes);
			} else {
				strategy = new PathExtensionContentNegotiationStrategy(this.mediaTypes);
			}
			strategy.setIgnoreUnknownExtensions(this.ignoreUnknownPathExtensions);
			if (this.useJaf != null) {
				strategy.setUseJaf(this.useJaf);
			}
			strategies.add(strategy);
		}

		// 默认favorParameter=false 木有开启滴
		if (this.favorParameter) {
			ParameterContentNegotiationStrategy strategy = new ParameterContentNegotiationStrategy(this.mediaTypes);
			strategy.setParameterName(this.parameterName);
			strategies.add(strategy);
		}

		// 注意这前面有个!，所以默认Accept也是支持的
		if (!this.ignoreAcceptHeader) {
			strategies.add(new HeaderContentNegotiationStrategy());
		}

		// 若你喜欢，你可以设置一个defaultNegotiationStrategy  最终也会被add进去
		if (this.defaultNegotiationStrategy != null) {
			strategies.add(this.defaultNegotiationStrategy);
		}

		// 这部分我需要提醒注意的是：这里使用的是ArrayList，所以你add的顺序就是u最后的执行顺序
		// 所以若你指定了defaultNegotiationStrategy，它也是放到最后的
		this.contentNegotiationManager = new ContentNegotiationManager(strategies);
	}

	// 三个接口方法
	@Override
	public ContentNegotiationManager getObject() {
		return this.contentNegotiationManager;
	}
	@Override
	public Class<?> getObjectType() {
		return ContentNegotiationManager.class;
	}
	@Override
	public boolean isSingleton() {
		return true;
	}
}
```



### 内容协商的配置：`ContentNegotiationConfigurer`

`ContentNegotiationConfigurer`可以认为是提供一个设置`ContentNegotiationManagerFactoryBean`的入口（自己内容new了一个它的实例），最终交给`WebMvcConfigurationSupport`向容器内注册这个Bean

```java
public class ContentNegotiationConfigurer {

	private final ContentNegotiationManagerFactoryBean factory = new ContentNegotiationManagerFactoryBean();
	private final Map<String, MediaType> mediaTypes = new HashMap<String, MediaType>();

	public ContentNegotiationConfigurer(@Nullable ServletContext servletContext) {
		if (servletContext != null) {
			this.factory.setServletContext(servletContext);
		}
	}
	// @since 5.0
	public void strategies(@Nullable List<ContentNegotiationStrategy> strategies) {
		this.factory.setStrategies(strategies);
	}
	...
	public ContentNegotiationConfigurer defaultContentTypeStrategy(ContentNegotiationStrategy defaultStrategy) {
		this.factory.setDefaultContentTypeStrategy(defaultStrategy);
		return this;
	}

	// 手动创建出一个ContentNegotiationManager 此方法是protected 
	// 唯一调用处是：WebMvcConfigurationSupport
	protected ContentNegotiationManager buildContentNegotiationManager() {
		this.factory.addMediaTypes(this.mediaTypes);
		return this.factory.build();
	}
}
```



## AbstractMessageConverterMethodProcessor#writeWithMessageConverters()



## ContentNegotiatingViewResolver：内容协商视图解析器



```java
// @since 3.0
public class ContentNegotiatingViewResolver extends WebApplicationObjectSupport implements ViewResolver, Ordered, InitializingBean {
	// 用于内容协商的管理器
	@Nullable
	private ContentNegotiationManager contentNegotiationManager;
	private final ContentNegotiationManagerFactoryBean cnmFactoryBean = new ContentNegotiationManagerFactoryBean();

	// 如果没有合适的view的时候，是否使用406这个状态码（HttpServletResponse#SC_NOT_ACCEPTABLE）
	// 默认值是false：表示没有找到就返回null，而不是406
	private boolean useNotAcceptableStatusCode = false;
	// 当无法获取到具体的视图时，会走defaultViews
	@Nullable
	private List<View> defaultViews;
	
	@Nullable
	private List<ViewResolver> viewResolvers;
	private int order = Ordered.HIGHEST_PRECEDENCE; // 默认，优先级就是最高的

	// 复写：WebApplicationObjectSupport的方法
	// 它在setServletContext和initApplicationContext会调用（也就是容器启动时候会调用）
	@Override
	protected void initServletContext(ServletContext servletContext) {
		Collection<ViewResolver> matchingBeans = BeanFactoryUtils.beansOfTypeIncludingAncestors(obtainApplicationContext(), ViewResolver.class).values();
		//容器内找到了  就以容器内所有已经配置好的视图解析器都拿出来（包含父容器）
		if (this.viewResolvers == null) {
			this.viewResolvers = new ArrayList<>(matchingBeans.size());
			for (ViewResolver viewResolver : matchingBeans) {
				if (this != viewResolver) { // 排除自己
					this.viewResolvers.add(viewResolver);
				}
			}
		} else { // 进入这里证明是调用者自己set进来的
			for (int i = 0; i < this.viewResolvers.size(); i++) {
				ViewResolver vr = this.viewResolvers.get(i);
				if (matchingBeans.contains(vr)) {
					continue;
				}
				String name = vr.getClass().getName() + i;
				// 对视图解析器完成初始化工作~~~~~
				// 关于AutowireCapableBeanFactory的使用，参见：https://blog.csdn.net/f641385712/article/details/88651128
				obtainApplicationContext().getAutowireCapableBeanFactory().initializeBean(vr, name);
			}

		}

		// 找到所有的ViewResolvers排序后，放进ContentNegotiationManagerFactoryBean里
		AnnotationAwareOrderComparator.sort(this.viewResolvers);
		this.cnmFactoryBean.setServletContext(servletContext);
	}

	// 从这一步骤可以知道：contentNegotiationManager 可以自己set
	// 也可以通过工厂来生成  两种方式均可
	@Override
	public void afterPropertiesSet() {
		if (this.contentNegotiationManager == null) {
			this.contentNegotiationManager = this.cnmFactoryBean.build();
		}
		if (this.viewResolvers == null || this.viewResolvers.isEmpty()) {
			logger.warn("No ViewResolvers configured");
		}
	}

	// 处理逻辑视图到View 在此处会进行内容协商
	@Override
	@Nullable
	public View resolveViewName(String viewName, Locale locale) throws Exception {
		RequestAttributes attrs = RequestContextHolder.getRequestAttributes();
		Assert.state(attrs instanceof ServletRequestAttributes, "No current ServletRequestAttributes");

		// getMediaTypes()这个方法完成了
		// 1、通过contentNegotiationManager.resolveMediaTypes(webRequest)得到请求的MediaTypes
		// 2、拿到服务端能够提供的MediaTypes  producibleMediaTypes
		// (请注意因为没有消息转换器，所以它的值的唯一来源是：request.getAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE))
		// (若没有指定producers的值，那就是ALL)
		// 3、按照优先级，协商出`selectedMediaTypes`（是个List）
		List<MediaType> requestedMediaTypes = getMediaTypes(((ServletRequestAttributes) attrs).getRequest());

		// 进入此处：说明协商出了有可用的MediaTypes（至少有一个嘛）
		if (requestedMediaTypes != null) {

			// getCandidateViews()这个很重要的方法，见下文
			List<View> candidateViews = getCandidateViews(viewName, locale, requestedMediaTypes);

			// 上面一步骤解析出了多个符合条件的views，这里就是通过MediaType、attrs等等一起决定出一个，一个，一个最佳的
			// getBestView()方法描述如下：
			// 第一大步：遍历所有的candidateViews，只要是smartView.isRedirectView()，就直接return
			// 第二大步：遍历所有的requestedMediaTypes，针对每一种MediaType下再遍历所有的candidateViews
			// 1、针对每一种MediaType，拿出View.getContentType()，只会看这个值不为null的
			// 2、view的contentType!=null，继续看看mediaType.isCompatibleWith(candidateContentType) 若不匹配这个视图就略过
			// 3、若匹配：attrs.setAttribute(View.SELECTED_CONTENT_TYPE, mediaType, RequestAttributes.SCOPE_REQUEST)  然后return掉此视图作为best最佳的
			View bestView = getBestView(candidateViews, requestedMediaTypes, attrs);
			if (bestView != null) { // 很显然，找到了最佳的就返回渲染吧
				return bestView;
			}
		}
		
		... 
		// useNotAcceptableStatusCode=true没找到视图就返回406
		// NOT_ACCEPTABLE_VIEW是个private内部静态类View，它的render方法只有一句话:
		// response.setStatus(HttpServletResponse.SC_NOT_ACCEPTABLE);
		if (this.useNotAcceptableStatusCode) {
			return NOT_ACCEPTABLE_VIEW;
		} else {
			return null;
		}
	}

	// 根据viewName、requestedMediaTypes等等去得到所有的备选的Views~~
	// 这这里会调用所有的viewResolvers.resolveViewName()来分别处理~~~所以可能生成多多个viewo ~
	private List<View> getCandidateViews(String viewName, Locale locale, List<MediaType> requestedMediaTypes) throws Exception {
		List<View> candidateViews = new ArrayList<>();
		if (this.viewResolvers != null) {
			Assert.state(this.contentNegotiationManager != null, "No ContentNegotiationManager set");

			// 遍历所有的viewResolvers，多逻辑视图一个一个的处理
			for (ViewResolver viewResolver : this.viewResolvers) {
				View view = viewResolver.resolveViewName(viewName, locale);
				if (view != null) {
					candidateViews.add(view); // 处理好的就装进来
				}


				// 另外还没有完：遍历所有支持的MediaType，拿到它对应的扩展名们（一个MediaType可以对应多个扩展名）
				// 如果viewName + '.' + extension能被处理成一个视图，也是ok的
				// 也就是说index和index.jsp都能被解析成view视图~~~
				for (MediaType requestedMediaType : requestedMediaTypes) {
					// resolveFileExtensions()方法可以说这里是唯一调用的地方
					List<String> extensions = this.contentNegotiationManager.resolveFileExtensions(requestedMediaType);
					for (String extension : extensions) {
						String viewNameWithExtension = viewName + '.' + extension;
						view = viewResolver.resolveViewName(viewNameWithExtension, locale);
						if (view != null) {
							candidateViews.add(view); // 带上后缀名也能够处理的  这种视图也ok
						}
					}
				}
			}
		}
		// 若指定了默认视图，把视图也得加上（在最后面哦~）
		if (!CollectionUtils.isEmpty(this.defaultViews)) {
			candidateViews.addAll(this.defaultViews);
		}
		return candidateViews;
	}
}
```





```java
public class ViewResolverRegistry {
	...
	public void enableContentNegotiation(View... defaultViews) {
		initContentNegotiatingViewResolver(defaultViews);
	}
	public void enableContentNegotiation(boolean useNotAcceptableStatus, View... defaultViews) {
		ContentNegotiatingViewResolver vr = initContentNegotiatingViewResolver(defaultViews);
		vr.setUseNotAcceptableStatusCode(useNotAcceptableStatus);
	}
	// 初始化一个内容协商视图解析器
	private ContentNegotiatingViewResolver initContentNegotiatingViewResolver(View[] defaultViews) {
		// ContentNegotiatingResolver in the registry: elevate its precedence!
		// 请保证它是最高优先级的：在所有视图解析器之前执行
		// 这样即使你配置了其它的视图解析器  也会先执行这个（后面的被短路掉）
		this.order = (this.order != null ? this.order : Ordered.HIGHEST_PRECEDENCE);

		// 调用者自己已经配置好了一个contentNegotiatingResolver，那就用他的
		if (this.contentNegotiatingResolver != null) {
			// 若存在defaultViews，那就处理一下把它放进contentNegotiatingResolver里面
			if (!ObjectUtils.isEmpty(defaultViews) && !CollectionUtils.isEmpty(this.contentNegotiatingResolver.getDefaultViews())) {
				List<View> views = new ArrayList<>(this.contentNegotiatingResolver.getDefaultViews());
				views.addAll(Arrays.asList(defaultViews));
				this.contentNegotiatingResolver.setDefaultViews(views);
			}
		} else { // 若没配置就自己new一个 并且设置好viewResolvers
			this.contentNegotiatingResolver = new ContentNegotiatingViewResolver();
			this.contentNegotiatingResolver.setDefaultViews(Arrays.asList(defaultViews));
			// 注意：这个viewResolvers是通过此ViewResolverRegistry配置进来的
			// 若仅仅是容器内的Bean，这里可捕获不到。所以如果你有特殊需求建议你自己set
			// 若仅仅是jsp()/tiles()/freeMarker()/groovy()/beanName()这些，内置的支持即可满足要求儿聊
			// ViewResolverRegistry.viewResolver()可调用多次，因此可以多次指定  若有需要个性化，可以调用此方法
			this.contentNegotiatingResolver.setViewResolvers(this.viewResolvers);
			if (this.contentNegotiationManager != null) {
				this.contentNegotiatingResolver.setContentNegotiationManager(this.contentNegotiationManager);
			}
		}
		return this.contentNegotiatingResolver;
	}
}
```





