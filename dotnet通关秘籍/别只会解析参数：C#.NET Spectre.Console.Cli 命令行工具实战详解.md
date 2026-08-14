# 别只会解析参数：C#.NET Spectre.Console.Cli 命令行工具实战详解

命令行工具不只是把 `args` 解析成几个字符串。

一个真正方便使用的 CLI，通常还需要：

```text
清晰的子命令
自动生成帮助信息
强类型参数
参数校验
依赖注入
彩色输出、表格和进度条
明确的错误信息和退出码
```

`Spectre.Console.Cli` 把命令行解析和 `Spectre.Console` 的终端呈现能力组合在一起，适合构建部署工具、文件处理工具、代码生成器、数据迁移工具和 .NET Global Tool。

本文以 `Spectre.Console.Cli 0.55.0` 为基础，从一个最小的问候命令开始，逐步完成一个带 `scan`、`copy` 子命令的文件工具。示例会覆盖参数定义、默认值、必填选项、枚举、多值参数、验证、异步命令、依赖注入和终端表格输出。

## Spectre.Console.Cli 和 Spectre.Console 是什么关系？

这两个包经常一起出现，但职责不同。

### Spectre.Console：负责终端呈现

`Spectre.Console` 主要处理终端 UI：

```csharp
using Spectre.Console;

AnsiConsole.MarkupLine("[green]操作成功[/]");
```

它还支持：

* `Table`：表格
* `Panel`：面板
* `Tree`：树形结构
* `Progress`：进度条
* `Status`：状态动画
* `SelectionPrompt<T>`：交互式选择
* `TextPrompt<T>`：交互式输入
* Markdown 和 ANSI 颜色

### Spectre.Console.Cli：负责命令分发

`Spectre.Console.Cli` 主要处理：

* 命令和子命令注册
* 位置参数和命名选项
* 字符串到强类型的转换
* 默认值和必填项
* 自动帮助信息
* 参数验证
* 同步和异步命令
* 命令类的依赖注入
* 退出码和异常处理

可以把两者理解成：

```text
Spectre.Console.Cli  = 命令怎么定义、怎么解析、执行哪个类
Spectre.Console      = 执行过程中怎么把结果显示得更清楚
```

只做参数解析时，`CommandLineParser` 或 `System.CommandLine` 也能胜任；如果终端输出同样是产品体验的一部分，`Spectre.Console.Cli` 的组合会更顺手。

## 核心结构

`Spectre.Console.Cli` 的核心调用链如下：

```text
CommandApp
    ↓
Command<TSettings>
    ↓
CommandSettings
    ↓
Execute(context, settings, cancellation)
    ↓
Spectre.Console 输出业务结果
```

几个核心类型的职责：

| 类型 | 作用 | 常见写法 |
| --- | --- | --- |
| `CommandApp` | CLI 应用入口，负责注册和调度命令 | `new CommandApp()` |
| `CommandApp<TCommand>` | 单命令模式，泛型命令作为默认命令 | `new CommandApp<GreetCommand>()` |
| `Command<TSettings>` | 同步命令基类 | `Command<ScanSettings>` |
| `AsyncCommand<TSettings>` | 异步命令基类 | `AsyncCommand<CopySettings>` |
| `CommandSettings` | 参数模型，属性上放命令特性 | `[CommandOption]` |
| `CommandContext` | 当前命令的上下文 | 读取命令名称、参数信息等 |
| `ITypeRegistrar` | DI 容器适配接口 | 接入 Microsoft DI |
| `IAnsiConsole` | 可注入的终端输出接口 | 方便测试和替换输出 |

它的设计偏向“约定优于配置”：参数写在 Settings 类上，命令逻辑写在 Command 类里，程序入口只负责把命令注册到 `CommandApp`。

## 安装 NuGet 包

创建控制台项目：

```bash
dotnet new console -n FileTool
cd FileTool
dotnet add package Spectre.Console.Cli --version 0.55.0
```

如果需要显式使用 `AnsiConsole`、`Table`、`Progress` 等 UI 类型，可以同时添加：

```bash
dotnet add package Spectre.Console --version 0.55.0
```

项目文件示例：

```xml
<PackageReference Include="Spectre.Console.Cli" Version="0.55.0" />
<PackageReference Include="Spectre.Console" Version="0.55.0" />
```

`Spectre.Console.Cli` 的版本会独立演进，不能简单假设它永远和 `Spectre.Console` 使用同一个版本号。升级时以 NuGet 依赖解析结果为准，并在项目中锁定经过验证的版本。

## 第一个 Demo：问候命令

目标命令：

```bash
dotnet run -- World --count 3
```

输出：

```text
Hello, World!
Hello, World!
Hello, World!
```

### 定义 Settings

新建 `GreetSettings.cs`：

```csharp
using System.ComponentModel;
using Spectre.Console.Cli;

public sealed class GreetSettings : CommandSettings
{
    [CommandArgument(0, "<name>")]
    [Description("要问候的名称")]
    public string Name { get; init; } = string.Empty;

    [CommandOption("-c|--count")]
    [Description("问候次数")]
    [DefaultValue(1)]
    public int Count { get; init; }
}
```

这里有两个参数：

* `[CommandArgument(0, "<name>")]`：第一个位置参数，尖括号表示必填。
* `[CommandOption("-c|--count")]`：命名选项，同时支持 `-c` 和 `--count`。

`[DefaultValue(1)]` 表示不传 `--count` 时使用 `1`。属性初始化器也可以写成 `= 1`，但为了让自动帮助信息展示默认值，建议保留 `[DefaultValue]`。

### 定义 Command

新建 `GreetCommand.cs`：

```csharp
using Spectre.Console;
using Spectre.Console.Cli;

public sealed class GreetCommand : Command<GreetSettings>
{
    protected override int Execute(
        CommandContext context,
        GreetSettings settings,
        CancellationToken cancellation)
    {
        for (var i = 0; i < settings.Count; i++)
        {
            AnsiConsole.MarkupLine(
                $"Hello, [green]{Markup.Escape(settings.Name)}[/]!");
        }

        return 0;
    }
}
```

`Markup.Escape` 很重要。如果命令行输入中包含 `[` 或 `]`，直接拼到 Markup 字符串里可能被当成格式标签。外部输入进入 `MarkupLine` 前，应该先进行转义。

### 注册并运行

`Program.cs`：

```csharp
using Spectre.Console.Cli;

var app = new CommandApp<GreetCommand>();
return app.Run(args);
```

运行：

```bash
dotnet run -- World --count 3
```

`CommandApp<TCommand>` 适合单命令模式，命令本身不需要一个额外的名字。多命令工具则使用普通的 `CommandApp`，通过 `Configure` 注册子命令。

## CommandSettings 参数模型

`CommandSettings` 是命令的输入模型。框架在命令执行前，会把命令行文本转换并绑定到 Settings 属性。

### 位置参数

```csharp
public sealed class CopySettings : CommandSettings
{
    [CommandArgument(0, "<source>")]
    [Description("源文件")]
    public string Source { get; init; } = string.Empty;

    [CommandArgument(1, "[destination]")]
    [Description("目标路径，默认使用当前目录")]
    public string? Destination { get; init; }
}
```

模板中的：

* `<source>`：必填位置参数。
* `[destination]`：可选位置参数。

如果数组参数接收多个文件，它必须放在最后：

```csharp
[CommandArgument(0, "<files>")]
[Description("要处理的文件列表")]
public string[] Files { get; init; } = [];
```

调用：

```bash
tool file1.txt file2.txt file3.txt
```

### 命名选项

```csharp
public sealed class BuildSettings : CommandSettings
{
    [CommandOption("-c|--configuration")]
    [Description("构建配置")]
    [DefaultValue("Debug")]
    public string Configuration { get; init; } = "Debug";

    [CommandOption("--no-restore")]
    [Description("构建前不恢复 NuGet 包")]
    public bool NoRestore { get; init; }
}
```

布尔属性就是开关：

```bash
tool build --no-restore
```

出现 `--no-restore` 时值为 `true`，省略时为 `false`。普通开关不需要再跟 `true` 或 `false`。

### 必填选项

选项默认是可选的。如果命令要求必须提供某个命名选项，可以在特性中设置 `isRequired: true`：

```csharp
public sealed class DeploySettings : CommandSettings
{
    [CommandOption("-e|--environment <TARGET>", isRequired: true)]
    [Description("部署环境")]
    public string Environment { get; init; } = string.Empty;

    [CommandOption("-v|--version <VERSION>", isRequired: true)]
    [Description("部署版本")]
    public string Version { get; init; } = string.Empty;

    [CommandOption("--dry-run")]
    [Description("只预览，不执行修改")]
    public bool DryRun { get; init; }
}
```

缺少 `--environment` 或 `--version` 时，框架会在执行命令前输出错误，不会进入 `Execute`。

### 枚举参数

有限选项适合使用枚举：

```csharp
public enum OutputFormat
{
    Text,
    Json
}

public sealed class ScanSettings : CommandSettings
{
    [CommandOption("--format")]
    [Description("输出格式")]
    [DefaultValue(OutputFormat.Text)]
    public OutputFormat Format { get; init; } = OutputFormat.Text;
}
```

调用：

```bash
tool scan --format Json
```

输入不存在的枚举值时，解析会失败，并提示可接受的值。相比用字符串再手写判断，枚举的帮助信息和业务代码都更清楚。

### 多值选项

数组选项可以重复传入：

```csharp
public sealed class TagSettings : CommandSettings
{
    [CommandOption("-t|--tag <TAG>")]
    [Description("标签，可重复传入")]
    public string[] Tags { get; init; } = [];
}
```

调用：

```bash
tool --tag api --tag production --tag v2
```

最终 `Tags` 中包含三个值。相比把多个值拼成逗号分隔字符串，数组选项更适合脚本和参数中可能包含逗号的场景。

## 多命令应用：构建文件工具

单命令模式适合 `tool <arguments>`；当工具包含多个动作时，使用子命令：

```text
filetool
├── scan
├── copy
└── report
```

目标调用形式：

```bash
filetool scan ./logs --extension log --format Json
filetool copy ./logs/app.log ./backup/app.log --force
filetool report ./logs --top 10
```

### 配置命令注册

`Program.cs`：

```csharp
using Spectre.Console.Cli;

var app = new CommandApp();

app.Configure(config =>
{
    config.SetApplicationName("filetool");
    config.AddCommand<ScanCommand>("scan")
        .WithDescription("扫描目录中的文件");
    config.AddCommand<CopyCommand>("copy")
        .WithDescription("复制文件");
});

return app.Run(args);
```

`AddCommand<TCommand>("name")` 中的 `name` 就是命令行中的子命令名称。`WithDescription` 会影响自动生成的帮助信息。

## 完整 Demo：扫描目录并复制文件

下面的示例把命令解析、终端输出和文件业务串起来。

### `ScanSettings.cs`

```csharp
using System.ComponentModel;
using Spectre.Console.Cli;

public enum OutputFormat
{
    Text,
    Json
}

public sealed class ScanSettings : CommandSettings
{
    [CommandArgument(0, "<root>")]
    [Description("要扫描的目录")]
    public string Root { get; init; } = string.Empty;

    [CommandOption("-e|--extension <EXT>")]
    [Description("扩展名，可重复传入，例如 log、txt")]
    public string[] Extensions { get; init; } = [];

    [CommandOption("--format")]
    [Description("输出格式")]
    [DefaultValue(OutputFormat.Text)]
    public OutputFormat Format { get; init; } = OutputFormat.Text;

    [CommandOption("-v|--verbose")]
    [Description("显示匹配到的文件")]
    public bool Verbose { get; init; }

    public override ValidationResult Validate()
    {
        if (!Directory.Exists(Root))
        {
            return ValidationResult.Error($"目录不存在：{Root}");
        }

        return ValidationResult.Success();
    }
}
```

`Validate()` 在命令执行前调用。目录不存在属于 Settings 本身就能判断的条件，放在这里比进入 `Execute` 后再判断更合适。

### `ScanCommand.cs`

```csharp
using System.Text.Json;
using Spectre.Console;
using Spectre.Console.Cli;

public sealed class ScanCommand : Command<ScanSettings>
{
    private readonly IAnsiConsole _console;

    public ScanCommand(IAnsiConsole console)
    {
        _console = console;
    }

    protected override int Execute(
        CommandContext context,
        ScanSettings settings,
        CancellationToken cancellation)
    {
        HashSet<string> extensions = settings.Extensions
            .Select(NormalizeExtension)
            .ToHashSet(StringComparer.OrdinalIgnoreCase);

        FileInfo[] files = new DirectoryInfo(settings.Root)
            .EnumerateFiles("*", SearchOption.AllDirectories)
            .Where(file =>
                extensions.Count == 0 ||
                extensions.Contains(file.Extension))
            .ToArray();

        if (settings.Verbose)
        {
            var table = new Table();
            table.AddColumn("文件");
            table.AddColumn("大小");

            foreach (FileInfo file in files)
            {
                table.AddRow(
                    Markup.Escape(file.FullName),
                    $"{file.Length:N0} bytes");
            }

            _console.Write(table);
        }

        if (settings.Format == OutputFormat.Json)
        {
            string json = JsonSerializer.Serialize(new
            {
                Root = settings.Root,
                Count = files.Length,
                TotalBytes = files.Sum(file => file.Length)
            });

            _console.WriteLine(json);
        }
        else
        {
            long totalBytes = files.Sum(file => file.Length);
            _console.MarkupLine(
                $"共找到 [yellow]{files.Length}[/] 个文件，" +
                $"总大小 [green]{totalBytes:N0}[/] bytes。");
        }

        return 0;
    }

    private static string NormalizeExtension(string extension)
    {
        return extension.StartsWith('.') ? extension : $".{extension}";
    }
}
```

这里使用构造函数注入 `IAnsiConsole`，没有直接依赖静态 `AnsiConsole`。这样做有两个好处：

* 命令类更容易测试，可以注入 `TestConsole`。
* 输出目标可以替换成日志、文件或其他终端实现。

### `CopySettings.cs`

```csharp
using System.ComponentModel;
using Spectre.Console.Cli;

public sealed class CopySettings : CommandSettings
{
    [CommandArgument(0, "<source>")]
    [Description("源文件")]
    public string Source { get; init; } = string.Empty;

    [CommandArgument(1, "<destination>")]
    [Description("目标文件")]
    public string Destination { get; init; } = string.Empty;

    [CommandOption("-f|--force")]
    [Description("目标文件存在时覆盖")]
    public bool Force { get; init; }

    public override ValidationResult Validate()
    {
        if (!File.Exists(Source))
        {
            return ValidationResult.Error($"源文件不存在：{Source}");
        }

        if (Path.GetFullPath(Source)
            .Equals(Path.GetFullPath(Destination), StringComparison.OrdinalIgnoreCase))
        {
            return ValidationResult.Error("源文件和目标文件不能是同一个文件。");
        }

        return ValidationResult.Success();
    }
}
```

### `CopyCommand.cs`

```csharp
using Spectre.Console;
using Spectre.Console.Cli;

public sealed class CopyCommand : Command<CopySettings>
{
    private readonly IAnsiConsole _console;

    public CopyCommand(IAnsiConsole console)
    {
        _console = console;
    }

    protected override int Execute(
        CommandContext context,
        CopySettings settings,
        CancellationToken cancellation)
    {
        if (File.Exists(settings.Destination) && !settings.Force)
        {
            _console.MarkupLine(
                "[red]目标文件已存在，请添加 --force。[/]");
            return 3;
        }

        string? directory = Path.GetDirectoryName(
            Path.GetFullPath(settings.Destination));

        if (!string.IsNullOrWhiteSpace(directory))
        {
            Directory.CreateDirectory(directory);
        }

        File.Copy(settings.Source, settings.Destination, settings.Force);
        _console.MarkupLine(
            $"[green]复制完成：[/]{Markup.Escape(settings.Destination)}");

        return 0;
    }
}
```

### `Program.cs`

```csharp
using Microsoft.Extensions.DependencyInjection;
using Spectre.Console.Cli;

var services = new ServiceCollection();
var registrar = new TypeRegistrar(services);

var app = new CommandApp(registrar);

app.Configure(config =>
{
    config.SetApplicationName("filetool");

    config.AddCommand<ScanCommand>("scan")
        .WithDescription("扫描目录中的文件");

    config.AddCommand<CopyCommand>("copy")
        .WithDescription("复制文件");
});

return app.Run(args);
```

运行前还需要安装 Microsoft DI：

```bash
dotnet add package Microsoft.Extensions.DependencyInjection
```

`TypeRegistrar` 和 `TypeResolver` 是连接 Microsoft DI 与 `Spectre.Console.Cli` 的桥接代码：

```csharp
using Microsoft.Extensions.DependencyInjection;
using Spectre.Console.Cli;

public sealed class TypeRegistrar : ITypeRegistrar
{
    private readonly IServiceCollection _services;

    public TypeRegistrar(IServiceCollection services)
    {
        _services = services;
    }

    public ITypeResolver Build()
    {
        return new TypeResolver(_services.BuildServiceProvider());
    }

    public void Register(Type service, Type implementation)
    {
        _services.AddSingleton(service, implementation);
    }

    public void RegisterInstance(Type service, object implementation)
    {
        _services.AddSingleton(service, implementation);
    }

    public void RegisterLazy(Type service, Func<object> factory)
    {
        _services.AddSingleton(service, _ => factory());
    }
}

public sealed class TypeResolver : ITypeResolver, IDisposable
{
    private readonly IServiceProvider _provider;

    public TypeResolver(IServiceProvider provider)
    {
        _provider = provider;
    }

    public object? Resolve(Type? type)
    {
        return type is null ? null : _provider.GetService(type);
    }

    public void Dispose()
    {
        (_provider as IDisposable)?.Dispose();
    }
}
```

完整项目文件可以是：

```text
FileTool/
├── Program.cs
├── TypeRegistrar.cs
├── ScanSettings.cs
├── ScanCommand.cs
├── CopySettings.cs
└── CopyCommand.cs
```

运行扫描：

```bash
dotnet run -- scan ./logs
dotnet run -- scan ./logs --extension log --extension txt --verbose
dotnet run -- scan ./logs --format Json
```

运行复制：

```bash
dotnet run -- copy ./logs/app.log ./backup/app.log
dotnet run -- copy ./logs/app.log ./backup/app.log --force
```

查看帮助：

```bash
dotnet run -- --help
dotnet run -- scan --help
dotnet run -- copy --help
```

## 验证机制：`Settings.Validate()` 和命令验证

`Spectre.Console.Cli` 的验证可以分成两层。

### Settings 验证

`CommandSettings.Validate()` 适合检查参数自身和参数之间的关系：

```csharp
public override ValidationResult Validate()
{
    if (string.IsNullOrWhiteSpace(Source))
    {
        return ValidationResult.Error("必须提供源文件。");
    }

    if (Force && string.IsNullOrWhiteSpace(Destination))
    {
        return ValidationResult.Error(
            "使用 --force 时必须提供目标文件。");
    }

    return ValidationResult.Success();
}
```

### Command 验证

当验证需要 `CommandContext`、当前命令信息或外部状态时，可以在命令类中重写验证方法：

```csharp
public override ValidationResult Validate(
    CommandContext context,
    CopySettings settings)
{
    if (settings.Force &&
        Path.GetExtension(settings.Destination) != ".bak")
    {
        return ValidationResult.Error(
            "--force 只允许写入 .bak 文件。");
    }

    return ValidationResult.Success();
}
```

一般情况下，Settings 验证已经足够。需要访问上下文或服务时，再放到 Command 验证阶段。

### 验证时机

完整生命周期可以理解为：

```text
app.Run(args)
    ↓
解析命令和参数
    ↓
绑定 CommandSettings
    ↓
settings.Validate()
    ↓
command.Validate(...)
    ↓
Execute / ExecuteAsync
```

验证失败不会进入 `Execute`，可以避免业务方法里堆满参数防御代码。

## 异步命令和取消操作

长时间运行的文件、网络或数据库操作，适合继承 `AsyncCommand<TSettings>`：

```csharp
using Spectre.Console;
using Spectre.Console.Cli;

public sealed class UploadSettings : CommandSettings
{
    [CommandArgument(0, "<file>")]
    public string File { get; init; } = string.Empty;
}

public sealed class UploadCommand : AsyncCommand<UploadSettings>
{
    protected override async Task<int> ExecuteAsync(
        CommandContext context,
        UploadSettings settings,
        CancellationToken cancellation)
    {
        await UploadFileAsync(settings.File, cancellation);
        AnsiConsole.MarkupLine("[green]上传完成[/]");
        return 0;
    }

    private static async Task UploadFileAsync(
        string file,
        CancellationToken cancellation)
    {
        await Task.Delay(TimeSpan.FromSeconds(2), cancellation);
    }
}
```

不要在异步命令中忽略 `CancellationToken`。用户按下 Ctrl+C 后，取消信号应该继续传递到 HTTP 请求、文件流和数据库操作中。

## 用进度条改善长任务体验

`Spectre.Console` 的进度条可以直接放进命令的 `Execute` 中：

```csharp
using Spectre.Console;

AnsiConsole.Progress()
    .Start(ctx =>
    {
        ProgressTask task = ctx.AddTask("扫描文件");
        task.MaxValue = files.Length;

        foreach (FileInfo file in files)
        {
            ProcessFile(file);
            task.Increment(1);
        }
    });
```

如果业务是异步的，可以使用 `ProgressTask.Increment` 配合 `await`：

```csharp
await AnsiConsole.Progress()
    .StartAsync(async ctx =>
    {
        ProgressTask task = ctx.AddTask("上传文件");
        task.MaxValue = files.Length;

        foreach (FileInfo file in files)
        {
            await UploadAsync(file, cancellation);
            task.Increment(1);
        }
    });
```

进度条适合有明确总量的任务。总量未知或任务很短时，`Status` 或普通日志更合适。

## 分支命令和嵌套命令

命令层级较深时，可以使用 Branch：

```bash
filetool project build
filetool project clean
```

配置：

```csharp
app.Configure(config =>
{
    config.AddBranch("project", branch =>
    {
        branch.AddCommand<BuildCommand>("build")
            .WithDescription("构建项目");

        branch.AddCommand<CleanCommand>("clean")
            .WithDescription("清理构建输出");
    });
});
```

Branch 本身通常是命令分组，不负责实际业务。叶子命令才绑定 `Execute` 或 `ExecuteAsync`。

层级设计建议：

```text
领域分组      动作
project   →  build / clean
package   →  add / remove / list
database  →  migrate / seed / backup
```

命令名称应描述动作，分支名称应描述领域或资源。这样 `--help` 产生的结构更容易阅读。

## 自定义帮助和命令描述

帮助信息的质量取决于三个地方：命令描述、参数模板和属性描述。

```csharp
app.Configure(config =>
{
    config.SetApplicationName("filetool");
    config.SetApplicationVersion("1.2.0");

    config.AddCommand<ScanCommand>("scan")
        .WithDescription("递归扫描目录并统计文件大小")
        .WithExample(new[] { "scan", "./logs", "--extension", "log" });
});
```

参数写法：

```csharp
[CommandOption("-e|--extension <EXT>")]
[Description("扩展名，可以重复传入")]
public string[] Extensions { get; init; } = [];
```

好的帮助信息应该让使用者在不打开源码的情况下回答三个问题：

1. 这个命令做什么？
2. 参数是否必填，默认值是什么？
3. 一条可复制的调用示例是什么？

## 错误处理和退出码

命令行工具中的错误至少分成三类：

| 错误类型 | 示例 | 处理位置 |
| --- | --- | --- |
| 语法错误 | 少了必填参数、未知选项 | 框架自动处理 |
| 参数验证错误 | 目录不存在、参数组合不合法 | `Validate()` |
| 执行错误 | 权限不足、复制失败、网络异常 | `Execute` 中处理或统一异常处理 |

业务代码返回退出码：

```csharp
if (File.Exists(settings.Destination) && !settings.Force)
{
    AnsiConsole.MarkupLine("[red]目标文件已存在。[/]");
    return 3;
}

return 0;
```

建议建立简单的退出码约定：

```text
0：成功
1：通用错误
2：输入或资源不存在
3：冲突或拒绝覆盖
4：外部服务失败
```

脚本和 CI/CD 不应该依赖中文提示或彩色文本判断成功与否，应该读取退出码。

## 依赖注入：让 Command 只负责协调

命令类不适合承载所有文件、数据库和网络逻辑。可以把业务抽成服务：

```csharp
public interface IFileScanner
{
    IReadOnlyList<FileInfo> Scan(
        string root,
        IReadOnlySet<string> extensions);
}

public sealed class FileScanner : IFileScanner
{
    public IReadOnlyList<FileInfo> Scan(
        string root,
        IReadOnlySet<string> extensions)
    {
        return new DirectoryInfo(root)
            .EnumerateFiles("*", SearchOption.AllDirectories)
            .Where(file =>
                extensions.Count == 0 ||
                extensions.Contains(file.Extension))
            .ToArray();
    }
}
```

在 `Program.cs` 注册：

```csharp
var services = new ServiceCollection();
services.AddSingleton<IFileScanner, FileScanner>();

var registrar = new TypeRegistrar(services);
var app = new CommandApp(registrar);
```

Command 中构造函数注入：

```csharp
public sealed class ScanCommand : Command<ScanSettings>
{
    private readonly IFileScanner _scanner;
    private readonly IAnsiConsole _console;

    public ScanCommand(
        IFileScanner scanner,
        IAnsiConsole console)
    {
        _scanner = scanner;
        _console = console;
    }

    protected override int Execute(
        CommandContext context,
        ScanSettings settings,
        CancellationToken cancellation)
    {
        var extensions = settings.Extensions.ToHashSet(
            StringComparer.OrdinalIgnoreCase);
        var files = _scanner.Scan(settings.Root, extensions);

        _console.MarkupLine($"找到 [yellow]{files.Count}[/] 个文件。");
        return 0;
    }
}
```

这样 Command 负责协调输入、服务和输出，文件扫描服务可以单独测试，也可以替换成数据库或远程 API 实现。

## 测试 CLI 命令

`IAnsiConsole` 可以被 `TestConsole` 替换，命令不必直接操作真实终端。

安装测试包：

```bash
dotnet add package Spectre.Console.Testing
```

简单输出测试：

```csharp
using Spectre.Console.Testing;

var console = new TestConsole();
console.WriteLine("扫描完成");

string output = console.Output;
Assert.Contains("扫描完成", output);
```

命令测试的重点通常有三类：

* 参数是否正确绑定到 Settings。
* 非法参数是否被 `Validate()` 拦截。
* Execute 是否返回预期退出码，并写出预期信息。

文件系统测试可以使用临时目录，网络服务测试可以注入假的服务实现。CLI 的解析、业务和终端输出拆开后，测试不会被真实终端环境绑死。

## 和 CommandLineParser、System.CommandLine 怎么选？

| 方案 | 特点 | 更适合的场景 |
| --- | --- | --- |
| `CommandLineParser` | 属性模型成熟，解析功能直接 | 传统 .NET CLI、已有 2.x 项目 |
| `System.CommandLine` | 官方命令树模型，API 细粒度 | 复杂命令树、现代 .NET CLI |
| `Spectre.Console.Cli` | Settings + Command 约定，终端 UI 组合自然 | 需要漂亮输出、表格、进度条和 DI 的 CLI |

选择 `Spectre.Console.Cli` 的主要理由不是“解析参数更快”，而是命令类、强类型 Settings、依赖注入和终端体验能够放在同一个使用模型里。

如果应用只是一次性读取两个参数，直接使用 `args` 反而更简单；如果应用需要多个命令、自动帮助、可测试输出和长期维护，再引入框架更划算。

## 常见坑

### 把 `Spectre.Console.Cli` 当成完整终端 UI 框架

`Spectre.Console.Cli` 负责 CLI 结构，表格、进度条、提示框来自 `Spectre.Console`。需要 UI 类型时，应确保引用了对应包和命名空间。

### 外部输入直接拼接到 Markup

错误写法：

```csharp
AnsiConsole.MarkupLine($"[green]{settings.Name}[/]");
```

更安全的写法：

```csharp
AnsiConsole.MarkupLine(
    $"[green]{Markup.Escape(settings.Name)}[/]");
```

### 所有逻辑都塞进 Execute

命令类过大后，参数解析、文件操作、输出和异常处理会混在一起。服务抽离后，Command 保持短小，测试也更容易写。

### 忽略异步取消

`AsyncCommand` 已经提供取消令牌。长任务不传递令牌，会导致 Ctrl+C 后程序仍然继续执行。

### 用彩色输出判断成功失败

颜色只服务阅读体验，自动化脚本应读取退出码。成功返回 `0`，失败返回非零值。

### 误以为分支命令会执行逻辑

`AddBranch` 创建的是命令分组。实际业务逻辑应放在分支下的叶子命令中。

## 总结

`Spectre.Console.Cli` 的核心思路可以概括为：

```text
CommandSettings 描述输入
        ↓
Command<TSettings> 负责业务动作
        ↓
CommandApp 负责命令注册和调度
        ↓
Spectre.Console 负责终端呈现
```

单命令工具使用 `CommandApp<TCommand>`；多命令工具使用 `CommandApp` + `AddCommand`；命令分组使用 `AddBranch`；参数校验放在 `Validate()`；文件、数据库和网络逻辑通过 DI 注入；长任务使用 `AsyncCommand<TSettings>` 和取消令牌。

最适合 `Spectre.Console.Cli` 的项目，不是只需要解析几个参数的临时脚本，而是希望同时拥有清晰命令结构、强类型设置、可测试业务、依赖注入和良好终端体验的长期 CLI 工具。

参考资料：

* [Spectre.Console.Cli 官方文档](https://spectreconsole.net/cli/)
* [官方：定义命令和参数](https://spectreconsole.net/cli/how-to/defining-commands-and-arguments/)
* [官方：依赖注入](https://spectreconsole.net/cli/tutorials/dependency-injection-in-cli-apps/)
* [官方：命令生命周期](https://spectreconsole.net/cli/explanation/command-lifecycle-and-execution-flow/)
* [Spectre.Console.Cli NuGet 页面](https://www.nuget.org/packages/Spectre.Console.Cli)
