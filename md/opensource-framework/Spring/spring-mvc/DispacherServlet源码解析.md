# DispacherServlet

DispacherServlet ： 接收 request 

IOC容器启动时就会调用 onRefresh方法，
请求处理 doService => doDispatch  
HandlerMapping : url 与 方法 对应关系  
HandlerMapping 和 HandlerAdapter 都是通过注解扫描获得

## 初始化 onRefresh

```java
protected void onRefresh(ApplicationContext context){
    initStrategies(context);
}

protected void initStrategies(ApplicationContext context) {
    // 请求解析
    initMultipartResolver(context);
    // 多语言，国际化
    initLocaleResolver(context);
    // 主题 view 层
    initThemeResolver(context);
    // 解析 url 和 method 的关联关系 xxxx
    initHandlerMappings(context);
    // 适配器（匹配的过程：参数，类型）
    initHandlerAdapters(context);
    initHandlerExceptionResolvers(context);
    // 视图转发(根据视图名字匹配到一个具体的模板)
    initRequestToViewNameTranslator(context);
    // 解析模板中的内容（拿到服务器传过来的数据，生成HTML代码）
    initViewResolvers(context);
    initFlashMapManager(context);
}
```

### initHandlerMappings

### initHandlerAdapters

### initViewResolvers



## 请求处理 doDispatch

### 流程

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
	...
	try {
		ModelAndView mv = null;
		Exception dispatchException = null;
		try {
			processedRequest = checkMultipart(request);
			multipartRequestParsed = (processedRequest != request);
			// 1.调用 handlerMapping 获取 handlerChain
			mappedHandler = getHandler(processedRequest);
			if (mappedHandler == null || mappedHandler.getHandler() == null) {
				noHandlerFound(processedRequest, response);
				return;
			}
			// 2.获取支持该 handler 解析的 HandlerAdapter
			HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());
			...
			// 3.使用 HandlerAdapter 完成 handler 处理
			mv = ha.handle(processedRequest, response, mappedHandler.getHandler());
			if (asyncManager.isConcurrentHandlingStarted()) {
				return;
			}
			// view 格式处理
			applyDefaultViewName(request, mv);
			mappedHandler.applyPostHandle(processedRequest, response, mv);
		}
		catch (Exception ex) {
			dispatchException = ex;
		}
        // // 4.视图处理(页面渲染) 
		processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
	}
	...
}
```

调用HandlerMapping得到HandlerChain(Handler+Interceptor)
调用HandlerAdapter执行handle过程(参数解析 过程调用)
异常处理(过程类似HanderAdapter)
调用ViewResolver进行视图解析
渲染视图

### HandlerAdapter#handle(...)

### processDispatchResult 视图处理(页面渲染)

#### render

##### resolveViewName

```java
@Nullable
	protected View resolveViewName(String viewName, @Nullable Map<String, Object> model,
			Locale locale, HttpServletRequest request) throws Exception {

		if (this.viewResolvers != null) {
			// viewResolver 处理
            for (ViewResolver viewResolver : this.viewResolvers) {
				View view = viewResolver.resolveViewName(viewName, locale);
				if (view != null) {
					return view;
				}
			}
		}
		return null;
	}
```



## HandlerAdapter

### RequestMappingHandlerAdapter

```
adapter.supports(handler)
       handler instanceof HandlerMethod
```

**HandlerMethod**



#### ha.handle(processedRequest, response, mappedHandler.getHandler())

invokeHandlerMethod

ServletInvocableHandlerMethod

#### HandlerMethodArgumentResolver

```java
supportsParameter(MethodParameter)
MethodParameter:parameterAnnotations  1:N
resolveArgument(MethodParameter, ModelAndViewContainer,NativeWebRequest, WebDataBinderFactory)
```





* PathVariableMethodArgumentResolver
* RequestParamMethodArgumentResolver

#### HandlerMethodReturnValueHandler

```
boolean supportsReturnType(MethodParameter returnType);

void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
			ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception;
```

* RequestResponseBodyMethodProcessor

HttpMessageConverter

​      