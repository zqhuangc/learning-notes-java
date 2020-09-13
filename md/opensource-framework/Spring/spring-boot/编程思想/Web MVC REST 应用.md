#### Spring Web MVC REST 内容协商处理流程

##### 源码分析

##### 理解请求的媒体类型

经过  ContentNegotiationManager 的  `ContentNegotiationStrategy 解析请求中的媒体类型，比如： Accept 请求头
如果成功解析，返回合法 MediaType 列表
否则，返回单元素  */* 媒体类型列表 -  MediaType.ALL

#####理解可生成的媒体类型
返回  @Controller HandlerMethod @RequestMapping.produces() 属性所指定的 MediaType 列表：
如果  @RequestMapping.produces() 存在，返回指定 MediaType 列表
否则，返回已注册的  HttpMessageConverter 列表中支持的  MediaType 列表

##### 理解 @RequestMapping#consumes
用于  @Controller HandlerMethod 匹配：
如果请求头  Content-Type 媒体类型兼容  @RequestMapping.consumes() 属性，执行该  HandlerMethod
否则  HandlerMethod 不会被调用

##### 理解 @RequestMapping#produces
用于获取可生成的 MediaType 列表
如果该列表与请求的媒体类型兼容，执行第一个兼容  HttpMessageConverter 的实现，默认
@RequestMapping#produces 内容到响应头  Content-Type
否则，抛出  HttpMediaTypeNotAcceptableException , HTTP Status Code : 415



#### 扩展 REST 内容协商



##### 自定义 `HttpMessageConverter`

###### 需求

实现` Content-Type` 为 ` text/properties` 媒体类型的  `HttpMessageConverter`
###### 实现步骤
* 实现 `HttpMessageConverter `-  `PropertiesHttpMessageConverter`
* 配置  `PropertiesHttpMessageConverter `到  `WebMvcConfigurer#extendMessageConverters`



##### 自定义 `HandlerMethodArgumentResolver`

######需求
* 不依赖  `@RequestBody `， 实现  `Properties `格式请求内容，解析为  `Properties `对象的方法参数
* 复用 `PropertiesHttpMessageConverter`

###### 实现步骤

* 实现 `HandlerMethodArgumentResolver `-  `PropertiesHandlerMethodArgumentResolver`
* ~~配置  `PropertiesHandlerMethodArgumentResolver `到  `WebMvcConfigurer#addArgumentResolvers`~~
* `RequestMappingHandlerAdapter#setArgumentResolvers`



##### 自定义 `HandlerMethodReturnValueHandler`

######需求
* 不依赖  `@ResponseBody `，实现 ` Properties `类型方法返回值，转化为  `Properties` 格式内容响应内容
* 复用 `PropertiesHttpMessageConverter`

###### 实现步骤
* 实现 HandlerMethodReturnValueHandler -  PropertiesHandlerMethodReturnValueHandler
* ~~配置  `PropertiesHandlerMethodReturnValueHandler` 到  `WebMvcConfigurer#addReturnValueHandlers`~~
* `RequestMappingHandlerAdapter#setReturnValueHandlers`



注解驱动 @CrossOrigin
代码驱动 WebMvcConfigurer#addCorsMappings