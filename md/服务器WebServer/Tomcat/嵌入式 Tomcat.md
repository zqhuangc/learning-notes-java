jmeter

# 嵌入式 Tomcat

##Web 技术栈

### Servlet 技术栈



### Web Flux（Netty）

BIO    NIO



看一个类，看整体风貌，不要只看一部分

往前看技术由来（recommend），往后看技术走向

## Web 自动装配

### API 角度分析

servlet3.0 + api 实现 web 自动装配 `ServletContainerInitializer`

Tomcat 7 是 Servlet 3.0 的实现

Tomcat 8 是 Servlet 3.1 的实现，非阻塞NIO（性能跟实际情况相关，不一定提高，可能还会差，多工、读写、同步）

### 容器角度分析

嵌入式 api

传统的 Web 应用，将 webapp 部署到 servlet 容器中。

嵌入式 Web 应用，灵活部署，任意指定位置（或者通过复杂的条件判断）





## Tomcat Maven插件

Tomcat 8 插件：tomcat8-maven-plugin

https://artifacts.alfresco.com/nexus/content/repositories/public/



```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.tomcat.maven</groupId>
            <artifactId>tomcat7-maven-plugin</artifactId>
            <version>2.2</version>
            <executions>
                <execution>
                    <id>tomcat-run</id>
                    <goals>
                        <goal>exec-war-on</goal>
                    </goals>
                    <phase>package</phase>
                    <configuration>
                        <!-- ServletContext path-->
                        <path>/</path>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

mvn -Dmaven.test.skip -U clean package

### jar 启动

`java-jar`或者 `jar` 读取 *.jar 内的 `/META-INF/MANIFEST.MF `，其中属性`Main-Class`就是引导类所在

* Tomcat7RunnerCli
  解压再读取设置
* Tomcat7Runner

>  参考 Java API：`java.util.ar.Manifest`



JarFile extends ZipFile



## Tomcat7 API 编程（Maven 插件）

Embedded/Tomcat 

### 确定 ClassPath 目录

classpath 绝对路径： ${user.dir}\target\classes

```java
String classpath  =System.getProperty("user.dir") + File.separator + "target" + File.separator + "classes";
```

### 创建 `Tomcat` 实例

```java
Tomcat tomcat = new Tomcat();
//设置端口
tomcat.setPort(9090);
// 设置 Host
/** ... */
//启动
tomcat.start();
// 阻塞
tomcat.getServer().await();
```

maven 坐标：

### 设置 `Host` 对象

```java
// 设置 Host
Host host = tomcat.getHost();
host.setName("localhost");
host.setAppBase("webapps");
```



### 设置 Classpath



Classpath 读取资源：配置、类文件

`conf/web.xml` 作为配置文件，并放置在 Classpath 目录下（绝对路径）



```java
// 设置 Context
//tomcat.addContext();
// maven 打包编译时 webapp 目录的分离
String webapp= System.getProperty("user.dir") + File.separator +"src" + File.separator + "main" + File.separator + "webapp";
String contextPath = "/";
// 设置 webapp 绝对路径到 Context，作为 docBase
Context context = tomcat.addWebapp(contextPath, webapp);
if(context instanceof StandardContext){
    StandardContext standardContext = (StandardContext)context;
    // 设置默认的 web.xml 文件到 Context
    standardContext.setDefaultWebXml(classpath + File.separator + "conf/web.xml");
    // 设置 Classpath 到 Context   
    //standardContext.setResources();
    //设置 DemoServlet 到 Tomcat 容器
    Wrapper wrapper =tomcat.addServlet(contextPath, "DemoServlet", new DemoServlet());
    wrapper.addMapping("/demo");
}

// 设置 Service
Service service = tomcat.getService();
Connector connector = new Connector();
connector.setXXX();
service.addConnector(connector);
```



## Spring Boot(1.x.x) 嵌入式 Tomcat

* EmbeddedServletContainerCustomizer
* ConfigurableEmbeddedServletContainer
  - TomcatEmbeddedServletContainerFactory
* EmbeddedServletContainer
* TomcatContextCustomizer
* TomcatConnectorCustomizer

`new SpringApplicatonBuilder().sources(...).run();`

`org.springframework.boot.context.embedded.tomcat.TomcatEmbeddedServletContainerFactory`

### 实现 EmbeddedServletContainerCustomizer

```java
@Configuration
public class TomcatConfiguration implements EmbeddedServletContainerCustomizer {
    @Override
    public void customize(ConfigurableEmbeddedServletContainer container) {
        //System.err.println(container.getClass());
        if(container instanceof TomcatEmbeddedServletContainerFactory){
            TomcatEmbeddedServletContainerFactory factory = (TomcatEmbeddedServletContainerFactory) container;

            Connector connector = new Connector();
            connector.setPort(9090);
            connector.setURIEncoding("UTF-8");
            connector.setProtocol("HTTP/1.1");
            factory.addAdditionalTomcatConnectors(connector);
        }
    }
}
```



### 自定义 `Context`

实现 `TomcatContextCustomizer`

```java
// 相当于 new TomcatContextCustomizer
 factory.addConnectorCustomizers(context -> {
               if(context instanceof StandardContext){
                   StandardContext standardContext = (StandardContext) context;
                   //standardContext.setDefaultWebXml();
               }
           });
```



### 自定义 `Connector`

实现 `TomcatConnectorCustomizer`

```java
factory.addConnectorCustomizers(connector -> {
    connector.setPort(8081);
});
```



* Bean

> pre 实例化      TomcatEmbeddedServletContainerFactory
>
> 实例化             TomcatEmbeddedServletContainerFactory
>
> post 实例化     TomcatEmbeddedServletContainerFactory
>
> pre 初始化       TomcatEmbeddedServletContainerFactory
>
> 初始化              启动 Tomcat
>
> post 初始化





