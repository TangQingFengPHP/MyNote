### 简介

只要你的 `.NET` 系统开始从单体走向微服务，很快就会遇到这个问题：

```text
订单写成功了
库存扣失败了
余额没扣
消息却发出去了
```

这时候你会发现，单库事务那套熟悉的心智模型开始失效了。

在单体应用里，你可以靠：

* `SqlTransaction`
* `DbTransaction`
* `TransactionScope`

把多步操作包成一个本地事务。

但当业务已经跨：

* 多个服务
* 多个数据库
* 多种存储
* 消息队列

本地事务就不够了。

这篇文章重点不是只讲概念，而是讲清楚几件事：

* 分布式事务到底在解决什么问题；
* 为什么它本质上是在“一致性、可用性、复杂度”之间做交易；
* 2PC、TCC、Saga、本地消息表、事务消息分别适合什么场景；
* `.NET` 里 `TransactionScope、MSDTC、CAP、DTM、MassTransit` 各自的边界是什么；
* 实战里到底该怎么选，而不是一上来就把所有系统都改成 TCC。

一句话先说透：

> 分布式事务不是“把数据库事务搬到微服务里”，而是为跨服务状态一致性设计的一整套补偿、协调和重试策略。

### 分布式事务到底是什么？

它解决的不是“数据库怎么提交”这么简单，而是：

* 一笔业务跨了多个参与方
* 每个参与方都可能单独成功或失败
* 但业务上又不能接受状态长期错乱

例如最典型的下单流程：

```text
订单服务：创建订单
库存服务：扣减库存
账户服务：扣减余额
积分服务：增加积分
```

如果只有一个数据库，本地事务很好办。

但一旦拆成多个服务，就会出现：

* A 成功
* B 失败
* C 超时
* D 根本没收到请求

于是分布式事务真正要回答的是：

> 当多个独立参与方不能共享一个本地数据库事务时，系统如何仍然把业务结果收敛到可接受的一致状态。

### 为什么本地事务那套在微服务里不够用了？

因为本地事务的前提通常是：

* 一个数据库连接
* 一个事务管理器
* 一组可一起提交或回滚的资源

而微服务会打破这个前提：

* 服务和数据库分开部署
* 资源管理器不同
* 网络调用可能超时
* 消息系统不是数据库事务资源

这意味着：

* 你很难再用“一次 `Commit` 全部成功，一次 `Rollback` 全部失败”的方式解决问题

于是系统就不得不进入另一种世界：

* 要么付出极高代价换强一致
* 要么接受最终一致并设计补偿

### 为什么大家总说“优先避免分布式事务”？

因为它的成本非常真实，而且不是只有代码复杂。

分布式事务的代价通常包括：

* 链路变长
* 失败模式变多
* 重试逻辑复杂
* 幂等要求上升
* 补偿语义难设计
* 监控和排障成本显著增加

所以更务实的工程原则通常是：

1. 能不能重新拆业务边界，避免跨服务强一致？
2. 能不能通过异步化把强一致需求降级成最终一致？
3. 只有真的不能降，才考虑 TCC、Saga 这类更重的方案。

这不是保守，而是成熟系统设计的基本判断。

### 先把几种主流方案的定位看清

可以先看这张压缩表：

| 方案 | 一致性 | 侵入性 | 性能/可用性 | 常见场景 |
| --- | --- | --- | --- | --- |
| `2PC / XA / DTC` | 强一致 | 中 | 较差 | 传统跨库事务、老系统 |
| `TCC` | 强一致倾向 | 很高 | 中 | 资金、库存冻结类业务 |
| `Saga` | 最终一致 | 中 | 较好 | 长流程业务、订单编排 |
| `Outbox / 本地消息表` | 最终一致 | 中低 | 很好 | 事件驱动、异步解耦 |
| `事务消息` | 最终一致 | 中 | 较好 | 依赖特定 MQ 能力 |

如果只记一句话：

> 大多数互联网微服务系统真正落地最多的，其实不是 2PC，而是 Saga、Outbox 和事务消息这类最终一致方案。

### 2PC / XA / DTC 是什么，为什么现在很少当默认答案？

2PC 的核心思路很直接：

* 第一阶段：所有参与者先 `Prepare`
* 第二阶段：协调者决定 `Commit` 或 `Rollback`

优点当然很明显：

* 强一致

但问题也同样明显：

* 协调成本高
* 阻塞明显
* 对参与资源要求高
* 高延迟和网络分区下表现差
* 云原生、多存储、多中间件环境里兼容性很差

在 `.NET` 里，大家最熟悉的相关入口通常是：

```csharp
TransactionScope
```

以及背后的：

```text
System.Transactions
MSDTC
```

但要特别注意一点：

> `TransactionScope` 很适合本地事务和部分受支持资源的协调，它不是“现代微服务跨一切资源的通用分布式事务银弹”。

对今天大多数云原生 `.NET` 微服务来说，直接把希望押在 `MSDTC` 上，通常并不是优先路径。

### `.NET` 里的 `TransactionScope` 到底该怎么理解？

这也是很多人最容易误会的地方。

`TransactionScope` 的价值是：

* 提供一种环境事务模型
* 让受支持的资源自动参与当前事务范围

它在这些场景里非常好用：

* 单库事务
* 同类资源且运行环境支持
* 传统企业系统

但你不能把它直接等同成：

* 微服务时代的完整分布式事务方案

尤其当你的链路里已经出现：

* HTTP 服务调用
* RabbitMQ / Kafka
* Redis
* 非 DTC 资源

这时候它的覆盖能力就远没有很多人想象得那么大。

### `TransactionScope` 最小集成流程：怎么配，怎么跑？

如果你只是想把它跑起来，建立最基本的环境事务心智模型，可以按这个最小流程来。

#### 第一步：准备两个 SQL Server 库

例如：

* `OrderDb`
* `BillingDb`

并在两个库里各建一张简单表：

```sql
create table Orders(Id int primary key, Amount decimal(18,2));
create table BillingLogs(Id int identity primary key, OrderId int, Amount decimal(18,2));
```

#### 第二步：写一个最小控制台程序或 API

```csharp
using System.Transactions;
using Microsoft.Data.SqlClient;

var orderConn = "Server=.;Database=OrderDb;Trusted_Connection=True;TrustServerCertificate=True";
var billingConn = "Server=.;Database=BillingDb;Trusted_Connection=True;TrustServerCertificate=True";

using var scope = new TransactionScope(TransactionScopeAsyncFlowOption.Enabled);

await using var conn1 = new SqlConnection(orderConn);
await conn1.OpenAsync();

await using var conn2 = new SqlConnection(billingConn);
await conn2.OpenAsync();

await new SqlCommand("insert into Orders(Id, Amount) values (1, 100)", conn1).ExecuteNonQueryAsync();
await new SqlCommand("insert into BillingLogs(OrderId, Amount) values (1, 100)", conn2).ExecuteNonQueryAsync();

scope.Complete();
```

#### 第三步：怎么验证？

故意在第二条 SQL 后抛一个异常：

```csharp
throw new Exception("boom");
```

然后确认：

* 两个库都没有落库

这能帮助你理解：

* `TransactionScope` 能协调的是“受支持资源”
* 它不是拿来直接包跨 HTTP 微服务调用的

#### 一个实践案例

最贴近它的真实案例往往不是微服务，而是：

* 一个后台任务
* 同时写两个 SQL Server 数据库
* 想保证两边一起提交或一起回滚

如果场景已经变成：

* 调远程库存服务
* 再发一条 MQ

那就已经超出这套模型的舒适区了。

### TCC 到底适合什么场景？

TCC 是：

```text
Try
Confirm
Cancel
```

它的核心思想不是直接提交，而是先预留资源。

例如订单扣库存和扣余额：

* `Try`：冻结库存、冻结余额
* `Confirm`：正式扣减
* `Cancel`：解冻资源

它最适合的业务形态通常是：

* 资源可以预留
* 资源可以确认
* 资源可以释放

这类业务最典型的是：

* 库存冻结
* 账户余额冻结
* 优惠券占用

它的优点是：

* 一致性强
* 业务语义清晰
* 对失败恢复控制力高

但代价也非常大：

* 每个参与方都要写 `Try / Confirm / Cancel`
* 幂等、空回滚、防悬挂必须设计
* 业务侵入性很强

所以它不是“高级一点就上”的方案，而是：

* 你真的需要更强一致性，并且业务能明确拆成三段时才值得用

### TCC 最小集成流程：怎么配，怎么跑？

如果你真要把 TCC 跑起来，通常至少要有：

* 一个事务协调器
* 每个参与服务的 `Try / Confirm / Cancel`
* 事务状态和幂等表

在 `.NET` 生态里，常见的一条落地路径是：

* `DTM` 做协调器
* ASP.NET Core 服务实现三组接口

#### 第一步：先起协调器

本地体验可以先起一个 `DTM(Distributed Transactions Manager)`：

```bash
docker run --rm -it -p 36789:36789 -p 36790:36790 yedf/dtm
```

#### 第二步：准备参与服务

例如准备：

* `inventory-service`
* `account-service`

库存服务最小接口长这样：

```csharp
app.MapPost("/inventory/try", async (FreezeStockRequest req, InventoryDb db) =>
{
    // 1. 校验库存
    // 2. 冻结库存
    // 3. 写 TCC 状态
    return Results.Ok();
});

app.MapPost("/inventory/confirm", async (FreezeStockRequest req, InventoryDb db) =>
{
    // 正式扣减库存
    return Results.Ok();
});

app.MapPost("/inventory/cancel", async (FreezeStockRequest req, InventoryDb db) =>
{
    // 解冻库存
    return Results.Ok();
});
```

#### 第三步：发起全局事务

示意代码：

```csharp
// 伪代码，重点是流程
var tcc = CreateTcc("http://localhost:36789");

tcc.AddBranch(
    tryUrl: "http://localhost:5001/inventory/try",
    confirmUrl: "http://localhost:5001/inventory/confirm",
    cancelUrl: "http://localhost:5001/inventory/cancel",
    payload: new { OrderId = "O1001", ProductId = 1, Count = 2 });

tcc.AddBranch(
    tryUrl: "http://localhost:5002/account/try",
    confirmUrl: "http://localhost:5002/account/confirm",
    cancelUrl: "http://localhost:5002/account/cancel",
    payload: new { OrderId = "O1001", UserId = 10, Amount = 100 });

await tcc.SubmitAsync();
```

#### 第四步：怎么验证？

最简单的方式是故意让一个分支的 `Try` 返回失败。

然后观察：

* 另一个已成功的 `Try` 是否收到 `Cancel`
* 状态表是否进入回滚态
* 重复执行 `Cancel` 时是否仍然幂等

#### 一个实践案例

最典型的就是：

* 下单时冻结库存
* 同时冻结余额
* 两边都成功才 `Confirm`
* 任一失败就都 `Cancel`

这类“资源先冻结、再正式扣减”的业务，才是 TCC 的舒适区。

### Saga 又是什么？

如果说 TCC 是“先预留再确认”，那 Saga 更像：

* 把一个长事务拆成多个本地事务
* 每个本地事务都配一个补偿动作

例如下单链路可以拆成：

1. 创建订单
2. 扣库存
3. 扣余额
4. 发优惠券

如果第三步失败，前面已经成功的步骤就通过补偿动作回退：

* 订单取消
* 库存回补

Saga 最大的优点在于：

* 更适合长流程
* 不要求所有参与方都做资源冻结
* 比 TCC 的业务侵入性低一些

但它也有天然代价：

* 只能做到最终一致
* 补偿不是数据库回滚，而是新的业务动作
* 业务语义设计不清会很痛苦

所以 Saga 更像：

* 长流程业务的一致性编排方案

而不是：

* 每个小事务都要上的通用模板

### Saga 还有编排和协同两种思路，怎么区分？

这也是实战里很容易被问到的一层。

#### 编排式 Saga

它更像有一个明确的协调者：

* 它知道当前流程走到哪一步
* 它决定下一步该调用谁
* 它在失败时决定回补哪些动作

这种方式的优点是：

* 流程集中
* 可观测性更强
* 更适合复杂业务链路

代价是：

* 协调器本身会更重
* 业务编排逻辑容易集中膨胀

#### 协同式 Saga

它更像事件接力：

* 订单服务发事件
* 库存服务消费后再发事件
* 账户服务继续消费再发事件

这种方式的优点是：

* 更松耦合
* 服务自治更强

代价是：

* 流程全貌更难追踪
* 排障更难
* 补偿链路更容易散

所以更务实的判断通常是：

* 流程清晰但步骤多，且想强控制，优先考虑编排
* 事件驱动架构成熟、团队对消息建模有经验，协同也可以成立

在 `.NET` 里，`MassTransit Saga State Machine` 更适合做有状态的 Saga 编排或半编排式工作流，这一点官方文档也明确强调了 Saga 的核心是状态持久化和事件驱动状态迁移。  

### Saga 最小集成流程：怎么配，怎么跑？

如果你要把 Saga 跑起来，最常见的一条 `.NET` 路线是：

* `MassTransit`
* `RabbitMQ`
* 一个 Saga 状态存储

#### 第一步：先起 RabbitMQ

```bash
docker run --rm -it -p 5672:5672 -p 15672:15672 rabbitmq:3-management
```

管理页面：

```text
http://localhost:15672
```

#### 第二步：安装常见包

```bash
dotnet add package MassTransit
dotnet add package MassTransit.RabbitMQ
dotnet add package MassTransit.EntityFrameworkCore
```

#### 第三步：定义 Saga 状态

```csharp
public class OrderState : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; } = string.Empty;
    public string OrderId { get; set; } = string.Empty;
}
```

#### 第四步：定义状态机

```csharp
public class OrderStateMachine : MassTransitStateMachine<OrderState>
{
    public State Submitted { get; private set; } = null!;
    public Event<OrderSubmitted> OrderSubmitted { get; private set; } = null!;
    public Event<StockReserved> StockReserved { get; private set; } = null!;
    public Event<StockReserveFailed> StockReserveFailed { get; private set; } = null!;

    public OrderStateMachine()
    {
        Initially(
            When(OrderSubmitted)
                .Then(ctx => ctx.Saga.OrderId = ctx.Message.OrderId)
                .Publish(ctx => new ReserveStock(ctx.Message.OrderId, ctx.Message.ProductId, ctx.Message.Count))
                .TransitionTo(Submitted));

        During(Submitted,
            When(StockReserved)
                .Publish(ctx => new ReserveBalance(ctx.Saga.OrderId, ctx.Message.UserId, ctx.Message.Amount))
                .Finalize(),

            When(StockReserveFailed)
                .Publish(ctx => new CancelOrder(ctx.Saga.OrderId))
                .Finalize());
    }
}
```

#### 第五步：接到 RabbitMQ 和持久化

```csharp
builder.Services.AddMassTransit(x =>
{
    x.AddSagaStateMachine<OrderStateMachine, OrderState>()
        .EntityFrameworkRepository(r =>
        {
            r.ExistingDbContext<OrderSagaDbContext>();
            r.UseSqlServer();
        });

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("localhost", "/", h =>
        {
            h.Username("guest");
            h.Password("guest");
        });

        cfg.ConfigureEndpoints(context);
    });
});
```

#### 第六步：怎么验证？

最小验证链路通常是：

1. 发一条 `OrderSubmitted`
2. 看是否发出了 `ReserveStock`
3. 人工发一条 `StockReserved` 或 `StockReserveFailed`
4. 看状态机是否推进或走补偿

#### 一个实践案例

最典型的是订单长流程：

* `OrderSubmitted`
* `StockReserved`
* `BalanceReserved`
* `OrderCompleted`
* `OrderCancelled`

Saga 管的是状态流转和补偿，不是数据库式回滚。

### 本地消息表 / Outbox 为什么是互联网系统最常见的落地方式之一？

因为它足够务实。

它的关键思想是：

* 本地业务数据变更
* 和待发送消息
* 放进同一个本地事务里

例如：

1. 创建订单
2. 向 `outbox` 表插入一条“订单已创建”消息
3. 一起提交本地事务
4. 后台投递器把 `outbox` 消息发到 MQ
5. 消费方再处理扣库存、积分等后续动作

它的好处非常现实：

* 不依赖跨服务强一致协议
* 可靠性高
* 性能和可用性通常更好
* 很适合事件驱动架构

但代价也要正视：

* 是最终一致，不是强一致
* 需要投递器、重试、死信、补偿
* 消费端必须做幂等

在 `.NET` 里，这条路线最常见的落地框架之一就是：

```text
DotNetCore.CAP
```

CAP 官方文档也明确强调，它不是基于 DTC/2PC 去做分布式事务，而是走本地消息表/Outbox 这条更适合分布式系统的路径。

### Outbox / 本地消息表最小集成流程：怎么配，怎么跑？

如果你要一个今天就能在项目里开始落地的模式，Outbox 往往是最务实的第一选择。

在 `.NET` 里，常见路径之一就是：

* `DotNetCore.CAP`

#### 第一步：先起数据库和 RabbitMQ

RabbitMQ：

```bash
docker run --rm -it -p 5672:5672 -p 15672:15672 rabbitmq:3-management
```

数据库就用你项目当前已经在用的 SQL Server 或 MySQL 即可。

#### 第二步：安装包

```bash
dotnet add package DotNetCore.CAP
dotnet add package DotNetCore.CAP.SqlServer
dotnet add package DotNetCore.CAP.RabbitMQ
```

#### 第三步：配置 CAP

```csharp
builder.Services.AddCap(x =>
{
    x.UseSqlServer("Server=.;Database=OrderDb;Trusted_Connection=True;TrustServerCertificate=True");
    x.UseRabbitMQ(options =>
    {
        options.HostName = "localhost";
        options.UserName = "guest";
        options.Password = "guest";
    });

    x.UseDashboard();
});
```

#### 第四步：把本地事务和发事件放一起

```csharp
app.MapPost("/orders", async (CreateOrderRequest req, OrderDbContext db, ICapPublisher capBus) =>
{
    await using var transaction = await db.Database.BeginTransactionAsync(capBus, autoCommit: false);

    var order = new Order
    {
        Id = Guid.NewGuid(),
        UserId = req.UserId,
        Amount = req.Amount
    };

    db.Orders.Add(order);
    await db.SaveChangesAsync();

    await capBus.PublishAsync("order.created", new
    {
        OrderId = order.Id,
        order.UserId,
        order.Amount
    });

    await transaction.CommitAsync();
    return Results.Ok(order.Id);
});
```

#### 第五步：消费端订阅

```csharp
public class InventorySubscriber : ICapSubscribe
{
    [CapSubscribe("order.created")]
    public async Task Handle(dynamic message)
    {
        // 扣库存
        await Task.CompletedTask;
    }
}
```

#### 第六步：怎么验证？

最小验证顺序通常是：

1. 调用创建订单接口
2. 确认订单表落库
3. 确认 CAP 相关消息表有记录
4. 确认消费者收到 `order.created`
5. 故意让消费端抛异常，看重试是否生效

#### 一个实践案例

最典型的是：

* 订单服务写订单
* 发出 `order.created`
* 库存服务扣库存
* 积分服务加积分
* 通知服务发短信

这些动作允许短暂延迟，因此非常适合 Outbox。

### 一个更像生产环境的 Outbox 表结构应该怎么设计？

如果你真的要落地 Outbox，通常不能只有一个“消息内容”字段就结束。

更务实的表结构一般至少要覆盖这些信息：

```sql
create table outbox_messages
(
    id uniqueidentifier primary key,
    message_type nvarchar(200) not null,
    business_key nvarchar(100) null,
    payload nvarchar(max) not null,
    headers nvarchar(max) null,
    status int not null,
    retry_count int not null default 0,
    next_retry_time datetime2 null,
    created_at datetime2 not null,
    sent_at datetime2 null,
    last_error nvarchar(2000) null
);
```

这里最关键的不是字段名，而是背后的几个设计点：

* `id`：消息唯一标识
* `business_key`：帮助你做业务幂等和排障定位
* `status`：至少区分待发送、已发送、失败、死信
* `retry_count` / `next_retry_time`：支持延迟重试
* `last_error`：失败后能回溯原因

很多 Outbox 方案最后不是死在原理上，而是死在：

* 表结构太简陋
* 失败态不可追踪
* 无法人工补偿和重放

所以表结构设计本身就是方案的一部分。

### 幂等键、补偿表、事务状态表，为什么最好一开始就设计？

因为分布式事务真正落地时，最怕的不是一次失败，而是：

* 失败后不知道自己执行到了哪
* 重试后不知道是不是重复执行
* 补偿时不知道是不是已经补过

更实战一点说，通常至少要有下面这些“状态承载点”：

#### 1. 幂等记录

用于判断：

* 这条消息我是不是已经消费过
* 这次 Confirm / Cancel 是不是已经做过

典型字段包括：

* 幂等键
* 处理状态
* 首次处理时间
* 最后处理结果

#### 2. 事务状态表

用于判断：

* 这笔业务当前是在 `Try`、`Confirm`、`Cancel`
* 还是在 Saga 的哪个步骤
* 当前是否已经终态

#### 3. 补偿记录

用于判断：

* 某个补偿动作是否已经执行
* 执行结果如何
* 是否允许再次补偿

所以更稳的工程习惯是：

> 不要把幂等和补偿当成“后面再补”，而要当成事务方案设计的第一层结构。

### 事务消息又是什么？

事务消息可以理解成：

* 消息中间件直接参与“消息是否真正可见”的判定

它的目标和本地消息表很像：

* 既不想丢消息
* 又不想让业务提交和消息发送完全脱节

它通常依赖特定 MQ 的能力，因此：

* 优点是集成度更高
* 缺点是更依赖消息中间件生态

所以工程上常见的判断是：

* 如果你已经用了支持事务消息的 MQ，并且团队能驾驭它，可以考虑
* 如果没有，Outbox 往往更通用、更容易控制

### 事务消息最小集成流程：怎么配，怎么跑？

这部分要先说前提：

* 它强依赖具体 MQ 产品能力
* 不像 Outbox 那样有一套几乎通用的落地方式

所以更适合把它理解成：

1. 先发一条半消息
2. 执行业务本地事务
3. 根据本地事务结果提交或回滚消息
4. 消费端只会看到已提交消息

#### 第一步：准备支持事务消息的 MQ

例如 RocketMQ 这类支持事务消息的产品。

#### 第二步：定义本地事务执行器

示意代码：

```csharp
public class OrderTransactionListener
{
    public async Task<TransactionState> ExecuteLocalTransaction(CreateOrderMessage message)
    {
        // 1. 写订单
        // 2. 更新业务状态
        // 3. 成功则 Commit，失败则 Rollback
        return TransactionState.Commit;
    }

    public async Task<TransactionState> CheckLocalTransaction(string messageId)
    {
        // MQ 回查本地事务状态
        return TransactionState.Commit;
    }
}
```

#### 第三步：发送事务消息

```csharp
// 伪代码，重点是流程
await producer.SendTransactionalMessageAsync(
    topic: "order-created",
    payload: new { OrderId = "O1001", UserId = 10, Amount = 100 });
```

#### 第四步：怎么验证？

最简单的方式是：

* 本地事务成功时，消费者能看到消息
* 本地事务失败时，消费者看不到消息
* 故意让事务状态不明确，观察 MQ 的回查逻辑是否生效

#### 一个实践案例

最典型的是：

* 订单服务创建订单
* 订单成功后才让“扣库存事件”真正可见
* 库存服务再去消费

如果团队对具体 MQ 的事务能力不熟，优先级通常仍然低于 Outbox。

### `.NET` 里几种常见框架应该怎么理解？

这是实战里最容易问到的一组。

#### `TransactionScope` / `System.Transactions`

更偏：

* 本地事务
* 传统受支持资源的协调
* 经典企业应用模型

#### `DotNetCore.CAP`

更偏：

* Outbox / 本地消息表
* 可靠事件发布
* 最终一致

#### `DTM`

更偏：

* TCC
* Saga
* 编排型分布式事务协调

#### `MassTransit Saga`

更偏：

* 消息驱动工作流
* 状态机式 Saga
* 长流程补偿编排

所以别把它们混成一类：

* 有的是事务范围 API
* 有的是消息一致性框架
* 有的是事务协调器
* 有的是工作流/状态机基础设施

### 如果按“今天就能跑起来”的角度看，哪种门槛最低？

可以直接按集成门槛来理解：

* 最容易先落地：`TransactionScope`、`CAP / Outbox`
* 中等门槛：`MassTransit Saga`
* 最高门槛：`DTM + TCC`、事务消息

这个判断背后的原因很现实：

* 有的只是先搭起基础设施就能跑
* 有的则要求你先把业务语义、补偿模型、状态表、幂等表全部想清楚

### 一个更贴近项目的案例怎么选型？

以最典型的电商下单为例：

```text
创建订单
-> 扣库存
-> 扣余额
-> 发优惠券
-> 发通知
```

更务实的选型通常不是“一把梭 TCC”，而是分层判断：

#### 情况一：库存和余额必须非常谨慎控制

如果业务不能接受：

* 已经显示下单成功
* 结果库存和余额状态长时间不一致

那库存/账户这两步更可能需要：

* TCC
* 或者更严格的资源冻结方案

#### 情况二：通知、积分、营销动作允许延后

这类步骤通常更适合：

* Outbox + MQ
* CAP

因为它们天然允许：

* 稍后到达
* 稍后重试
* 最终一致

#### 情况三：整条链路特别长

如果整个业务已经变成：

* 订单
* 支付
* 发货
* 物流
* 售后

那更像 Saga 的战场，而不是局部小事务。

### 一个更完整的电商下单链路，可以怎么落地？

可以把一笔订单拆成三层一致性要求：

#### 第一层：必须尽量强的资源动作

例如：

* 冻结库存
* 冻结余额

这些更适合：

* TCC
* 或者至少明确冻结/解冻语义

#### 第二层：订单主状态流转

例如：

* 订单已创建
* 支付中
* 支付成功
* 扣库存成功
* 已完成
* 已取消

这更像：

* Saga 状态机

如果用 `MassTransit Saga`，更像是：

* 每收到一个业务事件，就推进一次订单状态
* 状态机实例持续保存当前状态
* 某一步失败时转入补偿路径或取消路径

MassTransit 官方文档也明确要求 Saga 需要持久化状态，否则后续事件就无法继续关联到同一个长事务实例。  

#### 第三层：允许延后的外围动作

例如：

* 发积分
* 发优惠券
* 发短信通知
* 发站内信

这些通常最适合：

* Outbox + MQ

因为它们天然允许：

* 稍后执行
* 失败重试
* 对主链路降级处理

所以一笔订单在真正成熟的系统里，往往不是只选一种模式，而是：

* 核心资源动作更严格
* 主状态流转用 Saga 管
* 外围通知营销走 Outbox

这比“一把梭统一方案”更符合实际。

### 幂等、补偿、重试，为什么它们比“选哪种模式”还重要？

因为分布式事务真正落地时，失败从来不是例外，而是常态。

你真正会面对的是这些问题：

* 消息重复投递
* 请求超时但对方其实成功了
* Cancel 比 Try 先到
* 补偿动作执行多次
* 消费者重启后重复消费

所以无论你选：

* TCC
* Saga
* Outbox
* 事务消息

最后都逃不开这三件事：

* 幂等
* 补偿
* 重试

如果这三件事没设计好，方案名字再高级也没用。

### 分布式事务最常见的误区有哪些？

#### 1. 一上来就想强一致

这通常会把系统拖进不必要的复杂度。

#### 2. 把 `TransactionScope` 当成微服务银弹

这在今天的大多数云原生系统里都不现实。

#### 3. 只设计主流程，不设计补偿

补偿不是边角料，而是主流程的一半。

#### 4. 忽略幂等

重试存在，就一定要正视幂等。

#### 5. 用消息表了，却没有可靠投递和失败兜底

这会让 Outbox 只剩半套。

### 一个非常务实的选择顺序

如果你在做 `.NET` 分布式事务选型，可以先按这个顺序判断：

1. 能不能通过业务拆分避免跨服务强一致？
2. 如果不能，哪些步骤必须强一致，哪些步骤可以最终一致？
3. 能最终一致的，优先考虑 Outbox / 事务消息
4. 长流程业务优先考虑 Saga
5. 只有在资源冻结语义明确、且强一致要求确实高时，再考虑 TCC
6. 不要把 `TransactionScope` 默认等同成微服务分布式事务方案

这个顺序很重要。

因为很多系统失败，不是失败在“没选高级框架”，而是失败在一开始就把所有步骤都当成同一种一致性要求。

### 面试里怎么答比较到位？

如果面试官问：

“分布式事务有哪些方案，怎么选？”

一个比较稳的回答可以是：

> 分布式事务本质是在跨服务、跨数据库、跨消息系统时保证业务状态一致。主流方案大致分成强一致和最终一致两类。2PC / DTC 更偏传统强一致，但性能和兼容性都不适合现代微服务默认使用；TCC 适合资源可冻结、对一致性要求高的业务；Saga 适合长流程补偿；Outbox / 本地消息表和事务消息更适合互联网系统里的最终一致落地。在 .NET 里，TransactionScope 更偏本地事务和传统协调，CAP 更偏 Outbox，DTM 更偏 TCC/Saga，MassTransit 更偏消息驱动 Saga。

如果继续追问“为什么很多系统更喜欢 Outbox 而不是 2PC”，可以答：

> 因为分布式系统里可用性和性能通常比理论上的强一致更重要。Outbox 把一致性问题收敛到本地事务里，再通过可靠消息和幂等消费实现最终一致，整体更适合微服务的失败模型。

如果再追问“最大的难点是什么”，优先答这三个：

* 幂等
* 补偿语义设计
* 重试与失败可观测性

如果继续追问“Saga 编排和协同怎么选”，可以补一句：

> 编排更适合流程复杂、想集中控制的业务；协同更适合事件驱动、服务自治更强的系统。但协同的排障和全链路可观测性通常更难，很多团队低估了这一点。

如果追问“Outbox 设计里最容易漏什么”，可以答：

> 不是少一个字段那么简单，而是很多实现缺少状态、重试时间、最后错误、业务键和人工补偿入口，结果一到线上失败就只能人工查日志。

如果追问“为什么幂等比模式选择还重要”，可以答：

> 因为无论 TCC、Saga 还是 Outbox，都绕不过重试、重复消费和补偿重放。模式只是框架，幂等才是这些模式能不能活下来的地基。

### 总结

分布式事务最值得记住的，不是那几个方案名，而是这句话：

> 它解决的不是“怎么把数据库事务搬远一点”，而是“跨多个参与方后，业务状态怎么在失败和重试中仍然收敛到正确结果”。

如果你只想记住几句话，可以记这几条：

* 能避免分布式事务，就优先避免；
* 大多数微服务系统真正常用的是最终一致，不是 2PC；
* TCC 适合资源冻结类强一致业务，Saga 适合长流程补偿；
* Outbox / 本地消息表是很务实的主流方案；
* 不管选哪种方案，幂等、补偿、重试和监控都不能省。

---

参考资料：

* Microsoft Learn: `System.Transactions` / `TransactionScope`
* CAP official docs: Transactions / Outbox pattern
* DTM official docs: TCC / Saga
* MassTransit official docs: Saga
