### 简介

把一个系统拆成多个项目，并不自动等于微服务。

真正的微服务架构里，每个服务围绕清晰的业务能力组织，可以独立构建、部署和扩缩容；服务管理自己的业务规则和数据，通过网络协议协作。它带来的不是免费的弹性，而是一整套新的分布式问题：

* 网络会超时、断开或重复发送；
* 服务地址会变化，需要服务发现或部署平台网络；
* 一个请求可能跨多个进程，排查要靠日志、指标和追踪；
* 数据分属不同服务，不能再随意跨库 JOIN 或依赖单个数据库事务；
* 部署、配置、认证、健康检查和版本兼容都会更复杂。

因此，微服务适合有明确业务边界、独立发布和独立扩缩容需求的团队，不是所有项目的默认答案。规模不大时，模块化单体往往更简单，也能保留清晰边界。

本文先把架构原则讲清，再用 ASP.NET Core、YARP 和 HttpClientFactory 搭建一个商品服务、订单服务、网关组成的最小实战。代码重点是演示职责和调用链，不把内存集合伪装成生产数据库。

### 微服务到底是什么

可以把微服务理解为按业务能力拆分的一组独立应用：

~~~text
客户端
  ↓
API Gateway
  ├── Catalog Service：商品目录和价格
  └── Order Service：订单和订单状态
~~~

微服务的关键字不是“小”，而是：

* **业务边界明确**：一个服务负责一块相对完整的业务能力；
* **独立部署**：商品服务发布不必让订单服务一起重新发布；
* **数据归属清楚**：商品数据由商品服务负责，订单数据由订单服务负责；
* **网络协作**：进程之间通过 HTTP、gRPC 或消息系统通信；
* **故障隔离**：单个服务失败时，其他服务尽量继续提供能力。

一个服务可以由多个内部项目组成，但不能把共享数据库、同步发布和跨服务直接访问表结构当成“自治”。

### 单体、模块化单体和微服务

| 维度 | 单体 | 模块化单体 | 微服务 |
| --- | --- | --- | --- |
| 部署单元 | 一个应用 | 一个应用 | 多个服务 |
| 代码边界 | 容易混在一起 | 模块边界清晰 | 服务边界通过网络隔开 |
| 数据访问 | 常见共用数据库 | 可按模块隔离 | 每个服务拥有自己的数据 |
| 本地调用 | 进程内方法调用 | 进程内模块接口 | HTTP、gRPC、消息 |
| 运维成本 | 低 | 低到中 | 高 |
| 独立扩缩容 | 不方便 | 不方便 | 可以 |
| 分布式故障 | 少 | 少 | 必须处理 |

模块化单体通常是合理的起点：先把业务模块和依赖方向治理好，等某个边界确实需要独立部署、团队自治或单独扩缩容时，再拆成服务。直接从一团耦合代码切出十几个 API，只会把单体中的耦合搬到网络上。

### 服务边界怎么拆

服务边界首先是业务问题，不是按技术层拆文件夹。

不推荐这样拆：

~~~text
CustomerController Service
CustomerBusinessService
CustomerRepositoryService
~~~

这通常只是按 Controller、Service、Repository 分进程，每个请求仍要跨多个服务走一圈，变成“网络版三层架构”。

更合理的切分方式是围绕业务能力和领域边界：

~~~text
商品目录：商品信息、价格、上下架
订单：下单、订单状态、取消规则
库存：可售数量、预占和释放
支付：支付请求、支付结果
~~~

判断某块业务是否应该独立成服务，可以检查：

* 它有没有稳定、清楚的业务职责？
* 它的数据和业务规则能不能由同一个团队独立维护？
* 它是否真的需要独立扩容或独立发布？
* 拆开后，跨服务调用次数和运维成本是否可以接受？
* 它能否在其他服务不可用时提供合理行为？

如果每次改一个字段都需要同时改四个服务，边界大概率还没划好。

### 服务与数据库：数据归属是硬边界

微服务通常遵循 Database per Service：每个服务拥有自己的业务数据，其他服务不能绕过 API 直接读写它的表。

~~~text
Catalog Service  → Catalog DB
Order Service    → Order DB
Inventory Service → Inventory DB
~~~

“每服务一个数据库”不一定要求每个服务购买一台数据库服务器。开发和小规模部署可以共用数据库实例，但应保持独立数据库或 schema、独立迁移和独立访问权限。核心规则是所有权独立，不能让其他服务依赖内部表结构。

订单需要保存商品信息时，通常保存下单时的商品名称和价格快照，而不是每次读订单都跨库查询商品表。跨服务数据组合可通过 API 查询、事件复制必要字段或专门的查询聚合层完成。

不同服务的数据模型可以不一样。订单关心的商品快照和商品目录自己的完整商品实体，不必强行复用同一个数据库实体类。

### 服务之间怎么通信

#### HTTP / REST

适合：

* 查询当前数据；
* 面向浏览器、移动端或外部系统开放 API；
* 调用链短、需要立即返回结果的操作。

缺点是调用方必须等被调用服务回应。服务不可用、网络延迟或下游超时都会影响当前请求。

#### gRPC

适合内部服务之间高频、强类型、需要 HTTP/2 或流式通信的场景。契约用 Protocol Buffers 描述，生成客户端和服务端代码。使用门槛和代理、负载均衡、调试要求也比普通 HTTP 高。不要为了“微服务就得 gRPC”把所有接口改成 gRPC。

#### 消息队列和事件

适合：

* 不需要在当前 HTTP 请求中等处理完成的工作；
* 广播状态变化，例如 OrderCreated；
* 解耦上下游发布节奏；
* 服务暂时不可用时需要排队重试。

消息带来最终一致性、重复投递、幂等消费、死信处理和消息顺序等新问题。消息系统不是把 HTTP URL 换掉就结束。

### 同步调用和异步事件的取舍

一次下单请求如果同步执行：

~~~text
客户端 → 网关 → 订单服务 → 商品服务 → 库存服务 → 支付服务
~~~

每多一个同步依赖，请求就多一个故障点和延迟来源。若支付或库存服务变慢，订单请求会跟着变慢；下游不可用时，整条链可能失败。

更稳妥的流程通常是：

~~~text
同步：校验必要条件并创建订单
异步：发布 OrderCreated
      → 库存服务预占
      → 支付服务发起支付
      → 通知服务发送消息
~~~

是否异步取决于业务承诺。如果页面必须立即显示“库存已锁定”，库存预占可能仍需同步；但通知、报表和后续流程通常可以异步。关键是明确用户可见的状态和补偿路径，而不是机械地追求全异步。

### .NET 微服务常见构件

| 能力 | .NET 常见方案 | 主要职责 |
| --- | --- | --- |
| 服务 API | ASP.NET Core Minimal API / Controller | 对外或对内提供 HTTP API |
| API Gateway | YARP | 统一入口、路由和代理策略 |
| 服务间 HTTP | HttpClientFactory | 管理 HttpClient 和 handler 生命周期 |
| 强类型 RPC | gRPC | 内部高效 RPC 和流式通信 |
| 消息传递 | RabbitMQ、Azure Service Bus、Kafka 等 | 异步命令和事件 |
| 本地分布式应用编排 | .NET Aspire | 开发时启动依赖、服务发现和诊断 |
| 容器 | Docker / OCI | 打包应用和运行环境 |
| 生产编排 | Kubernetes、云托管容器平台 | 调度、扩缩容、探针和滚动发布 |
| 可观测性 | OpenTelemetry | 日志、指标、分布式追踪关联 |

工具选择不是架构本身。引入 Kubernetes 不会自动让服务自治，安装消息队列也不会自动产生可靠事件流程。

### 实战 Demo：商品服务、订单服务和 YARP 网关

这个 Demo 展示三条链路：

~~~text
浏览器 → YARP Gateway → Catalog Service
浏览器 → YARP Gateway → Order Service
Order Service → HttpClientFactory → Catalog Service
~~~

示例使用内存数据，只为演示服务边界和 HTTP 调用。生产环境应替换为每个服务自己管理的数据库。

#### 建立解决方案

需要安装当前受支持的 .NET SDK，并准备 YARP 包：

~~~shell
dotnet new sln -n ShopMicroservices
dotnet new web -n CatalogService
dotnet new web -n OrderService
dotnet new web -n ApiGateway

dotnet sln add CatalogService/CatalogService.csproj
dotnet sln add OrderService/OrderService.csproj
dotnet sln add ApiGateway/ApiGateway.csproj

dotnet add ApiGateway/ApiGateway.csproj package Yarp.ReverseProxy
~~~

本地运行时为三个项目分配不同端口：

~~~text
CatalogService：http://localhost:5101
OrderService：  http://localhost:5102
ApiGateway：    http://localhost:5000
~~~

也可以通过 launchSettings.json 设置端口。下面用 --urls 直接启动，减少模板配置干扰。

#### CatalogService：商品服务

替换 CatalogService/Program.cs：

~~~csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddHealthChecks();

var app = builder.Build();

var products = new Dictionary<int, Product>
{
    [1] = new(1, "机械键盘", 399m, 20),
    [2] = new(2, "无线鼠标", 129m, 50)
};

app.MapGet("/health/live", () => Results.Ok(new { status = "live" }));
app.MapGet("/health/ready", () => Results.Ok(new { status = "ready" }));

app.MapGet("/products/{id:int}", (int id) =>
    products.TryGetValue(id, out var product)
        ? Results.Ok(product)
        : Results.NotFound());

app.MapGet("/products", () => Results.Ok(products.Values));

app.Run();

public sealed record Product(int Id, string Name, decimal Price, int Stock);
~~~

商品服务拥有商品价格和库存展示数据。订单服务不应该直接连商品数据库，而是经由商品服务 API 获取必要信息。

#### OrderService：订单服务及下游 HTTP 调用

替换 OrderService/Program.cs：

~~~csharp
using System.Net;
using System.Net.Http.Json;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddHealthChecks();

var catalogUrl = builder.Configuration["CatalogUrl"]
    ?? "http://localhost:5101/";

builder.Services.AddHttpClient<CatalogClient>(client =>
{
    client.BaseAddress = new Uri(catalogUrl);
    client.Timeout = TimeSpan.FromSeconds(3);
});

var app = builder.Build();

app.MapGet("/health/live", () => Results.Ok(new { status = "live" }));
app.MapGet("/health/ready", () => Results.Ok(new { status = "ready" }));

app.MapPost("/orders", async (
    CreateOrderRequest request,
    CatalogClient catalog,
    CancellationToken cancellationToken) =>
{
    if (request.Quantity <= 0)
    {
        return Results.BadRequest(new { error = "数量必须大于 0" });
    }

    ProductDto? product;
    try
    {
        product = await catalog.GetProductAsync(
            request.ProductId,
            cancellationToken);
    }
    catch (HttpRequestException)
    {
        return Results.Problem(
            title: "商品服务暂时不可用",
            statusCode: StatusCodes.Status503ServiceUnavailable);
    }
    catch (TaskCanceledException) when (!cancellationToken.IsCancellationRequested)
    {
        return Results.Problem(
            title: "调用商品服务超时",
            statusCode: StatusCodes.Status504GatewayTimeout);
    }

    if (product is null)
    {
        return Results.NotFound(new { error = "商品不存在" });
    }

    if (product.Stock < request.Quantity)
    {
        return Results.Conflict(new { error = "库存不足" });
    }

    // 这里只演示请求校验。真实服务应将订单写入自己的数据库，
    // 并明确库存预占、重复请求和失败补偿策略。
    var order = new OrderResponse(
        Guid.NewGuid(),
        product.Id,
        product.Name,
        product.Price,
        request.Quantity,
        DateTimeOffset.UtcNow);

    return Results.Created($"/orders/{order.Id}", order);
});

app.Run();

public sealed record CreateOrderRequest(int ProductId, int Quantity);

public sealed record ProductDto(
    int Id,
    string Name,
    decimal Price,
    int Stock);

public sealed record OrderResponse(
    Guid Id,
    int ProductId,
    string ProductName,
    decimal UnitPrice,
    int Quantity,
    DateTimeOffset CreatedAt);

public sealed class CatalogClient(HttpClient httpClient)
{
    public async Task<ProductDto?> GetProductAsync(
        int id,
        CancellationToken cancellationToken)
    {
        using var response = await httpClient.GetAsync(
            $"products/{id}",
            cancellationToken);

        if (response.StatusCode == HttpStatusCode.NotFound)
        {
            return null;
        }

        response.EnsureSuccessStatusCode();

        return await response.Content.ReadFromJsonAsync<ProductDto>(
            cancellationToken: cancellationToken);
    }
}
~~~

这里用 typed client 把服务地址、超时和 HTTP 解析集中在 CatalogClient，而不是把 HttpClient 逻辑散落在每个 endpoint。CancellationToken 会把浏览器断开或上游取消继续传给下游调用。

订单响应保存下单时的商品名称和单价快照。真实系统中创建订单与预占库存之间还涉及并发、幂等和最终一致性，不能只靠读取 Stock 后再写订单来保证不超卖。

#### ApiGateway：统一 HTTP 入口

在 ApiGateway/appsettings.json 配置 YARP 路由：

~~~json
{
  "ReverseProxy": {
    "Routes": {
      "catalog-route": {
        "ClusterId": "catalog",
        "Match": {
          "Path": "/api/catalog/{**remainder}"
        },
        "Transforms": [
          {
            "PathRemovePrefix": "/api/catalog"
          }
        ]
      },
      "orders-route": {
        "ClusterId": "orders",
        "Match": {
          "Path": "/api/orders/{**remainder}"
        },
        "Transforms": [
          {
            "PathRemovePrefix": "/api"
          }
        ]
      }
    },
    "Clusters": {
      "catalog": {
        "Destinations": {
          "catalog-1": {
            "Address": "http://localhost:5101/"
          }
        }
      },
      "orders": {
        "Destinations": {
          "orders-1": {
            "Address": "http://localhost:5102/"
          }
        }
      }
    }
  }
}
~~~

替换 ApiGateway/Program.cs：

~~~csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

var app = builder.Build();

app.MapGet("/health/live", () => Results.Ok(new { status = "live" }));
app.MapReverseProxy();

app.Run();
~~~

路由转换结果：

~~~text
/api/catalog/products/1
    → http://localhost:5101/products/1

/api/orders
    → http://localhost:5102/orders
~~~

YARP 负责入口路由和代理，不应把订单业务规则、跨服务事务或数据库查询都塞进网关。

#### 启动和验证

分别打开三个终端：

~~~shell
dotnet run --project CatalogService --urls http://localhost:5101
~~~

~~~shell
dotnet run --project OrderService --urls http://localhost:5102
~~~

~~~shell
dotnet run --project ApiGateway --urls http://localhost:5000
~~~

查询商品：

~~~shell
curl http://localhost:5000/api/catalog/products/1
~~~

创建订单：

~~~shell
curl -X POST http://localhost:5000/api/orders -H "Content-Type: application/json" -d '{"productId":1,"quantity":2}'
~~~

订单服务会经由 CatalogClient 请求商品服务，成功时返回 201 Created 和订单快照。商品服务停掉时，订单服务会返回服务不可用或超时，而不是无限期等待。

检查健康端点：

~~~shell
curl http://localhost:5101/health/ready
curl http://localhost:5102/health/ready
curl http://localhost:5000/health/live
~~~

健康检查要区分：

* **Liveness**：进程是否还活着，失败时编排平台可能重启容器；
* **Readiness**：实例是否准备好接流量，失败时应暂时从流量中摘除。

生产中的 readiness 还要结合必要依赖状态，避免把“一个可选依赖暂时故障”变成所有实例都被摘除。

### 服务调用不能只写一个 URL

Demo 为了本机运行使用 localhost 固定地址。容器或集群部署时，容器里的 localhost 指向当前容器，不是另一项服务。服务地址应来自：

* 容器网络 DNS；
* Kubernetes Service 名称；
* .NET Aspire 注入的服务发现地址；
* Consul 等注册发现系统；
* 云平台提供的服务名称和配置。

不要把开发机 localhost 地址复制到生产配置。

ASP.NET Core 中推荐使用 IHttpClientFactory 或 typed client 管理 HttpClient 生命周期。不要每个请求都 new HttpClient，也不要把下游调用写成没有 timeout、取消传播和错误处理的无限等待。

### 超时、重试和熔断

网络调用默认有失败可能。基本防护包括：

* **超时**：限制单次调用最长等待时间；
* **重试**：只对适合重试的临时故障重试，并使用退避和抖动；
* **熔断**：下游持续失败时短时间停止调用，避免请求继续堆积；
* **舱壁/并发限制**：限制某个下游占用的并发资源；
* **降级**：对非关键功能返回缓存或部分结果。

不能对所有请求无脑重试。POST 已经成功但响应丢失时，客户端重试可能创建重复订单。涉及状态变更的接口需要幂等键或业务去重。

在现代 .NET 中，可以通过 Microsoft.Extensions.Http.Resilience 为 HttpClient 配置标准弹性策略；先确认当前目标框架与包版本，再按业务请求类型配置。重试次数、总超时和下游 deadline 需要一起计算，避免一层重试变成多层重试风暴。

### 数据一致性：不要跨服务假装有一个大事务

单体数据库中常用的一次事务，拆服务后通常无法横跨多个独立数据库。

一个订单流程可能包含：

~~~text
创建订单
→ 预占库存
→ 支付
→ 确认订单
~~~

中间任一步失败，都需要业务层定义状态和补偿：

* 库存预占成功、支付失败：释放库存；
* 支付成功、订单更新失败：通过重试或对账修复；
* 消息重复投递：消费者需要幂等；
* 数据短时间不同步：查询模型要能接受最终一致。

常见方案包括 Saga、Transactional Outbox、Inbox/去重表和定期对账。它们不是统一的“分布式事务开关”，而是围绕业务失败路径设计的流程。

Transactional Outbox 的典型做法是在一个本地数据库事务中同时写业务数据和待发送事件，再由后台任务可靠投递消息，避免“数据库写成功但消息没发出去”。

### .NET Aspire、Docker 和 Kubernetes 的关系

#### .NET Aspire

Aspire 是面向分布式应用的开发和编排工具，可以在 AppHost 中用代码描述项目、容器、依赖和服务发现关系，并提供本地运行诊断入口。

它主要改善本地多服务开发体验，不等同于生产集群，也不自动替代 Kubernetes、云部署流水线或生产级服务治理。

典型的 Aspire 关系可能写成：

~~~csharp
var catalog = builder.AddProject<Projects.CatalogService>("catalog");

builder.AddProject<Projects.OrderService>("orders")
    .WithReference(catalog);
~~~

引用关系可以让本地应用使用服务名发现 CatalogService，而不是把 localhost 端口写死。Aspire 模板和 API 会随版本演进，新增项目时应使用当前官方模板并按生成的项目结构配置。

#### Docker

Docker/OCI 镜像把应用和运行时环境封装起来，方便在开发、测试和部署环境保持一致。容器化是微服务常见部署方式，但不是微服务架构的定义条件。

#### Kubernetes

Kubernetes 负责容器调度、服务发现、健康探测、扩缩容和滚动更新等集群工作。它解决的是运行与编排问题，不会替代业务边界设计、数据库归属或幂等处理。

### 可观测性：分布式系统要能串起一条请求

单体应用中看一份日志可能就够；微服务的一次请求会经过多个进程，至少要建立三类信号：

* **日志**：发生了什么，包含结构化字段和 TraceId；
* **指标**：错误率、延迟、吞吐、队列深度和资源使用；
* **分布式追踪**：一个请求跨过哪些服务，每段耗时多少。

使用 OpenTelemetry 时，需要让 HTTP 客户端和 ASP.NET Core 请求自动传播 trace context，并导出到本地 Dashboard 或生产遥测平台。日志中应能找到 TraceId；不要把密码、Token、支付信息和完整个人数据放入日志。

本地开发可使用 Aspire Dashboard 或 OpenTelemetry Collector；生产环境要明确采样率、保留期、访问控制和告警规则。

### 认证、授权和网关边界

网关可以承担统一认证入口、TLS 终止、路由、限流和请求头处理，但不能成为唯一的安全边界。服务之间仍要验证调用身份和授权范围，特别是服务可以被集群内部其他工作负载直接访问时。

常见做法包括 OAuth 2.0 / OpenID Connect、JWT 验证、工作负载身份和 mTLS。网关不应仅凭“请求来自内网”就信任所有服务调用。

微服务之间的权限粒度应与业务操作匹配，例如“读取商品”与“创建订单”不是同一种权限。

### 微服务常见坑

#### 按技术层拆服务

Controller、Service、Repository 各自一个服务，会把进程内方法调用变成大量网络调用，延迟和故障点都增加。

#### 所有服务共用同一套业务表

只要服务可以直接访问别人的表，表结构就无法独立演进，部署自治也会被破坏。需要访问数据时应通过 API、事件复制或明确的读模型。

#### 同步调用链越来越长

网关调订单，订单调库存，库存再调用户和支付，最后一个下游慢会拖慢整个请求。缩短同步链，把不需要实时完成的动作移到消息流程。

#### 每个服务都复制一份巨大共享 DTO 项目

共享契约可以共享稳定的消息格式或公共基础设施类型，但不应把各服务内部领域模型合并成一套公共实体。共享模型改动会导致多个服务被迫同时升级。

#### 重试导致重复操作

重试会重新发送请求。创建订单、扣款和库存变更需要幂等键、去重机制和可查询的操作状态。

#### 把日志当作可观测性全部

没有指标无法看趋势，没有 TraceId 无法串联调用链。日志、指标、追踪需要相互补充。

#### 一开始就引入完整技术栈

网关、服务网格、Kubernetes、消息队列、注册中心、Saga 和多个数据库会同时增加团队负担。按真实需求逐项引入，先做好边界和可观测性。

### 什么时候不适合微服务

以下情况通常先采用模块化单体更合适：

* 团队规模小，服务无法由不同小组独立负责；
* 发布频率和扩容需求相近，没有明显的独立部署收益；
* 业务边界还在快速变化；
* 运维没有容器平台、告警、追踪和自动化发布能力；
* 拆分后每个请求都要同步访问多个服务；
* 项目主要问题是代码结构混乱，而不是部署和扩展受限。

微服务把进程边界变成网络边界，问题从编译期移动到运行期。拆分之前先把模块化、契约测试和依赖方向做好，后续迁移会轻得多。

### 总结

微服务不是把一个 Web 项目拆成多个端口，而是让业务能力、数据所有权和部署生命周期都形成清晰边界。

~~~text
客户端
  ↓
网关：路由、入口策略
  ↓
业务服务：拥有自己的规则和数据
  ↔
HTTP / gRPC / 消息
  ↓
日志 + 指标 + Trace
~~~

落地顺序可以保持务实：

1. 先按业务边界做模块化；
2. 找到真正需要独立部署、扩缩容或团队自治的模块；
3. 定义服务 API 和数据所有权；
4. 为网络调用设置超时、取消和弹性策略；
5. 为跨服务业务设计幂等、最终一致和补偿；
6. 接入可观测性后再扩展部署规模。

实战 Demo 已覆盖网关路由、服务间 HTTP 调用、健康检查和故障响应。YARP、gRPC、Consul、OpenTelemetry 和分布式事务的深入配置可分别参考项目中的专题文章。

参考资料：

* [Microsoft Learn：.NET 微服务架构指南](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/)
* [Microsoft Learn：微服务之间的通信](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/architect-microservice-container-applications/communication-in-microservice-architecture)
* [Microsoft Learn：微服务数据自治](https://learn.microsoft.com/en-us/dotnet/architecture/microservices/architect-microservice-container-applications/data-sovereignty-per-microservice)
* [Microsoft Learn：.NET Aspire 概览](https://learn.microsoft.com/en-us/dotnet/aspire/get-started/aspire-overview)
* [Microsoft Learn：构建第一个 Aspire 应用](https://learn.microsoft.com/en-us/dotnet/aspire/get-started/build-your-first-aspire-app)
* [Microsoft Learn：.NET HTTP 客户端指南](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient-guidelines)
* [Microsoft Learn：.NET 应用弹性](https://learn.microsoft.com/en-us/dotnet/core/resilience/)
