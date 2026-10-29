# C 语言 strcpy() 用法详解与实战 Demo

`strcpy()` 是 C 标准库 `<string.h>` 中提供的字符串复制函数，用于将一个 C 字符串复制到另一个字符数组中。

它是 C 语言字符串操作中最基础的函数之一，经常与 `malloc()`、`calloc()`、`strlen()`、`strcmp()`、`strncpy()`、`memcpy()` 一起使用。

学习重点：`strcpy()` 不会自动申请内存，也不会检查目标缓冲区大小，因此必须确保目标内存足够。

## 一、strcpy() 基本概念

### 1. 函数原型

```
#include <string.h>

char *strcpy(char *dest, const char *src);
```

| 参数   | 类型           | 说明                                 |
| ------ | -------------- | ------------------------------------ |
| `dest` | `char *`       | 目标内存地址，用于存储复制后的字符串 |
| `src`  | `const char *` | 源字符串地址                         |
| 返回值 | `char *`       | 返回 `dest` 指针                     |

例如：

```
#include <stdio.h>
#include <string.h>

int main(void) {
    char source[] = "Hello World";
    char destination[50];

    strcpy(destination, source);

    printf("Source: %s\n", source);
    printf("Destination: %s\n", destination);

    return 0;
}
```

输出：

```
Source: Hello World
Destination: Hello World
```

执行过程：

1. `source` 保存 `"Hello World"`。
2. `destination` 是一个容量为 50 字节的字符数组。
3. `strcpy()` 从 `source` 读取字符。
4. 将所有字符复制到 `destination`。
5. 最后将字符串结束符 `'\0'` 一起复制。

### 2. strcpy() 的内存结构

![alt text](/images/C/strcpy/image.png)

虽然 `"Hello"` 只有 5 个可见字符，但实际需要 6 字节：

![alt text](/images/C/strcpy/image-1.png)

其中额外的 1 字节用于保存 `'\0'`。

## 二、为什么 strcpy() 必须复制 `'\0'`？

C 语言字符串不是一个自带长度信息的对象。

例如：

```
char name[] = "Alice";
```

实际内存布局：

```
'A' 'l' 'i' 'c' 'e' '\0'
```

`strlen()`、`printf("%s")` 等字符串函数通过寻找 `'\0'` 判断字符串结束位置。

下面的代码：

```
char source[] = "Hello";
char dest[20];

strcpy(dest, source);
```

相当于复制：

```
dest[0] = 'H';
dest[1] = 'e';
dest[2] = 'l';
dest[3] = 'l';
dest[4] = 'o';
dest[5] = '\0';
```

如果缺少 `'\0'`，后续调用：

```
printf("%s\n", dest);
```

可能继续读取数组后面的内存，导致越界访问等未定义行为。

## 三、实战 Demo 1：使用 malloc() 动态复制字符串

结合前面学习的 `malloc()`，我们可以实现一个动态字符串复制函数。

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

char *string_duplicate(const char *source) {
    if (source == NULL) {
        return NULL;
    }

    size_t length = strlen(source);

    char *dest = malloc(length + 1);

    if (dest == NULL) {
        return NULL;
    }

    strcpy(dest, source);

    return dest;
}

int main(void) {
    const char *source = "Hello C Language";

    char *result = string_duplicate(source);

    if (result == NULL) {
        fprintf(stderr, "Memory allocation failed\n");
        return EXIT_FAILURE;
    }

    printf("Source: %s\n", source);
    printf("Result: %s\n", result);

    // 修改复制后的字符串
    result[0] = 'h';

    printf("Modified: %s\n", result);
    printf("Original: %s\n", source);

    free(result);
    result = NULL;

    return EXIT_SUCCESS;
}
```

输出：

```
Source: Hello C Language
Result: Hello C Language
Modified: hello C Language
Original: Hello C Language
```

### 关键知识点

1\. `malloc(length + 1)`

动态申请足够的内存，用于存储字符串及结束符。

2\. `strcpy(dest, source)`

将字符串复制到新申请的内存中。

3\. 复制后的字符串独立存储

修改 `result` 不会影响 `source`。

4\. 需要调用 `free()`

因为目标内存由 `malloc()` 分配，需要手动释放。

这实际上就是一个简化版的 `strdup()`。

在 POSIX 环境中，也可以直接使用：

```
char *result = strdup("Hello World");
```

`strdup()` 会申请内存并复制字符串，返回的内存同样需要 `free()`。

## 四、实战 Demo 2：结构体中的字符串复制

在 C 语言中，结构体经常包含固定大小的字符数组。

例如用户信息：

```
typedef struct {
    int id;
    char username[32];
    char email[128];
} User;
```

如果需要为 `username` 赋值，不能这样写：

```
User user;

user.username = "Alice"; // 错误
```

因为 `username` 是字符数组，C 语言不允许对数组直接赋值。

可以使用 `strcpy()`。

### 完整 Demo

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    int id;
    char username[32];
    char email[128];
} User;

int user_init(
    User *user,
    int id,
    const char *username,
    const char *email
) {
    if (user == NULL || username == NULL || email == NULL) {
        return -1;
    }

    // 检查目标数组容量
    if (strlen(username) >= sizeof(user->username)) {
        return -1;
    }

    if (strlen(email) >= sizeof(user->email)) {
        return -1;
    }

    user->id = id;

    strcpy(user->username, username);
    strcpy(user->email, email);

    return 0;
}

int main(void) {
    User user = {0};

    if (user_init(
        &user,
        1001,
        "Alice",
        "alice@example.com"
    ) != 0) {
        fprintf(stderr, "User initialization failed\n");
        return EXIT_FAILURE;
    }

    printf("ID: %d\n", user.id);
    printf("Username: %s\n", user.username);
    printf("Email: %s\n", user.email);

    return EXIT_SUCCESS;
}
```

输出：

```
ID: 1001
Username: Alice
Email: alice@example.com
```

这里的重点是：

```
strlen(username) >= sizeof(user->username)
```

为什么使用 `>=`，而不是 `>`？

假设目标数组容量为 32 字节：

- 字符串长度为 30：需要 31 字节，可以复制。
- 字符串长度为 31：需要 32 字节，可以复制。
- 字符串长度为 32：需要 33 字节，不能复制。

因此，必须保证：

![alt text](/images/C/strcpy/image-2.png)

这里 `dest` 必须是能通过 `sizeof` 获得完整容量的数组表达式，而不是已经退化为指针的函数参数。

## 五、实战 Demo 3：动态结构体对象中的字符串管理

上一节的结构体使用固定大小数组。

如果用户名长度不固定，也可以使用 `char *` 动态分配。

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    int id;
    char *name;
    char *email;
} User;

char *copy_string(const char *source) {
    size_t length = strlen(source);

    char *dest = malloc(length + 1);

    if (dest == NULL) {
        return NULL;
    }

    strcpy(dest, source);

    return dest;
}

User *user_create(
    int id,
    const char *name,
    const char *email
) {
    if (name == NULL || email == NULL) {
        return NULL;
    }

    User *user = malloc(sizeof(*user));

    if (user == NULL) {
        return NULL;
    }

    user->id = id;
    user->name = NULL;
    user->email = NULL;

    user->name = copy_string(name);

    if (user->name == NULL) {
        free(user);
        return NULL;
    }

    user->email = copy_string(email);

    if (user->email == NULL) {
        free(user->name);
        free(user);
        return NULL;
    }

    return user;
}

void user_destroy(User *user) {
    if (user == NULL) {
        return;
    }

    free(user->name);
    free(user->email);
    free(user);
}

int main(void) {
    User *user = user_create(
        1001,
        "Alice",
        "alice@example.com"
    );

    if (user == NULL) {
        return EXIT_FAILURE;
    }

    printf("ID: %d\n", user->id);
    printf("Name: %s\n", user->name);
    printf("Email: %s\n", user->email);

    user_destroy(user);

    return EXIT_SUCCESS;
}
```

这里有一个很重要的内存管理问题。

![alt text](/images/C/strcpy/image-3.png)

这也是 C 语言与 C#、Java 内存管理差异较大的地方。

## 六、实战 Demo 4：实现安全的字符串复制函数

`strcpy()` 最大的问题是：不知道目标缓冲区的容量。

例如：

```
char dest[5];

strcpy(dest, "Hello World");
```

目标数组只有 5 字节，却要写入 12 字节。

这会造成缓冲区越界，属于未定义行为，可能导致崩溃、数据破坏或安全漏洞。

我们可以封装一个安全版本，要求调用者显式传入目标容量。

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int safe_strcpy(
    char *dest,
    size_t dest_size,
    const char *src
) {
    if (dest == NULL || src == NULL || dest_size == 0) {
        return -1;
    }

    size_t length = strlen(src);

    if (length >= dest_size) {
        return -1;
    }

    strcpy(dest, src);

    return 0;
}

int main(void) {
    char buffer[10] = {0};

    if (safe_strcpy(
        buffer,
        sizeof(buffer),
        "Hello"
    ) == 0) {
        printf("Success: %s\n", buffer);
    }

    if (safe_strcpy(
        buffer,
        sizeof(buffer),
        "Hello World"
    ) != 0) {
        printf("Copy failed: buffer too small\n");
    }

    return EXIT_SUCCESS;
}
```

输出：

```
Success: Hello
Copy failed: buffer too small
```

这个版本的策略是：如果目标容量不足，直接拒绝复制，而不是截断字符串。

需要注意，它仍然要求 `src` 是有效且以 `'\0'` 结尾的字符串，并且源与目标内存不能重叠。

## 七、strcpy()、strncpy()、memcpy()、strdup() 的区别

这几个函数经常容易混淆。

| 函数        | 用途              | 自动分配内存 | 自动复制 `\0`          |
| ----------- | ----------------- | ------------ | ---------------------- |
| `strcpy()`  | 复制 C 字符串     | 否           | 是                     |
| `strncpy()` | 最多复制 n 个字节 | 否           | 不一定                 |
| `memcpy()`  | 复制指定字节数    | 否           | 否，除非指定范围包含它 |
| `strdup()`  | 动态复制字符串    | 是           | 是                     |

### 1. strcpy()

```
char dest[20];

strcpy(dest, "Hello");
```

适合目标缓冲区容量已知且足够的情况。

### 2. strncpy()

```
char dest[5];

strncpy(dest, "Hello World", sizeof(dest));
```

这里 `dest` 不会包含 `'\0'`，因此不能安全地当作 C 字符串使用。

如果需要手动终止：

```
char dest[5];

strncpy(dest, "Hello World", sizeof(dest) - 1);
dest[sizeof(dest) - 1] = '\0';
```

结果是 `"Hell"`。

但 `strncpy()` 有补零行为，且不保证截断后自动终止，因此并不是所有场景下都适合作为 `strcpy()` 的安全替代品。

### 3. memcpy()

```
char source[] = "Hello";
char dest[20];

memcpy(dest, source, strlen(source) + 1);
```

这里复制了包括 `'\0'` 在内的 6 字节。

`memcpy()` 不要求数据必须是字符串，也可以复制结构体、数组和二进制缓冲区，但源和目标内存区域不能重叠。

如果内存可能重叠，应使用 `memmove()`。

### 4. strdup()

```
char *dest = strdup("Hello");

if (dest != NULL) {
    printf("%s\n", dest);
    free(dest);
}
```

`strdup()` 是 POSIX 函数，在 macOS 和 Linux 上常见，但它不是 ISO C11/C17 标准库的一部分；C23 已将其纳入标准。

## 八、strcpy() 常见错误

### 1. 目标指针没有分配内存

错误：

```
char *dest;

strcpy(dest, "Hello");
```

`dest` 是未初始化指针，没有指向有效的可写内存。

正确：

```
char *dest = malloc(6);

if (dest != NULL) {
    strcpy(dest, "Hello");
    free(dest);
}
```

### 2. 修改字符串字面量

错误：

```
char *dest = "Hello";

strcpy(dest, "World");
```

字符串字面量不能被修改，这属于未定义行为。

正确：

```
char dest[] = "Hello";

strcpy(dest, "World");
```

这里目标数组容量为 6 字节，刚好能容纳 `"World"` 和结束符。

### 3. 目标数组容量不足

错误：

```
char dest[5];

strcpy(dest, "Hello");
```

`"Hello"` 需要 6 字节，而不是 5 字节。

### 4. 源字符串没有结束符

错误：

```
char source[3] = {'A', 'B', 'C'};
char dest[10];

strcpy(dest, source);
```

`source` 不是有效的 C 字符串，因为缺少 `'\0'`。

`strcpy()` 会继续读取数组后面的内存，造成未定义行为。

正确：

```
char source[] = "ABC";
char dest[10];

strcpy(dest, source);
```

### 5. 源和目标内存重叠

错误：

```
char text[20] = "Hello World";

strcpy(text + 2, text);
```

`strcpy()` 不允许源和目标内存区域重叠。

如果需要处理重叠区域，可以使用：

```
char text[20] = "Hello World";

memmove(text + 2, text, strlen(text) + 1);
```

执行后：

```
HeHello World
```

前提是目标缓冲区容量足够。

## 九、strcpy() 的底层实现原理

为了更好理解指针，可以自己实现一个简化版 `strcpy()`。

```
#include <stdio.h>

char *my_strcpy(char *dest, const char *src) {
    char *original = dest;

    while ((*dest++ = *src++) != '\0') {
        // 循环复制
    }

    return original;
}

int main(void) {
    char dest[20];

    my_strcpy(dest, "Hello");

    printf("%s\n", dest);

    return 0;
}
```

输出：

```
Hello
```

最重要的是这一行：

```
while ((*dest++ = *src++) != '\0')
```

可以拆解成：

```
while (1) {
    char ch = *src;

    *dest = ch;

    src++;
    dest++;

    if (ch == '\0') {
        break;
    }
}
```

它的执行逻辑是：

1. 读取 `src` 当前指向的字符。
2. 写入 `dest` 当前指向的内存。
3. 两个指针分别向后移动一个字符。
4. 判断复制的字符是否为 `'\0'`。
5. 如果是结束符，停止循环。

注意，`dest++` 和 `src++` 是后置自增，先使用当前指针，再移动指针。

### 为什么需要保存 original？

```
char *original = dest;
```

因为复制过程中 `dest` 会不断向后移动。

如果直接返回最终的 `dest`，返回的就不是目标字符串的起始地址。

因此要保存原始指针：

```
return original;
```

这也解释了为什么标准库 `strcpy()` 返回 `char *`，而不是 `void`。

例如可以这样写：

```
char buffer[20];

printf("%s\n", strcpy(buffer, "Hello"));
```

不过，这个自定义实现只是教学示例，不包含容量检查，也不能处理重叠内存。

## 十、实战 Demo 5：解析字符串并保存到结构体

假设从文件或者网络接口读取了一行数据：

```
1001,Alice,25
```

需要解析成结构体：

```
typedef struct {
    int id;
    char name[32];
    int age;
} User;
```

完整示例：

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    int id;
    char name[32];
    int age;
} User;

int parse_user(const char *line, User *user) {
    if (line == NULL || user == NULL) {
        return -1;
    }

    int id;
    int age;
    char name[32];
    int consumed = 0;

    // 限制字符串读取长度
    int matched = sscanf(
        line,
        "%d,%31[^,],%d%n",
        &id,
        name,
        &age,
        &consumed
    );

    if (matched != 3 || line[consumed] != '\0') {
        return -1;
    }

    user->id = id;
    user->age = age;

    strcpy(user->name, name);

    return 0;
}

int main(void) {
    const char *data = "1001,Alice,25";

    User user = {0};

    if (parse_user(data, &user) != 0) {
        fprintf(stderr, "Parse failed\n");
        return EXIT_FAILURE;
    }

    printf("ID: %d\n", user.id);
    printf("Name: %s\n", user.name);
    printf("Age: %d\n", user.age);

    return EXIT_SUCCESS;
}
```

输出：

```
ID: 1001
Name: Alice
Age: 25
```

这里 `sscanf()` 的 `%31[^,]` 最多读取 31 个非逗号字符，并保留一个字节给 `'\0'`，因此后面的 `strcpy()` 是安全的。

这个示例适合学习简单格式解析。生产环境中还需要考虑整数范围、空白字符、编码、CSV 引号和转义规则等问题。

## 十一、strcpy() 的使用建议

实际项目中可以按照下面的原则选择：

| 场景                             | 建议                                |
| -------------------------------- | ----------------------------------- |
| 字符串常量复制到足够大的固定数组 | `strcpy()`                          |
| 复制外部输入的字符串             | 先检查长度和容量                    |
| 复制指定长度的二进制数据         | `memcpy()`                          |
| 复制可能重叠的内存               | `memmove()`                         |
| 动态复制字符串                   | `strdup()` 或 `malloc() + strcpy()` |
| 需要截断并保证字符串终止         | 显式检查容量，使用受限复制方案      |

另外，在 macOS/Linux 的实际开发中，可以考虑 `snprintf()`：

```
char dest[10];

int n = snprintf(dest, sizeof(dest), "%s", "Hello World");

if (n < 0) {
    fprintf(stderr, "Formatting failed\n");
} else if ((size_t)n >= sizeof(dest)) {
    fprintf(stderr, "String was truncated\n");
}
```

这里目标数组最多容纳 9 个可见字符和一个 `'\0'`，因此字符串会被截断为 `"Hello Wor"`。

`snprintf()` 能限制写入的字节数，但仍然需要检查返回值，判断是否发生截断。