### 简介

`ReentrantLock` 是 Java 并发包 `java.util.concurrent.locks` 里的可重入互斥锁。

它和 `synchronized` 一样，都能解决多线程同时修改共享数据带来的线程安全问题。

一句话概括：

```text
ReentrantLock 是一把需要手动加锁、手动释放的可重入锁，比 synchronized 多了超时获取、可中断等待、公平锁和多个 Condition 等能力。
```

典型写法：

```java
lock.lock();
try {
    // 临界区代码
} finally {
    lock.unlock();
}
```

`finally` 很关键。

只要拿到了锁，就必须保证最后释放。否则业务代码中途抛异常后，锁一直不释放，后面的线程会全部卡住。

### 为什么需要 ReentrantLock

先看 `synchronized`。

```java
public synchronized void increment() {
    count++;
}
```

这种写法简单直接。

但它有几个限制：

* 等锁时不能设置超时时间
* 等锁时不方便响应中断
* 不能选择公平锁
* 只能配合一个对象监视器等待队列
* 不能像 API 一样灵活组合

`ReentrantLock` 提供了更多控制能力：

| 能力 | 说明 |
| --- | --- |
| `lock()` | 阻塞获取锁 |
| `tryLock()` | 立即尝试获取锁，拿不到直接返回 |
| `tryLock(timeout, unit)` | 最多等待一段时间 |
| `lockInterruptibly()` | 等锁时可以响应中断 |
| `newCondition()` | 创建多个条件队列 |
| 公平锁 | 可按等待顺序分配锁 |
| 查询方法 | 查看锁状态、等待线程估算数量、重入次数 |

所以，简单同步用 `synchronized` 就够。

需要更强控制时，再考虑 `ReentrantLock`。

### 可重入是什么意思

`ReentrantLock` 里的 `Reentrant` 表示可重入。

可重入的意思是：

```text
同一个线程已经拿到锁后，可以再次拿同一把锁，不会把自己锁死。
```

示例：

```java
import java.util.concurrent.locks.ReentrantLock;

public class ReentrantDemo {

    private final ReentrantLock lock = new ReentrantLock();

    public void outer() {
        lock.lock();
        try {
            System.out.println("outer holdCount=" + lock.getHoldCount());
            inner();
        } finally {
            lock.unlock();
            System.out.println("outer unlock");
        }
    }

    private void inner() {
        lock.lock();
        try {
            System.out.println("inner holdCount=" + lock.getHoldCount());
        } finally {
            lock.unlock();
            System.out.println("inner unlock");
        }
    }

    public static void main(String[] args) {
        new ReentrantDemo().outer();
    }
}
```

输出类似：

```text
outer holdCount=1
inner holdCount=2
inner unlock
outer unlock
```

注意：

```text
加锁几次，就要释放几次。
```

同一个线程加了 2 次锁，只释放 1 次，锁仍然没有完全释放。

### ReentrantLock 和 synchronized 对比

| 对比项 | synchronized | ReentrantLock |
| --- | --- | --- |
| 类型 | Java 关键字 | JUC API |
| 释放锁 | 自动释放 | 必须手动 `unlock()` |
| 可重入 | 支持 | 支持 |
| 公平锁 | 不支持显式选择 | 构造函数可指定 |
| 尝试获取 | 不支持 | `tryLock()` |
| 超时获取 | 不支持 | `tryLock(time, unit)` |
| 可中断等待 | 不支持 | `lockInterruptibly()` |
| 条件队列 | `wait/notify` 一个等待集合 | 可创建多个 `Condition` |
| 可读性 | 简单场景更清楚 | 复杂场景更灵活 |
| 性能 | 现代 JVM 优化后很好 | 和场景有关，不应只为性能替换 |

常见选择：

```text
简单互斥：synchronized
需要超时、取消、公平、多条件队列：ReentrantLock
读多写少：ReentrantReadWriteLock 或 StampedLock
限制并发数量：Semaphore
```

### 构造方法

默认构造：

```java
ReentrantLock lock = new ReentrantLock();
```

等价于：

```java
ReentrantLock lock = new ReentrantLock(false);
```

也就是非公平锁。

公平锁：

```java
ReentrantLock fairLock = new ReentrantLock(true);
```

两者区别：

| 锁 | 说明 |
| --- | --- |
| 非公平锁 | 新来的线程可以直接竞争锁，吞吐量通常更高 |
| 公平锁 | 倾向于让等待时间最长的线程先拿锁，减少饥饿 |

公平锁不是绝对严格的全局公平。

官方文档里有一个容易忽略的细节：即使创建了公平锁，普通 `tryLock()` 只要看到锁空闲，也会立即尝试获取，可能不遵守公平排队。如果希望尽量遵守公平策略，可以使用 `tryLock(0, TimeUnit.SECONDS)`。

### 常用方法

| 方法 | 作用 |
| --- | --- |
| `lock()` | 获取锁，拿不到就一直等 |
| `lockInterruptibly()` | 获取锁，等待期间可响应中断 |
| `tryLock()` | 立即尝试获取锁，成功返回 `true` |
| `tryLock(time, unit)` | 在指定时间内尝试获取锁 |
| `unlock()` | 释放锁 |
| `newCondition()` | 创建条件变量 |
| `isLocked()` | 锁是否被任意线程持有 |
| `isHeldByCurrentThread()` | 当前线程是否持有锁 |
| `getHoldCount()` | 当前线程重入次数 |
| `isFair()` | 是否公平锁 |
| `hasQueuedThreads()` | 是否有线程在等待锁 |
| `getQueueLength()` | 估算等待锁的线程数量 |

这些状态查询方法适合监控和排查问题。

业务逻辑不要强依赖 `getQueueLength()` 这类估算结果做精确判断。

### 第一个 Demo：计数器线程安全

`count++` 不是原子操作。

它大致包含：

```text
读取 count
加 1
写回 count
```

多个线程同时执行时，结果可能丢失。

不加锁示例：

```java
public class UnsafeCounter {

    private int count;

    public void increment() {
        count++;
    }

    public int getCount() {
        return count;
    }
}
```

使用 `ReentrantLock`：

```java
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.locks.ReentrantLock;

public class SafeCounterDemo {

    static class SafeCounter {
        private final ReentrantLock lock = new ReentrantLock();
        private int count;

        public void increment() {
            lock.lock();
            try {
                count++;
            } finally {
                lock.unlock();
            }
        }

        public int getCount() {
            lock.lock();
            try {
                return count;
            } finally {
                lock.unlock();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        SafeCounter counter = new SafeCounter();
        List<Thread> threads = new ArrayList<>();

        for (int i = 0; i < 100; i++) {
            Thread thread = new Thread(() -> {
                for (int j = 0; j < 1000; j++) {
                    counter.increment();
                }
            });
            threads.add(thread);
            thread.start();
        }

        for (Thread thread : threads) {
            thread.join();
        }

        System.out.println(counter.getCount());
    }
}
```

输出：

```text
100000
```

### 为什么 unlock 必须放 finally

错误写法：

```java
lock.lock();
doBusiness();
lock.unlock();
```

如果 `doBusiness()` 抛异常，`unlock()` 不会执行。

正确写法：

```java
lock.lock();
try {
    doBusiness();
} finally {
    lock.unlock();
}
```

如果代码里存在多个返回分支，也一样：

```java
lock.lock();
try {
    if (condition) {
        return;
    }
    doBusiness();
} finally {
    lock.unlock();
}
```

`finally` 会保证方法返回前释放锁。

### tryLock：拿不到锁就不等

`tryLock()` 适合“能做就做，不能做就跳过”的场景。

比如后台刷新缓存，没必要多个线程一起排队刷新。

```java
import java.util.concurrent.locks.ReentrantLock;

public class CacheRefreshService {

    private final ReentrantLock refreshLock = new ReentrantLock();

    public void refresh() {
        if (!refreshLock.tryLock()) {
            System.out.println("已有线程在刷新缓存，本次跳过");
            return;
        }

        try {
            System.out.println("开始刷新缓存");
            sleep(1000);
            System.out.println("缓存刷新完成");
        } finally {
            refreshLock.unlock();
        }
    }

    private void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

`tryLock()` 不会阻塞等待。

拿不到锁就返回 `false`。

### tryLock 超时：最多等一会

有些业务不能无限等锁。

比如库存扣减、订单状态更新、用户提交请求。

等太久不如直接返回失败。

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

public class StockService {

    private final ReentrantLock lock = new ReentrantLock();
    private int stock = 10;

    public boolean deduct(int amount) {
        boolean locked = false;

        try {
            locked = lock.tryLock(500, TimeUnit.MILLISECONDS);
            if (!locked) {
                System.out.println("系统繁忙，稍后重试");
                return false;
            }

            if (stock < amount) {
                System.out.println("库存不足");
                return false;
            }

            stock -= amount;
            System.out.println("扣减成功，剩余库存：" + stock);
            return true;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.out.println("等待锁时被中断");
            return false;
        } finally {
            if (locked) {
                lock.unlock();
            }
        }
    }
}
```

这里要注意：

```java
if (locked) {
    lock.unlock();
}
```

只有成功拿到锁，才能释放锁。

没拿到锁却调用 `unlock()`，会抛 `IllegalMonitorStateException`。

### lockInterruptibly：等待锁时可以取消

`lock()` 等待锁时不会因为普通中断直接退出。

`lockInterruptibly()` 可以在等待锁时响应中断。

适合可取消任务、后台任务停止、避免线程长期挂起。

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

public class InterruptibleLockDemo {

    private final ReentrantLock lock = new ReentrantLock();

    public void longTask() {
        lock.lock();
        try {
            System.out.println("线程 A 持有锁");
            sleep(5000);
        } finally {
            lock.unlock();
        }
    }

    public void cancelableTask() {
        boolean locked = false;

        try {
            lock.lockInterruptibly();
            locked = true;
            System.out.println("线程 B 获取锁成功");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.out.println("线程 B 等锁时被中断，直接退出");
        } finally {
            if (locked) {
                lock.unlock();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        InterruptibleLockDemo demo = new InterruptibleLockDemo();

        Thread threadA = new Thread(demo::longTask, "A");
        Thread threadB = new Thread(demo::cancelableTask, "B");

        threadA.start();
        TimeUnit.MILLISECONDS.sleep(100);

        threadB.start();
        TimeUnit.SECONDS.sleep(1);

        threadB.interrupt();

        threadA.join();
        threadB.join();
    }

    private void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

输出类似：

```text
线程 A 持有锁
线程 B 等锁时被中断，直接退出
```

### Condition 是什么

`Condition` 可以理解成和 `ReentrantLock` 绑定的等待队列。

它类似 `Object.wait()` / `notify()`，但更灵活。

`synchronized` 只有一个等待集合：

```text
同一把锁
  |
  v
一个 wait 队列
```

`ReentrantLock` 可以创建多个 `Condition`：

```text
同一把锁
  |
  +--> notFull 队列
  |
  +--> notEmpty 队列
```

这样生产者只唤醒消费者，消费者只唤醒生产者，逻辑更清楚。

常用方法：

| 方法 | 作用 |
| --- | --- |
| `await()` | 当前线程等待，并释放锁 |
| `await(timeout, unit)` | 限时等待 |
| `signal()` | 唤醒一个等待线程 |
| `signalAll()` | 唤醒所有等待线程 |

`await()` 有两个关键点：

```text
调用 await 时会释放当前锁。
被唤醒返回前，需要重新获取同一把锁。
```

### Condition Demo：有界队列

下面实现一个简单有界队列。

队列满时，生产者等待。

队列空时，消费者等待。

```java
import java.util.ArrayDeque;
import java.util.Queue;
import java.util.concurrent.locks.Condition;
import java.util.concurrent.locks.ReentrantLock;

public class BoundedQueue<T> {

    private final Queue<T> queue = new ArrayDeque<>();
    private final int capacity;
    private final ReentrantLock lock = new ReentrantLock();
    private final Condition notFull = lock.newCondition();
    private final Condition notEmpty = lock.newCondition();

    public BoundedQueue(int capacity) {
        if (capacity <= 0) {
            throw new IllegalArgumentException("capacity must be positive");
        }
        this.capacity = capacity;
    }

    public void put(T item) throws InterruptedException {
        lock.lock();
        try {
            while (queue.size() == capacity) {
                notFull.await();
            }

            queue.offer(item);
            notEmpty.signal();
        } finally {
            lock.unlock();
        }
    }

    public T take() throws InterruptedException {
        lock.lock();
        try {
            while (queue.isEmpty()) {
                notEmpty.await();
            }

            T item = queue.poll();
            notFull.signal();
            return item;
        } finally {
            lock.unlock();
        }
    }

    public int size() {
        lock.lock();
        try {
            return queue.size();
        } finally {
            lock.unlock();
        }
    }
}
```

测试：

```java
public class BoundedQueueDemo {

    public static void main(String[] args) {
        BoundedQueue<Integer> queue = new BoundedQueue<>(3);

        Thread producer = new Thread(() -> {
            try {
                for (int i = 1; i <= 10; i++) {
                    queue.put(i);
                    System.out.println("生产：" + i + "，size=" + queue.size());
                    Thread.sleep(100);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "producer");

        Thread consumer = new Thread(() -> {
            try {
                for (int i = 1; i <= 10; i++) {
                    Integer item = queue.take();
                    System.out.println("消费：" + item + "，size=" + queue.size());
                    Thread.sleep(300);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "consumer");

        producer.start();
        consumer.start();
    }
}
```

这里必须用 `while` 判断条件，不能用 `if`。

原因有两个：

* 线程被唤醒后，不代表条件一定满足
* `Condition` 允许虚假唤醒

正确写法：

```java
while (queue.isEmpty()) {
    notEmpty.await();
}
```

不要写：

```java
if (queue.isEmpty()) {
    notEmpty.await();
}
```

### signal 和 signalAll 怎么选

`signal()` 唤醒一个等待线程。

`signalAll()` 唤醒所有等待线程。

选择建议：

| 方法 | 适合场景 |
| --- | --- |
| `signal()` | 每次状态变化只够一个线程继续执行 |
| `signalAll()` | 状态变化可能让多个线程继续执行，或条件复杂 |

有界队列里：

```java
notEmpty.signal();
notFull.signal();
```

通常够用。

如果业务条件很多、容易漏唤醒，先用 `signalAll()` 保守一点，再结合性能情况优化。

### 公平锁 Demo

公平锁会尽量按等待顺序分配锁。

非公平锁可能让新来的线程插队。

```java
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

public class FairLockDemo {

    private final ReentrantLock lock = new ReentrantLock(true);

    public void work() {
        lock.lock();
        try {
            System.out.println(Thread.currentThread().getName() + " 获取锁");
            TimeUnit.MILLISECONDS.sleep(200);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) {
        FairLockDemo demo = new FairLockDemo();

        for (int i = 0; i < 5; i++) {
            new Thread(demo::work, "thread-" + i).start();
        }
    }
}
```

公平锁的代价是吞吐量通常更低。

原因是它更强调排队顺序，会带来更多线程调度和上下文切换。

大多数业务优先使用默认非公平锁。

只有明确需要减少线程饥饿或顺序性时，再考虑公平锁。

### 实战 Demo：转账避免死锁

转账经常需要同时锁两个账户。

如果两个线程这样执行：

```text
线程 1：锁 A -> 等 B
线程 2：锁 B -> 等 A
```

就可能死锁。

解决方式之一：按固定顺序加锁。

```java
import java.math.BigDecimal;
import java.util.concurrent.locks.ReentrantLock;

public class Account {

    private final long id;
    private BigDecimal balance;
    private final ReentrantLock lock = new ReentrantLock();

    public Account(long id, BigDecimal balance) {
        this.id = id;
        this.balance = balance;
    }

    public static boolean transfer(Account from, Account to, BigDecimal amount) {
        Account first = from.id < to.id ? from : to;
        Account second = from.id < to.id ? to : from;

        first.lock.lock();
        try {
            second.lock.lock();
            try {
                if (from.balance.compareTo(amount) < 0) {
                    return false;
                }

                from.balance = from.balance.subtract(amount);
                to.balance = to.balance.add(amount);
                return true;
            } finally {
                second.lock.unlock();
            }
        } finally {
            first.lock.unlock();
        }
    }

    public BigDecimal getBalance() {
        lock.lock();
        try {
            return balance;
        } finally {
            lock.unlock();
        }
    }
}
```

关键点：

```text
所有线程都按账户 id 从小到大加锁。
```

只要加锁顺序统一，就不会出现 A 等 B、B 等 A 的环路。

另一种方式是用 `tryLock(timeout)`，拿不到第二把锁就释放第一把锁并返回失败。

适合不想长期等待的业务。

### 实战 Demo：转账超时失败

```java
import java.math.BigDecimal;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.locks.ReentrantLock;

public class TimeoutTransferService {

    static class Account {
        private final String name;
        private BigDecimal balance;
        private final ReentrantLock lock = new ReentrantLock();

        Account(String name, BigDecimal balance) {
            this.name = name;
            this.balance = balance;
        }
    }

    public boolean transfer(Account from, Account to, BigDecimal amount) {
        boolean fromLocked = false;
        boolean toLocked = false;

        try {
            fromLocked = from.lock.tryLock(300, TimeUnit.MILLISECONDS);
            if (!fromLocked) {
                return false;
            }

            toLocked = to.lock.tryLock(300, TimeUnit.MILLISECONDS);
            if (!toLocked) {
                return false;
            }

            if (from.balance.compareTo(amount) < 0) {
                return false;
            }

            from.balance = from.balance.subtract(amount);
            to.balance = to.balance.add(amount);
            return true;
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        } finally {
            if (toLocked) {
                to.lock.unlock();
            }
            if (fromLocked) {
                from.lock.unlock();
            }
        }
    }
}
```

这种写法的重点是：

```text
分别记录每把锁是否获取成功。
finally 里只释放已经拿到的锁。
释放顺序和加锁顺序相反。
```

### ReentrantLock 底层和 AQS

`ReentrantLock` 底层基于 `AbstractQueuedSynchronizer`，简称 `AQS`。

简单理解：

```text
AQS 用一个 int state 表示同步状态
用队列管理等待线程
ReentrantLock 用 state 表示锁重入次数
```

大致关系：

```text
state = 0：锁空闲
state = 1：当前线程持有锁 1 次
state = 2：当前线程重入 2 次
```

线程获取不到锁时，会进入 AQS 等待队列。

释放锁时，`state` 减 1。

只有减到 0，锁才真正释放，后面的线程才有机会获取。

这也是“加几次锁，就要解几次锁”的底层原因。

### 和虚拟线程的关系

JDK 21 之后，虚拟线程已经正式可用。

在虚拟线程里使用 `ReentrantLock` 是可以的。

示例：

```java
import java.util.concurrent.Executors;
import java.util.concurrent.locks.ReentrantLock;

public class VirtualThreadLockDemo {

    private static final ReentrantLock lock = new ReentrantLock();
    private static int count;

    public static void main(String[] args) {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 0; i < 1000; i++) {
                executor.submit(() -> {
                    lock.lock();
                    try {
                        count++;
                    } finally {
                        lock.unlock();
                    }
                });
            }
        }

        System.out.println(count);
    }
}
```

但锁竞争本身不会因为虚拟线程消失。

如果所有虚拟线程都抢同一把锁，临界区仍然只能一个线程执行。

虚拟线程降低的是线程成本，不是把串行临界区变成并行。

### 常见坑

### 忘记 unlock

最严重的错误：

```java
lock.lock();
doBusiness();
```

没有 `unlock()`，其他线程会一直等。

标准写法：

```java
lock.lock();
try {
    doBusiness();
} finally {
    lock.unlock();
}
```

### 没拿到锁也 unlock

错误写法：

```java
try {
    lock.tryLock(1, TimeUnit.SECONDS);
    doBusiness();
} finally {
    lock.unlock();
}
```

如果 `tryLock` 返回 `false`，这里会释放一把并未持有的锁，抛 `IllegalMonitorStateException`。

正确写法：

```java
boolean locked = false;
try {
    locked = lock.tryLock(1, TimeUnit.SECONDS);
    if (!locked) {
        return;
    }
    doBusiness();
} finally {
    if (locked) {
        lock.unlock();
    }
}
```

### Condition 用 if 判断

错误写法：

```java
if (queue.isEmpty()) {
    notEmpty.await();
}
```

正确写法：

```java
while (queue.isEmpty()) {
    notEmpty.await();
}
```

原因是线程醒来后必须重新检查条件。

### await / signal 时没有持有锁

`Condition` 的 `await()`、`signal()` 通常要求当前线程已经持有对应的锁。

错误写法：

```java
notEmpty.signal();
```

正确写法：

```java
lock.lock();
try {
    notEmpty.signal();
} finally {
    lock.unlock();
}
```

否则可能抛 `IllegalMonitorStateException`。

### 公平锁当成性能优化

公平锁主要解决等待顺序和饥饿问题。

它通常不是性能优化手段。

高吞吐场景一般优先默认非公平锁。

### 锁粒度太大

锁保护的代码越多，竞争越严重。

不推荐：

```java
lock.lock();
try {
    queryDatabase();
    callRemoteApi();
    updateMemoryState();
} finally {
    lock.unlock();
}
```

更好的方向是缩小临界区：

```text
锁内只保护必须同步的共享状态。
耗时 I/O 尽量放到锁外。
```

### 能用原子类却用了锁

简单计数可以用：

```java
AtomicInteger
LongAdder
```

比如高并发统计计数，`LongAdder` 往往比一把锁更合适。

`ReentrantLock` 更适合保护一段复合逻辑，而不只是单个自增。

### 总结

`ReentrantLock` 可以按这条线理解：

```text
它是一把可重入互斥锁。
同一线程可以重复加锁。
每次加锁都要对应一次释放。
tryLock 支持不等待或限时等待。
lockInterruptibly 支持等待时取消。
Condition 支持多个等待队列。
公平锁可以减少饥饿，但通常牺牲吞吐量。
```

实际使用时，优先记住这些规则：

* `lock()` 后一定 `try-finally-unlock`
* `tryLock()` 后只在成功获取锁时释放
* `Condition.await()` 放在 `while` 里
* `await()` 和 `signal()` 必须在持锁状态下调用
* 公平锁慎用
* 临界区尽量短
* 简单同步优先考虑 `synchronized`
* 简单计数优先考虑原子类

`ReentrantLock` 的价值不在于替代所有 `synchronized`，而是在复杂并发控制里提供更细的操作空间。需要超时、取消、公平、多条件等待时，它比内置锁更合适；只是普通互斥时，`synchronized` 往往更简单，也更不容易忘记释放锁。
