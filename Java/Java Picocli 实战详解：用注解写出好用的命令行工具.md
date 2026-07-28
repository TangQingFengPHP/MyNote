### 简介

`Picocli` 是 Java 生态里很常用的命令行工具开发库。

它主要解决这些问题：

```text
解析命令行参数
处理短选项和长选项
支持位置参数
自动做类型转换
生成帮助文档
生成版本信息
支持子命令
支持命令补全
支持打包成可执行 Jar
支持 GraalVM Native Image
```

一句话概括：

```text
Picocli 用注解描述命令、选项和参数，把 String[] args 解析成 Java 对象。
```

没有 Picocli 时，命令行参数通常要手动解析：

```java
public static void main(String[] args) {
    for (int i = 0; i < args.length; i++) {
        if ("--file".equals(args[i])) {
            String file = args[++i];
            System.out.println(file);
        }
    }
}
```

这种写法很快会变乱：

* `--file a.txt`
* `--file=a.txt`
* `-f a.txt`
* `-abc`
* 必填参数
* 默认值
* 参数类型转换
* 参数错误提示
* `--help`
* 子命令

Picocli 把这些通用能力封装好了。

### Picocli 适合什么场景

常见场景如下：

| 场景 | 示例 |
| --- | --- |
| 开发者工具 | 代码生成、项目初始化、接口调试 |
| 运维工具 | 发布、备份、清理、巡检 |
| 数据处理工具 | CSV 转换、日志分析、批量导入 |
| 管理脚本 | 用户管理、配置检查、任务触发 |
| Spring Boot CLI | 复用 Service 做命令行管理工具 |
| 本地小工具 | 文件重命名、图片处理、目录扫描 |

它和一些常见 CLI 工具的定位类似：

| 语言 | 常见 CLI 框架 |
| --- | --- |
| Java | Picocli |
| Go | Cobra |
| Python | argparse / Click / Typer |
| Node.js | Commander |
| .NET | System.CommandLine |

### Maven 依赖

当前 `picocli` 最新稳定版本是 `4.7.7`。

Maven：

```xml
<dependency>
    <groupId>info.picocli</groupId>
    <artifactId>picocli</artifactId>
    <version>4.7.7</version>
</dependency>
```

Gradle Kotlin DSL：

```kotlin
dependencies {
    implementation("info.picocli:picocli:4.7.7")
}
```

如果要生成 GraalVM Native Image 配置、命令补全脚本等，可以再加 `picocli-codegen`。

```xml
<dependency>
    <groupId>info.picocli</groupId>
    <artifactId>picocli-codegen</artifactId>
    <version>4.7.7</version>
    <scope>provided</scope>
</dependency>
```

### 核心注解

Picocli 最常用的注解不多。

| 注解 | 作用 |
| --- | --- |
| `@Command` | 定义一个命令 |
| `@Option` | 定义选项，比如 `-f`、`--file` |
| `@Parameters` | 定义位置参数，比如 `copy a.txt b.txt` 里的两个文件 |
| `@Mixin` | 复用一组公共选项 |
| `@ParentCommand` | 在子命令里拿到父命令对象 |
| `@Spec` | 拿到当前命令的元数据 |

一个命令类通常实现 `Runnable` 或 `Callable<Integer>`。

```text
Runnable：只执行逻辑，不关心返回值
Callable<Integer>：可以返回退出码，更适合正式 CLI
```

正式工具更推荐 `Callable<Integer>`。

退出码约定：

```text
0：成功
非 0：失败
```

### 第一个 Demo：hello 命令

先写一个最简单的命令。

目标：

```shell
java -jar hello.jar 张三
```

输出：

```text
Hello, 张三
```

代码：

```java
package com.example.picoclidemo;

import picocli.CommandLine;
import picocli.CommandLine.Command;
import picocli.CommandLine.Parameters;

import java.util.concurrent.Callable;

@Command(
        name = "hello",
        description = "打印问候语",
        mixinStandardHelpOptions = true,
        version = "hello 1.0.0"
)
public class HelloCommand implements Callable<Integer> {

    @Parameters(index = "0", description = "名称")
    private String name;

    @Override
    public Integer call() {
        System.out.println("Hello, " + name);
        return 0;
    }

    public static void main(String[] args) {
        int exitCode = new CommandLine(new HelloCommand()).execute(args);
        System.exit(exitCode);
    }
}
```

运行：

```shell
java com.example.picoclidemo.HelloCommand 张三
```

输出：

```text
Hello, 张三
```

查看帮助：

```shell
java com.example.picoclidemo.HelloCommand --help
```

输出类似：

```text
Usage: hello [-hV] <name>
打印问候语
      <name>      名称
  -h, --help      Show this help message and exit.
  -V, --version   Print version information and exit.
```

`mixinStandardHelpOptions = true` 会自动添加：

```text
-h, --help
-V, --version
```

这两个选项几乎每个正式 CLI 都应该有。

### Option 和 Parameters 的区别

命令行参数大致分两类。

### Option

带名字的参数叫选项。

比如：

```shell
tool --file app.log --limit 100 --verbose
```

对应 Picocli：

```java
@Option(names = {"-f", "--file"}, description = "文件路径")
private File file;

@Option(names = {"-l", "--limit"}, defaultValue = "100", description = "最多处理多少行")
private int limit;

@Option(names = {"-v", "--verbose"}, description = "输出详细日志")
private boolean verbose;
```

### Parameters

没有名字、靠位置识别的参数叫位置参数。

比如：

```shell
copy source.txt target.txt
```

对应 Picocli：

```java
@Parameters(index = "0", description = "源文件")
private File source;

@Parameters(index = "1", description = "目标文件")
private File target;
```

简单判断：

```text
有 - 或 -- 前缀：@Option
没有前缀，靠顺序：@Parameters
```

### 实战 Demo：文件统计工具

下面做一个稍微实用点的工具：统计文本文件。

支持功能：

```text
统计行数
统计字符数
忽略空行
限制最多读取多少行
输出详细信息
```

命令示例：

```shell
textstat README.md --chars --ignore-blank --limit 1000 -v
```

### 命令类

```java
package com.example.picoclidemo;

import picocli.CommandLine;
import picocli.CommandLine.Command;
import picocli.CommandLine.Option;
import picocli.CommandLine.Parameters;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;
import java.util.concurrent.Callable;

@Command(
        name = "textstat",
        description = "统计文本文件的行数和字符数",
        mixinStandardHelpOptions = true,
        version = "textstat 1.0.0"
)
public class TextStatCommand implements Callable<Integer> {

    @Parameters(index = "0", description = "要统计的文本文件")
    private Path file;

    @Option(names = {"-c", "--chars"}, description = "统计字符数")
    private boolean countChars;

    @Option(names = {"--ignore-blank"}, description = "忽略空行")
    private boolean ignoreBlank;

    @Option(names = {"-l", "--limit"}, defaultValue = "0", description = "最多读取多少行，0 表示不限制")
    private int limit;

    @Option(names = {"-v", "--verbose"}, description = "输出详细信息")
    private boolean verbose;

    @Override
    public Integer call() {
        if (!Files.exists(file)) {
            System.err.println("文件不存在: " + file);
            return 2;
        }

        if (!Files.isRegularFile(file)) {
            System.err.println("不是普通文件: " + file);
            return 2;
        }

        try {
            List<String> lines = Files.readAllLines(file, StandardCharsets.UTF_8);

            if (limit > 0 && lines.size() > limit) {
                lines = lines.subList(0, limit);
            }

            long lineCount = lines.stream()
                    .filter(line -> !ignoreBlank || !line.isBlank())
                    .count();

            System.out.println("lines=" + lineCount);

            if (countChars) {
                int chars = lines.stream()
                        .filter(line -> !ignoreBlank || !line.isBlank())
                        .mapToInt(String::length)
                        .sum();
                System.out.println("chars=" + chars);
            }

            if (verbose) {
                System.out.println("file=" + file.toAbsolutePath());
                System.out.println("limit=" + limit);
                System.out.println("ignoreBlank=" + ignoreBlank);
            }

            return 0;
        } catch (IOException e) {
            System.err.println("读取文件失败: " + e.getMessage());
            return 1;
        }
    }

    public static void main(String[] args) {
        int exitCode = new CommandLine(new TextStatCommand()).execute(args);
        System.exit(exitCode);
    }
}
```

运行：

```shell
java com.example.picoclidemo.TextStatCommand README.md
```

输出：

```text
lines=128
```

统计字符数：

```shell
java com.example.picoclidemo.TextStatCommand README.md --chars
```

输出：

```text
lines=128
chars=5042
```

忽略空行并输出详细信息：

```shell
java com.example.picoclidemo.TextStatCommand README.md --ignore-blank -v
```

输出：

```text
lines=96
file=/Users/test/project/README.md
limit=0
ignoreBlank=true
```

缺少文件参数：

```shell
java com.example.picoclidemo.TextStatCommand
```

Picocli 会直接提示参数缺失，并打印用法。

### 常用 Option 写法

### 必填选项

```java
@Option(names = {"-u", "--username"}, required = true, description = "用户名")
private String username;
```

没传会报错：

```text
Missing required option: '--username=<username>'
```

### 默认值

```java
@Option(names = {"-p", "--port"}, defaultValue = "8080", description = "端口，默认 ${DEFAULT-VALUE}")
private int port;
```

帮助信息里可以用：

```text
${DEFAULT-VALUE}
```

显示默认值。

### 布尔开关

```java
@Option(names = {"-v", "--verbose"}, description = "详细输出")
private boolean verbose;
```

只要命令里出现 `-v`，值就是 `true`。

```shell
tool -v
tool --verbose
```

### 多值参数

```java
@Option(names = {"-t", "--tag"}, description = "标签，可重复")
private List<String> tags;
```

运行：

```shell
tool -t java -t cli -t demo
```

结果：

```text
[java, cli, demo]
```

也可以用 `split`：

```java
@Option(names = "--tags", split = ",", description = "逗号分隔的标签")
private List<String> tags;
```

运行：

```shell
tool --tags java,cli,demo
```

### 可选值

有些选项传不传值都可以。

比如：

```shell
tool --color
tool --color=always
tool --color=never
```

写法：

```java
@Option(
        names = "--color",
        arity = "0..1",
        fallbackValue = "always",
        defaultValue = "auto",
        description = "颜色模式: auto, always, never"
)
private String color;
```

含义：

```text
不传 --color：auto
只传 --color：always
传 --color=never：never
```

### 枚举参数

```java
enum OutputFormat {
    TEXT, JSON, CSV
}

@Option(names = {"-f", "--format"}, defaultValue = "TEXT", description = "输出格式: ${COMPLETION-CANDIDATES}")
private OutputFormat format;
```

`COMPLETION-CANDIDATES` 会显示候选值。

运行：

```shell
tool --format JSON
```

### 位置参数

单个位置参数：

```java
@Parameters(index = "0", description = "输入文件")
private Path input;
```

多个位置参数：

```java
@Parameters(index = "0", description = "源文件")
private Path source;

@Parameters(index = "1", description = "目标文件")
private Path target;
```

剩余参数：

```java
@Parameters(index = "0..*", description = "文件列表")
private List<Path> files;
```

运行：

```shell
tool a.txt b.txt c.txt
```

### 类型转换

Picocli 内置了很多常见类型转换。

常见类型：

| Java 类型 | 命令行传值 |
| --- | --- |
| `String` | `hello` |
| `int` / `Integer` | `8080` |
| `long` / `Long` | `10000` |
| `boolean` | `true` / `false`，或布尔开关 |
| `File` | `./a.txt` |
| `Path` | `./a.txt` |
| `URL` | `https://example.com` |
| `URI` | `file:///tmp/a.txt` |
| `Duration` | `PT10S` |
| `LocalDate` | `2026-01-01` |
| `Enum` | `JSON` |

如果内置转换不够，可以写自定义转换器。

### 自定义 LocalDate 转换器

假设日期想支持：

```text
2026-07-18
```

转换器：

```java
package com.example.picoclidemo;

import picocli.CommandLine.ITypeConverter;

import java.time.LocalDate;
import java.time.format.DateTimeFormatter;

public class LocalDateConverter implements ITypeConverter<LocalDate> {

    @Override
    public LocalDate convert(String value) {
        return LocalDate.parse(value, DateTimeFormatter.ISO_LOCAL_DATE);
    }
}
```

使用：

```java
@Option(names = "--date", converter = LocalDateConverter.class, description = "日期，格式 yyyy-MM-dd")
private LocalDate date;
```

运行：

```shell
tool --date 2026-07-18
```

### 子命令 Demo：filecli

复杂 CLI 一般不是一个命令解决所有问题，而是类似：

```shell
git add
git commit
kubectl get pods
docker image ls
```

Picocli 支持子命令。

下面做一个 `filecli`：

```text
filecli stat README.md
filecli copy a.txt b.txt --force
filecli delete temp.log --dry-run
```

### 主命令

```java
package com.example.picoclidemo.filecli;

import picocli.CommandLine;
import picocli.CommandLine.Command;

@Command(
        name = "filecli",
        description = "文件处理命令行工具",
        mixinStandardHelpOptions = true,
        version = "filecli 1.0.0",
        subcommands = {
                StatCommand.class,
                CopyCommand.class,
                DeleteCommand.class
        }
)
public class FileCliCommand implements Runnable {

    @Override
    public void run() {
        CommandLine.usage(this, System.out);
    }

    public static void main(String[] args) {
        int exitCode = new CommandLine(new FileCliCommand()).execute(args);
        System.exit(exitCode);
    }
}
```

### stat 子命令

```java
package com.example.picoclidemo.filecli;

import picocli.CommandLine.Command;
import picocli.CommandLine.Parameters;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.concurrent.Callable;

@Command(name = "stat", description = "查看文件基本信息")
public class StatCommand implements Callable<Integer> {

    @Parameters(index = "0", description = "文件路径")
    private Path file;

    @Override
    public Integer call() throws Exception {
        if (!Files.exists(file)) {
            System.err.println("文件不存在: " + file);
            return 2;
        }

        System.out.println("path=" + file.toAbsolutePath());
        System.out.println("size=" + Files.size(file));
        System.out.println("directory=" + Files.isDirectory(file));
        System.out.println("regularFile=" + Files.isRegularFile(file));
        return 0;
    }
}
```

### copy 子命令

```java
package com.example.picoclidemo.filecli;

import picocli.CommandLine.Command;
import picocli.CommandLine.Option;
import picocli.CommandLine.Parameters;

import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardCopyOption;
import java.util.concurrent.Callable;

@Command(name = "copy", description = "复制文件")
public class CopyCommand implements Callable<Integer> {

    @Parameters(index = "0", description = "源文件")
    private Path source;

    @Parameters(index = "1", description = "目标文件")
    private Path target;

    @Option(names = {"-f", "--force"}, description = "覆盖已存在文件")
    private boolean force;

    @Override
    public Integer call() throws Exception {
        if (!Files.exists(source)) {
            System.err.println("源文件不存在: " + source);
            return 2;
        }

        if (Files.exists(target) && !force) {
            System.err.println("目标文件已存在，使用 --force 覆盖: " + target);
            return 3;
        }

        if (force) {
            Files.copy(source, target, StandardCopyOption.REPLACE_EXISTING);
        } else {
            Files.copy(source, target);
        }

        System.out.println("复制完成: " + source + " -> " + target);
        return 0;
    }
}
```

### delete 子命令

```java
package com.example.picoclidemo.filecli;

import picocli.CommandLine.Command;
import picocli.CommandLine.Option;
import picocli.CommandLine.Parameters;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.concurrent.Callable;

@Command(name = "delete", description = "删除文件")
public class DeleteCommand implements Callable<Integer> {

    @Parameters(index = "0", description = "文件路径")
    private Path file;

    @Option(names = "--dry-run", description = "只打印将要执行的操作，不真正删除")
    private boolean dryRun;

    @Override
    public Integer call() throws Exception {
        if (!Files.exists(file)) {
            System.err.println("文件不存在: " + file);
            return 2;
        }

        if (dryRun) {
            System.out.println("[dry-run] 将删除: " + file.toAbsolutePath());
            return 0;
        }

        Files.delete(file);
        System.out.println("删除完成: " + file.toAbsolutePath());
        return 0;
    }
}
```

运行：

```shell
java com.example.picoclidemo.filecli.FileCliCommand stat README.md
java com.example.picoclidemo.filecli.FileCliCommand copy a.txt b.txt --force
java com.example.picoclidemo.filecli.FileCliCommand delete temp.log --dry-run
```

查看子命令帮助：

```shell
java com.example.picoclidemo.filecli.FileCliCommand copy --help
```

输出类似：

```text
Usage: filecli copy [-f] <source> <target>
复制文件
      <source>   源文件
      <target>   目标文件
  -f, --force    覆盖已存在文件
```

### 复用公共选项：Mixin

很多子命令都会有公共参数，比如：

```text
--verbose
--profile
--config
```

可以用 `@Mixin` 复用。

公共选项：

```java
package com.example.picoclidemo.filecli;

import picocli.CommandLine.Option;

import java.nio.file.Path;

public class CommonOptions {

    @Option(names = {"-v", "--verbose"}, description = "输出详细日志")
    boolean verbose;

    @Option(names = "--config", description = "配置文件路径")
    Path config;
}
```

子命令使用：

```java
@Mixin
private CommonOptions commonOptions;
```

完整示例：

```java
@Command(name = "stat", description = "查看文件基本信息")
public class StatCommand implements Callable<Integer> {

    @Mixin
    private CommonOptions commonOptions;

    @Parameters(index = "0", description = "文件路径")
    private Path file;

    @Override
    public Integer call() throws Exception {
        if (commonOptions.verbose) {
            System.out.println("config=" + commonOptions.config);
        }
        System.out.println("path=" + file.toAbsolutePath());
        return 0;
    }
}
```

运行：

```shell
filecli stat README.md --verbose --config ./app.properties
```

### 交互式输入

密码、Token 这类敏感参数不适合直接写在命令里。

可以使用交互式输入：

```java
@Option(names = "--username", required = true, description = "用户名")
private String username;

@Option(
        names = "--password",
        interactive = true,
        arity = "0..1",
        description = "密码，不在命令行中明文显示"
)
private char[] password;
```

运行：

```shell
tool --username admin --password
```

控制台会提示输入密码。

相比：

```shell
tool --password 123456
```

交互式输入更适合敏感信息。

原因是命令行参数可能被历史记录、进程列表、审计日志记录下来。

### 参数校验

Picocli 本身会做一些基本校验：

* 必填选项
* 参数个数
* 类型转换
* 枚举值是否合法

比如：

```java
@Option(names = "--port", required = true)
private int port;
```

传入：

```shell
tool --port abc
```

会提示无法转换成整数。

更复杂的业务校验可以放到 `call()` 里。

示例：

```java
@Option(names = "--port", defaultValue = "8080", description = "端口")
private int port;

@Override
public Integer call() {
    if (port < 1 || port > 65535) {
        System.err.println("端口范围必须是 1 到 65535");
        return 2;
    }

    return 0;
}
```

也可以使用自定义转换器提前拦截非法值。

### 错误处理和退出码

`CommandLine.execute(args)` 会返回退出码。

常见做法：

```java
public static void main(String[] args) {
    int exitCode = new CommandLine(new AppCommand()).execute(args);
    System.exit(exitCode);
}
```

常见退出码可以这样约定：

| 退出码 | 含义 |
| --- | --- |
| `0` | 成功 |
| `1` | 普通执行失败 |
| `2` | 参数或输入文件问题 |
| `3` | 目标已存在、状态冲突 |

Picocli 内置常量：

```java
CommandLine.ExitCode.OK
CommandLine.ExitCode.USAGE
CommandLine.ExitCode.SOFTWARE
```

示例：

```java
return CommandLine.ExitCode.OK;
```

如果需要统一处理异常，可以配置异常处理器：

```java
CommandLine commandLine = new CommandLine(new FileCliCommand());
commandLine.setExecutionExceptionHandler((ex, cmd, parseResult) -> {
    cmd.getErr().println("执行失败: " + ex.getMessage());
    return 1;
});

int exitCode = commandLine.execute(args);
System.exit(exitCode);
```

### 帮助文档写好一点

命令行工具的帮助信息很重要。

推荐给 `@Command` 写清楚这些字段：

```java
@Command(
        name = "filecli",
        description = "文件处理命令行工具",
        synopsisHeading = "%n用法:%n  ",
        descriptionHeading = "%n说明:%n",
        optionListHeading = "%n选项:%n",
        parameterListHeading = "%n参数:%n",
        commandListHeading = "%n子命令:%n",
        mixinStandardHelpOptions = true,
        version = "filecli 1.0.0"
)
```

选项描述里可以使用变量：

```java
@Option(
        names = "--format",
        defaultValue = "TEXT",
        description = "输出格式: ${COMPLETION-CANDIDATES}，默认 ${DEFAULT-VALUE}"
)
private OutputFormat format;
```

常见变量：

| 变量 | 说明 |
| --- | --- |
| `${DEFAULT-VALUE}` | 默认值 |
| `${COMPLETION-CANDIDATES}` | 候选值 |
| `${FALLBACK-VALUE}` | 可选值参数的 fallback 值 |

### 命令补全

Picocli 可以生成命令补全脚本。

补全能力包括：

* 命令名
* 子命令
* 选项
* 枚举候选值

添加 codegen 依赖后，可以生成脚本。

常见方式是在主命令里加入 `AutoComplete.GenerateCompletion` 子命令：

```java
@Command(
        name = "filecli",
        mixinStandardHelpOptions = true,
        subcommands = {
                StatCommand.class,
                CopyCommand.class,
                DeleteCommand.class,
                picocli.AutoComplete.GenerateCompletion.class
        }
)
public class FileCliCommand implements Runnable {
    @Override
    public void run() {
        CommandLine.usage(this, System.out);
    }
}
```

生成脚本：

```shell
java -cp "target/filecli.jar" com.example.picoclidemo.filecli.FileCliCommand generate-completion
```

也可以用 `picocli.AutoComplete` 工具类按命令类生成。

不同 Shell 的安装方式不同，常见是把生成脚本放到 Bash、Zsh 或 Fish 的补全目录里。

### 打包成可执行 Jar

命令行工具通常希望这样运行：

```shell
java -jar filecli.jar stat README.md
```

可以用 Maven Shade Plugin 打包 fat jar。

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-shade-plugin</artifactId>
            <version>3.6.0</version>
            <executions>
                <execution>
                    <phase>package</phase>
                    <goals>
                        <goal>shade</goal>
                    </goals>
                    <configuration>
                        <transformers>
                            <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                                <mainClass>com.example.picoclidemo.filecli.FileCliCommand</mainClass>
                            </transformer>
                        </transformers>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

打包：

```shell
mvn clean package
```

运行：

```shell
java -jar target/filecli-1.0.0.jar stat README.md
```

为了像普通命令一样运行，可以加一个脚本。

`filecli`：

```shell
#!/usr/bin/env sh
java -jar /opt/filecli/filecli-1.0.0.jar "$@"
```

授权：

```shell
chmod +x filecli
```

运行：

```shell
./filecli stat README.md
```

### GraalVM Native Image

CLI 工具很适合打成 Native Image。

好处：

* 启动快
* 不需要目标机器安装 JVM
* 分发更像普通二进制命令

Picocli 对 GraalVM Native Image 支持比较好。

Maven Native Build Tools 示例：

```xml
<profiles>
    <profile>
        <id>native</id>
        <build>
            <plugins>
                <plugin>
                    <groupId>org.graalvm.buildtools</groupId>
                    <artifactId>native-maven-plugin</artifactId>
                    <version>1.1.2</version>
                    <extensions>true</extensions>
                    <configuration>
                        <mainClass>com.example.picoclidemo.filecli.FileCliCommand</mainClass>
                        <imageName>filecli</imageName>
                    </configuration>
                    <executions>
                        <execution>
                            <id>build-native</id>
                            <goals>
                                <goal>compile-no-fork</goal>
                            </goals>
                            <phase>package</phase>
                        </execution>
                    </executions>
                </plugin>
            </plugins>
        </build>
    </profile>
</profiles>
```

构建：

```shell
mvn -Pnative package
```

运行：

```shell
./target/filecli stat README.md
```

Native Image 构建需要安装 GraalVM 和本地 C 编译工具链。

如果项目引入了反射、动态代理、资源文件，可能还需要额外配置。纯 Picocli 小工具通常比较顺。

### Spring Boot 集成

如果 CLI 需要复用 Spring Bean，比如：

* `UserService`
* `OrderService`
* `Repository`
* 配置文件
* 数据源

可以使用 `picocli-spring-boot-starter`。

依赖：

```xml
<dependency>
    <groupId>info.picocli</groupId>
    <artifactId>picocli-spring-boot-starter</artifactId>
    <version>4.7.7</version>
</dependency>
```

命令类：

```java
package com.example.bootcli;

import org.springframework.stereotype.Component;
import picocli.CommandLine.Command;
import picocli.CommandLine.Option;

import java.util.concurrent.Callable;

@Component
@Command(name = "user", description = "用户管理命令", mixinStandardHelpOptions = true)
public class UserCommand implements Callable<Integer> {

    private final UserService userService;

    @Option(names = "--list", description = "列出用户")
    private boolean list;

    public UserCommand(UserService userService) {
        this.userService = userService;
    }

    @Override
    public Integer call() {
        if (list) {
            userService.findAll().forEach(System.out::println);
            return 0;
        }

        System.out.println("没有指定操作");
        return 2;
    }
}
```

启动类：

```java
package com.example.bootcli;

import org.springframework.boot.CommandLineRunner;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import picocli.CommandLine;
import picocli.CommandLine.IFactory;

@SpringBootApplication
public class BootCliApplication implements CommandLineRunner {

    private final UserCommand userCommand;
    private final IFactory factory;

    public BootCliApplication(UserCommand userCommand, IFactory factory) {
        this.userCommand = userCommand;
        this.factory = factory;
    }

    public static void main(String[] args) {
        SpringApplication.run(BootCliApplication.class, args);
    }

    @Override
    public void run(String... args) {
        int exitCode = new CommandLine(userCommand, factory).execute(args);
        System.exit(exitCode);
    }
}
```

运行：

```shell
java -jar boot-cli.jar --list
```

这种方式的关键点是：

```text
命令类交给 Spring 管理
CommandLine 使用 Spring 提供的 IFactory 创建对象
子命令、转换器等也可以走 Spring 注入
```

如果只是一个很小的 CLI，不一定需要 Spring Boot。Spring Boot 启动成本更高，适合需要复用完整业务容器的场景。

### 单元测试

CLI 工具也应该测试。

Picocli 测试很方便，可以直接构造命令对象，捕获输出。

示例命令：

```java
@Command(name = "hello")
class HelloCommand implements Callable<Integer> {
    @Parameters(index = "0")
    String name;

    PrintWriter out = new PrintWriter(System.out, true);

    @Override
    public Integer call() {
        out.println("Hello, " + name);
        return 0;
    }
}
```

测试：

```java
package com.example.picoclidemo;

import org.junit.jupiter.api.Test;
import picocli.CommandLine;

import java.io.PrintWriter;
import java.io.StringWriter;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

class HelloCommandTest {

    @Test
    void shouldPrintHello() {
        HelloCommand command = new HelloCommand();
        StringWriter output = new StringWriter();
        command.out = new PrintWriter(output);

        int exitCode = new CommandLine(command).execute("张三");

        assertEquals(0, exitCode);
        assertTrue(output.toString().contains("Hello, 张三"));
    }
}
```

也可以直接测试错误参数：

```java
@Test
void shouldReturnUsageExitCodeWhenMissingName() {
    int exitCode = new CommandLine(new HelloCommand()).execute();

    assertEquals(CommandLine.ExitCode.USAGE, exitCode);
}
```

### 和 Apache Commons CLI 的区别

Java 里还有一个老牌库叫 `Apache Commons CLI`。

两者大概区别：

| 对比项 | Picocli | Apache Commons CLI |
| --- | --- | --- |
| 开发方式 | 注解驱动，也支持编程式 API | 编程式 API |
| 帮助文档 | 自动生成能力强 | 相对基础 |
| 子命令 | 支持较好 | 需要自行组织 |
| 类型转换 | 内置很多类型 | 通常手动取字符串再转换 |
| GraalVM | 支持较好 | 不是主要卖点 |
| 适合场景 | 完整 CLI 工具 | 简单参数解析 |

新写命令行工具，Picocli 更省事。

如果只是解析几个参数，Commons CLI 也够用。

### 常见坑

### main 方法里没有 System.exit

错误写法：

```java
new CommandLine(new AppCommand()).execute(args);
```

脚本调用时无法正确拿到退出码。

推荐：

```java
int exitCode = new CommandLine(new AppCommand()).execute(args);
System.exit(exitCode);
```

### 还在使用 CommandLine.run 或 call

旧示例里经常看到：

```java
CommandLine.run(new AppCommand(), args);
CommandLine.call(new AppCommand(), args);
```

这些便捷方法在 Picocli 4.x 里已经不推荐。

推荐：

```java
new CommandLine(new AppCommand()).execute(args);
```

### 布尔参数写成 Boolean 但没有默认值

```java
@Option(names = "--verbose")
private Boolean verbose;
```

如果没传，值是 `null`。

大部分布尔开关用基本类型更简单：

```java
@Option(names = "--verbose")
private boolean verbose;
```

### 位置参数顺序不清楚

```java
@Parameters
private Path source;

@Parameters
private Path target;
```

多个位置参数建议明确 `index`：

```java
@Parameters(index = "0")
private Path source;

@Parameters(index = "1")
private Path target;
```

### 选项名太随意

不建议写一堆只有短选项的参数：

```java
-a -b -c -x
```

正式工具最好短选项和长选项都提供：

```java
@Option(names = {"-f", "--file"})
@Option(names = {"-o", "--output"})
@Option(names = {"-v", "--verbose"})
```

长选项更适合脚本，可读性更好。

### 密码直接从参数传入

不推荐：

```shell
tool --password 123456
```

更推荐：

```shell
tool --password
```

配合：

```java
@Option(names = "--password", interactive = true, arity = "0..1")
private char[] password;
```

### 总结

Picocli 可以按这条线理解：

```text
@Command 定义命令
@Option 定义具名选项
@Parameters 定义位置参数
CommandLine.execute 负责解析和执行
Callable<Integer> 返回退出码
subcommands 组织复杂工具
Mixin 复用公共参数
```

日常开发可以按这个顺序落地：

* 先写单命令 Demo
* 再补齐 `--help` 和 `--version`
* 参数尽量用明确类型，比如 `Path`、`int`、`Enum`
* 复杂工具拆成子命令
* 公共选项用 `@Mixin`
* 正式发布时打包成可执行 Jar
* 对启动速度敏感时考虑 Native Image
* 需要复用业务 Bean 时接入 Spring Boot

Picocli 的价值不是“把参数解析换成注解”这么简单。真正省事的地方在于帮助文档、错误提示、子命令、类型转换、退出码这些细节都已经有一套成熟约定。命令行工具越往后做，越能感受到这些约定带来的稳定性。
