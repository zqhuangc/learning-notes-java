

# Servlet

### 域对象

1. Request
  request是表示一个请求，只要发出一个请求就会创建一个request，它的作用域：仅在当前请求中有效。
2. Session
  服务器会为每个会话创建一个session对象，所以session中的数据可供当前会话中所有servlet共享。
3. Application（ServletContext）
  作用范围：所有的用户都可以取得此信息，此信息在整个服务器上被保留。Application属性范围值，只要设置一次，则所有的网页窗口都可以取得数据。ServletContext在服务器启动时创建，在服务器关闭时销毁，一个JavaWeb应用只创建一个ServletContext对象，所有的客户端在访问服务器时都共享同一个ServletContext对象;ServletContext对象一般用于在多个客户端间共享数据时使用;
4. Page

### 过滤器



### 监听器





# JSP

### 一. 四大域对象

1. PageContext ：页面范围的数据

2. ServletRequest：请求范围的数据

3. HttpSession：会话范围的数据

4. ServletContext：应用范围的数据

### 二. 9个内置对象

1.request对象

   request 对象是 javax.servlet.httpServletRequest类型的对象。 该对象代表了客户端的请求信息，主要用于接受通过HTTP协议传送到服务器的数据。

 （包括头信息. 系统信息. 请求方式以及请求参数等）。request对象的作用域为一次请求。

2.response对象

   response 代表的是对客户端的响应，主要是将JSP容器处理过的对象传回到客户端。response对象也具有作用域，它只在JSP页面内有效。

3.session对象

   session 对象是由服务器自动创建的与用户请求相关的对象。服务器为每个用户都生成一个session对象，用于保存该用户的信息，跟踪用户的操作状态。

   session对象内部使用Map类来保存数据，因此保存数据的格式为 “Key/value”。 session对象的value可以使复杂的对象类型，而不仅仅局限于字符串类型。

4.application对象

   application 对象可将信息保存在服务器中，直到服务器关闭，否则application对象中保存的信息会在整个应用中都有效。

   与session对象相比，application对象生命周期更长，类似于系统的“全局变量”。

5.out 对象

   out 对象用于在Web浏览器内输出信息，并且管理应用服务器上的输出缓冲区。

   在使用 out 对象输出数据时，可以对数据缓冲区进行操作，及时清除缓冲区中的残余数据，为其他的输出让出缓冲空间。待数据输出完毕后，要及时关闭输出流。

6.pageContext 对象

   pageContext 对象的作用是取得任何范围的参数，通过它可以获取 JSP页面的out. request. reponse. session. application 等对象。

   pageContext对象的创建和初始化都是由容器来完成的，在JSP页面中可以直接使用 pageContext对象。

7.config 对象

   config 对象的主要作用是取得服务器的配置信息。通过 pageConext对象的 getServletConfig() 方法可以获取一个config对象。

   当一个Servlet 初始化时，容器把某些信息通过 config对象传递给这个 Servlet。

   开发者可以在web.xml 文件中为应用程序环境中的Servlet程序和JSP页面提供初始化参数。

8.page 对象

   page 对象代表JSP本身，只有在JSP页面内才是合法的。 page隐含对象本质上包含当前 Servlet接口引用的变量，类似于Java编程中的 this 指针。

9. exception 对象

   exception 对象的作用是显示异常信息，只有在包含 isErrorPage="true" 的页面中才可以被使用，在一般的JSP页面中使用该对象将无法编译JSP文件。

   excepation对象和Java的所有对象一样，都具有系统提供的继承结构。exception 对象几乎定义了所有异常情况。

   如果在JSP页面中出现没有捕获到的异常，就会生成 exception 对象，并把 exception 对象传送到在page指令中设定的错误页面中，然后在错误页面中处理相应的 exception 对象。



### 三. EL表达式及其11个隐式对象

#### 请求参数

1. param 包含所有的参数的Map可以获取参数返回String
2. paramValues 包含所有参数的Map,可以获取参数的数组返回String[]

头信息

   3. header 包含所有的头信息的Map。可以获取头信息返回String

   4. headerValues 包含所有的头信息的Map。可以获取头信息数组返回String[]

Cookie

   5. cookie包含所有cookie的Map,key为Cookie的name属性值

初始化参数

   6. iniParam 包含所有的初始化参数的Map，可以获取初始化的参数.

#### 作用域

1. pageScope 包含page作用域内的Map.
2. requestScope 包含request作用域内的Map
3. sessionScope 包含session作用域内的Map
4. applicationScope 包含application作用域内的Map
5. pageContext 包含页面内的变量的Map，包含request，response，page，application，config等所有的隐藏对象



### EL表达式

<%= %>