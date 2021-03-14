浏览器只能通过 HTTP GET 或 HTTP POST 提交表单数据，但非浏览器客户机也可以使用 HTTP PUT、 PATCH 和 DELETE。Servlet API 要求 ServletRequest.getParameter * ()方法只支持对 HTTP POST 的表单字段访问。

Spring-web 模块提供 `FormContentFilter `来拦截内容类型为 application/x-www-form-urlencoded 的 HTTP PUT、 PATCH 和 DELETE 请求，从请求体中读取表单数据，并封装 ServletRequest 以使表单数据通过 ServletRequest.getparameter * ()方法家族可用。





#### Forwarded Headers

当请求通过代理(比如负载均衡器)时，主机、端口和方案可能会发生变化，这使得从客户端角度创建指向正确主机、端口和方案的链接成为一个挑战。

RFC 7239定义了转发 HTTP 头，代理可以使用它来提供有关原始请求的信息。还有其他非标准头，包括 X-Forwarded-Host、 X-Forwarded-Port、 X-Forwarded-Proto、 X-Forwarded-Ssl 和 X-Forwarded-Prefix。

`ForwardedHeaderFilter `是一个 Servlet 过滤器，它修改请求，以便 a)基于 Forwarded header 更改主机、端口和方案，b)删除这些 header 以消除进一步的影响。过滤器依赖于包装请求，因此它必须排在其他过滤器之前，比如 `RequestContextFilter`，这些过滤器应该能够处理被修改的请求，而不是原始请求。

由于应用程序不能知道标头是否按预期由代理添加，还是由恶意客户机添加，因此转发标头需要考虑安全性问题。这就是为什么应该配置位于信任边界的代理来删除来自外部的不受信任的 Forwarded 头。还可以使用 removeOnly = true 配置 forwaredheaderfilter，在这种情况下，它删除但不使用标头。

为了支持异步请求和错误分派，这个过滤器应该使用 DispatcherType.ASYNC 和 DispatcherType.ERROR 进行映射。如果使用 Spring 框架的 abstractannotationconfigcherservletinitializer (参见 Servlet 配置) ，所有过滤器都会自动注册为所有调度类型。但是，如果通过 web.xml 或者通过 FilterRegistrationBean 在 Spring Boot 中注册过滤器，请确保除了 DispatcherType.REQUEST 之外还包含 DispatcherType.ASYNC 和 DispatcherType.ERROR。





#### Shallow ETag

`Shallowtagheaderfilter `过滤器通过缓存写入响应的内容并从中计算 MD5散列来创建一个“浅”ETag。下一次客户端发送时，它将执行相同的操作，但是它还将计算值与 If-None-Match 请求头进行比较，如果两者相等，则返回一个304(NOT _ modified)。

这种策略节省了网络带宽，但没有节省 CPU，因为必须为每个请求计算完整的响应。前面描述的控制器级别的其他策略可以避免计算。参见 HTTP 缓存。

这个过滤器有一个 writeWeakETag 参数，该参数将过滤器配置为编写类似于以下内容的弱 ETags: w/“02a2d595e6ed9a0b24f027f2b63b134d6”(如 RFC 7232 Section 2.3中定义的)。

为了支持异步请求，这个过滤器必须使用 DispatcherType.ASYNC 进行映射，以便过滤器能够延迟并成功地生成一个 ETag，直到最后的异步分派结束。如果使用 Spring 框架的 abstractannotationconfigcherservletinitializer (参见 Servlet 配置) ，所有过滤器都会自动注册为所有调度类型。但是，如果通过 web.xml 或者通过 FilterRegistrationBean 在 Spring Boot 中注册过滤器，请确保包含 DispatcherType.ASYNC。







在某些情况下，您可能需要在运行时用 AOP 代理装饰控制器。一个例子是，如果您选择将@transactional 注释直接放在控制器上。在这种情况下，特别是对于控制器，我们建议使用基于类的代理。这通常是使用控制器的默认选择。但是，如果控制器必须实现不是 Spring Context 回调的接口(例如 InitializingBean、 * Aware 等) ，则可能需要显式配置基于类的代理。例如，使用 < tx: annotation-driven/> 可以更改为 < tx: annotation-driven proxy-target-class = " true"/> ，使用@enabletransactionmanagement 可以更改为@enabletransactionmanagement (proxyTargetClass = true)。





##### [URI patterns](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/web.html#mvc-ann-requestmapping)

- `PathPattern` — a pre-parsed pattern matched against the URL path also pre-parsed as `PathContainer`. Designed for web use, this solution deals effectively with encoding and path parameters, and matches efficiently.

  PathPattern ー与 URL 路径匹配的预解析模式，也预解析为 PathContainer。该解决方案专为网络设计，有效地处理编码和路径参数，并有效地匹配。

- `AntPathMatcher` — match String patterns against a String path. This is the original solution also used in Spring configuration to select resources on the classpath, on the filesystem, and other locations. It is less efficient and the String path input is a challenge for dealing effectively with encoding and other issues with URLs.

  AntPathMatcher ー将字符串模式与字符串路径匹配。这是 Spring 配置中也使用的原始解决方案，用于选择类路径、文件系统和其他位置上的资源。它的效率较低，而且 String 路径输入对于有效处理 url 的编码和其他问题是一个挑战。



`PathPattern` is the recommended solution for web applications and it is the only choice in Spring WebFlux. Prior to version 5.3, `AntPathMatcher` was the only choice in Spring MVC and continues to be the default. However `PathPattern` can be enabled in the [MVC config](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/web.html#mvc-config-path-matching).

PathPattern 是 web 应用程序的推荐解决方案，也是 Spring WebFlux 的唯一选择。在版本5.3之前，AntPathMatcher 是 Spring MVC 中的唯一选择，并且仍然是默认选项。然而，在 MVC 配置中可以启用 PathPattern。

`PathPattern` supports the same pattern syntax as `AntPathMatcher`. In addition it also supports the capturing pattern, e.g. `{*spring}`, for matching 0 or more path segments at the end of a path. `PathPattern` also restricts the use of `**` for matching multiple path segments such that it’s only allowed at the end of a pattern. This eliminates many cases of ambiguity when choosing the best matching pattern for a given request. For full pattern syntax please refer to [PathPattern](https://docs.spring.io/spring-framework/docs/5.3.4/javadoc-api/org/springframework/web/util/pattern/PathPattern.html) and [AntPathMatcher](https://docs.spring.io/spring-framework/docs/5.3.4/javadoc-api/org/springframework/util/AntPathMatcher.html).

PathPattern 支持与 AntPathMatcher 相同的模式语法。此外，它还支持捕获模式，例如{ * spring } ，用于匹配路径末端的0个或更多路径段。PathPattern 还限制使用 * * 匹配多个路径段，这样它只允许在模式的末尾使用。这消除了在为给定请求选择最佳匹配模式时出现的许多歧义情况。对于完整的模式语法，请参考 PathPattern 和 AntPathMatcher。https://docs.spring.io/spring-framework/docs/5.3.4/javadoc-api/org/springframework/util/AntPathMatcher.html)



##### Pattern Comparison

- [`PathPattern.SPECIFICITY_COMPARATOR`](https://docs.spring.io/spring-framework/docs/5.3.4/javadoc-api/org/springframework/web/util/pattern/PathPattern.html#SPECIFICITY_COMPARATOR)
- [`AntPathMatcher.getPatternComparator(String path)`](https://docs.spring.io/spring-framework/docs/5.3.4/javadoc-api/org/springframework/util/AntPathMatcher.html#getPatternComparator-java.lang.String-)





从5.3开始，默认情况下，Spring MVC不再执行`.*`后缀模式匹配，其中映射到/ person的控制器也隐式映射到`/person.*`。 因此，路径扩展不再用于解释响应所请求的内容类型，例如/person.pdf、/person.xml等。



当浏览器用于发送难以一致解释的 Accept 头时，以这种方式使用（接收）文件扩展名是必要的。目前，这已经不再是必要的，使用 Accept 头文件应该是首选。

随着时间的推移，文件扩展名的使用（`.pdf|.xml|.*`）已被证明存在各种各样的问题。当使用 URI 变量、路径参数和 URI 编码时，它可能会导致模糊性。推理基于 url 的授权和安全性也变得更加困难(更多详细信息请参阅下一节)



要完全禁用5.3版本之前的路径扩展，请设置以下内容:

useSuffixPatternMatching (false) ，请参阅  [PathMatchConfigurer](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/web.html#mvc-config-path-matching)

Favorpaxtension (false) ，请参阅 [ContentNegotiationConfigurer](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/web.html#mvc-config-content-negotiation)

除了通过“ Accept”头请求内容类型之外，还有一种请求内容类型的方法仍然是有用的，例如在浏览器中键入 URL 时。路径扩展的一个安全的替代方法是使用**查询参数策略**。如果必须使用文件扩展名，可以考虑通过 contentnegotiationconfigureer 的 mediaTypes 属性将它们限制为显式注册的扩展名列表。



反射文件下载(reflected file download，RFD)攻击类似于 XSS，因为它依赖于响应中反射的请求输入(例如，查询参数和 URI 变量)。然而，RFD 攻击不是将 JavaScript 插入到 HTML 中，而是依赖于浏览器切换来执行下载，并在以后双击时将响应视为可执行脚本。

在 Spring MVC 中,@responsebody 和 ResponseEntity 方法面临风险，因为它们可以呈现不同的内容类型，客户端可以通过 URL 路径扩展请求这些内容类型。禁用后缀/模式匹配和使用路径扩展进行内容协商可以降低风险，但不足以防止 RFD 攻击。

为了防止 RFD 攻击，在呈现响应体之前，Spring MVC 添加了一个 Content-Disposition: inline; filename = f.txt 头文件，以建议一个固定的安全下载文件。只有当 URL 路径包含既不允许安全也不允许为内容协商显式注册的文件扩展名时，才会执行此操作。然而，当 url 直接在浏览器中输入时，它可能有潜在的副作用。

默认情况下，许多常见的路径扩展都是安全的。具有自定义 HttpMessageConverter 实现的应用程序可以为内容协商显式注册文件扩展名，以避免为这些扩展名添加 Content-Disposition 头。参见内容类型。

参见  [CVE-2015-5211](https://pivotal.io/security/cve-2015-5211) ，了解与《可持续发展战略》有关的其他建议。



You can narrow the request mapping based on the `Content-Type` of the request, as the following example shows:

您可以根据请求的 Content-Type 缩小请求映射范围

The `consumes` attribute also supports negation expressions — for example, `!text/plain` means any content type other than `text/plain`.

Consumes 属性还支持否定表达式ー例如，! text/plain 表示除 text/plain 以外的任何内容类型。

You can declare a shared `consumes` attribute at the class level. Unlike most other request-mapping attributes, however, when used at the class level, a method-level `consumes` attribute overrides rather than extends the class-level declaration.

可以在类级别声明共享的 consume 属性。但是，与大多数其他请求映射属性不同，当在类级别使用时，方法级别使用属性覆盖而不是扩展类级别声明





Spring MVC also supports custom request-mapping attributes with custom request-matching logic. This is a more advanced option that requires subclassing `RequestMappingHandlerMapping` and overriding the `getCustomMethodCondition` method, where you can check the custom attribute and return your own `RequestCondition`.

Springmvc 还支持自定义请求映射属性，并支持自定义请求匹配逻辑。这是一个更高级的选项，它需要子类化 requestmappinglermapping 并重写 getCustomMethodCondition 方法，在这里您可以检查自定义属性并返回自己的 RequestCondition。

```
@Configuration
public class MyConfig {

    @Autowired
    public void setHandlerMapping(RequestMappingHandlerMapping mapping, UserHandler handler) 
            throws NoSuchMethodException {

        RequestMappingInfo info = RequestMappingInfo
                .paths("/user/{id}").methods(RequestMethod.GET).build(); 

        Method method = UserHandler.class.getMethod("getUser", Long.class); 

        mapping.registerMapping(info, handler, method); 
    }
}
```



RequestToViewNameTranslator 

###### MatrixVariable

```
// GET /owners/42;q=11/pets/21;q=22

@GetMapping("/owners/{ownerId}/pets/{petId}")
public void findPet(
        @MatrixVariable(name="q", pathVar="ownerId") int q1,
        @MatrixVariable(name="q", pathVar="petId") int q2) {

    // q1 == 11
    // q2 == 22
}
```

Note that you need to enable the use of matrix variables. In the MVC Java configuration, you need to set a `UrlPathHelper` with `removeSemicolonContent=false` through [Path Matching](https://docs.spring.io/spring-framework/docs/5.3.x/reference/html/web.html#mvc-config-path-matching). In the MVC XML namespace, you can set `<mvc:annotation-driven enable-matrix-variables="true"/>`.

注意，您需要启用矩阵变量。在 MVC Java 配置中，需要通过 Path Matching 设置一个带 removeSemicolonContent = false 的 UrlPathHelper。在 MVC XML 命名空间中，可以设置 < MVC: annotation-driven enable-matrix-variables = " true"/> 。



Note that use of `@RequestParam` is optional (for example, to set its attributes). By default, any argument that is a simple value type (as determined by [BeanUtils#isSimpleProperty](https://docs.spring.io/spring-framework/docs/5.3.4/javadoc-api/org/springframework/beans/BeanUtils.html#isSimpleProperty-java.lang.Class-)) and is not resolved by any other argument resolver, is treated as if it were annotated with `@RequestParam`.

请注意,@requestparam 的使用是可选的(例如，用于设置其属性)。默认情况下，任何属于简单值类型(由 BeanUtils # issimpleproperty 确定)并且不能由任何其他参数解析器解析的参数都被视为使用@requestparam 注释的参数。





