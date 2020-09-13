# Java Beans 内省机制

### 认识 Java Beans

贫血模型，状态 Bean， JSF，反射（如何优雅类型转换）

Beans 



Enterprise Java Beans 规范

![](https://ws1.sinaimg.cn/large/006xzusPly1g5ita2unfxj30k10gw3zu.jpg)

1. 传递了一个字符串（Text）类型
2. 传递值转化成对应的数据类型，并且赋值
3. 对应事件发生（不同属性对应的事件处理可能不同）

## 反射

Java Bean：

* Class 信息（单一职责）
  * 构造器（Constructor）
  * 方法（Method）
  * 字段（Field）（可能没有）

ClassLoader 同一个类只加载一次

## 内省

多实例 

java BeanInfo：

* Bean 描述符（BeanDescriptor）
* 属性描述符（PropertyDescriptor）
  - PropertyEditor#setAsText（参考别的实现）
    - Spring 内部实现可能有问题
  - PropertyEditorSupport#firePropertyChange
* 方法描述符（MethodDescriptor）

idea 提示用到了内省（Introspection）

get，set不一定成对出现（至少有一个，判断是否 public），如 Object



类（模板）没有状态，实例（Bean）有状态（每一个状态信息都可能不同）

![](https://ws1.sinaimg.cn/large/006xzusPly1g5icis9uwzj30oo0czwld.jpg)



### Java Beans 事件监听

* 属性变化监听器（PropertyChangeListener）
* 属性变化事件（PropertyChangeEvent）
  - 事件源（Source）
  - 属性名称（PropertyName）
  - 变化前值（OldValue）
  - 变化后值（NewValue）

### Spring Bean 属性处理

* 属性修改器（PropertyEditor）
* 属性修改器注册（PropertyEditorRegistry）
* PropertyEditor注册器（PropertyEditorRegistrar）
* 自定义PropertyEditor配置器（CustomEditorConfigurer）

### 内省机制在 Spring 中的应用

@ConfigurationProperties   配置与类如何对应的？

`Introspector#getBeanInfo`



#### Introspector

Java Beans Introspector是一个类，位置在Java.bean.Introspector，这个类的用途是发现java类是否符合javaBean规范，也就是这个类是不是javabean。具体用法可以参照jdk文档；

上面的意思就是，如果有的框架或者程序用到了Java Beans Introspector了，那么就启用了一个系统级别的缓存，这个缓存会存放一些曾加载并分析过的javabean的引用，当web服务器关闭的时候，由于这个缓存中存放着这些javabean的引用，所以垃圾回收器不能对web容器中的javaBean对象进行回收，导致内存越来越大。

spring提供的org.springframework.web.util.IntrospectorCleanupListener就解决了这个问题，他会在web服务器停止的时候，清理一下这个Introspector缓存。使那些javabean能被垃圾回收器正确回收。

spring 不会出现这种问题，因为spring在加载并分析完一个类之后会马上刷新JavaBeans Introspector缓存，这样就保证了spring不会出现这种内存泄漏的问题。

但是有很多程序和框架在使用了JavaBeans Introspector之后，都没有进行清理工作，比如quartz、struts；解决办法很简单，就是上面的那个配置。

用法很简单，就是在web.xml中加入:   

```xml
<listener>    
    <listener-class>org.springframework.web.util.IntrospectorCleanupListener</listener-class>    
</listener>  
```









# Java 资源管理

+ 文件资源
  - XML 文件
  - Properties 文件
+ 网络资源
  - HTTP
  - FTP
+ 类路径资源



Path，资源路径

> System.getProperties("ParamName");

移植一致性

* maven  

  ```xml
  <build>
      <resourses>
          <resource>
              <directory>${basedir}/.../...</directory>
              <includes>
                  <include></include>
              </includes>
              <excludes>
                  <exclude>*.properties</exclude>
              </excludes>
          </resource>
      </resourses>
  </build>
  ```


#### Java URL 协议扩展

* URL#openStream

* URLConnection

* URLStreamHandler

  > sun.net.www.protocol.${protocol}.handler:
  >
  > sun.net.www.protocol.http.handler
  >
  > sun.net.www.protocol.file.handler

* URLStreamHandlerFactory

  ClassLoader 读取资源

  Thread.getCurrentThread().getContextClassLoader

**Url 读取资源的方式**

* 如何扩展到自定义协议？

> 创建相同包 sun.net.www.protocol
>
> 自定义协议
>
> 参考现有协议实现方式实现自定义 Handler，UrlConnection

![](https://ws1.sinaimg.cn/large/006xzusPly1g5j4qsq8wkj30g90a076v.jpg)



# Spring资源管理

* Resource

  不同实现类

* ResourceLoader

  默认实现，getResource，addProtocolResolver

* ProtocolResolver（since 4.3）



可类比 Url 的实现方式



