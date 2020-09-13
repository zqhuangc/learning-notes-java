# ConcurrentHashMap

锁分段技术（java1.8之前）：HashTable容器在竞争激烈的并发环境下表现出效率低下的原因，是因为所有访问HashTable的线程都必须竞争同一把锁，那假如容器里有多把锁，每一把锁用于锁容器其中一部分数据，那么当多线程访问容器里不同数据段的数据时，线程间就不会存在锁竞争，从而可以有效的提高并发访问效率，这就是ConcurrentHashMap所使用的锁分段技术，首先将数据分成一段一段的存储，然后给每一段数据配一把锁，当一个线程占用锁访问其中一个段数据的时候，其他段的数据也能被其他线程访问。  
JDK1.8的实现已经摒弃了Segment的概念，而是直接用Node数组+链表+红黑树的数据结构来实现，并发控制使用Synchronized和CAS来操作，整个看起来就像是优化过且线程安全的HashMap，虽然在JDK1.8中还能看到Segment的数据结构，但是已经简化了属性，只是为了兼容旧版本。
* 关键属性
```java
    // ConcurrentHashMap核心数组
    transient volatile Node<K,V>[] table;
 
    // 扩容时才会用的一个临时数组
    private transient volatile Node<K,V>[] nextTable;
    
    /**
     * table初始化 和 resize控制字段
     * 负数表示table正在初始化 或 resize。-1表示正在初始化，-N表示有N-1个线程正在resize操作
     * 当table为null的时候，保存初始化表的大小以用于创建时使用，或者直接采用默认值0
     * table初始化之后，保存下一次扩容的的大小，跟HashMap的threshold = loadFactor*capacity作用相同
     */
    private transient volatile int sizeCtl;
 
    // resize的时候下一个需要处理的元素下标为 index=transferIndex-1
    private transient volatile int transferIndex;
 
    // 通过CAS无锁更新，ConcurrentHashMap元素总数，但不是准确值
    // 因为多个线程同时更新会导致部分线程更新失败，失败时会将元素数目变化存储在counterCells中
    private transient volatile long baseCount;
 
    // resize或者创建CounterCells时的一个标志位
    private transient volatile int cellsBusy;
 
    // 用于存储元素变动
    private transient volatile CounterCell[] counterCells;

```

* CAS（比较并交换）
Unsafe.compareAndSwapXXX方法是jdk.internal.misc.Unsafe类中的方法，Unsafe类用于执行低级别、不安全操作的方法集合。
```java
private static final Unsafe U = Unsafe.getUnsafe();

static final <K,V> boolean casTabAt(Node<K,V>[] tab, int i, Node<K,V> c, Node<K,V> v) {
    return U.compareAndSetObject(tab, ((long)i << ASHIFT) + ABASE, c, v);
}

```
CAS的含义是：“我认为V的值应该是A，如果是，那么将V的值更新为B；否则，不修改并告诉V的值实际是多少"

CAS方法都是native方法，可以保证原子性，并且效率比synchronized高。

## 常用方法

volatile



### spread()  
ConcurrentHashMap中没有hash()方法，取而代之的是spread方法。spread方法解释：

```java
static final int HASH_BITS = 0x7fffffff;//用来屏蔽符号位
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;//先对低16位进行扰动处理，然后屏蔽符号位，结果为32位int型非负数
}
int h = spread(key.hashCode());//调用
e = tabAt(tab, (n - 1) & h)//和hash()一样，不会直接使用，根据数组长度只取低位哈希值

```

### get()  
get操作不需要锁。除非读到的值是空的才会加锁重读。

```java
static final <K,V> Node<K,V> tabAt(Node<K,V>[] tab, int i) {
    return (Node<K,V>)U.getObjectAcquire(tab, ((long)i << ASHIFT) + ABASE);//获取传入的哈希值对应的节点
}
/**
*数组元素定位：
*
*Unsafe类中有很多以BASE_OFFSET结尾的常量，比如ARRAY_INT_BASE_OFFSET，ARRAY_BYTE_BASE_OFFSET等，
*这些常量值是通过arrayBaseOffset方法得到的。arrayBaseOffset方法是一个本地方法，可以获取数组第一个元素的偏移地址。
*Unsafe类中还有很多以INDEX_SCALE结尾的常量，比如 ARRAY_INT_INDEX_SCALE ， ARRAY_BYTE_INDEX_SCALE等，
*这些常量值是通过arrayIndexScale方法得到的。arrayIndexScale方法也是一个本地方法，
*可以获取数组的转换因子，也就是数组中元素的增量地址。
*将arrayBaseOffset与arrayIndexScale配合使用，可以定位数组中每个元素在内存中的位置。
**/
ABASE = U.arrayBaseOffset(Node[].class);//获取数组第一个元素的偏移地址
int scale = U.arrayIndexScale(Node[].class);//获取数组中元素的增量
if ((scale & (scale - 1)) != 0)//scale不是2的整数次方则出错
    throw new Error("array index scale not a power of two");
ASHIFT = 31 - Integer.numberOfLeadingZeros(scale);//scale非0位的位数
 
 
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
        (e = tabAt(tab, (n - 1) & h)) != null) {
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        else if (eh < 0)
            return (p = e.find(h, key)) != null ? p.val : null;
        while ((e = e.next) != null) {
            if (e.hash == h &&
                ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}

```
### put()

不允许 null 值

jdk1.7分段（segment）锁   转为  CAS+synchronize

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}
final V putVal(K key, V value, boolean onlyIfAbsent) {
    // 不允许 null 值
    if (key == null || value == null) throw new NullPointerException();
    // 计算 hash 值
    int hash = spread(key.hashCode());
    // 用于记录相应链表的长度
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // 如果数组"空"，进行数组初始化
        if (tab == null || (n = tab.length) == 0)
            // 初始化数组，后面会详细介绍
            tab = initTable();
 
        // 找该 hash 值对应的数组下标，得到第一个节点 f
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 如果数组该位置为空，
            // 用一次 CAS 操作将这个新值放入其中即可，这个 put 操作差不多就结束了，可以拉到最后面了
            //  如果 CAS 失败，那就是有并发操作，进到下一个循环就好了
            if (casTabAt(tab, i, null,
                         new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // hash 居然可以等于 MOVED，这个需要到后面才能看明白，不过从名字上也能猜到，肯定是因为在扩容
        else if ((fh = f.hash) == MOVED)
            // 帮助数据迁移，这个等到看完数据迁移部分的介绍后，再理解这个就很简单了
            tab = helpTransfer(tab, f);
 
        else { // 到这里就是说，f 是该位置的头结点，而且不为空
 
            V oldVal = null;
            // 获取数组该位置的头结点的监视器锁
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) { // 头结点的 hash 值大于 0，说明是链表
                        // 用于累加，记录链表的长度
                        binCount = 1;
                        // 遍历链表
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 如果发现了"相等"的 key，判断是否要进行值覆盖，然后也就可以 break 了
                            if (e.hash == hash &&
                                ((ek = e.key) == key ||
                                 (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            // 到了链表的最末端，将这个新值放到链表的最后面
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                                          value, null);
                                break;
                            }
                        }
                    }
                    else if (f instanceof TreeBin) { // 红黑树
                        Node<K,V> p;
                        binCount = 2;
                        // 调用红黑树的插值方法插入新节点
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                                       value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            // binCount != 0 说明上面在做链表操作
            if (binCount != 0) {
                // 判断是否要将链表转换为红黑树，临界值和 HashMap 一样，也是 8
                if (binCount >= TREEIFY_THRESHOLD)
                    // 这个方法和 HashMap 中稍微有一点点不同，
                    // 那就是它不是一定会进行红黑树转换，
                    // 如果当前数组的长度小于 64，那么会选择进行数组扩容
                    // 而不是转换为红黑树
                    // 具体源码我们就不看了，扩容部分后面说
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }
    // 记录元素个数，检查当前容量是否需要进行扩容
    addCount(1L, binCount);
    return null;
}



private final void addCount(long x, int check) {
    ... 省略部分代码
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        //当条件满足开始扩容
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
               (n = tab.length) < MAXIMUM_CAPACITY) {
            
            int rs = resizeStamp(n);
            if (sc < 0) {
                //如果小于0说明已经有线程在进行扩容操作了
                //一下的情况说明已经有在扩容或者多线程进行了扩容，其他线程直接break不要进入扩容操作
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                    transferIndex <= 0)
                    break;
                //如果相等说明扩容已经完成，可以继续扩容
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }
            //这个时候sizeCtl已经等于(rs << RESIZE_STAMP_SHIFT) + 2
            //等于一个大的负数，这边加上2很巧妙,因为transfer后面对
            //sizeCtl--操作的时候，最多只能减两次就结束
            else if (U.compareAndSwapInt(this, SIZECTL, sc,
                                         (rs << RESIZE_STAMP_SHIFT) + 2))
                // 扩容
                transfer(tab, null);
            s = sumCount();
        }
    }
}
```

### 初始化 initTable

通过 CAS 保证只有一个线程进行初始化

sizeCtl   sc

- **-1 **代表table正在初始化
- **-N **表示有N-1个线程正在进行扩容操作

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        //如果一个线程发现sizeCtl<0，意味着另外的线程执行CAS操作成功，当前线程只需要让出cpu时间片
        if ((sc = sizeCtl) < 0) 
            Thread.yield(); // lost initialization race; just spin
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    @SuppressWarnings("unchecked")
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    sc = n - (n >>> 2);
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```



### 扩容 transfer

stribe  

```
TREEIFY_THRESHOLD = 8;
UNTREEIFY_THRESHOLD = 6;
MIN_TREEIFY_CAPACITY=64


RESIZE_STAMP_BITS = 16;
RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
```

binCount >= TREEIFY_THRESHOLD(默认是8)，但总节点数超过64才会转红黑树

* ForwardingNode节点，意味有其它线程正在扩容

```java
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }
    //省略部分代码
}
```

* transfer

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    if (nextTab == null) {            // initiating
        try {
            @SuppressWarnings("unchecked")
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;
    //构建一个连节点的指针，用于标识位
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    
    boolean advance = true;
    //循环的关键变量，判断是否已经扩容完成，完成就return，退出循环
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        //循环的关键i，i--操作保证了倒序遍历数组
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                //nextIndex=transferIndex=n=tab.length(默认16)
                i = -1;
                advance = false;
            }
            else if (U.compareAndSwapInt
                     (this, TRANSFERINDEX, nextIndex,
                      nextBound = (nextIndex > stride ?
                                   nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        //i<0说明已经遍历完旧的数组tab；i>=n什么时候有可能呢？在下面看到i=n,所以目前i最大应该是n吧。
        //i+n>=nextn,nextn=nextTab.length，所以如果满足i+n>=nextn说明已经扩容完成
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            if (finishing) {// a
                nextTable = null;
                table = nextTab;
                sizeCtl = (n << 1) - (n >>> 1);
                return;
            }
            //利用CAS方法更新这个扩容阈值，在这里面sizectl值减一，说明新加入一个线程参与到扩容操作,参考sizeCtl的注释
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                //如果有多个线程进行扩容，那么这个值在第二个线程以后就不会相等，因为sizeCtl已经被减1了，所以后面的线程就只能直接返回,始终保证只有一个线程执行了 a(上面注释a)
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT)
                    return;
                finishing = advance = true;//finishing和advance保证线程已经扩容完成了可以退出循环
                i = n; // recheck before commit
            }
        }
        else if ((f = tabAt(tab, i)) == null)//如果tab[i]为null，那么就把fwd插入到tab[i]，表明这个节点已经处理过了
            advance = casTabAt(tab, i, null, fwd);
        else if ((fh = f.hash) == MOVED)//那么如果f.hash=-1的话说明该节点为ForwardingNode，说明该节点已经处理过了
            advance = true; // already processed
        else {
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    if (fh >= 0) {
                        int runBit = fh & n;
                        // 先处理，链表最后一个节点
                        Node<K,V> lastRun = f;
                        //对链表进行遍历,这边的的算法和hashmap的算法又不一样了，这是有点对半拆分的感觉
                        //把链表分表拆分为，hash&n等于0和不等于0的，然后分别放在新表的i和i+n位置
                        //此方法同hashmap的resize
                        for (Node<K,V> p = f.next; p!= null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        // 是否需要迁移
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        //把已经替换的节点的旧tab的i的位置用fwd替换，fwd包含nextTab
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }//下面红黑树基本和链表差不多
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }
                        //判断扩容后是否还需要红黑树结构
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                            (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                            (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}

```



### 帮助数据迁移

helpTransfer

```java
/**
 * Helps transfer if a resize is in progress.
 */
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
        (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
               (sc = sizeCtl) < 0) {
            //下面几种情况和addCount的方法一样，请参考addCount的备注
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```





## 问题

### 锁分段技术

- (1)ConcurrentHashMap的锁分段技术。
  写加锁

### 读是否要加锁

- (2)ConcurrentHashMap的读是否要加锁，为什么。  
  不加，CAS
- (3)ConcurrentHashMap的迭代器是强一致性的迭代器还是弱一致性的迭代器。

### 弱一致性

ConcurrentHashMap使用了不同于传统集合的快速失败迭代器（见之前的文章《JAVA API备忘---集合》）的另一种迭代方式，我们称为弱一致迭代器。在这种迭代方式中，当iterator被创建后集合再发生改变就不再是抛出ConcurrentModificationException，取而代之的是在改变时new新的数据从而不影响原有的数据，iterator完成后再将头指针替换为新的数据，这样iterator线程可以使用原来老的数据，而写线程也可以并发的完成改变，更重要的，这保证了多个线程并发执行的连续性和扩展性，是性能提升的关键。



## **弱一致性**（jdk1.7）

ConcurrentHashMap是弱一致的。 ConcurrentHashMap的get，containsKey，clear，iterator 都是弱一致性的。

get和containsKey都是无锁的操作，均通过getObjectVolatile()提供的原子读语义来获得Segment以及对应的链表，然后遍历链表。由于遍历期间其他线程可能对链表结构做了调整，因此get和containsKey返回的可能是过时的数据。如在get执行UNSAFE.getObjectVolatile(segments, u)之后，其他线程若执行了clear操作，则get将读到失效的数据。

由于clear没有使用全局的锁，当清除完一个segment之后，开始清理下一个segment的时候，已经清理segments可能又被加入了数据，因此clear返回的时候，ConcurrentHashMap中是可能存在数据的。因此，clear方法是弱一致的。

ConcurrentHashMap中的迭代器主要包括entrySet、keySet、values方法，迭代器在遍历期间如果已经遍历的table上的内容变化了，迭代器不会抛出ConcurrentModificationException异常。如果未遍历的数组上的内容发生了变化，则有可能反映到迭代过程中，这就是ConcurrentHashMap迭代器弱一致的表现。



## [Bug](https://mp.weixin.qq.com/s/Lw793fGWdpv5BHEAE38xHA)



# 原因

```
map.computeIfAbsent(key1, mappingFunction)
```

如果当前key1-hash对应的tab位(可以理解为槽)刚好是空的,在计算mappingFunction之前会

- step1: 先往对应位置放一个ReservationNode占位
- step2: 然后计算mappingFunction的值value,
- step3: 再将value组装成最终NODE, 把占位的ReservationNode换成最终NODE;

这时如果:
mappingFunction 中用到了 当前map的computeIfAbsent方法, 很不巧 key2-hash的槽为和key1的是同一个,
因为key1已经在槽中放入了占位节点, 在处理key2时候for循环的所以处理条件都不符合 程序进入了死循环