### 简介

SLF4J 是 Java 里很常见的日志门面。

门面这两个字听起来有点抽象，放到日志里理解就很简单：业务代码只调用 SLF4J 的 API，真正写日志的工作交给 Logback、Log4j2、JDK Logging 这类日志实现。

也就是说，代码里写的是：

```java
log.info("订单创建成功，orderId={}", orderId);
```

至于这条日志最后输出到控制台、文件、Kafka、ELK，还是按天滚动保存，属于日志实现和配置文件要处理的事情。

SLF4J 的价值主要在于解耦。

业务代码不用直接绑定某一个日志框架，依赖库也不用强迫应用使用某一个日志实现。应用最终选择一个日志实现即可，比如 Spring Boot 默认常用的 Logback。

### SLF4J、Logback、Log4j2 到底是什么关系

先把几个名字分清楚。

| 名称 | 角色 | 说明 |
| --- | --- | --- |
| SLF4J | 日志门面 | 提供统一日志 API，本身不负责真正写日志 |
| Logback | 日志实现 | 常见实现之一，Spring Boot 默认日志实现通常就是它 |
| Log4j2 | 日志实现 | 另一套成熟日志实现，性能和异步能力都比较完整 |
| java.util.logging | JDK 自带日志实现 | 不需要额外依赖，但工程里直接使用较少 |
| slf4j-simple | 简单日志实现 | 适合小 demo，不适合复杂生产配置 |

一个典型调用链大概是这样：

```text
业务代码
  -> SLF4J API
      -> Logback / Log4j2 / JUL 等日志实现
          -> 控制台、文件、日志系统
```

SLF4J 只定义怎么调用日志，Logback 这类实现负责日志格式、级别、文件滚动、异步输出等具体行为。

### SLF4J 1.x 和 2.x 的一个重要变化

SLF4J 1.x 时代经常能看到 binding 这个词，比如 `slf4j-log4j12`、`logback-classic`。

SLF4J 2.x 以后，官方更强调 provider 机制。底层基于 Java 的 `ServiceLoader` 查找日志提供者。

实际使用时可以这样理解：

| 版本 | 关键词 | 常见现象 |
| --- | --- | --- |
| SLF4J 1.x | binding | classpath 里找绑定实现 |
| SLF4J 2.x | provider | classpath 里找日志提供者 |

只有 `slf4j-api` 没有日志实现时，程序可以正常启动，但日志不会真正输出，控制台通常会提示没有找到 provider。

所以依赖要分两层：

```text
slf4j-api       日志 API
logback-classic 日志实现，也会带上 Logback core
```

### 普通 Java 项目怎么引入

普通 Maven 项目可以这样写：

```xml
<properties>
    <slf4j.version>2.0.18</slf4j.version>
    <logback.version>1.5.15</logback.version>
</properties>

<dependencies>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>${slf4j.version}</version>
    </dependency>

    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>${logback.version}</version>
    </dependency>
</dependencies>
```

`slf4j-api` 提供 `Logger`、`LoggerFactory` 这些接口。

`logback-classic` 是真正的日志实现。没有它，日志 API 可以调用，但日志没有地方落地。

### Spring Boot 项目还要单独加吗

大多数 Spring Boot Web 项目不需要手动添加 `slf4j-api` 和 `logback-classic`。

例如引入了：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

它会间接带上：

```text
spring-boot-starter-logging
  -> logback-classic
  -> slf4j-api
```

所以 Spring Boot 项目里通常直接写日志代码即可。

如果要从 Logback 切到 Log4j2，一般不是再额外加一个日志实现，而是排除默认的 `spring-boot-starter-logging`，再引入 `spring-boot-starter-log4j2`。

### 最小可运行 Demo

先看一个普通 Java 类：

```java
package com.example.logdemo;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class OrderService {

    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public void createOrder(Long userId, Long productId) {
        log.info("开始创建订单，userId={}, productId={}", userId, productId);

        long orderId = System.currentTimeMillis();

        log.info("订单创建成功，orderId={}", orderId);
    }

    public static void main(String[] args) {
        OrderService orderService = new OrderService();
        orderService.createOrder(1001L, 2002L);
    }
}
```

运行后大概会看到这样的日志：

```text
12:30:15.123 [main] INFO com.example.logdemo.OrderService -- 开始创建订单，userId=1001, productId=2002
12:30:15.126 [main] INFO com.example.logdemo.OrderService -- 订单创建成功，orderId=1780000000000
```

日志里的时间、线程名、类名、级别、消息格式都不是 SLF4J 决定的，而是 Logback 配置决定的。

### 日志级别怎么选

日志级别不是越多越好，重点是能帮助定位问题。

| 级别 | 常见用途 | 示例 |
| --- | --- | --- |
| TRACE | 非常细的流程日志 | RPC 编解码、框架内部细节 |
| DEBUG | 调试信息 | SQL 参数、分支判断、缓存命中情况 |
| INFO | 正常业务节点 | 应用启动、订单创建、任务完成 |
| WARN | 有风险但还能继续 | 重试、降级、配置缺失但有默认值 |
| ERROR | 当前操作失败 | 下单失败、数据库异常、第三方接口异常 |

业务系统最常用的是 `INFO`、`WARN`、`ERROR`。

`DEBUG` 适合开发和排查问题时打开，生产环境默认打开太多 `DEBUG` 日志，容易带来日志量和性能压力。

### 占位符写法

SLF4J 推荐使用 `{}` 占位符。

```java
log.info("用户登录成功，userId={}, ip={}", userId, ip);
```

相比字符串拼接：

```java
log.info("用户登录成功，userId=" + userId + ", ip=" + ip);
```

占位符有两个好处。

第一，代码更清楚。

第二，当对应级别没有开启时，SLF4J 可以少做一些字符串拼接工作。

不过要注意，方法参数本身还是会先执行。

```java
log.debug("商品详情={}", buildLargeProductText(product));
```

即使 `DEBUG` 没开启，`buildLargeProductText(product)` 也已经执行了。

这种成本较高的内容可以加级别判断：

```java
if (log.isDebugEnabled()) {
    log.debug("商品详情={}", buildLargeProductText(product));
}
```

SLF4J 2.x 也可以使用 Fluent API 的延迟参数：

```java
log.atDebug()
        .setMessage("商品详情={}")
        .addArgument(() -> buildLargeProductText(product))
        .log();
```

### 异常日志怎么写

异常日志最重要的是保留堆栈。

推荐写法：

```java
try {
    orderRepository.save(order);
} catch (Exception e) {
    log.error("保存订单失败，orderId={}", order.getId(), e);
    throw e;
}
```

最后一个参数传入异常对象，日志实现会打印完整堆栈。

下面这种写法只能看到异常消息，看不到完整调用链：

```java
log.error("保存订单失败，原因={}", e.getMessage());
```

排查线上问题时，只有一行错误原因通常不够用。堆栈信息能直接指出异常从哪一行抛出、经过了哪些方法。

### SLF4J 2.x Fluent API

SLF4J 2.x 增加了 Fluent API，适合结构化日志和可读性更强的复杂日志。

```java
log.atInfo()
        .setMessage("订单创建成功")
        .addKeyValue("orderId", orderId)
        .addKeyValue("userId", userId)
        .addKeyValue("amount", amount)
        .log();
```

输出效果取决于日志实现和 pattern 配置，可能类似：

```text
订单创建成功 orderId=10001 userId=90001 amount=99.80
```

Fluent API 有一个细节：最后要调用 `log()`。

```java
log.atInfo()
        .setMessage("订单创建成功")
        .addKeyValue("orderId", orderId);
```

上面这段没有 `log()`，日志不会输出。

### Spring Boot 里的日志写法

Spring Boot 项目中可以手动声明 logger：

```java
package com.example.order;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

@Service
public class OrderService {

    private static final Logger log = LoggerFactory.getLogger(OrderService.class);

    public Long create(OrderCreateRequest request) {
        log.info("收到创建订单请求，userId={}, skuId={}, count={}",
                request.userId(), request.skuId(), request.count());

        Long orderId = System.currentTimeMillis();

        log.info("订单创建完成，orderId={}", orderId);
        return orderId;
    }
}
```

如果项目使用 Lombok，也可以用 `@Slf4j`：

```java
package com.example.order;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

@Slf4j
@Service
public class OrderService {

    public Long create(OrderCreateRequest request) {
        log.info("收到创建订单请求，userId={}, skuId={}, count={}",
                request.userId(), request.skuId(), request.count());

        Long orderId = System.currentTimeMillis();

        log.info("订单创建完成，orderId={}", orderId);
        return orderId;
    }
}
```

`@Slf4j` 会在编译期生成一个名为 `log` 的静态日志字段，效果接近手写 `LoggerFactory.getLogger(...)`。

### logback-spring.xml 示例

Spring Boot 项目常见配置文件名是 `logback-spring.xml`，放在 `src/main/resources` 下。

下面是一份控制台加滚动文件的基础配置：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <property name="LOG_PATH" value="logs"/>
    <property name="APP_NAME" value="order-service"/>

    <property name="LOG_PATTERN"
              value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level [%X{traceId:-}] %logger{36} - %msg%n"/>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${LOG_PATH}/${APP_NAME}.log</file>
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
            <charset>UTF-8</charset>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_PATH}/${APP_NAME}.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
            <maxFileSize>100MB</maxFileSize>
            <maxHistory>30</maxHistory>
            <totalSizeCap>10GB</totalSizeCap>
        </rollingPolicy>
    </appender>

    <logger name="com.example" level="INFO"/>
    <logger name="org.springframework" level="WARN"/>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>

</configuration>
```

这份配置做了几件事：

| 配置 | 含义 |
| --- | --- |
| `CONSOLE` | 输出到控制台 |
| `FILE` | 输出到文件 |
| `SizeAndTimeBasedRollingPolicy` | 按时间和大小滚动日志文件 |
| `maxHistory` | 保留最近 30 天 |
| `totalSizeCap` | 总日志大小上限 |
| `%X{traceId:-}` | 从 MDC 中取 `traceId` |
| `logger name="com.example"` | 单独控制业务包日志级别 |
| `root` | 根日志配置，兜底接收其他 logger |

### MDC：把 traceId 放进每一行日志

MDC 可以理解为日志线程上下文。

一次 HTTP 请求进来时，把 `traceId` 放进 MDC。后续同一个线程里打印日志时，Logback 的 pattern 可以自动取到它。

先写一个过滤器：

```java
package com.example.config;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.slf4j.MDC;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.util.UUID;

@Component
public class TraceIdFilter extends OncePerRequestFilter {

    private static final String TRACE_ID = "traceId";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        String traceId = request.getHeader("X-Trace-Id");
        if (traceId == null || traceId.isBlank()) {
            traceId = UUID.randomUUID().toString().replace("-", "");
        }

        try {
            MDC.put(TRACE_ID, traceId);
            response.setHeader("X-Trace-Id", traceId);
            filterChain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

配置文件里加入：

```xml
<property name="LOG_PATTERN"
          value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level [%X{traceId:-}] %logger{36} - %msg%n"/>
```

日志会变成这样：

```text
2026-07-15 10:20:30.100 [http-nio-8080-exec-1] INFO  [8f3a9c1b2d...] c.e.order.OrderService - 订单创建完成，orderId=10001
```

有了 `traceId`，一次请求经过 Controller、Service、Repository、远程调用时，相关日志可以串起来看。

线程池、异步任务、消息消费等场景要额外处理上下文传递。MDC 默认绑定在线程上，任务切到另一个线程后，原线程里的 MDC 不会自动过去。

### 异步任务里的 MDC 处理

下面是一个简单的线程池装饰器示例，提交任务时复制 MDC，任务执行结束后清理 MDC：

```java
package com.example.config;

import org.slf4j.MDC;
import org.springframework.core.task.TaskDecorator;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
public class MdcTaskDecorator implements TaskDecorator {

    @Override
    public Runnable decorate(Runnable runnable) {
        Map<String, String> contextMap = MDC.getCopyOfContextMap();

        return () -> {
            Map<String, String> previous = MDC.getCopyOfContextMap();
            try {
                if (contextMap != null) {
                    MDC.setContextMap(contextMap);
                } else {
                    MDC.clear();
                }
                runnable.run();
            } finally {
                if (previous != null) {
                    MDC.setContextMap(previous);
                } else {
                    MDC.clear();
                }
            }
        };
    }
}
```

线程池配置：

```java
package com.example.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;

import java.util.concurrent.Executor;

@Configuration
public class AsyncConfig {

    @Bean
    public Executor applicationExecutor(MdcTaskDecorator mdcTaskDecorator) {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(8);
        executor.setMaxPoolSize(16);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("app-task-");
        executor.setTaskDecorator(mdcTaskDecorator);
        executor.initialize();
        return executor;
    }
}
```

这样异步任务里打印日志时，仍然能拿到请求入口处放入的 `traceId`。

### 日志桥接包是什么

真实项目里可能同时出现多种日志 API。

有的依赖库使用 Commons Logging，有的使用 java.util.logging，有的还在使用旧 Log4j API。

桥接包的作用是把这些 API 的日志转到 SLF4J，再由统一的日志实现输出。

| 桥接包 | 作用 |
| --- | --- |
| `jcl-over-slf4j` | 把 Commons Logging 转到 SLF4J |
| `jul-to-slf4j` | 把 java.util.logging 转到 SLF4J |
| `log4j-over-slf4j` | 把 Log4j 1.x API 转到 SLF4J |

常见方向是：

```text
其他日志 API
  -> SLF4J
      -> Logback
```

桥接时要避免形成循环。

比如一边把 Log4j 1.x 转到 SLF4J，另一边又把 SLF4J 转回 Log4j 1.x，就会出现日志调用来回转发的问题。

### 使用 Log4j2 作为实现

如果应用想用 Log4j2 作为日志实现，可以使用 Log4j2 提供的 SLF4J 2.x 适配实现。

```xml
<dependencies>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>${slf4j.version}</version>
    </dependency>

    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-slf4j2-impl</artifactId>
        <version>${log4j2.version}</version>
    </dependency>

    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-core</artifactId>
        <version>${log4j2.version}</version>
    </dependency>
</dependencies>
```

Spring Boot 项目更常见的方式是使用：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

同时排除默认的 `spring-boot-starter-logging`。

### 常见告警和处理方式

#### No SLF4J providers were found

常见原因是只有 `slf4j-api`，没有日志实现。

处理方式是添加一个 provider，例如 Logback：

```xml
<dependency>
    <groupId>ch.qos.logback</groupId>
    <artifactId>logback-classic</artifactId>
    <version>${logback.version}</version>
</dependency>
```

#### Class path contains multiple SLF4J providers

常见原因是 classpath 里同时存在多个日志实现。

比如同时引入了：

```text
logback-classic
slf4j-simple
log4j-slf4j2-impl
```

处理方式是保留一个实现，排除其他实现。

可以通过 Maven 查看依赖树：

```bash
mvn dependency:tree
```

如果只想看日志相关依赖：

```bash
mvn dependency:tree -Dincludes=org.slf4j,ch.qos.logback,org.apache.logging.log4j
```

#### SLF4J 版本和实现版本不匹配

SLF4J 2.x 项目里混入老的 1.7 binding，可能导致 provider 无法正常识别。

处理方式是统一日志版本。Spring Boot 项目优先交给 Spring Boot 的依赖管理，不随意手写日志框架版本。

### AsyncAppender：日志异步输出

日志写文件也会消耗时间。流量较高的系统可以考虑异步日志。

Logback 里常见写法是用 `AsyncAppender` 包一层真实 appender：

```xml
<appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
    <queueSize>1024</queueSize>
    <discardingThreshold>0</discardingThreshold>
    <appender-ref ref="FILE"/>
</appender>

<root level="INFO">
    <appender-ref ref="CONSOLE"/>
    <appender-ref ref="ASYNC_FILE"/>
</root>
```

异步日志可以降低业务线程等待日志写入的时间，但也带来队列积压、应用异常退出时日志丢失等问题。

核心业务错误日志、审计日志、支付流水日志这类内容，需要结合业务要求单独设计，不适合只靠普通异步日志兜底。

### 日志里适合放什么

日志不是越详细越好，关键是能复盘现场。

一条有价值的业务日志通常包含：

| 内容 | 示例 |
| --- | --- |
| 业务动作 | 创建订单、支付回调、库存扣减 |
| 关键 ID | `orderId`、`userId`、`requestId` |
| 外部结果 | 第三方状态码、响应耗时 |
| 异常堆栈 | 完整 exception |
| 链路标识 | `traceId` |

敏感信息要谨慎处理。

密码、身份证号、银行卡号、手机号完整值、Token、Cookie 这类内容不适合直接写入日志。确实需要排查时，可以做脱敏。

```java
log.info("发送短信验证码，phone={}", maskPhone(phone));
```

```java
private String maskPhone(String phone) {
    if (phone == null || phone.length() < 7) {
        return "****";
    }
    return phone.substring(0, 3) + "****" + phone.substring(phone.length() - 4);
}
```

### 一个完整的订单接口示例

Controller：

```java
package com.example.order;

import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/orders")
public class OrderController {

    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping
    public OrderCreateResponse create(@RequestBody OrderCreateRequest request) {
        Long orderId = orderService.create(request);
        return new OrderCreateResponse(orderId);
    }
}
```

Service：

```java
package com.example.order;

import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;

@Slf4j
@Service
public class OrderService {

    public Long create(OrderCreateRequest request) {
        log.info("开始创建订单，userId={}, skuId={}, count={}",
                request.userId(), request.skuId(), request.count());

        try {
            BigDecimal amount = calculateAmount(request.skuId(), request.count());
            Long orderId = saveOrder(request, amount);

            log.atInfo()
                    .setMessage("订单创建成功")
                    .addKeyValue("orderId", orderId)
                    .addKeyValue("userId", request.userId())
                    .addKeyValue("amount", amount)
                    .log();

            return orderId;
        } catch (Exception e) {
            log.error("订单创建失败，userId={}, skuId={}", request.userId(), request.skuId(), e);
            throw e;
        }
    }

    private BigDecimal calculateAmount(Long skuId, Integer count) {
        log.debug("计算订单金额，skuId={}, count={}", skuId, count);
        return BigDecimal.valueOf(99L * count);
    }

    private Long saveOrder(OrderCreateRequest request, BigDecimal amount) {
        return System.currentTimeMillis();
    }
}
```

请求对象：

```java
package com.example.order;

public record OrderCreateRequest(
        Long userId,
        Long skuId,
        Integer count
) {
}
```

响应对象：

```java
package com.example.order;

public record OrderCreateResponse(
        Long orderId
) {
}
```

这个例子里包含了几类常见日志：

| 日志 | 作用 |
| --- | --- |
| 创建开始 | 记录入口参数 |
| `DEBUG` 金额计算 | 排查细节时打开 |
| 创建成功 | 记录核心结果 |
| 创建失败 | 记录关键参数和完整异常 |
| MDC traceId | 串起同一次请求日志 |

### 依赖库应该怎么写日志

如果开发的是一个公共 jar，通常只依赖 `slf4j-api`。

```xml
<dependency>
    <groupId>org.slf4j</groupId>
    <artifactId>slf4j-api</artifactId>
    <version>${slf4j.version}</version>
</dependency>
```

公共库不适合强行引入 `logback-classic`、`slf4j-simple` 这类具体实现。

原因很直接：最终应用可能使用 Logback，也可能使用 Log4j2。公共库只提供日志调用，具体实现交给应用选择，会更灵活。

### 日志实践建议

| 场景 | 建议 |
| --- | --- |
| 正常业务节点 | 使用 `INFO` |
| 可恢复异常 | 使用 `WARN` |
| 当前操作失败 | 使用 `ERROR`，带完整异常 |
| 高频循环 | 控制日志量，避免每次循环都打 `INFO` |
| 大对象输出 | 使用 `DEBUG`，必要时加 `isDebugEnabled()` |
| 链路排查 | 使用 MDC 写入 `traceId` |
| 敏感信息 | 脱敏后再输出 |
| 公共 jar | 只依赖 `slf4j-api` |

### 小结

SLF4J 的核心定位是日志门面，业务代码面向统一 API 写日志，真正输出交给 Logback、Log4j2 等实现。

普通应用至少需要 `slf4j-api` 加一个日志实现。Spring Boot 项目通常已经通过 starter 带好默认日志依赖。

日常开发里最常用的能力包括 `{}` 占位符、异常堆栈输出、日志级别控制、`logback-spring.xml` 配置、MDC 链路标识、SLF4J 2.x Fluent API。

日志写得好，线上问题会少很多盲查时间。关键不是打印更多内容，而是在合适的位置记录合适的信息。
