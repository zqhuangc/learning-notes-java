##　本地缓存

### JVM 缓存
只能显式的写入，清除数据。
不能按照一定的规则淘汰数据，如 LRU，LFU，FIFO 等。
清除数据时的回调通知。
其他一些定制功能等。

### Ehcache、Guava Cache

专门用作 JVM 缓存的开源工具
具有上文 JVM 缓存不具有的功能，如自动清除数据、多种清除算法、清除回调



Guava Cache 使用场景：



### 分布式缓存

Google 出的 Guava 是 Java 核心增强的库
核心的数据结构就是按照 ConcurrentHashMap　 jdk1.7（先找到 Segment，再找具体的位置，等于是做了两次 Hash 定位。）
内部会维护两个队列 accessQueue,writeQueue 用于记录缓存顺序，这样才可以按照顺序淘汰数据（类似于利用 LinkedHashMap 来做 LRU 缓存）

Guava Cache 就只支持堆内缓存
* 事件回调  
事件回调其实是一种常见的设计模式
Caller 向 Notifier 提问。
提问方式是异步，接着做其他事情。
Notifier 收到问题执行计算然后回调 Caller 告知结果。


## 数据结构
类似jdk1.7以前concurrenthashmap的segment

## 缓存回收：
* 基于容量回收（Size-based Eviction）  
* 基于时间回收（Timed Eviction）  
  CacheBuilder.expireAfterAccess(duration,unit)：缓存项在给定时间内没有给访问过（包括读写操作）则会被回收，回收方式按照最近最少使用的原则。  
  CacheBuilder.expireAfterWrite (duration, unit)，另外一种方式是在缓存项在给定时间内没有被更新（包括创建和覆盖），则可以回收。  
* 基于引用类型的回收（Reference-based Eviction）：
  CacheBuilder.weakKeys()
  CacheBuilder.weakValues()
  CacheBuilder.softValues()  
* 手动回收方式：  
  回收单个缓存项，Cache.invalidate(key)
  批量回收，Cache.invalidateAll(keys)
  全部回收 Cache.invalidateAll()



####　运维

Guava Cache 还提供了一些运维方法，帮助我们监控缓存运行状使用 CacheBuilder.recordStats()开启统计功能，cache.stats() 返回的 CacheStats 对象来查看运行的统计信息，可以监控好的信息有，主要的统计功能包括：
hitRate(), 缓存命中率。
hitCount()，缓存命中次数。
loadCount()，新值加载次数。
requestCount()，缓存访问次数，是命中次数和非命中次数之和。
averageLoadPenalty(),加载新值时的平均耗时，以纳秒为单位。
evictionCount(), 除了手动清除意外的缓存回收总次数。  


## 并发级别设置：

类似于 ConcurrentHashMap 中的并发级别，Guava Cache 通过改值将内部数据结构拆分成固定数量的段，缓存数据在段中存储，每个段有自己的写锁，段与段之间互不影响，所以并发级别越大，分的段越多，并发能力越强，如果你的服务QPS比较大的花，可以适当调大并发级别，用法如：
CacheBuilder.newBuilder().concurrencyLevel(20)





## 核心类

cachebuilder

cacheloader

localcache 

localmanualcache



* 移除监听 removelistener