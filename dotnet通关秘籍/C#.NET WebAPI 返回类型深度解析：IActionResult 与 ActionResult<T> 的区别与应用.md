### 简介

### 核心概念对比

|  特性   |  `IActionResult`   |  `ActionResult<T>`   |
| --- | --- | --- |
|  引入版本   |  ASP.NET Core 1.0   |  ASP.NET Core 2.1   |
|  主要用途   |  表示HTTP响应（状态码+内容）   |  类型化HTTP响应   |
|  返回值类型   |  接口（多种实现）   |  泛型类   |
|  内容类型安全   |  ❌ 无编译时检查   |  ✅ 编译时类型检查   |
|  `OpenAPI/Swagger`   |  需手动添加 `[ProducesResponseType]`   |  自动推断响应类型   |
|  适用场景   |  需要灵活返回多种响应的场景   |  强类型API响应   |

#### 类型签名与意图

* `IActionResult`

    * 接口，表示任何可执行产生 `HTTP` 响应的结果类型。

    * 方法签名：
    
    ```csharp
    public IActionResult Get() { … }
    ```

    * 意图：方法只承诺会返回一个“动作结果”，但没有声明具体的响应体类型。

* `ActionResult<T>`

    * 泛型类，结合了“动作结果”与“强类型返回值”。

    * 方法签名：

    ```csharp
    public ActionResult<Product> Get(int id) { … }
    ```

    * 意图：正常情况下返回 `T`（框架会自动包装为 200 OK 与 JSON），或返回任意派生自 `ActionResult` 的其他结果（如 404、201 等）。

#### 返回值灵活性

| 返回方式           | `IActionResult`           | `ActionResult<T>`                                      |
| ------------------ | ------------------------- | ------------------------------------------------------ |
| 返回特定类型       | 需手动包装：`Ok(product)` | 可以直接 `return product;`（自动封装为 `Ok(product)`） |
| 返回状态码（无体） | `return NoContent();`     | `return NoContent();` （同样有效）                     |
| 返回错误与状态     | `return NotFound();`      | `return NotFound();`                                   |

### 代码示例

#### 使用 IActionResult

```csharp
[HttpGet("{id}")]
public IActionResult Get(int id)
{
    var prod = _svc.Find(id);
    if (prod == null)
        return NotFound();
    return Ok(prod);
}
```

#### 使用 ActionResult<T>

```csharp
[HttpGet("{id}")]
public ActionResult<Product> Get(int id)
{
    var prod = _svc.Find(id);
    if (prod == null)
        return NotFound();        // 隐式转换为 ActionResult<Product>
    return prod;                  // 隐式包装为 Ok(prod)
}
```

### 最佳实践总结

#### 统一选择策略

* 新项目：优先使用 `ActionResult<T>`

* 旧项目迁移：新 `API` 使用 `ActionResult<T>`，旧 `API` 逐步迁移

* 混合响应：当方法可能返回多种不相关类型时使用 `IActionResult`

#### 推荐使用模式

```csharp
// 标准API控制器模式
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // 查询单个资源：ActionResult<T>
    [HttpGet("{id}")]
    public ActionResult<Product> Get(int id) { /* ... */ }
    
    // 创建资源：ActionResult<T>
    [HttpPost]
    public ActionResult<Product> Post([FromBody] Product product) { /* ... */ }
    
    // 文件下载：IActionResult
    [HttpGet("download/{id}")]
    public IActionResult Download(int id) { /* ... */ }
    
    // 重定向：IActionResult
    [HttpGet("legacy/{id}")]
    public IActionResult LegacyRedirect(int id) 
        => RedirectToAction(nameof(Get), new { id });
}
```

### 框架行为

* 模型绑定与文档

    * `ActionResult<T>` 更易让工具（如 `Swagger、NSwag`）推断出返回类型，生成准确的 `API` 文档。

* 异步场景

    * 异步版本对应 `Task<IActionResult>` 与 `Task<ActionResult<T>`>，使用方式完全一致。

### 推荐场景

* 强类型返回推荐 `ActionResult<T>`

    * 当 `API` 主要返回某个实体或 `DTO` 时，`ActionResult<T>` 简化代码、提升可读性，并让文档工具更准确地生成响应模式。

* 多种返回类型场景使用 `IActionResult`

    * 如果方法可能返回多种截然不同的 `DTO`、文件流、视图或跳转等，且没有单一“主”实体类型，使用 `IActionResult` 更灵活。

### 总结

* `IActionResult`：通用接口，灵活但缺少类型信息，需要手动包装响应体。

* `ActionResult<T>`：带泛型的结果类型，直接返回 `T` 更简洁，兼容所有 `ActionResult`，并改善文档与类型安全。

### 资源和文档

* 官方文档：

    * `IActionResult`：https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.iactionresult

    * `ActionResult<T>`：https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.actionresult-1

    * `ASP.NET Core Web API`：https://learn.microsoft.com/en-us/aspnet/core/web-api

* `NuGet` 包：https://www.nuget.org/packages/Microsoft.AspNetCore.Mvc.Core

* `GitHub`：https://github.com/dotnet/aspnetcore