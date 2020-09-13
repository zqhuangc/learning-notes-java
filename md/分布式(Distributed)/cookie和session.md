#### Session和Cookie的区别和联系以及Session的实现原理 :
1. session保存在服务器，客户端不知道其中的信息；cookie保存在客户端，服务器能够知道其中的信息。
2. session中保存的是对象，cookie中保存的是字符串。
3. session不能区分路径，同一个用户在访问一个网站期间，所有的session在任何一个地方都可以访问到。而cookie中如果设置了路径参数，那么同一个网站中不同路径下的cookie互相是访问不到的。
4. session需要借助cookie才能正常工作。如果客户端完全禁止cookie，session将失效。

http是无状态的协议，客户每次读取web页面时，服务器都打开新的会话，而且服务器也不会自动维护客户的上下文信息，那么要怎么才能实现网上商店中的购物车呢，session就是一种保存上下文信息的机制，它是针对每一个用户的，变量的值保存在服务器端，通过SessionID来区分不同的客户,session是以cookie或URL重写为基础的，默认使用cookie来实现，系统会创造一个名为JSESSIONID的输出cookie，我们叫做session cookie,以区别persistent cookies,也就是我们通常所说的cookie,注意session cookie是存储于**浏览器内存**中的，并不是写到硬盘上的，这也就是我们刚才看到的JSESSIONID，我们通常情是看不到JSESSIONID的，但是当我们把浏览器的cookie禁止后，web服务器会采用URL重写的方式传递Sessionid，我们就可以在地址栏看到 sessionid=KWJHUG6JJM65HS2K6之类的字符串。
明白了原理，我们就可以很容易的分辨出persistent cookies和session cookie的区别了，网上那些关于两者安全性的讨论也就一目了然了，session cookie针对某一次会话而言，会话结束session cookie也就随着消失了，而persistent cookie只是存在于客户端硬盘上的一段文本（通常是加密的），而且可能会遭到cookie欺骗以及针对cookie的跨站脚本攻击，自然不如 session cookie安全了。
通常session cookie是不能跨窗口使用的，当你新开了一个浏览器窗口进入相同页面时，系统会赋予你一个新的sessionid，这样我们信息共享的目的就达不到了，此时我们可以先把sessionid保存在persistent cookie中，然后在新窗口中读出来，就可以得到上一个窗口SessionID了，这样通过session cookie和persistent cookie的结合我们就实现了跨窗口的session tracking（会话跟踪）。
在一些web开发的书中，往往只是简单的把Session和cookie作为两种并列的http传送信息的方式，session cookies位于服务器端，persistent cookie位于客户端，可是session又是以cookie为基础的，明白的两者之间的联系和区别，我们就不难选择合适的技术来开发web service了。

## 总之： 
### 一、cookie机制和session机制的区别 
具体来说cookie机制采用的是在客户端保持状态的方案，而session机制采用的是在服务器端保持状态的方案。   
同时我们也看到，由于在服务器端保持状态的方案在客户端也需要保存一个标识，所以session机制可能需要借助于cookie机制来达到保存标识的目的，但实际上还有其他选择。 
### 二、会话cookie和持久cookie的区别 
如果不设置过期时间，则表示这个cookie生命周期为浏览器会话期间，只要关闭浏览器窗口，cookie就消失了。这种生命期为浏览会话期的cookie被称为会话cookie。会话cookie一般不保存在硬盘上而是保存在内存里。   
如果设置了过期时间，浏览器就会把cookie保存到硬盘上，关闭后再次打开浏览器，这些cookie依然有效直到超过设定的过期时间。   
存储在硬盘上的cookie可以在不同的浏览器进程间共享，比如两个IE窗口。而对于保存在内存的cookie，不同的浏览器有不同的处理方式。 
### 三、如何利用实现自动登录 
当用户在某个网站注册后，就会收到一个惟一用户ID的cookie。客户后来重新连接时，这个用户ID会自动返回，服务器对它进行检查，确定它是否为注册用户且选择了自动登录，从而使用户无需给出明确的用户名和密码，就可以访问服务器上的资源。 
### 四、如何根据用户的爱好定制站点 
网站可以使用cookie记录用户的意愿。对于简单的设置，网站可以直接将页面的设置存储在cookie中完成定制。然而对于更复杂的定制，网站只需仅将一个惟一的标识符发送给用户，由服务器端的数据库存储每个标识符对应的页面设置。 
### 五、cookie的发送 
1. 创建Cookie对象 
2. 设置最大时效 
3. 将Cookie放入到HTTP响应报头   

如果你创建了一个cookie，并将他发送到浏览器，默认情况下它是一个会话级别的cookie:存储在浏览器的内存中，用户退出浏览器之后被删除。如果你希望浏览器将该cookie存储在磁盘上，则需要使用maxAge，并给出一个以秒为单位的时间。将最大时效设为0则是命令浏览器删除该 cookie。

发送cookie需要使用HttpServletResponse的addCookie方法，将cookie插入到一个 Set-Cookie HTTP请求报头中。由于这个方法并不修改任何之前指定的Set-Cookie报头，而是创建新的报头，因此我们将这个方法称为是addCookie，而非setCookie。同样要记住响应报头必须在任何文档内容发送到客户端之前设置。

### 六、cookie的读取 
1. 调用request.getCookie  
要获取有浏览器发送来的cookie，需要调用HttpServletRequest的getCookies方法，这个调用返回Cookie对象的数组，对应由HTTP请求中Cookie报头输入的值。 
2. 对数组进行循环，调用每个cookie的getName方法，直到找到感兴趣的cookie为止   
cookie与你的主机(域)相关，而非你的servlet或JSP页面。因而，尽管你的servlet可能只发送了单个cookie，你也可能会得到许多不相关的cookie。 
例如： 
```java
　　String cookieName = “userID”; 
Cookie cookies［］ = request.getCookies(); 
if (cookies!=null){ 
for(int i=0;i 
Cookie cookie = cookies［i］; 
if (cookieName.equals(cookie.getName())){ 
doSomethingWith(cookie.getValue()); 
} 
} 
} 
```
### 七、如何使用cookie检测初访者 
A. 调用HttpServletRequest.getCookies()获取Cookie数组 
B. 在循环中检索指定名字的cookie是否存在以及对应的值是否正确 
C. 如果是则退出循环并设置区别标识 
D. 根据区别标识判断用户是否为初访者从而进行不同的操作 
### 八、使用cookie检测初访者的常见错误 
不能仅仅因为cookie数组中不存在在特定的数据项就认为用户是个初访者。如果cookie数组为null，客户可能是一个初访者，也可能是由于用户将cookie删除或禁用造成的结果。   
但是，如果数组非null,也不过是显示客户曾经到过你的网站或域，并不能说明他们曾经访问过你的servlet。其它servlet、JSP页面以及非Java Web应用都可以设置cookie，依据路径的设置，其中的任何cookie都有可能返回给用户的浏览器。   
正确的做法是判断cookie数组是否为空且是否存在指定的Cookie对象且值正确。 
### 九、使用cookie属性的注意问题 
属性是从服务器发送到浏览器的报头的一部分；但它们不属于由浏览器返回给服务器的报头。  　 
因此除了名称和值之外，cookie属性只适用于从服务器输出到客户端的cookie；服务器端来自于浏览器的cookie并没有设置这些属性。  　 
因而不要期望通过request.getCookies得到的cookie中可以使用这个属性。这意味着，你不能仅仅通过设置cookie的最大时效，发出它，在随后的输入数组中查找适当的cookie,读取它的值，修改它并将它存回Cookie，从而实现不断改变的cookie值。 
### 十、如何使用cookie记录各个用户的访问计数 
1. 获取cookie数组中专门用于统计用户访问次数的cookie的值 
2. 将值转换成int型 
3. 将值加1并用原来的名称重新创建一个Cookie对象 
4. 重新设置最大时效 
5. 将新的cookie输出 
### 十一、session在不同环境下的不同含义 
session，中文经常翻译为会话，其本来的含义是指有始有终的一系列动作/消息，比如打电话是从拿起电话拨号到挂断电话这中间的一系列过程可以称之为一个session。   
然而当session一词与网络协议相关联时，它又往往隐含了“面向连接”和/或“保持状态”这样两个含义。   
session在Web开发环境下的语义又有了新的扩展，它的含义是指一类用来在客户端与服务器端之间保持状态的解决方案。有时候Session也用来指这种解决方案的存储结构。   
### 十二、session的机制 
session机制是一种服务器端的机制，服务器使用一种类似于散列表的结构(也可能就是使用散列表)来保存信息。   
但程序需要为某个客户端的请求创建一个session的时候，服务器首先检查这个客户端的请求里是否包含了一个session标识－称为session id,如果已经包含一个session id则说明以前已经为此客户创建过session，服务器就按照session id把这个session检索出来使用(如果检索不到，可能会新建一个，这种情况可能出现在服务端已经删除了该用户对应的session对象，但用户人为地在请求的URL后面附加上一个JSESSION的参数)。   
如果客户请求不包含session id，则为此客户创建一个session并且生成一个与此session相关联的session id，这个session id将在本次响应中返回给客户端保存。   
### 十三、保存session id的几种方式 
A．保存session id的方式可以采用cookie，这样在交互过程中浏览器可以自动的按照规则把这个标识发送给服务器。   
B．由于cookie可以被人为的禁止，必须有其它的机制以便在cookie被禁止时仍然能够把session id传递回服务器，经常采用的一种技术叫做URL重写，就是把session id附加在URL路径的后面，附加的方式也有两种，一种是作为URL路径的附加信息，另一种是作为查询字符串附加在URL后面。网络在整个交互过程中始终保持状态，就必须在每个客户端可能请求的路径后面都包含这个session id。   
C．另一种技术叫做表单隐藏字段。就是服务器会自动修改表单，添加一个隐藏字段，以便在表单提交时能够把session id传递回服务器。   
### 十四、session什么时候被创建 
一个常见的错误是以为session在有客户端访问时就被创建，然而事实是直到某server端程序(如Servlet)调用HttpServletRequest.getSession(true)这样的语句时才会被创建。 
### 十五、session何时被删除 
session在下列情况下被删除： 
A．程序调用HttpSession.invalidate() 
B．距离上一次收到客户端发送的session id时间间隔超过了session的最大有效时间 
C．服务器进程被停止 
再次注意关闭浏览器只会使存储在客户端浏览器内存中的session cookie失效，不会使服务器端的session对象失效。