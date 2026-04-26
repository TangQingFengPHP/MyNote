### 简介

很多业务系统刚开始都差不多：

* 一个 `Service`
* 里面既有新增、修改、删除
* 也有列表、详情、统计、搜索

前期这样写很顺手。

但只要项目稍微复杂一点，问题就会慢慢冒出来：

* 读逻辑和写逻辑揉在一起
* 一个接口改动，连带影响一堆查询
* 写入要走校验、事务、领域规则
* 查询却只想尽快把数据拿出来
* 列表页、报表页、详情页要的数据结构都不一样

这时候继续把所有东西都塞进同一个模型里，代码很容易越长越拧巴。

`CQRS` 就是拿来解决这类问题的。

一句话先说透：

> `CQRS` 的核心不是“上大架构”，而是把写操作和读操作分开对待，因为这两件事本来就不是一回事。

这篇文章重点会讲清楚：

* `CQRS` 到底是什么
* 它为什么不等于“必须分库分表”
* 在 `.NET` 里通常怎么落地
* 怎样写一个像样的 `Command / Query / Handler` Demo
* 它和 `CRUD`、`MediatR`、读写分离、事件驱动之间的边界是什么

### `CQRS` 到底是什么？

`CQRS` 全称是：

```text
Command Query Responsibility Segregation
```

翻成大白话就是：

* 改数据的操作，归一类
* 查数据的操作，归另一类

这里有两个核心概念：

#### 1. Command

`Command` 是命令，负责改变系统状态。

比如：

* 创建订单
* 修改商品价格
* 取消订单
* 审核通过

它的重点是“做一件会产生副作用的事”。

#### 2. Query

`Query` 是查询，负责读取数据，但不修改系统状态。

比如：

* 查订单详情
* 查用户列表
* 查销售报表
* 查商品库存

它的重点是“把数据拿出来”，而不是“顺手更新点什么”。

所以 `CQRS` 最基本的规则其实很简单：

* 查询只负责查
* 命令只负责改

### 它为什么会出现？

因为传统 `CRUD` 写法，太容易把读和写混成一团。

先看一个很常见的服务类：

```csharp
public class OrderService
{
    public async Task<int> CreateAsync(CreateOrderRequest request) { ... }
    public async Task CancelAsync(int orderId) { ... }
    public async Task<OrderDto?> GetByIdAsync(int orderId) { ... }
    public async Task<List<OrderDto>> GetListAsync(OrderFilter filter) { ... }
    public async Task<decimal> GetSalesAmountAsync(DateOnly day) { ... }
}
```

看起来没什么问题。

但项目一大，这个类通常会变成这样：

* 创建订单要校验库存、校验状态、写事务
* 查询订单详情要关联多个表
* 列表页要分页、筛选、排序
* 报表页要聚合、统计、缓存

也就是说：

* 写操作更关注业务规则、一致性、事务
* 读操作更关注性能、结构、展示、缓存

两边目标完全不一样。

这就是 `CQRS` 的出发点：

> 既然读和写关注点不同，那就别再强行共用一套思路。

### `CQRS` 不等于“必须两套数据库”

这是最容易误解的一点。

很多人一听 `CQRS`，脑子里立刻出现：

* 写库一套
* 读库一套
* 消息队列一套
* 最终一致性一套

这其实是“完整形态的 CQRS”，不是起步门槛。

`CQRS` 可以分成两种理解：

#### 1. 简化版 CQRS

* `Command` 和 `Query` 在代码层分开
* 处理器分开
* 模型分开
* 但底层还是同一个数据库

这是最常见、也最实用的落地方式。

#### 2. 完整版 CQRS

* 写入走写模型、写库
* 读取走读模型、读库
* 中间通过事件或消息同步
* 读端和写端可能最终一致，而不是强一致

这是更重的一套架构，适合读写压力差异明显、业务复杂度高的系统。

> `CQRS` 首先是一种代码和职责拆分方式，分库分表只是它在某些场景下的进一步演化。

### 最简单的理解方式

可以把 `CQRS` 想成一条很直白的分界线：

#### 写操作这一侧关心什么？

* 参数是否合法
* 当前状态能不能改
* 业务规则是否满足
* 是否需要事务
* 是否需要发布领域事件

#### 读操作这一侧关心什么？

* 查得快不快
* 数据结构是不是刚好够页面用
* 是否需要分页、排序、过滤
* 是否需要缓存
* 是否要跨表拼装

这样一拆，就能看出差别了：

* 写操作像“处理业务命令”
* 读操作像“组织结果数据”

### 一个订单模块的传统写法

先看不拆分时常见的样子：

```csharp
public class OrderService
{
    private readonly AppDbContext _db;

    public OrderService(AppDbContext db)
    {
        _db = db;
    }

    public async Task<int> CreateAsync(CreateOrderRequest request)
    {
        // 校验、扣库存、写订单、写明细
        // SaveChanges
        return 1;
    }

    public async Task CancelAsync(int orderId)
    {
        // 查订单、判断状态、修改状态
    }

    public async Task<OrderDetailDto?> GetDetailAsync(int orderId)
    {
        // Include 多表查询
        return null;
    }

    public async Task<PagedResult<OrderListItemDto>> SearchAsync(OrderSearchRequest request)
    {
        // 筛选、分页、排序、投影
        return new PagedResult<OrderListItemDto>();
    }
}
```

这类写法的问题不是不能用，而是很容易越写越臃肿。

尤其是下面这些情况：

* 查询越来越多，而且每个页面要的数据都不一样
* 写操作越来越重，而且规则越来越多
* 接口层想做统一日志、校验、事务
* 同一个服务被很多接口反复调用，改起来牵一发动全身

### 换成 `CQRS` 后会变成什么样？

最基础的拆法一般是这样：

```text
Controller
   ├── Command -> CommandHandler
   └── Query   -> QueryHandler
```

也就是：

* 创建订单，走 `CreateOrderCommand`
* 取消订单，走 `CancelOrderCommand`
* 查订单详情，走 `GetOrderDetailQuery`
* 查订单列表，走 `GetOrderListQuery`

每个请求对象只表达一件事，每个处理器也只做一件事。

### Demo：先定义领域对象和数据库上下文

先准备一个足够简单、能说明问题的订单模型。

```csharp
public enum OrderStatus
{
    Pending = 1,
    Paid = 2,
    Cancelled = 3
}

public class Order
{
    public int Id { get; set; }
    public string OrderNo { get; set; } = string.Empty;
    public int CustomerId { get; set; }
    public decimal TotalAmount { get; set; }
    public OrderStatus Status { get; set; }
    public DateTime CreatedAt { get; set; }
    public DateTime? CancelledAt { get; set; }

    public void Cancel()
    {
        if (Status == OrderStatus.Cancelled)
        {
            throw new InvalidOperationException("订单已经取消，不能重复取消。");
        }

        if (Status == OrderStatus.Paid)
        {
            throw new InvalidOperationException("已支付订单不能直接取消。");
        }

        Status = OrderStatus.Cancelled;
        CancelledAt = DateTime.UtcNow;
    }
}

public class AppDbContext : DbContext
{
    public DbSet<Order> Orders => Set<Order>();

    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
    {
    }
}
```

这里先不铺太多领域建模细节，重点是把读写职责分开。

### Demo：定义 Command

先看写操作。

#### 创建订单命令

```csharp
public sealed record CreateOrderCommand(
    int CustomerId,
    decimal TotalAmount
);
```

#### 取消订单命令

```csharp
public sealed record CancelOrderCommand(int OrderId);
```

`Command` 的命名有个很重要的特点：

* 它表达的是意图
* 不是表达“某个通用动作”

所以通常会写成：

* `CreateOrderCommand`
* `CancelOrderCommand`
* `ApproveInvoiceCommand`

而不是模糊地写成：

* `SaveOrderCommand`
* `UpdateDataCommand`

因为业务意图越明确，代码越不容易拧巴。

### Demo：定义 Query

再看查询。

#### 查订单详情

```csharp
public sealed record GetOrderDetailQuery(int OrderId);
```

#### 查订单列表

```csharp
public sealed record GetOrderListQuery(
    int CustomerId,
    int PageNumber,
    int PageSize
);
```

读操作的重点通常不是“实体本身”，而是“页面真正需要什么结构”。

所以查询端往往不会直接返回 `Order` 实体，而是返回专门的 `DTO`。

### Demo：定义查询返回模型

```csharp
public sealed class OrderDetailDto
{
    public int Id { get; set; }
    public string OrderNo { get; set; } = string.Empty;
    public int CustomerId { get; set; }
    public decimal TotalAmount { get; set; }
    public string StatusText { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
}

public sealed class OrderListItemDto
{
    public int Id { get; set; }
    public string OrderNo { get; set; } = string.Empty;
    public decimal TotalAmount { get; set; }
    public string StatusText { get; set; } = string.Empty;
    public DateTime CreatedAt { get; set; }
}

public sealed class PagedResult<T>
{
    public int TotalCount { get; set; }
    public List<T> Items { get; set; } = new();
}
```

注意这里的思路：

* 写端用实体和规则保证数据正确
* 读端用 `DTO` 直接面向展示和返回结果

这就是 `CQRS` 很关键的一层拆分。

### Demo：不用任何库，先定义自己的 Handler 接口

为了把核心思路讲清楚，先不急着上 `MediatR`，直接定义最简单的接口。

```csharp
public interface ICommandHandler<TCommand, TResult>
{
    Task<TResult> HandleAsync(TCommand command, CancellationToken cancellationToken);
}

public interface ICommandHandler<TCommand>
{
    Task HandleAsync(TCommand command, CancellationToken cancellationToken);
}

public interface IQueryHandler<TQuery, TResult>
{
    Task<TResult> HandleAsync(TQuery query, CancellationToken cancellationToken);
}
```

这样更容易看出 `CQRS` 的本质：

* 先有命令和查询
* 再有各自的处理器
* `MediatR` 只是调度器，不是 `CQRS` 本身

### Demo：创建订单处理器

```csharp
public sealed class CreateOrderCommandHandler
    : ICommandHandler<CreateOrderCommand, int>
{
    private readonly AppDbContext _db;

    public CreateOrderCommandHandler(AppDbContext db)
    {
        _db = db;
    }

    public async Task<int> HandleAsync(
        CreateOrderCommand command,
        CancellationToken cancellationToken)
    {
        if (command.TotalAmount <= 0)
        {
            throw new ArgumentException("订单金额必须大于 0。");
        }

        var order = new Order
        {
            OrderNo = $"ORD{DateTime.UtcNow:yyyyMMddHHmmssfff}",
            CustomerId = command.CustomerId,
            TotalAmount = command.TotalAmount,
            Status = OrderStatus.Pending,
            CreatedAt = DateTime.UtcNow
        };

        _db.Orders.Add(order);
        await _db.SaveChangesAsync(cancellationToken);

        return order.Id;
    }
}
```

这里能看出写处理器的典型特点：

* 做参数校验
* 创建或修改实体
* 持久化
* 必要时放进事务

### Demo：取消订单处理器

```csharp
public sealed class CancelOrderCommandHandler
    : ICommandHandler<CancelOrderCommand>
{
    private readonly AppDbContext _db;

    public CancelOrderCommandHandler(AppDbContext db)
    {
        _db = db;
    }

    public async Task HandleAsync(
        CancelOrderCommand command,
        CancellationToken cancellationToken)
    {
        var order = await _db.Orders
            .FirstOrDefaultAsync(x => x.Id == command.OrderId, cancellationToken);

        if (order is null)
        {
            throw new KeyNotFoundException("订单不存在。");
        }

        order.Cancel();
        await _db.SaveChangesAsync(cancellationToken);
    }
}
```

这里的重点不是“改字段”，而是“执行业务意图”。

所以取消订单不应该只是：

```csharp
order.Status = OrderStatus.Cancelled;
```

而应该通过领域方法统一约束规则。

### Demo：查询订单详情处理器

```csharp
public sealed class GetOrderDetailQueryHandler
    : IQueryHandler<GetOrderDetailQuery, OrderDetailDto?>
{
    private readonly AppDbContext _db;

    public GetOrderDetailQueryHandler(AppDbContext db)
    {
        _db = db;
    }

    public async Task<OrderDetailDto?> HandleAsync(
        GetOrderDetailQuery query,
        CancellationToken cancellationToken)
    {
        return await _db.Orders
            .AsNoTracking()
            .Where(x => x.Id == query.OrderId)
            .Select(x => new OrderDetailDto
            {
                Id = x.Id,
                OrderNo = x.OrderNo,
                CustomerId = x.CustomerId,
                TotalAmount = x.TotalAmount,
                StatusText = x.Status.ToString(),
                CreatedAt = x.CreatedAt
            })
            .FirstOrDefaultAsync(cancellationToken);
    }
}
```

查询处理器和命令处理器的味道明显不一样：

* 基本不改状态
* 一般会用 `AsNoTracking()`
* 更关注投影和返回结构

### Demo：查询订单列表处理器

```csharp
public sealed class GetOrderListQueryHandler
    : IQueryHandler<GetOrderListQuery, PagedResult<OrderListItemDto>>
{
    private readonly AppDbContext _db;

    public GetOrderListQueryHandler(AppDbContext db)
    {
        _db = db;
    }

    public async Task<PagedResult<OrderListItemDto>> HandleAsync(
        GetOrderListQuery query,
        CancellationToken cancellationToken)
    {
        IQueryable<Order> source = _db.Orders
            .AsNoTracking()
            .Where(x => x.CustomerId == query.CustomerId);

        int totalCount = await source.CountAsync(cancellationToken);

        List<OrderListItemDto> items = await source
            .OrderByDescending(x => x.CreatedAt)
            .Skip((query.PageNumber - 1) * query.PageSize)
            .Take(query.PageSize)
            .Select(x => new OrderListItemDto
            {
                Id = x.Id,
                OrderNo = x.OrderNo,
                TotalAmount = x.TotalAmount,
                StatusText = x.Status.ToString(),
                CreatedAt = x.CreatedAt
            })
            .ToListAsync(cancellationToken);

        return new PagedResult<OrderListItemDto>
        {
            TotalCount = totalCount,
            Items = items
        };
    }
}
```

这段代码很好地体现了 `CQRS` 里读模型的特点：

* 不一定返回实体
* 不一定关心全部字段
* 查询结构直接为页面服务

### Demo：在 Controller 里怎么调用？

如果先不引入 `MediatR`，可以直接注入各自处理器。

```csharp
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> Create(
        [FromBody] CreateOrderCommand command,
        [FromServices] CreateOrderCommandHandler handler,
        CancellationToken cancellationToken)
    {
        int orderId = await handler.HandleAsync(command, cancellationToken);
        return Ok(orderId);
    }

    [HttpPost("{id}/cancel")]
    public async Task<IActionResult> Cancel(
        int id,
        [FromServices] CancelOrderCommandHandler handler,
        CancellationToken cancellationToken)
    {
        await handler.HandleAsync(new CancelOrderCommand(id), cancellationToken);
        return NoContent();
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetDetail(
        int id,
        [FromServices] GetOrderDetailQueryHandler handler,
        CancellationToken cancellationToken)
    {
        var result = await handler.HandleAsync(new GetOrderDetailQuery(id), cancellationToken);
        return result is null ? NotFound() : Ok(result);
    }

    [HttpGet]
    public async Task<IActionResult> GetList(
        [FromQuery] int customerId,
        [FromQuery] int pageNumber,
        [FromQuery] int pageSize,
        [FromServices] GetOrderListQueryHandler handler,
        CancellationToken cancellationToken)
    {
        var result = await handler.HandleAsync(
            new GetOrderListQuery(customerId, pageNumber, pageSize),
            cancellationToken);

        return Ok(result);
    }
}
```

这样一来，每个入口动作都非常清晰：

* 这个接口到底是命令还是查询
* 它交给哪个处理器
* 它返回什么结构

### `.NET` 里为什么经常会配 `MediatR`？

因为直接在控制器里注入很多处理器，随着功能增长会越来越散。

这时候通常会引入 `MediatR` 做请求分发。

先把定位说清楚：

> `MediatR` 不是 `CQRS`，它只是 `.NET` 生态里很适合拿来承接 `Command / Query` 调度的一层工具。

它擅长做的事是：

* 统一发送请求
* 自动匹配处理器
* 串上日志、校验、事务等管道行为

### 用 `MediatR` 改造后的写法

先定义命令：

```csharp
using MediatR;

public sealed record CreateOrderCommand(
    int CustomerId,
    decimal TotalAmount
) : IRequest<int>;
```

然后定义处理器：

```csharp
using MediatR;

public sealed class CreateOrderCommandHandler
    : IRequestHandler<CreateOrderCommand, int>
{
    private readonly AppDbContext _db;

    public CreateOrderCommandHandler(AppDbContext db)
    {
        _db = db;
    }

    public async Task<int> Handle(
        CreateOrderCommand request,
        CancellationToken cancellationToken)
    {
        if (request.TotalAmount <= 0)
        {
            throw new ArgumentException("订单金额必须大于 0。");
        }

        var order = new Order
        {
            OrderNo = $"ORD{DateTime.UtcNow:yyyyMMddHHmmssfff}",
            CustomerId = request.CustomerId,
            TotalAmount = request.TotalAmount,
            Status = OrderStatus.Pending,
            CreatedAt = DateTime.UtcNow
        };

        _db.Orders.Add(order);
        await _db.SaveChangesAsync(cancellationToken);

        return order.Id;
    }
}
```

控制器里就可以统一变成这样：

```csharp
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    private readonly IMediator _mediator;

    public OrdersController(IMediator mediator)
    {
        _mediator = mediator;
    }

    [HttpPost]
    public async Task<IActionResult> Create(
        [FromBody] CreateOrderCommand command,
        CancellationToken cancellationToken)
    {
        int orderId = await _mediator.Send(command, cancellationToken);
        return Ok(orderId);
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetDetail(int id, CancellationToken cancellationToken)
    {
        var result = await _mediator.Send(new GetOrderDetailQuery(id), cancellationToken);
        return result is null ? NotFound() : Ok(result);
    }
}
```

这套写法的好处是：

* 控制器非常薄
* 每个请求都是一个独立入口对象
* 处理逻辑集中在处理器里

### `CQRS` 最重要的收益到底是什么？

#### 1. 职责清晰

写操作和读操作分开后，代码阅读成本会明显下降。

看到 `CreateOrderCommandHandler`，就知道它是改数据的。

看到 `GetOrderListQueryHandler`，就知道它是查数据的。

#### 2. 读模型可以按页面定制

传统做法很容易纠结：

* 是不是要复用同一个 `OrderDto`
* 为了一个列表页又要加几个字段
* 为了另一个详情页又删几个字段

`CQRS` 下这事反而简单了：

* 列表页就用列表页自己的 `DTO`
* 详情页就用详情页自己的 `DTO`
* 报表页就单独建报表模型

因为查询模型本来就是给读场景服务的。

#### 3. 写规则更容易收口

命令处理器天然适合承接：

* 状态校验
* 业务规则
* 事务控制
* 领域方法调用

这会比把一堆规则散落在控制器和服务里更稳。

#### 4. 方便加横切逻辑

如果配合 `MediatR` 管道行为，很多通用处理都能统一做：

* 参数校验
* 日志记录
* 性能统计
* 事务提交
* 幂等控制

### 它和“读写分离”是什么关系？

这两个词经常一起出现，但不是一个层级。

#### `CQRS`

更偏代码职责和模型拆分。

#### 读写分离

更偏基础设施层面的数据库架构拆分。

也就是说：

* 可以做 `CQRS`，但不做物理读写分离
* 也可以数据库有主从，但代码层仍然写得很 `CRUD`

真正完整的高阶形态，一般是两者结合：

* 写命令进写库
* 读查询走读库
* 读模型按查询需求单独优化

### 进一步升级：读库和写库彻底分开

如果订单写入压力和查询压力差异很大，架构可能会继续演化。

例如：

```text
Command -> Write DB -> Domain Event / MQ -> Read DB
Query   -> Read DB
```

这种架构的优势是：

* 写端专注事务一致性
* 读端专注查询性能
* 读模型可以冗余字段、提前聚合、做缓存

但代价也很明显：

* 系统复杂度会上升
* 需要处理最终一致性
* 需要处理消息丢失、重试、重复消费

### 什么时候适合上 `CQRS`？

比较适合：

* 读多写少，查询形态很多
* 写规则复杂，状态流转多
* 系统在持续变大，服务类越来越臃肿
* 需要很薄的控制器和明确的用例边界
* 后面有读写分离、消息驱动、事件投影的演进空间

尤其是这些项目类型：

* 中大型后台系统
* 电商订单、库存、支付类业务
* 审批流、状态机明显的系统
* 微服务或模块边界比较清晰的项目

### 什么时候不一定要上？

也要把边界说透。

不太适合：

* 很小的后台工具
* 只有少量接口的管理页
* 查询和写入都非常简单
* 团队当前还压不住额外抽象

因为 `CQRS` 不是没有成本。

它会带来的代价包括：

* 文件数明显增加
* 同一个模块会拆出更多请求对象和处理器
* 对命名、目录结构、规范要求更高

如果业务复杂度还没到那个程度，强上只会让简单问题复杂化。

### `CQRS` 最常见的几个误区

#### 误区 1：用了 `MediatR` 就等于用了 `CQRS`

不是。

`MediatR` 只是发送和分发请求的工具。

真正的 `CQRS` 是：

* 命令和查询职责分离
* 读写模型分离
* 处理方式分离

没有这些分离，只是套了个 `Mediator` 壳子，不算真正意义上的 `CQRS`。

#### 误区 2：命令一定不能返回任何东西

严格说，命令更关注副作用，而不是回传查询数据。

但工程上返回一些必要信息完全正常，比如：

* 新建后的 `Id`
* 成功/失败结果
* 版本号

真正需要避免的是：命令处理器顺手查一大坨展示数据再返回。

#### 误区 3：查询端必须复用实体

恰恰相反，查询端很多时候就不该复用实体。

查询端应该优先返回最贴近页面需求的 `DTO` 或读模型。

#### 误区 4：所有项目都应该从第一天开始完整 CQRS

没必要。

很多项目的合理路径其实是：

1. 先保持清晰的分层
2. 读写复杂度上来后，先做简化版 `CQRS`
3. 压力继续增长，再考虑读写库拆分和事件投影

这是更稳的演进路线。

### 目录结构怎么组织更合适？

`.NET` 里常见有两种组织方式。

#### 1. 按类型分目录

```text
Application
├── Commands
├── Queries
├── Handlers
└── Dtos
```

这种方式一开始很好理解，但功能多了以后，跳文件会比较频繁。

#### 2. 按功能切片

```text
Features
└── Orders
    ├── CreateOrder
    │   ├── CreateOrderCommand.cs
    │   └── CreateOrderCommandHandler.cs
    ├── CancelOrder
    │   ├── CancelOrderCommand.cs
    │   └── CancelOrderCommandHandler.cs
    ├── GetOrderDetail
    │   ├── GetOrderDetailQuery.cs
    │   ├── GetOrderDetailQueryHandler.cs
    │   └── OrderDetailDto.cs
    └── GetOrderList
        ├── GetOrderListQuery.cs
        ├── GetOrderListQueryHandler.cs
        └── OrderListItemDto.cs
```

这类“按功能切片”的写法，在 `CQRS` 里通常更顺手。

原因很简单：

* 一个功能相关的文件都放在一起
* 看、改、测都更集中

### 落地时几条很实用的建议

#### 1. 命令名一定要表达业务意图

优先：

* `CreateOrderCommand`
* `CancelOrderCommand`
* `ApproveRefundCommand`

少用：

* `SaveOrderCommand`
* `EditOrderCommand`

#### 2. 查询模型不要强行复用

列表页、详情页、报表页的数据结构不同，本来就应该拆开。

#### 3. 写处理器里优先保证规则和一致性

不要把命令处理器写成“查一堆 + 拼一堆返回”的混合体。

#### 4. 查询端优先考虑投影

能直接 `Select` 到 `DTO`，就别先整实体再映射一遍。

#### 5. 别把每一个小功能都过度仪式化

不是所有按钮点击都值得拆成十几个类。

拆分是为了清晰，不是为了表演架构。

### 总结

* `CQRS` 先是职责分离，其次才是架构升级
* 起步完全可以只做代码层拆分，不必一上来就双库双模型
* 真正适合 `CQRS` 的系统，通常都是“读写关注点明显不同”的系统

