

### Spring Security 控制Session详解

1. 控制什么时候创建Session
2. 创建Session时低层的机制简介；
3. 什么是并发session以及对并发Sessionde设置；
4. Session 超时以及处理；
5. 防止通过URL参数来暴露Session
6. 什么是固定Session攻击以及如何避免；

------

### 控制什么时候创建Session？

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



![img](https:////upload-images.jianshu.io/upload_images/1813825-2be6393f5ce46fb5.png?imageMogr2/auto-orient/strip|imageView2/2/w/960/format/webp)

SessionCreatePolicy.java

> 通过上面的Java Configuration只能控制Spring security对session的创建，而不是控制整个应用Session的创建。

严格的机制下是不实用cookie的，在这种无状态的情况下，因此每次http请求，都需要中重新的认证。所以能够很好的处理REST API，这些严格的机制，也能很好的使用基本身份认证和摘要身份认证等身份认证机制。

------

### 运行机制

在执行授权过程之前，Spring security将会运行一个filter（SecurityContextPersistenceFilter）来存储管理不同请求之间的context。这个Context将会更具一定的策略来进行存储，默认情况下是HttpSesssionSecurityContextRepository负责，它使用的是HttpSession。

对于最严格的创建条件下，将会使用NullSecurityContextRepository

------

### 并发Session的控制

> 并发Session指的是，已经授权过的用户再次进行授权。

对并发Session的处理有两种方式：
 1，使用新创建的Session，而弃用旧的Session；
 2，是多个Session同时存在；

通过添加下面的Java Configuration来开启对并发Session的控制：

```java
@Bean
public HttpSessionEventPublisher httpSessoinEventPublisher() {
    return new HttpSessionEventPublisher();
}
```

这对于确保在销毁session时通知Spring security 注册中心是至关重要的。

启用允许一个用户有多个session是非常必要的。下面的代码是设置一个用户最多可以共存两个session。

```java
@Override
public void configure(HttpSecurity http) throws Exception {
    http.sessionManagement().maximumSessions(2);
}
```

------

### Session过期设置

可以在Spring Boot的配置文件中配置配置，Session的超时时间，如下设置Session有效期为10s；

```undefined
server.servlet.session.timeout=10
```

session超时之后，可以通过Spring Security 设置跳转的路径。

```bash
http.sessionManagement()
    .expiredUrl("/login?error=EXPIRED_SESSION")
    .and()
    .invalidSessionUrl("/login?error=INVALID_SESSION");
```

------

### 防止通过URL参数来暴露Session

在安全排行榜中，暴露Session信息的问题正在快速攀升（OWASP的TOP 10 安全问题排行榜中从2007年的第七名，攀升到2013年的第2名）。

在Spring3.0时，URL重写逻辑会在URL中追加jsessionid，其实可以通过设置来关闭。

```java
@Configuration
public class WebConfiguration implements WebApplicationInitializer {

  @Override
  public void onStartup(ServletContext servlet Context) throws ServletException {
    servletContext.setSessionTrackingModes(EnumSet.of(SessionTrackingMode.COOKIE));
  }
}
```

这能够控制是将JSESSION存储在cookie中还是URL参数中。

------

### Spring Security 固定Session的保护

> Session固定攻击：Session Fixation Attack。利用服务器的Session不变的机制，借助他人用相同的SessionId获取认证和授权，然后利用该SessionID劫持他人的session，以成功冒充他人。



![img](https:////upload-images.jianshu.io/upload_images/1813825-22692c1f1dd7d8f4.png?imageMogr2/auto-orient/strip|imageView2/2/w/567/format/webp)

Session Fixation Attact

攻击过程：
 1，攻击这Attacker登陆网站；
 2，网站返回一个SessionID=abcd；
 3，Accacker用自己的SessionID伪造一个链接（包含自己账户的SessionID），并发送Victim。
 4-5，Victim使用链接登陆网站，但是网站识别的是Accacker的sessionId，所以Accacker就有了操作victim的资源。
 6，攻击者Attacker，成功劫持Victim的会话。

攻击修复：

1，登陆成功后从新建立会话；

每次登陆成功之后，都重新生成一个SessionId。这样上面的过程4-5，Victim登陆成功了，并不会根据传递的SessionId=abcd来使用旧session，而是创建一个新的Session。

```bash
session.invalidate();
session = request.getSession(true);
```

2，Spring Security 提供了配置来避免典型的固定Session攻击。

默认情况下，Spring Security拥有这个允许 migrateSession的保护：创建一个新的Http Session，旧的Http Session将会是非法的，并将旧的Session上属性复制到新的Session上。

如果不期望以上的行为。那么有两个选择：
 （1）， 设置为none，原来的session将不会非法；
 （2），设置为newSession是，将会创建新的session，而久的session及其属性竟会被清除。

3 ，禁用Cookie和加固Cookie

我们可以使用httpOnly 和 secure 来保护cookie：
 httpOnly： 如果设置为true，浏览器叫门将不能访问cookie；
 secure: 如果设置为true，cookie将会使用https从新；

```java
@Configuration
public class WebConfiguration implements WebApplicationInitializer {

  @Override
  public void onStartup(ServletContext servlet Context) throws ServletException {
  //...      
  servletContext.getSessionCookieConfig().setHttpOnly(true);
  servletContext.getSessionCookieConfig().setSecure(true);
  }
}
```

上面的代码等价于在Spring Boot的配置文件中添加如下配置：

```bash
server.servlet.session.cookie.http-only=true
server.servlet.session.cookie.secure=true
```

------

#### 参考文章：

1. [Control the Session with Spring Security](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.baeldung.com%2Fspring-security-session)
2. [漏洞：会话固定攻击（session fixation attack）](https://www.jianshu.com/p/a5ed607cb48b)

