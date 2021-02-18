`CORS`的全称是：跨域资源共享（Cross-origin resource sharing），它是[W3C(万维网联盟)](https://www.w3.org/)的标准，它定义了在跨域访问资源时浏览器和服务器之间如何通信。它是为突破同源策略的限制而出现的一种**官方标准的跨域解决方案**。



传统的ajax请求只能获取在同一个域名下的资源，但是Html5打破了这个限制：**允许ajax发起跨域请求**。跨域的解决方案有多种：JSONP、Flash、IFrame以及 CORS 等

### 同源策略

`JavaScript`或`Cookie`只能访问同源(同协议、同域名、同端口下的内容



###  为何需要跨域请求？？？

这是跨域请求产生的背景，最主要是随着互联网的发展，忘了改善网络应用程序的环境增强其功能，开发人员要求浏览器供应商允许跨域请求，能带来如下好处：javascript可以使用ajax方式跨域访问资源
CSS可以使用@font-face跨域调用字体
通过canvas标签，绘制图表和视频



CORS发送出来的请求分为两种：

简单请求。需要同时满足下面三个要求
1. 请求方法只能是GET、POST、HEAD
2. Content-Type只能是三个值的任意一个application/x-www-form-urlencoded、multipart/form-data、text/plain（备注：若使用jquery的ajax发送请求，没指定Content-Type的情况下，默认它的值是application/x-www-form-urlencoded。源生的ajax请求请手动显示指定）
3. 无自定义请求头（除了Accept、Content-Type等等一些内置的头之外的头都叫自定义）
  非简单请求。除了简单请求之外都是它（带预检，也就是我们常见的OPTIONS请求）。



实际生产应用场景中我们最为常见的非简单请求场景大致有如下三种case：

1. ajax发送put、delete请求
2. 发送json格式数据（`Content-Type`为`application/json`）
3. 自定义请求头（比如自定义鉴权请求头`Authorization`）



* Access-Control-Allow-Origin
  该响应头是服务器必须返回的。它的值要么是请求时Origin的值（可从request里获取），要么是*这样浏览器才会接受服务器的返回结果。

* Access-Control-Allow-Credentials
  该响应头非必须，值是bool类型，表示是否允许发送Cookie

* Access-Control-Expose-Headers

CORS带来的问题
带来的安全隐患，最主要的便是著名的跨站请求伪造CSRF（Cross-site request forgery），所以要做好这块的安全工作（建议可开启withCredentials的cookie认证）
因为增加了OPTIONS预检请求，无疑增加了系统的开销（本一个请求搞定的变成了需要两个请求），所以需要做好缓存策略以及确保缓存能够生效
可能影响到你的限流，需要特殊处理。由于OPTIONS请求和实际请求的发送时间间隔非常短，此时若你限流如：同一IP每秒只能访问1次，那真实请求就会被拒绝了，因此此时就要排除掉OPTIONS这种预检请求的影响
同样的，若你的Filter/拦截器里，若有需要也是要对OPTIONS方法进行特殊处理的，否则可能就会执行多次造成一些麻烦

##### 如何理解`Access-Control-Max-Age`对相同URL生效？？？

**在实战场景中：能控制服务器的情况下，一般都是服务器上正确配置CORS**。可以在服务器API层（`Controller`层）进行精细化控制配置，也可以在`nginx`层进行统一配置（这样后端新加服务器不用再配置），最好配置上白名单而不是简单的粗暴的全是`*`。



### CorsConfiguration

### CorsConfigurationSource

- AbstractHandlerMapping.getHandler()/getCorsConfiguration()
- CorsFilter.doFilterInternal()
- HandlerMappingIntrospector.getCorsConfiguration()



#### `UrlBasedCorsConfigurationSource`



##### HandlerMappingIntrospector



##### CorsInterceptor



### CorsProcessor（重要）

##### DefaultCorsProcessor 处理过程

处理过程如下：

1. 若不是跨域请求，不处理（注意是return true后面拦截器还得执行呢）。若是跨域请求继续处理。（是否是跨域请求就看请求头是否有Origin这个头）
2. 判断response是否有Access-Control-Allow-Origin这个响应头，若有说明已经被处理过，那本处理器就不再处理了
3. 判断是否是同源：即使有Origin请求头，但若是同源的也不处理
4. 是否配置了CORS规则，若没有配置：
   1. 若是预检请求，直接决绝403，return false
   2. 若不是预检请求，则本处理器不处理
5. 正常处理CORS请求，大致是如下步骤：
   1. 判断 origin 是否合法
   2. 判断 method 是否合法
   3. 判断 header是否合法
   4. 若其中有一项不合法，直接决绝掉403并return false。都合法的话：就在response设置上一些头信息



### CorsFilter

```
// @since 4.2
public class CorsFilter extends OncePerRequestFilter {

	private final CorsConfigurationSource configSource;
	// 默认使用的DefaultCorsProcessor，当然你也可以自己指定
	private CorsProcessor processor = new DefaultCorsProcessor();

	@Override
	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) throws ServletException, IOException {
		// 只处理跨域请求
		if (CorsUtils.isCorsRequest(request)) {
			// Spring这里有个bug：因为它并不能保证configSource肯定被初始化了
			CorsConfiguration corsConfiguration = this.configSource.getCorsConfiguration(request);
			if (corsConfiguration != null) {
				boolean isValid = this.processor.processRequest(corsConfiguration, request, response);

	
				// 若处理后返回false，或者该请求本身就是个Options请求，那后面的Filter也不要处理了~~~~~
				if (!isValid || CorsUtils.isPreFlightRequest(request)) {
					return;
				}
			}
		}

		filterChain.doFilter(request, response);
	}
}
```



#### 使用

```
@Configuration
public class CorsConfig {


    private CorsConfiguration buildConfig() {
        CorsConfiguration corsConfiguration = new CorsConfiguration();
        corsConfiguration.addAllowedOrigin("*");
        corsConfiguration.addAllowedHeader("*");
        corsConfiguration.addAllowedMethod("*");
        // 预检请求的有效期，单位为秒。
        corsConfiguration.setMaxAge(3600L);
        // 是否支持安全证书(必需参数)
        corsConfiguration.setAllowCredentials(true);
        return corsConfiguration;
    }

    @Bean
    public CorsFilter corsFilter() {
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", buildConfig());
        return new CorsFilter(source);
    }
}
```



### CorsRegistry / CorsRegistration

#### 使用

```
@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")
                .allowedOrigins("*")
                .allowedMethods("GET", "HEAD", "POST", "PUT", "DELETE", "OPTIONS")
                .allowCredentials(true)
                .maxAge(3600)
                .allowedHeaders("*");
    }
}
```



### `@CrossOrigin`

RequestMappingHandlerMapping





PreFlightHandler





















## CORS 技术

为了解决浏览器跨域问题，`W3C` 提出了跨源资源共享方案，即 `CORS`([Cross-Origin Resource Sharing](https://www.w3.org/TR/cors/))。

`CORS` 可以在不破坏即有规则的情况下，通过后端服务器实现 `CORS` 接口，就可以实现跨域通信。

`CORS` 将请求分为两类：简单请求和非简单请求，分别对跨域通信提供了支持。

### 1、简单请求

在`CORS`出现前，发送`HTTP`请求时在头信息中不能包含任何自定义字段，且 `HTTP` 头信息不超过以下几个字段：

- `Accept`
- `Accept-Language`
- `Content-Language`
- `Last-Event-ID`
- `Content-Type` （仅限于 [`application/x-www-form-urlencoded` 、`multipart/form-data`、`text/plain` ] 类型）

一个简单请求的例子：

```
GET /test HTTP/1.1
Accept: */*
Accept-Encoding: gzip, deflate, sdch, br
Origin: http://www.test.com
Host: www.test.com
```

对于简单请求，`CORS`的策略是请求时在请求头中增加一个`Origin`字段，服务器收到请求后，根据该字段判断是否允许该请求访问。

- 如果允许，则在 HTTP 头信息中添加 `Access-Control-Allow-Origin` 字段，并返回正确的结果 ；
- 如果不允许，则不在 HTTP 头信息中添加 `Access-Control-Allow-Origin` 字段 。

除了上面提到的 `Access-Control-Allow-Origin` ，还有几个字段用于描述 `CORS` 返回结果 ：

- `Access-Control-Allow-Credentials`： 可选，用户是否可以发送、处理 `cookie`；
- `Access-Control-Expose-Headers`：可选，可以让用户拿到的字段。有几个字段无论设置与否都可以拿到的，包括：`Cache-Control`、`Content-Language`、`Content-Type`、`Expires`、`Last-Modified`、`Pragma` 。

### 2、非简单请求

对于非简单请求的跨源请求，浏览器会在真实请求发出前，增加一次`OPTION`请求，称为预检请求(`preflight request`)。预检请求将真实请求的信息，包括请求方法、自定义头字段、源信息添加到 HTTP 头信息字段中，询问服务器是否允许这样的操作。

例如一个`GET`请求：

```
OPTIONS /test HTTP/1.1
Origin: http://www.test.com
Access-Control-Request-Method: GET
Access-Control-Request-Headers: X-Custom-Header
Host: www.test.com
```

与 `CORS` 相关的字段有：

- 请求使用的 `HTTP` 方法 `Access-Control-Request-Method`
- 请求中包含的自定义头字段 `Access-Control-Request-Headers`

服务器收到请求时，需要分别对 `Origin`、`Access-Control-Request-Method`、`Access-Control-Request-Headers` 进行验证，验证通过后，会在返回 `HTTP`头信息中添加 ：

```
Access-Control-Allow-Origin: http://www.test.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: X-Custom-Header
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 1728000
```

他们的含义分别是：

- Access-Control-Allow-Methods: 真实请求允许的方法
- Access-Control-Allow-Headers: 服务器允许使用的字段
- Access-Control-Allow-Credentials: 是否允许用户发送、处理 cookie
- Access-Control-Max-Age: 预检请求的有效期，单位为秒。有效期内，不会重复发送预检请求

当预检请求通过后，浏览器才会发送真实请求到服务器。这样就实现了跨域资源的请求访问。

## 项目添加跨域支持

增加配置文件CorsConfig.java

```
package com.louis.kitty.boot.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
public class CorsConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/**")    // 允许跨域访问的路径
        .allowedOrigins("*")    // 允许跨域访问的源
        .allowedMethods("POST", "GET", "PUT", "OPTIONS", "DELETE")    // 允许请求方法
        .maxAge(168000)    // 预检间隔时间
        .allowedHeaders("*")  // 允许头部设置
        .allowCredentials(true);    // 是否发送cookie
    }
}
```

### 2.修改过滤器

#### 2.1 Shiro 导致的跨域问题

按照正常逻辑，添加了上面的跨域配置类就可以实现跨域支持了。然而，我们使用了 Shiro 就不一样了。我们上面讲到，对于非简单的跨域请求，会事先发起一个OPTION类型的预检请求，只有预检请求成功才会发起真正的请求，而这个预检请求是不带 token 的，这就意味着这个预检请求会被 shiro 过滤器拦截并在 token 校验失败之后返回失败信息，从而不会再发起真正的请求。

![img](https://www.codepeople.cn/imges/cors/001.png)

#### 2.2 跨域解决方案

解决思路很简单，既然是因为预检请求失败导致的问题，那就让预检请求自动放行就可以了。

![img](https://www.codepeople.cn/imges/cors/002.png)







```

@Configuration
public class CorsConfig {
 
	private CorsConfiguration buildConfig() {
		CorsConfiguration corsConfiguration = new CorsConfiguration();
		corsConfiguration.addAllowedOrigin("*");
		corsConfiguration.addAllowedHeader("*");
		corsConfiguration.addAllowedMethod("*");		
		corsConfiguration.setMaxAge(3600L);         // 预检请求的有效期，单位为秒。
		corsConfiguration.setAllowCredentials(true);// 是否支持安全证书(必需参数)
		return corsConfiguration;
	}
 
	@Bean
	public CorsFilter corsFilter() {
		UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
		source.registerCorsConfiguration("/**", buildConfig());
		return new CorsFilter(source);
	}
}
```



















