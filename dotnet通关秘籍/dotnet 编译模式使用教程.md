### 简介

在 `.NET` 中，`Debug` 和 `Release` 是两种常见的编译模式，它们的主要区别在于 编译优化、调试支持、性能 等方面。此外，`.NET` 也支持自定义编译模式，比如 `Staging`、`Production` 等，适用于不同的环境。

### Debug 与 Release 模式对比

|  特性   |  Debug   |  Release   |
| --- | --- | --- |
|  JIT 优化   |  关闭优化，便于调试   |  启用优化，提高性能   |
|  调试符号   |  生成完整的 PDB 调试信息   |  可能不会生成完整的调试信息   |
|  断点与调试   |  支持完整调试   |  可能无法调试某些优化代码   |
|  预处理指令   |  DEBUG 预处理指令有效   |  DEBUG 预处理指令无效   |
|  性能   |  代码运行较慢   |  代码运行更快   |
|  日志   |  通常启用详细日志   |  通常减少日志输出   |
|  代码移除   |  不会移除 Debug.Assert 代码   |  移除 Debug.Assert 代码   |
|  应用场景   |  开发和测试阶段   |  生产环境部署   |

### Debug 模式

`Debug` 模式主要用于开发和调试阶段，支持完整的调试信息和断点调试。

```csharp
using System;
using System.Diagnostics;

class Program
{
    static void Main()
    {
        Console.WriteLine("This is Debug mode.");

        Debug.WriteLine("This message appears only in Debug mode.");
        Debug.Assert(1 + 1 == 2, "Math is broken!");
    }
}
```

编译结果

* 生成 `.pdb` 调试符号文件，可用于 调试。

* `Debug.WriteLine` 代码 在 `Release` 模式下不会执行。

* `Debug.Assert` 代码 在 `Release` 模式下被移除。

### Release 模式

`Release` 模式用于 生产环境，编译时会进行 优化，移除不必要的调试信息，提高运行效率。

```csharp
using System;

class Program
{
    static void Main()
    {
        Console.WriteLine("This is Release mode.");

        Trace.WriteLine("This message appears in both Debug and Release modes.");
    }
}
```

编译结果

* 进行 `JIT` 优化，提高代码运行效率。

* `Debug.WriteLine、Debug.Assert` 代码 被移除。

* `Trace.WriteLine` 仍然可用，可以在调试和发布时进行日志记录。

### 自定义编译模式

在 `.csproj` 文件中添加如下内容：

```xml
<PropertyGroup Condition="'$(Configuration)' == 'Staging'">
  <DefineConstants>STAGING</DefineConstants> <!-- 定义预处理符 -->
  <Optimize>true</Optimize> <!-- 开启优化 -->
  <DebugType>portable</DebugType> <!-- 可选: full, pdbonly, portable -->
  <PlatformTarget>AnyCPU</PlatformTarget>
  <OutputPath>bin\Staging\</OutputPath> <!-- 输出路径 -->
</PropertyGroup>
```

说明：

* `DefineConstants`: 可在代码中使用 `#if STAGING` 条件编译

* `Optimize`: 控制是否优化 `IL` 代码

* `DebugType`: 控制生成哪种类型的调试信息

#### 使用条件编译指令

```csharp
#if DEBUG
    Console.WriteLine("调试模式");
#elif STAGING
    Console.WriteLine("预发布环境（Staging）");
#elif RELEASE
    Console.WriteLine("生产环境");
#endif
```

#### 命令行编译自定义模式

```shell
dotnet build -c Staging
```

### 自定义编译模式完整示例

#### 项目结构

```plaintext
StagingSample/
├── Program.cs
└── StagingSample.csproj
```

#### StagingSample.csproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <RootNamespace>StagingSample</RootNamespace>
  </PropertyGroup>

  <!-- Debug 模式配置 -->
  <PropertyGroup Condition="'$(Configuration)' == 'Debug'">
    <DefineConstants>DEBUG;TRACE</DefineConstants>
    <OutputPath>bin\Debug\</OutputPath>
    <DebugSymbols>true</DebugSymbols>
    <DebugType>portable</DebugType>
    <Optimize>false</Optimize>
  </PropertyGroup>

  <!-- Release 模式配置 -->
  <PropertyGroup Condition="'$(Configuration)' == 'Release'">
    <DefineConstants>RELEASE;TRACE</DefineConstants>
    <OutputPath>bin\Release\</OutputPath>
    <DebugSymbols>false</DebugSymbols>
    <DebugType>portable</DebugType>
    <Optimize>true</Optimize>
  </PropertyGroup>

  <!-- Staging 模式配置 -->
  <PropertyGroup Condition="'$(Configuration)' == 'Staging'">
    <DefineConstants>STAGING;TRACE</DefineConstants>
    <OutputPath>bin\Staging\</OutputPath>
    <DebugSymbols>true</DebugSymbols>
    <DebugType>portable</DebugType>
    <Optimize>true</Optimize>
  </PropertyGroup>

</Project>
```

#### Program.cs

```csharp
using System;

namespace StagingSample
{
    class Program
    {
        static void Main(string[] args)
        {
#if DEBUG
            Console.WriteLine("当前是 Debug 模式");
#elif RELEASE
            Console.WriteLine("当前是 Release 模式");
#elif STAGING
            Console.WriteLine("当前是 Staging 模式");
#else
            Console.WriteLine("未知模式");
#endif
            Console.WriteLine("Hello, .NET 编译模式演示！");
        }
    }
}
```

#### 编译方式

```shell
dotnet build -c Staging
```

输出文件会在：

```shell
bin\Staging\
```

运行后输出为：

```text
当前是 Staging 模式
Hello, .NET 编译模式演示！
```

#### 配置环境变量和参数

在 `Staging` 配置中加载不同的配置文件（如 `appsettings.Staging.json`）。

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  },
  "ApiEndpoint": "https://staging.api.example.com"
}
```

在代码中加载配置

```csharp
var builder = new ConfigurationBuilder()
    .SetBasePath(Directory.GetCurrentDirectory())
    .AddJsonFile("appsettings.json")
    .AddJsonFile($"appsettings.{Environment.GetEnvironmentVariable("ASPNETCORE_ENVIRONMENT")}.json", optional: true)
    .Build();

var endpoint = builder["ApiEndpoint"];
Console.WriteLine($"API 地址: {endpoint}");
```