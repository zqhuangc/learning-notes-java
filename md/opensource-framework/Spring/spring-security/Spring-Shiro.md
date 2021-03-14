## 使用

Authentication
Authorication

```
https://crossoverjie.top/2016/07/15/SSM3/
```
*Info

 UsernamePasswordToken
 //权限验证
 doGetAuthorizationInfo(PrincipalCollection principalCollection)
 //登录前验证
 doGetAuthenticationInfo(AuthenticationToken token)


 Subject subject = SecurityUtils.getSubject() ;
UsernamePasswordToken token = new UsernamePasswordToken(user.getUserName(),user.getPassword()) ;
subject.login(token);

web.xml
org.springframework.web.filter.DelegatingFilterProxy





### 核心类

Subject (org.apache.shiro.subject.Subject) 
​    与应用交互的主体，例如用户，第三方应用等。
SecurityManager (org.apache.shiro.mgt.SecurityManager)
​    SecurityManager是shiro的核心，负责整合所有的组件，使他们能够方便快捷完成某项功能。例如：身份验证，权限验证等。
Authenticator (org.apache.shiro.authc.Authenticator)
​     认证器，负责主体认证的，这是一个扩展点，如果用户觉得Shiro默认的不好，可以自定义实现；其需要认证策略（Authentication Strategy），即什么情况下算用户认证通过了。
Authorizer (org.apache.shiro.authz.Authorizer)
​      来决定主体是否有权限进行相应的操作；即控制着用户能访问应用中的哪些功能。
SessionManager (org.apache.shiro.session.mgt.SessionManager) 
​     会话管理。
SessionDAO (org.apache.shiro.session.mgt.eis.SessionDAO) 
  数据访问对象，对session进行CRUD。
CacheManager (org.apache.shiro.cache.CacheManager)
​    缓存管理器。创建和管理缓存，为 authentication, authorization 和 session management 提供缓存数据，避免直接访问数据库，提高效率。
Cryptography (org.apache.shiro.crypto.*)
​    密码模块，提供加密组件。
Realms (org.apache.shiro.realm.Realm)
​    可以有1个或多个Realm，可以认为是安全实体数据源，即用于获取安全实体的；可以是JDBC实现，也可以是LDAP实现，或者内存实现等等；由用户提 供；注意：Shiro不知道你的用户/权限存储在哪及以何种格式存储；所以我们一般在应用中都需要实现自己的Realm。 

//Shiro生命周期处理器
LifecycleBeanPostProcessor

DefaultAdvisorAutoProxyCreator是用来扫描上下文，寻找所有的Advistor(通知器），将这些Advisor应用到所有符合切入点的Bean中。

### 独立应用

```xml
<!-- The filter-name matches name of a 'shiroFilter' bean inside applicationContext.xml -->
<filter>
    <filter-name>shiroFilter</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
    <init-param>
        <param-name>targetFilterLifecycle</param-name>
        <param-value>true</param-value>
    </init-param>
</filter>

...

<!-- Make sure any request you want accessible to Shiro is filtered. /* catches all -->
<!-- requests.  Usually this filter mapping is defined first (before all others) to -->
<!-- ensure that Shiro works in subsequent filters in the filter chain:             -->
<filter-mapping>
    <filter-name>shiroFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```



### 集成 Spring

```xml
<!-- 1、添加shiroFilter定义  -->
<!-- Shiro Filter -->  
<bean id="shiroFilter" class="org.apache.shiro.spring.web.ShiroFilterFactoryBean">  
    <property name="securityManager" ref="securityManager" />  
    <property name="loginUrl" value="/login" />  
    <property name="successUrl" value="/user/list" />  
    <property name="unauthorizedUrl" value="/login" />  
    <property name="filterChainDefinitions">  
        <value>  
            /login = anon  
            /user/** = authc  
            /role/edit/* = perms[role:edit]  
            /role/save = perms[role:edit]  
            /role/list = perms[role:view]  
            /** = authc  
        </value>  
    </property>  
</bean> 
<!-- 2、添加securityManager定义   -->
<bean id="securityManager" class="org.apache.shiro.web.mgt.DefaultWebSecurityManager">  
    <property name="realm" ref="myRealm" />  
</bean> 

<!-- 3、添加realm定义  -->
<bean id=" myRealm" class="com.jay.demo.shiro.UserRealm/>

<!-- 4、配置EhCache -->  
<bean id="cacheManager" class="org.apache.shiro.cache.ehcache.EhCacheManager" />

<!-- 5、保证实现了Shiro内部lifecycle函数的bean执行 -->

<bean id="lifecycleBeanPostProcessor" class="org.apache.shiro.spring.LifecycleBeanPostProcessor"/>

                                                                                     
                                                                                     <!-- Enable Shiro Annotations for Spring-configured beans.  Only run after -->
<!-- the lifecycleBeanProcessor has run: -->        

<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator" depends-on="lifecycleBeanPostProcessor"/>
                                   
<bean class="org.apache.shiro.spring.security.interceptor.AuthorizationAttributeSourceAdvisor">
    <property name="securityManager" ref="securityManager"/>
</bean>
```

#### 远程

* 服务端

```xml
<!-- Secure Spring remoting:  Ensure any Spring Remoting method invocations -->
<!-- can be associated with a Subject for security checks. -->
<!--  定义此bean后，必须将其插入到 Exporter 用于导出/公开服务的远程处理中。Exporter实现是根据使用的远程处理机制/协议定义的-->
<bean id="secureRemoteInvocationExecutor" class="org.apache.shiro.spring.remoting.SecureRemoteInvocationExecutor">
    <property name="securityManager" ref="securityManager"/>
</bean>


<bean name="/someService" class="org.springframework.remoting.httpinvoker.HttpInvokerServiceExporter">
    <property name="service" ref="someService"/>
    <property name="serviceInterface" value="com.pkg.service.SomeService"/>
    <property name="remoteInvocationExecutor" ref="secureRemoteInvocationExecutor"/>
</bean>
```

* 客户端

```xml
<bean id="secureRemoteInvocationFactory" class="org.apache.shiro.spring.remoting.SecureRemoteInvocationFactory"/>

<bean id="someService" class="org.springframework.remoting.httpinvoker.HttpInvokerProxyFactoryBean">
    <property name="serviceUrl" value="http://host:port/remoting/someService"/>
    <property name="serviceInterface" value="com.pkg.service.SomeService"/>
    <property name="remoteInvocationFactory" ref="secureRemoteInvocationFactory"/>
</bean>
```

### [集成CAS](http://shiro.apache.org/cas.html)



##  Shiro权限管理的默认过滤器

```java
默认过滤器(10个) 
anon -- org.apache.shiro.web.filter.authc.AnonymousFilter
authc -- org.apache.shiro.web.filter.authc.FormAuthenticationFilter
authcBasic -- org.apache.shiro.web.filter.authc.BasicHttpAuthenticationFilter
perms -- org.apache.shiro.web.filter.authz.PermissionsAuthorizationFilter
port -- org.apache.shiro.web.filter.authz.PortFilter
rest -- org.apache.shiro.web.filter.authz.HttpMethodPermissionFilter
roles -- org.apache.shiro.web.filter.authz.RolesAuthorizationFilter
ssl -- org.apache.shiro.web.filter.authz.SslFilter
user -- org.apache.shiro.web.filter.authc.UserFilter
logout -- org.apache.shiro.web.filter.authc.LogoutFilter
 
 
anon:例子/admins/**=anon 没有参数，表示可以匿名使用。
authc:例如/admins/user/**=authc表示需要认证(登录)才能使用，没有参数 
roles：例子/admins/user/**=roles[admin],参数可以写多个，多个时必须加上引号，并且参数之间用逗号分割，当有多个参数时，例如admins/user/**=roles["admin,guest"],每个参数通过才算通过，相当于hasAllRoles()方法。 
perms：例子/admins/user/**=perms[user:add:*],参数可以写多个，多个时必须加上引号，并且参数之间用逗号分割，例如/admins/user/**=perms["user:add:*,user:modify:*"]，当有多个参数时必须每个参数都通过才通过，想当于isPermitedAll()方法。 
rest：例子/admins/user/**=rest[user],根据请求的方法，相当于/admins/user/**=perms[user:method] ,其中method为post，get，delete等。 
port：例子/admins/user/**=port[8081],当请求的url的端口不是8081是跳转到schemal://serverName:8081?queryString,其中schmal是协议http或https等，serverName是你访问的host,8081是url配置里port的端口，queryString是你访问的url里的？后面的参数。 
authcBasic：例如/admins/user/**=authcBasic没有参数表示httpBasic认证 
ssl:例子/admins/user/**=ssl没有参数，表示安全的url请求，协议为https 
user:例如/admins/user/**=user没有参数表示必须存在用户，当登入操作时不做检查 

```



## [核心](https://www.infoq.com/articles/apache-shiro/)

### Subject

#### `DelegatingSubject 认证授权的调用入口`

#### InheritableThreadLocal

```
SecurityUtils.getSubject（）;
一旦获得 Subject，你就可以立即获得你希望用 Shiro 为当前用户做的 90% 的事情，如登录、登出、访问会话、执行授权检查等 
```



### SecurityManager

Subject 代表了当前用户的安全操作，SecurityManager 则管理**所有**用户的安全操作。

Web 应用通常会在 Web.xml 中指定一个 Shiro Servlet Filter，这会创建 SecurityManager 实例

Shiro 借助基于文本的[ INI ](http://en.wikipedia.org/wiki/INI_file)配置提供了一个缺省的“公共”解决方案。

```ini
[main]
cm = org.apache.shiro.authc.credential.HashedCredentialsMatcher
cm.hashAlgorithm = SHA-512
cm.hashIterations = 1024
# Base64 encoding (less text):
cm.storedCredentialsHexEncoded = false

iniRealm.credentialsMatcher = $cm

[users] 
jdoe = TWFuIGlzIGRpc3Rpbmd1aXNoZWQsIG5vdCBvbmx5IGJpcyByZWFzb2 
asmith = IHNpbmd1bGFyIHBhc3Npb24gZnJvbSBvdGhlciBhbXNoZWQsIG5vdCB
```

[main] 段落是配置 SecurityManager 对象及其使用的其他任何对象（如 Realms）的地方。

cm 对象，是 Shiro 的 HashedCredentialsMatcher 类实例。

iniRealm 对象，它被 SecurityManager 用来表示以 INI 格式定义的用户帐户。

```java
// 1。加载INI配置
Factory <SecurityManager> factory =
new IniSecurityManagerFactory（“ classpath：shiro.ini”）;

// 2。创建SecurityManager
SecurityManager securityManager = factory.getInstance（）;

// 3。使它可访问
SecurityUtils.setSecurityManager（securityManager）;
```

`SecurityManager` delegates the task of `Permission` or `Role` checking to [Authorizer](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/authz/Authorizer.html), defaulted to ModularRealmAuthorizer.







#### ModularRealmAuthorizer



### Realms

Realm 实质上是一个安全相关的[ DAO ](http://en.wikipedia.org/wiki/Data_access_object)：它封装了数据源的连接细节，并在需要时将相关数据提供给 Shiro。当配置 Shiro 时，你必须至少指定一个 Realm，用于认证和（或）授权。配置多个 Realm 是可以的，但是至少需要一个。



Shiro  内置了可以连接大量安全数据源（又名目录）的 Realm，如 LDAP、关系数据库（JDBC）、类似 INI 的文本配置资源以及属性文件等。如果缺省的 Realm 不能满足需求，你还可以插入代表自定义数据源的自己的 Realm 实现。



WildcardPermissionResolver



```ini
fooRealm = com.company.foo.Realm
barRealm = com.company.another.Realm
bazRealm = com.company.baz.Realm

securityManager.realms = $fooRealm, $barRealm, $bazRealm
```



`CredentialsMatcher` 



AuthenticationTokens

AuthenticationInfo

AuthorizingRealm



supports 方法



## Authentication

principals，credentials

```
//1. Acquire submitted principals and credentials:
AuthenticationToken token = new UsernamePasswordToken(username, password);
//2. Get the current Subject:
Subject currentUser = SecurityUtils.getSubject();

//3. Login:
currentUser.login(token);
```



### AbstractAuthenticationStrategy



## Authorization

### Role-Based Authorization

### Permission-Based Authorization

```java
// 角色名字在这里是硬编码，所以，如果你修改了角色名字或配置，你的代码就会乱套！
if ( subject.hasRole(“administrator”) ) {
    //show the ‘Create User’ button
} else {
    //grey-out the button?
}

// 通过让权限反映应用的原始功能，在改变应用功能时，你只需要改变权限检查。进而，你可以在运行时按需将权限分配给角色或用户。
if ( subject.isPermitted(“user:create”) ) {
    //show the ‘Create User’ button
} else {
    //grey-out the button?
} 

//  Instance-Level Permission Check
if ( subject.isPermitted(“user:delete:jsmith”) ) {
    //delete the ‘jsmith’ user
} else {
    //don’t delete ‘jsmith’
}
```

* `WildcardPermission`格式 -- user:create



### Annotation-based Authorization

@RequiresAuthentication

@RequiresPermissions("account:create")

@RequiresRoles("administrator")

@RequiresGuest

@RequiresUser



###  [Authorization Sequence](http://shiro.apache.org/authorization.html#authorization-sequence)

> **Step 1**: Application or framework code invokes any of the `Subject` `hasRole*`, `checkRole*`, `isPermitted*`, or `checkPermission*` method variants, passing in whatever permission or role representation is required.
>
> **Step 2**: The `Subject` instance, typically a [`DelegatingSubject`](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/subject/support/DelegatingSubject.html) (or a subclass) delegates to the application’s `SecurityManager` by calling the `securityManager`’s nearly identical respective `hasRole*`, `checkRole*`, `isPermitted*`, or `checkPermission*` method variants (the `securityManager` implements the [`org.apache.shiro.authz.Authorizer`](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/authz/Authorizer.html) interface, which defines all Subject-specific authorization methods).
>
> **Step 3**: The `SecurityManager`, being a basic ‘umbrella’ component, relays/delegates to its internal [`org.apache.shiro.authz.Authorizer`](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/authz/Authorizer.html) instance by calling the `authorizer`’s respective `hasRole*`, `checkRole*`, `isPermitted*`, or `checkPermission*` method. The `authorizer` instance is by default a [`ModularRealmAuthorizer`](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/authz/ModularRealmAuthorizer.html) instance, which supports coordinating one or more `Realm` instances during any authorization operation.
>
> **Step 4**: Each configured `Realm` is checked to see if it implements the same [`Authorizer`](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/authz/Authorizer.html) interface. If so, the Realm’s own respective `hasRole*`, `checkRole*`, `isPermitted*`, or `checkPermission*` method is called.

### [ModularRealmAuthorizer](http://shiro.apache.org/authorization.html#modularrealmauthorizer)

####  PermissionResolver

解析自定义字符串解析

> Realm Authorization Order
>
> Configuring a global `PermissionResolver`
>
> Configuring a global `RolePermissionResolver`

RolePermissionResolver

这对于支持可能没有权限概念的旧数据源或不灵活的数据源特别有用

由于将角色名称转换为权限的概念是特定于应用程序的，因此Shiro的默认`Realm`实现不使用它们

```ini
globalPermissionResolver = com.foo.bar.authz.MyPermissionResolver
...
securityManager.authorizer.permissionResolver = $globalPermissionResolver
...
permissionResolver = com.foo.bar.authz.MyPermissionResolver

realm = com.foo.bar.realm.MyCustomRealm
realm.permissionResolver = $permissionResolver
...
```



#### Custom Authorizer

```ini
[main]
globalRolePermissionResolver = com.foo.bar.authz.MyPermissionResolver
...
securityManager.authorizer.rolePermissionResolver = $globalRolePermissionResolver

...
authorizer = com.foo.bar.authz.CustomAuthorizer

securityManager.authorizer = $authorizer
```





### Session Management

```java
Session session = subject.getSession();
//Session session = subject.getSession(boolean create);
session.getAttribute(“key”, someValue);
Date start = session.getStartTimestamp();
Date timestamp = session.getLastAccessTime();
session.setTimeout(millis);
```

DefaultSessionManager

SessionListenerAdapter

SessionDAO   EHCache SessionDAO

CacheManager      ehcache.xml





SessionValidationScheduler ----- Session orphans

ExecutorServiceSessionValidationScheduler

Invalid Session Deletion

Session cluster ------  Distributed Caches



SessionStorageEvaluator

## Cryptography

### Hashing



```
// JDK的MessageDigest
try {
    MessageDigest md = MessageDigest.getInstance("MD5");
    md.digest(bytes);
    byte[] hashed = md.digest();
} catch (NoSuchAlgorithmException e) {
    e.printStackTrace();
} 

// shiro
String hex = new Md5Hash(myFile).toHex(); 

String encodedPassword = new Sha512Hash(password, salt, count).toBase64();
```

### Ciphers

```
AesCipherService cipherService = new AesCipherService();
cipherService.setKeySize(256);

// 创建一个测试密钥： 
byte[] testKey = cipherService.generateNewKey();
// 加密文件的字节： 
byte[] encrypted = cipherService.encrypt(fileBytes, testKey);
```



**路径特定的 Filter 链**

```
[main]
...
myFilter = com.company.web.some.FilterImplementation
myFilter.property1 = value1
...

[urls]
...
/some/path/** = myFilter

/assets/** = anon
/user/signup = anon
/user/** = user
/rpc/rest/** = perms[rpc:invoke], authc
/** = authc
```



## Web

EnvironmentLoaderListener

WebEnvironment ---- WebUtils.getRequiredWebEnvironment(servletContext)

ShiroFilter

IniWebEnvironment

##### [Custom Configuration Locations](http://shiro.apache.org/web.html#custom-configuration-locations)



_URL_Ant_Path_Expression_ = _Path_Specific_Filter_Chain_

等号（=）左侧的标记是相对于Web应用程序上下文根的[Ant](http://ant.apache.org/)样式的路径表达式

等号（=）右侧的标记是逗号分隔的过滤器列表

```
filter1[optional_config1], filter2[optional_config2], ..., filterN[optional_configN]
```

- *filterN*是本`[main]`中定义的过滤器bean的名称，Shiro创建一些默认`Filter`实例
- `[optional_configN]`是一个可选的带括号的字符串，对于该特定过滤器具有*特定路径的*含义（每个过滤器，*特定于路径的*配置！）。如果过滤器不需要该URL路径的特定配置，则可以将方括号删除，以使其`filterN[]`变为`filterN`。
- [PathMatchingFilter](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/filter/PathMatchingFilter.html)



##### Enabling and Disabling Filters

OncePerRequestFilter#isEnabled(request, response)

`enabled`属性



web环境 默认 `ServletContainerSessionManager`

* 本地 DefaultWebSessionManager
  * `sessionIdCookieEnabled` (a boolean)
  * `sessionIdCookie`, a [Cookie](http://shiro.apache.org/static/current/apidocs/org/apache/shiro/web/servlet/Cookie.html) instance.



`RememberMeAuthenticationToken#isRememberMe()`  --> `UsernamePasswordToken` 

`FormAuthenticationFilter` 

cookie ----  RememberMeManager

## 框架局限

常识告诉我们，Apache Shiro 不是“银弹” - 它不能毫不费力的解决所有安全问题。如下是 Shiro 还未解决，但是值得知道的：

- **虚拟机级别的问题**：Apache Shiro 当前还未处理虚拟机级别的安全，比如基于访问控制策略，阻止类加载器中装入某个类。然而，Shiro 集成现有的 JVM 安全操作并非白日做梦 - 只是没人给项目贡献这方面的工作。
- **多阶段认证**：目前，Shiro 不支持“多阶段”认证，即用户可能通过一种机制登录，当被要求再次登录时，使用另一种机制登录。这在基于 Shiro 的应用中已经实现，但是通过应用预先收集所有必需信息再跟 Shiro 交互。这个功能在 Shiro 的未来版本中非常有可能得到支持。
- **Realm 写操作**：目前所有 Realm 实现都支持“读”操作来获取验证和授权数据以执行登录和访问控制。诸如创建用户帐户、组和角色或与用户相关的角色组和权限这类“写”操作还不支持。这是因为支持这些操作的应用数据模型变化太大，很难为所有的 Shiro 用户强制定义“写”API。



#### 常见异常

- DisabledAccountException（禁用的帐号）
- LockedAccountException（锁定的帐号）
- UnknownAccountException（错误的帐号）
- ExcessiveAttemptsException（登录失败次数过多）
- IncorrectCredentialsException （错误的凭证）
- ExpiredCredentialsException（过期的凭证）



# 工作流程

```


ShiroFilterFactoryBean
FilterChainManager#
PathMatchingFilterChainResolver#setFilterChainManager
SpringShiroFilter((WebSecurityManager) securityManager, chainResolver);

AdviceFilter#doFilterInternal


protected FilterChainManager createFilterChainManager() {

        DefaultFilterChainManager manager = new DefaultFilterChainManager();
        Map<String, Filter> defaultFilters = manager.getFilters();
        //apply global settings if necessary:
        for (Filter filter : defaultFilters.values()) {
            applyGlobalPropertiesIfNecessary(filter);
        }

        //Apply the acquired and/or configured filters:
        Map<String, Filter> filters = getFilters();
        if (!CollectionUtils.isEmpty(filters)) {
            for (Map.Entry<String, Filter> entry : filters.entrySet()) {
                String name = entry.getKey();
                Filter filter = entry.getValue();
                applyGlobalPropertiesIfNecessary(filter);
                if (filter instanceof Nameable) {
                    ((Nameable) filter).setName(name);
                }
                //'init' argument is false, since Spring-configured filters should be initialized
                //in Spring (i.e. 'init-method=blah') or implement InitializingBean:
                manager.addFilter(name, filter, false);
            }
        }

        // set the global filters
        manager.setGlobalFilters(this.globalFilters);

        //build up the chains:
        Map<String, String> chains = getFilterChainDefinitionMap();
        if (!CollectionUtils.isEmpty(chains)) {
            for (Map.Entry<String, String> entry : chains.entrySet()) {
                String url = entry.getKey();
                String chainDefinition = entry.getValue();
                manager.createChain(url, chainDefinition);
            }
        }

        // create the default chain, to match anything the path matching would have missed
        manager.createDefaultChain("/**"); // TODO this assumes ANT path matching, which might be OK here

        return manager;
    }
```



## Filter  机制

```java
public enum DefaultFilter {
    anon(AnonymousFilter.class),
    authc(FormAuthenticationFilter.class),           认证相关
    authcBasic(BasicHttpAuthenticationFilter.class), 认证相关
    logout(LogoutFilter.class),
    noSessionCreation(NoSessionCreationFilter.class),
    perms(PermissionsAuthorizationFilter.class),    权限相关
    port(PortFilter.class),
    rest(HttpMethodPermissionFilter.class),
    roles(RolesAuthorizationFilter.class),          权限相关
    ssl(SslFilter.class),
    user(UserFilter.class);

    private final Class<? extends Filter> filterClass;

    private DefaultFilter(Class<? extends Filter> filterClass) {
        this.filterClass = filterClass;
    }

    public Filter newInstance() {
        return (Filter)ClassUtils.newInstance(this.filterClass);
    }

    public Class<? extends Filter> getFilterClass() {
        return this.filterClass;
    }

    public static Map<String, Filter> createInstanceMap(FilterConfig config) {
        Map<String, Filter> filters = new LinkedHashMap(values().length);
        DefaultFilter[] var2 = values();
        int var3 = var2.length;

        for(int var4 = 0; var4 < var3; ++var4) {
            DefaultFilter defaultFilter = var2[var4];
            Filter filter = defaultFilter.newInstance();
            if (config != null) {
                try {
                    filter.init(config);
                } catch (ServletException var9) {
                    String msg = "Unable to correctly init default filter instance of type " + filter.getClass().getName();
                    throw new IllegalStateException(msg, var9);
                }
            }

            filters.put(defaultFilter.name(), filter);
        }

        return filters;
    }
}

```

#### filter chain 构建

```
getObjectFromFactoryBean
doGetObjectFromFactoryBean
factory.getObject();

ShiroFilterFactoryBean#createFilterChainManager

先配置 默认的filter 配置 url  映射
后配置 开发自定义的 filter 配置 url 映射 
构造 FilterChain

整个 filterchain 以 AbstractShiroFilter 的子类 SpringShiroFilter 当做入口
由 PathMatchingFilterChainResolver 解析 根据 url 匹配到处理 filter 

OnePerRequestFilter
AbstractShiroFilter#doFilterInternal

ProxiedFilterChain#doFilter
OncePerRequestFilter#doFilter
AdviceFilter#doFilterInternal
PathMatchingFilter#prehandle
-- isFilterChainContinued
-- onPreHandle   Accesscontrolfilter
--- executeChain（一个 Chain 多个 filter）

AdviceFilter#executeChain

```



## 认证

1、通过ini配置文件创建securityManager

2、调用subject.login方法主体提交认证，提交的token

3、securityManager进行认证，securityManager最终由ModularRealmAuthenticator进行认证。

4、ModularRealmAuthenticator调用IniRealm(给realm传入token) 去ini配置文件中查询用户信息

5、IniRealm根据输入的token（UsernamePasswordToken）从 shiro.ini查询用户信息，根据账号查询用户信息（账号和密码）

​         如果查询到用户信息，就给ModularRealmAuthenticator返回用户信息（账号和密码）

​         如果查询不到，就给ModularRealmAuthenticator返回null

6、ModularRealmAuthenticator接收IniRealm返回Authentication认证信息

​         如果返回的认证信息是null，ModularRealmAuthenticator抛出异常（org.apache.shiro.authc.UnknownAccountException）
​         如果返回的认证信息不是null（说明inirealm找到了用户），对IniRealm返回用户密码 （在ini文件中存在）

​         和 token中的密码 进行对比，如果不一致抛出异常（org.apache.shiro.authc.IncorrectCredentialsException）  

### 执行流程

```
SecurityUtils#getSubject
|
currentUser.login(token);
DelegatingSubject#login
|
Subject subject = this.securityManager.login(this, token);
DefaultSecurityManager#login
// createSubject 认证成功 构造 Subject SubjectContext ***
// RememberMeManager 处理
|
AuthenticationInfo info = this.authenticate(token);
AuthenticatingSecurityManager#authenticate(AuthenticationToken token)
|
this.authenticator.authenticate(token);
AbstractAuthenticator#authenticate
// 异常 notifyFailure(); AuthenticationListener#onFailure
// 成功 notifySucess
|
AuthenticationInfo info = this.doAuthenticate(token);
ModularRealmAuthenticator#doAuthenticate
|
// realm从哪来？ RealmSecurityManager#setRealms
assertRealmsConfigured(); // 没有 realms 报错
Collection<Realm> realms = this.getRealms();
doSingleRealmAuthentication(realms.iterator().next(), authenticationToken);
or doMultiRealmAuthentication(realms, authenticationToken);
ModularRealmAuthenticator#doSingleRealmAuthentication
|
AuthenticationInfo info = realm.getAuthenticationInfo(token); 
AuthenticatingRealm#getAuthenticationInfo ****
|
getCachedAuthenticationInfo == null
AuthenticationInfo info = realm.doGetAuthenticationInfo(token);
realmImpl#doGetAuthenticationInfo
```

从login到委托给authenticator，再最后调用realm的doGetAuthenticationInfo方法。



#### `DefaultSecurityManager#createSubject`

```
protected Subject createSubject(AuthenticationToken token, AuthenticationInfo info, Subject existing) {
        SubjectContext context = createSubjectContext();
        context.setAuthenticated(true);
        context.setAuthenticationToken(token);
        context.setAuthenticationInfo(info);
        context.setSecurityManager(this);
        if (existing != null) {
            context.setSubject(existing);
        }
        return createSubject(context);
    }
    
    
public Subject createSubject(SubjectContext subjectContext) {
        //create a copy so we don't modify the argument's backing map:
        SubjectContext context = copy(subjectContext);

        //ensure that the context has a SecurityManager instance, and if not, add one:
        context = ensureSecurityManager(context);

        //Resolve an associated Session (usually based on a referenced session ID), and place it in the context before
        //sending to the SubjectFactory.  The SubjectFactory should not need to know how to acquire sessions as the
        //process is often environment specific - better to shield the SF from these details:
        context = resolveSession(context);

        //Similarly, the SubjectFactory should not require any concept of RememberMe - translate that here first
        //if possible before handing off to the SubjectFactory:
        context = resolvePrincipals(context);

        Subject subject = doCreateSubject(context);

        //save this subject for future reference if necessary:
        //(this is needed here in case rememberMe principals were resolved and they need to be stored in the
        //session, so we don't constantly rehydrate the rememberMe PrincipalCollection on every operation).
        //Added in 1.2:
        save(subject);

        return subject;
    }    
```





### 核心类

#### Token

* AuthenticationToken
  * RememberMeAuthenticationToken
    * UsernamePasswordToken
  * HostAuthenticationToken
    * UsernamePasswordToken

#### Realm

* Realm
  * CachingRealm
    - AuthenticatingRealm(Authenticate 相关)#getAuthenticationInfo ***
      - AuthorizingRealm(Authorize 相关)



##### AuthenticatingRealm#getAuthenticationInfo

```
public final AuthenticationInfo getAuthenticationInfo(AuthenticationToken token) throws AuthenticationException {

        AuthenticationInfo info = getCachedAuthenticationInfo(token);
        if (info == null) {
            //otherwise not cached, perform the lookup:
            info = doGetAuthenticationInfo(token);
            log.debug("Looked up AuthenticationInfo [{}] from doGetAuthenticationInfo", info);
            if (token != null && info != null) {
                cacheAuthenticationInfoIfPossible(token, info);
            }
        } else {
            log.debug("Using cached authentication info [{}] to perform credentials matching.", info);
        }

        if (info != null) {
            assertCredentialsMatch(token, info);
        } else {
            log.debug("No AuthenticationInfo found for submitted AuthenticationToken [{}].  Returning null.", token);
        }

        return info;
    }
    
    
    protected void assertCredentialsMatch(AuthenticationToken token, AuthenticationInfo info) throws AuthenticationException {
        CredentialsMatcher cm = getCredentialsMatcher();
        if (cm != null) {
            if (!cm.doCredentialsMatch(token, info)) {
                //not successful - throw an exception to indicate this:
                String msg = "Submitted credentials for token [" + token + "] did not match the expected credentials.";
                throw new IncorrectCredentialsException(msg);
            }
        } else {
            throw new AuthenticationException("A CredentialsMatcher must be configured in order to verify " +
                    "credentials during authentication.  If you do not wish for credentials to be examined, you " +
                    "can configure an " + AllowAllCredentialsMatcher.class.getName() + " instance.");
        }
    }
```



#### Authenticator

##### SecurityManager

* Authenticator
  * AbstractAuthenticator#authenticate
    * ModularRealmAuthenticator#doAuthenticate
  * SecurityManager
    * WebSecurityManager
      * DefaultWebSecurityManager
    * CachingSecurityManager
      * RealmSecurityManager#setRealms
        * AuthenticatingSecurityManager#authenticate
          * AuthorizingSecurityManager
            * SessionsSecurityManager
              * DefaultSecurityManager#login
                * `DefaultWebSecurityManager`


#### AuthenticationInfo

* AuthenticationInfo

  * ...

    * SimpleAuthenticationInfo
    * SimpleAccount




```java
SecurityUtils#getSubject 没有则构造新 DelegatingSubject 
DelegatingSubject#login
DefaultSecurityManager#login 获取 info 配置 DelegatingSubject

SubjectContext

DefaultSecurityManager：
public Subject login(Subject subject, AuthenticationToken token) throws AuthenticationException {
        AuthenticationInfo info;
        try {
            info = authenticate(token);
        } catch (AuthenticationException ae) {
            try {
                onFailedLogin(token, ae, subject);
            } catch (Exception e) {
                if (log.isInfoEnabled()) {
                    log.info("onFailedLogin method threw an " +
                            "exception.  Logging and propagating original AuthenticationException.", e);
                }
            }
            throw ae; //propagate
        }
        
        // 构造 Subject 上下文信息
        Subject loggedIn = createSubject(token, info, subject);

        // 根据配置 RememberMeManager 设置 标识
        // RememberMeManager-->AbstractRememberMeManager--> CookieRememberMeManager
        onSuccessfulLogin(token, info, loggedIn);

        return loggedIn;
    }
```



### 加密算法

#### Bcrypt

Shiro 框架没有内置 BCrypt 。需要我们引入新的库 `jBCrypt`:

```XML
<dependency>
    <groupId>de.svenkubiak</groupId>
    <artifactId>jBCrypt</artifactId>
    <version>0.4.1</version>
</dependency>
1234567
```

在注册用户时修改加密方式,在 BCrypt 中我们不需要为每个用户分配不同的盐，只要使用 `BCrypt.gensalt()` 就可以生成盐。

```Java
public String encodeByBCrypt(String password) {
    return BCrypt.hashpw(password, BCrypt.gensalt());
}
12345
```

在 Shiro 配置文件 `ShiroConfig` 中为我们自定义的 `MyShiroRealm` Bean 添加凭证匹配器 `HashedCredentialsMatcher`。

```Java
/**
    * 自定义的Realm
    */
@Bean(name = "myShiroRealm")
public MyShiroRealm myShiroRealm() {
    MyShiroRealm myShiroRealm = new MyShiroRealm();
    //配置单项hash
    //myShiroRealm.setCredentialsMatcher(hashedCredentialsMatcher());

    //配置 BCrypt
    myShiroRealm.setCredentialsMatcher(new CredentialsMatcher() {
        @Override
        public boolean doCredentialsMatch(AuthenticationToken token, AuthenticationInfo info) {
            UsernamePasswordToken userToken = (UsernamePasswordToken) token;
            //要验证的明文密码
            String plaintext = new String(userToken.getPassword());
            //数据库中的加密后的密文
            String hashed = info.getCredentials().toString();

            return BCrypt.checkpw(plaintext, hashed);
        }
    });
    return myShiroRealm;
}

1234567891011121314151617181920212223242526
```

到此为止，我们使用 BCrypt 算法加密了密码。如下图所示：



## 授权（访问控制）

```
需要认证的 filter 执行  isFilterChainContinued
AccessControlFilter#onPreHandle
public boolean onPreHandle(ServletRequest request, ServletResponse response, Object mappedValue) throws Exception {
        return isAccessAllowed(request, response, mappedValue) || onAccessDenied(request, response, mappedValue);
    }
    
   
RolesAuthorizationFilter    
    
doGetAuthorizationInfo  可缓存(不推荐官方的缓存方式，不便于修改)
```



####  硬编码配置处理

PermissionsAuthorizationFilter/RolesAuthorizationFilter  过滤

AccessControlFilter#onPreHandle 进行校验

 1、对subject进行授权，调用方法 isPermitted（"permission"）/hasRole

2、SecurityManager 执行授权，通过ModularRealmAuthorizer执行授权

3、ModularRealmAuthorizer执行realm（自定义的Realm）从数据库查询权限数据

　　调用realm的授权方法：doGetAuthorizationInfo
4、realm从数据库查询权限数据，返回 ModularRealmAuthorizer

5、ModularRealmAuthorizer 调用 PermissionResolver 进行权限串比对

6、如果比对后，isPermitted中"permission"在realm查询到权限数据中，说明用户访问permission串有权限，否则 没有权限，抛出异常。

```java
currentUser.hasRole("schwartz")
|
this.securityManager.hasRole(this.getPrincipals(), roleIdentifier)
|
this.authorizer.hasRole(principals, roleIdentifier)
|
AuthorizationInfo info = this.getAuthorizationInfo(principal);
return info.getRoles().contains(roleIdentifier)
|
info = this.doGetAuthorizationInfo(principals);(realm)
```

所以shiro判断用户是否有权限首先会从realm中获取用户所拥有的权限角色信息，然后再匹配当前的角色或权限是否包含，从而判定用户是否有权限！

> 硬编码：subject.hasRole(“admin”)
>
> 注解：@RequiresRoles("admin") 
>
> xml：\<shiro:hasRole name="admin">  
>
> <!-- 有权限 -->  
>
> \<shiro:hasRole>
>
> jsp : <%@ taglib prefix="shiro" uri="http://shiro.apache.org/tags"%>



#### Aop 注解处理

注解标注方法： aop

AuthorizationAttributeSourceAdvisor   shiro  注解处理

AopAllianceAnnotationsAuthorizingMethodInterceptor（Advice）

AnnotationsAuthorizingMethodInterceptor#assertAuthorized

AuthorizingAnnotationHandler(s)#assertAuthorized （策略模式找出可处理的类）

RoleAnnotationHandler#assertAuthorized    (PermissionAnnotationHandler#assertAuthorized )

getSubject().checkRole  ( SecurityUtils.getSubject() )

DelegatingSubject 相关权限判断

DelegatingSubject#checkRole  // 首先检测非匿名  

AuthorizingSecurityManager#checkRole  --> this.authorizer.checkRole(principals, role);

--ModularRealmAuthorizer#checkRole --> hasRole   --> realm(s)#hasRole --> realm#getAuthorizationInfo --> hasRole(roleIdentifier, info) --> true/false

--AuthorizingRealm#checkRole--> getAuthorizationInfo-->doGetAuthorizationInfo

AllPermission 都允许
WildcardPermission   通配符
 getParts



#### 相关MethodInterceptor

```
org.apache.shiro.authz.aop.AnnotationsAuthorizingMethodInterceptor#assertAuthorized

protected void assertAuthorized(MethodInvocation methodInvocation) throws AuthorizationException {
        //default implementation just ensures no deny votes are cast:
        Collection<AuthorizingAnnotationMethodInterceptor> aamis = getMethodInterceptors();
        if (aamis != null && !aamis.isEmpty()) {
            for (AuthorizingAnnotationMethodInterceptor aami : aamis) {
                if (aami.supports(methodInvocation)) {
                    aami.assertAuthorized(methodInvocation);
                }
            }
        }
    }
AuthorizingAnnotationMethodInterceptor#supports   策略模式，
 
public AnnotationsAuthorizingMethodInterceptor() {
        methodInterceptors = new ArrayList<AuthorizingAnnotationMethodInterceptor>(5);
        methodInterceptors.add(new RoleAnnotationMethodInterceptor());
        methodInterceptors.add(new PermissionAnnotationMethodInterceptor());
        methodInterceptors.add(new AuthenticatedAnnotationMethodInterceptor());
        methodInterceptors.add(new UserAnnotationMethodInterceptor());
        methodInterceptors.add(new GuestAnnotationMethodInterceptor());
    }
```



# session 管理

### 创建

SessionsSecurityManager#start

```
DefaultSecurityManager#login

DefaultSecurityManager#createSubject
protected Subject doCreateSubject(SubjectContext context) {
    return getSubjectFactory().createSubject(context);
}


DefaultSecurityManager#save
DefaultSubjectDAO#saveToSession
# session 创建时机？
# 若没有 session，这里调用的方法会判断是否需要创建，因为要将认证信息保存到 session 属性中
protected void saveToSession(Subject subject) {
        //performs merge logic, only updating the Subject's session if it does not match the current state:
        mergePrincipals(subject);
        mergeAuthenticationState(subject);
    } 


DelegatingSubject#getSession() 
SessionSecurityManager#start
AbstractNativeSessionManager#start
public Session start(SessionContext context) {
    Session session = createSession(context); // 创建session，类型是SimpleSession
    applyGlobalSessionTimeout(session);       // 设置session的timeout，也就是有效时间，默认30分钟
    onStart(session, context);     //DefaultWebSessionManager设置sessionIdCookie           
    notifyStart(session);                     // 通知session监听器
    //Don't expose the EIS-tier Session object to the client-tier:
    return createExposedSession(session, context);        // 创建对外暴露的 session，SimpleSession 的代理；类型是 DelegatingSession，持有sessionManager的引用，子类 DefaultWebSessionManager 中重写 SessionKey 为 WebSessionKey
}


protected Session createSession(SessionContext context) throws AuthorizationException {
    enableSessionValidationIfNecessary();        // 初次被调用时启动定时任务来验证session
    return doCreateSession(context);            // 真正创建session
}

AbstractSessionDAO#create
创建session，并生成sessionId，session是shiro的SimpleSession类型，sessionId 默认采用的是 随机的UUID字符串；

SessionIdGenerator 负责生成 sessionId



saveRequestAndRedirectToLogin
WebUtil#saveRequest
```

#### DefaultWebSessionManager

jsessionid cookie 设置与校验

#### AbstractShiroFilter#createSubject

每次请求都会创建一个 subject， 根据 sessionId 查询缓存 装配信息到 subject

sessionDAO 根据sessionid 缓存着 session，session 默认保存 principal 和 认证状态

若设置 remembermeManager 会存储 principal 到 cookie

```
protected void doFilterInternal(ServletRequest servletRequest, ServletResponse servletResponse, final FilterChain chain)
            throws ServletException, IOException {

        Throwable t = null;

        try {
            // 包装为 ShiroHttpServletRequest
            final ServletRequest request = prepareServletRequest(servletRequest, servletResponse, chain);
            // 包装为 ShiroHttpServletResponse
            final ServletResponse response = prepareServletResponse(request, servletResponse, chain);
            // 绑定 subject到当前线程,以及校验 session 
            final Subject subject = createSubject(request, response);

            //noinspection unchecked
            subject.execute(new Callable() {
                public Object call() throws Exception {
                    updateSessionLastAccessTime(request, response);
                    executeChain(request, response, chain);
                    return null;
                }
            });
     ...
    }
    
DefaultWebSubjectFactory#createSubject
根据 session 配置 subject 认证状态和 principle
 resolvePrincipals 
 public PrincipalCollection resolvePrincipals() {
        PrincipalCollection principals = getPrincipals();

        if (isEmpty(principals)) {
            //check to see if they were just authenticated:
            AuthenticationInfo info = getAuthenticationInfo();
            if (info != null) {
                principals = info.getPrincipals();
            }
        }

        if (isEmpty(principals)) {
            Subject subject = getSubject();
            if (subject != null) {
                principals = subject.getPrincipals();
            }
        }

        if (isEmpty(principals)) {
            //try the session:
            Session session = resolveSession();
            if (session != null) {
                principals = (PrincipalCollection) session.getAttribute(PRINCIPALS_SESSION_KEY);
            }
        }

        return principals;
    }
    
```



### 检索

```
DefaultSecurityManager#createSubject

protected Subject doCreateSubject(SubjectContext context) {
    return getSubjectFactory().createSubject(context);
}

DefaultSecurityManager#save(subject)
DefaultSecurityManager#resolveSession(SubjectContext)
DefaultSubjectContext#resolveSession
resolveContextSession

org.apache.shiro.mgt.SessionsSecurityManager#getSession

AbstractNativeSessionManager#getSession
lookupSession

AbstractValidatingSessionManager#doGetSession

SessionValidationScheduler   默认 1小时 校验 停止，过期
AbstractValidatingSessionManager#validateSessions
DelegatingSession

DefaultSessionManager#retrieveSession 查询
retrieveSessionFromDataSource
SessionDAO#readSession
doReadSession  查询是否有 session



redis 与 本地缓存（默认启动）
```



### sessionValidationScheduler：会话验证调度器



@Autowired`

private SessionDAO sessionDAO;`

//获取会话数量`

int size = sessionDAO.getActiveSessions().size()`

//强制退出`

Session session = sessionDAO.readSession(subject.getSession().getId());`

sessionDAO.delete(session);`

// logout，可作为强制退出`

subject.logout();`

Assert.isTrue(!subject.isAuthenticated());`



#### 