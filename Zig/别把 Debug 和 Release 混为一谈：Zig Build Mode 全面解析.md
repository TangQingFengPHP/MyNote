# 别把 Debug 和 Release 混为一谈：Zig Build Mode 全面解析

编译同一份 Zig 代码时，Debug 和 ReleaseFast 可能得到完全不同的运行行为。

Build Mode 不只是“要不要优化”的开关，还会影响：

- 编译器优化程度；
- 运行时安全检查；
- 生成文件大小；
- 编译速度；
- unreachable、整数溢出和数组越界等非法行为的处理方式；
- 代码中通过 builtin.mode 读取到的编译期配置。

Zig 提供四种常用模式：

~~~text
Debug
ReleaseSafe
ReleaseFast
ReleaseSmall
~~~

本文按 Zig 0.16.0 语法编写，示例可以保存为 main.zig 后运行。

## 一、四种 Build Mode

官方定义可以先记成下面这张表：

~~~text
模式             优化             安全检查             主要目标
Debug            关闭             开启                 调试和快速编译
ReleaseSafe      开启             开启                 性能与安全平衡
ReleaseFast      开启             关闭                 运行速度优先
ReleaseSmall     体积优化         关闭                 二进制体积优先
~~~

“关闭安全检查”不是说程序自动变得不安全，而是许多运行时安全检查默认不再插入。代码仍然必须遵守 Zig 语言规则；一旦触发未检查的非法行为，结果可能不可预测。

## 二、Debug：开发阶段的默认选择

直接编译时，Debug 通常是默认模式：

~~~bash
zig run main.zig
~~~

也可以显式指定：

~~~bash
zig run main.zig -O Debug
~~~

Debug 的特点：

- 优化较少；
- 编译速度快；
- 运行时安全检查开启；
- 便于定位越界、溢出和非法状态；
- 二进制通常较大，运行速度通常较慢。

例如数组越界：

~~~zig
const std = @import("std");

pub fn main() void {
    const numbers = [_]i32{ 10, 20, 30 };
    const index: usize = 10;

    std.debug.print("{}\n", .{numbers[index]});
}
~~~

numbers 只有 0、1、2 三个合法下标。Debug 模式会进行边界检查，运行时通常直接报告 index out of bounds，而不是继续访问未知内存。

Debug 适合：

- 日常开发；
- 单元测试；
- 调试内存和控制流问题；
- 观察错误回溯；
- 验证输入校验是否完整。

## 三、ReleaseSafe：生产环境的稳妥选择

ReleaseSafe 同时开启优化和安全检查：

~~~bash
zig run main.zig -O ReleaseSafe
~~~

它可以理解成：

~~~text
Debug 的安全检查
        +
Release 的优化
~~~

ReleaseSafe 通常适合 CLI 工具、后端服务、网络程序和长时间运行的服务。

ReleaseSafe 并不保证所有逻辑错误都会被发现。它主要保留语言运行时层面的安全检查，业务规则仍然需要通过普通条件判断和错误返回处理。

例如用户输入的端口号不能只依赖安全检查：

~~~zig
const ConfigError = error{
    InvalidPort,
};

fn parsePort(value: u32) ConfigError!u16 {
    if (value == 0 or value > 65535) {
        return error.InvalidPort;
    }

    return @intCast(value);
}
~~~

这是业务校验，四种模式都应该执行。

## 四、ReleaseFast：运行速度优先

ReleaseFast 开启优化，并默认关闭运行时安全检查：

~~~bash
zig run main.zig -O ReleaseFast
~~~

主要特点：

- 运行速度优先；
- 优化开启；
- 安全检查关闭；
- 编译时间通常更长；
- 二进制通常仍然可能较大。

ReleaseFast 适合性能基准测试、数值计算、图形和音视频处理、游戏核心循环，以及已经充分验证过的高性能程序。

ReleaseFast 不是“更正式的 Debug”，也不是任何生产服务的默认答案。代码中的非法行为一旦失去安全检查，可能变成未定义行为。

## 五、ReleaseSmall：二进制体积优先

ReleaseSmall 以减小生成文件为主要目标：

~~~bash
zig run main.zig -O ReleaseSmall
~~~

适合 WebAssembly、嵌入式程序、固件、Bootloader、极简容器和下载体积敏感的小工具。

ReleaseSmall 同样默认关闭运行时安全检查。体积变小不代表运行速度一定最快，优化目标不同，不能把 ReleaseSmall 当成 ReleaseFast 的替代品。

## 六、命令行中的 -O

直接操作单个 Zig 文件时，使用 -O 指定模式：

~~~bash
zig build-exe main.zig -O Debug
zig build-exe main.zig -O ReleaseSafe
zig build-exe main.zig -O ReleaseFast
zig build-exe main.zig -O ReleaseSmall
~~~

zig run 也支持：

~~~bash
zig run main.zig -O Debug
zig run main.zig -O ReleaseSafe
zig run main.zig -O ReleaseFast
zig run main.zig -O ReleaseSmall
~~~

可以通过 -femit-bin 指定输出文件名，方便比较产物：

~~~bash
zig build-exe main.zig -O Debug -femit-bin=demo-debug
zig build-exe main.zig -O ReleaseSafe -femit-bin=demo-safe
zig build-exe main.zig -O ReleaseFast -femit-bin=demo-fast
zig build-exe main.zig -O ReleaseSmall -femit-bin=demo-small
~~~

查看文件大小：

~~~bash
ls -lh demo-*
~~~

性能比较也应该分别使用 Release 模式运行，Debug 的结果没有参考价值：

~~~bash
time ./demo-safe
time ./demo-fast
time ./demo-small
~~~

## 七、zig build-exe 和 zig build 的区别

这两个命令经常同时出现，但职责不同：

~~~text
zig build-exe   直接编译一个 Zig 源文件
zig build       执行项目中的 build.zig
~~~

zig build-exe 不需要 build.zig，只关心源文件到可执行文件的转换：

~~~bash
zig build-exe main.zig
zig build-exe main.zig -O ReleaseFast
~~~

这种方式适合学习语法、编写小 Demo、快速验证代码和制作单文件工具。

zig build 则是项目级构建命令，会查找并执行 build.zig：

~~~text
build.zig
    ↓
决定编译哪些目标、使用哪些参数、执行哪些步骤
    ↓
调用底层编译器
~~~

一个常见项目结构：

~~~text
demo/
├── build.zig
└── src/
    └── main.zig
~~~

build.zig 可以统一管理：

- 可执行文件；
- 静态库和动态库；
- 测试；
- 运行步骤；
- 外部依赖；
- 安装构建产物；
- 交叉编译；
- 自定义命令行参数。

常用命令可以这样区分：

~~~bash
zig build-exe main.zig       # 单文件编译
zig build                    # 按 build.zig 构建项目
zig build run                # 执行 build.zig 中的 run 步骤
zig build test               # 执行 build.zig 中的 test 步骤
~~~

可以把两者的关系理解成：

~~~text
zig build-exe
    编译器命令
    源文件 → 可执行文件

zig build
    构建系统命令
    build.zig → 管理整个项目的构建流程
~~~

zig build 内部可能调用可执行文件、库或测试相关的底层编译动作，但它本身负责的是构建流程编排。

因此，单文件 Demo 使用 zig build-exe 更直接；完整项目使用 zig build，并通过 build.zig 统一管理不同 Build Mode。

## 八、build.zig 中的 -Doptimize

使用 Zig Build System 时，命令行参数变成：

~~~bash
zig build -Doptimize=Debug
zig build -Doptimize=ReleaseSafe
zig build -Doptimize=ReleaseFast
zig build -Doptimize=ReleaseSmall
~~~

build.zig 使用 standardOptimizeOption 接收这个选项：

~~~zig
const std = @import("std");

pub fn build(b: *std.Build) void {
    const target = b.standardTargetOptions(.{});
    const optimize = b.standardOptimizeOption(.{});

    const exe = b.addExecutable(.{
        .name = "demo",
        .root_module = b.createModule(.{
            .root_source_file = b.path("src/main.zig"),
            .target = target,
            .optimize = optimize,
        }),
    });

    b.installArtifact(exe);
}
~~~

关键在这两行：

~~~zig
const optimize = b.standardOptimizeOption(.{});
.optimize = optimize,
~~~

第一行读取命令行中的 -Doptimize，第二行把模式传给可执行文件的 root module。

如果 build.zig 没有调用 standardOptimizeOption，命令行中的 -Doptimize 就不会自动成为项目选项。

## 九、build.zig 中固定模式

也可以把模式直接写死：

~~~zig
.optimize = .ReleaseFast,
~~~

这种写法适合某些专用目标，例如专门构建 WebAssembly 发布包。但普通项目通常应该保留 standardOptimizeOption，让构建命令决定模式。

固定模式的缺点是本地调试不方便切回 Debug，测试和发布也容易误用同一种模式。

## 十、在代码中读取当前模式

builtin 是编译器提供的内置模块：

~~~zig
const std = @import("std");
const builtin = @import("builtin");

pub fn main() void {
    std.debug.print("mode = {s}\n", .{
        @tagName(builtin.mode),
    });
}
~~~

不同命令会得到不同输出：

~~~text
zig run main.zig -O Debug
mode = Debug

zig run main.zig -O ReleaseFast
mode = ReleaseFast
~~~

builtin.mode 是编译期已知值，可以用于编译期条件分支：

~~~zig
const std = @import("std");
const builtin = @import("builtin");

fn debugLog(comptime format: []const u8, args: anytype) void {
    if (builtin.mode == .Debug) {
        std.debug.print(format, args);
    }
}

pub fn main() void {
    debugLog("debug id = {}\n", .{1001});
    std.debug.print("application started\n", .{});
}
~~~

Debug 模式输出两行，Release 模式只输出 application started。编译器可以删除永远不会执行的分支。

这里使用的仍然是正常 Zig 语法，分支条件只是编译期已经确定。

## 十一、安全检查与非法行为

Zig 中很多操作可能触发非法行为，例如数组越界、整数溢出、除零、错误的 optional 解包、错误的指针转换和执行 unreachable。

Debug 和 ReleaseSafe 通常保留相应的安全检查，ReleaseFast 和 ReleaseSmall 默认关闭安全检查以便优化。

### 整数溢出

~~~zig
const std = @import("std");

pub fn main() void {
    var value: u8 = 255;
    value += 1;
    std.debug.print("{}\n", .{value});
}
~~~

在安全检查开启的模式下，这通常会触发 integer overflow。

如果业务语义就是允许环绕，应该明确使用 wrapping 运算：

~~~zig
var value: u8 = 255;
value +%= 1;
~~~

结果是 0。不要依赖 Build Mode 决定溢出后的行为。

### unreachable

~~~zig
const value: ?i32 = null;
const number = value orelse unreachable;
~~~

unreachable 不是普通错误处理，而是向编译器声明这条路径不可能发生。Debug 和 ReleaseSafe 通常会在实际到达时触发 panic；ReleaseFast 和 ReleaseSmall 可能按照这个假设进行激进优化。

只有经过可靠证明的分支才适合使用 unreachable。用户输入、网络响应和文件内容都不满足这个条件。

## 十二、局部恢复安全检查

ReleaseFast 和 ReleaseSmall 默认关闭安全检查，但可以在局部重新开启：

~~~zig
const std = @import("std");

pub fn main() void {
    @setRuntimeSafety(true);

    var value: u8 = 255;
    value += 1;

    std.debug.print("{}\n", .{value});
}
~~~

setRuntimeSafety 会影响所在函数或作用域，具体范围由调用位置决定。也可以只包住需要额外保护的局部 block：

~~~zig
pub fn main() void {
    {
        @setRuntimeSafety(true);
        // 这个作用域中的安全检查被强制开启
    }
}
~~~

这类写法适合经过严格验证的底层热点代码，不适合用来掩盖未经处理的输入错误。

## 十三、断言和业务校验不是一回事

断言适合表达程序内部必须成立的条件：

~~~zig
std.debug.assert(index < items.len);
~~~

用户输入则应使用普通条件和错误返回：

~~~zig
const InputError = error{
    InvalidIndex,
};

fn getItem(items: []const i32, index: usize) InputError!i32 {
    if (index >= items.len) {
        return error.InvalidIndex;
    }

    return items[index];
}
~~~

区别可以概括为：

~~~text
assert       程序内部假设，失败说明程序存在 bug
error return 外部输入，失败属于正常业务分支
~~~

Build Mode 可能影响断言和安全检查的表现，因此业务正确性不能只依赖断言。

## 十四、完整 Demo：根据 Build Mode 控制调试日志

~~~zig
const std = @import("std");
const builtin = @import("builtin");

fn debugLog(comptime format: []const u8, args: anytype) void {
    if (builtin.mode == .Debug) {
        std.debug.print(format, args);
    }
}

fn calculate(items: []const i32) i32 {
    var sum: i32 = 0;

    for (items) |item| {
        sum += item;
    }

    debugLog("sum calculated, count = {}\n", .{items.len});
    return sum;
}

pub fn main() void {
    const items = [_]i32{ 10, 20, 30 };
    const sum = calculate(&items);

    std.debug.print(
        "mode = {s}, sum = {}\n",
        .{ @tagName(builtin.mode), sum },
    );
}
~~~

运行：

~~~bash
zig run main.zig -O Debug
zig run main.zig -O ReleaseSafe
zig run main.zig -O ReleaseFast
~~~

Debug 会看到 debug 日志，ReleaseSafe 和 ReleaseFast 不会看到。这种方式适合调试日志、开发诊断信息和测试辅助输出。

不要用它来删除必须存在的业务校验。业务校验应和 Build Mode 无关。

## 十五、完整 Demo：比较四种产物

main.zig：

~~~zig
const std = @import("std");
const builtin = @import("builtin");

fn fibonacci(n: u64) u64 {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

pub fn main() void {
    const result = fibonacci(40);

    std.debug.print("mode = {s}\n", .{
        @tagName(builtin.mode),
    });
    std.debug.print("result = {}\n", .{result});
}
~~~

分别构建：

~~~bash
zig build-exe main.zig -O Debug -femit-bin=fib-debug
zig build-exe main.zig -O ReleaseSafe -femit-bin=fib-safe
zig build-exe main.zig -O ReleaseFast -femit-bin=fib-fast
zig build-exe main.zig -O ReleaseSmall -femit-bin=fib-small
~~~

查看体积：

~~~bash
ls -lh fib-*
~~~

查看运行时间：

~~~bash
time ./fib-safe
time ./fib-fast
time ./fib-small
~~~

这个 Demo 只用于观察构建差异。实际性能测试还需要多次运行、排除启动开销，并使用真实业务数据。

## 十六、常见误区

### 1. ReleaseFast 就是生产模式

错误。ReleaseFast 的安全检查默认关闭，适合经过验证的性能敏感代码。普通服务更常见的选择是 ReleaseSafe。

### 2. ReleaseSmall 一定比 ReleaseFast 快

错误。ReleaseSmall 的主要目标是二进制体积，不是运行速度。

### 3. Release 模式会自动修复错误

错误。优化不会修复数组越界、错误的指针使用和数据竞争。关闭安全检查后，问题可能更难发现。

### 4. Debug 和 ReleaseSafe 完全一样

错误。两者都开启安全检查，但 Debug 优化较少、编译较快；ReleaseSafe 开启优化，运行特征不同。

### 5. 用 builtin.mode 替代业务判断

不推荐：

~~~zig
if (builtin.mode == .Debug) {
    validateUserInput();
}
~~~

用户输入的合法性在所有模式都应该检查。Build Mode 只适合控制调试行为、诊断信息和经过确认的优化策略。

## 十七、日常选择建议

~~~text
学习、开发、测试       Debug
一般生产服务            ReleaseSafe
高性能基准和计算        ReleaseFast
WASM、固件、极小工具    ReleaseSmall
~~~

发布前至少检查两件事：

1. Debug 和 ReleaseSafe 下测试是否都通过；
2. ReleaseFast 或 ReleaseSmall 下是否依赖了本应保留的安全检查。

如果程序包含大量底层指针操作、手写 SIMD、底层内存访问或 unreachable，ReleaseFast 前需要增加更完整的测试和基准验证。

## 总结

Build Mode 不只是编译器优化开关，而是优化策略、安全检查和编译期配置的组合：

~~~text
Debug
    优化少，安全检查开，适合开发调试

ReleaseSafe
    优化开，安全检查开，适合稳妥的生产程序

ReleaseFast
    优化开，安全检查关，适合速度优先的程序

ReleaseSmall
    体积优化，安全检查关，适合体积敏感的程序
~~~

最常用的命令：

~~~bash
zig build-exe main.zig -O Debug
zig build-exe main.zig -O ReleaseSafe
zig build-exe main.zig -O ReleaseFast
zig build-exe main.zig -O ReleaseSmall
~~~

Build System 中使用：

~~~bash
zig build -Doptimize=ReleaseSafe
~~~

代码中读取：

~~~zig
const builtin = @import("builtin");

if (builtin.mode == .Debug) {
    // 只保留 Debug 诊断逻辑
}
~~~

最终原则：

> 业务正确性不能依赖 Build Mode；Build Mode 只决定已经正确的代码如何构建和运行。

参考：

- Zig 0.16.0 Language Reference：Build Mode、Illegal Behavior、setRuntimeSafety
- Zig 0.16.0 Language Reference：Compile Variables、Zig Build System
