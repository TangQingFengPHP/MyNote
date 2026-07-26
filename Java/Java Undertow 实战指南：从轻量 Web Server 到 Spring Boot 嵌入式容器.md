### 简介

Undertow 是一个轻量级 Java Web Server。

它可以直接嵌入到 Java 代码里运行，也可以作为 Servlet 容器使用，还曾经是 WildFly 的默认 Web Server。

简单理解：

```text
Undertow = 轻量 HTTP Server + Servlet 容器 + WebSocket 支持
```

它和 Tomcat、Jetty 一样，都可以承载 Java Web 应用。

但 Undertow 的风格更偏底层、更偏嵌入式：一个 HTTP 服务可以由几个 Handler 直接拼出来，不一定要完整走传统 Servlet 应用结构。

常见使用方式：

| 用法 | 说明 |
| --- | --- |
| Undertow Core | 直接用 `HttpHandler` 写轻量 HTTP 服务 |
| Undertow Servlet | 用 Undertow 承载 Servlet / Filter / Listener |
| Spring Boot 3 + Undertow | 替换默认 Tomcat，使用 Undertow 作为嵌入式容器 |
| WildFly | Undertow 作为应用服务器里的 Web 子系统 |

一句话概括：

```text
Undertow 适合轻量嵌入式 HTTP 服务、Servlet 应用容器、WebSocket、网关类组件和对启动速度、资源占用比较敏感的场景。
```

### Undertow 解决什么问题

一个 Java Web 应用通常需要一个 HTTP 容器来做这些事：

```text
监听端口
接收 HTTP 请求
解析请求头和请求体
路由到处理逻辑
执行 Servlet / Handler
返回响应
管理连接和线程
```

Tomcat 是最常见的 Servlet 容器。

Undertow 也能做这件事，但它提供了更轻量的 Handler API。

比如一个最小 HTTP 服务可以直接写成：

```java
Undertow server = Undertow.builder()
        .addHttpListener(8080, "0.0.0.0")
        .setHandler(exchange -> exchange.getResponseSender().send("Hello Undertow"))
        .build();

server.start();
```

这类写法适合：

* 写一个内嵌健康检查端口
* 写轻量本地管理接口
* 写简单 HTTP 网关
* 写性能测试工具
* 写不需要完整 Spring MVC 的小服务

如果是标准业务 REST API，Spring MVC + Tomcat 仍然是默认选择。

### Undertow、Tomcat、Jetty 的区别

这几个都可以作为 Java Web 容器。

| 对比项 | Tomcat | Jetty | Undertow |
| --- | --- | --- | --- |
| 常见程度 | 很高 | 较高 | 相对少一些 |
| Spring Boot 默认 | 是 | 否 | Spring Boot 3 支持，Spring Boot 4 移除 |
| Servlet 支持 | 强 | 强 | 支持 |
| 嵌入式使用 | 支持 | 支持 | 很轻量 |
| Handler API | 不是主要特点 | 支持 Handler 思路 | 核心特色之一 |
| 常见场景 | 普通 Web 应用 | 嵌入式服务、长连接 | 轻量服务、WildFly、网关组件 |

简单理解：

```text
Tomcat：Java Web 默认选项，生态最常见
Jetty：轻量、嵌入式场景常见
Undertow：轻量、可组合 Handler、偏底层控制
```

新项目选容器时，默认 Tomcat 通常最稳。

已经有 Undertow 经验，或项目明确需要 Undertow 的 Handler 模型、资源占用特征、嵌入式能力时，再考虑 Undertow。

### 版本和 Spring Boot 支持情况

Undertow 当前常见版本线有 2.3.x、2.4.x。

Spring Boot 3.x 可以通过 `spring-boot-starter-undertow` 使用 Undertow。

Spring Boot 4.0 已经移除 Undertow 支持，原因是 Spring Boot 4 基于 Servlet 6.1，而 Undertow 在该基线下暂时不兼容。

对照关系可以这样看：

| 场景 | Undertow 使用建议 |
| --- | --- |
| Spring Boot 2.x | 可用 Undertow，但包名多为 `javax.servlet` 体系 |
| Spring Boot 3.x | 可用 `spring-boot-starter-undertow` |
| Spring Boot 4.x | 不再支持 Undertow starter，建议 Tomcat 或 Jetty |
| 独立嵌入式服务 | 可直接使用 Undertow Core |
| WildFly | Undertow 是 Web 子系统 |

如果文章或老项目里看到：

```xml
<artifactId>spring-boot-starter-undertow</artifactId>
```

需要先确认 Spring Boot 大版本。

Spring Boot 3 可以用。

Spring Boot 4 不适合继续按这个方式使用 Undertow。

### Undertow 核心模块

Undertow 常见 Maven 模块有几个。

| 模块 | 作用 |
| --- | --- |
| `undertow-core` | 核心 HTTP Server、Handler、非阻塞能力 |
| `undertow-servlet` | Servlet 支持 |
| `undertow-websockets-jsr` | Java WebSocket API 支持 |

只写轻量 Handler 服务时，通常只需要：

```xml
<dependency>
    <groupId>io.undertow</groupId>
    <artifactId>undertow-core</artifactId>
    <version>${undertow.version}</version>
</dependency>
```

需要 Servlet 时再加：

```xml
<dependency>
    <groupId>io.undertow</groupId>
    <artifactId>undertow-servlet</artifactId>
    <version>${undertow.version}</version>
</dependency>
```

### 第一个 Undertow Core Demo

先写一个最小 HTTP 服务。

Maven 依赖：

```xml
<properties>
    <undertow.version>2.4.2.Final</undertow.version>
</properties>

<dependency>
    <groupId>io.undertow</groupId>
    <artifactId>undertow-core</artifactId>
    <version>${undertow.version}</version>
</dependency>
```

启动类：

```java
package com.example.undertow.core;

import io.undertow.Undertow;
import io.undertow.util.Headers;

public class HelloUndertowServer {

    public static void main(String[] args) {
        Undertow server = Undertow.builder()
                .addHttpListener(8080, "0.0.0.0")
                .setHandler(exchange -> {
                    exchange.getResponseHeaders().put(Headers.CONTENT_TYPE, "text/plain;charset=UTF-8");
                    exchange.getResponseSender().send("Hello Undertow");
                })
                .build();

        server.start();
        System.out.println("Undertow started at http://localhost:8080");
    }
}
```

访问：

```text
GET http://localhost:8080
```

响应：

```text
Hello Undertow
```

这段代码没有 Spring、没有 Servlet、没有 Controller。

请求进来后直接交给 `HttpHandler` 处理。

### HttpHandler 和 HttpServerExchange

Undertow Core 的核心接口是 `HttpHandler`。

```java
public interface HttpHandler {
    void handleRequest(HttpServerExchange exchange) throws Exception;
}
```

`HttpServerExchange` 表示一次 HTTP 请求和响应。

常用能力：

| 操作 | 示例 |
| --- | --- |
| 获取请求方法 | `exchange.getRequestMethod()` |
| 获取请求路径 | `exchange.getRequestPath()` |
| 获取查询参数 | `exchange.getQueryParameters()` |
| 设置响应头 | `exchange.getResponseHeaders().put(...)` |
| 设置状态码 | `exchange.setStatusCode(404)` |
| 写响应 | `exchange.getResponseSender().send(...)` |

示例：

```java
package com.example.undertow.core;

import io.undertow.server.HttpHandler;
import io.undertow.server.HttpServerExchange;
import io.undertow.util.Headers;
import io.undertow.util.StatusCodes;

public class UserHandler implements HttpHandler {

    @Override
    public void handleRequest(HttpServerExchange exchange) {
        String path = exchange.getRequestPath();

        if ("/users".equals(path)) {
            exchange.getResponseHeaders().put(Headers.CONTENT_TYPE, "application/json;charset=UTF-8");
            exchange.getResponseSender().send("""
                    [{"id":1,"name":"Tom"},{"id":2,"name":"Jerry"}]
                    """);
            return;
        }

        exchange.setStatusCode(StatusCodes.NOT_FOUND);
        exchange.getResponseSender().send("Not Found");
    }
}
```

启动：

```java
Undertow server = Undertow.builder()
        .addHttpListener(8080, "0.0.0.0")
        .setHandler(new UserHandler())
        .build();

server.start();
```

这种写法适合非常轻量的 HTTP 服务。

接口多起来以后，需要引入路由 Handler 或上层 Web 框架。

### PathHandler：按路径路由

Undertow 内置了一些 Handler。

`PathHandler` 可以按路径分发请求。

```java
package com.example.undertow.core;

import io.undertow.Handlers;
import io.undertow.Undertow;
import io.undertow.server.handlers.PathHandler;
import io.undertow.util.Headers;

public class PathHandlerServer {

    public static void main(String[] args) {
        PathHandler pathHandler = Handlers.path()
                .addExactPath("/health", exchange -> {
                    exchange.getResponseHeaders().put(Headers.CONTENT_TYPE, "application/json;charset=UTF-8");
                    exchange.getResponseSender().send("""
                            {"status":"UP"}
                            """);
                })
                .addPrefixPath("/api/users", exchange -> {
                    exchange.getResponseHeaders().put(Headers.CONTENT_TYPE, "application/json;charset=UTF-8");
                    exchange.getResponseSender().send("""
                            [{"id":1,"name":"Tom"}]
                            """);
                });

        Undertow server = Undertow.builder()
                .addHttpListener(8080, "0.0.0.0")
                .setHandler(pathHandler)
                .build();

        server.start();
    }
}
```

访问：

```text
GET /health
GET /api/users
```

`PathHandler` 适合简单路由。

如果要做完整 REST API，Spring MVC、Jersey、JAX-RS 或其他框架会更省心。

### Handler 链

Undertow 没有 Netty 那种 Pipeline 概念，但 Handler 可以手动链式组合。

比如加一个请求日志 Handler。

```java
package com.example.undertow.core;

import io.undertow.server.HttpHandler;
import io.undertow.server.HttpServerExchange;

public class AccessLogHandler implements HttpHandler {

    private final HttpHandler next;

    public AccessLogHandler(HttpHandler next) {
        this.next = next;
    }

    @Override
    public void handleRequest(HttpServerExchange exchange) throws Exception {
        long start = System.currentTimeMillis();
        try {
            next.handleRequest(exchange);
        } finally {
            long cost = System.currentTimeMillis() - start;
            System.out.printf(
                    "%s %s cost=%dms%n",
                    exchange.getRequestMethod(),
                    exchange.getRequestPath(),
                    cost
            );
        }
    }
}
```

组合：

```java
HttpHandler businessHandler = exchange -> exchange.getResponseSender().send("ok");
HttpHandler rootHandler = new AccessLogHandler(businessHandler);

Undertow server = Undertow.builder()
        .addHttpListener(8080, "0.0.0.0")
        .setHandler(rootHandler)
        .build();
```

这种方式类似“责任链”。

日志、限流、鉴权、压缩、静态资源，都可以通过 Handler 组合出来。

### 阻塞和非阻塞

Undertow 支持非阻塞和阻塞任务。

一个重要原则：

```text
I/O 线程不适合执行阻塞操作。
```

比如数据库查询、远程 HTTP 调用、文件读取，都可能阻塞当前线程。

Undertow 可以通过 `exchange.dispatch(...)` 把任务派发到 worker 线程池。

```java
package com.example.undertow.core;

import io.undertow.server.HttpHandler;
import io.undertow.server.HttpServerExchange;

public class BlockingWorkHandler implements HttpHandler {

    @Override
    public void handleRequest(HttpServerExchange exchange) {
        if (exchange.isInIoThread()) {
            exchange.dispatch(this);
            return;
        }

        String result = queryDatabase();
        exchange.getResponseSender().send(result);
    }

    private String queryDatabase() {
        return "db result";
    }
}
```

这里先判断是否处在 I/O 线程。

如果是，就 `dispatch` 到 worker 线程，再执行阻塞逻辑。

这点和 Netty 的 EventLoop 思路很像：I/O 线程要尽量保持轻快。

### 读取请求体

读取 JSON 请求体时，可以先 `startBlocking()`，再从输入流读取。

```java
package com.example.undertow.core;

import io.undertow.server.HttpHandler;
import io.undertow.server.HttpServerExchange;
import io.undertow.util.Headers;

import java.nio.charset.StandardCharsets;

public class JsonBodyHandler implements HttpHandler {

    @Override
    public void handleRequest(HttpServerExchange exchange) throws Exception {
        if (exchange.isInIoThread()) {
            exchange.dispatch(this);
            return;
        }

        exchange.startBlocking();
        byte[] bytes = exchange.getInputStream().readAllBytes();
        String body = new String(bytes, StandardCharsets.UTF_8);

        exchange.getResponseHeaders().put(Headers.CONTENT_TYPE, "application/json;charset=UTF-8");
        exchange.getResponseSender().send("""
                {"received":%s}
                """.formatted(toJsonString(body)));
    }

    private String toJsonString(String value) {
        return "\"" + value.replace("\\", "\\\\").replace("\"", "\\\"") + "\"";
    }
}
```

这只是演示请求体读取。

业务项目里通常会用 Jackson、JSON-B 或框架自带的 JSON 绑定能力。

### Undertow Servlet Demo

Undertow 不只是 Handler Server，也可以承载 Servlet。

依赖：

```xml
<dependency>
    <groupId>io.undertow</groupId>
    <artifactId>undertow-servlet</artifactId>
    <version>${undertow.version}</version>
</dependency>
```

Servlet：

```java
package com.example.undertow.servlet;

import jakarta.servlet.http.HttpServlet;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;

import java.io.IOException;

public class HelloServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        resp.setContentType("text/plain;charset=UTF-8");
        resp.getWriter().write("Hello Servlet on Undertow");
    }
}
```

启动：

```java
package com.example.undertow.servlet;

import io.undertow.Undertow;

import static io.undertow.servlet.Servlets.defaultContainer;
import static io.undertow.servlet.Servlets.deployment;
import static io.undertow.servlet.Servlets.servlet;

public class UndertowServletServer {

    public static void main(String[] args) throws Exception {
        var servletInfo = servlet("helloServlet", HelloServlet.class)
                .addMapping("/hello");

        var deploymentInfo = deployment()
                .setClassLoader(UndertowServletServer.class.getClassLoader())
                .setContextPath("/")
                .setDeploymentName("undertow-servlet-demo")
                .addServlet(servletInfo);

        var manager = defaultContainer().addDeployment(deploymentInfo);
        manager.deploy();

        Undertow server = Undertow.builder()
                .addHttpListener(8080, "0.0.0.0")
                .setHandler(manager.start())
                .build();

        server.start();
    }
}
```

访问：

```text
GET http://localhost:8080/hello
```

这说明 Undertow 可以跑标准 Servlet 应用。

Spring MVC 本质上也可以运行在 Servlet 容器上。

### Spring Boot 3 替换 Tomcat 为 Undertow

Spring Boot Web 默认使用 Tomcat。

在 Spring Boot 3 中，可以排除 Tomcat，引入 Undertow starter。

Maven：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

Gradle Kotlin DSL：

```kotlin
dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web") {
        exclude(group = "org.springframework.boot", module = "spring-boot-starter-tomcat")
    }
    implementation("org.springframework.boot:spring-boot-starter-undertow")
}
```

启动日志里一般会看到 Undertow 相关信息。

Controller 不需要改：

```java
package com.example.undertow.boot;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
public class HealthController {

    @GetMapping("/health")
    public String health() {
        return "UP";
    }
}
```

这就是嵌入式容器替换的好处：

```text
Controller、Service、Repository 基本不变
底层 HTTP 容器从 Tomcat 换成 Undertow
```

### Spring Boot 3 Undertow 配置

Spring Boot 3 可以通过 `server.undertow.*` 配置 Undertow。

```yaml
server:
  port: 8080
  undertow:
    io-threads: 4
    worker-threads: 32
    buffer-size: 16384
    direct-buffers: true
```

常见配置：

| 配置 | 说明 |
| --- | --- |
| `server.undertow.io-threads` | I/O 线程数 |
| `server.undertow.worker-threads` | Worker 线程数 |
| `server.undertow.buffer-size` | Buffer 大小 |
| `server.undertow.direct-buffers` | 是否使用直接内存 Buffer |

配置不是越大越好。

I/O 线程负责网络事件，worker 线程处理阻塞任务和 Servlet 调用。

如果接口主要卡在数据库或外部 HTTP 调用，单纯调大 Undertow 线程数不一定能提升吞吐，还可能增加上下文切换。

### 编程式定制 Undertow

Spring Boot 3 里可以用 `UndertowServletWebServerFactory` 定制 Undertow。

```java
package com.example.undertow.boot;

import io.undertow.UndertowOptions;
import org.springframework.boot.web.embedded.undertow.UndertowServletWebServerFactory;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class UndertowConfig {

    @Bean
    public WebServerFactoryCustomizer<UndertowServletWebServerFactory> undertowCustomizer() {
        return factory -> factory.addBuilderCustomizers(builder -> {
            builder.setServerOption(UndertowOptions.MAX_HEADER_SIZE, 64 * 1024);
            builder.setServerOption(UndertowOptions.MAX_PARAMETERS, 2000);
        });
    }
}
```

常见定制项：

| 选项 | 说明 |
| --- | --- |
| `MAX_HEADER_SIZE` | 请求头最大大小 |
| `MAX_ENTITY_SIZE` | 请求体最大大小 |
| `MAX_PARAMETERS` | 查询参数最大数量 |
| `MAX_HEADERS` | 请求头最大数量 |
| `ALLOW_ENCODED_SLASH` | 是否允许编码后的斜杠 |

这些配置常用于安全限制和兼容老系统。

### Access Log

Spring Boot 3 可以开启 Undertow access log。

```yaml
server:
  undertow:
    accesslog:
      enabled: true
      dir: logs
      pattern: common
      prefix: access_log
      suffix: log
```

常见用途：

```text
记录请求路径
记录状态码
记录响应时间
排查慢请求
分析入口流量
```

应用日志和 access log 不一样。

应用日志记录业务过程。

access log 记录 HTTP 访问情况。

### WebSocket 支持

Undertow 支持 WebSocket。

如果直接使用 Undertow 模块，可以加入：

```xml
<dependency>
    <groupId>io.undertow</groupId>
    <artifactId>undertow-websockets-jsr</artifactId>
    <version>${undertow.version}</version>
</dependency>
```

在 Spring Boot 项目里，如果使用 Spring WebSocket，通常更关注 Spring 的 WebSocket 配置，底层容器负责承载连接。

长连接场景要关注：

| 点 | 说明 |
| --- | --- |
| 连接数 | 是否会大量长连接 |
| 心跳 | 客户端和服务端是否有心跳 |
| 超时 | 空闲连接是否清理 |
| 内存 | 每个连接的缓冲和上下文 |
| 反向代理 | Nginx / LB 是否支持升级协议 |

### Undertow 和 Netty 的区别

Undertow 和 Netty 都偏高性能网络方向，但定位不同。

| 对比项 | Undertow | Netty |
| --- | --- | --- |
| 主要定位 | HTTP Server / Servlet 容器 | 通用网络通信框架 |
| 常见协议 | HTTP、Servlet、WebSocket | TCP、UDP、HTTP、WebSocket、自定义协议 |
| 编程模型 | `HttpHandler` / Servlet | `ChannelPipeline` / `ChannelHandler` |
| Spring Boot Servlet Web | Spring Boot 3 支持 | 不作为 Servlet 容器使用 |
| Spring WebFlux 默认 | 否 | Reactor Netty 默认 |
| 自定义 TCP 协议 | 不适合 | 适合 |

简单理解：

```text
Undertow 更像轻量 HTTP / Servlet Server
Netty 更像通用网络通信底座
```

写 HTTP 服务、Servlet 容器，可以考虑 Undertow。

写自定义 TCP、RPC、IM、游戏协议，Netty 更合适。

### 常见问题

#### Spring Boot 4 为什么找不到 spring-boot-starter-undertow

Spring Boot 4 已经移除 Undertow starter。

如果项目升级到 Spring Boot 4，需要改用 Tomcat 或 Jetty。

Tomcat 默认依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

Jetty 示例：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

#### I/O 线程和 worker 线程怎么理解

I/O 线程负责网络事件。

worker 线程负责阻塞任务、Servlet 请求等工作。

阻塞逻辑不适合长时间运行在 I/O 线程上。

Undertow Core 里可以用 `exchange.dispatch(...)` 派发到 worker 线程。

#### Undertow 是否一定比 Tomcat 快

不一定。

性能和接口类型、业务耗时、数据库、JVM 参数、线程配置、负载模型都有关。

如果接口大部分时间花在数据库和外部 HTTP 调用上，换容器带来的提升可能很小。

容器选型需要压测验证，而不是只看框架名。

#### Spring MVC 代码切换 Undertow 的改动范围

Spring Boot 3 中，只是把嵌入式容器从 Tomcat 换成 Undertow，Controller 通常不需要改。

需要关注：

| 关注点 | 说明 |
| --- | --- |
| 依赖 | 排除 Tomcat，引入 Undertow |
| 配置 | `server.tomcat.*` 改成 `server.undertow.*` |
| Servlet 兼容性 | Filter、Listener、Servlet 是否正常 |
| Access Log | 配置项和日志格式不同 |
| 压测 | 线程和 buffer 配置要实测 |

### 实践建议

| 场景 | 建议 |
| --- | --- |
| 普通 Spring Boot 业务系统 | 默认 Tomcat 更稳 |
| Spring Boot 3 想尝试 Undertow | 排除 Tomcat，引入 Undertow starter |
| Spring Boot 4 项目 | 使用 Tomcat 或 Jetty |
| 轻量 HTTP 服务 | 可以直接用 Undertow Core |
| 阻塞业务逻辑 | 使用 `exchange.dispatch(...)` |
| 请求体较大 | 配置最大请求体和 buffer |
| 安全限制 | 配置 header、参数数量、实体大小 |
| 线上切换容器 | 做压测和灰度 |

### 小结

Undertow 是一个轻量、高性能、可嵌入的 Java Web Server。

它既可以用 `HttpHandler` 直接写 HTTP 服务，也可以通过 `undertow-servlet` 承载 Servlet 应用，还可以在 Spring Boot 3 中替换默认 Tomcat。

使用 Undertow 时，核心要理解 Handler、`HttpServerExchange`、I/O 线程、worker 线程、阻塞任务派发、Servlet 支持和 Spring Boot 版本边界。

如果项目是 Spring Boot 4，Undertow starter 已经不再可用，需要选择 Tomcat 或 Jetty。如果项目是 Spring Boot 3，Undertow 仍然可以作为嵌入式 Servlet 容器使用。
