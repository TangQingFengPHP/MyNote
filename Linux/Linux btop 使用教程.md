### 简介

`btop` 是一个基于终端的现代系统资源监控器，具有美观的图形界面、响应快、功能丰富等特点。它支持查看 `CPU`、内存、磁盘、网络、进程，并可以方便地筛选和管理进程。

### 功能总览

启动命令：

```shell
btop
```

界面分为以下几部分：

* CPU 区域：显示每个核心的使用率、频率、温度等

* 内存区域：显示总内存、缓存、`swap`、当前使用率

* 磁盘区域：每个设备或挂载点的读写速度、使用率

* 网络区域：显示各网卡的收发速率、`IP`、数据量等

* 进程区域：显示活跃进程，支持排序、搜索、终止

![alt text](/images/Linux/btop-image-1.png)

### 快捷键操作

* `ESC`：打开/关闭设置菜单

* `m`：切换内存显示单位（KB/MB/GB）

* `e`：展开/折叠进程树（默认是平铺）

* `f`：搜索进程名（实时筛选）

* `↑ / ↓`：上下移动进程光标

* `← / →`：横向移动到不同模块（CPU/内存/磁盘）

* `Enter`：进入设置菜单或确认

* `k`：`kill` 选中进程（发送默认 `SIGTERM`）

* `z`：显示详细的进程信息（类似 `top` 的详情）

* `q`：退出 `btop`

* `s`：修改进程排序方式

### 设置菜单（按 ESC 进入）

* 主题（Theme）

* 是否启用图形动画（Graph mode）

* 更新频率（Update time）

* 默认排序方式（Process sorting）

* 启动时是否展开进程树

* 是否启用 Swap 显示等

### 进程管理功能

* 使用 `↑ / ↓` 选择进程

* 按 k 终止（`kill`）它（发送 SIGTERM）

* 使用 z 查看进程的详细状态（如 CPU time、线程数等）

* 搜索进程（按 f，输入关键字即可筛选）

### 配置文件位置

```shell
~/.config/btop/btop.conf
```

### 主题切换

查看主题

```shell
ls /usr/share/btop/themes/
```

切换方法：

```shell
btop --theme monokai
```

### 设置排序

#### 方法一：使用 UI 设置

* 在 `btop` 主界面中按 `Esc` 键，进入设置菜单

* 使用方向键移动到 "Options" 或 "Process options"

* 找到 `Process sorting` 选项

* 按 `← / →` 左右键进行切换，支持的选项包括：
    * `cpu`：按 CPU 使用率排序
    * `mem`：按内存使用率排序找到如下配置行：
    * `pid`：按进程 ID 排序
    * `time`：按运行时间排序
    * `user`：按所属用户排序

* 设置后按 `Esc` 退出设置界面即可生效

#### 方法二：修改配置文件

```shell
vim ~/.config/btop/btop.conf
```

找到如下配置行：

```ini
proc_sorting="cpu"
```

将 `cpu` 替换为需要的排序方式，例如 `mem、pid、user` 等，然后保存。

#### 方法三：在进程区域按左右键，直接切换排序方式

### 常用选项

* `--theme <name>`：启动时使用指定主题

* `--utf-force`：强制使用 UTF8 图形

* `--no-update`：不自动检查更新

* `--help`：查看帮助信息

### 与其他工具对比

|  工具   |  优势   |  劣势   |
| --- | --- | --- |
|  `top`   |  系统内置，最轻量   | 界面难看、信息少    |
|  `htop`   |  可交互，界面稍好   |  不支持磁盘/网络显示   |
|  `btop`   |  图形界面炫酷，功能全面，易用   |  稍占资源（图形渲染）   |
