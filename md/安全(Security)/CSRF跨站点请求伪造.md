摘自https://coderxing.gitbooks.io/architecture-evolution/di-san-pian-ff1a-bu-luo/641-web-an-quan-fang-fan/6412-csrf.html
## CSRF (Cross—Site Request Forgery)
即跨站点请求伪造，也被叫做 XSRF，和 XSS 一样也是一种比较常见的 web 攻击。CSRF攻击者会过过构造的第三方页面诱导受害者完成加载或者点击，利用受害者的权限，以其身份向合法网站发起恶意请求，通常用户发生状态改变的请求，比如虚拟货币的转账，账号信息修改，恶意发邮件等等，由于具有一定的隐蔽性，所以比较防范。 


## CSRF 的防范

目前主要有如下几种方式： 
* 添加 Referer 域名白名单  
* 令牌 (Token )验证  
为了更可靠的安全性，token 在使用之后一般置为失效，避免被窃取。  
* 二次验证：比如设计到交易的操作，可以更加严格一些。在用户提交时可以让用户输入验证码，或者再次输入交易密码，确保是用户的真实操作，而不是机器触发的。  
* 使用 SameSite Cookie 属性：SameSite 是在跨域请求时是否传输 Cookie 的一种约束，顾名思义在 SameSite 的限制下，Cookie 的传输必须在同一个域名下。SameSite 属性有两种限制模式 strict 和 lax，默认为 lax。
**目前只有 Chrome 和 Opera 浏览器支持 SameSite 属性。** 


对于JSON 劫持的方法其实和CSRF的防范方式是一脉相承的+

对 Referer 进行严格检查，通过白名单方式进行拦截过滤，并包括 Referer 为空的情况，不在白名单内的域名，或Referer为空均返回失败。
使用一次性TOKEN，和JSONP请求同时发送。
对callback参数进行校验，限制长度，并过滤掉符号字符。