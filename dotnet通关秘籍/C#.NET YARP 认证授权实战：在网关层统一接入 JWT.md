### 简介

上一篇文章已经把 `YARP` 的基础反向代理、路由、负载均衡、健康检查、限流都串了一遍。

这篇继续往下走，专门讲一个网关项目里很常见的能力：

> 在 `YARP Gateway` 层统一接入 `JWT` 认证授权。

也就是说，请求进来以后，先经过网关校验 `JWT`，校验通过后再转发到后端服务。

请求链路大概是这样：

```text
客户端
  |
  | Authorization: Bearer xxx
  v
YARP Gateway
  |
  | 校验 Token
  | 判断权限策略
  v
ProductService / OrderService
```

这样做的好处很直接：

* 认证逻辑集中在网关入口
* 后端服务不用重复写一堆通用鉴权代码
* 不同路由可以配置不同授权策略
* 无效 Token 在进入后端之前就被拦住
* 统一返回 `401` 和 `403`，排查更清楚

不过也要先说清楚边界：

* 网关层适合做统一入口认证、通用权限判断
* 后端服务仍然可以保留关键业务权限校验
* 生产环境 Token 应该由标准身份系统签发，例如 `IdentityServer`、`Keycloak`、`Microsoft Entra ID` 等
* 这个 Demo 里的本地签发 Token 只适合学习和本地测试

### 认证和授权先分清楚

很多项目里会把“认证”和“授权”混在一起说，但它们不是一回事。

| 概念 | 解决的问题 |
| --- | --- |
| 认证 Authentication | 这个请求是谁发来的 |
| 授权 Authorization | 这个身份能不能访问当前资源 |

放到 `JWT` 里就是：

```text
认证：Token 是否合法，签名是否正确，是否过期，Issuer/Audience 是否匹配
授权：Token 里的角色、权限、Claim 是否满足当前接口要求
```

例如：

```text
没有 Token -> 401 Unauthorized
Token 伪造或过期 -> 401 Unauthorized
Token 合法但不是管理员 -> 403 Forbidden
```

`401` 和 `403` 的区别很重要：

* `401`：身份没通过，连“是谁”都没确认
* `403`：身份确认了，但权限不够

### YARP 里的认证授权是怎么工作的？

`YARP` 本身基于 `ASP.NET Core` 管道，所以认证授权并不是重新发明一套机制。

核心流程是：

```text
AddAuthentication 注册认证方式
AddAuthorization 注册授权策略
UseAuthentication 启用认证中间件
UseAuthorization 启用授权中间件
MapReverseProxy 映射代理路由
Route.AuthorizationPolicy 指定某条代理路由使用哪个授权策略
```

最关键的是路由配置里的这个字段：

```json
"AuthorizationPolicy": "authenticated"
```

它表示这条 `YARP` 路由必须满足名为 `authenticated` 的授权策略，满足后才会继续代理到后端。

`YARP` 官方文档里也明确提到：默认情况下，代理请求不会自动认证或授权，除非在路由或应用配置中启用。

### Demo 目标

这篇文章继续沿用上一篇的商城网关结构：

```text
YarpShopDemo
├── Gateway
├── ProductService
└── OrderService
```

目标效果：

| 地址 | 权限 |
| --- | --- |
| `GET /gateway/ping` | 匿名访问 |
| `POST /auth/token` | 匿名访问，用于本地测试签发 Token |
| `GET /api/products` | 登录后访问 |
| `GET /api/products/1` | 登录后访问 |
| `GET /api/orders` | 管理员访问 |
| `GET /api/orders/1001` | 管理员访问 |

也就是说：

* 商品接口只要求登录
* 订单接口要求管理员角色
* 网关健康测试和本地 Token 签发接口允许匿名访问

### 创建项目

```bash
mkdir YarpJwtDemo
cd YarpJwtDemo

dotnet new sln -n YarpJwtDemo

dotnet new web -n Gateway
dotnet new web -n ProductService
dotnet new web -n OrderService

dotnet sln add Gateway/Gateway.csproj
dotnet sln add ProductService/ProductService.csproj
dotnet sln add OrderService/OrderService.csproj

dotnet add Gateway/Gateway.csproj package Yarp.ReverseProxy
dotnet add Gateway/Gateway.csproj package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add Gateway/Gateway.csproj package System.IdentityModel.Tokens.Jwt
```

如果项目使用的是 `.NET 8`，`Microsoft.AspNetCore.Authentication.JwtBearer` 建议安装同大版本的 `8.x`；如果项目使用的是 `.NET 9`，建议安装 `9.x`。认证包版本和目标框架大版本保持一致，踩坑概率更低。

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
        User = context.Request.Headers["X-User-Name"].ToString(),
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
        User = context.Request.Headers["X-User-Name"].ToString(),
        Product = new { Id = id, Name = $"Product-{id}", Price = 100 + id }
    });
});

app.MapGet("/health", () => Results.Ok("Healthy"));

app.Run();
```

这里故意读取了 `X-User-Name` 请求头，用来观察网关是否把用户信息传给了后端。

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
        User = context.Request.Headers["X-User-Name"].ToString(),
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
        User = context.Request.Headers["X-User-Name"].ToString(),
        Order = new { Id = id, Status = "Paid" }
    });
});

app.MapGet("/health", () => Results.Ok("Healthy"));

app.Run();
```

后端服务这里只做演示，不直接启用 JWT 校验。真实项目里，如果后端服务也可能被绕过网关直接访问，就应该在后端继续保留必要的认证授权。

### 网关配置 appsettings.json

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
  "Jwt": {
    "Issuer": "YarpJwtDemo",
    "Audience": "YarpJwtDemo.Api",
    "SigningKey": "YarpJwtDemo_This_Is_A_Local_Test_Key_Only_123456"
  },
  "ReverseProxy": {
    "Routes": {
      "product-route": {
        "ClusterId": "product-cluster",
        "AuthorizationPolicy": "authenticated",
        "Match": {
          "Path": "/api/products/{**catch-all}"
        },
        "Transforms": [
          {
            "PathRemovePrefix": "/api"
          },
          {
            "RequestHeader": "X-Gateway",
            "Set": "YarpJwtDemo"
          }
        ]
      },
      "order-route": {
        "ClusterId": "order-cluster",
        "AuthorizationPolicy": "admin-only",
        "Match": {
          "Path": "/api/orders/{**catch-all}"
        },
        "Transforms": [
          {
            "PathRemovePrefix": "/api"
          },
          {
            "RequestHeader": "X-Gateway",
            "Set": "YarpJwtDemo"
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

这里最重要的是两条路由：

```json
"AuthorizationPolicy": "authenticated"
```

和：

```json
"AuthorizationPolicy": "admin-only"
```

商品接口使用 `authenticated` 策略，只要登录就能访问。

订单接口使用 `admin-only` 策略，必须带有管理员角色。

### 网关 Program.cs

修改 `Gateway/Program.cs`：

```csharp
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using Yarp.ReverseProxy.Transforms;

var builder = WebApplication.CreateBuilder(args);

var jwtSection = builder.Configuration.GetSection("Jwt");
var issuer = jwtSection["Issuer"]!;
var audience = jwtSection["Audience"]!;
var signingKey = jwtSection["SigningKey"]!;
var securityKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(signingKey));

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = issuer,
            ValidateAudience = true,
            ValidAudience = audience,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = securityKey,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromSeconds(30)
        };
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("authenticated", policy =>
    {
        policy.RequireAuthenticatedUser();
    });

    options.AddPolicy("admin-only", policy =>
    {
        policy.RequireAuthenticatedUser();
        policy.RequireRole("Admin");
    });
});

builder.Services
    .AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"))
    .AddTransforms(transformBuilderContext =>
    {
        transformBuilderContext.AddRequestTransform(transformContext =>
        {
            var user = transformContext.HttpContext.User;

            if (user.Identity?.IsAuthenticated == true)
            {
                var userName = user.FindFirstValue(ClaimTypes.Name) ?? "";
                transformContext.ProxyRequest.Headers.Remove("X-User-Name");
                transformContext.ProxyRequest.Headers.TryAddWithoutValidation("X-User-Name", userName);
            }

            return ValueTask.CompletedTask;
        });
    });

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapPost("/auth/token", (LoginRequest request) =>
{
    if (request.UserName == "admin" && request.Password == "123456")
    {
        var token = CreateToken(request.UserName, "Admin", issuer, audience, securityKey);
        return Results.Ok(new { AccessToken = token });
    }

    if (request.UserName == "user" && request.Password == "123456")
    {
        var token = CreateToken(request.UserName, "User", issuer, audience, securityKey);
        return Results.Ok(new { AccessToken = token });
    }

    return Results.Unauthorized();
}).AllowAnonymous();

app.MapGet("/gateway/ping", () => Results.Ok(new
{
    Service = "Gateway",
    Time = DateTimeOffset.Now
})).AllowAnonymous();

app.MapReverseProxy();

app.Run();

static string CreateToken(
    string userName,
    string role,
    string issuer,
    string audience,
    SecurityKey securityKey)
{
    var claims = new List<Claim>
    {
        new(ClaimTypes.Name, userName),
        new(ClaimTypes.Role, role),
        new(JwtRegisteredClaimNames.Sub, userName),
        new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString("N"))
    };

    var credentials = new SigningCredentials(securityKey, SecurityAlgorithms.HmacSha256);

    var token = new JwtSecurityToken(
        issuer: issuer,
        audience: audience,
        claims: claims,
        expires: DateTime.UtcNow.AddMinutes(30),
        signingCredentials: credentials);

    return new JwtSecurityTokenHandler().WriteToken(token);
}

public sealed record LoginRequest(string UserName, string Password);
```

这段代码做了几件事：

* `AddAuthentication` 注册 `JWT Bearer` 认证
* `AddJwtBearer` 配置 Token 校验规则
* `AddAuthorization` 定义两个授权策略
* `UseAuthentication` 启用认证中间件
* `UseAuthorization` 启用授权中间件
* `/auth/token` 用于本地测试签发 Token
* `MapReverseProxy` 负责把通过授权的请求转发到后端
* `AddRequestTransform` 把当前用户名写入 `X-User-Name` 请求头

中间件顺序要注意：

```csharp
app.UseAuthentication();
app.UseAuthorization();
app.MapReverseProxy();
```

认证授权中间件要放在 `MapReverseProxy` 前面，这样代理转发前才能完成身份校验。

### JWT 校验配置说明

核心配置在这里：

```csharp
options.TokenValidationParameters = new TokenValidationParameters
{
    ValidateIssuer = true,
    ValidIssuer = issuer,
    ValidateAudience = true,
    ValidAudience = audience,
    ValidateIssuerSigningKey = true,
    IssuerSigningKey = securityKey,
    ValidateLifetime = true,
    ClockSkew = TimeSpan.FromSeconds(30)
};
```

逐项解释：

| 配置项 | 说明 |
| --- | --- |
| `ValidateIssuer` | 是否校验签发方 |
| `ValidIssuer` | 合法签发方，例如 `YarpJwtDemo` |
| `ValidateAudience` | 是否校验接收方 |
| `ValidAudience` | 合法接收方，例如 `YarpJwtDemo.Api` |
| `ValidateIssuerSigningKey` | 是否校验签名密钥 |
| `IssuerSigningKey` | 用来验证签名的密钥 |
| `ValidateLifetime` | 是否校验过期时间 |
| `ClockSkew` | 时间偏移容忍范围 |

`ClockSkew` 默认值通常比较宽，本地 Demo 设成 30 秒更容易观察过期效果。

生产环境里，`SigningKey` 不应该写死在 `appsettings.json`。更合理的方式是：

* 环境变量
* 密钥管理服务
* Kubernetes Secret
* Azure Key Vault
* 从身份提供方的元数据地址读取公钥

### AuthorizationPolicy 配置说明

商品路由：

```json
"product-route": {
  "ClusterId": "product-cluster",
  "AuthorizationPolicy": "authenticated",
  "Match": {
    "Path": "/api/products/{**catch-all}"
  }
}
```

对应策略：

```csharp
options.AddPolicy("authenticated", policy =>
{
    policy.RequireAuthenticatedUser();
});
```

含义是：只要 Token 合法，且能识别出登录用户，就允许访问商品接口。

订单路由：

```json
"order-route": {
  "ClusterId": "order-cluster",
  "AuthorizationPolicy": "admin-only",
  "Match": {
    "Path": "/api/orders/{**catch-all}"
  }
}
```

对应策略：

```csharp
options.AddPolicy("admin-only", policy =>
{
    policy.RequireAuthenticatedUser();
    policy.RequireRole("Admin");
});
```

含义是：必须登录，并且角色必须是 `Admin`。

这里的策略名大小写不敏感，但项目里建议统一使用小写或统一使用短横线风格，比如：

```text
authenticated
admin-only
order-read
product-write
```

### default 和 anonymous

`YARP` 的 `AuthorizationPolicy` 除了能写自定义策略名，还支持两个特殊值：

| 值 | 说明 |
| --- | --- |
| `default` | 使用 ASP.NET Core 默认授权策略 |
| `anonymous` | 明确允许匿名访问 |

例如：

```json
"AuthorizationPolicy": "default"
```

表示这条路由使用默认授权策略。默认策略通常要求已登录用户。

再比如：

```json
"AuthorizationPolicy": "anonymous"
```

表示这条代理路由允许匿名访问，即使应用设置了全局兜底授权策略，也不会拦它。

不过在实际项目里，公开接口最好显式写清楚，不要靠猜：

```json
"public-route": {
  "ClusterId": "public-cluster",
  "AuthorizationPolicy": "anonymous",
  "Match": {
    "Path": "/api/public/{**catch-all}"
  }
}
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

### 测试匿名接口

```bash
curl http://localhost:5000/gateway/ping
```

这个接口没有走代理，也不需要 Token。

### 不带 Token 访问商品接口

```bash
curl -i http://localhost:5000/api/products
```

预期结果：

```text
HTTP/1.1 401 Unauthorized
```

因为 `product-route` 要求 `authenticated`，没有 Token 就无法通过认证。

### 获取普通用户 Token

```bash
curl -X POST http://localhost:5000/auth/token \
  -H "Content-Type: application/json" \
  -d '{"userName":"user","password":"123456"}'
```

返回结果类似：

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

把 Token 放到变量里：

```bash
USER_TOKEN="这里替换成 user 的 accessToken"
```

访问商品接口：

```bash
curl http://localhost:5000/api/products \
  -H "Authorization: Bearer $USER_TOKEN"
```

这次会返回商品数据。

多请求几次，可以看到 `Node` 在 `5101` 和 `5102` 之间变化，说明请求已经通过网关鉴权，并且继续进入商品服务集群负载均衡。

### 普通用户访问订单接口

```bash
curl -i http://localhost:5000/api/orders \
  -H "Authorization: Bearer $USER_TOKEN"
```

预期结果：

```text
HTTP/1.1 403 Forbidden
```

原因是普通用户已经通过认证，但不满足 `admin-only` 策略里的 `RequireRole("Admin")`。

这就是 `401` 和 `403` 的区别：

```text
没有合法身份 -> 401
有合法身份但权限不够 -> 403
```

### 获取管理员 Token

```bash
curl -X POST http://localhost:5000/auth/token \
  -H "Content-Type: application/json" \
  -d '{"userName":"admin","password":"123456"}'
```

把 Token 放到变量里：

```bash
ADMIN_TOKEN="这里替换成 admin 的 accessToken"
```

访问订单接口：

```bash
curl http://localhost:5000/api/orders \
  -H "Authorization: Bearer $ADMIN_TOKEN"
```

这次可以正常返回订单数据。

### 把用户信息传给后端

在前面的网关代码里，有这样一段 Transform：

```csharp
.AddTransforms(transformBuilderContext =>
{
    transformBuilderContext.AddRequestTransform(transformContext =>
    {
        var user = transformContext.HttpContext.User;

        if (user.Identity?.IsAuthenticated == true)
        {
            var userName = user.FindFirstValue(ClaimTypes.Name) ?? "";
            transformContext.ProxyRequest.Headers.Remove("X-User-Name");
            transformContext.ProxyRequest.Headers.TryAddWithoutValidation("X-User-Name", userName);
        }

        return ValueTask.CompletedTask;
    });
});
```

它的作用是：网关从当前 `ClaimsPrincipal` 里取出用户名，然后转成 `X-User-Name` 请求头传给后端。

后端收到的请求大概是：

```text
X-User-Name: admin
X-Gateway: YarpJwtDemo
Authorization: Bearer xxx
```

需要注意一点：默认情况下，客户端传进来的 `Authorization` 请求头也会继续转发到后端。也就是说，后端如果启用了 JWT 校验，也可以继续验证同一个 Token。

如果后端完全信任网关传来的 `X-User-Name`，就必须保证后端服务不能被外部绕过网关直接访问，否则客户端可以伪造这个请求头。

更稳的做法是：

* 后端服务只允许网关网络访问
* 网关转发前移除客户端伪造的身份头
* 后端关键接口继续校验 Token 或校验内部签名
* 服务间通信使用 mTLS 或内网身份机制

### 是否要在后端再次校验 JWT？

这要看部署边界。

#### 只允许通过网关访问

如果后端服务只暴露在内网，外部完全无法绕过网关访问，网关层 JWT 校验通常可以承担大部分通用认证工作。

后端仍然建议保留业务级权限判断，例如：

* 订单只能由订单所属用户查看
* 管理员只能操作授权范围内的数据
* 租户数据不能跨租户访问

#### 后端可能被直接访问

如果后端服务地址可能被客户端、内部系统、测试工具直接访问，就不能只依赖网关。

这种场景下，后端也应该启用 JWT 校验，至少保护核心接口。

更简单地说：

```text
网关鉴权解决统一入口问题。
后端鉴权解决服务自身安全边界问题。
```

### 生产环境不要这样签发 Token

Demo 里为了方便测试，在网关里写了 `/auth/token`：

```csharp
app.MapPost("/auth/token", ...)
```

这只是本地演示。

生产环境不要直接用这种“用户名密码换 Token”的简化写法。更推荐的方式是接入标准身份系统：

* OpenID Connect
* OAuth 2.0
* Authorization Code + PKCE
* Client Credentials
* Microsoft Entra ID
* Keycloak
* IdentityServer

生产环境里，网关通常只负责验证访问令牌，不负责用明文密码签发令牌。

### 常见问题

#### 为什么配置了 AddJwtBearer 还是能匿名访问？

只注册认证方式，不等于所有接口自动要求登录。

要让代理路由必须登录，需要：

```json
"AuthorizationPolicy": "authenticated"
```

或者配置全局 `FallbackPolicy`。

#### 为什么返回 401？

常见原因：

* 没传 `Authorization` 请求头
* 请求头格式不是 `Bearer token`
* Token 过期
* 签名不匹配
* `Issuer` 不匹配
* `Audience` 不匹配
* 签名密钥和签发 Token 时不一致

#### 为什么返回 403？

说明 Token 已经通过认证，但权限不够。

常见原因：

* 缺少角色
* 角色名称不匹配
* Claim 类型不匹配
* 授权策略写得比预期更严格

#### 为什么 RequireRole 不生效？

常见原因是角色 Claim 类型不匹配。

当前 Demo 使用的是：

```csharp
new(ClaimTypes.Role, role)
```

`RequireRole("Admin")` 默认能识别这种角色 Claim。

如果 Token 里的角色字段是 `role`、`roles` 或其他自定义名称，可以显式配置：

```csharp
options.TokenValidationParameters = new TokenValidationParameters
{
    RoleClaimType = "role"
};
```

注意不要把其他校验项覆盖掉，实际代码里应该和 `ValidateIssuer`、`ValidateAudience`、`IssuerSigningKey` 等配置放在同一个 `TokenValidationParameters` 里。

#### 要不要把 Authorization 请求头转发给后端？

默认会转发。

如果后端也要校验 JWT，就保留它。

如果后端只信任网关，不想收到外部 Token，可以在 Transform 里移除：

```json
"Transforms": [
  {
    "RequestHeaderRemove": "Authorization"
  }
]
```

是否移除取决于后端安全模型，不是固定答案。

### 总结

`YARP` 接入 `JWT` 的核心不是复杂 API，而是把几个边界摆正：

```text
AddAuthentication 负责识别身份
AddAuthorization 负责定义权限策略
UseAuthentication / UseAuthorization 负责启用中间件
AuthorizationPolicy 负责把策略挂到代理路由上
Transform 可以把身份信息传给后端
```

网关层统一接入 `JWT` 后，后端服务可以少处理很多通用入口逻辑。但安全边界不能只靠“感觉上所有流量都会经过网关”。只要后端可能被绕过访问，后端就仍然需要自己的保护。

### 参考资料

* Microsoft Learn：YARP Authentication and Authorization  
  https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/yarp/authn-authz

* Microsoft Learn：Configure JWT bearer authentication in ASP.NET Core  
  https://learn.microsoft.com/en-us/aspnet/core/security/authentication/configure-jwt-bearer-authentication

* Microsoft Learn：Policy-based authorization in ASP.NET Core  
  https://learn.microsoft.com/en-us/aspnet/core/security/authorization/policies
