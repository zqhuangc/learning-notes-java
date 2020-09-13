# [Spring AOP 使用介绍](https://mp.weixin.qq.com/s/KOV_lWOTPYMi8-2kFZCS9Q)

由于 Spring 强大的向后兼容性，实际代码中往往会出现很多配置混杂的情况，而且居然还能工作，本文希望帮助大家理清楚这些知识。

目录：

## AOP, AspectJ, Spring AOP

我们先来把它们的概念和关系说说清楚。

AOP 要实现的是在我们原来写的代码的基础上，进行一定的包装，如在方法执行前、方法返回后、方法抛出异常后等地方进行一定的拦截处理或者叫增强处理。

AOP 的实现并不是因为 Java 提供了什么神奇的钩子，可以把方法的几个生命周期告诉我们，而是我们要实现一个代理，实际运行的实例其实是生成的代理类的实例。

作为 Java 开发者，我们都很熟悉 AspectJ 这个词，甚至于我们提到 AOP 的时候，想到的往往就是 AspectJ，即使你可能不太懂它是怎么工作的。这里，我们把 AspectJ 和 Spring AOP 做个简单的对比：

Spring AOP：

- 它基于动态代理来实现。默认地，如果使用接口的，用 JDK 提供的动态代理实现，如果没有接口，使用 CGLIB 实现。大家一定要明白背后的意思，包括什么时候会不用 JDK 提供的动态代理，而用 CGLIB 实现。
- Spring 3.2 以后，spring-core 直接就把 CGLIB 和 ASM 的源码包括进来了，这也是为什么我们不需要显式引入这两个依赖
- Spring 的 IOC 容器和 AOP 都很重要，Spring AOP 需要依赖于 IOC 容器来管理。
- 如果你是 web 开发者，有些时候，你可能需要的是一个 Filter 或一个 Interceptor，而不一定是 AOP。
- Spring AOP 只能作用于 Spring 容器中的 Bean，它是使用纯粹的 Java 代码实现的，只能作用于 bean 的方法。
- Spring 提供了 AspectJ 的支持，后面我们会单独介绍怎么使用，一般来说我们用纯的 Spring AOP 就够了。
- 很多人会对比 Spring AOP 和 AspectJ 的性能，Spring AOP 是基于代理实现的，在容器启动的时候需要生成代理实例，在方法调用上也会增加栈的深度，使得 Spring AOP 的性能不如 AspectJ 那么好。

AspectJ：

- AspectJ 出身也是名门，来自于 Eclipse 基金会，link：https://www.eclipse.org/aspectj

- 属于静态织入，它是通过修改代码来实现的，它的织入时机可以是：

- - Compile-time weaving：编译期织入，如类 A 使用 AspectJ 添加了一个属性，类 B 引用了它，这个场景就需要编译期的时候就进行织入，否则没法编译类 B。
  - Post-compile weaving：也就是已经生成了 .class 文件，或已经打成 jar 包了，这种情况我们需要增强处理的话，就要用到编译后织入。
  - Load-time weaving：指的是在加载类的时候进行织入，要实现这个时期的织入，有几种常见的方法。1、自定义类加载器来干这个，这个应该是最容易想到的办法，在被织入类加载到 JVM 前去对它进行加载，这样就可以在加载的时候定义行为了。2、在 JVM 启动的时候指定 AspectJ 提供的 agent：`-javaagent:xxx/xxx/aspectjweaver.jar`。

- AspectJ 能干很多 Spring AOP 干不了的事情，它是 AOP 编程的完全解决方案。Spring AOP 致力于解决的是企业级开发中最普遍的 AOP 需求（方法织入），而不是力求成为一个像 AspectJ 一样的 AOP 编程完全解决方案。

- 因为 AspectJ 在实际代码运行前完成了织入，所以大家会说它生成的类是没有额外运行时开销的。

- 很快我会专门写一篇文章介绍 AspectJ 的使用，以及怎么在 Spring 应用中使用 AspectJ。



## AOP 术语解释

在这里，不准备解释那么多 AOP 编程中的术语了，我们碰到一个说一个吧。

Advice、Advisor、Pointcut、Aspect、Joinpoint 等等。

## Spring AOP

首先要说明的是，这里介绍的 Spring AOP 是纯的 Spring 代码，和 AspectJ 没什么关系，但是 Spring 延用了 AspectJ 中的概念，包括使用了 AspectJ 提供的 jar 包中的注解，但是不依赖于其实现功能。

> 后面介绍的如 @Aspect、@Pointcut、@Before、@After 等注解都是来自于 AspectJ，但是功能的实现是纯 Spring AOP 自己实现的。

下面我们来介绍 Spring AOP 的使用方法，先从最简单的配置方式开始说起，这样读者想看源码也会比较容易。

目前 Spring AOP 一共有三种配置方式，Spring 做到了很好地向下兼容，所以大家可以放心使用。

- Spring 1.2 基于接口的配置：最早的 Spring AOP 是完全基于几个接口的，想看源码的同学可以从这里起步。
- Spring 2.0 schema-based 配置：Spring 2.0 以后使用 XML 的方式来配置，使用 命名空间 `<aop />`
- Spring 2.0 @AspectJ 配置：使用注解的方式来配置，这种方式感觉是最方便的，还有，这里虽然叫做 `@AspectJ`，但是这个和 AspectJ 其实没啥关系。

### Spring 1.2 中的配置

这节我们将介绍 Spring 1.2 中的配置，这是最古老的配置，但是由于 Spring 提供了很好的向后兼容，以及很多人根本不知道什么配置是什么版本的，以及是否有更新更好的配置方法替代，所以还是会有很多代码是采用这种古老的配置方式的，这里说的古老并没有贬义的意思。

下面用一个简单的例子来演示怎么使用 Spring 1.2 的配置方式。

首先，我们先定义两个接口 `UserService` 和 `OrderService`，以及它们的实现类 `UserServiceImpl` 和 `OrderServiceImpl`：

![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE2kefum7Lq2phKfdhp20P912OuONIia1UVkoRaibb1JZCYlrNG3e57g8Z5ZNQrib2BSqcpjiargNQxicJA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE2kefum7Lq2phKfdhp20P91qw9rvnTQ28lmhXeLWrTGOYguXhk9aC0ib4tIOsNOmib17GvbVoNzK9Dw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

接下来，我们定义两个 advice，分别用于拦截方法执行前和方法返回后：

> advice 是我们接触的第一个概念，记住它是干什么用的。

![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE2kefum7Lq2phKfdhp20P91a2wZdP92DgRVqJsuD0pXOvPd4DqC1As0rwRibKhiaEX7yZHqiadUK51lQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

上面的两个 Advice 分别用于方法调用前输出参数和方法调用后输出结果。

现在可以开始配置了，我们配置一个名为 spring_1_2.xml 的文件：

![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE2kefum7Lq2phKfdhp20P91BicT3yaOhPjxqUfPZMXq01ic8wyP6VWUE58Hrf84VIEib0uRTGDjRVic1Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

接下来，我们跑起来看看：

![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE2kefum7Lq2phKfdhp20P91icWhXYTQTSA3wqb4G2H2BvK3PEYHE8y9YTMHC2k8qjmXm3w1DJ8YNmA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

查看输出结果：

```
准备执行方法: createUser, 参数列表：[Tom, Cruise, 55]
方法返回：User{firstName='Tom', lastName='Cruise', age=55, address='null'}
准备执行方法: queryUser, 参数列表：[]
方法返回：User{firstName='Tom', lastName='Cruise', age=55, address='null'}
```

从结果可以看到，对 UserService 中的两个方法都做了前、后拦截。这个例子理解起来应该非常简单，就是一个代理实现。

> 代理模式需要一个接口、一个具体实现类，然后就是定义一个代理类，用来包装实现类，添加自定义逻辑，在使用的时候，需要用代理类来生成实例。

此中方法有个致命的问题，如果我们需要拦截 OrderService 中的方法，那么我们还需要定义一个 OrderService 的代理。如果还要拦截 PostService，得定义一个 PostService 的代理......

而且，我们看到，我们的拦截器的粒度只控制到了类级别，类中所有的方法都进行了拦截。接下来，我们看看怎么样只拦截特定的方法。

在上面的配置中，配置拦截器的时候，interceptorNames 除了指定为 Advice，是还可以指定为 Interceptor 和 Advisor 的。

这里我们来理解 Advisor 的概念，它也比较简单，它内部需要指定一个 Advice，Advisor 决定该拦截哪些方法，拦截后需要完成的工作还是内部的 Advice 来做。

它有好几个实现类，这里我们使用实现类 NameMatchMethodPointcutAdvisor 来演示，从名字上就可以看出来，它需要我们给它提供方法名字，这样符合该配置的方法才会做拦截。

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

> 我们可以看到，userServiceProxy 这个 bean 配置了一个 advisor，advisor 内部有一个 advice。advisor 负责匹配方法，内部的 advice 负责实现方法包装。
>
> 注意，这里的 mappedNames 配置是可以指定多个的，用逗号分隔，可以是不同类中的方法。相比直接指定 advice，advisor 实现了更细粒度的控制，因为在这里配置 advice 的话，所有方法都会被拦截。

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

输出结果如下，只有 createUser 方法被拦截：

```
准备执行方法: createUser, 参数列表：[Tom, Cruise, 55]
```

到这里，我们已经了解了 Advice 和 Advisor 了，前面也说了还可以配置 Interceptor。

对于 Java 开发者来说，对 Interceptor 这个概念肯定都很熟悉了，这里就不做演示了，贴一下实现代码：

```
public class DebugInterceptor implements MethodInterceptor {

    public Object invoke(MethodInvocation invocation) throws Throwable {
        System.out.println("Before: invocation=[" + invocation + "]");
        // 执行 真实实现类 的方法
        Object rval = invocation.proceed();
        System.out.println("Invocation returned");
        return rval;
    }
}
```

上面，我们介绍完了 Advice、Advisor、Interceptor 三个概念，相信大家应该很容易就看懂它们了。

它们有个共同的问题，那就是我们得为每个 bean 都配置一个代理，之后获取 bean 的时候需要获取这个代理类的 bean 实例（如 `(UserService) context.getBean("userServiceProxy")`），这显然非常不方便，不利于我们之后要使用的自动根据类型注入。下面介绍 autoproxy 的解决方案。

autoproxy：从名字我们也可以看出来，它是实现自动代理，也就是说当 Spring 发现一个 bean 需要被切面织入的时候，Spring 会自动生成这个 bean 的一个代理来拦截方法的执行，确保定义的切面能被执行。

这里强调自动，也就是说 Spring 会自动做这件事，而不用像前面介绍的，我们需要显式地指定代理类的 bean。

我们去掉原来的 ProxyFactoryBean 的配置，改为使用 BeanNameAutoProxyCreator 来配置：

![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE2kefum7Lq2phKfdhp20P91p8rZeicicWP3ZnO20aa52MTicCwWI0joSPjzMTVC7fPDkRpXxILZTM5GQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

配置很简单，beanNames 中可以使用正则来匹配 bean 的名字。这样配置出来以后，userServiceBeforeAdvice 和 userServiceAfterAdvice 这两个拦截器就不仅仅可以作用于 UserServiceImpl 了，也可以作用于 OrderServiceImpl、PostServiceImpl、ArticleServiceImpl......等等，也就是说不再是配置某个 bean 的代理了。

> 注意，这里的 InterceptorNames 和前面一样，也是可以配置成 Advisor 和 Interceptor 的。

然后我们修改下使用的地方：

![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE2kefum7Lq2phKfdhp20P91Xb0AkibWKtXic4LTuwYMKbCq5g0icSjPibRz4kSsURaDicG8UQTyibK3Mmqg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

发现没有，我们在使用的时候，完全不需要关心代理了，直接使用原来的类型就可以了，这是非常方便的。

输出结果就是 OrderService 和 UserService 中的每个方法都得到了拦截：

```
准备执行方法: createUser, 参数列表：[Tom, Cruise, 55]
方法返回：User{firstName='Tom', lastName='Cruise', age=55, address='null'}
准备执行方法: queryUser, 参数列表：[]
方法返回：User{firstName='Tom', lastName='Cruise', age=55, address='null'}
准备执行方法: createOrder, 参数列表：[Leo, 随便买点什么]
方法返回：Order{username='Leo', product='随便买点什么'}
准备执行方法: queryOrder, 参数列表：[Leo]
方法返回：Order{username='Leo', product='随便买点什么'}
```

到这里，是不是发现 BeanNameAutoProxyCreator 非常好用，它需要指定被拦截类名的模式(如 *ServiceImpl)，它可以配置多次，这样就可以用来匹配不同模式的类了。

另外，在 BeanNameAutoProxyCreator 同一个包中，还有一个非常有用的类 DefaultAdvisorAutoProxyCreator，比上面的 BeanNameAutoProxyCreator 还要方便。

之前我们说过，advisor 内部包装了 advice，advisor 负责决定拦截哪些方法，内部 advice 定义拦截后的逻辑。所以，仔细想想其实就是只要让我们的 advisor 全局生效就能实现我们需要的自定义拦截功能、拦截后的逻辑处理。

> BeanNameAutoProxyCreator 是自己匹配方法，然后交由内部配置 advice 来拦截处理；
>
> 而 DefaultAdvisorAutoProxyCreator 是让 ioc 容器中的所有 advisor 来匹配方法，advisor 内部都是有 advice 的，让它们内部的 advice 来执行拦截处理。

1、我们需要再回头看下 Advisor 的配置，上面我们用了 NameMatchMethodPointcutAdvisor 这个类：

```
<bean id="logCreateAdvisor" class="org.springframework.aop.support.NameMatchMethodPointcutAdvisor">
    <property name="advice" ref="logArgsAdvice" />
    <property name="mappedNames" value="createUser,createOrder" />
</bean>
```

其实 Advisor 还有一个更加灵活的实现类 RegexpMethodPointcutAdvisor，它能实现正则匹配，如：

```
<bean id="logArgsAdvisor" class="org.springframework.aop.support.RegexpMethodPointcutAdvisor">
    <property name="advice" ref="logArgsAdvice" />
    <property name="pattern" value="com.javadoop.*.service.*.create.*" />
</bean>
```

也就是说，我们能通过配置 Advisor，精确定位到需要被拦截的方法，然后使用内部的 Advice 执行逻辑处理。

2、之后，我们需要配置 DefaultAdvisorAutoProxyCreator，它的配置非常简单，直接使用下面这段配置就可以了，它就会使得所有的 Advisor 自动生效，无须其他配置。

```
<bean class="org.springframework.aop.framework.autoproxy.DefaultAdvisorAutoProxyCreator" />
```

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

然后我们运行一下：

![img](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

输出：

```
准备执行方法: createUser, 参数列表：[Tom, Cruise, 55]
方法返回：User{firstName='Tom', lastName='Cruise', age=55, address='null'}
准备执行方法: createOrder, 参数列表：[Leo, 随便买点什么]
方法返回：Order{username='Leo', product='随便买点什么'}
```

从结果可以看出，create* 方法使用了 logArgsAdvisor 进行传参输出，query* 方法使用了 logResultAdvisor 进行了返回结果输出。

到这里，Spring 1.2 的配置就要介绍完了。本文不会介绍得面面俱到，主要是关注最核心的配置，如果读者感兴趣，要学会自己去摸索，比如这里的 Advisor 就不只有我这里介绍的 NameMatchMethodPointcutAdvisor 和 RegexpMethodPointcutAdvisor，AutoProxyCreator 也不仅仅是 BeanNameAutoProxyCreator 和 DefaultAdvisorAutoProxyCreator。

> 读到这里，我想对于很多人来说，就知道怎么去阅读 Spring AOP 源码了。

### Spring 2.0 @AspectJ 配置

Spring 2.0 以后，引入了 @AspectJ 和 Schema-based 的两种配置方式，我们先来介绍 @AspectJ 的配置方式，之后我们再来看使用 xml 的配置方式。

注意了，@AspectJ 和 AspectJ 没多大关系，并不是说基于 AspectJ 实现的，而仅仅是使用了 AspectJ 中的概念，包括使用的注解也是直接来自于 AspectJ 的包。

首先，我们需要依赖 `aspectjweaver.jar` 这个包，这个包来自于 AspectJ：

```
<dependency>
    <groupId>org.aspectj</groupId>
    <artifactId>aspectjweaver</artifactId>
    <version>1.8.11</version>
</dependency>
```

如果是使用 Spring Boot 的话，添加以下依赖即可：

```
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

在 @AspectJ 的配置方式中，之所以要引入 aspectjweaver 并不是因为我们需要使用 AspectJ 的处理功能，而是因为 Spring 使用了 AspectJ 提供的一些注解，实际上还是纯的 Spring AOP 代码。

说了这么多，明确一点，@AspectJ 采用注解的方式来配置使用 Spring AOP。

首先，我们需要开启 @AspectJ 的注解配置方式，有两种方式：

1、在 xml 中配置：

```
<aop:aspectj-autoproxy/>
```

1. 使用 @EnableAspectJAutoProxy

```
@Configuration
@EnableAspectJAutoProxy
public class AppConfig {

}
```

一旦开启了上面的配置，那么所有使用 @Aspect 注解的 bean 都会被 Spring 当做用来实现 AOP 的配置类，我们称之为一个 Aspect。

> 注意了，@Aspect 注解要作用在 bean 上面，不管是使用 @Component 等注解方式，还是在 xml 中配置 bean，首先它需要是一个 bean。

比如下面这个 bean，它的类名上使用了 @Aspect，它就会被当做 Spring AOP 的配置。

```
<bean id="myAspect" class="org.xyz.NotVeryUsefulAspect">
    <!-- configure properties of aspect here as normal -->
</bean>
package org.xyz;
import org.aspectj.lang.annotation.Aspect;

@Aspect
public class NotVeryUsefulAspect {

}
```

接下来，我们需要关心的是 @Aspect 注解的 bean 中，我们需要配置哪些内容。

首先，我们需要配置 Pointcut，Pointcut 在大部分地方被翻译成切点，用于定义哪些方法需要被增强或者说需要被拦截，有点类似于之前介绍的 Advisor 的方法匹配。

Spring AOP 只支持 bean 中的方法（不像 AspectJ 那么强大），所以我们可以认为 Pointcut 就是用来匹配 Spring 容器中的所有 bean 的方法的。

```
@Pointcut("execution(* transfer(..))")// the pointcut expression
private void anyOldTransfer() {}// the pointcut signature
```

我们看到，@Pointcut 中使用了 execution 来正则匹配方法签名，这也是最常用的，除了 execution，我们再看看其他的几个比较常用的匹配方式：

- within：指定所在类或所在包下面的方法（Spring AOP 独有）

  > 如 @Pointcut("within(com.javadoop.springaoplearning.service..*)")

- @annotation：方法上具有特定的注解，如 @Subscribe 用于订阅特定的事件。

  > 如 @Pointcut("execution(* *.*(..)) && @annotation(com.javadoop.annotation.Subscribe)")

- bean(idOrNameOfBean)：匹配 bean 的名字（Spring AOP 独有）

  > 如 @Pointcut("bean(*Service)")

Tips：上面匹配中，通常 "." 代表一个包名，".." 代表包及其子包，方法参数任意匹配使用两个点 ".."。

对于 web 开发者，Spring 有个很好的建议，就是定义一个 SystemArchitecture：

```
@Aspect
public class SystemArchitecture {

    // web 层
    @Pointcut("within(com.javadoop.web..*)")
    public void inWebLayer() {}

    // service 层
    @Pointcut("within(com.javadoop.service..*)")
    public void inServiceLayer() {}

    // dao 层
    @Pointcut("within(com.javadoop.dao..*)")
    public void inDataAccessLayer() {}

    // service 实现，注意这里指的是方法实现，其实通常也可以使用 bean(*ServiceImpl)
    @Pointcut("execution(* com.javadoop..service.*.*(..))")
    public void businessService() {}

    // dao 实现
    @Pointcut("execution(* com.javadoop.dao.*.*(..))")
    public void dataAccessOperation() {}
}
```

上面这个 SystemArchitecture 很好理解，该 Aspect 定义了一堆的 Pointcut，随后在任何需要 Pointcut 的地方都可以直接引用（如 xml 中的 pointcut-ref=""）。

配置 pointcut 就是配置我们需要拦截哪些方法，接下来，我们要配置需要对这些被拦截的方法做什么，也就是前面介绍的 Advice。

接下来，我们要配置 Advice。

下面这块代码示例了各种常用的情况：

> 注意，实际写代码的时候，不要把所有的切面都揉在一个 class 中。

```
@Aspect
public class AdviceExample {

    // 这里会用到我们前面说的 SystemArchitecture
    // 下面方法就是写拦截 "dao层实现"
    @Before("com.javadoop.aop.SystemArchitecture.dataAccessOperation()")
    public void doAccessCheck() {
        // ... 实现代码
    }

    // 当然，我们也可以直接"内联"Pointcut，直接在这里定义 Pointcut
    // 把 Advice 和 Pointcut 合在一起了，但是这两个概念我们还是要区分清楚的
    @Before("execution(* com.javadoop.dao.*.*(..))")
    public void doAccessCheck() {
        // ... 实现代码
    }

    @AfterReturning("com.javadoop.aop.SystemArchitecture.dataAccessOperation()")
    public void doAccessCheck() {
        // ...
    }

    @AfterReturning(
        pointcut="com.javadoop.aop.SystemArchitecture.dataAccessOperation()",
        returning="retVal")
    public void doAccessCheck(Object retVal) {
        // 这样，进来这个方法的处理时候，retVal 就是相应方法的返回值，是不是非常方便
        //  ... 实现代码
    }

    // 异常返回
    @AfterThrowing("com.javadoop.aop.SystemArchitecture.dataAccessOperation()")
    public void doRecoveryActions() {
        // ... 实现代码
    }

    @AfterThrowing(
        pointcut="com.javadoop.aop.SystemArchitecture.dataAccessOperation()",
        throwing="ex")
    public void doRecoveryActions(DataAccessException ex) {
        // ... 实现代码
    }

    // 注意理解它和 @AfterReturning 之间的区别，这里会拦截正常返回和异常的情况
    @After("com.javadoop.aop.SystemArchitecture.dataAccessOperation()")
    public void doReleaseLock() {
        // 通常就像 finally 块一样使用，用来释放资源。
        // 无论正常返回还是异常退出，都会被拦截到
    }

    // 感觉这个很有用吧，既能做 @Before 的事情，也可以做 @AfterReturning 的事情
    @Around("com.javadoop.aop.SystemArchitecture.businessService()")
    public Object doBasicProfiling(ProceedingJoinPoint pjp) throws Throwable {
        // start stopwatch
        Object retVal = pjp.proceed();
        // stop stopwatch
        return retVal;
    }

}
```

细心的读者可能发现了有些 Advice 缺少方法传参，如在 @Before 场景中参数往往是非常有用的，比如我们要用日志记录下来被拦截方法的入参情况。

Spring 提供了非常简单的获取入参的方法，使用 org.aspectj.lang.JoinPoint 作为 Advice 的第一个参数即可，如：

```
@Before("com.javadoop.springaoplearning.aop_spring_2_aspectj.SystemArchitecture.businessService()")
public void logArgs(JoinPoint joinPoint) {
    System.out.println("方法执行前，打印入参：" + Arrays.toString(joinPoint.getArgs()));
}
```

> 注意：第一，必须放置在第一个参数上；第二，如果是 @Around，我们通常会使用其子类 ProceedingJoinPoint，因为它有 procceed()/procceed(args[]) 方法。

到这里，我们介绍完了 @AspectJ 配置方式中的 Pointcut 和 Advice 的配置。对于开发者来说，其实最重要的就是这两个了，定义 Pointcut 和使用合适的 Advice 在各个 Pointcut 上。

下面，我们用这一节介绍的 @AspectJ 来实现上一节实现的记录方法传参和记录方法返回值。

![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE2kefum7Lq2phKfdhp20P91nfn7lN0cczg7fflsOcfp3jWpruXGNJp9t5MYniblIHPiaQpbCkHHpjDg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

xml 的配置非常简单：

![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE2kefum7Lq2phKfdhp20P91R4ls6T9EIIanrtDCibrzVVrQgBDmyxF2A7nFoPx46jhL5ZAXfS6dvYA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

> 这里是示例，所以 bean 的配置还是使用了 xml 的配置方式。

测试一下：

![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE2kefum7Lq2phKfdhp20P91KbibFLB6cwzovfvMUJMzULee7yP2BSoicQicPumlsW8WPoRHvmJagqeCQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

输出结果：

```
方法执行前，打印入参：[Tom, Cruise, 55]
User{firstName='Tom', lastName='Cruise', age=55, address='null'}
方法执行前，打印入参：[]
User{firstName='Tom', lastName='Cruise', age=55, address='null'}
```

JoinPoint 除了 getArgs() 外还有一些有用的方法，大家可以进去稍微看一眼。

最后提一点，@Aspect 中的配置不会作用于使用 @Aspect 注解的 bean。

### Spring 2.0 schema-based 配置

本节将介绍的是 Spring 2.0 以后提供的基于 `<aop />` 命名空间的 XML 配置。这里说的 schema-based 就是指基于 `aop` 这个 schema。

> 介绍 IOC 的时候也介绍过 Spring 是怎么解析各个命名空间的（各种 *NamespaceHandler），解析 `<aop />` 的源码在 org.springframework.aop.config.AopNamespaceHandler 中。

有了前面的 @AspectJ 的配置方式的知识，理解 xml 方式的配置非常简单，所以我们就可以废话少一点了。

这里先介绍配置 Aspect，便于后续理解：

```
<aop:config>
    <aop:aspect id="myAspect" ref="aBean">
        ...
    </aop:aspect>
</aop:config>

<bean id="aBean" class="...">
    ...
</bean>
```

> 所有的配置都在 `<aop:config>` 下面。
>
> `<aop:aspect >` 中需要指定一个 bean，和前面介绍的 LogArgsAspect 和 LogResultAspect 一样，我们知道该 bean 中我们需要写处理代码。
>
> 然后，我们写好 Aspect 代码后，将其“织入”到合适的 Pointcut 中，这就是面向切面。

然后，我们需要配置 Pointcut，非常简单，如下：

```
<aop:config>

    <aop:pointcut id="businessService"
        expression="execution(* com.javadoop.springaoplearning.service.*.*(..))"/>

    <!--也可以像下面这样-->
    <aop:pointcut id="businessService2"
        expression="com.javadoop.SystemArchitecture.businessService()"/>

</aop:config>
```

> 将 `<aop:pointcut>` 作为 `<aop:config>` 的直接子元素，将作为全局 Pointcut。

我们也可以在 `<aop:aspect />`内部配置 Pointcut，这样该 Pointcut 仅用于该 Aspect：

```
<aop:config>
    <aop:aspect ref="logArgsAspect">
        <aop:pointcut id="internalPointcut"
                expression="com.javadoop.SystemArchitecture.businessService()" />
    </aop:aspect>
</aop:config>
```

接下来，我们应该配置 Advice 了，为了避免废话过多，我们直接上实例吧，非常好理解，将上一节用 @AspectJ 方式配置的搬过来：

![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE2kefum7Lq2phKfdhp20P91mz9u6pafY8p30CzN2CGx9qj8Qjf7gjMSicriaKtJibrIMia9JX0D4b19Jg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

上面的例子中，我们配置了两个 LogArgsAspect 和一个 LogResultAspect。

其实基于 XML 的配置也是非常灵活的，这里没办法给大家演示各种搭配，大家抓住基本的 Pointcut、Advice 和 Aspect 这几个概念，就很容易配置了。

## 小结

到这里，本文介绍了 Spring AOP 的三种配置方式，我们要知道的是，到目前为止，我们使用的都是 Spring AOP，和 AspectJ 没什么关系。

下一篇文章，将会介绍 AspectJ 的使用方式，以及怎样在 Spring 应用中使用 AspectJ。之后差不多就可以出 Spring AOP 源码分析了。

## 附录

本文使用的测试源码已上传到 Github: hongjiev/spring-aop-learning。

建议读者 clone 下来以后，通过命令行进行测试，而不是依赖于 IDE，因为 IDE 太"智能"了：

1. mvn clean package

2. java -jar target/spring-aop-learning-1.0-jar-with-dependencies.jar

   > pom.xml 中配置了 assembly 插件，打包的时候会将所有 jar 包依赖打到一起。

3. 修改 Application.java 中的代码，或者其他代码，然后重复 1 和 2

# [Spring AOP 源码解析](https://mp.weixin.qq.com/s/ICnowr49gV9W63Cum7qNeQ)

之前写过 IOC 的源码分析，那篇文章真的有点长，看完需要点耐心。很多读者希望能写一写 Spring AOP 的源码分析文章，这样读者看完 IOC + AOP 也就对 Spring 会有比较深的理解了。今天终于成文了，可能很多读者早就不再等待了，不过主要为了后来者吧。

本文不会像 IOC 源码分析那篇文章一样，很具体地分析每一行 Spring AOP 的源码，目标读者是已经知道 Spring IOC 源码是怎么回事的读者，因为 Spring AOP 终归是依赖于 IOC 容器来管理的。

阅读建议：1、先搞懂 IOC 容器的源码[【Spring源码】Spring IOC 容器源码分析（一）](http://mp.weixin.qq.com/s?__biz=MzUxNDA1NDI3OA==&mid=2247485878&idx=1&sn=1dbc1545a91dc709998f9f9c013cf412&chksm=f94a885fce3d0149b295cf10b729cfe0de6d00a04a5ce7df642da4fa68f1b5bea1a280aff009&scene=21#wechat_redirect)，AOP 依赖于 IOC 容器来管理。2、仔细看完[Spring AOP 使用介绍，从前世到今生](http://mp.weixin.qq.com/s?__biz=MzUxNDA1NDI3OA==&mid=2247485890&idx=1&sn=9b76f4fe52452646bea35343e1b1a0f9&chksm=f94a882bce3d013d25594409f3076be664e077b354bf8eebe6892691ba054baf2515b07c05fe&scene=21#wechat_redirect)这篇文章，先搞懂各种使用方式，你才能"猜到"应该怎么实现。

Spring AOP 的源码并不简单，因为它多，所以阅读源码最好就是找到一个分支，追踪下去。本文定位为走马观花，看个大概，不具体到每一个细节。

目录：

## 前言

这一节，我们先来"猜猜" Spring 是怎么实现 AOP 的。

在 Spring 的容器中，我们面向的对象是一个个的 bean 实例，bean 是什么？我们可以简单理解为是 BeanDefinition 的实例，Spring 会根据 BeanDefinition 中的信息为我们生产合适的 bean 实例出来。

当我们需要使用 bean 的时候，通过 IOC 容器的 getBean(…) 方法从容器中获取 bean 实例，只不过大部分的场景下，我们都用了依赖注入，所以很少手动调用 getBean(...) 方法。

Spring AOP 的原理很简单，就是动态代理，它和 AspectJ 不一样，AspectJ 是直接修改掉你的字节码。

代理模式很简单，接口 + 真实实现类 + 代理类，其中 真实实现类 和 代理类 都要实现接口，实例化的时候要使用代理类。所以，Spring AOP 需要做的是生成这么一个代理类，然后替换掉真实实现类来对外提供服务。

替换的过程怎么理解呢？在 Spring IOC 容器中非常容易实现，就是在 getBean(…) 的时候返回的实际上是代理类的实例，而这个代理类我们自己没写代码，它是 Spring 采用 JDK Proxy 或 CGLIB 动态生成的。

> getBean(…) 方法用于查找或实例化容器中的 bean，这也是为什么 Spring AOP 只能作用于 Spring 容器中的 bean 的原因，对于不是使用 IOC 容器管理的对象，Spring AOP 是无能为力的。

## 本文使用的调试代码

阅读源码很好用的一个方法就是跑代码来调试，因为自己一行一行地看的话，比较枯燥，而且难免会漏掉一些东西。

下面，我们先准备一些简单的调试用的代码。

首先先定义两个 Service 接口：

```
// OrderService.java
public interface OrderService {

    Order createOrder(String username, String product);

    Order queryOrder(String username);
}
// UserService.java
public interface UserService {

    User createUser(String firstName, String lastName, int age);

    User queryUser();
}
```

然后，分别来一个接口实现类：

```
// OrderServiceImpl.java
public class OrderServiceImpl implements OrderService {

    @Override
    public Order createOrder(String username, String product) {
        Order order = new Order();
        order.setUsername(username);
        order.setProduct(product);
        return order;
    }

    @Override
    public Order queryOrder(String username) {
        Order order = new Order();
        order.setUsername("test");
        order.setProduct("test");
        return order;
    }
}

// UserServiceImpl.java
public class UserServiceImpl implements UserService {

    @Override
    public User createUser(String firstName, String lastName, int age) {
        User user = new User();
        user.setFirstName(firstName);
        user.setLastName(lastName);
        user.setAge(age);
        return user;
    }

    @Override
    public User queryUser() {
        User user = new User();
        user.setFirstName("test");
        user.setLastName("test");
        user.setAge(20);
        return user;
    }
}
```

写两个 Advice：

```
public class LogArgsAdvice implements MethodBeforeAdvice {

    @Override
    public void before(Method method, Object[] args, Object target) throws Throwable {
        System.out.println("准备执行方法: " + method.getName() + ", 参数列表：" + Arrays.toString(args));
    }
}
public class LogResultAdvice implements AfterReturningAdvice {

    @Override
    public void afterReturning(Object returnValue, Method method, Object[] args, Object target)
            throws Throwable {
        System.out.println(method.getName() + "方法返回：" + returnValue);
    }
}
```

配置一下：

![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE1vdJw82Jwd27p1fFOpUGJgKPXVNYp15sanmmV7GOTXhYMcmhjVXNYxLYuDic4PYZYOSx5x1Q4RthA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

> 我们这边使用了前面文章介绍的配置 Advisor 的方式，我们回顾一下。
>
> 每个 advisor 内部持有 advice 实例，advisor 负责匹配，内部的 advice 负责实现拦截处理。配置了各个 advisor 后，配置 DefaultAdvisorAutoProxyCreator 使得所有的 advisor 配置自动生效。

启动：

```
public class SpringAopSourceApplication {

   public static void main(String[] args) {

      // 启动 Spring 的 IOC 容器
      ApplicationContext context = new ClassPathXmlApplicationContext("classpath:DefaultAdvisorAutoProxy.xml");

      UserService userService = context.getBean(UserService.class);
      OrderService orderService = context.getBean(OrderService.class);

      userService.createUser("Tom", "Cruise", 55);
      userService.queryUser();

      orderService.createOrder("Leo", "随便买点什么");
      orderService.queryOrder("Leo");
   }
}
```

输出：

```
准备执行方法: createUser, 参数列表：[Tom, Cruise, 55]
queryUser方法返回：User{firstName='test', lastName='test', age=20, address='null'}
准备执行方法: createOrder, 参数列表：[Leo, 随便买点什么]
queryOrder方法返回：Order{username='test', product='test'}
```

> 从输出结果，我们可以看到：
>
> LogArgsAdvice 作用于 UserService#createUser(…) 和 OrderService#createOrder(…) 两个方法；
>
> LogResultAdvice 作用于 UserService#queryUser() 和 OrderService#queryOrder(…) 两个方法；

下面的代码分析中，我们将基于这个简单的例子来介绍。

## IOC 容器管理 AOP 实例

本节介绍 Spring AOP 是怎么作用于 IOC 容器中的 bean 的。

Spring AOP 的使用介绍 那篇文章已经介绍过 DefaultAdvisorAutoProxyCreator 类了，它能实现自动将所有的 advisor 生效。

我们来追踪下 DefaultAdvisorAutoProxyCreator 类，看看它是怎么一步步实现的动态代理。然后在这个基础上，我们再简单追踪下 @AspectJ 配置方式下的源码实现。

首先，先看下 DefaultAdvisorAutoProxyCreator 的继承结构：

![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE1vdJw82Jwd27p1fFOpUGJgKbibb5VYDUPtY4wUXxJiaicPZuMiaHAacbzTu0rUjYicRNVduw8KZR6qiaicA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

我们可以发现，DefaultAdvisorAutoProxyCreator 最后居然是一个 BeanPostProcessor，在 Spring IOC 源码分析的时候说过，BeanPostProcessor 的两个方法，分别在 init-method 的前后得到执行。

```
public interface BeanPostProcessor {
    Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
    Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
}
```

这里再贴一下 IOC 的源码，我们回顾一下：

// AbstractAutowireCapableBeanFactory

```
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final Object[] args)
            throws BeanCreationException {

    // Instantiate the bean.
    BeanWrapper instanceWrapper = null;
    if (mbd.isSingleton()) {
        instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
    }
    if (instanceWrapper == null) {
        // 1. 创建实例
        instanceWrapper = createBeanInstance(beanName, mbd, args);
    }
    ...

    // Initialize the bean instance.
    Object exposedObject = bean;
    try {
        // 2. 装载属性
        populateBean(beanName, mbd, instanceWrapper);
        if (exposedObject != null) {
            // 3. 初始化
            exposedObject = initializeBean(beanName, exposedObject, mbd);
        }
    }
    ...
}
```

在上面第 3 步 initializeBean(...) 方法中会调用 BeanPostProcessor 中的方法，如下：

```
protected Object initializeBean(final String beanName, final Object bean, RootBeanDefinition mbd) {
   ...
   Object wrappedBean = bean;
   if (mbd == null || !mbd.isSynthetic()) {
      // 1. 执行每一个 BeanPostProcessor 的 postProcessBeforeInitialization 方法
      wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
   }

   try {
      // 调用 bean 配置中的 init-method="xxx"
      invokeInitMethods(beanName, wrappedBean, mbd);
   }
   ...
   if (mbd == null || !mbd.isSynthetic()) {
      // 我们关注的重点是这里！！！
      // 2. 执行每一个 BeanPostProcessor 的 postProcessAfterInitialization 方法
      wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
   }
   return wrappedBean;
}
```

也就是说，Spring AOP 会在 IOC 容器创建 bean 实例的最后对 bean 进行处理。其实就是在这一步进行代理增强。

我们回过头来，DefaultAdvisorAutoProxyCreator 的继承结构中，postProcessAfterInitialization() 方法在其父类 AbstractAutoProxyCreator 这一层被覆写了：

// AbstractAutoProxyCreator

```
@Override
public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
   if (bean != null) {
      Object cacheKey = getCacheKey(bean.getClass(), beanName);
      if (!this.earlyProxyReferences.contains(cacheKey)) {
         return wrapIfNecessary(bean, beanName, cacheKey);
      }
   }
   return bean;
}
```

继续往里看 wrapIfNecessary(...) 方法，这个方法将返回代理类（如果需要的话）：

```
protected Object wrapIfNecessary(Object bean, String beanName, Object cacheKey) {
   if (beanName != null && this.targetSourcedBeans.contains(beanName)) {
      return bean;
   }
   if (Boolean.FALSE.equals(this.advisedBeans.get(cacheKey))) {
      return bean;
   }
   if (isInfrastructureClass(bean.getClass()) || shouldSkip(bean.getClass(), beanName)) {
      this.advisedBeans.put(cacheKey, Boolean.FALSE);
      return bean;
   }

   // 返回匹配当前 bean 的所有的 advisor、advice、interceptor
   // 对于本文的例子，"userServiceImpl" 和 "OrderServiceImpl" 这两个 bean 创建过程中，
   //   到这边的时候都会返回两个 advisor
   Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null);
   if (specificInterceptors != DO_NOT_PROXY) {
      this.advisedBeans.put(cacheKey, Boolean.TRUE);
      // 创建代理...创建代理...创建代理...
      Object proxy = createProxy(
            bean.getClass(), beanName, specificInterceptors, new SingletonTargetSource(bean));
      this.proxyTypes.put(cacheKey, proxy.getClass());
      return proxy;
   }

   this.advisedBeans.put(cacheKey, Boolean.FALSE);
   return bean;
}
```

这里有两个点提一下：

getAdvicesAndAdvisorsForBean(bean.getClass(), beanName, null)，这个方法将得到所有的可用于拦截当前 bean 的 advisor、advice、interceptor。

另一个就是 TargetSource 这个概念，它用于封装真实实现类的信息，上面用了 SingletonTargetSource 这个实现类，其实我们这里也不太需要关心这个，知道有这么回事就可以了。

我们继续往下看 createProxy(…) 方法：

```
// 注意看这个方法的几个参数，
//   第三个参数携带了所有的 advisors
//   第四个参数 targetSource 携带了真实实现的信息
protected Object createProxy(
      Class<?> beanClass, String beanName, Object[] specificInterceptors, TargetSource targetSource) {

   if (this.beanFactory instanceof ConfigurableListableBeanFactory) {
      AutoProxyUtils.exposeTargetClass((ConfigurableListableBeanFactory) this.beanFactory, beanName, beanClass);
   }

   // 创建 ProxyFactory 实例
   ProxyFactory proxyFactory = new ProxyFactory();
   proxyFactory.copyFrom(this);

   // 在 schema-based 的配置方式中，我们介绍过，如果希望使用 CGLIB 来代理接口，可以配置
   // proxy-target-class="true",这样不管有没有接口，都使用 CGLIB 来生成代理：
   //   <aop:config proxy-target-class="true">......</aop:config>
   if (!proxyFactory.isProxyTargetClass()) {
      if (shouldProxyTargetClass(beanClass, beanName)) {
         proxyFactory.setProxyTargetClass(true);
      }
      else {
         // 点进去稍微看一下代码就知道了，主要就两句：
         // 1. 有接口的，调用一次或多次：proxyFactory.addInterface(ifc);
         // 2. 没有接口的，调用：proxyFactory.setProxyTargetClass(true);
         evaluateProxyInterfaces(beanClass, proxyFactory);
      }
   }

   // 这个方法会返回匹配了当前 bean 的 advisors 数组
   // 对于本文的例子，"userServiceImpl" 和 "OrderServiceImpl" 到这边的时候都会返回两个 advisor
   // 注意：如果 specificInterceptors 中有 advice 和 interceptor，它们也会被包装成 advisor，进去看下源码就清楚了
   Advisor[] advisors = buildAdvisors(beanName, specificInterceptors);
   for (Advisor advisor : advisors) {
      proxyFactory.addAdvisor(advisor);
   }

   proxyFactory.setTargetSource(targetSource);
   customizeProxyFactory(proxyFactory);

   proxyFactory.setFrozen(this.freezeProxy);
   if (advisorsPreFiltered()) {
      proxyFactory.setPreFiltered(true);
   }

   return proxyFactory.getProxy(getProxyClassLoader());
}
```

我们看到，这个方法主要是在内部创建了一个 ProxyFactory 的实例，然后 set 了一大堆内容，剩下的工作就都是这个 ProxyFactory 实例的了，通过这个实例来创建代理: `getProxy(classLoader)`。

## ProxyFactory 详解

根据上面的源码，我们走到了 ProxyFactory 这个类了，我们到这个类来一看究竟。

顺着上面的路子，我们首先到 ProxyFactory#getProxy(classLoader) 方法：

```
public Object getProxy(ClassLoader classLoader) {
   return createAopProxy().getProxy(classLoader);
}
```

该方法首先通过 createAopProxy() 创建一个 AopProxy 的实例：

```
protected final synchronized AopProxy createAopProxy() {
   if (!this.active) {
      activate();
   }
   return getAopProxyFactory().createAopProxy(this);
}
```

创建 AopProxy 之前，我们需要一个 AopProxyFactory 实例，然后看 ProxyCreatorSupport 的构造方法：

```
public ProxyCreatorSupport() {
   this.aopProxyFactory = new DefaultAopProxyFactory();
}
```

这样就将我们导到 `DefaultAopProxyFactory` 这个类了，我们看它的 createAopProxy(…) 方法：

```
public class DefaultAopProxyFactory implements AopProxyFactory, Serializable {

   @Override
   public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
      // (我也没用过这个optimize，默认false) || (proxy-target-class=true) || (没有接口)
      if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
         Class<?> targetClass = config.getTargetClass();
         if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: " +
                  "Either an interface or a target is required for proxy creation.");
         }
         // 如果要代理的类本身就是接口，也会用 JDK 动态代理
         // 我也没用过这个。。。
         if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
         }
         return new ObjenesisCglibAopProxy(config);
      }
      else {
         // 如果有接口，会跑到这个分支
         return new JdkDynamicAopProxy(config);
      }
   }
   // 判断是否有实现自定义的接口
   private boolean hasNoUserSuppliedProxyInterfaces(AdvisedSupport config) {
      Class<?>[] ifcs = config.getProxiedInterfaces();
      return (ifcs.length == 0 || (ifcs.length == 1 && SpringProxy.class.isAssignableFrom(ifcs[0])));
   }

}
```

到这里，我们知道 createAopProxy 方法有可能返回 JdkDynamicAopProxy 实例，也有可能返回 ObjenesisCglibAopProxy 实例，这里总结一下：

如果被代理的目标类实现了一个或多个自定义的接口，那么就会使用 JDK 动态代理，如果没有实现任何接口，会使用 CGLIB 实现代理，如果设置了 proxy-target-class="true"，那么都会使用 CGLIB。

JDK 动态代理基于接口，所以只有接口中的方法会被增强，而 CGLIB 基于类继承，需要注意就是如果方法使用了 final 修饰，或者是 private 方法，是不能被增强的。

有了 AopProxy 实例以后，我们就回到这个方法了：

```
public Object getProxy(ClassLoader classLoader) {
   return createAopProxy().getProxy(classLoader);
}
```

我们分别来看下两个 AopProxy 实现类的 getProxy(classLoader) 实现。

JdkDynamicAopProxy 类的源码比较简单，总共两百多行，

```
@Override
public Object getProxy(ClassLoader classLoader) {
   if (logger.isDebugEnabled()) {
      logger.debug("Creating JDK dynamic proxy: target source is " + this.advised.getTargetSource());
   }
   Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
   findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
   return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```

java.lang.reflect.Proxy.newProxyInstance(…) 方法需要三个参数，第一个是 ClassLoader，第二个参数代表需要实现哪些接口，第三个参数最重要，是 InvocationHandler 实例，我们看到这里传了 this，因为 JdkDynamicAopProxy 本身实现了 InvocationHandler 接口。

InvocationHandler 只有一个方法，当生成的代理类对外提供服务的时候，都会导到这个方法中：

```
public interface InvocationHandler {
    public Object invoke(Object proxy, Method method, Object[] args)
        throws Throwable;
}
```

下面来看看 JdkDynamicAopProxy 对其的实现：

```
@Override
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
   MethodInvocation invocation;
   Object oldProxy = null;
   boolean setProxyContext = false;

   TargetSource targetSource = this.advised.targetSource;
   Class<?> targetClass = null;
   Object target = null;

   try {
      if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
         // The target does not implement the equals(Object) method itself.
         // 代理的 equals 方法
         return equals(args[0]);
      }
      else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
         // The target does not implement the hashCode() method itself.
         // 代理的 hashCode 方法
         return hashCode();
      }
      else if (method.getDeclaringClass() == DecoratingProxy.class) {
         // There is only getDecoratedClass() declared -> dispatch to proxy config.
         //
         return AopProxyUtils.ultimateTargetClass(this.advised);
      }
      else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
            method.getDeclaringClass().isAssignableFrom(Advised.class)) {
         // Service invocations on ProxyConfig with the proxy config...
         return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
      }

      Object retVal;

      // 如果设置了 exposeProxy，那么将 proxy 放到 ThreadLocal 中
      if (this.advised.exposeProxy) {
         // Make invocation available if necessary.
         oldProxy = AopContext.setCurrentProxy(proxy);
         setProxyContext = true;
      }

      // May be null. Get as late as possible to minimize the time we "own" the target,
      // in case it comes from a pool.
      target = targetSource.getTarget();
      if (target != null) {
         targetClass = target.getClass();
      }

      // Get the interception chain for this method.
      // 创建一个 chain，包含所有要执行的 advice
      List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

      // Check whether we have any advice. If we don't, we can fallback on direct
      // reflective invocation of the target, and avoid creating a MethodInvocation.
      if (chain.isEmpty()) {
         // We can skip creating a MethodInvocation: just invoke the target directly
         // Note that the final invoker must be an InvokerInterceptor so we know it does
         // nothing but a reflective operation on the target, and no hot swapping or fancy proxying.
         // chain 是空的，说明不需要被增强，这种情况很简单
         Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
         retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
      }
      else {
         // We need to create a method invocation...
         // 执行方法，得到返回值
         invocation = new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
         // Proceed to the joinpoint through the interceptor chain.
         retVal = invocation.proceed();
      }

      // Massage return value if necessary.
      Class<?> returnType = method.getReturnType();
      if (retVal != null && retVal == target &&
            returnType != Object.class && returnType.isInstance(proxy) &&
            !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
         // Special case: it returned "this" and the return type of the method
         // is type-compatible. Note that we can't help if the target sets
         // a reference to itself in another returned object.
         retVal = proxy;
      }
      else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
         throw new AopInvocationException(
               "Null return value from advice does not match primitive return type for: " + method);
      }
      return retVal;
   }
   finally {
      if (target != null && !targetSource.isStatic()) {
         // Must have come from TargetSource.
         targetSource.releaseTarget(target);
      }
      if (setProxyContext) {
         // Restore old proxy.
         AopContext.setCurrentProxy(oldProxy);
      }
   }
}
```

> 上面就三言两语说了一下，感兴趣的读者自己去深入探索下，不是很难。简单地说，就是在执行每个方法的时候，判断下该方法是否需要被一次或多次增强（执行一个或多个 advice）。

说完了 JDK 动态代理 JdkDynamicAopProxy#getProxy(classLoader)，我们再来瞄一眼 CGLIB 的代理实现 ObjenesisCglibAopProxy#getProxy(classLoader)。

ObjenesisCglibAopProxy 继承了 CglibAopProxy，而 CglibAopProxy 继承了 AopProxy。

> ObjenesisCglibAopProxy 使用了 Objenesis 这个库，和 cglib 一样，我们不需要在 maven 中进行依赖，因为 spring-core.jar 直接把它的源代码也搞过来了。
>
> ![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE1vdJw82Jwd27p1fFOpUGJgibNfpfP595sTXtjYwqNZDduyktzB8iax2zicTSNtmjfI4Z3qjPPNT0ibPA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

通过 CGLIB 生成代理的代码量有点大，我们就不进行深入分析了，我们看下大体的骨架。它的 getProxy(classLoader) 方法在父类 CglibAopProxy 类中：

// CglibAopProxy#getProxy(classLoader)

```
@Override
public Object getProxy(ClassLoader classLoader) {
      ...
      // Configure CGLIB Enhancer...
      Enhancer enhancer = createEnhancer();
      if (classLoader != null) {
         enhancer.setClassLoader(classLoader);
         if (classLoader instanceof SmartClassLoader &&
               ((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
            enhancer.setUseCache(false);
         }
      }
      enhancer.setSuperclass(proxySuperClass);
      enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
      enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
      enhancer.setStrategy(new ClassLoaderAwareUndeclaredThrowableStrategy(classLoader));

      Callback[] callbacks = getCallbacks(rootClass);
      Class<?>[] types = new Class<?>[callbacks.length];
      for (int x = 0; x < types.length; x++) {
         types[x] = callbacks[x].getClass();
      }
      // fixedInterceptorMap only populated at this point, after getCallbacks call above
      enhancer.setCallbackFilter(new ProxyCallbackFilter(
            this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
      enhancer.setCallbackTypes(types);

      // Generate the proxy class and create a proxy instance.
      return createProxyClassAndInstance(enhancer, callbacks);
   }
   catch (CodeGenerationException ex) {
      ...
   }
   catch (IllegalArgumentException ex) {
      ...
   }
   catch (Throwable ex) {
      ...
   }
}
```

CGLIB 生成代理的核心类是 Enhancer 类，这里就不展开说了。

## 基于注解的 Spring AOP 源码分析

上面我们走马观花地介绍了使用 DefaultAdvisorAutoProxyCreator 来实现 Spring AOP 的源码，这里，我们也同样走马观花地来看下 @AspectJ 的实现原理。

我们之前说过，开启 @AspectJ 的两种方式，一个是 `<aop:aspectj-autoproxy/>`，一个是 `@EnableAspectJAutoProxy`，它们的原理是一样的，都是通过注册一个 bean 来实现的。

解析 `<aop:aspectj-autoproxy/>` 需要用到 AopNamespaceHandler：

![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE1vdJw82Jwd27p1fFOpUGJgQiamFvEjkxA0QWoY0K4XfWoDDsyZ4y4iagL2iagwkDvyaPhccYNcuBeIg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

然后到类 AspectJAutoProxyBeanDefinitionParser：

```
class AspectJAutoProxyBeanDefinitionParser implements BeanDefinitionParser {

   @Override
   @Nullable
   public BeanDefinition parse(Element element, ParserContext parserContext) {
      AopNamespaceUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(parserContext, element);
      extendBeanDefinition(element, parserContext);
      return null;
   }
   ...
}
```

进去 registerAspectJAnnotationAutoProxyCreatorIfNecessary(...) 方法：

```
public static void registerAspectJAnnotationAutoProxyCreatorIfNecessary(
      ParserContext parserContext, Element sourceElement) {

   BeanDefinition beanDefinition = AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(
         parserContext.getRegistry(), parserContext.extractSource(sourceElement));
   useClassProxyingIfNecessary(parserContext.getRegistry(), sourceElement);
   registerComponentIfNecessary(beanDefinition, parserContext);
}
```

再进去 AopConfigUtils#registerAspectJAnnotationAutoProxyCreatorIfNecessary(...)：

```
public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry,
      @Nullable Object source) {

   return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
}
```

最终我们看到，Spring 注册了一个 AnnotationAwareAspectJAutoProxyCreator 的 bean，beanName 为："org.springframework.aop.config.internalAutoProxyCreator"。

我们看下 AnnotationAwareAspectJAutoProxyCreator 的继承结构：

![img](https://mmbiz.qpic.cn/mmbiz_png/YUYc62VIvE1vdJw82Jwd27p1fFOpUGJgCS37AF6CtVkUib2VZG20BjD37SW5rrllyBvHCD4W5IuxRNYAGvpWoLg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

和前面介绍的 DefaultAdvisorAutoProxyCreator 一样，它也是一个 BeanPostProcessor，剩下的我们就不说了，它和它的父类 AspectJAwareAdvisorAutoProxyCreator 都不复杂。

## 闲聊 InstantiationAwareBeanPostProcessor

为什么要说这个呢？因为我发现，很多人都以为 Spring AOP 是通过这个接口来作用于 bean 生成代理的。

```
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {

   Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException;

   boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException;

   PropertyValues postProcessPropertyValues(
         PropertyValues pvs, PropertyDescriptor[] pds, Object bean, String beanName) throws BeansException;

}
```

它和 BeanPostProcessor 的方法非常相似，而且它还继承了 BeanPostProcessor。

不仔细看还真的不好区分，下面是 BeanPostProcessor 中的两个方法：

```
Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException;
Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException;
```

发现没有，InstantiationAwareBeanPostProcessor 是 `Instantiation`，BeanPostProcessor 是 `Initialization`，它代表的是 bean 在实例化完成并且属性注入完成，在执行 init-method 的前后进行作用的。

而 InstantiationAwareBeanPostProcessor 的执行时机要前面一些，大家需要翻下 IOC 的源码：

```
// AbstractAutowireCapableBeanFactory 447行
protected Object createBean(String beanName, RootBeanDefinition mbd, Object[] args) throws BeanCreationException {
   ...
   try {
      // 让 InstantiationAwareBeanPostProcessor 在这一步有机会返回代理
      Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
      if (bean != null) {
         return bean;
      }
   }
   // BeanPostProcessor 是在这里面实例化后才能得到执行
   Object beanInstance = doCreateBean(beanName, mbdToUse, args);
   ...
   return beanInstance;
}
```

点进去看 resolveBeforeInstantiation(beanName, mbdToUse) 方法，然后就会导到 InstantiationAwareBeanPostProcessor 的 postProcessBeforeInstantiation 方法，对于我们分析的 AOP 来说，该方法的实现在 AbstractAutoProxyCreator 类中：

```
@Override
public Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {
    ...
    if (beanName != null) {
      TargetSource targetSource = getCustomTargetSource(beanClass, beanName);
      if (targetSource != null) {
         this.targetSourcedBeans.add(beanName);
         Object[] specificInterceptors = getAdvicesAndAdvisorsForBean(beanClass, beanName, targetSource);
         Object proxy = createProxy(beanClass, beanName, specificInterceptors, targetSource);
         this.proxyTypes.put(cacheKey, proxy.getClass());
         return proxy;
      }
   }

   return null;
}
```

我们可以看到，这里也有创建代理的逻辑，以至于很多人会搞错。确实，这里是有可能创建代理的，但前提是对于相应的 bean 我们有自定义的 TargetSource 实现，进到 getCustomTargetSource(...) 方法就清楚了，我们需要配置一个 customTargetSourceCreators，它是一个 TargetSourceCreator 数组。

这里就不再展开说 TargetSource 了，请参考 Spring Reference 中的 Using TargetSources。

## 小结

本文真的是走马观花，和我之前写的文章有很大的不同，希望读者不会嫌弃。

不过如果读者有看过之前的 Spring IOC 源码分析和 Spring AOP 使用介绍 这两篇文章的话，通过看本文应该能对 Spring AOP 的源码实现有比较好的理解了。

本文说细节说得比较少，如果你在看源码的时候碰到不懂的，欢迎在评论区留言与大家进行交流。