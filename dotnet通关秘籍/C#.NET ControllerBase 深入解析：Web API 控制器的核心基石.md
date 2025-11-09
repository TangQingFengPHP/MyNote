### 简介

`ControllerBase` 是 `ASP.NET Core` 中构建 `Web API` 控制器的基类，位于 `Microsoft.AspNetCore.Mvc` 命名空间。它提供了丰富的功能来处理 `HTTP` 请求，但不包含视图支持。

核心功能：

* `HTTP` 响应：提供方法（如 `Ok、NotFound`）生成标准 `HTTP` 响应。

* 模型绑定：自动将请求数据绑定到参数（如查询字符串、请求体）。

* 验证支持：结合 `[ApiController]` 实现自动模型验证。

* 异步友好：支持 `async/await` 处理高并发请求。

* 轻量设计：不包含 `MVC` 视图相关的功能，适合 `API` 场景。

### ControllerBase 的定位

* 位于命名空间 `Microsoft.AspNetCore.Mvc`，是所有 `Web API` 控制器的基类。

* 与 MVC 的 `Controller` 相比，`ControllerBase` 不包含视图（`Razor`）相关功能，仅提供 内容结果、状态码、路由绑定、模型验证 等 `Web API` 核心支持。

* 通常搭配 `[ApiController]` 特性使用，开启自动模型验证、参数绑定规则和返回行为。

### 继承体系

```csharp
object
 └─ ControllerBase
     ├─ Controller         // 包含 View()、PartialView() 等 MVC 视图方法
     └─ 用户自定义 Controllers…
```

* `ControllerBase`

    * 提供结果工厂方法（`Ok(), CreatedAtAction()` 等）

    * 包含属性如 `Request、Response、ModelState、User`

    * 实现接口 `IActionFilterMetadata、IActionResult` 等元数据标记

* `Controller`（典型用于 `MVC`）

    * 继承自 `ControllerBase` 并额外添加视图渲染方法

### 构造注入与内置属性

在自定义 `API` 控制器中，通常这样声明：

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductService _svc;
    public ProductsController(IProductService svc)
    {
        _svc = svc;
    }
    // Action 方法…
}
```

* `[ApiController]`

    * 自动执行模型验证，错误时直接返回 400

    * 简化参数绑定（如 `[FromBody]、[FromQuery]` 默认行为）

* `Route、HttpXXX` 特性

    * 定义路由模板与 `HTTP` 动作要匹配的请求

#### 常用内置属性

| 属性          | 说明                             |
| ------------- | -------------------------------- |
| `HttpContext` | 当前请求上下文                   |
| `Request`     | 快速访问 `HttpContext.Request`   |
| `Response`    | 快速访问 `HttpContext.Response`  |
| `User`        | 当前认证用户的 `ClaimsPrincipal` |
| `RouteData`   | 路由参数及其绑定值               |
| `ModelState`  | 模型验证状态与错误详情           |


### 结果工厂方法

`ControllerBase` 提供一系列返回 `ActionResult` 的方法，用于快速构造 `HTTP` 响应：

#### 成功响应

| 方法                                           | HTTP 状态码 | 说明                                    |
| ---------------------------------------------- | ----------- | --------------------------------------- |
| `Ok()` / `Ok(object value)`                    | 200         | 返回 200，带或不带响应体                |
| `Created(string uri, object value)`            | 201         | 资源已创建，Location 头部指定新资源 URI |
| `CreatedAtAction(...)` / `CreatedAtRoute(...)` | 201         | 指定路由或动作生成 Location             |
| `NoContent()`                                  | 204         | 成功但无响应体                          |

#### 客户端错误

| 方法                                        | HTTP 状态码 | 说明                             |
| ------------------------------------------- | ----------- | -------------------------------- |
| `BadRequest()` / `BadRequest(object error)` | 400         | 返回模型验证错误或自定义错误消息 |
| `Unauthorized()`                            | 401         | 未认证                           |
| `Forbid()`                                  | 403         | 无访问权限                       |
| `NotFound()` / `NotFound(object value)`     | 404         | 资源未找到                       |
| `Conflict()`                                | 409         | 冲突，如重复资源                 |
| `UnsupportedMediaType()`                    | 415         | 不支持的媒体类型                 |

#### 服务端错误

| 方法                         | HTTP 状态码 | 说明                             |
| ---------------------------- | ----------- | -------------------------------- |
| `StatusCode(int statusCode)` | 自定义      | 返回任意状态码                   |
| `Problem()`                  | 500+        | 按 RFC 7807 生成标准化错误响应体 |

> Tip: 这些工厂方法本质上返回了实现 IActionResult 的对象，框架在管道末端执行并写入 HTTP。

### 核心功能详解

#### HTTP 响应方法

状态码响应

```csharp
[HttpGet("{id}")]
public IActionResult GetProduct(int id)
{
    var product = _repository.Get(id);
    
    if (product == null)
        return NotFound(); // 404
    
    return Ok(product); // 200 + 数据
}
```

文件响应

```csharp
[HttpGet("download/{fileName}")]
public IActionResult DownloadFile(string fileName)
{
    var filePath = Path.Combine(_env.ContentRootPath, "Files", fileName);
    
    if (!System.IO.File.Exists(filePath))
        return NotFound();
    
    var fileBytes = System.IO.File.ReadAllBytes(filePath);
    return File(fileBytes, "application/octet-stream", fileName);
}
```

#### 请求上下文访问

常用属性

```csharp
public class ProductsController : ControllerBase
{
    [HttpGet("info")]
    public IActionResult GetInfo()
    {
        // 获取请求信息
        var method = Request.Method;
        var contentType = Request.ContentType;
        
        // 获取用户信息
        var userName = User.Identity?.Name;
        var isAdmin = User.IsInRole("Admin");
        
        // 获取路由数据
        var routeId = RouteData.Values["id"];
        
        return Ok(new { method, userName });
    }
}
```

#### 模型绑定与验证

* 自动验证（启用 `[ApiController]` 时）

    * 在 `Action` 执行前对标注了验证特性的模型进行验证

    * `ModelState.IsValid == false` 则自动返回 400 `Bad Request`，响应体包含错误详情

```csharp
[HttpPost]
public IActionResult CreateProduct(
    [FromBody] Product product, // 从JSON绑定
    [FromQuery] string category, // 从查询字符串绑定
    [FromHeader] string apiKey) // 从请求头绑定
{
    // 自动模型验证
    if (!ModelState.IsValid)
    {
        return BadRequest(ModelState);
    }
    
    // 业务逻辑处理
    var createdProduct = _service.Create(product, category);
    
    return CreatedAtAction(nameof(GetProduct), 
        new { id = createdProduct.Id }, createdProduct);
}
```

### 高级功能与最佳实践

#### 操作结果包装

自定义统一响应格式

```csharp
public class ApiResult<T> : IActionResult
{
    public T Data { get; }
    public int StatusCode { get; }
    public string? Message { get; }

    public ApiResult(T data, int statusCode = 200, string? message = null)
    {
        Data = data;
        StatusCode = statusCode;
        Message = message;
    }

    public async Task ExecuteResultAsync(ActionContext context)
    {
        var result = new ObjectResult(new
        {
            Success = StatusCode >= 200 && StatusCode < 300,
            Data,
            Message,
            Timestamp = DateTime.UtcNow
        })
        {
            StatusCode = StatusCode
        };

        await result.ExecuteResultAsync(context);
    }
}

// 在控制器中使用
[HttpGet("{id}")]
public ApiResult<Product> GetProduct(int id)
{
    var product = _repository.Get(id);
    return product == null 
        ? new ApiResult<Product>(null, 404, "Product not found") 
        : new ApiResult<Product>(product);
}
```

#### 内容协商

支持多种响应格式

```csharp
[ApiController]
[Route("api/[controller]")]
[Produces("application/json", "application/xml")] // 支持的响应类型
public class ProductsController : ControllerBase
{
    [HttpGet]
    public ActionResult<IEnumerable<Product>> GetAll()
    {
        return _repository.GetAll();
    }
}
```

#### API 版本控制

实现 `API` 版本管理

```csharp
[ApiController]
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    [MapToApiVersion("1.0")]
    public IEnumerable<Product> GetV1()
    {
        // 版本1.0的实现
    }

    [HttpGet]
    [MapToApiVersion("2.0")]
    public ActionResult<IEnumerable<ProductDto>> GetV2()
    {
        // 版本2.0的实现
    }
}
```

#### 异步支持

* `ControllerBase` 建议写 异步 `Action`，返回 `Task<IActionResult>` 或直接 `Task<T>`（结合泛型 `ActionResult<T>`）：

```csharp
[HttpGet("{id}")]
public async Task<ActionResult<Product>> Get(int id)
{
    var p = await _svc.FindAsync(id);
    return p is null ? NotFound() : Ok(p);
}
```

* `ActionResult<T>`（.NET Core 2.1+）

    * 允许方法既返回 `T`（自动包装为 200）也返回 `ActionResult`（可自定义状态码）。

### 过滤器与管道

* `ControllerBase` 支持注册和应用各种 过滤器：

    * 授权过滤器（`[Authorize]`）

    * 资源过滤器（`IResourceFilter`）

    * 动作过滤器（`IActionFilter`）

    * 异常过滤器（`IExceptionFilter`）

    * 结果过滤器（`IResultFilter`）

* 示例：全局注册

```csharp
services.AddControllers(options =>
{
    options.Filters.Add<GlobalExceptionFilter>();
});
```

* 示例：单控制器/Action 应用

```csharp
[TypeFilter(typeof(LogActionFilter))]
public class MyController : ControllerBase { … }
```

### 安全最佳实践

#### 身份验证与授权

```csharp
[ApiController]
[Authorize] // 控制器级别要求认证
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    [HttpGet]
    [Authorize(Roles = "Admin")] // 要求管理员角色
    public IActionResult GetAllOrders() { /* ... */ }

    [HttpGet("my")]
    public IActionResult GetMyOrders()
    {
        // 获取当前用户ID
        var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
        return Ok(_repository.GetOrdersByUser(userId));
    }

    [HttpPost]
    [Authorize(Policy = "RequireVerifiedEmail")] // 使用自定义策略
    public IActionResult CreateOrder([FromBody] Order order) { /* ... */ }
}
```

#### 跨域资源共享 (CORS)

```csharp
[ApiController]
[Route("api/[controller]")]
[EnableCors("AllowSpecificOrigin")] // 启用CORS策略
public class PublicDataController : ControllerBase
{
    [HttpGet]
    public IActionResult GetPublicData() { /* ... */ }
}

// 在Startup中配置CORS策略
services.AddCors(options =>
{
    options.AddPolicy("AllowSpecificOrigin",
        builder => builder.WithOrigins("https://example.com")
                          .AllowAnyMethod()
                          .AllowAnyHeader());
});
```

### 常见问题解决方案

#### 模型验证统一处理

```csharp
// 创建统一验证过滤器
public class ValidateModelAttribute : ActionFilterAttribute
{
    public override void OnActionExecuting(ActionExecutingContext context)
    {
        if (!context.ModelState.IsValid)
        {
            context.Result = new BadRequestObjectResult(new
            {
                Code = 400,
                Message = "参数验证失败",
                Errors = context.ModelState
                    .SelectMany(m => m.Value.Errors)
                    .Select(e => e.ErrorMessage)
            });
        }
    }
}

// 全局注册
services.AddControllers(options => 
{
    options.Filters.Add<ValidateModelAttribute>();
});
```

#### 全局异常处理

```csharp
public class ApiExceptionFilter : IExceptionFilter
{
    public void OnException(ExceptionContext context)
    {
        var exception = context.Exception;
        var statusCode = exception switch
        {
            NotFoundException => 404,
            ValidationException => 400,
            UnauthorizedAccessException => 401,
            _ => 500
        };

        context.Result = new ObjectResult(new
        {
            Code = statusCode,
            Message = exception.Message
        })
        {
            StatusCode = statusCode
        };
        
        context.ExceptionHandled = true;
    }
}
```

### ControllerBase 最佳实践总结

#### API 设计规范

```csharp
// 使用HTTP动词特性
[HttpGet("{id}")]
[HttpPost]
[HttpPut("{id}")]
[HttpDelete("{id}")]
```

#### 响应类型声

```csharp
[ProducesResponseType(StatusCodes.Status200OK, Type = typeof(Product))]
[ProducesResponseType(StatusCodes.Status404NotFound)]
public IActionResult GetProduct(int id) { /* ... */ }
```

#### 异步优先

```csharp
public async Task<IActionResult> Get() { /* ... */ }
```

#### 依赖注入

```csharp
public MyController(IMyService service) { /* ... */ }
```

#### 安全加固

```csharp
[Authorize]
[ValidateAntiForgeryToken]
[EnableCors("SafePolicy")]
```

#### 版本管理

```csharp
[ApiVersion("1.0")]
[Route("api/v{version:apiVersion}/[controller]")]
```

#### 统一响应格式

```csharp
return Ok(new { Data = result, Message = "Success" });
```

### 资源和文档

* 官方文档：

    * `Microsoft Learn`：https://learn.microsoft.com/en-us/aspnet/core/mvc/controllers

    * `ControllerBase`：https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.controllerbase

* `NuGet` 包：https://www.nuget.org/packages/Microsoft.AspNetCore.Mvc.Core

* `GitHub`：https://github.com/dotnet/aspnetcore