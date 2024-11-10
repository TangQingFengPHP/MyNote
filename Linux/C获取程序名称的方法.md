### 方法一：

> 使用 `extern char *__progname`

#### 介绍：

`__progname` 是一个定义在C标准库中的特殊的全局变量，持有当前运行程序的名称，它仅在类Unix系统中可用，如：Linux、MacOS。

#### 解析：

* `extern`：的含义是声明此变量是定义在其他地方，通常是在C运行时中。

* `char *__progname`：表示为一个指向字符数组的指针，值包含正在运行的可执行文件的名称

* `__progname`：的值通常不是文件的全路径，仅包含文件的名称，如：`/usr/bin/myapp`，则 `__progname` 为 `myapp`

#### 使用示例：

```c
#include <stdio.h>

extern char *__progname;

int main(void) {
    printf("This program is called: %s\n", __progname);
    return 0;
}
```

### 方法二：

> 使用 `argv[0]` 获取

#### 介绍：

这是通用获取程序名称的方式，在类Unix系统和Windows系统中都可以用。

#### 解析：

如果只聚焦与类Unix平台，不兼容扩平台，直接使用 `__progname` 的方式比较方便，`argv[0]` 需要显式的在方法中指定参数。

#### 使用示例：

```c
#include <stdio.h>

int main(int argc, char *argv[]) {
    printf("Program name: %s\n", argv[0]);
    return 0;
}
```

### 方法三：

> 使用 `/proc/self/exe` 获取

#### 介绍：

此方法只能在Linux系统中使用，通过读取 `/proc/self/exe` 这个软链接来获取程序的执行路径。

#### 解析：

`/proc/self/exe` 是一个指向当前进程的可执行文件的软链接。

`proc` 牵扯到虚拟文件系统（提供进程和系统的信息）

`self` 实际上是指向当前运行进程的PID，例如当前的PID是：1234，则 `/proc/1234/exe` 与 `/proc/self/exe` 相等。

#### 使用示例：

```c
#include <stdio.h>
#include <unistd.h>

int main() {
    char path[1024];
    ssize_t len = readlink("/proc/self/exe", path, sizeof(path) - 1);
    if (len != -1) {
        path[len] = '\0';  // Null-terminate the string
        printf("Executable path: %s\n", path);
    } else {
        perror("readlink");
    }
    return 0;
}
```

### 方法四：

> 使用 `GetModuleFileName` API获取

#### 介绍：

此方法只能在Windows系统中使用。

#### 解析：

`GetModuleFileName` 可以获取到可执行文件的完整路径

#### 使用示例：

```c
#include <windows.h>
#include <stdio.h>

int main() {
    char path[MAX_PATH];
    GetModuleFileName(NULL, path, MAX_PATH);
    printf("Program path: %s\n", path);
    return 0;
}
```

### 番外：

#### 如何下载获取到C运行时的源码

* 使用包管理器下载，如在Debian/Ubuntu系统中

```shell
sudo apt-get source libc6
```

* 从官网下载压缩包

https://www.gnu.org/software/libc/

* 从Github镜像仓库中下载

```shell
git clone https://github.com/bminor/glibc.git
```