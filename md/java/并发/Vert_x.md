## [Vert.x](http://vertx.io/docs/)

**类似java 版的 nodejs**

基于 JVM 构建 Reactive 应用工具箱

基于 netty 开发

多编程语言支持

多模块支持（数据库、日志、服务发现）

#### 核心概念

* 线程和编程模型
  - 异步 I/O（asynchronous I/O）
  - 事件循环（Event-loops）
    操作系统-----事件消息循环
* 事件总线（Event Bus）

#### 应用场景

* Reactive 编程模型
  - 函数式
  - 异步
  - 非阻塞

* Web 技术栈
* 适合嵌入式或微服务应用









事件   消息

本地事件

分布式事件



```java
//Builder 模型    fluent Api
//vertx-core
Vertx vertx = Vertx.vertx();
vertx.serPeriodic(500,System.out::println);
vertx.deployVerticle#start,#stop


/**Event Bus*/
// 事件发布
// 消息通过地址区分，事件通过类型
vertx.eventBus()#xxx
// 事件订阅
vertx.eventBus()#xxx

/**web vertx-web*/
Router router = Router.router(vertx);
router.get(path).handler
router::accept
vertx.createHttpServer().requestHandler(...).listen(port);

/** 
web 使用跟 web flux 类似 不完全异步，需要声明

底层 线程池 excutors.newXXX
包装 
*/
```