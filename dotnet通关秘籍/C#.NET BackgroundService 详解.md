### 简介

`BackgroundService` 是 `.NET Core` 引入的用于实现长时间运行后台任务的基类，位于 `Microsoft.Extensions.Hosting` 命名空间。它是构建 `Worker Service` 和后台处理的核心组件。

**为什么使用 BackgroundService？**

* 优雅的生命周期管理：自动处理启动、运行、停止，响应主机生命周期（如应用关闭）。

* 异步支持：与 `Task` 和 `CancellationToken` 无缝集成，适合异步操作。

* 线程安全：通过 `CancellationToken` 和并发集合（如 `ConcurrentQueue<T>`）支持多线程场景。

* 与依赖注入集成：通过构造函数注入服务（如 `ILogger、HttpClient`）。

* 替代传统方法：比手动使用 `Thread` 或 `Timer` 更安全、可控。

**适用场景**

* 定时任务：如每分钟检查数据库更新。

* 消息队列处理：消费 `RabbitMQ` 或 `Kafka` 消息。

* 日志收集：异步写入日志到文件或服务。

* 后台计算：周期性执行 `CPU` 密集型任务。

* 事件监听：监听外部事件（如文件变化、网络请求）。

* 数据同步服务：定期从外部系统同步数据

* 批处理服务：定期执行批量数据处理任务

* 缓存刷新服务：定时更新应用程序缓存

### 核心概念

* 继承关系：继承自 `IHostedService` 接口的抽象类

* 生命周期：随宿主（`Host`）启动而开始，随宿主关闭而停止

* 执行线程：默认在后台线程执行（非主线程）

* 典型应用场景：定时任务、消息队列处理、数据同步、监控检查等

* 托管方式：由 `.NET` 通用主机（`Generic Host`）管理

### 核心方法解析

```csharp
public abstract class BackgroundService : IHostedService, IDisposable
{
    // 必须重写 - 核心任务逻辑
    protected abstract Task ExecuteAsync(CancellationToken stoppingToken);

    // 启动时调用（通常无需重写）
    public virtual Task StartAsync(CancellationToken cancellationToken);

    // 停止时调用（通常无需重写）
    public virtual Task StopAsync(CancellationToken cancellationToken);
}
```

### 实现 BackgroundService

实现自定义后台服务需要继承 `BackgroundService` 并覆盖关键方法

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

namespace BackgroundServiceDemo
{
    // 自定义后台服务示例
    public class MyBackgroundService : BackgroundService
    {
        private readonly ILogger<MyBackgroundService> _logger;
        private readonly int _interval;

        // 通过依赖注入获取所需服务
        public MyBackgroundService(ILogger<MyBackgroundService> logger)
        {
            _logger = logger;
            _interval = 5000; // 5秒执行一次
        }

        // 核心执行方法，服务启动后持续运行
        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            _logger.LogInformation("MyBackgroundService 已启动");
            
            while (!stoppingToken.IsCancellationRequested)
            {
                try
                {
                    // 执行后台任务
                    await DoWorkAsync(stoppingToken);
                    
                    // 等待下一次执行
                    await Task.Delay(_interval, stoppingToken);
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "后台服务执行出错");
                    // 错误处理和重试策略
                    await Task.Delay(TimeSpan.FromSeconds(3), stoppingToken);
                }
            }
            
            _logger.LogInformation("MyBackgroundService 已停止");
        }

        // 具体的工作方法
        private async Task DoWorkAsync(CancellationToken stoppingToken)
        {
            _logger.LogInformation($"执行后台工作 - {DateTime.Now}");
            // 模拟异步操作
            await Task.Delay(1000, stoppingToken);
        }

        // 服务停止时的清理工作
        public override Task StopAsync(CancellationToken cancellationToken)
        {
            _logger.LogInformation("MyBackgroundService 正在停止...");
            return base.StopAsync(cancellationToken);
        }
    }
}
```

### 实现 IHostedService 接口

```csharp
public class SimpleHostedService : IHostedService
{
    private readonly ILogger<SimpleHostedService> _logger;

    public SimpleHostedService(ILogger<SimpleHostedService> logger)
    {
        _logger = logger;
    }

    public Task StartAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("SimpleHostedService 已启动");
        // 启动后台任务
        _ = ExecuteAsync(cancellationToken);
        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        _logger.LogInformation("SimpleHostedService 已停止");
        return Task.CompletedTask;
    }

    private async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // 后台任务逻辑
    }
}
```

### BackgroundService 生命周期

* 构造函数：创建服务实例，通常用于依赖注入

* `StartAsync(CancellationToken)`：服务启动时调用，初始化资源

* `ExecuteAsync(CancellationToken)`：核心方法，服务的主要工作在此执行

* `StopAsync(CancellationToken)`：服务停止时调用，用于清理资源

* `Dispose(Boolean)`：释放非托管资源

**详解**

* 启动（`StartAsync`）：

    * 主机启动时调用，初始化任务。

    * 默认实现调用 `ExecuteAsync(stoppingToken)`。

    * 可通过 `cancellationToken` 取消启动。

* 执行（`ExecuteAsync`）

    * 任务的主逻辑，运行在后台线程或线程池。

    * 通常包含循环，检查 `stoppingToken.IsCancellationRequested`。

    * 必须是异步的，返回 `Task`。

* 停止（`StopAsync`）：

    * 主机关闭时调用，优雅停止任务。

    * 默认等待 `ExecuteAsync` 完成，或通过 `cancellationToken` 强制停止。

    * 可自定义清理逻辑。

```text
主机启动 -> StartAsync -> ExecuteAsync (循环运行) -> 主机关闭 -> StopAsync
```

### 典型使用场景

#### 定时轮询任务

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        await DoWorkAsync();
        await Task.Delay(TimeSpan.FromMinutes(5), stoppingToken); // 每5分钟执行
    }
}
```

#### 消息队列消费

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        var message = await _queueClient.ReceiveMessageAsync();
        if (message != null) ProcessMessage(message);
    }
}
```

#### 数据库同步任务

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    await SyncDataFromSourceToTarget();
    // 单次执行后退出
}
```

#### 系统健康检查

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    var timer = new PeriodicTimer(TimeSpan.FromSeconds(30));
    
    while (await timer.WaitForNextTickAsync(stoppingToken))
    {
        await CheckSystemHealth();
    }
}
```

#### 作用域服务处理

```csharp
public class ScopedBackgroundService : BackgroundService
{
    private readonly IServiceScopeFactory _scopeFactory;

    public ScopedBackgroundService(IServiceScopeFactory scopeFactory)
    {
        _scopeFactory = scopeFactory;
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var scope = _scopeFactory.CreateScope();
        var service = scope.ServiceProvider.GetRequiredService<IScopedProcessingService>();
        await service.DoWork(ct);
    }
}
```

#### 健康检查集成

```csharp
public class HealthCheckService : BackgroundService, IHealthCheck
{
    private volatile bool _isHealthy = true;
    
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            _isHealthy = await CheckExternalDependency();
            await Task.Delay(TimeSpan.FromSeconds(10), ct);
        }
    }
    
    public Task<HealthCheckResult> CheckHealthAsync(HealthCheckContext context, 
        CancellationToken ct = default)
    {
        return _isHealthy 
            ? Task.FromResult(HealthCheckResult.Healthy())
            : Task.FromResult(HealthCheckResult.Unhealthy("依赖服务不可用"));
    }
}
```

### 完整创建流程

#### 创建服务类

```csharp
public class DataSyncService : BackgroundService
{
    private readonly ILogger<DataSyncService> _logger;

    // 通过构造函数注入依赖
    public DataSyncService(ILogger<DataSyncService> logger)
    {
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("数据同步服务启动");
        
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await SyncDataAsync();
                await Task.Delay(TimeSpan.FromHours(1), stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "数据同步失败");
            }
        }
    }
}
```

#### 注册服务

```csharp
// Program.cs
var builder = Host.CreateDefaultBuilder(args);

builder.ConfigureServices(services => 
{
    services.AddHostedService<DataSyncService>();
});

await builder.Build().RunAsync();
```

### 关键特性详解

#### 优雅关闭（Graceful Shutdown）

* 当主机收到关闭信号时：

    * 触发 `stoppingToken.IsCancellationRequested`

    * 调用 `StopAsync()` 方法

    * 默认等待 5 秒（可通过 `ShutdownTimeout` 配置）

```csharp
// 配置关闭等待时间
builder.ConfigureHostOptions(opts => 
{
    opts.ShutdownTimeout = TimeSpan.FromSeconds(10);
});
```

#### 错误处理策略

* 未处理异常会导致服务崩溃

* 推荐做法：在循环内捕获异常

```csharp
while (!stoppingToken.IsCancellationRequested)
{
    try
    {
        // 业务逻辑
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "后台任务出错");
        await Task.Delay(TimeSpan.FromMinutes(5)); // 避免错误循环
    }
}
```

#### 依赖注入支持

* 支持通过构造函数注入任意服务

```csharp
builder.Services.AddSingleton<IEmailService, EmailService>();
public class EmailNotificationService : BackgroundService
{
    private readonly IEmailService _emailService;
    
    public EmailNotificationService(IEmailService emailService)
    {
        _emailService = emailService;
    }
}
```

#### 并发控制

* 默认单实例运行

* 需要并行处理时使用 `Task.WhenAll`

```csharp
protected override async Task ExecuteAsync(CancellationToken token)
{
    var tasks = new List<Task>();
    
    for (int i = 0; i < 5; i++)
    {
        tasks.Add(ProcessChannelAsync($"Channel-{i}", token));
    }
    
    await Task.WhenAll(tasks);
}
```

#### 与 IHostedService 的关系

* 直接实现 `IHostedService`：需要手动管理执行逻辑和取消令牌

* 继承 `BackgroundService`：只需关注 `ExecuteAsync` 的业务实现

### 最佳实践

#### 错误处理与重试机制

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        try
        {
            // 执行工作，包含重试逻辑
            await RetryPolicy.ExecuteAsync(
                async () => await DoWorkWithRetryAsync(stoppingToken),
                stoppingToken);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "后台服务执行出错");
            // 可选：记录错误后等待一段时间再重试
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}

// 使用 Polly 库实现重试策略
private readonly AsyncRetryPolicy _retryPolicy = Policy
    .Handle<Exception>()
    .WaitAndRetryAsync(
        retryCount: 3,
        retryAttempt => TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)),
        (exception, timeSpan, retryCount, context) =>
        {
            _logger.LogWarning($"操作失败，{retryCount}秒后重试: {exception.Message}");
        });
```

#### 响应取消请求

```csharp
while (!stoppingToken.IsCancellationRequested) // 正确检查取消令牌
```

#### 使用异步延迟

```csharp
await Task.Delay(interval, stoppingToken); // 支持取消
```

#### 资源清理

```csharp
public override void Dispose()
{
    _databaseConnection?.Close();
    base.Dispose();
}
```

#### 日志记录

```csharp
_logger.LogInformation("服务启动");
_logger.LogInformation("服务停止");
```

#### 避免阻塞异步方法

```csharp
// 错误：阻塞异步操作
Task.Run(DoWork).Wait(); 
```

#### 避免忽略取消令牌

```csharp
// 错误：无法响应关闭请求
while (true) 
{
    // ...
}
```

#### 避免长时间同步操作

```csharp
// 错误：阻塞线程池线程
protected override Task ExecuteAsync(CancellationToken token)
{
    LongRunningSynchronousWork(); // 应改为异步
    return Task.CompletedTask;
}
```

#### 注入 IHostApplicationLifetime 监听生命周期

```csharp
public class LifecycleService : BackgroundService
{
    private readonly IHostApplicationLifetime _lifetime;
    
    public LifecycleService(IHostApplicationLifetime lifetime) => _lifetime = lifetime;

    protected override Task ExecuteAsync(CancellationToken token)
    {
        _lifetime.ApplicationStarted.Register(() => Console.WriteLine("服务启动"));
        _lifetime.ApplicationStopping.Register(() => Console.WriteLine("服务停止中"));
        _lifetime.ApplicationStopped.Register(() => Console.WriteLine("服务已停止"));
        
        return Task.CompletedTask;
    }
}
```

### 高级应用场景

#### 多个后台服务协调

```csharp
// 注册多个服务
services.AddHostedService<ServiceA>();
services.AddHostedService<ServiceB>();

// 使用通道（Channel）通信
var channel = Channel.CreateBounded<Message>(100);
services.AddSingleton(channel);
```

#### 基于Quartz.NET的定时任务

```csharp
protected override async Task ExecuteAsync(CancellationToken token)
{
    var scheduler = await SchedulerBuilder.Create()
        .UseDefaultThreadPool()
        .BuildScheduler();

    var job = JobBuilder.Create<MyJob>().Build();
    var trigger = TriggerBuilder.Create()
        .WithCronSchedule("0 0/5 * * * ?")
        .Build();

    await scheduler.ScheduleJob(job, trigger, token);
    await scheduler.Start(token);
}
```

#### 后台服务状态监控

```csharp
public class MonitorService : BackgroundService
{
    private readonly IEnumerable<IHostedService> _services;
    
    public MonitorService(IEnumerable<IHostedService> services)
    {
        _services = services;
    }
    
    protected override async Task ExecuteAsync(CancellationToken token)
    {
        foreach (var service in _services.OfType<BackgroundService>())
        {
            // 检查服务状态
        }
    }
}
```

#### 使用PeriodicTimer替代Task.Delay

```csharp
// 使用PeriodicTimer替代Task.Delay (避免计时器累积)
private readonly PeriodicTimer _timer = new(TimeSpan.FromMinutes(5));

protected override async Task ExecuteAsync(CancellationToken ct)
{
    while (await _timer.WaitForNextTickAsync(ct))
    {
        await DoWork();
    }
}
```

#### 限制并发处理数量

```csharp
// 限制并发处理数量
private readonly SemaphoreSlim _throttler = new(5, 5);

private async Task ProcessItemAsync(Item item)
{
    await _throttler.WaitAsync();
    try
    {
        await _processor.Process(item);
    }
    finally
    {
        _throttler.Release();
    }
}
```

#### 分布式场景处理

```csharp
protected override async Task ExecuteAsync(CancellationToken ct)
{
    if (!await _distributedLock.AcquireLockAsync("service-lock", ct))
    {
        _logger.LogInformation("另一个实例正在运行，本实例退出");
        return;
    }
    
    try
    {
        // 获得锁的实例执行任务
        while (!ct.IsCancellationRequested)
        {
            await ProcessDistributedWork();
            await Task.Delay(60000, ct);
        }
    }
    finally
    {
        await _distributedLock.ReleaseLockAsync("service-lock");
    }
}
```

#### 依赖注入的进阶用法

`BackgroundService` 默认为单例，无法直接注入 `Scoped` 服务，使用手动创建作用域的方式

```csharp
public class DbSyncService : BackgroundService
{
    private readonly IServiceProvider _services;
    public DbSyncService(IServiceProvider services) => _services = services;

    protected override async Task ExecuteAsync(CancellationToken token)
    {
        using var scope = _services.CreateScope();
        var dbContext = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // 使用 dbContext 操作数据库...
    }
}

```

### 与其他后台处理技术的比较

|  技术方案   |  适用场景   |  特点   |
| --- | --- | --- |
| BackgroundService    |  长时间运行的后台服务   |  与宿主集成，支持依赖注入，生命周期管理完善   |
|  Task.Run   |  短时间异步任务   |  轻量级，适合快速创建后台任务，但缺乏生命周期管理   |
|  Timer   |  定时执行的简单任务   |  实现简单，但功能有限，不支持异步操作   |
|  BackgroundWorker   |  .NET Framework 后台任务   |  传统方案，适合 Windows 桌面应用，在 .NET Core 中逐渐被淘汰   |
|  Hangfire   |  分布式任务调度   |  支持任务持久化、重试、优先级等高级功能，适合分布式系统   |

### 性能考虑与注意事项

* 避免阻塞线程：在 `ExecuteAsync` 中使用异步方法，避免使用同步方法导致线程阻塞

* 正确处理取消令牌：在所有异步操作中传递 `CancellationToken`，确保服务可优雅停止

* 资源管理：在 `StopAsync` 和 `Dispose` 中释放所有资源，避免内存泄漏

* 异常处理：实现完善的异常处理和重试机制，确保服务稳定性

* 日志记录：添加足够的日志，便于监控和调试后台服务

* 并发控制：如果需要处理多个任务，考虑添加并发控制机制

### BackgroundService 在不同项目类型中的应用

#### 支持使用 BackgroundService 的项目类型

##### Console 控制台应用

* 最直接的应用场景：控制台应用可以作为 `Windows` 服务或 `Linux` 守护进程运行。

* 优势：轻量级、易于部署，适合作为后台服务的宿主。

##### Class Library 类库项目

* 封装可复用的服务逻辑：将 `BackgroundService` 实现封装在类库中，供不同类型的应用程序引用。

* 依赖注入设计：通过接口暴露服务，便于在不同宿主环境中集成。

##### Worker Service 项目模板

* 专门为后台服务设计：`.NET` 提供的专用模板，内置 `BackgroundService` 和 `HostBuilder`。

* 开箱即用：默认包含配置、日志和依赖注入支持。

##### Windows 服务

* 长时间运行的桌面服务：通过 `Microsoft.Extensions.Hosting.WindowsServices` 包将控制台应用转换为 `Windows` 服务。

##### Linux 守护进程

* 在 `Linux` 系统上运行：通过 `systemd` 或其他服务管理器管理 `.NET` 控制台应用。

#### 在非 Web 项目中使用 BackgroundService 的步骤

##### Console 控制台应用示例

* 创建控制台项目

```shell
dotnet new console -n MyBackgroundServiceApp
cd MyBackgroundServiceApp
```

* 添加必要的 `NuGet` 包

```shell
dotnet add package Microsoft.Extensions.Hosting
```

* 实现自定义 `BackgroundService`

```csharp
// MyBackgroundService.cs
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

public class MyBackgroundService : BackgroundService
{
    private readonly ILogger<MyBackgroundService> _logger;

    public MyBackgroundService(ILogger<MyBackgroundService> logger)
    {
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("服务已启动，每5秒执行一次");
        
        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation($"执行任务: {DateTime.Now}");
            await Task.Delay(5000, stoppingToken);
        }
        
        _logger.LogInformation("服务已停止");
    }
}
```

* 配置和启动主机

```csharp
// Program.cs
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

var host = Host.CreateDefaultBuilder(args)
    .ConfigureLogging(logging =>
    {
        logging.AddConsole();
        logging.SetMinimumLevel(LogLevel.Information);
    })
    .ConfigureServices(services =>
    {
        // 注册后台服务
        services.AddHostedService<MyBackgroundService>();
        
        // 注册其他依赖服务
        services.AddSingleton<IMyService, MyService>();
    })
    .Build();

await host.RunAsync();
```

##### Class Library 类库项目示例

* 创建类库项目

```shell
dotnet new classlib -n MyServiceLibrary
cd MyServiceLibrary
```

* 添加必要的 `NuGet` 包

```shell
dotnet add package Microsoft.Extensions.Hosting
```

* 在类库中实现 `BackgroundService`

```csharp
// MyLibraryService.cs
using System.Threading;
using System.Threading.Tasks;
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

namespace MyServiceLibrary
{
    public class MyLibraryService : BackgroundService
    {
        private readonly ILogger<MyLibraryService> _logger;
        private readonly IMyDependency _dependency;

        public MyLibraryService(ILogger<MyLibraryService> logger, IMyDependency dependency)
        {
            _logger = logger;
            _dependency = dependency;
        }

        protected override async Task ExecuteAsync(CancellationToken stoppingToken)
        {
            while (!stoppingToken.IsCancellationRequested)
            {
                await _dependency.ProcessDataAsync(stoppingToken);
                await Task.Delay(5000, stoppingToken);
            }
        }
    }

    public interface IMyDependency
    {
        Task ProcessDataAsync(CancellationToken cancellationToken);
    }

    public class MyDependency : IMyDependency
    {
        private readonly ILogger<MyDependency> _logger;

        public MyDependency(ILogger<MyDependency> logger)
        {
            _logger = logger;
        }

        public Task ProcessDataAsync(CancellationToken cancellationToken)
        {
            _logger.LogInformation("处理数据...");
            return Task.CompletedTask;
        }
    }
}
```

* 在控制台应用中引用并使用类库服务

```csharp
// 在引用类库的控制台应用中
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;
using MyServiceLibrary;

var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices(services =>
    {
        // 注册类库中的服务
        services.AddHostedService<MyLibraryService>();
        services.AddSingleton<IMyDependency, MyDependency>();
    })
    .Build();

await host.RunAsync();
```

##### Worker Service 项目

* 创建 `Worker Service` 项目

```shell
dotnet new worker -n MyWorkerService
cd MyWorkerService
```

* 默认结构

`Worker Service` 模板已经创建了基本的 `BackgroundService` 实现

```csharp
// Worker.cs
public class Worker : BackgroundService
{
    private readonly ILogger<Worker> _logger;

    public Worker(ILogger<Worker> logger)
    {
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("Worker running at: {time}", DateTimeOffset.Now);
            await Task.Delay(1000, stoppingToken);
        }
    }
}
```

#### 在不同环境中启动 BackgroundService

##### 作为普通控制台应用运行

直接执行可执行文件

```shell
dotnet MyBackgroundServiceApp.dll
```

##### 作为 Windows 服务运行

* 添加 `Windows` 服务支持包

```shell
dotnet add package Microsoft.Extensions.Hosting.WindowsServices
```

* 修改 `Program.cs` 以支持 `Windows` 服务

```csharp
using Microsoft.Extensions.Hosting.WindowsServices;

var host = Host.CreateDefaultBuilder(args)
    .UseWindowsService(options =>
    {
        options.ServiceName = "MyBackgroundService";
    }) // 添加 Windows 服务支持
    .ConfigureServices(services =>
    {
        services.AddHostedService<MyBackgroundService>();
    })
    .Build();

await host.RunAsync();
```

* 安装和管理 `Windows` 服务

```powershell
# 安装服务
sc create MyService binPath="C:\Path\To\MyBackgroundServiceApp.exe"

# 启动服务
sc start MyService

# 停止服务
sc stop MyService

# 删除服务
sc delete MyService
```

##### 作为 Linux 守护进程运行（使用 systemd）

* 添加 `NuGet` 包

```shell
dotnet add package Microsoft.Extensions.Hosting.Systemd
```

* 修改启动配置

```csharp
await Host.CreateDefaultBuilder(args)
    .UseSystemd() // 关键配置
    .ConfigureServices(services => services.AddHostedService<Worker>())
    .Build()
    .RunAsync();
```

* 创建 `systemd` 服务配置文件

```shell
sudo nano /etc/systemd/system/my-service.service
```

```ini
[Unit]
Description=My Background Service
After=network.target

[Service]
WorkingDirectory=/path/to/application
ExecStart=/usr/bin/dotnet /path/to/application/MyBackgroundServiceApp.dll
Restart=always
# Restart service after 10 seconds if the dotnet service crashes:
RestartSec=10
KillSignal=SIGINT
SyslogIdentifier=my-service
User=www-data
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false

[Install]
WantedBy=multi-user.target
```

* 启动和管理服务

```shell
# 重新加载 systemd 配置
sudo systemctl daemon-reload

# 启动服务
sudo systemctl start my-service

# 设置开机自启
sudo systemctl enable my-service

# 查看服务状态
sudo systemctl status my-service

# 停止服务
sudo systemctl stop my-service
```

#### 通用启动模式代码模板

```csharp
// 通用主机构建模板
var hostBuilder = Host.CreateDefaultBuilder(args)
    .ConfigureAppConfiguration((hostingContext, config) =>
    {
        config.AddJsonFile("appsettings.json", optional: true)
              .AddEnvironmentVariables();
    })
    .ConfigureLogging(logging =>
    {
        logging.AddConsole();
        logging.AddDebug();
#if DEBUG
        logging.SetMinimumLevel(LogLevel.Debug);
#else
        logging.SetMinimumLevel(LogLevel.Information);
#endif
    })
    .ConfigureServices((hostContext, services) =>
    {
        // 注册后台服务
        services.AddHostedService<YourBackgroundService>();
        
        // 注册其他依赖服务
        services.AddSingleton<IServiceDependency, ServiceImplementation>();
    });

// 根据环境选择运行方式
#if WINDOWS_SERVICE
hostBuilder.UseWindowsService();
#elif LINUX_SERVICE
hostBuilder.UseSystemd();
#endif

// 构建并运行主机
await hostBuilder.Build().RunAsync();
```

### Worker Service 项目模板与 Console 控制台项目的区别

|  特性   |  Worker Service 项目   |  Console 控制台项目   |
| --- | --- | --- |
|  宿主模型   |  内置通用主机（ `IHost` ），自动管理生命周期、依赖注入、日志和配置   |  需手动实现主机逻辑（如启动/停止逻辑、依赖注入容器）   |
|  依赖注入   |  原生支持（通过 `ConfigureServices` 注册服务）   |  需手动集成（如使用 `Microsoft.Extensions.DependencyInjection` ）   |
|  日志系统   |  默认集成 `ILogger` ，支持多种输出（ `Console、EventLog` 等）   |  需手动配置日志框架（如 `NLog、Serilog` ）   |
|  配置管理   |  自动加载 `appsettings.json` 和环境变量   |  需手动解析配置文件   |
|  后台任务管理   |  通过 `BackgroundService` 基类封装异步任务生命周期   |  需自行实`CancellationToken` 响应和循环逻辑   |
|  跨平台服务支持   |  原生提供 `UseWindowsService()/UseSystemd()`   |  需手动处理服务安装信号（如 `Systemd.SdNotify` ）   |
|  典型应用场景   |  长时间运行的后台服务（如消息处理、定时任务）   |  短期运行的命令行工具或脚本   |

#### 项目结构

* `Worker Service`

```text
MyWorkerService/
├── Program.cs         // 包含 HostBuilder 配置
├── Worker.cs          // 默认实现的 BackgroundService
├── appsettings.json   // 配置文件
├── appsettings.Production.json
└── MyWorkerService.csproj
```

* `Console` 项目

```text
MyConsoleApp/
├── Program.cs         // 简单的 Main 方法入口
└── MyConsoleApp.csproj
```

#### 代码实现差异

* Worker Service 的 Program.cs

```csharp
public static IHostBuilder CreateHostBuilder(string[] args) =>
    Host.CreateDefaultBuilder(args)
        .ConfigureServices((hostContext, services) =>
        {
            services.AddHostedService<Worker>();
        });
```

* Console 项目的 Program.cs

```csharp
class Program
{
    static void Main(string[] args)
    {
        Console.WriteLine("Hello World!");
    }
}
```

#### 服务生命周期管理

* `Worker Service`：

    * 自动处理启动 / 停止信号

    * 内置 `IHostApplicationLifetime` 管理应用生命周期

    * 支持优雅关闭（`graceful shutdown`）

* `Console` 项目：

    * 需要手动捕获终止信号（如 `Ctrl+C` ）

    * 需要手动实现资源清理逻辑


### Linux 下 systemd 部署为何需要 UseSystemd()

即使可以直接通过 `systemd` 执行 `dotnet xxx.dll`，`UseSystemd()` 仍提供以下不可替代的功能：

#### 优势

#####  生命周期深度集成

* 信号响应：`UseSystemd()` 使应用能响应 `systemd` 的 `SIGTERM` 信号，实现优雅关闭（例如完成当前任务再退出）

* 状态通知：自动向 `systemd` 发送 `"READY=1"` 等状态通知，避免服务被误判为启动超时

##### 日志无缝对接

* `Journald` 集成：日志直接输出到 `systemd journal`，可通过 `journalctl -u your-service` 查看，无需额外配置

* 日志分级：自动映射 `.NET` 日志级别到 `systemd` 的日志优先级（如 `Information → syslog` 的 `LOG_INFO`）

##### 工作目录与路径修正

* `ContentRootPath` 问题：未使用 `UseSystemd()` 时，应用可能误将 `/` 作为根路径，导致配置文件加载失败（例如 `appsettings.json` 找不到）

* 解决方案：`UseSystemd()` 自动设置 `ContentRootPath` 为程序所在目录（等价于 `Directory.GetCurrentDirectory()` ）

##### 环境变量管理

* 环境一致性：自动继承 `systemd` 服务文件中定义的 `Environment=` 变量，避免手动注入

#### 健康状态报告

```csharp
public class Worker : BackgroundService
{
    private readonly ILogger<Worker> _logger;
    private readonly ISystemdNotifier _notifier;

    public Worker(ILogger<Worker> logger, ISystemdNotifier notifier)
    {
        _logger = logger;
        _notifier = notifier;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // 通知systemd服务已就绪
        _notifier.Notify(ServiceState.Ready);
        
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await DoWork();
                // 发送心跳
                _notifier.Notify(ServiceState.Heartbeat);
            }
            catch (Exception ex)
            {
                // 报告错误状态
                _notifier.Notify(ServiceState.Error, $"Error: {ex.Message}");
            }
            
            await Task.Delay(5000, stoppingToken);
        }
    }
}
```

#### UseSystemd() 的核心功能

|  功能   |  直接运行   |  UseSystemd()   |  优势   |
| --- | --- | --- | --- |
|  启动通知   | ❌    |  ✅ Ready 状态通知   |  systemd 准确知道服务何时可用   |
|  优雅关闭   |  有限支持   |  ✅ 自定义超时处理   |  确保完成正在处理的任务   |
|  心跳监测   |  ❌   |  ✅ 定期心跳信号   |  自动重启无响应服务   |
|  状态报告   |  ❌   |  ✅ 自定义状态码   |  精确监控服务健康状态   |
|  日志集成   |  文本日志   |  ✅ Journald 结构化日志   |  支持高级日志查询和过滤   |
|  资源压力处理   |  ❌   |  ✅ 内存压力通知   |  系统OOM前主动降级   |

#### UseSystemd () 的实现原理

当调用 `UseSystemd()` 时，`.NET` 运行时会：

* 设置 `EnvironmentVariable` 以指示应用正在 `systemd` 环境中运行

* 注册信号处理器（如 `SIGTERM、SIGHUP`）

* 通过 `sd_notify` 协议与 `systemd` 通信：

    * 发送 `READY=1` 通知服务已就绪

    * 发送 `STATUS=...` 更新服务状态信息

    * 发送 `WATCHDOG=1` 实现服务健康监控

#### 核心集成点源码解析

```csharp
// Microsoft.Extensions.Hosting.Systemd 源码简化
public static IHostBuilder UseSystemd(this IHostBuilder hostBuilder)
{
    if (SystemdHelpers.IsSystemdService())
    {
        // 1. 替换生命周期管理器
        hostBuilder.ConfigureServices(services => 
        {
            services.AddSingleton<IHostLifetime, SystemdLifetime>();
        });
        
        // 2. 配置日志系统
        hostBuilder.ConfigureLogging(logging => 
        {
            logging.AddSystemdConsole();
        });
    }
    return hostBuilder;
}
```

#### SystemdLifetime 核心逻辑

```csharp
// SystemdLifetime 简化实现
public class SystemdLifetime : IHostLifetime
{
    public Task WaitForStartAsync(CancellationToken cancellationToken)
    {
        // 通知systemd服务正在启动
        SystemdNotifier.Notify("EXTEND_TIMEOUT_USEC=30000000"); // 30秒超时
        SystemdNotifier.Notify("STATUS=Initializing...");
        
        // 返回控制器
        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        // 接收SIGTERM后通知
        SystemdNotifier.Notify("STATUS=Shutting down...");
        
        // 执行清理逻辑
        CleanupResources();
        
        // 最终状态通报
        SystemdNotifier.Notify("STATUS=Stopped");
        return Task.CompletedTask;
    }
}
```

### Linux 下部署最佳实践

#### 完整部署流程

* 发布应用

```shell
dotnet publish -c Release -o ./publish --self-contained true -r linux-x64
```

* 创建 `systemd` 服务文件

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My .NET Worker Service
After=network.target

[Service]
Type=notify                   # 关键：启用通知机制
NotifyAccess=all              # 允许所有通知
ExecStart=/var/app/myapp      # 直接执行二进制
Restart=always
RestartSec=10
Environment=DOTNET_ROOT=/usr/share/dotnet
SyslogIdentifier=myapp-service
User=appuser
Group=appgroup

# 资源限制
MemoryLimit=500M
CPUQuota=80%

# 工作目录
WorkingDirectory=/var/app

[Install]
WantedBy=multi-user.target
```

* 启用服务

```shell
sudo systemctl daemon-reload
sudo systemctl enable myapp.service
sudo systemctl start myapp.service
```

#### 高级监控与维护

* 查看集成状态

```shell
systemctl status myapp.service
# 输出包含 Active: active (running) since... 状态

journalctl -u myapp.service -f
# 查看结构化日志
```

* 关键状态监控点

1. `NotifyAccess` 必须设为 `all`
2. `Type` 必须设为 `notify`
3. 服务日志中应出现 `Application started. Press Ctrl+C to shut down.`
4. `systemd` 日志应显示 `Started My .NET Worker Service.`

### 高可靠 Worker Service 示例

#### 完整生产级实现

```csharp
public class ProductionWorker : BackgroundService
{
    private readonly ILogger<ProductionWorker> _logger;
    private readonly ISystemdNotifier _systemdNotifier;
    private readonly IHealthCheckService _healthCheck;

    public ProductionWorker(
        ILogger<ProductionWorker> logger,
        ISystemdNotifier systemdNotifier,
        IHealthCheckService healthCheck)
    {
        _logger = logger;
        _systemdNotifier = systemdNotifier;
        _healthCheck = healthCheck;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        // 通知systemd服务已就绪
        _systemdNotifier.Notify(ServiceState.Ready);
        _logger.LogInformation("Worker started at: {time}", DateTimeOffset.Now);

        try
        {
            while (!stoppingToken.IsCancellationRequested)
            {
                try
                {
                    await DoCriticalWork(stoppingToken);
                    
                    // 发送心跳信号
                    _systemdNotifier.Notify(ServiceState.Heartbeat);
                }
                catch (OperationCanceledException)
                {
                    // 正常退出
                }
                catch (Exception ex)
                {
                    _logger.LogError(ex, "工作周期失败");
                    
                    // 报告错误状态
                    _systemdNotifier.Notify(ServiceState.Error, 
                        $"Worker failed: {ex.Message}");
                    
                    await Task.Delay(5000, stoppingToken);
                }
                
                // 健康检查
                var report = await _healthCheck.CheckHealthAsync(stoppingToken);
                _systemdNotifier.Notify(report.Status == HealthStatus.Healthy
                    ? ServiceState.Heartbeat
                    : ServiceState.Error);
                
                await Task.Delay(10000, stoppingToken);
            }
        }
        finally
        {
            _logger.LogInformation("Worker stopping at: {time}", DateTimeOffset.Now);
            _systemdNotifier.Notify(ServiceState.Stopping);
        }
    }

    private async Task DoCriticalWork(CancellationToken ct)
    {
        // 关键业务逻辑
        using var activity = Telemetry.ActivitySource.StartActivity("Processing");
        _logger.LogInformation("处理关键任务...");
        await Task.Delay(2000, ct);
    }
}
```

#### Systemd 集成配置

```csharp
var host = Host.CreateDefaultBuilder(args)
    .UseSystemd()  // Linux 系统集成
    .ConfigureServices((context, services) =>
    {
        services.AddHostedService<ProductionWorker>();
        
        // 健康检查集成
        services.AddHealthChecks()
            .AddCheck("database", new DatabaseHealthCheck())
            .AddCheck("external-api", new ApiHealthCheck());
        
        // 系统通知服务
        services.AddSingleton<ISystemdNotifier, SystemdNotifier>();
    })
    .ConfigureLogging(logging => 
    {
        logging.AddJsonConsole(); // 结构化日志
    })
    .Build();
```

