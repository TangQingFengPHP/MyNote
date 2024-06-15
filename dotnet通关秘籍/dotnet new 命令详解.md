## 一、简介

dotnet new 命令用于基于指定的模板创建新项目、配置文件、解决方案。

## 二、常用选项

* `-o, --output <output>`：指定创建项目后放置的目录名

示例：

```shell
dotnet new console -o MyConsoleApp
```

* `-n, --name <name>`：指定项目的名称，不指定则使用项目目录名称

示例：

```shell
dotnet new console -o MyConsoleApp -n MyConsoleApp1
```

* `--dry-run`：模拟项目创建过程，实际上不创建项目

示例：

```shell
dotnet new console --dry-run
```

* `--force`：如果项目已经存在，强制覆盖

```shell
dotnet new console --force
```

* `-lang, --language {C#|F#|VB}`：指定项目使用的语言，项目模板具体支持的语言需查看模板详情，例如：使用 `dotnet new list console` 可查看console模板支持 C#、F#、VB。

![alt text](/images/dotnet-new-image.png)

示例：

```shell
dotnet new console -lang F#
```

* `-f, --framework <FRAMEWORK>`：指定项目使用的目标框架，此选项选择的框架版本会写到项目配置文件中，例如：net6.0、net7.0，项目模板具体支持的框架需查看模板帮助信息，例如：`dotnet new console -h` 可查看到console模板支持 net6.0、net7.0、net8.0。

![alt text](/images/dotnet-new-image-1.png)

示例：

```shell
dotnet new console -f net8.0
```

* `-v, --verbosity <LEVEL>`：指定命令控制台输出信息的级别，可用的值有：q[uiet]、m[inimal]、n[ormal]、diag[nostic]，默认是normal，加括号的意思是可输入首字母或整个单词。

示例：

```shell
dotnet new console -v m
```

* `-d, --diagnostics`：启动诊断输出。

示例：

```shell
dotnet new console -d
```

* `-?, -h, --help`：显示命令行帮助信息

示例：

```shell
dotnet new console -h
```

* 查看模板的具体选项，例如：查看console模板的选项

![alt text](/images/dotnet-new-image-2.png)

```shell
dotnet new console -h
```

> -h 这个选项可以作为公共选项，即子命令或子选项都可以使用
>
> 例如：`dotnet new console -h -lang F#`

## 三、常用子命令

* `dotnet new list`：列出可用的模板，如果没有指定模板名，则列出所有模板，没有额外手动安装模板包的情况下，显示的是安装sdk时已经内置好的模板包。

列出所有模板示例：

```shell
dotnet new list
```

![alt text](/images/dotnet-new-image-3.png)

列出指定模板示例：

```shell
dotnet new list console
```

![alt text](/images/dotnet-new-image-4.png)

* `dotnet new search`：在 NuGet.org上搜索指定的模板

示例：

```shell
dotnet new search spa
```

![alt text](/images/dotnet-new-image-5.png)

* `dotnet new install`：安装指定的模板包，既是包则可以包含多个模板，参数接包id，即上图包名称。

示例：安装上图第一个模板：websharper-spa，使用它的包名称。

```shell
dotnet new install WebSharper.Templates
```

![alt text](/images/dotnet-new-image-6.png)

可以看到一个包里面包含多个模板。

* `dotnet new uninstall`：卸载模板包，如果指定了参数，则卸载指定的模板包，否则列出所有手动安装的模板包。

![alt text](/images/dotnet-new-image-7.png)

示例：

```shell
dotnet new uninstall WebSharper.Templates
```

* `dotnet new update`：检查当前安装的模板包是否有更新，然后安装更新。

示例：

```shell
dotnet new update
```

## 四、如何配置shell使dotnet命令自动补全？

* zsh shell

在 `~/.zshrc` 配置文件中添加以下行：

```shell
# zsh parameter completion for the dotnet CLI

_dotnet_zsh_complete()
{
  local completions=("$(dotnet complete "$words")")

  # If the completion list is empty, just continue with filename selection
  if [ -z "$completions" ]
  then
    _arguments '*::arguments: _normal'
    return
  fi

  # This is not a variable assignment, don't remove spaces!
  _values = "${(ps:\n:)completions}"
}

compdef _dotnet_zsh_complete dotnet
```

* bash shell

在 `~/.bashrc` 配置文件中添加以下行：

```shell
# bash parameter completion for the dotnet CLI

function _dotnet_bash_complete()
{
  local cur="${COMP_WORDS[COMP_CWORD]}" IFS=$'\n' # On Windows you may need to use use IFS=$'\r\n'
  local candidates

  read -d '' -ra candidates < <(dotnet complete --position "${COMP_POINT}" "${COMP_LINE}" 2>/dev/null)

  read -d '' -ra COMPREPLY < <(compgen -W "${candidates[*]:-}" -- "$cur")
}

complete -f -F _dotnet_bash_complete dotnet
```

