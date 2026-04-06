### 简介

很多人第一次认真看 `gRPC`，通常不是因为“想学一个新协议”，而是因为项目已经出现了这些信号：

* 服务和服务之间调用越来越多
* `REST` 的 JSON 体积、契约漂移、字段随意扩张开始让人烦
* 想要更明确的接口定义
* 想做流式推送，又不想自己再拼一套长连接协议

这时候，`gRPC` 往往就会进入视野。

一句话先说透：

> `gRPC` 本质上是一种基于 `HTTP/2 + Protocol Buffers` 的远程调用方式，它最适合的不是“替代所有 HTTP API”，而是进程间、服务间那种对契约、性能和通信模型更敏感的调用场景。

这篇文章我不打算写成“概念词典”。

更务实一点，我想围绕这些问题来讲：

* 为什么有些项目一上来就适合 `gRPC`，有些项目反而用 `REST` 更顺手；
* `.proto` 到底解决了什么，不只是“换了个序列化格式”；
* 四种调用模型到底什么时候真有价值；
* 在 `.NET` 里，服务端和客户端怎么接，哪里最容易踩坑；
* 它和 `REST`、`SignalR`、MQ 的边界到底在哪。

### 先别急着记定义，先看它解决什么问题

假设你现在有三个内部服务：

* 订单服务
* 商品服务
* 库存服务

如果它们之间全用传统 `REST + JSON` 调：

* 地址当然能调通
* 接口当然也能跑
* 但时间一长，问题会慢慢冒出来

例如：

* DTO 契约容易散
* 字段变更靠人约定
* 错误码和异常风格各家都不一样
* 文档和真实实现容易脱节
* 高频内部调用下，文本协议的开销也不是完全没感觉

`gRPC` 首先解决的，其实不是“更快”这么简单，而是：

* 契约更强
* 调用模型更明确
* 服务间通信更偏工程化

### `gRPC` 到底是什么？

可以先把它拆成两层来看：

#### 第一层：传输层

它基于：

* `HTTP/2`

这意味着它天然能用到：

* 多路复用
* 头压缩
* 更适合长连接和高频请求的传输模型

#### 第二层：消息定义层

它通常使用：

* `Protocol Buffers`

也就是 `.proto` 文件来定义：

* 服务
* 方法
* 请求消息
* 响应消息

所以你可以把它理解成：

* `HTTP/2` 负责怎么传
* `Proto` 负责传什么

### 为什么很多人会把它和 `REST` 放在一起比较？

因为它们都能做“远程调用”。

但这两者最适合的问题形状并不一样。

`REST` 更像：

* 面向资源
* 对外接口友好
* 调试门槛低
* 浏览器生态天然兼容

`gRPC` 更像：

* 面向服务契约
* 面向内部调用
* 强类型生成客户端
* 更适合高频、稳定、规范化的服务间通信

所以更务实地说：

* 面向前端、开放平台、第三方集成，`REST` 往往更自然
* 面向内部服务互调，`gRPC` 往往更顺手

这不是“谁更高级”，而是接口受众不同。

### `Proto` 真正值钱的地方是什么？

很多文章会把重点放在“二进制更小更快”。

这当然没错，但我觉得更值钱的是另外一件事：

> 接口契约终于从“口头约定 + JSON 示例”变成了一个真正可编译、可生成代码、可版本化的文件。

例如：

```proto
syntax = "proto3";

option csharp_namespace = "ProductGrpc";

service ProductService {
  rpc GetProduct (GetProductRequest) returns (GetProductReply);
}

message GetProductRequest {
  int32 id = 1;
}

message GetProductReply {
  int32 id = 1;
  string name = 2;
  double price = 3;
}
```

这段 `.proto` 不只是“文档”，它本身就是契约来源。

然后服务端和客户端都围绕它生成代码。

这件事的工程价值非常大：

* 契约更统一
* 生成的客户端更强类型
* 变更更容易被发现
* 服务间调用不再只是“猜这个 JSON 长什么样”

### `.proto` 里字段编号为什么比字段名更重要？

这是很多人第一次上手 `gRPC` 时最容易低估的一点。

在 `Protocol Buffers` 里，真正参与网络传输标识的，不是字段名，而是：

* 字段编号

例如：

```proto
message GetProductReply {
  int32 id = 1;
  string name = 2;
  double price = 3;
}
```

这里真正敏感的是：

* `1`
* `2`
* `3`

而不是 `id`、`name`、`price` 这些名字本身。

更务实地说：

* 改字段名，通常不一定是协议级破坏
* 改字段编号，才是真正危险的事

微软的 gRPC versioning 文档也明确提到了这一点：  
字段号才是 Protobuf 在线路上的识别依据，修改字段号属于协议级破坏性变更。  
来源：
https://learn.microsoft.com/en-us/aspnet/core/grpc/versioning?view=aspnetcore-8.0

### `.proto` 兼容性规则，项目里最该记哪些？

如果你不想把接口版本演进搞乱，至少记住这几条。

#### 1. 新增字段，通常比修改旧字段安全

这是最基本的原则。

如果只是新增一个可选语义字段，老客户端通常还能继续工作。

#### 2. 不要改字段编号

这是最重要的一条。

字段号一旦上线，就应该尽量视为稳定协议的一部分。

#### 3. 不要随便重用废弃字段号

哪怕某个字段你已经不想用了，也别轻易拿它的编号去给新字段复用。

更稳的习惯是：

* 保留旧编号
* 必要时显式标记废弃

#### 4. 改字段类型要非常谨慎

哪怕代码能编译，也不代表线上老客户端能正确反序列化。

#### 5. 改服务名、方法名、包名也要谨慎

因为 gRPC 调用路径本身就和这些名字有关。

这类变更不是“重构一下名字”那么轻。

如果只记一句话：

> `.proto` 的版本兼容，首先要保护字段号，其次才是代码里的类型名和属性名。

### 在 `.NET` 里，一个 gRPC 服务最小会长什么样？

最基础的流程其实很短。

#### 第一步：建项目或加包

服务端通常会用：

```bash
dotnet add package Grpc.AspNetCore
```

客户端通常会用：

```bash
dotnet add package Grpc.Net.Client
```

如果是 ASP.NET Core gRPC 模板，很多东西会直接帮你铺好。

#### 第二步：定义 `.proto`

比如：

```proto
syntax = "proto3";

option csharp_namespace = "OrderGrpc";

service OrderService {
  rpc GetOrder (GetOrderRequest) returns (GetOrderReply);
}

message GetOrderRequest {
  int32 id = 1;
}

message GetOrderReply {
  int32 id = 1;
  string status = 2;
}
```

#### 第三步：在项目文件里声明它

服务端常见写法：

```xml
<ItemGroup>
  <Protobuf Include="Protos\order.proto" GrpcServices="Server" />
</ItemGroup>
```

客户端常见写法：

```xml
<ItemGroup>
  <Protobuf Include="Protos\order.proto" GrpcServices="Client" />
</ItemGroup>
```

### `OrderService.OrderServiceBase` 到底是什么？

这个点很容易让人第一次看示例时觉得有点“凭空冒出来一个基类”。

它不是你手写的类，而是：

* 根据 `.proto` 文件
* 在编译时自动生成出来的服务端基类

也就是说，当你在项目文件里写了：

```xml
<Protobuf Include="Protos\order.proto" GrpcServices="Server" />
```

编译器会基于：

```proto
service OrderService {
  rpc GetOrder (GetOrderRequest) returns (GetOrderReply);
}
```

生成一批 C# 代码。

这些生成代码里，最常见的两类东西分别是：

* `OrderService.OrderServiceBase`
* `OrderService.OrderServiceClient`

可以先这样记：

* `OrderServiceBase`：给服务端继承
* `OrderServiceClient`：给客户端调用

所以你看到这段代码：

```csharp
public class OrderGrpcService : OrderService.OrderServiceBase
{
    public override Task<GetOrderReply> GetOrder(GetOrderRequest request, ServerCallContext context)
    {
        ...
    }
}
```

本质上就是：

* 继承代码生成器生成的服务端基类
* 重写 `.proto` 里定义的 RPC 方法
* 把“契约”落成“实现”

如果非要打个比方，它有点像：

* ASP.NET Core 里的 `ControllerBase`

区别在于：

* `ControllerBase` 是框架固定提供的
* `OrderServiceBase` 是根据你的 `.proto` 契约动态生成的

所以它看起来像“魔法”，其实只是代码生成。

#### 第四步：实现服务

```csharp
public class OrderGrpcService : OrderService.OrderServiceBase
{
    public override Task<GetOrderReply> GetOrder(GetOrderRequest request, ServerCallContext context)
    {
        return Task.FromResult(new GetOrderReply
        {
            Id = request.Id,
            Status = "Paid"
        });
    }
}
```

#### 第五步：注册 gRPC

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddGrpc();

var app = builder.Build();

app.MapGrpcService<OrderGrpcService>();

app.Run();
```

这就是最小服务端链路。

### 客户端怎么调？

如果是最直接的方式，大概像这样：

```csharp
using Grpc.Net.Client;

using var channel = GrpcChannel.ForAddress("https://localhost:5001");
var client = new OrderService.OrderServiceClient(channel);

var reply = await client.GetOrderAsync(new GetOrderRequest
{
    Id = 1
});

Console.WriteLine(reply.Status);
```

这段代码里最值钱的地方，不是“少写几行”。

而是：

* 你拿到的是强类型客户端
* 方法签名和消息结构都来自同一份 `.proto`

这和“手写一个 `HttpClient` 然后拼 JSON 再反序列化”是很不一样的体验。

### 拦截器在 gRPC 里到底值不值得用？

如果你只做一个 demo，当然可以不用。

但一旦项目进入多人协作和多服务互调阶段，拦截器通常就会变得很有价值。

它最适合处理这些横切逻辑：

* 日志
* TraceId / CorrelationId
* 鉴权元数据
* 统一异常映射
* 指标采集

也就是说，它和 ASP.NET Core 里的中间件很像，但作用点更贴近 gRPC 调用本身。

微软的拦截器文档也明确把 deadline、cancellation、metadata 这些调用上下文信息放进了拦截器的典型使用场景里。  
来源：
https://learn.microsoft.com/en-us/aspnet/core/grpc/interceptors?view=aspnetcore-10.0

### 一个最小的服务端拦截器长什么样？

例如最常见的日志拦截器：

```csharp
using Grpc.Core;
using Grpc.Core.Interceptors;

public sealed class ServerLoggingInterceptor : Interceptor
{
    private readonly ILogger<ServerLoggingInterceptor> _logger;

    public ServerLoggingInterceptor(ILogger<ServerLoggingInterceptor> logger)
    {
        _logger = logger;
    }

    public override async Task<TResponse> UnaryServerHandler<TRequest, TResponse>(
        TRequest request,
        ServerCallContext context,
        UnaryServerMethod<TRequest, TResponse> continuation)
    {
        _logger.LogInformation("gRPC {Method} started", context.Method);

        var response = await continuation(request, context);

        _logger.LogInformation("gRPC {Method} finished", context.Method);
        return response;
    }
}
```

注册方式也很直接：

```csharp
builder.Services.AddGrpc(options =>
{
    options.Interceptors.Add<ServerLoggingInterceptor>();
});
```

这类拦截器在真实项目里非常常见，因为它能把公共逻辑从每个服务方法里抽掉。

### 截止时间和取消传播，为什么在 gRPC 里特别重要？

因为 gRPC 很容易被拿来做内部高频同步调用。

而同步调用最怕的就是：

* 下游慢
* 上游一直等
* 超时没控制好
* 整条链越拖越长

所以 deadline 不是“高级特性”，而是可靠性基础设施。

微软文档明确提到两件事：

* gRPC 调用默认没有 deadline
* deadline 超时后，客户端会终止调用，服务端侧则会触发 `ServerCallContext.CancellationToken`

来源：
https://learn.microsoft.com/en-us/aspnet/core/grpc/deadlines-cancellation?view=aspnetcore-9.0

### 客户端怎么设 deadline？

最直接的方式就是在调用时带上：

```csharp
var reply = await client.GetOrderAsync(
    new GetOrderRequest { Id = 1 },
    deadline: DateTime.UtcNow.AddSeconds(3));
```

这个配置的意义非常现实：

* 最多等 3 秒
* 超了就别再无休止挂着

如果只记一句话：

* 没有 deadline 的内部 RPC，迟早会有人把它调用到挂死

### 服务端为什么不能只靠客户端超时，还要处理取消？

因为客户端超时之后，服务端不一定会自动“马上停止业务逻辑”。

如果你的服务端还在：

* 查数据库
* 调别的 gRPC 服务
* 调外部 HTTP 服务

那最稳的写法是把：

```csharp
context.CancellationToken
```

一路往下传。

例如：

```csharp
public override async Task<GetOrderReply> GetOrder(GetOrderRequest request, ServerCallContext context)
{
    var order = await _repository.GetByIdAsync(request.Id, context.CancellationToken);

    return new GetOrderReply
    {
        Id = order.Id,
        Status = order.Status
    };
}
```

这不是“写得规范一点”的问题，而是：

* 如果不传，服务端可能还在白白消耗资源

### 调用链里 deadline 要不要继续往下传？

要。

这是很多服务间调用链最容易漏掉的一点。

假设链路是：

```text
前端服务 -> 订单服务 -> 商品服务
```

如果前端对订单服务设了 3 秒 deadline，那订单服务继续调商品服务时，通常也应该尊重这条 deadline，而不是重新当作一条“无限等待”的新请求。

最直接的方式是手动传：

```csharp
var product = await productClient.GetProductAsync(
    new GetProductRequest { Id = request.ProductId },
    deadline: context.Deadline,
    cancellationToken: context.CancellationToken);
```

如果项目已经用了 gRPC client factory，也可以考虑：

* `EnableCallContextPropagation()`

让 deadline 和 cancellation 自动往下传。

这一点微软文档也单独强调过。  
来源：
https://learn.microsoft.com/en-us/aspnet/core/grpc/deadlines-cancellation?view=aspnetcore-9.0

### 真正需要理解的，是四种调用模型

这才是 `gRPC` 和普通 HTTP JSON API 拉开差距的地方。

#### 1. Unary

最普通的一种：

* 一个请求
* 一个响应

这和常规 RPC 很像，也是最容易落地的一种。

如果你只是想替换服务间同步查询，很多时候先从它开始就够了。

最小示例：

```proto
service ProductService {
  rpc GetProduct (GetProductRequest) returns (GetProductReply);
}
```

服务端：

```csharp
public override Task<GetProductReply> GetProduct(GetProductRequest request, ServerCallContext context)
{
    return Task.FromResult(new GetProductReply
    {
        Id = request.Id,
        Name = "Mechanical Keyboard",
        Price = 499
    });
}
```

客户端：

```csharp
var reply = await client.GetProductAsync(new GetProductRequest
{
    Id = 1
});
```

#### 2. Server streaming

* 客户端发一个请求
* 服务端持续返回多条消息

这很适合：

* 导出进度
* 日志流
* 大结果集分段返回
* 服务端逐步推送状态

最小示例：

```proto
service ImportService {
  rpc ImportProducts (ImportProductsRequest) returns (stream ImportProductsReply);
}
```

服务端：

```csharp
public override async Task ImportProducts(
    ImportProductsRequest request,
    IServerStreamWriter<ImportProductsReply> responseStream,
    ServerCallContext context)
{
    for (var i = 1; i <= 5; i++)
    {
        await Task.Delay(300, context.CancellationToken);

        await responseStream.WriteAsync(new ImportProductsReply
        {
            Message = $"step {i}/5 finished"
        });
    }
}
```

客户端：

```csharp
using var call = client.ImportProducts(new ImportProductsRequest { FileName = "products.csv" });

await foreach (var item in call.ResponseStream.ReadAllAsync())
{
    Console.WriteLine(item.Message);
}
```

#### 3. Client streaming

* 客户端持续发多条消息
* 服务端最后统一返回一个响应

适合：

* 批量上传
* 客户端分片提交
* 收集一批数据后统一处理

最小示例：

```proto
service UploadService {
  rpc UploadLogs (stream UploadLogRequest) returns (UploadLogReply);
}
```

服务端：

```csharp
public override async Task<UploadLogReply> UploadLogs(
    IAsyncStreamReader<UploadLogRequest> requestStream,
    ServerCallContext context)
{
    var count = 0;

    await foreach (var item in requestStream.ReadAllAsync(context.CancellationToken))
    {
        count++;
    }

    return new UploadLogReply
    {
        Count = count
    };
}
```

客户端：

```csharp
using var call = client.UploadLogs();

for (var i = 1; i <= 3; i++)
{
    await call.RequestStream.WriteAsync(new UploadLogRequest
    {
        Message = $"log-{i}"
    });
}

await call.RequestStream.CompleteAsync();
var reply = await call;
Console.WriteLine(reply.Count);
```

#### 4. Bidirectional streaming

* 双方都能持续发消息

这已经不是“普通接口调用”的感觉了，更像一条双向通信通道。

适合：

* 实时状态同步
* 双向控制流
* 流式协作场景

但也正因为如此，它的心智负担会明显更高，不建议一上来就把所有接口都往这里堆。

最小示例：

```proto
service ChatService {
  rpc Chat (stream ChatMessage) returns (stream ChatMessage);
}
```

服务端：

```csharp
public override async Task Chat(
    IAsyncStreamReader<ChatMessage> requestStream,
    IServerStreamWriter<ChatMessage> responseStream,
    ServerCallContext context)
{
    await foreach (var item in requestStream.ReadAllAsync(context.CancellationToken))
    {
        await responseStream.WriteAsync(new ChatMessage
        {
            User = "server",
            Text = $"echo: {item.Text}"
        });
    }
}
```

客户端：

```csharp
using var call = client.Chat();

var readTask = Task.Run(async () =>
{
    await foreach (var item in call.ResponseStream.ReadAllAsync())
    {
        Console.WriteLine($"{item.User}: {item.Text}");
    }
});

await call.RequestStream.WriteAsync(new ChatMessage { User = "client", Text = "hello" });
await call.RequestStream.WriteAsync(new ChatMessage { User = "client", Text = "world" });
await call.RequestStream.CompleteAsync();

await readTask;
```

### 一个更贴近项目的例子：商品导入状态流

假设你有一个后台导入任务，前端想实时看到处理进度。

传统 REST 做法通常会变成：

* 前端不停轮询
* 服务端不停查状态表

这当然能做，但挺笨重。

如果用 `server streaming`，服务端可以这样持续推送：

```csharp
public override async Task ImportProducts(
    ImportProductsRequest request,
    IServerStreamWriter<ImportProductsReply> responseStream,
    ServerCallContext context)
{
    for (var i = 1; i <= 10; i++)
    {
        await Task.Delay(300, context.CancellationToken);

        await responseStream.WriteAsync(new ImportProductsReply
        {
            Message = $"step {i}/10 finished"
        });
    }
}
```

客户端则持续读取：

```csharp
using var call = client.ImportProducts(new ImportProductsRequest { FileName = "products.csv" });

await foreach (var item in call.ResponseStream.ReadAllAsync())
{
    Console.WriteLine(item.Message);
}
```

这个例子很能体现 `gRPC` 的价值：

* 你不用再自己设计一套轮询协议
* 接口契约还是统一在 `.proto`
* 流式语义本身就是一等公民

### 一个更完整的订单服务调用商品服务案例

如果你想把前面的点串起来，最贴近真实项目的例子通常是：

* `order-service` 对外提供下单查询接口
* 它内部通过 gRPC 调 `product-service`
* 拿到商品基础信息后再组装订单详情

#### 第一步：先定义共享契约

例如 `product.proto`：

```proto
syntax = "proto3";

option csharp_namespace = "ProductGrpc";

service ProductService {
  rpc GetProduct (GetProductRequest) returns (GetProductReply);
}

message GetProductRequest {
  int32 id = 1;
}

message GetProductReply {
  int32 id = 1;
  string name = 2;
  double price = 3;
  string category_name = 4;
}
```

这里最值得注意的不是字段内容，而是：

* 字段编号已经定下来了
* 后面扩展字段时尽量新增，不要乱改旧编号

#### 第二步：商品服务实现 gRPC 服务

```csharp
public sealed class ProductGrpcService : ProductService.ProductServiceBase
{
    public override Task<GetProductReply> GetProduct(GetProductRequest request, ServerCallContext context)
    {
        return Task.FromResult(new GetProductReply
        {
            Id = request.Id,
            Name = "Mechanical Keyboard",
            Price = 499,
            CategoryName = "Keyboard"
        });
    }
}
```

服务注册：

```csharp
builder.Services.AddGrpc(options =>
{
    options.Interceptors.Add<ServerLoggingInterceptor>();
});

var app = builder.Build();
app.MapGrpcService<ProductGrpcService>();
app.Run();
```

#### 第三步：订单服务注册 gRPC 客户端

```csharp
builder.Services
    .AddGrpcClient<ProductService.ProductServiceClient>(options =>
    {
        options.Address = new Uri("https://localhost:7001");
    });
```

如果订单服务本身也是一个 gRPC 服务，并且希望自动传播上下文，还可以继续补：

```csharp
builder.Services
    .AddGrpcClient<ProductService.ProductServiceClient>(options =>
    {
        options.Address = new Uri("https://localhost:7001");
    })
    .EnableCallContextPropagation();
```

#### 第四步：订单服务内部调用商品服务

```csharp
public sealed class OrderGrpcService : OrderService.OrderServiceBase
{
    private readonly ProductService.ProductServiceClient _productClient;

    public OrderGrpcService(ProductService.ProductServiceClient productClient)
    {
        _productClient = productClient;
    }

    public override async Task<GetOrderReply> GetOrder(GetOrderRequest request, ServerCallContext context)
    {
        var productReply = await _productClient.GetProductAsync(
            new GetProductRequest { Id = request.ProductId },
            deadline: context.Deadline,
            cancellationToken: context.CancellationToken);

        return new GetOrderReply
        {
            Id = request.Id,
            ProductName = productReply.Name,
            ProductPrice = productReply.Price,
            Status = "Paid"
        };
    }
}
```

这段代码里，真正像项目代码的点有三个：

* 强类型客户端不是手写拼出来的
* 上游 deadline 被继续往下传
* 上游 cancellation 也被继续往下传

#### 第五步：怎么验证这个案例真的成立？

最简单的验证方式通常是：

1. 先让 `product-service` 正常返回，确认 `order-service` 能拿到商品详情
2. 再故意让 `product-service` 延迟 5 秒
3. 在 `order-service` 调用时只给 2 秒 deadline
4. 看客户端是否拿到超时异常，同时服务端是否能尽快响应取消

这一步特别重要，因为它能把“调用能通”和“调用是可靠的”区分开。

### 在 `.NET` 里，gRPC 最容易踩的坑是什么？

#### 1. 把它当成“什么都该替换成 gRPC”

这通常是过度设计。

如果接口主要给前端、第三方、浏览器直接调用，很多时候 `REST` 更自然。

#### 2. 忽略 `HTTP/2` 运行环境

`gRPC` 对传输层是有要求的。

在 ASP.NET Core 里，官方文档明确强调 gRPC 依赖 `HTTP/2`，Kestrel 上的 gRPC 端点也通常应该配 `TLS`。  
来源：
https://learn.microsoft.com/en-us/aspnet/core/grpc/aspnetcore?view=aspnetcore-9.0

也就是说，别只在代码层看“能不能编译”，部署层也要确认：

* 反向代理是否支持
* `HTTP/2` 是否真的打通
* 证书和 TLS 是否配置正确

#### 3. 把流式调用当成“更高级的默认选择”

很多接口其实一个 `unary` 就够了。

流式调用真正值钱的前提是：

* 你真的有持续产出或持续消费消息的需求

不然只会增加实现复杂度。

#### 4. 版本演进没有设计好

虽然 `proto3` 对新增字段比较友好，但这不等于你可以乱改契约。

更稳的习惯仍然是：

* 能新增字段就别直接改旧语义
* 字段号要谨慎管理
* 别轻易重用历史字段号

### 它和 `SignalR`、MQ 的边界是什么？

这组对比特别值得讲清楚。

#### `gRPC` vs `SignalR`

`gRPC` 更像：

* 服务和服务之间的强契约调用
* 请求/响应或流式 RPC

`SignalR` 更像：

* 面向客户端连接管理
* 实时推送
* Hub 模型

所以不要把它们混着理解成“都能推消息，所以差不多”。

它们解决的重点完全不同。

#### `gRPC` vs MQ

`gRPC` 还是在线调用。

也就是说：

* 调用方和被调用方在调用时要同时在线
* 失败通常要立即感知

而 MQ 更偏：

* 异步解耦
* 削峰
* 最终一致
* 重试和延迟消费

所以 `gRPC` 再强，也不是消息队列替代品。

### 它和 REST 到底怎么选？

可以先按这个顺序判断：

#### 更适合 REST

* 面向前端/第三方开放
* 希望调试门槛低
* 强依赖浏览器生态
* 接口更偏资源风格

#### 更适合 gRPC

* 服务间内部通信
* 契约需要强约束
* 希望自动生成客户端
* 需要流式调用
* 高频调用，希望减少文本协议开销

这不是绝对二选一。

真实项目里很常见的组合反而是：

* 对外 REST
* 对内 gRPC

### 一个非常务实的选择顺序

如果你在做 `.NET` 服务通信选型，可以先按这个顺序判断：

1. 这是对外 API 还是内部服务调用？
2. 如果是对外，先想 REST 是不是已经够好
3. 如果是内部服务互调，再考虑 gRPC
4. 如果需要真正异步解耦，不要继续纠结 gRPC，直接看 MQ
5. 如果需要浏览器实时推送，优先看 SignalR

这个顺序很重要。

因为很多团队不是“不会用 gRPC”，而是一开始就没分清通信问题的形状。

### 面试里怎么答比较到位？

如果面试官问：

“gRPC 和 REST 的区别是什么？”

一个比较自然的回答可以是：

> `gRPC` 是基于 HTTP/2 和 Protocol Buffers 的 RPC 调用方式，更适合服务间内部通信。它的优势不只是序列化更紧凑，更重要的是契约更强、客户端能自动生成、并且天然支持 unary、server streaming、client streaming、bidirectional streaming 这几种调用模型。REST 则更适合对外接口、浏览器生态和资源风格 API。很多项目实际上会对外用 REST、对内用 gRPC。

如果继续追问“为什么 `.proto` 很重要”，可以答：

> 因为 `.proto` 不只是文档，它本身就是契约来源。服务端和客户端都围绕同一份定义生成代码，这比靠 JSON 示例和人工维护 DTO 协议要稳得多。

如果再追问“最大的坑是什么”，优先答这三个：

* 忽略部署层的 HTTP/2 / TLS 条件
* 把所有接口都想当然地替换成 gRPC
* 没想清楚流式调用到底是不是真需要

### 总结

`gRPC` 最值得记住的，不是“更快”这两个字，而是它解决的问题形状：

> 当你的系统已经明显进入“内部服务之间需要强契约、高频调用、甚至流式通信”的阶段时，`gRPC` 会比继续用松散的 JSON 接口更顺手。

如果你只想记住几句话，可以记这几条：

* 它更适合服务间内部通信，不是所有 API 的默认答案；
* `Proto` 的价值很大一部分在于契约治理，而不只是序列化性能；
* Unary 是默认主力，流式调用要按需上，不要为了“高级”而上；
* 它不是 MQ，也不是 `SignalR` 的替代品；
* 真实项目里，对外 REST、对内 gRPC 往往是很自然的组合。

参考资料：

* Microsoft Learn: Overview for gRPC on .NET  
  https://learn.microsoft.com/en-us/aspnet/core/grpc/?view=aspnetcore-9.0
* Microsoft Learn: gRPC services with ASP.NET Core  
  https://learn.microsoft.com/en-us/aspnet/core/grpc/aspnetcore?view=aspnetcore-9.0
* Microsoft Learn: Call gRPC services with the .NET client  
  https://learn.microsoft.com/en-us/aspnet/core/grpc/client?view=aspnetcore-9.0
* gRPC: What is gRPC?  
  https://grpc.io/docs/what-is-grpc/
