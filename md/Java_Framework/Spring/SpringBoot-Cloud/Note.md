[Spring-Cloud-code-location](https://github.com/mercyblitz/segmentfault-lessons/)

版本意识，不同版本间的差异，新的引入了什么，旧的移除了什么，原因

## [spring-reference5.x以下](https://docs.spring.io/spring/docs/)

JMX
MBeans

Spring Boot Actuator

Endpoint/restart  通过 HTTP

JMX restart operation 通过 RMI、相关协议



apollo

配置： zookeeper做管理

频繁读写 -->  高可用，最终一致可能就行了

强一致性 --> 读写不频繁



配置

monitor 模块
push notification

Watch  service event--kind



### 注解

@ConditionalOnBean（仅仅在当前上下文中存在某个对象时，才会实例化一个Bean）
@ConditionalOnClass（某个class位于类路径上，才会实例化一个Bean）
@ConditionalOnExpression（当表达式为true的时候，才会实例化一个Bean）
@ConditionalOnMissingBean（仅仅在当前上下文中不存在某个对象时，才会实例化一个Bean）
@ConditionalOnMissingClass（某个class类路径上不存在的时候，才会实例化一个Bean）
@ConditionalOnNotWebApplication（不是web应用）



@ConditionalOnClass：该注解的参数对应的类必须存在，否则不解析该注解修饰的配置类；
@ConditionalOnMissingBean：该注解表示，如果存在它修饰的类的bean，则不需要再创建这个bean；可以给该注解传入参数例如@ConditionOnMissingBean(name = "example")，这个表示如果name为“example”的bean存在，这该注解修饰的代码块不执行。



@Import()





•基本概念  

​    REST = RESTful = Representational State Transfer，is one way of providing interoperability between computer systems on the Internet.



•参考资源


​     [https](https://en.wikipedia.org/wiki/Representational_state_transfer)[://en.wikipedia.org/wiki/Representational_state_transfer](https://en.wikipedia.org/wiki/Representational_state_transfer)




•服务端核心接口

–定义相关

•@Controller

•@RestController

–映射相关

•@RequestMapping

•@PathVariable

–方法相关

•RequestMethod


–请求相关

•@RequestParam

•@RequestHeader


•@CookieValue

•RequestEntity

–响应相关

•@ResponseBody

•ResponseEntity

•HttpStatus




•目标

–理解“资源操作”（Manipulation of resources through representations）

•POST、GET、PUST、DELETE

•幂等方法

–理解“自描述消息” （Self-descriptive messages）

•Content-Type

•MIME-Type

–扩展“自描述消息”

•HttpMessageConvertor






•客户端核心接口

–URI相关

•UriComponents

•UriComponentsBuilder

–请求相关

•ClientHttpRequestFactory

•ClientHttpRequestInterceptor

–响应相关

•ClientHttpResponse

•ResponseEntity






•为什么要使用新 Web 框架

–非阻塞 Web 技术栈使用少量的内存处理并发并且占用少量的硬件资源达到应用伸缩性的目的。代表技术：Servlet 3.1

–函数式编程。代表技术：Java 8 Lambda 表达式



•为什么要使用 Reactive

–Reactive 编程模型的通知机制（Complete、Error 等信号）

–非阻塞反压力（Non-Blocking Back Pressure）






•函数式断点










