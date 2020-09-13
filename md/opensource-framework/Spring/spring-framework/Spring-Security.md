# [官方文档查阅](https://docs.spring.io/spring-security/site/docs/current/reference/htmlsingle/)



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

* SecurityContext 

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
* Authentication
  * 以Spring安全特定的方式表示主体
  * Spring Security使用一个`Authentication`对象来存储了当前与应用程序交互的主体的详细信息
  * UsernamePasswordAuthenticationToken
* GrantedAuthority

  * 反映授予给主体的应用程序范围的权限
  * Authentication：GrantedAuthority   1：N
* UserDetails

  - 从您的应用程序的DAOs或其他安全数据来源提供必要的信息来构建 `Authentication object`
* UserDetailsService

  * 在传递基于字符串的用户名(或证书ID或类似的)时，loadUserByUsername 创建  UserDetails



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

>

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

`BasicAuthenticationFilter`负责处理HTTP标头中显示的基本身份验证凭据



## Authorization

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



- Authentication#getAuthorities()

MethodInvocation

![111.png](http://ww1.sinaimg.cn/large/006xzusPly1g8ev0vfwzfj30o90flwhn.jpg)

#### AccessDecisionVoter

# Example

@EnableWebSecurity

WebSecurityConfigurerAdapter

AbstractAuthenticationProcessingFilter

UsernamePasswordAuthenticationFilter

ProviderManager



https://www.jianshu.com/p/4468a2fff879



![undefined](http://ww1.sinaimg.cn/large/006xzusPly1g8exaoa9suj30yg0vg752.jpg)

## [Spring Security with JWT](https://github.com/Snailclimb/spring-security-jwt-guide)

> Authorization: Bearer <token string>

### UsernamePasswordAuthenticationFilter

the UsernamePasswordAuthenticationFilter which is created by the <form-login> element,

AbstractAuthenticationProcessingFilter

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





### Security configuration

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