### 简介

`lscpu` 是 `Linux` 中的一个命令行工具，它通过读取 `/proc/cpuinfo` 和 `sysfs` 来显示详细的 `CPU` 架构信息，包括架构、核心数、线程数、缓存、NUMA 节点等。

### 基础使用

```shell
lscpu
```

显示 `CPU` 架构的摘要

示例

```shell
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                8
On-line CPU(s) list:   0-7
Thread(s) per core:    2
Core(s) per socket:    4
Socket(s):             1
NUMA node(s):          1
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 158
Model name:            Intel(R) Core(TM) i7-7700HQ CPU @ 2.80GHz
Stepping:              9
CPU MHz:               2808.002
BogoMIPS:              5616.00
Virtualization:        VT-x
L1d cache:             32K
L1i cache:             32K
L2 cache:              256K
L3 cache:              6144K
NUMA node0 CPU(s):     0-7
```

输出字段详解

**通用信息**

* `Architecture`: CPU 架构（如 `x86_64, aarch64`）。

* `CPU op-mode(s)`: 支持的运行模式（如 32 位、64 位）。

* `Byte Order`: 字节序（`Little Endian` 或 `Big Endian`）。

* `CPU(s)`: 逻辑 CPU 的总数（物理核心 × 线程数）。

**核心与线程**

* `Thread(s) per core`: 每个物理核心的线程数（超线程技术下通常为 2）。

* `Core(s) per socket`: 每个 CPU 插槽的物理核心数。

* `Socket(s)`: 物理 CPU 插槽的数量（即物理 CPU 数量）。

* `NUMA node(s)`: NUMA（非统一内存访问）节点的数量。

**CPU 型号与性能**

* `Vendor ID`: CPU 制造商（如 GenuineIntel、AuthenticAMD）。

* `CPU family/model/stepping`: CPU 型号的标识符。

* `Model name`: CPU 的完整型号名称。

* `CPU MHz`: CPU 的当前频率。

* `BogoMIPS`: 粗略计算的 CPU 性能指标（无实际意义）。

**缓存**

* `L1d cache/L1i cache`: 一级数据缓存和指令缓存。

* `L2 cache/L3 cache`: 二级和三级缓存大小。

**虚拟化与特性**

* `Virtualization`: 支持的虚拟化技术（如 VT-x、AMD-V）。

* `Flags`: CPU 支持的特性列表（如 sse, avx, avx2）。

### 常用选项

* `-e, --extended`：在表格中显示 CPU 拓扑，包括 CPU、核心、插槽、节点等

* `--json`：`JSON` 格式输出

* `-p, --parse`：可解析的输出（类似 CSV 格式）

* `-x`：以可解析的格式显示额外的详细信息

* `-s, --sysroot`：指定系统根目录（用于分析其他系统的信息）

* `-b, --online`：仅显示在线的 CPU

* `-c, --offline`：仅显示离线的 CPU

* `--all`：显示在线和离线 CPU

* `-y, --physical`：打印物理id而不是逻辑id

### 示例用法

#### 查看逻辑 CPU 布局（CPU-核心-插槽）

```shell
lscpu -e
```

* 查看哪些逻辑 `CPU` 属于同一个核心/插槽

* 针对性能敏感的应用程序（例如数据库、虚拟化）优化 `CPU` 固定

#### 查看 CPU 核心数

```shell
lscpu | grep 'Core(s) per socket'
```

#### 以 JSON 格式显示脚本

```shell
lscpu --json | jq .
```

#### 检查 CPU 限制

```shell
lscpu | grep MHz
```

检查 `CPU` 的扩展频率（对于性能调整或节能很重要）

#### 快速查找 CPU 供应商/型号

```shell
lscpu | grep -E 'Vendor|Model name'
```

#### 查看逻辑 CPU 数量

```shell
slscpu | grep "CPU(s):" | head -n 1
```

#### 检查是否启用超线程

```shell
lscpu | grep "Thread(s) per core"
# 输出为 2 表示启用超线程，1 表示未启用
```

#### 查看 NUMA 节点分布

```shell
lscpu | grep -i numa
numactl --hardware  # 更详细的 NUMA 信息
```

#### 获取 CPU 型号

```shell
lscpu | grep "Model name"
```

#### 用于脚本解析的输出

```shell
lscpu -p | grep -v '^#'  # 过滤注释行
# 输出格式：CPU,CORE,SOCKET,NODE,...
```

### CPU 拓扑结构图示

#### lscpu -e 输出

```shell
CPU NODE SOCKET CORE L1d:L1i:L2:L3 ONLINE MAXMHZ MINMHZ
0   0    0      0    0:0:0:0      yes    4000   400
1   0    0      0    0:0:0:0      yes    4000   400
2   0    0      1    1:1:1:0      yes    4000   400
3   0    0      1    1:1:1:0      yes    4000   400
4   0    0      2    2:2:2:0      yes    4000   400
5   0    0      2    2:2:2:0      yes    4000   400
6   0    0      3    3:3:3:0      yes    4000   400
7   0    0      3    3:3:3:0      yes    4000   400
```

#### 图形拓扑图

```shell
Socket 0
└── Core 0
    ├── CPU 0 (Logical thread 1)
    └── CPU 1 (Logical thread 2)
└── Core 1
    ├── CPU 2
    └── CPU 3
└── Core 2
    ├── CPU 4
    └── CPU 5
└── Core 3
    ├── CPU 6
    └── CPU 7
```

**字段解释**

* `CPU`（逻辑线程）：操作系统看到并调度的内容。

* 核心：可能有多个线程（超线程）的物理执行单元。

* `Socket`：物理CPU封装。

* 节点：`NUMA` 节点（用于内存局部性感知）。

