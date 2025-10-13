### 简介

#### 上下文：它出现在哪里？

常见于以下命令输出中：

```shell
$ git show
```

输出示例：

```diff
diff --git a/src/test.txt b/src/test.txt
new file mode 100644
index 0000000..7f3e5a4
--- /dev/null
+++ b/src/test.txt
@@ -0,0 +1,2 @@
hello
world
```

```shell
$ git commit -m "Add new files"
[main 1a2b3c4] Add new files
 2 files changed, 15 insertions(+)
 create mode 100644 README.md
 create mode 100755 script.sh
 create mode 120000 symlink.txt
```

这些都表示此提交中新增了一个文件，且该文件的“mode（模式）”为 100644。

### Git 的文件模式（mode）是什么？

`Git` 在内部保存每个文件的三类信息（在 `index` 和 `tree` 对象中）：

| 信息       | 含义                     |
| ---------- | ------------------------ |
| mode       | 文件类型 + 权限          |
| SHA-1/Hash | 文件内容对应的 blob 对象 |
| 文件名     | 文件路径名               |

所以，`Git` 实际上并不直接保存整个文件，而是保存 “文件内容（`blob`）+ `mode` + 文件名” 三元组。

### 100644 的含义分解

100644 是一个 文件模式（mode），用八进制表示。

文件类型部分（前两位）

| 模式前缀 | 文件类型                  |
| -------- | ------------------------- |
| `100`    | 普通文件（regular file）  |
| `120`    | 符号链接（symbolic link） |
| `160`    | Git 子模块（gitlink）     |

权限部分（后三位）

| 模式  | 含义                         |
| ----- | ---------------------------- |
| `644` | 普通文件（可读写、不可执行） |
| `755` | 可执行文件（脚本、二进制等） |

实际权限解析（`Linux` 风格）：

```ini
6 = rw- （所有者可读写）
4 = r-- （组可读）
4 = r-- （其他用户可读）
```

即 `rw-r--r--`。

### 常见几种模式对照表

| Git 模式（八进制） | 类型              | 含义                        | 示例                      |
| ------------------ | ----------------- | --------------------------- | ------------------------- |
| `100644`           | 普通文件          | 普通非执行文件（rw-r--r--） | 源代码、配置文件          |
| `100755`           | 普通文件          | 可执行文件（rwxr-xr-x）     | 脚本、二进制文件          |
| `120000`           | 符号链接          | 保存符号链接路径            | Linux 下的 symlink        |
| `160000`           | gitlink           | 子模块                      | `.gitmodules` 指向的 repo |
| `040000`           | 目录（tree 对象） | 存储 tree 引用              | 非实际文件                |

### 查看文件在 Git 中的 mode

可以用以下命令查看工作区、索引和历史中对应文件的 `mode`：

查看 `index`（暂存区）：

```shell
git ls-files -s
```

示例输出：

```
100644 4d1f48a5e9e6b032d212d24ec8594f98639b6f08 0       README.md
100755 05a43e2a77fdab64d45c601944bcdbba05cf8cb1 0       build.sh
```

第一列即是文件模式。

查看 `tree`（提交快照）：

```shell
git ls-tree HEAD
```

输出：

```
100644 blob 4d1f48a5e9e6b032d212d24ec8594f98639b6f08    README.md
100755 blob 05a43e2a77fdab64d45c601944bcdbba05cf8cb1    build.sh
```

### Git 为什么只保存两种权限（644 与 755）

* `Git` 主要在 跨平台 环境使用。

* `Windows` 文件系统不支持 `POSIX` 权限位（如执行权限）。

* 为了避免混乱，`Git` 简化为两种状态：

    * 普通文件（非执行）

    * 可执行文件（执行位）

> 也就是说，`Git` 只关心文件是否“可执行”，其余权限位不影响版本内容。

### 如何修改可执行标志

修改本地文件执行权限：

```shell
chmod +x script.sh
```

再次提交：

```shell
git add script.sh
git commit -m "make script executable"
```

提交后再次查看

```
mode change 100644 => 100755 script.sh
```

### 子模块（160000）与符号链接（120000）

| 模式     | 类型     | 存储内容               | 备注               |
| -------- | -------- | ---------------------- | ------------------ |
| `160000` | 子模块   | 存储子仓库的 commit ID | 不保存实际文件     |
| `120000` | 符号链接 | 存储符号链接目标路径   | 不保存实际文件内容 |

示例（符号链接）：

```shell
ln -s ../config.yml link.yml
git add link.yml
git commit -m "add symlink"
# 显示 create mode 120000 link.yml
```

### Git 是如何在内部保存这些信息的？

`Git` 提交由多个对象组成：

```sql
Commit → Tree → Blob
```

举例：

```sql
commit对象
  ↓
tree对象（保存 mode + 文件名 + blob 引用）
  ├── 100644 README.md → blob(内容)
  ├── 100755 build.sh  → blob(内容)
  └── 040000 src/ → tree(子目录)
```

也就是说，100644 这一信息实际上存在 `tree` 对象 中，是提交快照（`snapshot`）的一部分。

### 实际操作示例

#### 场景 1：添加普通文件

```shell
# 创建普通文本文件
echo "Hello World" > hello.txt

# 查看权限
ls -l hello.txt
# -rw-r--r-- 1 user group 12 Jan 1 10:00 hello.txt

# 添加到 Git 并提交
git add hello.txt
git commit -m "Add hello"
# create mode 100644 hello.txt
```

#### 场景 2：创建可执行脚本

```shell
# 创建脚本并设为可执行
echo '#!/bin/bash\necho "Hello"' > run.sh
chmod +x run.sh

# 查看权限
ls -l run.sh
# -rwxr-xr-x 1 user group 22 Jan 1 10:00 run.sh

# 添加到 Git 并提交
git add run.sh
git commit -m "Add script"
# create mode 100755 run.sh
```

#### 场景 3：创建符号链接

```shell
# 创建符号链接
ln -s target.txt link.txt

# 查看文件类型
ls -l link.txt
# lrwxrwxrwx 1 user group 10 Jan 1 10:00 link.txt -> target.txt

# 添加到 Git 并提交
git add link.txt
git commit -m "Add symlink"
# create mode 120000 link.txt
```

### 文件模式的管理

#### 查看 Git 中的文件模式

```shell
# 查看 Git 数据库中文件的模式
git ls-tree HEAD
# 100644 blob 89abcde    README.md
# 100755 blob 2345678    script.sh
# 120000 blob 3456789    link.txt
# 040000 tree 4567890    src

# 查看特定提交的文件模式
git ls-tree 1a2b3c4
```

#### 修改文件模式

```shell
# 如果文件模式不正确，可以修改文件权限后重新添加
chmod +x my-script.py
git add my-script.py
git commit -m "Fix file permissions"

# 或者使用 Git 命令更新索引中的文件模式
git update-index --chmod=+x my-script.py
```

#### 忽略文件权限变化

```shell
# 告诉 Git 忽略文件权限变化（在跨平台协作时有用）
git config core.filemode false

# 检查当前配置
git config core.filemode
```

#### 跨平台注意：

* `Linux/Mac`：`Git` 尊重文件权限。

* `Windows`：权限支持有限，默认 `core.fileMode = false`，模式变化不会被跟踪。