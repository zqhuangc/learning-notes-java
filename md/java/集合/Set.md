### Navigable接口
### Set接口（无序，不可以重复的集合）
+ HashSet:线程不安全，存取速度快。底层是以hash表实现的。 
  - 数据结构：基于HashMap的key，value为一个固定对象
  - 应用：用于存储无序(存入和取出的顺序不一定相同)元素，值不能重复
+ TreeSet:红-黑树的数据结构(红黑树算法的规则: 左小右大。)，默认对元素进行自然排序（String）。如果在比较的时候两个对象返回值.
  - 数据结构：基于TreeMap的key，value为一个固定对象
  - 应用：可以自然排序,那么TreeSet必定是有排序规则的。
    1. 让存入的元素自定义比较规则。
    2. 给TreeSet指定排序规则 
* LinkedSet应用：会保存插入的顺序。  

*接口Comparable，Comparator*

```
看到Array，就要想到角标。
看到Link，就要想到first，last。
看到Hash，就要想到hashCode,equals.
看到Tree，就要想到两个接口。
```
### HashSet如何检查重复?
当你把对象加入HashSet时，HashSet会先计算对象的hashcode值来判断对象加入的位置，同时也会与其他加入的对象的hashcode值作比较，如果没有相符的hashcode，HashSet会假设对象没有重复出现。  
但是如果发现有相同hashcode值的对象，这时会调用equals（）方法来检查hashcode相等的对象是否真的相同。如果两者相同，HashSet就不会让加入操作成功。。