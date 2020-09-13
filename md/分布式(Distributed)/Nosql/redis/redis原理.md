[TOC]

支持的数据类型：字符串（Strings），散列（Hash），列表（List），集合（Set），有序集合（Sorted Set或者是ZSet）与范围查询，Bitmaps，Hyperloglogs 和地理空间（Geospatial）索引半径查询。其中常见的数据结构类型有：String、List、Set、Hash、ZSet这5种。

单线程指的是网络请求模块使用了一个线程，其他模块用了多个线程。

缓存数据中**list，array数据序列化后存储**

## Redis协议规范(RESP)

Redis 即 REmote Dictionary Server (远程字典服务)；  
而Redis的协议规范是**Redis Serialization Protocol (Redis序列化协议)** 
该协议是用于与Redis服务器通信的，用的较多的是Redis-cli通过pipe与Redis服务器联系；  

* 协议如下：  
客户端以规定格式的形式发送命令给服务器；  
服务器在执行最后一条命令后，返回结果。  

客户端发送命令的格式(类型)：5种类型

*间隔符号，在Linux下是\r\n，在Windows下是\n*

#### 1. 简单字符串 Simple Strings, 以 "+"加号 开头  

格式：+ 字符串 \r\n  
字符串不能包含 CR或者 LF(不允许换行)
eg: "+OK\r\n"

注意：为了发送二进制安全的字符串，一般推荐使用后面的 Bulk Strings类型  

#### 2. 错误 Errors, 以"-"减号 开头

格式：- 错误前缀 错误信息 \r\n  
错误信息不能包含 CR或者 LF(不允许换行)，Errors与Simple Strings很相似，不同的是Erros会被当作异常来看待

eg: "-Error unknow command 'foobar'\r\n"

#### 3. 整数型 Integer， 以 ":" 冒号开头 

格式：: 数字 \r\n  
eg: ":1000\r\n"  

#### 4. 大字符串类型 Bulk Strings, 以 "$"美元符号开头，长度限制512M

格式：$ 字符串的长度 \r\n 字符串 \r\n  
字符串不能包含 CR或者 LF(不允许换行);  

eg: "$6\r\nfoobar\r\n"    其中字符串为 foobar，而6就是foobar的字符长度

"$0\r\n\r\n"       空字符串  
"$-1\r\n"           null  

#### 5. 数组类型 Arrays，以 "*"星号开头  

格式：* 数组元素个数 \r\n 其他所有类型 (结尾不需要\r\n)

注意：只有元素个数后面的\r\n是属于该数组的，结尾的\r\n一般是元素的

eg: "*0\r\n"      空数组
"*2\r\n$2\r\nfoo\r\n$3\r\nbar\r\n"      数组包含2个元素，分别是字符串foo和bar
"*3\r\n:1\r\n:2\r\n:3\r\n"       数组包含3个整数：1、2、3
"*5\r\n:1\r\n:2\r\n:3\r\n:4\r\n$6\r\nfoobar\r\n"  包含混合类型的数组
"*-1\r\n"         Null数组
"*2\r\n*3\r\n:1\r\n:2\r\n:3\r\n*2\r\n+Foo\r\n-Bar\r\n"   数组嵌套，外层数组包含2个数组，整理后如下：

"*2\r\n
　*3\r\n:1\r\n:2\r\n:3\r\n
　*2\r\n+Foo\r\n-Bar\r\n"

```一条命令
Requests格式

* 参数的个数 CRLF 
$ 第一个参数的长度CRLF 
第一个参数CRLF ... 
$ 第N个参数的长度CRLF 
第N个参数CRLF

例如：
* 2  //(两个参数)
& 3  //(第一个参数长度)
get
& 2  //(第二个参数长度)
aa  

每行以\r\n结尾


public String command(final String command, String[] args) throws IOException {
        StringBuilder sb = new StringBuilder();
        // *表示数组 后面数字表示数组长度(命令+参数的参数个数)
        sb.append("*" + (args.length + 1).append("\r\n");
        sb.append("$" + command.length()).append("\r\n");
        sb.append(command).append("\r\n");
        for (String arg : args) {
            sb.append("$").append(arg.getBytes().length).append("\r\n");
            sb.append(arg).append("\r\n");
        }

        socket.getOutputStream().write(sb.toString().getBytes());
        byte[] b = new byte[2048];
        socket.getInputStream().read(b);
        return new String(b);
    }
```
## 持久化策略
**rdb** 数据fork副本快照（持久化过程占用相同内存）  
触发配置（时间，修改键数），耗内存，会压缩
**aof** 持久化执行的命令（默认 30 s 同步）可设置，持久化空白期命令可能因宕机丢失  

根据实际情况，可以每隔一定时间将数据集导出到磁盘（快照），或者追加到命令日志中（AOF只追加文件），他会在执行写命令时，将被执行的写命令复制到硬盘里面。您也可以关闭持久化功能，将Redis作为一个高效的网络的缓存数据功能使用。

## 启动命令
redis-server /etc/redis.conf（配置文件路径）

## 主从复制（读写分离）
一主多从
主从从
```
在redis中设置主从有2种方式：

1、	在redis.conf中设置slaveof
a)	slaveof <masterip> <masterport>
2、	使用redis-cli客户端连接到redis服务，执行slaveof命令
a)	slaveof <masterip> <masterport>

第二种方式在重启后将失去主从复制关系。


查看主从信息：INFO replication

role：角色
connected_slaves：从库数量
slave0：从库信息

默认情况下redis数据库充当slave角色时是只读的不能进行写操作。
可以在配置文件中开启非只读：slave-read-only no
```

## 复制的过程原理
1、	当从库和主库建立MS关系后，会向主数据库发送SYNC命令；
2、	主库接收到SYNC命令后会开始在后台保存快照（RDB持久化过程），并将期间接收到的写命令缓存起来；
3、	当快照完成后，主Redis会将快照文件和所有缓存的写命令发送给从Redis；
4、	从Redis接收到后，会载入快照文件并且执行收到的缓存的命令；
5、	之后，主Redis每当接收到写命令时就会将命令发送从Redis，从而保证数据的一致；  
## 无磁盘复制
原理：Redis在与从数据库进行复制初始化时将不会将快照存储到磁盘，而是直接通过网络发送给从数据库，避免了IO性能差问题。  

开启无磁盘复制：repl-diskless-sync yes

## 宕机  
主从复制架构中出现宕机的情况，需要分情况看：
#### 1、从Redis宕机
a)	这个相对而言比较简单，在Redis中从库重新启动后会自动加入到主从架构中，自动完成同步数据；
b)	问题？ 如果从库在断开期间，主库的变化不大，从库再次启动后，主库依然会将所有的数据做RDB操作吗？还是增量更新？（从库有做持久化的前提下）
i.	不会的，因为在Redis2.8版本后就实现了，主从断线后恢复的情况下实现增量复制。
#### 2、主Redis宕机
a)	分以下2步完成
i.	第一步，在从数据库中执行SLAVEOF NO ONE命令，断开主从关系并且提升为主库继续服务；
ii.	第二步，将主库重新启动后，执行SLAVEOF命令，将其设置为其他库的从库，这时数据就能更新回来；
b)	这个手动完成恢复的过程其实是比较麻烦的并且容易出错，有没有好办法解决呢？当前有的，Redis提供的哨兵（sentinel）的功能。


## 哨兵（sentinel）  
#### 作用
哨兵的作用就是对Redis的系统的运行情况的监控，它是一个独立进程。它的功能有2个：  
1. 监控主数据库和从数据库是否运行正常；
2. 主数据出现故障后自动将从数据库转化为主数据库；

#### 原理
* 单哨兵  
监控主从数据库
* 多哨兵    
多个哨兵，不仅同时监控主从数据库，而且哨兵之间互为监控。

#### 配置
* 启动哨兵进程  
1. 首先需要创建哨兵配置文件：  
vim sentinel.conf  
2. 输入内容：
单哨兵：
sentinel monitor melodyMaster 127.0.0.1 6379 1
配置多个哨兵：
sentinel monitor melodyMaster 127.0.0.1 6381 2
sentinel monitor melodyMaster 127.0.0.1 6381 1


* 说明：  
melodyMaster：监控主数据的名称，自定义即可，可以使用大小写字母和“.-_”符号  
127.0.0.1：监控的主数据库的IP  
6379：监控的主数据库的端口  
1：最低通过票数  

* 启动
redis-sentinel ./sentinel.conf  

* 启动后  
1. 哨兵已经启动，它的id为9059917216012421e8e89a4aa02f15b75346d2b7  
2. 为master数据库添加了一个监控  
3. 发现了2个slave（由此可以看出，**哨兵无需配置slave，只需要指定master，哨兵会自动发现slave**）  

* 宕机  
slave宕机
slave启动会重新加入到了主从复制中。-sdown：说明是恢复服务。
```流程
2989:X 05 Jun 20:16:50.300 # +sdown master taotaoMaster 127.0.0.1 6379  说明master服务已经宕机
2989:X 05 Jun 20:16:50.300 # +odown master taotaoMaster 127.0.0.1 6379 #quorum 1/1  
2989:X 05 Jun 20:16:50.300 # +new-epoch 1
2989:X 05 Jun 20:16:50.300 # +try-failover master taotaoMaster 127.0.0.1 6379  开始恢复故障
2989:X 05 Jun 20:16:50.304 # +vote-for-leader 9059917216012421e8e89a4aa02f15b75346d2b7 1  投票选举哨兵leader，现在就一个哨兵所以leader就自己
2989:X 05 Jun 20:16:50.304 # +elected-leader master taotaoMaster 127.0.0.1 6379  选中leader
2989:X 05 Jun 20:16:50.304 # +failover-state-select-slave master taotaoMaster 127.0.0.1 6379 选中其中的一个slave当做master
2989:X 05 Jun 20:16:50.357 # +selected-slave slave 127.0.0.1:6381 127.0.0.1 6381 @ taotaoMaster 127.0.0.1 6379  选中6381
2989:X 05 Jun 20:16:50.357 * +failover-state-send-slaveof-noone slave 127.0.0.1:6381 127.0.0.1 6381 @ taotaoMaster 127.0.0.1 6379  发送slaveof no one命令
2989:X 05 Jun 20:16:50.420 * +failover-state-wait-promotion slave 127.0.0.1:6381 127.0.0.1 6381 @ taotaoMaster 127.0.0.1 6379   等待升级master
2989:X 05 Jun 20:16:50.515 # +promoted-slave slave 127.0.0.1:6381 127.0.0.1 6381 @ taotaoMaster 127.0.0.1 6379  升级6381为master
2989:X 05 Jun 20:16:50.515 # +failover-state-reconf-slaves master taotaoMaster 127.0.0.1 6379
2989:X 05 Jun 20:16:50.566 * +slave-reconf-sent slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster 127.0.0.1 6379
2989:X 05 Jun 20:16:51.333 * +slave-reconf-inprog slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster 127.0.0.1 6379
2989:X 05 Jun 20:16:52.382 * +slave-reconf-done slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster 127.0.0.1 6379
2989:X 05 Jun 20:16:52.438 # +failover-end master taotaoMaster 127.0.0.1 6379 故障恢复完成
2989:X 05 Jun 20:16:52.438 # +switch-master taotaoMaster 127.0.0.1 6379 127.0.0.1 6381  主数据库从6379转变为6381
2989:X 05 Jun 20:16:52.438 * +slave slave 127.0.0.1:6380 127.0.0.1 6380 @ taotaoMaster 127.0.0.1 6381  添加6380为6381的从库
2989:X 05 Jun 20:16:52.438 * +slave slave 127.0.0.1:6379 127.0.0.1 6379 @ taotaoMaster 127.0.0.1 6381  添加6379为6381的从库
2989:X 05 Jun 20:17:22.463 # +sdown slave 127.0.0.1:6379 127.0.0.1 6379 @ taotaoMaster 127.0.0.1 6381 发现6379已经宕机，等待6379的恢复
```

## 集群（分片）

即使有了主从复制，每个数据库都要保存整个集群中的所有数据，容易形成**木桶效应**。　

twemproxy代理

### 架构

(1)所有的redis节点彼此互联(PING-PONG机制),内部使用二进制协议优化传输速度和带宽.
(2)节点的fail是通过集群中超过半数的节点检测失效时才生效.
(3)客户端与redis节点直连,不需要中间proxy层.客户端不需要连接集群所有节点,连接集群中任何一个可用节点即可
(4)redis-cluster把所有的物理节点映射到[0-16383]slot（插槽）上,cluster 负责维护node<->slot<->value

### 配置
1、	设置不同的端口，6379、6380、6381
2、	开启集群，cluster-enabled yes
3、	指定集群的配置文件，cluster-config-file "nodes-xxxx.conf"

###  创建集群

####  安装ruby环境

#### 创建集群

执行命令：  
```java
./redis-trib.rb create --replicas 0 192.168.56.102:6379 192.168.56.102:6380 192.168.56.102:6381  

//--replicas 0：指定了从数据的数量为 0 

注意：这里不能使用127.0.0.1，否则在Jedis客户端使用时无法连接到！  

客户端跟踪重定向:
redis-cli -c 
```

### 插槽的分配
通过 cluster nodes 命令可以查看当前集群的信息：  
该信息反映出了集群中的每个节点的id、身份、连接数、插槽数等。

./redis-trib.rb 脚本实现了是将16384个插槽平均分配给了N个节点。

注意：如果插槽数有部分是没有指定到节点的，那么这部分插槽所对应的key将不能使用

Redis 集群中内置了 **16384** 个哈希槽，当需要在 Redis 集群中放置一个 key-value 时，redis 先对 key 的有效部分使用 crc16 算法算出一个结果，然后把结果对 16384 求余数，得到插槽值。

这样每个 key 都会对应一个编号在 0-16383 之间的哈希槽，redis 会根据节点数量大致均等的将哈希槽映射到不同的节点。  
### 新增集群节点  
执行脚本：
./redis-trib.rb add-node 192.168.56.102:6382 192.168.56.102:6379
### 插槽分配
执行脚本：./redis-trib.rb reshard 192.168.56.102:6380 
移动插槽数
接收id
提取的节点id，所有all

### 删除节点
1、首先转移节点的所有插槽
2、	使用redis-trib.rb删除节点
a)	./redis-trib.rb del-node 192.168.56.102:6380 4a9b8886ba5261e82597f5590fcdb49ea47c4c6c
b)	del-node host:port node_id 

## 故障转移
### 故障机制
1. 集群中的每个节点都会定期的向其它节点发送PING命令，并且通过有没有收到回复判断目标节点是否下线；
2. 集群中每一秒就会随机选择5个节点，然后选择其中最久没有响应的节点放PING命令；
3. 如果一定时间内目标节点都没有响应，那么该节点就认为目标节点疑似下线；
4. 当集群中的节点超过半数认为该目标节点疑似下线，那么该节点就会被标记为下线；
5. 当集群中的任何一个节点下线，就会导致插槽区有空档，不完整，那么该**集群将不可用**；
6. 如何解决上述问题？
	a)	在Redis集群中可以使用主从模式实现某一个节点的高可用
	b)	当该节点（master）宕机后，**集群**会将该节点的从数据库（slave）转变为（master）继续完成集群服务；（所以集群不需要哨兵机制）

#### 主从集群
创建集群，指定了从库数量为1，创建顺序为主库（3个）、从库（3个）：
./redis-trib.rb create --replicas 1 192.168.56.102:6379 192.168.56.102:6380 192.168.56.102:6381 192.168.56.102:6479 192.168.56.102:6480 192.168.56.102:6481


#### 使用集群需要注意的事项(可能是版本低的问题)
1、	多键的命令操作（如MGET、MSET），如果每个键都位于同一个节点，则可以正常支持，否则会提示错误。
2、	集群中的节点只能使用0号数据库，如果执行SELECT切换数据库会提示错误。

多路 I/O 复用模型
“多路”指的是多个网络连接，“复用”指的是复用同一个线程。采用多路 I/O 复用技术可以让单个线程高效的处理多个连接请求（尽量减少网络 IO 的时间消耗），且 Redis 在内存中操作数据的速度非常快，也就是说内存内的操作不会成为影响Redis性能的瓶颈



### 数据分布优化方案

区间划分

取模

一致性 hash 方案

预分配hash槽

字符串 hash

权重 1 1

# redis持久化机制

redis提供了两种持久化策略

## RDB

RDB的持久化策略： 按照规则定时讲内存的数据同步到磁盘

snapshot

redis在指定的情况下会触发快照

1. 自己配置的快照规则

*save <seconds> <changes>*

*save 900 1*  *当在900秒内被更改的key的数量大于1的时候，就执行快照*

*save 300 10*

*save 60 10000*

2. save或者bgsave

save: 执行内存的数据同步到磁盘的操作，这个操作会阻塞客户端的请求

bgsave: 在后台异步执行快照操作，这个操作不会阻塞客户端的请求

3. 执行flushall的时候

清除内存的所有数据，只要快照的规则不为空，也就是第一个规则存在。那么redis会执行快照

4. 执行复制的时候

### 快照的实现原理

1. redis使用 fork 函数复制一份当前进程的副本(子进程)

2. 父进程继续接收并处理客户端发来的命令，而子进程开始将内存中的数据写入硬盘中的临时文件

3. 当子进程写入完所有数据后会用该临时文件替换旧的RDB文件，至此，一次快照操作完成。  

 注意：redis在进行快照的过程中不会修改RDB文件，只有快照结束后才会将旧的文件替换成新的，也就是说任何时候RDB文件都是完整的。 这就使得我们可以通过定时备份RDB文件来实现redis数据库的备份， RDB文件是经过压缩的二进制文件，占用的空间会小于内存中的数据，更加利于传输。

### RDB的优缺点

1. 使用RDB方式实现持久化，一旦Redis异常退出，就会**丢失**最后一次快照以后更改的所有**数据**。这个时候我们就需要根据具体的应用场景，通过组合设置自动快照条件的方式来将可能发生的数据损失控制在能够接受范围。如果数据相对来说比较重要，希望将损失降到最小，则可以使用AOF方式进行持久化

2. RDB可以最大化Redis的性能：父进程在保存RDB文件时唯一要做的就是fork出一个子进程，然后这个子进程就会处理接下来的所有保存工作，父进程无序执行任何磁盘I/O操作。同时这个也是一个缺点，如果数据集比较大的时候，fork可以能比较耗时，造成服务器在一段时间内停止处理客户端的请求；

### 实践

修改redis.conf中的appendonly yes ; 重启后执行对数据的变更命令， 会在bin目录下生成对应的.aof文件， aof文件中会记录所有的操作命令

如下两个参数可以去对 aof 文件做优化

auto-aof-rewrite-percentage 100  表示当前aof文件大小超过上一次aof文件大小的百分之多少的时候会进行重写。如果之前没有重写过，以启动时aof文件大小为准

auto-aof-rewrite-min-size 64mb   限制允许重写最小aof文件大小，也就是文件大小小于64mb的时候，不需要进行优化

## AOF

AOF 可以将 Redis 执行的每一条写命令追加到硬盘文件中，这一过程显然会降低Redis的性能，但大部分情况下这个影响是能够接受的，另外使用较快的硬盘可以提高AOF的性能

### 实践

默认情况下Redis没有开启AOF（append only file）方式的持久化，可以通过appendonly参数启用，在redis.conf中找到 appendonly yes

开启AOF持久化后每执行一条会更改Redis中的数据的命令后，Redis就会将该命令写入硬盘中的AOF文件。AOF文件的保存位置和RDB文件的位置相同，都是通过dir参数设置的，默认的文件名是apendonly.aof. 可以在redis.conf中的属性 appendfilename appendonlyh.aof修改

### aof重写的原理

Redis 可以在 AOF 文件体积变得过大时，自动地在后台对 AOF 进行重写： 重写后的新 AOF 文件包含了恢复当前数据集所需的**最小命令集合**（只保留操作每条数据的最新命令，把冗余命令去掉）。 整个重写操作是绝对安全的，因为 Redis 在创建新 AOF 文件的过程中，会继续将命令追加到现有的 AOF 文件里面，即使重写过程中发生停机，现有的 AOF 文件也不会丢失。 而一旦新 AOF 文件创建完毕，Redis 就会从旧 AOF 文件切换到新 AOF 文件，并开始对新 AOF 文件进行追加操作。AOF 文件有序地保存了对数据库执行的所有写入操作， 这些写入操作以 Redis 协议的格式保存， 因此 AOF 文件的内容非常容易被人读懂， 对文件进行分析（parse）也很轻松

### 同步磁盘数据

redis 每次更改数据的时候， aof 机制都会将命令记录到aof文件，但是实际上由于操作系统的缓存机制，数据并**没有实时写入**到硬盘，而是进入硬盘缓存。再通过硬盘缓存机制去刷新到保存到文件（整个过程有数据丢失的可能）

\# appendfsync always  每次执行写入都会进行同步  ， 这个是最安全但是是效率比较低的方式

appendfsync everysec   每一秒执行

\# appendfsync no  不主动进行同步操作，由操作系统去执行，这个是最快但是最不安全的方式

### aof文件损坏以后如何修复

服务器可能在程序正在对 AOF 文件进行写入时停机， 如果停机造成了 AOF 文件出错（corrupt）， 那么 Redis 在重启时会拒绝载入这个 AOF 文件， 从而确保数据的一致性不会被破坏。

当发生这种情况时， 可以用以下方法来修复出错的 AOF 文件：

1. 为现有的 AOF 文件创建一个备份。

2. 使用 Redis 附带的 redis-check-aof 程序，对原来的 AOF 文件进行修复。
   **redis-check-aof --fix**  去掉有误的语法命令

重启 Redis 服务器，等待服务器载入修复后的 AOF 文件，并进行数据恢复。

## RDB 和 AOF ,如何选择

一般来说,如果对数据的安全性要求非常高的话，应该同时使用两种持久化功能。如果可以承受数分钟以内的数据丢失，那么可以只使用 RDB 持久化。有很多用户都只使用 AOF 持久化， 但并不推荐这种方式： 因为定时生成 RDB 快照（snapshot）非常便于进行数据库备份， 并且 RDB 恢复数据集的速度也要比 AOF 恢复的速度要快 。

**两种持久化策略可以同时使用，也可以使用其中一种。如果同时使用的话， 那么Redis重启时，会优先使用AOF文件来还原数据**

 

# 集群

## 复制（master、slave）

### 配置过程

修改11.140和11.141的redis.conf文件，增加 slaveof masterip masterport

slaveof 192.168.11.138 6379

### 实现原理

1. slave第一次或者重连到master上以后，会向master发送一个SYNC的命令

2. master收到SYNC的时候，会做两件事
   a)     执行bgsave（rdb的快照文件）
   b)    master会把新收到的修改命令存入到缓冲区

* 缺点：
   没有办法对master进行动态选举，因此引入了 sentinel 机制

#### 监控同步过程

replconf listening-port 6379（slave port）

sync

同步时不会阻塞，仍可接收读命令，那怎么避免脏数据？配置 slave-serve-stale-data

### 复制的方式

1. 基于rdb文件的复制（第一次连接或者重连的时候）
   若master禁用rdb持久化方式，仍然会生成 rdb 快照，但时间受连接影响，被动生成

2. 无硬盘复制

   Redis在与从数据库进行复制初始化时将不会将快照存储到磁盘，而是直接通过网络发送给从数据库，避免了IO性能差问题。

   repl-diskless-sync

3. 增量复制
   PSYNC master run id.        offset（缓冲区命令偏移量）

## 哨兵机制

sentinel

1. 监控master和salve是否正常运行

2. 如果master出现故障，那么会把其中一台salve数据升级为master（配置文件的自动修改）

sentinel down-after-milliseconds mymaster 3000

日志：sdown    odown



## 集群（redis3.0以后的功能）

master-slave-sentinel 集群结构，单点存储压力大



**集群不需要哨兵，拥有自动回复的选举处理**

进行分片，每个分片形成 master-slave集群结构，降低单点存储压力



根据 key 的 hash 值取模 服务器的数量 。 

hash

### Redis集群的原理

Redis Cluster中，Sharding 采用 slot(槽 )的概念，一共分成 16384 槽，这有点儿类似前面讲的pre sharding思路。对于每个进入Redis的键值对，根据key进行散列，分配到这16384个slot中的某一个中。使用的hash算法也比较简单，就是CRC16后16384取模。Redis集群中的每个node(节点)负责分摊这16384个slot中的一部分，也就是说，每个 slot 都对应一个 node 负责处理。当动态添加或减少 node 节点时，需要将16384个槽做个再分配，槽中的键值也要迁移。当然，这一过程，在目前实现中，还处于半自动状态，需要人工介入。Redis集群，要保证16384个槽对应的node都正常工作，如果某个node发生故障，那它负责的slots也就失效，整个集群将不能工作。为了增加集群的可访问性，官方推荐的方案是将 node 配置成 主从结构，即一个master主节点，挂n个slave从节点。这时，如果主节点失效，Redis Cluster会根据选举算法从slave节点中选择一个上升为主节点，整个集群继续对外提供服务。这非常类似服务器节点通过 Sentinel 监控架构成 主从结构，只是 Redis Cluster 本身提供了故障转移容错的能力。

 

slot（槽）的概念，在redis集群中一共会有16384个槽，

根据key 的 CRC16 算法，得到的结果再对 16384 进行取模。 假如有3个节点

node1  0 5460

node2  5461 10922

node3  10923 16383

* 节点新增

node4  0-1364,5461-6826,10923-12287

* 删除节点

先将节点的数据移动到其他节点上，然后才能执行删除

 

### 市面上提供了集群方案

1. redis shardding   而且jedis客户端就支持shardding操作  SharddingJedis ； 增加和减少节点的问题； 

   pre shardding：
   3台虚拟机 redis 。但是我部署了9个节点 。每一台部署3个redis增加cpu的利用率
   9台虚拟机单独拆分到9台服务器

2. codis基于redis2.8.13分支开发了一个codis-server，解决了分片数据自动迁移和扩容问题

3. twemproxy  twitter提供的开源解决方案



redis 环境
密码设置  配置 requirepass

### 缺点

Redis分片介绍
分片的一些常见问题：
I.   不能直接对映射到多个实例的数据进行交集操作
2．跨实例不能进行事务
3．采用分片机制后，数据备份将变的复杂
4，添加和删除容量也变的复杂一些



# 常见缓存问题

## Redis 缓存更新策略

1. 先删除缓存，再更新数据库
2. 先更新数据库，更新成功后，再让缓存失效
3. 更新数据时只更新缓存，不更新数据库，然后同步异步调度去批量更新数据库

## 缓存击穿

### 缓存雪崩

雪崩是指当大量缓存失效时，导致大量的请求访问数据库，导致数据库服务器，无法抗住请求或挂掉的情况。

解决方法：

（1）合理规划缓存的失效时间；

​        在缓存的时候给过期时间加上一个随机值

（2）合理评估数据库的负载压力；

（3）对数据库进行过载保护或应用层限流；

（4）多级缓存设计，缓存高可用。



热点数据，key 设置 expire，时间排序，部分失效

kafaka 收集 key，flink 分析热点，通知监控调控 

互斥锁 setnx



## 缓存穿透

缓存一般是Key，value方式存在，当某一个Key不存在时会查询数据库，假如这个Key，一直不存在，则会频繁的请求数据库，对数据库造成访问压力。

解决方法：

（1）对结果为空的数据也进行缓存，当此key有数据后，清理缓存；  key -> "&&&"

（2）一定不存在的key，采用布隆过滤器，建立一个大的Bitmap中，查询时通过该bitmap过滤。

















 

 


