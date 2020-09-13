
## 堆内缓存（on-heap）:

​这种模式使用JVM 中的堆空间来保存缓存对象，是最常见的缓存模式，好处是可以直接存储Java对象数据结构，而不需要强制序列化，前面介绍的 Guava Cache 就只支持堆内缓存，缺点是会挤占JVM内存空间，会引发GC（垃圾回收）的频次和时间会变长。

## 堆外缓存（Off-Heap）

​相比于堆内缓存，堆外缓存使用JVM对以外的内存空间，也就意味着可以使用更大的内存空间，空间大小只受限于本机内地大小，而且不受GC管理(内存对象转移，释放等等。。)，整体上可以减少GC带来的停顿影响，但缺点是，堆外内存中的对象必须要序列化，键和值必须实现Serializable接口，因此存取速度上会比堆内缓存慢。在Java中可以通过 -XX:MaxDirectMemorySize 参数设置堆外内存的上限。
```java
// 堆外内存不能按照存储条目限制，只能按照内存大小进行限制，超过限制则回收缓存
ResourcePoolsBuilder.newResourcePoolsBuilder().offheap(200, MemoryUnit.MB);
```

## 磁盘缓存（Disk）

​缓存数据存储在磁盘上，可以持久化存储数据可以不丢失，也意味着可以使用更大的存储空间，缓存在磁盘上的数据也需要支持序列化，速度会被比内存更慢，在使用时推荐使用更快的磁盘带来更大的吞吐率，比如使用闪存代替机械磁盘。在Ehcache中使用 PersistentCacheManager 来管理磁盘存储，还要指定一个磁盘存储路径



Ehcache支持者三总模式的组合包括：
* heap + offheap
* heap + disk
* heap + offheap + disk  

​上一级比下一级速度更快，另外，下一级比上一级存储空间更大，这也是ehcache的轻质要求，否则会报错。 ehcache 会将最热的数据保存在高一级的缓存

在 ehcache 的多层缓存结构中，**最底层的称之为 Authoritative Tier，其余的缓存层称为 “Caching Tier”**。Authoritative Tier 按照字面意意思可以理解为“权威层”，以为该层数据是最全的，其余层的数据都是该层的数据子集，只是临时存储数据，所以可以列为“缓存层”  
* 对于写操作：在写缓存的时候，Ehcache 直接将数据写到 Authoritative Tier（最底层），同时将 Caching Tier 中的旧数据失效掉。
* 对于读操作：Ehcache 会直接从 Caching Tier 中读取信息（Caching Tier 要比 Authoritative Tier 速度更快），如果读到，则直接返回。如果读不到，则从 Authoritative Tier 中再次读取，如果读到，则先将数据写入 Caching Tier，让后在返回，如果Authoritative Tier 读不到，则返回null。

## 缓存失效：

Ehcache 通过 Expirations 来设置缓存失效时间。Expirations 有三种方式：  
* 永不失效：Expirations.noExpiration();
* TTL（time-to-live）：自缓存项创建之后多久失效，例如：Expirations.timeToLiveExpiration(Duration.of(10,TimeUnit.MINUTES));
* TTI（time-to-idle）：缓存项自上次访问之后空闲（没有被再次访问）多久后失效，例如：Expirations.timeToIdleExpiration(Duration.of(5, TimeUnit.MINUTES));  

Ehcache 中还可以通过自行实现Expiry接口来自定义缓存失效策略。
并发级别设置：
通过 withDispatcherConcurrency设置并发级别，


ehcache用在方法上，在失效时间内，缓存方法返回结果，缓存一般用于查询
spring 配置
cache
cachemanager
cache配置

@Cacheable





ehcachemanagerfactorybean