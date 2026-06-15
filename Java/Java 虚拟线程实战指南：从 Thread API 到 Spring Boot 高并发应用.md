### 简介

虚拟线程的英文名是 `Virtual Thread`，它是 Project Loom 带来的轻量级线程实现。

虚拟线程在 JDK 19、JDK 20 中经历了两轮预览，到了 JDK 21 正式发布。

简单理解：

```text
平台线程：Java 线程长期绑定操作系统线程
虚拟线程：大量 Java 线程由 JVM 调度到少量操作系统线程上
```

传统 Java 服务经常采用“一请求一线程”的处理方式。

代码很直观，但平台线程数量有限。当大量请求都在等待数据库、HTTP 接口、文件或消息队列时，线程本身会先成为瓶颈。

虚拟线程让这种同步写法重新具备更高的并发承载能力：

```text
一个任务对应一个虚拟线程
阻塞时暂时释放底层平台线程
I/O 就绪后继续执行原来的代码
```

一句话概括：

```text
虚拟线程让阻塞式 Java 代码可以用更低的线程成本处理大量并发 I/O 任务。
```

### 版本要求

| JDK 版本 | 虚拟线程状态 |
| --- | --- |
| JDK 19 | 第一次预览 |
| JDK 20 | 第二次预览 |
| JDK 21 | 正式特性 |
| JDK 24+ | `synchronized` 场景下的线程固定问题得到大幅改善 |

最低运行版本：

```text
JDK 21
```

JDK 21 已经可以在生产项目中使用虚拟线程。

如果项目允许使用更新版本，JDK 24 及以上对虚拟线程的体验更完整，主要原因是 JEP 491 改进了 `synchronized` 与虚拟线程的配合方式。

### 平台线程为什么容易成为瓶颈

传统 `Thread` 默认创建的是平台线程。

```java
Thread thread = new Thread(() -> {
    System.out.println("当前线程：" + Thread.currentThread());
});

thread.start();
thread.join();
```

平台线程和操作系统线程大致是 `1:1` 的关系：

```text
Java 平台线程 1 -> OS 线程 1
Java 平台线程 2 -> OS 线程 2
Java 平台线程 3 -> OS 线程 3
```

平台线程需要操作系统参与创建、调度和上下文切换。

因此，平台线程通常会放进线程池重复使用：

```java
ExecutorService executor = Executors.newFixedThreadPool(200);
```

线程池可以降低反复创建线程的成本，但不会增加同时可用的平台线程数量。

假设线程池只有 200 个线程，每个任务都需要等待远程接口 1 秒，那么同一时间最多只有 200 个任务在执行，其余任务只能排队。

### 虚拟线程的工作方式

虚拟线程也是 `java.lang.Thread` 的实例，但不会在整个生命周期里固定占用某个操作系统线程。

大致关系如下：

```text
虚拟线程 1 --\
虚拟线程 2 ---\
虚拟线程 3 ----> 少量平台线程 -> 操作系统线程
虚拟线程 4 ---/
虚拟线程 5 --/
```

负责承载虚拟线程运行的平台线程，也叫载体线程，英文是 `Carrier Thread`。

当虚拟线程执行普通计算时，它会挂载到载体线程上。

当虚拟线程执行可感知的阻塞 I/O 时，JVM 可以暂停虚拟线程，并把载体线程交给其他任务使用。

```text
虚拟线程发起 I/O
  |
  v
虚拟线程暂停并卸载
  |
  v
载体线程执行其他虚拟线程
  |
  v
I/O 完成
  |
  v
虚拟线程重新挂载并继续执行
```

这个过程由 JVM 完成，业务代码仍然可以保持同步写法。

### 虚拟线程提升的是吞吐量

虚拟线程并不是运行速度更快的线程。

例如，一个接口调用本身需要 500 毫秒，换成虚拟线程后，这次调用通常仍然需要接近 500 毫秒。

变化主要体现在并发数量：

```text
单次任务耗时：通常不会明显降低
同时处理的任务数：可以明显增加
系统整体吞吐量：I/O 密集场景下可能提高
```

所以虚拟线程更适合：

* JDBC 数据库访问
* 同步 HTTP 调用
* RPC 调用
* 文件和网络 I/O
* 消息消费
* 大量并发请求

对于长时间占用 CPU 的计算任务，线程数量超过 CPU 核心数后，并不会因为使用虚拟线程而得到更高算力。

### 第一个虚拟线程 Demo

最简单的创建方式是 `Thread.startVirtualThread`。

```java
public class FirstVirtualThreadDemo {

    public static void main(String[] args) throws InterruptedException {
        Thread thread = Thread.startVirtualThread(() -> {
            Thread current = Thread.currentThread();

            System.out.println("线程名称：" + current.getName());
            System.out.println("是否为虚拟线程：" + current.isVirtual());
            System.out.println("线程信息：" + current);
        });

        thread.join();
    }
}
```

输出类似：

```text
线程名称：
是否为虚拟线程：true
线程信息：VirtualThread[#21]/runnable@ForkJoinPool-1-worker-1
```

`join()` 用来等待虚拟线程执行结束。

虚拟线程属于守护线程。如果主线程先结束，JVM 不会仅仅为了等待虚拟线程而继续运行，所以独立 Demo 中通常需要 `join()` 或执行器关闭等待。

### 使用 Thread.ofVirtual

`Thread.ofVirtual()` 可以设置线程名称，并选择立即启动或稍后启动。

立即启动：

```java
Thread thread = Thread.ofVirtual()
        .name("order-query")
        .start(() -> {
            System.out.println(Thread.currentThread());
        });

thread.join();
```

先创建再启动：

```java
Thread thread = Thread.ofVirtual()
        .name("report-task")
        .unstarted(() -> {
            System.out.println("生成报表");
        });

thread.start();
thread.join();
```

### 使用 ThreadFactory

需要统一线程名称时，可以创建虚拟线程工厂。

```java
import java.util.concurrent.ThreadFactory;

public class VirtualThreadFactoryDemo {

    public static void main(String[] args) throws InterruptedException {
        ThreadFactory factory = Thread.ofVirtual()
                .name("worker-", 0)
                .factory();

        Thread first = factory.newThread(() -> printTask("任务 A"));
        Thread second = factory.newThread(() -> printTask("任务 B"));

        first.start();
        second.start();

        first.join();
        second.join();
    }

    private static void printTask(String taskName) {
        System.out.println(taskName + "，线程=" + Thread.currentThread().getName());
    }
}
```

输出类似：

```text
任务 A，线程=worker-0
任务 B，线程=worker-1
```

### 每个任务一个虚拟线程

业务代码里更常用的是：

```java
Executors.newVirtualThreadPerTaskExecutor()
```

它会为每个提交的任务创建一个新的虚拟线程。

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class VirtualThreadExecutorDemo {

    public static void main(String[] args) {
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            for (int i = 1; i <= 5; i++) {
                int taskId = i;

                executor.submit(() -> {
                    try {
                        Thread.sleep(500);
                        System.out.printf(
                                "任务 %d 完成，线程=%s，virtual=%s%n",
                                taskId,
                                Thread.currentThread().getName(),
                                Thread.currentThread().isVirtual()
                        );
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    }
                });
            }
        }
    }
}
```

`ExecutorService` 在这里不是传统意义上的固定大小线程池。

它的含义是：

```text
提交一个任务
  |
  v
创建一个虚拟线程
  |
  v
任务结束
  |
  v
虚拟线程结束
```

虚拟线程创建成本低，不需要像平台线程那样池化复用。

### 一万个阻塞任务 Demo

下面的示例同时执行一万个休眠任务，用来观察虚拟线程处理大量阻塞任务的方式。

```java
import java.time.Duration;
import java.time.Instant;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.stream.IntStream;

public class ManyVirtualThreadsDemo {

    public static void main(String[] args) {
        int taskCount = 10_000;
        Instant start = Instant.now();

        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            IntStream.range(0, taskCount).forEach(taskId ->
                    executor.submit(() -> {
                        try {
                            Thread.sleep(1000);
                        } catch (InterruptedException e) {
                            Thread.currentThread().interrupt();
                        }
                    })
            );
        }

        long millis = Duration.between(start, Instant.now()).toMillis();
        System.out.println("任务数：" + taskCount);
        System.out.println("总耗时：" + millis + " ms");
    }
}
```

这里的任务几乎不消耗 CPU，大部分时间都在等待。

因此，一万个任务可以同时处于等待状态，而不需要创建一万个操作系统线程。

这段代码适合观察机制，不适合作为严谨性能基准。正式压测还需要考虑 JVM 预热、连接池大小、下游容量、内存和监控开销。

### Callable 和 Future

虚拟线程和普通 `ExecutorService` 一样支持 `Callable`、`Future`。

```java
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class VirtualThreadFutureDemo {

    public static void main(String[] args) throws Exception {
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            Future<String> userFuture = executor.submit(() -> {
                Thread.sleep(300);
                return "用户信息";
            });

            Future<String> orderFuture = executor.submit(() -> {
                Thread.sleep(500);
                return "订单信息";
            });

            String result = userFuture.get() + " + " + orderFuture.get();
            System.out.println(result);
        }
    }
}
```

两个任务可以并发等待，代码仍然保持从上往下的同步结构。

### 实战：并发聚合多个 HTTP 接口

下面使用 Java `HttpClient` 的同步 `send` 方法模拟接口聚合。

```java
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.time.Duration;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.Future;

public class HttpAggregationService {

    private final HttpClient httpClient = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(2))
            .build();

    public UserPage loadUserPage(long userId) throws Exception {
        try (ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor()) {
            Future<String> userFuture = executor.submit(() ->
                    get("http://localhost:8081/users/" + userId)
            );

            Future<String> orderFuture = executor.submit(() ->
                    get("http://localhost:8082/orders?userId=" + userId)
            );

            Future<String> accountFuture = executor.submit(() ->
                    get("http://localhost:8083/accounts/" + userId)
            );

            return new UserPage(
                    userFuture.get(),
                    orderFuture.get(),
                    accountFuture.get()
            );
        }
    }

    private String get(String url) throws Exception {
        HttpRequest request = HttpRequest.newBuilder(URI.create(url))
                .timeout(Duration.ofSeconds(3))
                .GET()
                .build();

        return httpClient.send(
                request,
                HttpResponse.BodyHandlers.ofString()
        ).body();
    }

    public record UserPage(
            String user,
            String orders,
            String account
    ) {
    }
}
```

三个远程调用之间没有依赖关系，所以可以分别放进虚拟线程并发执行。

### 异常和中断处理

虚拟线程仍然遵守 Java 线程的中断规则。

捕获 `InterruptedException` 后，常见处理方式是恢复中断标记：

```java
try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    return;
}
```

使用 `Future` 时，可以取消任务：

```java
Future<String> future = executor.submit(() -> loadRemoteData());

if (requestCancelled) {
    future.cancel(true);
}
```

`cancel(true)` 会尝试中断正在运行任务的线程。

业务代码需要正确响应中断，才能及时停止任务。

### 限制下游并发数量

虚拟线程数量可以很多，但数据库、远程服务、连接池和文件句柄仍然是有限资源。

比如某个第三方接口最多允许 20 个并发请求，可以使用 `Semaphore` 限流。

```java
import java.util.concurrent.Semaphore;

public class LimitedRemoteClient {

    private final Semaphore semaphore = new Semaphore(20);

    public String call(String request) throws InterruptedException {
        semaphore.acquire();

        try {
            return callRemoteService(request);
        } finally {
            semaphore.release();
        }
    }

    private String callRemoteService(String request) throws InterruptedException {
        Thread.sleep(200);
        return "result:" + request;
    }
}
```

这里限制的是“同时访问下游的任务数”，不是虚拟线程总数。

简单关系：

```text
虚拟线程负责表达任务
Semaphore 负责限制并发
连接池负责限制数据库连接
```

### 虚拟线程和数据库连接池

开启虚拟线程后，数据库连接池仍然需要保留。

例如：

```text
同时有 5000 个虚拟线程查询数据库
HikariCP 最大连接数是 30
```

同一时间仍然只有 30 个任务能够拿到数据库连接，其余任务会等待连接。

虚拟线程降低的是“等待线程”的成本，不会让数据库凭空增加处理能力。

连接池大小仍然需要结合以下因素设置：

* 数据库最大连接数
* SQL 执行时间
* 数据库 CPU 和磁盘能力
* 应用实例数量
* 事务持有连接的时间

### 虚拟线程和 ThreadLocal

虚拟线程支持 `ThreadLocal`。

请求上下文、事务信息、链路追踪 ID 等常见用法可以继续工作。

```java
private static final ThreadLocal<String> TRACE_ID = new ThreadLocal<>();

public void handle(String traceId) {
    TRACE_ID.set(traceId);

    try {
        process();
    } finally {
        TRACE_ID.remove();
    }
}
```

需要关注的是另一类用法：把昂贵对象缓存到 `ThreadLocal`，希望多个任务反复复用。

虚拟线程通常一个任务使用一次。如果每个虚拟线程都创建一份大型对象，内存消耗会随着并发任务数增长。

因此，适合放入 `ThreadLocal` 的通常是轻量上下文，而不是连接、客户端或大型缓存对象。

### 虚拟线程和 synchronized

这一部分需要按 JDK 版本区分。

#### JDK 21 到 JDK 23

虚拟线程在 `synchronized` 代码块或同步方法内部发生阻塞时，可能被固定到载体线程。

```java
public synchronized String loadData() throws InterruptedException {
    Thread.sleep(1000);
    return "data";
}
```

如果大量虚拟线程长时间以这种方式阻塞，可用载体线程会减少，吞吐量可能下降。

#### JDK 24 及以上

JEP 491 改进了 JVM 的监视器实现。

虚拟线程在 `synchronized` 方法、同步代码块、等待监视器时，也可以释放载体线程。因 `synchronized` 导致的固定问题已基本消除。

因此，在 JDK 24 及以上版本中，不需要仅仅为了虚拟线程而把所有 `synchronized` 改成 `ReentrantLock`。

锁的选择可以重新回到代码语义：

```text
synchronized：语法简单，适合普通互斥
ReentrantLock：适合超时、可中断、公平锁、多个 Condition 等高级需求
```

即使使用较新 JDK，锁范围仍然适合保持精简，长时间持锁执行 I/O 也容易造成业务层面的竞争。

#### 仍可能发生固定的场景

JDK 24 以后仍有少量固定场景，主要和原生代码、Foreign Function 调用及部分 JVM 内部过程有关。

这类问题通常需要结合 JFR 和线程转储定位。

### 虚拟线程和 CPU 密集型任务

下面的任务一直占用 CPU：

```java
public long calculate() {
    long result = 0;

    for (long i = 0; i < 5_000_000_000L; i++) {
        result += i;
    }

    return result;
}
```

这种任务没有多少等待时间。

如果同时创建十万个虚拟线程执行计算，CPU 核心数量不会增加，反而会带来更多调度和竞争。

CPU 密集型任务通常仍然适合使用有界平台线程池：

```java
int processors = Runtime.getRuntime().availableProcessors();
ExecutorService cpuExecutor = Executors.newFixedThreadPool(processors);
```

可以用一句话区分：

```text
大量时间在等待 I/O：考虑虚拟线程
大量时间在执行计算：考虑有界平台线程池
```

### Spring Boot 开启虚拟线程

较新的 Spring Boot 项目可以通过配置开启虚拟线程：

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

Properties 写法：

```properties
spring.threads.virtual.enabled=true
```

虚拟线程要求 Java 21 或更高版本。

开启后，Spring Boot 会在支持的自动配置位置使用虚拟线程，例如任务执行、Spring MVC 异步处理以及 Web 容器请求处理等场景。

如果项目自定义了 `Executor`、`AsyncConfigurer` 或 `applicationTaskExecutor`，需要确认自定义 Bean 是否覆盖了自动配置。

### Spring Boot Web Demo

Maven 依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>

<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
```

配置：

```yaml
spring:
  threads:
    virtual:
      enabled: true

  datasource:
    url: jdbc:mysql://localhost:3306/virtual_thread_demo
    username: root
    password: 123456
    hikari:
      maximum-pool-size: 20

server:
  port: 8080
```

准备数据表：

```sql
CREATE DATABASE virtual_thread_demo DEFAULT CHARACTER SET utf8mb4;

USE virtual_thread_demo;

CREATE TABLE users (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  username VARCHAR(50) NOT NULL,
  email VARCHAR(100) NOT NULL,
  status VARCHAR(20) NOT NULL
);

INSERT INTO users (username, email, status) VALUES
('张三', 'zhangsan@example.com', 'ACTIVE'),
('李四', 'lisi@example.com', 'ACTIVE');
```

实体：

```java
public record User(
        Long id,
        String username,
        String email,
        String status
) {
}
```

Repository：

```java
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Repository;

import java.util.Optional;

@Repository
public class UserRepository {

    private final JdbcTemplate jdbcTemplate;

    public UserRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public Optional<User> findById(Long id) {
        String sql = """
                select id, username, email, status
                from users
                where id = ?
                """;

        return jdbcTemplate.query(sql, (rs, rowNum) -> new User(
                rs.getLong("id"),
                rs.getString("username"),
                rs.getString("email"),
                rs.getString("status")
        ), id).stream().findFirst();
    }
}
```

Service：

```java
import org.springframework.stereotype.Service;

@Service
public class UserService {

    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User findById(Long id) {
        return userRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("用户不存在，id=" + id));
    }
}
```

Controller：

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public Map<String, Object> findById(@PathVariable Long id) {
        Thread thread = Thread.currentThread();
        User user = userService.findById(id);

        return Map.of(
                "thread", thread.toString(),
                "virtual", thread.isVirtual(),
                "user", user
        );
    }
}
```

访问：

```text
GET http://localhost:8080/api/users/1
```

响应示例：

```json
{
  "thread": "VirtualThread[#42,tomcat-handler-0]/runnable",
  "virtual": true,
  "user": {
    "id": 1,
    "username": "张三",
    "email": "zhangsan@example.com",
    "status": "ACTIVE"
  }
}
```

这套写法仍然是同步阻塞风格：

```text
Controller
  |
  v
Service
  |
  v
JdbcTemplate
  |
  v
数据库
```

区别在于每个请求可以由独立虚拟线程处理，等待 JDBC 返回时不会长期占用稀缺的平台线程。

### Spring @Async Demo

开启异步支持：

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;

@Configuration
@EnableAsync
public class AsyncConfig {
}
```

异步任务：

```java
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.util.concurrent.CompletableFuture;

@Service
public class ReportService {

    @Async
    public CompletableFuture<String> generate(Long reportId) {
        try {
            Thread.sleep(1000);

            String result = "报表生成完成，id=" + reportId
                    + "，virtual=" + Thread.currentThread().isVirtual();

            return CompletableFuture.completedFuture(result);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return CompletableFuture.failedFuture(e);
        }
    }
}
```

在没有自定义执行器覆盖自动配置的情况下，启用 `spring.threads.virtual.enabled=true` 后，Spring Boot 自动配置的异步任务执行器会使用虚拟线程。

### 虚拟线程和 WebFlux 的区别

虚拟线程与 WebFlux 都能提高 I/O 密集型服务的并发能力，但编程模型不同。

| 对比项 | 虚拟线程 | Spring WebFlux |
| --- | --- | --- |
| 编程风格 | 同步阻塞 | 响应式非阻塞 |
| 代码结构 | 普通方法、循环、`try/catch` | `Mono`、`Flux`、操作符链 |
| JDBC / JPA | 可以直接使用 | 会阻塞事件循环，需要隔离或使用 R2DBC |
| 调试方式 | 普通线程栈 | 响应式调用链 |
| 常见场景 | 传统 MVC、同步 SDK、JDBC | 流式响应、网关、全链路响应式系统 |

两者不是简单的替代关系。

普通 Spring MVC、JDBC、JPA、同步 HTTP 客户端项目，虚拟线程通常更容易接入。

SSE、背压、持续数据流、响应式数据源等场景，WebFlux 依然有明确价值。

### 线程池参数为什么可能失效

开启 Spring Boot 虚拟线程后，原先用于配置平台线程池大小的参数可能不再生效。

原因是虚拟线程不是在应用自己的固定线程池里反复复用，而是由 JVM 的全局虚拟线程调度器运行。

例如下面这类参数需要重新评估：

```text
核心线程数
最大线程数
线程空闲时间
任务队列容量
```

虚拟线程场景下，更值得关注的是：

* 数据库连接池
* HTTP 连接池
* 下游并发限制
* 请求超时
* 内存使用
* CPU 使用率
* 任务积压和失败率

### 监控和排查

虚拟线程数量可能非常多，传统 `jstack` 平铺展示所有线程并不方便。

可以使用 `jcmd` 生成新的线程转储。

文本格式：

```bash
jcmd <PID> Thread.dump_to_file -format=text thread-dump.txt
```

JSON 格式：

```bash
jcmd <PID> Thread.dump_to_file -format=json thread-dump.json
```

JDK Flight Recorder 也可以记录虚拟线程相关事件。

启动时录制：

```bash
java -XX:StartFlightRecording=filename=virtual-thread.jfr,duration=60s -jar app.jar
```

对于 JDK 21 到 JDK 23，还可以使用下面的参数排查因 `synchronized` 导致的固定：

```bash
java -Djdk.tracePinnedThreads=full -jar app.jar
```

JDK 24 已通过 JEP 491 移除这项诊断参数的实际作用，因为 `synchronized` 不再是主要固定来源。较新版本更适合使用 JFR 观察剩余固定场景。

### 常见使用建议

### 一个任务对应一个虚拟线程

虚拟线程很轻，不需要创建固定大小的虚拟线程池。

推荐入口：

```java
Executors.newVirtualThreadPerTaskExecutor()
```

### 用专门机制限制资源并发

限制外部接口并发时使用 `Semaphore`，限制数据库并发时使用数据库连接池。

虚拟线程负责运行任务，资源组件负责控制容量。

### 为 I/O 设置超时

虚拟线程降低了等待成本，但无限等待仍然会积累任务和内存。

数据库、HTTP、RPC、消息处理都适合设置合理超时。

```java
HttpRequest request = HttpRequest.newBuilder(uri)
        .timeout(Duration.ofSeconds(3))
        .GET()
        .build();
```

### 保留中断状态

```java
catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}
```

中断是取消任务、关闭应用和超时控制的重要协作机制。

### 压测需要覆盖真实下游

只用 `Thread.sleep` 能说明虚拟线程可承载大量等待任务，但不能代表真实业务性能。

正式验证至少要包含：

* 数据库连接池容量
* SQL 和索引性能
* HTTP 连接池配置
* 下游限流策略
* JVM 堆内存
* P95、P99 延迟
* 错误率和超时率

### 常用 API 汇总

| API | 作用 |
| --- | --- |
| `Thread.startVirtualThread(task)` | 创建并立即启动虚拟线程 |
| `Thread.ofVirtual()` | 创建虚拟线程构建器 |
| `Thread.Builder.start(task)` | 创建并启动线程 |
| `Thread.Builder.unstarted(task)` | 创建尚未启动的线程 |
| `Thread.Builder.factory()` | 创建线程工厂 |
| `Thread.isVirtual()` | 判断是否为虚拟线程 |
| `Executors.newVirtualThreadPerTaskExecutor()` | 每个任务创建一个虚拟线程 |
| `Thread.join()` | 等待线程结束 |
| `Thread.interrupt()` | 请求中断线程 |
| `Future.cancel(true)` | 取消任务并尝试中断线程 |
| `Semaphore` | 限制某项资源的并发访问量 |

### 总结

虚拟线程没有改变 Java 代码的基本写法。

普通同步代码仍然可以使用：

```text
方法调用
循环
try/catch
JDBC
同步 HTTP 客户端
ThreadLocal
```

变化发生在线程实现层面：

```text
平台线程数量有限，需要池化复用
虚拟线程成本更低，一个任务可以对应一个线程
阻塞 I/O 时，虚拟线程可以释放底层载体线程
```

虚拟线程适合大量并发、I/O 等待明显的应用。

CPU 密集型任务、数据库容量、连接池大小、下游限流等问题不会因为虚拟线程自动消失。

更准确的使用方式是：用虚拟线程承载大量任务，再用连接池、信号量、超时和监控控制有限资源。
