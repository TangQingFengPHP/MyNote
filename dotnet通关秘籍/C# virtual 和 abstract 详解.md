### 简介

在 `C#` 中，`virtual` 和 `abstract` 关键字都用于面向对象编程中的继承和多态，它们主要用于方法、属性和事件的定义，但在用法上存在一些重要的区别。

### virtual 关键字

`virtual` 表示可重写的方法，但可以提供默认实现，派生类可以选择是否重写。

> 使用规则

* 只能在类成员（方法、属性、事件）上使用，不能用于字段。
* 必须有默认实现，子类可以选择是否 `override` 进行重写。
* 在基类中调用时，调用的是基类的方法，但如果子类重写了，则会调用子类的实现（运行时多态）。
* 不能用于 `static` 方法，但可以用于属性和事件。

示例：`virtual` 方法

```csharp
using System;

class BaseClass
{
    public virtual void ShowMessage()
    {
        Console.WriteLine("基类的 ShowMessage() 方法");
    }
}

class DerivedClass : BaseClass
{
    public override void ShowMessage()
    {
        Console.WriteLine("子类的 ShowMessage() 方法");
    }
}

class Program
{
    static void Main()
    {
        BaseClass obj1 = new BaseClass();
        obj1.ShowMessage(); // 输出: 基类的 ShowMessage() 方法

        DerivedClass obj2 = new DerivedClass();
        obj2.ShowMessage(); // 输出: 子类的 ShowMessage() 方法

        BaseClass obj3 = new DerivedClass();
        obj3.ShowMessage(); // 输出: 子类的 ShowMessage() 方法 （运行时多态）
    }
}
```

`virtual` 属性

```csharp
class Animal
{
    public virtual string Name { get; set; } = "Unknown Animal";
}

class Dog : Animal
{
    public override string Name { get; set; } = "Dog";
}

class Program
{
    static void Main()
    {
        Animal a = new Dog();
        Console.WriteLine(a.Name); // 输出: Dog
    }
}
```

### abstract 关键字

`abstract` 表示抽象成员，没有实现，必须在派生类中重写。它只能出现在 `abstract` 类中。

> 使用规则

* 只能在 `abstract` 类中使用，不能用于 `sealed` 类（密封类）。
* 没有方法体，子类必须 `override` 提供具体实现。
* 抽象方法不能有 `private` 修饰符，但可以是 `protected` 或 `public`。
* 抽象方法在基类中不能有默认实现，必须在子类实现。
* 抽象类本身不能被实例化，但可以有构造函数。

示例：`abstract` 方法

```csharp
using System;

abstract class Animal
{
    public abstract void MakeSound(); // 没有方法体，子类必须实现
}

class Dog : Animal
{
    public override void MakeSound()
    {
        Console.WriteLine("汪汪汪！");
    }
}

class Cat : Animal
{
    public override void MakeSound()
    {
        Console.WriteLine("喵喵喵！");
    }
}

class Program
{
    static void Main()
    {
        Animal dog = new Dog();
        dog.MakeSound(); // 输出: 汪汪汪！

        Animal cat = new Cat();
        cat.MakeSound(); // 输出: 喵喵喵！
    }
}
```

`abstract` 属性

```csharp
abstract class Animal
{
    public abstract string Name { get; set; }
}

class Dog : Animal
{
    public override string Name { get; set; } = "Dog";
}

class Program
{
    static void Main()
    {
        Animal a = new Dog();
        Console.WriteLine(a.Name); // 输出: Dog
    }
}
```

### virtual vs abstract 的区别

|  关键字   |  virtual   |  abstract   |
| --- | --- | --- |
|  是否有实现   |  有默认实现   |  没有默认实现，必须在子类实现   |
|  是否必须重写   |  可以重写，也可以不重写   |  必须被子类重写   |
|  是否能在非 abstract 类中使用   |  可以   |  只能用于 abstract 类   |
|  能否实例化   |  可以实例化基类   |  抽象类不能实例化   |
|  适用范围   |  方法、属性、事件   |  方法、属性、事件   |

### 什么时候用 virtual，什么时候用 abstract？

用 `virtual`

* 适用于提供默认行为，但允许子类覆盖的场景。
* 例如：基类提供 `virtual` 方法 `SaveToDatabase()`，子类可以重写也可以直接使用。

用 `abstract`

* 适用于基类无法提供合理默认实现，必须由子类提供具体实现的情况。
* 例如：`Animal` 类的 `MakeSound()` 方法，每种动物的叫声都不同，所以必须由子类实现。

### 综合示例

```csharp
using System;

abstract class Animal
{
    public abstract void MakeSound(); // 必须由子类实现

    public virtual void Sleep()
    {
        Console.WriteLine("动物正在睡觉...");
    }
}

class Dog : Animal
{
    public override void MakeSound()
    {
        Console.WriteLine("汪汪汪！");
    }

    public override void Sleep()
    {
        Console.WriteLine("狗狗正在睡觉...");
    }
}

class Program
{
    static void Main()
    {
        Animal myDog = new Dog();
        myDog.MakeSound(); // 汪汪汪！
        myDog.Sleep();     // 狗狗正在睡觉...
    }
}
```

### virtual 方法如何调用基类方法？

当子类重写了 `virtual` 方法后，仍然可以在 `override` 方法中调用 基类的实现，这在 扩展父类功能 时特别有用。

在子类 `override` 方法中调用 `base` 方法

```csharp
using System;

class BaseClass
{
    public virtual void ShowMessage()
    {
        Console.WriteLine("基类的 ShowMessage() 方法");
    }
}

class DerivedClass : BaseClass
{
    public override void ShowMessage()
    {
        base.ShowMessage();  // 先调用基类方法
        Console.WriteLine("子类的 ShowMessage() 方法");
    }
}

class Program
{
    static void Main()
    {
        DerivedClass obj = new DerivedClass();
        obj.ShowMessage();
    }
}
```

输出：

```
基类的 ShowMessage() 方法
子类的 ShowMessage() 方法
```

* 这里 `base.ShowMessage()`; 先执行基类的方法，再执行子类的逻辑。
* 这种方式避免了完全覆盖，而是基于已有逻辑做扩展。

### abstract 和 interface 的区别

在 `C#` 中，`abstract` 和 `interface` 都可以定义必须由子类实现的方法，但它们有一些关键区别。

|     |  abstract   |  interface   |
| --- | --- | --- |
|  是否有方法实现   |  不一定，可以有 abstract 也可以 virtual   |  C# 8.0+ 支持默认实现，但大多数情况下没有   |
|  是否可以包含字段	   |  可以（非 static）   |  不可以   |
|  是否可以有构造函数   |  可以   |  不可以   |
|  继承方式   |  只能继承一个   |  可以实现多个   |

```csharp
abstract class Animal
{
    public abstract void MakeSound();
    public virtual void Sleep()
    {
        Console.WriteLine("睡觉中...");
    }
}

interface IAnimal
{
    void MakeSound();
    void Sleep();
}

class Dog : Animal, IAnimal  // 可以同时继承抽象类和接口
{
    public override void MakeSound()
    {
        Console.WriteLine("汪汪汪！");
    }

    public void Sleep()
    {
        Console.WriteLine("狗狗正在睡觉...");
    }
}
```

`abstract class` 适用于有共享逻辑的场景，而 `interface` 适用于不同类之间的通用行为。

### virtual 和 abstract 结合使用

在某些情况下，抽象类可以定义 `virtual` 方法，让子类选择是否覆盖。

```csharp
abstract class Shape
{
    public abstract void Draw(); // 必须重写

    public virtual void Move()
    {
        Console.WriteLine("Shape is moving...");
    }
}

class Circle : Shape
{
    public override void Draw()
    {
        Console.WriteLine("画一个圆形");
    }

    public override void Move()
    {
        Console.WriteLine("圆形在移动...");
    }
}

class Program
{
    static void Main()
    {
        Shape shape = new Circle();
        shape.Draw(); // 画一个圆形
        shape.Move(); // 圆形在移动...
    }
}
```

* `Draw()` 是抽象方法，必须重写。
* `Move()` 是虚方法，子类可以选择是否重写。

### virtual 和 abstract 在性能上的考虑

在 `C#` 中，`virtual` 方法会在运行时通过虚方法表（`VTable`） 进行调用，而 `abstract` 只是提供一个强制实现的约定，子类实现后仍然是 `virtual` 方法。

普通方法调用（非 `virtual`）

```csharp
public class MyClass
{
    public void NormalMethod() { }
}

// 直接调用，无额外开销。
```

`virtual` 方法调用

```csharp
public class MyClass
{
    public virtual void VirtualMethod() { }
}

// 通过 VTable 进行间接调用，可能稍微慢一点（但一般无影响）。
```

`abstract` 方法调用

```csharp
public abstract class MyBaseClass
{
    public abstract void AbstractMethod();
}

public class MyClass : MyBaseClass
{
    public override void AbstractMethod() { }
}

// abstract 方法在子类实现后，仍然是虚方法（virtual），运行时仍然使用 VTable 机制。
```

### sealed 和 override 一起使用

`sealed` 关键字可以阻止子类进一步重写 `virtual` 方法。

```csharp
class Parent
{
    public virtual void Show()
    {
        Console.WriteLine("父类方法");
    }
}

class Child : Parent
{
    public sealed override void Show()
    {
        Console.WriteLine("子类方法，不能再被重写");
    }
}

class GrandChild : Child
{
    // 这里会报错，因为 `Show` 方法被 `sealed` 了
    // public override void Show() { }
}
```

* 子类 `Child` 已经 `sealed override`，所以 `GrandChild` 不能再重写 `Show()` 方法。

* 适用于限制特定方法的继承，比如框架设计时控制扩展性。

### 什么时候用 virtual，什么时候用 abstract？

用 `virtual`

* 当希望提供一个默认实现，但允许子类根据需要重写时。

用 `abstract`

* 当基类无法提供默认实现，必须要求子类强制实现时。

用 `sealed override`

* 当希望限制某个方法的进一步重写时。

用 abstract + virtual 组合

* 适用于部分方法必须实现，部分方法可以选择重写的情况。