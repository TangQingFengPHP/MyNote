### 简介

前面几篇已经把 `YARP` 的基础代理、JWT、CORS、OpenTelemetry 都串起来了。

服务发现是网关实战里很关键的一步：

```text
YARP 后端地址不要写死，改成从服务发现系统里动态获取。
```

普通配置里，后端地址通常写在 `appsettings.json`：

```json
"Destinations": {
  "product-1": {
    "Address": "http://localhost:5101/"
  },
  "product-2": {
    "Address": "http://localhost:5102/"
  }
}
```

这在本地 Demo 没问题，但到了真实环境会遇到这些情况：

* 服务实例会扩容、缩容
* 实例地址会变化
* 某个实例挂了以后不能继续转发过去
* 后端服务部署在容器、虚拟机或 Kubernetes 里
* 网关配置不能每次都手动改 JSON

这时就需要服务发现。

典型链路变成：

```text
后端服务启动
  |
  v
注册到 Consul / Kubernetes
  |
  v
YARP Gateway 查询健康实例
  |
  v
动态生成 Routes / Clusters
  |
  v
请求转发到当前可用实例
```

一句话说清楚：

> `YARP` 负责代理转发，`Consul` 或 `Kubernetes` 负责告诉 YARP 当前有哪些服务实例可用。

### YARP 自己有没有服务发现？

`YARP` 本身不绑定某一个服务发现系统。

它的核心配置仍然是：

```text
Route
Cluster
Destination
```

但是 `YARP` 提供了扩展点，可以把配置来源换掉。

最关键的是：

```csharp
IProxyConfigProvider
IProxyConfig
IChangeToken
```

官方文档里提到，`IProxyConfigProvider` 负责返回当前代理配置，`IProxyConfig` 里包含 routes、clusters 和一个 `IChangeToken`。当配置变化时，触发 change token，`YARP` 就会重新加载配置。

所以服务发现接入 `YARP` 的本质是：

```text
从 Consul / Kubernetes 查询实例
  |
  v
转换成 YARP RouteConfig / ClusterConfig / DestinationConfig
  |
  v
通知 YARP 配置已经变化
```

### Consul 和 Kubernetes 怎么选？

两者解决的问题有重叠，但使用场景不一样。

| 方案 | 更适合的场景 |
| --- | --- |
| `Consul` | 虚拟机、物理机、混合部署、非 Kubernetes 环境 |
| `Kubernetes Service` | 服务都跑在 Kubernetes 内部 |
| `Kubernetes EndpointSlice` | 需要感知 Pod 级实例变化 |
| `DNS` | 只需要服务级负载均衡，不关心具体实例 |

最务实的判断方式：

```text
服务已经在 Kubernetes 里，优先使用 Kubernetes Service。
服务分散在 VM、物理机、容器混合环境里，Consul 更自然。
```

下面先完整做 `Consul` 实战，再讲 `Kubernetes` 的两种落地方式。

### Consul Demo 目标

项目结构：

```text
YarpDiscoveryDemo
├── Gateway
└── ProductService
```

目标效果：

* 启动本地 Consul
* 启动两个 `ProductService` 实例
* 两个实例注册到 Consul
* `Gateway` 从 Consul 查询健康实例
* `Gateway` 动态生成 `YARP` 后端集群
* 请求 `/api/products` 自动转发到健康实例

链路：

```text
curl /api/products
  |
  v
Gateway
  |
  | 查询 Consul 里的 product-service 健康实例
  v
ProductService:5101 / ProductService:5102
```

### 启动 Consul

本地可以直接用 Docker 启动 Consul：

```bash
docker run --rm --name consul \
  -p 8500:8500 \
  hashicorp/consul:1.19 \
  agent -dev -client=0.0.0.0
```

打开 Consul UI：

```text
http://localhost:8500
```

### 创建项目

```bash
mkdir YarpDiscoveryDemo
cd YarpDiscoveryDemo

dotnet new sln -n YarpDiscoveryDemo

dotnet new web -n Gateway
dotnet new web -n ProductService

dotnet sln add Gateway/Gateway.csproj
dotnet sln add ProductService/ProductService.csproj

dotnet add Gateway/Gateway.csproj package Yarp.ReverseProxy
dotnet add Gateway/Gateway.csproj package Consul

dotnet add ProductService/ProductService.csproj package Consul
```

### ProductService：注册到 Consul

`ProductService` 启动时把自己注册到 Consul，停止时从 Consul 注销。

修改 `ProductService/Program.cs`：

```csharp
using Consul;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton<IConsulClient>(_ =>
{
    return new ConsulClient(options =>
    {
        options.Address = new Uri("http://localhost:8500");
    });
});

builder.Services.AddHostedService<ConsulRegistrationService>();

var app = builder.Build();

app.MapGet("/products", (IConfiguration configuration) =>
{
    return Results.Ok(new
    {
        Service = "ProductService",
        InstanceId = configuration["Service:Id"],
        Port = configuration["Service:Port"],
        Data = new[]
        {
            new { Id = 1, Name = "Keyboard", Price = 199 },
            new { Id = 2, Name = "Mouse", Price = 99 }
        }
    });
});

app.MapGet("/health", () => Results.Ok("Healthy"));

app.Run();

public sealed class ConsulRegistrationService : IHostedService
{
    private readonly IConsulClient _consulClient;
    private readonly IConfiguration _configuration;
    private readonly ILogger<ConsulRegistrationService> _logger;
    private string? _serviceId;

    public ConsulRegistrationService(
        IConsulClient consulClient,
        IConfiguration configuration,
        ILogger<ConsulRegistrationService> logger)
    {
        _consulClient = consulClient;
        _configuration = configuration;
        _logger = logger;
    }

    public async Task StartAsync(CancellationToken cancellationToken)
    {
        var serviceName = _configuration["Service:Name"] ?? "product-service";
        var serviceId = _configuration["Service:Id"] ?? $"{serviceName}-{Guid.NewGuid():N}";
        var serviceAddress = _configuration["Service:Address"] ?? "host.docker.internal";
        var servicePort = int.Parse(_configuration["Service:Port"] ?? "5101");

        _serviceId = serviceId;

        var registration = new AgentServiceRegistration
        {
            ID = serviceId,
            Name = serviceName,
            Address = serviceAddress,
            Port = servicePort,
            Tags = new[] { "yarp", "product" },
            Check = new AgentServiceCheck
            {
                HTTP = $"http://{serviceAddress}:{servicePort}/health",
                Interval = TimeSpan.FromSeconds(10),
                Timeout = TimeSpan.FromSeconds(2),
                DeregisterCriticalServiceAfter = TimeSpan.FromMinutes(1)
            }
        };

        await _consulClient.Agent.ServiceRegister(registration, cancellationToken);

        _logger.LogInformation(
            "Registered service {ServiceName}, id={ServiceId}, address={Address}:{Port}",
            serviceName,
            serviceId,
            serviceAddress,
            servicePort);
    }

    public async Task StopAsync(CancellationToken cancellationToken)
    {
        if (!string.IsNullOrWhiteSpace(_serviceId))
        {
            await _consulClient.Agent.ServiceDeregister(_serviceId, cancellationToken);
            _logger.LogInformation("Deregistered service {ServiceId}", _serviceId);
        }
    }
}
```

这里有一个容易踩坑的点：

```csharp
var serviceAddress = _configuration["Service:Address"] ?? "host.docker.internal";
```

如果 Consul 运行在 Docker 容器里，而 `ProductService` 运行在宿主机上，注册成 `localhost` 通常不对。因为对 Consul 容器来说，`localhost` 指的是 Consul 容器自己，不是宿主机上的服务。

在 macOS 和 Windows Docker Desktop 里，`host.docker.internal` 通常可以从容器访问宿主机。

Linux 环境要按实际网络调整，可以使用宿主机 IP，或者把服务也放进同一个 Docker 网络。

### 启动两个 ProductService 实例

启动第一个实例：

```bash
dotnet run --project ProductService/ProductService.csproj \
  --urls http://localhost:5101 \
  --Service:Name=product-service \
  --Service:Id=product-service-5101 \
  --Service:Address=host.docker.internal \
  --Service:Port=5101
```

启动第二个实例：

```bash
dotnet run --project ProductService/ProductService.csproj \
  --urls http://localhost:5102 \
  --Service:Name=product-service \
  --Service:Id=product-service-5102 \
  --Service:Address=host.docker.internal \
  --Service:Port=5102
```

打开 Consul UI：

```text
http://localhost:8500
```

应该能看到：

```text
product-service
```

并且有两个健康实例。

### Gateway：动态配置提供器

网关不再从 `appsettings.json` 读取后端地址，而是从 Consul 查询实例，再动态生成 YARP 配置。

先写一个 `DynamicProxyConfig`。

新建 `Gateway/DynamicProxyConfig.cs`：

```csharp
using Microsoft.Extensions.Primitives;
using Yarp.ReverseProxy.Configuration;

public sealed class DynamicProxyConfig : IProxyConfig
{
    public DynamicProxyConfig(
        IReadOnlyList<RouteConfig> routes,
        IReadOnlyList<ClusterConfig> clusters,
        IChangeToken changeToken)
    {
        Routes = routes;
        Clusters = clusters;
        ChangeToken = changeToken;
    }

    public IReadOnlyList<RouteConfig> Routes { get; }

    public IReadOnlyList<ClusterConfig> Clusters { get; }

    public IChangeToken ChangeToken { get; }
}
```

再写一个配置提供器。

新建 `Gateway/DynamicProxyConfigProvider.cs`：

```csharp
using Microsoft.Extensions.Primitives;
using Yarp.ReverseProxy.Configuration;

public sealed class DynamicProxyConfigProvider : IProxyConfigProvider
{
    private readonly object _lock = new();
    private CancellationTokenSource _changeTokenSource = new();
    private DynamicProxyConfig _config;

    public DynamicProxyConfigProvider()
    {
        _config = new DynamicProxyConfig(
            Array.Empty<RouteConfig>(),
            Array.Empty<ClusterConfig>(),
            new CancellationChangeToken(_changeTokenSource.Token));
    }

    public IProxyConfig GetConfig()
    {
        return _config;
    }

    public void Update(
        IReadOnlyList<RouteConfig> routes,
        IReadOnlyList<ClusterConfig> clusters)
    {
        CancellationTokenSource previousToken;

        lock (_lock)
        {
            previousToken = _changeTokenSource;
            _changeTokenSource = new CancellationTokenSource();

            _config = new DynamicProxyConfig(
                routes,
                clusters,
                new CancellationChangeToken(_changeTokenSource.Token));
        }

        previousToken.Cancel();
    }
}
```

这段代码的关键是：

```text
Update 新配置
  |
  v
替换 Routes / Clusters
  |
  v
触发旧 ChangeToken
  |
  v
YARP 重新调用 GetConfig()
```

### Gateway：从 Consul 同步实例

接着写一个后台服务，定时从 Consul 拉取健康实例，并更新 YARP 配置。

新建 `Gateway/ConsulYarpConfigRefreshService.cs`：

```csharp
using Consul;
using Yarp.ReverseProxy.Configuration;

public sealed class ConsulYarpConfigRefreshService : BackgroundService
{
    private readonly IConsulClient _consulClient;
    private readonly DynamicProxyConfigProvider _configProvider;
    private readonly ILogger<ConsulYarpConfigRefreshService> _logger;

    public ConsulYarpConfigRefreshService(
        IConsulClient consulClient,
        DynamicProxyConfigProvider configProvider,
        ILogger<ConsulYarpConfigRefreshService> logger)
    {
        _consulClient = consulClient;
        _configProvider = configProvider;
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            try
            {
                await RefreshAsync(stoppingToken);
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Refresh YARP config from Consul failed");
            }

            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }

    private async Task RefreshAsync(CancellationToken cancellationToken)
    {
        var result = await _consulClient.Health.Service(
            service: "product-service",
            tag: null,
            passingOnly: true,
            ct: cancellationToken);

        var destinations = result.Response
            .Select(entry =>
            {
                var address = string.IsNullOrWhiteSpace(entry.Service.Address)
                    ? entry.Node.Address
                    : entry.Service.Address;

                return new
                {
                    Id = entry.Service.ID,
                    Config = new DestinationConfig
                    {
                        Address = $"http://{address}:{entry.Service.Port}/"
                    }
                };
            })
            .GroupBy(x => x.Id)
            .ToDictionary(x => x.Key, x => x.First().Config);

        var routes = new[]
        {
            new RouteConfig
            {
                RouteId = "product-route",
                ClusterId = "product-cluster",
                Match = new RouteMatch
                {
                    Path = "/api/products/{**catch-all}"
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
                Destinations = destinations
            }
        };

        _configProvider.Update(routes, clusters);

        _logger.LogInformation(
            "YARP config refreshed from Consul, destination count: {Count}",
            destinations.Count);
    }
}
```

这段代码每 5 秒做一次：

```text
查询 product-service 健康实例
  |
  v
生成 DestinationConfig
  |
  v
生成 ClusterConfig
  |
  v
调用 DynamicProxyConfigProvider.Update()
  |
  v
YARP 热更新配置
```

### Gateway Program.cs

修改 `Gateway/Program.cs`：

```csharp
using Consul;
using Yarp.ReverseProxy.Configuration;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton<IConsulClient>(_ =>
{
    return new ConsulClient(options =>
    {
        options.Address = new Uri("http://localhost:8500");
    });
});

builder.Services.AddSingleton<DynamicProxyConfigProvider>();
builder.Services.AddSingleton<IProxyConfigProvider>(sp =>
    sp.GetRequiredService<DynamicProxyConfigProvider>());

builder.Services.AddHostedService<ConsulYarpConfigRefreshService>();

builder.Services.AddReverseProxy();

var app = builder.Build();

app.MapGet("/gateway/config", (DynamicProxyConfigProvider provider) =>
{
    var config = provider.GetConfig();

    return Results.Ok(new
    {
        Routes = config.Routes.Select(x => x.RouteId),
        Clusters = config.Clusters.Select(x => new
        {
            x.ClusterId,
            Destinations = x.Destinations?.Keys
        })
    });
});

app.MapReverseProxy();

app.Run();
```

这里没有：

```csharp
.LoadFromConfig(...)
```

因为配置来源已经不是 `appsettings.json`，而是自定义的 `IProxyConfigProvider`。

### 启动 Gateway

```bash
dotnet run --project Gateway/Gateway.csproj
```

查看当前网关动态配置：

```bash
curl http://localhost:5000/gateway/config
```

正常会看到：

```json
{
  "routes": [
    "product-route"
  ],
  "clusters": [
    {
      "clusterId": "product-cluster",
      "destinations": [
        "product-service-5101",
        "product-service-5102"
      ]
    }
  ]
}
```

访问商品接口：

```bash
curl http://localhost:5000/api/products
```

多请求几次，会看到不同实例返回的 `InstanceId`：

```text
product-service-5101
product-service-5102
```

这说明：

* Consul 里有两个健康实例
* Gateway 从 Consul 拉到了实例
* YARP 动态生成了 destination
* RoundRobin 负载均衡已经生效

### 故障测试

停掉 `5101` 这个实例。

等待 Consul 健康检查把它标记为不健康，再请求：

```bash
curl http://localhost:5000/gateway/config
curl http://localhost:5000/api/products
```

配置里应该只剩：

```text
product-service-5102
```

请求也会继续转发到 `5102`。

这就是服务发现接入 YARP 后的核心效果：

```text
后端实例变动
  |
  v
Consul 健康状态变化
  |
  v
Gateway 重新生成 YARP 配置
  |
  v
新请求只打到健康实例
```

### 为什么不用 YARP 自带健康检查？

这里容易混。

`YARP` 自己也有健康检查，但它解决的是：

```text
已知后端列表里，哪些目标还能不能用。
```

服务发现解决的是：

```text
后端列表本身从哪里来。
```

如果后端实例是固定的，可以只用 YARP 健康检查。

如果后端实例会动态变化，就需要 Consul、Kubernetes 或其他服务发现系统来提供实例列表。

两者可以一起用：

```text
Consul 提供实例列表
YARP 对已发现实例再做被动健康判断
```

### Kubernetes 方式一：直接转发到 Service DNS

在 Kubernetes 里，最常见、最推荐的方式不是让 YARP 直接感知每个 Pod，而是直接转发到 Kubernetes Service。

比如有一个 `product-service`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: product-service
  namespace: default
spec:
  selector:
    app: product-service
  ports:
    - name: http
      port: 80
      targetPort: 8080
```

YARP 直接配置：

```json
{
  "ReverseProxy": {
    "Routes": {
      "product-route": {
        "ClusterId": "product-cluster",
        "Match": {
          "Path": "/api/products/{**catch-all}"
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
        "Destinations": {
          "product-service": {
            "Address": "http://product-service.default.svc.cluster.local/"
          }
        }
      }
    }
  }
}
```

这时动态发现由 Kubernetes Service 完成：

```text
YARP -> Kubernetes Service -> Pod A / Pod B / Pod C
```

这种方式的优点：

* 配置简单
* 不需要 YARP 监听 Kubernetes API
* Pod 扩缩容由 Kubernetes Service 自动处理
* 适合绝大多数集群内服务调用

缺点是：

* YARP 看不到每个 Pod
* 细粒度负载均衡策略交给 Kubernetes
* 不适合按 Pod 元数据做复杂路由

多数情况下，这已经够用。

### Kubernetes 方式二：监听 EndpointSlice 动态生成 YARP 配置

如果确实需要 YARP 感知每个 Pod，例如：

* 按 Pod 版本做灰度
* 按标签筛选实例
* 把不同 Pod 生成不同 destination
* 需要 YARP 自己做负载均衡

就可以监听 Kubernetes `EndpointSlice`。

思路和 Consul 类似：

```text
Kubernetes API
  |
  v
查询 EndpointSlice
  |
  v
提取 Pod IP + Port
  |
  v
生成 YARP DestinationConfig
  |
  v
触发 IProxyConfigProvider 更新
```

需要安装 Kubernetes 客户端：

```bash
dotnet add Gateway/Gateway.csproj package KubernetesClient
```

代码骨架大概是：

```csharp
using k8s;
using k8s.Models;
using Yarp.ReverseProxy.Configuration;

public sealed class KubernetesYarpConfigRefreshService : BackgroundService
{
    private readonly IKubernetes _kubernetes;
    private readonly DynamicProxyConfigProvider _configProvider;

    public KubernetesYarpConfigRefreshService(
        IKubernetes kubernetes,
        DynamicProxyConfigProvider configProvider)
    {
        _kubernetes = kubernetes;
        _configProvider = configProvider;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var endpointSlices = await _kubernetes.CustomObjects
                .ListNamespacedCustomObjectAsync(
                    group: "discovery.k8s.io",
                    version: "v1",
                    namespaceParameter: "default",
                    plural: "endpointslices",
                    cancellationToken: stoppingToken);

            // 实战里建议把返回对象反序列化成 EndpointSlice 类型，
            // 再提取 addresses、ports、conditions.ready 等字段。
            // 最后生成 DestinationConfig：
            //
            // "pod-10-1-2-3" -> http://10.1.2.3:8080/

            await Task.Delay(TimeSpan.FromSeconds(5), stoppingToken);
        }
    }
}
```

这只是骨架，不建议直接复制进生产。

生产里要继续处理：

* RBAC 权限
* namespace
* label selector
* EndpointSlice 分页
* endpoint ready 状态
* IPv4 / IPv6
* service port 名称
* watch 断线重连
* 配置变化去重

所以在 Kubernetes 里，除非确实需要 Pod 级别控制，否则优先用 Service DNS。

### Kubernetes 里 YARP 网关怎么部署？

一个常见部署结构：

```text
Ingress / LoadBalancer
  |
  v
YARP Gateway Service
  |
  v
ProductService Service
  |
  v
ProductService Pods
```

`Gateway` 自身可以用 Deployment 部署：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: yarp-gateway
spec:
  replicas: 2
  selector:
    matchLabels:
      app: yarp-gateway
  template:
    metadata:
      labels:
        app: yarp-gateway
    spec:
      containers:
        - name: yarp-gateway
          image: example/yarp-gateway:1.0.0
          ports:
            - containerPort: 8080
```

对外再暴露一个 Service：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: yarp-gateway
spec:
  selector:
    app: yarp-gateway
  ports:
    - port: 80
      targetPort: 8080
```

`YARP` 后端地址则写 Kubernetes Service DNS：

```text
http://product-service.default.svc.cluster.local/
```

### Consul 和 Kubernetes 接法对比

| 维度 | Consul | Kubernetes Service DNS | Kubernetes EndpointSlice |
| --- | --- | --- | --- |
| 实例来源 | Consul Catalog / Health API | Service DNS | Kubernetes API |
| YARP 是否感知实例 | 是 | 否 | 是 |
| 实现复杂度 | 中 | 低 | 高 |
| 健康状态 | Consul Health Check | Kubernetes readiness | Endpoint ready 条件 |
| 适合场景 | VM、混合部署、非 K8s | K8s 内部常规服务调用 | Pod 级动态路由 |
| 推荐优先级 | 非 K8s 优先 | K8s 优先 | 特殊需求再用 |

### 生产环境注意点

#### 1. 不要频繁刷新 YARP 配置

服务发现系统可能会频繁变化。

如果每次微小变化都触发 YARP 配置更新，网关会产生不必要的抖动。

建议：

* 配置刷新加间隔
* 对比新旧 destination，变化后再更新
* 避免每秒全量刷新
* watch 断线后再 fallback 到轮询

#### 2. 空实例要有预期

Consul 或 Kubernetes 里可能一时没有健康实例。

这时 YARP 对应 cluster 没有可用 destination，通常会返回 `503`。

生产环境要配好：

* 告警
* 熔断
* 降级页
* 灰度回滚
* 上游重试策略

#### 3. 注册地址要能被网关访问

服务注册进 Consul 的地址，不一定就是网关能访问的地址。

常见错误：

```text
服务注册 localhost
网关在另一个容器里访问 localhost
结果访问到了网关容器自己
```

一定要确认：

```text
Consul 里的 Address + Port
```

对 Gateway 来说是真正可访问的。

#### 4. 健康检查不要只返回固定字符串

Demo 里：

```csharp
app.MapGet("/health", () => Results.Ok("Healthy"));
```

生产环境应该检查关键依赖：

* 数据库
* Redis
* MQ
* 下游服务
* 磁盘空间

否则服务看起来健康，实际业务请求仍然失败。

#### 5. 服务发现不是配置中心

服务发现负责：

```text
服务名 -> 健康实例列表
```

它不应该承载所有网关规则。

比如：

* 路由前缀
* 鉴权策略
* 限流策略
* CORS 策略
* 灰度规则

这些更适合放在配置中心、数据库、GitOps 配置或 Kubernetes CRD 里。

### 常见问题

#### Consul 里服务是红的

常见原因：

* `/health` 不通
* 注册地址写成了错误的 `localhost`
* Consul 容器访问不到宿主机服务
* 端口写错
* 防火墙拦截

先在 Consul 容器里验证：

```bash
docker exec -it consul sh
wget -qO- http://host.docker.internal:5101/health
```

能返回 `Healthy`，Consul 才能探活成功。

#### Gateway 配置里没有 destination

先查 Consul API：

```bash
curl "http://localhost:8500/v1/health/service/product-service?passing=true"
```

如果返回空数组，说明 Consul 没有健康实例。

如果 Consul 有实例，但 Gateway 没有 destination，再看 Gateway 日志里的刷新结果。

#### 为什么 Kubernetes 里不建议直接查 Pod？

因为 Kubernetes Service 已经帮忙维护了 endpoint 和负载均衡。

直接查 Pod 意味着网关要处理：

* Pod 创建
* Pod 删除
* readiness 变化
* EndpointSlice 分片
* watch 重连
* 网络策略

只有当业务真的需要 Pod 级别路由时，才值得做。

#### YARP 服务发现能不能和 JWT、CORS、OpenTelemetry 一起用？

可以。

服务发现只改变 `Clusters.Destinations` 的来源。

JWT、CORS、OpenTelemetry 仍然是 ASP.NET Core 管道和 YARP 路由层能力。

常见组合是：

```text
YARP
  + JWT 认证授权
  + CORS
  + OpenTelemetry
  + Consul / Kubernetes 服务发现
```

### 总结

`YARP` 接服务发现的关键不是某个固定库，而是理解这个模型：

```text
服务发现系统提供健康实例
  |
  v
转换成 YARP RouteConfig / ClusterConfig / DestinationConfig
  |
  v
IProxyConfigProvider 提供当前配置
  |
  v
IChangeToken 通知 YARP 配置变化
  |
  v
YARP 热更新路由和后端目标
```

在非 Kubernetes 环境里，`Consul + IProxyConfigProvider` 是很自然的方案。

在 Kubernetes 环境里，优先把 YARP 转发到 `Service DNS`，让 Kubernetes Service 处理 Pod 动态变化。只有需要 Pod 级别灰度、标签路由、实例感知时，再考虑监听 `EndpointSlice` 动态生成 YARP 配置。

### 参考资料

* [Microsoft Learn：YARP Configuration Providers](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/yarp/config-providers)

* [Microsoft Learn：YARP Configuration Files](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/yarp/config-files)

* [Consul API：Health - Service](https://developer.hashicorp.com/consul/api-docs/health#read-health-information-for-a-service)

* [Kubernetes：Services, Load Balancing, and Networking](https://kubernetes.io/docs/concepts/services-networking/service/)

* [Kubernetes：EndpointSlices](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)
