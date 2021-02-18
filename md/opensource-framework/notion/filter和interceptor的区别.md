# SpringBoot2.x拦截器与过滤器的应用场景及配置



## 一、拦截器和过滤器有什么区别呢？

## 1、对拦截器和过滤器的理解：

（1）过滤器（Filter）：前端访问后台请求或者静态资源文件时，你只希望符合你要求的一些请求可以访问。定义这些要求的工具，就是过滤器。（理解：就是一堆请求过来时，过滤掉一些不合法的请求）

（2）拦截器（Interceptor）：在后台一个处理流程正在进行的时候，你希望干预它的进展，或者在它执行前后进行相关处理，甚至终止它进行，这是拦截器做的事情。（理解：就是执行一个方法时，在方法执行前后打印日志类似的操作）。一般，拦截器存在于在将请求发送到控制器之前和在将响应发送给客户端之前。例如，使用拦截器在将请求发送到控制器之前添加请求标头，并在将响应发送到客户端之前添加响应标头。

还可以这样理解：

（1）拦截器 ：是在面向切面编程的就是在你的service或者一个方法，前调用一个方法，或者在方法后调用一个方法比如动态代理就是拦截器的简单实现，在你调用方法前打印出字符串（或者做其它业务逻辑的操作），也可以在你调用方法后打印出字符串，甚至在你抛出异常的时候做业务逻辑的操作。

（2）过滤器：是在javaweb中，你传入的request、response提前过滤掉一些信息，或者提前设置一些参数，然后再传入servlet或者struts的action进行业务逻辑，比如过滤掉非法url（比如请求地址中含有非法字符），或者在传入servlet或者 struts的action前统一设置字符集，或者去除掉一些非法字符.。

## 2、拦截器和过滤器应用场景

拦截器应用场景：

1） 权限检查：如登录检测，进入处理器检测检测是否登录，如果没有直接返回到登录页面

2） 日志记录：记录请求信息的日志，以便进行信息监控、信息统计、计算PV（Page View）等。

3） 性能监控：有时候系统在某段时间莫名其妙的慢，可以通过拦截器在进入处理器之前记录开始时间，在处理完后记录结束时间，从而得到该请求的处理时间（如果有反向代理，如apache可以自动记录）

4） 通用行为：读取cookie得到用户信息并将用户对象放入请求，从而方便后续流程使用，还有如提取Locale、Theme信息等，只要是多个处理器都需要的即可使用拦截器实现5）

**过滤器应用场景：**

1）过滤敏感词汇（防止sql注入）

2）这是字符编码

3）URL级别的权限访问控制

4）压缩响应信息

## 3、具体区别

拦截器是面向切面 AOP( Aspect-Oriented Programming)的一种实现，底层通过**动态代理**模式完成。

区别：

（1）拦截器是基于java的反射机制的，而过滤器是基于函数回调。

（2）拦截器不依赖于servlet容器，而过滤器依赖于servlet容器。

（3）拦截器只能对action请求起作用，而过滤器则可以对几乎所有的请求起作用。

（4）拦截器可以访问action上下文、值栈里的对象，而过滤器不能。

（5）在action的生命周期中，拦截器可以多次被调用，而过滤器只能在容器初始化时被调用一次。

![img](https://pic2.zhimg.com/80/v2-aad7a627b99ea7d565989d7166c2b25d_720w.jpg)

> 两者的本质区别：从灵活性上说拦截器功能更强大些，Filter能做的事情，他都能做，而且可以在请求前，请求后执行，比较灵活。Filter主要是针对URL地址做一个编码的事情、过滤掉没用的参数、安全校验（比较泛的，比如登录不登录之类），太细的话，还是建议用interceptor。不过还是根据不同情况选择合适的。

![img](https://pic1.zhimg.com/80/v2-a6fe07431204a9c26edaa418880a846c_720w.jpg)

## 4、Spring boot2.x 版本下的拦截器配置

我这里使用的是springboot2.2.4的版本，众所周知springboot2.x多了很多新特性，刚好拦截器配置这里就做了些许改变，为了避免大家采坑，小编特意写下这篇文章。废话少说，先上代码：

1、自定义拦截器实现

问：Spring Boot怎么配置拦截器？

答：配置一个拦截器需要两步完成。

> 1）自定义拦截器，实现HandlerInterceptor这个接口。这个接口包括三个方法，preHandle是请求执行前执行的，postHandler是请求结束执行的，但只有preHandle方法返回true的时候才会执行，afterCompletion是视图渲染完成后才执行，同样需要preHandle返回true，该方法通常用于清理资源等工作。
> 2） 注册拦截器。
> 作用是确定拦截器和拦截的URL。需要继承WebMvcConfigurationSupport并重写addInterceptor方法，在spring boot2.x中WebMvcConfigureAdapter已经过时了！！

代码MyInterceptor.java如下：

```text
package com.test.config;

import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

public class MyInterceptor implements HandlerInterceptor {

    //方法执行前
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        System.out.println("zhixing preHandle");
        return true;
    }

    //方法执行后
    @Override
    public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) {
        System.out.println("zhixing postHandle");
    }

    //页面渲染前
    @Override
    public void afterCompletion(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        System.out.println("zhixing afterCompletion");
    }
}
```

在WebMvcConfigurationSupport的继承类中，重写addInterceptor方法：

代码如下：

```text
package com.test.config;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.CorsRegistry;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.ResourceHandlerRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurationSupport;
/**
 * @author houpeibin
 * @Date: 2020/2/10 17:21
 */
@Configuration
public class WebMvcConfigurer extends WebMvcConfigurationSupport {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        //'/*'匹配一个请求
        registry.addInterceptor(new MyInterceptor()).addPathPatterns("/api/*/**");
        WebMvcConfigurer.super.addInterceptors(registry);
    }
}
```

TestController.java类代码如下：

```text
package com.test.controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import javax.servlet.http.HttpServletResponse;

@RestController
@RequestMapping("/api")
public class TestController {
    @RequestMapping("/index")
    public String index(Model model, HttpServletResponse response) {
        System.out.println("zhixing TestController");
        return "hello spring boot index 123";
    }
}
```

入口类：

```text
package com.test;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
@SpringBootApplication
public class Demo2Application {
	public static void main(String[] args) {
		SpringApplication.run(Demo2Application.class, args);
	}
}
```

运行入口类，访问：[http://localhost:8080/api/index](https://link.zhihu.com/?target=http%3A//localhost%3A8080/api/index) 地址打印日志如下：

![img](https://pic4.zhimg.com/80/v2-05abe509b274fae6ec869a9ed985023b_720w.jpg)

这样一个拦截器就写完了。

**==坑坑坑==：**

拦截器不生效常见问题：

1）是否有加@Configuration

2）拦截路径是否有问题 ** 和 *

3）拦截器最后路径一定要 “/**”， 如果是目录的话则是 /*/

总结一下:创建拦截器需要两步：

> 1、自定义拦截器
> 2、注册拦截器

## 5、Spring boot2.x 版本下的过滤器配置

在上面的实例基础上，创建LoginFilter.java类

```text
package com.test.config;

import javax.servlet.*;
import javax.servlet.annotation.WebFilter;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import java.io.IOException;

@WebFilter(urlPatterns = "/api/*", filterName = "loginFilter")
public class LoginFilter  implements Filter {
    /**
     * 容器加载的时候调用
     */
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        System.out.println("init loginFilter");
    }
    /**
     * 请求被拦截的时候进行调用
     */
    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        System.out.println("doFilter loginFilter");

        HttpServletRequest req = (HttpServletRequest) servletRequest;
        HttpServletResponse resp = (HttpServletResponse) servletResponse;
        String username = req.getParameter("username");

        if ("xdclass".equals(username)) {
            filterChain.doFilter(servletRequest,servletResponse);
        } else {
            resp.sendRedirect("/index.html");
            return;
        }
    }
    /**
     * 容器被销毁的时候被调用
     */
    @Override
    public void destroy() {
        System.out.println("destroy loginFilter");
    }
}
```

**此段代码利用@WebFilter注解定义了一个过滤器。这时候还不能生效，必须在Application启动类添加@ServletComponentScan注解才行，下面是加入后的启动类（也叫入口类）**

```text
package com.test;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.web.servlet.ServletComponentScan;

@SpringBootApplication
@ServletComponentScan   //使用过滤器时使用
public class Demo2Application {
	public static void main(String[] args) {
		SpringApplication.run(Demo2Application.class, args);
	}
}
```

此时访问：[http://localhost:8080/api/index](https://link.zhihu.com/?target=http%3A//localhost%3A8080/api/index) 会被过滤器过滤到没有登录，跳转到：[http://localhost:8080/index.html](https://link.zhihu.com/?target=http%3A//localhost%3A8080/index.html) 地址；

如果访问：[http://localhost:8080/api/index?username=xdclass](https://link.zhihu.com/?target=http%3A//localhost%3A8080/api/index%3Fusername%3Dxdclass) 地址时，才能通过登录验证，成功访问到对应的页面。

总结一下，创建过滤器需要两步：

> 第一步：利用@WebFilter创建Filter过滤器类
> 第二步;Application启动类添加@ServletComponentScan注解

就成功了。