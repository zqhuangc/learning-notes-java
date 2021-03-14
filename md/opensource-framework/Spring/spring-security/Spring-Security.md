# 官方文档查阅](https://docs.spring.io/spring-security/site/docs/current/reference/htmlsingle/)

[参考博客](https://blog.csdn.net/qq_32867467?t=1)

https://blog.csdn.net/weixin_37689658/article/details/92752890?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-3.channel_param

https://wangsong.blog.csdn.net/article/details/89629415?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.channel_param

spring security webflux

## 引言

认证

授权/访问控制



session 管理   HttpSesssionSecurityContextRepository

InMemoryUserDetailsManager

## Java 硬编码方式

###  AbstractSecurityWebApplicationInitializer

* without spring
  1. Automatically register the `springSecurityFilterChain `Filter for every URL in your application
  2. Add a `ContextLoaderListener `that loads the WebSecurityConfig.

```
//WebSecurityConfig 的引入
//without spring
public class SecurityWebApplicationInitializer extends AbstractSecurityWebApplicationInitializer {
    public SecurityWebApplicationInitializer() {
    	super(WebSecurityConfig.class);
    }
}

// with spring mvc
public class MvcWebApplicationInitializer extends
AbstractAnnotationConfigDispatcherServletInitializer {
    // 确保 WebSecurityConfig 被加载
    @Override
    protected Class<?>[] getRootConfigClasses() {
    	return new Class[] { WebSecurityConfig.class };
    }
    // ... other overrides ...
}

```

WebApplicationInitializer

SecurityWebApplicationInitializer



### HttpSecurity

```java
public final class HttpSecurity extends
		AbstractConfiguredSecurityBuilder<DefaultSecurityFilterChain, HttpSecurity>
		implements SecurityBuilder<DefaultSecurityFilterChain>,
		HttpSecurityBuilder<HttpSecurity> {...}
```



#### AbstractConfiguredSecurityBuilder<DefaultSecurityFilterChain, HttpSecurity>



#### WebSecurityConfigurerAdapter

`WebSecurityConfigurerAdapter#configure(HttpSecurity http)`

多个 HttpSecurity，多次实现 WebSecurityConfigurerAdapter, 通过@Order 指定优先

#### Authorize Requests

antMatchers url 访问权限控制设置

```java
protected void configure(HttpSecurity http) throws Exception {
    http
    	.authorizeRequests()
    		.antMatchers("/resources/**", "/signup", "/about").permitAll() 
    		.antMatchers("/admin/**").hasRole("ADMIN") 
    		.antMatchers("/db/**").access("hasRole('ADMIN') and hasRole('DBA')") 
    		.anyRequest().authenticated() 
    		.and()
    	// ...
    	.formLogin();
}
```



### Method Security

注解 `@Security` 、`@PreAuthorize` 注解式给方法添加权限

@EnableGlobalMethodSecurity(securedEnabled = true)

@EnableGlobalMethodSecurity(jsr250Enabled = true)

@EnableGlobalMethodSecurity(prePostEnabled = true)

只有 Spring 装配的类才会受保护，如何不是需要另外配置

```xml
<global-method-security>
<protect-pointcut expression="execution(* com.mycompany.*Service.*(..))"
access="ROLE_USER"/>
</global-method-security>
```



#### Post Processing

### Custom DSLs

AbstractHttpConfigurer<MyCustomDsl, HttpSecurity>



## Namespace（xml配置方式）

SecurityFilterChain:

- xml 配置

```xml
<security:http >
        <security:intercept-url pattern="/**" access="hasRole('ROLE_USER')"/>
        <security:form-login/>
        <security:http-basic/>
        <security:logout/>
</security:http>

    <security:authentication-manager>
        <security:authentication-provider>
            <security:user-service>
                <security:user name="user" password="123456" authorities="ROLE_USER"/>
                <security:user name="admin" password="123456" authorities="ROLE_USER, ROLE_ADMIN"/>
            </security:user-service>
        </security:authentication-provider>
    </security:authentication-manager>
```

BCryptPasswordEncoder



SecurityContextHolder ---- MODE ---- ThreadLocal or other

SecurityContextHolder.getContext().getAuthentication().getPrincipal()

(UserDetails)principal

RoleVoter



## Architecture and Implementation

### Core Components

UserDetailsService(DAO)

> InMemoryDaoImpl
>
> JdbcDaoImpl

#### SecurityContext 

  * 持有身份验证和可能的特定于请求的安全性信息

  * > 默认情况下，securitycontext使用ThreadLocal来存储这些细节，
    >
    > 这意味着安全上下文总是对同一执行线程中的方法可用，
    >
    > 即使没有将安全上下文作为参数显式地传递给这些方法。
    >
    >
    >
    > securitycontext可以配置一个打开的策略启动，以指定希望如何存储上下文。
    >
    > SecurityContextHolder.MODE_GLOBAL  全局
    >
    > SecurityContextHolder.MODE_INHERITABLETHREADLOCAL  子线程
    >
    > SecurityContextHolder.MODE_THREADLOCAL 仅当前线程
    >
    >
    >
    > SecurityContextHolder.getContext().getAuthentication().getPrincipal();
#### Authentication
  * 以Spring安全特定的方式表示主体
  * Spring Security使用一个`Authentication`对象来存储了当前与应用程序交互的主体的详细信息
  * UsernamePasswordAuthenticationToken
####  GrantedAuthority

  * 反映授予给主体的应用程序范围的权限
  * Authentication：GrantedAuthority   1：N
#### UserDetails

  - 从您的应用程序的DAOs或其他安全数据来源提供必要的信息来构建 `Authentication object`

username

password

authorities     ----  {ROLE_XXX, ROLE_XXX,...}

####  UserDetailsService

  * 在传递基于字符串的用户名(或证书ID或类似的)时，loadUserByUsername 创建  UserDetails



 顺序

UserDetailsService#loadUserByUsername(String)  --> UserDetails -->  Authentication  -->  SecurityContextHolder



#### Authentication

* 认证流程

1. 获取用户名和密码并将其组合到一个实例中` UsernamePasswordAuthenticationToken` (身份验证接口的实例，我们之前看到过)。
2. 令牌被传递给`AuthenticationManager`实例进行验证。
3. 身份验证成功后，`AuthenticationManager`返回一个完全填充的`Authentication instance`。
4. 通过调用`SecurityContextHolder.getContext().setAuthentication(…)`来创建`SecurityContext`，并传入返回的`Authentication`。会话级生命周期，共享



* AbstractSecurityInterceptor（access-control）
* ExceptionTranslationFilter：负责检测抛出的任何Spring Security异常。 通常，此类异常由
  AbstractSecurityInterceptor产生，它是授权服务的主要提供者。
* AuthenticationEntryPoint
* SecurityContextPersistenceFilter



## AbstractSecurityInterceptor（access-control）



![111.png](http://ww1.sinaimg.cn/large/006xzusPly1g8ebj3nqsnj30o90jr796.jpg)

### Core Services

#### AuthenticationManager

check multiple authentication databases

* `ProviderManager`  namespace
  * `AuthenticationProvider`s 列表

认证来源 UserDetails UserDetailsService  --> DaoAuthenticationProvider



```xml
<bean id="authenticationManager"
class="org.springframework.security.authentication.ProviderManager">
    <constructor-arg>
        <list>
            <ref local="daoAuthenticationProvider"/>
            <ref local="anonymousAuthenticationProvider"/>
            <ref local="ldapAuthenticationProvider"/>
        </list>
    </constructor-arg>
</bean>

```

身份验证机制

* CasAuthenticationProvider



默认情况下(从Spring Security 3.1开始)，ProviderManager将尝试清除任何成功返回的来自身份验证对象的敏感凭据信息身份验证请求，

无状态应用程序中，缓存在凭据清除后引用无法认证的问题

1. 副本，缓存实现 或 AuthenticationProvider 创建返回的Authentication对象
2. ProviderManager.eraseCredentialsAfterAuthentication



#### AuthenticationProvider

* DaoAuthenticationProvider

##### UserDetailsService

> username, password and GrantedAuthority s
>
> UsernamePasswordAuthenticationToken

`UserDetails loadUserByUsername(String username) throws UsernameNotFoundException;`

* In-Memory Authentication
* JdbcDaoImpl

```java
implements WebSecurityConfigurerAdapter:    
    @Bean
    @Override
    protected UserDetailsService userDetailsService() {
        //直接建两个用户存在内存中，生产环境可以从数据库中读取,对应管理器JdbcUserDetailsManager
        InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
        // 创建两个用户
        //通过密码的前缀区分编码方式,推荐,这种加密方式很好的利用了委托者模式，使得程序可以使用多种加密方式，并且会自动
        //根据前缀找到对应的密码编译器处理。
        manager.createUser(User.withUsername("guest").password("{bcrypt}" +
                new BCryptPasswordEncoder().encode("123456")).roles("USER").build());
        manager.createUser(User.withUsername("root").password("{sha256}" +
                new StandardPasswordEncoder().encode("666666"))
                .roles("ADMIN", "USER").build());
        return manager;
    }
```





##### PasswordEncoder

* DelegatingPasswordEncoder

> NoOpPasswordEncoder
>
> BCryptPasswordEncoder
>
> Pbkdf2PasswordEncoder
>
> SCryptPasswordEncoder

```xml
// 方式一
PasswordEncoder passwordEncoder =
PasswordEncoderFactories.createDelegatingPasswordEncoder();
// 自定义
String idForEncode = "bcrypt";
Map encoders = new HashMap<>();
encoders.put(idForEncode, new BCryptPasswordEncoder());
encoders.put("noop", NoOpPasswordEncoder.getInstance());
encoders.put("pbkdf2", new Pbkdf2PasswordEncoder());
encoders.put("scrypt", new SCryptPasswordEncoder());
encoders.put("sha256", new StandardPasswordEncoder());
PasswordEncoder passwordEncoder =
new DelegatingPasswordEncoder(idForEncode, encoders);
```

存储格式：`{id}encodedPassword`

如：`{bcrypt}$2a$10$dXJ3SW6G7P50lGmMkkmwe.20cQQubK3.HZWzG3YB1tlRy.fqvM/BG`

不容易获取原密码，难迁移

#### AccessDecisionManager



## Web Application Security

 HttpServletRequest、HttpServletResponse、HttpInvoker

DelegatingFilterProxy

FilterChainProxy

Bypassing the Filter Chain

### BasicAuthenticationFilter

`BasicAuthenticationFilter`负责处理HTTP标头中显示的基本身份验证凭据

###　AbstractAuthenticationProcessingFilter





#### UsernamePasswordAuthenticationFilter





## Authorization（授权与访问控制）

Authentication --> list GrantedAuthority --> GrantedAuthority#getAuthority()

AuthenticationManager 存储  GrantedAuthority 到 Authentication 

AccessDecisionManager 读取 GrantedAuthority

AbstractSecurityInterceptor 调用 AccessDecisionManager 

### AccessDecisionManager (Authorization)

AccessDecisionManager is called by the AbstractSecurityInterceptor

```java
void decide(Authentication authentication, Object secureObject, Collection<ConfigAttribute> attrs) throws AccessDeniedException;
boolean supports(ConfigAttribute attribute);
boolean supports(Class clazz);
```

* AccessDecisionManager 

  * AbstractAccessDecisionManager#support

    * 下面类只实现 decide

    * AffirmativeBased#decide（只要有 voter 返回 deny）默认
    * ConsensusBased#decide（deny 占大多数）
    * UnanimousBased#decide（全部 deny）

* AbstractAccessDecisionManager(support 方法 通用实现)

```java
public boolean supports(xxx) {
		for (AccessDecisionVoter voter : this.decisionVoters) {
			if (voter.supports(xxx)) {
				return true;
			}
		}

		return false;
	}
```

* AffirmativeBased#decide

```java
/**
authentication  token
object FilterChain
configAttributes  
UrlMapping#configAttributes  
UrlMapping#requestMatcher   antMatchers 
*/
public void decide(Authentication authentication, Object object,
			Collection<ConfigAttribute> configAttributes) throws AccessDeniedException {
    
		int deny = 0;
		for (AccessDecisionVoter voter : getDecisionVoters()) {
			int result = voter.vote(authentication, object, configAttributes);
			if (logger.isDebugEnabled()) {
				logger.debug("Voter: " + voter + ", returned: " + result);
			}
			switch (result) {
			case AccessDecisionVoter.ACCESS_GRANTED:
				return;
			case AccessDecisionVoter.ACCESS_DENIED:
				deny++;

				break;

			default:
				break;
			}
		}
		if (deny > 0) {
			throw new AccessDeniedException(messages.getMessage(
					"AbstractAccessDecisionManager.accessDenied", "Access is denied"));
		}
		// To get this far, every AccessDecisionVoter abstained
		checkAllowIfAllAbstainDecisions();
	}
```





- Authentication#getAuthorities()

MethodInvocation

![111.png](http://ww1.sinaimg.cn/large/006xzusPly1g8ev0vfwzfj30o90flwhn.jpg)

### AccessDecisionVoter

```java
/**
 * Indicates a class is responsible for voting on authorization decisions.
 */
public interface AccessDecisionVoter<S> {

	int ACCESS_GRANTED = 1;
	int ACCESS_ABSTAIN = 0;
	int ACCESS_DENIED = -1;

	/**
	 * Indicates whether this {@code AccessDecisionVoter} is able to vote on the passed
	 */
	boolean supports(ConfigAttribute attribute);

	/**
	 * Indicates whether the {@code AccessDecisionVoter} implementation is able to provide
	 */
	boolean supports(Class<?> clazz);

	/**
	 * Indicates whether or not access is granted.
	 */
	int vote(Authentication authentication, S object,
			Collection<ConfigAttribute> attributes);
    /**
    object是用户要访问的资源,
    
    ConfigAttribute则是访问object要满足的条件ConfigAttribute哪来的?                WebSecurityConfigurerAdapter#configure(HttpSecurity http) 配置
    
    通常payload是字符串，比如ROLE_ADMIN
    */
}

```

* RoleVoter

```java
private String rolePrefix = "ROLE_";
public boolean supports(ConfigAttribute attribute) {
		if ((attribute.getAttribute() != null)
				&& attribute.getAttribute().startsWith(getRolePrefix())) {
			return true;
		}
		else {
			return false;
		}
	}

public int vote(Authentication authentication, Object object,
			Collection<ConfigAttribute> attributes) {
		if (authentication == null) {
			return ACCESS_DENIED;
		}
		int result = ACCESS_ABSTAIN;
		Collection<? extends GrantedAuthority> authorities = extractAuthorities(authentication);

		for (ConfigAttribute attribute : attributes) {
			if (this.supports(attribute)) {
				result = ACCESS_DENIED;

				// Attempt to find a matching granted authority
				for (GrantedAuthority authority : authorities) {
					if (attribute.getAttribute().equals(authority.getAuthority())) {
						return ACCESS_GRANTED;
					}
				}
			}
		}

		return result;
	}


```



## Session 管理

```
创建  HttpSession
存储
校验
SessionInformation

SessionManagementConfigurer#configurer    postProcess 类型处理
SecurityContextRepository   
httpsecurity#sharedObjects

SessionAuthenticationStrategy(SessionRegistry)#onAuthentication(authResult, request, response);
不配置时默认 NullAuthenticatedSessionStrategy

SessionRegistryImpl 内存维护



SessionManagementFilter sessionManagementFilter = 
new SessionManagementFilter(
				securityContextRepository, 
				getSessionAuthenticationStrategy(http));
		
// session 失效 url
InvalidSessionStrategy
SimpleRedirectInvalidSessionStrategy

// session 管理策略
CompositeSessionAuthenticationStrategy
ConcurrentSessionControlAuthenticationStrategy
AbstractSessionFixationProtectionStrategy
ChangeSessionIdAuthenticationStrategy
RegisterSessionAuthenticationStrategy#registerNewSession

ConcurrentSessionFilter
SessionInformationExpiredStrategy
LogoutConfigurer
LogoutHandler

HttpSessionSecurityContextRepository
```





SpringSession中Session的创建机制：

| 机制       | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| always     | 如果没有session存在就创建一个                                |
| ifRequired | 如果需要就创建一个Session（默认）                            |
| never      | SpringSecurity 将不会创建Session，但是如果应用中其他地方创建了Session，那么Spring Security将会使用它。 |
| stateless  | SpringSecurity将绝对不会创建Session                          |

Java Configuration

```css
@Override
public void configure(HttpSecurity http) throws Exception {
  http.sessionManagement()
      .sessionCreatePolicy(SessionCreatePolicy.IF_REQUIRED);
}
```





并发 session ：已经授权过的用户再次进行授权

```java
@Bean
public HttpSessionEventPublisher httpSessoinEventPublisher() {
    // 确保在销毁session时通知Spring security 注册中心
    return new HttpSessionEventPublisher();
}


@Override
public void configure(HttpSecurity http) throws Exception {
    // 
    http.sessionManagement().maximumSessions(2);
}

server.servlet.session.timeout=10
    
    
    
@Configuration
public class WebConfiguration implements WebApplicationInitializer {

  @Override
  public void onStartup(ServletContext servlet Context) throws ServletException {
    servletContext.setSessionTrackingModes(EnumSet.of(SessionTrackingMode.COOKIE));
  }
}
固定Session攻击,每次登陆成功后新建
session.invalidate();
session = request.getSession(true);

http.sessionManagement()
// .invalidSessionUrl("http://localhost:8080/#/login")
　　.invalidSessionStrategy(invalidSessionStrategy)//session无效处理策略
　　.maximumSessions(1) //允许最大的session
// .maxSessionsPreventsLogin(true) //只允许一个地点登录，再次登陆报错
　　.expiredSessionStrategy(sessionInformationExpiredStrategy) //session过期处理策略，被顶号了
;
```

session 存储策略配置，security 给我们提供了一个参数配置 session 存储策略类型 spring.session.store-type = REDIS，可配置类型由 org.springframework.boot.autoconfigure.session.StoreType 类决定。REDIS,MONGODB,JDBC,HAZELCAST,NONE

 @EnableRedisHttpSession(maxInactiveIntervalInSeconds = 3600)。标识启用Redis 的session管理策略。



#### SecurityContextPersistenceFilter#doFilter

在执行授权过程之前，Spring security将会运行一个filter（SecurityContextPersistenceFilter）来存储管理不同请求之间的context。这个Context将会根据一定的策略来进行存储，默认情况下是HttpSesssionSecurityContextRepository负责，它使用的是HttpSession。



security过滤链执行完时org.springframework.security.web.context.SecurityContextPersistenceFilter类会

​     调用SecurityContextHolder.clearContext() 把SecurityContextHolder清空，所以之后 SecurityContextHolder.getContext().getAuthentication() .getPrincipal() 使用 会得到null。   

 经过spring security**认证后**， security会把一个SecurityContextImpl对象存储到session中，**此对象中有当前用户的各种信息**

HttpSessionSecurityContextRepository#saveContext



存储 SecurityContext 为 session 属性 SPRING_SECURITY_CONTEXT_KEY

```

```



#### SecurityContextHolder 如何保证同一个 SecurityContext？

SecurityContextPersistenceFilter#doFilter  finally  session存储 

#### 通过 Spring Session 管理 session

SpringSessionBackedSessionRegistry

# 源码分析





* SecurityBuilder
  - AbstractSecurityBuilder
    - AbstractConfiguredSecurityBuilder
      - AuthenticationManagerBuilder
      - HttpSecurity
      - WebSecurity

AbstractConfiguredSecurityBuilder#build  -》SecurityConfigurer#configure(HttpSecurity)

* SecurityConfigurer<O, B extends SecurityBuilder<O>>
  - WebSecurityConfigurer
    - WebSecurityConfigurerAdapter



```java
HttpSecurity{
    public ExpressionUrlAuthorizationConfigurer<HttpSecurity>.ExpressionInterceptUrlRegistry authorizeRequests()
			throws Exception {
		ApplicationContext context = getContext();
		return getOrApply(new ExpressionUrlAuthorizationConfigurer<>(context))
				.getRegistry();
	}
}


ExpressionUrlAuthorizationConfigurer#createMetadataSource


AbstractConfigAttributeRequestMatcherRegistry:
final LinkedHashMap<RequestMatcher, Collection<ConfigAttribute>> createRequestMap() {
		if (unmappedMatchers != null) {
			throw new IllegalStateException(
					"An incomplete mapping was found for "
							+ unmappedMatchers
							+ ". Try completing it with something like requestUrls().<something>.hasRole('USER')");
		}

		LinkedHashMap<RequestMatcher, Collection<ConfigAttribute>> requestMap = new LinkedHashMap<>();
		for (UrlMapping mapping : getUrlMappings()) {
			RequestMatcher matcher = mapping.getRequestMatcher();
			Collection<ConfigAttribute> configAttrs = mapping.getConfigAttrs();
			// 访问控制时 vote 所需 configAttrs
            requestMap.put(matcher, configAttrs);
		}
		return requestMap;
	}
```



## Filter 机制

Spring Security 通过 **FilterChainProxy 作为单一的Filter** 注册到 web层，Proxy内部的Filter。

#### spring 请求执行链

```
OncePerRequestFilter#doFilter (filter 区分处理)
-- doFilterInternal
```

filter 与 servlet 的执行顺序

>  ApplicationFilterChain#internalDoFilter  --> ApplicationFilterChain#doFilter --> FilterChainProxy#doFilter  --> ApplicationFilterChain#doFilter ->.. servlet#service
>
>  web层 : client -> filter  ->  filter   -> FilterChainProxy   -> filter ->  servlet
>
>  Spring Security FilterChainProxy  : filter  ->  filter -> filter



FilterChainProxy相当于一个filter的容器，通过 VirtualFilterChain 来依次调用各个内部 filter

```java
    public void doFilter(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException {
        boolean clearContext = request.getAttribute(FILTER_APPLIED) == null;
        if (clearContext) {
            try {
                request.setAttribute(FILTER_APPLIED, Boolean.TRUE);
                doFilterInternal(request, response, chain);
            }
            finally {
                SecurityContextHolder.clearContext();
                request.removeAttribute(FILTER_APPLIED);
           }
        }
        else {
            doFilterInternal(request, response, chain);
        }
    }

    private void doFilterInternal(ServletRequest request, ServletResponse response,
            FilterChain chain) throws IOException, ServletException {
    
        FirewalledRequest fwRequest = firewall
                .getFirewalledRequest((HttpServletRequest) request);
        HttpServletResponse fwResponse = firewall
                .getFirewalledResponse((HttpServletResponse) response);
    
        List<Filter> filters = getFilters(fwRequest);
    
        if (filters == null || filters.size() == 0) {
            if (logger.isDebugEnabled()) {
                logger.debug(UrlUtils.buildRequestUrl(fwRequest)
                        + (filters == null ? " has no matching filters"
                                : " has an empty filter list"));
            }
    
            fwRequest.reset();
    
            chain.doFilter(fwRequest, fwResponse);
    
            return;
        }
    
        VirtualFilterChain vfc = new VirtualFilterChain(fwRequest, chain, filters);
        vfc.doFilter(fwRequest, fwResponse);
    }
    
    private static class VirtualFilterChain implements FilterChain {
        private final FilterChain originalChain;
        private final List<Filter> additionalFilters;
        private final FirewalledRequest firewalledRequest;
        private final int size;
        private int currentPosition = 0;
    
        private VirtualFilterChain(FirewalledRequest firewalledRequest,
                FilterChain chain, List<Filter> additionalFilters) {
            this.originalChain = chain;
            this.additionalFilters = additionalFilters;
            this.size = additionalFilters.size();
            this.firewalledRequest = firewalledRequest;
        }
    
        public void doFilter(ServletRequest request, ServletResponse response)
                throws IOException, ServletException {
            if (currentPosition == size) {
                if (logger.isDebugEnabled()) {
                    logger.debug(UrlUtils.buildRequestUrl(firewalledRequest)
                            + " reached end of additional filter chain; proceeding with original chain");
                }
    
                // Deactivate path stripping as we exit the security filter chain
                this.firewalledRequest.reset();
    
                originalChain.doFilter(request, response);
            }
            else {
                currentPosition++;
    
                Filter nextFilter = additionalFilters.get(currentPosition - 1);
    
                if (logger.isDebugEnabled()) {
                    logger.debug(UrlUtils.buildRequestUrl(firewalledRequest)
                            + " at position " + currentPosition + " of " + size
                            + " in additional filter chain; firing Filter: '"
                            + nextFilter.getClass().getSimpleName() + "'");
                }
    
                nextFilter.doFilter(request, response, this);
            }
        }
    }


javax.servlet.Filter ***

org.springframework.security.web.access.intercept.FilterSecurityInterceptor  filter interceptor 适配接口

- org.springframework.web.filter.GenericFilterBean ***

-- org.springframework.web.filter.OncePerRequestFilter

-- org.springframework.security.web
.authentication.AbstractAuthenticationProcessingFilter  权限认证相关

org.springframework.web.filter.DelegatingFilterProxy

FilterChainProxy
FilterChainProxy$VirtualFilterChain

************* WebSecurityConfiguration 核心配置构建类**************
httpSecurity#performBuild
org.springframework.security.web.SecurityFilterChain
DefaultSecurityFilterChain


AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME
springSecurityFilterChain

setFilterChainProxySecurityConfigurer

********  filterChain 的加载进spring filter chain  ***********
AbstractSecurityWebApplicationInitializer$onStartup

servletContext.addFilter(springSecurityFilterChain)
```



### Spring Security 内置  Filter 顺序

```java
HttpSecurity{
    private List<Filter> filters = new ArrayList<>();
    private FilterComparator comparator = new FilterComparator();
}

AbstractConfiguredSecurityBuilder#build() 时 执行  filters.sort
httpsecurity#performbuild() 会调用 filters#sort
// 默认order
FilterComparator() {
		Step order = new Step(INITIAL_ORDER, ORDER_STEP);
		put(ChannelProcessingFilter.class, order.next());
		put(ConcurrentSessionFilter.class, order.next());
		put(WebAsyncManagerIntegrationFilter.class, order.next());
		put(SecurityContextPersistenceFilter.class, order.next());
		put(HeaderWriterFilter.class, order.next());
		put(CorsFilter.class, order.next());
		put(CsrfFilter.class, order.next());
		put(LogoutFilter.class, order.next());
		filterToOrder.put(
			"org.springframework.security.oauth2.client.web.OAuth2AuthorizationRequestRedirectFilter",
				order.next());
		filterToOrder.put(
				"org.springframework.security.saml2.provider.service.servlet.filter.Saml2WebSsoAuthenticationRequestFilter",
				order.next());
		put(X509AuthenticationFilter.class, order.next());
		put(AbstractPreAuthenticatedProcessingFilter.class, order.next());
		filterToOrder.put("org.springframework.security.cas.web.CasAuthenticationFilter",
				order.next());
		filterToOrder.put(
			"org.springframework.security.oauth2.client.web.OAuth2LoginAuthenticationFilter",
				order.next());
		filterToOrder.put(
				"org.springframework.security.saml2.provider.service.servlet.filter.Saml2WebSsoAuthenticationFilter",
				order.next());
		put(UsernamePasswordAuthenticationFilter.class, order.next());
		put(ConcurrentSessionFilter.class, order.next());
		filterToOrder.put(
				"org.springframework.security.openid.OpenIDAuthenticationFilter", order.next());
		put(DefaultLoginPageGeneratingFilter.class, order.next());
		put(DefaultLogoutPageGeneratingFilter.class, order.next());
		put(ConcurrentSessionFilter.class, order.next());
		put(DigestAuthenticationFilter.class, order.next());
		filterToOrder.put(
				"org.springframework.security.oauth2.server.resource.web.BearerTokenAuthenticationFilter", order.next());
		put(BasicAuthenticationFilter.class, order.next());
		put(RequestCacheAwareFilter.class, order.next());
		put(SecurityContextHolderAwareRequestFilter.class, order.next());
		put(JaasApiIntegrationFilter.class, order.next());
		put(RememberMeAuthenticationFilter.class, order.next());
		put(AnonymousAuthenticationFilter.class, order.next());
		filterToOrder.put(
			"org.springframework.security.oauth2.client.web.OAuth2AuthorizationCodeGrantFilter",
				order.next());
		put(SessionManagementFilter.class, order.next());
		put(ExceptionTranslationFilter.class, order.next());
		put(FilterSecurityInterceptor.class, order.next());
		put(SwitchUserFilter.class, order.next());
	}


```



### 默认加载的 filter chain



```
SpringSecurity 采用的是责任链的设计模式，它有一条很长的过滤器链。现在对这条过滤器链的各个进行说明:

WebAsyncManagerIntegrationFilter：将 Security 上下文与 Spring Web 中用于处理异步请求映射的 WebAsyncManager 进行集成。

SecurityContextPersistenceFilter：在每次请求处理之前将该请求相关的安全上下文信息加载到 SecurityContextHolder 中，然后在该次请求处理完成之后，将 SecurityContextHolder 中关于这次请求的信息存储到一个“仓储”中，然后将 SecurityContextHolder 中的信息清除，例如在Session中维护一个用户的安全信息就是这个过滤器处理的。

HeaderWriterFilter：用于将头信息加入响应中。

CsrfFilter：用于处理跨站请求伪造。

LogoutFilter：用于处理退出登录。

UsernamePasswordAuthenticationFilter：用于处理基于表单的登录请求，从表单中获取用户名和密码。默认情况下处理来自 /login 的请求。从表单中获取用户名和密码时，默认使用的表单 name 值为 username 和 password，这两个值可以通过设置这个过滤器的usernameParameter 和 passwordParameter 两个参数的值进行修改。

DefaultLoginPageGeneratingFilter：如果没有配置登录页面，那系统初始化时就会配置这个过滤器，并且用于在需要进行登录时生成一个登录表单页面。

BasicAuthenticationFilter：检测和处理 http basic 认证。

RequestCacheAwareFilter：用来处理请求的缓存。

SecurityContextHolderAwareRequestFilter：主要是包装请求对象request。

AnonymousAuthenticationFilter：检测 SecurityContextHolder 中是否存在 Authentication 对象，如果不存在为其提供一个匿名 Authentication。

SessionManagementFilter：管理 session 的过滤器

ExceptionTranslationFilter：处理 AccessDeniedException 和 AuthenticationException 异常。

FilterSecurityInterceptor：（授权相关）可以看做过滤器链的出口。

RememberMeAuthenticationFilter：当用户没有登录而直接访问资源时, 从 cookie 里找出用户的信息, 如果 Spring Security 能够识别出用户提供的remember me cookie, 用户将不必填写用户名和密码, 而是直接登录进入系统，该过滤器默认不开启。
```



```
SecurityContextHolder.getContext().getAuthentication() != null 代表前面已认证成功
```

DefaultMethodSecurityExpressionHandler



## 配置元数据构建 

应用启动时自动构建

### builder#build

```
配置构建入口  WebSecurityConfiguration

@Bean(name = AbstractSecurityWebApplicationInitializer.DEFAULT_FILTER_NAME)
	public Filter springSecurityFilterChain() throws Exception {
		boolean hasConfigurers = webSecurityConfigurers != null
				&& !webSecurityConfigurers.isEmpty();
		if (!hasConfigurers) {
			WebSecurityConfigurerAdapter adapter = objectObjectPostProcessor
					.postProcess(new WebSecurityConfigurerAdapter() {
					});
			webSecurity.apply(adapter);
		}
		return webSecurity.build();
	}


AbstractConfiguredSecurityBuilder#doBuild
@Override
	protected final O doBuild() throws Exception {
		synchronized (configurers) {
			buildState = BuildState.INITIALIZING;

			beforeInit();
			init();

			buildState = BuildState.CONFIGURING;

			beforeConfigure();
			configure();

			buildState = BuildState.BUILDING;

			O result = performBuild();

			buildState = BuildState.BUILT;

			return result;
		}
	}


private void init() throws Exception {
		Collection<SecurityConfigurer<O, B>> configurers = getConfigurers();

		for (SecurityConfigurer<O, B> configurer : configurers) {
			configurer.init((B) this);
		}

		for (SecurityConfigurer<O, B> configurer : configurersAddedInInitializing) {
			configurer.init((B) this);
		}
	}
	
private void configure() throws Exception {
		Collection<SecurityConfigurer<O, B>> configurers = getConfigurers();

		for (SecurityConfigurer<O, B> configurer : configurers) {
			configurer.configure((B) this);
		}
	}

AbstractHttpConfigurer#configure
```



### configurer#configure(决定创建哪些filter)

xxxBuilder --> xxxConfigurer



configurer的执行顺序

```java
implement WebSecurityConfigurerAdapter:
{
    /**
    * 认证信息配置
    */
    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    // 设置自定义的userDetailsService以及密码编码器
    auth.userDetailsService(userDetailsServiceImpl)
      .passwordEncoder(bCryptPasswordEncoder());
    }
    
    /**
     * 授权信息配置
     * @param http
     * @throws Exception
     */
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.cors().and()
            // 禁用 CSRF
            .csrf().disable()
            // ExpressionUrlAuthorizationConfigurer$ExpressionInterceptUrlRegistry
            .authorizeRequests()
            .antMatchers(HttpMethod.POST, "/auth/login").permitAll()
            // 指定路径下的资源需要验证了的用户才能访问
            .antMatchers("/api/**").authenticated()
            .antMatchers(HttpMethod.DELETE, "/api/**").hasRole("ADMIN")
            // 其他都放行了
            .anyRequest().permitAll()
            .and()
            //添加自定义Filter
            .addFilter(new JwtAuthenticationFilter(authenticationManager()))
            .addFilter(new JwtAuthorizationFilter(authenticationManager()))
            // 不需要session（不创建会话，使用jwt）
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS).and()
            // 授权异常处理
            .exceptionHandling()
            .authenticationEntryPoint(new JwtAuthenticationEntryPoint())
            .accessDeniedHandler(new JwtAccessDeniedHandler());

    }
}
```



#### AbstractInterceptUrlConfigurer#configure

```
@Override
	public void configure(H http) throws Exception {
		FilterInvocationSecurityMetadataSource metadataSource = createMetadataSource(http);
		if (metadataSource == null) {
			return;
		}
		FilterSecurityInterceptor securityInterceptor = createFilterSecurityInterceptor(
				http, metadataSource, http.getSharedObject(AuthenticationManager.class));
		if (filterSecurityInterceptorOncePerRequest != null) {
			securityInterceptor
					.setObserveOncePerRequest(filterSecurityInterceptorOncePerRequest);
		}
		securityInterceptor = postProcess(securityInterceptor);
		http.addFilter(securityInterceptor);
		http.setSharedObject(FilterSecurityInterceptor.class, securityInterceptor);
	}

ExpressionUrlAuthorizationConfigurer#createMetadataSource
@Override
	ExpressionBasedFilterInvocationSecurityMetadataSource createMetadataSource(
			H http) {
		LinkedHashMap<RequestMatcher, Collection<ConfigAttribute>> requestMap = REGISTRY
				.createRequestMap();
		if (requestMap.isEmpty()) {
			throw new IllegalStateException(
					"At least one mapping is required (i.e. authorizeRequests().anyRequest().authenticated())");
		}
		return new ExpressionBasedFilterInvocationSecurityMetadataSource(requestMap,
				getExpressionHandler(http));
	}
	
public ExpressionBasedFilterInvocationSecurityMetadataSource(
			LinkedHashMap<RequestMatcher, Collection<ConfigAttribute>> requestMap,
			SecurityExpressionHandler<FilterInvocation> expressionHandler) {
		super(processMap(requestMap, expressionHandler.getExpressionParser()));
		Assert.notNull(expressionHandler,
				"A non-null SecurityExpressionHandler is required");
	}


AbstractConfigAttributeRequestMatcherRegistry#createRequestMap
final LinkedHashMap<RequestMatcher, Collection<ConfigAttribute>> createRequestMap() {
		if (unmappedMatchers != null) {
			throw new IllegalStateException(
					"An incomplete mapping was found for "
							+ unmappedMatchers
							+ ". Try completing it with something like requestUrls().<something>.hasRole('USER')");
		}

		LinkedHashMap<RequestMatcher, Collection<ConfigAttribute>> requestMap = new LinkedHashMap<>();
		for (UrlMapping mapping : getUrlMappings()) {
			RequestMatcher matcher = mapping.getRequestMatcher();
			Collection<ConfigAttribute> configAttrs = mapping.getConfigAttrs();
			requestMap.put(matcher, configAttrs);
		}
		return requestMap;
	}
	
	
	
	
	
private FilterSecurityInterceptor createFilterSecurityInterceptor(H http,
			FilterInvocationSecurityMetadataSource metadataSource,
			AuthenticationManager authenticationManager) throws Exception {
		FilterSecurityInterceptor securityInterceptor = new FilterSecurityInterceptor();
		securityInterceptor.setSecurityMetadataSource(metadataSource);
		// 若没有配置，创建默认的 AccessDecisionManager
		securityInterceptor
		.setAccessDecisionManager(getAccessDecisionManager(http));
		securityInterceptor.setAuthenticationManager(authenticationManager);
		// 校验所有必需参数
		securityInterceptor.afterPropertiesSet();
		return securityInterceptor;
	}
	
	
private AccessDecisionManager createDefaultAccessDecisionManager(H http) {
		AffirmativeBased result = new AffirmativeBased(getDecisionVoters(http));
		return postProcess(result);
	}
	
	
	
	WebExpressionConfigAttribute
DefaultFilterInvocationSecurityMetadataSource
AntPathRequestMatcher
```





##  Spring Security 认证调用流程

#### 是否需要认证？判断已认证？

#### filter chain 构造时机？filter 执行顺序？

HttpSecurity#performBuild（过滤器排序，封装到 filter chain）



通过用户名查询用户，密码通过加密算法比对，



#### 认证流程

一个请求过来时通过各个过滤器，最后通过FilterSecurityInterceptor来判断这个请求url是否是不需要验证的，如果是就直接访问到我们的接口api，如果不是的话，再判断当前请求线程中是否有authentication的认证对象，如果有就放行，如果没有就返回登录页面（比如这里我们设置的是登录表单的方式），来到登录页面输入账号密码登录后就会来到    UsernamePasswordAuthenticationFilter，经过一系列的操作，最后验证成功就会把认证对象authentication放进securityContext中，然后FilterSecurityInterceptor判断到当前请求线程中这个认证对象就放行，返回的时候最后会通过securityContextpersistenceFilter，判断当前线程是否有securityContext，如果有就放进session，那么下次再请求这个url的时候会首先通过securityContextpersistenceFilter这个过滤器，判断session中是否有securityContextduxiiang，如果有就放进当前请求线程中，然后最后经过FilterSecurityInterceptor时再判断当前请求线程是否有认证对象，由于最前面经过securityContextpersistenceFilter，已经从session中把认证对象放进了当前请求线程中，所以FilterSecurityInterceptor会直接放行，这样就访问到我们的接口api。

![undefined](http://ww1.sinaimg.cn/large/006xzusPly1g8exaoa9suj30yg0vg752.jpg)

- OncePerRequestFilter#doFilterInternal  

  - ...#doFilter

  - DelegatingFilterProxy#invokeDelegate
    - FilterChainProxy$VirtualFilterChain#doFilter（spring security 的 filter）
      * AbstractAuthenticationProcessingFilter#dofilter（模板方法）
        * requiresAuthentication(request, response) 是否需要认证
          * RequestMatcher#matches
          * AntPathRequestMatcher#matches  （登录匹配）
          * SpringAntMatcher   AntPathMatcher
        * UsernamePasswordAuthenticationFilter#attemptAuthenticate
          * 官方内部唯一实现
        * SessionAuthenticationStrategy#onAuthentication(authResult, request, response);

认证流程与 shiro 类似

* （interface）AuthenticationManager#authenticate(authentication)

  * ProviderManager#authenticate  

    遍历匹配 AuthenticationProvider#support 

    * AuthenticationProvider#authenticate...

    * DaoAuthenticationProvider#authenticate(UsernamePasswordAuthenticationToken)

    * AbstractUserDetailsAuthenticationProvider#authenticate(authentication)

      只支持 UsernamePasswordAuthenticationToken

      * AbstractUserDetailsAuthenticationProvider#retrieveUser(username,
        ​      (UsernamePasswordAuthenticationToken) authentication)

        * DaoAuthenticationProvider#retrieveUser

          ```java
          // 检查是否有 passwordEncoder
          this.prepareTimingAttackProtection();
          // 缓存中没有 user,则调用
          this.getUserDetailsService().loadUserByUsername(username);
          // UserDetailsService什么时候 设置的
          
          自定义认证，使用 DaoAuthenticationProvider，只需要为其提供 PasswordEncoder 和 UserDetailsService
          
          AuthenticationManagerBuilder 定制 Authentication Managers
          ```

          * UserDetailsService#loadUserByUsername

      * AbstractUserDetailsAuthenticationProvider#preAuthenticationChecks.check(user);

        // 预先检查，DefaultPreAuthenticationChecks，检查用户是否被lock或者账号是否可用

      * // 抽象方法，自定义密码检验

        AbstractUserDetailsAuthenticationProvider#additionalAuthenticationChecks(user,    

        (UsernamePasswordAuthenticationToken) authentication);  

        * ```
          有默认登录账户 User 实现 UserDetails，可自定义扩展实现 UserDetails
          DefaultPreAuthenticationChecks#check（校验账户状态）
          
          DaoAuthenticationProvider#additionalAuthenticationChecks
          PasswordEncoder#matches `凭证匹配
          
          DefaultPostAuthenticationChecks#check（校验凭证有效期）
          
          异常时，判断是否是缓存失效，重新通过数据库校验
          ```

      *  // 抽象方法，自定义检验

        AbstractUserDetailsAuthenticationProvider#createSuccessAuthentication(user/username, authentication, (UserDetails) user)

        ```java
        protected Authentication createSuccessAuthentication(Object principal,
        			Authentication authentication, UserDetails user) {
        		// Ensure we return the original credentials the user supplied,
        		// so subsequent attempts are successful even with encoded passwords.
        		// Also ensure we return the original getDetails(), so that future
        		// authentication events after cache expiry contain the details
        		UsernamePasswordAuthenticationToken result = new UsernamePasswordAuthenticationToken(
        				principal, authentication.getCredentials(),
        				authoritiesMapper.mapAuthorities(user.getAuthorities()));
        		result.setDetails(authentication.getDetails());
        
        		return result;
        	}
        ```

#### 配置 UserDetailsService 与 PasswordEncoder

WebSecurityConfigurerAdapter#init

WebSecurityConfigurerAdapter#getHttp

WebSecurityConfigurerAdapter#authenticationManager

WebSecurityConfigurerAdapter（实现类）#configure(AuthenticationManagerBuilder)

通过 AuthenticationManagerBuilder#userDetailsService（自定义 UserDetailsService 设置的时机）

new DaoAuthenticationConfigurer<>(userDetailsService)  通过构造函数设置的

```
public <T extends UserDetailsService> DaoAuthenticationConfigurer<AuthenticationManagerBuilder, T> userDetailsService(
			T userDetailsService) throws Exception {
		this.defaultUserDetailsService = userDetailsService;
		return apply(new DaoAuthenticationConfigurer<>(
				userDetailsService));
	}
```



## Spring Security 授权与访问控制

AccessDecisionManager

`SecurityConfigAttributes` 

### 动态权限校验

FilterSecurityInterceptor#SecurityMetadataSource 

可自定义 filter 放在 FilterSecurityInterceptor 之前 处理

* 自定义处理

1. 定义一个自己的`SecurityMetadataSource` 使其从数据库中查询权限自己构建 `ConfigAttribute`
2. 定义一个具有自己需求的`AccessDecisionVoter`
3. 规定一个自己的`AccessDecisionManager`



### 授权

用户授权 GrantedAuthority 在   认证成功时 的 loadUserByUsername  获取权限设置到返回的  `UserDetail`

### 访问控制



#### Filter（处理请求） --》 AOP （Interceptor 方法拦截）--》 方法调用

#### url 访问权限配置

* 硬编码  HttpSecurity
* FilterSecurityInterceptor 执行基于 spring servlet/filter#dofilter



#### 接口方法权限配置

* MethodSecurityInterceptor   执行基于 apo  MethodInterceptor#invoke

* GlobalMethodSecurityConfiguration

* 注解方式

  * 开启  @EnableGlobalMethodSecurity

  * security  el 表达式 

    * 解析 `DefaultMethodSecurityExpressionHandler`

    * SecurityExpressionRoot
    * SpelNodeImpl

  ```
  UrlAuthorizationConfigurer  AuthorizedUrl
  
  SecurityMetaSource
  PrePostAnnotationSecurityMetadataSource#getAttributes
  
  返回true 就是有权限 false 则是无权限  
  ```

@EnableGlobalMethodSecurity

```
-GlobalMethodSecuritySelector
-- GlobalMethodSecurityConfiguration
--- methodSecurityInterceptor 
----  accessDecisionManager 如何鉴权的 voter 有哪些，support 哪些
--- methodSecurityMetadataSource 关注 configattribute 的**类型**封装

AutoProxyRegistrar
GlobalMethodSecurityAspectJAutoProxyRegistrar

MethodSecurityMetadataSourceAdvisorRegistrar
MethodSecurityMetadataSourceAdvisor
MethodSecurityMetadataSourcePointcut

MethodSecurityInterceptor
```

#### SecurityExpressionRoot#hasPermission 默认全为 false



#### 执行流程

AbstractInterceptUrlConfigurer#configure

```
UrlAuthorizationConfigurer#createMetadataSource
ExpressionUrlAuthorizationConfigurer#createMetadataSource **
```

AbstractInterceptUrlConfigurer#createFilterSecurityInterceptor

##### FilterSecurityInterceptor#doFilter

##### MethodSecurityInterceptor#invoke

* FilterSecurityInterceptor#invoke 

* AbstractSecurityInterceptor#beforeInvocation  (提取 ConfigAttribute)

  ```java
  Collection<ConfigAttribute> attributes = this.obtainSecurityMetadataSource()
  				.getAttributes(object);
  ```

* 是否认证（避免认证已失效的情况）

* AccessDecisionManager#decide

* AccessDecisionVoter#vote

  * 注解权限处理：

  * `WebExpressiondenyVoter`#vote

  * `PreInvocationAuthorizationAdviceVoter`#vote

    ```
    ExpressionBasedPreInvocationAdvice#before
    ExpressionUtils.evaluateAsBoolean(preAuthorize, ctx);
    ```

* RoleVoter#vote( authority（ROLE）与 ConfigAttribute  match )



#####  configAttributes 构建方式

ExpressionUrlAuthorizationConfigurer#createMetadataSource **



* AbstractRequestMatcherRegistry
  - AbstractConfigAttributeRequestMatcherRegistry
    - AbstractInterceptUrlConfigurer$AbstractInterceptUrlRegistry
      -  ExpressionUrlAuthorizationConfigurer$ExpressionInterceptUrlRegistry



antMatchers --> requestMatcher

ExpressionUrlAuthorizationConfigurer$AuthorizedUrl#xxx

ExpressionUrlAuthorizationConfigurer$AuthorizedUrl#access -->    

ExpressionUrlAuthorizationConfigurer$interceptUrl  --> 

AbstractConfigAttributeRequestMatcherRegistry$UrlMapping(requestMatcher, SecurityConfig.createList(attribute))


```java
public final class ExpressionUrlAuthorizationConfigurer<H extends HttpSecurityBuilder<H>>
		extends
		AbstractInterceptUrlConfigurer<ExpressionUrlAuthorizationConfigurer<H>, H> {
	static final String permitAll = "permitAll";
	private static final String denyAll = "denyAll";
	private static final String anonymous = "anonymous";
	private static final String authenticated = "authenticated";
	private static final String fullyAuthenticated = "fullyAuthenticated";
	private static final String rememberMe = "rememberMe";

	private final ExpressionInterceptUrlRegistry REGISTRY;

	private SecurityExpressionHandler<FilterInvocation> expressionHandler;
	
	/**
	 * Creates a new instance
	 * @see HttpSecurity#authorizeRequests()
	 */
	public ExpressionUrlAuthorizationConfigurer(ApplicationContext context) {
		this.REGISTRY = new ExpressionInterceptUrlRegistry(context);
	}
    
     /**
      *
      */
     public ExpressionUrlAuthorizationConfigurer<HttpSecurity>.ExpressionInterceptUrlRegistry authorizeRequests()
			throws Exception {
		ApplicationContext context = getContext();
		return getOrApply(new ExpressionUrlAuthorizationConfigurer<>(context))
				.getRegistry();
	}
	
	/**
	 * Allows registering multiple {@link RequestMatcher} instances to a collection of {@link ConfigAttribute} instances
	 */
	private void interceptUrl(Iterable<? extends RequestMatcher> requestMatchers,
			Collection<ConfigAttribute> configAttributes) {
		for (RequestMatcher requestMatcher : requestMatchers) {
			REGISTRY.addMapping(new AbstractConfigAttributeRequestMatcherRegistry.UrlMapping(
					requestMatcher, configAttributes));
		}
	}
	
	/**
	 * 内部类
	 */
	public class ExpressionInterceptUrlRegistry
			extends
			ExpressionUrlAuthorizationConfigurer<H>.AbstractInterceptUrlRegistry<ExpressionInterceptUrlRegistry, AuthorizedUrl> {

		/**
		 * @param context
		 */
		private ExpressionInterceptUrlRegistry(ApplicationContext context) {
			setApplicationContext(context);
		}
		...
  
 AbstractRequestMatcherRegistry<C> {
    public C antMatchers(HttpMethod method, String... antPatterns) {
		Assert.state(!this.anyRequestConfigured, "Can't configure antMatchers after anyRequest");
		return chainRequestMatchers(RequestMatchers.antMatchers(method, antPatterns));
	}
}}
	
```


* ExpressionUrlAuthorizationConfigurer$AuthorizedUrl

```java
public class AuthorizedUrl {
		private List<? extends RequestMatcher> requestMatchers;
		private boolean not;

		/**
		 * Creates a new instance
		 *
		 * @param requestMatchers the {@link RequestMatcher} instances to map
		 */
		private AuthorizedUrl(List<? extends RequestMatcher> requestMatchers) {
			this.requestMatchers = requestMatchers;
		}
       
        /**
		 * Allows specifying that URLs are secured by an arbitrary expression
		 *
		 * @param attribute the expression to secure the URLs (i.e.
		 * "hasRole('ROLE_USER') and hasRole('ROLE_SUPER')")
		 * @return the {@link ExpressionUrlAuthorizationConfigurer} for further
		 * customization
		 */
		public ExpressionInterceptUrlRegistry access(String attribute) {
			if (not) {
				attribute = "!" + attribute;
			}
			interceptUrl(requestMatchers, SecurityConfig.createList(attribute));
			return ExpressionUrlAuthorizationConfigurer.this.REGISTRY;
		}

		/**
		 * Negates the following expression.
		 */
		public AuthorizedUrl not() {
			this.not = true;
			return this;
		}

		/**
		 * Shortcut for specifying URLs require a particular role. If you do not want to
		 * have "ROLE_" automatically inserted see {@link #hasAuthority(String)}.
		 */
		public ExpressionInterceptUrlRegistry hasRole(String role) {
			return access(ExpressionUrlAuthorizationConfigurer.hasRole(role));
		}

		/**
		 * Shortcut for specifying URLs require any of a number of roles. If you do not want to have "ROLE_" automatically inserted see
		 */
		public ExpressionInterceptUrlRegistry hasAnyRole(String... roles) {
			return access(ExpressionUrlAuthorizationConfigurer.hasAnyRole(roles));
		}

		/**
		 * Specify that URLs require a particular authority.
		 */
		public ExpressionInterceptUrlRegistry hasAuthority(String authority) {
			return access(ExpressionUrlAuthorizationConfigurer.hasAuthority(authority));
		}

		/**
		 * Specify that URLs requires any of a number authorities.
		 */
		public ExpressionInterceptUrlRegistry hasAnyAuthority(String... authorities) {
			return access(ExpressionUrlAuthorizationConfigurer
					.hasAnyAuthority(authorities));
		}

		/**
		 * Specify that URLs requires a specific IP Address or <a href=
		 * "https://forum.spring.io/showthread.php?102783-How-to-use-hasIpAddress&p=343971#post343971"
		 * >subnet</a>.
		 */
		public ExpressionInterceptUrlRegistry hasIpAddress(String ipaddressExpression) {
			return access(ExpressionUrlAuthorizationConfigurer
					.hasIpAddress(ipaddressExpression));
		}

		/**
		 * Specify that URLs are allowed by anyone.
		 * @return the {@link ExpressionUrlAuthorizationConfigurer} for further
		 * customization
		 */
		public ExpressionInterceptUrlRegistry permitAll() {
			return access(permitAll);
		}

		/**
		 * Specify that URLs are allowed by anonymous users.
		 */
		public ExpressionInterceptUrlRegistry anonymous() {
			return access(anonymous);
		}

		/**
		 * Specify that URLs are allowed by users that have been remembered.
		 */
		public ExpressionInterceptUrlRegistry rememberMe() {
			return access(rememberMe);
		}

		/**
		 * Specify that URLs are not allowed by anyone.
		 */
		public ExpressionInterceptUrlRegistry denyAll() {
			return access(denyAll);
		}

		/**
		 * Specify that URLs are allowed by any authenticated user.
		 */
		public ExpressionInterceptUrlRegistry authenticated() {
			return access(authenticated);
		}

		/**
		 * Specify that URLs are allowed by users who have authenticated and were not "remembered".
		 */
		public ExpressionInterceptUrlRegistry fullyAuthenticated() {
			return access(fullyAuthenticated);
		}
	}
```



##### ExpressionBasedFilterInvocationSecurityMetadataSource(官方configAttribute最后的处理)

```
extends DefaultFilterInvocationSecurityMetadataSource

public ExpressionBasedFilterInvocationSecurityMetadataSource(
			LinkedHashMap<RequestMatcher, Collection<ConfigAttribute>> requestMap,
			SecurityExpressionHandler<FilterInvocation> expressionHandler) {
		super(processMap(requestMap, expressionHandler.getExpressionParser()));
		Assert.notNull(expressionHandler,
				"A non-null SecurityExpressionHandler is required");
	}
	

private static LinkedHashMap<RequestMatcher, Collection<ConfigAttribute>> processMap(
			LinkedHashMap<RequestMatcher, Collection<ConfigAttribute>> requestMap,
			ExpressionParser parser) {
		Assert.notNull(parser, "SecurityExpressionHandler returned a null parser object");

		LinkedHashMap<RequestMatcher, Collection<ConfigAttribute>> requestToExpressionAttributesMap = new LinkedHashMap<>(
				requestMap);

		for (Map.Entry<RequestMatcher, Collection<ConfigAttribute>> entry : requestMap
				.entrySet()) {
			RequestMatcher request = entry.getKey();
			Assert.isTrue(entry.getValue().size() == 1,
					() -> "Expected a single expression attribute for " + request);
			ArrayList<ConfigAttribute> attributes = new ArrayList<>(1);
			String expression = entry.getValue().toArray(new ConfigAttribute[1])[0]
					.getAttribute();
			logger.debug("Adding web access control expression '" + expression + "', for "
					+ request);

			AbstractVariableEvaluationContextPostProcessor postProcessor = createPostProcessor(
					request);
			try {
				attributes.add(new WebExpressionConfigAttribute(
						parser.parseExpression(expression), postProcessor));
			}
			catch (ParseException e) {
				throw new IllegalArgumentException(
						"Failed to parse expression '" + expression + "'");
			}

			requestToExpressionAttributesMap.put(request, attributes);
		}

		return requestToExpressionAttributesMap;
	}
```



```
WebExpressionVoter#vote
public int vote(Authentication authentication, FilterInvocation fi,
			Collection<ConfigAttribute> attributes) {
		assert authentication != null;
		assert fi != null;
		assert attributes != null;

		WebExpressionConfigAttribute weca = findConfigAttribute(attributes);

		if (weca == null) {
			return ACCESS_ABSTAIN;
		}

		EvaluationContext ctx = expressionHandler.createEvaluationContext(authentication,
				fi);
		ctx = weca.postProcess(ctx, fi);

		return ExpressionUtils.evaluateAsBoolean(weca.getAuthorizeExpression(), ctx) ? ACCESS_GRANTED
				: ACCESS_DENIED;
	}
	
AbstractSecurityExpressionHandler#createEvaluationContext
public final EvaluationContext createEvaluationContext(Authentication authentication,
			T invocation) {
		SecurityExpressionOperations root = createSecurityExpressionRoot(authentication,
				invocation);
		StandardEvaluationContext ctx = createEvaluationContextInternal(authentication,
				invocation);
		ctx.setBeanResolver(br);
		ctx.setRootObject(root);

		return ctx;
	}


SpelNodeImpl
getValueInternal

PropertyOrFieldReference

PropertyAccessor

createOptimalAccessor

WebSecurityExpressionRoot

SecurityExpressionOperations
```

## 总结

- 如果认证没有成功则会默认由**AnonymousAuthenticationFilter** 设置一个匿名用户

- 进入**FilterSecurityInterceptor** 去调用 **accessDecisionManager** 来进行一个权限认证

- **accessDecisionManager** 用相应的策略循环 不同的 **AccessDecisionVoter** 进行投票

- 投票完毕后进行判断来决定是否抛出 **AccessDeniedException** 异常

- 抛出的异常由 ExceptionTranslationFilter 处理

  - 判断是否是匿名用户是的出重新去认证
  - 不是的话调用`AccessDeniedException` 去处理异常







# Example

@EnableWebSecurity

WebSecurityConfigurerAdapter

AbstractAuthenticationProcessingFilter

UsernamePasswordAuthenticationFilter

ProviderManager



https://www.jianshu.com/p/4468a2fff879



## [Spring Security with JWT](https://github.com/Snailclimb/spring-security-jwt-guide)

> Authorization: Bearer <token string>

### AbstractAuthenticationProcessingFilter

ApplicationFilterChain



模板方法：AbstractAuthenticationProcessingFilter#doFilter

基于浏览器的HTTP认证请求的抽象处理器

* 需要 注入AuthenticationManager 来处理  实现类提供的令牌（token）

* attemptAuthentication(HttpServletRequest,HttpServletResponse)

#### UsernamePasswordAuthenticationFilter

the UsernamePasswordAuthenticationFilter which is created by the <form-login> element,



```java
public class JwtAuthenticationFilter extends UsernamePasswordAuthenticationFilter {

    private final AuthenticationManager authenticationManager;

    public JwtAuthenticationFilter(AuthenticationManager authenticationManager) {
        this.authenticationManager = authenticationManager;

        setFilterProcessesUrl(SecurityConstants.AUTH_LOGIN_URL);
    }

    /**
    * 从请求数据或输入流获取 username，password
    * 根据实际需要重写逻辑
    */
    @Override
    public Authentication attemptAuthentication(HttpServletRequest request, HttpServletResponse response) {
        var username = request.getParameter("username");
        var password = request.getParameter("password");
        var authenticationToken = new UsernamePasswordAuthenticationToken(username, password);

        return authenticationManager.authenticate(authenticationToken);
    }
    
    /*
    @Override
    protected void successfulAuthentication(HttpServletRequest request, HttpServletResponse response,
                                            FilterChain filterChain, Authentication authentication) {
        var user = ((User) authentication.getPrincipal());

        var roles = user.getAuthorities()
            .stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.toList());

        var signingKey = SecurityConstants.JWT_SECRET.getBytes();

        var token = Jwts.builder()
            .signWith(Keys.hmacShaKeyFor(signingKey), SignatureAlgorithm.HS512)
            .setHeaderParam("typ", SecurityConstants.TOKEN_TYPE)
            .setIssuer(SecurityConstants.TOKEN_ISSUER)
            .setAudience(SecurityConstants.TOKEN_AUDIENCE)
            .setSubject(user.getUsername())
            .setExpiration(new Date(System.currentTimeMillis() + 864000000))
            .claim("rol", roles)
            .compact();

        response.addHeader(SecurityConstants.TOKEN_HEADER, SecurityConstants.TOKEN_PREFIX + token);
    }*/
    
    /**
     * 如果验证成功，就生成token并返回
     */
    @Override
    protected void successfulAuthentication(HttpServletRequest request,
                                            HttpServletResponse response,
                                            FilterChain chain,
                                            Authentication authentication) {

        JwtUser jwtUser = (JwtUser) authentication.getPrincipal();
        List<String> roles = jwtUser.getAuthorities()
                .stream()
                .map(GrantedAuthority::getAuthority)
                .collect(Collectors.toList());
        // 创建 Token
        String token = JwtTokenUtils.createToken(jwtUser.getUsername(), roles, rememberMe.get());
        // Http Response Header 中返回 Token
        response.setHeader(SecurityConstants.TOKEN_HEADER, token);
    }


    @Override
    protected void unsuccessfulAuthentication(HttpServletRequest request, HttpServletResponse response, AuthenticationException authenticationException) throws IOException {
        response.sendError(HttpServletResponse.SC_UNAUTHORIZED, authenticationException.getMessage());
    }
}

```



### BasicAuthenticationFilter

> Processes a HTTP request's BASIC authorization headers, putting the result into the SecurityContextHolder

```java
/**
 * 过滤器处理所有HTTP请求，并检查是否存在带有正确令牌的Authorization标头。例如，如果令牌未过期或签名密钥正确。
 *
 * 
 */
public class JWTAuthorizationFilter extends BasicAuthenticationFilter {

    private static final Logger logger = Logger.getLogger(JWTAuthorizationFilter.class.getName());

    public JWTAuthorizationFilter(AuthenticationManager authenticationManager) {
        super(authenticationManager);
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {

        String authorization = request.getHeader(SecurityConstants.TOKEN_HEADER);
        // 如果请求头中没有token信息则直接放行了
        if (authorization == null || !authorization.startsWith(SecurityConstants.TOKEN_PREFIX)) {
            chain.doFilter(request, response);
            return;
        }
        // 如果请求头中有token，则进行解析，并且设置授权信息
        SecurityContextHolder.getContext().setAuthentication(getAuthentication(authorization));
        super.doFilterInternal(request, response, chain);
    }

    /**
     * 获取用户认证信息 Authentication
     */
    private UsernamePasswordAuthenticationToken getAuthentication(String authorization) {
        String token = authorization.replace(SecurityConstants.TOKEN_PREFIX, "");
        try {
            String username = JwtTokenUtils.getUsernameByToken(token);
            logger.info("checking username:" + username);
            // 通过 token 获取用户具有的角色
            List<SimpleGrantedAuthority> userRolesByToken = JwtTokenUtils.getUserRolesByToken(token);
            if (!StringUtils.isEmpty(username)) {
                return new UsernamePasswordAuthenticationToken(username, null, userRolesByToken);
            }
        } catch (SignatureException | ExpiredJwtException | MalformedJwtException | IllegalArgumentException exception) {
            logger.warning("Request to parse JWT with invalid signature . Detail : " + exception.getMessage());
        }
        return null;
    }
    
    /**
    private UsernamePasswordAuthenticationToken getAuthentication(HttpServletRequest request) {
        var token = request.getHeader(SecurityConstants.TOKEN_HEADER);
        if (StringUtils.isNotEmpty(token) && token.startsWith(SecurityConstants.TOKEN_PREFIX)) {
            try {
                var signingKey = SecurityConstants.JWT_SECRET.getBytes();

                var parsedToken = Jwts.parser()
                    .setSigningKey(signingKey)
                    .parseClaimsJws(token.replace("Bearer ", ""));

                var username = parsedToken
                    .getBody()
                    .getSubject();

                var authorities = ((List<?>) parsedToken.getBody()
                    .get("rol")).stream()
                    .map(authority -> new SimpleGrantedAuthority((String) authority))
                    .collect(Collectors.toList());

                if (StringUtils.isNotEmpty(username)) {
                    return new UsernamePasswordAuthenticationToken(username, null, authorities);
                }
            } catch (ExpiredJwtException exception) {
                log.warn("Request to parse expired JWT : {} failed : {}", token, exception.getMessage());
            } catch (UnsupportedJwtException exception) {
                log.warn("Request to parse unsupported JWT : {} failed : {}", token, exception.getMessage());
            } catch (MalformedJwtException exception) {
                log.warn("Request to parse invalid JWT : {} failed : {}", token, exception.getMessage());
            } catch (SignatureException exception) {
                log.warn("Request to parse JWT with invalid signature : {} failed : {}", token, exception.getMessage());
            } catch (IllegalArgumentException exception) {
                log.warn("Request to parse empty or null JWT : {} failed : {}", token, exception.getMessage());
            }
        }

        return null;
    }*/
}

```



### ExceptionTranslationFilter(相关异常处理委派)

#### AccessDeniedHandler

##### AccessDeniedException

```java
用于处理通过认证的用户访问无权限资源时响应异常
/**
 * Used by {@link ExceptionTranslationFilter} to handle an
 * <code>AccessDeniedException</code>.
 *
 * @author Ben Alex
 */
public interface AccessDeniedHandler {
	// ~ Methods
	// ========================================================================================================

	/**
	 * Handles an access denied failure.
	 *
	 * @param request that resulted in an <code>AccessDeniedException</code>
	 * @param response so that the user agent can be advised of the failure
	 * @param accessDeniedException that caused the invocation
	 *
	 * @throws IOException in the event of an IOException
	 * @throws ServletException in the event of a ServletException
	 */
	void handle(HttpServletRequest request, HttpServletResponse response,
			AccessDeniedException accessDeniedException) throws IOException,
			ServletException;
}
```

#### AuthenticationEntryPoint

##### AuthenticationException

```
用于处理未通过身份认证的用户（如匿名）
如访问需要权限才能访问的资源时响应异常，或者重定向到特定的网页

/**
 * Used by {@link ExceptionTranslationFilter} to commence an authentication scheme.
 *
 * @author Ben Alex
 */
public interface AuthenticationEntryPoint {

	/**
	 * Commences an authentication scheme.
	 * <p>
	 * <code>ExceptionTranslationFilter</code> will populate the <code>HttpSession</code>
	 * attribute named
	 * <code>AbstractAuthenticationProcessingFilter.SPRING_SECURITY_SAVED_REQUEST_KEY</code>
	 * with the requested target URL before calling this method.
	 * <p>
	 * Implementations should modify the headers on the <code>ServletResponse</code> as
	 * necessary to commence the authentication process.
	 *
	 * @param request that resulted in an <code>AuthenticationException</code>
	 * @param response so that the user agent can begin authentication
	 * @param authException that caused the invocation
	 *
	 */
	void commence(HttpServletRequest request, HttpServletResponse response,
			AuthenticationException authException) throws IOException, ServletException;
}
```







### Security Configuration

#### @EnableWebSecurity( 激活 Spring web Security )

```
1: 加载了WebSecurityConfiguration配置类, 配置安全认证策略。
2: 加载了AuthenticationConfiguration, 配置了认证信息

@Import({ WebSecurityConfiguration.class,
		SpringWebMvcImportSelector.class,
		OAuth2ImportSelector.class })
@EnableGlobalAuthentication
@Configuration
public @interface EnableWebSecurity {
	/**
	 * Controls debugging support for Spring Security. Default is false.
	 * @return if true, enables debug support with Spring Security
	 */
	boolean debug() default false;
}

```

 

#### @EnableGlobalMethodSecurity( 开启Spring方法级安全 )



```java
@Import({ GlobalMethodSecuritySelector.class })
@EnableGlobalAuthentication
@Configuration
public @interface EnableGlobalMethodSecurity {

	/**
	 * 确定 前置注解[@PreAuthorize,@PostAuthorize,..] 是否启用
	 */
	boolean prePostEnabled() default false;

	/**
	 * 确定安全注解 [@Secured] 是否启用
	 */
	boolean securedEnabled() default false;

	/**
	 * 确定 JSR-250注解 [@RolesAllowed..]是否启用
	 */
	boolean jsr250Enabled() default false;

	/**
	 * Indicate whether subclass-based (CGLIB) proxies are to be created ({@code true}) as
	 * opposed to standard Java interface-based proxies ({@code false}). The default is
	 * {@code false}. <strong>Applicable only if {@link #mode()} is set to
	 * {@link AdviceMode#PROXY}</strong>.
	 *
	 * <p>
	 * Note that setting this attribute to {@code true} will affect <em>all</em>
	 * Spring-managed beans requiring proxying, not just those marked with the Security
	 * annotations. For example, other beans marked with Spring's {@code @Transactional}
	 * annotation will be upgraded to subclass proxying at the same time. This approach
	 * has no negative impact in practice unless one is explicitly expecting one type of
	 * proxy vs another, e.g. in tests.
	 *
	 * @return true if CGILIB proxies should be created instead of interface based
	 * proxies, else false
	 */
	boolean proxyTargetClass() default false;

	/**
	 * Indicate how security advice should be applied. The default is
	 * {@link AdviceMode#PROXY}.
	 * @see AdviceMode
	 *
	 * @return the {@link AdviceMode} to use
	 */
	AdviceMode mode() default AdviceMode.PROXY;

	/**
	 * Indicate the ordering of the execution of the security advisor when multiple
	 * advices are applied at a specific joinpoint. The default is
	 * {@link Ordered#LOWEST_PRECEDENCE}.
	 *
	 * @return the order the security advisor should be applied
	 */
	int order() default Ordered.LOWEST_PRECEDENCE;
}
```





#### WebSecurityConfigurerAdapter

```
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    UserDetailsServiceImpl userDetailsServiceImpl;

    /**
     * 密码编码器
     */
    @Bean
    public BCryptPasswordEncoder bCryptPasswordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public UserDetailsService createUserDetailsService() {
        return userDetailsServiceImpl;
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        // 设置自定义的userDetailsService以及密码编码器
        auth.userDetailsService(userDetailsServiceImpl).
        passwordEncoder(bCryptPasswordEncoder());
    }

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.cors().and()
                // 禁用 CSRF
                .csrf().disable()
                .authorizeRequests()
                .antMatchers(HttpMethod.POST, "/auth/login").permitAll()
                // 指定路径下的资源需要验证了的用户才能访问
                .antMatchers("/api/**").authenticated()
                .antMatchers(HttpMethod.DELETE, "/api/**").hasRole("ADMIN")
                // 其他都放行了
                .anyRequest().permitAll()
                .and()
                //添加自定义Filter
                .addFilter(new JWTAuthenticationFilter(authenticationManager()))
                .addFilter(new JWTAuthorizationFilter(authenticationManager()))
                // 不需要session（不创建会话）
                .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                // 授权异常处理
                .exceptionHandling().authenticationEntryPoint(new JWTAuthenticationEntryPoint())
                .accessDeniedHandler(new JWTAccessDeniedHandler());

    }

}
```





 {"username": "12356", "password": "123456","rememberMe":true}

RememberMeAuthenticationFilter 的作用？？