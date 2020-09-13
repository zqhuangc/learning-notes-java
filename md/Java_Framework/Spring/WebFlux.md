### 题外

tomcat 8+  默认 nio

RequestContextHolder   ThreadLocal

> Servlet ------ NIO部分 ,listener
>
> SpringBoot ----- ServletContainers
>
> WatchedEvent ----------- Kind



* [异步web](./Servlet3.1和异步web.md)

Upgrade Processing  HttpUpgradeHandler

WebConnection

WebSocket 半长连接

HTTP2 长连接



JDO

从规范入手



处理json性能比tomcat好

## WebFlux（Spring 5.x+，SpringBoot 2.x+）

**非线程安全**

非阻塞编程

- NIO
- Reactive

函数式编程

- Lambda
- Kotlin

### 初始

* Annotation Controller
* WebFlux 配置
* Reactor 框架

默认非异步

NettyWebServer 屏蔽底层实现 可以用原来 mvc的方式

Mono，Optional

Flux，Collections

跨域 CORS

模仿了 REST Web Service

> java6  webServer
>
> ejb3  注解注入

jdbc5   reactive

reactive-data    mongodb

### 函数式 Endpoint

`spring-boot-starter-webflux`

* Handler Function

  ```java
  public interface HandlerFunction<T extends ServerResponse> {
  
  	/**
  	 * Handle the given request.
  	 * @param request the request to handle
  	 * @return the response
  	 */
  	Mono<T> handle(ServerRequest request);
  
  }
  ```

* RouterFunction

  - RouterFunctions
  - RequestPredicates

  ```java
      @Bean
      public RouterFunction routerFunction(){
          return RouterFunctions.route(RequestPredicates.GET("/webflux"),
                  request -> ServerResponse.ok().body(Mono.just("just for test route"), String.class)
                  );
  
      }
  ```



```java
public static <T extends ServerResponse> RouterFunction<T> route(
			RequestPredicate predicate, HandlerFunction<T> handlerFunction) {

		return new DefaultRouterFunction<>(predicate, handlerFunction);
	}
```

