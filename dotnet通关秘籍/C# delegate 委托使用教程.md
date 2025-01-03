### 什么是委托？ 

委托是定义方法签名的引用类型数据类型，可以定义委托的变量，就像其他数据类型一样，可以引用与委托具有相同签名的任何方法。

它允许方法作为参数传递，并允许事件驱动编程。它们提供了一种以类型安全的方式封装方法引用的方法。

* 委托是一种类型，类似于 `C++` 的函数指针，但更安全和灵活。

* 委托可以存储对方法的引用（或者多个方法）。

* 委托是实现事件和回调的基础。

### 为什么使用委托？

1. 类型安全：委托提供一种类型安全的方法来处理方法引用，确保方法签名与委托签名相匹配。

2. 灵活性：它们允许将方法作为参数传递，从而实现动态方法调用和回调机制。

3. 事件处理：委托是 C# 中事件处理的基础

### 创建和使用委托

#### 示例一

1. 定义委托

```csharp
// 定义一个委托类型
public delegate void PrintDelegate(string message);
```

2. 定义方法

```csharp
public class Printer
{
    public void PrintMessage(string message)
    {
        Console.WriteLine("Message: " + message);
    }

    public void PrintUppercase(string message)
    {
        Console.WriteLine("Uppercase: " + message.ToUpper());
    }
}
```

3. 使用委托

```csharp
class Program
{
    static void Main(string[] args)
    {
        // 实例化委托
        Printer printer = new Printer();
        PrintDelegate printDelegate = new PrintDelegate(printer.PrintMessage);

        // 调用委托
        printDelegate("Hello, Delegates!");

        // 替换委托目标
        printDelegate = printer.PrintUppercase;
        printDelegate("Hello again!");
    }
}
```

**输出**：

```makefile
Message: Hello, Delegates!
Uppercase: HELLO AGAIN!
```

#### 示例二

1. 定义委托

```csharp
public delegate void MyDelegate(string msg);
```

2. 定义方法

```csharp
// 方法1：实例化委托，把方法名作为参数传进去
MyDelegate del = new MyDelegate(MethodA);

// 方法2：直接把方法名赋值给委托的实例
MyDelegate del = MethodA; 

// 方法3：把Lambda表达式赋值给委托的实例
MyDelegate del = (string msg) => Console.WriteLine(msg);

// 目标方法
static void MethodA(string message)
{
    Console.WriteLine(message);
}
```

3. 使用委托

```csharp
// 方法1：使用委托实例名.Invoke调用目标方法
del.Invoke("Hello World!");

// 方法2：直接使用委托实例名作为方法调用
del("Hello World!");
```

### 将委托作为参数传递

> 方法可以有一个委托类型的参数，也就是回调函数

```csharp
public delegate void MyDelegate(string msg); //declaring a delegate

class Program
{
    static void Main(string[] args)
    {
        MyDelegate del = ClassA.MethodA;
        InvokeDelegate(del);

        del = ClassB.MethodB;
        InvokeDelegate(del);

        del = (string msg) => Console.WriteLine("Called lambda expression: " + msg);
        InvokeDelegate(del);
    }

    static void InvokeDelegate(MyDelegate del) // MyDelegate type parameter
    {
        del("Hello World");
    }
}

class ClassA
{
    static void MethodA(string message)
    {
        Console.WriteLine("Called ClassA.MethodA() with parameter: " + message);
    }
}

class ClassB
{
    static void MethodB(string message)
    {
        Console.WriteLine("Called ClassB.MethodB() with parameter: " + message);
    }
}
```

### 多播代理

委托可以指向多个方法，指向多个方法的委托称为多播委托。`+` 或 `+=` 运算符将函数添加到调用列表中，`-` 和 `-=` 运算符将其删除

> 如果委托返回一个值，那么在调用多播委托时将返回最后分配的目标方法的值

**多播无返回值的示例**

```csharp
public delegate void MyDelegate(string msg); //declaring a delegate

class Program
{
    static void Main(string[] args)
    {
        MyDelegate del1 = ClassA.MethodA;
        MyDelegate del2 = ClassB.MethodB;

        MyDelegate del = del1 + del2; // combines del1 + del2
        del("Hello World");

        MyDelegate del3 = (string msg) => Console.WriteLine("Called lambda expression: " + msg);
        del += del3; // combines del1 + del2 + del3
        del("Hello World");

        del = del - del2; // removes del2
        del("Hello World");

        del -= del1 // removes del1
        del("Hello World");
    }
}

class ClassA
{
    static void MethodA(string message)
    {
        Console.WriteLine("Called ClassA.MethodA() with parameter: " + message);
    }
}

class ClassB
{
    static void MethodB(string message)
    {
        Console.WriteLine("Called ClassB.MethodB() with parameter: " + message);
    }
}
```

**多播有返回值的示例**

```csharp
public delegate int MyDelegate(); //declaring a delegate

class Program
{
    static void Main(string[] args)
    {
        MyDelegate del1 = ClassA.MethodA;
        MyDelegate del2 = ClassB.MethodB;

        MyDelegate del = del1 + del2; 
        Console.WriteLine(del());// returns 200
    }
}

class ClassA
{
    static int MethodA()
    {
        return 100;
    }
}

class ClassB
{
    static int MethodB()
    {
        return 200;
    }
}
```

### 泛型委托

泛型委托的定义方式与委托相同，但使用泛型类型参数或返回类型，设置目标方法时必须指定泛型类型。

```csharp
public delegate T add<T>(T param1, T param2); // generic delegate

class Program
{
    static void Main(string[] args)
    {
        add<int> sum = Sum;
        Console.WriteLine(sum(10, 20));

        add<string> con = Concat;
        Console.WriteLine(conct("Hello ","World!!"));
    }

    public static int Sum(int val1, int val2)
    {
        return val1 + val2;
    }

    public static string Concat(string str1, string str2)
    {
        return str1 + str2;
    }
}
```

### `Func` 委托

**特性**

* 用于有返回值的方法。

* 最后一个泛型参数是返回类型。

* 支持 0 到 16 个输入参数。

* `Func` 委托不允许 `ref` 和 `out` 参数

* `Func` 委托类型可以与匿名方法或 `lambda` 表达式一起使用

`Func` 是包含在 `System` 命名空间中的泛型委托。它有零个或多个输入参数和一个输出参数，最后一个参数被认为是输出参数。

可以包含 0 到 16 个不同类型的输入参数，但是它必须包含一个用于结果的输出参数。

`Func` 委托签名

```csharp
// 尖括号 <> 中的最后一个参数被视为返回类型，其余参数被视为输入参数类型
namespace System
{    
    public delegate TResult Func<in T, out TResult>(T arg);
}
```

#### 普通方法赋值给 `Func` 委托

```csharp
class Program
{
    static int Sum(int x, int y)
    {
        return x + y;
    }

    static void Main(string[] args)
    {
        Func<int, int, int> add = Sum;

        int result = add(10, 10);

        Console.WriteLine(result); // 输出20
    }
}
```

#### `Lambda` 表达式赋值给 `Func` 委托

```csharp
Func<int> getRandomNumber = () => new Random().Next(1, 100);

Func<int, int, int> Sum = (x, y) => x + y;
```

### `Action` 委托

**特性**

* 用于无返回值的方法。

* 支持 0 到 16 个输入参数。

* `Action` 委托类型可以与匿名方法或 `lambda` 表达式一起使用

`Action` 委托是 `System` 命名空间中定义的委托类型，与 `Func` 委托相同，只是 `Action` 委托不返回值。即 `Action` 委托可以与具有 `void` 返回类型的方法一起使用。

#### 定义类似于 `Action` 的委托

```csharp
public delegate void Print(int val);

static void ConsolePrint(int i)
{
    Console.WriteLine(i);
}

static void Main(string[] args)
{           
    Print prnt = ConsolePrint;
    prnt(10); // 输出10
}
```

#### 使用 `Action` 委托代替上面的

```csharp
static void ConsolePrint(int i)
{
    Console.WriteLine(i);
}

static void Main(string[] args)
{
    Action<int> printActionDel = ConsolePrint;
    printActionDel(10);
}
```

#### 匿名方法赋值给 `Action` 委托

```csharp
static void Main(string[] args)
{
    Action<int> printActionDel = delegate(int i)
                                {
                                    Console.WriteLine(i);
                                };

    printActionDel(10);
}
```

#### `Lambda` 表达式赋值给 `Action` 委托

```csharp
static void Main(string[] args)
{

    Action<int> printActionDel = i => Console.WriteLine(i);
       
    printActionDel(10);
}
```

### `Predicate` (谓词) 委托

谓词是类似于 `Func` 和 `Action` 委托的委托，它表示一个包含一组条件的方法，并检查传递的参数是否满足这些条件。谓词委托方法必须接受一个输入参数并返回一个布尔值：`true` 或 `false` 。

**`Predicate`签名**

#### 普通方法赋值给谓词委托

```csharp
static bool IsUpperCase(string str)
{
    return str.Equals(str.ToUpper());
}

static void Main(string[] args)
{
    Predicate<string> isUpper = IsUpperCase;

    bool result = isUpper("hello world!!");

    Console.WriteLine(result);
}
```

#### 匿名方法赋值给谓词委托

```csharp
static void Main(string[] args)
{
    Predicate<string> isUpper = delegate(string s) { return s.Equals(s.ToUpper());};
    bool result = isUpper("hello world!!");
}
```

#### `Lambda` 表达式赋值给谓词委托

```csharp
static void Main(string[] args)
{
    Predicate<string> isUpper = s => s.Equals(s.ToUpper());
    bool result = isUpper("hello world!!");
}
```

#### `Action` 和 `Func` 与 `LINQ` 的结合

> 筛选和映射操作：

```csharp
List<int> numbers = new List<int> { 1, 2, 3, 4, 5, 6 };

// 使用 Func 进行映射
List<int> squaredNumbers = numbers.Select(x => x * x).ToList();
Console.WriteLine("Squared Numbers: " + string.Join(", ", squaredNumbers));

// 使用 Predicate 或 Func 进行过滤
List<int> evenNumbers = numbers.Where(x => x % 2 == 0).ToList();
Console.WriteLine("Even Numbers: " + string.Join(", ", evenNumbers));
```

### 匿名方法

`C#` 中的匿名方法可以使用 `delegate` 关键字定义，并可以分配给委托类型的变量。

**普通用法**

```csharp
public delegate void Print(int value);

static void Main(string[] args)
{
    Print print = delegate(int val) { 
        Console.WriteLine("Inside Anonymous method. Value: {0}", val); 
    };

    print(100);
}
```

**匿名方法可以访问外部函数中定义的变量**

```csharp
public delegate void Print(int value);

static void Main(string[] args)
{
    int i = 10;
    
    Print prnt = delegate(int val) {
        val += i;
        Console.WriteLine("Anonymous method: {0}", val); 
    };

    prnt(100);
}
```

**匿名方法作为参数**

```csharp
public delegate void Print(int value);

class Program
{
    public static void PrintHelperMethod(Print printDel,int val)
    { 
        val += 10;
        printDel(val);
    }

    static void Main(string[] args)
    {
        PrintHelperMethod(delegate(int val) { Console.WriteLine("Anonymous method: {0}", val); }, 100);
    }
}
```

**匿名方法用作事件处理程序**

```csharp
saveButton.Click += delegate(Object o, EventArgs e)
{ 
    System.Windows.Forms.MessageBox.Show("Save Successfully!"); 
};
```