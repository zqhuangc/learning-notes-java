ha
shMap/ConcurrentHashMap详解
https://blog.csdn.net/u010292561/article/details/80472555

# Map用法
### 类型介绍
Java 自带了各种 Map 类，这些 Map 类可归为三种类型：
### 通用Map
用于在应用程序中管理映射，通常在 java.util 程序包中实现
HashMap、Hashtable、Properties、LinkedHashMap、IdentityHashMap、TreeMap、WeakHashMap、ConcurrentHashMap
### 专用Map
通常我们不必亲自创建此类Map，而是通过某些其他类对其进行访问
```
java.util.jar.Attributes、
javax.print.attribute.standard.PrinterStateReasons、
java.security.Provider、
java.awt.RenderingHints、
javax.swing.UIDefaults
```
### 自行实现Map
一个用于帮助我们实现自己的Map类的抽象类
AbstractMap

# 常用Map
## HashMap：
最常用的Map,它根据键的HashCode 值存储数据,根据键可以直接获取它的值，具有很快的访问速度。HashMap最多只允许一条记录的键为Null(多条会覆盖);允许多条记录的值为 Null。非同步的。
index =  HashCode（Key） &  （Length - 1） 
## TreeMap：
能够把它保存的记录根据键(key)排序,默认是按升序排序，也可以指定排序的比较器，当用Iterator 遍历TreeMap时，得到的记录是排过序的。TreeMap不允许key的值为null。非同步的。
## Hashtable：
与 HashMap类似,不同的是:key和value的值均不允许为null;它支持线程的同步，即任一时刻只有一个线程能写Hashtable,因此也导致了Hashtale在写入时会比较慢。  
  Entry数组
## LinkedHashMap：
保存了记录的插入顺序，在用Iterator遍历LinkedHashMap时，先得到的记录肯定是先插入的.在遍历的时候会比HashMap慢。key和value均允许为空，非同步的。
 LinkedHashMap 与 LRU(Least recently used，最近最少使用)算法
 使用LinkedHashMap实现LRU的必要前提是将accessOrder标志位设为true以便开启按访问顺序排序的模式。
### Map 遍历
#### 增强for循环遍历：
* 使用keySet()遍历
* 使用entrySet()遍历
#### 迭代器遍历：
* 使用keySet()遍历
* 使用entrySet()遍历
#### 总结
增强for循环使用方便，但性能较差，不适合处理超大量级的数据。  
迭代器的遍历速度要比增强for循环快很多，是增强for循环的2倍左右。
