### 简介

* `Hangfire` 是一款开源的 `.NET` 后台任务调度框架，支持持久化存储、自动重试、并发控制与可视化管理面板，免去自己维护 `Windows Service` 或 `Quartz` 之类的复杂度。

* 核心优势：

    * 零侵入：只需引用包、注册服务，即可将任意方法变成后台任务；

    * 持久化可靠：任务状态保存在数据库（如 `SQL Server、Redis、PostgreSQL` 等），进程重启后自动恢复；

    * 可视化仪表盘：内置 `Web Dashboard`，可查看任务队列、执行历史、失败重试等；

    * 多种调度模式：即时任务、延迟任务、定时任务（`Cron`）与批量任务。

* 三大核心组件

|  组件   |  职责   |  部署方式   |
| --- | --- | --- |
|  客户端   |  创建任务（立即、延迟、周期性）   |  可嵌入 WebAPI/控制台等应用   |
|  服务端   |  从存储拉取任务并执行，支持多线程处理   |  可宿主于 IIS/Windows 服务/控制台程序   |
|  持久化存储   |  保存任务信息（方法名、参数、状态）   |  SQL Server/MySQL/Redis/MongoDB 等 |

### 安装与基础配置

#### 安装 NuGet 包

```shell
# 核心包
Install-Package Hangfire.Core
# ASP.NET Core 集成
Install-Package Hangfire.AspNetCore
Install-Package Hangfire.SqlServer    # 或者 Hangfire.Redis、Hangfire.PostgreSql 等
```

#### 在 ASP.NET Core 中配置（Program.cs）

```csharp
var builder = WebApplication.CreateBuilder(args);
// 注册 Hangfire 服务
builder.Services.AddHangfire(cfg => cfg
    .UseSimpleAssemblyNameTypeSerializer()
    .UseRecommendedSerializerSettings()
    .UseSqlServerStorage(builder.Configuration.GetConnectionString("Hangfire"), new SqlServerStorageOptions {
        CommandBatchMaxTimeout = TimeSpan.FromMinutes(5),
        SlidingInvisibilityTimeout = TimeSpan.FromMinutes(5),
        QueuePollInterval = TimeSpan.Zero,
        UseRecommendedIsolationLevel = true,
        UsePageLocksOnDequeue = true,
        DisableGlobalLocks = true
    }));
// 添加后台服务器
builder.Services.AddHangfireServer();  

var app = builder.Build();
// 挂载 Dashboard（可选授权）
app.UseHangfireDashboard("/hangfire");
app.MapControllers();
app.Run();
```

#### 持久化选项

* 默认支持 `SQL Server`；也可使用 `Redis、PostgreSQL、MySql、MongoDB` 等第三方 `Storage` 包。

* `Storage` 由 `IStorage` 接口抽象，灵活替换。

### 核心用法

#### 即时执行（Fire‑and‑Forget）

```csharp
// 任意方法入队，立即返回
BackgroundJob.Enqueue(() => Console.WriteLine("Hello, Hangfire!"));
```

#### 带参数的任务

```csharp
BackgroundJob.Enqueue<EmailService>(
    x => x.SendWelcomeEmail("user@example.com"));
```

#### 延迟执行

```csharp
// 延迟 5 分钟后执行
BackgroundJob.Schedule(() => SendEmail(userId), TimeSpan.FromMinutes(5));
```

#### 定时任务（Recurring Jobs）

```csharp
// 每天凌晨 2 点执行的任务
RecurringJob.AddOrUpdate(() => CleanupDatabase(), Cron.Daily(2, 0));

// 使用自定义 Cron 表达式
RecurringJob.AddOrUpdate("daily-report", () => GenerateDailyReport(), "0 2 * * *");
```

#### 继续任务（Continuation）

```csharp
var parentId = BackgroundJob.Enqueue(() => Step1());
BackgroundJob.ContinueWith(parentId, () => Step2());
```

#### 自定义任务方法

```csharp
// 定义可被 Hangfire 调用的方法
public static class EmailService
{
    public static void SendEmail(string recipient)
    {
        // 发送邮件的逻辑
        Console.WriteLine($"Sending email to {recipient}");
    }
}

// 在控制器中使用
[ApiController]
[Route("[controller]")]
public class JobController : ControllerBase
{
    [HttpPost]
    public IActionResult CreateJob()
    {
        // 入队一个任务
        BackgroundJob.Enqueue(() => EmailService.SendEmail("user@example.com"));
        return Ok();
    }
}
```

#### 批量任务（Batch，需额外包 Hangfire.Pro 或社区插件）

```csharp
var batchId = BatchJob.StartNew(batch => {
    batch.Enqueue(() => TaskA());
    batch.Enqueue(() => TaskB());
});

// 批处理完成后执行
Batch.ContinueBatchWith(batchId, 
    () => SendNotification("Batch completed!"));
```

### 存储结构与特性

#### 核心表结构（以 SQL Server 为例）

* `HangFire.Job`: 存储序列化后的工作项

* `HangFire.State`: 记录每个 `Job` 的各类状态（`Enqueued、Processing、Failed、Succeeded`）

* `HangFire.Hash, Set, List, Counter`: 用于实现队列、定时集合、重试次数等

* `HangFire.Schema` 版本迁移机制，自动升级表结构

* 事务与幂等

    * 默认将调度写入 `Storage` 时与当前数据库事务同步，保证一致性。

    * 在业务方法内，如果要保证幂等，建议自行实现去重或使用分布式锁。

### 仪表盘与监控

#### Dashboard UI

* 默认路径 `/hangfire`，提供 `Overview、Failed Jobs、Succeeded Jobs、Recurring Jobs、Servers、Queues` 等页面；

* 支持基于 `ASP.NET Core` 授权策略限制访问；

#### 实时监控

* 可以在代码中订阅事件：

```csharp
GlobalJobFilters.Filters.Add(new ProlongExpirationTimeAttribute());
```

* 或使用内置的 `Dashboard Hooks` 扩展点，嵌入自定义监控面板。

### 高级与扩展

#### 筛选器（Filters）

* 实现 `IElectStateFilter、IApplyStateFilter、IServerFilter` 可织入任务执行前后逻辑（如自动重试、异常通知、指标埋点）。

```csharp
public class LoggingAttribute : JobFilterAttribute, IServerFilter {
    public void OnPerforming(PerformingContext filterContext) { /* 前置 */ }
    public void OnPerformed(PerformedContext filterContext) { /* 后置 */ }
}
```

```csharp
// 创建自定义过滤器
public class AutomaticRetryAttribute : JobFilterAttribute, IClientFilter, IServerFilter
{
    public int Attempts { get; set; } = 3;

    public void OnCreating(CreatingContext context)
    {
        context.SetJobParameter("RetryCount", Attempts);
    }

    public void OnCreated(CreatedContext context) { }

    public void OnPerforming(PerformingContext context) { }

    public void OnPerformed(PerformedContext context)
    {
        if (context.Exception != null)
        {
            var retryCount = context.GetJobParameter<int>("RetryCount");
            if (retryCount > 0)
            {
                context.Requeue();
                context.SetJobParameter("RetryCount", retryCount - 1);
            }
        }
    }
}

// 全局注册过滤器
GlobalConfiguration.Configuration.UseFilter(new AutomaticRetryAttribute { Attempts = 5 });
```

* 自动重试策略

```csharp
// 全局
GlobalJobFilters.Filters.Add(new AutomaticRetryAttribute { Attempts = 5, DelaysInSeconds = new[]{10,60,300} });
// 单个任务
[AutomaticRetry(Attempts = 3, OnAttemptsExceeded = AttemptsExceededAction.Delete)]
public void DoWork() { /* ... */ }
```

#### 配置工作进程

```csharp
services.AddHangfireServer(options =>
{
    options.WorkerCount = Environment.ProcessorCount * 2; // 设置工作进程数量
    options.Queues = new[] { "critical", "default", "low" }; // 配置队列优先级
});
```

#### 仪表盘扩展

```csharp
// 自定义仪表盘页面
app.UseHangfireDashboard("/hangfire", new DashboardOptions
{
    AppPath = "/", // 返回主应用的链接
    DashboardTitle = "任务调度中心",
    Authorization = new[] { new HangfireDashboardAuthorizationFilter() },
    DisplayStorageConnectionString = false,
    IgnoreAntiforgeryToken = true,
    
    // 添加自定义菜单
    AdditionalDashboardUrls = new []
    {
        new DashboardUrl
        {
            Url = "/custom",
            Title = "自定义监控",
            Type = DashboardUrlType.Link
        }
    }
});
```

#### 集成依赖注入

```csharp
// 在控制器中注入 IBackgroundJobClient
public class OrderController : ControllerBase
{
    private readonly IBackgroundJobClient _backgroundJobClient;

    public OrderController(IBackgroundJobClient backgroundJobClient)
    {
        _backgroundJobClient = backgroundJobClient;
    }

    [HttpPost]
    public IActionResult CreateOrder(Order order)
    {
        // 创建订单
        _backgroundJobClient.Enqueue<OrderService>(x => x.ProcessOrder(order.Id));
        return Ok();
    }
}

// 使用依赖注入的服务
public class OrderService
{
    private readonly ILogger<OrderService> _logger;
    private readonly IOrderRepository _orderRepository;

    public OrderService(ILogger<OrderService> logger, IOrderRepository orderRepository)
    {
        _logger = logger;
        _orderRepository = orderRepository;
    }

    public void ProcessOrder(int orderId)
    {
        _logger.LogInformation($"Processing order {orderId}");
        var order = _orderRepository.GetById(orderId);
        // 处理订单逻辑
    }
}
```

#### 多队列管理

```csharp
// 配置作业服务器处理特定队列
services.AddHangfireServer(options => {
    options.Queues = new[] { "critical", "default", "low" };
});

// 提交任务到特定队列
BackgroundJob.Enqueue<CriticalService>(
    x => x.ProcessCriticalTask(), 
    "critical"); // 指定队列名称
```

#### Redis存储优化

```csharp
config.UseRedisStorage(redisConnection, new RedisStorageOptions
{
    Prefix = "hangfire:",
    Db = 1,
    InvisibilityTimeout = TimeSpan.FromMinutes(5),
    FetchTimeout = TimeSpan.FromSeconds(15)
});
```

### 最佳实践

* 存储选择：高频任务用 `Redis`，低频用 `SQL Server`。

* 队列分离：`CPU` 密集型与 `I/O` 密集型任务分到不同队列。

* 避免长任务：超过 30 分钟的任务需拆解，防止服务端超时。

#### 周期性任务未触发

* 确认作业服务器正在运行

* 检查 `CRON` 表达式是否正确

* 查看作业是否在仪表盘中注册

* 检查服务器时间是否一致

#### 高并发下性能问题

* 使用 `Redis` 作为存储后端

* 增加作业服务器实例

* 使用多队列分离不同优先级任务

* 优化 `SQL Server` 存储架构

### Hangfire与其他方案对比

|  特性   |  Hangfire   |  Quartz.NET   |  Azure Functions   |
| --- | --- | --- | --- |
|  部署方式   |  集成应用   |  独立进程/集成   |  无服务器   |
|  持久化存储   |  多种选择   |  多种选择   |   Azure存储  |
|  监控界面   |  内置完善   |  有限或第三方   |  Azure门户   |
|  扩展性   |  良好   |  良好   |  自动扩展   |
|  成本   | 免费开源    |  免费开源   |  按使用付费   |
| 学习曲线   |  中等   |  较陡峭   |  中等   |
|  适用场景   |  应用内任务调度   | 复杂调度系统	    |  云原生应用   |

### 总结

#### Hangfire 核心优势：

* 简单集成：与 `ASP.NET` 应用无缝集成

* 可靠性：持久化存储确保任务不丢失

* 可视化：内置强大的仪表盘

* 灵活性：支持多种任务类型和调度策略

* 可扩展：支持多服务器分布式部署

#### 典型应用场景：

* 邮件发送、通知推送

* 数据报表生成

* 数据库清理和维护

* 批量数据处理

* 定时任务调度

* 长时间运行的后台作业

#### 何时选择Hangfire：

* 需要与应用紧密集成的任务调度

* 需要可视化监控任务状态

* 已有 `.NET` 基础设施

* 无需云端无服务器架构

* 需要灵活的调度策略

### 资源与扩展

* 官网：https://www.hangfire.io

* 文档：https://docs.hangfire.io

* `GitHub`：https://github.com/HangfireIO/Hangfire

* 扩展包：

    * `Hangfire.Console`：在仪表板中显示任务日志。

    * `Hangfire.Redis.StackExchange`：Redis 存储支持。

    * `Hangfire.Pro`：商业版，提供高级功能（如批处理、性能优化）。