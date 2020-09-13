`CORS`的全称是：跨域资源共享（Cross-origin resource sharing），它是[W3C(万维网联盟)](https://www.w3.org/)的标准，它定义了在跨域访问资源时浏览器和服务器之间如何通信。它是为突破同源策略的限制而出现的一种**官方标准的跨域解决方案**。



传统的ajax请求只能获取在同一个域名下的资源，但是Html5打破了这个限制：**允许ajax发起跨域请求**。跨域的解决方案有多种：JSONP、Flash、IFrame以及 CORS 等

#### 同源策略

`JavaScript`或`Cookie`只能访问同源(同协议、同域名、同端口下的内容



####  为何需要跨域请求？？？
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



Access-Control-Allow-Origin
该响应头是服务器必须返回的。它的值要么是请求时Origin的值（可从request里获取），要么是*这样浏览器才会接受服务器的返回结果。

Access-Control-Allow-Credentials
该响应头非必须，值是bool类型，表示是否允许发送Cookie

Access-Control-Expose-Headers

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

##### DefaultCorsProcessor

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



### `@CrossOrigin`

RequestMappingHandlerMapping

### CorsRegistry / CorsRegistration





PreFlightHandler





















