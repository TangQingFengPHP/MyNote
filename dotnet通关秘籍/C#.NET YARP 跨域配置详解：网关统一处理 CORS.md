### 简介

前后端分离项目里，`CORS` 基本绕不开。

本地开发时经常是这样：

```text
前端：http://localhost:5173
网关：http://localhost:5000
商品服务：http://localhost:5101
订单服务：http://localhost:5201
```

浏览器页面从 `http://localhost:5173` 请求 `http://localhost:5000/api/products`，协议、域名、端口只要有一个不同，就属于跨域请求。

`YARP` 做网关时，跨域更推荐放在网关层统一处理：

```text
浏览器
  |
  | Origin: http://localhost:5173
  v
YARP Gateway 处理 CORS
  |
  v
ProductService / OrderService
```

这样做的好处很明显：

* 跨域策略集中在网关入口
* 后端服务不用重复配置 CORS
* 多个服务返回的跨域响应更一致
* 预检请求可以在网关层直接处理，减少后端压力
* 前端只需要访问统一网关地址

`YARP` 官方文档也明确提到：反向代理可以在请求被代理到目标服务之前处理跨域请求，从而减少目标服务负载，并让不同应用使用一致的跨域策略。

### CORS 到底解决什么问题？

`CORS` 全称是 `Cross-Origin Resource Sharing`，中文通常叫跨域资源共享。

它解决的是浏览器安全限制问题。

比如页面地址是：

```text
http://localhost:5173
```

页面里的 JavaScript 请求：

```text
http://localhost:5000/api/products
```

这就是跨域。浏览器会先判断服务端是否允许这个来源访问。

如果服务端响应头里没有正确的跨域信息，浏览器就会拦截响应，控制台常见报错类似：

```text
Access to fetch at 'http://localhost:5000/api/products'
from origin 'http://localhost:5173'
has been blocked by CORS policy
```

注意：跨域是浏览器限制，不是 `curl`、`Postman`、后端代码的限制。

所以经常会出现这种情况：

```text
curl 能请求成功
Postman 能请求成功
浏览器 fetch 请求失败
```

这通常就是 CORS 配置问题。

### 简单请求和预检请求

跨域请求里有一个很关键的概念：预检请求。

浏览器发现某些跨域请求比较“敏感”时，会先发一个 `OPTIONS` 请求问服务端：

```text
这个 Origin 能不能访问？
这个 Method 能不能用？
这些 Header 能不能带？
```

这个 `OPTIONS` 请求就是预检请求，也叫 `Preflight Request`。

比如前端请求里带了 `Authorization`：

```javascript
fetch("http://localhost:5000/api/products", {
  headers: {
    Authorization: "Bearer xxx"
  }
});
```

浏览器通常会先发：

```http
OPTIONS /api/products HTTP/1.1
Origin: http://localhost:5173
Access-Control-Request-Method: GET
Access-Control-Request-Headers: authorization
```

如果网关允许，才会继续发真正的 `GET /api/products`。

所以跨域配置不能只关注 `GET`、`POST`，还要让预检请求能正确通过。

### YARP 里的 CORS 是怎么工作的？

`YARP` 没有重新发明一套 CORS 机制，而是复用 `ASP.NET Core` 的 CORS 中间件。

核心流程是：

```text
AddCors 注册 CORS 策略
UseCors 启用 CORS 中间件
Route.CorsPolicy 指定某条 YARP 路由使用哪个 CORS 策略
MapReverseProxy 执行代理转发
```

配置上主要分两部分：

#### 第一部分：在代码里定义 CORS 策略

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("frontend-dev", policy =>
    {
        policy.WithOrigins("http://localhost:5173")
              .AllowAnyHeader()
              .AllowAnyMethod();
    });
});
```

#### 第二部分：在 YARP 路由里引用策略

```json
"product-route": {
  "ClusterId": "product-cluster",
  "CorsPolicy": "frontend-dev",
  "Match": {
    "Path": "/api/products/{**catch-all}"
  }
}
```

`CorsPolicy` 的值就是 `AddCors` 里定义的策略名。

策略名大小写不敏感，但项目里建议统一写成一种风格，例如：

```text
frontend-dev
frontend-prod
admin-portal
```

### Demo 目标

这一篇继续沿用之前的项目结构：

```text
YarpCorsDemo
├── Gateway
├── ProductService
└── OrderService
```

目标效果：

| 地址 | 说明 |
| --- | --- |
| `http://localhost:5173` | 模拟前端页面 |
| `http://localhost:5000` | YARP 网关 |
| `http://localhost:5101` | 商品服务实例 1 |
| `http://localhost:5102` | 商品服务实例 2 |
| `http://localhost:5201` | 订单服务 |

跨域规则：

* 只允许 `http://localhost:5173` 访问网关 API
* 允许 `GET`、`POST`、`PUT`、`DELETE`
* 允许 `Authorization` 和 `Content-Type` 请求头
* 商品和订单代理路由都使用同一个 CORS 策略
* 后端服务不配置 CORS

### 创建项目

```bash
mkdir YarpCorsDemo
cd YarpCorsDemo

dotnet new sln -n YarpCorsDemo

dotnet new web -n Gateway
dotnet new web -n ProductService
dotnet new web -n OrderService

dotnet sln add Gateway/Gateway.csproj
dotnet sln add ProductService/ProductService.csproj
dotnet sln add OrderService/OrderService.csproj

dotnet add Gateway/Gateway.csproj package Yarp.ReverseProxy
```

### 商品服务 ProductService

修改 `ProductService/Program.cs`：

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/products", (HttpContext context) =>
{
    return Results.Ok(new
    {
        Service = "ProductService",
        Node = context.Connection.LocalPort,
        Origin = context.Request.Headers.Origin.ToString(),
        Data = new[]
        {
            new { Id = 1, Name = "Keyboard", Price = 199 },
            new { Id = 2, Name = "Mouse", Price = 99 }
        }
    });
});

app.MapGet("/products/{id:int}", (int id, HttpContext context) =>
{
    return Results.Ok(new
    {
        Service = "ProductService",
        Node = context.Connection.LocalPort,
        Origin = context.Request.Headers.Origin.ToString(),
        Product = new { Id = id, Name = $"Product-{id}", Price = 100 + id }
    });
});

app.MapGet("/health", () => Results.Ok("Healthy"));

app.Run();
```

这里没有配置任何 CORS。跨域由网关统一处理。

### 订单服务 OrderService

修改 `OrderService/Program.cs`：

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/orders", (HttpContext context) =>
{
    return Results.Ok(new
    {
        Service = "OrderService",
        Node = context.Connection.LocalPort,
        Origin = context.Request.Headers.Origin.ToString(),
        Data = new[]
        {
            new { Id = 1001, ProductId = 1, Count = 2, Amount = 398 },
            new { Id = 1002, ProductId = 2, Count = 1, Amount = 99 }
        }
    });
});

app.MapGet("/orders/{id:int}", (int id, HttpContext context) =>
{
    return Results.Ok(new
    {
        Service = "OrderService",
        Node = context.Connection.LocalPort,
        Origin = context.Request.Headers.Origin.ToString(),
        Order = new { Id = id, Status = "Paid" }
    });
});

app.MapGet("/health", () => Results.Ok("Healthy"));

app.Run();
```

### 网关 Program.cs

修改 `Gateway/Program.cs`：

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddCors(options =>
{
    options.AddPolicy("frontend-dev", policy =>
    {
        policy.WithOrigins("http://localhost:5173")
              .WithMethods("GET", "POST", "PUT", "DELETE")
              .WithHeaders("Authorization", "Content-Type")
              .WithExposedHeaders("X-Trace-Id")
              .SetPreflightMaxAge(TimeSpan.FromMinutes(10));
    });
});

builder.Services
    .AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

var app = builder.Build();

app.UseCors();

app.MapGet("/gateway/ping", () => Results.Ok(new
{
    Service = "Gateway",
    Time = DateTimeOffset.Now
}));

app.MapReverseProxy();

app.Run();
```

这段代码里，`UseCors` 要放在 `MapReverseProxy` 前面。这样预检请求和真实跨域请求才能先经过 CORS 中间件处理，然后再决定是否进入代理转发。

### CORS 配置项说明

上面的策略叫 `frontend-dev`：

```csharp
options.AddPolicy("frontend-dev", policy =>
{
    policy.WithOrigins("http://localhost:5173")
          .WithMethods("GET", "POST", "PUT", "DELETE")
          .WithHeaders("Authorization", "Content-Type")
          .WithExposedHeaders("X-Trace-Id")
          .SetPreflightMaxAge(TimeSpan.FromMinutes(10));
});
```

逐项解释：

| 配置项 | 说明 |
| --- | --- |
| `"frontend-dev"` | CORS 策略名称，YARP 路由通过 `CorsPolicy` 引用它 |
| `WithOrigins` | 允许哪些前端来源访问 |
| `WithMethods` | 允许哪些 HTTP 方法 |
| `WithHeaders` | 允许跨域请求携带哪些请求头 |
| `WithExposedHeaders` | 允许浏览器 JavaScript 读取哪些响应头 |
| `SetPreflightMaxAge` | 预检请求结果缓存多久 |

这里的 `WithHeaders("Authorization", "Content-Type")` 很常见。

原因是：

* `Authorization` 用来传 `Bearer Token`
* `Content-Type` 常用于 `POST` JSON 请求

如果前端还会传自定义请求头，例如：

```text
X-Tenant-Id
X-Request-Id
```

也要加进去：

```csharp
.WithHeaders("Authorization", "Content-Type", "X-Tenant-Id", "X-Request-Id")
```

也可以用：

```csharp
.AllowAnyHeader()
```

但生产环境里更推荐明确列出允许的请求头。

### 网关 appsettings.json

修改 `Gateway/appsettings.json`：

```json
{
  "Urls": "http://localhost:5000",
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "Yarp": "Information"
    }
  },
  "AllowedHosts": "*",
  "ReverseProxy": {
    "Routes": {
      "product-route": {
        "ClusterId": "product-cluster",
        "CorsPolicy": "frontend-dev",
        "Match": {
          "Path": "/api/products/{**catch-all}"
        },
        "Transforms": [
          {
            "PathRemovePrefix": "/api"
          },
          {
            "ResponseHeader": "X-Trace-Id",
            "Set": "demo-trace-id",
            "When": "Always"
          }
        ]
      },
      "order-route": {
        "ClusterId": "order-cluster",
        "CorsPolicy": "frontend-dev",
        "Match": {
          "Path": "/api/orders/{**catch-all}"
        },
        "Transforms": [
          {
            "PathRemovePrefix": "/api"
          },
          {
            "ResponseHeader": "X-Trace-Id",
            "Set": "demo-trace-id",
            "When": "Always"
          }
        ]
      }
    },
    "Clusters": {
      "product-cluster": {
        "LoadBalancingPolicy": "RoundRobin",
        "HealthCheck": {
          "Active": {
            "Enabled": true,
            "Interval": "00:00:10",
            "Timeout": "00:00:02",
            "Policy": "ConsecutiveFailures",
            "Path": "/health"
          }
        },
        "Destinations": {
          "product-1": {
            "Address": "http://localhost:5101/"
          },
          "product-2": {
            "Address": "http://localhost:5102/"
          }
        }
      },
      "order-cluster": {
        "HealthCheck": {
          "Active": {
            "Enabled": true,
            "Interval": "00:00:10",
            "Timeout": "00:00:02",
            "Policy": "ConsecutiveFailures",
            "Path": "/health"
          }
        },
        "Destinations": {
          "order-1": {
            "Address": "http://localhost:5201/"
          }
        }
      }
    }
  }
}
```

重点是：

```json
"CorsPolicy": "frontend-dev"
```

这表示当前代理路由启用名为 `frontend-dev` 的 CORS 策略。

### CorsPolicy 的特殊值

`YARP` 路由里的 `CorsPolicy` 支持普通策略名，也支持特殊值。

| 值 | 说明 |
| --- | --- |
| 自定义策略名 | 使用 `AddCors` 中定义的命名策略 |
| `default` | 使用 ASP.NET Core 默认 CORS 策略 |
| `disable` | 禁用这条路由的 CORS |

例如：

```json
"CorsPolicy": "default"
```

表示使用默认 CORS 策略。

再比如：

```json
"CorsPolicy": "disable"
```

表示这条路由拒绝 CORS 请求。

实际项目里更推荐使用命名策略，因为可读性更好：

```json
"CorsPolicy": "frontend-dev"
```

### 启动服务

启动两个商品服务实例：

```bash
dotnet run --project ProductService/ProductService.csproj --urls http://localhost:5101
```

```bash
dotnet run --project ProductService/ProductService.csproj --urls http://localhost:5102
```

启动订单服务：

```bash
dotnet run --project OrderService/OrderService.csproj --urls http://localhost:5201
```

启动网关：

```bash
dotnet run --project Gateway/Gateway.csproj
```

### 用 curl 测试预检请求

浏览器会自动发预检请求。用 `curl` 也可以模拟：

```bash
curl -i -X OPTIONS http://localhost:5000/api/products \
  -H "Origin: http://localhost:5173" \
  -H "Access-Control-Request-Method: GET" \
  -H "Access-Control-Request-Headers: authorization"
```

如果配置正确，响应头里会看到类似内容：

```text
Access-Control-Allow-Origin: http://localhost:5173
Access-Control-Allow-Methods: GET,POST,PUT,DELETE
Access-Control-Allow-Headers: Authorization,Content-Type
Access-Control-Max-Age: 600
```

这说明预检请求已经被网关允许。

### 用 curl 测试真实请求

```bash
curl -i http://localhost:5000/api/products \
  -H "Origin: http://localhost:5173" \
  -H "Authorization: Bearer test-token"
```

响应头里应该能看到：

```text
Access-Control-Allow-Origin: http://localhost:5173
X-Trace-Id: demo-trace-id
```

因为配置了：

```csharp
.WithExposedHeaders("X-Trace-Id")
```

浏览器端 JavaScript 才能通过 `response.headers.get("X-Trace-Id")` 读取这个响应头。

### 写一个最小前端页面测试

可以创建一个简单目录：

```bash
mkdir Frontend
cd Frontend
```

创建 `index.html`：

```html
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="utf-8">
  <title>YARP CORS Demo</title>
</head>
<body>
  <button id="load">Load Products</button>
  <pre id="result"></pre>

  <script>
    document.querySelector("#load").addEventListener("click", async () => {
      const response = await fetch("http://localhost:5000/api/products", {
        method: "GET",
        headers: {
          "Authorization": "Bearer test-token"
        }
      });

      const traceId = response.headers.get("X-Trace-Id");
      const json = await response.json();

      document.querySelector("#result").textContent = JSON.stringify({
        traceId,
        data: json
      }, null, 2);
    });
  </script>
</body>
</html>
```

启动一个前端静态服务：

```bash
python3 -m http.server 5173
```

浏览器打开：

```text
http://localhost:5173
```

点击按钮后，页面能正常拿到商品数据，说明网关 CORS 配置生效。

### 带 Cookie 的跨域怎么配？

有些项目不用 `Authorization: Bearer xxx`，而是用 Cookie 保存登录态。

这种情况下，前端请求通常会写：

```javascript
fetch("http://localhost:5000/api/products", {
  credentials: "include"
});
```

网关 CORS 策略要允许凭据：

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("frontend-cookie", policy =>
    {
        policy.WithOrigins("http://localhost:5173")
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials();
    });
});
```

注意：`AllowCredentials` 不能和 `AllowAnyOrigin` 一起用。

下面这种配置是错误的：

```csharp
policy.AllowAnyOrigin()
      .AllowCredentials();
```

因为带凭据的跨域请求不能使用 `*` 作为允许来源。ASP.NET Core 官方文档也明确说明，`AllowAnyOrigin` 和 `AllowCredentials` 组合是不安全配置，框架会返回无效的 CORS 响应。

正确写法是明确来源：

```csharp
policy.WithOrigins("http://localhost:5173")
      .AllowCredentials();
```

### 开发环境和生产环境怎么拆？

开发环境通常允许本地前端地址：

```csharp
policy.WithOrigins("http://localhost:5173")
      .AllowAnyHeader()
      .AllowAnyMethod();
```

生产环境应该写真实前端域名：

```csharp
policy.WithOrigins("https://www.example.com", "https://admin.example.com")
      .WithMethods("GET", "POST", "PUT", "DELETE")
      .WithHeaders("Authorization", "Content-Type");
```

更推荐把允许来源放到配置文件：

```json
{
  "Cors": {
    "AllowedOrigins": [
      "http://localhost:5173",
      "https://www.example.com"
    ]
  }
}
```

然后在代码里读取：

```csharp
var allowedOrigins = builder.Configuration
    .GetSection("Cors:AllowedOrigins")
    .Get<string[]>() ?? [];

builder.Services.AddCors(options =>
{
    options.AddPolicy("frontend", policy =>
    {
        policy.WithOrigins(allowedOrigins)
              .AllowAnyHeader()
              .AllowAnyMethod();
    });
});
```

这样不同环境只改配置，不改代码。

### 和 JWT 认证一起用时的顺序

如果网关同时启用了 `CORS` 和 `JWT`，中间件顺序建议这样：

```csharp
app.UseCors();

app.UseAuthentication();
app.UseAuthorization();

app.MapReverseProxy();
```

原因是预检请求通常不带 `Authorization`，如果认证授权先处理，可能会把 `OPTIONS` 预检请求拦掉。

完整结构大概是：

```csharp
var app = builder.Build();

app.UseCors();

app.UseAuthentication();
app.UseAuthorization();

app.MapReverseProxy();

app.Run();
```

### 常见问题

#### 为什么 curl 正常，浏览器不正常？

因为 CORS 是浏览器安全策略。`curl` 和 `Postman` 不受浏览器同源策略限制。

排查跨域问题时，要看浏览器开发者工具里的：

* Console 报错
* Network 里的 `OPTIONS` 请求
* 响应头是否有 `Access-Control-Allow-Origin`
* 响应头是否有 `Access-Control-Allow-Headers`
* 响应头是否有 `Access-Control-Allow-Methods`

#### 为什么预检请求返回 404？

常见原因是 YARP 路由没有启用 `CorsPolicy`，或者没有配置默认 CORS 策略。

`YARP` 文档里提到，预检请求不会自动匹配，除非在路由或应用配置中启用。

解决方式是在代理路由里加：

```json
"CorsPolicy": "frontend-dev"
```

并确保代码里有：

```csharp
app.UseCors();
```

#### 为什么提示 Authorization 不被允许？

常见报错类似：

```text
Request header field authorization is not allowed by Access-Control-Allow-Headers
```

原因是 CORS 策略没有允许 `Authorization` 请求头。

解决方式：

```csharp
.WithHeaders("Authorization", "Content-Type")
```

或者：

```csharp
.AllowAnyHeader()
```

#### 为什么带 Cookie 的请求还是失败？

需要同时满足三点：

前端：

```javascript
fetch(url, {
  credentials: "include"
});
```

网关：

```csharp
policy.WithOrigins("http://localhost:5173")
      .AllowCredentials();
```

Cookie 本身：

```text
SameSite=None; Secure
```

如果是 `http://localhost` 本地开发，Cookie 的 `Secure`、`SameSite`、浏览器策略还可能带来额外影响。

#### 为什么不能直接 AllowAnyOrigin？

开发阶段为了省事，很多人会写：

```csharp
policy.AllowAnyOrigin()
      .AllowAnyHeader()
      .AllowAnyMethod();
```

这在纯公开接口里可以临时用，但不适合需要登录态、后台管理、内部业务 API 的网关。

生产环境更推荐明确来源：

```csharp
policy.WithOrigins("https://www.example.com")
      .AllowAnyHeader()
      .AllowAnyMethod();
```

跨域不是越开放越方便，而是越开放越容易把风险扩大。

### 总结

`YARP` 统一处理 CORS 的关键点可以记成这几步：

```text
AddCors 定义跨域策略
UseCors 启用 CORS 中间件
Route.CorsPolicy 把策略挂到代理路由上
预检请求先在网关层处理
后端服务不需要重复配置 CORS
```

网关层 CORS 的价值不是“少写几行配置”，而是让整个系统的浏览器访问边界变得统一。哪些前端域名能访问、能带哪些请求头、能用哪些方法、能不能带 Cookie，都应该在网关入口清清楚楚地定义。

### 参考资料

* Microsoft Learn：YARP Cross-Origin Requests  
  https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/yarp/cors

* Microsoft Learn：Enable Cross-Origin Requests in ASP.NET Core  
  https://learn.microsoft.com/en-us/aspnet/core/security/cors
