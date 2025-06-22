### 简介

`HttpClient` 是 `.NET` 中用于发送 `HTTP` 请求和接收 `HTTP` 响应的现代化 `API`，它取代了过时的 `WebClient` 和 `HttpWebRequest` 类。

`HttpClient` 是 `.NET Framework 4.5` + 和 `.NET Core/.NET 5+` 中提供的、基于消息处理管道（`message handler pipeline`）的现代 `HTTP` 客户端库。

相比早期的 `HttpWebRequest`，它更易用、支持异步、可扩展性强，并且在 `.NET Core` 中底层使用 `SocketsHttpHandler`，性能与可配置性都有显著提升。

**底层架构**

* `HttpClient`

    * 封装请求创建、发送及响应处理逻辑，提供 `GetAsync、PostAsync、SendAsync` 等方法。

* `HttpMessageHandler` 管道

    * 默认使用 `SocketsHttpHandler（.NET Core/.NET 5+）`，`.NET Framework` 下为 `HttpClientHandler`。

    * 可以通过继承 `DelegatingHandler` 串联多个自定义中间件（例如日志、重试、认证）。

* 连接池与长连接

    * `SocketsHttpHandler` 内置连接池和 `HTTP/2` 支持，同一目标主机复用 `TCP` 连接，减少握手与重建成本。

### HttpClient 核心特性

|   特性  |  说明   |  优势   |
| --- | --- | --- |
|  异步支持   |  所有方法都有异步版本   |  避免阻塞线程，提高并发能力   |
|  连接池   |  自动管理 HTTP 连接   |  减少 TCP 连接开销   |
|  超时控制   |  可配置请求超时时间   |  防止长时间阻塞   |
|  内容处理   |  内置多种内容处理器   |  简化 JSON/XML 处理   |
|  取消支持   |  支持 CancellationToken   |  优雅终止长时间请求   |
|  扩展性   |  可通过 DelegatingHandler 扩展	   |  实现日志、重试等中间件   |

### 常用方法

#### 使用单例

```csharp
public static class HttpClientSingleton
{
    public static readonly HttpClient Instance = new HttpClient();
}

// 使用
await HttpClientSingleton.Instance.GetAsync(url);
```

* 全局唯一，避免重复创建 `handler`，复用连接池。

* 需注意：设置到期（`Timeout`）、默认头（`DefaultRequestHeaders`）等全局属性变更会影响所有调用。

#### 使用 `HttpClientFactory` 工厂

```csharp
services.AddHttpClient("GitHub", client =>
{
    client.BaseAddress = new Uri("https://api.github.com/");
    client.DefaultRequestHeaders.Add("User-Agent", "MyApp");
})
.AddTransientHttpErrorPolicy(policy =>
    policy.WaitAndRetryAsync(3, _ => TimeSpan.FromSeconds(2)));

// 使用
var client = _httpClientFactory.CreateClient("GitHub");
```

* 框架管理生命周期、稳定重用 `SocketsHttpHandler`。

* 支持命名客户端、类型化客户端、配置管道、添加 `Polly` 弹性策略。

#### 类型化客户端

```csharp
public class GitHubService {
    private readonly HttpClient _client;
    public GitHubService(HttpClient client) => _client = client;
    // 封装API方法
}
// 注册
services.AddHttpClient<GitHubService>(client => { /* 配置 */ });
```

#### `GET` 请求

```csharp
var client = HttpClientSingleton.Instance;
var response = await client.GetAsync("https://api.example.com/items/1");
response.EnsureSuccessStatusCode();
string json = await response.Content.ReadAsStringAsync();
```

#### `POST` 请求

```csharp
var content = new StringContent(
    JsonSerializer.Serialize(new { Name = "测试" }),
    Encoding.UTF8, "application/json");
var postResp = await client.PostAsync("https://api.example.com/items", content);
postResp.EnsureSuccessStatusCode();
```

#### `SendAsync` 自定义请求

```csharp
var request = new HttpRequestMessage(HttpMethod.Put, $"items/{id}")
{
    Content = new StringContent(json, Encoding.UTF8, "application/json")
};
request.Headers.Add("X-Custom", "Value");

using var resp = await client.SendAsync(request, HttpCompletionOption.ResponseHeadersRead);
resp.EnsureSuccessStatusCode();
```

#### 流式上传/下载

```csharp
// 下载
using var resp = await client.GetAsync(fileUrl, HttpCompletionOption.ResponseHeadersRead);
using var stream = await resp.Content.ReadAsStreamAsync();
// 将 stream 写入文件 ...

// 上传
using var fs = File.OpenRead(localFilePath);
var streamContent = new StreamContent(fs);
var uploadResp = await client.PostAsync(uploadUrl, streamContent);
```

#### DelegatingHandler 管道

```csharp
public class LoggingHandler : DelegatingHandler
{
    protected override async Task<HttpResponseMessage> SendAsync(
        HttpRequestMessage request, CancellationToken ct)
    {
        Console.WriteLine($"Request: {request}");
        var resp = await base.SendAsync(request, ct);
        Console.WriteLine($"Response: {resp}");
        return resp;
    }
}

// 注册（HttpClientFactory）
services.AddHttpClient("WithLogging")
    .AddHttpMessageHandler<LoggingHandler>();
```

#### Polly 弹性策略

```csharp
services.AddHttpClient("GitHub")
    .AddTransientHttpErrorPolicy(p =>
        p.WaitAndRetryAsync(3, retry => TimeSpan.FromSeconds(1)));
```

* `AddTransientHttpErrorPolicy` 捕获 `5xx、408、HttpRequestException` 等 `transient` 错误；

* 可组合断路器、超时、隔离策略。

#### 认证与配置

* `Bearer Token`

```csharp
client.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", accessToken);
```

* `Client Certificates`

```csharp
var handler = new HttpClientHandler();
handler.ClientCertificates.Add(new X509Certificate2("cert.pfx", "pwd"));
var client = new HttpClient(handler);
```

* `代理`

```csharp
handler.Proxy = new WebProxy("http://127.0.0.1:8888");
handler.UseProxy = true;
```

#### 设置超时和请求头

```csharp
httpClient.Timeout = TimeSpan.FromSeconds(30); // 设置全局超时
httpClient.DefaultRequestHeaders.Add("User-Agent", "MyApp/1.0");
httpClient.DefaultRequestHeaders.Authorization = 
    new AuthenticationHeaderValue("Bearer", "your-token");
```  

#### 处理不同类型的响应

```csharp
// 获取 JSON 响应
var jsonResponse = await httpClient.GetStringAsync("https://api.example.com/json");

// 获取二进制响应（如文件下载）
var bytes = await httpClient.GetByteArrayAsync("https://example.com/file.pdf");

// 获取流响应（大文件下载，避免内存溢出）
await using var stream = await httpClient.GetStreamAsync("https://example.com/largefile.zip");

await using var fileStream = File.Create("largefile.zip");
await stream.CopyToAsync(fileStream);
```

#### 发送带参数的请求

```csharp
// GET 请求带查询参数
var queryParams = new Dictionary<string, string>
{
    { "page", "1" },
    { "size", "20" }
};
var uri = new UriBuilder("https://api.example.com/data")
{
    Query = new FormUrlEncodedContent(queryParams).ReadAsStringAsync().Result
};
var response = await httpClient.GetAsync(uri.Uri);

// POST 请求带表单数据
var formContent = new FormUrlEncodedContent(new[]
{
    new KeyValuePair<string, string>("username", "test"),
    new KeyValuePair<string, string>("password", "pass")
});
await httpClient.PostAsync("https://example.com/login", formContent);
```

#### 处理 HTTP 错误

```csharp
try
{
    var response = await httpClient.GetAsync("https://api.example.com/resource");
    response.EnsureSuccessStatusCode(); // 非 200 状态码会抛出异常
}
catch (HttpRequestException ex)
{
    Console.WriteLine($"HTTP Error: {ex.StatusCode}");
    Console.WriteLine($"Message: {ex.Message}");
}
```

#### 处理 gzip/deflate 压缩响应

```csharp
var httpClientHandler = new HttpClientHandler
{
    AutomaticDecompression = DecompressionMethods.GZip | DecompressionMethods.Deflate
};
var httpClient = new HttpClient(httpClientHandler);
```

#### 取消操作

```csharp
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(10)); // 10秒超时
try {
    var response = await client.GetAsync("https://slow-api.com", cts.Token);
} catch (TaskCanceledException) {
    // 处理超时
}
```