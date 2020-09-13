### XML解析方式
Java 内置解析 XML API: DOM、SAX
XML 解析框架 JDOM
XML 解析框架 DOM4J
XML 解析框架 XStream
### JavaWeb开发中跨域问题的解决方案
没有权限
1. 设置 document.domain（一级域名相同的情况之下）
2. HTML标签中 src 属性，只支持get请求，允许跨域
3. <script src=""> JSON格式 eval
4. iframe 之间交互 window.postMessage(字符串限长255个)
5. 服务器后台做文章：CORS（安全沙箱）
   Acess-Control-Allow-Origin:*;