### 简介

中间件（`Middleware`） 是 `ASP.NET Core` 的核心组件，用于处理 `HTTP` 请求和响应的管道机制。它是基于管道模型的轻量级、模块化设计，允许开发者在请求处理过程中插入自定义逻辑。中间件广泛应用于日志记录、认证授权、异常处理、路由等场景。

* 定义：中间件是处理 `HTTP` 请求和响应的组件，位于服务器接收到请求到最终返回响应之间的“管道”中。

* 作用：可用于身份认证、授权、日志、静态文件、异常处理、`CORS`、压缩、路由等横切关注点。

* 职责链：请求依次经过每个中间件，执行“前置逻辑”→调用下一个中间件→执行“后置逻辑”，最终形成责任链。

### 请求管道（Request Pipeline）

* 构建：在 `Startup.Configure(IApplicationBuilder app)` 中，通过 `app.UseXXX()、app.Run()` 等方法，按注册顺序构建一条 `RequestDelegate` 链。

* 执行：框架在收到 `HTTP` 请求时，从管道头部（第一个中间件）开始执行，直到找到终结点（`UseEndpoints/Run`）或管道结束。

* 短路：中间件可以选择不调用 `next()`，直接生成响应，阻断后续中间件。

* 请求流程：Client → Middleware1 → Middleware2 → ... → Endpoint

* 响应流程：Endpoint → ... → Middleware2 → Middleware1 → Client

```css
HTTP → [Middleware A] → [Middleware B] → [Middleware C] → Response
               ↑                ↓
         前置逻辑 A        后置逻辑 B
```

### 请求委托（RequestDelegate）

每个中间件通过 `RequestDelegate` 封装处理逻辑，接收 `HttpContext` 并异步执行操作

```csharp
public async Task InvokeAsync(HttpContext context, RequestDelegate next)
{
    // 预处理请求
    await next(context); // 调用下一个中间件
    // 后处理响应
}
```

### 短路（Short-Circuiting）

中间件可通过不调用 `next()` 终止管道，直接返回响应（如静态文件中间件找到文件时）

### 中间件的注册与类型

#### 注册方式

通过 `IApplicationBuilder` 在 `Program.cs` 或 `Startup.Configure` 中注册，顺序决定执行优先级

|  方法   |  方法   |  示例   |
| --- | --- | --- |
|  `Use`   |  添加可调用下一个中间件的组件   |  `app.Use(async (ctx, next) => { ... await next(); ... })`   |
|  `Run`   |  添加终端中间件（强制短路）   |  `app.Run(ctx => ctx.Response.WriteAsync("End"))`   |
|  `Map`   |  根据路径分支管道   |  `app.Map("/admin", branch => { ... })`   |
|  `MapWhen`   |  按条件分支管道   |  `app.MapWhen(ctx => ctx.Request.Query.ContainsKey("log"), branch => ...)`   |

#### 生命周期管理

* 单例构造：中间件类在应用启动时实例化（构造函数仅执行一次）

* 按请求执行：`Invoke` 或 `InvokeAsync` 方法对每个请求调用一次

### 自定义中间件实现

#### 基本模板

```csharp
public class MyMiddleware
{
    private readonly RequestDelegate _next;

    public MyMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        // 前置逻辑
        // e.g. 日志：Console.WriteLine("Request start: " + context.Request.Path);

        await _next(context);  // 调用下一个中间件

        // 后置逻辑
        // e.g. 日志：Console.WriteLine("Request end: " + context.Response.StatusCode);
    }
}
```

#### 扩展方法

```csharp
public static class MyMiddlewareExtensions
{
    public static IApplicationBuilder UseMyMiddleware(this IApplicationBuilder app)
    {
        return app.UseMiddleware<MyMiddleware>();
    }
}
```

在 `Startup.Configure` 或 `Program` 中调用：

```csharp
app.UseMyMiddleware();
```

### 内置中间件概览

| 中间件                                              | 功能                                     |
| --------------------------------------------------- | ---------------------------------------- |
| `UseRouting` / `UseEndpoints`                       | 路由匹配与终结点执行                     |
| `UseStaticFiles`                                    | 提供静态文件服务                         |
| `UseAuthentication`                                 | 验证用户身份                             |
| `UseAuthorization`                                  | 授权策略检查                             |
| `UseCors`                                           | 跨域请求支持                             |
| `UseResponseCompression`                            | 响应内容压缩                             |
| `UseSession`                                        | 会话状态管理                             |
| `UseExceptionHandler` / `UseDeveloperExceptionPage` | 全局异常捕获与开发环境错误页             |
| `UseHttpsRedirection`                               | HTTP 自动重定向到 HTTPS                  |
| `UseForwardedHeaders`                               | X-Forwarded-For / X-Forwarded-Proto 处理 |

典型的注册顺序：

```csharp
app.UseExceptionHandler("/Error");      // 异常处理（最先）
app.UseHttpsRedirection();             // HTTPS 重定向
app.UseStaticFiles();                  // 静态文件
app.UseRouting();                      // 路由
app.UseAuthentication();               // 认证
app.UseAuthorization();                // 授权
app.UseEndpoints(endpoints => {        // 端点处理（最后）
    endpoints.MapControllers();
});
```

### 注册顺序与注意事项

* 异常处理：

    * `UseDeveloperExceptionPage`（开发环境）或 `UseExceptionHandler` 应放在最前面，捕获后续所有异常。

* `HTTPS` 重定向：

    * `UseHttpsRedirection` 需在路由、静态文件之前。

* 静态文件：

    * 若站点静态资源较多，应尽早调用 `UseStaticFiles`，避免无谓路由匹配。

* 路由与终结点：

    * 必须在 `UseRouting` 后调用 `UseAuthentication/UseAuthorization`，再调用 `UseEndpoints`。

示例：

```csharp
public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    if (env.IsDevelopment())
        app.UseDeveloperExceptionPage();
    else
        app.UseExceptionHandler("/Home/Error");

    app.UseHttpsRedirection();
    app.UseStaticFiles();

    app.UseRouting();

    app.UseAuthentication();
    app.UseAuthorization();

    app.UseEndpoints(endpoints =>
    {
        endpoints.MapControllers();
        endpoints.MapRazorPages();
    });
}
```

### 终结点与分支

* 终结点（`Endpoint`）：`UseEndpoints` 中注册具体的请求处理，如 `MVC` 控制器、`Razor` 页面、`SignalR、gRPC`。

* 分支管道：`app.Map("/path", branch => { /* branch.UseXXX() */ })`;

    * 根据路径或条件，将请求“分流”到不同的子管道。

```csharp
app.Map("/api", apiApp =>
{
    apiApp.UseRouting();
    apiApp.UseAuthorization();
    apiApp.UseEndpoints(endpoints => endpoints.MapControllers());
});
```

### 进阶用法

#### 条件执行

```csharp
app.UseWhen(ctx => ctx.Request.Path.StartsWithSegments("/mobile"), mobileApp =>
{
    mobileApp.UseMiddleware<MobileSpecificMiddleware>();
});
```

#### 中间件短路

```csharp
app.Use(async (ctx, next) =>
{
    if (ctx.Request.Path == "/health")
    {
        ctx.Response.StatusCode = 200;
        await ctx.Response.WriteAsync("OK");
        return;  // 不调用 next，直接返回
    }

    await next();
});
```

#### 捕获并处理异常

```csharp
public class ErrorHandlingMiddleware
{
    private readonly RequestDelegate _next;
    public ErrorHandlingMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext ctx)
    {
        try
        {
            await _next(ctx);
        }
        catch (Exception ex)
        {
            ctx.Response.StatusCode = 500;
            await ctx.Response.WriteAsync("Internal Server Error");
            // TODO: 日志记录 ex
        }
    }
}
```

#### 分支中间件（Map/MapWhen）

```csharp
// 基于路径分支
app.Map("/api", apiApp => {
    apiApp.UseAuthentication();
    apiApp.UseEndpoints(endpoints => {
        endpoints.MapControllers();
    });
});

// 基于条件分支
app.MapWhen(context => context.Request.Query.ContainsKey("debug"), debugApp => {
    debugApp.UseMiddleware<DebugLoggingMiddleware>();
});
```

#### 终结点中间件（Run）

```csharp
// 直接终止请求管道，不调用后续中间件
app.Run(async context => {
    await context.Response.WriteAsync("Hello from terminal middleware!");
});
```

#### 中间件与依赖注入一

* 在中间件中使用服务

```csharp
public class AuthenticationMiddleware {
    private readonly RequestDelegate _next;
    private readonly IAuthService _authService;

    public AuthenticationMiddleware(RequestDelegate next, IAuthService authService) {
        _next = next;
        _authService = authService;
    }

    public async Task InvokeAsync(HttpContext context) {
        var token = context.Request.Headers["Authorization"];
        var user = await _authService.ValidateToken(token);
        context.User = user;
        await _next(context);
    }
}
```

* 作用域服务注入

```csharp
public async Task InvokeAsync(HttpContext context) {
    // 手动解析作用域服务
    using (var scope = context.RequestServices.CreateScope()) {
        var scopedService = scope.ServiceProvider.GetRequiredService<IScopedService>();
        await scopedService.ProcessRequest(context);
    }
    
    await _next(context);
}
```

#### 带配置的中间件

```csharp
public class CustomHeaderMiddleware
{
    private readonly RequestDelegate _next;
    private readonly string _headerValue;
    
    public CustomHeaderMiddleware(
        RequestDelegate next, 
        string headerValue) // 配置参数
    {
        _next = next;
        _headerValue = headerValue;
    }
    
    public async Task Invoke(HttpContext context)
    {
        context.Response.Headers["X-Custom"] = _headerValue;
        await _next(context);
    }
}

// 注册时传递配置
app.UseMiddleware<CustomHeaderMiddleware>("MyValue");
```

#### 中间件工厂 (IMiddlewareFactory)

```csharp
public class CustomMiddlewareFactory : IMiddlewareFactory
{
    public IMiddleware Create(Type middlewareType)
    {
        return ActivatorUtilities.CreateInstance(_serviceProvider, middlewareType) as IMiddleware;
    }

    public void Release(IMiddleware middleware) => (middleware as IDisposable)?.Dispose();
}

// 注册工厂
builder.Services.AddSingleton<IMiddlewareFactory, CustomMiddlewareFactory>();
```

#### 中间件依赖注入二

```csharp
public class MyMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IUserService _userService;

    public MyMiddleware(RequestDelegate next, IUserService userService)
    {
        _next = next;
        _userService = userService;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var userName = _userService.GetUserName(1);
        await context.Response.WriteAsync($"User: {userName}");
        await _next(context);
    }
}

public static class MyMiddlewareExtensions
{
    public static IApplicationBuilder UseMyMiddleware(this IApplicationBuilder builder)
    {
        return builder.UseMiddleware<MyMiddleware>();
    }
}

// 注册服务
builder.Services.AddScoped<IUserService, UserService>();
app.UseMyMiddleware();
```

### 内置中间件

#### 内置中间件

```csharp
app.UseRouting(); // 路由匹配

app.UseEndpoints(endpoints =>
{
    endpoints.MapControllers(); // API控制器
    endpoints.MapRazorPages();  // Razor页面
    endpoints.MapHub<ChatHub>("/chat"); // SignalR
});
```

#### 异常处理中间件

```csharp
app.UseExceptionHandler(errorApp =>
{
    errorApp.Run(async context =>
    {
        var exceptionHandler = context.Features.Get<IExceptionHandlerFeature>();
        // 记录异常
        await context.Response.WriteAsJsonAsync(new 
        {
            Error = exceptionHandler.Error.Message
        });
    });
});
```

#### 静态文件中间件

```csharp
app.UseStaticFiles(new StaticFileOptions 
{
    FileProvider = new PhysicalFileProvider(
        Path.Combine(builder.Environment.ContentRootPath, "Assets")),
    RequestPath = "/static",
    OnPrepareResponse = ctx => 
    {
        ctx.Context.Response.Headers.Append("Cache-Control", "public,max-age=3600");
    }
});
```