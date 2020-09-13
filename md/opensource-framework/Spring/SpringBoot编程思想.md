# 核心篇

## 第1部分 总览Spring Boot
### 第1章 初览Spring Boot 2

#### 1.1 Spring Framework时代 2

#### 1.2 Spring Boot简介 3

#### 1.3 Spring Boot的特性 5

#### 1.4 准备运行环境 5
##### 1.4.1 装配JDK 8 5

##### 1.4.2 装配Maven 6

##### 1.4.3 装配IDE（集成开发环境） 8

### 第2章 理解独立的Spring应用 9

#### 2.1 创建SpringBoot应用 10

##### 2.1.1 命令行方式创建Spring Boot应用 11
##### 2.1.2 图形化界面创建Spring Boot应用 21
##### 2.1.3 创建SpringBoot应用可执行JAR 29
#### 2.2 运行SpringBoot应用 31
##### 2.2.1 执行SpringBoot应用可执行JAR 32

##### 2.2.2 Spring Boot应用可执行JAR资源结构 32
##### 2.2.3 FAT JAR和WAR执行模块——spring-boot-loader 36

##### 2.2.4 JarLauncher的实现原理 40

### 第3章 理解固化的Maven依赖 58

#### 3.1 spring-boot-starter-parent与spring-boot-dependencies简介 58

#### 3.2 理解spring-boot-starter-parent与spring-boot- dependencies 61

### 第4章 理解嵌入式Web容器 70

#### 4.1 嵌入式ServletWeb容器 71
##### 4.1.1 Tomcat作为嵌入式Servlet Web容器 72

##### 4.1.2 Jetty作为嵌入式ServletWeb容器 77

##### 4.1.3 Undertow作为嵌入式Servlet Web容器 80

#### 4.2 嵌入式ReactiveWeb容器 82
##### 4.2.1 UndertowServletWebServer作为嵌入式Reactive Web容器 82

##### 4.2.2 UndertowWebServer作为嵌入式Reactive Web容器 84

##### 4.2.3 WebServerInitializedEvent 91

##### 4.2.4 Jetty作为嵌入式ReactiveWeb容器 93

##### 4.2.5 Tomcat作为嵌入式Reactive Web容器 94

### 第5章 理解自动装配 96
#### 5.1 理解@SpringBootApplication注解语义 97

#### 5.2 @SpringBootApplication属性别名 103

#### 5.3 @SpringBootApplication标注非引导类 107

#### 5.4 @EnableAutoConfiguration激活自动装配 108

#### 5.5 @SpringBootApplication“继承”@Configuration CGLIB提升特性 110

#### 5.6 理解自动配置机制 112

#### 5.7 创建自动配置类 116

### 第6章 理解Production- Ready特性 119

#### 6.1 理解Production-Ready一般性定义 120

#### 6.2 理解SpringBoot Actuator 123

#### 6.3 Spring Boot Actuator Endpoints 124

#### 6.4 理解“外部化配置” 129

#### 6.5 理解“规约大于配置” 132

#### 6.6 小马哥有话说 134

#### 6.6.1 Spring Boot作为微服务中间件 134

#### 6.6.2 Spring Boot作为Spring Cloud基础设施 135

#### 6.7 下一站：走向自动装配 135

## 第2部分 走向自动装配

### 第7章 走向注解驱动编程（Annotation-Driven） 138

### 7.1 注解驱动发展史 138
##### 7.1.1 注解驱动启蒙时代：Spring Framework 1.x 138

##### 7.1.2 注解驱动过渡时代：Spring Framework 2.x 139

##### 7.1.3 注解驱动黄金时代：Spring Framework 3.x 142

##### 7.1.4 注解驱动完善时代：Spring Framework 4.x 146

##### 7.1.5 注解驱动当下时代：Spring Framework 5.x 151

####7.2 Spring核心注解场景分类 152

#### 7.3 Spring注解编程模型 154
##### 7.3.1 元注解（Meta-Annotations） 154

##### 7.3.2 Spring模式注解（Stereotype Annotations） 155

##### 7.3.3 Spring组合注解（Composed Annotations） 187

##### 7.3.4 Spring注解属性别名和覆盖（Attribute Aliases and Overrides） 195

### 第8章 Spring注解驱动设计模式 225

#### 8.1 Spring @Enable模块驱动 225
##### 8.1.1 理解@Enable模块驱动 225

##### 8.1.2 自定义@Enable模块驱动 226

##### 8.1.3 @Enable模块驱动原理 236

#### 8.2 Spring Web自动装配 250
##### 8.2.1 理解Web自动装配 250

##### 8.2.2 自定义Web自动装配 254

##### 8.2.3 Web自动装配原理 258

#### 8.3 Spring条件装配 270
##### 8.3.1 理解配置条件装配 271

##### 8.3.2 自定义配置条件装配 274

##### 8.3.3 配置条件装配原理 277

### 第9章 Spring Boot自动装配 292
#### 9.1 理解SpringBoot自动装配 295

##### 9.1.1 理解@EnableAutoConfiguration 296

##### 9.1.2 优雅地替换自动装配 298

##### 9.1.3 失效自动装配 298

#### 9.2 Spring Boot自动装配原理 299
##### 9.2.1 @EnableAutoConfiguration读取候选装配组件 301

##### 9.2.2 @EnableAutoConfiguration排除自动装配组件 305

##### 9.2.3 @EnableAutoConfiguration过滤自动装配组件 307

##### 9.2.4 @EnableAutoConfiguration自动装配事件 313

##### 9.2.5 @EnableAutoConfiguration自动装配生命周期 317

##### 9.2.6 @EnableAutoConfiguration排序自动装配组件 324

##### 9.2.7 @EnableAutoConfiguration自动装配BasePackages 332

#### 9.3 自定义SpringBoot自动装配 337
##### 9.3.1 自动装配Class命名的潜规则 338

##### 9.3.2 自动装配package命名的潜规则 338

##### 9.3.3 自定义SpringBoot Starter 340

#### 9.4 Spring Boot条件化自动装配 346
##### 9.4.1 Class条件注解 347

##### 9.4.2 Bean条件注解 358

##### 9.4.3 属性条件注解 370

##### 9.4.4 Resource条件注解 376

##### 9.4.5 Web应用条件注解 391

##### 9.4.6 Spring表达式条件注解 397

#### 9.5 小马哥有话说 401
#### 9.6 下一站：理解SpringApplication 402

### 第3部分 理解SpringApplication
### 第10章 SpringApplication初始化阶段 405
#### 10.1 SpringApplication构造阶段 405
##### 10.1.1 理解SpringApplication主配置类 406

##### 10.1.2 SpringApplication的构造过程 410

##### 10.1.3 推断Web应用类型 411

##### 10.1.4 加载Spring应用上下文初始化器（ApplicationContextInitializer） 412

##### 10.1.5 加载Spring应用事件器（ApplicationListener） 415

##### 10.1.6 推断应用引导类 416

#### 10.2 SpringApplication配置阶段 417
##### 10.2.1 自定义SpringApplication 417

##### 10.2.2 调整SpringApplication设置 417

##### 10.2.3 增加SpringApplication配置源 420

##### 10.2.4 调整SpringBoot外部化配置 423

### 第11章 SpringApplication运行阶段 425
#### 11.1 SpringApplication准备阶段 425
##### 11.1.1 理解SpringApplicationRunListeners 426

##### 11.1.2 理解SpringApplicationRunListener 428

##### 11.1.3 理解SpringBoot事件 431

##### 11.1.4 理解Spring事件/机制 432

##### 11.1.5 理解SpringBoot事件/机制 492

##### 11.1.6 装配ApplicationArguments 509

##### 11.1.7 准备ConfigurableEnvironment 512

##### 11.1.8 创建Spring应用上下文（ConfigurableApplicationContext） 512

##### 11.1.9 Spring应用上下文运行前准备 516

#### 11.2 Spring应用上下文启动阶段 537
#### 11.3 Spring应用上下文启动后阶段 539
##### 11.3.1 afterRefresh方法签名的变化 540

##### 11.3.2 afterRefresh方法语义的变化 541

##### 11.3.3 Spring Boot事件ApplicationStartedEvent语义的变化 543

##### 11.3.4 执行CommandLineRunner和ApplicationRunner 548

### 第12章 SpringApplication结束阶段 550
#### 12.1 SpringApplication正常结束 550
#### 12.2 SpringApplication异常结束 555
##### 12.2.1 Spring Boot异常处理 556

##### 12.2.2 错误分析报告器——FailureAnalysisReporter 562

##### 12.2.3 自定义实现FailureAnalyzer和FailureAnalysisReporter 564

##### 12.2.4 Spring Boot 2.0重构handleRunFailure和reportFailure方法 566

##### 12.2.5 Spring Boot 2.0的SpringBootExceptionReporter接口 567

### 第13章 Spring Boot应用退出 571
#### 13.1 Spring Boot应用正常退出 572
##### 13.1.1 ExitCodeGenerator Bean生成退出码 572

##### 13.1.2 ExitCodeGenerator Bean退出码使用场景 576

#### 13.2 Spring Boot应用异常退出 580
##### 13.2.1 ExitCodeGenerator异常使用场景 582

##### 13.2.2 ExitCodeExceptionMapper Bean映射异常与退出码 587

##### 13.2.3 退出码用于SpringApplication异常结束 589

#### 13.3 小马哥有话说 594



