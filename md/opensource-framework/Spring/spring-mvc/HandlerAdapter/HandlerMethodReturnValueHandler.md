

# HandlerMethodReturnValueHandler

* HandlerMethodArgumentResolver [see](../HandlerMethod.md)
  * AbstractMessageConverterMethodArgumentResolver

```java
// @since 3.1  出现得相对还是比较晚的。因为`RequestMappingHandlerAdapter`也是这个时候才出来
// Strategy interface to handle the value returned from the invocation of a handler method
public interface HandlerMethodReturnValueHandler {

	// 每种处理器实现类，都对应着它能够处理的返回值类型~~~
	boolean supportsReturnType(MethodParameter returnType);

	// Handle the given return value by adding attributes to the model and setting a view or setting the
	// {@link ModelAndViewContainer#setRequestHandled} flag to {@code true} to indicate the response has been handled directly.
	// 简单的说就是处理返回值，可以处理着向Model里设置一个view
	// 或者ModelAndViewContainer#setRequestHandled设置true说我已经直接处理了，后续不要需要再继续渲染了~
	void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception;

}
```



### `AbstractMessageConverterMethodProcessor`#writeWithMessageConverters

```java
// @since 3.1  会发现它也处理请求，但是不是本文讨论的重点
//return values by writing to the response with {@link HttpMessageConverter HttpMessageConverters}
public abstract class AbstractMessageConverterMethodProcessor extends AbstractMessageConverterMethodArgumentResolver
		implements HandlerMethodReturnValueHandler {
	...
	// 此处我们只关注它处理返回值的和信访方法
	// Writes the given return type to the given output message
	// 从JavaDoc解释可以看出，它的作用很“单一“：就是把返回值写进output message~~~
	@SuppressWarnings({"rawtypes", "unchecked"})
	protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
			ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {

		Object body;
		Class<?> valueType;
		Type targetType;

		// 注意此处的特殊处理，相当于把所有的CharSequence类型的，都最终当作String类型处理的~
		if (value instanceof CharSequence) {
			body = value.toString();
			valueType = String.class;
			targetType = String.class;
		}
		// 我们本例；body为返回值对象  Person@5229
		// valueType为：class com.fsx.bean.Person
		// targetType：class com.fsx.bean.Person
		else {
			body = value;
			valueType = getReturnValueType(body, returnType);
	
			// 此处相当于兼容了泛型类型的处理
			targetType = GenericTypeResolver.resolveType(getGenericType(returnType), returnType.getContainingClass());
		}

		// 若返回值是个org.springframework.core.io.Resource  就走这里  此处忽略~~
		if (isResourceType(value, returnType)) {
			outputMessage.getHeaders().set(HttpHeaders.ACCEPT_RANGES, "bytes");
			if (value != null && inputMessage.getHeaders().getFirst(HttpHeaders.RANGE) != null &&
					outputMessage.getServletResponse().getStatus() == 200) {
				Resource resource = (Resource) value;
				try {
					List<HttpRange> httpRanges = inputMessage.getHeaders().getRange();
					outputMessage.getServletResponse().setStatus(HttpStatus.PARTIAL_CONTENT.value());
					body = HttpRange.toResourceRegions(httpRanges, resource);
					valueType = body.getClass();
					targetType = RESOURCE_REGION_LIST_TYPE;
				}
				catch (IllegalArgumentException ex) {
					outputMessage.getHeaders().set(HttpHeaders.CONTENT_RANGE, "bytes */" + resource.contentLength());
					outputMessage.getServletResponse().setStatus(HttpStatus.REQUESTED_RANGE_NOT_SATISFIABLE.value());
				}
			}
		}

		// selectedMediaType表示最终被选中的MediaType，毕竟请求放可能是接受N多种MediaType的~~~
		MediaType selectedMediaType = null;
		// 一般情况下 请求方很少会指定contentType的~~~
		// 如果请求方法指定了，就以它的为准，就相当于selectedMediaType 里面就被找打了
		// 否则就靠系统自己去寻找到一个最为合适的~~~
		MediaType contentType = outputMessage.getHeaders().getContentType();
		if (contentType != null && contentType.isConcrete()) {
			if (logger.isDebugEnabled()) {
				logger.debug("Found 'Content-Type:" + contentType + "' in response");
			}
			selectedMediaType = contentType;
		}
		else {
			HttpServletRequest request = inputMessage.getServletRequest();
			// 前面我们说了 若是谷歌浏览器  默认它的accept为：text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/s
			// 所以此处数组解析出来有7对
			List<MediaType> acceptableTypes = getAcceptableMediaTypes(request);

			// 这个方法就是从所有已经注册的转换器里面去找，看看哪些转换器.canWrite，然后把他们所支持的MediaType都加入进来~~~
			// 比如此例只能匹配到MappingJackson2HttpMessageConverter，所以匹配上的有application/json、application/*+json两个
			List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType, targetType);
			
			// 这个异常应该我们经常碰到：有body体，但是并没有能够支持的转换器，就是这额原因~~~
			if (body != null && producibleTypes.isEmpty()) {
				throw new HttpMessageNotWritableException("No converter found for return value of type: " + valueType);
			}

			// 下面相当于从浏览器可议接受的MediaType里面，最终抉择出N个来
			// 原理也非常简单：你能接受的isCompatibleWith上了我能处理的，那咱们就好说，处理就完了
			List<MediaType> mediaTypesToUse = new ArrayList<>();
			for (MediaType requestedType : acceptableTypes) {
				for (MediaType producibleType : producibleTypes) {
					if (requestedType.isCompatibleWith(producibleType)) {
					// 从两个中选择一个最匹配的  主要是根据q值来比较  排序
					// 比如此例，最终匹配上的有两个：application/json;q=0.8和application/*+json;q=0.8
						mediaTypesToUse.add(getMostSpecificMediaType(requestedType, producibleType));
					}
				}
			}
			
			// 这个异常也不少见，比如此处如果没有导入Jackson相关依赖包
			// 就会抛出这个异常了：HttpMediaTypeNotAcceptableException：Could not find acceptable representation
			if (mediaTypesToUse.isEmpty()) {
				if (body != null) {
					throw new HttpMediaTypeNotAcceptableException(producibleTypes);
				}
				if (logger.isDebugEnabled()) {
					logger.debug("No match for " + acceptableTypes + ", supported: " + producibleTypes);
				}
				return;
			}
			
			// 根据Q值进行排序：
			MediaType.sortBySpecificityAndQuality(mediaTypesToUse);
			// 因为已经排过
			for (MediaType mediaType : mediaTypesToUse) {
				if (mediaType.isConcrete()) {
					selectedMediaType = mediaType;
					break;
				}
				else if (mediaType.isPresentIn(ALL_APPLICATION_MEDIA_TYPES)) {
					selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
					break;
				}
			}

			if (logger.isDebugEnabled()) {
				logger.debug("Using '" + selectedMediaType + "', given " +
						acceptableTypes + " and supported " + producibleTypes);
			}
		}

		// 最终的最终 都会找到一个决定write的类型，必粗此处为：application/json;q=0.8
		//  因为最终会决策出来一个MediaType，所以此处就是要根据此MediaType找到一个合适的消息转换器，把body向outputstream写进去~~~
		// 注意此处：是RequestResponseBodyAdviceChain执行之处~~~~
		if (selectedMediaType != null) {
			selectedMediaType = selectedMediaType.removeQualityValue();
			for (HttpMessageConverter<?> converter : this.messageConverters) {

		
				// 从这个判断可以看出 ，处理body里面内容，GenericHttpMessageConverter类型的转换器是优先级更高，优先去处理的
				GenericHttpMessageConverter genericConverter = (converter instanceof GenericHttpMessageConverter ? (GenericHttpMessageConverter<?>) converter : null);
				if (genericConverter != null ? ((GenericHttpMessageConverter) converter).canWrite(targetType, valueType, selectedMediaType) :
						converter.canWrite(valueType, selectedMediaType)) {

					// 在写body之前执行~~~~  会调用我们注册的所有的合适的ResponseBodyAdvice#beforeBodyWrite方法
					// 相当于在写之前，我们可以介入对body体进行处理
					body = getAdvice().beforeBodyWrite(body, returnType, selectedMediaType,
							(Class<? extends HttpMessageConverter<?>>) converter.getClass(),
							inputMessage, outputMessage);
					if (body != null) {
						Object theBody = body;
						LogFormatUtils.traceDebug(logger, traceOn ->
								"Writing [" + LogFormatUtils.formatValue(theBody, !traceOn) + "]");

						// 给响应Response设置一个Content-Disposition的请求头（若需要的话）  若之前已经设置过了，此处将什么都不做
						// 比如我们常见的：response.setHeader("Content-Disposition", "attachment; filename=" + java.net.URLEncoder.encode(fileName, "UTF-8"));
						//Content-disposition 是 MIME 协议的扩展，MIME 协议指示 MIME 用户代理如何显示附加的文件。
						// 当 Internet Explorer 接收到头时，它会激活文件下载对话框，它的文件名框自动填充了头中指定的文件名
						addContentDispositionHeader(inputMessage, outputMessage);
						if (genericConverter != null) {
							genericConverter.write(body, targetType, selectedMediaType, outputMessage);
						}
						else {
							((HttpMessageConverter) converter).write(body, selectedMediaType, outputMessage);
						}
					}
					// 如果return null，body里面是null 那就啥都不写，输出一个debug日志即可~~~~
					else {
						if (logger.isDebugEnabled()) {
							logger.debug("Nothing to write: null body");
						}
					}

					// 这一句表示：只要一个一个消息转换器处理了，就立马停止~~~~
					return;
				}
			}
		}

		if (body != null) {
			throw new HttpMediaTypeNotAcceptableException(this.allSupportedMediaTypes);
		}
	}
	...
}
```



#### `RequestResponseBodyMethodProcessor`

@ResponseBody

```java
public class RequestResponseBodyMethodProcessor extends AbstractMessageConverterMethodProcessor {

	// 显然可以发现，方法上或者类上标注有@ResponseBody都是可以的~~~~
	// 这也就是为什么现在@RestController可以代替我们的的@Controller + @ResponseBody生效了
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		return (AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), ResponseBody.class) ||
				returnType.hasMethodAnnotation(ResponseBody.class));
	}

	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest)
			throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
		
		// 首先就标记：此请求已经被处理了~~~
		mavContainer.setRequestHandled(true);
		ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
		ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);

		// Try even with null return value. ResponseBodyAdvice could get involved.
		// 这个方法是核心，也会处理null值~~~  这里面一些Advice会生效~~~~
		// 会选择到合适的HttpMessageConverter,然后进行消息转换~~~~（这里只指写~~~）  这个方法在父类上，是非常核心关键自然也是非常复杂的~~~
		writeWithMessageConverters(returnValue, returnType, inputMessage, outputMessage);
	}
}

// @since 3.1
public class RequestResponseBodyMethodProcessor extends AbstractMessageConverterMethodProcessor {

	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		return parameter.hasParameterAnnotation(RequestBody.class);
	}
	// 类上或者方法上标注了@ResponseBody注解都行
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		return (AnnotatedElementUtils.hasAnnotation(returnType.getContainingClass(), ResponseBody.class) || returnType.hasMethodAnnotation(ResponseBody.class));
	}
	
	// 这是处理入参封装校验的入口，也是本文关注的焦点
	@Override
	public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {
		
		// 它是支持`Optional`容器的
		parameter = parameter.nestedIfOptional();
		// 使用消息转换器HttpInputMessage把request请求转换出来，拿到值~~~
		// 此处注意：比如本例入参是Person类，所以经过这里处理会生成一个空的Person对象出来（反射）
		Object arg = readWithMessageConverters(webRequest, parameter, parameter.getNestedGenericParameterType());

		// 获取到入参的名称,其实不叫形参名字，应该叫objectName给校验时用的
		// 请注意：这里的名称是类名首字母小写，并不是你方法里写的名字。比如本利若形参名写为personAAA，但是name的值还是person
		// 但是注意：`parameter.getParameterName()`的值可是personAAA
		String name = Conventions.getVariableNameForParameter(parameter);

		// 只有存在binderFactory才会去完成自动的绑定、校验~
		// 此处web环境为：ServletRequestDataBinderFactory
		if (binderFactory != null) {
			WebDataBinder binder = binderFactory.createBinder(webRequest, arg, name);

			// 显然传了参数才需要去绑定校验嘛
			if (arg != null) {

				// 这里完成数据绑定+数据校验~~~~~（绑定的错误和校验的错误都会放进Errors里）
				// Applicable：适合
				validateIfApplicable(binder, parameter);

				// 若有错误消息hasErrors()，并且仅跟着的一个参数不是Errors类型，Spring MVC会主动给你抛出MethodArgumentNotValidException异常
				// 否则，调用者自行处理
				if (binder.getBindingResult().hasErrors() && isBindExceptionRequired(binder, parameter)) {
					throw new MethodArgumentNotValidException(parameter, binder.getBindingResult());
				}
			}
		
			// 把错误消息放进去 证明已经校验出错误了~~~
			// 后续逻辑会判断MODEL_KEY_PREFIX这个key的~~~~
			if (mavContainer != null) {
				mavContainer.addAttribute(BindingResult.MODEL_KEY_PREFIX + name, binder.getBindingResult());
			}
		}

		return adaptArgumentIfNecessary(arg, parameter);
	}

	// 校验，如果合适的话。使用WebDataBinder，失败信息最终也都是放在它身上~  本方法是本文关注的焦点
	// 入参：MethodParameter parameter
	protected void validateIfApplicable(WebDataBinder binder, MethodParameter parameter) {
		// 拿到标注在此参数上的所有注解们（比如此处有@Valid和@RequestBody两个注解）
		Annotation[] annotations = parameter.getParameterAnnotations();
		for (Annotation ann : annotations) {
			// 先看看有木有@Validated
			Validated validatedAnn = AnnotationUtils.getAnnotation(ann, Validated.class);

			// 这个里的判断是关键：可以看到标注了@Validated注解 或者注解名是以Valid打头的 都会有效哦
			//注意：这里可没说必须是@Valid注解。实际上你自定义注解，名称只要一Valid开头都成~~~~~
			if (validatedAnn != null || ann.annotationType().getSimpleName().startsWith("Valid")) {
				// 拿到分组group后，调用binder的validate()进行校验~~~~
				// 可以看到：拿到一个合适的注解后，立马就break了~~~
				// 所以若你两个主机都标注@Validated和@Valid，效果是一样滴~
				Object hints = (validatedAnn != null ? validatedAnn.value() : AnnotationUtils.getValue(ann));
				Object[] validationHints = (hints instanceof Object[] ? (Object[]) hints : new Object[] {hints});
				binder.validate(validationHints);
				break;
			}
		}
	}
	...
}
```



### AsyncHandlerMethodReturnValueHandler

```java
// @since 4.2
// 支持异步类型的返回值处理程序。此类返回值类型需要优先处理，以便异步值可以“展开”。
// 异步实现此接口并不是必须的，但是若你需要在处理程序之前执行，就需要实现这个接口了~~~
// 因为默认情况下：我们自定义的Handler它都是在内置的Handler后面去执行的~~~~
public interface AsyncHandlerMethodReturnValueHandler extends HandlerMethodReturnValueHandler {
	// 给定的返回值是否表示异步计算
	boolean isAsyncReturnValue(@Nullable Object returnValue, MethodParameter returnType);
}
```











### MapMethodProcessor

```java
// @since 3.1
public class MapMethodProcessor implements HandlerMethodArgumentResolver, HandlerMethodReturnValueHandler {

	@Override
	public boolean supportsParameter(MethodParameter parameter) {
		return Map.class.isAssignableFrom(parameter.getParameterType());
	}

	@Override
	@Nullable
	public Object resolveArgument(MethodParameter parameter, @Nullable ModelAndViewContainer mavContainer,
			NativeWebRequest webRequest, @Nullable WebDataBinderFactory binderFactory) throws Exception {

		Assert.state(mavContainer != null, "ModelAndViewContainer is required for model exposure");
		return mavContainer.getModel();
	}

	// ==================上面是处理入参的，不是本文的重点~~~====================
	// 显然只有当你的返回值是个Map时候，此处理器才会生效
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		return Map.class.isAssignableFrom(returnType.getParameterType());
	}

	@Override
	@SuppressWarnings({"unchecked", "rawtypes"})
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
		
		// 做的事非常简单，仅仅只是把我们的Map放进Model里面~~~
		// 但是此处需要注意的是：它并没有setViewName，所以它此时是没有视图名称的~~~
		// ModelAndViewContainer#setRequestHandled(true) 所以后续若还有处理器可以继续处理
		if (returnValue instanceof Map){
			mavContainer.addAllAttributes((Map) returnValue);
		}
	}

}
```



### ViewNameMethodReturnValueHandler

```java
// 可以直接返回一个视图名，最终会交给`RequestToViewNameTranslator`翻译一下~~~
public class ViewNameMethodReturnValueHandler implements HandlerMethodReturnValueHandler {

	// Spring4.1之后支持自定义重定向的匹配规则
	// Spring4.1之前还只能支持redirect:这个固定的前缀~~~~
	private String[] redirectPatterns;
	public void setRedirectPatterns(String... redirectPatterns) {
		this.redirectPatterns = redirectPatterns;
	}
	public String[] getRedirectPatterns() {
		return this.redirectPatterns;
	}
	protected boolean isRedirectViewName(String viewName) {
		return (PatternMatchUtils.simpleMatch(this.redirectPatterns, viewName) || viewName.startsWith("redirect:"));
	}


	// 支持void和CharSequence类型（子类型）
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		Class<?> paramType = returnType.getParameterType();
		return (void.class == paramType || CharSequence.class.isAssignableFrom(paramType));
	}

	// 注意：若返回值是void，此方法都不会进来
	@Override
	public void handleReturnValue(Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

		// 显然如果是void，returnValue 为null，不会走这里。
		// 也就是说viewName设置不了，所以会出现和上诉一样的循环报错~~~~~ 因此不要单独使用
		if (returnValue instanceof CharSequence) {
			String viewName = returnValue.toString();
			mavContainer.setViewName(viewName);
	
			// 做一个处理：如果是重定向的view，那就
			if (isRedirectViewName(viewName)) {
				mavContainer.setRedirectModelScenario(true);
			}
		}
		// 下面这都是不可达的~~~~~setRedirectModelScenario(true)标记一下
		// 此处不仅仅是else，而是还有个！=null的判断  
		// 那是因为如果是void的话这里返回值是null，属于正常的~~~~ 只是什么都不做而已~（viewName也没有设置哦~~~）
		else if (returnValue != null) {
			// should not happen
			throw new UnsupportedOperationException("Unexpected return type: " +
					returnType.getParameterType().getName() + " in method: " + returnType.getMethod());
		}
	}

}
```



### ViewMethodReturnValueHandler

MappingJackson2JsonView、AbstractPdfView、MarshallingView、RedirectView、JstlView

```java
// javadoc上有说明：此处理器需要配置在支持`@ModelAttribute`或者`@ResponseBody`的处理器们前面。防止它被取代~~~~
public class ViewMethodReturnValueHandler implements HandlerMethodReturnValueHandler {
	
	// 处理素有的View.class类型
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		return View.class.isAssignableFrom(returnType.getParameterType());
	}

	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
			
		// 这个处理逻辑几乎完全同上
		// 最终也是为了mavContainer.setView(view);
		// 也会对重定向视图进行特殊的处理~~~~~~
		if (returnValue instanceof View) {
			View view = (View) returnValue;
			mavContainer.setView(view);
			if (view instanceof SmartView && ((SmartView) view).isRedirectView()) {
				mavContainer.setRedirectModelScenario(true);
			}
		}
		else if (returnValue != null) {
			// should not happen
			throw new UnsupportedOperationException("Unexpected return type: " +
					returnType.getParameterType().getName() + " in method: " + returnType.getMethod());
		}
	}
}
```



### HttpHeadersReturnValueHandler

```java
public class HttpHeadersReturnValueHandler implements HandlerMethodReturnValueHandler {

	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		return HttpHeaders.class.isAssignableFrom(returnType.getParameterType());
	}

	@Override
	@SuppressWarnings("resource")
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

		// 请注意这里：已经标记该请求已经被处理过了~~~~~
		mavContainer.setRequestHandled(true);

		Assert.state(returnValue instanceof HttpHeaders, "HttpHeaders expected");
		HttpHeaders headers = (HttpHeaders) returnValue;

		// 返回值里自定义返回的响应头。这里会帮你设置到HttpServletResponse 里面去的~~~~
		if (!headers.isEmpty()) {
			HttpServletResponse servletResponse = webRequest.getNativeResponse(HttpServletResponse.class);
			Assert.state(servletResponse != null, "No HttpServletResponse");
			ServletServerHttpResponse outputMessage = new ServletServerHttpResponse(servletResponse);
			outputMessage.getHeaders().putAll(headers);
			outputMessage.getBody();  // flush headers
		}
	}

}

```



### ModelMethodProcessor

### ModelAndViewMethodReturnValueHandler

> **ModelAndView = model + view + HttpStatus**

```java
public class ModelAndViewMethodReturnValueHandler implements HandlerMethodReturnValueHandler {

	// Spring4.1后一样  增加自定义重定向前缀的支持
	@Nullable
	private String[] redirectPatterns;
	public void setRedirectPatterns(@Nullable String... redirectPatterns) {
		this.redirectPatterns = redirectPatterns;
	}
	@Nullable
	public String[] getRedirectPatterns() {
		return this.redirectPatterns;
	}
	protected boolean isRedirectViewName(String viewName) {
		return (PatternMatchUtils.simpleMatch(this.redirectPatterns, viewName) || viewName.startsWith("redirect:"));
	}


	// 显然它只处理ModelAndView这种类型~
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		return ModelAndView.class.isAssignableFrom(returnType.getParameterType());
	}

	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

		// 如果调用者返回null 那就标注此请求被处理过了~~~~ 不需要再渲染了
		// 浏览器的效果就是：一片空白
		if (returnValue == null) {
			mavContainer.setRequestHandled(true);
			return;
		}

		ModelAndView mav = (ModelAndView) returnValue;
	
		// isReference()方法为：(this.view instanceof String)
		// 这里专门处理视图就是一个字符串的情况，else是处理视图是个View对象的情况
		if (mav.isReference()) {
			String viewName = mav.getViewName();
			mavContainer.setViewName(viewName);
			if (viewName != null && isRedirectViewName(viewName)) {
				mavContainer.setRedirectModelScenario(true);
			}
		}
		// 处理view  顺便处理重定向
		else {
			View view = mav.getView();
			mavContainer.setView(view);
			
			// 此处所有的view，只有RedirectView的isRedirectView()才是返回true，其它都是false
			if (view instanceof SmartView && ((SmartView) view).isRedirectView()) {
				mavContainer.setRedirectModelScenario(true);
			}
		}
		// 把status和model都设置进去
		mavContainer.setStatus(mav.getStatus());
		mavContainer.addAllAttributes(mav.getModel());
	}
}
```

### ModelAndViewResolverMethodReturnValueHandler

#### ModelAndViewResolver

### ModelAttributeMethodProcessor

#### ServletModelAttributeMethodProcessor



### StreamingResponseBodyReturnValueHandler

```java
public class StreamingResponseBodyReturnValueHandler implements HandlerMethodReturnValueHandler {

	// 显然这里支持返回值直接是StreamingResponseBody类型，也支持你用`ResponseEntity`在包一层
	// ResponseEntity的泛型类型必须是StreamingResponseBody类型~~~
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		if (StreamingResponseBody.class.isAssignableFrom(returnType.getParameterType())) {
			return true;
		} else if (ResponseEntity.class.isAssignableFrom(returnType.getParameterType())) {
			Class<?> bodyType = ResolvableType.forMethodParameter(returnType).getGeneric().resolve();
			return (bodyType != null && StreamingResponseBody.class.isAssignableFrom(bodyType));
		}
		return false;
	}

	@Override
	@SuppressWarnings("resource")
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

		// 从这句代码也可以看出，只有返回值为null了，它才关闭，否则可以持续不断的向response里面写东西
		if (returnValue == null) {
			mavContainer.setRequestHandled(true);
			return;
		}

		HttpServletResponse response = webRequest.getNativeResponse(HttpServletResponse.class);
		Assert.state(response != null, "No HttpServletResponse");
		ServerHttpResponse outputMessage = new ServletServerHttpResponse(response);

		// 从ResponseEntity里body提取出来~~~~~
		if (returnValue instanceof ResponseEntity) {
			ResponseEntity<?> responseEntity = (ResponseEntity<?>) returnValue;
			response.setStatus(responseEntity.getStatusCodeValue());
			outputMessage.getHeaders().putAll(responseEntity.getHeaders());
			returnValue = responseEntity.getBody();
			if (returnValue == null) {
				mavContainer.setRequestHandled(true);
				outputMessage.flush();
				return;
			}
		}

		ServletRequest request = webRequest.getNativeRequest(ServletRequest.class);
		Assert.state(request != null, "No ServletRequest");
		ShallowEtagHeaderFilter.disableContentCaching(request); // 禁用内容缓存

		Assert.isInstanceOf(StreamingResponseBody.class, returnValue, "StreamingResponseBody expected");
		StreamingResponseBody streamingBody = (StreamingResponseBody) returnValue;

		// 最终也是开启了一个Callable 任务，最后交给WebAsyncUtils去执行的~~~~
		Callable<Void> callable = new StreamingResponseBodyTask(outputMessage.getBody(), streamingBody);

		// WebAsyncUtils.getAsyncManager得到的是一个`WebAsyncManager`对象
		// startCallableProcessing会把callable任务都包装成一个`WebAsyncTask`,最终交给`AsyncTaskExecutor`执行
		// 至于异步的详细执行原理，请参考上面的相关博文，此处只点一下~~~~
		WebAsyncUtils.getAsyncManager(webRequest).startCallableProcessing(callable, mavContainer);
	}

	// 这个任务很简单，实现了Callable的call方法，它就是相当于启一个线程，把本次body里面的内容写进response输出流里面~~~
	// 但是此时输出流并不会关闭~~~~
	private static class StreamingResponseBodyTask implements Callable<Void> {

		private final OutputStream outputStream;

		private final StreamingResponseBody streamingBody;

		public StreamingResponseBodyTask(OutputStream outputStream, StreamingResponseBody streamingBody) {
			this.outputStream = outputStream;
			this.streamingBody = streamingBody;
		}

		@Override
		public Void call() throws Exception {
			this.streamingBody.writeTo(this.outputStream);
			return null;
		}
	}

}
```



### DeferredResultMethodReturnValueHandler

```java
public class DeferredResultMethodReturnValueHandler implements HandlerMethodReturnValueHandler {

	// 它支持处理丰富的数据类型
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		Class<?> type = returnType.getParameterType();
		return (DeferredResult.class.isAssignableFrom(type) ||
				ListenableFuture.class.isAssignableFrom(type) ||
				CompletionStage.class.isAssignableFrom(type));
	}

	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

		// 一样的  只有返回null了才代表此请求处理完成了
		if (returnValue == null) {
			mavContainer.setRequestHandled(true);
			return;
		}

		DeferredResult<?> result;

		// 此处是适配器模式的使用，最终都适配成了一个DeferredResult（使用的内部类实现的~~~）
		if (returnValue instanceof DeferredResult) {
			result = (DeferredResult<?>) returnValue;
		} else if (returnValue instanceof ListenableFuture) {
			result = adaptListenableFuture((ListenableFuture<?>) returnValue);
		} else if (returnValue instanceof CompletionStage) {
			result = adaptCompletionStage((CompletionStage<?>) returnValue);
		} else {
			// Should not happen...
			throw new IllegalStateException("Unexpected return value type: " + returnValue);
		}
		// 此处调用的异步方法是：startDeferredResultProcessing
		WebAsyncUtils.getAsyncManager(webRequest).startDeferredResultProcessing(result, mavContainer);
	}

	// 下为两匿名内部实现类做的兼容适配、兼容处理~~~~~非常的简单~~~~
	private DeferredResult<Object> adaptListenableFuture(ListenableFuture<?> future) {
		DeferredResult<Object> result = new DeferredResult<>();
		future.addCallback(new ListenableFutureCallback<Object>() {
			@Override
			public void onSuccess(@Nullable Object value) {
				result.setResult(value);
			}
			@Override
			public void onFailure(Throwable ex) {
				result.setErrorResult(ex);
			}
		});
		return result;
	}

	private DeferredResult<Object> adaptCompletionStage(CompletionStage<?> future) {
		DeferredResult<Object> result = new DeferredResult<>();
		future.handle((BiFunction<Object, Throwable, Object>) (value, ex) -> {
			if (ex != null) {
				result.setErrorResult(ex);
			}
			else {
				result.setResult(value);
			}
			return null;
		});
		return result;
	}

}
```



### CallableMethodReturnValueHandler

### ResponseBodyEmitterReturnValueHandler

### AsyncTaskMethodReturnValueHandler

### HandlerMethodReturnValueHandlerComposite：处理器合成

```java
// 首先发现，它也实现了接口HandlerMethodReturnValueHandler 
// 它会缓存以前解析的返回类型以加快查找速度
public class HandlerMethodReturnValueHandlerComposite implements HandlerMethodReturnValueHandler {

	private final List<HandlerMethodReturnValueHandler> returnValueHandlers = new ArrayList<>();
	// 返回的是一个只读视图
	public List<HandlerMethodReturnValueHandler> getHandlers() {
		return Collections.unmodifiableList(this.returnValueHandlers);
	}
	public HandlerMethodReturnValueHandlerComposite addHandler(HandlerMethodReturnValueHandler handler) {
		this.returnValueHandlers.add(handler);
		return this;
	}
	public HandlerMethodReturnValueHandlerComposite addHandlers( @Nullable List<? extends HandlerMethodReturnValueHandler> handlers) {
		if (handlers != null) {
			this.returnValueHandlers.addAll(handlers);
		}
		return this;
	}
	
	// 由这两个可议看出，但凡有一个Handler支持处理这个返回值，就是支持的~~~
	@Override
	public boolean supportsReturnType(MethodParameter returnType) {
		return getReturnValueHandler(returnType) != null;
	}
	@Nullable
	private HandlerMethodReturnValueHandler getReturnValueHandler(MethodParameter returnType) {
		for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
			if (handler.supportsReturnType(returnType)) {
				return handler;
			}
		}
		return null;
	}

	// 这里就是处理返回值的核心内容~~~~~
	@Override
	public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {

		// selectHandler选择收个匹配的Handler来处理这个返回值~~~~ 若一个都木有找到  抛出异常吧~~~~
		// 所有很重要的一个方法是它：selectHandler()  它来匹配，以及确定优先级
		HandlerMethodReturnValueHandler handler = selectHandler(returnValue, returnType);
		if (handler == null) {
			throw new IllegalArgumentException("Unknown return value type: " + returnType.getParameterType().getName());
		}
		handler.handleReturnValue(returnValue, returnType, mavContainer, webRequest);
	}

	// 根据返回值，以及返回类型  来找到一个最为合适的HandlerMethodReturnValueHandler
	@Nullable
	private HandlerMethodReturnValueHandler selectHandler(@Nullable Object value, MethodParameter returnType) {
		// 这个和我们上面的就对应上了  第一步去判断这个返回值是不是一个异步的value（AsyncHandlerMethodReturnValueHandler实现类只能我们自己来写~）
		boolean isAsyncValue = isAsyncReturnValue(value, returnType);
		for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
			// 如果判断发现这个值是异步的value，那它显然就只能交给你自己定义的异步处理器处理了，别的处理器肯定就靠边站~~~~~
			if (isAsyncValue && !(handler instanceof AsyncHandlerMethodReturnValueHandler)) {
				continue;
			}
			if (handler.supportsReturnType(returnType)) {
				return handler;
			}
		}
		return null;
	}
	private boolean isAsyncReturnValue(@Nullable Object value, MethodParameter returnType) {
		for (HandlerMethodReturnValueHandler handler : this.returnValueHandlers) {
			if (handler instanceof AsyncHandlerMethodReturnValueHandler && ((AsyncHandlerMethodReturnValueHandler) handler).isAsyncReturnValue(value, returnType)) {
				return true;
			}
		}
		return false;
	}
}
```
