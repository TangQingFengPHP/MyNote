### 简介

`Linux` 中的 `cmp` 命令用于逐字节比较两个文件。它通常用于检查两个文件是否相同，如果不相同，则检查它们之间的差异。

### 基础语法

```shell
cmp [OPTION]... FILE1 [FILE2 [SKIP1 [SKIP2]]]
```

* `FILE1, FILE2`：要比较的文件。如果省略 `FILE2`，则将 `FILE1` 与标准输入进行比较。

* `SKIP1, SKIP2`：开始比较之前在每个文件中跳过的可选字节偏移量。

### 常用选项

* `-b`：打印不同的字节。

* `-i <N>`：忽略两个文件的前 `N` ​​个字节。

* `-i <N:M>`：忽略文件 `1` 的 `N` 个字节和文件 `2` 的 `M` 个字节。

* `-l`：显示所有不同的字节。

* `-n <NUM>`：仅比较前 `NUM` 个字节。

* `-s`：静默模式（无输出，仅退出状态）。

* `--help`：显示帮助信息

* `--version`：显示版本信息

### 用法示例

#### 比较两个文件

```shell
cmp file1.txt file2.txt
```

* 如果文件相同：无输出（退出代码 0）

* 如果不同：则打印第一个差异的字节和行号

#### 安静模式比较

```shell
cmp -s file1.txt file2.txt && echo "Same" || echo "Different"
```

非常适合脚本编写，仅使用退出代码。

#### 显示所有差异

```shell
cmp -l file1.txt file2.txt
```

打印每个不同的字节以及字节偏移量和值（八进制）。

#### 比较文件，跳过字节

```shell
cmp file1.txt file2.txt 10 10
```

比较之前跳过两个文件的前 10 个字节。

#### 仅比较前 N 个字节

```shell
cmp -n 100 file1.txt file2.txt
```

仅比较每个文件的前 100 个字节。


### 退出码

* `0`：文件相同

* `1`：文件不同

* `2`：发生错误

### 使用场景

* 二进制文件比较（例如，比较编译的文件或图像）

* 调试文件生成中的差异

* 验证文件传输或备份

* 在脚本中用于检查更改

### 自动化脚本中使用cmp

#### 新建 `compare-files.sh` 文件

```shell
#!/bin/bash

# Usage: ./compare-files.sh file1 file2

FILE1="$1"
FILE2="$2"
LOG="compare.log"

# Check input
if [[ ! -f "$FILE1" || ! -f "$FILE2" ]]; then
  echo "Both arguments must be existing files."
  exit 2
fi

# Compare files silently
if cmp -s "$FILE1" "$FILE2"; then
  echo "✅ Files are identical."
  echo "$(date): $FILE1 and $FILE2 are identical." >> "$LOG"
else
  echo "❌ Files differ:"
  cmp -b "$FILE1" "$FILE2" | head -n 10
  echo "$(date): $FILE1 and $FILE2 differ." >> "$LOG"
fi
```

#### 添加可执行权限

```shell
chmod +x compare-files.sh
```

#### 运行测试

```shell
./compare-files.sh fileA.txt fileB.txt
```

输出示例

* 如果文件不同：

```shell
❌ Files differ:
byte 15, line 2 is 110 m  111 n
byte 28, line 3 is 141 a  145 e
...
```

* 如果文件相同

```shell
✅ Files are identical.
```

日志保存在 `compare.log` 中。

### 添加颜色输出及集成fzf

#### 修改 `compare-files.sh`

```shell
#!/bin/bash

# Color codes
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

LOG="compare.log"

# Check if fzf is installed
if ! command -v fzf &> /dev/null; then
  echo -e "${RED}fzf is not installed. Please install it to use this script.${NC}"
  exit 1
fi

echo -e "${YELLOW}Select the first file:${NC}"
FILE1=$(find . -type f | fzf)
[[ -z "$FILE1" ]] && echo -e "${RED}No file selected. Exiting.${NC}" && exit 1

echo -e "${YELLOW}Select the second file:${NC}"
FILE2=$(find . -type f | fzf)
[[ -z "$FILE2" ]] && echo -e "${RED}No file selected. Exiting.${NC}" && exit 1

echo -e "${YELLOW}Comparing: $FILE1 ↔ $FILE2${NC}"

# Compare files silently
if cmp -s "$FILE1" "$FILE2"; then
  echo -e "${GREEN}✅ Files are identical.${NC}"
  echo "$(date): $FILE1 and $FILE2 are identical." >> "$LOG"
else
  echo -e "${RED}❌ Files differ:${NC}"
  cmp -b "$FILE1" "$FILE2" | head -n 10
  echo "$(date): $FILE1 and $FILE2 differ." >> "$LOG"
fi
```

#### 运行测试

* 运行 `./compare-files.sh`

* 提示两次使用 `fzf` 选择文件

* 如果不同则显示红色，如果相同则显示绿色✅

* 将结果记录到 `compare.log`