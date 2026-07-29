### 简介

`Reactor Netty` 是 Project Reactor 体系里的响应式网络库。

它基于 Netty，提供了更贴近 Reactor 编程模型的 API。

一句话概括：

```text
Reactor Netty = Netty 的网络能力 + Reactor 的 Mono / Flux 响应式模型。
```

它主要提供这些能力：

```text
HTTP Server
HTTP Client
WebSocket
TCP Server
TCP Client
UDP
连接池
SSL / TLS
HTTP/2
指标监控
链路追踪
```

Spring WebFlux 默认底层服务器就是 Reactor Netty。

Spring Cloud Gateway 底层也大量使用 Reactor Netty。

所以理解 Reactor Netty，不只是为了直接写网络程序，也能帮助理解 WebFlux、WebClient、Gateway 的线程模型、连接池、超时和排障方式。

### Reactor Netty 在技术栈里的位置

典型关系如下：

```text
Spring WebFlux / Spring Cloud Gateway / WebClient
  |
  v
Reactor Netty
  |
  v
Netty
  |
  v
Java NIO / Native Transport
```

Netty 负责底层网络：

* EventLoop
* Channel
* ByteBuf
* Pipeline
* Handler
* 编解码

Reactor Netty 在 Netty 上面封装出响应式 API：

* `Mono`
* `Flux`
* `HttpServer`
* `HttpClient`
* `TcpServer`
* `TcpClient`

这样大多数场景不需要直接写 `ChannelHandler`，也不需要手动管理复杂的 Netty Pipeline。

### Reactor Netty 和 Netty 的区别

| 对比项 | Netty | Reactor Netty |
| --- | --- | --- |
| 定位 | 底层网络通信框架 | 响应式网络库 |
| 编程模型 | Channel、Handler、Pipeline | Mono、Flux、函数式 Handler |
| 抽象层级 | 更底层 | 更上层 |
| HTTP Client | 需要自己组合更多细节 | `HttpClient` 开箱可用 |
| HTTP Server | 需要理解 Netty HTTP 编解码 | `HttpServer.route` 直接写路由 |
| 适合场景 | 自定义协议、RPC、IM、网关底座 | 响应式 HTTP/TCP 服务、WebFlux 底层、WebClient |

简单理解：

```text
Netty 关心“连接和字节怎么流动”。
Reactor Netty 关心“用 Mono / Flux 怎么处理这些网络数据”。
```

### Reactor Netty 和 WebFlux 的关系

Spring WebFlux 是 Web 框架。

Reactor Netty 是网络运行时。

关系类似：

```text
Controller / RouterFunction
  |
  v
Spring WebFlux
  |
  v
Reactor Netty HttpServer
  |
  v
Netty EventLoop
```

写 WebFlux Controller 时，代码里通常看不到 Reactor Netty：

```java
@GetMapping("/hello")
public Mono<String> hello() {
    return Mono.just("hello");
}
```

但请求进入应用、连接接入、HTTP 解析、响应写回，默认都是 Reactor Netty 在处理。

### 适合什么场景

Reactor Netty 适合这些场景：

| 场景 | 说明 |
| --- | --- |
| WebFlux 应用 | 默认底层运行时 |
| Spring Cloud Gateway | 高并发转发、代理、过滤 |
| 响应式 HTTP 客户端 | WebClient 底层常用 Reactor Netty |
| 高并发 I/O 服务 | 大量连接、大量等待远程响应 |
| WebSocket | 实时推送、长连接 |
| TCP 服务 | 简单私有协议、设备接入、内部通信 |
| 代理服务 | HTTP 转发、协议转换 |

不太适合的场景：

* 大量 JDBC / JPA 阻塞调用
* 主要是 CPU 密集型计算
* 团队不熟悉 Reactor 操作符
* 普通后台 CRUD，没有明显并发 I/O 压力

Reactor Netty 不是把所有服务都变快的按钮。

如果请求里大量代码仍然是阻塞式调用，EventLoop 被阻塞后性能反而会变差。

### 版本和依赖

当前 Reactor Netty release 线已经到 `1.3.x`，`reactor-netty-http` 最新 release 为 `1.3.6`。

Reactor Netty 1.3.x 依赖：

```text
Reactor Core 3.x
Netty 4.2.x
Reactive Streams 1.0.4
```

普通 Maven 项目可以直接引入：

```xml
<dependency>
    <groupId>io.projectreactor.netty</groupId>
    <artifactId>reactor-netty-http</artifactId>
    <version>1.3.6</version>
</dependency>
```

如果只写 TCP，也可以引入 core：

```xml
<dependency>
    <groupId>io.projectreactor.netty</groupId>
    <artifactId>reactor-netty-core</artifactId>
    <version>1.3.6</version>
</dependency>
```

更推荐使用 Reactor BOM 管版本：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>io.projectreactor</groupId>
            <artifactId>reactor-bom</artifactId>
            <version>2025.0.6</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>io.projectreactor.netty</groupId>
        <artifactId>reactor-netty-http</artifactId>
    </dependency>
</dependencies>
```

Spring Boot 项目里通常不用手写 Reactor Netty 版本。

引入 WebFlux：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

版本交给 Spring Boot 依赖管理。

### 核心组件

常用组件如下：

| 组件 | 作用 |
| --- | --- |
| `HttpServer` | 创建 HTTP 服务端 |
| `HttpClient` | 创建 HTTP 客户端 |
| `TcpServer` | 创建 TCP 服务端 |
| `TcpClient` | 创建 TCP 客户端 |
| `DisposableServer` | 已启动的服务端，可以关闭、等待关闭 |
| `Connection` | TCP 客户端连接 |
| `ConnectionProvider` | HTTP 客户端连接池 |
| `LoopResources` | EventLoop 资源 |
| `NettyInbound` | 入站数据 |
| `NettyOutbound` | 出站数据 |

常见启动方式：

```java
DisposableServer server = HttpServer.create()
        .port(8080)
        .bindNow();

server.onDispose().block();
```

`bindNow()` 会阻塞等待服务启动完成。

`onDispose().block()` 用来阻塞主线程，避免 main 方法退出。

### 第一个 HTTP Server Demo

先写一个 HTTP 服务。

提供几个接口：

| 方法 | 路径 | 作用 |
| --- | --- | --- |
| `GET` | `/hello` | 返回文本 |
| `GET` | `/users/{id}` | 读取路径参数 |
| `POST` | `/echo` | 原样返回请求体 |
| `GET` | `/events` | SSE 流式返回 |

代码：

```java
package com.example.reactornetty;

import reactor.core.publisher.Flux;
import reactor.core.publisher.Mono;
import reactor.netty.DisposableServer;
import reactor.netty.http.server.HttpServer;

import java.time.Duration;

public class DemoHttpServer {

    public static void main(String[] args) {
        DisposableServer server = HttpServer.create()
                .host("0.0.0.0")
                .port(8080)
                .route(routes -> routes
                        .get("/hello", (request, response) ->
                                response.header("Content-Type", "text/plain;charset=UTF-8")
                                        .sendString(Mono.just("Hello Reactor Netty")))

                        .get("/users/{id}", (request, response) -> {
                            String id = request.param("id");
                            String json = """
                                    {"id":%s,"name":"user-%s"}
                                    """.formatted(id, id);

                            return response.header("Content-Type", "application/json;charset=UTF-8")
                                    .sendString(Mono.just(json));
                        })

                        .post("/echo", (request, response) ->
                                response.header("Content-Type", "application/json;charset=UTF-8")
                                        .send(request.receive().retain()))

                        .get("/events", (request, response) -> {
                            Flux<String> events = Flux.interval(Duration.ofSeconds(1))
                                    .take(5)
                                    .map(i -> "data: event-" + i + "\n\n");

                            return response.header("Content-Type", "text/event-stream;charset=UTF-8")
                                    .sendString(events);
                        })
                )
                .bindNow();

        System.out.println("HTTP server started: http://localhost:8080");
        server.onDispose().block();
    }
}
```

测试：

```shell
curl http://localhost:8080/hello
```

输出：

```text
Hello Reactor Netty
```

路径参数：

```shell
curl http://localhost:8080/users/100
```

输出：

```json
{"id":100,"name":"user-100"}
```

POST echo：

```shell
curl -X POST http://localhost:8080/echo \
  -H 'Content-Type: application/json' \
  -d '{"message":"hello"}'
```

输出：

```json
{"message":"hello"}
```

SSE：

```shell
curl http://localhost:8080/events
```

输出类似：

```text
data: event-0

data: event-1

data: event-2
```

### send 和 sendString 的区别

常见写法有两个：

```java
response.sendString(Mono.just("hello"))
```

适合发送字符串。

```java
response.send(request.receive().retain())
```

适合发送二进制数据或直接转发收到的 `ByteBuf`。

这里的 `retain()` 很重要。

`request.receive()` 返回的是 Netty `ByteBuf` 数据流。`ByteBuf` 有引用计数，直接把入站数据转给出站时，需要保留引用，避免写出前被释放。

如果只是读取成字符串，一般不需要手动 `retain()`：

```java
request.receive()
        .aggregate()
        .asString()
```

### 读取请求体

很多接口需要读 JSON 请求体。

示例：

```java
routes.post("/users", (request, response) ->
        request.receive()
                .aggregate()
                .asString()
                .flatMap(body -> {
                    String result = """
                            {"received":%s}
                            """.formatted(body);

                    return response.header("Content-Type", "application/json;charset=UTF-8")
                            .sendString(Mono.just(result))
                            .then();
                })
)
```

`aggregate()` 会把请求体聚合起来。

适合小 JSON。

如果是大文件上传、大响应转发，不应该随便聚合到内存里，而应该用流式处理。

### HTTP Client Demo

`HttpClient` 可以直接发 HTTP 请求。

GET 示例：

```java
package com.example.reactornetty;

import reactor.netty.http.client.HttpClient;

import java.time.Duration;

public class DemoHttpClient {

    public static void main(String[] args) {
        HttpClient client = HttpClient.create()
                .baseUrl("http://localhost:8080")
                .responseTimeout(Duration.ofSeconds(3));

        String hello = client.get()
                .uri("/hello")
                .responseContent()
                .aggregate()
                .asString()
                .block();

        System.out.println(hello);
    }
}
```

POST JSON 示例：

```java
package com.example.reactornetty;

import reactor.core.publisher.Mono;
import reactor.netty.http.client.HttpClient;

import java.time.Duration;

public class DemoHttpPostClient {

    public static void main(String[] args) {
        HttpClient client = HttpClient.create()
                .baseUrl("http://localhost:8080")
                .responseTimeout(Duration.ofSeconds(3));

        String result = client.post()
                .uri("/echo")
                .send((request, outbound) -> {
                    request.requestHeaders().set("Content-Type", "application/json");
                    return outbound.sendString(Mono.just("{\"name\":\"张三\"}"));
                })
                .responseContent()
                .aggregate()
                .asString()
                .block();

        System.out.println(result);
    }
}
```

正式响应式代码里不建议随便 `.block()`。

这里是在 `main` 方法和 Demo 测试里阻塞等待结果。

如果是在 WebFlux Controller、Gateway Filter、Reactor 流中，应该返回 `Mono` 或 `Flux`，不要在 EventLoop 上阻塞。

### 获取响应状态码

如果需要同时拿状态码和响应体：

```java
Mono<String> result = client.get()
        .uri("/users/100")
        .responseSingle((response, body) -> {
            int status = response.status().code();
            return body.asString()
                    .map(content -> "status=" + status + ", body=" + content);
        });
```

调用：

```java
System.out.println(result.block());
```

### 连接池配置

`HttpClient.create()` 默认会使用共享连接池。

官方文档里，默认 HTTP 客户端使用按远端地址区分的 fixed 连接池；默认最大活跃连接数是 `500`，等待获取连接的队列默认最多 `1000`。

自定义连接池：

```java
package com.example.reactornetty;

import reactor.netty.http.client.HttpClient;
import reactor.netty.resources.ConnectionProvider;

import java.time.Duration;

public class PooledHttpClientFactory {

    public static HttpClient create() {
        ConnectionProvider provider = ConnectionProvider.builder("api-pool")
                .maxConnections(100)
                .pendingAcquireMaxCount(200)
                .pendingAcquireTimeout(Duration.ofSeconds(5))
                .maxIdleTime(Duration.ofSeconds(30))
                .maxLifeTime(Duration.ofMinutes(5))
                .evictInBackground(Duration.ofSeconds(30))
                .metrics(true)
                .build();

        return HttpClient.create(provider)
                .responseTimeout(Duration.ofSeconds(3));
    }
}
```

几个配置说明：

| 配置 | 说明 |
| --- | --- |
| `maxConnections` | 每个远端连接池最大连接数 |
| `pendingAcquireMaxCount` | 连接不够时最多排队多少个请求 |
| `pendingAcquireTimeout` | 排队等连接最多等多久 |
| `maxIdleTime` | 连接空闲多久后可关闭 |
| `maxLifeTime` | 连接最多存活多久 |
| `evictInBackground` | 后台定期清理空闲或过期连接 |
| `metrics(true)` | 开启连接池指标 |

连接池不是越大越好。

需要同时看：

* 下游服务承载能力
* 当前服务并发量
* 请求耗时
* pending 队列长度
* 超时设置
* 机器文件描述符限制

如果下游接口很慢，把连接池调得很大只是把压力更快打到下游。

### 超时配置

Reactor Netty 里常见超时分几类。

| 超时 | 作用 |
| --- | --- |
| 连接超时 | TCP 连接建立最多等多久 |
| 响应超时 | 请求发出后等待响应最多多久 |
| 连接池获取超时 | 等空闲连接最多等多久 |
| 读超时 / 写超时 | Channel 读写空闲控制 |
| SSL 握手超时 | TLS 握手最多等多久 |

客户端示例：

```java
package com.example.reactornetty;

import io.netty.channel.ChannelOption;
import io.netty.handler.timeout.ReadTimeoutHandler;
import io.netty.handler.timeout.WriteTimeoutHandler;
import reactor.netty.http.client.HttpClient;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

public class TimeoutHttpClientFactory {

    public static HttpClient create() {
        return HttpClient.create()
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
                .responseTimeout(Duration.ofSeconds(5))
                .doOnConnected(connection -> connection
                        .addHandlerLast(new ReadTimeoutHandler(5, TimeUnit.SECONDS))
                        .addHandlerLast(new WriteTimeoutHandler(5, TimeUnit.SECONDS)));
    }
}
```

`responseTimeout` 通常比 Reactor 的 `timeout()` 操作符更适合 HTTP 客户端，因为它语义更具体。

`Mono.timeout()` 是整个响应式链路超时，范围更大，容易把连接建立、连接池等待、业务操作混在一起。

### Wiretap 调试日志

`wiretap` 可以打印网络数据，适合排查协议、请求头、响应体。

```java
HttpClient client = HttpClient.create()
        .wiretap(true);
```

生产环境不要长期打开完整 wiretap。

原因很直接：

* 日志量很大
* 影响性能
* 可能打印敏感数据

更适合本地或临时排障。

### WebSocket Demo

服务端：

```java
package com.example.reactornetty;

import reactor.core.publisher.Flux;
import reactor.netty.DisposableServer;
import reactor.netty.http.server.HttpServer;

import java.time.Duration;

public class DemoWebSocketServer {

    public static void main(String[] args) {
        DisposableServer server = HttpServer.create()
                .port(8081)
                .route(routes -> routes.ws("/ws", (inbound, outbound) -> {
                    Flux<String> receive = inbound.receive()
                            .asString()
                            .doOnNext(message -> System.out.println("server receive: " + message))
                            .map(message -> "echo: " + message);

                    Flux<String> heartbeat = Flux.interval(Duration.ofSeconds(5))
                            .map(i -> "heartbeat-" + i);

                    return outbound.sendString(Flux.merge(receive, heartbeat));
                }))
                .bindNow();

        System.out.println("WebSocket server started: ws://localhost:8081/ws");
        server.onDispose().block();
    }
}
```

客户端：

```java
package com.example.reactornetty;

import reactor.core.publisher.Flux;
import reactor.netty.http.client.HttpClient;

import java.time.Duration;

public class DemoWebSocketClient {

    public static void main(String[] args) {
        HttpClient.create()
                .websocket()
                .uri("ws://localhost:8081/ws")
                .handle((inbound, outbound) -> {
                    Flux<String> messages = Flux.just("hello", "reactor", "netty")
                            .delayElements(Duration.ofSeconds(1));

                    return outbound.sendString(messages)
                            .then(inbound.receive()
                                    .asString()
                                    .take(5)
                                    .doOnNext(message -> System.out.println("client receive: " + message))
                                    .then());
                })
                .block();
    }
}
```

WebSocket 是长连接，注意不要在消息处理链路里做阻塞操作。

如果需要调用阻塞服务，要切到专门线程池。

### TCP Echo Demo

TCP 服务端：

```java
package com.example.reactornetty;

import reactor.netty.DisposableServer;
import reactor.netty.tcp.TcpServer;

public class DemoTcpServer {

    public static void main(String[] args) {
        DisposableServer server = TcpServer.create()
                .host("0.0.0.0")
                .port(9000)
                .handle((inbound, outbound) ->
                        outbound.sendString(
                                inbound.receive()
                                        .asString()
                                        .map(message -> "echo: " + message)
                        )
                )
                .bindNow();

        System.out.println("TCP server started: localhost:9000");
        server.onDispose().block();
    }
}
```

TCP 客户端：

```java
package com.example.reactornetty;

import reactor.core.publisher.Mono;
import reactor.netty.Connection;
import reactor.netty.tcp.TcpClient;

public class DemoTcpClient {

    public static void main(String[] args) {
        Connection connection = TcpClient.create()
                .host("localhost")
                .port(9000)
                .connectNow();

        String response = connection.outbound()
                .sendString(Mono.just("hello tcp"))
                .then(connection.inbound()
                        .receive()
                        .asString()
                        .next())
                .block();

        System.out.println(response);
        connection.disposeNow();
    }
}
```

这个示例适合学习。

真实 TCP 协议通常还需要解决：

* 粘包半包
* 编解码
* 心跳
* 连接鉴权
* 断线重连
* 消息 ID
* 超时重试

复杂协议可以继续使用 Netty Pipeline，加自定义 Handler。

### 加 Netty Handler

Reactor Netty 仍然可以访问 Netty Pipeline。

比如给 TCP 连接加读超时：

```java
TcpServer.create()
        .port(9000)
        .doOnConnection(connection ->
                connection.addHandlerLast(new ReadTimeoutHandler(30)))
        .handle((inbound, outbound) ->
                outbound.sendString(inbound.receive().asString()))
        .bindNow()
        .onDispose()
        .block();
```

HTTP Client 里也能加：

```java
HttpClient.create()
        .doOnConnected(connection ->
                connection.addHandlerLast(new ReadTimeoutHandler(5)));
```

这说明 Reactor Netty 不是把 Netty 完全藏起来。

常见场景用 Reactor API。

特殊场景还能回到底层 Netty 能力。

### Spring WebFlux 中的 Reactor Netty

Spring Boot WebFlux 默认使用 Reactor Netty。

依赖：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

配置端口：

```yaml
server:
  port: 8080
```

自定义 Reactor Netty 服务端：

```java
package com.example.webflux.config;

import org.springframework.boot.web.embedded.netty.NettyReactiveWebServerFactory;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.stereotype.Component;

import java.time.Duration;

@Component
public class NettyServerConfig implements WebServerFactoryCustomizer<NettyReactiveWebServerFactory> {

    @Override
    public void customize(NettyReactiveWebServerFactory factory) {
        factory.addServerCustomizers(server ->
                server.idleTimeout(Duration.ofSeconds(20)));
    }
}
```

Spring Boot 4 的包名有调整，`NettyReactiveWebServerFactory` 位于：

```java
org.springframework.boot.reactor.netty.NettyReactiveWebServerFactory
```

Spring Boot 3 常见包名是：

```java
org.springframework.boot.web.embedded.netty.NettyReactiveWebServerFactory
```

按项目实际 Spring Boot 版本选择 import。

### WebClient 使用 Reactor Netty

Spring WebFlux 里的 `WebClient` 默认可以使用 Reactor Netty 作为底层 HTTP 客户端。

自定义连接池和超时：

```java
package com.example.webflux.config;

import io.netty.channel.ChannelOption;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.reactive.ReactorClientHttpConnector;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.netty.http.client.HttpClient;
import reactor.netty.resources.ConnectionProvider;

import java.time.Duration;

@Configuration
public class WebClientConfig {

    @Bean
    public WebClient apiWebClient() {
        ConnectionProvider provider = ConnectionProvider.builder("api-client")
                .maxConnections(100)
                .pendingAcquireTimeout(Duration.ofSeconds(5))
                .maxIdleTime(Duration.ofSeconds(30))
                .build();

        HttpClient httpClient = HttpClient.create(provider)
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 3000)
                .responseTimeout(Duration.ofSeconds(5));

        return WebClient.builder()
                .baseUrl("https://api.example.com")
                .clientConnector(new ReactorClientHttpConnector(httpClient))
                .build();
    }
}
```

使用：

```java
@Service
public class UserApiClient {

    private final WebClient apiWebClient;

    public UserApiClient(WebClient apiWebClient) {
        this.apiWebClient = apiWebClient;
    }

    public Mono<String> findUser(Long id) {
        return apiWebClient.get()
                .uri("/users/{id}", id)
                .retrieve()
                .bodyToMono(String.class);
    }
}
```

这里不要写：

```java
return apiWebClient.get()
        .uri("/users/{id}", id)
        .retrieve()
        .bodyToMono(String.class)
        .block();
```

WebFlux 链路里调用 `.block()` 很容易堵住 EventLoop。

### EventLoop 线程模型

Reactor Netty 底层使用 Netty EventLoop。

大致可以理解成：

```text
少量 EventLoop 线程
  |
  v
处理大量连接的 I/O 事件
  |
  v
通过 Mono / Flux 继续编排异步处理
```

EventLoop 线程适合做：

* 读取请求
* 写响应
* 编解码
* 短小计算
* 异步任务编排

不适合做：

* JDBC 查询
* JPA 懒加载
* Thread.sleep
* 大文件阻塞读写
* 远程同步 HTTP 调用
* 大量 CPU 计算

如果必须调用阻塞代码，要切到专门线程池。

示例：

```java
Mono.fromCallable(() -> blockingUserService.findById(id))
        .subscribeOn(Schedulers.boundedElastic());
```

`boundedElastic` 适合包一层少量阻塞调用。

如果阻塞逻辑是主路径，整体技术选型可能就不该用 WebFlux / Reactor Netty。

### 背压怎么理解

Reactor Netty 支持 Reactive Streams 背压。

可以简单理解为：

```text
消费者处理不过来时，可以向上游表达“慢一点”。
```

在网络场景里，背压不是万能限流器。

它能帮助响应式链路减少无节制推送，但仍然需要配合：

* 连接池限制
* 请求超时
* 网关限流
* 队列长度限制
* 下游熔断
* 业务降级

比如 `HttpClient` 调下游接口时，如果下游慢，连接池和 pending 队列会先成为压力点。

这时要看：

```text
active connections
idle connections
pending connections
pending acquire timeout
response timeout
```

### 指标监控

Reactor Netty 可以开启 Micrometer 指标。

客户端连接池：

```java
ConnectionProvider provider = ConnectionProvider.builder("api-pool")
        .maxConnections(100)
        .metrics(true)
        .build();
```

HttpClient：

```java
HttpClient client = HttpClient.create(provider)
        .metrics(true, uri -> uri.startsWith("/users/") ? "/users/{id}" : uri);
```

注意 URI 标签不要带高基数字段。

不推荐：

```text
/users/1
/users/2
/users/3
```

更推荐归一化成：

```text
/users/{id}
```

否则监控系统里会产生大量时间序列。

常见关注指标：

| 指标 | 含义 |
| --- | --- |
| active connections | 正在使用的连接 |
| idle connections | 空闲连接 |
| pending connections | 等待连接的请求 |
| response time | 响应耗时 |
| errors | 异常数量 |
| data received / sent | 收发数据量 |

### 常见坑

### 在 EventLoop 上 block

最常见问题：

```java
Mono<String> result = webClient.get()
        .uri("/api")
        .retrieve()
        .bodyToMono(String.class);

String body = result.block();
```

如果这段代码运行在 WebFlux 请求线程里，很容易堵住 EventLoop。

正确思路是继续返回 `Mono`：

```java
return webClient.get()
        .uri("/api")
        .retrieve()
        .bodyToMono(String.class);
```

### 混用阻塞数据库访问

WebFlux + Reactor Netty 里直接调用 JPA：

```java
User user = userRepository.findById(id).orElseThrow();
return Mono.just(user);
```

这仍然是阻塞。

要么换 R2DBC 等响应式数据访问。

要么明确切线程池：

```java
Mono.fromCallable(() -> userRepository.findById(id).orElseThrow())
        .subscribeOn(Schedulers.boundedElastic());
```

### 连接池没有限制

高并发调用下游时，不设置连接池和 pending 限制，容易让问题扩散。

建议至少设置：

```java
maxConnections
pendingAcquireMaxCount
pendingAcquireTimeout
responseTimeout
```

### responseTimeout 和 timeout 混淆

HTTP 客户端优先使用：

```java
responseTimeout(Duration.ofSeconds(5))
```

`Mono.timeout()` 更像兜底的整体链路超时。

二者语义不一样。

### ByteBuf retain / release 搞错

直接转发入站 `ByteBuf` 时，常见写法：

```java
response.send(request.receive().retain())
```

如果只是转字符串：

```java
request.receive().aggregate().asString()
```

通常不需要手动处理引用计数。

写复杂 Netty Handler 时，要重新重视 `ByteBuf` 生命周期。

### wiretap 长期开启

`wiretap(true)` 很方便，但不适合长期生产开启。

它可能带来：

* 性能损耗
* 大量日志
* 敏感信息泄露

### 把 Reactor Netty 当业务框架

Reactor Netty 是网络库，不是完整业务 Web 框架。

如果需要：

* 参数校验
* 统一异常
* 权限认证
* JSON 序列化
* OpenAPI
* 业务分层

更适合使用 Spring WebFlux。

直接写 Reactor Netty 适合网关底层、代理、协议服务、小型高性能网络服务。

### 总结

Reactor Netty 可以按这条线理解：

```text
Netty 负责底层网络 I/O
Reactor Core 提供 Mono / Flux
Reactor Netty 把 Netty 封装成响应式 HTTP / TCP API
Spring WebFlux 和 WebClient 在上层使用它
```

日常使用可以分成几类：

* 写 HTTP 服务：`HttpServer`
* 写 HTTP 客户端：`HttpClient`
* 写长连接：WebSocket
* 写私有协议：`TcpServer` / `TcpClient`
* 在 Spring 里使用：WebFlux / WebClient

真正落地时，重点不是背几个 API，而是把这些边界处理好：

* 不阻塞 EventLoop
* 配好连接池和超时
* 明确 ByteBuf 生命周期
* 控制指标标签基数
* 区分 Demo 里的 `block()` 和生产链路里的非阻塞返回
* 需要完整 Web 能力时交给 WebFlux

Reactor Netty 的价值在于把 Netty 的高性能网络能力接到 Reactor 响应式模型里。写 WebFlux、WebClient、Gateway 时，大多数网络问题最后都会落到连接池、超时、EventLoop、ByteBuf 和下游背压这些点上。把这些点搞清楚，排查响应式服务问题会轻松很多。
