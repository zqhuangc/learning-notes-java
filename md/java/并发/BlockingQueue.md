生产者-消费者

* ArrayBlockingQueue：  
读写**共用**一个锁对象，每次只能读或写一个操作
put  
take  

* LinkedBlockingQueue  
读写分别有一个锁对象  
takeLock和putLock  

## ArrayBlockingQueue与LinkedBlockingQueue的比较
* 相同点：  
  ArrayBlockingQueue和LinkedBlockingQueue都是通过condition通知机制来实现可阻塞式插入和删除元素，并满足线程安全的特性；
* 不同点：
1. ArrayBlockingQueue底层是采用的数组进行实现，而LinkedBlockingQueue则是采用链表数据结构；
2.  ArrayBlockingQueue插入和删除数据，只采用了一个lock，而LinkedBlockingQueue则是在插入和删除分别采用了putLock和takeLock，这样可以降低线程由于线程无法获取到 lock 而进入 WAITING 状态的可能性，从而提高了线程并发执行的效率。



### SynchronousQueue

