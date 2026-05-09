### 简介

`YARP` 全称是 `Yet Another Reverse Proxy`，是微软开源的 `.NET` 反向代理库。

一句话说清楚：

> `YARP` 不是一个单独安装的网关软件，而是一个装进 `ASP.NET Core` 项目里的反向代理组件。

所以它和 `Nginx`、`Envoy` 这类独立代理不太一样。`YARP` 更像一个“网关积木”，路由、负载均衡、请求头处理、认证授权、限流、健康检查这些能力都能放进 `.NET` 项目里统一控制。

典型请求链路长这样：

```text
浏览器 / App / 第三方系统
        |
        v
YARP Gateway
        |
        +--> ProductService
        |
        +--> OrderService
        |
        +--> UserService
```

对 `.NET` 微服务项目来说，`YARP` 最大的吸引力在于：

* 直接使用 `ASP.NET Core` 管道
* 直接使用 C# 扩展
* 配置能热更新
* 能和认证、授权、限流、日志、OpenTelemetry 等能力放在一起
* 不需要为了简单网关逻辑再维护一套完全不同的技术栈

### YARP 适合解决什么问题？

项目拆成多个服务后，常见问题马上就会出现：

* 前端不想记一堆后端服务地址
* 对外只想暴露一个统一入口
* `/api/products` 要转到商品服务
* `/api/orders` 要转到订单服务
* 后端有多个实例，需要负载均衡
* 认证、限流、跨域、日志最好在入口统一处理
* 后端真实地址不希望直接暴露出去

`YARP` 处理的就是这些入口层问题。

比如：

```text
http://localhost:5000/api/products
        |
        v
YARP
        |
        v
http://localhost:5101/products
```

客户端只访问网关，后端服务藏在网关后面。

### 核心概念

`YARP` 最核心的概念就三个：

| 概念 | 说明 |
| --- | --- |
| `Route` | 路由规则，决定什么请求能匹配进来 |
| `Cluster` | 后端服务集群，决定请求转发到哪一组服务 |
| `Destination` | 后端服务的具体实例地址 |

可以这样理解：

```text
Route 负责判断：这个请求归谁管？
Cluster 负责判断：这类请求有哪些后端能处理？
Destination 负责表示：真正的后端地址是什么？
```

一个简单例子：

```json
{
  "ReverseProxy": {
    "Routes": {
      "product-route": {
        "ClusterId": "product-cluster",
        "Match": {
          "Path": "/api/{**catch-all}"
        }
      }
    },
    "Clusters": {
      "product-cluster": {
        "Destinations": {
          "product-1": {
            "Address": "http://localhost:5101/"
          }
        }
      }
    }
  }
}
```

这段配置表达的意思是：

* 匹配 `/api/products/...` 这类请求
* 命中后转给 `product-cluster`
* `product-cluster` 当前只有一个实例：`http://localhost:5101/`

### 安装 YARP

创建一个空的 `ASP.NET Core` 项目：

```bash
dotnet new web -n YarpGateway
cd YarpGateway
```

安装 NuGet 包：

```bash
dotnet add package Yarp.ReverseProxy
```

目前 NuGet 上的 `Yarp.ReverseProxy` 稳定版已经到 `2.3.0`，包目标框架包含 `net6.0`，在现代 `.NET` 项目中使用通常没有门槛。实际项目建议优先使用当前稳定版，不建议继续使用早期存在安全风险的旧版本。

### 最小可运行 Demo

下面做一个真正能跑的例子：

* 一个 `YarpGateway` 网关，监听 `5000`
* 一个 `ProductService` 后端服务，分别启动两个实例：`5101`、`5102`
* 请求 `/api/products` 进入网关
* 网关去掉 `/api` 前缀，再转发到后端 `/products`
* 两个后端实例通过轮询方式接收请求

#### 创建后端服务

```bash
dotnet new web -n ProductService
cd ProductService
```

修改 `Program.cs`：

```csharp
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

app.MapGet("/products", (HttpContext context) =>
{
    return Results.Ok(new
    {
        Service = "ProductService",
        Node = context.Connection.LocalPort,
        Items = new[]
        {
            new { Id = 1, Name = "Keyboard", Price = 199 },
            new { Id = 2, Name = "Mouse", Price = 99 }
        }
    });
});

app.MapGet("/health", () => Results.Ok("Healthy"));

app.Run();
```

启动两个后端实例：

```bash
dotnet run --urls http://localhost:5101
```

另开一个终端：

```bash
dotnet run --urls http://localhost:5102
```

直接访问后端：

```bash
curl http://localhost:5101/products
curl http://localhost:5102/products
```

能看到返回里的 `Node` 分别是 `5101` 和 `5102`。

#### 创建 YARP 网关

```bash
dotnet new web -n YarpGateway
cd YarpGateway
dotnet add package Yarp.ReverseProxy
```

修改 `Program.cs`：

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

var app = builder.Build();

app.MapReverseProxy();

app.Run();
```

修改 `appsettings.json`：

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
        "Match": {
          "Path": "/api/{**catch-all}"
        },
        "Transforms": [
          {
            "PathRemovePrefix": "/api"
          }
        ]
      }
    },
    "Clusters": {
      "product-cluster": {
        "LoadBalancingPolicy": "RoundRobin",
        "Destinations": {
          "product-1": {
            "Address": "http://localhost:5101/"
          },
          "product-2": {
            "Address": "http://localhost:5102/"
          }
        },
        "HealthCheck": {
          "Active": {
            "Enabled": true,
            "Interval": "00:00:10",
            "Timeout": "00:00:02",
            "Policy": "ConsecutiveFailures",
            "Path": "/health"
          }
        }
      }
    }
  }
}
```

启动网关：

```bash
dotnet run
```

访问网关：

```bash
curl http://localhost:5000/api/products
```

多请求几次，会看到返回里的 `Node` 在 `5101` 和 `5102` 之间变化，这说明负载均衡已经生效。

这条请求实际发生了几件事：

```text
客户端请求：http://localhost:5000/api/products
命中路由：product-route
移除前缀：/api
转发路径：/products
后端集群：product-cluster
后端实例：http://localhost:5101 或 http://localhost:5102
```

### Route 路由详解

`Route` 负责匹配请求。

常见匹配方式有：

* 按路径匹配
* 按域名匹配
* 按请求头匹配
* 按请求方法匹配
* 按查询参数匹配

最常见的是路径匹配：

```json
"Match": {
  "Path": "/api/{**catch-all}"
}
```

`{**catch-all}` 表示后面任意路径都能匹配。

例如：

```text
/api/products
/api/products/1
/api/products/category/keyboard
```

都能命中这条路由。

如果有多条路由，`YARP` 会按更具体的规则优先匹配。需要手动控制优先级时，可以加 `Order`：

```json
"product-detail-route": {
  "Order": 0,
  "ClusterId": "product-cluster",
  "Match": {
    "Path": "/api/products/{id}"
  }
}
```

`Order` 数值越小，优先级越高。

### Cluster 集群详解

`Cluster` 负责描述一组后端服务。

一个集群可以只有一个后端：

```json
"Clusters": {
  "product-cluster": {
    "Destinations": {
      "product-1": {
        "Address": "http://localhost:5101/"
      }
    }
  }
}
```

也可以有多个后端：

```json
"Clusters": {
  "product-cluster": {
    "LoadBalancingPolicy": "RoundRobin",
    "Destinations": {
      "product-1": {
        "Address": "http://localhost:5101/"
      },
      "product-2": {
        "Address": "http://localhost:5102/"
      }
    }
  }
}
```

后端地址建议以 `/` 结尾，比如：

```text
http://localhost:5101/
```

这样做能减少路径拼接时的歧义。

### 负载均衡策略

当一个 `Cluster` 下有多个健康的 `Destination` 时，`YARP` 要决定请求交给哪一个实例。

常用策略如下：

| 策略 | 说明 |
| --- | --- |
| `PowerOfTwoChoices` | 默认策略，随机选两个可用实例，再挑未完成请求较少的那个 |
| `RoundRobin` | 轮询，按顺序分配 |
| `Random` | 随机选择 |
| `LeastRequests` | 选择当前未完成请求最少的实例 |
| `FirstAlphabetical` | 选择名称排序最靠前的实例，常用于主备切换 |

配置方式：

```json
"product-cluster": {
  "LoadBalancingPolicy": "LeastRequests",
  "Destinations": {
    "product-1": {
      "Address": "http://localhost:5101/"
    },
    "product-2": {
      "Address": "http://localhost:5102/"
    }
  }
}
```

多数场景下，不写 `LoadBalancingPolicy` 也可以，默认的 `PowerOfTwoChoices` 已经适合很多高并发场景。

### Transform 请求转换

`Transform` 是 `YARP` 很实用的能力。

它能在转发前修改代理请求，常见用途包括：

* 删除路径前缀
* 增加路径前缀
* 修改请求头
* 增加响应头
* 传递原始客户端 IP

前面的 Demo 用到了：

```json
"Transforms": [
  {
    "PathRemovePrefix": "/api"
  }
]
```

它的作用是：

```text
/api/products -> /products
```

再看一个添加请求头的例子：

```json
"Transforms": [
  {
    "RequestHeader": "X-Gateway",
    "Set": "YARP"
  }
]
```

转发到后端时，后端会收到：

```text
X-Gateway: YARP
```

需要注意一点：`YARP` 的内置 Transform 主要处理路径、查询字符串、请求头、响应头等内容。请求体和响应体的转换不是内置常规能力，复杂 body 改写通常要用中间件或重新设计接口边界。

### 认证授权

`YARP` 基于 `ASP.NET Core`，所以认证授权可以直接放在网关层。

例如启用 JWT：

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;

var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://identity.example.com";
        options.Audience = "gateway-api";
    });

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("product-policy", policy =>
    {
        policy.RequireAuthenticatedUser();
    });
});

builder.Services
    .AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapReverseProxy();

app.Run();
```

路由上指定授权策略：

```json
"product-route": {
  "ClusterId": "product-cluster",
  "AuthorizationPolicy": "product-policy",
  "Match": {
    "Path": "/api/products/{**catch-all}"
  }
}
```

这样请求在转发到后端之前，就会先经过网关授权。

这类做法适合统一入口鉴权。后端服务仍然可以保留必要的服务内权限校验，避免把全部安全边界都压在网关一层。

### 限流

`.NET` 自带限流中间件，和 `YARP` 放在一起很自然。

示例：

```csharp
using Microsoft.AspNetCore.RateLimiting;
using System.Threading.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", limiter =>
    {
        limiter.PermitLimit = 100;
        limiter.Window = TimeSpan.FromMinutes(1);
        limiter.QueueLimit = 0;
        limiter.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
    });
});

builder.Services
    .AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

var app = builder.Build();

app.UseRateLimiter();

app.MapReverseProxy();

app.Run();
```

这段代码用的是固定窗口限流，配置项可以这样理解：

| 配置项 | 说明 |
| --- | --- |
| `"fixed"` | 限流策略名称，后面在路由配置里通过 `RateLimiterPolicy` 引用它 |
| `PermitLimit = 100` | 一个时间窗口内最多允许 100 个请求通过 |
| `Window = TimeSpan.FromMinutes(1)` | 时间窗口长度是 1 分钟 |
| `QueueLimit = 0` | 超过限流阈值后不排队，直接拒绝请求 |
| `QueueProcessingOrder = QueueProcessingOrder.OldestFirst` | 如果启用了排队，优先处理最早进入队列的请求 |

所以这段配置的实际效果是：

```text
每 1 分钟最多放行 100 个请求。
第 101 个请求开始，如果还在同一个时间窗口内，就不会进入等待队列，而是直接被限流。
```

这里的 `QueueLimit = 0` 很关键。它表示不让请求在网关里堆积等待，超过限制就快速失败。对网关来说，这通常比让大量请求排队更稳，因为排队会占用连接和内存，后端慢的时候还可能把网关也拖慢。

如果希望短时间突发请求可以稍微等一下，可以把队列打开：

```csharp
options.AddFixedWindowLimiter("fixed", limiter =>
{
    limiter.PermitLimit = 100;
    limiter.Window = TimeSpan.FromMinutes(1);
    limiter.QueueLimit = 20;
    limiter.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
});
```

这表示 1 分钟内先放行 100 个请求，超过后最多再排队 20 个请求。排队也不是越大越好，队列越大，请求等待越久，网关资源占用也越高。

路由配置：

```json
"product-route": {
  "ClusterId": "product-cluster",
  "RateLimiterPolicy": "fixed",
  "Match": {
    "Path": "/api/products/{**catch-all}"
  }
}
```

这表示命中 `product-route` 的请求会使用 `fixed` 限流策略。

### 健康检查

网关转发时，不能只知道“有哪些后端地址”，还要知道“哪些后端地址还能用”。

`YARP` 支持主动健康检查：

```json
"HealthCheck": {
  "Active": {
    "Enabled": true,
    "Interval": "00:00:10",
    "Timeout": "00:00:02",
    "Policy": "ConsecutiveFailures",
    "Path": "/health"
  }
}
```

这表示每 10 秒访问一次后端实例的 `/health`，连续失败后会把实例标记为不健康。

后端服务提供健康检查地址：

```csharp
app.MapGet("/health", () => Results.Ok("Healthy"));
```

生产环境里，健康检查不要只返回一个固定字符串。更合理的做法是检查关键依赖，例如数据库、缓存、消息队列等是否可用。

### 配置热更新

`YARP` 从 `IConfiguration` 加载配置时，配置源变更后可以触发重新加载。

也就是说，`appsettings.json` 中的路由和集群发生变化后，代理配置可以更新，不需要像传统静态配置那样每次手动重启进程。

不过生产环境不建议直接手改服务器上的 JSON 文件。更稳的方式是：

* 配置中心
* 数据库
* Consul
* Kubernetes 配置
* 自定义 `IProxyConfigProvider`

`IProxyConfigProvider` 适合需要从数据库或服务发现系统动态生成路由的场景。

内存配置示例：

```csharp
using Yarp.ReverseProxy.Configuration;

var routes = new[]
{
    new RouteConfig
    {
        RouteId = "product-route",
        ClusterId = "product-cluster",
        Match = new RouteMatch
        {
            Path = "/api/{**catch-all}"
        },
        Transforms = new[]
        {
            new Dictionary<string, string>
            {
                ["PathRemovePrefix"] = "/api"
            }
        }
    }
};

var clusters = new[]
{
    new ClusterConfig
    {
        ClusterId = "product-cluster",
        LoadBalancingPolicy = "RoundRobin",
        Destinations = new Dictionary<string, DestinationConfig>
        {
            ["product-1"] = new DestinationConfig
            {
                Address = "http://localhost:5101/"
            },
            ["product-2"] = new DestinationConfig
            {
                Address = "http://localhost:5102/"
            }
        }
    }
};

builder.Services
    .AddReverseProxy()
    .LoadFromMemory(routes, clusters);
```

这种方式适合配置来源不是 JSON 文件的项目。

### 完整实战 Demo：商品服务和订单服务统一入口

前面的 Demo 适合理解 `YARP` 最小用法。实际项目里，网关通常不会只代理一个服务，而是把多个后端服务统一到一个入口下面。

这一节做一个更接近真实项目的 Demo：

```text
YarpShopDemo
├── Gateway
├── ProductService
└── OrderService
```

最终效果：

| 网关地址 | 实际转发到 |
| --- | --- |
| `http://localhost:5000/api/products` | `ProductService` |
| `http://localhost:5000/api/orders` | `OrderService` |
| `http://localhost:5000/gateway/ping` | 网关自身接口 |

这个 Demo 会用到：

* 多服务路由
* 商品服务双实例负载均衡
* 订单服务单实例转发
* `/api` 前缀移除
* 主动健康检查
* 固定窗口限流
* 网关请求头标记

#### 1. 创建解决方案和项目

```bash
mkdir YarpShopDemo
cd YarpShopDemo

dotnet new sln -n YarpShopDemo

dotnet new web -n Gateway
dotnet new web -n ProductService
dotnet new web -n OrderService

dotnet sln add Gateway/Gateway.csproj
dotnet sln add ProductService/ProductService.csproj
dotnet sln add OrderService/OrderService.csproj

dotnet add Gateway/Gateway.csproj package Yarp.ReverseProxy
```

#### 2. 商品服务 ProductService

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
        Data = new[]
        {
            new { Id = 1, Name = "Keyboard", Price = 199 },
            new { Id = 2, Name = "Mouse", Price = 99 },
            new { Id = 3, Name = "Monitor", Price = 899 }
        }
    });
});

app.MapGet("/products/{id:int}", (int id, HttpContext context) =>
{
    return Results.Ok(new
    {
        Service = "ProductService",
        Node = context.Connection.LocalPort,
        Product = new { Id = id, Name = $"Product-{id}", Price = 100 + id }
    });
});

app.MapGet("/health", () => Results.Ok("Healthy"));

app.Run();
```

#### 3. 订单服务 OrderService

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
        Order = new { Id = id, Status = "Paid" }
    });
});

app.MapGet("/health", () => Results.Ok("Healthy"));

app.Run();
```

#### 4. 网关 Gateway

修改 `Gateway/Program.cs`：

```csharp
using Microsoft.AspNetCore.RateLimiting;
using System.Threading.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", limiter =>
    {
        limiter.PermitLimit = 100;
        limiter.Window = TimeSpan.FromMinutes(1);
        limiter.QueueLimit = 0;
        limiter.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
    });
});

builder.Services
    .AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"));

var app = builder.Build();

app.UseRateLimiter();

app.MapGet("/gateway/ping", () => Results.Ok(new
{
    Service = "Gateway",
    Time = DateTimeOffset.Now
}));

app.MapReverseProxy();

app.Run();
```

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
        "RateLimiterPolicy": "fixed",
        "Match": {
          "Path": "/api/products/{**catch-all}"
        },
        "Transforms": [
          {
            "PathRemovePrefix": "/api"
          },
          {
            "RequestHeader": "X-Gateway",
            "Set": "YarpShopDemo"
          }
        ]
      },
      "order-route": {
        "ClusterId": "order-cluster",
        "RateLimiterPolicy": "fixed",
        "Match": {
          "Path": "/api/orders/{**catch-all}"
        },
        "Transforms": [
          {
            "PathRemovePrefix": "/api"
          },
          {
            "RequestHeader": "X-Gateway",
            "Set": "YarpShopDemo"
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

#### 5. 启动服务

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

#### 6. 测试网关转发

测试网关自身接口：

```bash
curl http://localhost:5000/gateway/ping
```

测试商品列表：

```bash
curl http://localhost:5000/api/products
```

多执行几次，返回里的 `Node` 会在 `5101` 和 `5102` 之间切换，说明商品服务的轮询负载均衡已经生效。

测试商品详情：

```bash
curl http://localhost:5000/api/products/1
```

测试订单列表：

```bash
curl http://localhost:5000/api/orders
```

测试订单详情：

```bash
curl http://localhost:5000/api/orders/1001
```

#### 7. 这个 Demo 的请求链路

访问商品接口：

```text
GET http://localhost:5000/api/products/1
    |
    v
Gateway 命中 product-route
    |
    v
PathRemovePrefix 移除 /api
    |
    v
转发为 /products/1
    |
    v
RoundRobin 选择 product-1 或 product-2
```

访问订单接口：

```text
GET http://localhost:5000/api/orders/1001
    |
    v
Gateway 命中 order-route
    |
    v
PathRemovePrefix 移除 /api
    |
    v
转发为 /orders/1001
    |
    v
发送到 order-1
```

#### 8. 故障测试

停止 `http://localhost:5101` 这个商品服务实例，再继续请求：

```bash
curl http://localhost:5000/api/products
```

主动健康检查发现 `5101` 不健康后，请求会继续转发到 `5102`。这就是健康检查和负载均衡一起工作的效果。

如果两个商品服务实例都停止，再访问商品接口时，网关就没有可用目标，通常会返回 `503`。

这个 Demo 基本覆盖了一个 `.NET` 项目里使用 `YARP` 做网关的主干能力。后续要继续增强，可以再加：

* JWT 认证授权
* CORS 策略
* OpenTelemetry 链路追踪
* Swagger 聚合
* Consul 或 Kubernetes 服务发现
* 灰度发布和动态路由配置

### YARP 和 Ocelot、Nginx 怎么选？

| 方案 | 更适合的场景 |
| --- | --- |
| `YARP` | `.NET` 技术栈、需要 C# 深度定制、网关和业务基础设施集成较多 |
| `Ocelot` | 较简单的 `.NET` API 网关场景，已有项目继续维护 |
| `Nginx` | 通用反向代理、静态资源、TLS 入口、边缘层流量转发 |
| `Envoy` | 服务网格、复杂流量治理、云原生基础设施层 |

`YARP` 不应该被理解成“全面替代 Nginx”。更准确的说法是：

* `Nginx` 更像基础设施层通用入口
* `YARP` 更像 `.NET` 应用层可编程网关

很多生产架构也会组合使用：

```text
Internet
   |
   v
Nginx / Ingress
   |
   v
YARP Gateway
   |
   v
.NET Services
```

边缘入口归基础设施处理，业务网关归 `.NET` 项目处理。

### 生产环境注意点

#### 1. 网关不要写太多业务逻辑

`YARP` 很容易扩展，但网关层不适合塞复杂业务。

适合放在网关层的逻辑：

* 认证授权
* 限流
* 跨域
* 路由
* 请求头处理
* 日志追踪
* 简单灰度规则

不适合放在网关层的逻辑：

* 复杂订单计算
* 复杂状态变更
* 大量业务查询拼装
* 强业务规则判断

网关一旦变成“超级业务服务”，后期维护会很难。

#### 2. 超时要明确配置

网关默认帮忙转发，但不会自动替项目设计容错边界。

后端变慢时，网关需要明确超时策略，避免请求一直占着连接。

可以在集群里配置 HTTP 客户端行为：

```json
"HttpClient": {
  "MaxConnectionsPerServer": 100
},
"HttpRequest": {
  "ActivityTimeout": "00:00:30"
}
```

具体数值要结合业务接口耗时、并发量、后端容量压测后确定。

#### 3. 日志和链路追踪要先接好

网关是所有请求的入口，一旦出问题，经常会被第一时间怀疑。

至少要能看到：

* 请求命中了哪条路由
* 转发到了哪个集群
* 后端返回了什么状态码
* 请求总耗时是多少
* 后端连接是否失败
* 是否触发限流或授权失败

再进一步，可以接入 `OpenTelemetry`，把网关和后端服务串成完整链路。

#### 4. 路由命名要稳定

路由名、集群名不要随便写：

```text
route1
cluster1
test
aaa
```

更推荐这样：

```text
product-route
product-cluster
order-route
order-cluster
```

命名稳定后，日志、监控、排障都会更清楚。

#### 5. 配置变更要有校验

动态配置很方便，但也意味着错误配置可能很快影响线上流量。

生产环境建议加上：

* 配置格式校验
* 路由冲突检查
* 后端地址合法性检查
* 灰度发布
* 配置版本记录
* 快速回滚机制

### 常见问题

#### YARP 能不能代理 WebSocket？

可以。`YARP` 支持 `WebSocket`，常见的 `SignalR` 反向代理场景也能处理。

#### YARP 能不能代理 gRPC？

可以。`YARP` 基于 `.NET` 的 HTTP 基础设施，适合代理 `HTTP/2` 和 `gRPC` 场景。生产环境要注意端到端协议、TLS、HTTP/2 配置是否正确。

#### YARP 是不是只能做微服务网关？

不是。它本质是反向代理库，也能做：

* 本地开发代理
* BFF 网关
* 多后端统一入口
* 老系统迁移时的路径转发
* 灰度发布入口
* 内部服务统一访问层

#### YARP 能不能替代业务服务之间的服务发现？

不完全是。`YARP` 解决的是流量入口和代理转发。服务发现属于实例地址管理问题，可以配合 `Consul`、`Kubernetes`、`.NET Aspire` 或自定义配置源一起使用。

### 总结

`YARP` 是 `.NET` 里非常实用的反向代理库，核心思路并不复杂：

```text
Route 匹配请求
Cluster 管理后端集群
Destination 表示后端实例
Transform 修改代理请求
ASP.NET Core 管道负责扩展认证、限流、日志等能力
```

它最适合的场景，是 `.NET` 项目需要一个可编程、可扩展、能和现有技术栈深度集成的网关。

如果只是做静态资源和最外层入口，`Nginx` 依然很合适；如果需要在 `.NET` 微服务体系里做统一认证、路由、限流、灰度和服务聚合，`YARP` 会更顺手。

### 参考资料

* Microsoft Learn：Getting started with YARP  
  https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/yarp/getting-started

* Microsoft Learn：YARP Configuration Files  
  https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/yarp/config-files

* Microsoft Learn：YARP Load Balancing  
  https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/yarp/load-balancing

* Microsoft Learn：YARP Request and Response Transforms  
  https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/yarp/transforms

* NuGet：Yarp.ReverseProxy  
  https://www.nuget.org/packages/Yarp.ReverseProxy
