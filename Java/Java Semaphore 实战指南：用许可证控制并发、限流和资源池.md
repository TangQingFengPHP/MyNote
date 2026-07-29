### 简介

`Semaphore` 是 Java 并发包 `java.util.concurrent` 里的同步工具。

它的中文名一般叫信号量。

最容易理解的说法是：

```text
Semaphore 像一组许可证。
线程先拿许可证，拿到后才能继续执行。
执行完再归还许可证。
许可证用完后，后面的线程等待或直接失败。
```

比如：

```java
Semaphore semaphore = new Semaphore(3);
```

表示一共有 3 个许可证。

同一时刻最多 3 个线程能拿到许可证并进入受保护的代码段。

一句话概括：

```text
Semaphore 适合控制并发数量，比如接口并发限制、资源池、连接数控制、批量任务限流。
```

### Semaphore 解决什么问题

一个接口可能会访问下游资源：

```text
HTTP 请求
  |
  v
订单接口
  |
  v
数据库 / Redis / 第三方接口
```

如果瞬间来了很多请求，全部直接打到下游，可能出现：

```text
数据库连接耗尽
第三方接口超时
线程池队列堆积
CPU 上下文切换变多
服务响应变慢
```

Semaphore 可以在入口处加一道并发闸门：

```text
1000 个请求
  |
  v
Semaphore(50)
  |
  +--> 同一时刻最多 50 个请求进入业务逻辑
  +--> 其他请求等待、超时或直接返回繁忙
```

它控制的是“同时正在执行的数量”。

这点和按时间窗口统计 QPS 的限流器不一样。

### 核心模型：许可证

Semaphore 内部维护一个许可数量。

```java
Semaphore semaphore = new Semaphore(2);
```

初始状态：

```text
permits = 2
```

线程 A 调用：

```java
semaphore.acquire();
```

拿走 1 个许可：

```text
permits = 1
```

线程 B 再拿走 1 个：

```text
permits = 0
```

线程 C 再调用 `acquire()`：

```text
没有可用许可，线程 C 阻塞等待
```

当线程 A 执行完：

```java
semaphore.release();
```

归还 1 个许可：

```text
permits = 1
```

等待中的线程就有机会继续执行。

### 常用 API

| 方法 | 说明 |
| --- | --- |
| `acquire()` | 获取 1 个许可，没有许可就阻塞，可响应中断 |
| `acquire(int permits)` | 一次获取多个许可 |
| `acquireUninterruptibly()` | 获取 1 个许可，阻塞时不响应中断 |
| `tryAcquire()` | 尝试获取 1 个许可，立即返回成功或失败 |
| `tryAcquire(timeout, unit)` | 在指定时间内尝试获取许可 |
| `release()` | 释放 1 个许可 |
| `release(int permits)` | 释放多个许可 |
| `availablePermits()` | 查看当前可用许可数量 |
| `getQueueLength()` | 估算正在等待许可的线程数量 |
| `hasQueuedThreads()` | 判断是否有线程在等待 |
| `drainPermits()` | 一次性拿走当前所有可用许可 |

最常用的是：

```text
acquire()
tryAcquire()
tryAcquire(timeout, unit)
release()
```

### 第一个 Demo：最多 3 个线程同时执行

下面模拟 10 个任务，但同一时刻最多只允许 3 个任务执行。

```java
package com.example.semaphore;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class SemaphoreBasicDemo {

    private static final Semaphore SEMAPHORE = new Semaphore(3);

    public static void main(String[] args) {
        for (int i = 1; i <= 10; i++) {
            int taskId = i;

            Thread thread = new Thread(() -> runTask(taskId), "task-" + taskId);
            thread.start();
        }
    }

    private static void runTask(int taskId) {
        boolean acquired = false;

        try {
            SEMAPHORE.acquire();
            acquired = true;

            System.out.printf(
                    "任务 %d 开始执行，可用许可=%d%n",
                    taskId,
                    SEMAPHORE.availablePermits()
            );

            TimeUnit.SECONDS.sleep(2);

            System.out.printf("任务 %d 执行完成%n", taskId);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            System.out.printf("任务 %d 被中断%n", taskId);
        } finally {
            if (acquired) {
                SEMAPHORE.release();
            }
        }
    }
}
```

这里用了一个 `acquired` 标记。

原因很关键：

```text
只有真正拿到许可后，finally 里才释放许可。
```

如果线程在 `acquire()` 阻塞等待时被中断，实际并没有拿到许可。此时如果无条件 `release()`，许可数量会被凭空增加，并发控制会失效。

### acquire 和 release 要成对

正确写法：

```java
boolean acquired = false;

try {
    semaphore.acquire();
    acquired = true;

    // 业务逻辑
} finally {
    if (acquired) {
        semaphore.release();
    }
}
```

如果 `acquire()` 已经成功，并且业务逻辑抛异常，`finally` 也能释放许可。

如果 `acquire()` 没成功，`finally` 不会释放许可。

这点比直接写下面这种更稳：

```java
try {
    semaphore.acquire();
    // 业务逻辑
} finally {
    semaphore.release();
}
```

后者在 `acquire()` 被中断时可能多释放一次。

### tryAcquire：拿不到就直接拒绝

接口限流里，经常不希望请求一直排队。

如果许可已经用完，可以直接返回繁忙。

```java
package com.example.semaphore;

import java.util.concurrent.Semaphore;

public class FastRejectLimiter {

    private final Semaphore semaphore;

    public FastRejectLimiter(int maxConcurrent) {
        this.semaphore = new Semaphore(maxConcurrent);
    }

    public String handle(int requestId) {
        boolean acquired = semaphore.tryAcquire();

        if (!acquired) {
            return "请求 " + requestId + " 被限流";
        }

        try {
            return doBusiness(requestId);
        } finally {
            semaphore.release();
        }
    }

    private String doBusiness(int requestId) {
        return "请求 " + requestId + " 处理成功";
    }
}
```

`tryAcquire()` 不会阻塞。

适合：

```text
没有容量就快速失败
避免请求长期排队
保护后端资源
```

### tryAcquire 超时等待

有些场景可以等一小会儿。

比如最多等 200 毫秒，拿到许可就处理，拿不到就返回繁忙。

```java
package com.example.semaphore;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class TimeoutLimiter {

    private final Semaphore semaphore = new Semaphore(5);

    public String handle(int requestId) {
        boolean acquired = false;

        try {
            acquired = semaphore.tryAcquire(200, TimeUnit.MILLISECONDS);

            if (!acquired) {
                return "系统繁忙，请稍后再试";
            }

            return doBusiness(requestId);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return "请求被中断";
        } finally {
            if (acquired) {
                semaphore.release();
            }
        }
    }

    private String doBusiness(int requestId) {
        return "request-" + requestId;
    }
}
```

相比 `acquire()` 无限等待，带超时的 `tryAcquire` 更适合 Web 接口。

### Spring Boot 接口并发限制

下面用 `Semaphore` 给某个接口做并发限制。

```java
package com.example.semaphore.web;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/reports")
public class ReportController {

    private final Semaphore reportSemaphore = new Semaphore(10);

    @GetMapping("/export")
    public ResponseEntity<String> exportReport() {
        boolean acquired = false;

        try {
            acquired = reportSemaphore.tryAcquire(1, TimeUnit.SECONDS);

            if (!acquired) {
                return ResponseEntity
                        .status(HttpStatus.TOO_MANY_REQUESTS)
                        .body("报表导出请求过多，请稍后重试");
            }

            String fileUrl = exportLargeReport();
            return ResponseEntity.ok(fileUrl);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
                    .body("报表导出被中断");
        } finally {
            if (acquired) {
                reportSemaphore.release();
            }
        }
    }

    private String exportLargeReport() {
        return "https://cdn.example.com/report/report-10001.xlsx";
    }
}
```

这个接口同一时刻最多允许 10 个导出任务执行。

其他请求最多等 1 秒，仍然拿不到许可就返回 `429 Too Many Requests`。

这种做法适合保护 CPU、数据库、文件导出、第三方接口等有限资源。

### 连接池 Demo

Semaphore 很适合控制有限资源。

下面写一个简化版连接池：

```java
package com.example.semaphore.pool;

import java.util.ArrayDeque;
import java.util.Queue;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class SimpleConnectionPool {

    private final Semaphore semaphore;

    private final Queue<MockConnection> idleConnections = new ArrayDeque<>();

    public SimpleConnectionPool(int size) {
        this.semaphore = new Semaphore(size, true);
        for (int i = 1; i <= size; i++) {
            idleConnections.add(new MockConnection("conn-" + i));
        }
    }

    public MockConnection borrow(long timeout, TimeUnit unit) throws InterruptedException {
        boolean acquired = semaphore.tryAcquire(timeout, unit);
        if (!acquired) {
            return null;
        }

        synchronized (idleConnections) {
            MockConnection connection = idleConnections.poll();
            if (connection == null) {
                semaphore.release();
                throw new IllegalStateException("许可和连接池状态不一致");
            }
            return connection;
        }
    }

    public void release(MockConnection connection) {
        if (connection == null) {
            return;
        }

        synchronized (idleConnections) {
            idleConnections.offer(connection);
        }
        semaphore.release();
    }

    public static class MockConnection {

        private final String name;

        public MockConnection(String name) {
            this.name = name;
        }

        public void execute(String sql) {
            System.out.println(name + " 执行 SQL：" + sql);
        }
    }
}
```

测试：

```java
package com.example.semaphore.pool;

import java.util.concurrent.TimeUnit;

public class ConnectionPoolDemo {

    public static void main(String[] args) {
        SimpleConnectionPool pool = new SimpleConnectionPool(3);

        for (int i = 1; i <= 8; i++) {
            int taskId = i;
            new Thread(() -> useConnection(pool, taskId), "worker-" + taskId).start();
        }
    }

    private static void useConnection(SimpleConnectionPool pool, int taskId) {
        SimpleConnectionPool.MockConnection connection = null;

        try {
            connection = pool.borrow(1, TimeUnit.SECONDS);

            if (connection == null) {
                System.out.println("任务 " + taskId + " 获取连接超时");
                return;
            }

            connection.execute("select * from user where id = " + taskId);
            TimeUnit.MILLISECONDS.sleep(500);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            pool.release(connection);
        }
    }
}
```

这个例子里：

```text
Semaphore 控制最多同时借出几个连接
Queue 保存真正的连接对象
synchronized 保护 Queue 的线程安全
```

Semaphore 只负责控制数量，不负责保存资源对象。

资源对象需要另外的数据结构管理。

### 公平模式和非公平模式

Semaphore 构造方法有一个 `fair` 参数。

非公平模式：

```java
Semaphore semaphore = new Semaphore(5);
```

公平模式：

```java
Semaphore semaphore = new Semaphore(5, true);
```

区别：

| 模式 | 说明 | 特点 |
| --- | --- | --- |
| 非公平 | 释放许可后，后来的线程也可能先抢到 | 吞吐通常更高 |
| 公平 | 等待较久的线程优先获取许可 | 排队更有秩序，但开销更高 |

接口限流、高吞吐场景一般用默认非公平模式。

资源池、等待时间比较敏感的场景可以考虑公平模式。

### 一次获取多个许可

有些任务占用的资源不是 1 份。

比如批量导出任务按数据量占用不同许可。

```java
package com.example.semaphore;

import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class MultiPermitDemo {

    private static final Semaphore SEMAPHORE = new Semaphore(10);

    public static void main(String[] args) {
        runJob("small-job", 2);
        runJob("big-job", 7);
        runJob("medium-job", 4);
    }

    private static void runJob(String jobName, int permits) {
        Thread thread = new Thread(() -> {
            boolean acquired = false;

            try {
                SEMAPHORE.acquire(permits);
                acquired = true;

                System.out.printf("%s 获取 %d 个许可%n", jobName, permits);
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                if (acquired) {
                    SEMAPHORE.release(permits);
                    System.out.printf("%s 释放 %d 个许可%n", jobName, permits);
                }
            }
        });

        thread.start();
    }
}
```

使用多个许可时，获取和释放数量要一致。

```text
acquire(permits)
release(permits)
```

### drainPermits：临时抽干许可

`drainPermits()` 会拿走当前所有可用许可，并返回拿走的数量。

```java
int drained = semaphore.drainPermits();
```

它适合临时暂停新任务进入。

```java
package com.example.semaphore;

import java.util.concurrent.Semaphore;

public class DrainPermitsDemo {

    private final Semaphore semaphore = new Semaphore(5);

    public int pauseNewRequests() {
        return semaphore.drainPermits();
    }

    public void resumeNewRequests(int permits) {
        semaphore.release(permits);
    }
}
```

注意，`drainPermits()` 只会拿走当前还没被线程占用的许可。

已经拿到许可并正在执行的线程不会被强制停止。

### Semaphore 和虚拟线程

虚拟线程可以创建很多并发任务，但下游资源并不会因为虚拟线程变多而自动变强。

比如同时发起 10000 个 HTTP 调用，如果第三方接口只能承受 100 个并发，仍然需要限制并发。

Semaphore 可以和虚拟线程一起使用。

```java
package com.example.semaphore.virtualthread;

import java.util.concurrent.Executors;
import java.util.concurrent.Semaphore;
import java.util.concurrent.TimeUnit;

public class VirtualThreadSemaphoreDemo {

    private static final Semaphore SEMAPHORE = new Semaphore(100);

    public static void main(String[] args) {
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 1; i <= 1000; i++) {
                int taskId = i;
                executor.submit(() -> callRemoteService(taskId));
            }
        }
    }

    private static void callRemoteService(int taskId) {
        boolean acquired = false;

        try {
            acquired = SEMAPHORE.tryAcquire(2, TimeUnit.SECONDS);

            if (!acquired) {
                System.out.println("任务 " + taskId + " 获取许可超时");
                return;
            }

            TimeUnit.MILLISECONDS.sleep(200);
            System.out.println("任务 " + taskId + " 调用完成");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            if (acquired) {
                SEMAPHORE.release();
            }
        }
    }
}
```

虚拟线程解决的是线程成本问题。

Semaphore 解决的是下游资源并发上限问题。

两者并不冲突。

### 和线程池的区别

线程池也能限制并发。

比如：

```java
Executors.newFixedThreadPool(10);
```

它限制的是同时执行任务的线程数量。

Semaphore 限制的是进入某段代码或访问某个资源的并发数量。

对比：

| 对比项 | 线程池 | Semaphore |
| --- | --- | --- |
| 控制对象 | 任务执行线程数 | 某段代码或资源的并发数 |
| 是否负责执行任务 | 负责 | 不负责 |
| 是否保存任务队列 | 通常有 | 没有 |
| 是否能包住局部逻辑 | 不方便 | 很方便 |
| 常见场景 | 执行异步任务 | 限制资源访问 |

两者可以一起使用：

```text
线程池负责执行任务
Semaphore 负责限制访问数据库、接口、文件等资源
```

### 和其他同步工具对比

| 工具 | 作用 | 特点 |
| --- | --- | --- |
| `synchronized` | 互斥访问 | 同一时刻只允许一个线程进入 |
| `ReentrantLock` | 互斥锁 | 比 `synchronized` 更灵活 |
| `Semaphore` | 控制并发数量 | 同一时刻允许 N 个线程进入 |
| `CountDownLatch` | 等待一组任务完成 | 一次性倒计时 |
| `CyclicBarrier` | 多个线程互相等待 | 可以循环使用 |

如果资源只能一个线程访问，用锁。

如果资源允许 N 个线程同时访问，用 Semaphore。

### 常见问题

#### release 次数超过 acquire 会怎样

Semaphore 不会检查“最大许可数”。

下面这段代码会让许可数量变大：

```java
Semaphore semaphore = new Semaphore(1);

semaphore.release();
semaphore.release();

System.out.println(semaphore.availablePermits());
```

输出可能是：

```text
3
```

所以 `release()` 要和成功的 `acquire()` 配对。

#### availablePermits 能不能做业务判断

`availablePermits()` 适合观察和监控，不适合做并发安全的业务判断。

比如：

```java
if (semaphore.availablePermits() > 0) {
    semaphore.acquire();
}
```

这不是原子操作。

判断时有许可，不代表下一行一定还能拿到许可。

需要尝试获取时，使用：

```java
boolean acquired = semaphore.tryAcquire();
```

#### Semaphore 能不能做分布式限流

不能直接做分布式限流。

Semaphore 只在当前 JVM 进程内生效。

如果应用部署了 5 个实例，每个实例都有一个 `Semaphore(100)`，整体并发上限可能变成：

```text
5 * 100 = 500
```

集群级限流通常需要 Redis、网关限流、服务治理框架或专门限流组件。

#### acquire 被中断后怎么处理

`acquire()` 会响应中断。

捕获 `InterruptedException` 后，建议恢复中断标记：

```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

这样上层代码还有机会感知中断状态。

#### Semaphore 是不是 QPS 限流器

Semaphore 控制的是同一时刻有多少任务正在执行。

QPS 限流控制的是单位时间内允许多少请求进入。

对比：

| 类型 | 控制目标 |
| --- | --- |
| Semaphore | 并发数 |
| 令牌桶 / 漏桶 | 单位时间流量 |

如果目标是“最多 100 个请求同时处理”，Semaphore 很合适。

如果目标是“每秒最多 1000 个请求”，更适合令牌桶、漏桶或网关限流。

### 实践建议

| 场景 | 建议 |
| --- | --- |
| 接口并发保护 | 使用 `tryAcquire(timeout)` |
| 快速失败 | 使用 `tryAcquire()` |
| 资源池 | Semaphore 控制数量，队列保存资源对象 |
| Web 接口 | 超时后返回 `429` 或业务繁忙 |
| 释放许可 | 只在成功获取许可后释放 |
| 中断处理 | 捕获后恢复中断状态 |
| 监控 | 观察 `availablePermits()` 和 `getQueueLength()` |
| 高吞吐 | 默认非公平模式 |
| 排队敏感 | 考虑公平模式 |
| 集群限流 | 使用分布式限流方案 |

### 小结

Semaphore 的核心是许可证。

`acquire()` 获取许可，`release()` 归还许可。许可数量决定同一时刻最多有多少线程可以进入受保护逻辑。

它和锁不同，锁通常只允许一个线程进入，Semaphore 可以允许 N 个线程同时进入。

日常项目里，Semaphore 最适合做单机并发控制：接口并发限制、连接池、导出任务保护、第三方接口保护、虚拟线程下游限流。

真正使用时，关键点不是 API 名字，而是几个工程细节：拿到许可再释放、超时获取、恢复中断状态、避免超额释放、区分并发限制和 QPS 限流。
