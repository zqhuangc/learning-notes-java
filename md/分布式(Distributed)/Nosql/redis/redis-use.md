# redis的优势

## 存储结构

1. 字符类型

2. 散列类型

3. 列表类型

4. 集合类型

5. 有序集合

## 功能

1. 可以为每个key设置超时时间；

2. 可以通过列表类型来实现分布式队列的操作

3. 支持发布订阅的消息模式

## 简单

1. 提供了很多命令与redis进行交互

# redis的应用场景

1. 数据缓存（商品数据、新闻、热点数据）

2. 单点登录

3. 秒杀、抢购

4. 网站访问排名…

5. 应用的模块开发

 

# redis的安装

1. 下载redis安装包 

2. tar -zxvf 安装包

3. 在redis目录下 执行 make

4. 可以通过make test测试编译状态

5. make install [prefix=/path]完成安装

 

## 启动停止redis

./redis-server ../redis.conf

./redis-cli shutdown

以后台进程的方式启动，修改redis.conf   daemonize =yes

 

连接到redis的命令

 ./redis-cli -h 127.0.0.1 -p 6379

 

## 其他命令说明

Redis-server             启动服务

Redis-cli                    访问到redis的控制台

redis-benchmark    性能测试的工具

redis-check-aof       aof文件进行检测的工具

redis-check-dump  rdb文件检查工具

redis-sentinel         sentinel 服务器配置

# 多数据支持

默认支持16个数据库；可以理解为一个命名空间

跟关系型数据库不一样的点

1. redis不支持自定义数据库名词

2. 每个数据库不能单独设置授权

3. 每个数据库之间并不是完全隔离的。 可以通过flushall命令清空redis实例面的所有数据库中的数据

通过  select dbid 去选择不同的数据库命名空间 。 dbid的取值范围默认是0 -15

 

# 使用入门

1. 获得一个符合匹配规则的键名列表

keys pattern  [? / * /[]] 

如：keys mic:hobby 



2. 判断一个键是否存在 ， EXISTS key \3.   type key 去获得这个key的数据结构类型

# 各种数据结构的使用

## 字符类型

一个字符类型的key默认存储的最大容量是512M

赋值和取值

SET key  value

GET key

**递增数字 incr key**

 

**错误的演示**

**int value= get key;**

**value =value +1;**

**set key value;**



#### key的设计

**对象类型:对象id:对象属性:对象子属性**

**建议对key进行分类，同步在wiki统一管理**

**短信重发机制：sms:limit:mobile 138。。。。。 expire** 

 

**incryby key increment**  **递增指定的整数**

**decr key**   **原子递减**

**append key value**   **向指定的key追加字符串**

**strlen  key**  **获得key对应的value的长度**

**mget  key key..**  **同时获得多个key的value**

**mset key value  key value  key value …**

setnx 

## 列表类型

**list,** **可以存储一个有序的字符串列表**

LPUSH/RPUSH： 从左边或者右边push数据

LPUSH/RPUSH key value value …

｛17 20 19 18 16｝

#### 具体可看官网命令的使用

llen num  获得列表的长度

lrange key  start stop   获得列表片段;  索引可以是负数， -1表示最右边的第一个元素

lrem key count value   删除指定元素（count 表示个数，正负表示方向）

lset key index value     设置索引对应的值

LPOP/RPOP : 取数据

**应用场景：可以用来做分布式消息队列**

## 散列类型

hash key value  不支持数据类型的嵌套

比较适合存储对象

person

age  18

sex   男

name mic

..

hset key field value

hget key filed 

 

hmset key filed value [filed value …]  一次性设置多个值

hmget key field field …  一次性获得多个值                  multi get

hgetall key  获得hash的所有信息，包括key和value

hexists key field 判断字段是否存在。 存在返回1. 不存在返回0

hincryby

hsetnx

hdel key field [field …] 删除一个或者多个字段

## 集合类型

set 跟list 不一样的点。 集合类型不能存在重复的数据。而且是无序的

sadd key member [member ...] 增加数据； 如果value已经存在，则会忽略存在的值，并且返回成功加入的元素的数量

srem key member  删除元素

smembers key 获得所有数据

 

sdiff key1 key2 …  对多个集合执行差集运算（key1 去掉与 key2 相同元素）

sunion 对多个集合执行并集操作, 同时存在在两个集合里的所有值

## 有序集合

zadd key score member [score member]...

 

zrange key start stop [withscores] 去获得元素。 withscores是可以获得元素的分数

如果两个元素的score是相同的话，那么根据(0<9<A<Z<a<z) 方式从小到大

* 网站访问的前10名

# redis的事务处理

MULTI 去开启事务

命令

EXEC 去执行事务



>事务在  ，命令 EXEC 时出错，不回滚

 命令放在 Queued

# 过期时间

expire key seconds 

ttl  获得key的过期时间

 

# 发布订阅

publish channel message

subscribe channel [ …]

 

codis . twmproxy

 配置修改  注释掉bind  修改 protected-mode 可让外网访问（不推荐）

# redis实现分布式锁

数据库可以做     如：activemq  master选举的时候    lock 表

 

缓存 -redis  setnx

 

zookeeper 



# 分布式锁的实现

锁是用来解决什么问题的;

1. 一个进程中的多个线程，多个线程并发访问同一个资源的时候，如何解决线程安全问题。
2. 一个分布式架构系统中的两个模块同时去访问一个文件对文件进行读写操作
3. 多个应用对同一条数据做修改的时候，如何保证数据的安全性

资源共享竞争，数据安全性

在单进程中，我们可以用到 synchronized、lock 之类的同步操作去解决，但是对于分布式架构下多进程的情况下，如何做到跨进程的锁。就需要借助一些**第三方手段**来完成

# 设计一个分布式所需要解决的问题

分布式锁的解决方案

1. 怎么去获取锁

## 数据库，通过唯一约束

#### 锁表结构

> lock(
>
> ​    id  int(11),
>
> ​    methodName  varchar(100),
>
> ​    memo varchar(1000) ,
>
> ​    modifyTime timestamp,
>
> ​     unique key mn (method)  --唯一约束
>
> )

#### 获取锁的伪代码

> try{
>
> ​    exec  insert into lock(methodName,memo) values(‘method’,’desc’);    methodA
>
> ​    return true;
>
> }Catch(DuplicateException e){
>
>    return false;
>
> }

释放锁

> delete from lock where methodName=’’;

### 存在的需要思考的问题

1. 锁没有失效时间，一旦解锁操作失败，就会导致锁记录一直在数据库中，其他线程无法再获得到锁

2. 锁是非阻塞的，数据的insert操作，一旦插入失败就会直接报错。没有获得锁的线程并不会进入排队队列，要想再次获得锁就要再次触发获得锁操作

3. 锁是非重入的，同一个线程在没有释放锁之前无法再次获得该锁

## zookeeper实现分布式锁

利用zookeeper的**唯一**节点特性**或者**有序临时节点特性获得**最小**节点作为锁. zookeeper 的实现相对简单，通过curator客户端，已经对锁的操作进行了封装，原理看 zookeeper 介绍，

> /Locker
>
> ​    | clientA-seq
>
> ​    | clientB-seq

### zookeeper的优势

1. 可靠性高、实现简单

2. zookeeper因为临时节点的特性，如果因为其他客户端或异常和zookeeper连接中断了，那么节点会被删除，意味着锁会被自动释放

3. zookeeper本身提供了一套很好的集群方案，比较稳定

4. 释放锁操作，会有watch通知机制，也就是服务器端会主动发送消息给客户端这个锁已经被释放了

## 基于缓存的分布式锁实现

redis中有一个setNx命令，这个命令只有在key不存在的情况下为key设置值。所以可以利用这个特性来实现分布式锁的操作

### 具体实现代码

1. 添加依赖包

2. 编写redis连接的代码

   释放锁的代码

3. 分布式锁的具体实现
   过期时间，ttl，
   问题：过期后自动释放锁问题，A持有锁，操作时间过长过期后自动释放锁，B拿到锁操作，A释放锁（把 B的锁释放了？？？？？）

4. 怎么释放锁，watch（监控指定 key），key若发生变化后面的代码将不会执行，unwatch()



`jedis.set(String key, String value, String nxxx, String expx, int time)`

- 第一个为key，我们使用key来当锁，因为key是唯一的。
- 第二个为value，我们传的是requestId，很多童鞋可能不明白，有key作为锁不就够了吗，为什么还要用到value？原因就是我们在上面讲到可靠性时，分布式锁要满足第四个条件**解铃还须系铃人**，通过给value赋值为requestId，我们就知道这把锁是哪个请求加的了，在解锁的时候就可以有依据。requestId可以使用`UUID.randomUUID().toString()`方法生成。
- 第三个为nxxx，这个参数我们填的是NX，意思是SET IF NOT EXIST，即当key不存在时，我们进行set操作；若key已经存在，则不做任何操作；
- 第四个为expx，这个参数我们传的是PX，意思是我们要给这个key加一个过期的设置，具体时间由第五个参数决定。
- 第五个为time，与第四个参数相呼应，代表key的过期时间。

```java
public static boolean releaseDistributedLock(Jedis jedis, String lockKey, String requestId) {

        String script = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        Object result = jedis.eval(script, Collections.singletonList(lockKey), Collections.singletonList(requestId));

        if (RELEASE_SUCCESS.equals(result)) {
            return true;
        }
        return false;

    }
```



# redis多路复用机制

linux 的内核会把所有外部设备都 看作一个文件 来操作，对一个文件的读写操作会调用内核提供的系统命令，返回一个 file descriptor（文件描述符）。对于一个socket的读写也会有响应的描述符，称为socketfd(socket 描述符)。而**IO多路复用**是指内核一旦发现进程指定的一个或者多个文件描述符IO条件准备好以后就*通知*该进程

*IO多路复用又称为事件驱动*，操作系统提供了一个功能，当某个socket可读或者可写的时候，它会给一个通知。当配合非阻塞socket使用时，只有当系统通知我哪个描述符可读了，我才去执行read操作，可以保证每次read都能读到有效数据。操作系统的功能通过 **select/pool/epoll/kqueue** 之类的系统调用函数来使用，这些函数可以同时监视多个描述符的读写就绪情况，这样多个描述符的I/O操作都能在一个线程内并发交替完成，这就叫I/O多路复用，这里的复用指的是同一个线程

多路复用的优势在于用户可以在一个线程内同时处理多个socket的 io请求。达到同一个线程同时处理多个IO请求的目的。而在同步阻塞模型中，必须通过多线程的方式才能达到目的

 

# redis中使用lua脚本

## lua脚本

Lua是一个高效的轻量级脚本语言，用标准C语言编写并以源代码形式开放， 其设计目的是为了嵌入应用程序中，从而为应用程序提供灵活的扩展和定制功能

## 使用脚本的好处

1. 减少网络开销，在Lua脚本中可以把多个命令放在同一个脚本中运行

2. 原子操作，redis会将整个脚本作为一个整体执行，中间不会被其他命令插入。换句话说，编写脚本的过程中无需担心会出现竞态条件

3. 复用性，客户端发送的脚本会永远存储在redis中，这意味着其他客户端可以复用这一脚本来完成同样的逻辑 

## Lua在linux中的安装

到官网下载lua的tar.gz的源码包

tar -zxvf lua-5.3.0.tar.gz

进入解压的目录：

cd lua-5.2.0

make linux  (linux环境下编译)

make install

如果报错，说找不到readline/readline.h, 可以通过yum命令安装

yum -y install readline-devel ncurses-devel

安装完以后再make linux  / make install

最后，直接输入 lua命令即可进入lua的控制台

## lua的语法

暂无

lua 动态类型语言

变量

全局变量，局部变量

a=1;local b=2;

逻辑表达式：+、-、\*、/、==、~=、>、<、%

类型不会自动转换

逻辑运算符 and / or  not(a and b)

连接  a..b    #a 计算字符长度

可参考 [菜鸟教程](http://www.runoob.com/lua/lua-miscellaneous-operator.html)

## Redis与Lua

在Lua脚本中调用Redis命令，可以使用redis.call函数调用。比如我们调用string类型的命令

redis.call(‘set’,’hello’,’world’)

redis.call 函数的返回值就是redis命令的执行结果。前面我们介绍过redis的5中类型的数据返回的值的类型也都不一样。redis.call函数会将这5种类型的返回值转化对应的Lua的数据类型

## 从Lua脚本中获得返回值

在很多情况下我们都需要脚本可以有返回值，在脚本中可以使用return 语句将值返回给redis客户端，通过return语句来执行，如果没有执行return，默认返回为nil。

## 如何在redis中执行lua脚本

Redis提供了EVAL命令可以使开发者像调用其他Redis内置命令一样调用脚本。

\[EVAL]\[脚本内容] \[key参数的数量]\[key …]\[arg …]

可以通过key和arg这两个参数向脚本中传递数据，他们的值可以在脚本中分别使用**KEYS**和**ARGV** 这两个类型的全局变量访问。比如我们通过脚本实现一个set命令，通过在redis客户端中调用，那么执行的语句是：

lua脚本的内容为： return redis.call(‘set’,KEYS[1],ARGV[1])         //KEYS和ARGV必须大写

eval "return redis.call('set',KEYS[1],ARGV[1])" **1** hello world

EVAL命令是根据 key参数的数量-也就是上面例子中的1来将后面所有参数分别存入脚本中KEYS和ARGV两个表类型的全局变量。当脚本不需要任何参数时也不能省略这个参数。如果没有参数则为0

eval "return redis.call(‘get’,’hello’)" **0**



### EVALSHA命令

考虑到我们通过 eval 执行 lua 脚本，脚本比较长的情况下，每次调用脚本都需要把整个脚本传给redis，比较占用带宽。为了解决这个问题，redis提供了EVALSHA命令允许开发者通过脚本内容的SHA1摘要来执行脚本。该命令的用法和EVAL一样，只不过是将脚本内容替换成脚本内容的SHA1摘要

 

1. Redis在执行EVAL命令时会计算脚本的SHA1摘要并记录在脚本缓存中

2. 执行EVALSHA命令时Redis会根据提供的摘要从脚本缓存中查找对应的脚本内容，如果找到了就执行脚本，否则返回“NOSCRIPT No matching script,Please use EVAL”

 

通过以下案例来演示EVALSHA命令的效果

script load "return redis.call('get','hello')"          将脚本加入缓存并生成sha1命令

evalsha "a5a402e90df3eaeca2ff03d56d99982e05cf6574" 0

我们在调用eval命令之前，先执行evalsha命令，如果提示脚本不存在，则再调用eval命令

## lua脚本实战

实现一个针对某个手机号的访问频次， 以下是lua脚本，保存为phone_limit.lua

local num=redis.call('incr',KEYS[1])

if tonumber(num)==1 then

   redis.call('expire',KEYS[1],ARGV[1])

   return 1

elseif tonumber(num)>tonumber(ARGV[2]) then

   return 0

else

   return 1

end

通过如下命令调用

./redis-cli --eval phone_limit.lua rate.limiting:13700000000 , 10 3



>  语法为 ./redis-cli --eval \[lua脚本\]\[key…\]空格,空格\[args…\]（**注**：空格不能少）
>
>





## 脚本的原子性

redis 的脚本执行是原子的，即脚本执行期间 Redis 不会执行其他命令。所有的命令必须等待脚本执行完以后才能执行。为了防止某个脚本执行时间过程导致Redis无法提供服务。Redis提供了lua-time-limit参数限制脚本的最长运行时间。默认是5秒钟。

当脚本运行时间超过这个限制后，Redis将开始接受其他命令但不会执行（以确保脚本的原子性），而是返回BUSY的错误

### 实践操作

打开两个客户端窗口

在第一个窗口中执行lua脚本的死循环

eval “while true do end” 0

 

在第二个窗口中运行get hello

 

最后第二个窗口的运行结果是Busy, 可以通过script kill命令终止正在执行的脚本。如果当前执行的lua脚本对redis的数据进行了修改，比如（set）操作，那么script kill命令没办法终止脚本的运行，因为要保证lua脚本的原子性。如果执行一部分终止了，就违背了这一个原则

在这种情况下，只能通过 **shutdown nosave**命令强行终止

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 



 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 

 