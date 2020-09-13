# ViewResolver

```java
// 这个接口非常简单，就一个方法:把一个逻辑视图viewName解析为一个真正的视图View，Local表示国际化相关内容~
public interface ViewResolver {
	@Nullable
	View resolveViewName(String viewName, Locale locale) throws Exception;
}
```



## AbstractCachingViewResolver 非常重要

```java
// 该首相类完成的主要是缓存的相关逻辑~~~
public abstract class AbstractCachingViewResolver extends WebApplicationObjectSupport implements ViewResolver {
	
	// Map的最大值，1024我觉得还是挺大的了~
	/** Default maximum number of entries for the view cache: 1024. */
	public static final int DEFAULT_CACHE_LIMIT = 1024;

	// 表示没有被解析过的View~~~
	private static final View UNRESOLVED_VIEW = new View() {
		@Override
		@Nullable
		public String getContentType() {
			return null;
		}
		@Override
		public void render(@Nullable Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) {
		}
	};
	private volatile int cacheLimit = DEFAULT_CACHE_LIMIT;
	private boolean cacheUnresolved = true;
	// 此处使用的是ConcurrentHashMap，key是Object
	private final Map<Object, View> viewAccessCache = new ConcurrentHashMap<>(DEFAULT_CACHE_LIMIT);

	// 通过它来实现缓存最大值： removeEldestEntry表示当你往里put成为为true的时候，会执行它
	// 此处可以看到，当size大于1024时，会把Map里面最老的那个值给remove掉~~~viewAccessCache.remove(eldest.getKey());
	private final Map<Object, View> viewCreationCache =
			new LinkedHashMap<Object, View>(DEFAULT_CACHE_LIMIT, 0.75f, true) {
				@Override
				protected boolean removeEldestEntry(Map.Entry<Object, View> eldest) {
					if (size() > getCacheLimit()) {
						viewAccessCache.remove(eldest.getKey());
						return true;
					}
					else {
						return false;
					}
				}
			};
	
	...

	// 通过逻辑视图，来找到一个View真正的视图~~~~
	@Override
	@Nullable
	public View resolveViewName(String viewName, Locale locale) throws Exception {
		if (!isCache()) {
			return createView(viewName, locale);
		} else {
			// cacheKey其实就是 viewName + '_' + locale
			Object cacheKey = getCacheKey(viewName, locale);
			View view = this.viewAccessCache.get(cacheKey);
			if (view == null) {
				synchronized (this.viewCreationCache) {
					view = this.viewCreationCache.get(cacheKey);
					if (view == null) {
						// Ask the subclass to create the View object.
						// 具体的创建视图的逻辑  交给子类的去完成~~~~
						view = createView(viewName, locale);
						// 此处需要注意：若调用者返回的是null，并且cacheUnresolved，那就返回一个未经处理的视图~~~~
						if (view == null && this.cacheUnresolved) {
							view = UNRESOLVED_VIEW;
						}
						// 缓存起来~~~~
						if (view != null) {
							this.viewAccessCache.put(cacheKey, view);
							this.viewCreationCache.put(cacheKey, view);
						}
					}
				}
			}
			else {
				if (logger.isTraceEnabled()) {
					logger.trace(formatKey(cacheKey) + "served from cache");
				}
			}
		
			// 这个很重要，因为没有被解析过  都会返回null
			// 而再真正责任链处理的时候，第一个不返回null的view，最终就会被返回了~~~
			return (view != UNRESOLVED_VIEW ? view : null);
		}
	}

	// 逻辑比较简单~~~
	public void removeFromCache(String viewName, Locale locale) {
		...
	}
	public void clearCache() {
		logger.debug("Clearing all views from the cache");
		synchronized (this.viewCreationCache) {
			this.viewAccessCache.clear();
			this.viewCreationCache.clear();
		}
	}
}
```

### InternalResourceViewResolver

```java
public class InternalResourceViewResolver extends UrlBasedViewResolver {

	// 如果你导入了JSTL的相关的包，这个解析器也会支持JSTLView的~~~~
	private static final boolean jstlPresent = ClassUtils.isPresent(
			"javax.servlet.jsp.jstl.core.Config", InternalResourceViewResolver.class.getClassLoader());

	// 指定是否始终包含视图而不是转发到视图
	// 默认值为“false”。打开此标志以强制使用servlet include，即使可以进行转发
	// InternalResourceView#setAlwaysInclude
	@Nullable
	private Boolean alwaysInclude;

	@Override
	protected Class<?> requiredViewClass() {
		return InternalResourceView.class;
	}

	// 默认情况下，它可能会设置一个JstlView 或者 InternalResourceView
	public InternalResourceViewResolver() {
		Class<?> viewClass = requiredViewClass();
		if (InternalResourceView.class == viewClass && jstlPresent) {
			viewClass = JstlView.class;
		}
		setViewClass(viewClass);
	}
	public InternalResourceViewResolver(String prefix, String suffix) {
		this(); // 先调用空构造
		setPrefix(prefix);
		setSuffix(suffix);
	}

	// 在父类实现的记仇上，设置上了alwaysInclude，并且view.setPreventDispatchLoop(true)
	@Override
	protected AbstractUrlBasedView buildView(String viewName) throws Exception {
		InternalResourceView view = (InternalResourceView) super.buildView(viewName);
		if (this.alwaysInclude != null) {
			view.setAlwaysInclude(this.alwaysInclude);
		}
		view.setPreventDispatchLoop(true);
		return view;
	}

}
```

### AbstractTemplateViewResolver

```java
// 模板视图解析程序的抽象基类，尤其是FreeMarker视图的抽象基类
// @since 1.1  对应的View是AbstractTemplateView
public class AbstractTemplateViewResolver extends UrlBasedViewResolver {

	// 是否吧所有热request里面的attributes都加入合并到模版的Model，默认是false
	private boolean exposeRequestAttributes = false;
	// 是否允许request里面的属性，当name相同的时候，复写model里面的 默认是false
	private boolean allowRequestOverride = false;

	// session相关，语义同上
	private boolean exposeSessionAttributes = false;
	private boolean allowSessionOverride = false;

	// Set whether to expose a RequestContext for use by Spring's macro library 默认值是true
	private boolean exposeSpringMacroHelpers = true;

	// 它只会处理AbstractTemplateView 比如FreeMarkerView是它的实现类
	@Override
	protected Class<?> requiredViewClass() {
		return AbstractTemplateView.class;
	}

	// 模版操作：其实就是多设置了一些开关属性~~~~
	@Override
	protected AbstractUrlBasedView buildView(String viewName) throws Exception {
		AbstractTemplateView view = (AbstractTemplateView) super.buildView(viewName);
		view.setExposeRequestAttributes(this.exposeRequestAttributes);
		view.setAllowRequestOverride(this.allowRequestOverride);
		view.setExposeSessionAttributes(this.exposeSessionAttributes);
		view.setAllowSessionOverride(this.allowSessionOverride);
		view.setExposeSpringMacroHelpers(this.exposeSpringMacroHelpers);
		return view;
	}
}
```



## ViewResolverComposite

```java
// @since 4.1
public class ViewResolverComposite implements ViewResolver, Ordered, InitializingBean, ApplicationContextAware, ServletContextAware {

	private final List<ViewResolver> viewResolvers = new ArrayList<>();
	private int order = Ordered.LOWEST_PRECEDENCE;

	// 若直接set了，就以自己的set为主
	public void setViewResolvers(List<ViewResolver> viewResolvers) {
		this.viewResolvers.clear();
		if (!CollectionUtils.isEmpty(viewResolvers)) {
			this.viewResolvers.addAll(viewResolvers);
		}
	}

	// 为每一个实现了接口ApplicationContextAware的 都设置一个 下面还有其它的
	@Override
	public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
		for (ViewResolver viewResolver : this.viewResolvers) {
			if (viewResolver instanceof ApplicationContextAware) {
				((ApplicationContextAware)viewResolver).setApplicationContext(applicationContext);
			}
		}
	}
	...


	// 这是核心   遍历所有的viewResolvers,第一个不返回null的，就标出处理了~~~~
	@Override
	@Nullable
	public View resolveViewName(String viewName, Locale locale) throws Exception {
		for (ViewResolver viewResolver : this.viewResolvers) {
			View view = viewResolver.resolveViewName(viewName, locale);
			if (view != null) {
				return view;
			}
		}
		return null;
	}
}
```



ContentNegotiatingViewResolver

ResourceBundleViewResolver



# View

```java
public interface View {

	// @since 3.0
	// HttpStatus的key，可议根据此key去获取。备注：并不是每个视图都需要实现的。目前只有`RedirectView`有处理
	String RESPONSE_STATUS_ATTRIBUTE = View.class.getName() + ".responseStatus";

	// @since 3.1  也会这样去拿：request.getAttribute(View.PATH_VARIABLES)
	String PATH_VARIABLES = View.class.getName() + ".pathVariables";

	// The {@link org.springframework.http.MediaType} selected during content negotiation
	// @since 3.2
	// MediaType mediaType = (MediaType) request.getAttribute(View.SELECTED_CONTENT_TYPE)
	String SELECTED_CONTENT_TYPE = View.class.getName() + ".selectedContentType";


	// Return the content type of the view, if predetermined（预定的）
	@Nullable
	default String getContentType() {
		return null;
	}

	// 这是最重要的 根据model里面的数据，request等  把渲染好的数据写进response里~
	void render(@Nullable Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) throws Exception;
}
```



## `AbstractView`

```java
public abstract class AbstractView extends WebApplicationObjectSupport implements View, BeanNameAware {
	/** Default content type. Overridable as bean property. */
	public static final String DEFAULT_CONTENT_TYPE = "text/html;charset=ISO-8859-1";
	/** Initial size for the temporary output byte array (if any). */
	private static final int OUTPUT_BYTE_ARRAY_INITIAL_SIZE = 4096;

	// 这几个属性值，没有陌生的。在视图解析器章节里面都有解释过~~~
	@Nullable
	private String contentType = DEFAULT_CONTENT_TYPE;
	@Nullable
	private String requestContextAttribute;
	// "Static" attributes are fixed attributes that are specified in the View instance configuration
	// "Dynamic" attributes, on the other hand,are values passed in as part of the model.
	private final Map<String, Object> staticAttributes = new LinkedHashMap<>();
	private boolean exposePathVariables = true;
	private boolean exposeContextBeansAsAttributes = false;
	@Nullable
	private Set<String> exposedContextBeanNames;

	@Nullable
	private String beanName;

	// 把你传进俩的Properties 都合并进来~~~
	public void setAttributes(Properties attributes) {
		CollectionUtils.mergePropertiesIntoMap(attributes, this.staticAttributes);
	}
	...

	@Override
	public void render(@Nullable Map<String, ?> model, HttpServletRequest request, HttpServletResponse response) throws Exception {
		// 合并staticAttributes、pathVars、model数据到一个Map里来
		// 其中：后者覆盖前者的值（若有相同key的话~~）也就是所谓的model的值优先级最高~~~~
		// 最终还会暴露RequestContext对象到Model里，因此model里可以直接访问RequestContext对象哦~~~~
		Map<String, Object> mergedModel = createMergedOutputModel(model, request, response);
		// 默认实现为设置几个响应头~~~
		// 备注：默认情况下pdf的view、xstl的view会触发下载~~~
		prepareResponse(request, response);
		// getRequestToExpose表示吧request暴露成：ContextExposingHttpServletRequest（和容器相关，以及容器内的BeanNames）
		// renderMergedOutputModel是个抽象方法 由子类去实现~~~~
		renderMergedOutputModel(mergedModel, getRequestToExpose(request), response);
	}

	//================下面是一些方法，父类提供  子类可以直接使用的方法==============
	// 一个temp输出流，缓冲区大小为4096  字节流
	protected ByteArrayOutputStream createTemporaryOutputStream() {
		return new ByteArrayOutputStream(OUTPUT_BYTE_ARRAY_INITIAL_SIZE);
	}

	// 把字节流写进response里面~~~
	protected void writeToResponse(HttpServletResponse response, ByteArrayOutputStream baos) throws IOException {
		// Write content type and also length (determined via byte array).
		response.setContentType(getContentType());
		response.setContentLength(baos.size());

		// Flush byte array to servlet output stream.
		ServletOutputStream out = response.getOutputStream();
		baos.writeTo(out);
		out.flush();
	}

	// 相当于如果request.getAttribute(View.SELECTED_CONTENT_TYPE) 指定了就以它为准~
	protected void setResponseContentType(HttpServletRequest request, HttpServletResponse response) {
		MediaType mediaType = (MediaType) request.getAttribute(View.SELECTED_CONTENT_TYPE);
		if (mediaType != null && mediaType.isConcrete()) {
			response.setContentType(mediaType.toString());
		}
		else {
			response.setContentType(getContentType());
		}
	}
	...
}
```

### AbstractJackson2View

主要做的操作是将model中的参数和request中的参数全部都放到Request中，然后就转发Request

```java
//@since 4.1 
// Compatible with Jackson 2.6 and higher, as of Spring 4.3.
public abstract class AbstractJackson2View extends AbstractView {
	private ObjectMapper objectMapper;
	private JsonEncoding encoding = JsonEncoding.UTF8;
	@Nullable
	private Boolean prettyPrint;
	private boolean disableCaching = true;
	protected boolean updateContentLength = false;

	// 唯一构造函数，并且还是protected的~~
	protected AbstractJackson2View(ObjectMapper objectMapper, String contentType) {
		this.objectMapper = objectMapper;
		configurePrettyPrint();
		setContentType(contentType);
		setExposePathVariables(false);
	}
	... // get/set方法

	// 复写了父类的此方法~~~   setResponseContentType是父类的哟~~~~
	@Override
	protected void prepareResponse(HttpServletRequest request, HttpServletResponse response) {
		setResponseContentType(request, response);
		// 设置编码格式，默认是UTF-8
		response.setCharacterEncoding(this.encoding.getJavaName());
		if (this.disableCaching) {
			response.addHeader("Cache-Control", "no-store");
		}
	}


	// 实现了父类的渲染方法~~~~
	@Override
	protected void renderMergedOutputModel(Map<String, Object> model, HttpServletRequest request,
			HttpServletResponse response) throws Exception {

		ByteArrayOutputStream temporaryStream = null;
		OutputStream stream;

		// 注意此处：updateContentLength默认值是false   所以会直接从response里面吧输出流拿出来   而不用temp流
		if (this.updateContentLength) {
			temporaryStream = createTemporaryOutputStream();
			stream = temporaryStream;
		}
		else {
			stream = response.getOutputStream();
		}

		Object value = filterAndWrapModel(model, request);
		// value是最终的从model中出来的~~~~这里就是把value值写进去~~~~
		// 先通过stream得到一个JsonGenerator，然后先writePrefix(generator, object)
		// 然后objectMapper.writerWithView
		// 最后writeSuffix(generator, object);  然后flush即可~
		writeContent(stream, value);

		if (temporaryStream != null) {
			writeToResponse(response, temporaryStream);
		}
	}

	// 筛选Model并可选地将其包装在@link mappingjacksonvalue容器中
	protected Object filterAndWrapModel(Map<String, Object> model, HttpServletRequest request) {
		// filterModel抽象方法，从指定的model中筛选出不需要的属性值~~~~~
		Object value = filterModel(model);
	
		// 把这两个属性值，选择性的放进container容器里面  最终返回~~~~
		Class<?> serializationView = (Class<?>) model.get(JsonView.class.getName());
		FilterProvider filters = (FilterProvider) model.get(FilterProvider.class.getName());
		if (serializationView != null || filters != null) {
			MappingJacksonValue container = new MappingJacksonValue(value);
			if (serializationView != null) {
				container.setSerializationView(serializationView);
			}
			if (filters != null) {
				container.setFilters(filters);
			}
			value = container;
		}
		return value;
	}

}
```

### MappingJackson2JsonView

用于返回Json视图的

```java
// @since 3.1.2 可议看到它出现得还是比较早的~
public class MappingJackson2JsonView extends AbstractJackson2View {

	public static final String DEFAULT_CONTENT_TYPE = "application/json";
	@Nullable
	private String jsonPrefix;
	@Nullable
	private Set<String> modelKeys;
	private boolean extractValueFromSingleKeyModel = false;

	@Override
	protected Object filterModel(Map<String, Object> model) {
		Map<String, Object> result = new HashMap<>(model.size());
		Set<String> modelKeys = (!CollectionUtils.isEmpty(this.modelKeys) ? this.modelKeys : model.keySet());
	
		// 遍历model所有内容~ 
		model.forEach((clazz, value) -> {
			// 符合下列条件的会给排除掉~~~
			// 不是BindingResult类型 并且  modelKeys包含此key 并且此key不是JsonView和FilterProvider  这种key就排除掉~~~
			if (!(value instanceof BindingResult) && modelKeys.contains(clazz) &&
					!clazz.equals(JsonView.class.getName()) &&
					!clazz.equals(FilterProvider.class.getName())) {
				result.put(clazz, value);
			}
		});
		// 如果只需要排除singleKey，那就返回第一个即可，否则result全部返回
		return (this.extractValueFromSingleKeyModel && result.size() == 1 ? result.values().iterator().next() : result);
	}

	// 如果配置了前缀，把前缀写进去~~~
	@Override
	protected void writePrefix(JsonGenerator generator, Object object) throws IOException {
		if (this.jsonPrefix != null) {
			generator.writeRaw(this.jsonPrefix);
		}
	}
}
```



## `AbstractUrlBasedView`





## AbstractTemplateView



### InternalResourceView

**Internal：内部的。所以该视图表示：内部资源视图。**

```java
// @since 17.02.2003  第一版就有了
public class InternalResourceView extends AbstractUrlBasedView {

	// 指定是否始终包含视图而不是转发到视图
	//默认值为“false”。打开此标志以强制使用servlet include，即使可以进行转发
	private boolean alwaysInclude = false;
	// 设置是否显式阻止分派回当前处理程序路径 表示是否组织循环转发，比如自己转发自己
	// 我个人认为这里默认值用true反而更好~~~因为需要递归的情况毕竟是极少数~
	// 其实可以看到InternalResourceViewResolver的buildView方法里是把这个属性显示的设置为true了的~~~
	private boolean preventDispatchLoop = false;

	public InternalResourceView(String url, boolean alwaysInclude) {
		super(url);
		this.alwaysInclude = alwaysInclude;
	}

	@Override
	protected boolean isContextRequired() {
		return false;
	}


	// 请求包含、请求转发是它特有的~~~~~
	@Override
	protected void renderMergedOutputModel(Map<String, Object> model, HttpServletRequest request, HttpServletResponse response) throws Exception {

		// Expose the model object as request attributes.
		// 把model里的数据都request.setAttribute里
		// 因为最终JSP里面取值其实都是从request等域对象里面取~
		exposeModelAsRequestAttributes(model, request);

		// Expose helpers as request attributes, if any.
		// JstlView有实现此protected方法~
		exposeHelpers(request);

		// Determine the path for the request dispatcher.
		String dispatcherPath = prepareForRendering(request, response);

		// Obtain a RequestDispatcher for the target resource (typically a JSP).  注意：此处特指JSP
		// 就是一句话：request.getRequestDispatcher(path)
		RequestDispatcher rd = getRequestDispatcher(request, dispatcherPath);
		if (rd == null) {
			throw new ServletException("Could not get RequestDispatcher for [" + getUrl() +
					"]: Check that the corresponding file exists within your web application archive!");
		}

		// If already included or response already committed, perform include, else forward.
		//useInclude：若alwaysInclude==true或者该request是incluse请求或者response.isCommitted()==true
		// 那就走incluse，否则走forward~~~~~
		if (useInclude(request, response)) {
			response.setContentType(getContentType());
			if (logger.isDebugEnabled()) {
				logger.debug("Including [" + getUrl() + "]");
			}
			rd.include(request, response);
		}

		else {
			// Note: The forwarded resource is supposed to determine the content type itself.
			if (logger.isDebugEnabled()) {
				logger.debug("Forwarding to [" + getUrl() + "]");
			}
			rd.forward(request, response);
		}
	}

	// 拿到URL，做一个循环检查~~~  若是循环转发就报错~~
	protected String prepareForRendering(HttpServletRequest request, HttpServletResponse response)
			throws Exception {

		String path = getUrl();
		Assert.state(path != null, "'url' not set");

		if (this.preventDispatchLoop) {
			String uri = request.getRequestURI();
			if (path.startsWith("/") ? uri.equals(path) : uri.equals(StringUtils.applyRelativePath(uri, path))) {
				throw new ServletException("Circular view path [" + path + "]: would dispatch back " +
						"to the current handler URL [" + uri + "] again. Check your ViewResolver setup! " +
						"(Hint: This may be the result of an unspecified view, due to default view name generation.)");
			}
		}
		return path;
	}
}
```



# ModelAndViewContainer



```java
// @since 3.1
public class ModelAndViewContainer {
	// =================它所持有的这些属性还是蛮重要的=================
	// redirect时,是否忽略defaultModel 默认值是false：不忽略
	private boolean ignoreDefaultModelOnRedirect = false;
	// 此视图可能是个View，也可能只是个逻辑视图String
	@Nullable
	private Object view;
	// defaultModel默认的Model
	// 注意：ModelMap 只是个Map而已，但是实现类BindingAwareModelMap它却实现了org.springframework.ui.Model接口
	private final ModelMap defaultModel = new BindingAwareModelMap();
	// 重定向时使用的模型（提供set方法设置进来）
	@Nullable
	private ModelMap redirectModel;
	// 控制器是否返回重定向指令
	// 如：使用了前缀"redirect:xxx.jsp"这种，这个值就是true。然后最终是个RedirectView
	private boolean redirectModelScenario = false;
	// Http状态码
	@Nullable
	private HttpStatus status;
	
	private final Set<String> noBinding = new HashSet<>(4);
	private final Set<String> bindingDisabled = new HashSet<>(4);

	// 很容易想到，它和@SessionAttributes标记的元素有关
	private final SessionStatus sessionStatus = new SimpleSessionStatus();
	// 这个属性老重要了：标记handler是否**已经完成**请求处理
	// 在链式操作中，这个标记很重要
	private boolean requestHandled = false;
	...

	public void setViewName(@Nullable String viewName) {
		this.view = viewName;
	}
	public void setView(@Nullable Object view) {
		this.view = view;
	}
	// 是否是视图的引用
	public boolean isViewReference() {
		return (this.view instanceof String);
	}

	// 是否使用默认的Model
	private boolean useDefaultModel() {
		return (!this.redirectModelScenario || (this.redirectModel == null && !this.ignoreDefaultModelOnRedirect));
	}
	
	// 注意子方法和下面getDefaultModel()方法的区别
	public ModelMap getModel() {
		if (useDefaultModel()) { // 使用默认视图
			return this.defaultModel;
		} else {
			if (this.redirectModel == null) { // 若重定向视图为null，就new一个空的返回
				this.redirectModel = new ModelMap();
			}
			return this.redirectModel;
		}
	}
	// @since 4.1.4
	public ModelMap getDefaultModel() {
		return this.defaultModel;
	}

	// @since 4.3 可以设置响应码，最终和ModelAndView一起被View渲染时候使用
	public void setStatus(@Nullable HttpStatus status) {
		this.status = status;
	}

	// 以编程方式注册一个**不应**发生数据绑定的属性，对于随后声明的@ModelAttribute也是不能绑定的
	// 虽然方法是set 但内部是add哦  ~~~~
	public void setBindingDisabled(String attributeName) {
		this.bindingDisabled.add(attributeName);
	}
	public boolean isBindingDisabled(String name) {
		return (this.bindingDisabled.contains(name) || this.noBinding.contains(name));
	}
	// 注册是否应为相应的模型属性进行数据绑定
	public void setBinding(String attributeName, boolean enabled) {
		if (!enabled) {
			this.noBinding.add(attributeName);
		} else {
			this.noBinding.remove(attributeName);
		}
	}

	// 这个方法需要重点说一下：请求是否已在处理程序中完全处理
	// 举个例子：比如@ResponseBody标注的方法返回值，无需View继续去处理，所以就可以设置此值为true了
	// 说明：这个属性也就是可通过源生的ServletResponse、OutputStream来达到同样效果的
	public void setRequestHandled(boolean requestHandled) {
		this.requestHandled = requestHandled;
	}
	public boolean isRequestHandled() {
		return this.requestHandled;
	}

	// =========下面是Model的相关方法了==========
	// addAttribute/addAllAttributes/mergeAttributes/removeAttributes/containsAttribute
}
```

> 它维护了模型model：包括defaultModle和redirectModel
> defaultModel是默认使用的Model，redirectModel是用于传递redirect时的Model
> 在Controller处理器入参写了Model或ModelMap类型时候，实际传入的是defaultModel。
> - defaultModel它实际是BindingAwareModel，是个Map。而且继承了ModelMap又实现了Model接口，所以在处理器中使用Model或ModelMap时，其实都是使用同一个对象~~~
> - 可参考MapMethodProcessor，它最终调用的都是mavContainer.getModel()方法
>   若处理器入参类型是RedirectAttributes类型，最终传入的是redirectModel。
> - 至于为何实际传入的是defaultModel？？参考：RedirectAttributesMethodArgumentResolver，使用的是new RedirectAttributesModelMap(dataBinder)。
>   维护视图view（兼容支持逻辑视图名称）
>   维护是否redirect信息,及根据这个判断HandlerAdapter使用的是defaultModel或redirectModel
>   维护@SessionAttributes注解信息状态
>   维护handler是否处理标记（重要）



## Model

```java
//  @since 2.5.1 它是一个接口
public interface Model {
	...
	// addAttribute/addAllAttributes/mergeAttributes/containsAttribute
	...
	// Return the current set of model attributes as a Map.
	Map<String, Object> asMap();
}
```



#### RedirectAttributes



RedirectAttributesModelMap

#### ConcurrentModel

## ModelMap

##### BindingAwareModelMap



## ModelAndView

```java
public class ModelAndView {
	@Nullable
	private Object view; // 可以是View，也可以是String
	@Nullable
	private ModelMap model;

	// 显然，你也可以自己就放置好一个http状态码进去
	@Nullable
	private HttpStatus status;	
	// 标记这个实例是否被调用过clear()方法~~~
	private boolean cleared = false;

	// 总共这几个属性：它提供的构造函数非常的多  这里我就不一一列出
	public void setViewName(@Nullable String viewName) {
		this.view = viewName;
	}
	public void setView(@Nullable View view) {
		this.view = view;
	}
	@Nullable
	public String getViewName() {
		return (this.view instanceof String ? (String) this.view : null);
	}
	@Nullable
	public View getView() {
		return (this.view instanceof View ? (View) this.view : null);
	}
	public boolean hasView() {
		return (this.view != null);
	}
	public boolean isReference() {
		return (this.view instanceof String);
	}

	// protected方法~~~
	@Nullable
	protected Map<String, Object> getModelInternal() {
		return this.model;
	}
	public ModelMap getModelMap() {
		if (this.model == null) {
			this.model = new ModelMap();
		}
		return this.model;
	}

	// 操作ModelMap的一些方法如下：
	// addObject/addAllObjects

	public void clear() {
		this.view = null;
		this.model = null;
		this.cleared = true;
	}
	// 前提是：this.view == null 
	public boolean isEmpty() {
		return (this.view == null && CollectionUtils.isEmpty(this.model));
	}
	
	// 竟然用的was，歪果仁果然严谨  哈哈
	public boolean wasCleared() {
		return (this.cleared && isEmpty());
	}
}
```



ModelFactory