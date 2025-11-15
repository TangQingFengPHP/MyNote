### 简介

全局异常拦截是构建健壮企业级应用的关键基础设施，它能统一处理系统中未捕获的异常，提供友好的错误响应，同时记录完整的异常信息。

### 背景和作用

在 `ASP.NET Core` 应用中，异常可能在控制器、数据库操作或中间件中发生。如果每个动作方法都手动处理异常（如 `try-catch`），代码会变得冗长且难以维护。全局异常拦截器解决了以下问题：

* 统一错误处理：集中捕获所有未处理异常，返回标准化的错误响应。

* 标准化响应：符合 `RESTful API` 规范（如 `RFC 7807 Problem Details`）。

* 日志记录：记录异常详情，便于调试和监控。

* 用户体验：返回友好的错误信息，而非默认错误页面或堆栈跟踪。

* 性能优化：减少重复的异常处理代码，提升开发效率。

### 主要功能

* 捕获异常：捕获控制器、服务层或其他代码中的未处理异常。

* 标准化响应：返回 `JSON` 格式的错误详情（如状态码、错误消息）。

* 日志记录：记录异常信息（包括堆栈跟踪）到日志系统。

* 自定义处理：根据异常类型返回不同状态码或消息（如 400、409、500）。

* 异步支持：兼容 `async/await`，适合异步操作。

* `DI` 集成：通过依赖注入访问服务（如日志、缓存）。

### 常见实现方式

#### 异常处理中间件（推荐）

在 `ASP.NET Core` 管道最前端注册一个中间件，捕获所有后续中间件/终结点抛出的异常。

```csharp
public class ExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ExceptionHandlingMiddleware> _logger;

    public ExceptionHandlingMiddleware(RequestDelegate next,
        ILogger<ExceptionHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext ctx)
    {
        try
        {
            await _next(ctx);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Unhandled exception");
            await HandleExceptionAsync(ctx, ex);
        }
    }

    private static Task HandleExceptionAsync(HttpContext ctx, Exception ex)
    {
        ctx.Response.ContentType = "application/problem+json";
        ctx.Response.StatusCode = ex switch
        {
            ArgumentException _ => StatusCodes.Status400BadRequest,
            KeyNotFoundException _ => StatusCodes.Status404NotFound,
            _ => StatusCodes.Status500InternalServerError
        };

        var problem = new ProblemDetails
        {
            Title = ex.Message,
            Status = ctx.Response.StatusCode,
            Detail  = ctx.Response.StatusCode == 500 ? "请稍后重试或联系管理员" : null,
            Instance = ctx.Request.Path
        };
        var json = JsonSerializer.Serialize(problem);
        return ctx.Response.WriteAsync(json);
    }
}
```

注册中间件

在 `Program.cs`（或 `Startup.cs`）中最早添加：

```csharp
app.UseMiddleware<ExceptionHandlingMiddleware>();

// 或更简洁的扩展方法
app.UseExceptionHandling();  // 需自行实现 UseExceptionHandling 扩展
```

优势

* 捕获范围最大：包括所有 `MVC、Minimal API`、静态文件等。

* 性能开销小，代码集中清晰。

* 易于与依赖注入、日志系统联动。

#### 全局异常过滤器（IExceptionFilter / IAsyncExceptionFilter）

如果只想拦截 MVC Controller 的异常，可实现异常过滤器。

```csharp
public class GlobalExceptionFilter : IExceptionFilter
{
    private readonly ILogger<GlobalExceptionFilter> _logger;
    public GlobalExceptionFilter(ILogger<GlobalExceptionFilter> logger)
        => _logger = logger;

    public void OnException(ExceptionContext context)
    {
        var ex = context.Exception;
        _logger.LogError(ex, "Unhandled exception in controller");

        var problem = new ProblemDetails
        {
            Title = "请求失败",
            Status = StatusCodes.Status500InternalServerError,
            Detail = ex.Message
        };
        context.Result = new ObjectResult(problem)
        {
            StatusCode = problem.Status
        };
        context.ExceptionHandled = true;
    }
}
```

注册过滤器

```csharp
services.AddControllers(options =>
{
    options.Filters.Add<GlobalExceptionFilter>();
});
```

局限

* 只拦截通过 `MVC` 管道执行的 `Action` 抛出的异常。

* 不会捕获例如中间件或 `Minimal API` 的异常。

#### .NET 7+ 新增的 IExceptionHandler（推荐）

```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;
    private readonly IProblemDetailsService _problemDetailsService;

    public GlobalExceptionHandler(
        ILogger<GlobalExceptionHandler> logger,
        IProblemDetailsService problemDetailsService)
    {
        _logger = logger;
        _problemDetailsService = problemDetailsService;
    }

    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        _logger.LogError(exception, "全局异常: {Message}", exception.Message);
        
        var statusCode = GetStatusCode(exception);
        var problemContext = new ProblemDetailsContext
        {
            HttpContext = httpContext,
            Exception = exception,
            ProblemDetails = new ProblemDetails
            {
                Title = "服务器错误",
                Status = statusCode,
                Detail = httpContext.RequestServices.GetRequiredService<IWebHostEnvironment>().IsDevelopment() 
                    ? exception.ToString() 
                    : "请稍后再试",
                Type = $"https://httpstatuses.io/{statusCode}"
            }
        };
        
        // 添加自定义扩展
        problemContext.ProblemDetails.Extensions.Add("requestId", httpContext.TraceIdentifier);
        
        await _problemDetailsService.WriteAsync(problemContext);
        return true; // 标记为已处理
    }
}
```

注册服务：

```csharp
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails(); // 添加ProblemDetails支持

var app = builder.Build();
app.UseExceptionHandler(); // 启用异常处理中间件
```

#### 内置 UseExceptionHandler

`ASP.NET Core` 自带的异常处理终结点：

```csharp
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler(errorApp =>
    {
        errorApp.Run(async ctx =>
        {
            var feature = ctx.Features.Get<IExceptionHandlerFeature>();
            var ex = feature?.Error;
            // 记录日志…
            ctx.Response.StatusCode = 500;
            await ctx.Response.WriteAsJsonAsync(new { message = "服务器内部错误" });
        });
    });
}
else
{
    app.UseDeveloperExceptionPage();
}
```

* 优点：无需自定义 `Middleware`，框架内置。

* 注意：要在其他中间件之前注册，并在生产/开发环境中分开处理。

### 资源和文档

* 官方文档：

    * `Exception Handling`：https://learn.microsoft.com/en-us/aspnet/core/fundamentals/error-handling

    * `Problem Details`：https://learn.microsoft.com/en-us/aspnet/core/web-api/handle-errors