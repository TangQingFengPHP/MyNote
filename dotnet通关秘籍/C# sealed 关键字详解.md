### 简介

`sealed` 关键字在 `C#` 中用于阻止继承和重写，通常用于类或方法，以增强代码的安全性和稳定性。

### sealed 用于类

当一个类被 `sealed` 修饰时，该类不能被继承。这样可以防止其他类扩展它的功能，从而保护类的实现。

```csharp
sealed class MyClass
{
    public void Show()
    {
        Console.WriteLine("Hello from MyClass");
    }
}
```

不能继承 `MyClass`，否则会编译报错

```csharp
class DerivedClass : MyClass // 报错：无法从密封类 'MyClass' 派生
{
}
```

适用场景

* 安全性：防止恶意或意外的继承，保护关键逻辑不被更改。

* 优化性能：密封类可以让 `JIT`（`Just-In-Time` 编译器）优化方法调用，提高执行速度。

### sealed 用于方法

如果一个方法在基类中是 `virtual` 或 `override`，可以使用 `sealed` 防止子类进一步重写它。

```csharp
class BaseClass
{
    public virtual void Show()
    {
        Console.WriteLine("BaseClass Show");
    }
}

class DerivedClass : BaseClass
{
    public sealed override void Show()
    {
        Console.WriteLine("DerivedClass Show");
    }
}

class SubDerivedClass : DerivedClass
{
    // 报错：无法重写密封的方法 "Show"
    // public override void Show() { }
}
```

适用场景

* 防止进一步重写：如果一个方法被 `sealed` 了，子类无法重写它，确保逻辑稳定。

* 提高性能：密封方法可以提高方法调用效率，因为 `JIT` 编译器可以直接调用它，而不需要查找虚方法表（`vtable`）。

### sealed 结合 abstract

* `sealed` 不能和 `abstract` 一起用于类，因为抽象类必须允许继承，而密封类不能被继承。

* 但是，抽象类的 `override` 方法可以是 `sealed`，防止进一步重写。

```csharp
abstract class AbstractClass
{
    public abstract void Display();
}

class ConcreteClass : AbstractClass
{
    public sealed override void Display()
    {
        Console.WriteLine("ConcreteClass Display");
    }
}

class SubConcreteClass : ConcreteClass
{
    // 报错：无法重写密封方法 "Display"
    // public override void Display() { }
}
```

### sealed 结合 struct

C# 中，所有 `struct` 默认是 `sealed`，不能被继承，所以不需要显式声明 `sealed`。

```csharp
struct MyStruct
{
    public int Value;
}

// 报错：结构不能被继承
// class DerivedStruct : MyStruct { }
```

### 总结

|  用法   |  作用   |  示例   |
| --- | --- | --- |
|  `sealed` 类   |  防止继承   |  `sealed class MyClass { }`   |
|  `sealed` 方法   |  防止方法被进一步重写   |  `public sealed override void Method() { }`   |
|  `sealed + abstract`   |  限制抽象方法的继承   |  `public sealed override void AbstractMethod() { }`   |
|  `sealed` 结构体   |  默认不能被继承   |  `struct MyStruct { }`   |

### C# sealed 与 Java final 比较

#### 类（防止继承）

* C#：使用 sealed

```csharp
sealed class MyClass
{
    public void Show()
    {
        Console.WriteLine("Hello from MyClass");
    }
}

// 下面的代码会报错
// class DerivedClass : MyClass { } // 错误：无法从密封类 'MyClass' 派生
```

* Java：使用 final

```java
final class MyClass {
    void show() {
        System.out.println("Hello from MyClass");
    }
}

// 下面的代码会报错
// class DerivedClass extends MyClass {} // 错误：无法继承 final 类
```

`C#` 的 `sealed class` 和 `Java` 的 `final class` 作用一样，都用于防止类被继承。

#### 方法（防止子类重写）

* C#：使用 sealed

```csharp
class BaseClass
{
    public virtual void Show()
    {
        Console.WriteLine("BaseClass Show");
    }
}

class DerivedClass : BaseClass
{
    public sealed override void Show()
    {
        Console.WriteLine("DerivedClass Show");
    }
}

// 下面的代码会报错
// class SubDerivedClass : DerivedClass
// {
//     public override void Show() { } // 错误：无法重写密封的方法 "Show"
// }
```

* Java：使用 final

```java
class BaseClass {
    public void show() {
        System.out.println("BaseClass Show");
    }
}

class DerivedClass extends BaseClass {
    @Override
    public final void show() {
        System.out.println("DerivedClass Show");
    }
}

// 下面的代码会报错
// class SubDerivedClass extends DerivedClass {
//     @Override
//     public void show() {} // 错误：无法重写 final 方法
// }
```

`C#` 的 `sealed override` 和 `Java` 的 `final method` 作用相同，都用于防止方法被子类重写。

#### 变量（防止修改）

`Java` 支持 `final` 变量，`C#` 需要用 `readonly` 或 `const` 实现类似功能。

* Java：final 变量

```java
class Example {
    final int number = 10; // 只能赋值一次

    void changeNumber() {
        // 报错：无法修改 final 变量
        // number = 20;
    }
}
```

* C#：使用 readonly 或 const

```csharp
class Example
{
    public readonly int number = 10; // 只能在构造函数中赋值
    public const int constNumber = 20; // 编译时常量，必须初始化

    public Example()
    {
        number = 15; // 允许
        // constNumber = 30; // 报错：const 变量不能修改
    }
}
```

* `Java` 的 `final` 可以用于变量，防止变量被修改。

* `C#` 没有 `final` 变量，但可以用 `readonly`（运行时常量）或 `const`（编译时常量）来替代。