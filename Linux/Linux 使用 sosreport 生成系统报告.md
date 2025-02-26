### 简介

`sosreport` 命令是许多 `Linux` 发行版上可用的工具，特别是基于 `Red hat` 的系统（RHEL、CentOS、Fedora），它有助于收集系统配置详细信息、日志和诊断信息，以便进行故障排除。它生成一个压缩的 `tarball`（存档文件），其中包含各种系统信息，这些信息对于调试、诊断问题或向支持团队提供数据非常有用。

### 安装

* `RHEL/CentOS/Fedora`

```shell
sudo yum install sos
# or
sudo dnf install sos
```

* `Ubuntu/Debian`

```shell
sudo apt install sosreport
```

### 示例用法

#### 生成基本报告

```shell
sudo sosreport
```

* 这将创建一个包含所有收集到的系统诊断信息的压缩 `tarball` 文件

* 该 `tarball` 通常命名为 `sosreport-<hostname>-<timestamp>.tar.xz`，其中 `<hostname>` 是系统的主机名，`<timestamp>` 是生成报告的日期和时间。

#### 为报告指定自定义目录

```shell
sudo sosreport --output /path/to/output/directory
```

#### 以安静模式运行 sosreport

```shell
sudo sosreport -q
```

#### 使用特定配置运行 sosreport 

```shell
sudo sosreport --profile network

# 这将限制诊断报告仅与网络相关的信息
```

#### 收集一组特定的信息

可以使用各种选项自定义 `sosreport` 收集哪些信息。例如，可以分别收集日志、配置文件和其他系统数据

```shell
sudo sosreport --no-compress --skip-logs
```

* `--no-compress`：禁用最终报告的压缩

* `--skip-logs`：跳过收集日志

#### 按时间限制报告收集

可以使用 `--skip-timestamp` 选项将日志和信息的收集限制在特定日期

```shell
sudo sosreport --skip-timestamp
```

#### 添加自定义信息

这允许收集根据特定需求定制的额外非标准信息。

```shell
sudo sosreport --add-custom <module-name>
```

#### 提供案例 ID 或描述

可以在生成报告时添加案例 ID 或说明。这在与支持团队合作时特别有用，因为他们可以快速识别的报告并将其与案例编号关联起来

```shell
sudo sosreport --case-id <case-id> --description "System Troubleshooting"
```

#### 使用特定模块运行 sosreport

```shell
sudo sosreport --module network,storage

# 这将仅收集与网络和存储相关的信息，从而可能减少报告的大小
```

#### 启用调试模式

这将收集更详细的诊断数据，可能对高级故障排除有用

```shell
sudo sosreport --debug
```

`sosreport` 完成后，它生成一个压缩文件（通常是 `.tar.xz` 格式)。在这个文件中，将发现组织到目录中的各种日志、配置文件和系统详细信息。一些常见的目录包括：

* `/etc/`：/etc/ 中的配置文件，包括网络配置、防火墙设置等

* `/var/log/`：系统日志、应用程序日志和其他重要日志文件

* `/proc/`：有关系统进程和资源的信息

* `/sys/`：有关内核和系统参数的信息

* `/home/`：用户家目录（有时包含用于特定调试）