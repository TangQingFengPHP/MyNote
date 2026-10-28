# C 语言 malloc() 详解与实战 Demo

`malloc()` 是 C 语言动态内存管理中最核心的函数之一，用于在程序运行期间动态申请内存。

它和 `malloc()` 最大的区别是：

- `malloc()`：申请指定大小的内存，不会初始化内存内容。
- `calloc()`：申请指定数量和大小的内存，并将所有字节清零。

本篇从基础语法、内存原理、指针操作、结构体、动态数组、链表、内存池等方面展开，并提供可运行的实战 Demo。

## 一、malloc() 基本概念

### 1. 函数原型

`malloc()` 定义在 `<stdlib.h>` 头文件中。

```
#include <stdlib.h>

void *malloc(size_t size);
```

| 参数   | 类型     | 说明                                |
| ------ | -------- | ----------------------------------- |
| `size` | `size_t` | 申请的内存字节数                    |
| 返回值 | `void *` | 成功返回内存首地址，失败返回 `NULL` |

例如：

```
int *p = malloc(sizeof(int));
```

这段代码表示：

1. `sizeof(int)` 计算 `int` 类型占用的字节数。
2. `malloc()` 申请对应大小的内存。
3. 返回申请到的内存首地址。
4. 将地址赋值给 `int *p`。
5. 此时 `*p` 尚未初始化，不能直接读取其值。

假设 `int` 占 4 字节，则申请 4 字节内存。

### 2. malloc() 的内存结构

![alt text](/images/C/malloc/image.png)

`malloc()` 返回的内存满足适用于普通对象类型的对齐要求，但它不会告诉你分配了多少个元素，也不会自动管理生命周期。

## 二、基础 Demo：动态申请一个整数

```
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int *p = malloc(sizeof(*p));

    if (p == NULL) {
        perror("malloc failed");
        return EXIT_FAILURE;
    }

    *p = 100;

    printf("Value: %d\n", *p);
    printf("Address: %p\n", (void *)p);
    printf("Size: %zu bytes\n", sizeof(*p));

    free(p);
    p = NULL;

    return EXIT_SUCCESS;
}
```

运行结果（地址每次可能不同）：

```
Value: 100
Address: 0x600003a24010
Size: 4 bytes
```

这里需要理解几个关键语法。

| 表达式               | 含义                    |
| -------------------- | ----------------------- |
| `int *p`             | 声明一个指向 int 的指针 |
| `malloc(sizeof(*p))` | 申请一个 int 大小的内存 |
| `p`                  | 内存首地址              |
| `*p`                 | 解引用，访问指向的整数  |
| `&p`                 | 获取指针变量自身的地址  |
| `free(p)`            | 释放指针指向的动态内存  |

### 为什么 malloc() 不需要强制类型转换？

```
int *p = malloc(sizeof(int));
```

因为 `malloc()` 返回 `void *`，在 C 语言中可以隐式转换成对象指针类型。

不推荐这样写：

```
int *p = (int *)malloc(sizeof(int));
```

虽然合法，但在 C 语言中通常没有必要。

注意，C++ 的规则不同，不能直接套用。

## 三、实战 Demo 1：动态整数数组

这是 `malloc()` 最常见的应用。

```
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>

int main(void) {
    size_t count;

    printf("请输入数组长度: ");

    if (scanf("%zu", &count) != 1 || count == 0) {
        fprintf(stderr, "Invalid count\n");
        return EXIT_FAILURE;
    }

    // 防止乘法溢出
    if (count > SIZE_MAX / sizeof(int)) {
        fprintf(stderr, "Array too large\n");
        return EXIT_FAILURE;
    }

    int *numbers = malloc(count * sizeof(*numbers));

    if (numbers == NULL) {
        perror("malloc");
        return EXIT_FAILURE;
    }

    // 初始化数组
    for (size_t i = 0; i < count; i++) {
        numbers[i] = (int)(i % 1000);
    }

    // 输出数组
    for (size_t i = 0; i < count; i++) {
        printf("%d ", numbers[i]);
    }

    printf("\n");

    free(numbers);
    numbers = NULL;

    return EXIT_SUCCESS;
}
```

示例：

```
请输入数组长度: 5
0 1 2 3 4
```

这里 `numbers[i]` 实际上等价于：

```
*(numbers + i)
```

例如：

```
numbers[2] = 30;

// 等价于
*(numbers + 2) = 30;
```

注意，指针加 2 不是增加 2 字节，而是前进两个 `int` 元素的距离。

假设 `sizeof(int) == 4`，实际前进 8 字节。

## 四、实战 Demo 2：动态创建结构体对象

在业务开发中，结构体类似于 C#、Java 中的数据对象，但 C 结构体没有内置构造函数。

例如创建用户对象：

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct {
    int id;
    char name[64];
    int age;
} User;

User *user_create(int id, const char *name, int age) {
    if (name == NULL || strlen(name) >= sizeof(((User *)0)->name)) {
        return NULL;
    }

    User *user = malloc(sizeof(*user));

    if (user == NULL) {
        return NULL;
    }

    user->id = id;
    strcpy(user->name, name);
    user->age = age;

    return user;
}

void user_destroy(User *user) {
    free(user);
}

int main(void) {
    User *user = user_create(1001, "Alice", 25);

    if (user == NULL) {
        fprintf(stderr, "Create user failed\n");
        return EXIT_FAILURE;
    }

    printf("ID: %d\n", user->id);
    printf("Name: %s\n", user->name);
    printf("Age: %d\n", user->age);

    user_destroy(user);

    return EXIT_SUCCESS;
}
```

输出：

```
ID: 1001
Name: Alice
Age: 25
```

这里有两个值得注意的知识点。

第一，`->` 运算符。

```
user->id
```

等价于：

```
(*user).id
```

因为 `user` 是结构体指针，而不是结构体变量。

第二，结构体创建和销毁函数。

```
User *user_create(...);
void user_destroy(User *user);
```

这种设计很常见。它让调用者不需要知道对象内部的内存申请细节，类似于面向对象语言中的工厂方法和资源释放方法。

上面的字符串长度检查也可以更直观地写成：

```
if (name == NULL || strlen(name) >= 64) {
    return NULL;
}
```

但使用 `sizeof` 的形式，在修改结构体字段大小时更容易维护。

## 五、实战 Demo 3：malloc() 实现单向链表

链表是理解动态内存分配的重要数据结构。

数组通常是一整块连续内存，而链表的节点可以分散在不同的内存位置，通过指针连接。

### 链表结构

![alt text](/images/C/malloc/image-1.png)

### 完整代码

```
#include <stdio.h>
#include <stdlib.h>

typedef struct Node {
    int data;
    struct Node *next;
} Node;

// 创建节点
Node *node_create(int value) {
    Node *node = malloc(sizeof(*node));

    if (node == NULL) {
        return NULL;
    }

    node->data = value;
    node->next = NULL;

    return node;
}

// 头部插入
int list_push_front(Node **head, int value) {
    Node *node = node_create(value);

    if (node == NULL) {
        return -1;
    }

    node->next = *head;
    *head = node;

    return 0;
}

// 遍历
void list_print(const Node *head) {
    while (head != NULL) {
        printf("%d -> ", head->data);
        head = head->next;
    }

    printf("NULL\n");
}

// 释放链表
void list_destroy(Node **head) {
    Node *current = *head;

    while (current != NULL) {
        Node *next = current->next;

        free(current);

        current = next;
    }

    *head = NULL;
}

int main(void) {
    Node *head = NULL;

    if (list_push_front(&head, 30) != 0 ||
        list_push_front(&head, 20) != 0 ||
        list_push_front(&head, 10) != 0) {
        list_destroy(&head);
        return EXIT_FAILURE;
    }

    list_print(head);

    list_destroy(&head);

    return EXIT_SUCCESS;
}
```

输出：

```
10 -> 20 -> 30 -> NULL
```

### 为什么使用 `Node **head`？

这里涉及二级指针：

```
int list_push_front(Node **head, int value)
```

因为函数需要修改调用者的 `head` 指针本身。

例如：

```
Node *head = NULL;

list_push_front(&head, 10);
```

传入 `&head`，相当于传入指针变量的地址。

在函数内部：

```
*head = node;
```

就能修改外部 `head` 的值。

如果只传 `Node *head`，修改的是指针参数的局部副本，不会改变调用者的 `head`。

## 六、实战 Demo 4：malloc() 实现二维动态数组

二维数组可以采用两种主要的动态分配方式。

### 方案一：单块连续内存

```
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>

int main(void) {
    size_t rows = 3;
    size_t cols = 4;

    if (rows > SIZE_MAX / cols ||
        rows * cols > SIZE_MAX / sizeof(int)) {
        return EXIT_FAILURE;
    }

    int *matrix = malloc(rows * cols * sizeof(*matrix));

    if (matrix == NULL) {
        return EXIT_FAILURE;
    }

    for (size_t i = 0; i < rows; i++) {
        for (size_t j = 0; j < cols; j++) {
            matrix[i * cols + j] = (int)(i * cols + j + 1);
        }
    }

    for (size_t i = 0; i < rows; i++) {
        for (size_t j = 0; j < cols; j++) {
            printf("%3d ", matrix[i * cols + j]);
        }

        printf("\n");
    }

    free(matrix);
    return EXIT_SUCCESS;
}
```

输出：

```
  1   2   3   4
  5   6   7   8
  9  10  11  12
```

访问公式：

![alt text](/images/C/malloc/image-2.png)


这种方案的优点是内存连续、缓存局部性较好、只需要一次 `malloc()` 和一次 `free()`。

### 方案二：二级指针分配

```
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    size_t rows = 3;
    size_t cols = 4;

    int **matrix = malloc(rows * sizeof(*matrix));

    if (matrix == NULL) {
        return EXIT_FAILURE;
    }

    for (size_t i = 0; i < rows; i++) {
        matrix[i] = malloc(cols * sizeof(**matrix));

        if (matrix[i] == NULL) {
            for (size_t j = 0; j < i; j++) {
                free(matrix[j]);
            }

            free(matrix);
            return EXIT_FAILURE;
        }
    }

    for (size_t i = 0; i < rows; i++) {
        for (size_t j = 0; j < cols; j++) {
            matrix[i][j] = (int)(i * cols + j);
        }
    }

    for (size_t i = 0; i < rows; i++) {
        for (size_t j = 0; j < cols; j++) {
            printf("%d ", matrix[i][j]);
        }

        printf("\n");
    }

    for (size_t i = 0; i < rows; i++) {
        free(matrix[i]);
    }

    free(matrix);

    return EXIT_SUCCESS;
}
```

这种方案的特点：

- 每一行可以单独申请内存。
- 每行可以拥有不同长度。
- 行与行之间不保证连续。
- 需要分别释放每一行，最后释放指针数组。

对于固定大小的数值矩阵，通常更推荐连续内存方案。

## 七、实战 Demo 5：malloc() 实现动态字符串

C 语言没有内置的可自动扩容字符串对象，字符串通常使用字符数组表示。

例如动态复制一个字符串：

```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

char *string_duplicate(const char *source) {
    if (source == NULL) {
        return NULL;
    }

    size_t length = strlen(source);

    char *result = malloc(length + 1);

    if (result == NULL) {
        return NULL;
    }

    memcpy(result, source, length + 1);

    return result;
}

int main(void) {
    char *name = string_duplicate("Hello C Language");

    if (name == NULL) {
        return EXIT_FAILURE;
    }

    printf("%s\n", name);
    printf("Length: %zu\n", strlen(name));

    free(name);

    return EXIT_SUCCESS;
}
```

输出：

```
Hello C Language
Length: 16
```

这里为什么需要 `length + 1`？

因为 C 字符串必须以 `'\0'` 结束。

例如：

```
"hello"
```

实际需要存储：

```
'h' 'e' 'l' 'l' 'o' '\0'
```

所以长度是 5，但需要 6 字节内存。

`memcpy(result, source, length + 1)` 也会复制结尾的 `'\0'`。

## 八、实战 Demo 6：malloc() 与 realloc() 实现动态数组

假设你需要一个类似 C# `List<int>` 的动态数组，容量不够时自动扩容。

```
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>

typedef struct {
    int *data;
    size_t size;
    size_t capacity;
} Vector;

int vector_init(Vector *v, size_t capacity) {
    if (capacity == 0) {
        capacity = 4;
    }

    if (capacity > SIZE_MAX / sizeof(*v->data)) {
        return -1;
    }

    v->data = malloc(capacity * sizeof(*v->data));

    if (v->data == NULL) {
        return -1;
    }

    v->size = 0;
    v->capacity = capacity;

    return 0;
}

int vector_push(Vector *v, int value) {
    if (v->size == v->capacity) {
        if (v->capacity > SIZE_MAX / 2) {
            return -1;
        }

        size_t new_capacity = v->capacity * 2;

        if (new_capacity > SIZE_MAX / sizeof(*v->data)) {
            return -1;
        }

        int *new_data = realloc(
            v->data,
            new_capacity * sizeof(*v->data)
        );

        if (new_data == NULL) {
            return -1;
        }

        v->data = new_data;
        v->capacity = new_capacity;
    }

    v->data[v->size++] = value;

    return 0;
}

void vector_destroy(Vector *v) {
    free(v->data);

    v->data = NULL;
    v->size = 0;
    v->capacity = 0;
}

int main(void) {
    Vector v = {0};

    if (vector_init(&v, 2) != 0) {
        return EXIT_FAILURE;
    }

    for (int i = 1; i <= 10; i++) {
        if (vector_push(&v, i * 10) != 0) {
            vector_destroy(&v);
            return EXIT_FAILURE;
        }
    }

    printf("Size: %zu\n", v.size);
    printf("Capacity: %zu\n", v.capacity);

    for (size_t i = 0; i < v.size; i++) {
        printf("%d ", v.data[i]);
    }

    printf("\n");

    vector_destroy(&v);

    return EXIT_SUCCESS;
}
```

输出：

```
Size: 10
Capacity: 16
10 20 30 40 50 60 70 80 90 100
```

扩容过程：

![alt text](/images/C/malloc/image-3.png)

这里 `realloc()` 有一个重要规则：

```
int *temp = realloc(v->data, new_size);
```

必须先用临时指针保存返回值。

不要直接写：

```
v->data = realloc(v->data, new_size);
```

因为一旦分配失败，返回 `NULL`，原来的内存仍然有效，但你已经丢失了原始地址，造成内存泄漏。

## 九、malloc() 的常见错误

### 1. 未初始化就读取

```
int *p = malloc(sizeof(int));

printf("%d\n", *p);  // 错误：读取未初始化的值
```

正确：

```
int *p = malloc(sizeof(int));

if (p != NULL) {
    *p = 100;
    printf("%d\n", *p);
    free(p);
}
```

### 2. 内存泄漏

```
void test(void) {
    int *p = malloc(100 * sizeof(int));

    // 忘记 free(p)
}
```

函数返回后，指针丢失，内存无法再通过它释放。

### 3. Use After Free

```
int *p = malloc(sizeof(int));

if (p != NULL) {
    *p = 10;
    free(p);

    // printf("%d", *p); // 错误
}
```

释放后的指针不能再用于访问原来的内存。

### 4. Double Free

```
free(p);
free(p); // 错误：重复释放
```

可以在释放后将指针置为 `NULL`，但这不能解决所有别名指针的问题。

### 5. 数组越界

```
int *p = malloc(5 * sizeof(int));

if (p != NULL) {
    p[5] = 100; // 错误：有效索引是 0~4
    free(p);
}
```

### 6. 错误使用 sizeof()

```
int *p = malloc(sizeof(p));
```

这里 `sizeof(p)` 得到的是指针自身的大小，不是 `int` 的大小。

正确写法：

```
int *p = malloc(sizeof(*p));
```

在 64 位系统上，指针通常占 8 字节，而 `int` 通常占 4 字节。虽然前一种写法可能恰好分配了足够内存，但语义不正确，换成其他类型就可能出现问题。

## 十、malloc()、calloc()、realloc()、free() 对比

| 函数                 | 作用                | 是否初始化       |
| -------------------- | ------------------- | ---------------- |
| `malloc(size)`       | 申请指定字节数      | 否               |
| `calloc(n, size)`    | 申请 n 个元素的内存 | 所有字节清零     |
| `realloc(ptr, size)` | 调整内存大小        | 新增部分不初始化 |
| `free(ptr)`          | 释放内存            | 不适用           |

### malloc() 申请的内存一定在堆上吗？

从 C 语言标准角度来说，`malloc()` 获取的是动态分配的存储空间，标准并没有强制要求它必须位于某个名为“堆”的内存区域。

在常见的 Linux/macOS 实现中，内存分配器可能使用堆区域，也可能通过 `mmap()` 等机制获得内存。

因此，“malloc 在堆上分配内存”是常见的简化说法，而不是所有平台都必须遵守的实现细节。

### malloc() 与栈内存有什么区别？

```
void test(void) {
    int a = 10;

    int *p = malloc(sizeof(int));

    if (p == NULL) {
        return;
    }

    *p = 20;

    free(p);
}
```

- `a` 是具有自动存储期的局部变量，通常位于栈中。
- `p` 本身是局部指针变量，通常也位于栈中。
- `p` 指向的内存由 `malloc()` 动态分配。
- 函数返回时，局部变量的生命周期结束。
- 动态分配的内存不会因为函数返回就自动释放。

## 十一、如何检测 malloc() 相关的内存错误？

例如：macOS + CLion，可以使用 Clang 的 AddressSanitizer 检查内存错误。

例如：

```
clang -g -O1 \
  -fsanitize=address \
  -fno-omit-frame-pointer \
  main.c -o main
```

执行：

```
./main
```

AddressSanitizer 可以帮助检测：

- Heap Buffer Overflow（堆缓冲区越界）
- Use After Free（释放后访问）
- Double Free（重复释放）
- 部分内存泄漏（取决于平台及运行时支持）

如果使用 CMake，可以为调试构建添加：

```
target_compile_options(fork_demo PRIVATE
    -fsanitize=address
    -fno-omit-frame-pointer
)

target_link_options(fork_demo PRIVATE
    -fsanitize=address
)
```

这里 `fork_demo` 替换成实际 CMake target 名称。