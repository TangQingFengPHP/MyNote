### 简介

`Linux` 中的 `halt` 命令用于立即关闭系统。它还可用于关闭电源或重新启动机器，具体取决于所使用的选项。

### 基础语法

```shell
halt [OPTION]
```

默认情况下，`halt` 需要 `root` 权限

```shell
sudo halt
```

### 常用选项

* `-p`：停止后关闭系统电源。（与 `poweroff` 相同）

* `--reboot`：重新启动系统而不是停止系统

* `--force`：强制立即停止而不通知进程

* `--help`：显示帮助信息

### 示例用法

#### 停止系统

```shell
sudo halt

# 这将停止所有进程并停止系统，但可能不会关闭电源。
```

#### 停止并关闭电源

```shell
sudo halt -p
```

#### 强制停止

```shell
sudo halt --force

# 这会强制立即停止，而不会正确停止进程
```

#### 重新启动而不是停止

```shell
sudo halt --reboot
```

### 可选的命令

* `poweroff`：相当于 `halt -p`（停止和关闭电源）

* `shutdown -h now`：类似于 `halt` 但允许调度

* `reboot`：类似于 `halt——reboot`，重新启动系统

### halt poweroff shutdown 三者的区别

* `halt`：停止系统但可能不会关闭电源

* `halt -p / poweroff`：停止并关闭系统电源

* `shutdown -h now`：更平滑，能调度时间进行关机

* `reboot`：重新启动系统

