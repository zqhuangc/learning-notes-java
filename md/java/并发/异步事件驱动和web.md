tomcat   cometProcessor

eventType



```java
@WebServlet(path,async = true)


AsyncContext asyncContext = request.startAsync();
asyncContext.addListener

//asyncContext.complete();
asyncContext.start(()->asyncContext.complete();));
```



找老技术在新技术中的影子



* 异步缺点

异常无法捕捉，在另一个线程，需要回调

执行时间相对不可知，