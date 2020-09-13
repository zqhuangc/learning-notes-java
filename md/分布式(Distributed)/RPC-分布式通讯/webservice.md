wsdl

soap

RPC 包含的要素

webservice

​        协议层：tcp/ip

应用层： http协议

SOAP： http+xml 



**Wsimport工具说明：**

wsimport是jdk自带的,可以根据wsdl文档生成客户端调用代码的工具.当然,无论服务器端的WebService是用什么语言写的,都将在客户端生成Java代码.服务器端用什么写的并不重要.

wsimport.exe位于JAVA_HOME\bin目录下.

**常用参数为:**

• -d<目录> - 将生成.class文件。默认参数。

• -s<目录> - 将生成.java文件。

• -p<生成的新包名> -将生成的类，放于指定的包下。

(wsdlurl) - http://server:port/service?wsdl，必须的参数



>  获取 webservise 代码    
>
> publish 的 url： http://localhost:8080/echo
>
> wsimport -s . http://localhost:8080/echo?wsdl

# 分布式通信框架-webservice分析

## 什么是webservice

webservice也可以叫xml web service webservice, 轻量级的独立的通讯技术

1. 基于web的服务：服务端提供的服务接口让客户端访问

2. 跨平台、跨语言的整合方案

## 为什么要使用webservice

跨语言调用的解决方案

 

## 什么时候要去使用webservice

电商平台，订单的物流状态。 

 .net实现的webservice服务接口

## webservice中的一些概念

### WSDL(web service definition language  webservice 定义语言)

webservice服务需要通过wsdl文件来说明自己有什么服务可以对外调用。并且有哪些方法、方法里面有哪些参数

wsdl基于XML（可扩展标记语言）去定义的

1. 对应一个.wsdl的文件类型

2. 定义了webservice的服务器端和客户端应用进行交互的传递数据和响应数据格式和方式

3. 一个webservice对应唯一一个wsdl文档

### SOAP（simple object access protocal简单对象访问协议）

http+xml 

webservice通过http协议发送和接收请求时， 发送的内容（请求报文）和接收的内容（响应报文）都是采用xml格式进行封装  

这些特定的HTTP消息头和XML内容格式就是SOAP协议  

1. 一种简单、基于HTTP和XML的协议

2. soap消息：请求和响应消息

3. http+xml报文

### SEI（webservice endpoint interface webservice的终端接口）

webservice服务端用来处理请求的接口，也就是发布出去的接口。

 

## 开发一个webservice的实例



## 分析WSDL文档



 

### Types标签

定义整服务端的数据报文

#### Schema标签



```xml
<sayHello>

   <arg0>String</arg0>

</sayHello>



<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:web="http://webservice.mic.vip.gupao.com/">

   <soapenv:Header/>   
   <soapenv:Body>
      <web:sayHello>
         <!--Optional:-->
         <arg0>?</arg0>
      </web:sayHello>
   </soapenv:Body>
</soapenv:Envelope>

 

<sayHelloResponse>
   <return>string</return>
</sayHelloResponse>

 

<S:Envelope xmlns:S="http://schemas.xmlsoap.org/soap/envelope/">
   <S:Body>
      <ns2:sayHelloResponse xmlns:ns2="http://webservice.mic.vip.gupao.com/">
         <return>Hello ,Mic,I'am 菲菲</return>
      </ns2:sayHelloResponse>
   </S:Body>
</S:Envelope>
```





 

### Message

![img](file:///C:\Users\lenovo\AppData\Local\Temp\msohtmlclip1\01\clip_image003.png)

定义了在通信中使用的消息的数据结构

 

### PortType

定义服务器端的SEI

input/output表示输入/输出数据

### Binding标签



\1.   type属性： 引用porttype

<soap:binding style=”document”> 

\2.   operation : 指定实现方法

\3.   input/output 表示输入和输出的数据类型

 

### Service标签



service： 服务器端的一个webservice的容器

name属性： 指定客户端的容器类

 

address： 当前webservice的请求地址



 

 



 

## Axis/Axis2

apache开源的webservice工具

### CXF

Celtix+Xfire 。 用的很广泛，因为集成到了spring

### Xfire

高性能的Webservice

HTTP+JSON (新的webservice)

HTTP+XML

## spring cxf+REST实现一个webservice服务

 

springmvc+REST实现的新webservice 

 

linux： centos7;  vm可以copy 设置一个备份点

jdk、tomcat

 

RMI、 http协议/https、webservice、 TCP协议、UDP协议、 

socket编程、bio /nio模型、分布式架构、集群、架构演进过程



一个公开的接口，其他系统不管你是用什么语言来编写的都可以调用这个接口，并可以返回相应的数据给你

jaxws

webService三要素(SOAP, WSDL (Web Services Description Language),UDDI( Universal Description Discovery and Integration ))

## SOAP 简单对象访问协议   

SOAP（Simple Object Access Protocol）
基于已经广泛使用的两个协议：HTTP和XML

## WSDL

WSDL（Web Services Description Language），网络服务描述语言，是一个用来描述Web服务和说明如何与Web服务通信的XML（标准通用标记语言的子集）语言。为用户提供详细的接口说明书。

WSDL文档是一个遵循WSDL XML模式的XML文档（文档实例）；类似于：SOAP文档是一个遵循SOAP XML模式的XML文档（文档实例）；  
一个WSDL文档的根元素是definitions元素，WSDL文档包含7个重要的元素：types, import, message, portType, operations, binding和service元素。  
 Messaging Exchange Patterns(MEP)
 Web服务中使用了四种消息交换模式，即请求/响应、单向、通知以及恳求/响应模式。大多数基于WSDL的web服务使用请求/响应和单向两种模式。

## CXF(Apache CXF = Celtix + XFire)

用 Frontend 编程 API 来构建和开发 Services ，像 JAX-WS 。这些 Services 可以支持多种协议，比如：SOAP、XML/HTTP、RESTful HTTP 或者 CORBA ，并且可以在多种传输协议上运行



泛型技术是给编译器使用的技术,用于编译时期。确保了类型的安全。

  运行时，会将泛型去掉，生成的class文件中是不带泛型的,这个称为泛型的擦除。
  为什么擦除呢？因为为了兼容运行的类加载器。
  泛型的补偿：在类加载器原有基础上，编写一个补偿程序。在运行时，通过反射，
  获取元素的类型进行转换动作。不用使用者在强制转换了。







### 代码

##注解方式

```java
public class WebServiceBootstrap {

    public static void main(String[] args) {

        Endpoint.publish("http://localhost:8080/echo",new EchoServiceImpl());

        System.out.println("publish success");
    }
}

@WebService //SE和SEI的实现类
public interface IEchoService {

    @WebMethod //SEI中的方法
    String echo(String message);
}

@WebService
public class EchoServiceImpl implements IEchoService {

    public String echo(String message) {
        System.out.println("call echo()");
        return "echo message：" + message;
    }
}



通过 wsimport 获取代码后
public class WebServiceTest {

    public static void main(String[] args) {
        EchoServiceImpl echoService = new EchoServiceImplService().getEchoServiceImplPort();
        echoService.echo("this is a echo service");
    }
}
```



 

 

 

 

 

 

 

 

 

 

 

 

