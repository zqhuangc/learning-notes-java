Unsafe源码 https://my.oschina.net/weichou/blog/704843 
C语言内存屏障和volatile关键字：https://www.cnblogs.com/god-of-death/p/7852394.html
java内存屏障相关：https://www.jianshu.com/p/c9ac99b87d56 
         https://blog.csdn.net/javazejian/article/details/72772470
         https://blog.csdn.net/javazejian/article/details/72772461#可见性

​                                   https://tech.meituan.com/java_memory_reordering.html



https://blog.csdn.net/klordy_123/article/details/82933115

两个核心静态内部类 Segment和HashEntry



![](https://ws1.sinaimg.cn/large/006xzusPgy1g23f89ntifj30dy0exmy7.jpg)

`HashEntry`–> `Segment`–> `ConcurrentHashMap`

## 核心

定位到 segment ，调用 segment 方法操作数据



//依据给定的concurrencyLevel并行度，找到最适合的segments数组的长度，
// 为上文默认并行度参数说明的大于concurrencyLevel的最小的2的n次方

concurrencyLevel   segment内容量

### Segment关键属性

```java
/**
 * scanAndLockForPut中自旋循环获取锁的最大自旋次数。
 */
static final int MAX_SCAN_RETRIES =
    Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;
/**
 * 存储结构
 */
transient volatile HashEntry<K,V>[] table;

/**
 * 元素的个数，这里没有加volatile修饰，所以只能在加锁或者确保可见性(如Unsafe.getObjectVolatile)的情况下进行访问，不然无法保证数据的正确性
 */
transient int count;

/**
 * segment元素修改次数记录，由于未进行volatile修饰，所以访问规则和count类似
 */
transient int modCount;

/**
 * 扩容指标
 */
transient int threshold;
```



### Segment

#### scanAndLockForPut

定位

```java
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    int retries = -1; // negative while locating node
    //如果尝试加锁失败，那么就对segment[hash]对应的链表进行遍历找到需要put的这个entry所在的链表中的位置，
    //这里之所以进行一次遍历找到坑位，主要是为了通过遍历过程将遍历过的entry全部放到CPU高速缓存中，
    //这样在获取到锁了之后，再次进行定位的时候速度会十分快，这是在线程无法获取到锁前并等待的过程中的一种预热方式。
    while (!tryLock()) {
        HashEntry<K,V> f; // to recheck first below
        //获取锁失败，初始时retries=-1必然开始先进入第一个if
        if (retries < 0) {//<1>
            if (e == null) { //<1.1>
                //e=null代表两种意思，第一种就是遍历链表到了最后，仍然没有发现指定key的entry；
                //第二种情况是刚开始时确实太过entryForHash找到的HashEntry就是空的，即通过hash找到的table中对应位置链表为空
                //当然这里之所以还需要对node==null进行判断，是因为有可能在第一次给node赋值完毕后，然后预热准备工作已经搞定，
                //然后进行循环尝试获取锁，在循环次数还未达到<2>以前，某一次在条件<3>判断时发现有其它线程对这个segment进行了修改，
                //那么retries被重置为-1，从而再一次进入到<1>条件内，此时如果再次遍历到链表最后时，因为上一次遍历时已经给node赋值过了，
                //所以这里判断node是否为空，从而避免第二次创建对象给node重复赋值。
                if (node == null) // speculatively create node
                    node = new HashEntry<K,V>(hash, key, value, null);
                retries = 0;
            }
            else if (key.equals(e.key))//<1.2>   遍历过程发现链表中找到了我们需要的key的坑位
                retries = 0;
            else//<1.3>   当前位置对应的key不是我们需要的，遍历下一个
                e = e.next;
        }
        else if (++retries > MAX_SCAN_RETRIES) {//<2>
            // 尝试获取锁次数超过设置的最大值，直接进入阻塞等待，这就是所谓的有限制的自旋获取锁，
            //之所以这样是因为如果持有锁的线程要过很久才释放锁，这期间如果一直无限制的自旋其实是对系统性能有消耗的，
            //这样无限制的自旋是不利的，所以加入最大自旋次数，超过这个次数则进入阻塞状态等待对方释放锁并获取锁。
            lock();
            break;
        }
        else if ((retries & 1) == 0 &&
                 (f = entryForHash(this, hash)) != first) {//<3>
            // 遍历过程中，有可能其它线程改变了遍历的链表，这时就需要重新进行遍历了。
            e = first = f; // re-traverse if entry changed
            retries = -1;
        }
    }
    return node;
}
```



#### put()

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
    //先尝试对segment加锁，如果直接加锁成功，那么node=null；如果加锁失败，则会调用scanAndLockForPut方法去获取锁，
    //在这个方法中，获取锁后会返回对应HashEntry（要么原来就有要么新建一个）
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    V oldValue;
    try {
        //这里是一个优化点，由于table自身是被volatile修饰的，然而put这一块代码本身是加锁了的，所以同一时间内只会有一个线程操作这部分内容，
        //所以不再需要对这一块内的变量做任何volatile修饰，因为变量加了volatile修饰后，变量无法进行编译优化等，会对性能有一定的影响
        //故将table赋值给put方法中的一个局部变量，从而使得能够减少volatile带来的不必要消耗。
        HashEntry<K,V>[] tab = table;
        int index = (tab.length - 1) & hash;
        //这里有一个问题：为什么不直接使用数组下标获取HashEntry，而要用entryAt来获取链表？
        //这里结合网上内容个人理解是：由于Segment继承的是ReentrantLock，所以它是一个可重入锁，那么是否存在某种场景下，
        //会导致同一个线程连续两次进入put方法，而由于put最终使用的putOrderedObject只是禁止了写写重排序无法保证内存可见性，
        //所以这种情况下第二次put在获取链表时必须用entryAt中的volatile语义的get来获取链表，因为这种情况下下标获取的不一定是最新数据。
        //不过并没有想到哪里会存在这种场景，有谁能想到的或者是我的理解有误请指出！
        HashEntry<K,V> first = entryAt(tab, index);//先获取需要put的<k,v>对在当前这个segment中对应的链表的表头结点。

        for (HashEntry<K,V> e = first;;) {//开始遍历first为头结点的链表
            if (e != null) {//<1>
                //e不为空，说明当前键值对需要存储的位置有hash冲突，直接遍历当前链表，如果链表中找到一个节点对应的key相同，
                //依据onlyIfAbsent来判断是否覆盖已有的value值。
                K k;
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    //进入这个条件内说明需要put的<k,y>对应的key节点已经存在，直接判断是否更新并最后break退出循环。
                    oldValue = e.value;
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    break;
                }
                e = e.next;//未进入上面的if条件中，说明当前e节点对应的key不是需要的，直接遍历下一个节点。
            }
            else {//<2>
                //进入到这个else分支，说明e为空，对应有两种情况下e可能会为空，即：
                // 1>. <1>中进行循环遍历，遍历到了链表的表尾仍然没有满足条件的节点。
                // 2>. e=first一开始就是null（可以理解为即一开始就遍历到了尾节点）
                if (node != null) //这里有可能获取到锁是通过scanAndLockForPut方法内自旋获取到的，这种情况下依据找好或者说是新建好了对应节点，node不为空
                    node.setNext(first);
                else// 当然也有可能是这里直接第一次tryLock就获取到了锁，从而node没有分配对应节点，即需要给依据插入的k,v来创建一个新节点
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1; //总数+1 在这里依据获取到了锁，即是线程安全的！对应了上述对count变量的使用规范说明。
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)//判断是否需要进行扩容
                    //扩容是直接重新new一个新的HashEntry数组，这个数组的容量是老数组的两倍，
                    //新数组创建好后再依次将老的table中的HashEntry插入新数组中，所以这个过程是十分费时的，应尽量避免。
                    //扩容完毕后，还会将这个node插入到新的数组中。
                    rehash(node);
                else
                    //数组无需扩容，那么就直接插入node到指定index位置，这个方法里用的是UNSAFE.putOrderedObject
                    //网上查阅到的资料关于使用这个方法的原因都是说因为它使用的是StoreStore屏障，而不是十分耗时的StoreLoad屏障
                    //给我个人感觉就是putObjectVolatile是对写入对象的写入赋予了volatile语义，但是代价是用了StoreLoad屏障
                    //而putOrderedObject则是使用了StoreStore屏障保证了写入顺序的禁止重排序，但是未实现volatile语义导致更新后的不可见性，
                    //当然这里由于是加锁了，所以在释放锁前会将所有变化从线程自身的工作内存更新到主存中。
                    //这一块对于putOrderedObject和putObjectVolatile的区别有点混乱，不是完全理解，网上也没找到详细解答，查看了C源码也是不大确定。
                    //希望有理解的人看到能指点一下，后续如果弄明白了再更新这一块。
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```



#### rehash()

```java
/**
 * Doubles size of table and repacks entries, also adding the
 * given node to new table
 * 对数组进行扩容，由于扩容过程需要将老的链表中的节点适用到新数组中，所以为了优化效率，可以对已有链表进行遍历，
 * 对于老的oldTable中的每个HashEntry，从头结点开始遍历，找到第一个后续所有节点在新table中index保持不变的节点fv，
 * 假设这个节点新的index为newIndex，那么直接newTable[newIndex]=fv，即可以直接将这个节点以及它后续的链表中内容全部直接复用copy到newTable中
 * 这样最好的情况是所有oldTable中对应头结点后跟随的节点在newTable中的新的index均和头结点一致，那么就不需要创建新节点，直接复用即可。
 * 最坏情况当然就是所有节点的新的index全部发生了变化，那么就全部需要重新依据k,v创建新对象插入到newTable中。
*/
@SuppressWarnings("unchecked")
private void rehash(HashEntry<K,V> node) {
    HashEntry<K,V>[] oldTable = table;
    int oldCapacity = oldTable.length;
    int newCapacity = oldCapacity << 1;
    threshold = (int)(newCapacity * loadFactor);
    HashEntry<K,V>[] newTable =
        (HashEntry<K,V>[]) new HashEntry[newCapacity];
    int sizeMask = newCapacity - 1;
    for (int i = 0; i < oldCapacity ; i++) {
        HashEntry<K,V> e = oldTable[i];
        if (e != null) {
            HashEntry<K,V> next = e.next;
            int idx = e.hash & sizeMask;
            if (next == null)   //  Single node on list 只有单个节点
                newTable[idx] = e;
            else { // Reuse consecutive sequence at same slot
                HashEntry<K,V> lastRun = e;
                int lastIdx = idx;
                for (HashEntry<K,V> last = next;
                     last != null;
                     last = last.next) {
                    int k = last.hash & sizeMask;
                    if (k != lastIdx) {
                        lastIdx = k;
                        lastRun = last;
                    }
                }//这个for循环就是找到第一个后续节点新的index不变的节点。
                newTable[lastIdx] = lastRun;
                // Clone remaining nodes
                //第一个后续节点新index不变节点前所有节点均需要重新创建分配。
                for (HashEntry<K,V> p = e; p != lastRun; p = p.next) {
                    int h = p.hash;
                    int k = h & sizeMask;
                    HashEntry<K,V> n = newTable[k];
                    newTable[k] = new HashEntry<K,V>(h, p.key, p.value, n);
                }
            }
        }
    }
    int nodeIndex = node.hash & sizeMask; // add the new node
    node.setNext(newTable[nodeIndex]);
    newTable[nodeIndex] = node;
    table = newTable;
}
```





### ensureSegment()

延迟创建`segments`数组

用于确定指定的Segment是否存在，不存在则会创建

#### map关键属性

```java
/**
 * 用于索引segment的掩码值，key哈希码的高位用于选择segment
 */
final int segmentMask;

/**
 * 用于索引segment偏移值
 */
final int segmentShift;

/**
 * Segment数组
 */
final Segment<K,V>[] segments;
```

### Unsafe



### get()

```java
public V get(Object key) {
    Segment<K,V> s; // manually integrate access methods to reduce overhead
    HashEntry<K,V>[] tab;
    int h = hash(key);//获取key对应hash值
    long u = (((h >>> segmentShift) & segmentMask) << SSHIFT) + SBASE;//获取对应h值存储所在segments数组中内存偏移量
    if ((s = (Segment<K,V>)UNSAFE.getObjectVolatile(segments, u)) != null &&
        (tab = s.table) != null) {
        //通过Unsafe中的getObjectVolatile方法进行volatile语义的读，获取到segments在偏移量为u位置的分段Segment，
        //并且分段Segment中对应table数组不为空
        for (HashEntry<K,V> e = (HashEntry<K,V>) UNSAFE.getObjectVolatile
                 (tab, ((long)(((tab.length - 1) & h)) << TSHIFT) + TBASE);
             e != null; e = e.next) {//获取h对应这个分段中偏移量为xxx下的HashEntry的链表头结点，然后对链表进行 遍历
            //###这里第一次初始化通过getObjectVolatile获取HashEntry时，获取到的是主存中最新的数据，但是在后续遍历过程中，有可能数据被其它线程修改
            //从而导致其实这里最终返回的可能是过时的数据，所以这里就是ConcurrentHashMap所谓的弱一致性的体现，containsKey方法也一样！！！！
            K k;
            if ((k = e.key) == key || (e.hash == h && key.equals(k)))
                return e.value;
        }
    }
    return null;
}
```



### size()

cas 超过次数， lock









































