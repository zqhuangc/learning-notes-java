[跟我学 shiro](https://www.iteye.com/blog/jinnianshilongnian-2018936)

## 配置

SecurityManager：DefaultWebSecurityManager

Realm

ShiroFilterFactoryBean： urlPath   filter

SubjectFactory

`CredentialsMatcher 密码`   DefaultPasswordService

开启 Shiro 的 spring 相关注解

* ShiroAnnotationProcessorConfiguration
  * DefaultAdvisorAutoProxyCreator
  * AuthorizationAttributeSourceAdvisor
* ShiroBeanConfiguration
  * LifecycleBeanPostProcessor

- 缓存 session 

CacheManager：RedisCacheManager (RedisManager)

- session 管理

SessionIdGenerator： 

SessionDAO ： RedisSessionDAO

SessionManager： DefaultWebSessionManager



自定义 sessionIdCookie 统一 sessionid 管理

###[shiro-session 共享原理](https://www.cnblogs.com/youzhibing/p/9749427.html)

```java
https://my.oschina.net/ouminzy/blog/1056183 
/**
     * session管理器
     */
    @Bean
    public DefaultWebSessionManager defaultWebSessionManager(CacheManager cacheShiroManager, GunsProperties gunsProperties) {
        // 本质上 shiro 操作的 session 就是 http web session 的 
    	DefaultWebSessionManager sessionManager = new DefaultWebSessionManager();
        sessionManager.setCacheManager(cacheShiroManager);
        sessionManager.setSessionValidationInterval(gunsProperties.getSessionValidationInterval() * 1000);
        sessionManager.setGlobalSessionTimeout(gunsProperties.getSessionInvalidateTime() * 1000);
        sessionManager.setDeleteInvalidSessions(true);
        sessionManager.setSessionValidationSchedulerEnabled(true);
        Cookie cookie = new SimpleCookie(ShiroHttpSession.DEFAULT_SESSION_ID_NAME);
        cookie.setName("shiroCookie"); // 修改默认的 session 的 名称  , 这里没有 设置  session的 cookie 生命周期，
        	//默认就是 浏览器关闭时候或者服务器设置的 session 的时间
        cookie.setHttpOnly(true);
        sessionManager.setSessionIdCookie(cookie);
        return sessionManager;
    }
```

### 默认 filter

```
anon(AnonymousFilter.class),
authc(FormAuthenticationFilter.class),
authcBasic(BasicHttpAuthenticationFilter.class),
logout(LogoutFilter.class),
noSessionCreation(NoSessionCreationFilter.class),
perms(PermissionsAuthorizationFilter.class),
port(PortFilter.class),
rest(HttpMethodPermissionFilter.class),
roles(RolesAuthorizationFilter.class),
ssl(SslFilter.class),
user(UserFilter.class);
```

# shiro 源码

整个项目克隆自shiro[官方仓库](https://github.com/apache/shiro)，只是在子项目 `samples`里面添加了2个小项目，用于debug。

- shiro_learn

  (项目根路径) 

  - samples 
    - shiro_cas_service   使用springboot简单模拟cas服务(通过debug就可以完全了解cas服务请求过程)
    - shiro_client      源码分析开始的地方，shiro大部分配置已经配置好了

本文将从三个个方面进行分析：

1. **shiro配置（很全的）**
2. **Shiro web 过滤器 加载过程**
3. **完整分析一个shiro cas请求**

### **shiro 配置**

------



![img](https://user-gold-cdn.xitu.io/2019/11/25/16ea0501b5d85f2f?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



上图便是shiro所有功能展示图，从上图可以看出主要分为2大块，

​		一块是`Security Manager` ，这个模块及授权验证于一身（对应类为：`SecurityManager` 的子类），并通过Subject接口对外提供服务，例如登录、验权等等使用subject就好了（具体使用，后面会详细分析）；

​		另一块就是使用方，就是`SecurityManager` 已经配置好了，第三方应该如何使用呢，本文以`Web MVC` 这个第三方来叙述，`Web MVC`  主要通过 Filter 过滤器拦下所有用户请求，然后分发给相应的 shiro Filter进行相应的处理（例如看看这个请求有没有登录，有没有权限），在Filter中 使用 `SecurityManager` 提供的 `Subject`接口 进行相应的操作。

#### `SecurityManager`配置



![img](https://user-gold-cdn.xitu.io/2019/11/25/16ea0505db048b5a?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



通过上图可以看出DefaultSecurityManager可以配置的属性有哪些都可以看出来，下面通过表格一一说明：

| 属性               | 是否存在默认配置                  | 默认配置类                                                   | 作用说明                                                     | 常用配置          |
| ------------------ | --------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ | ----------------- |
| subjectFactory     | 是                                | DefaultSubjectFactory                                        | Subject的具体实现类（对外服务接口，如果是cas服务，建议使用**CasSubjectFactory**覆盖） | 否                |
| subjectDAO         | 是                                | DefaultSubjectDAO                                            | 主要用于将subject中最新信息保存到session里面                 | 否                |
| rememberMeManager  | 只存在DefaultWebSecurityManager中 | CookieRememberMeManager                                      | 用于管理rememberMe这个cookie，一般不用                       | 否                |
| **sessionManager** | 是                                | DefaultSessionManager (DefaultWebSecurityManager 就是DefaultWebSessionManager) | 有关session的操作最终都会委托给他做（他本身还可以配置，见下表） | 是                |
| authorizer         | 是                                | ModularRealmAuthorizer                                       | 授权策略（多个realm时，可以设置自己的策略）                  | 否                |
| authenticator      | 是                                | ModularRealmAuthenticator                                    | 认证策略（多个realm时可以设置自己的策略）                    | 是                |
| realm              | 否                                | CasRealm、JdbcRealm ......                                   | 落实认证和授权操作，需要自己配置（后面会举个例子细讲）       | 是                |
| cacheManager       | 否                                | 使用者最好继承AbstractCacheManager这个抽象类（支持shiro默认周期管理） | 在realm认证和授权的时候会用到（相当于加了一层缓存，cas认证就不需要了，如果是用户名密码，可以使用，增加认证授权速度） | 否（cas就不需要） |

DefaultSecurityManager的sessionManager (DefaultSessionManager)属性配置：



![img](https://user-gold-cdn.xitu.io/2019/11/25/16ea05092e4c6a64?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



| 属性              | 是否存在默认配置 | 默认配置类                        | 作用说明                                                     | 常用配置 |
| ----------------- | ---------------- | --------------------------------- | ------------------------------------------------------------ | -------- |
| sessionFactory    | 是               | SimpleSessionFactory              | 用于创建Session的，一般不配置                                | 否       |
| sessionDAO        | 是               | MemorySessionDAO                  | 用于保存Session的接口，一般通过继承**AbstractSessionDAO**类，并使用Redis重新配置一个，将Session保存在redis里面（此抽象类可以配置自己的**SessionIdGenerator** => 用于产生session id的） | 是       |
| casheManager      | 否               |                                   | 可以跟实现了CashManagerAware 这个类 的sessionDAO 联合使用，一般就配置sessionDAO就完事 | 否       |
| *sessionIdCookie* | *是*             | *new SimpleCookie("JSESSIONID");* | *这个存在于本类的子类DefaultWebSessionManager中，一般都重新配置，这样cookie名字可以改成自己的（一般有关cookie底层的操作都委托给他来做，例如读取此cookie的值，配置cookie ......）* | 是       |

> 注：这些属性通过对象的set方法都可以设置

### `Web MVC`过滤器配置

​		DefaultWebSecurityManager 跟DefaultSecurityManager 配置一样，以下说的securityManager 就是指DefaultWebSecurityManager 。

​		现在DefaultWebSecurityManager 已经配置好了，这个东西应该放在那里呢，肯定得应用到shiro过滤器里面去。

​		shiro 过滤器相关配置存放在`ShiroFilterFactoryBean`这个类里面，然后使用`DelegatingFilterProxy`将`ShiroFilterFactoryBean`注入到web容器里。

- 如果是springmvc，直接在web.xml内配置这个`DelegatingFilterProxy`就好，过滤器名字就是`ShiroFilterFactoryBean`这个bean的名字；
- 如果是springboot，使用springboot的 `FilterRegistrationBean`进行注册就好（*servlet3.0可以使用ServletContext进行注册，可以注册servlet、filter*......） => **后续会细讲**

​		`ShiroFilterFactoryBean` ，通过名字就可以看出来，他不是真正的Filter，他使用来配置和管理shirofilter的。



![img](https://user-gold-cdn.xitu.io/2019/11/25/16ea050f35c29c7c?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



从上图可以看出来，通过FactoryBean 来返回SpringShiroFilter这个对象。`ShiroFilterFactoryBean` 配置说明如下：

| 属性                                             | 作用说明                                                     |
| ------------------------------------------------ | ------------------------------------------------------------ |
| `securityManager`                                | shiro核心 ......                                             |
| `filters (Map<String, Filter>)`                  | 将自定义的shiro Filter都放在这个map里面，一般与下面这个定义联合使用 |
| `filterChainDefinitionMap (Map<String, String>)` | Map<请求url表达式, 用到的过滤器名字（多个用逗号隔开）> （原文解释：urlPathExpression_to_comma-delimited-filter-chain-definition） |
| `loginUrl`                                       | 登录URL                                                      |
| `successUrl`                                     | 登录成功后跳转的URL(一般不会使用这个跳转，而使使用第一次访问时保存的url) |
| `unauthorizedUrl`                                | 授权不成功跳转的URL                                          |

**以上所有配置例子 放在** `shiro_learn/samples/shiro_clien/src/main/java/com/nice01qc/config/shiro/ShiroCasConfig.java` **这个类里面, 可以对应着看。**

## Shiro web 过滤器 加载过程

这个加载过程其实就是 ShiroFilterFactoryBean 加载过程

​		现在securityManager已经手动配置好了，没什么好说的。let's coding .......，所有源码，只指出比较关键的节点，具体细节自己落实。



SpringShiroFilter注册到spring容器，会被包装成FilterRegistrationBean，通过FilterRegistrationBean注册到servlet容器；

一般而言，shiro的PathMatchingFilterChainResolver会匹配所有的请求，Shiro对Servlet容器的FilterChain进行了代理，生成代理FilterChain：ProxiedFilterChain，请求先走Shiro自己的Filter链，再走Servelt容器的Filter链；

### ShiroFilterFactoryBean.java 加载过程

```
public class ShiroFilterFactoryBean implements FactoryBean, BeanPostProcessor {
    
	// 就是从这个地方开始（如果不知道，请搜索 FactoryBean的作用）
    public Object getObject() throws Exception {
        if (instance == null) {
            instance = createInstance();  //直接看这个就好
        }
        return instance;
    }
    
 
    protected AbstractShiroFilter createInstance() throws Exception {
        SecurityManager securityManager = getSecurityManager();
        // 封装Filter调用管理，继续深入
        FilterChainManager manager = createFilterChainManager();
        
        PathMatchingFilterChainResolver chainResolver = new PathMatchingFilterChainResolver();
        chainResolver.setFilterChainManager(manager);
        
        //此处创建真正的 总览全局的 shiro filter
        return new SpringShiroFilter((WebSecurityManager) securityManager, chainResolver);
    }
    
    protected FilterChainManager createFilterChainManager() {
        DefaultFilterChainManager manager = new DefaultFilterChainManager();
        // 获取默认Filter ，取自DefaultFilter枚举类（共12个）
        Map<String, Filter> defaultFilters = manager.getFilters(); 
        // 将loginUrl、successUrl、unauthorizedUrl填充到符合要求的 filter 内
        for (Filter filter : defaultFilters.values()) {
            applyGlobalPropertiesIfNecessary(filter);
        }

        //这是你自己定义的Filter
        Map<String, Filter> filters = getFilters();
        if (!CollectionUtils.isEmpty(filters)) {
            for (Map.Entry<String, Filter> entry : filters.entrySet()) {
                String name = entry.getKey();
                Filter filter = entry.getValue();
                applyGlobalPropertiesIfNecessary(filter); // 填充一波
                if (filter instanceof Nameable) {
                    ((Nameable) filter).setName(name);
                }
                //'init' argument is false, since Spring-configured filters should be initialized
                //in Spring (i.e. 'init-method=blah') or implement InitializingBean:
                manager.addFilter(name, filter, false);
            }
        }

        //build up the chains:
        Map<String, String> chains = getFilterChainDefinitionMap(); // filterChainDefinitionMap
        if (!CollectionUtils.isEmpty(chains)) {
            for (Map.Entry<String, String> entry : chains.entrySet()) {
                // 例如 filterChainDefinitionMap.put("/index", "authc[config1,config2]");
                String url = entry.getKey();	// url 就是 "/index"
                String chainDefinition = entry.getValue(); // "authc[config1,config2]"
                // 会将 chainDefinition的“[config1,config2]” 解析后封装在 authc对应的filter内部，后续会用到
                manager.createChain(url, chainDefinition);
            }
        }
        return manager;
    }
    
    // 填充那三个值的地方
    private void applyGlobalPropertiesIfNecessary(Filter filter) {
        applyLoginUrlIfNecessary(filter);
        applySuccessUrlIfNecessary(filter);
        applyUnauthorizedUrlIfNecessary(filter);
    }
    private void applyLoginUrlIfNecessary(Filter filter) {
        String loginUrl = getLoginUrl();
        if (StringUtils.hasText(loginUrl) && (filter instanceof AccessControlFilter)) {
            AccessControlFilter acFilter = (AccessControlFilter) filter;
            String existingLoginUrl = acFilter.getLoginUrl();
            if (AccessControlFilter.DEFAULT_LOGIN_URL.equals(existingLoginUrl)) {
                acFilter.setLoginUrl(loginUrl);
            }
        }
    }
}
复制代码
```

以上就是ShiroFilterFactoryBean类初始化过程，后续`DelegatingFilterProxy`通过getBean("ShiroFilterFactoryBean的bean name") 来获取这个bean，并将请求委托给他。

### `shiro Filter`继承体系详细介绍

```
spring filter 方式
FilterRegistrationBean
```

DefaultFilterChainManager

​	*在继续深入之前，先介绍下shiro filter的特色，就是你想实现不同功能的filter，通过继承shiro自带的Filter就好，然后在这基础之上再做修改。*



![img](https://user-gold-cdn.xitu.io/2019/11/26/16ea86721d7c3717?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



以上每一个抽象类都有不同的作用，分工明确，下面一个一个来分析（上面的loginUrl在上面初始化的时候就注入进去了）：=> 这个很关键

```
AbstractFilter.java
// 实现了Filter的init接口，并对外暴露了onFilterConfigSet 接口
public abstract class AbstractFilter extends ServletContextSupport implements Filter {
    
    // 实现了Filter的init接口，并对外暴露了onFilterConfigSet 接口，这个接口在AbstractShiroFilter中覆盖了这个方法，其中AbstractShiroFilter 是 SpringShiroFilter的父类喔，SpringShiroFilter的父类喔，SpringShiroFilter的父类喔
    public final void init(FilterConfig filterConfig) throws ServletException {
        setFilterConfig(filterConfig);
        try {
            onFilterConfigSet();
        } catch (Exception e) {
        }
    }
   
    public void destroy() {
    }
    
}
复制代码
NameableFilter.java
public abstract class NameableFilter extends AbstractFilter implements Nameable { 
    // 设置filter 名字用的
	public void setName(String name) {
        this.name = name;
    }
}
复制代码
OncePerRequestFilter.java
public abstract class OncePerRequestFilter extends NameableFilter {
    // 保证每次请求只访问一次
    public final void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain) throws ServletException, IOException {
        // 查看是否已经访问过一次
        String alreadyFilteredAttributeName = getAlreadyFilteredAttributeName();
        // 如果访问了一次，则跳过这个filter，继续下一个
        if ( request.getAttribute(alreadyFilteredAttributeName) != null ) {
            filterChain.doFilter(request, response);
        } else if (!isEnabled(request, response) || shouldNotFilter(request) ) {
            filterChain.doFilter(request, response);
        } else {
            // 没有访问，现在标记
            request.setAttribute(alreadyFilteredAttributeName, Boolean.TRUE);
            try {
               	// 执行本filter 内容
                doFilterInternal(request, response, filterChain);
            } finally {
                request.removeAttribute(alreadyFilteredAttributeName);
            }
        }
    }
    
    // 子类需要实现的接口，不需要实现doFilter这个方法，通过实现这个方法，可以干更多事
    protected abstract void doFilterInternal(ServletRequest request, ServletResponse response, 		FilterChain chain) throws ServletException, IOException;
    
}
复制代码
AdviceFilter.java
public abstract class AdviceFilter extends OncePerRequestFilter {
    
    // 实现了OncePerRequestFilter这个方法，在这个方法有点像aop的风格
    public void doFilterInternal(ServletRequest request, ServletResponse response, FilterChain chain)
            throws ServletException, IOException {
        try {
		   // 在执行之前，先执行 preHandle 方法
            boolean continueChain = preHandle(request, response);
		   // 如果 preHandle 没通过，将不再继续往下执行
            if (continueChain) {
                executeChain(request, response, chain);
            }
            // 执行之后再执行的方法
            postHandle(request, response);
        } catch (Exception e) {
            exception = e;
        } finally {
            cleanup(request, response, exception);
        }
    }
    // 将这个方法暴露出去
    protected boolean preHandle(ServletRequest request, ServletResponse response) throws Exception {
        return true;
    }
    // 将这个方法也暴露出去
    protected void postHandle(ServletRequest request, ServletResponse response) throws Exception {
    }      
}
复制代码
PathMatchingFilter.java
// 用于判断请求是否符合本 Filter,只有请求URL 跟本Filter对应的url匹配规则对上
public abstract class PathMatchingFilter extends AdviceFilter implements PathConfigProcessor {
    // ShiroFilterFactoryBean 初始化时就填充进去了，
    // 里面value就是那个autho[config1,config2] 中这个[config1,config2]数组
    protected Map<String, Object> appliedPaths = new LinkedHashMap<String, Object>();
    	// 覆盖了父类的preHandle方法，首先判断请求url是否匹配 本Filter的appliedPaths
        protected boolean preHandle(ServletRequest request, ServletResponse response) throws Exception {
		// 如果Filter本身没有匹配url，返回true
        if (this.appliedPaths == null || this.appliedPaths.isEmpty()) {
            return true;
        }

        for (String path : this.appliedPaths.keySet()) {
            //(first match 'wins'):
            if (pathsMatch(path, request)) {
                Object config = this.appliedPaths.get(path);
                // 如果匹配上了就执行这个方法，它会进一步交给onPreHandle方法，并对外暴露这个方法
                return isFilterChainContinued(request, response, path, config);
            }
        }

        //no path matched, allow the request to go through:
        return true;
    }

	private boolean isFilterChainContinued(ServletRequest request, ServletResponse response,
                                           String path, Object pathConfig) throws Exception {

        if (isEnabled(request, response, path, pathConfig)) { //isEnabled check added in 1.2

            return onPreHandle(request, response, pathConfig);
        }
        return true;
    }
    
   // 对外暴露此接口
	protected boolean onPreHandle(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {
        return true;
    }    
}
复制代码
```

`AccessControlFilter.java`（如果你要验证授权什么的，这个类还是比较关键的）

```
// 此接口用于判断请求是否可以通过，不通过就跳登录，符合要求就使用subject进行登录 等等
public abstract class AccessControlFilter extends PathMatchingFilter {
    // 覆盖父类PathMatchingFilter 暴露的方法，并在方法内添加了两种方法
    // 一个是isAccessAllowed，用于判断是否已经验证过了，例如用户已经登录过了有session
    // 另一个是 onAccessDenied 验证失败，失败后干嘛，鬼知道，你看他怎么实现的
     public boolean onPreHandle(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {
        return isAccessAllowed(request, response, mappedValue) || onAccessDenied(request, response, mappedValue);
    }
    
    // 直接暴露给外界，自己看着实现吧，可别把onAccessDenied的活也干了就好了
    protected abstract boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception;
    
    // 这个老哥，提供了两个方式供外界覆盖，也就是方法 重载，就是看你要不要那个参数
    protected boolean onAccessDenied(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {
        return onAccessDenied(request, response);
    }
    // .......
    protected abstract boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception;
    
    // 判断是不是登录请求
    protected boolean isLoginRequest(ServletRequest request, ServletResponse response) {
        return pathsMatch(getLoginUrl(), request);
    }
    // 保存请求并重定向到登录页面
    protected void saveRequestAndRedirectToLogin(ServletRequest request, ServletResponse response) throws IOException {
        saveRequest(request);
        redirectToLogin(request, response);
    }
    
    protected void saveRequest(ServletRequest request) {
        WebUtils.saveRequest(request);
    }
    
    protected void redirectToLogin(ServletRequest request, ServletResponse response) throws IOException {
        String loginUrl = getLoginUrl();
        WebUtils.issueRedirect(request, response, loginUrl);
    }
        
}
复制代码
```

##### **以上做个小总结：**

1. 如果你就想写个简单的Filter，请直接实现Filter接口
2. 如果你想给Filter搞个名字，请继承NameableFilter抽象类
3. 如果你想保证你的filter只被调用了一次，请继承OncePerRequestFilter抽象类
4. 如果你想你的filter在dofilter 方法前后（类似这个方法的aop），请继承AdviceFilter（看着这个Advice是不是特别熟悉-----spring aop也有advice这个概念）
5. 如果你想写个验证和授权的Filter，请继续往下看，因为有两种实现了AccessControlFilter的类（轮子已经建好，上车吧）

**\*一种是 认证类的（Authenticate）：***

`AuthenticationFilter.java` (一般以ion结尾的单词类，一般提供很基础的服务)

```
public abstract class AuthenticationFilter extends AccessControlFilter {
    	// 提供基础的验证
        protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
        Subject subject = getSubject(request, response);
        return subject.isAuthenticated() && subject.getPrincipal() != null;
    }
    
        protected void issueSuccessRedirect(ServletRequest request, ServletResponse response) throws Exception {
            // 通过验证后，会被重定向到自己设定好的url，这个可以改
        WebUtils.redirectToSavedRequest(request, response, getSuccessUrl());
    }
    
}
复制代码
```

`AuthenticatingFilter.java`（真正干活的，以后认证什么的继承他就好了，稍微修改就好了）

```
public abstract class AuthenticatingFilter extends AuthenticationFilter {
    
    // 覆盖了，父类的方法，并在这个方法内部添加了vip功能（可以使用isPermissive走vip通道）
    // 如果是 AuthorizationFilter 的这个方法，mappedValue就是用于角色验证了，默认是所有角色都必须通过，可以覆盖这个方法
        @Override
    protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
        return super.isAccessAllowed(request, response, mappedValue) ||
                (!isLoginRequest(request, response) && isPermissive(mappedValue));
    }
    
    // 这是vip模板，仅仅是vip，普通乘客就别走这个了
    protected boolean isPermissive(Object mappedValue) {
        if(mappedValue != null) {
            String[] values = (String[]) mappedValue;
            return Arrays.binarySearch(values, PERMISSIVE) >= 0;
        }
        return false;
    }
    
    
    // 这个方法一般提供给onAccessDenied 方法的，就是你没通过认证，应该登录认证一波了
    protected boolean executeLogin(ServletRequest request, ServletResponse response) throws Exception {
        AuthenticationToken token = createToken(request, response);
        if (token == null) {
            throw new IllegalStateException(msg);
        }
        try {
            Subject subject = getSubject(request, response);
            subject.login(token);
            return onLoginSuccess(token, subject, request, response);
        } catch (AuthenticationException e) {
            return onLoginFailure(token, e, request, response);
        }
    }
    
    // 对外暴露接口，用于生成token，因为登录必须拿着token去登录，默认提供了两种生成token的方法，就在这个方法下面喔
    protected abstract AuthenticationToken createToken(ServletRequest request, ServletResponse response) throws Exception;
}
复制代码
```

**最后来一个例子： **

### 一个`shiro Filter`示例介绍

**CasFilter.java** (来看看cas他干了什么)

```
public class CasFilter extends AuthenticatingFilter {
    // 直接覆盖父类这个方法，意思就是，你竟然遇到了我，那你就是没有认证（需要被安排下）
        @Override
    protected boolean isAccessAllowed(ServletRequest request, ServletResponse response, Object mappedValue) {
        return false;
    }
    
    // 妥妥的实现这个方法，带我去登录吧，我准备好了
        @Override
    protected AuthenticationToken createToken(ServletRequest request, ServletResponse response) throws Exception {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        String ticket = httpRequest.getParameter(TICKET_PARAMETER);
        return new CasToken(ticket);
    }
    
    // 去吧皮卡丘，送你去登录
        @Override
    protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
        // 所以casFilter 直接执行这一步，前面那些方法都没起作用
        return executeLogin(request, response);
    }
    
}
复制代码
```

*授权类就不说了（同上）*，下面是上面Filter的一些默认实现(DefaultFilter枚举类中可以看到)（是不是很眼熟）

| filter名字          | 对应的Filter                          |
| ------------------- | ------------------------------------- |
| `anon`              | `AnonymousFilter.java`                |
| `authc`             | `FormAuthenticationFilter.java`       |
| `authcBasic`        | `BasicHttpAuthenticationFilter.java`  |
| `authcBearer`       | `BearerHttpAuthenticationFilter.java` |
| `logout`            | `LogoutFilter.java`                   |
| `noSessionCreation` | `NoSessionCreationFilter.java`        |
| `perms`             | `PermissionsAuthorizationFilter.java` |
| `port`              | `PortFilter.java`                     |
| `rest`              | `HttpMethodPermissionFilter.java`     |
| `roles`             | `RolesAuthorizationFilter.java`       |
| `ssl`               | `SslFilter.java`                      |
| `user`              | `UserFilter.java`                     |

通过以上Filter的了解，你是否已经了解shiro系Filter，如果不理解，请再看一遍（这是死循环判断，除非已经看懂，否则不会break;）容器初始化讲完了，是时候来个请求了。

## 完整分析一个shiro 请求

​		那就从请求被Filter拦截那个地方说起吧，到底被谁拦截了，是DelegatingFilterProxy这个类，好了，就在这个类的doFilter方法上打个断点（打断点前顺便看下Filter的init() 方法吧）let's begin ......

### DelegatingFilterProxy.java （请求开始的地方）

```
public class DelegatingFilterProxy extends GenericFilterBean {
        
    // 这个方法是在Filter init时调用的，在启动服务过程就会调用，他会初始化delegate，而这个delegate就是ShiroFilterFactoryBean
    @Override
	protected void initFilterBean() throws ServletException {
		synchronized (this.delegateMonitor) {
			if (this.delegate == null) {
				// If no target bean name specified, use filter name.
                // 什么是targetBeanName 见 DelegatingFilterProxy构造函数
				if (this.targetBeanName == null) {
					this.targetBeanName = getFilterName();
				}

				WebApplicationContext wac = findWebApplicationContext();
				if (wac != null) {
					this.delegate = initDelegate(wac); // 直接看这个
				}
			}
		}
	}
    
    // 看这个就好了
    protected Filter initDelegate(WebApplicationContext wac) throws ServletException {
		String targetBeanName = getTargetBeanName();
        // 直接通过ApplicationContex直接getBean了
		Filter delegate = wac.getBean(targetBeanName, Filter.class);
		if (isTargetFilterLifecycle()) {
			delegate.init(getFilterConfig());
		}
		return delegate;
	}
   
   // targetBeanName的由来
   public DelegatingFilterProxy(String targetBeanName, @Nullable WebApplicationContext wac) {
		this.setTargetBeanName(targetBeanName);
		this.webApplicationContext = wac;
		if (wac != null) {
			this.setEnvironment(wac.getEnvironment());
		}
	}
    
    
    // doFilter 在这里，来吧！！！ 在这里，来吧！！！ 在这里，来吧！！！
    @Override
	public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {

		// Lazily initialize the delegate if necessary.
		Filter delegateToUse = this.delegate;
		if (delegateToUse == null) {
			synchronized (this.delegateMonitor) {
				delegateToUse = this.delegate;
				if (delegateToUse == null) {
					WebApplicationContext wac = findWebApplicationContext();
					delegateToUse = initDelegate(wac);
				}
				this.delegate = delegateToUse;
			}
		}

		// Let the delegate perform the actual doFilter operation.
        // 直接去这个方法看吧
		invokeDelegate(delegateToUse, request, response, filterChain);
	}
    
    // 啥也不干，直接就抛给了ShiroFilterFactoryBean的SpringShiroFilter这个内部类
	protected void invokeDelegate(
			Filter delegate, ServletRequest request, ServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {

		delegate.doFilter(request, response, filterChain);
	}
    
}
复制代码
SpringShiroFilter
```

`ShiroFilterFactoryBean$SpringShiroFilter`的父类`OncePerRequestFilter.java`（以下便是`SpringShiroFilter`使命）

```
public abstract class OncePerRequestFilter extends NameableFilter {
    // 跳到这里来了
    public final void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain) {
        String alreadyFilteredAttributeName = getAlreadyFilteredAttributeName();
        if ( request.getAttribute(alreadyFilteredAttributeName) != null ) {
            filterChain.doFilter(request, response);
        } else if (!isEnabled(request, response) || shouldNotFilter(request) ) {
            filterChain.doFilter(request, response);
        } else {
                
            request.setAttribute(alreadyFilteredAttributeName, Boolean.TRUE);

            try {
                // 直接看看覆盖了这个方法的类吧
                doFilterInternal(request, response, filterChain);
            } finally {
                request.removeAttribute(alreadyFilteredAttributeName);
            }
        }
    }
}
复制代码
SpringShiroFilter`的父类`AbstractShiroFilter.java
public abstract class AbstractShiroFilter extends OncePerRequestFilter {
    // 此处是真正的实现
    protected void doFilterInternal(ServletRequest servletRequest, ServletResponse servletResponse, final FilterChain chain)throws ServletException, IOException {

        try {
            final ServletRequest request = prepareServletRequest(servletRequest, servletResponse, chain);
            final ServletResponse response = prepareServletResponse(request, servletResponse, chain);
			// 在此处创建了 subject喔，记住了喔！！！
            final Subject subject = createSubject(request, response);

            //noinspection unchecked
            // subject创建好，并将下面两个方法封装在Callable()里面再执行，
            // 在执行这个call之前，先将subject绑定到当前线程，执行完后，清理当前线程的绑定
            // 为什么非要搞个Callable，直接在 这两个方法前后放两个方法就好了，可能是因为这样扩展性更强
            // 以后在外面再封装一层也方便，说不定还可以搞异步？？？
            subject.execute(new Callable() {
                public Object call() throws Exception {
                    // 更新session时间
                    updateSessionLastAccessTime(request, response);
                    //************************************************
                    // 执行shiro过滤链，（先讲subject创建过程吧，后续再讲）
                    //************************************************
                    executeChain(request, response, chain);
                    return null;
                }
            });
        } catch (ExecutionException ex) {
            t = ex.getCause();
        }
    }
	//*****************************************
	// 暂时记这个创建过程为subject创建过程
	//*****************************************
    // subjcet 创建过程交给了父类Subject$Builder了，并送了他一个securityManager
	protected WebSubject createSubject(ServletRequest request, ServletResponse response){
        return new WebSubject.Builder(getSecurityManager(), request,  response).buildWebSubject();
    }
    
    
}
//==============================================================>

// 简单展示，具体自己debug看看
public class SubjectCallable<V> implements Callable<V> {
    public V call() throws Exception {
    	try {
            // 绑定到当前线程
        	threadState.bind();
            // 执行自己实现的那个call方法
        	return doCall(this.callable);
    	} finally {
            // 清除数据
        	threadState.restore();
    	}
	}
}

复制代码
```

### subject创建过程

`Subject$Builder`（`Subject`内部静态类`Build`）

```
public interface Subject {
    public static class Builder {
        // 从这儿可以看出，最终委托给了SecurityManager来干这个
        public Subject buildSubject() {
            return this.securityManager.createSubject(this.subjectContext);
        }
    }
}
复制代码
```

`DefaultSecurityManager.java`(以下讨论的都本类和本类的父类体系内喔)

```
public class DefaultSecurityManager extends SessionsSecurityManager {
    // 在subjcetContext基础上重新new一个喔，不影响前面的subjectContext
	public Subject createSubject(SubjectContext subjectContext) {
        //create a copy so we don't modify the argument's backing map:
        // 就是一个Map，里面存储了很多验证过程的东西，例如是否认证，是否是remember等等，
        // 在DefaultSubjectContext类里面可以看到，有：securityManager,sessionId,authenticationToken,authenticationInfo,subject,principals,session,authenticated,host,sessionCreationEnabled,principalsSessionKey,authenticatedSessionKey
        // 就是一个临时状态和工具集合地，如果是DefaultWebSecurityManager就创建一个
        // DefaultWebSubjectContext实例，实际上本文讨论的就是web
        SubjectContext context = copy(subjectContext);

        //确保SecurityManager 已经放到context里面去了
        context = ensureSecurityManager(context);
        
		// 这个最关键，也是最复杂一个，也不复杂，就深入了几个类，这也是下面要讲的重点
        // step0.........
        context = resolveSession(context);
		
        // 这个嘛，等会儿说
        context = resolvePrincipals(context);

        // 前戏已经准备好了，改开始创建Subject了
        Subject subject = doCreateSubject(context);

        // 这个会把当前Subject中最新的信息同步到session里面，还有其他功能，后续可以深入看看
        save(subject);

        return subject;
    }
    
    // step1,去解决session
	protected SubjectContext resolveSession(SubjectContext context) {
        if (context.resolveSession() != null) {
            return context;
        }
        try {
            // 直接看resolveContextSession
            Session session = resolveContextSession(context);
            if (session != null) {
                context.setSession(session);
            }
        } catch (InvalidSessionException e) {
        }
        return context;
    }
    // step2,开始了
    protected Session resolveContextSession(SubjectContext context) throws InvalidSessionException {
        // 先看看context里面有没有，有的话就不用继续找了，省时间，没有的话，就将request和response包装到SessionKey里面
        SessionKey key = getSessionKey(context);
        if (key != null) {
            // 现在真正开始了，但这事得分工，直接交给专门管理session的父类		SessionsSecurityManager（代码里面的啃老族，把活全交给父类干，很正常喔）
            return getSession(key);
        }
        return null;
    }
}
复制代码
```

`SessionsSecurityManager.java`（所有session有关操作，由他管理）

```
public abstract class SessionsSecurityManager extends AuthorizingSecurityManager {
    // step3. 看到这里，发现SecurityManager整体是不会干活的，就管着整个流程，然后分发出去
    public Session getSession(SessionKey key) throws SessionException {
        // 这个sessionManger，如果你不覆盖它，默认就是DefaultSessionManager这个类，我们就从这个开始吧
        return this.sessionManager.getSession(key);
    }
}
复制代码
```

`AbstractNativeSessionManager.java`（`DefaultSessionManager`父类）

```
public abstract class AbstractNativeSessionManager extends AbstractSessionManager implements NativeSessionManager, EventBusAware {
    // step4. 别急，渐渐开始了
    public Session getSession(SessionKey key) throws SessionException {
        Session session = lookupSession(key);
        return session != null ? createExposedSession(session, key) : null;
    }
    
    // step5. 来了
    private Session lookupSession(SessionKey key) throws SessionException {
        if (key == null) {
            throw new NullPointerException("SessionKey argument cannot be null.");
        }
        // 看到do开头方法，就知道，开始真正干活了
        return doGetSession(key);
    }
    
}
复制代码
```

`AbstractValidatingSessionManager.java`（`DefaultSessionManager`父类）

```
public abstract class AbstractValidatingSessionManager extends AbstractNativeSessionManager        implements ValidatingSessionManager, Destroyable {
    
    // step6. 开始了
    @Override
    protected final Session doGetSession(final SessionKey key) throws InvalidSessionException {
        // 验证session的有效性，一般不启动这个分方法（感觉redis可以设置时间控制有效性，可以不启动验证）
        enableSessionValidationIfNecessary();
        // 直接看这个吧，这个直接跳到子类DefaultSessionManager中去了，go
        Session s = retrieveSession(key);
        if (s != null) {
            validate(s, key);
        }
        return s;
    }
    
}
复制代码
DefaultSessionManager.java
public class DefaultSessionManager extends AbstractValidatingSessionManager implements CacheManagerAware {
   	// step7. 继续
    protected Session retrieveSession(SessionKey sessionKey) throws UnknownSessionException {
        // 先取sessionId
        Serializable sessionId = getSessionId(sessionKey);
        if (sessionId == null) {
            return null;
        }
        // 通过sessionId来取session
        Session s = retrieveSessionFromDataSource(sessionId);
        if (s == null) {
            throw new UnknownSessionException(msg);
        }
        return s;
    }
    
    // step8. 开始找sessionId
    @Override
    public Serializable getSessionId(SessionKey key) {
        // 查看key中是否就存储了这个sessionId（shiro到处搞引用缓存，真几把绕）
        Serializable id = super.getSessionId(key);
        if (id == null && WebUtils.isWeb(key)) {
            ServletRequest request = WebUtils.getRequest(key);
            ServletResponse response = WebUtils.getResponse(key);
            // 继续从这儿开始
            id = getSessionId(request, response);
        }
        return id;
    }
    // 到这里了，继续
    protected Serializable getSessionId(ServletRequest request, ServletResponse response) {
        return getReferencedSessionId(request, response);
    }
    // step9. 这里就找完就结束了，没找到就没找到了，开始回到step7，假定已经找到了，ok，继续
    private Serializable getReferencedSessionId(ServletRequest request, ServletResponse response) {
		// 这个直接从cookie里面读，这个过程建议自己debug进去看看，我感觉挺重要的，也很简单，我就不写了
        String id = getSessionIdCookieValue(request, response);
        if (id != null) {
            // 保存到request里面去
            request.setAttribute(ShiroHttpServletRequest.REFERENCED_SESSION_ID_SOURCE,
                    ShiroHttpServletRequest.COOKIE_SESSION_ID_SOURCE);
        } else {

            // 如果没找到就从请求参数里面找，这个请求规则是这样的：http:localhost:8001?;ShiroHttpSession.DEFAULT_SESSION_ID_NAME=sessionId(熟称shiro小尾巴，真心不好看)
            //try the URI path segment parameters first:
            id = getUriPathSegmentParamValue(request, ShiroHttpSession.DEFAULT_SESSION_ID_NAME);

            if (id == null) {
                //not a URI path segment parameter, try the query parameters:
                String name = getSessionIdName();
                id = request.getParameter(name);
                if (id == null) {
                    //try lowercase:
                    // 还没找到，那就从request请求参数里面去找（所以就算浏览器存不了cookie，那只能保存到请求参数里面了）
                    id = request.getParameter(name.toLowerCase());
                }
            }
            if (id != null) {
                request.setAttribute(ShiroHttpServletRequest.REFERENCED_SESSION_ID_SOURCE,
                        ShiroHttpServletRequest.URL_SESSION_ID_SOURCE);
            }
        }
        // 把刚获取到的结果都放在request里面缓存起来
        if (id != null) {
            request.setAttribute(ShiroHttpServletRequest.REFERENCED_SESSION_ID, id);
            request.setAttribute(ShiroHttpServletRequest.REFERENCED_SESSION_ID_IS_VALID, Boolean.TRUE);
        }

        // always set rewrite flag - SHIRO-361
        request.setAttribute(ShiroHttpServletRequest.SESSION_ID_URL_REWRITING_ENABLED, isSessionIdUrlRewritingEnabled());

        return id;
    }
    
    // step10. 从你自己设定的存储来获取session，如果是redis，就从redis里面获取，就到这个了，剩下的自己看吧
    protected Session retrieveSessionFromDataSource(Serializable sessionId) throws UnknownSessionException {
        return sessionDAO.readSession(sessionId);
    }
}
复制代码
```

到这一步，`resolveSession(context)`这个方法已经完成，只剩下`doCreateSubject(context)`和`save(subject)`

```
public class DefaultSecurityManager extends SessionsSecurityManager {

    public Subject createSubject(SubjectContext subjectContext) {
        SubjectContext context = copy(subjectContext);
        context = ensureSecurityManager(context);
        context = resolveSession(context);
        context = resolvePrincipals(context);
		// 来这个很简单
        Subject subject = doCreateSubject(context);

        save(subject);

        return subject;
    }
    
    // 从方法就看出来，最终使用专门的subjectFactory来创建Subject，本文都在讲web
    // 所以默认是 DefaultWebSubjectFactory这个工厂方法
    protected Subject doCreateSubject(SubjectContext context) {
        return getSubjectFactory().createSubject(context);
    }
    
}
复制代码
DefaultWebSubjectFactory.java
public class DefaultWebSubjectFactory extends DefaultSubjectFactory {
    
    public Subject createSubject(SubjectContext context) {

        boolean isNotBasedOnWebSubject = context.getSubject() != null && !(context.getSubject() instanceof WebSubject);
        if (!(context instanceof WebSubjectContext) || isNotBasedOnWebSubject) {
            return super.createSubject(context);
        }
        // 从context这个存储里面取值了
        WebSubjectContext wsc = (WebSubjectContext) context;
        SecurityManager securityManager = wsc.resolveSecurityManager();
        Session session = wsc.resolveSession();
        boolean sessionEnabled = wsc.isSessionCreationEnabled();
        PrincipalCollection principals = wsc.resolvePrincipals(); // 
        boolean authenticated = wsc.resolveAuthenticated(); // 是否认证了
        String host = wsc.resolveHost();
		// 还有request和response，是不是subject存了很多，但你却基本上没用过，没事别乱搞事喔
        ServletRequest request = wsc.resolveServletRequest();	
        ServletResponse response = wsc.resolveServletResponse();
		// 创建一个真正的Subject
        return new WebDelegatingSubject(principals, authenticated, host, session, sessionEnabled, request, response, securityManager);
    }
}
复制代码
```

以上就是Subject创建过程，如果有session就填充进去，没有就不填充，但是Subject必须创建出来。好现在让我们回到AbstractShiroFilter这个类，继续看doFilterInternal这个方法，前戏已经足够充分了，改执行shiro 过滤链了。来，come on  ...... 马上要讲完了

### AbstractShiroFilter.java （shiro filter开始分发执行了）

```
public abstract class AbstractShiroFilter extends OncePerRequestFilter {
    protected void doFilterInternal(ServletRequest servletRequest, ServletResponse servletResponse, final FilterChain chain)
            throws ServletException, IOException {

        Throwable t = null;

        try {
            final ServletRequest request = prepareServletRequest(servletRequest, servletResponse, chain);
            final ServletResponse response = prepareServletResponse(request, servletResponse, chain);

            // 已经完成
            final Subject subject = createSubject(request, response);

            //noinspection unchecked
            subject.execute(new Callable() {
                public Object call() throws Exception {
                    updateSessionLastAccessTime(request, response);
                    // 来，come on
                    executeChain(request, response, chain);
                    return null;
                }
            });
        } catch (ExecutionException ex) {
            t = ex.getCause();
        }

        if (t != null) {
            throw new ServletException(msg, t);
        }
    }
    
    // 在这里
    protected void executeChain(ServletRequest request, ServletResponse response, FilterChain origChain)
            throws IOException, ServletException {
        // 这个作用就是根据请求的URL，
        // 从 Map<String，String>这个映射中找出第一个匹配的value，以及对应的filtes，映射长这么样
        // filterChainDefinitionMap.put("/login", "casFilter");
        // filterChainDefinitionMap.put("/favicon.ico", "anon");
        // filterChainDefinitionMap.put("/**/*.html", "anon");
        // filterChainDefinitionMap.put("/**", "authc,anon"); 这就有2个喔，也就是chain中有两个filter
        FilterChain chain = getExecutionChain(request, response, origChain);
        // 这个chain就是ProxiedFilterChain（web下就是这个喔），走，去这个类看看，功能很简单
        chain.doFilter(request, response);
    }
}
复制代码
```

`ProxiedFilterChain.java` (多个filter时，执行策略)

```
public class ProxiedFilterChain implements FilterChain {
    // 执行过滤链策略，其实就是把当前chain，当成所有filter的chain,使用本地index变量来确定下一个要执行的filter
	public void doFilter(ServletRequest request, ServletResponse response) throws IOException, ServletException {
        if (this.filters == null || this.filters.size() == this.index) {

            this.orig.doFilter(request, response);
        } else {

            this.filters.get(this.index++).doFilter(request, response, this);
        }
    }
}
复制代码
```

*到现在shiro已经讲了一大半了，还剩下实际运行一个filter过程，我就拿casFilter来讲吧。*

### shiro cas 请求时序图



![img](https://user-gold-cdn.xitu.io/2019/11/25/16ea051ee8e25b20?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)



从图中可以看出来整个认证过程，我就直接将casFilter.java，这个是用户拿到了那个token，然后向服务器发起了请求，现在CasFilter.java的doFilter拦截到他了，来是时候做个了断了（下面用的Filter抽象类，很熟悉吧，前面讲过喔）：

OncePerRequestFilter.java(CasFilter的父类)

```
public abstract class OncePerRequestFilter extends NameableFilter {
    
    public final void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain){
        String alreadyFilteredAttributeName = getAlreadyFilteredAttributeName();
        if ( request.getAttribute(alreadyFilteredAttributeName) != null ) {
            filterChain.doFilter(request, response);
        } else
            if (!isEnabled(request, response) || shouldNotFilter(request) ) {

            filterChain.doFilter(request, response);
        } else {
            request.setAttribute(alreadyFilteredAttributeName, Boolean.TRUE);
            try {
                // 直接看这个吧，我直接跳到CasFilter的onAccessDenied方法吧，
                // why => 前面这块讲得真的很清楚喔
                // 不知道的请跳到前面讲 shiro filter那块
                doFilterInternal(request, response, filterChain);
            } finally {
                request.removeAttribute(alreadyFilteredAttributeName);
            }
        }
    }
}
复制代码
```

### CasFilter.java

```
public class CasFilter extends AuthenticatingFilter {
    // 前面那些步骤啥也没干，直到这里开始干活了
    protected boolean onAccessDenied(ServletRequest request, ServletResponse response) throws Exception {
        // 这个方法是父类AuthenticatingFilter的，走去这里
        return executeLogin(request, response);
    }
}

```

AuthenticatingFilter.java

```
public abstract class AuthenticatingFilter extends AuthenticationFilter {
    
    protected boolean executeLogin(ServletRequest request, ServletResponse response) throws Exception {
        // 取出请求参数里面的token，并包装成casToken
        AuthenticationToken token = createToken(request, response);
        try {
            // 看到这里清楚了吧，使用SecurityManager提供的接口，开始验证了喔
            Subject subject = getSubject(request, response);
            // 登录，走起
            subject.login(token);
            return onLoginSuccess(token, subject, request, response);
        } catch (AuthenticationException e) {
            return onLoginFailure(token, e, request, response);
        }
    }
}

DelegatingSubject.java
public class DelegatingSubject implements Subject {

    public void login(AuthenticationToken token) throws AuthenticationException {
        clearRunAsIdentitiesInternal();
        // 直接看这个吧，登录操作肯定交给securityManager了
        // step1. 开始的地方
        Subject subject = securityManager.login(this, token);

        PrincipalCollection principals;

        String host = null;

        if (subject instanceof DelegatingSubject) {
            DelegatingSubject delegating = (DelegatingSubject) subject;
            principals = delegating.principals;
            host = delegating.host;
        } else {
            principals = subject.getPrincipals();
        }

        if (principals == null || principals.isEmpty()) {
            throw new IllegalStateException(msg);
        }
        this.principals = principals;
        this.authenticated = true;
        if (token instanceof HostAuthenticationToken) {
            host = ((HostAuthenticationToken) token).getHost();
        }
        if (host != null) {
            this.host = host;
        }
        Session session = subject.getSession(false);
        if (session != null) {
            this.session = decorate(session);
        } else {
            this.session = null;
        }
    }
    
}
复制代码
DefaultSecurityManager.java
public class DefaultSecurityManager extends SessionsSecurityManager {

    public Subject login(Subject subject, AuthenticationToken token) throws AuthenticationException {
        AuthenticationInfo info;
        try {
            // step2. 为托给了父类 AuthenticatingSecurityManager
            info = authenticate(token);
        } catch (AuthenticationException ae) {
            try {
                // 失败后，估计就跳转登录了
                onFailedLogin(token, ae, subject);
            } catch (Exception e) {
            }
            throw ae; //propagate
        }
		// 认证成功，重新封装subject
        Subject loggedIn = createSubject(token, info, subject);
		
        // 这个跟rememberMe有关
        onSuccessfulLogin(token, info, loggedIn);

        return loggedIn;
    } 
}


复制代码
AuthenticatingSecurityManager.java
public abstract class AuthenticatingSecurityManager extends RealmSecurityManager {
	// step3. 继续委托给Authenticator，如果你没配置，默认就是ModularRealmAuthenticator
    // 本文还是重新配置了（建议多个realm时必须重新配置这个，
    // 除非你的认证策略跟ModularRealmAuthenticator一样）
	public AuthenticationInfo authenticate(AuthenticationToken token) throws AuthenticationException {
        // 直接进入
        return this.authenticator.authenticate(token);
    }
}
复制代码
AbstractAuthenticator.java
public abstract class AbstractAuthenticator implements Authenticator, LogoutAware {
    
    public final AuthenticationInfo authenticate(AuthenticationToken token) throws AuthenticationException {

        if (token == null) {
            throw new IllegalArgumentException("Method argument (authentication token) cannot be null.");
        }
        AuthenticationInfo info;
        try {
            // step4. do开头说明真的开始了，
            // 本文是自己实现的Authenticator（MyModularRealmAuthenticator）,去这里
            info = doAuthenticate(token);
            if (info == null) {
                throw new AuthenticationException(msg);
            }
        } catch (Throwable t) {
            AuthenticationException ae = null;
            if (t instanceof AuthenticationException) {
                ae = (AuthenticationException) t;
            }
            if (ae == null) {
                ae = new AuthenticationException(msg, t);
            }
            try {
                notifyFailure(token, ae);
            } catch (Throwable t2) {
            }


            throw ae;
        }

        notifySuccess(token, info);

        return info;
    }
}
复制代码
MyModularRealmAuthenticator.java
public class MyModularRealmAuthenticator extends ModularRealmAuthenticator { 
    
    // 当有多个realm时，应该如何使用，本文策略就是：如果是castoken就让他走casRealm,其他的走单个认真方式
    @Override
    protected AuthenticationInfo doAuthenticate(AuthenticationToken authenticationToken) throws AuthenticationException {
        // 所有Realm
        Collection<Realm> realms = getRealms();
        HashMap<String, Realm> realmHashMap = new HashMap<>(realms.size());
        for (Realm realm : realms) {
            realmHashMap.put(realm.getName(), realm);
        }
        
        if (authenticationToken instanceof CasToken) {
            // step5. 直接进入这个方法吧
            return doSingleRealmAuthentication(realmHashMap.get("casRealm"), authenticationToken);
        } else {
            return doSingleRealmAuthentication(realmHashMap.get("tokenRealm"), authenticationToken);
        }
    }
}
复制代码
ModularRealmAuthenticator.java
public class ModularRealmAuthenticator extends AbstractAuthenticator {
    
    protected AuthenticationInfo doSingleRealmAuthentication(Realm realm, AuthenticationToken token) {
        if (!realm.supports(token)) {
            throw new UnsupportedTokenException(msg);
        }
         // step6. 直接进入相应realm了，本文是CasRealm，走去casrealm看看
        AuthenticationInfo info = realm.getAuthenticationInfo(token);
        if (info == null) {
            throw new UnknownAccountException(msg);
        }
        return info;
    }  
}
复制代码
```

`AuthenticatingRealm.java`（`CasRealm`父类）

```
public abstract class AuthenticatingRealm extends CachingRealm implements Initializable {
    
    public final AuthenticationInfo getAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
		// 先查看这个realm有没有配置缓存，有的话直接从缓存里面取
        // 如果你配置CacheManager，casRealm1.setAuthorizationCachingEnabled(true)，则会使用缓存喔，这个在用户名密码登录，在这里加一个缓存，可以加快认证速度，cas则不需要（不是不需要，是不能用）
        AuthenticationInfo info = getCachedAuthenticationInfo(token);
        if (info == null) {
            //otherwise not cached, perform the lookup:
            // step7. 来这里吧，一般自己写个realm，就覆盖do开头的方法（因为覆盖就是为了干活喔）
            info = doGetAuthenticationInfo(token);
            if (token != null && info != null) {
                cacheAuthenticationInfoIfPossible(token, info);
            }
        }

        if (info != null) {
            assertCredentialsMatch(token, info);
        } 

        return info;
    }
}
复制代码
```

### CasRealm.java

```
public class CasRealm extends AuthorizingRealm {

    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {
        CasToken casToken = (CasToken) token;
        if (token == null) {
            return null;
        }
        
        String ticket = (String)casToken.getCredentials();
        if (!StringUtils.hasText(ticket)) {
            return null;
        }
        // 默认使用 Cas20ServiceTicketValidator 来进行通信，跟前端调后端接口一样
        TicketValidator ticketValidator = ensureTicketValidator();

        try {
            // step8. 来这里吧，要开始跟cas服务器通信了，验证下token的正确性
            // 这个过程就不说了，建议自己debug进去看看，是怎么通信的，我在项目里写了这个模拟，可以看看
            Assertion casAssertion = ticketValidator.validate(ticket, getCasService());
            
            AttributePrincipal casPrincipal = casAssertion.getPrincipal();
            String userId = casPrincipal.getName();


            Map<String, Object> attributes = casPrincipal.getAttributes();
            // refresh authentication token (user id + remember me)
            casToken.setUserId(userId);
            String rememberMeAttributeName = getRememberMeAttributeName();
            String rememberMeStringValue = (String)attributes.get(rememberMeAttributeName);
            boolean isRemembered = rememberMeStringValue != null && Boolean.parseBoolean(rememberMeStringValue);
            if (isRemembered) {
                casToken.setRememberMe(true);
            }

            List<Object> principals = CollectionUtils.asList(userId, attributes);
            PrincipalCollection principalCollection = new SimplePrincipalCollection(principals, getName());
            
            // 上面不多说了设置
            return new SimpleAuthenticationInfo(principalCollection, ticket);
        } catch (TicketValidationException e) { 
            throw new CasAuthenticationException("Unable to validate ticket [" + ticket + "]", e);
        }
    }
    // doGetAuthorizationInfo 这个方法必须注意，后去验证角色，角色权限，都会调用到这个方法，
    // 因此请务必重写,注意点为：1为角色roles，2角色的权限permission
    // PS: 你可以把AuthorizationInfo封装后放在session里，这样每次调用这个方法就从session里面取
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principals) {
        // retrieve user information
        SimplePrincipalCollection principalCollection = (SimplePrincipalCollection) principals;
        List<Object> listPrincipals = principalCollection.asList();
        Map<String, String> attributes = (Map<String, String>) listPrincipals.get(1);
        // create simple authorization info
        SimpleAuthorizationInfo simpleAuthorizationInfo = new SimpleAuthorizationInfo();
        // add default roles
        addRoles(simpleAuthorizationInfo, split(defaultRoles));
        // add default permissions
        addPermissions(simpleAuthorizationInfo, split(defaultPermissions));
        // get roles from attributes
        List<String> attributeNames = split(roleAttributeNames);
        for (String attributeName : attributeNames) {
            String value = attributes.get(attributeName);
            addRoles(simpleAuthorizationInfo, split(value));
        }
        // get permissions from attributes
        attributeNames = split(permissionAttributeNames);
        for (String attributeName : attributeNames) {
            String value = attributes.get(attributeName);
            addPermissions(simpleAuthorizationInfo, split(value));
        }
        return simpleAuthorizationInfo;
    }
}
复制代码
```

到这里差不多就结束了，具体使用可以参见网上使用方法，你也可以在代码里使用注解，shiro有自己的aop实现，他会把那些打注解的类，方法进行代理。

> ​	最后说下可以优化和注意的点：

1. ​	`WebSessionManager`在每次获取`session`的时候都会从`SessionDAO`里面读取，如果缓存是`redis`，这样很消耗性能，最好重写retrieveSession这个方法，将第一次获取到的Session存放到request里面去，后面每次从这里面取。
2. 就是使用`Redis`将`Session`序列化存储的时候，`SimpleSession`里面字段都是`transient`修饰的，选择序列化方案时，请注意。要么自己重写`SimpleSession`，要么选一个不会忽略`transient`的序列化方式。
3. 那些不需要认证的资源跟需要认证的资源一样都会从 `SessionDAO`获取一次`Session`，其实这个完全没必要，可以，这个也可以在`WebSessionManager`里面进行优化。
4. 不管什么请求发送到服务器，服务器都会先把请求生成一个会话保存到 会话存储的地方，如果有人一直请求，会造成 会话存储跑满，最终造成拒绝服务攻击。（解决办法，将没有认证的会话和认证过的会话放在不同的地方，也可以不保存没有认真的会话，但不保存会导致用户第一次登录认证，不会导航到用户第一次访问的那个地址，而使原先设定好的地址，这会影响用户体验）
5. 

## License

[Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0.txt)

作者：nice01qc

链接：https://juejin.im/post/6844904009610821646

来源：掘金

著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。