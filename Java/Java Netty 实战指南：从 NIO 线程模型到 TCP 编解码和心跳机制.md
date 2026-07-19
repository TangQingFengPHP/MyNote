### 简介

Netty 是 Java 生态里很常见的高性能网络通信框架。

它基于 NIO 和事件驱动模型，把原生 NIO 里比较繁琐的 `Selector`、`Channel`、`ByteBuffer`、事件循环、半包处理、资源释放这些细节封装起来。

一句话概括：

```text
Netty 适合开发高并发、长连接、自定义协议、网关、RPC、IM、游戏服务、物联网接入这类网络应用。
```

常见应用场景：

| 场景 | 说明 |
| --- | --- |
| RPC 框架 | 服务之间走自定义 TCP 协议 |
| IM 聊天 | 长连接、消息推送、在线状态 |
| 游戏服务器 | 低延迟、长连接、频繁消息交互 |
| 网关 | TCP 网关、HTTP 网关、协议转换 |
| 物联网 | 设备接入、心跳、指令下发 |
| WebSocket | 实时通知、行情推送、在线客服 |

Netty 本身不是业务框架，它更像网络通信底座。HTTP、WebSocket、私有 TCP 协议、二进制协议，都可以在它上面实现。

### 为什么不直接用 Java Socket

最早的阻塞 Socket 写法大概是：

```text
一个连接
  -> 一个线程
      -> 阻塞读取
          -> 处理业务
```

连接少时，这种写法很直观。

连接多了以后，线程数量、上下文切换、阻塞等待、异常处理都会变复杂。

Java NIO 解决了阻塞问题，但原生 API 使用起来偏底层：

```text
Selector
SelectionKey
SocketChannel
ByteBuffer
OP_ACCEPT
OP_READ
OP_WRITE
```

Netty 在 NIO 之上做了一层工程化封装：

```text
连接接入
  -> EventLoop 处理 I/O 事件
      -> ChannelPipeline 编排处理器
          -> ChannelHandler 写业务逻辑
```

这样业务代码通常只需要关注消息怎么解码、怎么处理、怎么响应。

### Maven 依赖

普通 Maven 项目可以引入 `netty-all`：

```xml
<properties>
    <netty.version>4.1.136.Final</netty.version>
</properties>

<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-all</artifactId>
    <version>${netty.version}</version>
</dependency>
```

`netty-all` 方便学习和 Demo。

生产项目也可以按模块引入，比如：

```xml
<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-transport</artifactId>
    <version>${netty.version}</version>
</dependency>

<dependency>
    <groupId>io.netty</groupId>
    <artifactId>netty-codec</artifactId>
    <version>${netty.version}</version>
</dependency>
```

如果项目已经通过 Spring Cloud Gateway、Reactor Netty、gRPC 等框架间接使用 Netty，版本通常交给上层框架管理。

### 核心组件

Netty 的核心组件不算多，但关系要理清楚。

| 组件 | 作用 |
| --- | --- |
| `Bootstrap` | 客户端启动器 |
| `ServerBootstrap` | 服务端启动器 |
| `EventLoopGroup` | 事件循环线程组 |
| `EventLoop` | 事件循环线程，处理 I/O 事件和任务 |
| `Channel` | 一个网络连接的抽象 |
| `ChannelPipeline` | 处理器链 |
| `ChannelHandler` | 处理入站、出站事件 |
| `ByteBuf` | Netty 自己的字节缓冲区 |
| `ChannelFuture` | 异步操作结果 |

服务端整体流程：

```text
ServerBootstrap
  -> bossGroup 接收连接
      -> workerGroup 处理读写
          -> Channel
              -> ChannelPipeline
                  -> Decoder
                  -> Encoder
                  -> BusinessHandler
```

`bossGroup` 负责接收连接。

`workerGroup` 负责处理已经建立连接的读写事件。

每个连接都会对应一个 `Channel`，每个 `Channel` 都有自己的 `ChannelPipeline`。

### EventLoop 线程模型

Netty 很重要的一个设计是：

```text
一个 Channel 注册到一个 EventLoop
一个 EventLoop 绑定一个线程
一个 EventLoop 可以管理多个 Channel
```

这带来一个好处：同一个连接上的 I/O 事件通常由同一个线程处理，很多连接内状态不用到处加锁。

但也带来一个要求：`ChannelHandler` 里不适合执行耗时操作。

比如下面这些操作都容易卡住 EventLoop：

```text
慢 SQL
远程 HTTP 调用
大文件读写
复杂 CPU 计算
Thread.sleep
```

EventLoop 被卡住后，它负责的多个连接都会受影响。

耗时任务可以交给业务线程池处理，再把结果写回 Channel。

### 第一个 TCP Demo：按行收发消息

下面实现一个最小 TCP 服务：

```text
客户端发送一行文本
服务端收到后回一行文本
```

协议约定：

```text
每条消息以 \n 结尾
```

这种协议可以用 `LineBasedFrameDecoder` 解决 TCP 粘包和半包问题。

### 服务端 Handler

```java
package com.example.netty.line;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;

public class LineServerHandler extends SimpleChannelInboundHandler<String> {

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        System.out.println("客户端连接：" + ctx.channel().remoteAddress());
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) {
        String content = msg.trim();
        System.out.println("服务端收到：" + content);

        String response = "服务端已收到：" + content + "\n";
        ctx.writeAndFlush(response);
    }

    @Override
    public void channelInactive(ChannelHandlerContext ctx) {
        System.out.println("客户端断开：" + ctx.channel().remoteAddress());
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

这里继承 `SimpleChannelInboundHandler<String>`。

它会在 `channelRead0` 执行完后自动释放入站消息，适合处理已经解码后的对象。

### 服务端启动类

```java
package com.example.netty.line;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.ChannelOption;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.LineBasedFrameDecoder;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;
import io.netty.util.CharsetUtil;

public class LineNettyServer {

    private final int port;

    public LineNettyServer(int port) {
        this.port = port;
    }

    public void start() throws InterruptedException {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .option(ChannelOption.SO_BACKLOG, 128)
                    .childOption(ChannelOption.SO_KEEPALIVE, true)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) {
                            ch.pipeline()
                                    .addLast(new LineBasedFrameDecoder(1024))
                                    .addLast(new StringDecoder(CharsetUtil.UTF_8))
                                    .addLast(new StringEncoder(CharsetUtil.UTF_8))
                                    .addLast(new LineServerHandler());
                        }
                    });

            ChannelFuture future = bootstrap.bind(port).sync();
            System.out.println("Netty TCP 服务启动，端口：" + port);

            future.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new LineNettyServer(8888).start();
    }
}
```

Pipeline 顺序很关键：

```text
ByteBuf
  -> LineBasedFrameDecoder 按换行切包
      -> StringDecoder 转成 String
          -> LineServerHandler 处理业务
```

出站时方向相反：

```text
String
  -> StringEncoder 转成 ByteBuf
      -> 写回客户端
```

### 客户端 Handler

```java
package com.example.netty.line;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;

public class LineClientHandler extends SimpleChannelInboundHandler<String> {

    @Override
    public void channelActive(ChannelHandlerContext ctx) {
        ctx.writeAndFlush("Hello Netty\n");
    }

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) {
        System.out.println("客户端收到：" + msg.trim());
        ctx.close();
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

### 客户端启动类

```java
package com.example.netty.line;

import io.netty.bootstrap.Bootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioSocketChannel;
import io.netty.handler.codec.LineBasedFrameDecoder;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;
import io.netty.util.CharsetUtil;

public class LineNettyClient {

    private final String host;
    private final int port;

    public LineNettyClient(String host, int port) {
        this.host = host;
        this.port = port;
    }

    public void start() throws InterruptedException {
        EventLoopGroup group = new NioEventLoopGroup();

        try {
            Bootstrap bootstrap = new Bootstrap();
            bootstrap.group(group)
                    .channel(NioSocketChannel.class)
                    .handler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) {
                            ch.pipeline()
                                    .addLast(new LineBasedFrameDecoder(1024))
                                    .addLast(new StringDecoder(CharsetUtil.UTF_8))
                                    .addLast(new StringEncoder(CharsetUtil.UTF_8))
                                    .addLast(new LineClientHandler());
                        }
                    });

            ChannelFuture future = bootstrap.connect(host, port).sync();
            future.channel().closeFuture().sync();
        } finally {
            group.shutdownGracefully();
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new LineNettyClient("127.0.0.1", 8888).start();
    }
}
```

运行顺序：

```text
先启动 LineNettyServer
再启动 LineNettyClient
```

服务端输出：

```text
客户端连接：/127.0.0.1:xxxxx
服务端收到：Hello Netty
客户端断开：/127.0.0.1:xxxxx
```

客户端输出：

```text
客户端收到：服务端已收到：Hello Netty
```

### TCP 粘包和半包

TCP 是流协议，没有天然的消息边界。

应用层发送两条消息：

```text
hello
world
```

接收端可能看到：

```text
helloworld
```

也可能看到：

```text
hel
lowor
ld
```

这就是常说的粘包和半包。

解决办法是在应用协议里定义消息边界。

| 方案 | Netty 解码器 | 适合场景 |
| --- | --- | --- |
| 按换行切分 | `LineBasedFrameDecoder` | 命令行协议、简单文本协议 |
| 按分隔符切分 | `DelimiterBasedFrameDecoder` | 自定义分隔符文本协议 |
| 固定长度 | `FixedLengthFrameDecoder` | 每条消息长度固定 |
| 长度字段 | `LengthFieldBasedFrameDecoder` | RPC、自定义二进制协议 |

实际项目里，自定义协议最常见的是长度字段方案。

### 长度字段协议 Demo

定义一个简单协议：

```text
+----------------+----------------+
| 4 字节消息长度  | 消息内容 UTF-8  |
+----------------+----------------+
```

比如消息内容是 `ping`，长度就是 `4`。

Pipeline 可以这样配置：

```java
ch.pipeline()
        .addLast(new LengthFieldBasedFrameDecoder(
                1024 * 1024,
                0,
                4,
                0,
                4
        ))
        .addLast(new LengthFieldPrepender(4))
        .addLast(new StringDecoder(CharsetUtil.UTF_8))
        .addLast(new StringEncoder(CharsetUtil.UTF_8))
        .addLast(new LengthFieldServerHandler());
```

参数含义：

| 参数 | 值 | 说明 |
| --- | --- | --- |
| `maxFrameLength` | `1024 * 1024` | 单条消息最大 1MB |
| `lengthFieldOffset` | `0` | 长度字段从第 0 个字节开始 |
| `lengthFieldLength` | `4` | 长度字段占 4 字节 |
| `lengthAdjustment` | `0` | 长度值只表示消息体长度 |
| `initialBytesToStrip` | `4` | 解码后去掉长度字段，只保留消息体 |

`LengthFieldPrepender(4)` 会在出站消息前自动加 4 字节长度字段。

服务端 Handler：

```java
package com.example.netty.length;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;

public class LengthFieldServerHandler extends SimpleChannelInboundHandler<String> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) {
        System.out.println("收到长度字段消息：" + msg);
        ctx.writeAndFlush("pong: " + msg);
    }

    @Override
    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
        cause.printStackTrace();
        ctx.close();
    }
}
```

完整服务端：

```java
package com.example.netty.length;

import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.ChannelFuture;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.LengthFieldBasedFrameDecoder;
import io.netty.handler.codec.LengthFieldPrepender;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;
import io.netty.util.CharsetUtil;

public class LengthFieldNettyServer {

    public static void main(String[] args) throws InterruptedException {
        EventLoopGroup bossGroup = new NioEventLoopGroup(1);
        EventLoopGroup workerGroup = new NioEventLoopGroup();

        try {
            ServerBootstrap bootstrap = new ServerBootstrap();
            bootstrap.group(bossGroup, workerGroup)
                    .channel(NioServerSocketChannel.class)
                    .childHandler(new ChannelInitializer<SocketChannel>() {
                        @Override
                        protected void initChannel(SocketChannel ch) {
                            ch.pipeline()
                                    .addLast(new LengthFieldBasedFrameDecoder(1024 * 1024, 0, 4, 0, 4))
                                    .addLast(new LengthFieldPrepender(4))
                                    .addLast(new StringDecoder(CharsetUtil.UTF_8))
                                    .addLast(new StringEncoder(CharsetUtil.UTF_8))
                                    .addLast(new LengthFieldServerHandler());
                        }
                    });

            ChannelFuture future = bootstrap.bind(8899).sync();
            System.out.println("长度字段协议服务启动，端口：8899");
            future.channel().closeFuture().sync();
        } finally {
            bossGroup.shutdownGracefully();
            workerGroup.shutdownGracefully();
        }
    }
}
```

这个 Demo 适合继续扩展成：

```text
4 字节长度
1 字节消息类型
8 字节请求 ID
N 字节 JSON / Protobuf 内容
```

RPC 和内部 TCP 协议通常都会有类似的包头设计。

### ByteBuf 怎么理解

`ByteBuf` 是 Netty 的字节缓冲区。

它比 JDK `ByteBuffer` 更适合网络编程。

| 对比点 | ByteBuffer | ByteBuf |
| --- | --- | --- |
| 读写指针 | 一个 position，需要 flip | readerIndex 和 writerIndex 分开 |
| 扩容 | 使用不方便 | 可以自动扩容 |
| 池化 | 原生支持弱 | Netty 有池化分配器 |
| 引用计数 | 无 | 有引用计数 |

一个 `ByteBuf` 大概可以理解成：

```text
0                readerIndex        writerIndex              capacity
| 已读区域        | 可读区域           | 可写区域                 |
```

读取：

```java
int value = byteBuf.readInt();
```

写入：

```java
byteBuf.writeInt(100);
byteBuf.writeBytes("hello".getBytes(StandardCharsets.UTF_8));
```

### ByteBuf 释放

Netty 里很多 `ByteBuf` 是引用计数对象。

如果直接继承 `ChannelInboundHandlerAdapter` 并处理入站 `ByteBuf`，通常需要释放。

```java
package com.example.netty.buffer;

import io.netty.buffer.ByteBuf;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.util.ReferenceCountUtil;

public class RawByteBufHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void channelRead(ChannelHandlerContext ctx, Object msg) {
        try {
            ByteBuf byteBuf = (ByteBuf) msg;
            System.out.println("收到字节数：" + byteBuf.readableBytes());
        } finally {
            ReferenceCountUtil.release(msg);
        }
    }
}
```

如果使用 `SimpleChannelInboundHandler<T>`，入站消息默认会在处理后自动释放。

```java
public class TextHandler extends SimpleChannelInboundHandler<String> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) {
        System.out.println(msg);
    }
}
```

常见判断：

| 写法 | 释放方式 |
| --- | --- |
| `ChannelInboundHandlerAdapter` 直接消费 `ByteBuf` | 手动 `release` |
| `ctx.fireChannelRead(msg)` 继续传下去 | 当前 Handler 不释放 |
| `ctx.writeAndFlush(msg)` 把原消息写出去 | 写出流程完成后释放 |
| `SimpleChannelInboundHandler<T>` | 默认自动释放 |

资源释放是 Netty 开发里很重要的一块。

### ChannelFuture：异步结果

Netty 的 I/O 操作都是异步的。

```java
ChannelFuture future = ctx.writeAndFlush("hello");
```

这行代码返回时，数据不一定已经真正写到 socket。

如果要在写完后关闭连接，需要加监听器：

```java
ctx.writeAndFlush("bye\n")
        .addListener(future -> {
            if (future.isSuccess()) {
                future.channel().close();
            } else {
                future.cause().printStackTrace();
                future.channel().close();
            }
        });
```

也可以使用内置监听器：

```java
ctx.writeAndFlush("bye\n")
        .addListener(ChannelFutureListener.CLOSE);
```

### 心跳和空闲检测

长连接场景里，需要识别空闲连接。

Netty 提供 `IdleStateHandler`。

```java
ch.pipeline()
        .addLast(new IdleStateHandler(60, 0, 0))
        .addLast(new HeartbeatServerHandler());
```

含义：

| 参数 | 含义 |
| --- | --- |
| 第一个 `60` | 60 秒没有读事件，触发读空闲 |
| 第二个 `0` | 不检测写空闲 |
| 第三个 `0` | 不检测读写空闲 |

处理空闲事件：

```java
package com.example.netty.heartbeat;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.ChannelInboundHandlerAdapter;
import io.netty.handler.timeout.IdleState;
import io.netty.handler.timeout.IdleStateEvent;

public class HeartbeatServerHandler extends ChannelInboundHandlerAdapter {

    @Override
    public void userEventTriggered(ChannelHandlerContext ctx, Object evt) {
        if (evt instanceof IdleStateEvent event && event.state() == IdleState.READER_IDLE) {
            System.out.println("连接读空闲，关闭：" + ctx.channel().remoteAddress());
            ctx.close();
            return;
        }

        ctx.fireUserEventTriggered(evt);
    }
}
```

实际协议里通常会定义心跳消息：

```text
PING
PONG
```

客户端定时发送 `PING`，服务端回复 `PONG`。如果一段时间没有收到任何数据，服务端关闭连接。

### 耗时业务怎么处理

业务 Handler 里遇到慢任务，可以交给业务线程池。

```java
package com.example.netty.biz;

import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class BusinessHandler extends SimpleChannelInboundHandler<String> {

    private final ExecutorService businessExecutor = Executors.newFixedThreadPool(16);

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) {
        businessExecutor.submit(() -> {
            String result = handleBusiness(msg);
            ctx.executor().execute(() -> ctx.writeAndFlush(result + "\n"));
        });
    }

    private String handleBusiness(String msg) {
        return "result: " + msg;
    }
}
```

业务线程池处理完成后，通过 `ctx.executor().execute(...)` 回到当前 Channel 对应的 EventLoop，再写回结果。

这样可以减少并发问题，也能避免业务线程直接和 I/O 线程混在一起。

### HTTP Server Demo

Netty 也能直接写 HTTP 服务。

Pipeline：

```java
ch.pipeline()
        .addLast(new HttpServerCodec())
        .addLast(new HttpObjectAggregator(1024 * 1024))
        .addLast(new SimpleHttpServerHandler());
```

Handler：

```java
package com.example.netty.http;

import io.netty.buffer.ByteBuf;
import io.netty.buffer.Unpooled;
import io.netty.channel.ChannelFutureListener;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;
import io.netty.handler.codec.http.DefaultFullHttpResponse;
import io.netty.handler.codec.http.FullHttpRequest;
import io.netty.handler.codec.http.FullHttpResponse;
import io.netty.handler.codec.http.HttpHeaderNames;
import io.netty.handler.codec.http.HttpHeaderValues;
import io.netty.handler.codec.http.HttpResponseStatus;
import io.netty.handler.codec.http.HttpVersion;
import io.netty.util.CharsetUtil;

public class SimpleHttpServerHandler extends SimpleChannelInboundHandler<FullHttpRequest> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, FullHttpRequest request) {
        ByteBuf content = Unpooled.copiedBuffer("Hello Netty HTTP", CharsetUtil.UTF_8);

        FullHttpResponse response = new DefaultFullHttpResponse(
                HttpVersion.HTTP_1_1,
                HttpResponseStatus.OK,
                content
        );

        response.headers().set(HttpHeaderNames.CONTENT_TYPE, "text/plain; charset=UTF-8");
        response.headers().set(HttpHeaderNames.CONTENT_LENGTH, content.readableBytes());
        response.headers().set(HttpHeaderNames.CONNECTION, HttpHeaderValues.CLOSE);

        ctx.writeAndFlush(response).addListener(ChannelFutureListener.CLOSE);
    }
}
```

启动类和 TCP 服务类似，只是 Pipeline 换成 HTTP 编解码器。

如果目标是普通 Web API，Spring MVC、WebFlux、Spring Boot 内置服务器更省心。

Netty HTTP 更适合网关、协议转换、定制代理、底层通信组件这类场景。

### Spring Boot 管理 Netty 生命周期

在 Spring Boot 项目里，可以把 Netty 服务做成一个 Spring Bean，让应用启动时打开端口，关闭时释放资源。

```java
package com.example.netty.spring;

import com.example.netty.line.LineServerHandler;
import io.netty.bootstrap.ServerBootstrap;
import io.netty.channel.Channel;
import io.netty.channel.ChannelInitializer;
import io.netty.channel.EventLoopGroup;
import io.netty.channel.nio.NioEventLoopGroup;
import io.netty.channel.socket.SocketChannel;
import io.netty.channel.socket.nio.NioServerSocketChannel;
import io.netty.handler.codec.LineBasedFrameDecoder;
import io.netty.handler.codec.string.StringDecoder;
import io.netty.handler.codec.string.StringEncoder;
import io.netty.util.CharsetUtil;
import jakarta.annotation.PostConstruct;
import jakarta.annotation.PreDestroy;
import org.springframework.stereotype.Component;

@Component
public class NettyServerLifecycle {

    private EventLoopGroup bossGroup;

    private EventLoopGroup workerGroup;

    private Channel serverChannel;

    @PostConstruct
    public void start() throws InterruptedException {
        bossGroup = new NioEventLoopGroup(1);
        workerGroup = new NioEventLoopGroup();

        ServerBootstrap bootstrap = new ServerBootstrap();
        bootstrap.group(bossGroup, workerGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<SocketChannel>() {
                    @Override
                    protected void initChannel(SocketChannel ch) {
                        ch.pipeline()
                                .addLast(new LineBasedFrameDecoder(1024))
                                .addLast(new StringDecoder(CharsetUtil.UTF_8))
                                .addLast(new StringEncoder(CharsetUtil.UTF_8))
                                .addLast(new LineServerHandler());
                    }
                });

        serverChannel = bootstrap.bind(8888).sync().channel();
        System.out.println("Netty 服务已启动，端口：8888");
    }

    @PreDestroy
    public void stop() {
        if (serverChannel != null) {
            serverChannel.close();
        }
        if (workerGroup != null) {
            workerGroup.shutdownGracefully();
        }
        if (bossGroup != null) {
            bossGroup.shutdownGracefully();
        }
    }
}
```

这种方式适合在 Spring Boot 应用里额外暴露一个 TCP 端口，比如设备接入端口、内部协议端口。

### 常见 ChannelOption

服务端启动时经常会看到 `option` 和 `childOption`。

```java
bootstrap.option(ChannelOption.SO_BACKLOG, 1024)
        .childOption(ChannelOption.SO_KEEPALIVE, true)
        .childOption(ChannelOption.TCP_NODELAY, true);
```

区别如下：

| 方法 | 作用对象 |
| --- | --- |
| `option` | 服务端 ServerChannel，比如监听端口 |
| `childOption` | 已接入的客户端连接 Channel |

常见选项：

| 选项 | 说明 |
| --- | --- |
| `SO_BACKLOG` | 服务端等待连接队列大小 |
| `SO_KEEPALIVE` | TCP keepalive |
| `TCP_NODELAY` | 关闭 Nagle 算法，降低小包延迟 |
| `SO_REUSEADDR` | 地址复用 |
| `CONNECT_TIMEOUT_MILLIS` | 客户端连接超时时间 |

### Handler 共享

无状态 Handler 可以标记 `@ChannelHandler.Sharable`，多个 Channel 共用同一个实例。

```java
package com.example.netty.handler;

import io.netty.channel.ChannelHandler;
import io.netty.channel.ChannelHandlerContext;
import io.netty.channel.SimpleChannelInboundHandler;

@ChannelHandler.Sharable
public class LoggingTextHandler extends SimpleChannelInboundHandler<String> {

    @Override
    protected void channelRead0(ChannelHandlerContext ctx, String msg) {
        System.out.println("收到：" + msg);
        ctx.fireChannelRead(msg);
    }
}
```

如果 Handler 内部有连接级别状态，比如当前用户、登录状态、未完成请求 Map，就不适合共享。

### Netty 适合和不适合的场景

| 场景 | 是否适合 |
| --- | --- |
| 自定义 TCP 协议 | 适合 |
| 长连接推送 | 适合 |
| 网关和代理 | 适合 |
| RPC 通信层 | 适合 |
| 普通后台 CRUD 接口 | 通常 Spring MVC 更合适 |
| 简单 HTTP JSON API | 通常 Spring Boot 内置服务器更省心 |

Netty 的优势在底层通信和协议处理。

如果需求只是写几个 REST API，直接使用 Spring MVC 或 WebFlux 就够了。

### 常见问题

#### 服务端收不到完整消息

常见原因是 TCP 没有消息边界。

处理方式是定义应用层协议，并配置对应解码器。

```java
new LineBasedFrameDecoder(1024)
new LengthFieldBasedFrameDecoder(1024 * 1024, 0, 4, 0, 4)
```

#### Handler 里业务处理很慢

EventLoop 被慢任务阻塞后，多个连接都会受影响。

处理方式是把慢任务交给业务线程池，处理完成后再回到 EventLoop 写响应。

#### 内存持续上涨

常见原因是 `ByteBuf` 没有释放。

处理方式：

| 场景 | 建议 |
| --- | --- |
| 手动消费 ByteBuf | 使用 `ReferenceCountUtil.release(msg)` |
| 解码后只处理对象 | 使用 `SimpleChannelInboundHandler` |
| 继续向后传递 | 使用 `ctx.fireChannelRead(msg)` |
| 打开泄漏检测 | 设置 `-Dio.netty.leakDetection.level=advanced` |

#### 客户端断开后服务端还持有状态

可以在 `channelInactive` 中清理连接状态。

```java
@Override
public void channelInactive(ChannelHandlerContext ctx) {
    onlineChannels.remove(ctx.channel().id());
}
```

用户在线表、设备连接表、请求等待表，都适合在断开时清理。

#### write 后马上 close 导致消息没发完

`writeAndFlush` 是异步操作。

写完后关闭连接，要等 `ChannelFuture` 完成：

```java
ctx.writeAndFlush(response)
        .addListener(ChannelFutureListener.CLOSE);
```

### 实践建议

| 场景 | 建议 |
| --- | --- |
| 学习和 Demo | 使用 `netty-all` |
| 生产项目 | 按模块引入或交给上层框架管理版本 |
| TCP 协议 | 先设计消息边界 |
| 二进制协议 | 优先考虑长度字段 |
| 长连接 | 加心跳和空闲检测 |
| EventLoop | 避免执行慢任务 |
| ByteBuf | 明确释放责任 |
| Handler | 有状态不共享，无状态可共享 |
| 关闭应用 | 调用 `shutdownGracefully()` |
| 性能排查 | 关注 EventLoop 阻塞、内存泄漏、消息堆积 |

### 小结

Netty 的核心是异步事件驱动网络编程。

`EventLoopGroup` 管线程，`Channel` 表示连接，`ChannelPipeline` 编排处理链，`ChannelHandler` 承载业务处理，`ByteBuf` 负责高效读写字节数据。

写 Netty 程序时，最关键的不是背 API，而是抓住几个工程点：协议边界、线程模型、资源释放、心跳检测、异步结果处理。

把这些点处理好，Netty 就能成为稳定的 TCP、WebSocket、RPC、网关和设备接入通信底座。
