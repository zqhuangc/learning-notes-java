https://my.oschina.net/readjava/blog/282882
## 概念
此类以及每个使用它的线程与一个许可关联（从 Semaphore 类的意义上说）。如果该许可可用，并且可在进程中使用，则调用 park 将立即返回；否则可能 阻塞。如果许可尚不可用，则可以调用 unpark 使其可用。（但与 Semaphore 不同的是，许可不能累积，并且最多只能有一个许可。） 

**不可实例化的工具类**

## 源码
```java
package java.util.concurrent.locks;
import java.util.concurrent.*;
import sun.misc.Unsafe;

public class LockSupport {
    private LockSupport() {} // Cannot be instantiated.

    // Hotspot implementation via intrinsics API
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long parkBlockerOffset;

    static {
        try {
            parkBlockerOffset = unsafe.objectFieldOffset
                (java.lang.Thread.class.getDeclaredField("parkBlocker"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private static void setBlocker(Thread t, Object arg) {
        // Even though volatile, hotspot doesn't need a write barrier here.
        unsafe.putObject(t, parkBlockerOffset, arg);
    }

    public static void unpark(Thread thread) {
        if (thread != null)
            unsafe.unpark(thread);
    }

    public static void park(Object blocker) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        unsafe.park(false, 0L);
        setBlocker(t, null);
    }

    public static void parkNanos(Object blocker, long nanos) {
        if (nanos > 0) {
            Thread t = Thread.currentThread();
            setBlocker(t, blocker);
            unsafe.park(false, nanos);
            setBlocker(t, null);
        }
    }

    public static void parkUntil(Object blocker, long deadline) {
        Thread t = Thread.currentThread();
        setBlocker(t, blocker);
        unsafe.park(true, deadline);
        setBlocker(t, null);
    }

    public static Object getBlocker(Thread t) {
        return unsafe.getObjectVolatile(t, parkBlockerOffset);
    }

    public static void park() {
        unsafe.park(false, 0L);
    }

    public static void parkNanos(long nanos) {
        if (nanos > 0)
            unsafe.park(false, nanos);
    }

    public static void parkUntil(long deadline) {
        unsafe.park(true, deadline);
    }
}
```
**unsafe**：是JDK内部用的工具类。它通过暴露一些Java意义上说“不安全”的功能给Java层代码，来让JDK能够更多的使用Java代码来实现一些原本是平台相关的、需要使用native语言（例如C或C++）才可以实现的功能。该类不应该在JDK核心类库之外使用。 

parkBlokcerOffset：parkBlocker的偏移量,
**parkBlocker**是用于记录线程是被谁阻塞的。可以通过LockSupport的getBlocker获取到阻塞的对象。用于监控和分析线程用的