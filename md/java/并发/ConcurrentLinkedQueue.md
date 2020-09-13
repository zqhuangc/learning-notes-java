

```java
private static class Node<E> {
    //用volatile进行修饰的，以保证内存可见性
    //数据域item，另一个是next指针，用于指向下一个节点从而构成链式队列
    volatile E item;
    volatile Node<E> next;
    .......
}
```
```java
成员变量
private transient volatile Node<E> head;
private transient volatile Node<E> tail;
```
说明ConcurrentLinkedQueue通过持有头尾指针进行管理队列。


Node的CAS操作
```java
//更改Node中的数据域item	
boolean casItem(E cmp, E val) {
    return UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
}
//更改Node中的指针域next
void lazySetNext(Node<E> val) {
    UNSAFE.putOrderedObject(this, nextOffset, val);
}
//更改Node中的指针域next
boolean casNext(Node<E> cmp, Node<E> val) {
    return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
}

```

### offer方法
单个线程offer
多个线程offer
部分线程offer，部分线程poll
----offer的速度快于poll  
--------队列长度会越来越长，由于offer节点总是在对队列队尾，而poll节点总是在队列对头，也就是说offer线程和poll线程两者并无“交集”，也就是说两类线程间并不会相互影响，这种情况站在相对速率的角度来看，也就是一个"单线程offer"
----offer的速度慢于poll  
--------poll的相对速率快于offer，也就是队头删的速度要快于队尾添加节点的速度，导致的结果就是队列长度会越来越短，而offer线程和poll线程就会出现“交集”，即那一时刻就可以称之为offer线程和poll线程同时操作的节点为 临界点 ，且在该节点offer线程和poll线程必定相互影响。根据在临界点时offer和poll发生的相对顺序又可从两个角度去思考：
1. 执行顺序为offer-->poll-->offer，即表现为当offer线程在Node1后插入Node2时，此时poll线程已经将Node1删除，这种情况很显然需要在offer方法中考虑； 
2. 执行顺序可能为：poll-->offer-->poll，即表现为当poll线程准备删除的节点为null时（队列为空队列），此时offer线程插入一个节点使得队列变为非空队列


* offer()
```java
public boolean offer(E e) {
    checkNotNull(e);
    final Node<E> newNode = new Node<E>(e);

    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        if (q == null) {
            // p is last node
            if (p.casNext(null, newNode)) {
                // Successful CAS is the linearization point
                // for e to become an element of this queue,
               // and for newNode to become "live".
                if (p != t) // hop two nodes at a time
                    casTail(t, newNode);  // Failure is OK.
                return true;
            }
            // Lost CAS race to another thread; re-read next
        }
        else if (p == q)
            // We have fallen off list.  If tail is unchanged, it
            // will also be off-list, in which case we need to
            // jump to head, from which all live nodes are always
            // reachable.  Else the new tail is a better bet.
            p = (t != (t = tail)) ? t : head;
        else
            // Check for tail updates after two hops.
            p = (p != t && t != (t = tail)) ? t : q;
    }
}
```

* poll()
```java
public E poll() {
    restartFromHead:
    for (;;) {
        for (Node<E> h = head, p = h, q;;) {
            E item = p.item;

            if (item != null && p.casItem(item, null)) {
                // Successful CAS is the linearization point
                // for item to be removed from this queue.
                if (p != h) // hop two nodes at a time
                   updateHead(h, ((q = p.next) != null) ? q : p);
                return item;
            }
           else if ((q = p.next) == null) {
                updateHead(h, p);
                return null;
            }
            else if (p == q)
                continue restartFromHead;
            else
                p = q;
        }
    }
}
```
```
作图理解：p、q、t、head、tail在操作中的变化
```
###　哨兵节点
## HOPS(hop two nodes at a time)
**tail更新触发时机：** 当tail指向的节点的下一个节点不为null的时候，会执行定位队列真正的队尾节点的操作，找到队尾节点后完成插入之后才会通过casTail进行tail更新；当tail指向的节点的下一个节点为null的时候，**只插入节点不更新tail**。
**head更新触发时机：** 当head指向的节点的item域为null的时候，会执行定位队列真正的队头节点的操作，找到队头节点后完成删除之后才会通过updateHead进行head更新；当head指向的节点的item域不为null的时候，**只删除节点不更新head**。

**懒更新**