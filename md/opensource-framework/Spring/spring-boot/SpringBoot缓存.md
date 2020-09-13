# 缓存

## Java Cache（JSR-107）

缓存是一种久经考验并且显著地提升应用性能以及伸缩性的技术。缓存用作临时存储信息复本，该复本未来可能被再次使用，减少再次加载或创建的成本。

Java Caching API	
​	为Java 程序提供一种通用方式去创建、读取、更新以及删除缓存中的元素。

1.0 规范
发布时间：2013年12月16日

非规范目标
资源和内存限制配置
​	尽管许多缓存实现提供了运行时缓存资源限制能力，不过规范并不会定义功能性的配置。
缓存存储和拓扑结构
​	规范没有规定缓存的实现存储或者信息展示。
管理
​	规范没有规定缓存如何管理，定义了程序化配置缓存的机制以及通过Java Management Extensions（JMX）探测缓存的统计信息。
安全
​	规范没有规定缓存内容如何是否安全或者缓存操作如何控制。
外部资源同步
规范没有规定应用或缓存实现如何保持缓存与外部之间的内容同步。

### 核心接口

#### CachingProvider

​	定义构建、配置、获取、管理和控制零个以上CacheManager的机制，应用程序在运行时也可能读取或使用零个以上的CachingProvider。

#### CacheManager

​	定义构建、配置、获取、管理和控制零个以上不重名的Cache的机制。CacheManager 归属单个CacheProvider。



ConcurrentMap 只保证单操作同步

#### Cache

​	一种类似于Map的数据结构，允许Key-Value临时存储。Cache归属单个CacheManager。
Entry
​	单个 存储在Cache中的键值对。存储在缓存中的每个Entry存在一个持久时间。
ExpiryPolicy
定义Entry的过期策略。

#### 存储方式

- 值存储（Store-By-Value）
  ​	默认机制，在存储Key和Value前，需要实现创建一份复本数据，并且在读取缓存时，同样返回一份复制数据。一种简单键值复本的实现方式为Java序列化。规范推荐自定义Key和Value类均实现标准的Java 序列化，用户也可以自定义实现。
  ​	举例：分布式缓存 - Redis
- 引用存储（Store-By-Reference）
  ​	可选机制，存储与获取 Key 和 Value 的实现通过 Java 引用。
  ​	举例：JVM 本地缓存 - Guava

Cache 与 Map 
类似
缓存值均有关联键来存储
每个值可能仅关联单个键
特别注意key的可变性，可变的key可能会影响键的比较
自定义Key类应该添加合适的Object.hashCode方法

区别
缓存的键和值禁止为null
缓存项可能会过期
缓存项可能被移除
缓存支持Compare-And-Swap（CAS）操作
缓存的键和值可能需要某种方式的序列化

一致性（Consistency） 
非阻塞锁（lock-free）
保障：无法保证
类型：Happen-Before（HB）

乐观锁（optimistic locking）
一致性：保证
类型：Compare-And-Swap（CAS）

消极锁（pessimistic locking）
一致性：保证
类型：lock、mutex

## Spring Cache

从Spring 3.1 开始，Spring Framework 对已有的应用程序增添缓存的支持。类似于事务的支持，在最小影响的代码，缓存抽象允许持续使用各种缓存解决方案。

```
从Spring 4.1开始，缓存抽象对JSR-107注解以及其他自定义操作上得到重要的提升。
```

核心接口
org.springframework.cache.CacheManager
org.springframework.cache.Cache



核心注解
@Cacheable
@CacheEvict
@CachePut
@Caching
@CacheConfig

激活缓存
@EnableCaching



CacheInterceptor 

CacheAspectSupport

