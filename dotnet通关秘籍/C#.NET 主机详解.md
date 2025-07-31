### 简介

在 `C#.NET` 中，主机 (`Host`) 是一个用于封装应用程序资源和生命周期管理的组件。它提供了一种统一的方式来配置、启动和停止应用程序，是构建现代 `.NET` 应用的基础架构。

#### 普通主机 (HostBuilder)

普通主机是 `.NET Core 2.1` 引入的基础主机模型，主要用于非 `Web` 应用场景。它提供了以下核心功能：

* 依赖注入 (`DI`) - 注册服务和组件
* 配置管理 - 读取应用配置
* 日志记录 - 统一日志系统
* 生命周期管理 - 控制应用启动和停止

普通主机的基本示例：

```csharp
using Microsoft.Extensions.Hosting;

var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((hostContext, services) =>
    {
        // 注册服务
        services.AddTransient<IMyService, MyService>();
    })
    .Build();

// 启动主机
await host.StartAsync();

// 应用逻辑...

// 停止主机
await host.StopAsync();
```

#### 通用主机 (Generic Host)

通用主机是普通主机的增强版本，从 `.NET Core 3.0` 开始引入，专为构建非 `HTTP` 应用 (如控制台应用、`Worker` 服务、后台任务等) 而设计。

##### 核心特性

* 内置依赖注入容器 - 支持 `Scoped、Singleton` 和 `Transient` 生命周期

* 配置系统 - 支持多来源配置 (环境变量、命令行、`JSON` 文件等)

* 日志框架 - 集成 `Microsoft.Extensions.Logging`

* `IHostedService` 接口 - 定义后台服务的标准方式

* 环境区分 - 支持开发、测试和生产环境

通用主机创建 `Worker` 服务的示例：

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Logging;

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

// 主机配置
var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((hostContext, services) =>
    {
        // 添加Worker服务
        services.AddHostedService<Worker>();
    })
    .ConfigureLogging(logging =>
    {
        logging.AddConsole();
    })
    .Build();

await host.RunAsync();
```

##### 核心组件

* `IHost`：主机接口，表示应用程序的运行时环境。

* `HostBuilder`：用于配置和构建主机的构建器。

* `IHostedService`：定义后台服务的接口，用于实现长时间运行的任务。

* `Dependency Injection (DI)`：内置依赖注入容器（`Microsoft.Extensions.DependencyInjection`）。

* `Configuration`：支持从多种配置源加载设置。

* `Logging`：集成的日志系统。

##### 优点

* 统一性：为 `Web` 和非 `Web` 应用程序提供一致的托管模型。

* 灵活性：支持自定义服务、配置和生命周期管理。

* 模块化：通过依赖注入和中间件机制实现模块化开发。

* 跨平台：适用于 `Windows、Linux 和 macOS`。

* 可扩展性：易于集成第三方库和自定义逻辑。

##### 创建通用主机

通用主机通过 `Host.CreateDefaultBuilder()` 创建，它会自动配置一些默认行为（如日志、配置、依赖注入）。

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.DependencyInjection;
using System.Threading.Tasks;

class Program
{
    static async Task Main(string[] args)
    {
        // 创建主机
        using var host = Host.CreateDefaultBuilder(args)
            .ConfigureServices((context, services) =>
            {
                // 注册服务
                services.AddHostedService<MyHostedService>();
            })
            .Build();

        // 启动主机
        await host.RunAsync();
    }
}
```

##### 配置主机

`Host.CreateDefaultBuilder()` 提供默认配置，包括：

* 配置源：加载 `appsettings.json`、环境变量、命令行参数。

* 日志：配置控制台日志和调试日志。

* 依赖注入：提供内置 `DI` 容器。

可以通过 `HostBuilder` 自定义配置：

```csharp
using Microsoft.Extensions.Hosting;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using System.Threading.Tasks;

class Program
{
    static async Task Main(string[] args)
    {
        using var host = Host.CreateDefaultBuilder(args)
            .ConfigureAppConfiguration((context, config) =>
            {
                // 添加额外的配置源
                config.AddJsonFile("customsettings.json", optional: true);
            })
            .ConfigureServices((context, services) =>
            {
                // 注册自定义服务
                services.AddSingleton<IMyService, MyService>();
                services.AddHostedService<MyHostedService>();
            })
            .ConfigureLogging(logging =>
            {
                // 配置日志
                logging.AddConsole();
            })
            .Build();

        await host.RunAsync();
    }
}
```

##### 实现 IHostedService

`IHostedService` 是通用主机中用于定义后台服务的接口，适合长时间运行的任务（如定时任务、消息队列处理）。它包含两个主要方法：

* `StartAsync`：在主机启动时调用。

* `StopAsync`：在主机关闭时调用。

```csharp
using Microsoft.Extensions.Hosting;
using System.Threading;
using System.Threading.Tasks;

public class MyHostedService : IHostedService
{
    public Task StartAsync(CancellationToken cancellationToken)
    {
        // 启动逻辑
        Console.WriteLine("MyHostedService is starting...");
        return Task.CompletedTask;
    }

    public Task StopAsync(CancellationToken cancellationToken)
    {
        // 关闭逻辑
        Console.WriteLine("MyHostedService is stopping...");
        return Task.CompletedTask;
    }
}
```

##### 使用 BackgroundService

`BackgroundService` 是 `IHostedService` 的一个抽象基类，简化了后台任务的实现。它适合需要循环运行的任务（如定时任务）。

```csharp
using Microsoft.Extensions.Hosting;
using System;
using System.Threading;
using System.Threading.Tasks;

public class TimedHostedService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            Console.WriteLine($"TimedHostedService is running at {DateTime.Now}");
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}
```

注册服务：

```csharp
services.AddHostedService<TimedHostedService>();
```

##### 依赖注入

通用主机内置了依赖注入容器，可以注册服务并在应用程序中使用：

```csharp
public interface IMyService
{
    void DoWork();
}

public class MyService : IMyService
{
    public void DoWork() => Console.WriteLine("MyService is working!");
}

public class MyHostedService : BackgroundService
{
    private readonly IMyService _myService;

    public MyHostedService(IMyService myService)
    {
        _myService = myService;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            _myService.DoWork();
            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}
```

注册服务：

```csharp
services.AddSingleton<IMyService, MyService>();
services.AddHostedService<MyHostedService>();
```

##### 通用主机的典型应用场景

* 控制台应用程序：如定时任务、数据处理脚本。

* `Windows` 服务或 `Linux` 守护进程：托管长时间运行的服务。

* 消息队列处理：处理 `RabbitMQ、Kafka` 等消息队列。

* 后台任务：如日志清理、数据同步、通知服务。

* `ASP.NET Core Web` 应用：`.NET 6` 及以上版本的 `Web` 应用也使用通用主机。

#### 主机生命周期管理

主机提供了完整的生命周期管理，主要通过以下接口实现：

* `IHostedService` - 定义启动和停止逻辑

* `IApplicationLifetime` - 访问应用生命周期事件

`IHostedService` 接口

```csharp
public interface IHostedService
{
    Task StartAsync(CancellationToken cancellationToken);
    Task StopAsync(CancellationToken cancellationToken);
}
```

#### 生命周期事件

主机提供了三个主要的生命周期事件：

* `ApplicationStarted` - 应用完全启动后触发

* `ApplicationStopping` - 应用开始停止时触发

* `ApplicationStopped` - 应用完全停止后触发

```csharp
var host = Host.CreateDefaultBuilder(args)
    .Build();

var applicationLifetime = host.Services.GetRequiredService<IApplicationLifetime>();

applicationLifetime.ApplicationStarted.Register(() =>
{
    Console.WriteLine("应用已启动");
});

applicationLifetime.ApplicationStopping.Register(() =>
{
    Console.WriteLine("应用正在停止...");
});

applicationLifetime.ApplicationStopped.Register(() =>
{
    Console.WriteLine("应用已停止");
});

await host.RunAsync();
```

#### 配置与依赖注入

* 配置管理

```csharp
var host = Host.CreateDefaultBuilder(args)
    .ConfigureAppConfiguration((hostingContext, config) =>
    {
        // 添加JSON配置文件
        config.AddJsonFile("appsettings.json", optional: true, reloadOnChange: true);
        
        // 添加环境变量配置
        config.AddEnvironmentVariables();
        
        // 添加命令行参数配置
        config.AddCommandLine(args);
    })
    .Build();

// 获取配置
var configuration = host.Services.GetRequiredService<IConfiguration>();
var value = configuration["MySetting"];
```

* 依赖注入

```csharp
var host = Host.CreateDefaultBuilder(args)
    .ConfigureServices((hostContext, services) =>
    {
        // 注册单例服务
        services.AddSingleton<IMySingletonService, MySingletonService>();
        
        // 注册作用域服务
        services.AddScoped<IMyScopedService, MyScopedService>();
        
        // 注册瞬态服务
        services.AddTransient<IMyTransientService, MyTransientService>();
        
        // 注册主机服务
        services.AddHostedService<MyBackgroundService>();
    })
    .Build();
```

### 背景与演进

#### ASP.NET Core 1.x–2.x 的 Web 主机

* 引入了 `IWebHostBuilder、WebHost、Startup` 模式，专注于 `Web` 应用。

* 负责：`HTTP Server（Kestrel）`、中间件管道、依赖注入、配置与日志等。

#### .NET Core 3.0+ 的通用主机（Generic Host）

* 为了在控制台服务、后台任务、`Worker Service`、桌面应用等非 `Web` 场景中复用 `ASP.NET Core` 的依赖注入、配置、日志与生命周期管理，`.NET Core 3.0` 推出了通用主机。

* 主要接口升级为 `IHostBuilder/IHost`，并在内部可按需挂载 `Kestrel、gRPC、SignalR`、定时任务等功能。

* 在 `.NET 6/.NET 7/.NET 8` 中持续完善，已成为构建所有 `.NET Core/5+` 应用的统一入口。

### 主机类型对比

| 特性/接口    | Web Host (`IWebHostBuilder`)                     | Generic Host (`IHostBuilder`)                      |
| ------------ | ------------------------------------------------ | -------------------------------------------------- |
| 适用场景     | 仅限 ASP.NET Core Web 应用                       | 控制台、后台 Worker、Web 及混合场景                |
| 核心接口     | `IWebHostBuilder` → `IWebHost`                   | `IHostBuilder` → `IHost`                           |
| 默认组件     | Kestrel、IIS 集成、中间件管道                    | DI、配置（JSON/Env/Args）、日志、HostedService     |
| Startup 支持 | `Startup` 类 => `ConfigureServices`、`Configure` | 无 `Startup`，服务配置与管道在 `Program.cs` 中完成 |
| 扩展灵活度   | 对 Web 优化，扩展槽较少                          | 核心抽象化，扩展性极强                             |

### 核心概念与关键接口

#### IHostBuilder & Host

* `Host.CreateDefaultBuilder(args)`

    * 加载默认配置源（`appsettings.json`、环境变量、命令行等）

    * 注册 `ILogger、IConfiguration、IHostEnvironment`

    * 注册 `IHostedService` 管理生命周期

* `IHostBuilder`

    * `.ConfigureAppConfiguration(...)`：自定义配置源

    * `.ConfigureServices(...)`：注册依赖注入服务

    * `.ConfigureLogging(...)`：配置日志提供器与级别

    * `.UseServiceProviderFactory(...)`：替换 `DI` 容器（如 `Autofac`）

* `IHost`

    * `.StartAsync() / .RunAsync()`：启动所有 `IHostedService`

    * `.StopAsync()`：优雅停止

    * `.Services`：获得顶级 `IServiceProvider`

#### 通用主机（Generic Host）常用扩展包

* Worker Service（`Microsoft.Extensions.Hosting.WindowsServices、…Linux`)(systemd)

* `gRPC、SignalR、Grain（Orleans）`等中间件

* 定时任务库：`IHostedService` 或 `BackgroundService`

####  IHostedService 与 BackgroundService

* `IHostedService`：定义 `StartAsync(CancellationToken)、StopAsync(CancellationToken)`

* `BackgroundService`：抽象类，重写 `ExecuteAsync(CancellationToken)`，自动注册为单例 `IHostedService`

```csharp
public class TimedWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            // 周期性执行
            await DoWorkAsync();
            await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
        }
    }
}
```

### 配置与启动流程解析

#### Program.cs（.NET 6+ 最简模板）

```csharp
var builder = Host.CreateDefaultBuilder(args)
    .ConfigureServices((hostCtx, services) =>
    {
        services.AddHostedService<TimedWorker>();
        // 其它服务注册
    })
    .ConfigureLogging(logging =>
    {
        logging.ClearProviders();
        logging.AddConsole();
    });

var host = builder.Build();
await host.RunAsync();
```

#### 配置加载顺序

* `appsettings.json`

* `appsettings.{Environment}.json`

* 环境变量（`ASPNETCORE_ENVIRONMENT`）

* 命令行参数

#### 依赖注入生命周期

* `Singleton`：与主机同生命周期

* `Scoped`：在 `Web` 场景中每次请求新创建；在非 `HTTP` 场景，仅在显式 `IServiceScope` 中有效

* `Transient`：每次注入新实例

#### 日志管道

* 默认支持 `Console、Debug、EventSource` 等

* 可接入 `Seq、ELK、Application Insights` 等第三方 `Provider`

#### 主机构建流程

* `CreateDefaultBuilder` 注册服务与配置

* 用户链式调用 `.ConfigureXxx()` 定制

* `Build()` 汇总并生成 `IHost`

* `RunAsync()` 启动注册的所有 `IHostedService`

### 最小主机WebApplication

#### 概念背景

* 最小主机（`Minimal Hosting`） 出现在 `.NET 6` 中，旨在通过减少样板代码，用几行代码就能启动一个 `Web` 应用。

* 核心目标是“开箱即用”：无需显式 `Startup` 类，也不用手动创建 `IHostBuilder` 和 `IWebHostBuilder`，而是统一通过 `WebApplicationBuilder` 构建。

#### 基本概念

`WebApplication` 是一个集成了主机配置、服务注册和请求处理管道的单一类型，它结合了早期版本中的 `HostBuilder、WebHostBuilder` 和 `Startup` 类的功能。

**核心特点包括：**

* 单文件结构：通常在 `Program.cs` 中完成所有配置

* 简化的依赖注入：直接通过 `WebApplicationBuilder` 注册服务

* 最小 `API`：无需控制器，直接定义路由处理逻辑

* 隐式中间件：自动包含常用中间件（如路由、静态文件）
 
#### 核心类型

| 类型                    | 作用                                                   |
| ----------------------- | ------------------------------------------------------ |
| `WebApplicationBuilder` | 构建器，负责加载配置、DI 注册、日志及 Kestrel 默认设置 |
| `WebApplication`        | 真正的应用主机，承载中间件管道、路由和运行生命周期     |
| `IEndpointRouteBuilder` | 定义路由与端点，支持最小 API 风格                      |

#### 典型 Program.cs 模板

```csharp
var builder = WebApplication.CreateBuilder(args);

// 1. 服务注册（依赖注入）
builder.Services.AddControllers();
builder.Services.AddDbContext<AppDbContext>(opt =>
    opt.UseSqlServer(builder.Configuration.GetConnectionString("Default")));

// 2. 构建应用
var app = builder.Build();

// 3. 中间件配置
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
app.UseHttpsRedirection();
app.UseRouting();
app.UseAuthorization();

// 4. 路由与端点
app.MapControllers();
app.MapGet("/ping", () => Results.Ok("pong"));

// 5. 启动
app.Run();
```

* `CreateBuilder`：加载 `appsettings.json`、环境变量、命令行等配置，注册日志与 `IHostEnvironment`。

* `Services`：在 `builder.Services` 上注册所需服务（`DbContext、Identity、Swagger、HttpClient` 等）。

* `Build`：生成 `WebApplication` 实例，此时 `DI` 容器与中间件管道均已准备完毕。

* `Middleware`：在 `app` 上按顺序配置请求管道。

* `Endpoints`：使用 `app.MapXXX` 定义路由。

* `Run`：启动 `Kestrel`（或 `IIS` 集成）并开始监听。

#### 依赖注入与配置

```csharp
var builder = WebApplication.CreateBuilder(args);

// 添加控制器支持
builder.Services.AddControllers();

// 添加 Swagger/OpenAPI 支持
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// 添加自定义服务
builder.Services.AddScoped<IMyService, MyService>();

var app = builder.Build();
```

```csharp
var builder = WebApplication.CreateBuilder(args);

// 读取配置值
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection");

// 绑定配置到强类型对象
var settings = builder.Configuration.GetSection("AppSettings").Get<AppSettings>();

// 添加配置源
builder.Configuration.AddJsonFile("appsettings.Development.json", optional: true);
```

* `DI` 容器

    * 默认是 `Microsoft.Extensions.DependencyInjection`，支持三种生命周期：`Singleton、Scoped、Transient`。

    * 在最小主机中，`builder.Services` 相当于传统 `ConfigureServices`，而 `app.Services` 可以在运行时获取构建好的 `IServiceProvider`。

* 配置（`Configuration`）

    * `builder.Configuration` 已包含 `appsettings.json、appsettings.{Env}.json`、环境变量、命令行。

    * 可通过 `builder.Configuration.GetSection("Section").Bind(myOptions)` 或者

    ```csharp
    builder.Services.Configure<MyOptions>(builder.Configuration.GetSection("MyOptions"));
    ```

    * 在端点中也可直接注入 `IConfiguration` 或已绑定的 `IOptions<T>`。

#### 中间件与管道

```csharp
// 配置中间件
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();

// 映射控制器
app.MapControllers();

// 或使用最小 API
app.MapGet("/greet/{name}", (string name) => $"Hello, {name}!");
```

* 在 `app` 上直接调用 `UseXxx` 即可。常见中间件：

    * `UseDeveloperExceptionPage()、UseHttpsRedirection()`

    * `UseAuthentication()、UseAuthorization()`

    * `UseCors()、UseStaticFiles()`

* 顺序重要：认证/授权应在路由映射(`Map`)前，静态文件一般要放在最前面或合适位置。

#### 最小 API（Minimal API）

* `MapGet/MapPost/…`：最简端点定义方式，无需控制器。

```csharp
// 定义路由和处理逻辑
app.MapGet("/users", () =>
{
    var users = new[] { 
        new { Id = 1, Name = "Alice" },
        new { Id = 2, Name = "Bob" } 
    };
    return Results.Ok(users);
});

app.MapPost("/users", (User user) =>
{
    // 处理用户创建逻辑
    return Results.Created($"/users/{user.Id}", user);
});

// 带参数的路由
app.MapGet("/users/{id}", (int id) =>
{
    // 查找用户逻辑
    var user = GetUserById(id);
    return user is not null ? Results.Ok(user) : Results.NotFound();
});
```

* 自动绑定：路径参数、查询参数、主体和服务会自动注入到委托参数。

* `Results`：用于生成各种 `HTTP` 响应，如 `Results.Ok(), Results.NotFound()` 等。

#### 依赖注入与服务使用

```csharp
app.MapGet("/todos", async (ITodoService todoService) =>
    await todoService.GetAllTodos());

app.MapGet("/todos/{id}", async (int id, ITodoService todoService) =>
    await todoService.GetTodoById(id)
        is Todo todo
            ? Results.Ok(todo)
            : Results.NotFound());
```

#### 环境感知

通过 `app.Environment` 可以获取当前环境信息：

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}
```

#### 启动与生命周期

* 启动钩子

    * `app.Lifetime.ApplicationStarted.Register(() => { /* 启动后 */ })`;

    * `ApplicationStopping、ApplicationStopped` 类似。

* 优雅关闭

    * `app.Run()` 默认会监听 `Ctrl+C（SIGINT）`和 `SIGTERM`，自动触发优雅停机。

* 后台任务

    * 在最小主机中同样可注册 `BackgroundService`：

    ```csharp
    builder.Services.AddHostedService<MyWorker>();
    ```
#### 高级特性

* 过滤器

可以为路由添加全局或特定的过滤器：

```csharp
// 全局过滤器
app.MapGet("/", () => "Hello World!")
   .AddEndpointFilter(async (context, next) =>
   {
       Console.WriteLine("Before endpoint execution");
       var result = await next(context);
       Console.WriteLine("After endpoint execution");
       return result;
   });
```

* 组路由

组织相关路由到组中：

```csharp
var usersGroup = app.MapGroup("/users");

usersGroup.MapGet("/", GetAllUsers);
usersGroup.MapGet("/{id}", GetUserById);
usersGroup.MapPost("/", CreateUser);
```

* 请求验证

```csharp
app.MapPost("/products", (ProductRequest request) =>
{
    if (!ModelState.IsValid)
    {
        return Results.ValidationProblem(ModelState);
    }
    
    // 处理有效请求
    return Results.Created($"/products/{request.Id}", request);
});

public class ProductRequest
{
    public int Id { get; set; }
    
    [Required]
    public string Name { get; set; }
    
    [Range(0, 100)]
    public decimal Price { get; set; }
}
```

* 自定义中间件

```csharp
app.Use(async (context, next) =>
{
    // 请求前处理
    var start = DateTime.UtcNow;
    
    await next.Invoke();
    
    // 请求后处理
    var duration = DateTime.UtcNow - start;
    logger.LogInformation($"Request took {duration.TotalMilliseconds}ms");
});
```

* 错误处理：

```csharp
   app.UseExceptionHandler(exceptionHandlerApp => 
   {
       exceptionHandlerApp.Run(async context => 
       {
           context.Response.StatusCode = 500;
           await context.Response.WriteAsJsonAsync(new { 
               Error = "An unexpected error occurred" 
           });
       });
   });
```

#### 与传统 Startup/Host 对比

| 特性         | 传统模式（.NET 5 及以前）                  | 最小主机（.NET 6+）            |
| ------------ | ------------------------------------------ | ------------------------------ |
| 样板代码     | `Host.CreateDefaultBuilder` + `Startup` 类 | `WebApplication.CreateBuilder` |
| 服务注册分散 | `ConfigureServices` in `Startup`           | 统一在 `builder.Services`      |
| 中间件配置   | `Configure` in `Startup`                   | 直接在 `app` 上链式调用        |
| 路由定义     | 控制器 + `MapControllers()`                | 最小 API + 控制器混合          |
| 可读性       | 分文件稍散                                 | 单文件、扁平化，更易快速浏览   |

#### 实战示例：带 Swagger 的最小 API

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.MapGet("/hello/{name}", (string name) => $"Hello, {name}!")
   .WithName("SayHello")
   .WithOpenApi();

app.Run();
```

#### 最佳实践

* 按功能拆分文件：当路由和服务增多时，可将端点拆到拓展方法，如 `app.MapUserEndpoints()`。

* 中间件有序：注意静态文件、异常处理、认证授权的正确顺序。

* `Options` 验证：使用 `services.AddOptions<T>().Bind(...).ValidateDataAnnotations()` 提前发现配置错误。

* 日志分级：在 `builder.Logging` 中清理或添加日志提供器，避免过多噪音。

* 单元测试：最小主机可通过 `WebApplicationFactory<TEntryPoint>` 在测试中轻松启动。


