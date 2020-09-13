## HashMap
* 源码认识
```
   数据结构：
        jdk1.7以前->位桶+链表
        // 默认的初始容量是16，必须是2的幂。
        jdk1.8->位桶+链表+红黑树
```

HashMap本质是一个数组，数组的每个元素都是一个单链表。
```
transient Node<K,V>[] table;//table数组，每个数组元素都是一个链表，链表由0个或多个节点组成
```
### JDK1.7以前
```
// 存储数据的Entry数组，长度是2的幂。
// HashMap是采用拉链法实现的，每一个Entry本质上是一个单向链表
采用头插法
```

### JDk1.8

尾插法

* 节点类
```Java
//静态内部类的特点：在创建静态内部类的实例时，不必创建外部类的实例
static class Node<K,V> implements Map.Entry<K,V> {//Entry是Map接口中的一个内部接口
    final int hash;//此节点的哈希值，同一个链表上的哈希值不一定相同
    final K key;//键，不能修改
    V value;//值
    Node<K,V> next;//指向下一个节点
 
    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }
 
    public final K getKey()        { return key; }
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }
 
    public final int hashCode() {//此Node类的hashCode方法
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }
 
    public final V setValue(V newValue) {//重新设置节点Value，返回旧Value
        V oldValue = value;
        value = newValue;
        return oldValue;
    }
 
    public final boolean equals(Object o) {//判断节点相等的方法，
        if (o == this)//同一个对象，返回true
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;//键和值都相等则返回true
        }
        return false;
    }
}

```

## HashMap常用方法
### hash()

（低16位 & 高16位 ）提高离散率

```Java
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

static int indexFor(int h, int length) {
    return h & (length-1);
}

```
### get()

```Java
//获取键对应的值
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
//根据哈希值和键获取对应的值对象
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (first = tab[(n - 1) & hash]) != null) {//先通过hash值找到对应的链表头节点
        if (first.hash == hash && // always check first node
            ((k = first.key) == key || (key != null && key.equals(k))))//如果要找的key和第一个node的key是同一个对象or equals，则返回第一个节点
            return first;
        if ((e = first.next) != null) {
            if (first instanceof TreeNode)//如果此链表节点类型为红黑树节点，则以遍历红黑树的方式搜索节点
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
            do {//遍历链表
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))//如果hash值相等且key对象是同一个或equals，返回
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}

```

*HashMap在JDK1.8中新增的操作：桶的树形化——在Java 8 中，如果一个桶中的元素个数超过 TREEIFY_THRESHOLD(默认是 8 )，就使用红黑树来替换链表，从而提高速度*
```java
//一个桶的树化阈值
//当桶中元素个数超过这个值时，需要使用红黑树节点替换链表节点
//这个值必须为 8，要不然频繁转换效率也不高
static final int TREEIFY_THRESHOLD = 8;
 
//一个树的链表还原阈值
//当扩容时，桶中元素个数小于这个值，就会把树形的桶元素 还原（切分）为链表结构
//这个值应该比上面那个小，至少为 6，避免频繁转换
static final int UNTREEIFY_THRESHOLD = 6;
 
//哈希表的最小树形化容量
//当哈希表中的容量大于这个值时，表中的桶才能进行树形化
//否则桶内元素太多时会扩容，而不是树形化
//为了避免进行扩容、树形化选择的冲突，这个值不能小于 4 * TREEIFY_THRESHOLD
static final int MIN_TREEIFY_CAPACITY = 64;

```

### put()
```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}
 
// 第三个参数 onlyIfAbsent 如果是 true，那么只有在不存在该 key 时才会进行 put 操作
// 第四个参数 evict 我们这里不关心
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
               boolean evict) {
    Node<K,V>[] tab; Node<K,V> p; int n, i;
    // 第一次 put 值的时候，会触发下面的 resize()，类似 java7 的第一次 put 也要初始化数组长度
    // 第一次 resize 和后续的扩容有些不一样，因为这次是数组从 null 初始化到默认的 16 或自定义的初始容量
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
    // 找到具体的数组下标，如果此位置没有值，那么直接初始化一下 Node 并放置在这个位置就可以了
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
 
    else {// 数组该位置有数据
        Node<K,V> e; K k;
        // 首先，判断该位置的第一个数据和我们要插入的数据，key 是不是"相等"，如果是，取出这个节点
        if (p.hash == hash &&
            ((k = p.key) == key || (key != null && key.equals(k))))
            e = p;
        // 如果该节点是代表红黑树的节点，调用红黑树的插值方法，本文不展开说红黑树
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
            // 到这里，说明数组该位置上是一个链表
            for (int binCount = 0; ; ++binCount) {
                // 插入到链表的最后面(Java7 是插入到链表的最前面)
                if ((e = p.next) == null) {
                    p.next = newNode(hash, key, value, null);
                    // TREEIFY_THRESHOLD 为 8，所以，如果新插入的值是链表中的第 9 个
                    // 会触发下面的 treeifyBin，也就是将链表转换为红黑树
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                        treeifyBin(tab, hash);
                    break;
                }
                // 如果在该链表中找到了"相等"的 key(== 或 equals)
                if (e.hash == hash &&
                    ((k = e.key) == key || (key != null && key.equals(k))))
                    // 此时 break，那么 e 为链表中[与要插入的新值的 key "相等"]的 node
                    break;
                p = e;
            }
        }
        // e!=null 说明存在旧值的key与要插入的key"相等"
        // 对于我们分析的put操作，下面这个 if 其实就是进行 "值覆盖"，然后返回旧值
        if (e != null) {
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
    ++modCount;//modCount用于记录HashMap的修改次数
/**由于HashMap不是线程安全的,所以在迭代的时候,会将modCount赋值到迭代器的expectedModCount属性中,然后进行迭代,
*如果在迭代的过程中HashMap被其他线程修改了,modCount的数值就会发生变化,
*这个时候expectedModCount和ModCount不相等,
*迭代器就会抛出ConcurrentModificationException()异常
**/
    if (++size > threshold)//数组大小到达临界值则会double size
        resize();
    afterNodeInsertion(evict);// Callbacks to allow LinkedHashMap post-actions
    return null;//key存在时返回oldValue,不存在时返回null
}

```

+ 扩容 
  - 初始容量2^n和加载因子(默认0.75) 
  - 容量是哈希表中桶的数量，初始容量只是哈希表在创建时的容量。
  - 加载因子是哈希表在其容量自动增加之前可以达到多满的一种尺度。  
    

*当哈希表中条目的数目超过 容量乘加载因子 的时候，则要对该哈希表进行rehash操作，从而哈希表将具有大约两倍的桶数。（以上摘自JDK6）*

> 和 Java7 稍微有点不一样的地方就是，Java7 是先扩容后插入新值的，Java8 先插值再扩容，不过这个不重要。

### resize()

resize() 方法用于初始化数组或数组扩容，每次扩容后，容量为原来的 2 倍，并进行数据迁移。

```Java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) { // 对应数组扩容
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 将数组大小扩大一倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                 oldCap >= DEFAULT_INITIAL_CAPACITY)
            // 将阈值扩大一倍
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // 对应使用 new HashMap(int initialCapacity) 初始化后，第一次 put 的时候
        newCap = oldThr;
    else {// 对应使用 new HashMap() 初始化后，第一次 put 的时候
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
 
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                  (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
 
    // 用新的数组大小初始化新的数组
    Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab; // 如果是初始化数组，到这里就结束了，返回 newTab 即可
 
    if (oldTab != null) {
        // 开始遍历原数组，进行数据迁移。
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                // 如果该数组位置上只有单个元素，那就简单了，简单迁移这个元素就可以了
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                // 如果是红黑树，具体就不展开了
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else { 
                    // 这块是处理链表的情况，
                    // 需要将此链表拆成两个链表，放到新的数组中，并且保留原来的先后顺序
                    // loHead、loTail 对应一条链表，hiHead、hiTail 对应另一条链表，代码还是比较简单的
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 判断数据位置会不会移动，例:10000,hash变化主要多了一位
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    if (loTail != null) {
                        loTail.next = null;
                        // 第一条链表
                        newTab[j] = loHead;
                    }
                    if (hiTail != null) {
                        hiTail.next = null;
                        // 第二条链表的新的位置是 j + oldCap，这个很好理解
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}

```




## 高并发问题
多个线程进行resize(),resize包含扩容和reHash两个步骤，ReHash在并发的情况下可能会形成链表环。

数据迁移：头插法易造成的问题
Entry2.next = Entry3
Entry3.next = Entry2