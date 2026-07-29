### 简介

`RxJava` 是 JVM 上的响应式编程库。

它把一次请求、一个按钮事件、一批订单、一个定时任务，都看成一条可以被处理的数据流。

简单理解：

```text
数据源
  |
  v
操作符处理
  |
  v
订阅后执行
```

常见写法长这样：

```java
Observable.just("java", "spring", "rxjava")
        .map(String::toUpperCase)
        .filter(name -> name.length() > 4)
        .subscribe(System.out::println);
```

输出：

```text
SPRING
RXJAVA
```

这里的重点不是语法变短，而是把异步任务、事件处理、数据转换、异常兜底、线程切换放到一条链路里表达。

一句话概括：

```text
RxJava 适合处理异步事件流、任务编排、线程切换和连续数据处理。
```

### RxJava 适合什么场景

RxJava 最常见的价值在于“把一段一段的异步逻辑串起来”。

比如一个订单详情页，可能需要做这些事情：

```text
查订单
查用户
查优惠券
查物流
组装页面结果
```

普通回调写法容易变成多层嵌套。

RxJava 的写法更像流水线：

```text
订单 ID
  |
  v
查订单
  |
  v
并行查用户、物流、优惠券
  |
  v
组合结果
  |
  v
返回页面模型
```

适合的场景：

* Android 事件处理和网络请求编排
* Java 客户端 SDK 的异步 API 封装
* 后台服务里的异步任务组合
* 搜索输入防抖、按钮防连点、事件流过滤
* 上游数据很快、下游处理较慢的背压场景

不太适合的场景：

* 普通同步 CRUD
* 简单查库后直接返回
* 团队主要使用 Spring WebFlux 和 Reactor
* 代码里大量阻塞调用，又没有清晰的线程池隔离

### RxJava 和 Reactor 的关系

`RxJava` 和 `Reactor` 思路很接近，都是响应式流处理库。

Spring WebFlux 默认使用的是 Reactor。

| 对比项 | RxJava | Reactor |
| --- | --- | --- |
| 常见核心类型 | `Observable`、`Flowable`、`Single` | `Mono`、`Flux` |
| 典型生态 | Android、客户端 SDK、异步事件流 | Spring WebFlux、R2DBC、Spring Cloud Gateway |
| 0 到 1 个结果 | `Maybe` / `Single` | `Mono` |
| 0 到 N 个结果 | `Observable` / `Flowable` | `Flux` |
| 背压处理 | `Flowable` 支持 | `Flux` 原生支持 |

可以这样类比：

```text
RxJava Observable / Flowable  ≈  Reactor Flux
RxJava Single / Maybe         ≈  Reactor Mono
RxJava Completable            ≈  Mono<Void>
```

两套 API 不是谁替代谁，更多是生态选择不同。

### 版本和包名

RxJava 经历过几个主要版本，包名有明显区别。

| 版本 | Maven groupId | 常见包名 | 说明 |
| --- | --- | --- | --- |
| RxJava 1 | `io.reactivex` | `rx.*` | 老项目还能看到 |
| RxJava 2 | `io.reactivex.rxjava2` | `io.reactivex.*` | 引入 `Flowable`，区分背压 |
| RxJava 3 | `io.reactivex.rxjava3` | `io.reactivex.rxjava3.*` | 目前 Java 项目更常用 |
| RxJava 4 | `io.reactivex.rxjava4` | `io.reactivex.rxjava4.*` | 新版本规划中，包名继续变化 |

新项目通常直接使用 RxJava 3。

Maven 依赖：

```xml
<properties>
    <rxjava.version>3.1.12</rxjava.version>
</properties>

<dependencies>
    <dependency>
        <groupId>io.reactivex.rxjava3</groupId>
        <artifactId>rxjava</artifactId>
        <version>${rxjava.version}</version>
    </dependency>
</dependencies>
```

### 第一个 Demo：从集合到数据流

先看一个最简单的用户筛选例子。

```java
import io.reactivex.rxjava3.core.Observable;

import java.util.List;

public class RxJavaFirstDemo {

    public static void main(String[] args) {
        List<User> users = List.of(
                new User(1L, "Alice", 19),
                new User(2L, "Bob", 16),
                new User(3L, "Carol", 28)
        );

        Observable.fromIterable(users)
                .filter(user -> user.age() >= 18)
                .map(user -> user.name().toUpperCase())
                .subscribe(
                        name -> System.out.println("成年用户：" + name),
                        error -> System.out.println("处理失败：" + error.getMessage()),
                        () -> System.out.println("处理完成")
                );
    }

    record User(Long id, String name, int age) {
    }
}
```

输出：

```text
成年用户：ALICE
成年用户：CAROL
处理完成
```

这段代码可以拆成三段看：

```text
Observable.fromIterable(users)  创建数据源
filter + map                   处理数据
subscribe                      启动并消费结果
```

`subscribe()` 很关键。多数 RxJava 流在创建时只是描述流程，真正执行发生在订阅之后。

### 核心类型

RxJava 的类型比 Reactor 多一些，主要是为了表达不同数量的结果。

| 类型 | 结果数量 | 是否支持背压 | 常见场景 |
| --- | --- | --- | --- |
| `Observable<T>` | 0 到 N 个 | 不支持 | 短列表、UI 事件、普通事件流 |
| `Flowable<T>` | 0 到 N 个 | 支持 | 大量数据、上下游速度不一致 |
| `Single<T>` | 1 个或异常 | 不需要 | 查询单条数据、远程接口返回 |
| `Maybe<T>` | 0 个或 1 个或异常 | 不需要 | 按 ID 查询，可能不存在 |
| `Completable` | 没有值，只有完成或异常 | 不需要 | 删除、保存、发消息 |

选择时可以按结果数量判断：

```text
一定有一个结果       Single
可能有一个结果       Maybe
只关心完成状态       Completable
可能有多个结果       Observable
多个结果且需要背压   Flowable
```

### Observable：普通数据流

`Observable` 适合处理普通数量的数据。

```java
import io.reactivex.rxjava3.core.Observable;

public class ObservableDemo {

    public static void main(String[] args) {
        Observable.range(1, 5)
                .map(number -> number * number)
                .subscribe(
                        value -> System.out.println("结果：" + value),
                        Throwable::printStackTrace,
                        () -> System.out.println("结束")
                );
    }
}
```

输出：

```text
结果：1
结果：4
结果：9
结果：16
结果：25
结束
```

`Observable.range(1, 5)` 表示从 1 开始发射 5 个数字。

### Single：只返回一个结果

很多业务查询天然就是一个结果，比如按订单 ID 查询订单。

```java
import io.reactivex.rxjava3.core.Single;

public class SingleDemo {

    public static void main(String[] args) {
        findOrderAmount(1001L)
                .map(amount -> amount * 100)
                .subscribe(
                        cents -> System.out.println("订单金额分：" + cents),
                        error -> System.out.println("查询失败：" + error.getMessage())
                );
    }

    static Single<Integer> findOrderAmount(Long orderId) {
        return Single.fromCallable(() -> {
            Thread.sleep(300);
            return 199;
        });
    }
}
```

`Single.fromCallable()` 会在订阅时执行 `Callable`。

如果 `Callable` 抛出异常，异常会进入 `subscribe` 的错误分支。

### Maybe：可能查到，也可能为空

`Maybe` 比较适合按主键查询。

```java
import io.reactivex.rxjava3.core.Maybe;

import java.util.Map;

public class MaybeDemo {

    private static final Map<Long, String> USERS = Map.of(
            1L, "Alice",
            2L, "Bob"
    );

    public static void main(String[] args) {
        findUsername(3L)
                .defaultIfEmpty("匿名用户")
                .subscribe(
                        name -> System.out.println("用户：" + name),
                        error -> System.out.println("查询失败：" + error.getMessage())
                );
    }

    static Maybe<String> findUsername(Long userId) {
        String name = USERS.get(userId);
        return name == null ? Maybe.empty() : Maybe.just(name);
    }
}
```

输出：

```text
用户：匿名用户
```

`Maybe.empty()` 表示正常完成，但没有数据。

### Completable：只关心做完没有

删除缓存、写审计日志、发送通知，很多时候不需要返回值，只关心成功或失败。

```java
import io.reactivex.rxjava3.core.Completable;

public class CompletableDemo {

    public static void main(String[] args) {
        deleteCache("order:1001")
                .andThen(writeAuditLog("删除订单缓存"))
                .subscribe(
                        () -> System.out.println("操作完成"),
                        error -> System.out.println("操作失败：" + error.getMessage())
                );
    }

    static Completable deleteCache(String key) {
        return Completable.fromRunnable(() -> System.out.println("删除缓存：" + key));
    }

    static Completable writeAuditLog(String message) {
        return Completable.fromRunnable(() -> System.out.println("审计日志：" + message));
    }
}
```

`andThen` 表示前一个 `Completable` 成功后，再执行后一个。

### 常用创建方式

| 方法 | 作用 |
| --- | --- |
| `just(value...)` | 发射固定值 |
| `fromIterable(list)` | 从集合发射 |
| `fromArray(array)` | 从数组发射 |
| `range(start, count)` | 发射整数范围 |
| `interval(time, unit)` | 按时间间隔发射 |
| `fromCallable(callable)` | 订阅时执行并返回结果 |
| `fromRunnable(runnable)` | 订阅时执行，无返回值 |
| `create(emitter -> {})` | 自定义发射逻辑 |

`create` 自由度最高，也更容易写错。

示例：

```java
import io.reactivex.rxjava3.core.Observable;

public class CreateDemo {

    public static void main(String[] args) {
        Observable.<String>create(emitter -> {
                    emitter.onNext("订单创建");
                    emitter.onNext("库存扣减");
                    emitter.onNext("消息发送");
                    emitter.onComplete();
                })
                .subscribe(
                        event -> System.out.println("事件：" + event),
                        error -> System.out.println("异常：" + error.getMessage()),
                        () -> System.out.println("流程完成")
                );
    }
}
```

`onComplete()` 或 `onError()` 之后，流已经结束，后续再发数据没有意义。

### map、filter、flatMap

`map` 用于同步转换。

```java
Observable.just("100", "200", "300")
        .map(Integer::parseInt)
        .map(price -> price * 100)
        .subscribe(System.out::println);
```

`filter` 用于过滤。

```java
Observable.range(1, 10)
        .filter(number -> number % 2 == 0)
        .subscribe(System.out::println);
```

`flatMap` 用于把一个值转换成另一个流，常用于异步查询。

```java
import io.reactivex.rxjava3.core.Observable;
import io.reactivex.rxjava3.core.Single;

public class FlatMapDemo {

    public static void main(String[] args) {
        Observable.just(1001L, 1002L, 1003L)
                .flatMapSingle(FlatMapDemo::findOrderTitle)
                .subscribe(System.out::println);
    }

    static Single<String> findOrderTitle(Long orderId) {
        return Single.fromCallable(() -> "订单-" + orderId);
    }
}
```

`flatMap` 可能并发合并结果，输出顺序不一定和输入顺序一致。

如果结果顺序很重要，可以使用 `concatMap`。

```java
Observable.just(1001L, 1002L, 1003L)
        .concatMapSingle(orderId -> findOrderTitle(orderId))
        .subscribe(System.out::println);
```

### zip：组合多个异步结果

`zip` 适合多个异步结果都完成后再组装。

```java
import io.reactivex.rxjava3.core.Single;
import io.reactivex.rxjava3.schedulers.Schedulers;

public class ZipDemo {

    public static void main(String[] args) throws Exception {
        Single<OrderView> view = Single.zip(
                findOrder(1001L),
                findUser(7L),
                findDelivery(1001L),
                (order, user, delivery) -> new OrderView(order.title(), user.name(), delivery.status())
        );

        view.subscribe(
                result -> System.out.println(result),
                error -> System.out.println("查询失败：" + error.getMessage())
        );

        Thread.sleep(1000);
    }

    static Single<Order> findOrder(Long orderId) {
        return Single.fromCallable(() -> {
            Thread.sleep(300);
            return new Order(orderId, "机械键盘");
        }).subscribeOn(Schedulers.io());
    }

    static Single<User> findUser(Long userId) {
        return Single.fromCallable(() -> {
            Thread.sleep(200);
            return new User(userId, "Alice");
        }).subscribeOn(Schedulers.io());
    }

    static Single<Delivery> findDelivery(Long orderId) {
        return Single.fromCallable(() -> {
            Thread.sleep(400);
            return new Delivery(orderId, "运输中");
        }).subscribeOn(Schedulers.io());
    }

    record Order(Long id, String title) {
    }

    record User(Long id, String name) {
    }

    record Delivery(Long orderId, String status) {
    }

    record OrderView(String orderTitle, String username, String deliveryStatus) {
    }
}
```

输出类似：

```text
OrderView[orderTitle=机械键盘, username=Alice, deliveryStatus=运输中]
```

这里三个查询都使用 `Schedulers.io()`，适合模拟远程接口、文件读写、数据库访问这类 I/O 操作。

`Thread.sleep(1000)` 是为了让示例主线程等待异步任务完成。真实服务里通常由框架生命周期托管线程，不会在业务代码里这样等待。

### subscribeOn 和 observeOn

RxJava 的线程切换主要靠两个方法：

| 方法 | 作用 |
| --- | --- |
| `subscribeOn(scheduler)` | 指定上游从哪里开始执行 |
| `observeOn(scheduler)` | 指定后续操作符和订阅者在哪个线程消费 |

示例：

```java
import io.reactivex.rxjava3.core.Single;
import io.reactivex.rxjava3.schedulers.Schedulers;

public class SchedulerDemo {

    public static void main(String[] args) throws Exception {
        Single.fromCallable(() -> {
                    printThread("查询订单");
                    Thread.sleep(300);
                    return "order-1001";
                })
                .subscribeOn(Schedulers.io())
                .map(order -> {
                    printThread("转换订单");
                    return order.toUpperCase();
                })
                .observeOn(Schedulers.computation())
                .map(order -> {
                    printThread("计算展示字段");
                    return "展示：" + order;
                })
                .observeOn(Schedulers.single())
                .subscribe(result -> {
                    printThread("消费结果");
                    System.out.println(result);
                });

        Thread.sleep(1000);
    }

    static void printThread(String action) {
        System.out.println(action + " -> " + Thread.currentThread().getName());
    }
}
```

执行特点：

```text
subscribeOn 控制上游执行线程
observeOn 控制后面链路的消费线程
observeOn 可以多次切换
多个 subscribeOn 通常只有靠近源头的那个起作用
```

常见调度器：

| 调度器 | 适合场景 |
| --- | --- |
| `Schedulers.io()` | 网络、文件、数据库等阻塞 I/O |
| `Schedulers.computation()` | CPU 计算、定时器、数据计算 |
| `Schedulers.single()` | 单线程顺序执行 |
| `Schedulers.trampoline()` | 当前线程排队执行，测试时常见 |
| `Schedulers.from(executor)` | 使用自定义线程池 |

一个常见原则：

```text
阻塞 I/O 放到 io 或自定义线程池
CPU 计算放到 computation
需要严格隔离的业务使用自定义 Executor
```

### 错误处理

RxJava 的错误会沿着链路向下游传递。

常见错误处理方法：

| 方法 | 作用 |
| --- | --- |
| `onErrorReturnItem(value)` | 异常时返回固定值 |
| `onErrorReturn(error -> value)` | 异常时按异常类型返回值 |
| `onErrorResumeNext(source)` | 异常时切换到另一个流 |
| `retry(times)` | 失败后重试指定次数 |
| `timeout(time, unit)` | 超时后发出异常 |
| `doOnError(action)` | 异常经过时记录日志 |
| `doFinally(action)` | 成功、失败、取消都会执行 |

示例：

```java
import io.reactivex.rxjava3.core.Single;

import java.util.concurrent.TimeUnit;

public class ErrorDemo {

    public static void main(String[] args) {
        remotePrice()
                .timeout(1, TimeUnit.SECONDS)
                .retry(2)
                .doOnError(error -> System.out.println("接口异常：" + error.getMessage()))
                .onErrorReturnItem(0)
                .subscribe(price -> System.out.println("最终价格：" + price));
    }

    static Single<Integer> remotePrice() {
        return Single.fromCallable(() -> {
            throw new RuntimeException("远程价格服务不可用");
        });
    }
}
```

输出类似：

```text
接口异常：远程价格服务不可用
最终价格：0
```

`retry(2)` 表示失败后最多再试 2 次。

兜底逻辑适合放在链路靠后的地方，这样前面任何一步出错都能被统一处理。

### debounce：搜索输入防抖

搜索框、按钮点击、事件监听里，经常需要“短时间内只处理最后一次”。

`debounce` 就适合这种场景。

```java
import io.reactivex.rxjava3.core.Observable;

import java.util.concurrent.TimeUnit;

public class DebounceDemo {

    public static void main(String[] args) throws Exception {
        Observable.create(emitter -> {
                    emitter.onNext("j");
                    Thread.sleep(100);
                    emitter.onNext("ja");
                    Thread.sleep(100);
                    emitter.onNext("jav");
                    Thread.sleep(500);
                    emitter.onNext("java");
                    emitter.onComplete();
                })
                .debounce(300, TimeUnit.MILLISECONDS)
                .subscribe(keyword -> System.out.println("搜索：" + keyword));

        Thread.sleep(1000);
    }
}
```

输出：

```text
搜索：jav
搜索：java
```

`j` 和 `ja` 后面很快又来了新输入，所以被过滤掉。

### Flowable 和背压

背压可以理解成：

```text
上游生产太快，下游处理太慢，下游需要一种流量控制方式。
```

`Observable` 不处理背压。

`Flowable` 专门处理背压。

典型场景：

```text
消息消费
日志处理
文件逐行读取
大量数据库记录导出
传感器数据
```

`Flowable.create()` 需要指定背压策略。

| 策略 | 含义 | 适合场景 |
| --- | --- | --- |
| `BUFFER` | 缓存来不及处理的数据 | 数据不能丢，但要关注内存 |
| `DROP` | 下游忙时丢弃新数据 | 实时事件，允许丢 |
| `LATEST` | 只保留最新数据 | 进度、状态、位置 |
| `ERROR` | 来不及处理就报错 | 需要快速发现压力问题 |
| `MISSING` | 不指定策略，由后续操作符处理 | 高级场景 |

示例：

```java
import io.reactivex.rxjava3.core.BackpressureStrategy;
import io.reactivex.rxjava3.core.Flowable;
import io.reactivex.rxjava3.schedulers.Schedulers;

public class FlowableDemo {

    public static void main(String[] args) throws Exception {
        Flowable.create(emitter -> {
                    for (int i = 1; i <= 1000 && !emitter.isCancelled(); i++) {
                        emitter.onNext(i);
                    }
                    emitter.onComplete();
                }, BackpressureStrategy.BUFFER)
                .observeOn(Schedulers.computation(), false, 16)
                .subscribe(number -> {
                    Thread.sleep(20);
                    System.out.println("处理：" + number);
                });

        Thread.sleep(2000);
    }
}
```

这段代码里：

```text
上游快速发出 1000 个数字
下游每个数字处理 20 毫秒
BUFFER 策略会先缓存处理不过来的数据
observeOn 的 16 表示预取缓冲大小
```

如果这类数据量很大，缓存可能带来内存压力。

实时状态类数据可以考虑 `LATEST`。

```java
Flowable.create(emitter -> {
            for (int i = 1; i <= 1000 && !emitter.isCancelled(); i++) {
                emitter.onNext("进度：" + i);
            }
            emitter.onComplete();
        }, BackpressureStrategy.LATEST)
        .observeOn(Schedulers.computation())
        .subscribe(System.out::println);
```

`LATEST` 的意思是下游忙不过来时，中间旧值可以被覆盖，最终尽量处理最新值。

### Disposable：取消订阅和释放资源

`subscribe()` 通常会返回一个 `Disposable`。

它代表当前订阅关系。

```java
import io.reactivex.rxjava3.core.Observable;
import io.reactivex.rxjava3.disposables.Disposable;

import java.util.concurrent.TimeUnit;

public class DisposableDemo {

    public static void main(String[] args) throws Exception {
        Disposable disposable = Observable.interval(300, TimeUnit.MILLISECONDS)
                .subscribe(value -> System.out.println("心跳：" + value));

        Thread.sleep(1000);
        disposable.dispose();

        System.out.println("订阅已取消");
        Thread.sleep(1000);
    }
}
```

输出类似：

```text
心跳：0
心跳：1
心跳：2
订阅已取消
```

多个订阅可以放到 `CompositeDisposable` 里统一释放。

```java
import io.reactivex.rxjava3.disposables.CompositeDisposable;

public class CompositeDisposableDemo {

    private final CompositeDisposable disposables = new CompositeDisposable();

    public void start() {
        disposables.add(loadUsers().subscribe());
        disposables.add(loadOrders().subscribe());
    }

    public void stop() {
        disposables.clear();
    }

    private io.reactivex.rxjava3.core.Completable loadUsers() {
        return io.reactivex.rxjava3.core.Completable.fromRunnable(() -> System.out.println("加载用户"));
    }

    private io.reactivex.rxjava3.core.Completable loadOrders() {
        return io.reactivex.rxjava3.core.Completable.fromRunnable(() -> System.out.println("加载订单"));
    }
}
```

在 Android 页面、桌面客户端窗口、后台轮询任务中，订阅生命周期尤其重要。

### Spring Boot 里怎么用 RxJava

Spring WebFlux 默认使用 Reactor，不直接以 RxJava 作为核心模型。

不过普通 Spring Boot 项目里，RxJava 仍然可以用于内部异步编排。

示例：订单聚合服务。

```java
import io.reactivex.rxjava3.core.Single;
import io.reactivex.rxjava3.schedulers.Schedulers;
import org.springframework.stereotype.Service;

@Service
public class OrderQueryService {

    public Single<OrderDetailView> findDetail(Long orderId) {
        Single<Order> orderSingle = findOrder(orderId);
        Single<User> userSingle = findUser(7L);
        Single<Coupon> couponSingle = findCoupon(orderId);

        return Single.zip(
                orderSingle,
                userSingle,
                couponSingle,
                (order, user, coupon) -> new OrderDetailView(
                        order.id(),
                        order.title(),
                        user.name(),
                        coupon.name()
                )
        );
    }

    private Single<Order> findOrder(Long orderId) {
        return Single.fromCallable(() -> new Order(orderId, "显示器"))
                .subscribeOn(Schedulers.io());
    }

    private Single<User> findUser(Long userId) {
        return Single.fromCallable(() -> new User(userId, "Alice"))
                .subscribeOn(Schedulers.io());
    }

    private Single<Coupon> findCoupon(Long orderId) {
        return Single.fromCallable(() -> new Coupon(orderId, "满减券"))
                .subscribeOn(Schedulers.io());
    }

    public record Order(Long id, String title) {
    }

    public record User(Long id, String name) {
    }

    public record Coupon(Long orderId, String name) {
    }

    public record OrderDetailView(Long orderId, String title, String username, String couponName) {
    }
}
```

如果是 Spring MVC Controller，可以在边界层转成 `CompletableFuture`。

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RestController;

import java.util.concurrent.CompletableFuture;

@RestController
public class OrderController {

    private final OrderQueryService orderQueryService;

    public OrderController(OrderQueryService orderQueryService) {
        this.orderQueryService = orderQueryService;
    }

    @GetMapping("/orders/{orderId}")
    public CompletableFuture<OrderQueryService.OrderDetailView> detail(@PathVariable Long orderId) {
        return orderQueryService.findDetail(orderId)
                .toCompletionStage()
                .toCompletableFuture();
    }
}
```

如果直接调用 `blockingGet()`，当前请求线程会阻塞。

```java
OrderDetailView view = orderQueryService.findDetail(1001L).blockingGet();
```

这种写法适合命令行、测试代码、启动任务，不适合高并发接口里到处使用。

### 测试 RxJava 流

RxJava 提供了 `TestObserver` 和 `TestSubscriber`。

`Observable`、`Single`、`Maybe`、`Completable` 常用 `TestObserver`。

```java
import io.reactivex.rxjava3.core.Observable;
import io.reactivex.rxjava3.observers.TestObserver;
import org.junit.jupiter.api.Test;

public class UserStreamTest {

    @Test
    void shouldFilterAdultUsers() {
        TestObserver<String> observer = Observable.just(
                        new User("Alice", 19),
                        new User("Bob", 16),
                        new User("Carol", 28)
                )
                .filter(user -> user.age() >= 18)
                .map(User::name)
                .test();

        observer.assertValues("Alice", "Carol");
        observer.assertComplete();
        observer.assertNoErrors();
    }

    record User(String name, int age) {
    }
}
```

`Flowable` 测试可以使用 `TestSubscriber`。

```java
import io.reactivex.rxjava3.core.Flowable;
import io.reactivex.rxjava3.subscribers.TestSubscriber;
import org.junit.jupiter.api.Test;

public class FlowableTest {

    @Test
    void shouldEmitNumbers() {
        TestSubscriber<Integer> subscriber = Flowable.range(1, 3).test();

        subscriber.assertValues(1, 2, 3);
        subscriber.assertComplete();
        subscriber.assertNoErrors();
    }
}
```

### 常见使用细节

| 细节 | 说明 |
| --- | --- |
| 创建流不等于执行流 | 大部分流订阅后才执行 |
| `subscribe()` 要处理异常 | 只写成功回调，异常可能进入全局错误处理 |
| `flatMap` 不保证顺序 | 顺序敏感时用 `concatMap` |
| `Observable` 不处理背压 | 大量数据用 `Flowable` |
| `subscribeOn` 关注上游 | 多次调用通常只有第一个明显生效 |
| `observeOn` 关注下游 | 可以多次切换后续处理线程 |
| 阻塞调用需要隔离 | JDBC、文件、HTTP 同步客户端适合放到 I/O 线程池 |
| 长生命周期订阅需要释放 | 页面销毁、任务停止时调用 `dispose()` |

还有一个很常见的问题：

```java
Observable.just("java")
        .map(String::toUpperCase);
```

这段代码没有 `subscribe()`，所以只是声明了一条流，并没有消费结果。

### RxJava 和 Java Stream 的区别

`Java Stream` 和 `RxJava` 都能写链式操作，但定位不同。

| 对比项 | Java Stream | RxJava |
| --- | --- | --- |
| 主要处理 | 集合数据 | 异步事件流、数据流 |
| 数据来源 | 已有集合、数组、I/O Stream | 事件、异步任务、定时器、集合、回调 |
| 异步能力 | 不是核心能力 | 核心能力 |
| 线程切换 | 主要靠外部线程池或 parallel stream | `Scheduler` |
| 错误处理 | try-catch 或运行时异常 | `onError` 信号 |
| 背压 | 不涉及 | `Flowable` 支持 |

简单说：

```text
Java Stream 更像集合处理工具
RxJava 更像异步事件和任务编排工具
```

### 小结

`RxJava` 的核心是把数据和事件都放进一条流里处理。

常用选择可以记成：

```text
一个结果用 Single
可能为空用 Maybe
只关心完成用 Completable
多个普通结果用 Observable
大量结果和背压用 Flowable
```

实际开发里，最常用的是三块：

* 用 `map`、`filter`、`flatMap`、`zip` 组织业务流程
* 用 `subscribeOn`、`observeOn` 管理线程切换
* 用 `Flowable`、`Disposable` 处理压力和生命周期

如果项目已经是 Spring WebFlux，Reactor 通常更自然。

如果项目里有 Android、客户端 SDK、事件流、异步任务编排，RxJava 仍然是一套很成熟的工具。
