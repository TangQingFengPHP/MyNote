# 别再手写 args[0]：C#.NET CommandLineParser 命令行工具实战详解

控制台程序一开始往往只有几个参数：

```bash
dotnet run -- input.txt output.txt
```

参数一多，`args[0]`、`args[1]` 很快就会变成一场小型灾难：顺序记错会导致业务跑偏，漏传参数只能等到运行中抛异常，帮助信息和错误提示也需要另外维护。

`CommandLineParser` 把命令行参数映射到 C# 类型上，用属性声明参数，用一个解析入口完成类型转换、必填校验、默认值、帮助文本和错误处理。最终得到的是一个结构清楚的命令行工具，而不是一串难以维护的字符串数组。

本文使用 `CommandLineParser 2.x`，从一个最小示例开始，最后完成一个带 `scan`、`copy` 两个子命令的文件工具。

## CommandLineParser 解决了什么问题？

命令行传入的内容本质上都是字符串：

```text
--input ./data --count 10 --verbose
```

业务代码真正需要的却是：

```csharp
string input = "./data";
int count = 10;
bool verbose = true;
```

`CommandLineParser` 负责把前一种形式转换成后一种形式，并处理这些常见情况：

* `--input`、`-i` 这样的长参数和短参数
* `int`、`decimal`、`DateTime`、`enum`、可空类型等类型转换
* 必填参数和默认值
* `bool` 开关参数
* 一个参数接收多个值
* `git add` 一样的 Verb 子命令
* 自动生成 `--help` 和语法错误信息
* 将解析失败转换为合适的进程退出码

它只负责“把命令行解析成对象”，不会替业务代码执行文件复制、数据库迁移或网络请求。解析层和业务层分开，后续测试和维护都会轻松很多。

## 安装 NuGet 包

在控制台项目目录执行：

```bash
dotnet add package CommandLineParser --version 2.9.1
```

也可以直接在项目文件中添加：

```xml
<PackageReference Include="CommandLineParser" Version="2.9.1" />
```

官方包名是 `CommandLineParser`，代码中的命名空间是：

```csharp
using CommandLine;
```

该包的经典版本 2.9.1 面向 .NET Standard 2.0，同时兼容较新的 .NET 项目。项目升级时，建议通过 `dotnet list package --outdated` 检查实际可用版本，并锁定经过验证的版本。

## 第一个示例：把参数映射成 C# 对象

创建项目：

```bash
dotnet new console -n ParserDemo
cd ParserDemo
dotnet add package CommandLineParser --version 2.9.1
```

### 定义 Options 类

```csharp
using CommandLine;

public sealed class Options
{
    [Option('n', "name", Required = true, HelpText = "用户名称。")]
    public string Name { get; set; } = string.Empty;

    [Option('a', "age", Required = true, HelpText = "用户年龄。")]
    public int Age { get; set; }

    [Option('v', "verbose", HelpText = "输出详细信息。")]
    public bool Verbose { get; set; }
}
```

`[Option]` 中常见的几个参数如下：

| 写法 | 作用 |
| --- | --- |
| `'n'` | 短参数名，对应 `-n` |
| `"name"` | 长参数名，对应 `--name` |
| `Required = true` | 参数缺失时解析失败 |
| `HelpText` | 生成帮助信息时显示的说明 |

短参数和长参数指向同一个属性，因此下面两种写法等价：

```bash
dotnet run -- -n 张三 -a 28
dotnet run -- --name 张三 --age 28
```

`dotnet run --` 中间的双横线属于 `dotnet` 命令，作用是把后面的内容原样传给程序。发布成独立可执行文件后，可以直接写成：

```bash
ParserDemo --name 张三 --age 28
```

### 解析参数

`Program.cs`：

```csharp
using CommandLine;

return Parser.Default
    .ParseArguments<Options>(args)
    .MapResult(
        options => Run(options),
        errors =>
        {
            Console.Error.WriteLine("参数解析失败。");
            return 1;
        });

static int Run(Options options)
{
    Console.WriteLine($"姓名：{options.Name}");
    Console.WriteLine($"年龄：{options.Age}");

    if (options.Verbose)
    {
        Console.WriteLine("详细模式已开启。");
    }

    return 0;
}
```

运行结果：

```text
姓名：张三
年龄：28
详细模式已开启。
```

这里使用 `MapResult` 返回退出码。控制台程序约定返回 `0` 表示成功，非 `0` 表示失败，脚本和 CI/CD 就可以据此判断任务是否成功。

## 常用参数写法

### bool 是开关，不是普通字符串参数

下面的定义：

```csharp
[Option('v', "verbose", HelpText = "输出详细信息。")]
public bool Verbose { get; set; }
```

调用时只需要出现参数名：

```bash
app --verbose
```

参数出现后，`Verbose` 为 `true`；没有出现时为 `false`。通常不写成下面这样：

```bash
app --verbose true
```

对于 `bool` 选项，后面的 `true` 会被当成另一个位置参数，可能造成解析错误。如果业务确实需要显式传递真假值，可以使用 `bool?` 或字符串后自行转换，但普通开关不需要这一层复杂度。

### 默认值

`Default` 可以为可选参数提供默认值：

```csharp
public sealed class ExportOptions
{
    [Option('o', "output", Default = "./export", HelpText = "输出目录。")]
    public string Output { get; set; } = "./export";

    [Option('f', "format", Default = FileFormat.Json, HelpText = "导出格式。")]
    public FileFormat Format { get; set; }
}

public enum FileFormat
{
    Json,
    Csv
}
```

下面的命令没有传 `--output` 和 `--format`：

```bash
app
```

解析后等同于：

```text
Output = ./export
Format = Json
```

属性初始化器和 `Default` 最好保持一致。这样即使对象通过其他方式创建，默认行为也不会发生变化。

### 多值参数

使用 `IEnumerable<T>` 接收多个值：

```csharp
public sealed class SearchOptions
{
    [Option('e', "extension", Separator = ',', HelpText = "文件扩展名，使用逗号分隔。")]
    public IEnumerable<string> Extensions { get; set; } = Array.Empty<string>();
}
```

调用：

```bash
app --extension .jpg,.png,.gif
```

也可以让一个选项重复出现：

```bash
app --extension .jpg --extension .png
```

解析后的 `Extensions` 包含三个扩展名。`Separator = ','` 适合配置或脚本场景；重复参数更适合手工逐项追加。

### 位置参数

`[Value]` 用于接收没有 `--name` 的参数：

```csharp
public sealed class OpenOptions
{
    [Value(0, MetaName = "file", HelpText = "要打开的文件。")]
    public string File { get; set; } = string.Empty;

    [Value(1, MetaName = "line", HelpText = "起始行号。")]
    public int? Line { get; set; }
}
```

调用：

```bash
app notes.md 20
```

位置参数依赖顺序，可读性不如命名参数。参数数量固定、命令形式比较简短时适合使用；公共工具的关键参数通常更适合使用 `--input`、`--output` 这种命名方式。

## 自动生成帮助信息

给参数写好 `HelpText` 后，库会根据属性生成帮助文本：

```bash
app --help
```

输出大致如下：

```text
--name     Required. 用户名称。
--age      Required. 用户年龄。
-v, --verbose          输出详细信息。
--help                 Display this help screen.
--version              Display version information.
```

`--help`、`--version` 等内置行为能覆盖大多数工具的基础需求。帮助文本不是装饰，它是命令行工具的使用说明书，参数名、是否必填、默认值和示例都应该写清楚。

## 用 Verb 组织多个子命令

当一个程序同时支持“扫描文件”和“复制文件”时，把所有参数塞进一个 `Options` 类会变得混乱：扫描命令不需要输出目录，复制命令也不需要扩展名筛选。

这时可以使用 Verb：

```text
filetool scan --root ./logs --extension .log
filetool copy --source ./logs --target ./backup --overwrite
```

每个 Verb 对应一个独立的参数类，参数天然分组，帮助文本也会按命令分开显示。

## 完整 Demo：文件扫描与复制工具

下面的 Demo 包含：

* `scan`：递归统计指定目录下的文件
* `copy`：复制指定文件
* `enum`：将命令行文本转换为枚举
* `IEnumerable<string>`：接收多个扩展名
* `bool`：使用开关控制覆盖行为
* `MapResult`：把不同命令分发到不同方法
* 错误处理和退出码

### 创建项目

```bash
dotnet new console -n FileTool
cd FileTool
dotnet add package CommandLineParser --version 2.9.1
```

### 定义命令参数

新建 `Options.cs`：

```csharp
using CommandLine;

[Verb("scan", HelpText = "扫描目录并统计文件数量。")]
public sealed class ScanOptions
{
    [Option('r', "root", Required = true, HelpText = "要扫描的根目录。")]
    public string Root { get; set; } = string.Empty;

    [Option('e', "extension", Separator = ',', HelpText = "只统计指定扩展名，例如 .log,.txt。")]
    public IEnumerable<string> Extensions { get; set; } = Array.Empty<string>();

    [Option('v', "verbose", HelpText = "输出每个匹配文件。")]
    public bool Verbose { get; set; }
}

[Verb("copy", HelpText = "复制一个文件到目标路径。")]
public sealed class CopyOptions
{
    [Option('s', "source", Required = true, HelpText = "源文件路径。")]
    public string Source { get; set; } = string.Empty;

    [Option('t', "target", Required = true, HelpText = "目标文件路径。")]
    public string Target { get; set; } = string.Empty;

    [Option('o', "overwrite", HelpText = "目标文件存在时覆盖。")]
    public bool Overwrite { get; set; }
}
```

### 编写程序入口

`Program.cs`：

```csharp
using CommandLine;

return Parser.Default
    .ParseArguments<ScanOptions, CopyOptions>(args)
    .MapResult(
        (ScanOptions options) => RunScan(options),
        (CopyOptions options) => RunCopy(options),
        errors => HandleParseErrors(errors));

static int RunScan(ScanOptions options)
{
    if (!Directory.Exists(options.Root))
    {
        Console.Error.WriteLine($"目录不存在：{options.Root}");
        return 2;
    }

    var extensions = new HashSet<string>(
        options.Extensions.Select(NormalizeExtension),
        StringComparer.OrdinalIgnoreCase);

    var files = Directory.EnumerateFiles(
            options.Root,
            "*",
            SearchOption.AllDirectories)
        .Where(path =>
            extensions.Count == 0 ||
            extensions.Contains(Path.GetExtension(path)));

    var count = 0;
    foreach (var file in files)
    {
        count++;
        if (options.Verbose)
        {
            Console.WriteLine(file);
        }
    }

    Console.WriteLine($"共找到 {count} 个文件。");
    return 0;
}

static int RunCopy(CopyOptions options)
{
    if (!File.Exists(options.Source))
    {
        Console.Error.WriteLine($"源文件不存在：{options.Source}");
        return 2;
    }

    if (File.Exists(options.Target) && !options.Overwrite)
    {
        Console.Error.WriteLine("目标文件已存在，请添加 --overwrite。 ");
        return 3;
    }

    var targetDirectory = Path.GetDirectoryName(
        Path.GetFullPath(options.Target));

    if (!string.IsNullOrWhiteSpace(targetDirectory))
    {
        Directory.CreateDirectory(targetDirectory);
    }

    File.Copy(options.Source, options.Target, options.Overwrite);
    Console.WriteLine($"已复制：{options.Source} -> {options.Target}");
    return 0;
}

static int HandleParseErrors(IEnumerable<Error> errors)
{
    if (errors.Any(error => error is HelpRequestedError))
    {
        return 0;
    }

    if (errors.Any(error => error is VersionRequestedError))
    {
        return 0;
    }

    Console.Error.WriteLine("命令格式错误，请使用 --help 查看帮助。");
    return 1;
}

static string NormalizeExtension(string extension)
{
    return extension.StartsWith('.') ? extension : $".{extension}";
}
```

这段程序依赖顶层语句的静态局部函数，适用于现代 .NET 控制台项目。若项目使用传统 `Program` 类，只需将这些方法移动到类中即可。

### 运行 scan

扫描全部文件：

```bash
dotnet run -- scan --root ./logs
```

只扫描日志和文本文件，并显示文件名：

```bash
dotnet run -- scan --root ./logs --extension log,txt --verbose
```

### 运行 copy

```bash
dotnet run -- copy \
  --source ./logs/app.log \
  --target ./backup/app.log
```

目标文件已经存在时，程序返回退出码 `3`，并保留原文件：

```bash
dotnet run -- copy \
  --source ./logs/app.log \
  --target ./backup/app.log \
  --overwrite
```

### 查看帮助

```bash
dotnet run -- --help
dotnet run -- scan --help
dotnet run -- copy --help
```

为每个 Verb 单独生成帮助信息，是多命令工具比单个大参数类更容易维护的关键。

## `WithParsed`、`WithNotParsed` 和 `MapResult` 怎么选？

三种写法都能处理解析结果：

### `WithParsed`

只关心解析成功时的业务逻辑：

```csharp
Parser.Default
    .ParseArguments<Options>(args)
    .WithParsed(options => Run(options));
```

### `WithNotParsed`

需要单独处理错误集合时使用：

```csharp
Parser.Default
    .ParseArguments<Options>(args)
    .WithParsed(options => Run(options))
    .WithNotParsed(errors =>
    {
        foreach (var error in errors)
        {
            Console.Error.WriteLine(error.Tag);
        }
    });
```

### `MapResult`

命令行程序通常需要返回退出码，或者存在多个 Verb。`MapResult` 可以让每个分支直接返回同一种结果，完整 Demo 采用的就是这种方式：

```csharp
return Parser.Default
    .ParseArguments<ScanOptions, CopyOptions>(args)
    .MapResult(
        scan => RunScan(scan),
        copy => RunCopy(copy),
        errors => 1);
```

简单单命令程序使用 `WithParsed` 足够；多命令程序或需要严格控制退出码时，`MapResult` 更清楚。

## 参数类型转换与校验边界

库会负责基础类型转换，例如：

```csharp
[Option("port", Required = true)]
public int Port { get; set; }

[Option("mode", Default = RunMode.Safe)]
public RunMode Mode { get; set; }

public enum RunMode
{
    Safe,
    Fast
}
```

下面的调用会自动把文本转换成 `int` 和 `RunMode`：

```bash
app --port 8080 --mode Fast
```

但“格式正确”不等于“业务合法”。例如 `--port -1` 能转换成整数，却不是有效端口；目录路径格式正确，也可能并不存在。建议分成两层处理：

1. 让 `CommandLineParser` 负责参数是否存在、类型是否能转换。
2. 在 `Run` 方法中负责范围、文件存在性、参数组合等业务校验。

这种分工能避免把业务规则塞进属性类，也能让错误提示更贴近实际问题。

## 常见错误和避坑建议

### 把位置参数和选项参数混在一起

`[Value(0)]` 依赖顺序，`[Option("input")]` 依赖名称。一个命令中两者都很多时，调用方式容易变得不直观。公共 CLI 优先采用命名选项，位置参数只保留给最核心、最短的输入。

### 忘记处理解析失败

只写 `WithParsed`，不代表错误已经被正确处理。缺少必填参数、类型转换失败、未知参数都会进入未解析分支。至少应该输出简短错误信息并返回非零退出码。

### 把业务异常当成解析异常

参数解析成功后，目录不存在、权限不足、文件冲突都属于业务执行错误。它们不应该伪装成“命令格式错误”，否则排查时很难判断问题发生在哪一层。

### `bool` 参数后面不要随意跟值

开关参数用：

```bash
--verbose
```

而不是：

```bash
--verbose true
```

### 共享参数可以抽成基类

多个 Verb 都需要 `--verbose`、`--config` 时，可以使用公共基类：

```csharp
public abstract class CommonOptions
{
    [Option('v', "verbose", HelpText = "输出详细信息。")]
    public bool Verbose { get; set; }
}

[Verb("scan")]
public sealed class ScanOptions : CommonOptions
{
    [Option("root", Required = true)]
    public string Root { get; set; } = string.Empty;
}
```

公共参数不宜过多。每个命令真正需要的参数放在自己的类型中，帮助信息才不会变成一张参数清单。

## CommandLineParser 适合什么场景？

比较适合：

* 内部运维工具
* 文件处理、数据转换和代码生成工具
* 数据库迁移、初始化和批处理程序
* CI/CD 中调用的 .NET 控制台程序
* 已经使用 CommandLineParser 2.x 的老项目

如果项目需要交互式终端界面、彩色表格、参数补全、复杂的依赖注入集成，`Spectre.Console.Cli` 可能更顺手；如果需要基于现代 `System.CommandLine` 的符号树和中间件能力，则应评估 `System.CommandLine`。库的选择应结合既有代码、团队熟悉度和发布周期，不必为了一个简单工具引入过重的框架。

## 总结

`CommandLineParser` 的核心思路可以概括为：

```text
命令行字符串
    ↓
[Option]、[Value]、[Verb] 描述参数
    ↓
Parser.Default.ParseArguments(...)
    ↓
强类型 Options 对象
    ↓
业务执行与退出码
```

最小程序可以只使用一个 `Options` 类；参数开始分组后，使用 Verb 拆分命令；需要脚本和 CI/CD 可靠执行时，使用 `MapResult` 统一返回退出码。

完整资料：

* [CommandLineParser 官方 GitHub README](https://github.com/commandlineparser/commandline/blob/master/README.md)
* [CommandLineParser NuGet 页面](https://www.nuget.org/packages/CommandLineParser)
* [官方 Wiki](https://github.com/commandlineparser/commandline/wiki)
