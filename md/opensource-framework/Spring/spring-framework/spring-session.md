**一.为什么要spring-session**

**在传统单机web应用中，一般使用tomcat/jetty等web容器时，用户的session都是由容器管理。浏览器使用cookie中记sessionId，容器根据sessionId判断用户是否存在会话session。这里的限制是，session存储在web容器中，被单台服务器容器管理。**

**但是网站主键演变，分布式应用和集群是趋势（提高性能）。此时用户的请求可能被负载分发至不同的服务器，此时传统的web容器管理用户会话session的方式即行不通。除非集群或者分布式web应用能够共享session，尽管tomcat等支持这样做。但是这样存在以下两点问题：**

**需要侵入web容器，提高问题的复杂 。web容器之间共享session，集群机器之间势必要交互耦合**

**基于这些，必须提供新的可靠的集群分布式/集群session的解决方案，突破traditional-session单机限制（即web容器session方式，下面简称traditional-session），spring-session应用而生。**

**Spring Session使得支持集群会话变得微不足道，而不依赖于特定于应用程序容器的解决方案。它还提供透明集成：**

**HttpSession - 允许以应用程序容器（即Tomcat）中立方式替换HttpSession，支持在头文件中提供会话ID以使用RESTful API**

**WebSocket - 提供在接收WebSocket消息时保持HttpSession活动的能力**

**WebSession - 允许以应用程序容器中立方式替换Spring WebFlux的WebSession**

**jar包**

```java
        <dependency>



            <groupId>org.springframework.boot</groupId>



            <artifactId>spring-boot-starter-data-redis</artifactId>



        </dependency>



        <!--spring-session-->



         <dependency>



            <groupId>org.springframework.session</groupId>



            <artifactId>spring-session-data-redis</artifactId>



        </dependency>



        <dependency>



            <groupId>org.springframework.session</groupId>



            <artifactId>spring-session-jdbc</artifactId>



        </dependency>
```

- **yml文件配置**

   **其它都是默认配置如需修改自己添加**

```java
 #redis



  redis:



    host:#默认为localhost



    password: #默认为空



    timeout: PT30M #30分钟



  #spring-session



  session:



    store-type: redis



    timeout: PT30M
```

**spring-boot配置完成就是怎么简单**

```java
    @GetMapping("query")



    @ResponseBody



    public String query(HttpServletRequest request) {



        HttpSession session = request.getSession();



        session.setAttribute("hello","123123123");



        return String.valueOf(session.getAttridbute("hello"));



      



    }
```

**由spring session 拦截session进行处理** 

**使用redisManger查看存储到redis成功**