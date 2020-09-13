* JSESSIONID与cookie是什么关系，session与cookie到底有什么关系。

简单来说，当第一次request server时，server产生JSESSIONID对应的值1，通过http header set-cookie，传递给browser，browser检测到http response header 里带set-cookie，那么browser就会create一个cookie，key=JSESSIONID，value=值1，而后的每次请求，browser都会把cookie里的键值对，放到http request header里，传递给server。

当在server端调用http.getSession()方法时，server会先从http request header里解析出来JSESSIONID的值，再从一个Map容器里去找有没有value，如果没有，就会产生一个HttpSessioon对象，放到这个Map容器里，同时设置一个最大生存时间。HttpSession你也可以把它想象成是一个Map，可以getAttribute()，可以setAttribute()。

### 问题3，在集群中的某一台机器上，如何高效的判断Session过期
Timing Wheel(时间轮)的东西，这个东西很棒，可以高效的管理超大量的定时任务，也是能想到的最高效最好的设计方案，
### 问题4，分布式session，如何处理logout
loginIn还好说，直接根据sessionid从cache中拿数据即可。但是怎么处理logout呢，logout时，要清理local cache中与session相关的信息。  
由于 logout 对实时性要求几乎没有，晚个几少钟几十秒钟，都无所谓的，所以可以利用消息中间件来处理，比如ActiveMQ 等。

### 问题5，最好的分布式session处理方法是什么
最好的处理方法就是让前端的 LoadBalance，根据 sessionId，把请求进行hash或一致性hash，让用户固定到集群中的某台机器上，这样做最大的好处是，可以充分利用内存，相当于单机处理逻辑了。

**IP哈希** 解决方案，它可以让每次从同一IP过来的请求都转发到后端同一台服务器上

## 分布式系统session共享问题
### 方式一：
存储在数据库中，用户登入时，把session信息储存在数据库中，然后再需要获取session的地方进行读取。
* 优点：开发简单  
* 缺点 ：依赖性太强，业务量大的时候数据库压力大，数据库出现问题影响整个系统。

### 方式二 ：
cookie 共享session ；当用户登入时，把 cookie储存在客户端，当用户需要session判断是否登入时，首先判断本服务器是否有session，如果没有同步cookie信息。(购物车在数据库和cookie都存，就是为了把用户未登入时产生的购物车同步到登入用户上)
* 优点：用起来方便，开发效率高
* 缺点 ： cookie安全性不高，容易伪造，当客户端禁用时则该方法失效

### 方式三  ;
服务器共享session ，使用一台作为用户的登录服务器，当用户登录成功之后，会将session写到当前服务器上，我们通过脚本或者守护进程将session同步到其他服务器上，这时当用户跳转到其他服务器，session一致，也就不用再次登录。
* 优点：安全 ，一次配置永久使用
* 缺点 ; 同步效率低，慢 ，有时出现没同步现象

### 方式四  ：
通过缓存同步session，用户登入时，把登入信息放在redis或者memcache中，我们取session信息的时候都从缓存中取。（重写session创建 获取 销毁 方法 创建存缓存中  获取从缓存中获取  销毁缓存中记录和服务器记录）
* 优点： 比前几种方式效率高  读取速度快  安全
* 缺点 ：重写底层方法复杂 ，开发慢  

## 六种实现
### 一、常见的分布式session实现方式有以下几种

1. 基于数据库的Session共享
2. 基于NFS共享文件系统
3. 基于memcached 的session
4. 基于resin/tomcat web容器本身的session复制机制
5. 基于TT/Redis 或 jbosscache 进行 session 共享。
6. 基于cookie 进行session共享

### 二、优缺点分析

1. 基于数据库的session共享。
* 原理：就不用多说了吧，拿出一个数据库，专门用来存储session信息。保证session的持久化。
* 优点：服务器出现问题，session不会丢失。
* 缺点：如果网站的访问量很大，把session存储到数据库中，会对数据库造成很大压力，还需要增加额外的开销维护数据库。

2. 基于基于NFS共享文件系统
* 原理：拿出一个服务器，搭建NFS服务器来共享session。保证session的持久化。
* 优点：用NFS来存储session的缺点是，session过期后可以实现自动清除，必须自己设定回收机制，我们可以利用crontab来定期回收，用用以下shell命令即可：
find /tmp/php_sess -mmin +30 | xargs rm -fr
* 缺点：如果session量比较大并且所有的session文件都在同一个子目录下的话，那么可能会由此带来很严重的负载问题，甚至导致网站无法使用

3. 基于memcached 的session（不提倡）

这里memcached创建者Dormando很早就写过两篇文章，告诫开发人员不要用memcached存储Session。他在第一篇文章中给出的理由大致是说，如果用memcached存储Session，那么当memcached集群发生故障（比如内存溢出）或者维护（比如升级、增加或减少服务器）时，用户会无法登录，或者被踢掉线。而在第二篇文章中，他则指出，memcached的回收机制可能会导致用户无缘无故地掉线。

4. Session Replication 方式管理 (即session复制)  
* 原理：将一台机器上的Session数据广播复制到集群中其余机器上
* 优点：实现简单、配置较少、当网络中有机器Down掉时不影响用户访问
* 缺点：在机器较少，网络流量较小广播式复制到其余机器上，当机器数量增多时候会有一定廷时，带来一定网络开销

5. 基于TT/Redis 或 jbosscache 进行 session 共享（倾向于Redis）

所有Web服务器都把Session写入到或者redis，也都从memcache或者redis来获取。
* 优点:memcache或则redis本身就是一个分布式缓存，便于扩展。网络开销较小，几乎没有IO。性能也更好。
* 缺点：受制于Memcache的容量（除非你有足够内存存储），如果用户量突然增多cache由于容量的限制会将一些数据挤出缓存，另外memcache故障或重启session会完全丢失掉。所以更偏向于redis。

6. 基于cookie 进行session共享

将用户的session数据全部存放在cookie中，很多大型站点都在这么干。优点是服务器架构也变得简单，每台web服务器都可以很独立。没有网络开销和对磁盘IO，服务器重启也不会导致数据的丢失。缺点，cookie过于庞大会耗费单位页面的下载时间，所以要尽量保持cookie的精简。


## 三种实现
### 一、Session Replication 方式管理 (即session复制) 
* 简介：将一台机器上的Session数据广播复制到集群中其余机器上 
* 使用场景：机器较少，网络流量较小 
* 优点：实现简单、配置较少、当网络中有机器Down掉时不影响用户访问 
* 缺点：广播式复制到其余机器有一定廷时，带来一定网络开销

### 二、Session Sticky 方式管理 
* 简介：即粘性Session、当用户访问集群中某台机器后，强制指定后续所有请求均落到此机器上 
* 使用场景：机器数适中、对稳定性要求不是非常苛刻 
* 优点：实现简单、配置方便、没有额外网络开销 
* 缺点：网络中有机器Down掉时、用户Session会丢失、容易造成单点故障

### 三、缓存集中式管理 
* 简介：将Session存入分布式缓存集群中的某台机器上，当用户访问不同节点时先从缓存中拿Session信息 
* 使用场景：集群中机器数多、网络环境复杂 
* 优点：可靠性好 
* 缺点：实现复杂、稳定性依赖于缓存的稳定性、Session信息放入缓存时要有合理的策略写入