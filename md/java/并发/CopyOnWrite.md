读多写少：比如白名单，黑名单等
读线程的角度来看，即读线程任何时候都是获取到最新的数据，满足数据实时性。牺牲数据实时性满足数据的最终一致性即可  
CopyOnWriteArrayList就是通过Copy-On-Write(COW)，即写时复制的思想来通过延时更新的策略来实现数据的**最终一致性**，并且能够保证读线程间不阻塞。


## CopyOnWriteArrayList
* get方法  
所有的读线程只是会读取数据容器中的数据，并不会进行修改。不加锁
* add方法
```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
	//1. 使用Lock,保证写线程在同一时刻只有一个
    lock.lock();
    try {
		//2. 获取旧数组引用
        Object[] elements = getArray();
        int len = elements.length;
		//3. 创建新的数组，并将旧数组的数据复制到新数组中
        Object[] newElements = Arrays.copyOf(elements, len + 1);
		//4. 往新数组中添加新的数据	        
		newElements[len] = e;
		//5. 将旧数组引用指向新的数组
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```
* add小结
1. 采用ReentrantLock，保证同一时刻只有一个写线程正在进行数组的复制，否则的话内存中会有多份被复制的数据；
2. 前面说过数组引用是volatile修饰的，因此将旧的数组引用指向新的数组，根据volatile的happens-before规则，写线程对数组引用的修改对读线程是可见的。
3. 由于在写数据的时候，是在新的数组中插入数据的，从而保证读写实在两个不同的数据容器中进行操作。


## COW vs 读写锁
* 相同点：
1. 两者都是通过读写分离的思想实现；
2. 读线程间是互不阻塞的

* 不同点：  
对读线程而言，为了实现数据实时性，在写锁被获取后，读线程会等待或者当读锁被获取后，写线程会等待，从而解决“脏读”等问题。也就是说如果使用读写锁依然会出现读线程阻塞等待的情况。而COW则完全放开了牺牲数据实时性而保证数据最终一致性，即读线程对数据的更新是延时感知的，因此读线程不会存在等待的情况。


为什么需要复制呢？ 如果将array 数组设定为volitile的， 对volatile变量写happens-before读，读线程不是能够感知到volatile变量的变化。  
原因是，这里volatile的修饰的仅仅只是数组引用，数组中的元素的修改是不能保证可见性的。因此COW采用的是新旧两个数据容器，
为什么concurrentHashMap只具有弱一致性的原因？
