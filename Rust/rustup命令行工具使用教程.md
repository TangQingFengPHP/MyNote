### 简介

`rustup` 是 `Rust` 官方推荐的安装工具和版本管理器，用于安装、管理和更新 `Rust` 编译器（`rustc`）、包管理器（`cargo`）以及其他组件和工具链（`toolchains`）。

### 安装 rustup

#### 在 `macOS/Linux` 上：

```shell
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

#### `Windows` 上直接运行 .exe 安装程序（官网：https://rustup.rs）。

安装完后，默认会将 `cargo`, `rustc`, `rustup` 等可执行文件放在 `~/.cargo/bin` 下，并添加到 `$PATH`。

### 常用命令大全

#### 查看当前使用的 Rust 版本

```shell
rustup show

或

rustc --version
```

#### 安装指定版本的 Rust

```shell
rustup install stable
rustup install beta
rustup install nightly
rustup install 1.86.0  # 安装指定版本
```

#### 卸载某个版本

```shell
rustup uninstall nightly
```

#### 切换全局默认版本

```shell
rustup default stable
rustup default nightly
rustup default 1.86.0
```

#### 为某个项目设置局部 Rust 版本

* 在项目目录里运行：

```shell
rustup override set nightly
```

这会在当前目录创建一个 `rust-toolchain` 文件。

* 取消 override：

```shell
rustup override unset
```

#### 更新 rustup 和所有组件

```shell
rustup update
```

只更新某个版本：

```shell
rustup update nightly
```

#### 安装和管理组件（如 clippy、rustfmt）

```shell
rustup component add clippy
rustup component add rustfmt
rustup component list --installed
```

#### 卸载组件：

```shell
rustup component remove clippy
```

#### 添加目标平台（用于交叉编译）

```shell
rustup target add wasm32-unknown-unknown
```

#### 查看所有可用 `target`：

```shell
rustup target list
```

#### 查看真实路径（shim）

```shell
rustup which cargo
rustup which rustc
```

#### 使用某个版本运行命令

```shell
rustup run nightly cargo build
```

### rust-toolchain.toml（高级用法）

在项目根目录下创建：

```toml
[toolchain]
channel = "nightly-2024-04-01"
components = ["rustfmt", "clippy"]
targets = ["wasm32-unknown-unknown"]
```

这个文件可以让项目自动锁定特定版本和组件，非常适合团队协作。

### 卸载 rustup

```shell
rustup self uninstall
```

### 查看帮助

```shell
rustup help
rustup help install
```

