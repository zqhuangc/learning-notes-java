用 Apache Shiro 所做的事情：

1. 验证用户来核实他们的身份
2. 对用户执行访问控制，如：判断用户是否被分配了一个确定的安全角色；判断用户是否被允许做某事
3. 在任何环境下使用Session API，即使没有Web容器
4. 在身份验证，访问控制期间或在会话的生命周期，对事件作出反应
5. 聚集一个或多个用户安全数据的数据源，并作为一个单一的复合用户“视图”
6. 单点登录（SSO）功能
7. 为没有关联到登录的用户启用"Remember Me"服务
8. ...



Shiro 中有四大基石——身份验证，授权，会话管理和加密。

1. Authentication：有时也简称为“登录”，这是一个证明用户是谁的行为。
2. Authorization：访问控制的过程，也就是决定“谁”去访问“什么”。
3. Session Management：管理用户特定的会话，即使在非 Web 或 EJB 应用程序。
4. Cryptography：通过使用加密算法保持[数据安全](https://cloud.tencent.com/solution/data_protection?from=10680)同时易于使用。

除此之外，Shiro 也提供了额外的功能来解决在不同环境下所面临的安全问题，尤其是以下这些：

1. Web Support：Shiro 的 web 支持的 API 能够轻松地帮助保护 Web 应用程序。
2. Caching：缓存是 Apache Shiro 中的第一层公民，来确保安全操作快速而又高效。
3. Concurrency：Apache Shiro 利用它的并发特性来支持多线程应用程序。
4. Testing：测试支持的存在来帮助你编写单元测试和集成测试。
5. "Run As"：一个允许用户假设为另一个用户身份（如果允许）的功能，有时候在管理脚本很有用。
6. "Remember Me"：在会话中记住用户的身份，这样用户只需要在强制登录时候登录。



Shiro 的学习资料并不多，没看到有相关的书籍。张开涛的《跟我学Shiro》是一个非常不错的资料，小伙伴可以搜索了解下。



### filter

```
ServletContextSupport
org.apache.shiro.web.servlet.AbstractFilter

org.apache.shiro.web.servlet.OncePerRequestFilter
AdviceFilter
PathMatchingFilter
```

```
AccessControlFilter
AuthenticationFilter
- PassThruAuthenticationFilter
- AuthenticatingFilter
-- HttpAuthenticationFilter
--- BearerHttpAuthenticationFilter
--- BasicHttpAuthenticationFilter
-- FormAuthenticationFilter
AuthorizationFilter
-- PermissionsAuthorizationFilter
--- HttpMethodPermissionFilter
-- RolesAuthorizationFilter
```