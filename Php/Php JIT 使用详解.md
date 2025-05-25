### 简介

`PHP 8` 引入的 `JIT`（Just-In-Time 编译器） 是该版本的一个重要性能特性，首次让 `PHP` 有了运行时即时编译的能力，从解释型语言迈向了“编译执行”的方向。

### 什么是 JIT？

`JIT` 是 即时编译（Just-In-Time compilation） 的缩写，作用是在运行时把 `PHP` 字节码（`Opcode`）编译成本地机器码，跳过 `Zend VM` 的解释执行步骤，提高运行效率。

核心机制包括：

* 与 `OPcache` 集成 `JIT` 作为 `OPcache` 的扩展实现，在 `OPcache` 的优化（如操作码缓存、数据流分析）基础上进一步生成机器码。

* 多阶段编译
  * 解释执行阶段：`PHP` 代码首先被解析为 `OPcode`（中间代码）。

  * 热点识别：通过运行时统计（如循环次数、函数调用频率）确定需优化的代码段（即“热代码”）。

  * 机器码生成：使用 `DynASM`（基于 `LuaJIT` 的汇编器）将热代码编译为机器码，后续执行直接调用机器码。
  
简单来说：

* 之前的执行流程（`PHP 7` 及以下）：
`PHP` 代码 → 编译成 `Opcode` → `Zend` 虚拟机解释执行

* `PHP 8` 的 `JIT` 流程：
`PHP` 代码 → 编译成 `Opcode` → `JIT` 编译成本地机器码 → `CPU` 执行

### 如何启用 JIT？

`JIT` 是 `Opcache` 的一部分，配置文件在 `php.ini` 中进行设置。

* 步骤 1：开启 `Opcache`

```ini
opcache.enable=1
opcache.enable_cli=1
```

* 步骤 2：开启 `JIT`

```ini
opcache.jit=1255
opcache.jit_buffer_size=100M
```

参数说明：

* `opcache.jit_buffer_size：JIT` 使用的内存缓冲区，必须设置大于 0 才会启用 `JIT`（推荐 100M 起）。

* `opcache.jit`：`JIT` 的级别和策略编码（常用值见下方）。

### JIT 策略说明（opcache.jit）

`JIT` 有四种 策略模式（`strategy`），值的计算方式如下：

```ini
opcache.jit = function_tracing + (strategy * 256)
```

| 策略（strategy） | 名称     | 说明                       |
| ---------------- | -------- | -------------------------- |
| 0                | disable  | 禁用 JIT                   |
| 1                | tracing  | 最优化，适合长时间运行脚本 |
| 2                | function | 编译整个函数/方法          |
| 3                | return   | 编译返回点                 |
| 4                | call     | 编译调用点                 |

例如：

`1255 = 4 * 256 + 231`：表示策略为 `call`，函数级别为 `231`。

常用推荐配置：`opcache.jit=1255`

### JIT 对性能的影响

| 类型         | 性能提升                 |
| ------------ | ------------------------ |
| 数学密集运算 |  极大提升（最高 10 倍） |
| I/O 密集任务 |  几乎无提升             |
| Web 请求响应 |  提升有限（10\~20%）    |

`JIT` 对 `Web` 项目的实际性能提升 并不显著，因为 `Web` 应用大部分时间花在数据库、网络、`I/O` 等非 `CPU` 密集型操作上。

### 适合使用 JIT 的场景

* 图像处理、数学运算（如 `Mandelbrot、FFT`）

* 视频处理、机器学习扩展

* 长生命周期的 `PHP` 应用（如 `Swoole/Hyperf`、`RoadRunner`）

* 命令行 `PHP`（开启 `opcache.enable_cli=1`）

### 不建议使用 JIT 的场景

* `I/O` 密集型 `Web` 项目（如 `Laravel` 普通 `Web` 应用）

* 内存受限服务器

* 小型脚本或 `CLI` 工具（冷启动成本不划算）

### 如何检测 JIT 是否生效？

* 使用 `PHP CLI` 执行：

```shell
php -i | grep JIT
```

示例输出：

```ini
opcache.jit => 1255
opcache.jit_buffer_size => 100M
opcache.jit_debug => 0
opcache.jit_status => enabled
```

* 也可以通过 `Web` 页面：

查找 `JIT` 区块，确认是否启用。

```php
phpinfo();
```

### JIT 状态查看

```php
$jitStatus = opcache_get_status(true);
print_r($jitStatus['jit']);
```

### JIT 的限制与注意事项

* 内存开销：`JIT` 缓冲区（`opcache.jit_buffer_size`）会占用额外内存，需根据业务规模调整。

* 架构限制：仅支持 `x86-64` 架构，`ARM` 或其他架构（如 `M1` 芯片）暂不支持。

* 兼容性问题：部分 `PHP` 扩展（如 `xdebug`）可能与 `JIT` 冲突，需关闭调试工具。

* 版本差异：`PHP 8.0` 的 `JIT` 稳定性较弱，建议升级至 `PHP 8.1+`（修复了大量编译优化问题）。







