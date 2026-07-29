### 简介

`JCommander` 是一个 Java 命令行参数解析库。

它主要解决这些问题：

```text
解析 String[] args
把命令行参数绑定到 Java 对象
自动做基础类型转换
检查必填参数
生成帮助信息
支持子命令
支持动态参数
支持自定义参数校验和类型转换
```

一句话概括：

```text
JCommander 用注解描述命令行参数，把一堆字符串参数解析成结构清楚的 Java 对象。
```

没有命令行解析库时，经常会写出这种代码：

```java
public static void main(String[] args) {
    String input = null;
    boolean verbose = false;

    for (int i = 0; i < args.length; i++) {
        if ("--input".equals(args[i])) {
            input = args[++i];
        } else if ("--verbose".equals(args[i])) {
            verbose = true;
        }
    }

    System.out.println(input);
    System.out.println(verbose);
}
```

参数一多，代码很快变乱：

* 必填参数要手动判断
* 数字、文件、枚举要手动转换
* 错误提示要手动写
* `--help` 要手动生成
* 子命令要自己拆分
* 参数格式不统一时更难维护

JCommander 的目标就是让这些通用逻辑少写一点。

### JCommander 适合什么场景

常见场景如下：

| 场景 | 示例 |
| --- | --- |
| 简单 CLI 工具 | 文件处理、日志扫描、代码生成 |
| 运维脚本 | 备份、清理、发布、健康检查 |
| 数据处理程序 | CSV 导入、JSON 转换、批量处理 |
| 测试工具 | 压测参数、测试环境开关 |
| Jar 启动参数 | 非 Spring Boot 小程序启动配置 |

它更适合“参数解析”这个范围。

如果要做复杂命令行应用，比如：

* 多层子命令
* 自动补全
* 彩色帮助
* Native Image 支持
* 更丰富的帮助文档排版

`Picocli` 通常更现代、更完整。

如果只是想把 `args` 解析成对象，JCommander 依然很轻量。

### JCommander 和 Picocli 的区别

| 对比项 | JCommander | Picocli |
| --- | --- | --- |
| 定位 | 命令行参数解析库 | 更完整的 CLI 框架 |
| 注解驱动 | 支持 | 支持 |
| 基础参数解析 | 简单直接 | 很强 |
| 子命令 | 支持 | 更强 |
| 帮助文档 | 基础可用 | 更丰富 |
| 命令补全 | 不是重点 | 支持较好 |
| 生态活跃度 | 相对稳定 | 更活跃 |
| 适合场景 | 简单工具、老项目维护 | 新 CLI、复杂工具 |

简单理解：

```text
JCommander 更像轻量参数解析器。
Picocli 更像完整命令行应用框架。
```

### 版本和依赖

JCommander 早期常用坐标是：

```xml
<dependency>
    <groupId>com.beust</groupId>
    <artifactId>jcommander</artifactId>
    <version>1.82</version>
</dependency>
```

很多老文章和老项目都还在用这个版本。

新的 Maven 坐标已经迁移到：

```xml
<dependency>
    <groupId>org.jcommander</groupId>
    <artifactId>jcommander</artifactId>
    <version>3.0</version>
</dependency>
```

注意一个容易疑惑的点：

```text
Maven groupId 变成 org.jcommander
Java import 包名仍然是 com.beust.jcommander
```

所以代码里还是这样导入：

```java
import com.beust.jcommander.JCommander;
import com.beust.jcommander.Parameter;
```

Gradle Kotlin DSL：

```kotlin
dependencies {
    implementation("org.jcommander:jcommander:3.0")
}
```

如果项目已经锁在老版本，也可以继续使用 `com.beust:jcommander:1.82`。

新项目更建议先看 `org.jcommander:jcommander:3.0`。

### 核心注解

JCommander 常用注解不多。

| 注解 | 作用 |
| --- | --- |
| `@Parameter` | 定义普通参数、位置参数、帮助参数 |
| `@Parameters` | 定义命令或子命令描述 |
| `@DynamicParameter` | 接收动态 key-value 参数，比如 `-Denv=prod` |
| `@ParametersDelegate` | 复用一组公共参数 |

最核心的是 `@Parameter`。

常见属性：

| 属性 | 作用 |
| --- | --- |
| `names` | 参数名，比如 `-f`、`--file` |
| `description` | 帮助信息描述 |
| `required` | 是否必填 |
| `help` | 是否帮助参数 |
| `password` | 是否密码输入 |
| `variableArity` | 是否可变数量参数 |
| `arity` | 参数值个数 |
| `validateWith` | 参数校验器 |
| `converter` | 类型转换器 |
| `order` | 帮助信息排序 |

### 第一个 Demo：hello 命令

目标：

```shell
java -jar hello.jar --name 张三 --times 2
```

输出：

```text
Hello, 张三
Hello, 张三
```

参数类：

```java
package com.example.jcommanderdemo;

import com.beust.jcommander.Parameter;

public class HelloArgs {

    @Parameter(names = {"-n", "--name"}, description = "名称", required = true)
    private String name;

    @Parameter(names = {"-t", "--times"}, description = "打印次数")
    private int times = 1;

    @Parameter(names = {"-h", "--help"}, help = true, description = "显示帮助")
    private boolean help;

    public String getName() {
        return name;
    }

    public int getTimes() {
        return times;
    }

    public boolean isHelp() {
        return help;
    }
}
```

启动类：

```java
package com.example.jcommanderdemo;

import com.beust.jcommander.JCommander;
import com.beust.jcommander.ParameterException;

public class HelloMain {

    public static void main(String[] args) {
        HelloArgs helloArgs = new HelloArgs();

        JCommander commander = JCommander.newBuilder()
                .addObject(helloArgs)
                .programName("hello")
                .build();

        try {
            commander.parse(args);
        } catch (ParameterException e) {
            System.err.println("参数错误: " + e.getMessage());
            commander.usage();
            System.exit(2);
            return;
        }

        if (helloArgs.isHelp()) {
            commander.usage();
            return;
        }

        for (int i = 0; i < helloArgs.getTimes(); i++) {
            System.out.println("Hello, " + helloArgs.getName());
        }
    }
}
```

运行：

```shell
java com.example.jcommanderdemo.HelloMain --name 张三 --times 2
```

输出：

```text
Hello, 张三
Hello, 张三
```

查看帮助：

```shell
java com.example.jcommanderdemo.HelloMain --help
```

缺少必填参数：

```shell
java com.example.jcommanderdemo.HelloMain --times 2
```

输出类似：

```text
参数错误: The following option is required: [--name | -n]
```

### 常用参数类型

JCommander 会自动做一些基础类型转换。

```java
import com.beust.jcommander.Parameter;

import java.io.File;
import java.nio.file.Path;
import java.util.List;

public class CommonArgs {

    @Parameter(names = {"-p", "--port"}, description = "端口")
    private int port = 8080;

    @Parameter(names = {"-v", "--verbose"}, description = "详细日志")
    private boolean verbose;

    @Parameter(names = {"-f", "--file"}, description = "输入文件")
    private File file;

    @Parameter(names = "--path", description = "路径")
    private Path path;

    @Parameter(names = "--level", description = "日志级别")
    private LogLevel level = LogLevel.INFO;

    @Parameter(names = "--tag", description = "标签，可传多个")
    private List<String> tags;

    @Parameter(description = "位置参数")
    private List<String> arguments;

    enum LogLevel {
        DEBUG, INFO, WARN, ERROR
    }
}
```

示例命令：

```shell
tool --port 9090 --verbose --file app.log --level DEBUG --tag java --tag cli input.txt output.txt
```

说明：

| 字段 | 命令行 |
| --- | --- |
| `port` | `--port 9090` |
| `verbose` | `--verbose` |
| `file` | `--file app.log` |
| `level` | `--level DEBUG` |
| `tags` | `--tag java --tag cli` |
| `arguments` | `input.txt output.txt` |

### Option 和位置参数

带 `names` 的参数是具名参数：

```java
@Parameter(names = {"-i", "--input"})
private File input;
```

命令：

```shell
tool --input app.log
```

不带 `names` 的参数是位置参数：

```java
@Parameter(description = "文件列表")
private List<String> files;
```

命令：

```shell
tool a.txt b.txt c.txt
```

JCommander 里没有专门的 `@Parameters(index = "0")` 这种写法。

如果需要更精细的位置参数控制，Picocli 会更舒服。

### 实战 Demo：文件统计工具

下面做一个文件统计工具。

功能：

```text
统计文件行数
统计字符数
支持忽略空行
支持限制最多读取行数
支持多个输入文件
支持 verbose 输出
```

命令示例：

```shell
filestat --chars --ignore-blank --limit 1000 README.md app.log
```

参数类：

```java
package com.example.jcommanderdemo.filestat;

import com.beust.jcommander.Parameter;

import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;

public class FileStatArgs {

    @Parameter(names = {"-c", "--chars"}, description = "统计字符数")
    private boolean chars;

    @Parameter(names = "--ignore-blank", description = "忽略空行")
    private boolean ignoreBlank;

    @Parameter(names = {"-l", "--limit"}, description = "最多读取多少行，0 表示不限制")
    private int limit = 0;

    @Parameter(names = {"-v", "--verbose"}, description = "输出详细信息")
    private boolean verbose;

    @Parameter(names = {"-h", "--help"}, help = true, description = "显示帮助")
    private boolean help;

    @Parameter(description = "输入文件")
    private List<Path> files = new ArrayList<>();

    public boolean isChars() {
        return chars;
    }

    public boolean isIgnoreBlank() {
        return ignoreBlank;
    }

    public int getLimit() {
        return limit;
    }

    public boolean isVerbose() {
        return verbose;
    }

    public boolean isHelp() {
        return help;
    }

    public List<Path> getFiles() {
        return files;
    }
}
```

业务类：

```java
package com.example.jcommanderdemo.filestat;

import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.util.List;

public class FileStatService {

    public int run(FileStatArgs args) {
        if (args.getFiles().isEmpty()) {
            System.err.println("至少需要一个输入文件");
            return 2;
        }

        for (Path file : args.getFiles()) {
            int exitCode = printStat(file, args);
            if (exitCode != 0) {
                return exitCode;
            }
        }

        return 0;
    }

    private int printStat(Path file, FileStatArgs args) {
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

            if (args.getLimit() > 0 && lines.size() > args.getLimit()) {
                lines = lines.subList(0, args.getLimit());
            }

            List<String> effectiveLines = lines.stream()
                    .filter(line -> !args.isIgnoreBlank() || !line.isBlank())
                    .toList();

            System.out.println("file=" + file);
            System.out.println("lines=" + effectiveLines.size());

            if (args.isChars()) {
                int chars = effectiveLines.stream()
                        .mapToInt(String::length)
                        .sum();
                System.out.println("chars=" + chars);
            }

            if (args.isVerbose()) {
                System.out.println("absolutePath=" + file.toAbsolutePath());
                System.out.println("limit=" + args.getLimit());
                System.out.println("ignoreBlank=" + args.isIgnoreBlank());
            }

            return 0;
        } catch (IOException e) {
            System.err.println("读取文件失败: " + e.getMessage());
            return 1;
        }
    }
}
```

启动类：

```java
package com.example.jcommanderdemo.filestat;

import com.beust.jcommander.JCommander;
import com.beust.jcommander.ParameterException;

public class FileStatMain {

    public static void main(String[] args) {
        FileStatArgs fileStatArgs = new FileStatArgs();

        JCommander commander = JCommander.newBuilder()
                .addObject(fileStatArgs)
                .programName("filestat")
                .build();

        try {
            commander.parse(args);
        } catch (ParameterException e) {
            System.err.println("参数错误: " + e.getMessage());
            commander.usage();
            System.exit(2);
            return;
        }

        if (fileStatArgs.isHelp()) {
            commander.usage();
            return;
        }

        int exitCode = new FileStatService().run(fileStatArgs);
        System.exit(exitCode);
    }
}
```

运行：

```shell
java com.example.jcommanderdemo.filestat.FileStatMain README.md
```

输出：

```text
file=README.md
lines=128
```

统计字符：

```shell
java com.example.jcommanderdemo.filestat.FileStatMain --chars README.md
```

忽略空行：

```shell
java com.example.jcommanderdemo.filestat.FileStatMain --ignore-blank README.md
```

多个文件：

```shell
java com.example.jcommanderdemo.filestat.FileStatMain --chars README.md app.log
```

### 参数校验

`limit` 不能小于 0。

可以写一个校验器。

```java
package com.example.jcommanderdemo.filestat;

import com.beust.jcommander.IParameterValidator;
import com.beust.jcommander.ParameterException;

public class NonNegativeIntegerValidator implements IParameterValidator {

    @Override
    public void validate(String name, String value) {
        int number;
        try {
            number = Integer.parseInt(value);
        } catch (NumberFormatException e) {
            throw new ParameterException(name + " 必须是整数");
        }

        if (number < 0) {
            throw new ParameterException(name + " 不能小于 0");
        }
    }
}
```

使用：

```java
@Parameter(
        names = {"-l", "--limit"},
        description = "最多读取多少行，0 表示不限制",
        validateWith = NonNegativeIntegerValidator.class
)
private int limit = 0;
```

运行：

```shell
filestat --limit -1 README.md
```

会报参数错误。

### 自定义类型转换器

JCommander 内置支持不少常见类型。

如果需要自定义格式，可以实现 `IStringConverter<T>`。

例如日期参数：

```java
package com.example.jcommanderdemo;

import com.beust.jcommander.IStringConverter;

import java.time.LocalDate;
import java.time.format.DateTimeFormatter;

public class LocalDateConverter implements IStringConverter<LocalDate> {

    @Override
    public LocalDate convert(String value) {
        return LocalDate.parse(value, DateTimeFormatter.ISO_LOCAL_DATE);
    }
}
```

使用：

```java
@Parameter(names = "--date", converter = LocalDateConverter.class, description = "日期，格式 yyyy-MM-dd")
private LocalDate date;
```

运行：

```shell
tool --date 2026-07-29
```

### 动态参数 DynamicParameter

有些参数名不固定。

比如：

```shell
task -Denv=prod -Dregion=shanghai -Dtimeout=3000
```

可以用 `@DynamicParameter`。

```java
package com.example.jcommanderdemo;

import com.beust.jcommander.DynamicParameter;
import com.beust.jcommander.Parameter;

import java.util.HashMap;
import java.util.Map;

public class TaskArgs {

    @Parameter(names = "--name", required = true, description = "任务名称")
    private String name;

    @DynamicParameter(names = "-D", description = "动态配置，格式 -Dkey=value")
    private Map<String, String> properties = new HashMap<>();

    public String getName() {
        return name;
    }

    public Map<String, String> getProperties() {
        return properties;
    }
}
```

运行：

```shell
task --name sync-user -Denv=prod -Dregion=shanghai
```

解析结果：

```text
name=sync-user
properties={env=prod, region=shanghai}
```

### 参数委托 ParametersDelegate

多个命令共享一组参数时，可以用 `@ParametersDelegate`。

公共参数：

```java
package com.example.jcommanderdemo;

import com.beust.jcommander.Parameter;

public class CommonOptions {

    @Parameter(names = {"-v", "--verbose"}, description = "详细输出")
    private boolean verbose;

    @Parameter(names = "--profile", description = "运行环境")
    private String profile = "default";

    public boolean isVerbose() {
        return verbose;
    }

    public String getProfile() {
        return profile;
    }
}
```

命令参数：

```java
package com.example.jcommanderdemo;

import com.beust.jcommander.Parameter;
import com.beust.jcommander.ParametersDelegate;

public class ImportArgs {

    @ParametersDelegate
    private CommonOptions commonOptions = new CommonOptions();

    @Parameter(names = "--file", required = true, description = "导入文件")
    private String file;

    public CommonOptions getCommonOptions() {
        return commonOptions;
    }

    public String getFile() {
        return file;
    }
}
```

运行：

```shell
import --file users.csv --profile prod --verbose
```

说明：

```text
公共参数放在 CommonOptions
具体参数放在 ImportArgs
JCommander 会一起解析
```

注意注解名是：

```java
@ParametersDelegate
```

不是：

```java
@ParameterDelegate
```

老资料里经常把这个名字写错。

### 密码参数

密码不适合直接写在命令行里。

JCommander 支持 `password = true`。

```java
@Parameter(names = "--password", password = true, description = "密码")
private String password;
```

运行：

```shell
tool --password
```

JCommander 会从控制台读取密码。

命令行参数可能被 shell 历史、进程列表、审计日志记录。

所以不推荐：

```shell
tool --password 123456
```

### 参数文件

JCommander 支持用 `@` 展开参数文件。

比如有一个文件：

```text
args.txt
```

内容：

```text
--chars
--ignore-blank
--limit
1000
README.md
```

运行：

```shell
filestat @args.txt
```

效果类似直接输入：

```shell
filestat --chars --ignore-blank --limit 1000 README.md
```

如果不想启用 `@` 参数文件，可以配置：

```java
JCommander commander = JCommander.newBuilder()
        .addObject(args)
        .allowParameterOverwriting(true)
        .build();

commander.setExpandAtSign(false);
```

参数文件适合命令很长的场景。

比如数据导入、压测配置、代码生成参数。

### 子命令 Demo：filecli

复杂一点的工具通常会有子命令。

目标：

```shell
filecli stat README.md
filecli copy a.txt b.txt --force
filecli delete temp.log --dry-run
```

根命令：

```java
package com.example.jcommanderdemo.filecli;

import com.beust.jcommander.Parameter;

public class RootCommand {

    @Parameter(names = {"-h", "--help"}, help = true, description = "显示帮助")
    private boolean help;

    @Parameter(names = {"-v", "--verbose"}, description = "详细输出")
    private boolean verbose;

    public boolean isHelp() {
        return help;
    }

    public boolean isVerbose() {
        return verbose;
    }
}
```

`stat` 子命令：

```java
package com.example.jcommanderdemo.filecli;

import com.beust.jcommander.Parameter;
import com.beust.jcommander.Parameters;

import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;

@Parameters(commandDescription = "查看文件信息")
public class StatCommand implements Runnable {

    @Parameter(description = "文件路径", required = true)
    private List<Path> files = new ArrayList<>();

    @Override
    public void run() {
        for (Path file : files) {
            System.out.println("file=" + file.toAbsolutePath());
        }
    }
}
```

`copy` 子命令：

```java
package com.example.jcommanderdemo.filecli;

import com.beust.jcommander.Parameter;
import com.beust.jcommander.Parameters;

import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardCopyOption;
import java.util.ArrayList;
import java.util.List;

@Parameters(commandDescription = "复制文件")
public class CopyCommand implements Runnable {

    @Parameter(description = "源文件和目标文件", required = true)
    private List<Path> paths = new ArrayList<>();

    @Parameter(names = {"-f", "--force"}, description = "覆盖已存在文件")
    private boolean force;

    @Override
    public void run() {
        if (paths.size() != 2) {
            throw new IllegalArgumentException("copy 需要源文件和目标文件");
        }

        Path source = paths.get(0);
        Path target = paths.get(1);

        try {
            if (force) {
                Files.copy(source, target, StandardCopyOption.REPLACE_EXISTING);
            } else {
                Files.copy(source, target);
            }
            System.out.println("复制完成: " + source + " -> " + target);
        } catch (Exception e) {
            throw new RuntimeException("复制失败: " + e.getMessage(), e);
        }
    }
}
```

`delete` 子命令：

```java
package com.example.jcommanderdemo.filecli;

import com.beust.jcommander.Parameter;
import com.beust.jcommander.Parameters;

import java.nio.file.Files;
import java.nio.file.Path;
import java.util.ArrayList;
import java.util.List;

@Parameters(commandDescription = "删除文件")
public class DeleteCommand implements Runnable {

    @Parameter(description = "文件路径", required = true)
    private List<Path> files = new ArrayList<>();

    @Parameter(names = "--dry-run", description = "只打印操作，不真正删除")
    private boolean dryRun;

    @Override
    public void run() {
        for (Path file : files) {
            if (dryRun) {
                System.out.println("[dry-run] 删除: " + file.toAbsolutePath());
                continue;
            }

            try {
                Files.delete(file);
                System.out.println("删除完成: " + file.toAbsolutePath());
            } catch (Exception e) {
                throw new RuntimeException("删除失败: " + file + ", " + e.getMessage(), e);
            }
        }
    }
}
```

主入口：

```java
package com.example.jcommanderdemo.filecli;

import com.beust.jcommander.JCommander;
import com.beust.jcommander.ParameterException;

public class FileCliMain {

    public static void main(String[] args) {
        RootCommand root = new RootCommand();
        StatCommand stat = new StatCommand();
        CopyCommand copy = new CopyCommand();
        DeleteCommand delete = new DeleteCommand();

        JCommander commander = JCommander.newBuilder()
                .addObject(root)
                .addCommand("stat", stat)
                .addCommand("copy", copy)
                .addCommand("delete", delete)
                .programName("filecli")
                .build();

        try {
            commander.parse(args);
        } catch (ParameterException e) {
            System.err.println("参数错误: " + e.getMessage());
            commander.usage();
            System.exit(2);
            return;
        }

        if (root.isHelp()) {
            commander.usage();
            return;
        }

        String command = commander.getParsedCommand();
        if (command == null) {
            commander.usage();
            return;
        }

        try {
            switch (command) {
                case "stat" -> stat.run();
                case "copy" -> copy.run();
                case "delete" -> delete.run();
                default -> commander.usage();
            }
        } catch (RuntimeException e) {
            System.err.println(e.getMessage());
            System.exit(1);
        }
    }
}
```

运行：

```shell
java com.example.jcommanderdemo.filecli.FileCliMain stat README.md
java com.example.jcommanderdemo.filecli.FileCliMain copy a.txt b.txt --force
java com.example.jcommanderdemo.filecli.FileCliMain delete temp.log --dry-run
```

打印子命令帮助可以在代码里指定命令名：

```java
commander.usage("copy");
```

JCommander 的子命令能用，但整体体验不如 Picocli 丰富。

复杂多层 CLI 更建议优先考虑 Picocli。

### 生成帮助信息

手动打印帮助：

```java
commander.usage();
```

指定程序名：

```java
JCommander.newBuilder()
        .addObject(args)
        .programName("filestat")
        .build();
```

给参数设置描述：

```java
@Parameter(names = {"-l", "--limit"}, description = "最多读取多少行")
private int limit;
```

控制排序：

```java
@Parameter(names = "--input", description = "输入文件", order = 1)
private String input;

@Parameter(names = "--output", description = "输出目录", order = 2)
private String output;
```

JCommander 的帮助信息比较基础。

正式工具至少应该包含：

* 程序名
* 参数说明
* 必填参数
* 示例命令
* 错误时友好提示

示例命令通常更适合写在 README 或 `--help` 外层说明里。

### 打包成可执行 Jar

可以用 Maven Shade Plugin 打包。

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
                                <mainClass>com.example.jcommanderdemo.filestat.FileStatMain</mainClass>
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
java -jar target/filestat.jar --chars README.md
```

如果想像普通命令一样运行，可以包一层脚本：

```shell
#!/usr/bin/env sh
java -jar /opt/filestat/filestat.jar "$@"
```

### 单元测试

参数解析也可以测试。

```java
package com.example.jcommanderdemo;

import com.beust.jcommander.JCommander;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

class HelloArgsTest {

    @Test
    void shouldParseHelloArgs() {
        HelloArgs args = new HelloArgs();

        JCommander.newBuilder()
                .addObject(args)
                .build()
                .parse("--name", "张三", "--times", "2");

        assertEquals("张三", args.getName());
        assertEquals(2, args.getTimes());
    }

    @Test
    void shouldParseHelp() {
        HelloArgs args = new HelloArgs();

        JCommander.newBuilder()
                .addObject(args)
                .build()
                .parse("--help");

        assertTrue(args.isHelp());
    }
}
```

测试重点：

* 必填参数
* 默认值
* 布尔开关
* 列表参数
* 非法参数
* 子命令路由
* 自定义校验器

### 常见坑

### 新坐标和旧包名混淆

依赖可以是：

```xml
<groupId>org.jcommander</groupId>
<artifactId>jcommander</artifactId>
<version>3.0</version>
```

但 import 仍然是：

```java
import com.beust.jcommander.JCommander;
```

不要误写成：

```java
import org.jcommander.JCommander;
```

### 忘记捕获 ParameterException

不建议直接：

```java
commander.parse(args);
```

更稳妥：

```java
try {
    commander.parse(args);
} catch (ParameterException e) {
    System.err.println("参数错误: " + e.getMessage());
    commander.usage();
    System.exit(2);
}
```

这样用户看到的是参数错误和帮助信息，而不是一长串异常堆栈。

### help 参数也触发 required 校验

如果有必填参数：

```java
@Parameter(names = "--name", required = true)
private String name;
```

同时有：

```java
@Parameter(names = "--help", help = true)
private boolean help;
```

传入 `--help` 时，JCommander 会识别这是帮助参数，不应该因为缺少 `--name` 报错。

所以帮助参数要加：

```java
help = true
```

### 布尔参数不要传 true / false

布尔开关通常这样用：

```shell
tool --verbose
```

不需要：

```shell
tool --verbose true
```

字段：

```java
@Parameter(names = "--verbose")
private boolean verbose;
```

如果确实希望接收显式布尔值，需要调整 `arity`。

### 位置参数控制能力有限

JCommander 的位置参数适合接收剩余参数列表：

```java
@Parameter(description = "files")
private List<String> files;
```

如果需要这种效果：

```text
第 0 个参数是 source
第 1 个参数是 target
```

通常要自己检查列表长度并取值。

Picocli 对这类场景更自然。

### 子命令没有自动执行

JCommander 负责解析子命令。

业务执行仍然需要根据解析结果分发：

```java
String command = commander.getParsedCommand();
```

然后：

```java
switch (command) {
    case "copy" -> copy.run();
}
```

它不像某些 CLI 框架那样天然把命令对象和执行方法完整绑定。

### 总结

JCommander 可以按这条线理解：

```text
@Parameter 定义参数
JCommander.parse 解析 args
字段拿到解析后的值
ParameterException 表示参数错误
usage 生成帮助信息
addCommand 支持子命令
DynamicParameter 支持 -Dkey=value
ParametersDelegate 复用公共参数
```

实际使用建议：

* 简单工具可以用 JCommander
* 新项目优先使用 `org.jcommander:jcommander:3.0`
* 老项目常见 `com.beust:jcommander:1.82`
* 必填参数用 `required = true`
* 帮助参数一定加 `help = true`
* 参数错误捕获 `ParameterException`
* 复杂 CLI 工具优先评估 Picocli
* 子命令执行逻辑需要自己分发

JCommander 的优势是简单、轻量、上手快。它适合把命令行参数从散乱的 `String[] args` 变成一个参数对象。需求停留在参数解析层面时，它足够好用；需求发展成完整命令行应用时，Picocli 的体验会更完整。
