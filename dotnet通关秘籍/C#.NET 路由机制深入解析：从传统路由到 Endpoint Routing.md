### 简介

路由是 `ASP.NET Core` 的核心基础设施，负责将 `HTTP` 请求映射到对应的处理程序（如控制器方法）。它决定了 `URL` 如何与应用程序代码交互，是现代 `Web` 开发的关键组件。

在 `ASP.NET Core` 中，路由系统解决了以下问题：

* `URL` 映射：将用户友好的 `URL` 映射到具体的处理程序。
 
* 灵活性：支持多种路由配置（如 `RESTful` 路径、动态参数）。
 
* 性能优化：高效解析请求，快速定位处理逻辑。
 
* 可扩展性：通过中间件和自定义路由约束扩展功能。
 
* 异步支持：与 `async/await` 无缝集成，适合现代 `Web` 应用。
 
#### 主要功能

* 基于约定的路由（`Convention-based Routing`）：
 
    * 使用模板（如 "`{controller}/{action}/{id?}`"）定义路由规则。
     
    * 适合传统 `MVC` 应用。

* 特性路由（`Attribute Routing`）：
 
    * 通过 `[Route]` 特性直接在控制器或动作上定义路由。
     
    * 适合 `RESTful API` 和复杂 `URL` 模式。

* 参数绑定：
 
    * 支持路由参数（如 `{id}`）、查询字符串、请求体等。
     
    * 支持可选参数、默认值和约束。

* 中间件集成：
 
    * 通过 `UseRouting` 和 `UseEndpoints` 中间件处理路由。
     
    * 支持自定义路由中间件。

* 路由约束：
 
    * 限制路由参数（如 `int、guid、regex`）。
     
    * 提高性能和安全性。

* 区域支持（`Areas`）：
 
    * 将控制器分组到不同区域（如 `Admin、User`）。

* 动态路由：
 
    * 支持动态生成路由（如基于数据库配置）。

* 端点路由：
 
    * 统一管理 `MVC、Razor Pages` 和 `SignalR` 的路由。

### 核心概念

| 名称                   | 说明                                                                   |
| ---------------------- | ---------------------------------------------------------------------- |
| **路由模板**           | 由文字、参数（`{id}`）、可选参数（`{id?}`）、默认值、约束组成的字符串  |
| **路由参数**           | 路径中的占位符，匹配 URL 段并绑定到 Action 方法参数                    |
| **路由数据**           | 匹配结果的键值对集合，可在中间件或 Controller 中通过 `RouteData` 访问  |
| **路由约束**           | 通过类型（`int`、`guid`）、正则（`regex(...)`）等限制参数匹配规则      |
| **默认值**             | 当 URL 中未提供参数时使用的值                                          |
| **终结点（Endpoint）** | 最终与请求匹配并执行的代码单元，如 MVC Action、Razor Page、Minimal API |

### 配置方式

#### 启用 Endpoint Routing

```csharp
// Program.cs （.NET 6+ Minimal Hosting）
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();   // MVC / API

var app = builder.Build();
app.UseRouting();                     // 启动路由中间件

app.UseAuthorization();               // 授权、认证等

app.UseEndpoints(endpoints =>
{
    endpoints.MapControllers();       // 特性路由 + 约定路由
    endpoints.MapRazorPages();        // Razor Pages
    endpoints.MapHub<ChatHub>("/hub"); // SignalR
    // endpoints.MapGet("/ping", () => "pong"); // Minimal API
});
app.Run();
```

#### 约定路由（Conventional Routing）

```csharp
builder.Services.AddControllersWithViews();
// ...
app.UseEndpoints(endpoints =>
{
    endpoints.MapControllerRoute(
        name: "default",
        pattern: "{controller=Home}/{action=Index}/{id?}");

    endpoints.MapAreaControllerRoute(
        name: "admin",
        areaName: "Admin",
        pattern: "Admin/{controller=Dashboard}/{action=Index}/{id?}");
});
```

* `pattern`：路由模板，包含默认值（`=Home`）和可选参数（?）。

* 在 `Controller` 中可不标注任何特性，直接按约定匹配 `URL`。

#### 特性路由（Attribute Routing）

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]                     // GET api/products
    public IActionResult GetAll() { … }

    [HttpGet("{id:int}")]        // GET api/products/5，且 id 必须是整数
    public IActionResult Get(int id) { … }

    [HttpPost("batch/{type?}")]   // POST api/products/batch 或 api/products/batch/special
    public IActionResult CreateBatch(string? type) { … }
}
```

* `[Route]、[HttpGet]、[HttpPost]` 等特性定义路由模板、方法限制。

* 支持在控制器和方法级别混用，覆盖或叠加。


### 路由模板详解

#### 模板语法元素

| 语法           |  示例   |  说明   |
|--------------| --- | --- |
| 字面值          |  `api/products`   | 固定匹配的路径段    |
| 参数 `{param}` |  `{controller}`   |  捕获值并绑定到参数   |
| 可选参数 `{id?}` |  `{id?}`   |  参数可选   |
|    默认值 `{id=5}`          |  `{page=1} `  |   未提供时的默认值  |
|     约束 `{id:int}`         |  `{id:min(1)}`   |   限制参数格式  |
|     通配符 `*`         |  `{*slug}`   |  捕获剩余路径   |
|     命名空间         |  `[Namespace("Admin")]`   |  控制器命名空间约束   |

#### 路由约束类型

| 约束  |示例   | 说明  |
|---|---|---|
| 类型约束  | `{id:int}`  | 必须是整数  |
| 范围约束  | `{age:range(18,99)}`  | 值在指定范围内  |
| 正则表达式  |  `{ssn:regex(^\\d{{3}}-\\d{{2}}-\\d{{4}}$)}` |  匹配正则模式 |
| 必需值  | `{name:required}`  | 必须提供非空值  |
| 自定义约束  |  `{code:validProductCode}` | 实现 `IRouteConstraint` 接口|

### 高级特性

#### 动态路由

基于数据库动态生成路由：

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Routing;
using System.Collections.Generic;
using System.Threading.Tasks;

public class DynamicRouteController : ControllerBase
{
    private readonly Dictionary<string, string> _routes = new()
    {
        { "page1", "Content for Page 1" },
        { "page2", "Content for Page 2" }
    };

    [HttpGet("{key}")]
    public IActionResult GetDynamic(string key)
    {
        if (_routes.TryGetValue(key, out var content))
            return Ok(content);
        return NotFound();
    }
}

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddControllers();
var app = builder.Build();
app.UseRouting();
app.UseEndpoints(endpoints => endpoints.MapControllers());
app.Run();
```

#### 多路由规则

```csharp
[Route("api/[controller]")]
[Route("v2/[controller]")] // 支持多个路由
public class ProductsController : ControllerBase
{
    [HttpGet("all")]
    [HttpGet("list")] // 多个路径映射同一方法
    public IActionResult ListProducts() { /*...*/ }
}
```

#### 路由参数转换

```csharp
[Route("products/{id:guid}")]
public IActionResult GetProductById(Guid id) { /*...*/ }

[Route("products/{slug:slugify}")] // 自定义 SlugConstraint
public IActionResult GetProductBySlug(string slug) { /*...*/ }
```

#### 端点元数据（Endpoint Metadata）

通过 `WithMetadata(...)` 或在特性上添加自定义属性，可为终结点附加元数据，在中间件中读取以执行额外逻辑（例如权限校验、文档生成）。

#### 路由优先级（Order）

特性路由和约定路由都可设置 `Order` 属性，高优先级先匹配：

```csharp
[Route("special", Order = 1)]
public IActionResult Special() { … }

[Route("{**catchAll}", Order = 100)]
public IActionResult CatchAll() { … }
```

#### 端点过滤器（Endpoint Filter，.NET 7+）

在 `Minimal APIs` 或 `Controller` 上，可注册 `Endpoint Filter` 以在终结点执行前后插入逻辑，相当于细粒度中间件。

#### 最小 API 路由

```csharp
app.MapGet("/weather/{day:datetime}", (DateTime day) =>
    $"Weather for {day:yyyy-MM-dd}");
```

* 直接在 `WebApplication` 上映射，既是特性路由也是约定路由的简化版。

* 支持参数绑定、依赖注入、请求处理器委托等。

#### 自定义路由约束

实现 `IRouteConstraint` 接口，注册到路由选项中，即可在模板里使用自定义约束

### 路由诊断与调试

#### 路由信息中间件

```csharp
app.UseEndpoints(endpoints => { /*...*/ });

// 添加诊断中间件
app.Use(async (context, next) =>
{
    var endpoint = context.GetEndpoint();
    if (endpoint != null)
    {
        Console.WriteLine($"Endpoint: {endpoint.DisplayName}");
        Console.WriteLine($"RoutePattern: {endpoint.Metadata.GetMetadata<RoutePattern>()}");
    }
    await next();
});
```

### 优缺点

优点

* 灵活性：支持约定路由和特性路由，适应多种场景。

* 性能高：基于端点路由，匹配速度快。

* 异步友好：支持 `async/await`，适合现代 `Web`。

* 可扩展：支持自定义约束、中间件和动态路由。

* `DI` 集成：与 `ASP.NET Core DI` 无缝结合。

缺点

* 配置复杂：复杂路由规则需仔细设计，避免冲突。

* 学习曲线：特性路由和约束需要熟悉语法。

* 调试难度：路由冲突或错误需调试工具（如日志）。

* 进程内限制：路由配置不跨实例，需结合外部配置。

### 资源和文档

* 官方文档：

    * `Microsoft Learn`：https://learn.microsoft.com/en-us/aspnet/core/fundamentals/routing

    * `ASP.NET Core MVC`：https://learn.microsoft.com/en-us/aspnet/core/mvc/controllers/routing

* `NuGet` 包：https://www.nuget.org/packages/Microsoft.AspNetCore.Mvc.Core

* `GitHub`：https://github.com/dotnet/aspnetcore