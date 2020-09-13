使用非对称加密协商加密算法
使用对称加密方式传输数据
使用第三方机构签发的证书，来加密公钥，用于公钥的安全传输、防止被中间人串改。 



## 八大免费SSL证书
一、Let's Encrypt  

1、Let's Encrypt是国外一个公共的免费SSL项目，由 Linux 基金会托管，它的来头不小，由Mozilla、思科、Akamai、IdenTrust和EFF等组织发起，目的就是向网站自动签发和管理免费证书，以便加速互联网由HTTP过渡到HTTPS。

2、Let's Encrypt安装部署简单、方便，目前Cpanel、Oneinstack等面板都已经集成了Let's Encrypt一键申请安装，网上也有不少的利用Let's Encrypt开源的特性制作的在线免费SSL证书申请网站，总之Let's Encrypt得到大家的认可。  
免费SSL证书Let’s Encrypt安装使用教程:Apache和Nginx配置SSL

二、StartSSL  
三、COMODO PositiveSSL  
四、CloudFlare SSL  
五、Wosign沃通SSL  
六、腾讯云DV SSL 证书  
七、loovit.net AlphaSSL  
八、360网站卫士、百度云加速免费SSL  
上面介绍的八大免费SSL证书，要说最让人放心的当属Let's Encrypt了，效果也可以参考部落网站。其它七个免费SSL证书，建议大家谨慎使用，对于一些重要的网站还是建议你直接购买SSL证书：Namecheap SSL一年就十美元





### **HTTPS和HTTP的区别**

超文本传输协议HTTP协议被用于在Web浏览器和网站服务器之间传递信息。HTTP协议以明文方式发送内容，不提供任何方式的数据加密，如果攻击者截取了Web浏览器和网站服务器之间的传输报文，就可以直接读懂其中的信息，因此HTTP协议不适合传输一些敏感信息，比如信用卡号、密码等。

为了解决HTTP协议的这一缺陷，需要使用另一种协议：安全套接字层超文本传输协议HTTPS。为了数据传输的安全，HTTPS在HTTP的基础上加入了SSL协议，SSL依靠证书来验证服务器的身份，并为浏览器和服务器之间的通信加密。

HTTPS和HTTP的区别主要为以下四点：

一、https协议需要到ca申请证书，一般免费证书很少，需要交费。

二、http是[超文本传输协议](https://baike.baidu.com/item/%E8%B6%85%E6%96%87%E6%9C%AC%E4%BC%A0%E8%BE%93%E5%8D%8F%E8%AE%AE)，信息是明文传输，https 则是具有[安全性](https://baike.baidu.com/item/%E5%AE%89%E5%85%A8%E6%80%A7)的[ssl](https://baike.baidu.com/item/ssl)加密传输协议。

三、http和https使用的是完全不同的连接方式，用的端口也不一样，前者是80，后者是443。

四、http的连接很简单，是无状态的；HTTPS协议是由SSL+HTTP协议构建的可进行加密传输、身份认证的[网络协议](https://baike.baidu.com/item/%E7%BD%91%E7%BB%9C%E5%8D%8F%E8%AE%AE)，比http协议安全。




  