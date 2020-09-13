在TCP/IP协议栈中的位置
 HTTP的请求响应模型  
 工作流程  
 头域  
 * host头域  

 Host头域指定请求资源的Intenet主机和端口号，必须表示请求url的原始服务器或网关的位置。HTTP/1.1请求必须包含主机头域，否则系统会以400状态码返回。  

 * Referer头域  

 Referer头域允许客户端指定请求uri的源资源地址，这可以允许服务器生成回退链表，可用来登陆、优化cache等。他也允许废除的或错误的连接由于维护的目的被追踪。如果请求的uri没有自己的uri地址，Referer不能被发送。如果指定的是部分uri地址，则此地址应该是一个相对地址。 
 
 * User-Agent头域  

 User-Agent头域的内容包含发出请求的用户信息。  

 * Cache-Control头域  

 Cache-Control指定请求和响应遵循的缓存机制。在请求消息或响应消息中设置Cache-Control并不会修改另一个消息处理过程中的缓存处理过程。请求时的缓存指令包括no-cache、no-store、max-age、max-stale、min-fresh、only-if-cached，响应消息中的指令包括public、private、no-cache、no-store、no-transform、must-revalidate、proxy-revalidate、max-age。  
 Date头域  

##  HTTP的几个重要概念  
连接：Connection  
消息：Message  
请求：Request  
响应：Response  
资源：Resource  
实体：Entity  
客户机：Client  
用户代理：UserAgent  
服务器：Server  
源服务器：Originserver  
代理：Proxy  
网关：Gateway  
通道：Tunnel  
缓存：Cache