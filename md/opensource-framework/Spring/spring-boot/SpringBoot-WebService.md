## 使用jdk自带的工具*xjc*生成 将schema文件生成对应的*java*文件

xcj命令有schema文件生成java实体类

* 使用方法
  * xjc  fileName.xsd -d 生成java实体类的目录(默认为当前路径) -p 生成的包名

* 指定编码
  * java -Dfile.encoding=UTF-8 -cp D:\java\java1.8\lib\tools.jar com.sun.tools.internal.xjc.Driver -
    p com.melody.domain user.xsd



### 服务端



### 客户端





PayloadRootAnnotationMethodEndpointMapping



```java
@PayloadRoot(namespace = "http://mycompany.com/hr/schemas", localPart = "HolidayRequest")
```



![](https://docs.spring.io/spring-ws/docs/3.0.4.RELEASE/reference/images/sequence.png)