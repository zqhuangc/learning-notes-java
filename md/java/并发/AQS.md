# AbstractQueuedSynchronizer(AQS)
AQS实现了对同步状态的管理，以及对阻塞线程进行排队，等待通知等等一些底层的实现处理。AQS的核心也包括了这些方面:*同步队列，独占式锁的获取和释放，共享锁的获取和释放以及可中断锁，超时等待锁获取*这些特性的实现

https://juejin.im/post/5aeb07ab6fb9a07ac36350c8

• AbstractQueuedSynchronizer （AQS  since 1.5）原理

• 条件变量（ConditionObject）原理

AbstractQueuedLongSynchronizer （AQLS  since 1.6）



* 独占式锁：
```java
void acquire(int arg)：独占式获取同步状态，如果获取失败则插入同步队列进行等待；
void acquireInterruptibly(int arg)：与acquire方法相同，但在同步队列中进行等待的时候可以检测中断；
boolean tryAcquireNanos(int arg, long nanosTimeout)：在acquireInterruptibly基础上增加了超时等待功能，在超时时间内没有获得同步状态返回false;
boolean release(int arg)：释放同步状态，该方法会唤醒在同步队列中的下一个节点
```

* 共享式锁：
```java
void acquireShared(int arg)：共享式获取同步状态，与独占式的区别在于同一时刻有多个线程获取同步状态；
void acquireSharedInterruptibly(int arg)：在acquireShared方法基础上增加了能响应中断的功能；
boolean tryAcquireSharedNanos(int arg, long nanosTimeout)：在acquireSharedInterruptibly基础上增加了超时等待的功能；
boolean releaseShared(int arg)：共享式释放同步状态
```

* 同步队列(双向)
当共享资源被某个线程占有，其他请求该资源的线程将会阻塞，从而进入同步队列。AQS中的同步队列则是通过链式方式进行实现。

AQS有一个静态内部类Node，其部分属性如下
```java
volatile int waitStatus; //节点状态
volatile Node prev; //当前节点/线程的前驱节点
volatile Node next; //当前节点/线程的后继节点
volatile Thread thread;//加入同步队列的线程引用
Node nextWaiter;//等待队列中的下一个节点
```

节点的状态
```java
int CANCELLED =  1;//节点从同步队列中取消
int SIGNAL    = -1;//后继节点的线程处于等待状态，如果当前节点释放同步状态会通知后继节点，使得后继节点的线程能够运行；
int CONDITION = -2;//当前节点进入等待队列中
int PROPAGATE = -3;//表示下一次共享式同步状态获取将会无条件传播下去
int INITIAL = 0;//初始状态
```

每个节点有两个域：prev(前驱)和next(后继)，并且每个节点用来保存获取同步状态失败的线程引用以及等待状态等信息。另外AQS中有两个重要的成员变量：
```java
private transient volatile Node head;
private transient volatile Node tail;
```
AQS实际上通过头尾指针来管理同步队列，同时实现包括获取锁失败的线程进行入队，释放锁时对同步队列中的线程进行通知

* 小结
```
* 节点的数据结构，即AQS的静态内部类Node,节点的等待状态等信息；
* 同步队列是一个双向队列，AQS通过持有头尾指针管理同步队列；
```


## 独占式锁
### 独占式锁获取
lock()方法基于acquire
* acquire()
```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}

//tryAcquire：去尝试获取锁(同步状态)，获取成功则设置锁状态并返回true，否则返回false。该方法自定义同步组件自己实现，该方法必须要保证线程安全的获取同步状态。
//addWaiter：如果tryAcquire返回FALSE（获取同步状态失败），则调用该方法将当前线程加入到CLH同步队列尾部。
//acquireQueued：当前线程会根据公平性原则来进行阻塞等待（自旋）,直到获取锁为止；并且返回当前线程在等待过程中有没有中断过。
//selfInterrupt：产生一个中断。
```
获取同步状态失败，入队操作
```java
private Node addWaiter(Node mode) {
		// 1. 将当前线程构建成Node类型
        Node node = new Node(Thread.currentThread(), mode);
        // Try the fast path of enq; backup to full enq on failure
        // 2. 当前尾节点是否为null？
		Node pred = tail;
        if (pred != null) {
			// 2.2 将当前节点尾插入的方式插入同步队列中
            node.prev = pred;
            if (compareAndSetTail(pred, node)) {
                pred.next = node;
                return node;
            }
        }
		// 2.1. 当前同步队列尾节点为null，说明当前线程是第一个加入同步队列进行等待的线程
        enq(node);
        return node;
}
```
* 代码小结
1. 当前同步队列的尾节点为null，调用方法enq()插入;
2. 当前队列的尾节点不为null，则采用尾插入（compareAndSetTail（）方法）的方式入队。

* enq()
```java
private Node enq(final Node node) {
        for (;;) {
            Node t = tail;
			if (t == null) { // Must initialize
				//1. 构造头结点
                if (compareAndSetHead(new Node()))
                    tail = head;
            } else {
				// 2. 尾插入，CAS操作失败自旋尝试
                node.prev = t;
                if (compareAndSetTail(t, node)) {
                    t.next = node;
                    return t;
                }
            }
        }
}
```
带头结点与不带头结点相比，会在入队和出队的操作中获得更大的便捷性，因此同步队列选择了**带头结点的链式存储结构**。

那么带头节点的队列初始化时机是什么？自然而然是在tail为null时，即当前线程是第一次插入同步队列。compareAndSetTail(t, node)方法会利用CAS操作设置尾节点，如果CAS操作失败会在for (;;)for死循环中不断尝试，直至成功return返回为止。因此，对enq()方法可以做这样的总结：

在当前线程是第一个加入同步队列时，调用compareAndSetHead(new Node())方法，完成链式队列的头结点的初始化；
自旋不断尝试CAS尾插入节点直至成功为止。

* 代码小结
1. 在当前线程是第一个加入同步队列时，调用compareAndSetHead(new Node())方法，完成链式队列的头结点的初始化；
2. 自旋不断尝试CAS尾插入节点直至成功为止。

* 如何获取锁？

* acquireQueued()

  acquireQueued方法为一个自旋的过程，也就是说当前线程（Node）进入同步队列后，就会进入一个自旋的过程，每个节点都会自省地观察，当条件满足，获取到同步状态后，就可以从这个自旋过程中退出，否则会一直执行下去。
```java
final boolean acquireQueued(final Node node, int arg) {
        boolean failed = true;
        try {
            boolean interrupted = false;
            for (;;) {
				// 1. 获得当前节点的先驱节点
                final Node p = node.predecessor();
				// 2. 当前节点能否获取独占式锁					
				// 2.1 如果当前节点的先驱节点是头结点并且成功获取同步状态，即可以获得独占式锁
                if (p == head && tryAcquire(arg)) {
					//队列头指针用指向当前节点
                    setHead(node);
					//释放前驱节点
                    p.next = null; // help GC
                    failed = false;
                    return interrupted;
                }
				// 2.2 获取锁失败，线程进入等待状态等待获取独占式锁
                if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                    interrupted = true;
            }
        } finally {
            if (failed)
                cancelAcquire(node);
        }
}
```
代码首先获取当前节点的先驱节点，如果**先驱节点是头结点的并且成功获得同步状态的时候（if (p == head && tryAcquire(arg))），当前节点所指向的线程能够获取锁**。反之，获取锁失败进入等待状态。

* 获取锁成功，出队操作
```java
//队列头结点引用指向当前节点
setHead(node);
//释放前驱节点
p.next = null; // help GC
failed = false;
return interrupted;

//将当前节点通过setHead()方法设置为队列的头结点
private void setHead(Node node) {
        head = node;
        node.thread = null;
        node.prev = null;
}
```

* shouldParkAfterFailedAcquire()和parkAndCheckInterrupt()  
  

shouldParkAfterFailedAcquire()方法主要逻辑是使用compareAndSetWaitStatus(pred, ws, Node.SIGNAL)使用CAS将节点状态由INITIAL设置成SIGNAL，表示当前线程阻塞。当compareAndSetWaitStatus设置失败则说明shouldParkAfterFailedAcquire方法返回false，然后会在acquireQueued()方法中for (;;)死循环中会继续重试，直至compareAndSetWaitStatus设置节点状态位为SIGNAL时shouldParkAfterFailedAcquire返回true时才会执行方法
* parkAndCheckInterrupt()方法
```java
private final boolean parkAndCheckInterrupt() {
        //使得该线程阻塞
		LockSupport.park(this);
        return Thread.interrupted();
}
```
1. 如果当前节点的前驱节点是头节点，并且能够获得同步状态的话，当前线程能够获得锁该方法执行结束退出；
2. 获取锁失败的话，先将节点状态设置成SIGNAL，然后调用LookSupport.park方法使得当前线程阻塞。

### 独占式锁释放（release()方法）
```java
public final boolean release(int arg){
    //同步状态释放成功（tryRelease返回true）则会执行if块中的代码，当head指向的头结点不为null，并且该节点的状态值不为0的话才会执行unparkSuccessor()方法
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

```

```java
private void unparkSuccessor(Node node) {
    /*
     * If status is negative (i.e., possibly needing signal) try
     * to clear in anticipation of signalling.  It is OK if this
     * fails or if status is changed by waiting thread.
     */
    int ws = node.waitStatus;
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);

    /*
     * Thread to unpark is held in successor, which is normally
     * just the next node.  But if cancelled or apparently null,
     * traverse backwards from tail to find the actual
     * non-cancelled successor.
     */

	//头节点的后继节点
    Node s = node.next;
    if (s == null || s.waitStatus > 0) {
        s = null;
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    if (s != null)
		//后继节点不为null时唤醒该线程
        LockSupport.unpark(s.thread);
}
```
首先获取头节点的后继节点，当后继节点的时候会调用LookSupport.unpark()方法，该方法会唤醒该节点的后继节点所包装的线程。因此，每一次锁释放后就会唤醒队列中该节点的后继节点所引用的线程，从而进一步可以佐证获得锁的过程是一个FIFO（先进先出）的过程。

1. 线程获取锁失败，线程被封装成Node进行入队操作，核心方法在于addWaiter()和enq()，同时enq()完成对同步队列的头结点初始化工作以及CAS操作失败的重试;
2. 线程获取锁是一个自旋的过程，当且仅当 当前节点的前驱节点是头结点并且成功获得同步状态时，节点出队即该节点引用的线程获得锁，否则，当不满足条件时就会调用LookSupport.park()方法使得线程阻塞；
3. 释放锁的时候会唤醒后继节点；
* 小结  
在获取同步状态时，AQS维护一个同步队列，获取同步状态失败的线程会加入到队列中进行自旋；移除队列（或停止自旋）的条件是前驱节点是头结点并且成功获得了同步状态。在释放同步状态时，同步器会调用unparkSuccessor()方法唤醒后继节点。

### 可中断式获取（acquireInterruptibly方法）
```java
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (!tryAcquire(arg))
		//线程获取锁失败
        doAcquireInterruptibly(arg);
}

private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
	//将节点插入到同步队列中
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            //获取锁出队
			if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
				//线程中断抛异常
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

```
### 超时等待式获取锁（tryAcquireNanos()方法）
通过调用lock.tryLock(timeout,TimeUnit)方式达到超时等待获取锁的效果，该方法会在三种情况下才会返回：
1. 在超时时间内，当前线程成功获取了锁；
2. 当前线程在超时时间内被中断；
3. 超时时间结束，仍未获得锁返回false。
```java
public final boolean tryAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    return tryAcquire(arg) ||
		//实现超时等待的效果
        doAcquireNanos(arg, nanosTimeout);
}

private boolean doAcquireNanos(int arg, long nanosTimeout)
        throws InterruptedException {
    if (nanosTimeout <= 0L)
        return false;
	//1. 根据超时时间和当前时间计算出截止时间
    final long deadline = System.nanoTime() + nanosTimeout;
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
			//2. 当前线程获得锁出队列
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return true;
            }
			// 3.1 重新计算超时时间
            nanosTimeout = deadline - System.nanoTime();
            // 3.2 已经超时返回false
			if (nanosTimeout <= 0L)
                return false;
			// 3.3 线程阻塞等待 
            if (shouldParkAfterFailedAcquire(p, node) &&
                nanosTimeout > spinForTimeoutThreshold)
                LockSupport.parkNanos(this, nanosTimeout);
            // 3.4 线程被中断抛出被中断异常
			if (Thread.interrupted())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}

```

## 共享锁
### 共享锁的获取（acquireShared()方法）
```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}

private void doAcquireShared(int arg) {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                int r = tryAcquireShared(arg);
                if (r >= 0) {
					// 当该节点的前驱节点是头结点且成功获取同步状态
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

逻辑几乎和独占式锁的获取一模一样，这里的自旋过程中能够退出的条件是当前节点的前驱节点是头结点并且tryAcquireShared(arg)返回值大于等于0即能成功获得同步状态。
### 共享锁的释放（releaseShared()方法）
```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}

private void doReleaseShared() {
    /*
     * Ensure that a release propagates, even if there are other
     * in-progress acquires/releases.  This proceeds in the usual
     * way of trying to unparkSuccessor of head if it needs
     * signal. But if it does not, status is set to PROPAGATE to
     * ensure that upon release, propagation continues.
     * Additionally, we must loop in case a new node is added
     * while we are doing this. Also, unlike other uses of
     * unparkSuccessor, we need to know if CAS to reset status
     * fails, if so rechecking.
     */
    for (;;) {
        Node h = head;
        if (h != null && h != tail) {
            int ws = h.waitStatus;
            if (ws == Node.SIGNAL) {
                if (!compareAndSetWaitStatus(h, Node.SIGNAL, 0))
                    continue;            // loop to recheck cases
                unparkSuccessor(h);
            }
            else if (ws == 0 &&
                     !compareAndSetWaitStatus(h, 0, Node.PROPAGATE))
                continue;                // loop on failed CAS
        }
        if (h == head)                   // loop if head changed
            break;
    }
}

```
### 可中断（acquireSharedInterruptibly()方法），超时等待（tryAcquireSharedNanos()方法）





#  CLH同步队列

AQS内部维护着一个FIFO队列，该队列就是CLH同步队列。

CLH同步队列是一个FIFO双向队列，AQS依赖它来完成同步状态的管理，当前线程如果获取同步状态失败时，AQS则会将当前线程已经等待状态等信息构造成一个节点（Node）并将其加入到CLH同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点唤醒（公平锁），使其再次尝试获取同步状态。

在CLH同步队列中，一个节点表示一个线程，它保存着线程的引用（thread）、状态（waitStatus）、前驱节点（prev）、后继节点（next），其定义如下：

```
static final class Node {    
    /** 共享 */    
    static final Node SHARED = new Node();    
    /** 独占 */    
    static final Node EXCLUSIVE = null;    
    /**     
    * 因为超时或者中断，节点会被设置为取消状态，被取消的节点时不会参与到竞争中的，他会一直保持取消状态不会转变为其他状态；     
    */    
    static final int CANCELLED =  1;    
    /**     
     * 后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或者被取消，将会通知后继节点，使后继节点的线程得以运行     
     */
     static final int SIGNAL    = -1;    
    /**     
    * 节点在等待队列中，节点线程等待在Condition上，当其他线程对Condition调用了signal()后，改节点将会从等待队列中转移到同步队列中，加入到同步状态的获取中     
    */    
    static final int CONDITION = -2;    
    /**     
    * 表示下一次共享式同步状态获取将会无条件地传播下去     
    */    
    static final int PROPAGATE = -3;    
    /** 等待状态 */    
    volatile int waitStatus;    
    /** 前驱节点 */    
    volatile Node prev;    
    /** 后继节点 */    
    volatile Node next;    
    /** 获取同步状态的线程 */    
    volatile Thread thread;    
    Node nextWaiter;
    
    final boolean isShared() {
        return nextWaiter == SHARED;    
    }    
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }
    
    Node() {    
    }    
    Node(Thread thread, Node mode) {
        this.nextWaiter = mode;
        this.thread = thread;
    }
    Node(Thread thread, int waitStatus) {
        this.waitStatus = waitStatus; 
        this.thread = thread;    
    }
}
```

CLH同步队列结构图如下：

![img](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZffPDiceiccIRHFYvon2HQYLiaorBQWJ0dTk8mloPCdJomaibyUFdcOEQIy6SobIIMxcyHoebb5K633QIw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 入列

CLH队列入列，tail指向新节点、新节点的prev指向当前最后的节点，当前最后一个节点的next指向当前节点。代码我们可以看看addWaiter(Node node)方法：

```
private Node addWaiter(Node mode) {        
    //新建Node        
    Node node = new Node(Thread.currentThread(), mode);
    //快速尝试添加尾节点
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        //CAS设置尾节点
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //多次尝试
    enq(node);
    return node;
 }
```

addWaiter(Node node)先通过快速尝试设置尾节点，如果失败，则调用enq(Node node)方法设置尾节点

```
private Node enq(final Node node) {
    //多次尝试，直到成功为止
    for (;;) {
        Node t = tail;
        //tail不存在，设置为首节点
        if (t == null) {
            if (compareAndSetHead(new Node()))
            	tail = head;
        } else {
            //设置为尾节点 
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

在上面代码中，两个方法都是通过一个CAS方法compareAndSetTail(Node expect, Node update)来设置尾节点，该方法可以确保节点是线程安全添加的。在enq(Node node)方法中，AQS通过“死循环”的方式来保证节点可以正确添加，只有成功添加后，当前线程才会从该方法返回，否则会一直执行下去。

过程图如下：![img](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZffPDiceiccIRHFYvon2HQYLiaokItsKl5gDEqkZSZDdibr2uibrpRgmOeBWP70pHaKMSQxnh0Ll4moD22Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 出列

CLH同步队列遵循FIFO，首节点的线程释放同步状态后，将会唤醒它的后继节点（next），而后继节点将会在获取同步状态成功时将自己设置为首节点，这个过程非常简单，head执行该节点并断开原首节点的next和当前节点的prev即可，注意在这个过程是不需要使用CAS来保证的，因为只有一个线程能够成功获取到同步状态。过程图如下：

![img](https://mmbiz.qpic.cn/mmbiz_png/JdLkEI9sZffPDiceiccIRHFYvon2HQYLiaoF9hckqpZtCz60zrwliamB9oEDGtqz7K9YuCUqA7ycZuEsR4Hb7HoicQg/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)