https://blog.csdn.net/u010399009/article/details/78465907

是什么
为什么
怎么做


强制赋值
依赖链

asm

javassist



1. ServletContainerInitializer#onStartup 当容器启动时
   Spring Web 实现
   HandlesTypes 注解用于表示感兴趣的一些类

   tomcat 6.x 实现 servlet2.5
   tomcat 7.x 实现 servlet3.0

   tomcat 8.x 实现 servlet3.1

2. ServletContextListener#contextInitialized



ContextLoaderListener





Condition接口

ConditionalOnClass：当指定的某个类存在时，满足条件

可选的时候









## WebMvcConfigurer

1. WebMvcConfigurer就是我们来配置web请求过程中的一些组件。

   页面跳转addViewControllers
   自定义资源映射addResourceHandlers
   addInterceptors(InterceptorRegistry registry)
   configureContentNegotiation(ContentNegotiationConfigurer configurer)

2. java8接口可以用default，也可以用static方法。

3. 针对接口编程，模板方法。



figureContentNegotiation(ContentNegotiationConfigurer configurer)



> 1. invokeBeanFactoryPostProcessors： 
>    初始化BeanFactoryPostProcessor，只要是实现该接口的都会在此处初始化，MyBeanFactoryPostProcessor是实现了该接口的所以会在这里实例化并且会调用它的postProcessBeanFactory方法。
>
> 2. registerBeanPostProcessors(beanFactory)： 
>    初始化拦截Bean的BeanPostProcessors。只要是BeanPostProcessors的子类，在初始构造函数时，都会调用子类的前后方法 
>    MyBeanPostProcessor和MyInstantiationAwareBeanPostProcessor都实现了该接口，会在这里执行它的初始化方法。
>
> 3. finishBeanFactoryInitialization(beanFactory)： 
>    这里bean 如果不是抽象，不是懒加载，不是原型的就会在此处初始化。 
>    bean在这里执行getBean,此时会发生bean的初始化，和相应的依赖注入。 
>    上面的结果中我注释了，因为Spring容器中不止存在我们上面写的那些bean还存在其他的。
>
> 这里注入时： 
> 构造器–>自动注入–>PostConstrut–>InitializingBean–>xml中配置init方法



### ConfigurationClassPostProcessor

注解



BeanDefinitionRegistryPostProcessor，在spring生命周期中执行顺序为

1. 构造函数
2. BeanDefinitionRegistryPostProcessor.postProcessBeanDefinitionRegistry
3. BeanFactoryPostProcessor.postProcessBeanFactory







### tomcat加载web.xml的顺序是:

<!-- 这个配置文件在容器启动的时候 就加载 --> <load-on-startup>1</load-on-startup>

context-param ---> listener ---> filter ---> servlet

首先tomcat会生成一个程序应用级ServletContext,全局唯一，其中将context-param放在第一位主要是listener和filter会用到配置的初始化参数，
比如Spring配置的contextConfigLocation,在ContextLoaderLister加载时会从ServletContext的初始化参数中获取配置文件，进行bean的初始化操作.
上面还有一点，那就是servlet的加载，当load-on-startup大于等于0时，表示在tomcat容器启动时加载这个Servlet,否则，在第一次使用时才加载.

### 零配置实现

从servlet3.0开始，web支持no web.xml实现容器的初始化工作，是得于javax.servlet.ServletContainerInitializer对初始化工作的支持.

spring通过实现javax.servlet.ServletContainerInitializer，重写onStartUp方法,来提供无xml文件的spring容器初始化工作.下面是一个spring实现初始化的例子.

```java
import java.util.EnumSet;

import javax.servlet.DispatcherType;
import javax.servlet.ServletContext;
import javax.servlet.ServletException;

import org.springframework.web.WebApplicationInitializer;
import org.springframework.web.context.ContextLoaderListener;
import org.springframework.web.context.support.AnnotationConfigWebApplicationContext;
import org.springframework.web.filter.CharacterEncodingFilter;

import cn.oracle.action.AppConfig;

public class DefaultConfigration implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext container) throws ServletException {

        // 配置Spring提供的字符编码过滤器
        javax.servlet.FilterRegistration.Dynamic filter = container.addFilter("encoding",
                new CharacterEncodingFilter());
        // 配置过滤器的过滤路径
        filter.addMappingForUrlPatterns(EnumSet.of(DispatcherType.REQUEST), true, "/");

        // 基于注解配置的Spring容器上下文
        AnnotationConfigWebApplicationContext rootContext = new AnnotationConfigWebApplicationContext();
        // 注册Spring容器配置类
        rootContext.register(AppConfig.class);
        container.addListener(new ContextLoaderListener(rootContext));        
    }
}
```

具体的 AppConfig 相当于原来的 application-*.xml 等 spring 的配置文件处理.

注 ：@HandlesTypes is used to declare the class types that a ServletContainerInitializer can handle.





Spring validator
第一步：定义一个Validator
第二步：使用Validator

@Valid

@Autowired  自动注入


图的alt  鼠标悬停在图时的信息