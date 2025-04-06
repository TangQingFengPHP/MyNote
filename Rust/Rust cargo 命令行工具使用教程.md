### 简介

`cargo` 是 `Rust` 的构建系统和包管理器，负责创建项目、编译代码、管理依赖、运行测试等，是日常开发中最常用的工具。

### 创建项目

```shell
cargo new project_name      # 创建 binary 项目（可执行）
cargo new --lib mylib       # 创建 library 项目（供其它项目调用）
```

它会创建一个项目结构：

```shell
project_name/
├── Cargo.toml        # 项目信息和依赖配置
└── src/
    └── main.rs       # 项目主入口（lib.rs 对于库）
```

### 项目结构和配置文件

`Cargo.toml` 是项目的核心配置文件，类似于 `Java` 的 `pom.xml` 或 `Node.js` 的 `package.json`：

```toml
[package]
name = "my_project"
version = "0.1.0"
edition = "2025"

[dependencies]
rand = "0.8"     # 添加依赖
```

### 常用命令

#### 编译项目

```shell
cargo build          # 构建项目（debug 模式）
cargo build --release  # 构建 release 模式（优化）
```

#### 运行项目

```shell
cargo run
```

#### 带参数运行

```shell
cargo run -- arg1 arg2
```

#### 检查语法和错误（不编译生成目标文件）

```shell
cargo check
```

#### 添加依赖包

```shell
cargo add serde        # 需要安装 cargo-edit 插件
```

安装 `cargo-edit`：

```shell
cargo install cargo-edit
```

### 依赖管理

#### 在 Cargo.toml 中手动添加：

```toml
[dependencies]
serde = "1.0"
reqwest = { version = "0.11", features = ["json"] }
```

#### 添加本地 crate：

```toml
[dependencies]
mycrate = { path = "../mycrate" }
```

#### 添加 Git 仓库依赖：

```toml
[dependencies]
mycrate = { git = "https://github.com/user/mycrate.git" }
```

### 测试 & 文档

#### 测试

```shell
cargo test
```

#### 生成文档

```shell
cargo doc --open
```

### 发布 Crate 到 crates.io

```shell
cargo login                # 登录 crates.io（需要 token）
cargo publish              # 发布
cargo package              # 打包并检查
```

### 构建配置与工作区（workspace）

如果有多个 `crate` 项目组成一个工程：

根目录 `Cargo.toml` 配置：

```toml
[workspace]
members = [
    "core",
    "utils",
    "web"
]
```

### 常用 cargo 插件

```shell
cargo install cargo-edit         # 管理依赖（cargo add/remove/etc）
cargo install cargo-watch        # 自动监控并重编译
cargo install cargo-audit        # 审计安全问题
cargo install cargo-outdated     # 查看依赖是否过期
```

### 命令速查表

* `cargo new`：	创建项目

* `cargo build`：编译项目

* `cargo run`：编译并运行

* `cargo check`：检查代码是否可编译

* `cargo test`：运行测试

* `cargo doc --open`：生成并打开文档

* `cargo add xxx`：添加依赖（需插件）

* `cargo update`：更新依赖到最新版本

* `cargo clean`：清理构建产物

* `cargo install`：安装二进制 `crate`（如 `ripgrep`）

