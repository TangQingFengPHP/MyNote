### 简介

索引器（`Indexer`）是 `C#` 中的一种特殊属性，它允许类或结构体像数组一样使用索引语法（例如 `obj[0]`）来访问或修改对象内部的成员。简单来说，它将对象的实例视为“可索引的集合”，提供类似于数组的访问方式。

* 核心特性：

    * 类似于属性（`Property`），但带有参数（通常是索引值，如整数或字符串）。

    * 支持 `get` 和 `set` 访问器，与属性类似。

    * 可以重载（`overload`），允许不同类型的索引参数。

    * 语法：`public 类型 this[参数类型 参数名] { get { ... } set { ... } }`

* 适用范围：类、结构体、接口。不能在委托或枚举中使用。

索引器本质上是方法（`get/set`），但编译器将其转换为特殊的属性调用，隐藏了底层实现细节。

语法结构：

```csharp
public <返回类型> this[<参数类型> index]
{
    get { ... }
    set { ... }
}
```

* `this[...]` 表示作用于当前对象实例。

* 可以有多个索引参数（如二维索引）。

### 为什么使用索引器？

在面向对象编程中，直接暴露内部数组或集合（如 `public int[] Data { get; }`）会破坏封装性（客户端可随意修改）。索引器提供受控访问：

* 封装性：隐藏内部存储（如 `List、Dictionary`），允许验证、转换或缓存逻辑。

* 直观性：代码更像数组操作，提升可读性（e.g., `cache["key"] = value`; 而非 `cache.Set("key", value)`;）。

* 适用场景：

    * 模拟数组/集合（如自定义列表、矩阵）。

    * 字典式访问（字符串键）。
    
    * 多维数据（如图像像素访问）。

    * 与 `LINQ` 或 `foreach` 集成（需实现 `IEnumerable`）。

相比普通方法，索引器更简洁；相比属性，它支持参数化访问。

### 基本示例

```csharp
public class MyList
{
    private string[] _data = new string[5];

    public string this[int index]
    {
        get => _data[index];
        set => _data[index] = value;
    }
}
```

使用方式：

```csharp
var list = new MyList();
list[0] = "Hello";
list[1] = "World";

Console.WriteLine(list[0]); // 输出：Hello
Console.WriteLine(list[1]); // 输出：World
```

看起来就像数组访问，但其实内部是属性访问。

### 索引器与属性的关系

| 对比项   | 属性 (Property) | 索引器 (Indexer)   |
| -------- | --------------- | ------------------ |
| 名称     | 有名字          | 名为 `this`        |
| 参数     | 无参数          | 有一个或多个参数   |
| 调用方式 | `obj.Property`  | `obj[index]`       |
| 典型用途 | 访问字段值      | 访问集合或映射内容 |

### 索引器的底层机制

编译后：

```csharp
obj[index]
```

其实会被编译为：

```csharp
obj.get_Item(index);    // 读取
obj.set_Item(index, x); // 赋值
```

也就是说，索引器就是名为 `Item` 的一对 `get/set` 方法（`CLR` 级别）。

### 不同类型的索引器

#### 整数索引器

```csharp
public class IntArrayWrapper
{
    private int[] _array;

    public IntArrayWrapper(int size)
    {
        _array = new int[size];
    }

    public int this[int index]
    {
        get => _array[index];
        set => _array[index] = value;
    }

    public int Length => _array.Length;
}

// 使用
var wrapper = new IntArrayWrapper(5);
wrapper[0] = 10;
wrapper[1] = 20;
Console.WriteLine(wrapper[0]); // 输出: 10
```

#### 字符串索引器

```csharp
public class DictionaryWrapper
{
    private Dictionary<string, string> _dictionary = new Dictionary<string, string>();

    public string this[string key]
    {
        get
        {
            _dictionary.TryGetValue(key, out string value);
            return value;
        }
        set
        {
            _dictionary[key] = value;
        }
    }
}

// 使用
var dictWrapper = new DictionaryWrapper();
dictWrapper["name"] = "John";
dictWrapper["age"] = "30";

Console.WriteLine(dictWrapper["name"]); // 输出: John
```

#### 多参数索引器

```csharp
public class Matrix
{
    private double[,] _matrix;

    public Matrix(int rows, int columns)
    {
        _matrix = new double[rows, columns];
    }

    // 多参数索引器
    public double this[int row, int column]
    {
        get => _matrix[row, column];
        set => _matrix[row, column] = value;
    }

    public int Rows => _matrix.GetLength(0);
    public int Columns => _matrix.GetLength(1);
}

// 使用
var matrix = new Matrix(3, 3);
matrix[0, 0] = 1.0;
matrix[1, 1] = 2.0;
matrix[2, 2] = 3.0;

Console.WriteLine(matrix[1, 1]); // 输出: 2.0
```

#### 重载索引器

```csharp
public class MultiIndexCollection
{
    private List<string> _items = new List<string>();

    public void Add(string item) => _items.Add(item);

    // 整数索引器
    public string this[int index]
    {
        get => _items[index];
        set => _items[index] = value;
    }

    // 字符串索引器 - 通过名称查找
    public string this[string name]
    {
        get => _items.Find(item => item.StartsWith(name));
    }
}

// 使用
var collection = new MultiIndexCollection();
collection.Add("Apple");
collection.Add("Banana");
collection.Add("Cherry");

Console.WriteLine(collection[0]);    // 输出: Apple
Console.WriteLine(collection["B"]);  // 输出: Banana
```

### 高级用法

#### 只读索引器

```csharp
public class Settings
{
    private readonly Dictionary<string, string> _values = new();

    public string this[string key]
    {
        get => _values.TryGetValue(key, out var value) ? value : string.Empty;
        set => _values[key] = value;
    }
}
```

使用：

```csharp
var s = new Settings();
s["Language"] = "Chinese";
s["Theme"] = "Dark";

Console.WriteLine(s["Language"]); // Chinese
```

只写：

```csharp
public string this[int index]
{
    set => _data[index] = value;
}
```

#### 索引器可以被继承或重写

父类定义：

```csharp
public class Base
{
    public virtual string this[int index]
    {
        get => $"Base:{index}";
        set => Console.WriteLine($"Set Base[{index}]={value}");
    }
}
```

子类重写：

```csharp
public class Derived : Base
{
    public override string this[int index]
    {
        get => $"Derived:{index}";
        set => Console.WriteLine($"Set Derived[{index}]={value}");
    }
}
```

#### 接口中的索引器

```csharp
public interface IListContainer<T>
{
    T this[int index] { get; set; }
    int Count { get; }
}

public class MyList<T> : IListContainer<T>
{
    private List<T> _items = new List<T>();

    public T this[int index]
    {
        get => _items[index];
        set => _items[index] = value;
    }

    public int Count => _items.Count;

    public void Add(T item) => _items.Add(item);
}
```

### 实际应用示例

#### 配置管理器

```csharp
public class Configuration
{
    private readonly Dictionary<string, object> _settings = new Dictionary<string, object>();

    public object this[string key]
    {
        get => _settings.TryGetValue(key, out object value) ? value : null;
        set => _settings[key] = value;
    }

    public T Get<T>(string key, T defaultValue = default)
    {
        if (_settings.TryGetValue(key, out object value) && value is T typedValue)
        {
            return typedValue;
        }
        return defaultValue;
    }
}

// 使用
var config = new Configuration();
config["DatabaseConnection"] = "Server=localhost;Database=Test;";
config["Timeout"] = 30;

string connection = config.Get<string>("DatabaseConnection");
int timeout = config.Get<int>("Timeout");
```

#### 自定义集合类

```csharp
public class SmartCollection<T>
{
    private T[] _items;
    
    public SmartCollection(int size) => _items = new T[size];
    
    public T this[int index]
    {
        get => _items[index];
        set => _items[index] = value;
    }
    
    // 重载索引器
    public T this[string name] => FindByName(name);
    
    private T FindByName(string name)
    {
        // 根据名称查找逻辑...
    }
}
```

#### 数据访问层封装

```csharp
public class DataRepository
{
    private List<Customer> _customers = new();
    
    public Customer this[int id]
    {
        get => _customers.FirstOrDefault(c => c.Id == id);
    }
    
    public Customer this[string email]
    {
        get => _customers.FirstOrDefault(c => c.Email == email);
    }
}
```

### 索引器与其他特性结合

#### 索引器与泛型

```csharp
public class GenericCollection<T>
{
    private T[] _items = new T[10];
    
    public T this[int index]
    {
        get => _items[index];
        set => _items[index] = value;
    }
}
```

#### 索引器与模式匹配（C# 8.0+）

```csharp
if (collection is IIndexable<int, string> indexable)
{
    Console.WriteLine(indexable[0]);
}
```

#### 索引器与范围支持（C# 8.0+）

```csharp
public class RangeCollection
{
    private int[] _items = {1, 2, 3, 4, 5};
    
    public int[] this[Range range]
    {
        get => _items[range];
    }
}

// 使用
var collection = new RangeCollection();
int[] sub = collection[1..4]; // [2, 3, 4]
```

### 常见应用场景

| 场景                 | 示例                              |
| -------------------- | --------------------------------- |
| 模拟集合/字典访问    | `myDict[key]`                     |
| 操作二维数据         | `matrix[row, col]`                |
| 管理配置项           | `settings["Theme"]`               |
| 实现对象简洁访问接口 | `student["Name"]`、`api["token"]` |

