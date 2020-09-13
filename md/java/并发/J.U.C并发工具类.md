## CountDownLatch（AQS的共享锁）

### 构造

CountDownLatch内部依赖Sync实现，而Sync继承AQS。CountDownLatch仅提供了一个构造方法：

CountDownLatch(int count) ： 构造一个用给定计数初始化的 CountDownLatch

```
    public CountDownLatch(int count) {
        if (count < 0) throw new IllegalArgumentException("count < 0");
        this.sync = new Sync(count);
    }
```

sync为CountDownLatch的一个内部类，其定义如下：

```
    private static final class Sync extends AbstractQueuedSynchronizer {
        private static final long serialVersionUID = 4982264981922014374L;

        Sync(int count) {
            setState(count);
        }

        //获取同步状态
        int getCount() {
            return getState();
        }

        //获取同步状态
        protected int tryAcquireShared(int acquires) {
            return (getState() == 0) ? 1 : -1;
        }

        //释放同步状态
        protected boolean tryReleaseShared(int releases) {
            for (;;) {
                int c = getState();
                if (c == 0)
                    return false;
                int nextc = c-1;
                if (compareAndSetState(c, nextc))
                    return nextc == 0;
            }
        }
    }
```



阻塞线程  
接着来看**await方法**，直接调用了 AQS 的 acquireSharedInterruptibly。

- await

```
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}

public final void acquireSharedInterruptibly(int arg) throws InterruptedException { 
    if (Thread.interrupted()) 
         throw new InterruptedException();
    if (tryAcquireShared(arg) < 0) 
         doAcquireSharedInterruptibly(arg);
    }
protected int tryAcquireShared(int acquires) {
   return (getState() == 0) ? 1 : -1;
}
```

doAcquireSharedInterruptibly的逻辑和独占功能的acquireQueued基本相同，阻塞线程的过程是一样的。**不同之处**：  

- 创建的Node是定义成共享的（Node.SHARED）；
  被唤醒后重新尝试获取锁，不只设置自己为head，还需要通知其他等待的线程。（重点看后文释放操作里的setHeadAndPropagate）

### 释放

​       用给定的计数初始化 CountDownLatch。由于调用了 countDown() 方法，所以在当前计数到达零之前，await 方法会一直受阻塞。之后，会释放所有等待的线程，await 的所有后续调用都将立即返回。这种现象只出现一次——计数无法被重置

构造器中的计数值（count）实际上就是闭锁需要等待的线程数量。这个值只能被设置一次，而且 CountDownLatch 没有提供任何机制去重新设置这个计数值。  
与 CountDownLatch 的第一次交互是主线程等待其他线程。主线程必须在启动其他线程后立即调用 CountDownLatch.await() 方法。这样主线程的操作就会在这个方法上阻塞，直到其他线程完成各自的任务。

## CyclicBarrier

CyclicBarrier 字面意思是可循环（Cyclic）使用的屏障（Barrier）。它要做的事情是让一组线程到达一个屏障（同步点）时被阻塞，直到最后一个线程到达屏障时候，屏障才会开门。所有被屏障拦截的线程才会运行。

### 底层原理实现

CyclicBarrier是由ReentrantLock可重入锁和Condition共同实现的。

barrierAction 为CyclicBarrier接收的Runnable命令，用于在线程到达屏障时，优先执行barrierAction ，用于处理更加复杂的业务场景。

await()方法内部调用dowait(boolean timed, long nanos)方法：

```
    private int dowait(boolean timed, long nanos)
            throws InterruptedException, BrokenBarrierException,
            TimeoutException {
        //获取锁
        final ReentrantLock lock = this.lock;
        lock.lock();
        try {
            //分代
            final Generation g = generation;

            //当前generation“已损坏”，抛出BrokenBarrierException异常
            //抛出该异常一般都是某个线程在等待某个处于“断开”状态的CyclicBarrie
            if (g.broken)
                //当某个线程试图等待处于断开状态的 barrier 时，或者 barrier 进入断开状态而线程处于等待状态时，抛出该异常
                throw new BrokenBarrierException();

            //如果线程中断，终止CyclicBarrier
            if (Thread.interrupted()) {
                breakBarrier();
                throw new InterruptedException();
            }

            //进来一个线程 count - 1
            int index = --count;
            //count == 0 表示所有线程均已到位，触发Runnable任务
            if (index == 0) {  // tripped
                boolean ranAction = false;
                try {
                    final Runnable command = barrierCommand;
                    //触发任务
                    if (command != null)
                        command.run();
                    ranAction = true;
                    //唤醒所有等待线程，并更新generation
                    nextGeneration();
                    return 0;
                } finally {
                    if (!ranAction)
                        breakBarrier();
                }
            }


            for (;;) {
                try {
                    //如果不是超时等待，则调用Condition.await()方法等待
                    if (!timed)
                        trip.await();
                    else if (nanos > 0L)
                        //超时等待，调用Condition.awaitNanos()方法等待
                        nanos = trip.awaitNanos(nanos);
                } catch (InterruptedException ie) {
                    if (g == generation && ! g.broken) {
                        breakBarrier();
                        throw ie;
                    } else {
                        // We're about to finish waiting even if we had not
                        // been interrupted, so this interrupt is deemed to
                        // "belong" to subsequent execution.
                        Thread.currentThread().interrupt();
                    }
                }

                if (g.broken)
                    throw new BrokenBarrierException();

                //generation已经更新，返回index
                if (g != generation)
                    return index;

                //“超时等待”，并且时间已到,终止CyclicBarrier，并抛出异常
                if (timed && nanos <= 0L) {
                    breakBarrier();
                    throw new TimeoutException();
                }
            }
        } finally {
            //释放锁
            lock.unlock();
        }
    }
```

其实await()的处理逻辑还是比较简单的：如果该线程不是到达的最后一个线程，则他会一直处于等待状态，除非发生以下情况：

1. 最后一个线程到达，即index == 0
2. 超出了指定时间（超时等待）
3. 其他的某个线程中断当前线程
4. 其他的某个线程中断另一个等待的线程
5. 其他的某个线程在等待barrier超时
6. 其他的某个线程在此barrier调用reset()方法。reset()方法用于将屏障重置为初始状态。

### 构造

```
//1.CyclicBarrier构造方法
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    // parties表示“必须同时到达barrier的线程个数”。
    this.parties = parties;
    // count表示“处在等待状态的线程个数”。
    this.count = parties;
    // barrierCommand表示“parties个线程到达barrier时，会执行的动作”。
    this.barrierCommand = barrierAction;
}

private static class Generation {
        boolean broken = false;
    }
```



当所有线程都已经到达barrier处（index == 0），则会通过nextGeneration()进行更新换地操作，在这个步骤中，做了三件事：唤醒所有线程，重置count，generation。

```
    private void nextGeneration() {
        trip.signalAll();
        count = parties;
        generation = new Generation();
    }
```

### 区别

CountDownLatch : 一个线程(或者多个)， 等待另外N个线程完成某个事情之后才能执行。

CyclicBarrier : N个线程相互等待，任何一个线程完成之前，所有的线程都必须等待。

CyclicBarrier是当 await 的数量达到了设置的数量后，才继续往下执行





## Semaphore

Semaphore分为单值和多值两种，前者只能被一个线程获得，后者可以被若干个线程获得。

Semaphore（信号量）是用来控制同时访问特定资源的线程数量，它通过协调各个线程，保证合理的使用公共资源。  
线程可以通过acquire()方法来获取信号量的许可，当信号量中没有可用的许可的时候，线程阻塞，直到有可用的许可为止。线程可以通过release()方法释放它持有的信号量的许可。    
控制并发线程数的Semaphore   
Semaphore（信号量）是用来控制同时访问特定资源的线程数量，它通过协调各个线程，保证合理的使用公共资源。  

信号量Semaphore是一个控制访问多个共享资源的计数器，和CountDownLatch一样，其本质上是一个“共享锁”。

线程可以通过acquire()方法来获取信号量的许可，当信号量中没有可用的许可的时候，线程阻塞，直到有可用的许可为止。线程可以通过release()方法释放它持有

```java
Semaphore提供了两个构造函数：

Semaphore(int permits) ：创建具有给定的许可数和非公平的公平设置的 Semaphore。
Semaphore(int permits, boolean fair) ：创建具有给定的许可数和给定的公平设置的 Semaphore。
实现如下：

    public Semaphore(int permits) {
        sync = new NonfairSync(permits);
    }

    public Semaphore(int permits, boolean fair) {
        sync = fair ? new FairSync(permits) : new NonfairSync(permits);
    }
Semaphore默认选择非公平锁。


公平

    protected int tryAcquireShared(int acquires) {
        for (;;) {
            //判断该线程是否位于CLH队列的列头
            if (hasQueuedPredecessors())
                return -1;
            //获取当前的信号量许可
            int available = getState();

            //设置“获得acquires个信号量许可之后，剩余的信号量许可数”
            int remaining = available - acquires;

            //CAS设置信号量
            if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                return remaining;
        }
    }
 

非公平

对于非公平而言，因为它不需要判断当前线程是否位于CLH同步队列列头，所以相对而言会简单些。

        protected int tryAcquireShared(int acquires) {
            return nonfairTryAcquireShared(acquires);
        }
		
		final int nonfairTryAcquireShared(int acquires) {
            for (;;) {
                int available = getState();
                int remaining = available - acquires;
                if (remaining < 0 ||
                    compareAndSetState(available, remaining))
                    return remaining;
            }
        }
信号量释放

获取了许可，当用完之后就需要释放，Semaphore提供release()来释放许可。

    public void release() {
        sync.releaseShared(1);
    }
内部调用AQS的releaseShared(int arg)：

    public final boolean releaseShared(int arg) {
        if (tryReleaseShared(arg)) {
            doReleaseShared();
            return true;
        }
        return false;
    }
releaseShared(int arg)调用Semaphore内部类Sync的tryReleaseShared(int arg)：

    protected final boolean tryReleaseShared(int releases) {
        for (;;) {
            int current = getState();
            //信号量的许可数 = 当前信号许可数 + 待释放的信号许可数
            int next = current + releases;
            if (next < current) // overflow
                throw new Error("Maximum permit count exceeded");
            //设置可获取的信号许可数为next
            if (compareAndSetState(current, next))
                return true;
        }
    }
```





## Exchanger



Exchanger是一个用于线程间协作的工具类，用于两个线程间能够交换。它提供了一个交换的同步点，在这个同步点两个线程能够交换数据。具体交换数据是通过exchange方法来实现的，如果一个线程先执行exchange方法，那么它会同步等待另一个线程也执行exchange方法，这个时候两个线程就都达到了同步点，两个线程就可以交换数据。



Exchange是最简单的也是最复杂的，简单在于API非常简单，就一个构造方法和两个exchange()方法，最复杂在于它的实现是最复杂的（反正我是看晕了的）。

在API是这么介绍的：可以在对中对元素进行配对和交换的线程的同步点。每个线程将条目上的某个方法呈现给 exchange 方法，与伙伴线程进行匹配，并且在返回时接收其伙伴的对象。Exchanger 可能被视为 SynchronousQueue 的双向形式。Exchanger 可能在应用程序（比如遗传算法和管道设计）中很有用。

Exchanger，它允许在并发任务之间交换数据。具体来说，Exchanger类允许在两个线程之间定义同步点。当两个线程都到达同步点时，他们交换数据结构，因此第一个线程的数据结构进入到第二个线程中，第二个线程的数据结构进入到第一个线程中。



```java
public class ExchangerTest {

    static class Producer implements Runnable{

        //生产者、消费者交换的数据结构
        private List<String> buffer;

        //步生产者和消费者的交换对象
        private Exchanger<List<String>> exchanger;

        Producer(List<String> buffer,Exchanger<List<String>> exchanger){
            this.buffer = buffer;
            this.exchanger = exchanger;
        }

        @Override
        public void run() {
            for(int i = 1 ; i < 5 ; i++){
                System.out.println("生产者第" + i + "次提供");
                for(int j = 1 ; j <= 3 ; j++){
                    System.out.println("生产者装入" + i  + "--" + j);
                    buffer.add("buffer：" + i + "--" + j);
                }

                System.out.println("生产者装满，等待与消费者交换...");
                try {
                    exchanger.exchange(buffer);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    static class Consumer implements Runnable {
        private List<String> buffer;

        private final Exchanger<List<String>> exchanger;

        public Consumer(List<String> buffer, Exchanger<List<String>> exchanger) {
            this.buffer = buffer;
            this.exchanger = exchanger;
        }

        @Override
        public void run() {
            for (int i = 1; i < 5; i++) {
                //调用exchange()与消费者进行数据交换
                try {
                    buffer = exchanger.exchange(buffer);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                System.out.println("消费者第" + i + "次提取");
                for (int j = 1; j <= 3 ; j++) {
                    System.out.println("消费者 : " + buffer.get(0));
                    buffer.remove(0);
                }
            }
        }
    }

    public static void main(String[] args){
        List<String> buffer1 = new ArrayList<String>();
        List<String> buffer2 = new ArrayList<String>();

        Exchanger<List<String>> exchanger = new Exchanger<List<String>>();

        Thread producerThread = new Thread(new Producer(buffer1,exchanger));
        Thread consumerThread = new Thread(new Consumer(buffer2,exchanger));

        producerThread.start();
        consumerThread.start();
    }
}
```































