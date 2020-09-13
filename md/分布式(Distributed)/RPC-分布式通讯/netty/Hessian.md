# 基于 HTTP 调用

```
<dependencies>
    <dependency>
        <groupId>com.caucho</groupId>
        <artifactId>hessian</artifactId>
        <version>4.0.63</version>
    </dependency>
</dependencies>
```

* 发布端

```java

```



* 调用端

```java
 String url = "http://localhost:8080/hessionDemo/hessian";
System.out.println("请求的服务端地址：" + url);

HessianProxyFactory factory = new HessianProxyFactory();
HelloService helloService = (HelloService) factory.create(HelloService.class, url);
System.out.println("服务端返回结果为：" + helloService.helloWorld("xxxx!"))
```



## 结合 spring

```xml
<!-- 调用 -->
<bean id="userLocaleService" class="org.springframework.remoting.caucho.HessianProxyFactoryBean">
    <property name="serviceUrl" value="http://127.0.0.1:8081/support/serviceUrl" />
    <property name="serviceInterface" value="service.UserLocaleService" />
    <property name="allowNonSerializable" value="true" />
    <property name="hessian2" value="true" />
</bean>


<!-- 发布 -->
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <!--userLocaleServiceImp是接口service.UserLocaleService的实现，通过注解@Service定义的-->
    <bean name="/userLocaleService" class="org.springframework.remoting.caucho.HessianServiceExporter">
        <property name="service" ref="userLocaleServiceImp"/>
        <property name="serviceInterface" value="service.UserLocaleService"/>
        <property name="allowNonSerializable" value="true"/>
    </bean>
</beans>
```



 