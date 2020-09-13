# 高性能反向代理服务器——Nginx



# 关于反向代理和正向代理

## 正向代理

正向代理的对象是客户端

## 反向代理

反向代理代理的是服务端

 

# 常用Web服务器介绍

apache、Nginx、tomcat、weblogic、iis、jboss、websphere、jetty、netty、lighttpd、glassfish、resin



#### 反向代理

LVS、Apache、Nginx、HAProxy、OpenResty

## Apache

Apache仍然是时长占用量最高的web服务器，据最新数据统计，市场占有率目前是50%左右。主要优势在于一个是比较早出现的一个Http静态资源服务器，同时又是开源的。所以在技术上的支持以及市面上的各种解决方案都比较成熟。Apache支持的模块非常丰富。

## Lighttpd

Lighttpd其设计目标是提供一个专门针对高性能网站、安全、快速、兼容性好并且灵活的web server环境。特点是：内存开销低、CPU占用率低、性能好、模块丰富。Lighttpd跟Nginx一样，是一款轻量级的Web服务器。跟Nginx的定位类似

## Tomcat

Tomcat大家都比较熟悉，是一个开源的JSP Servlet容器。

## Nginx

Nginx是俄罗斯人编写的一款高性能的HTTP和反向代理服务器，在高连接并发的情况下，它能够支持高达50000个并发连接数的响应，但是内存、CPU等系统资源消耗却很低，运行很稳定。目前Nginx在国内很多大型企业都有应用，据最新统计，Nginx的市场占有率已经到33%左右了。而Apache的市场占有率虽然仍然是最高的，但是是呈下降趋势。而Nginx的势头很明显。选择Nginx的理由也很简单：第一，它可以支持5W高并发连接；第二，内存消耗少；第三，成本低，如果采用F5、NetScaler等硬件负载均衡设备的话，需要大几十万。而Nginx是开源的，可以免费使用并且能用于商业用途

## Nginx服务器、Apache Http Server、Tomcat之间的关系

HTTP服务器本质上也是一种应用程序——它通常运行在服务器之上，绑定服务器的IP地址并监听某一个tcp端口来接收并处理HTTP请求，这样客户端（一般来说是IE, Firefox，Chrome这样的浏览器）就能够通过HTTP协议来获取服务器上的网页（HTML格式）、文档（PDF格式）、音频（MP4格式）、视频（MOV格式）等等资源



Apache Tomcat是Apache基金会下的一款开源项目，Tomcat能够动态的生成资源并返回到客户端。

Apache HTTP server 和Nginx都能够将某一个文本文件的内容通过HTTP协议返回到客户端，但是这些文本文件的内容是固定的，也就是说什么情况下访问该文本的内容都是完全一样的，这样的资源我们称之为静态资源。动态资源则相反，不同时间、不同客户端所得到的内容是不同的 ； （虽然apache和nginx本身不支持动态页面，但是他们可以集成模块来支持，比如PHP、Python）

如果想要使用java程序来动态生成资源内容，使用apache server和nginx这一类的http服务器是基本做不到。而Java Servlet技术以及衍生出来的（jsp）Java Server Pages技术可以让Java程序也具有处理HTTP请求并且返回内容的能力，而Apache Tomcat正是支持运行Servlet/JSP应用程序的容器



Tomcat运行在JVM之上，它和HTTP服务器一样，绑定IP地址并监听TCP端口，同时还包含以下职责：

- 管理Servlet程序的生命周期
- 将URL映射到指定的Servlet进行处理
- 与Servlet程序合作处理HTTP请求——根据HTTP请求生成HttpServletResponse对象并传递给Servlet进行处理，将Servlet中的HttpServletResponse对象生成的内容返回给浏览器

虽然Tomcat也可以认为是HTTP服务器，但通常它仍然会和Nginx配合在一起使用：

- 动静态资源分离——运用Nginx的反向代理功能分发请求：所有动态资源的请求交给Tomcat，而静态资源的请求（例如图片、视频、CSS、JavaScript文件等）则直接由Nginx返回到浏览器，这样能大大减轻Tomcat的压力。
- 负载均衡，当业务压力增大时，可能一个Tomcat的实例不足以处理，那么这时可以启动多个Tomcat实例进行水平扩展，而Nginx的负载均衡功能可以把请求通过算法分发到各个不同的实例进行处理

## Apache HTTP Server和Nginx的关系

Apache Http Server是使用比较广泛也是资格最老的web服务器，是Apache基金会下第一个开源的WEB服务器。在Nginx出现之前，大部分企业使用的都是Apache。

在互联网发展初期，流量不是特别大的时候，使用Apache完全满足需求。但是随着互联网的飞速发展，网站的流量以指数及增长，这个时候除了提升硬件性能以外，Apache Http server也开始遇到瓶颈了，于是这个时候Nginx的出现，就是为了解决大型网站高并发设计的，所以对于高并发来说，Nginx有先天的优势。因此Nginx也在慢慢取代Apache Http server。 而Nginx另一个强大的功能就是反向代理，现在大型网站分工详细，哪些服务器处理数据流，哪些处理静态文件，这些谁指挥，一般都是用nginx反向代理到内网服务器，这样就起到了负载均衡分流的作用。再次nginx高度模块化的设计，编写模块相对简单。

 

 

# Nginx的安装

## nginx安装

1. tar -zxvf 安装包

2. ./configure --prefix=/mic/data/program/nginx   默认安装到/usr/local/nginx  （模块安装）

3. make & make install

## 启动停止

./nginx -c /mic/data/program/nginx/conf/nginx.conf 启动nginx   -c表示指定nginx.conf的文件。如果不指定，默认为NGINX_HOME/conf/nginx.conf



* 发送信号的方式：
  kill -QUIT  进程号
  kil -TERM  进程号   相当于 kill -9
  停止nginx

* 命令方式：
  ./nginx -s stop  停止
  ./nginx -s quit   退出
  ./nginx -s reload  重新加载nginx.conf

 

安装过程中可能会出现的问题

缺少pcre的依赖   (正则相关)

缺少openssl的依赖 （加密相关）

yum install pcre-devel

yum install openssl-devel

yum install zlib-devel

# Nginx核心配置分析

**ngx_http_core_module** 

nginx的核心配置文件,主要包括三个段

Main、 Event 、 Http

# 虚拟主机配置

### 基于域名的虚拟主机

**修改windows/system32/drivers/etc/hosts**

```
ip 域名
```



**修改 nginx.conf 文件，在 http 段中增加如下内容**

```con
server {
    listen       80;
    server_name  域名;


    location / {
        root   html/domain;     # 文件目录
        index  index.html index.htm;  # 目录下默认首页
    }
}

```

### 基于端口的虚拟主机

```conf
server {
    listen       8080;
    server_name  localhost;


    location / {
        root   html/port;
        index  index.html;
    }
}

```



### 基于ip的虚拟主机



# Nginx的日志配置

通过 access_log 进行日志记录

nginx 中有两条是配置日志的：一条是 log_format 来设置日志格式 ； 另外一条是 access_log

 ```conf
#log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
#                  '$status $body_bytes_sent "$http_referer" '
#                  '"$http_user_agent" "$http_x_forwarded_for"';
# nginx 内置关键字
 ```

* access_log  格式

\#access_log  logs/access.log  main;

log 声明   路径及文件名 日志标识（log_format）

\#access_log off; 

* nginx日志切割

crontab

mv access.log access.log.20171206

kill -USR1 Nginx 主进程号  让nginx重新生成一个日志文件access.log

# location的语法和匹配规则

> location [~|=|^~|~*] /uri {
>
> }

## location的匹配规则

[参考理解](http://www.cnblogs.com/aiguang/p/3571207.html)

**url 内部重定向问题**

* 精准匹配
  location=/uri{} ，注意  location=/{} 的情况
  优先级最高的匹配规则 

* 一般匹配
  location /uri{}，
  普通匹配的优先级要高于正则匹配
  如果存在多个相同的前缀的一般匹配，那么最终会按照最大长度来做匹配

  注：location /{}  与 location =/{} 匹配规则问题

* 正则匹配
  正则 location 内部的匹配规则是：按照正则 location 在配置文件中的物理顺序（编辑顺序）匹配的，若规则匹配则不再搜索后面。

  **正则 location匹配让步一般location 的严格精确匹配结果；但覆盖普通 location的最大前缀匹配结果**

![1553532512304](..\image\nginx-正则匹配.png)

### 匹配优先问题

> 1. 一般 location 的最大前缀匹配结果与继续搜索的“正则location ”匹配结果的决策关系：如果继续搜索的“正则location ”也有匹配上的，那么“正则location ”**覆盖** “普通location ”的最大前缀匹配
>
> 2. 一般location 匹配完后，还会继续匹配正则location ；但是 nginx 允许你阻止这种行为，方法很简单，只需要在一般location 前加“^~ ”或“= ”。但其实还有一种“隐含”的方式来阻止正则location 的搜索，这种隐含的方式就是：当“最大前缀”匹配恰好就是一个“严格精确（exact match ）”匹配，照样会停止后面的搜索。原文字面意思是：只要遇到“精确匹配exact match ”，即使普通location 没有带“= ”或“^~ ”前缀，也一样会终止后面的匹配。





## Rewrite的使用

Rewrite 通过 ngx_http_rewrite_module 模块支持url重写、支持if判断，但**不支持else**

rewrite功能就是，使用 nginx 提供的全局变量或自己设置的变量，结合正则表达式和标志位实现url重写以及重定向

#### 作用范围

rewrite 只能放在server{},location{},if{}中，并且只能对域名后边的除去传递的参数外的字符串起作用



### 常用指令

#### If 空格 (条件) {设定条件进行重写}

条件的语法：

1. “=” 来判断相等，用于字符比较

2. “~” 用正则来匹配（表示区分大小写），“~*” 不区分大小写

3. “-f -d -e” 来判断是否为文件、目录、是否存在

#### return指令

语法：return code;

停止处理并返回指定状态码给客户端。

if ($request_uri ~ *\\.sh ){

  return 403

}

#### set指令

set variable value; 

定义一个变量并复制，值可以是文本、变量或者文本变量混合体



#### urlrewrite struts ???



#### rewrite指令

语法：rewrite regex replacement [flag]{last / break/ redirect 返回临时302/ permant  返回永久302}

* last: 停止处理后续的 rewrite 指令集、 然后对当前重写的 uri 在rewrite指令集上重新查找

* break：停止处理后续的 rewrite 指令集 ,并不会重新查找

综合实例

```conf
location / {
    rewrite '^/image/([a-z]{3})/(.*)\.(png|jpg)$' /melody?file=$2.$3;
    set $image_file $2;   # $表示匹配的内容为第几个
    set $image_type $3;
}

location /melody {
    root   html;
    try_file  /$arg_file /image404.html;
}

location =/image404.html {
    return 404 "image not found exception";
}

```



如上配置对于： /images/ttt/test.png 会重写到 /mic?file=test.png , 于是匹配到 location /mic ; 通过 try_files （按顺序）获取（第一个匹配）存在的文件进行返回。最后由于文件不存在所以直接返回404错误



### rewrite 匹配规则

表面看 rewrite 和 location 功能有点像，都能实现跳转，主要区别在于 rewrite 是在**同一域名内**更改获取资源的路径，而 location 是对**一类路径**做控制访问或反向代理，可以proxy_pass到其他机器。很多情况下 rewrite 也会写在location里，它们的执行顺序是：

·        执行 server 块的 rewrite 指令

·        执行 location 匹配

·        执行选定的 location 中的 rewrite 指令

如果其中某步 URI  **被重写**，则重新循环执行1-3，直到找到真实存在的文件；循环超过10次，则返回500 Internal Server Error错误

 

## 浏览器本地缓存配置及动静分离

注意：浏览器缓存失效配置

语法： expires 60s|m|h|d

操作步骤

1. 在html目录下创建一个 images 文件，在该文件中放一张图片

2. 修改 index.html, 增加图片展示

3. 修改 nginx.conf  配置。配置两个 location 实现动静分离，并且在静态文件中增加 expires 缓存期限

```conf
server {
    listen       80;
    server_name  localhost;


    location / {
        root   html;
        index  index.html;
    }
    
    location ~ \.(png|jpg|js|css|gif)$ {
        root   html/images;
        expires  5m;
    }
}
```



## Gzip压缩策略

浏览器请求 -> 告诉服务端当前浏览器可以支持压缩类型 -> 服务端会把内容根据浏览器所支持的压缩策略去进行压缩返回  ->浏览器拿到数据以后解码；  
常见的压缩方式：gzip、deflate 、sdch

```conf
server {
    listen       80;
    server_name  localhost;
    gzip on;
    gzip_buffer 4 16k;
    gzip_comp_level 7;
    gzip_min_length 500;
    gzip_types text/css application/xml application/javascript;

    location / {
        root   html;
        index  index.html;
    }
    
    location ~ \.(png|jpg|js|css|gif)$ {
        root   html/images;
        expires  5m;
    }
}
```



> Gzip on\|off                     #是否开启gzip压缩   
> Gzip_buffers 4 16k       #设置系统获取几个单位的缓存用于存储gzip的压缩结果数据流。4 16k代表以16k为单位，安装原始数据大小以16k为单位的4倍申请内存。   
> Gzip_comp_level[1-9]   #压缩级别， 级别越高，压缩越小，但是会占用CPU资源   
> Gzip_disable                  #正则匹配UA 表示什么样的浏览器不进行gzip   
> Gzip_min_length           #开始压缩的最小长度（小于多少就不做压缩）   
> Gzip_http_version   1.0\|1.1     # 表示开始压缩的http协议版本   
> Gzip_proxied                #（nginx 做前端代理时启用该选项，表示无论后端服务器的headers头返回什么信息，都无条件启用压缩）   
> Gzip_type   text/plain,application/xml  对那些类型的文件做压缩 （conf/mime.conf）   
> Gzip_vary   on\|off  是否传输gzip压缩标识



**注意点**

1. 图片、mp3这样的二进制文件，没必要做压缩处理，因为这类文件压缩比很小，压缩过程会耗费CPU资源

2. 太小的文件没必要压缩，因为压缩以后会增加一些头信息，反而导致文件变大

3. Nginx默认只对 text/html 进行压缩 ，如果要对 html 之外的内容进行压缩传输，我们需要手动来配置

## Nginx反向代理

#### 主要配置文件

在 http{}中添加  include  /etc/nginx/conf.d/\*.conf，

主配置文件主要配置公共属性

将 server 另外配置在 其他配置文件

Proxy_pass

通过反向代理把请求转发到百度

```conf
server {
    listen       80;
    server_name  域名;

    location =/s {
        proxy_set_header Host $host;
        proxy_set_header interface_version $host  
        proxy_set_header X-Real_IP $remote_addr;
        proxy_pass http://www.baidu.com;  # http://upstream_name
    }
     
}
```



Proxy_pass 既可以是 ip 地址，也可以是 域名，同时还可以 指定端口

Proxy_pass 指定的地址携带了 URI，看我们前面的配置【/s】，那么这里的 URI 将会替换请求 URI 中匹配location 参数部分；如上代码将会访问到 http://www.baidu.com/s

####  nginx 可能会拦截带下划线的参数（版本问题）

interface_version 

* 解决方式

 underscores_in_header on 



## 负载均衡

*upstream*是Nginx的HTTP Upstream模块，这个模块通过一个简单的调度算法来实现客户端 IP 到后端服务器的负载均衡

### Upstream常用参数介绍

 **语法：**server address [parameters]

其中关键字server必选。

 address 也必选，可以是主机名、域名、ip或unix socket，也可以指定端口号。

 parameters是可选参数，可以是如下参数：

​      **down**：表示当前server已停用

​      **backup**：表示当前server是备用服务器，只有其它非backup后端服务器都挂掉了或者很忙才会分配到请求

​      **weight**：表示当前server负载权重，权重越大被请求几率越大。默认是1

**max_fails **和 **fail_timeout** 一般会关联使用，如果某台server在 fail_timeout 时间内出现了 max_fails 次连接失败，那么Nginx会认为其已经挂掉了，从而在fail_timeout时间内不再去请求它，fail_timeout默认是10s，max_fails默认是1，即默认情况是只要发生错误就认为服务器挂掉了，如果将max_fails设置为0，则表示取消这项检查。

```conf
upstream tomcatserver{
    # ip_hash;
    server 192.168.11.140:8080;
    server 192.168.11.142:8080 weight=4;
}
```



* ups 支持的调度算法
  * ip_hash  根据ip的hash值来做转发
  * 默认是轮询机制
  * 权重 weight=x，x越大，权重越高
  * fair  根据服务器的响应时间来分配请求
  * url_hash
  * 可研究一下 dubbo 的负载均衡策略源码

 





## 关于web请求到服务端的整个过程的解析

 

## Nginx的实践演练

gupao-protal 首页

tomcat1 / tomcat2 

nginx配置对应的文件 ; /etc/nginx/conf.d/*.conf

upstream.conf  用来配置负载均衡的服务

  [www.xxx.com.conf](http://www.xxx.com.conf) 用来配置host信息

cdn.xxx.com

配置信息请参考对应的文件

```

#user  nobody;
worker_processes  2;
worker_cpu_affinity 0001 0010 0100 1000;

error_log /var/log/nginx/error.log warn;

worker_rlimit_nofile 10240; # 更改 worker 线程最大打开文件限制数（超了会报错），默认跟随系统

#pid        logs/nginx.pid;


events {
    use epoll; #select poll epoll kqueue 
    worker_connections  10240; # 小于 worker_rlimit_nofile
    accept_mutex off;  # 接受请求锁，惊群效应，worker线程竞争
}


http {
    include       mime.types;
    default_type  application/octet-stream;
    
    charset utf-8;
    
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/error.log  main;

    sendfile        on; # 高效传输
    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    gzip  on;
    gzip_buffer 4 16k;
    gzip_comp_level 7;
    gzip_min_length 5k;
    gzip_types text/css application/xml application/javascript image/jpeg;
    gzip_vary on;
    
    proxy_temp_path /melody/data/program;
    proxy_cache_path /melody/data/program/proxy_cache levels=1:2 keys_zone=melody:200m max_size=1g;
    
    include  /etc/nginx/conf.d/*.conf;

}
```

* 配置 负载均衡

```conf
upstream upstream_name{

    server 192.168.11.140:8080;
    server 192.168.11.142:8080 ;
}
```



* 配置server

```conf
server {
        listen       80;
        server_name  localhost;



       location / {                   
            proxy_pass http://upstream_name;
            proxy_set_header X-Real_IP $remote_addr;
            proxy_send_timeout 500;
        }
        
        location ~ \.(png|jpg|js|css|gif)$ {
            root   /data/program/tomcat/webapps/ROOT/;
            expires  5m;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

    }


    

    # HTTPS server
    #
    server {
        listen       443 ssl;
        server_name  localhost;
        ssl on;
        ssl_certificate      crt_path;#cert.pem;
        ssl_certificate_key  csr_key_path;

    #   ssl_session_cache    shared:SSL:1m;
    #   ssl_session_timeout  5m;

    #   ssl_ciphers  HIGH:!aNULL:!MD5;
    #   ssl_prefer_server_ciphers  on;

        location / {
            proxy_pass   http://upstream_name;
        }
    #}
```



 

## Nginx的进程模型

服务端响应客户端的方式，多进程、多线程（内存共享）、异步方式（同/异步、阻塞/非阻塞）

### Master进程

充当整个进程组与用户的交互接口，同时对进程进行监护。它不需要处理网络事件，不负责业务的执行，只会通过管理work进程来实现重启服务、平滑升级、更换日志文件、配置文件实时生效等功能。

* 主要是用来管理worker进程

1. 接收来自外界的信号 （前面提到的 kill -HUP 信号等）
   我们要控制nginx，只需要通过 kill 向 master 进程发送信号就行了。比如kill -HUP pid，则是告诉 nginx，从容地重启 nginx，我们一般用这个信号来重启 nginx，或重新加载配置，因为是从容地重启，因此服务是不中断的。
   * master进程在接收到HUP信号后是怎么做的呢？
     首先master进程在接到信号后，会先重新加载配置文件，然后再启动新的worker进程，并向所有老的worker进程发送信号，告诉他们可以光荣退休了。新的worker在启动后，就开始接收新的请求，而老的worker在收到来自master的信号后，就不再接收新的请求，并且在当前进程中的所有未处理完的请求处理完成后，再退出

2. 向各个worker进程发送信号

3. 监控worker进程的运行状态

4. 当worker进程退出后（异常情况下），会自动重新启动新的worker进程

### Work进程（异步非阻塞  epoll）

IO多路复用   epoll

主要是完成具体的任务逻辑。它的主要关注点是客户端和后端真实服务器之间的数据可读、可写等I/O交互事件

各个worker进程之间是对等且**相互独立**的，他们同等竞争来自客户端的请求，一个请求只可能在一个worker进程中处理，worker进程个数一般设置为 cpu核数

master进程先建好需要 listen 的 socket 后，然后再 fork 出多个woker进程，这样每个work进程都可以去accept这个socket。当一个client连接到来时，所有accept的work进程都会受到**通知**，但只有一个进程可以accept成功，其它的则会accept失败

 

 

## Nginx配置https的请求

·        https基于SSL/TLS这个协议；

·        非对称加密、对称加密、 hash算法

·        crt的证书 -> 返回给浏览器

 

### 创建证书

```shell
#创建服务器私钥
openssl genrsa -des3 -out server.key 1024

#创建签名请求的证书（csr）; csr核心内容是一个公钥
openssl req -new -key server.key -out server.csr

#去除使用私钥时的口令验证
cp server.key server.key.org

openssl rsa -in server.key.org -out server.key

#标记证书使用私钥和csr
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt    
```

x509是一种证书格式

server.crt就是我们需要的证书

CA签发

**nginx** 配置文件在git上的[www.xxx.com.conf](http://www.xxx.com.conf)**文件**

### tomcat增加对https的支持

Connector 8080节点加入 redirectPort="443" proxyPort="443"
redirectPort ：当http请求有安全约束才会转到443端口使用ssl传输



## Nginx+keepalived

keepalived  –>  VRRP(虚拟路由器冗余协议)

VRRP 全称 Virtual Router Redundancy Protocol，即 虚拟路由冗余协议。可以认为它是实现路由器高可用的容错协议，即将N台提供相同功能的路由器组成一个路由器组(Router Group)，这个组里面有一个 master 和多个backup，但在外界看来就像一台一样，构成虚拟路由器，拥有一个虚拟IP（vip，也就是路由器所在局域网内其他机器的默认路由），占有这个 IP 的 master 实际负责 ARP(Address Resolution Protocol)相应和转发 IP 数据包，组中的其它路由器作为备份的角色处于待命状态。master 会发组播消息，当 backup 在超时时间内收不到 vrrp 包时就认为 master 宕掉了，这时就需要根据 VRRP 的优先级来选举一个 backup 当 master，保证路由器的高可用。

### 安装keepalived

1. tar -zxvf keepalived.tar.gz

2. ./configure --prefix=/mic/data/program/keepalived --sysconf=/etc

3. 缺少依赖：yum install gcc ; yum install openssl-devel；yum -y install libnl libnl-devel  

4. 编译安装 make && make install

5. cd 到解压的包 /parker/data/program/keepalived-1.3.9

6. ln -s /mic/data/program/keepalived/sbin/keepalived    /sbin  --建立软链接

7. cp /mic/data/program/keepalived-1.3.9/keepalived/etc/init.d/keepalived      /etc/init.d/

8. 添加到系统服务
   a)     chkconfig --add keepalived
   b)    chkconfig keepalived on
   c)     Service keepalived start

### 基于keepalived+nginx的配置

**配置文件详见 keepalived.conf;  gitlab**

  ```con
! Configuration File for keepalived

#全局配置
global_defs {
   notification_email {  #指定keepalived在发生切换时需要发送email到的对象，一行一个
     acassen@firewall.loc     # 报警邮件
     failover@firewall.loc
     sysadmin@firewall.loc
   }
   notification_email_from Alexandre.Cassen@firewall.loc  #指定发件人
   #smtp_server 192.168.200.1                             #指定smtp服务器地址
   #smtp_connect_timeout 30                               #指定smtp连接超时时间
   router_id LVS_DEVEL                                    #运行keepalived机器的一个标识
}

##############
vrrp_script check_nginx { 
    script "/etc/keepalived/check_nginx.sh"         ##监控脚本 
    interval 2                                      ##时间间隔，2秒
    weight 2                                        ##权重 
}
#############
vrrp_instance VI_1 { 
    state MASTER           #标示状态为MASTER 备份机为BACKUP
    interface eth0         #设置实例绑定的网卡
    virtual_router_id 51   #同一实例下（主从）virtual_router_id必须相同
    priority 100           # MASTER 权重要高于 BACKUP 比如 BACKUP 为99  
    advert_int 1           # MASTER 与 BACKUP 负载均衡器之间同步检查的时间间隔，单位是秒
    authentication {       #设置认证
        auth_type PASS     #主从服务器验证方式
        auth_pass 1111
    }
    virtual_ipaddress {    #设置vip
        192.168.0.55       #可以多个虚拟IP，换行即可
    }
    #################
    track_script {
	    check_nginx        #监控脚本
    }

}

#虚拟服务器 80端口的配置
virtual_server 192.168.0.55 80 {
    delay_loop 6              #(每隔10秒查询realserver状态)  
    lb_algo rr                #负载均衡算法   lvs调度算法rr|wrr|lc|wlc|lblc|sh|dh
    lb_kind DR                #负载均衡转发规则NAT|DR|RUN
    #nat_mask 255.255.255.0   #掩码
    persistence_timeout 50    #会话保持时间(同一IP的连接60秒内被分配到同一台realserver)  
    protocol TCP              #使用的协议

    #实际服务器的IP和端口  
    real_server 192.168.0.48 80 {
        weight 1              #默认为1,0为失效
        HTTP_GET {             #使用http get检测方式
            url {
              path /index.html
              #digest ff20ad2481f97b1754ef3e12ecd3a9cc  #http://192.168.0.48/index.html的digest值 
	      status_code 200                           #http://192.168.0.48/index.html的返回状态码
            }
            connect_timeout 3     #连接超时时间
            nb_get_retry 3        #重连次数
            delay_before_retry 3  #重连间隔时间
        }
    }
}


  ```



 systemctl start keepalived

 systemctl status keepalived

 systemctl restart keepalived

/etc/sysconfig/keepalived

 KEEEPALIVED_OPTIONS="-D -d -S 0"

/etc/rsyslog   配 local0\.*  /var/log/keepalived.log



#### 还需引入脚本

```shell
#!/bin/bash
# 如果进程中没有nginx则将keepalived进程kill掉
A=`ps -C nginx --no-header |wc -l`      ## 查看是否有 nginx进程 把值赋给变量A 
if [ $A -eq 0 ];then                    ## 如果没有进程值得为 零
       service keepalived stop          ## 则结束 keepalived 进程
fi
```



 

 

 

 

 

 

 

 

 

 



​                                                                     



 

 

 