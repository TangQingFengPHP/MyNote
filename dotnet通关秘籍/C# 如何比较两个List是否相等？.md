### 简介

在 `C#` 里，比较两个 `List` 是否相等，需要考虑多个方面，例如列表中的元素顺序、元素本身是否相等。下面介绍几种常见的比较方法：

### 基本类型比较（元素顺序必须一致）

```csharp
var list1 = new List<int> { 1, 2, 3 };
var list2 = new List<int> { 1, 2, 3 };

bool areEqual = list1.SequenceEqual(list2); // ✅ true
```

### 忽略顺序比较

```csharp
var list1 = new List<int> { 1, 2, 3 };
var list2 = new List<int> { 3, 2, 1 };

bool areEqual = list1.OrderBy(x => x).SequenceEqual(list2.OrderBy(x => x)); // ✅ true
```

或先分别排完序，再比较：

```csharp
list3.Sort();
list4.Sort();
Console.WriteLine(list3.SequenceEqual(list4)); // 输出: True
```

### 复杂类型（自定义对象列表）

* 实现 `Equals` 和 `GetHashCode` 方法

```csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }

    public override bool Equals(object? obj)
    {
        if (obj is Person person)
        {
            return Name == person.Name && Age == person.Age;
        }
        return false;
    }

    public override int GetHashCode()
    {
        return HashCode.Combine(Name, Age);
    }
}
```

使用：

```csharp
Console.WriteLine(person1.SequenceEqual(person2)); // 输出: True
```

* 自定义比较器：

```csharp
public class Person
{
    public string Name { get; set; }
    public int Age { get; set; }
}
```

```csharp
public class PersonComparer : IEqualityComparer<Person>
{
    public bool Equals(Person? x, Person? y)
    {
        return x?.Name == y?.Name && x?.Age == y?.Age;
    }

    public int GetHashCode(Person obj)
    {
        return HashCode.Combine(obj.Name, obj.Age);
        // 还有一种写法：
        // return obj.Name.GetHashCode() ^ obj.Age.GetHashCode();
    }
}
```

使用方式：

```csharp
var list1 = new List<Person>
{
    new Person { Name = "Alice", Age = 25 },
    new Person { Name = "Bob", Age = 30 }
};

var list2 = new List<Person>
{
    new Person { Name = "Alice", Age = 25 },
    new Person { Name = "Bob", Age = 30 }
};

bool areEqual = list1.SequenceEqual(list2, new PersonComparer()); // ✅ true
```

### 判断是否完全包含对方（不关心顺序）

```csharp
bool setEqual = list1.Count == list2.Count &&
                !list1.Except(list2).Any() &&
                !list2.Except(list1).Any();
```

### 使用 SetEquals（无序、无重复判断）

```csharp
bool areEqual = new HashSet<int>(list1).SetEquals(list2);
```

或：

```csharp
HashSet<int> set1 = new HashSet<int>(list3);
HashSet<int> set2 = new HashSet<int>(list4);

bool isEqual = set1.SetEquals(set2);
Console.WriteLine(isEqual); // 输出: True
```

### 比较两个 null 列表

```csharp
List<int>? list5 = null;
List<int>? list6 = null;
Console.WriteLine(list5 == list6); // 输出: True
```

### 比较两个包含null元素的列表

```csharp
List<string?> list7 = new List<string?> { "a", null };
List<string?> list8 = new List<string?> { "a", null };
Console.WriteLine(list7.SequenceEqual(list8)); // 输出: True
```

### 性能优化建议

* 小规模数据：使用 `SequenceEqual` 或 `HashSet`。

* 大规模数据：
    * 先检查列表长度是否相同。
    * 使用并行化处理（如 `AsParallel().SequenceEqual()`）。

### 总结

|  场景   |  方法   |  是否考虑顺序   |  是否考虑重复次数   |
| --- | --- | --- | --- |
|  顺序敏感且内容相同   |  `SequenceEqual`   |  是   |  是   |
|  顺序不敏感且内容相同   |  `HashSet.SetEquals`   |  否   |  否   |
|  顺序不敏感但重复次数相同   |  排序后使用 `SequenceEqual`   |  否   |  是   |
|  自定义对象比较   |  重写 `Equals` 或使用 `IEqualityComparer`   |  可配置   |  可配置   |