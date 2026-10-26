# C 语言 `calloc()` 详解与实战 Demo

`calloc()` 是 C 标准库提供的动态内存分配函数，主要用于在堆上分配一块连续内存，并将分配的所有字节初始化为 0。

它与 `malloc()`、`realloc()`、`free()` 一起构成 C 语言动态内存管理的核心。

如果你熟悉 C#、Java，可以先这样理解：

- `malloc()`：申请一块内存，但不初始化其内容。
- `calloc()`：申请一块内存，并将所有字节清零。
- `realloc()`：调整已经分配的内存大小。
- `free()`：释放动态分配的内存。

需要注意，C 语言的动态内存分配没有 C#、Java 那样的垃圾回收机制，程序员需要自己管理内存的生命周期。

## 一、calloc() 基本概念

### 1. 函数原型

定义在 `<stdlib.h>` 中：

```C
#include <stdlib.h>

void *calloc(size_t nmemb, size_t size);
```

参数说明：

| 参数    | 类型     | 说明                                |
| ------- | -------- | ----------------------------------- |
| `nmemb` | `size_t` | 需要分配的元素数量                  |
| `size`  | `size_t` | 每个元素占用的字节数                |
| 返回值  | `void *` | 成功返回内存首地址，失败返回 `NULL` |

分配的总字节数为：

Total Bytes = nmemb * size

例如：

```C
int *arr = calloc(5, sizeof(int));
```

假设当前平台 `sizeof(int) == 4`，则：

```
5 × 4 = 20 字节
```

这段代码申请了 20 字节的连续内存，并将其全部清零。

### 2. 内存结构示意图

![alt text](/images/C/image.png)

注意：`calloc()` 清零的是字节，并不意味着所有类型的对象都获得了符合 C 标准的“零值”。例如，标准不保证所有位为 0 的指针一定表示空指针。

## 二、基础 Demo：使用 calloc 创建动态数组

```C
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    size_t count = 5;

    // 动态申请 5 个 int 的内存
    int *numbers = calloc(count, sizeof(*numbers));

    if (numbers == NULL) {
        perror("calloc failed");
        return EXIT_FAILURE;
    }

    // 查看初始化后的值
    for (size_t i = 0; i < count; i++) {
        printf("numbers[%zu] = %d\n", i, numbers[i]);
    }

    // 修改数组
    for (size_t i = 0; i < count; i++) {
        numbers[i] = (int)(i + 1) * 10;
    }

    printf("After modification:\n");

    for (size_t i = 0; i < count; i++) {
        printf("%d ", numbers[i]);
    }

    printf("\n");

    // 释放内存
    free(numbers);
    numbers = NULL;

    return EXIT_SUCCESS;
}
```

运行结果：

```
numbers[0] = 0
numbers[1] = 0
numbers[2] = 0
numbers[3] = 0
numbers[4] = 0
After modification:
10 20 30 40 50
```

### 为什么推荐 `sizeof(*numbers)`？

下面两种写法都可以：

```C
int *numbers = calloc(5, sizeof(int));

int *numbers = calloc(5, sizeof(*numbers));
```

但第二种更值得推荐。

假设以后你修改变量类型：

```C
double *numbers = calloc(5, sizeof(*numbers));
```

就不需要再手动修改 `sizeof(int)`。

另外，这里的 `*numbers` 出现在 `sizeof` 中，通常不会真正读取指针指向的内存，而是获取其指向类型的大小。

## 三、calloc() 与 malloc() 的区别

这是面试和实际开发中都非常常见的问题。

| 对比     | malloc()           | calloc()                         |
| -------- | ------------------ | -------------------------------- |
| 参数     | 字节数             | 元素数量、元素大小               |
| 初始化   | 不初始化           | 所有字节清零                     |
| 返回值   | `void *`           | `void *`                         |
| 分配失败 | `NULL`             | `NULL`                           |
| 内存释放 | `free()`           | `free()`                         |
| 溢出处理 | 调用方自行检查乘法 | 检查元素数量与大小的乘积是否溢出 |

### Demo：对比初始化行为

```C
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    int *a = malloc(5 * sizeof(int));
    int *b = calloc(5, sizeof(int));

    if (a == NULL || b == NULL) {
        free(a);
        free(b);
        return EXIT_FAILURE;
    }

    // malloc 分配的 int 未初始化，不能直接读取
    // printf("%d\n", a[0]); // 不要这样做

    // calloc 分配的 int 初始化为 0
    for (int i = 0; i < 5; i++) {
        printf("%d ", b[i]);
    }

    free(a);
    free(b);

    return 0;
}
```

这里特别强调：

`malloc()` 分配的内存不是一定包含随机数，也不是一定包含 0，而是内容未初始化，不能依赖其值。

`calloc()` 在需要零初始化时通常更方便。

### calloc() 一定比 malloc() 慢吗？

不一定。

虽然 `calloc()` 需要保证内存清零，但现代操作系统和内存分配器可能使用按需映射的零页等优化。

因此：

- 小块内存分配，两者性能差距可能很小。
- 大块内存分配，实际性能取决于分配器和访问模式。
- `malloc() + memset()` 不一定比 `calloc()` 快。
- 需要零初始化时，优先考虑 `calloc()`，不必为了猜测性能而手动清零。

## 四、实战 Demo 1：动态结构体数组

假设你开发一个学生管理系统，需要根据用户输入动态创建学生数组。

```C
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int id;
    char name[50];
    double score;
} Student;

int main(void) {
    size_t count = 3;

    Student *students = calloc(count, sizeof(*students));

    if (students == NULL) {
        perror("calloc");
        return EXIT_FAILURE;
    }

    students[0] = (Student){
        .id = 1,
        .name = "Alice",
        .score = 95.5
    };

    students[1] = (Student){
        .id = 2,
        .name = "Bob",
        .score = 88.0
    };

    students[2] = (Student){
        .id = 3,
        .name = "Charlie",
        .score = 92.5
    };

    for (size_t i = 0; i < count; i++) {
        printf(
            "ID: %d, Name: %s, Score: %.1f\n",
            students[i].id,
            students[i].name,
            students[i].score
        );
    }

    free(students);
    students = NULL;

    return EXIT_SUCCESS;
}
```

输出：

```
ID: 1, Name: Alice, Score: 95.5
ID: 2, Name: Bob, Score: 88.0
ID: 3, Name: Charlie, Score: 92.5
```

这里的 `students` 指向一块连续的结构体数组内存。

![alt text](/images/C/image-1.png)

这与 C# 中的 `Student[]` 在使用体验上有一定相似性，但 C 语言需要手动释放内存。

## 五、实战 Demo 2：二维动态数组

在实际项目中，经常需要动态创建二维数组，例如：

- 图像像素矩阵
- 游戏地图
- 数学矩阵
- 表格数据

推荐的一种方式是申请一块连续内存。

```C
#include <stdio.h>
#include <stdlib.h>

int main(void) {
    size_t rows = 3;
    size_t cols = 4;

    // 动态二维数组，指向含 4 个 int 的数组
    int (*matrix)[4] = calloc(rows, sizeof(*matrix));

    if (matrix == NULL) {
        perror("calloc");
        return EXIT_FAILURE;
    }

    for (size_t i = 0; i < rows; i++) {
        for (size_t j = 0; j < cols; j++) {
            matrix[i][j] = (int)(i * cols + j + 1);
        }
    }

    for (size_t i = 0; i < rows; i++) {
        for (size_t j = 0; j < cols; j++) {
            printf("%3d ", matrix[i][j]);
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

### `int (*matrix)[4]` 是什么？

这是一个指向包含 4 个 int 元素的数组的指针。

与下面的类型不同：

```
int *p;          // 指向一个 int
int *p2[4];      // 4 个 int* 组成的数组
int (*p3)[4];    // 指向 int[4] 数组的指针
```

其中 `matrix[i][j]` 能正确访问二维数组，是因为编译器知道每行包含 4 个 `int`。

如果行列数都需要运行时确定，在支持可变长度数组的 C99 编译器中，可以使用：

```C
size_t rows = 3;
size_t cols = 4;

int (*matrix)[cols] = calloc(rows, sizeof(*matrix));
```

不过，可变长度数组是可选特性，并非所有编译器都支持。

## 六、实战 Demo 3：动态扩容数组（calloc + realloc）

这是实际开发中更有价值的例子。

实现一个类似 C# `List<int>` 的动态数组。

```C
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>

typedef struct {
    int *data;
    size_t size;
    size_t capacity;
} IntList;

// 初始化
int list_init(IntList *list, size_t capacity) {
    if (capacity == 0) {
        capacity = 4;
    }

    list->data = calloc(capacity, sizeof(*list->data));

    if (list->data == NULL) {
        return -1;
    }

    list->size = 0;
    list->capacity = capacity;

    return 0;
}

// 添加元素
int list_add(IntList *list, int value) {
    if (list->size == list->capacity) {
        if (list->capacity > SIZE_MAX / 2) {
            return -1;
        }

        size_t new_capacity = list->capacity * 2;

        if (new_capacity > SIZE_MAX / sizeof(*list->data)) {
            return -1;
        }

        int *new_data = realloc(
            list->data,
            new_capacity * sizeof(*list->data)
        );

        if (new_data == NULL) {
            return -1;
        }

        list->data = new_data;
        list->capacity = new_capacity;
    }

    list->data[list->size++] = value;
    return 0;
}

// 释放
void list_free(IntList *list) {
    free(list->data);

    list->data = NULL;
    list->size = 0;
    list->capacity = 0;
}

int main(void) {
    IntList list = {0};

    if (list_init(&list, 2) != 0) {
        return EXIT_FAILURE;
    }

    for (int i = 1; i <= 10; i++) {
        if (list_add(&list, i * 10) != 0) {
            list_free(&list);
            return EXIT_FAILURE;
        }
    }

    printf("Size: %zu\n", list.size);
    printf("Capacity: %zu\n", list.capacity);

    for (size_t i = 0; i < list.size; i++) {
        printf("%d ", list.data[i]);
    }

    printf("\n");

    list_free(&list);
    return EXIT_SUCCESS;
}
```

输出：

```
Size: 10
Capacity: 16
10 20 30 40 50 60 70 80 90 100
```

### 扩容过程

![alt text](/images/C/image-2.png)

这里需要注意：

`realloc()` 扩容后，新增的内存不会自动清零。

例如从容量 4 扩容到 8，新增的 4 个元素内容未初始化。

但这个 Demo 不受影响，因为每次添加元素时都会先写入对应位置。

另外，`realloc()` 可能返回不同的内存地址，因此必须使用它返回的新指针，而不能继续使用旧指针。

## 七、实战 Demo 4：实现一个简单的内存池

假设你需要管理大量固定大小的对象，例如：

- 网络连接上下文
- 游戏实体
- 任务节点
- 消息队列节点

可以使用 `calloc()` 一次性申请多个对象，减少反复调用内存分配器的次数。

```C
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int id;
    int active;
    char buffer[256];
} Connection;

typedef struct {
    Connection *items;
    size_t capacity;
} ConnectionPool;

int pool_init(ConnectionPool *pool, size_t capacity) {
    pool->items = calloc(capacity, sizeof(Connection));

    if (pool->items == NULL) {
        return -1;
    }

    pool->capacity = capacity;
    return 0;
}

Connection *pool_acquire(ConnectionPool *pool) {
    for (size_t i = 0; i < pool->capacity; i++) {
        if (!pool->items[i].active) {
            pool->items[i].active = 1;
            pool->items[i].id = (int)i + 1;

            return &pool->items[i];
        }
    }

    return NULL;
}

void pool_release(Connection *conn) {
    conn->active = 0;
}

void pool_destroy(ConnectionPool *pool) {
    free(pool->items);
    pool->items = NULL;
    pool->capacity = 0;
}

int main(void) {
    ConnectionPool pool = {0};

    if (pool_init(&pool, 3) != 0) {
        return EXIT_FAILURE;
    }

    Connection *c1 = pool_acquire(&pool);
    Connection *c2 = pool_acquire(&pool);

    if (c1 != NULL && c2 != NULL) {
        printf("Connection %d acquired\n", c1->id);
        printf("Connection %d acquired\n", c2->id);

        pool_release(c1);
        printf("Connection %d released\n", c1->id);
    }

    pool_destroy(&pool);

    return EXIT_SUCCESS;
}
```

输出：

```
Connection 1 acquired
Connection 2 acquired
Connection 1 released
```

这个 Demo 是一个简化的固定容量对象池。

实际生产环境中还需要考虑线程安全、对象重置、资源清理以及避免释放后继续使用对象等问题。

特别是 `pool_release()` 只是把对象标记为空闲，并没有清空 `buffer`。如果对象可能存放敏感数据，重新分配时还需要做好数据清理。

## 八、calloc() 的常见错误

### 1. 忘记 free()，造成内存泄漏

错误：

```C
void test(void) {
    int *p = calloc(100, sizeof(int));

    // 使用 p ...

    // 忘记 free(p)
}
```

每次调用都会申请内存，但没有释放。

正确：

```C
void test(void) {
    int *p = calloc(100, sizeof(int));

    if (p == NULL) {
        return;
    }

    // 使用 p ...

    free(p);
}
```

### 2. 释放后继续使用

错误：

```C
int *p = calloc(5, sizeof(int));

free(p);

p[0] = 100;  // Use After Free，未定义行为
```

正确：

```C
free(p);
p = NULL;
```

注意：设置 `p = NULL` 只能防止通过这个变量继续访问，不能让其他指向同一块内存的指针自动失效。

### 3. 重复释放

错误：

```C
free(p);
free(p);  // Double Free，未定义行为
```

正确：

```C
free(p);
p = NULL;

free(p);  // free(NULL) 是安全的
```

### 4. 数组越界

错误：

```C
int *p = calloc(5, sizeof(int));

p[5] = 100;  // 越界
```

合法索引只有 `0` 到 `4`。

### 5. 分配 0 字节

```C
int *p = calloc(0, sizeof(int));
```

这是需要特别注意的情况。

C 标准允许零大小分配具有实现定义的行为，返回值可能是 `NULL`，也可能是一个不能用于访问元素的非空指针。

因此，实际开发中建议显式处理：

```C
if (count == 0) {
    return;
}
```

### 6. 整数溢出

下面的写法存在风险：

```C
size_t count = /* 外部输入 */;

int *p = malloc(count * sizeof(int));
```

如果乘法发生无符号整数溢出，可能申请到比预期更小的内存，导致后续越界写入。

相比之下：

```C
int *p = calloc(count, sizeof(int));
```

`calloc()` 会检查元素数量与元素大小相乘是否超出可表示的范围，并在无法满足分配时返回 `NULL`。

当然，分配成功也不代表可以忽略数组下标和实际容量检查。

## 九、malloc、calloc、realloc、free 怎么选？

| 场景                       | 推荐函数             |
| -------------------------- | -------------------- |
| 申请一块内存，准备完整写入 | `malloc()`           |
| 创建初始值为 0 的整数数组  | `calloc()`           |
| 创建需要字节清零的缓冲区   | `calloc()`           |
| 扩大或缩小已有动态数组     | `realloc()`          |
| 释放动态内存               | `free()`             |
| 创建并初始化复杂对象       | 分配内存后显式初始化 |

最后补充一个与 C#、Java 很不一样的知识点：`calloc()` 只负责分配和清零内存，不负责执行任何构造函数，也不负责自动释放对象。