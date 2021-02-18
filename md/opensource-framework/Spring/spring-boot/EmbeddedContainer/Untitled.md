**1、Servlets、Filters & Listeners**

这些组件可以同组件扫描注册，即把他们定义为Spring Bean。

默认情况下，如果只有一个servlet，则把它映射到/；如果有多个servlet，则加上bean name作为前缀然后映射到/*。

如果默认策略不能满足你，你可以通过ServletRegistrationBean、FilterRegistrationBean和ServletListenerRegistrationBean来完全控制。

如果Filter需要按顺序执行，则可以通过@Order注解定义Filter的顺序，或者实现Ordered接口。

**容器初始化**

嵌入式容器不会直接执行Servlet 3.0+ javax.servlet.ServletContainerInitializer或org.springframework.web.WebApplicationInitializer，这是故意为之，是为了防止第三方包程序破坏Spring Boot应用程序。

如果你需要执行容器初始化，可以通过实现注册一个org.springframework.web.WebApplicationInitializer Bean。这个接口只有一个方法onStartup，这个方法可以访问ServletContext。

当使用嵌入式容器时，可以通过@ServeltComponentScan启用@WebServlet，@WebFilter和@WebListener注解。

**ServletWebApplicationContext**

ServletWebApplicationContext是一个特殊的WebApplicationContext，主要用于嵌入式Servelt。

**自定义嵌入式容器**

一般Servlet容器的普通配置可以通过Spring的Environment属性配置，也就是在application.properties文件中配置。

支持的普通配置：

1. 网络设置：server.port服务端口； server.address服务地址。
2. Session配置：server.servlet.session.presistent配置是否启用session；

- server.servlet.session.timeout配置session超时时间；
- server.servlet.session.store-dir配置session存储位置；
- server.servlet.session.cookie.*配置session的cookie。

> 错误处理： 错误页面的位置server.error.path
> ssl
> http压缩

Spring Boot尽量统一不容器的配置，但是有些配置是容器特有的，这种情况下可以使用容器特有配置，如server.tomcat，server.undertow。



**程序化定制**
如果需要以编程方式配置嵌入的servlet容器，可以注册一个实现WebServerFactoryCustomizer 接口的SpringBean。WebServerFactoryCustomizer 提供对ConfigurableServletWebServerFactory的访问，其中包括许多自定义设置器方法。以下示例以编程方式显示如何设置端口：

import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.boot.web.servlet.server.ConfigurableServletWebServerFactory;
import org.springframework.stereotype.Component;

@Component
public class CustomizationBean implements WebServerFactoryCustomizer {

```
@Override
public void customize(ConfigurableServletWebServerFactory server) {
    server.setPort(9000);
}
```

}
TomcatServletWebServerFactory, JettyServletWebServerFactory 和UndertowServletWebServerFactory是可配置的 ConfigurableServletWebServerFactory的专用子类，分别为Tomcat、Jetty和Undertow提供了额外的自定义设置方法。
直接自定义ConfigurableServletWebServerFactory
如果前面的自定义技术太有限，您可以自己注册TomcatServletWebServerFactory, JettyServletWebServerFactory或 UndertowServletWebServerFactory的bean。

```
@Bean
public ConfigurableServletWebServerFactory webServerFactory() {
    TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
    factory.setPort(9000);
    factory.setSessionTimeout(10, TimeUnit.MINUTES);
    factory.addErrorPages(new ErrorPage(HttpStatus.NOT_FOUND, "/notfound.html"));
    return factory;
}
```

为许多配置选项提供了setter。如果您需要做一些更奇特的事情，还提供了一些受保护的“hooks”方法。有关详细信息，请参阅源代码文档。

**5. JSP限制**
当运行一个使用嵌入式servlet容器的Springboot应用程序（并打包为可执行的存档文件）时，JSP支持中存在一些限制:

(1)、通过Jetty和Tomcat，war包应该可以工作。一个可执行的war包将在java -jar启动时运行，并且也可以部署到任何标准容器中。使用可执行JAR时不支持JSP
(2)、Undertow不支持JSP。
(3)、创建一个自定义的error.jsp页面不会覆盖默认的错误处理视图。应改为使用自定义错误页









#### 1

最近，我们将应用程序从运行在tomcat中的Web应用程序移植到带有嵌入式tomcat的spring启动应用程序。

运行应用程序几天后，内存和CPU使用率已达到100％。在堆转储分析中，出现了一堆未被销毁的http会话对象。

我可以在调试中看到使用配置的超时值创建的会话，比方说，5分钟。但在此之后，不会触发失效。只有在超时期限后再次请求时才会调用它。

我已经将此行为与在tomcat中运行的app进行了比较，我可以看到会话失效是由ContainerBackgroungProcessor线程[StandardManager（ManagerBase）.processExpires（）]触发的。

我没有在spring boot应用程序中看到这个后台线程。

根据一些建议做了什么：

1. application.properties中设置的会话超时：server.session.timout = 300或者在EmbeddedServletContainerCustomizer @Bean中：factory.setSessionTimout（5，TimeUnit.MINUTES）
2. 添加了HttpSessionEventPublisher和SessionRegistry bean

没有任何帮助，会话在到期时间内没有失效。

关于这个的一些线索？

作者: Andrey

 

的来源

发布者: 2019 年 7 月 16 日

### 回应 (1)

------

**5**像

决定

经过一些调试和文档阅读后，这就是原因和解决方案：

在tomcat中，有一个代表root容器生成的线程，它定期扫描容器及其子容器会话池并使它们无效。每个容器/子容器可以配置为具有自己的后台处理器来完成工作或依赖其主机的后台处理器。这由context.backgroundProcessorDelay控制

[Apache Tomcat 8配置参考](https://tomcat.apache.org/tomcat-8.0-doc/config/engine.html)

> **backgroundProcessorDelay -**
> 此值表示在此引擎上调用backgroundProcess方法与其子容器（包括所有主机和上下文）之间的延迟（以秒为单位）。如果延迟值不是负数（这意味着他们使用自己的处理线程），则不会调用子容器。将此值设置为正值将导致生成线程。等待指定的时间后，线程将在此引擎及其所有子容器上调用backgroundProcess方法。如果未指定，则此属性的默认值为10，表示延迟10秒。

在嵌入式tomcat的spring boot应用程序中，有TomcatEmbeddedServletContainerFactory.configureEngine（），它为StandardEngine [Tomcat]设置了这个属性-1，这是tomcat层次结构中的根容器，据我所知。包括Web应用程序在内的所有子容器也将此参数设置为-1。这意味着他们都依赖别人来完成这项工作。春天不这样做，没有人这样做。

我的解决方案是为app上下文设置此参数：

```
@Bean
public EmbeddedServletContainerCustomizer servletContainerCustomizer() {
    return new EmbeddedServletContainerCustomizer() {

        @Override
        public void customize(ConfigurableEmbeddedServletContainer container) {
            if (container instanceof TomcatEmbeddedServletContainerFactory) {
                TomcatEmbeddedServletContainerFactory factory = (TomcatEmbeddedServletContainerFactory) container;
                TomcatContextCustomizer contextCustomizer = new TomcatContextCustomizer() {

                    @Override
                    public void customize(Context context) {
                        context.setBackgroundProcessorDelay(10);
                    }
                };
                List<TomcatContextCustomizer> contextCustomizers = new ArrayList<TomcatContextCustomizer>();
                contextCustomizers.add(contextCustomizer);
                factory.setTomcatContextCustomizers(contextCustomizers);
                customizeTomcat(factory);
            }
        }
```