在使用dubbo时，提供服务所需要的java代码（bean、interface等）单独打包成jar，提供使用
provider需要将接口方法实现。
consumer调用接口方法。

使用dubbo要求传输的对象必须实现序列化接口


监控服务：修改 monitor 中conf下的 dubbo.properties   修改注册中心的地址：改为你是用的注册中心地址， 如   dubbo.registry.address=zookeeper://127.0.0.1:2181？client=zkclient
dubbo.admin.root.password = 
dubbo.admin.guest.password = 

## 工作流程
应用信息
注册中心
协议端口

```
暴露接口api
服务方实现接口方法(配置说明具体实现类)
消费方调用接口方法
```

## pom.xml 依赖  

```xml
<dependencies>
      <!-- dubbo采用spring配置方式，所以需要导入spring容器依赖 -->
      <dependency>
         <groupId>org.springframework</groupId>
         <artifactId>spring-webmvc</artifactId>
         <version>4.1.3.RELEASE</version>
      </dependency>
      <dependency>
         <groupId>org.slf4j</groupId>
         <artifactId>slf4j-log4j12</artifactId>
         <version>1.6.4</version>
      </dependency>
     
      <dependency>
         <groupId>com.alibaba</groupId>
         <artifactId>dubbo</artifactId>
         <version>2.5.3</version>
         <exclusions>
            <exclusion>
               <!-- 排除传递spring依赖 -->
               <artifactId>spring</artifactId>
               <groupId>org.springframework</groupId>
            </exclusion>
         </exclusions>
      </dependency>
   </dependencies>
```

服务提供方dubbo配置（留意名称空间）

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
  xmlns:context="http://www.springframework.org/schema/context"xmlns:p="http://www.springframework.org/schema/p"
  xmlns:aop="http://www.springframework.org/schema/aop"xmlns:tx="http://www.springframework.org/schema/tx"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
  xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
   http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd
   http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.0.xsd http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-4.0.xsd
   http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd"> 
   <!-- 提供方应用信息，用于计算依赖关系 -->
   <dubbo:application name="dubbo-b-server" owner="melody（表示谁维护）"/>
   <!-- 这里使用的注册中心是zookeeper -->
<dubbo:registry address="zookeeper://127.0.0.1:2181"client="zkclient"/> 
   <!-- 用dubbo协议在20880端口暴露服务 -->
   <dubbo:protocol name="dubbo"port="20880"/>
   <!-- 将该接口暴露到dubbo中 -->
<dubbo:service interface="cn.itcast.dubbo.service.UserService" ref="userServiceImpl"/>
   <!-- 将具体的实现类加入到Spring容器中 -->
<bean id="userServiceImpl" class="cn.itcast.dubbo.service.impl.UserServiceImpl"/>

</beans>
```

导入zookeeper依赖
```xml
<dependency>
      <groupId>org.apache.zookeeper</groupId>
      <artifactId>zookeeper</artifactId>
      <version>3.3.3</version>
</dependency>

<dependency>
      <groupId>com.github.sgroschupf</groupId>
      <artifactId>zkclient</artifactId>
      <version>0.1</version>
</dependency>

```

编写Web.xml
```xml
<?xmlversion="1.0"encoding="UTF-8"?>
<web-appxmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"xmlns="http://java.sun.com/xml/ns/javaee"xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd"id="WebApp_ID"version="2.5">
<context-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>classpath:dubbo/dubbo-*.xml</param-value>
   </context-param>
  
   <!--Spring的ApplicationContext 载入 -->
   <listener>
   <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
   </listener>
```

消费方配置dubbo-consumer.xml：
```xml
<beansxmlns="http://www.springframework.org/schema/beans"
xmlns:context="http://www.springframework.org/schema/context"xmlns:p="http://www.springframework.org/schema/p"
xmlns:aop="http://www.springframework.org/schema/aop"xmlns:tx="http://www.springframework.org/schema/tx"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"xmlns:dubbo="http://code.alibabatech.com/schema/dubbo"
xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-4.0.xsd
   http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.0.xsd
   http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-4.0.xsd http://www.springframework.org/schema/tx http://www.springframework.org/schema/tx/spring-tx-4.0.xsd
   http://code.alibabatech.com/schema/dubbo http://code.alibabatech.com/schema/dubbo/dubbo.xsd">
 
   <!-- 消费方应用信息，用于计算依赖关系 -->
   <dubbo:application name="dubbo-a-consumer" />
 
   <!-- 使用的注册中心是zookeeper   N/A -->
<dubbo:registry address="zookeeper://127.0.0.1:2181" client="zkclient"/>
  
   <!-- 从注册中心中查找服务 -->
   <dubbo:reference id="userService" interface="cn.itcast.dubbo.service.UserService"/>
 
</beans>
```

### 服务使用

@Reference  注入服务实现类（与注解一起用？）







# 入门

# Dubbo 

各个应用节点中的url管理维护很困难、 依赖关系很模糊

 

每个应用节点的性能、访问量、响应时间，没办法评估

 

# Dubbo的使用入门

 url方式

**dubbo://177.1.1.82:20880/com.gupao.vip.mic.dubbo.order.IOrderServices**

 

?anyhost=true&application=order-provider&dubbo=2.5.3&interface=com.gupao.vip.mic.dubbo.order.IOrderServices&methods=doOrder&owner=mic&pid=12500&side=provider&timestamp=1502889986089, dubbo version: 2.5.3, current host: 127.0.0.1

 

dubbo://177.1.1.82/20880/com.gupao.vip.mic.dubbo.order.IOrderServices%3Fanyhost%3Dtrue%26application%3Dorder-provider%26dubbo%3D2.5.3%26interface%3Dcom.gupao.vip.mic.dubbo.order.IOrderServices%26methods%3DdoOrder%26owner%3Dmic%26pid%3D10804%26side%3Dprovider%26timestamp%3D1502890818766

 

 

# Main方法怎么启动的

 Main.main() ->container.start()（默认springcontainer，有配置文件），wait保持进程不终止

extension

# 日志怎么集成

 log4j.properties

loggerfactory

# admin控制台的安装

1. 下载dubbo的源码

2. 找到dubbo-admin

3. 修改webapp/WEB-INF/dubbo.properties

**dubbo.registry.address=zookeeper** **的集群地址**

 

控制中心是用来做服务治理的，比如控制服务的权重、服务的路由、。。。

 

# simple监控中心

Monitor也是一个dubbo服务，所以也会有端口和url

 

修改/conf目录下dubbo.properties /order-provider.xml

dubbo.registry.address=zookeeper://192.168.11.129:2181?backup=192.168.11.137:2181,192.168.11.138:2181,192.168.11.139:2181

 

监控服务的调用次数、调用关系、响应事件

 

# telnet命令

telnet  ip port 

 

ls、cd、pwd、clear、invoke

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 