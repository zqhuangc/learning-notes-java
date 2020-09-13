

**serializable，cloneable，hashcode， equals**

Arrays.copyOf

System.arraycopy

Fail-Fast()， Fail-Safe(CopyOnWrite)，遍历时操作数据

long  4字节，4字节

Deque（LinkedList）

是否可为 null

TreeMap 二叉索引

LinkedHashMap   accessOrder

[一致性hash](https://github.com/Jaskey/ConsistentHash,)

[RendezvousHash](https://github.com/clohfink/RendezvousHash)



## 总览

[官方文档](https://docs.oracle.com/javase/8/docs/technotes/guides/collections/overview.html)

+ 主要优势
  - Reduces programming effort（减少编程的负担）
  - Increases performance（提升性能）
  - Provides interoperability between unrelated APIs（提供⽆无关 API 之间的互⽤用性）
  - Reduces the effort required to learn APIs（减少学习 API 的负担）
  - Reduces the effort required to design and implement APIs（减少设计与实现 API 的负担）
  - Fosters software reuse（促进软件重⽤用）



+ 基本组成

  - Collection interfaces（集合接口）

  - Infrastructure（基础设施）

  - General-purpose implementations（通用实现）

  - Abstract implementations（抽象实现）

  - Legacy implementations（遗留实现）

  - Convenience implementations（便利实现）

  -  Wrapper implementations（包装实现）

  - Special-purpose implementations（特殊实现）

  - Array Utilities（数组工具类）

+ Algorithms（算法）
+ Concurrent implementations（并发实现）

资源推荐
《Java Generics and Collections》O’Reilly, October 2006, 0-596-52775-6

[Cheat Sheet](https://zeroturnaround.com/rebellabs/Java-Collections-Cheat-Sheet/)



## 内建实现

### 集合实现

#### 遗留实现

* java.util.Vector
* java.util.Stack
* java.util.Hashtable
* java.util.Enumeration
* java.util.BitSet

#### 通用接口

基于 java.util.Collection 接口

java.util.List
java.util.Set
java.util.SortedSet
java.util.NavigableSet（since Java 1.6）

java.util.Queue（since Java 1.5）
java.util.Deque（since Java 1.6）

基于 java.util.Map 接口或其他接口
java.util.SortedMap
java.util.NavigableMap（since Java 1.6）



#### 通用实现

![](https://ws1.sinaimg.cn/large/006xzusPly1g5tiymoxiij30la06ijsd.jpg)

#### 并发接口

基于 java.util.Collection 接口

java.util.concurrent.BlockingQueue（since Java 1.5）
java.util.concurrent.BlockingDeque（since Java 1.6）
java.util.concurrent.TransferQueue （since Java 1.7）

基于 java.util.Map 接口或其他接口

java.util.concurrent.ConcurrentMap（since Java 1.5）
java.util.concurrent.ConcurrentNavigableMap（since Java 1.6）

#### 并发实现
java.util.concurrent.LinkedBlockingQueue
java.util.concurrent.ArrayBlockingQueue
java.util.concurrent.PriorityBlockingQueue
java.util.concurrent.DelayQueue
java.util.concurrent.SynchronousQueue


java.util.concurrent.LinkedBlockingDeque
java.util.concurrent.LinkedTransferQueue
java.util.concurrent.CopyOnWriteArrayList
java.util.concurrent.CopyOnWriteArraySet
java.util.concurrent.ConcurrentSkipListSet


java.util.concurrent.ConcurrentHashMap
java.util.concurrent.ConcurrentSkipListMap



### 抽象实现

+ 基于 java.util.Collection 接口
  - java.util.AbstractCollection
    - java.util.AbstractList
    - java.util.AbstractSet
    - java.util.AbstractQueue（Since Java 1.5）

+ 基于 java.util.Map 接口
  - java.util.AbstractMap



## 高级应用

并发问题，位运算Modifier，color

#### Convenience implementations（便利实现）

* 基本介绍 
  High-performance "mini-implementations" of the collection interfaces.
  高性能的最小化的实现接口

* 主要入口
  * Java 9 前 ：java.util.Collections、java.util.Arrays、java.util.BitSet、java.util.EnumSet
  * Java 9 +：java.util.List、java.util.Set、java.util.Map


#### 接口类型

- 单例集合接口（Collections.singleton\*）

  设计原则：不变集合（Immutable Collection）

  * List: Collections.singletonList(T)
  * Set: Collections.singleton(T)
  * Map: Collections.singletonMap(K,V)

- 空集合接口（Collections.empty\*）

  > 集合方法入参优先：Iterable  -->  Collection  --> List/Set，禁止使用具体类型，（越具体，越难通用）
  >
  > 对自己：不返回 null，对别人：校验是否为 null

  * 枚举：Collections.emptyEnumeration()
  * 迭代器器：emptyIterator()、emptyListIterator()
  * List：emptyList()
  * Set: emptySet()、emptySortedSet()、emptyNavigableSet()
  * Map：emptyMap()、emptySortedMap()、emptyNavigableMap()

- 转换集合接口（Collections.\*、Arrays.\*）

  * Enumeration: Collections.enumeration(Collection)
  * List: Collections.list(Enumeration<T>)、Arrays.asList(T…)
  * Set: Collections.newSetFromMap(Map<E, Boolean>)
  * Queue: Collections.asLifoQueue(Deque<T>)
  * HashCode: Arrays.hashCode(…)
  * String: Arrays.toString(…)

- 列举集合接口（*.of(…)）

  * java.util.BitSet.valueOf(…)
  * java.util.EnumSet.of(…) （Since 1.5）
  * java.util.Stream.of(…) （Since 1.8）
  * java.util.List.of(…) （Since 9）
  * java.util.Set.of(…) （Since 9）
  * java.util.Map.of(…) （Since 9）



#### Wrapper implementations（包装实现）

* 基本介绍
  Add functionality, such as synchronization, to other implementations.
  功能性添加，比如同步以及其他实现
* 设计原则：Wrapper 模式原则，入参集合类型与返回类型相同或者其子类
* 主要入口：java.util.Collections

* 包装接口类型
  * 同步包装接口（java.util.Collections.synchronized\*）
  * 只读包装接口（java.util.Collections.unmodifiable\*）
  * 类型安全包装接口（java.util.Collections.checked\*) 运行时检查类型
    泛型编译时检查类型



#### Special-purpose implementations（特殊实现）

* 基本介绍
  Implementations designed for use in special situations. These implementations display
  nonstandard performance characteristics, usage restrictions, or behavior.
  为特殊场景设计实现，这些实现表现出非标准性能特征、使用限制或者行为。



##### 示例

* 弱引用 Map
  - java.util.WeakHashMap
  - java.lang.ThreadLocal.ThreadLocalMap

* 对象鉴定 Map
  - java.util.IdentityHashMap 系统 Identityhashcode

* 优先级 Queue
  - java.util.PriorityQueue

* 枚举 Set
  - java.util.EnumSet

expungeStaleEntries 淘汰引用

hashcode是一个创建方法，视作数据来源

equals 是比较方法，比较成员内容



## 算法

+ 排序算法
  - Java 排序算法实现
  - 二分搜索算法
  -  Java 二分搜索算法实现

分类
• 计算复杂度：最佳、最坏以及平均复杂度
• 内存使用：空间复杂度
• 递归算法：排序算法中是否用到了递归
• 稳定性：当相同的健存在时，经过排序后，其值也保持相对的顺序（不发生变化）
• 比较排序：集合中的两个元素进行比较排序

分类
• 串行或并行：是否运用串行或并行排序



### 比较排序

时间复杂度

- 冒泡排序（Bubble Sort）：最佳 O(n)、平均 O(n^2)、最坏 O(n^2)
- 插入排序（Insertion Sort）：最佳 O(n)、平均 O(n^2)、最坏 O(n^2)
- 快速排序（Quick Sort）：最佳 O(nlogn)、平均 O(nlogn)、最坏 O(n^2)
  步骤:（分区索引的选取）
  - 获取 pivot（轴）
  - 分区（Partitioning）
  - 递归执行

+ 合并排序（Merge Sort）：最佳 O(nlogn)、平均 O(nlogn)、最坏 O(nlogn)

  步骤:

  - 分块（Divide）
  - 递归合并（Conquer）

+ Tim 排序（TimSort）：最佳 O(n)、平均 O(nlogn)、最坏 O(nlogn)

  - 继承：合并排序和插⼊入排序



 #### 内建实现
+ 冒泡排序（Bubble Sort）：无

+ 插⼊入排序（Insertion Sort）：java.util.Arrays#mergeSort （当排序集合数量量⼩小于 7 时）

+ 快速排序（Quick Sort）：java.util.DualPivotQuicksort#sort（Since 1.7）

+ 合并排序（Merge Sort）：java.util.Arrays#mergeSort （1.7 之后需要激活）

+ Tim 排序（TimSort）：java.util.TimSort（Since 1.7）

### 二分搜索（binarySearch）

* 步骤

1. 假设 A 表示数组，V 表示搜索的数据，设置 L（低位） = 0，H（⾼高位）= 数组⻓长度 - 1
2. 如果 L > H 的话，搜索结束
3. 设置 M（中位） = L + H / 2
4. 如果 A[M] < V，设置 L = M + 1，重新执⾏行行步骤⼆二
5. 如果 A[M] > V，设置 H = M - 1，重新执⾏行行步骤⼆二
6. 当 A[M] 等于 V 时，搜索结束，返回 M

* 类比算法
  - Hash 算法
  - Trees 算法
  - 线性搜索

 #### 内建实现
java.util.Arrays#binarySearch
java.util.Collections#binarySearch
java.util.TreeMap





snapshot 快照  只读

StringTokenizer  implement enumeration（迭代早期实现）

assertation  java1.4 断言

Arrays.asList 数据可交换，不可写操作



数组 copy 类似 clone浅复制 



为什么会设计这样的接口、方法？？？多思考



Iterator 仅限在 next() 后 本身的操作才成功，近似只读的





https://www.journaldev.com/1330