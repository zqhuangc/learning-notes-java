OSGI的解释就是Open Service Gateway Initiative，直译过来就是“开放的服务入口(网关)的初始化”

Jetty

OSGI(Open Services Gateway Initiative)，或者通俗点说JAVA动态模块系统，定义了一套模块应用开发的框架。OSGI容器实现方案如Knopflerfish, Equinox, and Apache Felix允许你把你的应用分成多个功能模块，这样通过依赖管理这些功能会更加方便。
和Servlet和EJB规范类似，OSGI规范包含两大块：一个OSGI容器需要实现的服务集合；一种OSGI容器和应用之间通信的机制。开发OSGI平台意味着你需要使用OSGI API编写你的应用，然后将其部署到OSGI容器中。从开发者的视角来看，OSGI提供以下优势：

你可以动态地安装、卸载、启动、停止不同的应用模块，而不需要重启容器。
你的应用可以在同一时刻跑多个同一个模块的实例。
OSGI在SOA领域提供成熟的解决方案，包括嵌入式，移动设备和富客户端应用等。

OK，你已经有个Servlet容器来做web 应用，有了EJB容器来做事务处理，你可能在想为什么你还需要一个新的容器？简单点说，OSGI容器被设计专门用来开发可分解为功能模块的复杂的Java应用。

### 开源OSGI容器

从企业应用开发者的角度看，OSGI容器侵入性非常小，你可以方便地将其嵌入一个企业应用。举个例子来说，假设你在开发一个复杂的web应用。你希望将这个应用分解成多个功能模块。一个View层模块，一个Model层模块，一个DAO模块。使用嵌入式OSGI容器来跨依赖地管理这些模块可以让你随时更新你的DAO模块却不需要重启你的服务器。
只要你的应用完全符合OSGI规范，它就可以在所有符合OSGI规范的容器内运行。现在，有三种流行的开源OSGI容器：

Equinox是OSGI Service Platform Release 4的一个实现。是Eclipse 模块化运行时的核心。
Knopflerfish另一个选择。
Apache Felix是Apache软件基金会赞助的一个OSGI容器

### OSGI是什么？

1. OSGI是一套规范

 2. OSGI是一个容器，供Bundle生存的容器。

### OSGI主要干什么？

* OSGi主要做三件事：
  (1)Bundle管理；
  (2)Bundle的动态加载；
  (3)保证Bundle间使用服务机制来交互。