# 别把登录写散了：C#.NET IdentityServer4 统一认证与 JWT 授权实战

### 简介

`IdentityServer4` 是一个基于 `ASP.NET Core` 的认证授权框架。

它主要做一件事：

```text
集中处理登录、发放 Token、保护 API。
```

在单体项目里，登录通常直接写在业务系统里：

```text
用户 -> Web 系统 -> Cookie / Session -> 业务接口
```

项目一多，问题就来了：

* 每个系统都要写登录
* 每个系统都要管用户状态
* 前端、App、后台管理端都要各自适配
* 微服务之间调用也需要鉴权
* 第三方系统接入时很难统一控制权限

于是认证授权中心就出现了：

```text
用户 / 客户端
  |
  | 申请 Token
  v
IdentityServer4
  |
  | 返回 Access Token
  v
业务 API
```

业务 API 不再负责登录，只负责验证 Token 是否可信、权限是否足够。

### 先说现状

`IdentityServer4` 曾经是 .NET 生态里非常流行的开源认证授权方案，但现在已经停止维护。

截至 2026 年，`IdentityServer4` 仓库已经归档，官方后续路线是 `Duende IdentityServer`。新项目不建议继续选择 `IdentityServer4`，更推荐：

* `Duende IdentityServer`
* `OpenIddict`
* `Keycloak`
* `Auth0`
* `Microsoft Entra ID`
* 简单项目直接使用 `ASP.NET Core Identity + JWT`

那为什么还值得学？

因为很多旧项目还在使用它，而且它非常适合理解 `OAuth 2.0`、`OpenID Connect`、`JWT`、`Scope`、`Client` 这些核心概念。

一句话：

```text
新项目谨慎选型，旧项目维护和协议学习仍然很有价值。
```

### IdentityServer4 到底是什么？

`IdentityServer4` 本质上是一个 `Security Token Service`，简称 `STS`。

直白点说，它是一个发令牌的服务。

常见能力包括：

* 登录认证
* 单点登录
* 签发 `Access Token`
* 签发 `Identity Token`
* 签发 `Refresh Token`
* 管理客户端
* 管理 API 权限范围
* 支持 OAuth 2.0
* 支持 OpenID Connect

它不直接替代业务系统，也不直接替代用户表。

更准确的关系是：

```text
ASP.NET Core Identity 负责用户、密码、角色、登录表单
IdentityServer4 负责协议、客户端、Scope、Token 签发
业务 API 负责验证 Token 和执行授权策略
```

### OAuth 2.0 和 OpenID Connect 的区别

这两个词经常一起出现，但不是一回事。

`OAuth 2.0` 解决的是授权问题：

```text
这个客户端能不能访问这个 API？
能访问哪些范围？
```

`OpenID Connect` 解决的是登录身份问题：

```text
当前登录的人是谁？
用户 ID 是多少？
昵称、邮箱等身份信息是什么？
```

可以这样理解：

```text
OAuth 2.0 = 管权限
OpenID Connect = 管登录身份
IdentityServer4 = 同时实现这两套协议
```

### 几个核心概念

IdentityServer4 里最常见的概念有这几个。

| 概念 | 说明 |
| --- | --- |
| `Client` | 客户端应用，例如后台管理站点、移动 App、服务端程序 |
| `ApiScope` | API 权限范围，例如 `order.read`、`order.write` |
| `ApiResource` | 被保护的 API 资源，例如 `order-api` |
| `IdentityResource` | 用户身份信息，例如 `openid`、`profile`、`email` |
| `User` | 用户，可以来自测试用户、数据库、ASP.NET Core Identity |
| `Access Token` | 调用 API 时携带的访问令牌 |
| `Identity Token` | 表示登录用户身份的令牌 |
| `Refresh Token` | 用来刷新 Access Token 的令牌 |

最容易混淆的是 `ApiResource` 和 `ApiScope`。

简单理解：

```text
ApiResource 是 API 本身
ApiScope 是 API 上的权限范围
```

例如订单服务：

```text
ApiResource: order-api

ApiScope:
  order.read
  order.write
```

客户端不应该直接申请整个订单 API，而是应该申请具体权限：

```text
order.read
```

这样权限粒度更清楚。

### 常见授权模式

IdentityServer4 支持多种授权模式，实际项目里最常见的是这几类。

| 授权模式 | 适合场景 | 说明 |
| --- | --- | --- |
| `Client Credentials` | 服务调用服务 | 没有用户参与，只代表客户端自己 |
| `Authorization Code` | MVC、BFF、服务端 Web | 用户跳转登录，服务端换取 Token |
| `Authorization Code + PKCE` | SPA、移动端 | 更适合公开客户端 |
| `Resource Owner Password` | 旧系统、自家高度可信客户端 | 用户名密码直接交给客户端，不推荐新项目使用 |
| `Refresh Token` | 长会话 | Access Token 过期后换新 Token |

这一篇的 Demo 先使用 `Client Credentials`。

原因很简单：它没有登录页面干扰，最适合理解 Token 签发和 API 验证的完整链路。

### Demo 目标

准备三个项目：

```text
IdentityServer4Demo
├── AuthServer    认证授权中心，负责签发 Token
├── OrderApi      订单 API，负责验证 Token
└── DemoClient    控制台客户端，负责申请 Token 并调用 API
```

最终效果：

```text
DemoClient
  |
  | POST /connect/token
  v
AuthServer
  |
  | access_token
  v
DemoClient
  |
  | Authorization: Bearer xxx
  v
OrderApi
```

端口规划：

| 项目 | 地址 |
| --- | --- |
| `AuthServer` | `https://localhost:5001` |
| `OrderApi` | `https://localhost:6001` |
| `DemoClient` | 控制台程序 |

### 创建解决方案

`IdentityServer4` 主要面向 `.NET Core 3.1` 时代。下面 Demo 按旧项目维护场景来写，建议使用 `.NET Core 3.1 SDK` 创建项目。

```bash
mkdir IdentityServer4Demo
cd IdentityServer4Demo

dotnet new sln -n IdentityServer4Demo

dotnet new web -n AuthServer
dotnet new webapi -n OrderApi
dotnet new console -n DemoClient

dotnet sln add AuthServer/AuthServer.csproj
dotnet sln add OrderApi/OrderApi.csproj
dotnet sln add DemoClient/DemoClient.csproj
```

安装包：

```bash
dotnet add AuthServer/AuthServer.csproj package IdentityServer4 --version 4.1.2
dotnet add OrderApi/OrderApi.csproj package Microsoft.AspNetCore.Authentication.JwtBearer --version 3.1.32
dotnet add DemoClient/DemoClient.csproj package IdentityModel --version 5.2.0
```

如果本机只有较新的 .NET SDK，旧项目可能需要安装 `.NET Core 3.1 SDK` 或调整目标框架。维护历史项目时，这一点经常会遇到。

### AuthServer 配置

先写认证授权中心。

创建 `AuthServer/Config.cs`：

```csharp
using IdentityServer4.Models;
using System.Collections.Generic;

namespace AuthServer
{
    public static class Config
    {
        public static IEnumerable<ApiScope> ApiScopes =>
            new List<ApiScope>
            {
                new ApiScope("order.read", "读取订单"),
                new ApiScope("order.write", "写入订单")
            };

        public static IEnumerable<ApiResource> ApiResources =>
            new List<ApiResource>
            {
                new ApiResource("order-api", "订单 API")
                {
                    Scopes = { "order.read", "order.write" }
                }
            };

        public static IEnumerable<Client> Clients =>
            new List<Client>
            {
                new Client
                {
                    ClientId = "order-worker",
                    ClientName = "订单后台任务",
                    AllowedGrantTypes = GrantTypes.ClientCredentials,
                    ClientSecrets =
                    {
                        new Secret("order-secret".Sha256())
                    },
                    AllowedScopes =
                    {
                        "order.read"
                    }
                }
            };
    }
}
```

这段配置里有三个重点：

* `order-api` 表示被保护的订单 API
* `order.read` 表示读取订单的权限
* `order-worker` 表示一个可以申请 Token 的客户端

`ClientSecrets` 里不能直接保存明文，`Sha256()` 会把密钥转成哈希值。

### 注册 IdentityServer4

修改 `AuthServer/Startup.cs`：

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.AspNetCore.Http;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

namespace AuthServer
{
    public class Startup
    {
        public void ConfigureServices(IServiceCollection services)
        {
            services
                .AddIdentityServer()
                .AddDeveloperSigningCredential()
                .AddInMemoryApiScopes(Config.ApiScopes)
                .AddInMemoryApiResources(Config.ApiResources)
                .AddInMemoryClients(Config.Clients);
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseRouting();

            app.UseIdentityServer();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapGet("/", async context =>
                {
                    await context.Response.WriteAsync("IdentityServer4 is running.");
                });
            });
        }
    }
}
```

`AddDeveloperSigningCredential()` 会生成开发用签名密钥。

它只适合开发环境，生产环境必须换成正式证书：

```csharp
.AddSigningCredential(certificate)
```

Token 不是随便生成一个字符串。IdentityServer4 会对 Token 进行签名，API 验证签名后才能确认：

```text
这个 Token 确实来自可信的 AuthServer
```

### 查看发现文档

启动 `AuthServer` 后访问：

```text
https://localhost:5001/.well-known/openid-configuration
```

如果配置正常，会看到一段 JSON。

里面包含：

* `issuer`
* `authorization_endpoint`
* `token_endpoint`
* `jwks_uri`
* `scopes_supported`
* `grant_types_supported`

这个地址叫发现文档。

客户端和 API 可以通过它知道：

```text
Token 到哪里申请？
签名公钥到哪里取？
支持哪些授权模式？
```

### OrderApi 配置

订单 API 要做两件事：

* 验证 `Bearer Token`
* 检查是否有 `order.read` 权限

修改 `OrderApi/Startup.cs`：

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Hosting;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

namespace OrderApi
{
    public class Startup
    {
        public void ConfigureServices(IServiceCollection services)
        {
            services.AddControllers();

            services
                .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
                .AddJwtBearer(options =>
                {
                    options.Authority = "https://localhost:5001";
                    options.Audience = "order-api";
                });

            services.AddAuthorization(options =>
            {
                options.AddPolicy("order.read", policy =>
                {
                    policy.RequireAuthenticatedUser();
                    policy.RequireClaim("scope", "order.read");
                });
            });
        }

        public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
        {
            if (env.IsDevelopment())
            {
                app.UseDeveloperExceptionPage();
            }

            app.UseRouting();

            app.UseAuthentication();
            app.UseAuthorization();

            app.UseEndpoints(endpoints =>
            {
                endpoints.MapControllers();
            });
        }
    }
}
```

注意中间件顺序：

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

认证要放在授权前面。

认证负责识别身份：

```text
Token 是否合法？
Token 是否过期？
Token 签名是否正确？
```

授权负责检查权限：

```text
当前身份是否允许访问这个接口？
有没有 order.read 这个 scope？
```

### OrderApi 如何获取公钥？

`OrderApi` 没有手动配置公钥，也不会去 AuthServer 数据库里查密钥。

关键配置是这一段：

```csharp
options.Authority = "https://localhost:5001";
options.Audience = "order-api";
```

`Authority` 表示当前 API 信任哪个认证服务器。

`JwtBearer` 中间件会根据 `Authority` 自动读取 OpenID Connect 发现文档：

```text
https://localhost:5001/.well-known/openid-configuration
```

发现文档里会包含一个重要字段：

```json
{
  "issuer": "https://localhost:5001",
  "jwks_uri": "https://localhost:5001/.well-known/openid-configuration/jwks"
}
```

`jwks_uri` 指向公钥集合地址。

`OrderApi` 会继续访问这个地址，拿到 AuthServer 暴露出来的公钥：

```text
https://localhost:5001/.well-known/openid-configuration/jwks
```

返回内容通常是 `JSON Web Key Set`，简称 `JWKS`：

```json
{
  "keys": [
    {
      "kty": "RSA",
      "use": "sig",
      "kid": "xxx",
      "alg": "RS256",
      "n": "...",
      "e": "AQAB"
    }
  ]
}
```

这里的 `kid` 是密钥 ID。

JWT 的 Header 里也会带 `kid`：

```json
{
  "alg": "RS256",
  "kid": "xxx",
  "typ": "at+jwt"
}
```

`OrderApi` 会用 JWT Header 里的 `kid` 去 JWKS 里找到对应公钥，然后用这个公钥验证 Token 签名。

### OrderApi 如何验证 Token？

`JwtBearer` 中间件拿到请求头里的 Token：

```http
Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6...
```

然后会做一系列验证：

```text
Token 格式是不是合法 JWT
签名是否能用 AuthServer 公钥验证通过
issuer 是否可信
audience 是否匹配当前 API
exp 是否已经过期
nbf 是否还没生效
```

对应到配置就是：

```csharp
options.Authority = "https://localhost:5001";
options.Audience = "order-api";
```

`Authority` 用来验证 `iss`：

```json
{
  "iss": "https://localhost:5001"
}
```

`Audience` 用来验证 `aud`：

```json
{
  "aud": "order-api"
}
```

所以完整逻辑可以理解成：

```text
这个 Token 是不是 https://localhost:5001 签发的？
这个 Token 是不是签给 order-api 用的？
这个 Token 有没有被篡改？
这个 Token 有没有过期？
```

这些都通过，认证才算成功。

注意，认证成功只代表 Token 是真的，不代表一定能访问接口。

接口上还有授权策略：

```csharp
[Authorize(Policy = "order.read")]
```

策略里要求 Token 必须带有 `order.read`：

```csharp
policy.RequireClaim("scope", "order.read");
```

所以最终结果是：

| 情况 | 结果 |
| --- | --- |
| 没带 Token | `401 Unauthorized` |
| Token 伪造、过期、签名错误、`aud` 不匹配 | `401 Unauthorized` |
| Token 合法，但没有 `order.read` | `403 Forbidden` |
| Token 合法，并且有 `order.read` | 允许访问 |

公钥也不是每次请求都重新拉取。

`JwtBearer` 内部会缓存发现文档和 JWKS。通常只有第一次验证、缓存过期、找不到匹配 `kid`、AuthServer 做密钥轮换时，才会重新获取公钥。

一句话概括：

```text
OrderApi 通过 Authority 找发现文档，再通过 jwks_uri 获取公钥，用公钥验证 JWT 签名，然后检查 issuer、audience、过期时间和 scope。
```

### 添加订单接口

创建 `OrderApi/Controllers/OrdersController.cs`：

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System.Collections.Generic;
using System.Linq;

namespace OrderApi.Controllers
{
    [ApiController]
    [Route("api/orders")]
    public class OrdersController : ControllerBase
    {
        [HttpGet]
        [Authorize(Policy = "order.read")]
        public IActionResult Get()
        {
            var orders = new[]
            {
                new { Id = 1, OrderNo = "SO202605180001", Amount = 99.80 },
                new { Id = 2, OrderNo = "SO202605180002", Amount = 268.00 }
            };

            var claims = User.Claims.Select(x => new
            {
                x.Type,
                x.Value
            });

            return Ok(new
            {
                Message = "订单数据读取成功",
                Orders = orders,
                Claims = claims
            });
        }
    }
}
```

这个接口加了：

```csharp
[Authorize(Policy = "order.read")]
```

没有 Token，会返回 `401`。

有 Token 但没有 `order.read`，会返回 `403`。

### DemoClient 调用

修改 `DemoClient/Program.cs`：

```csharp
using IdentityModel.Client;
using System;
using System.Net.Http;
using System.Net.Http.Headers;
using System.Threading.Tasks;

namespace DemoClient
{
    internal class Program
    {
        private static async Task Main()
        {
            using var httpClient = new HttpClient();

            var discovery = await httpClient.GetDiscoveryDocumentAsync("https://localhost:5001");
            if (discovery.IsError)
            {
                Console.WriteLine(discovery.Error);
                return;
            }

            var tokenResponse = await httpClient.RequestClientCredentialsTokenAsync(
                new ClientCredentialsTokenRequest
                {
                    Address = discovery.TokenEndpoint,
                    ClientId = "order-worker",
                    ClientSecret = "order-secret",
                    Scope = "order.read"
                });

            if (tokenResponse.IsError)
            {
                Console.WriteLine(tokenResponse.Error);
                return;
            }

            Console.WriteLine("Access Token:");
            Console.WriteLine(tokenResponse.AccessToken);
            Console.WriteLine();

            using var apiClient = new HttpClient();
            apiClient.DefaultRequestHeaders.Authorization =
                new AuthenticationHeaderValue("Bearer", tokenResponse.AccessToken);

            var response = await apiClient.GetAsync("https://localhost:6001/api/orders");
            var content = await response.Content.ReadAsStringAsync();

            Console.WriteLine($"StatusCode: {(int)response.StatusCode}");
            Console.WriteLine(content);
        }
    }
}
```

客户端完整做了三步：

```text
读取发现文档
申请 access_token
携带 Bearer Token 调用订单 API
```

### 启动 Demo

先启动认证中心：

```bash
dotnet run --project AuthServer/AuthServer.csproj --urls "https://localhost:5001"
```

再启动订单 API：

```bash
dotnet run --project OrderApi/OrderApi.csproj --urls "https://localhost:6001"
```

最后运行客户端：

```bash
dotnet run --project DemoClient/DemoClient.csproj
```

如果一切正常，控制台会先打印 `Access Token`，再打印订单接口返回的数据。

### 直接用 curl 测试

不用控制台客户端，也可以直接用 `curl`。

先申请 Token：

```bash
curl -k -X POST https://localhost:5001/connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=order-worker" \
  -d "client_secret=order-secret" \
  -d "scope=order.read"
```

返回结果类似：

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6...",
  "expires_in": 3600,
  "token_type": "Bearer",
  "scope": "order.read"
}
```

再调用 API：

```bash
curl -k https://localhost:6001/api/orders \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6..."
```

如果 Token 合法，返回订单数据。

如果不带 Token：

```bash
curl -k https://localhost:6001/api/orders
```

会返回：

```text
401 Unauthorized
```

### Token 里通常有什么？

`Access Token` 常见格式是 `JWT`。

JWT 分三段：

```text
header.payload.signature
```

例如：

```text
eyJhbGciOiJSUzI1NiIsImtpZCI6...eyJpc3MiOiJodHRwczov...Qm9v...
```

每一段含义如下：

| 部分 | 说明 |
| --- | --- |
| `header` | 签名算法、密钥 ID |
| `payload` | 发行方、过期时间、scope、client_id 等声明 |
| `signature` | 签名，防止 Token 被篡改 |

订单 API 不需要访问 AuthServer 数据库。

它只需要拿到 AuthServer 的公钥，就能验证 Token 签名。

这也是 JWT 在微服务里常见的原因：

```text
签发和验证可以分离。
```

### Client Credentials 适合什么场景？

`Client Credentials` 没有用户登录。

Token 代表的是客户端程序本身，不代表某个用户。

适合这些场景：

* 后台任务调用订单 API
* 支付服务调用会员服务
* 网关调用内部服务
* 定时任务同步数据
* 内部系统之间机器对机器调用

不适合这些场景：

* 用户登录
* 获取用户昵称
* 判断当前用户角色
* 操作当前用户自己的数据

因为这种模式里根本没有“当前用户”。

### 用户登录该用什么？

用户登录更适合 `Authorization Code`。

大致流程是：

```text
浏览器访问客户端系统
  |
  v
客户端系统跳转到 AuthServer 登录页
  |
  v
用户输入账号密码
  |
  v
AuthServer 登录成功后回跳客户端
  |
  v
客户端用 code 换 Token
```

如果是 SPA 或移动端，还要配合 `PKCE`。

`Resource Owner Password` 也能用用户名密码换 Token，但它要求客户端直接接触用户密码，风险更高。新系统一般不推荐。

### Scope 授权怎么拆？

不要把所有接口都塞进一个 `api` Scope。

比较实用的拆法是按业务能力拆：

```text
order.read
order.write
product.read
product.write
member.read
member.write
```

只读客户端只给读权限：

```csharp
AllowedScopes = { "order.read" }
```

后台管理端可以给更多权限：

```csharp
AllowedScopes = { "order.read", "order.write", "product.read", "product.write" }
```

API 端再用策略限制：

```csharp
options.AddPolicy("order.write", policy =>
{
    policy.RequireClaim("scope", "order.write");
});
```

控制器上使用：

```csharp
[Authorize(Policy = "order.write")]
```

这样权限边界会比单纯判断“是否登录”清楚很多。

### 生产环境不能这么简单

上面的 Demo 使用了内存配置和开发证书，只适合本地学习。

生产环境至少要处理这些问题。

### 1. 签名证书

不能使用：

```csharp
AddDeveloperSigningCredential()
```

应该使用正式证书：

```csharp
AddSigningCredential(certificate)
```

签名证书一旦丢失或频繁变化，已经签发的 Token 可能无法验证。

### 2. 配置持久化

Demo 使用：

```csharp
AddInMemoryClients()
AddInMemoryApiScopes()
AddInMemoryApiResources()
```

生产环境通常要放到数据库：

```text
Clients
ApiScopes
ApiResources
PersistedGrants
```

`PersistedGrants` 里会涉及授权码、刷新令牌、用户同意记录等运行时数据。

### 3. HTTPS

认证授权链路必须使用 HTTPS。

Token、授权码、Cookie、客户端密钥都属于敏感数据，明文传输风险很高。

### 4. 客户端密钥管理

`ClientSecret` 不要写死在前端代码里。

浏览器、移动端、小程序都属于公开客户端，不能安全保存密钥。

这类客户端应使用：

```text
Authorization Code + PKCE
```

### 5. Token 生命周期

Access Token 不要设置得过长。

常见策略是：

```text
Access Token 短一些
Refresh Token 长一些
Refresh Token 支持撤销和轮换
```

### 6. 日志和审计

认证系统需要重点记录：

* 登录成功
* 登录失败
* Token 签发失败
* 客户端密钥错误
* Scope 越权申请
* Refresh Token 使用异常

这些日志对排查问题和安全审计都很重要。

### IdentityServer4 和 ASP.NET Core Identity 的区别

这两个名字都带 `Identity`，但职责不同。

| 对比项 | ASP.NET Core Identity | IdentityServer4 |
| --- | --- | --- |
| 用户表 | 负责 | 不直接负责 |
| 密码哈希 | 负责 | 不直接负责 |
| 登录页面 | 可以负责 | 可以集成 |
| OAuth 2.0 | 不负责 | 负责 |
| OpenID Connect | 不负责 | 负责 |
| Token 签发 | 不负责 | 负责 |
| 客户端管理 | 不负责 | 负责 |
| Scope 管理 | 不负责 | 负责 |

常见组合是：

```text
ASP.NET Core Identity 管用户
IdentityServer4 管协议和 Token
```

### IdentityServer4 和 JWT 的关系

`JWT` 是一种 Token 格式。

`IdentityServer4` 是一个认证授权服务器。

关系类似：

```text
IdentityServer4 负责签发 JWT
业务 API 负责验证 JWT
```

不要把它们混成一个概念。

简单项目可以自己签发 JWT：

```text
登录接口校验账号密码
生成 JWT
API 验证 JWT
```

复杂项目更适合认证中心：

```text
多个客户端
多个 API
单点登录
第三方接入
刷新令牌
统一 Scope
统一用户授权
```

### 常见错误

### invalid_client

客户端不存在，或者密钥不对。

重点检查：

```text
client_id
client_secret
ClientSecrets
```

### invalid_scope

申请了没有授权的 Scope。

例如客户端只允许：

```csharp
AllowedScopes = { "order.read" }
```

却申请：

```text
scope=order.write
```

就会失败。

### 401 Unauthorized

通常表示没有通过认证。

常见原因：

* 没带 `Authorization` 请求头
* Token 过期
* Token 签名验证失败
* `Authority` 配错
* API 无法访问 AuthServer 发现文档或公钥地址

### 403 Forbidden

通常表示认证成功，但权限不足。

例如 Token 里没有：

```text
scope=order.read
```

但接口要求：

```csharp
[Authorize(Policy = "order.read")]
```

这时就是 `403`。

### IDX10214 Audience validation failed

API 验证的 `Audience` 和 Token 里的 `aud` 对不上。

重点检查：

```csharp
options.Audience = "order-api";
```

以及 AuthServer 里配置的：

```csharp
new ApiResource("order-api", "订单 API")
```

### 总结

`IdentityServer4` 的核心价值是把认证授权从业务系统里抽出来，变成一个统一的认证授权中心。

它负责：

* 管客户端
* 管 Scope
* 管协议流程
* 签发 Token
* 暴露发现文档和公钥

业务 API 负责：

* 验证 Token
* 检查 Scope
* 执行业务逻辑

旧项目维护时，理解 `Client`、`ApiScope`、`ApiResource`、`Access Token`、`Authority`、`Audience` 这几个点，基本就能看懂大多数 IdentityServer4 配置。

新项目选型时，不建议继续把 IdentityServer4 作为首选。更现实的选择是 `Duende IdentityServer`、`OpenIddict` 或托管身份平台。
