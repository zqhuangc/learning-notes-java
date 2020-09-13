SPI(Service Provider Interface)  
是JDK内置的一种服务提供发现机制。 目前有不少框架用它来做服务的扩展发现， 简单来说，它就是一种动态替换发现的机制， 举个例子来说， 有个接口，想运行时动态的给它添加实现，你只需要添加一个实现，   

##SPI机制的约定

在META-INF/services/目录中创建以Service接口全限定名命名的**文件**，该文件内容为Service接口具体实现类的全限定名，文件编码必须为UTF-8。　　
使用ServiceLoader.load(Class class); 动态加载Service接口的实现类。　　
如SPI的实现类为jar，则需要将其放在当前程序的classpath下。　　
Service的具体实现类必须有一个不带参数的构造方法


java spi的具体约定为:当服务的提供者，提供了服务接口的一种实现之后，在jar包的META-INF/services/目录里同时创建一个以服务接口命名的文件。该文件里就是实现该服务接口的具体实现类。而当外部程序装配这个模块的时候，就能通过该jar包META-INF/services/里的配置文件找到具体的实现类名，并装载实例化，完成模块的注入。 基于这样一个约定就能很好的找到服务接口的实现类，而不需要再代码里制定。jdk提供服务实现查找的一个工具类：java.util.ServiceLoader
```java
//接口
public interface HelloInterface {
    public void sayHello();
}

//实现类 1
public class TextHello implements HelloInterface {
    @Override
    public void sayHello() {
    System.out.println("Text Hello.");
    }
    }
    public class ImageHello implements HelloInterface {
    @Override
    public void sayHello() {
    System.out.println("Image Hello");
    }
} 
//实现类 2
public class TextHello implements HelloInterface {
    @Override
    public void sayHello() {
    System.out.println("Text Hello.");
    }
    }
    public class ImageHello implements HelloInterface {
    @Override
    public void sayHello() {
    System.out.println("Image Hello");
    }
}

// SPI机制，客户端
public class SPIMain {
    public static void main(String[] args) {
        ServiceLoader<HelloInterface> loaders =
        ServiceLoader.load(HelloInterface.class);
        for (HelloInterface in : loaders) {
        in.sayHello();
        }
    }
}

```



## Dubbo 对 SPI 的扩展，看文档