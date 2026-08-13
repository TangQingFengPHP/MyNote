# 别再把命令行参数写成一团：C#.NET System.CommandLine 2.0 实战详解

控制台程序刚开始可能只有一个参数：

```bash
mytool --file app.log
```

随着功能增加，命令行往往会变成这样：

```bash
mytool config set --file appsettings.json --key Logging:LogLevel --value Debug
```

如果所有参数都靠 `args[0]`、`args[1]` 手动判断，代码很快会遇到这些问题：参数顺序难以记忆，数字和文件路径需要手动转换，错误提示不统一，帮助信息还得另外编写。

`System.CommandLine` 是 .NET 生态中的命令行应用框架，负责命令树、参数解析、类型转换、校验、帮助信息、响应文件和补全等基础工作。业务代码只需要处理已经解析好的值。

本文以 `System.CommandLine 2.0.x` 为基础，先完成一个简单的文件读取工具，再逐步扩展成带 `scan`、`copy` 子命令的文件工具。代码采用 2.0 的新 API，重点使用 `SetAction` 和 `ParseResult.GetValue`。

> 提示：网上仍然能看到 `SetHandler`、`CommandHandler.Create` 等写法，它们主要来自 2.0 beta 版本。新项目应优先参考当前 2.0.x API；旧项目升级时需要阅读官方迁移说明。

## System.CommandLine 到底解决什么问题？

命令行输入本质上是一组文本：

```text
--file app.log --count 10 --verbose
```

程序真正需要的却是强类型数据：

```csharp
FileInfo file;
int count;
bool verbose;
```

`System.CommandLine` 可以把这层转换和校验统一起来，同时根据命令定义自动生成帮助信息。

它的核心组成如下：

| 类型 | 白话解释 | 命令示例 |
| --- | --- | --- |
| `RootCommand` | 整个 CLI 程序的入口 | `mytool` |
| `Command` | 一个具体命令或命令分组 | `scan`、`config` |
| `Option<T>` | 带名称的选项 | `--file app.log` |
| `Argument<T>` | 按位置传入的参数 | `copy source.txt` |
| `ParseResult` | 解析后的结果，负责取值和错误信息 | `GetValue(option)` |
| `SetAction` | 命令匹配成功后执行的业务入口 | 扫描、复制、读取 |

结构可以理解成：

```text
用户输入
   ↓
命令树：RootCommand / Command
   ↓
Option<T> / Argument<T>
   ↓
ParseResult
   ↓
SetAction
   ↓
业务代码和退出码
```

## 安装 NuGet 包

创建控制台项目：

```bash
dotnet new console -n FileTool
cd FileTool
dotnet add package System.CommandLine
```

截至本文编写时，NuGet 上的稳定版本为 `2.0.10`。项目文件中也可以显式锁定版本：

```xml
<PackageReference Include="System.CommandLine" Version="2.0.10" />
```

版本号会随时间变化，正式项目应以 NuGet 页面为准，并通过依赖升级检查确认兼容性。文章中的 API 按 2.0.x 编写，不混用 beta4 或 beta5 的旧示例。

## 第一个 Demo：读取一个文件

先完成一个最小工具，支持下面的命令：

```bash
filetool --file ./README.md
```

### 定义 Option

`Option<FileInfo>` 表示 `--file` 后面必须是一个可以转换成 `FileInfo` 的值：

```csharp
using System.CommandLine;

Option<FileInfo> fileOption = new("--file")
{
    Description = "要读取的文件路径",
    Required = true
};
```

类型写成 `FileInfo` 后，业务层不必再手动 `new FileInfo(path)`。需要注意，`FileInfo` 能表示一个路径，并不代表文件一定存在；文件存在性仍然属于业务校验或自定义解析逻辑。

### 创建根命令并绑定动作

`Program.cs`：

```csharp
using System.CommandLine;

Option<FileInfo> fileOption = new("--file")
{
    Description = "要读取的文件路径",
    Required = true
};

RootCommand rootCommand = new("读取文本文件的命令行工具");
rootCommand.Options.Add(fileOption);

rootCommand.SetAction(parseResult =>
{
    FileInfo file = parseResult.GetValue(fileOption)!;

    if (!file.Exists)
    {
        Console.Error.WriteLine($"文件不存在：{file.FullName}");
        return 2;
    }

    foreach (string line in File.ReadLines(file.FullName))
    {
        Console.WriteLine(line);
    }

    return 0;
});

return rootCommand.Parse(args).Invoke();
```

运行：

```bash
dotnet run -- --file ./README.md
```

代码可以拆成四个动作：

1. 创建 `Option<FileInfo>`。
2. 将选项添加到根命令。
3. 用 `SetAction` 定义命令成功后的处理逻辑。
4. 调用 `Parse(args)` 解析，再调用 `Invoke()` 执行。

`Invoke()` 返回 `int` 退出码。约定 `0` 表示成功，非 `0` 表示失败，脚本、流水线和操作系统都可以根据这个值判断结果。

## `Option<T>`：命名选项

命名选项一般长这样：

```bash
mytool --port 8080
```

对应代码：

```csharp
Option<int> portOption = new("--port")
{
    Description = "监听端口",
    DefaultValueFactory = _ => 8080
};
```

读取选项值：

```csharp
int port = parseResult.GetValue(portOption);
```

### 添加短名称和别名

2.0 API 中，名称和别名可以分别添加：

```csharp
Option<string> outputOption = new("--output")
{
    Description = "输出文件路径"
};

outputOption.Aliases.Add("-o");
outputOption.Aliases.Add("--out");
```

下面三种写法都指向同一个选项：

```bash
mytool --output result.json
mytool --out result.json
mytool -o result.json
```

长名称表达语义，短名称适合频繁使用。别名不要堆得太多，否则帮助信息和使用习惯都会变复杂。

### `bool` 开关与默认值

```csharp
Option<bool> verboseOption = new("--verbose")
{
    Description = "输出详细日志"
};

Option<int> retryOption = new("--retry")
{
    Description = "失败后的重试次数",
    DefaultValueFactory = _ => 3
};
```

调用 `--verbose` 后值为 `true`，没有出现时为 `false`。一般不写成 `--verbose true`，因为后面的 `true` 可能被当成额外 token。

读取默认值：

```csharp
bool verbose = parseResult.GetValue(verboseOption);
int retry = parseResult.GetValue(retryOption);
```

默认值也可以依赖当前解析结果：

```csharp
Option<string> formatOption = new("--format")
{
    Description = "输出格式",
    DefaultValueFactory = parseResult =>
        parseResult.GetValue(verboseOption) ? "json" : "text"
};
```

默认值之间存在依赖时，建议写清楚依赖关系，避免一个默认值偷偷受另一个选项影响。

## `Argument<T>`：位置参数

位置参数不带 `--name`，而是按照命令定义的顺序传入：

```bash
mytool copy source.txt backup/source.txt
```

定义方式：

```csharp
Argument<FileInfo> sourceArgument = new("source")
{
    Description = "源文件"
};

Argument<FileInfo> targetArgument = new("target")
{
    Description = "目标文件"
};

Command copyCommand = new("copy", "复制文件")
{
    Arguments = { sourceArgument, targetArgument }
};
```

读取：

```csharp
FileInfo source = parseResult.GetValue(sourceArgument)!;
FileInfo target = parseResult.GetValue(targetArgument)!;
```

位置参数适合文件路径、任务 ID、资源名称等命令核心输入。选项很多时，关键配置更适合使用 `--source`、`--target`，否则参数顺序很容易记错。

## 子命令：从单功能工具升级为 CLI

只做一件事的工具可以把选项放在根命令上；功能增加后，推荐按照用户动作拆成子命令：

```text
filetool
├── scan
└── copy
```

对应调用：

```bash
filetool scan --root ./logs --extension .log
filetool copy source.txt backup/source.txt --overwrite
```

### 创建子命令和全局选项

```csharp
Option<bool> verboseOption = new("--verbose")
{
    Description = "输出详细信息",
    Recursive = true
};

RootCommand rootCommand = new("文件处理工具");
rootCommand.Options.Add(verboseOption);

Command scanCommand = new("scan", "扫描目录并统计文件数量");
Command copyCommand = new("copy", "复制文件");

rootCommand.Subcommands.Add(scanCommand);
rootCommand.Subcommands.Add(copyCommand);
```

`Recursive = true` 让根命令选项对子命令递归可用。`--verbose`、`--config`、`--no-color` 这类确实适用于所有命令的配置适合做全局选项；只服务某个命令的参数应放在对应子命令上。

## 完整 Demo：文件扫描与复制工具

下面的完整程序包含：

* `scan`：递归扫描目录
* `copy`：复制文件
* `Option<T>`、`Argument<T>`、枚举和多值参数
* 全局 `--verbose` 选项
* 文件存在性校验和退出码

### 创建项目

```bash
dotnet new console -n FileTool
cd FileTool
dotnet add package System.CommandLine --version 2.0.10
```

### 完整 `Program.cs`

```csharp
using System.CommandLine;

enum OutputFormat
{
    Text,
    Json
}

Option<bool> verboseOption = new("--verbose")
{
    Description = "输出详细信息",
    Recursive = true
};
verboseOption.Aliases.Add("-v");

Option<string[]> extensionOption = new("--extension")
{
    Description = "只匹配指定扩展名，例如 log txt",
    AllowMultipleArgumentsPerToken = true
};
extensionOption.Aliases.Add("-e");

Option<OutputFormat> formatOption = new("--format")
{
    Description = "统计结果格式",
    DefaultValueFactory = _ => OutputFormat.Text
};

Option<bool> overwriteOption = new("--overwrite")
{
    Description = "目标文件存在时覆盖"
};

Argument<DirectoryInfo> rootArgument = new("root")
{
    Description = "要扫描的目录"
};

Argument<FileInfo> sourceArgument = new("source")
{
    Description = "源文件"
};

Argument<FileInfo> targetArgument = new("target")
{
    Description = "目标文件"
};

RootCommand rootCommand = new("一个简单的文件处理命令行工具");
rootCommand.Options.Add(verboseOption);

Command scanCommand = new("scan", "递归扫描目录中的文件");
scanCommand.Arguments.Add(rootArgument);
scanCommand.Options.Add(extensionOption);
scanCommand.Options.Add(formatOption);
rootCommand.Subcommands.Add(scanCommand);

Command copyCommand = new("copy", "复制一个文件");
copyCommand.Arguments.Add(sourceArgument);
copyCommand.Arguments.Add(targetArgument);
copyCommand.Options.Add(overwriteOption);
rootCommand.Subcommands.Add(copyCommand);

scanCommand.SetAction(parseResult =>
{
    DirectoryInfo root = parseResult.GetValue(rootArgument)!;
    string[] extensions = parseResult.GetValue(extensionOption) ?? Array.Empty<string>();
    OutputFormat format = parseResult.GetValue(formatOption);
    bool verbose = parseResult.GetValue(verboseOption);

    if (!root.Exists)
    {
        Console.Error.WriteLine($"目录不存在：{root.FullName}");
        return 2;
    }

    HashSet<string> filters = extensions
        .Select(NormalizeExtension)
        .ToHashSet(StringComparer.OrdinalIgnoreCase);

    List<FileInfo> files = root
        .EnumerateFiles("*", SearchOption.AllDirectories)
        .Where(file => filters.Count == 0 || filters.Contains(file.Extension))
        .ToList();

    if (verbose)
    {
        foreach (FileInfo file in files)
        {
            Console.WriteLine(file.FullName);
        }
    }

    if (format == OutputFormat.Json)
    {
        Console.WriteLine($"{{\"count\":{files.Count}}}");
    }
    else
    {
        Console.WriteLine($"共找到 {files.Count} 个文件。");
    }

    return 0;
});

copyCommand.SetAction(parseResult =>
{
    FileInfo source = parseResult.GetValue(sourceArgument)!;
    FileInfo target = parseResult.GetValue(targetArgument)!;
    bool overwrite = parseResult.GetValue(overwriteOption);

    if (!source.Exists)
    {
        Console.Error.WriteLine($"源文件不存在：{source.FullName}");
        return 2;
    }

    if (target.Exists && !overwrite)
    {
        Console.Error.WriteLine("目标文件已存在，请添加 --overwrite。");
        return 3;
    }

    target.Directory?.Create();
    source.CopyTo(target.FullName, overwrite);
    Console.WriteLine($"已复制：{source.FullName} -> {target.FullName}");
    return 0;
});

return rootCommand.Parse(args).Invoke();

static string NormalizeExtension(string extension)
{
    return extension.StartsWith('.') ? extension : $".{extension}";
}
```

### 运行 `scan`

扫描全部文件：

```bash
dotnet run -- scan ./logs
```

只扫描日志和文本文件：

```bash
dotnet run -- scan ./logs --extension log txt
```

开启详细输出并生成 JSON 统计：

```bash
dotnet run -- scan ./logs --extension log --format Json --verbose
```

### 运行 `copy`

```bash
dotnet run -- copy ./logs/app.log ./backup/app.log
```

目标文件存在时，程序拒绝覆盖并返回退出码 `3`：

```bash
dotnet run -- copy ./logs/app.log ./backup/app.log --overwrite
```

### 查看帮助

```bash
dotnet run -- --help
dotnet run -- scan --help
dotnet run -- copy --help
```

根命令默认提供帮助和版本相关能力。命令、选项、参数的 `Description` 写得越清楚，自动生成的帮助信息越有价值。

## 类型转换与自定义校验

框架支持很多常见类型：

```csharp
Option<int> countOption = new("--count");
Option<bool> enabledOption = new("--enabled");
Option<DateTime> dateOption = new("--date");
Option<FileInfo> fileOption = new("--file");
Option<ConsoleColor> colorOption = new("--color");
Option<OutputFormat> formatOption = new("--format");
Option<string[]> tagsOption = new("--tag");
```

类型转换只能判断“能不能转成整数”，不能判断“这个整数是否适合业务”。例如重试次数不能为负数，端口必须在 `1` 到 `65535` 之间。

2.0 可以使用 `CustomParser` 在解析阶段完成参数校验：

```csharp
Option<int> retryOption = new("--retry")
{
    Description = "重试次数",
    CustomParser = result =>
    {
        if (!int.TryParse(result.Tokens.Single().Value, out int value))
        {
            result.AddError("重试次数必须是整数。");
            return 0;
        }

        if (value < 0 || value > 10)
        {
            result.AddError("重试次数必须在 0 到 10 之间。");
        }

        return value;
    }
};
```

另一种方式是使用 `Validators`：

```csharp
Option<int> portOption = new("--port")
{
    Description = "监听端口"
};

portOption.Validators.Add(result =>
{
    int port = result.GetValue(portOption);
    if (port is < 1 or > 65535)
    {
        result.AddError("端口必须在 1 到 65535 之间。");
    }
});
```

建议按规则分层：单个参数的格式检查放在解析或验证中；文件是否存在、两个路径是否冲突、多个参数能否组合放在命令动作中。这样错误信息更准确，也不会把所有业务逻辑塞进参数定义。

## `Parse`、`ParseResult` 和 `Invoke`

这三个概念可以分成两步：

```csharp
ParseResult parseResult = rootCommand.Parse(args);
return parseResult.Invoke();
```

`Parse(args)` 只负责把文本解析成结果，适合单元测试和需要先检查结果的场景：

```csharp
ParseResult result = rootCommand.Parse(new[] { "scan", "./logs" });

if (result.Errors.Count > 0)
{
    foreach (ParseError error in result.Errors)
    {
        Console.Error.WriteLine(error.Message);
    }
}
else
{
    DirectoryInfo root = result.GetValue(rootArgument)!;
}
```

`Invoke()` 才会执行 `SetAction` 注册的动作，并返回退出码。异步业务可以使用异步动作和 `InvokeAsync`：

```csharp
scanCommand.SetAction(async (parseResult, cancellationToken) =>
{
    DirectoryInfo root = parseResult.GetValue(rootArgument)!;
    await ScanAsync(root, cancellationToken);
    return 0;
});

return await rootCommand.Parse(args).InvokeAsync();
```

异步操作应继续传递 `CancellationToken`，这样用户按下 Ctrl+C 时，文件扫描、网络请求或数据库操作才有机会及时结束。

## 响应文件：参数太长时换成文件

当命令包含几十个参数，可以使用响应文件。创建 `scan.rsp`：

```text
scan
./logs
--extension
log
txt
--format
Json
```

执行：

```bash
filetool @scan.rsp
```

响应文件默认启用，适合构建脚本、批量任务和自动生成命令的场景。响应文件可能继续引用其他响应文件，因此文件来源必须可信；不可信输入环境可以关闭响应文件替换功能。

## Tab 补全

`System.CommandLine` 支持子命令、选项名称、枚举值和固定值的 Tab 补全，也支持运行时动态生成补全项。

自定义 CLI 通常需要安装 `dotnet-suggest` 并注册应用：

```bash
dotnet tool install -g dotnet-suggest
dotnet-suggest register --command-path /absolute/path/to/filetool
```

不同 shell 还需要加载对应的 shim：Bash 加入 profile，Zsh 加入 `.zshrc`，PowerShell 加入 PowerShell profile。传统 `cmd.exe` 没有可插拔的 Tab 补全机制，无法使用同样的 shim 方案。

## 测试命令行代码

命令行项目不应只靠手工输入测试。`Parse` 与 `Invoke` 分离后，可以直接检查解析结果：

```csharp
[Fact]
public void Scan_should_parse_root_directory()
{
    RootCommand command = BuildCommand();

    ParseResult result = command.Parse(new[] { "scan", "./logs" });

    Assert.Empty(result.Errors);
    Assert.Equal("./logs", result.GetValue(rootArgument)!.ToString());
}
```

还可以通过 `InvocationConfiguration` 注入输出和错误流，避免测试直接依赖 `Console.WriteLine`。业务动作单独放在方法或服务中后，参数解析测试和文件处理测试可以分别进行。

## 2.0 API 与旧版示例的区别

网上大量文章基于 `2.0.0-beta4` 或更早版本，常见写法包括：

```csharp
command.SetHandler(...);
CommandHandler.Create(...);
rootCommand.AddCommand(command);
```

2.0.x 文档中更推荐：

```csharp
command.SetAction(parseResult =>
{
    int count = parseResult.GetValue(countOption);
    return 0;
});

rootCommand.Subcommands.Add(command);
return rootCommand.Parse(args).Invoke();
```

升级旧项目时，重点检查：

* `SetHandler` 是否改成 `SetAction`。
* `GetValueForOption` 是否改成 `ParseResult.GetValue`。
* `AddCommand` 是否改成 `Subcommands.Add`。
* 默认值工厂是否使用 `DefaultValueFactory`。
* 同步、异步动作和调用方式是否匹配。
* 自定义帮助、补全和解析器配置是否仍然存在。

## 常见问题

### 根命令没有设置动作，直接运行为什么只显示帮助？

根命令包含子命令时，通常要求继续输入一个叶子命令：

```bash
filetool scan ./logs
```

只输入 `filetool` 时，框架会提示需要提供命令，并显示可用的子命令。分组命令一般不绑定动作，真正执行的逻辑绑定在叶子命令上。

### `FileInfo` 为什么没有自动报“文件不存在”？

`FileInfo` 是路径对象，不是文件内容，也不会自动保证目标存在。文件存在性属于业务规则，可以在动作中检查 `Exists`，或者通过 `CustomParser` 在解析阶段添加错误。

### 为什么不直接使用 `args`？

简单的一次性脚本当然可以直接读取 `args`。当程序需要必填校验、默认值、多个子命令、自动帮助、补全、响应文件或可靠退出码时，手写解析逻辑会重复实现大量基础设施，维护成本也会逐步上升。

## 什么时候适合使用 System.CommandLine？

适合的场景：

* .NET Global Tool 和 Local Tool
* 数据迁移、代码生成、文件转换工具
* CI/CD、部署和批处理程序
* 需要多级子命令的开发者工具
* 需要自动帮助、补全和响应文件的 CLI
* 希望兼容 Native AOT、裁剪和轻量发布的控制台应用

如果重点是漂亮的终端表格、进度条、颜色和交互式界面，`Spectre.Console` 更适合负责呈现；`System.CommandLine` 负责命令解析，两者也可以组合使用。

## 总结

`System.CommandLine` 的核心用法可以压缩成四步：

```text
创建 RootCommand
    ↓
添加 Command、Option<T>、Argument<T>
    ↓
使用 SetAction 绑定业务动作
    ↓
Parse(args).Invoke() 返回退出码
```

单功能工具可以从一个根命令开始；功能增加后，用子命令分组；共享配置放全局选项；业务动作中负责文件存在性和参数组合校验；需要测试时，单独调用 `Parse` 检查结果。

真正值得注意的是版本：`System.CommandLine 2.0.x` 的新 API 与旧 beta 文章存在明显差异，`SetAction`、`Subcommands`、`DefaultValueFactory` 和 `ParseResult.GetValue` 应作为新项目的优先写法。

参考资料：

* [Microsoft Learn：System.CommandLine 概览](https://learn.microsoft.com/en-us/dotnet/standard/commandline/)
* [Microsoft Learn：System.CommandLine 入门教程](https://learn.microsoft.com/en-us/dotnet/standard/commandline/get-started-tutorial)
* [Microsoft Learn：命令行语法概览](https://learn.microsoft.com/en-us/dotnet/standard/commandline/syntax)
* [Microsoft Learn：启用 Tab 补全](https://learn.microsoft.com/en-us/dotnet/standard/commandline/how-to-enable-tab-completion)
* [System.CommandLine NuGet 页面](https://www.nuget.org/packages/System.CommandLine)
