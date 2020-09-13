打包可执行 jar

#### 1、添加插件

```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
            <configuration>
                <classifier>exec-${env}</classifier>
            </configuration>
        </execution>
    </executions>
</plugin>
```

这里是添加了一个 Maven 打包插件，通过配置可以定制打成的 Jar 包的格式，如：javastack-exec-dev.jar。

如果你是用的 spring-boot-starter-parent 方式来使用 Spring Boot，那就不用写 executions 选项，因为它包括了 executions repackage 构建配置。



**这个插件的更多用法参考：**

> https://docs.spring.io/spring-boot/docs/2.1.3.RELEASE/maven-plugin/usage.html



#### 2、打成 Jar 包

使用 mvn package 命令或者 IDE 中的 Maven 插件都可以打包。



#### 3、运行 Jar 包

运行命令格式：

> $ java -jar xxx.jar --args



repackage 功能的 作用，就是在打包的时候，多做一点额外的事情：

1. 首先 `mvnpackage` 命令 对项目进行打包，打成一个 `jar`，这个 `jar` 就是一个普通的 `jar`，可以被其他项目依赖，但是不可以被执行
2. `repackage` 命令，对第一步 打包成的 `jar` 进行再次打包，将之打成一个 可执行 `jar` ，通过将第一步打成的 `jar`重命名为 `*.original` 文件



可执行 jar 中，我们自己的代码是存在 于 `BOOT-INF/classes/` 目录下，另外，还有一个 `META-INF` 的目录，该目录下有一个 `MANIFEST.MF` 文件，打开该文件，内容如下：

```
Manifest-Version: 1.0
Implementation-Title: restful
Implementation-Version: 0.0.1-SNAPSHOT
Start-Class:org.javaboy.restful.RestfulApplication
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Build-Jdk-Spec: 1.8
Spring-Boot-Version: 2.1.6.RELEASE
Created-By: Maven Archiver 3.4.0
Main-Class: org.springframework.boot.loader.JarLauncher
```

可以看到，这里定义了一个 `Start-Class`，这就是可执行 `jar` 的入口类， `Spring-Boot-Classes` 表示我们自己代码编译后的位置， `Spring-Boot-Lib` 则表示项目依赖的 `jar` 的位置。

换句话说，如果自己要打一个可执行 `jar` 包的话，除了添加相关依赖之外，还需要配置 `META-INF/MANIFEST.MF` 文件。

#### 打包可执行 和 普通 jar



```xml
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
    <executions>
        <execution>
            <goals>
                <goal>repackage</goal>
            </goals>
            <configuration>
                <classifier>exec-${env}</classifier>
            </configuration>
        </execution>
    </executions>
</plugin>
```

配置的 `classifier` 表示可执行 `jar` 的名字，配置了这个之后，在插件执行 `repackage` 命令时，就不会给 `mvn package` 所打成的 `jar` 重命名了



### 打包 war

#### 修改 pom

```
 <!--因配置外部TOMCAT 而配置-->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-tomcat</artifactId>
    <scope>provided</scope>
</dependency>

```



#### 修改启动类

```
@SpringBootApplication
public class MyApplication extends SpringBootServletInitializer{
	@Override
	protected SpringApplicationBuilder configure(SpringApplicationBuilder builder) {
		return builder.sources(MyApplication.class);
	}


	public static void main(String[] args) {
		SpringApplication.run(MyApplication.class, args);
	}
}

/**
关键：继承SpringBootServletInitializer并且重写了configure方法
*/
```



### jsp 资源文件



```java
@EnableWebMvc //mvc:annotation-driven
@Configuration
@ComponentScan({ "com.xxx.web" })
public class SpringWebConfig extends WebMvcConfigurerAdapter {
 
	@Override
	public void addResourceHandlers(ResourceHandlerRegistry registry) {

   registry.addResourceHandler("/resources/**").addResourceLocations("/resources/");
	}
	
	@Bean
	public InternalResourceViewResolver viewResolver() {
		InternalResourceViewResolver viewResolver = new InternalResourceViewResolver();
		viewResolver.setViewClass(JstlView.class);
		viewResolver.setPrefix("/WEB-INF/jsp/");
		viewResolver.setSuffix(".jsp");
		return viewResolver;
	}
 
}

或者静态属性配置

spring.mvc.static-path-pattern=/resources/**


也可以使用静态文件动态化

spring.resources.chain.strategy.content.enabled=true
spring.resources.chain.strategy.content.paths=/**
spring.resources.chain.strategy.fixed.enabled=true
spring.resources.chain.strategy.fixed.paths=/js/lib/
spring.resources.chain.strategy.fixed.version=v12
```







#### 其他

**添加war包打包插件**

如果你用的是继承spring-boot-starter-parent的形式使用Spring Boot，那可以跳过，因为它已经帮你配置好了。如果你使用的依赖spring-boot-dependencies形式，你需要添加以下插件。

```
<plugin>    
<groupId>org.apache.maven.plugins</groupId>    
<artifactId>maven-war-plugin</artifactId>    
<configuration>        
<failOnMissingWebXml>false</failOnMissingWebXml>
</configuration>
</plugin>
```

> failOnMissingWebXml需要开启为false，不然打包会报没有web.xml错误。





**打包出现某jar包无法打入**

实际是可以下载，但是无法将此打入包中

解决办法:

在pom.xml中添加

```
  <repositories>  
     <repository>  
            <id>spring-milestone</id>  
            <url>http://repo.spring.io/libs-release</url>  
     </repository>  
   </repositories> 
```







# 前言

https://mp.weixin.qq.com/s?src=3&timestamp=1574502249&ver=1&signature=zk7FQD8jLGZt5GgGGRFajBBIAxwv7ZSB5Pttm7TCsW7GqbzM*fPcvsDUDKU*Bjdx2zOCCCeLgxuXNMMvViS3X0uRbDZFv7Stiwh6rJGzmTtopG6QxRCs807hIhq37L-EGsH5rRprcT7nEDcpAhKjLBmsx5vS*ZW9sDWd8yCoFwA=



上一篇介绍了Spring Boot中使用Thymeleaf模板引擎，今天来介绍一下如何使用SpringBoot官方不推荐的jsp,虽然难度有点大，但是玩起来还是蛮有意思的。

# 正文

先来看看整体的框架结构，跟前面介绍Thymeleaf的时候差不多，只是多了webapp这个用来存放jsp的目录，静态资源还是放在resources的static下面。

![img](http://mmbiz.qpic.cn/mmbiz_jpg/pcT41pYWuUGIFq5OADicTfIXZlkBfA7GZYqib1lUib4fROeGpiac74d8y6J5Wky0JBvuhT530VwSQuUTibmxkYhr4zA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

# 引入依赖

![img](http://mmbiz.qpic.cn/mmbiz_jpg/pcT41pYWuUGIFq5OADicTfIXZlkBfA7GZfqV3600pKD2J1fDqrf5ceOnZiaGkDibLa9WGIP47X69jfiaHjrOtpmeEg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

使用内嵌的tomcat容器来运行的话只要这3个就好了。这里介绍下maven中scope依赖范围的概念，因为后续涉及到这个会有问题。

依赖范围就是用来控制依赖和三种classpath(编译classpath，测试classpath、运行classpath)的关系，Maven有如下几种依赖范围：

- compile:编译依赖范围。如果没有指定，就会默认使用该依赖范围。使用此依赖范围的Maven依赖，对于编译、测试、运行三种classpath都有效。典型的例子是spring-code,在编译、测试和运行的时候都需要使用该依赖。
- test: 测试依赖范围。使用次依赖范围的Maven依赖，只对于测试classpath有效，在编译主代码或者运行项目的使用时将无法使用此依赖。典型的例子是Jnuit,它只有在编译测试代码及运行测试的时候才需要。
- provided:已提供依赖范围。使用此依赖范围的Maven依赖，对于编译和测试classpath有效，但在运行时候无效。典型的例子是servlet-api,编译和测试项目的时候需要该依赖，但在运行项目的时候，由于容器以及提供，就不需要Maven重复地引入一遍。

# application.properties配置

要支持jsp，需要在application.properties中配置返回文件的路径以及类型![img](http://mmbiz.qpic.cn/mmbiz_jpg/pcT41pYWuUGIFq5OADicTfIXZlkBfA7GZicdHXrUrTTEUmuV06HDUod91yGdBR0jYHK8jd3ZNRoicibwPMjXbaxibRQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这里指定了返回文件类型为jsp,路径是在/WEB-INF/jsp/下面。

# 控制类

上面步骤有了，这里就开始写控制类，直接上简单的代码，跟正常的springMVC没啥区别：

![img](http://mmbiz.qpic.cn/mmbiz_jpg/pcT41pYWuUGIFq5OADicTfIXZlkBfA7GZxFV98ZBXlgkxyxHZ5jnPaXbrJU0VUrdjub2tomwz2Irbq3gwohEC9A/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

# jsp页面编写

![img](http://mmbiz.qpic.cn/mmbiz_jpg/pcT41pYWuUGIFq5OADicTfIXZlkBfA7GZUaqf5oaITs1RnKCI3bR4wp8D6zb8cO7uzOtlaZNTAOV5rPNAv63t2A/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

# 启动类

启动类不变还是最简单的

![img](http://mmbiz.qpic.cn/mmbiz_jpg/pcT41pYWuUGIFq5OADicTfIXZlkBfA7GZ1c5qUTEO628M2xTF5icNkN5GO9ERq1NHbfSFndibkeey3uib62wFnl6hQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

# 内嵌Tomcat容器运行项目

基本配置好了就可以启动项目，通过http://localhost:8080/learn 访问，我使用的SpringBoot是１.５.2版本，jdk1.8,以前介绍过，运行项目有三种方式，这里我都做过了一次测试，发现在maven中jasper依赖有加provided和注释掉该依赖范围运行的效果不大一样，具体对比如下：

有添加provided的情况：

- 右键运行启动类，访问页面报404错误
- 使用spring-boot:run运行正常
- 打包成jar，通过 java -jar demo-0.0.1-SNAPSHOT.jar 运行报错
- 打包成war，通过 java -jar demo-0.0.1-SNAPSHOT.war 运行正常

把provided 注释掉的情况：

- 右键运行启动类，访问页面正常
- spring-boot:run运行 访问页面正常
- 打包成jar，通过 java -jar demo-0.0.1-SNAPSHOT.jar 运行报错
- 打包成war，通过 java -jar demo-0.0.1-SNAPSHOT.war 运行正常

我测试了好几次都是这样，就是有加provided的时候，右键运行启动类访问页面的时候，提示404错误。

其他3种情况都一样， jar运行也报404，spring-boot:run以及war运行都可以。

为什么jar包运行不行呢，我们打开打包的jar和war分别看看区别，如下2图所示：

![img](http://mmbiz.qpic.cn/mmbiz_jpg/pcT41pYWuUGIFq5OADicTfIXZlkBfA7GZ3nu3jf2pUNTOe771BwY53f5S1GOduibJ0aVqN38LARWy83ScGDicluUw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![img](http://mmbiz.qpic.cn/mmbiz_jpg/pcT41pYWuUGIFq5OADicTfIXZlkBfA7GZVGzc2EGjuV5YcGXJFfVEd9TsshISsiaJHOp8Yv2eMwNu7ic3Ih1aLiakg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

从上面可以看出来，jar包运行的时候会404错误，因为默认jsp不会被拷贝进来，而war包里面有包含了jsp，所以没问题。

# 内嵌Tomcat属性配置

关于Tomcat的偶有属性都在org.springframework.boot.autoconfigure.web.ServerProperties配置类中做了定义，我们只需在application.properties配置属性做配置即可。通用的Servlet容器配置都已”server”左右前缀，而Tomcat特有配置都以”server.tomcat”作为前缀。下面举一些常用的例子。

配置Servlet容器：

![img](http://mmbiz.qpic.cn/mmbiz_jpg/pcT41pYWuUGIFq5OADicTfIXZlkBfA7GZQZG2UVBAT94fUUrWOlnt856FlMHM3JeEvfLNYsiaQauhfZcGP1PO0JQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

配置Tomcat：

![img](http://mmbiz.qpic.cn/mmbiz_jpg/pcT41pYWuUGIFq5OADicTfIXZlkBfA7GZnPlXq7jLiaBnCNyYbiauibuh3tuQhkucwbnynIpawicPepU6iaPh5N2OrEA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

更为详细的Servlet容器配置以及Tomcat配置，可以前往博主之前文章查看：Spring Boot干货系列：常用属性汇总

# 外部的Tomcat服务器部署war包

Spring Boot项目需要部署在外部容器中的时候，Spring Boot导出的war包如果直接在Tomcat的部署会报错，不信你可以试试看。

需要做到下面两点修改才可以：

- 继承SpringBootServletInitializer

  外部容器部署的话，就不能依赖于Application的main函数了，而是要以类似于web.xml文件配置的方式来启动Spring应用上下文，此时我们需要在启动类中继承SpringBootServletInitializer并实现configure方法：

![img](http://mmbiz.qpic.cn/mmbiz_jpg/pcT41pYWuUGIFq5OADicTfIXZlkBfA7GZ54WD3pticbGRHA1cqrT11Q5NQicFSmiaXhkUu6s0iaD71AzIZRzicfrib7pg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

这个类的作用与在web.xml中配置负责初始化Spring应用上下文的监听器作用类似，只不过在这里不需要编写额外的XML文件了。

- pom.xml修改tomcat相关的配置

  如果要将最终的打包形式改为war的话，还需要对pom.xml文件进行修改，因为spring-boot-starter-web中包含内嵌的tomcat容器，所以直接部署在外部容器会冲突报错。这里有两种方法可以解决，如下

  方法一：

![img](http://mmbiz.qpic.cn/mmbiz_jpg/pcT41pYWuUGIFq5OADicTfIXZlkBfA7GZ9aayib6OaaQmNhTRBZgMon5OTkBVJm1CvfseGr95qSHIib5lVnPJyMSA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在这里需要移除对嵌入式Tomcat的依赖，这样打出的war包中，在lib目录下才不会包含Tomcat相关的jar包，否则将会出现启动错误。

还有一个很关键的关键点，就是tomcat-embed-jasper中scope必须是provided。

![img](http://mmbiz.qpic.cn/mmbiz_jpg/pcT41pYWuUGIFq5OADicTfIXZlkBfA7GZHtg4XoC7nA9AAmwqFdyI7b2mgl0cIeQ573tN4Hdne5bwb5XF3ugCMQ/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

因为SpringBootServletInitializer需要依赖 javax.servlet，而tomcat-embed-jasper下面的tomcat-embed-core中就有这个javax.servlet，如果没用provided，最终打好的war里面会有servlet-api这个jar，这样就会跟tomcat本身的冲突了。这个关键点同样适应于下面说的第二种方法。

![img](http://mmbiz.qpic.cn/mmbiz_jpg/pcT41pYWuUGIFq5OADicTfIXZlkBfA7GZndl1p3PuKpo4HOLZ2L5g7csvoBob1b2bbnGf7uID4HDmIhMeS3ic8GA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

方法二：

直接添加如下配置即可：

![img](http://mmbiz.qpic.cn/mmbiz_jpg/pcT41pYWuUGIFq5OADicTfIXZlkBfA7GZ2HZWUJUsXk1U87o9XhvEicVIC0L9FiaHFxcacMgz7awcBbJkjUaGw4uw/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

provided的作用上面已经介绍的很透彻了，这里就不啰嗦了，这种方式的好处是，打包的war包同时适合java -jar命令启动以及部署到外部容器中。

如果你不喜欢默认的打包名称，你可以通过节点里添加内容。

![img](http://mmbiz.qpic.cn/mmbiz_jpg/pcT41pYWuUGIFq5OADicTfIXZlkBfA7GZVWFEXzEibIFEX9uPlhNbGlRZJxc11Y3ayAbxUCEaibSrchd9tuwDpCWg/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

最后启动tomcat输入http://localhost:8080/springBootJsp/learn 查看效果，还是美美哒

![img](http://mmbiz.qpic.cn/mmbiz_jpg/pcT41pYWuUGIFq5OADicTfIXZlkBfA7GZXZlYhqN89XSvH9aOApDErRicNSXKgNo1gtVyD4H8g8cwOjXSqLrVyUA/640?wx_fmt=jpeg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

# 关于使用jar部署

上面已经测试过了，正常情况下包含jsp的页面是无法用jar的运行的，因为jsp默认是在webapp目录下，可是打包成jar是没有webapp这个目录结构的。

虽然网上有介绍说通过pom.xml配置，把WEB-INF目录复制到META-INF/resources下面。但是博主试了一整天还是访问不了，最后放弃了。各位如何有兴趣可以继续尝试，毕竟war也可以通过java -jar命令来启动的不是么。











## **spring-loaded**

**在Plugins中添加依赖**

```
<build>
    <plugins>
       <plugin>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-maven-plugin</artifactId>
           <dependencies>
                   <!-- spring热部署 -->
             <dependency>
                <groupId>org.springframework</groupId>
                <artifactId>springloaded</artifactId>
                <version>1.2.6.RELEASE</version>
              </dependency>
           </dependencies>
       </plugin>
    </plugins>
</build>
```

# [打包原理](https://mp.weixin.qq.com/s?src=11&timestamp=1574490576&ver=1991&signature=XEqWlXlXHpIWxsl*3LL2ag4drJvXqUa5fudM9vyH1fAc-46CbBZgAB4K7cAraC1bxw8JTJimAbXNMEZJSdAqLr6VmFplKLzN86z0U0Z3ecOFAQeydf5-zjuQGRgs6TlY&new=1)

```java
private void repackage() throws MojoExecutionException {
   //获取使用maven-jar-plugin生成的jar，最终的命名将加上.orignal后缀
  Artifact source = getSourceArtifact();
  //最终文件，即Fat jar
  File target = getTargetFile();
  //获取重新打包器，将重新打包成可执行jar文件
  Repackager repackager = getRepackager(source.getFile());
  //查找并过滤项目运行时依赖的jar
  Set<Artifact> artifacts = filterDependencies(this.project.getArtifacts(),
     getFilters(getAdditionalFilters()));
  //将artifacts转换成libraries
  Libraries libraries = new ArtifactsLibraries(artifacts, this.requiresUnpack,
     getLog());
  try {
    //提供Spring Boot启动脚本
   LaunchScript launchScript = getLaunchScript();
    //执行重新打包逻辑，生成最后fat jar
   repackager.repackage(target, libraries, launchScript);
  }
  catch (IOException ex) {
   throw new MojoExecutionException(ex.getMessage(), ex);
  }
  //将source更新成 xxx.jar.orignal文件
  updateArtifact(source, target, repackager.getBackupFile());
}
```



**前言**

文章篇幅较长，但是包含了SpringBoot 可执行jar包从头到尾的原理，请读者耐心观看。同时文章是基于SpringBoot-2.1.3进行分析。涉及的知识点主要包括Maven的生命周期以及自定义插件，JDK提供关于jar包的工具类以及Springboot如何扩展，最后是自定义类加载器。

**spring-boot-maven-plugin**

SpringBoot 的可执行jar包又称fat jar ，是包含所有第三方依赖的 jar 包，jar 包中嵌入了除 java 虚拟机以外的所有依赖，是一个 all-in-one jar 包。普通插件maven-jar-plugin生成的包和spring-boot-maven-plugin生成的包之间的直接区别，是fat jar中主要增加了两部分，第一部分是lib目录，存放的是Maven依赖的jar包文件,第二部分是spring boot loader相关的类。



```
fat jar 
目录结构
├─BOOT-INF│  
	├─classes│  
		└─lib
├─META-INF│
	├─maven│
		├─app.properties│
├─MANIFEST.MF
	└─org
        └─springframework        
           └─boot            
              └─loader
                 ├─archive
                    ├─data 
                       ├─jar
                          └─util
```

也就是说想要知道fat jar是如何生成的，就必须知道spring-boot-maven-plugin工作机制，而spring-boot-maven-plugin属于自定义插件，因此我们又必须知道，Maven的自定义插件是如何工作的

**Maven的自定义插件**

Maven 拥有三套相互独立的生命周期: clean、default 和 site, 而每个生命周期包含一些phase阶段, 阶段是有顺序的, 并且后面的阶段依赖于前面的阶段。生命周期的阶段phase与插件的目标goal相互绑定，用以完成实际的构建任务。

```
<plugin>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-maven-plugin</artifactId>
  <executions>
    <execution>
      <goals>
        <goal>repackage</goal>
      </goals>
    </execution>
  </executions>
</plugin>
```

repackage目标对应的将执行到org.springframework.boot.maven.RepackageMojo#execute，该方法的主要逻辑是调用了org.springframework.boot.maven.RepackageMojo#repackage



```

private void repackage() throws MojoExecutionException {
   //获取使用maven-jar-plugin生成的jar，最终的命名将加上.orignal后缀
  Artifact source = getSourceArtifact();
  //最终文件，即Fat jar
  File target = getTargetFile();
  //获取重新打包器，将重新打包成可执行jar文件
  Repackager repackager = getRepackager(source.getFile());
  //查找并过滤项目运行时依赖的jar
  Set<Artifact> artifacts = filterDependencies(this.project.getArtifacts(),
     getFilters(getAdditionalFilters()));
  //将artifacts转换成libraries
  Libraries libraries = new ArtifactsLibraries(artifacts, this.requiresUnpack,
     getLog());
  try {
    //提供Spring Boot启动脚本
   LaunchScript launchScript = getLaunchScript();
    //执行重新打包逻辑，生成最后fat jar
   repackager.repackage(target, libraries, launchScript);
  }
  catch (IOException ex) {
   throw new MojoExecutionException(ex.getMessage(), ex);
  }
  //将source更新成 xxx.jar.orignal文件
  updateArtifact(source, target, repackager.getBackupFile());
}
```

我们关心一下org.springframework.boot.maven.RepackageMojo#getRepackager这个方法，知道Repackager是如何生成的，也就大致能够推测出内在的打包逻辑。



```

private Repackager getRepackager(File source) {
  Repackager repackager = new Repackager(source, this.layoutFactory);
  repackager.addMainClassTimeoutWarningListener(
     new LoggingMainClassTimeoutWarningListener());
  //设置main class的名称，如果不指定的话则会查找第一个包含main方法的类，repacke最后将会设置org.springframework.boot.loader.JarLauncher
  repackager.setMainClass(this.mainClass);
  if (this.layout != null) {
   getLog().info("Layout: " + this.layout);
    //重点关心下layout 最终返回了 org.springframework.boot.loader.tools.Layouts.Jar
   repackager.setLayout(this.layout.layout());
  }
  return repackager;
}
```

```

/**
 * Executable JAR layout.
 */
public static class Jar implements RepackagingLayout {
  @Override
  public String getLauncherClassName() {
   return "org.springframework.boot.loader.JarLauncher";
  }
  @Override
  public String getLibraryDestination(String libraryName, LibraryScope scope) {
   return "BOOT-INF/lib/";
  }
  @Override
  public String getClassesLocation() {
   return "";
  }
  @Override
  public String getRepackagedClassesLocation() {
   return "BOOT-INF/classes/";
  }
  @Override
  public boolean isExecutable() {
   return true;
  }
}
```

layout我们可以将之翻译为文件布局，或者目录布局，代码一看清晰明了，同时我们需要关注，也是下一个重点关注对象org.springframework.boot.loader.JarLauncher，从名字推断，这很可能是返回可执行jar文件的启动类。

**MANIFEST.MF文件内容**



```
Manifest-Version: 1.0
Implementation-Title: oneday-auth-server
Implementation-Version: 1.0.0-SNAPSHOT
Archiver-Version: Plexus Archiver
Built-By: oneday
Implementation-Vendor-Id: com.oneday
Spring-Boot-Version: 2.1.3.RELEASE
Main-Class: org.springframework.boot.loader.JarLauncher
Start-Class: com.oneday.auth.Application
Spring-Boot-Classes: BOOT-INF/classes/
Spring-Boot-Lib: BOOT-INF/lib/
Created-By: Apache Maven 3.3.9
Build-Jdk: 1.8.0_171
```



repackager生成的MANIFEST.MF文件为以上信息，可以看到两个关键信息Main-Class和Start-Class。我们可以进一步，程序的启动入口并不是我们SpringBoot中定义的main，而是JarLauncher#main，而再在其中利用反射调用定义好的Start-Class的main方法

**JarLauncher**

**重点类介绍**

- java.util.jar.JarFile JDK工具类提供的读取jar文件

- org.springframework.boot.loader.jar.JarFileSpringboot-loader 继承JDK提供JarFile类

- java.util.jar.JarEntryDK工具类提供的``jar```文件条目

- org.springframework.boot.loader.jar.JarEntry Springboot-loader 继承JDK提供JarEntry类

- org.springframework.boot.loader.archive.Archive Springboot抽象出来的统一访问资源的层

- - JarFileArchivejar包文件的抽象
  - ExplodedArchive文件目录

这里重点描述一下JarFile的作用，每个JarFileArchive都会对应一个JarFile。在构造的时候会解析内部结构，去获取jar包里的各个文件或文件夹类。我们可以看一下该类的注释。



```
/* Extended variant of {@link java.util.jar.JarFile} that behaves in the same way but
* offers the following additional functionality.
* <ul>
* <li>A nested {@link JarFile} can be {@link #getNestedJarFile(ZipEntry) obtained} based
* on any directory entry.</li>
* <li>A nested {@link JarFile} can be {@link #getNestedJarFile(ZipEntry) obtained} for
* embedded JAR files (as long as their entry is not compressed).</li>
**/ </ul>
```



jar里的资源分隔符是!/，在JDK提供的JarFile URL只支持一个'!/‘，而Spring boot扩展了这个协议，让它支持多个'!/‘，就可以表示jar in jar、jar in directory、fat jar的资源了。

**自定义类加载机制**

-   最基础：Bootstrap ClassLoader（加载JDK的/lib目录下的类）
-   次基础：Extension ClassLoader（加载JDK的/lib/ext目录下的类）
-   普通：Application ClassLoader（程序自己classpath下的类）

首先需要关注双亲委派机制很重要的一点是，如果一个类可以被委派最基础的ClassLoader加载，就不能让高层的ClassLoader加载，这样是为了范围错误的引入了非JDK下但是类名一样的类。其二，如果在这个机制下，由于fat jar中依赖的各个第三方jar文件，并不在程序自己classpath下，也就是说，如果我们采用双亲委派机制的话，根本获取不到我们所依赖的jar包，因此我们需要修改双亲委派机制的查找class的方法，自定义类加载机制。

先简单的介绍Springboot2中LaunchedURLClassLoader，该类继承了java.net.URLClassLoader,重写了java.lang.ClassLoader#loadClass(java.lang.String, boolean)，然后我们再探讨他是如何修改双亲委派机制。

在上面我们讲到Spring boot支持多个'!/‘以表示多个jar，而我们的问题在于，如何解决查找到这多个jar包。我们看一下LaunchedURLClassLoader的构造方法。

```
public LaunchedURLClassLoader(URL[] urls, ClassLoader parent) {
  super(urls, parent);
}
```

urls注释解释道the URLs from which to load classes and resources，即fat jar包依赖的所有类和资源，将该urls参数传递给父类java.net.URLClassLoader，由父类的java.net.URLClassLoader#findClass执行查找类方法，该类的查找来源即构造方法传递进来的urls参数

```

//LaunchedURLClassLoader的实现
protected Class<?> loadClass(String name, boolean resolve)
   throws ClassNotFoundException {
  Handler.setUseFastConnectionExceptions(true);
  try {
   try {
     //尝试根据类名去定义类所在的包，即java.lang.Package，确保jar in jar里匹配的manifest能够和关联        //的package关联起来
     definePackageIfNecessary(name);
   }
   catch (IllegalArgumentException ex) {
     // Tolerate race condition due to being parallel capable
     if (getPackage(name) == null) {
      // This should never happen as the IllegalArgumentException indicates
      // that the package has already been defined and, therefore,
      // getPackage(name) should not return null.
        
      //这里异常表明，definePackageIfNecessary方法的作用实际上是预先过滤掉查找不到的包
      throw new AssertionError("Package " + name + " has already been "
         + "defined but it could not be found");
     }
   }
   return super.loadClass(name, resolve);
  }
  finally {
   Handler.setUseFastConnectionExceptions(false);
  }
}
```



方法super.loadClass(name, resolve)实际上会回到了java.lang.ClassLoader#loadClass(java.lang.String, boolean)，遵循双亲委派机制进行查找类，而Bootstrap ClassLoader和Extension ClassLoader将会查找不到fat jar依赖的类，最终会来到Application ClassLoader，调用java.net.URLClassLoader#findClass

**如何真正的启动**

Springboot2和Springboot1的最大区别在于，Springboo1会新起一个线程，来执行相应的反射调用逻辑，而SpringBoot2则去掉了构建新的线程这一步。方法是org.springframework.boot.loader.Launcher#launch(java.lang.String[], java.lang.String, java.lang.ClassLoader)反射调用逻辑比较简单，这里就不再分析，比较关键的一点是，在调用main方法之前，将当前线程的上下文类加载器设置成LaunchedURLClassLoader



```
protected void launch(String[] args, String mainClass, ClassLoader classLoader)
   throws Exception {
  Thread.currentThread().setContextClassLoader(classLoader);
  createMainMethodRunner(mainClass, args, classLoader).run();
}
```

Demo:

```

public static void main(String[] args) throws ClassNotFoundException, MalformedURLException {
    JarFile.registerUrlProtocolHandler();
// 构造LaunchedURLClassLoader类加载器，这里使用了2个URL，分别对应jar包中依赖包spring-boot-loader和spring-boot，使用 "!/" 分开，需要org.springframework.boot.loader.jar.Handler处理器处理
    LaunchedURLClassLoader classLoader = new LaunchedURLClassLoader(
        new URL[] {
            new URL("jar:file:/E:/IdeaProjects/oneday-auth/oneday-auth-server/target/oneday-auth-server-1.0.0-SNAPSHOT.jar!/BOOT-INF/lib/spring-boot-loader-1.2.3.RELEASE.jar!/")
            , new URL("jar:file:/E:/IdeaProjects/oneday-auth/oneday-auth-server/target/oneday-auth-server-1.0.0-SNAPSHOT.jar!/BOOT-INF/lib/spring-boot-2.1.3.RELEASE.jar!/")
        },
        Application.class.getClassLoader());
// 加载类
// 这2个类都会在第二步本地查找中被找出(URLClassLoader的findClass方法)
    classLoader.loadClass("org.springframework.boot.loader.JarLauncher");
    classLoader.loadClass("org.springframework.boot.SpringApplication");
// 在第三步使用默认的加载顺序在ApplicationClassLoader中被找出
  classLoader.loadClass("org.springframework.boot.autoconfigure.web.DispatcherServletAutoConfiguration");
 
//    SpringApplication.run(Application.class, args);
  }
```



```

<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-loader -->
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-loader</artifactId>
  <version>2.1.3.RELEASE</version>
</dependency>
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-maven-plugin</artifactId>
  <version>2.1.3.RELEASE</version>
 
</dependency>
```



**总结**

对于源码分析，这次的较大收获则是不能一下子去追求弄懂源码中的每一步代码的逻辑，即便我知道该方法的作用。我们需要搞懂的是关键代码，以及涉及到的知识点。我从Maven的自定义插件开始进行追踪，巩固了对Maven的知识点，在这个过程中甚至了解到JDK对jar的读取是有提供对应的工具类。最后最重要的知识点则是自定义类加载器。

