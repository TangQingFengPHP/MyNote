### 简介

语言集成查询（`LINQ`）是一组强大的技术，它将查询功能直接集成到 `c#` 语言中。`LINQ` 查询是 `c#.net` 中的一等语言结构，就像类、方法、事件一样。`LINQ` 为查询对象（ `LINQ to objects` ）、关系数据库（`LINQ to SQL`）和 `XML`（`LINQ to XML`）提供了一致的查询体验。 

### 查询语法

`LINQ` 提供两种语法：

查询表达式语法（类似 `SQL`）
方法链式语法（基于扩展方法）

#### 查询表达式语法示例

```csharp
int[] numbers = { 1, 2, 3, 4, 5, 6 };

var evenNumbers = from n in numbers
                  where n % 2 == 0
                  select n;

foreach (var num in evenNumbers)
{
    Console.WriteLine(num);
}
```

#### 方法链式语法示例

```csharp
int[] numbers = { 1, 2, 3, 4, 5, 6 };

var evenNumbers = numbers.Where(n => n % 2 == 0);

foreach (var num in evenNumbers)
{
    Console.WriteLine(num);
}
```

### IEnumerable vs IQueryable

#### IEnumerable

`C#` 中的 `IEnumerable` 是一个接口，它定义了一个方法 `GetEnumerator` ，该方法返回一个 `IEnumerator` 对象。在系统中可以找到此接口。集合名称空间。它是 `.net` 的关键部分，用于迭代对象集合。

它提供对指定类型的集合的简单迭代。它主要用于数组、列表等内存集合

当在 `IEnumerable` 上使用 `LINQ` 方法时，查询将在客户端的内存中执行。这意味着所有数据都从数据源（如数据库）加载到内存中，然后执行操作，最适合处理数据集不是太大的内存数据。

示例：

```csharp
List<Student> studentList = new List<Student>()
{
    new Student(){ID = 1, Name = "James", Gender = "Male"},
    new Student(){ID = 2, Name = "Sara", Gender = "Female"},
    new Student(){ID = 3, Name = "Steve", Gender = "Male"},
    new Student(){ID = 4, Name = "Pam", Gender = "Female"}
};

//Linq Query to Fetch all students with Gender Male
IEnumerable<Student> QuerySyntax = from std in studentList
                                   where std.Gender == "Male"
                                   select std;
//Iterate through the collection
foreach (var student in QuerySyntax)
{
    Console.WriteLine( $"ID : {student.ID}  Name : {student.Name}");
}
```

#### IQueryable

`C#` 中的 `IQueryable` 是一个用于从数据源查询数据的接口。`IEnumerable` 用于遍历内存中的集合，而 `IQueryable` 不同于`IEnumerable`，它是为查询数据源而设计的，在这些数据源中，直到对象被枚举后才执行查询。这对于远程数据源（如数据库）特别有用，它允许在服务器端执行查询，从而实现高效查询。

`IQueryable` 查询表示为表达式树。表达式树是一种以树状格式表示代码的数据结构，其中每个节点都是一个表达式，例如方法调用或二进制操作。这允许将查询转换为数据源可以理解的格式，例如数据库的SQL。

`IQueryable` 依赖于 `IQueryProvider` 接口的实现来执行查询。提供程序将表达式树转换为可针对数据源执行的格式。

`IQueryable` 可以对大型数据集表现更好，因为它允许数据库优化和过滤数据。

示例：

```csharp
List<Student> studentList = new List<Student>()
{
    new Student(){ID = 1, Name = "James", Gender = "Male"},
    new Student(){ID = 2, Name = "Sara", Gender = "Female"},
    new Student(){ID = 3, Name = "Steve", Gender = "Male"},
    new Student(){ID = 4, Name = "Pam", Gender = "Female"}
};

//Linq Query to Fetch all students with Gender Male
IQueryable<Student> MethodSyntax = studentList.AsQueryable()
                    .Where(std => std.Gender == "Male");
                                  
//Iterate through the collection
foreach (var student in MethodSyntax)
{
    Console.WriteLine( $"ID : {student.ID}  Name : {student.Name}");
}
```

### 常用操作符

#### `Select` 操作符

`LINQ` 中的投影只是一种用于从数据源中选择数据的机制。可以选择相同表单（即原始表单）中的数据。还可以通过对数据执行一些操作来创建新形式的数据。

`LINQ` `Select` 操作符可以返回一个标量值、一个自定义类、一个自定义类的集合或一个匿名类型，其中包括我们业务需求的属性。

示例：

```csharp
//Query Syntax
IEnumerable<Employee> selectQuery = (from emp in Employee.GetEmployees()
select new Employee()
{
 FirstName = emp.FirstName,
 LastName = emp.LastName,
 Salary = emp.Salary
});

//Method Syntax
List<Employee> selectMethod = Employee.GetEmployees().
Select(emp => new Employee()
{
  FirstName = emp.FirstName,
  LastName = emp.LastName,
  Salary = emp.Salary
}).ToList();

// Anonymous Type
var selectMethod = Employee.GetEmployees().
Select(emp => new
{
  FirstName = emp.FirstName,
  LastName = emp.LastName,
  Salary = emp.Salary
}).ToList();

// 使用索引
var selectMethod = Employee.GetEmployees().
Select((emp, index) => new
{
  // 索引是基0的，每次+1
  IndexPosition = index,
  FullName = emp.FirstName + " " + emp.LastName,
  emp.Salary
});
```

#### `SelectMany` 操作符

`C#` 中的 `LINQ` `SelectMany` 方法用于将序列或集合或数据源的每个元素投影到 `IEnumerable<T>` 类型，并将结果序列平铺成一个序列。这意味着`SelectMany` 投影方法将结果序列中的记录组合起来，然后将它们转换为一个结果。

示例：

```csharp
// Method Syntax
List<string> nameList = new List<string>(){"Pranaya", "Kumar" };
IEnumerable<char> methodSyntax = nameList.SelectMany(x => x);
foreach(char c in methodSyntax)
{
    Console.Write(c + " ");
}

// Query Syntax
List<string> nameList =new List<string>(){"Pranaya", "Kumar" };
           
IEnumerable<char> querySyntax = from str in nameList
                                from ch in str
                                select ch;

foreach (char c in querySyntax)
{
    Console.Write(c + " ");
}

// 复杂的数据结构
public class Student
{
    public int ID { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public List<string> Programming { get; set; }

    public static List<Student> GetStudents()
    {
        return new List<Student>()
        {
            new Student(){ID = 1, Name = "James", Email = "James@j.com", Programming = new List<string>() { "C#", "Jave", "C++"} },
            new Student(){ID = 2, Name = "Sam", Email = "Sara@j.com", Programming = new List<string>() { "WCF", "SQL Server", "C#" }},
            new Student(){ID = 3, Name = "Patrik", Email = "Patrik@j.com", Programming = new List<string>() { "MVC", "Jave", "LINQ"} },
            new Student(){ID = 4, Name = "Sara", Email = "Sara@j.com", Programming = new List<string>() { "ADO.NET", "C#", "LINQ" } }
        };
    }
}

// Method Syntax
List<string> MethodSyntax = Student.GetStudents().SelectMany(std => std.Programming).ToList();

// Query Syntax
IEnumerable<string> QuerySyntax = from std in Student.GetStudents()
              from program in std.Programming
              select program;

// 同时取出Student的name和Programming列表的值
// Method Syntax
var MethodSyntax = Student.GetStudents()
                    .SelectMany(std => std.Programming,
                        (student, program) => new
                        {
                            StudentName = student.Name,
                            ProgramName = program
                        }
                    )
                    .ToList();

// Query Syntax
var QuerySyntax = (from std in Student.GetStudents()
                    from program in std.Programming
                    select new {
                        StudentName = std.Name,
                        ProgramName = program
                    }).ToList();

// 扁平化输出如下：
James - C#
James - Jave
James - C++
Sam - WCF
Sam - SQL Server
Sam - C#
Patrik - MVC
Patrik - Jave
Patrik - LINQ
Sara - ADO.NET
Sara - C#
Sara - LINQ
```

#### `Where` 操作符

`LINQ` `Where` 方法用于根据谓词筛选集合，该谓词接受集合中的每个元素并返回一个布尔值。如果函数对某个元素返回 `true`，则该元素包含在结果中；否则，它被排除在外。 
 
`Where` 方法总是期望至少有一个条件，我们可以使用谓词指定条件。可以使用`==、>=、<=、&&、|、|、>、<` 等符号来编写条件。

示例：

```csharp
var intList = new List<int> { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

// 普通用法
// Method Syntax
IEnumerable<int> filteredData = intList.Where(num => num > 5);

// 委托的写法
Func<int, bool> predicate = x => x > 5;
IEnumerable<int> filteredData = intList.Where(predicate);

// Query Syntax
IEnumerable<int> filteredResult = from num in intList
                                  where num > 5
                                  select num;

// 多个Where条件
//Query Syntax
var QuerySyntax = from employee in Employee.GetEmployees()
                  where employee.Salary > 500000 && employee.Gender == "Male"
                  select employee;
//Method Syntax
var MethodSyntax = Employee.GetEmployees()
                   .Where(emp => emp.Salary > 500000 && emp.Gender == "Male")
                   .ToList();
// 或者多次使用Where方法
var QuerySyntax = from employee in Employee.GetEmployees()
                  where employee.Salary > 500000
                  where employee.Gender == "Male"
                  select employee;
var MethodSyntax = Employee.GetEmployees()
                   .Where(emp => emp.Salary > 500000)
                   .Where(emp.Gender == "Male")
                   .ToList();

// 使用Where方法里面的索引
var MethodSyntax = Employee.GetEmployees()
                   .Where((x, i) => {
                        if (i % 2 == 0)
                        {
                            return true;
                        }
                        return false;
                   })
                   .ToList();
```

#### `OfType` 操作符

`C#` 中的 `LINQ OfType` 运算符根据我们传递给该运算符的数据类型从数据源中过滤特定的数据。例如，如果有一个存储整型值和字符串值的集合，并且需要从该集合中获取整型值或仅获取字符串值，则需要使用 `LINQ OfType` 操作符。

它根据元素转换为特定类型的能力来过滤元素。 
它排除不能强制转换为该类型的元素，而不是抛出 `InvalidCastException`。

工作原理：

* 基于类型的过滤：`OfType` 操作符检查源集合中的每个元素。它检查每个元素是指定类型还是派生类型。 

* 类型转换：如果元素是指定类型或派生类型，`OfType` 将其包含在结果中。它隐式地将元素强制转换为指定的类型。 

* 忽略不匹配的元素：不属于指定类型的元素将被忽略，并且不包含在结果中。 
返回一个过滤集合：操作符产生一个 `IEnumerable<T>` ，其中 `T` 是指定的类型。这意味着结果集合只包含指定类型的元素，并且它们已经被强制转换为该类型。

示例：

```csharp
var dataSource = new List<object>()
{
    "Tom", "Mary", 50, "Prince", "Jack", 10, 20, 30, 40, "James"
};

//Fetching only the Integer Data from the Data Source
// 使用方法语法
var intData = dataSource.OfType<int>().ToList();

// 使用方法语法并带上查询条件
var intData = dataSource.OfType<int>().Where(num => num > 30).ToList();

// 使用查询语法
var intData = (from num in dataSource.OfType<int>()
                select num).ToList();

// 使用查询语法并带上查询条件
var stringData2 = (from name in dataSource.OfType<string>()
                    where name.Length > 3
                    select name).ToList();

// 使用 is 操作符
var stringData = (from name in dataSource
                    where name is string
                    select name).ToList();

// 使用 is 操作符并带上查询条件
var stringData = (from name in dataSource
                    where name is string && name.ToString().Length > 3
                    select name).ToList();

// 过滤自定义对象类型
var animals = new List<Animal> { new Dog(), new Cat(), new Dog(), new Bird() };
var dogsOnly = animals.OfType<Dog>();
``` 

#### `Set` 操作符

用于对序列或集合（如数组、列表等）执行数学集合操作，这些操作符包括`Distinct`、`Union`、`Intersect` 和 `Except`，每个操作符返回一个新的结果序列，可用于实现 `IEnumerable<T>` 接口的任何类型

* `Distinct`：此运算符返回集合中的不同元素，可有效删除重复项

* `Union`：此运算符将两个序列合并，并返回一个包含两个序列中唯一元素的新序列，也是去重的

* `Intersect`：此运算符返回一个新序列，其中包含两个源序列中都存在的元素它，类似于数学上的集合交集

* `Except`：此运算符返回第一个序列中不存在但第二个序列中的元素，它类似于集合中的减法运算，即差集

* `Concat`：合并序列，不去重

示例：

```csharp
int[] sequence1 = { 1, 2, 3, 4, 5 };
int[] sequence2 = { 4, 5, 6, 7, 8 };

var distinct = sequence1.Distinct();
Console.WriteLine("Distinct: " + string.Join(", ", distinct));

var union = sequence1.Union(sequence2);
Console.WriteLine("Union: " + string.Join(", ", union));

var intersect = sequence1.Intersect(sequence2);
Console.WriteLine("Intersect: " + string.Join(", ", intersect));

var except = sequence1.Except(sequence2);
Console.WriteLine("Except: " + string.Join(", ", except));

var concatenated = sequence1.Concat(sequence2);
Console.WriteLine("Concat: " + string.Join(", ", concatenated));
```

#### `排序` 操作符

`LINQ` 中的排序操作符用于根据一个或多个标准对序列或集合中的元素进行排序，这些操作符允许我们按照指定的顺序排列数据，例如升序或降序。排序操作符可以使用方法语法和查询语法。

* `OrderBy`：根据指定的键对元素进行升序排序，还可以将 `OrderBy` 与附加的 `descending` 关键字结合使用，以按降序排序

* `OrderByDescending`：与 `OrderBy` 类似，根据指定的键按降序对元素进行排序

* `ThenBy`：通常在 `OrderBy` 或 `OrderByDescending` 之后使用，以提供二级排序。例如，可能首先按成绩对学生进行排序，然后使用 `ThenBy` 按姓名升序对具有相同成绩的学生进行排序

* `ThenByDescending`：与 `ThenBy` 类似，`ThenByDescending` 运算符在主排序之后对具有相同值的元素执行二次降序排序。

* `Reverse`：可反转序列中元素的顺序，从而有效地按降序对它们进行排序

示例：

```csharp
var people = new List<Person>
{
    new Person { FirstName = "John", LastName = "Doe", Age = 30 },
    new Person { FirstName = "Jane", LastName = "Doe", Age = 25 },
    new Person { FirstName = "Joe", LastName = "Bloggs", Age = 30 },
    // ... other people ...
};

var sortedPeople = people
    .OrderBy(p => p.LastName)
    .ThenBy(p => p.FirstName)
    .ThenByDescending(p => p.Age);

// 反转序列
var intArray = new int[] { 10, 30, 50, 40,60,20,70,100 };
IEnumerable<int> ArrayReversedData = intArray.Reverse();
```

#### `聚合` 操作符

`LINQ` 聚合运算符对数值数据序列执行数学运算，也用于从值序列中计算单个值。这些操作符用于将多行的值组合在一起作为输入，然后将输出作为单个值返回。因此，简单地说，`C#` 中的聚合方法将始终返回单个值。

可用的聚合方法：

* `Sum()`：此方法计算集合的总值

* `Max()`：此方法用于查找集合中的最大值

* `Min()`：此方法用于查找集合中的最小值

* `Average()`：此方法用于计算集合数字类型的平均值。

* `Count()`：此方法计算集合中的元素数量

* `Aggregate()`：此方法用于对集合的值执行自定义聚合操作

`Aggregate` 方法用于在序列上应用累加器函数，此方法可用于各种复杂的聚合操作，例如求和值、连接字符串，甚至计算自定义聚合。 
 
`Aggregate` 方法以传递结果的方式对每个集合项执行指定操作。就像滚动计算，将函数应用于前两个元素，然后将相同的函数应用于结果和下一个元素，以此类推，直到处理完整个集合。

示例：

```csharp
// Count
var numbers = new List<int> { 1, 2, 3, 4, 5 };
int countEven = numbers.Count(x => x % 2 == 0); // 2

// Sum
var expenses = new List<double> { 100.50, 75.25, 50.0, 30.75 };
double totalExpenses = expenses.Sum(); // 256.5

// Min
var temperatures = new List<int> { 10, 5, 15, 0, -5 };
int minTemperature = temperatures.Min(); // -5

// Max
var prices = new List<decimal> { 25.99m, 15.49m, 30.0m, 10.25m };
decimal maxPrice = prices.Max(); // 30.0

// Average
var scores = new List<double> { 85.5, 92.0, 78.25, 95.75 };
double averageScore = scores.Average(); // 87.625

// Aggregate
// 这是最灵活的聚合运算符，它将定义的函数应用于序列中的每个元素，并累积结果
var numbers = new List<int> { 1, 2, 3, 4, 5 };
int product = numbers.Aggregate((accumulator, number) => accumulator * number); //Return 120 => 1 * 2 * 3 * 4 * 5

// 提供初始种子值
int product = numbers.Aggregate(2, (accumulator, number) => accumulator * number);

// 复杂的对象类型
var listStudents = new List<Employee>()
{
    new Employee{ID= 101,Name = "Preety", Salary = 10000, Department = "IT"},
    new Employee{ID= 102,Name = "Priyanka", Salary = 15000, Department = "Sales"},
    new Employee{ID= 103,Name = "James", Salary = 50000, Department = "Sales"},
    new Employee{ID= 104,Name = "Hina", Salary = 20000, Department = "IT"},
    new Employee{ID= 105,Name = "Anurag", Salary = 30000, Department = "IT"},
    
};

// 第一个重载方法
var Salary = listStudents
            .Aggregate<Employee, int>(0,
            (TotalSalary, emp) => TotalSalary += emp.Salary);

// 第二个重载方法
// 提供种子值
var CommaSeparatedEmployeeNames = listStudents.Aggregate<Employee, string>(
            "Employee Names: ",  // seed value
            (employeeNames, employee) => employeeNames = employeeNames + employee.Name + ", ");

// 第三个重载方法
// 提供额外的逻辑处理
var CommaSeparatedEmployeeNames = listStudents.Aggregate<Employee, string, string>(
            "Employee Names: ",  // seed value
            (employeeNames, employee) => employeeNames = employeeNames + employee.Name + ", ",
            employeeNames => employeeNames.Substring(0, employeeNames.Length - 1));
```

#### `量词` 操作符

`LINQ` 量词运算符检查序列或集合中的部分或全部元素是否满足条件。
这些运算符可以应用于序列，以确定是否存在满足特定标准的元素。
这些操作符返回一个布尔值，该值指示是否对集合中的任何或所有元素的条件为真。

可用的量词操作符

* `Any`：

`Any` 操作符检查序列中的任何元素是否满足给定条件。如果至少有一个元素满足条件，则返回true；否则，返回false。
也可以不带条件地使用它来检查序列是否包含任何元素。

* `All`：

`All` 操作符确定序列中的所有元素是否满足指定条件。如果每个元素都满足条件，则返回true；
如果至少有一个元素不满足条件，则返回false。

* `Contains`：
`Contains` 操作符检查序列是否包含特定的元素。它可用于确定序列是否包含特定值，如果找到该值则返回true，否则返回false。
这也依赖于序列中元素类型的默认相等比较器。

示例：

```csharp
var numbers = new List<int> { 1, 2, 3, 4, 5 };
var anyGreaterThanThree = numbers.Any(x => x > 3); // true
var anyEven = numbers.Any(x => x % 2 == 0); // true

var allGreaterThanZero = numbers.All(x => x > 0); // true
var allEven = numbers.All(x => x % 2 == 0); // false

var containsFive = numbers.Contains(5); // true
```

#### `GroupBy` 方法

`LINQ GroupBy`方法是 `.net` 中的一个强大功能，它允许我们根据指定的键值将集合中的元素组织成组。`GroupBy` 方法将共享公共属性的元素分组，允许对每个组执行操作。此方法特别适用于聚合数据、对数据子集执行计算以及将数据组织成更易于管理的格式。 

方法签名：

```csharp
public static IEnumerable<IGrouping<TKey, TSource>> GroupBy<TSource, TKey>(
    this IEnumerable<TSource> source,
    Func<TSource, TKey> keySelector
)
```

* TSource：源序列元素的类型

* TKey：keySelector 返回的键的类型

* source：要分组的元素的序列

* keySelector：提取每个元素的键的函数

示例：

```csharp
public class Student
{
    public int ID { get; set; }
    public string Name { get; set; }
    public string Gender { get; set; }
    public string Barnch { get; set; }
    public int Age { get; set; }
    public static List<Student> GetStudents()
    {
        return new List<Student>()
        {
            new Student { ID = 1001, Name = "Preety", Gender = "Female", Barnch = "CSE", Age = 20 },
            new Student { ID = 1002, Name = "Snurag", Gender = "Male", Barnch = "ETC", Age = 21  },
            new Student { ID = 1003, Name = "Pranaya", Gender = "Male", Barnch = "CSE", Age = 21  },
            new Student { ID = 1004, Name = "Anurag", Gender = "Male", Barnch = "CSE", Age = 20  },
            new Student { ID = 1005, Name = "Hina", Gender = "Female", Barnch = "ETC", Age = 20 },
            new Student { ID = 1006, Name = "Priyanka", Gender = "Female", Barnch = "CSE", Age = 21 },
            new Student { ID = 1007, Name = "santosh", Gender = "Male", Barnch = "CSE", Age = 22  },
            new Student { ID = 1008, Name = "Tina", Gender = "Female", Barnch = "CSE", Age = 20  },
            new Student { ID = 1009, Name = "Celina", Gender = "Female", Barnch = "ETC", Age = 22 },
            new Student { ID = 1010, Name = "Sambit", Gender = "Male",Barnch = "ETC", Age = 21 }
        };
    }
}

```
```csharp
class Program
{
    static void Main(string[] args)
    {
        //Using Method Syntax
        IEnumerable<IGrouping<string, Student>> GroupByMS = Student.GetStudents().GroupBy(s => s.Barnch);

        //Using Query Syntax
        IEnumerable<IGrouping<string, Student>> GroupByQS = (from std in Student.GetStudents()
                                                                group std by std.Barnch);
        //It will iterate through each groups
        foreach (IGrouping<string, Student> group in GroupByMS)
        {
            Console.WriteLine(group.Key + " : " + group.Count());
            //Iterate through each student of a group
            foreach (var student in group)
            {
                Console.WriteLine("  Name :" + student.Name + ", Age: " + student.Age + ", Gender :" + student.Gender);
            }
        }
        Console.Read();
    }
}
```

#### 具有多个键的 `GroupBy` 方法

`LINQ` 中的 `GroupBy` 方法允许我们根据指定的键对序列元素进行分组。当按多个键进行分组时，通常使用匿名类型或元组，因为这些类型提供了一种方便的方法，可以将多个字段组合成一个可以用作键的单个对象。

示例：

```csharp
using System.Collections.Generic;
namespace GroupByDemo
{
    public class Student
    {
        public int ID { get; set; }
        public string Name { get; set; }
        public string Gender { get; set; }
        public string Branch { get; set; }
        public int Age { get; set; }

        public static List<Student> GetStudents()
        {
            return new List<Student>()
            {
                new Student { ID = 1001, Name = "Preety", Gender = "Female", Branch = "CSE", Age = 20 },
                new Student { ID = 1002, Name = "Snurag", Gender = "Male", Branch = "ETC", Age = 21  },
                new Student { ID = 1003, Name = "Pranaya", Gender = "Male", Branch = "CSE", Age = 21  },
                new Student { ID = 1004, Name = "Anurag", Gender = "Male", Branch = "CSE", Age = 20  },
                new Student { ID = 1005, Name = "Hina", Gender = "Female", Branch = "ETC", Age = 20 },
                new Student { ID = 1006, Name = "Priyanka", Gender = "Female", Branch = "CSE", Age = 21 },
                new Student { ID = 1007, Name = "santosh", Gender = "Male", Branch = "CSE", Age = 22  },
                new Student { ID = 1008, Name = "Tina", Gender = "Female", Branch = "CSE", Age = 20  },
                new Student { ID = 1009, Name = "Celina", Gender = "Female", Branch = "ETC", Age = 22 },
                new Student { ID = 1010, Name = "Sambit", Gender = "Male",Branch = "ETC", Age = 21 }
            };
        }
    }
}
```

```csharp
namespace GroupByDemo
{
    class Program
    {
        static void Main(string[] args)
        {
            //Using Method Syntax
            var GroupByMultipleKeysMS = Student.GetStudents()
                //Grouping Multiple Keys using an Anonymous Object
                .GroupBy(x => new { x.Branch, x.Gender })
                .Select(g => new
                {
                    Branch = g.Key.Branch,
                    Gender = g.Key.Gender,
                    Students = g.OrderBy(x => x.Name)
                }); ;

            //It will iterate throuVgh each group
            foreach (var group in GroupByMultipleKeysQS)
            {
                Console.WriteLine($"Barnch : {group.Branch} Gender: {group.Gender} No of Students = {group.Students.Count()}");
                //It will iterate through each item of a group
                foreach (var student in group.Students)
                {
                    Console.WriteLine($"  ID: {student.ID}, Name: {student.Name}, Age: {student.Age} ");
                }
                Console.WriteLine();
            }
            Console.Read();
        }
    }
}
```

#### `ToLookup` 方法

`ToLookup` 方法是一个用于从 `IEnumerable<T>` 创建 `Lookup<TKey, element>` 的方法。`Lookup<TKey, TElement>` 类似于`Dictionary<TKey, TValue>`，但是 `Dictionary` 将键映射到单个值，而`Lookup` 将键映射到值的集合。这使得 `Lookup` 在以下场景中非常有用：希望按某个键对序列中的元素进行分组，并在以后快速访问这些组。

**方法签名**

`var lookup = source.ToLookup(keySelector, elementSelector);`

* `source`: 数据源

* `keySelector`: 从每个元素中提取键的函数

* `elementSelector`: 将每个源元素映射到 `Lookup` 中的元素的函数。此参数是可选的；如果未指定，则使用元素本身 

示例：

```csharp
public class Student
{
    public int ID { get; set; }
    public string Name { get; set; }
    public string Gender { get; set; }
    public string Branch { get; set; }
    public int Age { get; set; }
    public static List<Student> GetStudents()
    {
        return new List<Student>()
        {
            new Student { ID = 1001, Name = "Preety", Gender = "Female", Branch = "CSE", Age = 20 },
            new Student { ID = 1002, Name = "Snurag", Gender = "Male", Branch = "ETC", Age = 21  },
            new Student { ID = 1003, Name = "Pranaya", Gender = "Male", Branch = "CSE", Age = 21  },
            new Student { ID = 1004, Name = "Anurag", Gender = "Male", Branch = "CSE", Age = 20  },
            new Student { ID = 1005, Name = "Hina", Gender = "Female", Branch = "ETC", Age = 20 },
            new Student { ID = 1006, Name = "Priyanka", Gender = "Female", Branch = "CSE", Age = 21 },
            new Student { ID = 1007, Name = "santosh", Gender = "Male", Branch = "CSE", Age = 22  },
            new Student { ID = 1008, Name = "Tina", Gender = "Female", Branch = "CSE", Age = 20  },
            new Student { ID = 1009, Name = "Celina", Gender = "Female", Branch = "ETC", Age = 22 },
            new Student { ID = 1010, Name = "Sambit", Gender = "Male",Branch = "ETC", Age = 21 }
        };
    }
}
```

```csharp
// 方法语法
var GroupByMS = Student.GetStudents().ToLookup(s => s.Branch);

// 查询语法
var GroupByQS = (from std in Student.GetStudents()
                    select std).ToLookup(x => x.Branch);
foreach (var group in GroupByMS)
{
    Console.WriteLine(group.Key + " : " + group.Count());
    foreach (var student in group)
    {
        Console.WriteLine("  Name :" + student.Name + ", Age: " + student.Age + ", Gender :" + student.Gender);
    }
}
```

![alt text](/images/dotnet/linq-image.png)

#### `JOIN` 操作

`Join` 操作用于根据两个数据源中存在的一些公共属性从两个或多个数据源中获取数据。

**`Join` 的类型**：

* `Inner Join (Join)`：

`Join` 方法通过基于匹配键关联两个集合的元素来执行内部连接。它返回一个新集合，其中包含键匹配的两个集合中的元素。如果其中一个集合中的元素在另一个集合中没有匹配项，则不包含在结果中。

* `Group Join (GroupJoin)`：

`GroupJoin` 方法类似于内部连接，但它不是返回匹配对的平面集合，而是为第一个集合中的每个元素分组第二个集合中的匹配元素。此操作对于分层数据关系非常有用，例如将一个集合中的每个项与另一个集合中的相关项的集合关联时。

* `Left Outer Join`：

虽然 `LINQ` 没有像内部连接和组连接那样的左外连接内置方法，但可以通过使用`GroupJoin` 方法，然后使用带有 `DefaultIfEmpty` 方法调用的`SelectMany` 方法来实现左外连接。该操作返回第一个集合中的所有元素，对于第一个集合中与第二个集合有匹配的元素，该操作还包括匹配的元素。

* `Cross Join`：

在交叉连接中，第一个集合的每个元素与第二个集合的每个元素组合在一起，产生两个集合的笛卡尔积。这种类型的连接不需要键来连接，并且使用 `SelectMany` 方法实现。

**`Inner Join`**

![alt text](/images/dotnet/linq-image2.png)

**示例**

```csharp
public class Employee
{
    public int ID { get; set; }
    public string Name { get; set; }
    public int AddressId { get; set; }
    public static List<Employee> GetAllEmployees()
    {
        return new List<Employee>()
        {
            new Employee { ID = 1, Name = "Preety", AddressId = 1 },
            new Employee { ID = 2, Name = "Priyanka", AddressId = 2 },
            new Employee { ID = 3, Name = "Anurag", AddressId = 3 },
            new Employee { ID = 4, Name = "Pranaya", AddressId = 4 },
            new Employee { ID = 5, Name = "Hina", AddressId = 5 },
            new Employee { ID = 6, Name = "Sambit", AddressId = 6 },
            new Employee { ID = 7, Name = "Happy", AddressId = 7},
            new Employee { ID = 8, Name = "Tarun", AddressId = 8 },
            new Employee { ID = 9, Name = "Santosh", AddressId = 9 },
            new Employee { ID = 10, Name = "Raja", AddressId = 10},
            new Employee { ID = 11, Name = "Sudhanshu", AddressId = 11}
        };
    }
}
```

```csharp
public class Address
{
    public int ID { get; set; }
    public string AddressLine { get; set; }
    public static List<Address> GetAllAddresses()
    {
        return new List<Address>()
        {
            new Address { ID = 1, AddressLine = "AddressLine1"},
            new Address { ID = 2, AddressLine = "AddressLine2"},
            new Address { ID = 3, AddressLine = "AddressLine3"},
            new Address { ID = 4, AddressLine = "AddressLine4"},
            new Address { ID = 5, AddressLine = "AddressLine5"},
            new Address { ID = 9, AddressLine = "AddressLine9"},
            new Address { ID = 10, AddressLine = "AddressLine10"},
            new Address { ID = 11, AddressLine = "AddressLine11"},
        };
    }
}
```

```csharp
class Program
{
    static void Main(string[] args)
    {
        var JoinUsingMS = Employee.GetAllEmployees()
                        .Join(
                        Address.GetAllAddresses(),
                        employee => employee.AddressId,
                        address => address.ID,
                        (employee, address) => new {
                            EmployeeName = employee.Name,
                            AddressLine = address.AddressLine
                        }).ToList();
    }
}
```

**`Group Join`**

**示例**

```csharp
public class Employee
{
    public int ID { get; set; }
    public string Name { get; set; }
    public int DepartmentId { get; set; }
    public static List<Employee> GetAllEmployees()
    {
        return new List<Employee>()
        {
            new Employee { ID = 1, Name = "Preety", DepartmentId = 10},
            new Employee { ID = 2, Name = "Priyanka", DepartmentId =20},
            new Employee { ID = 3, Name = "Anurag", DepartmentId = 30},
            new Employee { ID = 4, Name = "Pranaya", DepartmentId = 30},
            new Employee { ID = 5, Name = "Hina", DepartmentId = 20},
            new Employee { ID = 6, Name = "Sambit", DepartmentId = 10},
            new Employee { ID = 7, Name = "Happy", DepartmentId = 10},
            new Employee { ID = 8, Name = "Tarun", DepartmentId = 0},
            new Employee { ID = 9, Name = "Santosh", DepartmentId = 10},
            new Employee { ID = 10, Name = "Raja", DepartmentId = 20},
            new Employee { ID = 11, Name = "Ramesh", DepartmentId = 30}
        };
    }
} 
```

```csharp
public class Department
{
    public int ID { get; set; }
    public string Name { get; set; }
    public static List<Department> GetAllDepartments()
    {
        return new List<Department>()
        {
            new Department { ID = 10, Name = "IT"},
            new Department { ID = 20, Name = "HR"},
            new Department { ID = 30, Name = "Sales"  },
        };
    }
}
```

```csharp
class Program
{
    static void Main(string[] args)
    {
        //Group Employees by Department using Method Syntax
        var GroupJoinMS = Department.GetAllDepartments(). //Outer Data Source i.e. Departments
            GroupJoin( //Performing Group Join with Inner Data Source
                Employee.GetAllEmployees(), //Inner Data Source
                dept => dept.ID, //Outer Key Selector  i.e. the Common Property
                emp => emp.DepartmentId, //Inner Key Selector  i.e. the Common Property
                (dept, emp) => new { dept, emp } //Projecting the Result to an Anonymous Type
            );
        //Printing the Result set
        //Outer Foreach is for Each department
        foreach (var item in GroupJoinMS)
        {
            Console.WriteLine("Department :" + item.dept.Name);
            //Inner Foreach loop for each employee of a Particular department
            foreach (var employee in item.emp)
            {
                Console.WriteLine("  EmployeeID : " + employee.ID + " , Name : " + employee.Name);
            }
        }
        Console.ReadLine();
    }
}
```

**`Left Outer Join`**

![alt text](/images/dotnet/linq-image1.png)

**示例**

**查询语法**：

```csharp
var QSOuterJoin = from emp in Employee.GetAllEmployees() //Left Data Source
join add in Address.GetAddress() //Right Data Source
on emp.AddressId equals add.ID //Inner Join Condition
into EmployeeAddressGroup //Performing LINQ Group Join
from address in EmployeeAddressGroup.DefaultIfEmpty() //Performing Left Outer Join
select new { emp, address }; //Projecting the Result to Anonymous Type
```

**方法语法**:

```csharp
var MSOuterJOIN = Employee.GetAllEmployees() //Left Data Source
    //Performing Group join with Right Data Source
    .GroupJoin(
        Address.GetAddress(), //Right Data Source
        employee => employee.AddressId, //Outer Key Selector, i.e. Left Data Source Common Property
        address => address.ID, //Inner Key Selector, i.e. Right Data Source Common Property
        (employee, address) => new { employee, address } //Projecting the Result
    )
    .SelectMany(
        x => x.address.DefaultIfEmpty(), //Performing Left Outer Join 
        (employee, address) => new { employee, address } //Final Result Set
    );
```

**`Full Outer Join`**

**示例**

```csharp
class Program
{
    static void Main(string[] args)
    {
        //Performing Left Outer Join using LINQ using Method Syntax
        var MSLeftOuterJOIN = Employee.GetAllEmployees() //Left Data Source
                            //Performing Group join with Right Data Source
                            .GroupJoin(
                                Department.GetAllDepartments(), //Right Data Source
                                employee => employee.DepartmentId, //Outer Key Selector, i.e. Left Data Source Common Property
                                department => department.ID, //Inner Key Selector, i.e. Right Data Source Common Property
                                (employee, department) => new { employee, department } //Projecting the Result
                            )
                            .SelectMany(
                                x => x.department.DefaultIfEmpty(), //Performing Left Outer Join 
                                //Final Result Set
                                (employee, department) => new
                                {
                                    EmployeeId = employee?.employee?.ID,
                                    EmployeeName = employee?.employee?.Name,
                                    DepartmentName = department?.Name
                                }
                            );
        //Performing Right Outer Join using LINQ using Method Syntax
        var MSRightOuterJOIN = Department.GetAllDepartments() //Left Data Source
                            //Performing Group join with Right Data Source
                            .GroupJoin(
                                Employee.GetAllEmployees(), //Right Data Source
                                department => department.ID, //Outer Key Selector, i.e. Left Data Source Common Property
                                employee => employee.DepartmentId, //Inner Key Selector, i.e. Right Data Source Common Property
                                (department, employee) => new { department, employee } //Projecting the Result
                            )
                            .SelectMany(
                                x => x.employee.DefaultIfEmpty(), //Performing Left Outer Join 
                                //Final Result Set
                                (department, employee) => new
                                {
                                    EmployeeId = employee?.ID,
                                    EmployeeName = employee?.Name,
                                    DepartmentName = department?.department?.Name
                                }
                            );
        var FullOuterJoin = MSLeftOuterJOIN.Union(MSRightOuterJOIN);
        //Accessing the Elements using For Each Loop
        foreach (var emp in FullOuterJoin)
        {
            Console.WriteLine($"EmployeeId: {emp.EmployeeId}, Name: {emp.EmployeeName}, Department: {emp.DepartmentName}");
        }
        Console.ReadLine();
    }
}
```

**`Cross Join`**：

**示例**

```csharp
public class Student
{
    public int ID { get; set; }
    public string Name { get; set; }
    public static List<Student> GetAllStudents()
    {
        return new List<Student>()
        {
            new Student { ID = 1, Name = "Preety"},
            new Student { ID = 2, Name = "Priyanka"},
            new Student { ID = 3, Name = "Anurag"},
            new Student { ID = 4, Name = "Pranaya"},
            new Student { ID = 5, Name = "Hina"}
        };
    }
}
```

```csharp
public class Subject
{
    public int ID { get; set; }
    public string SubjectName { get; set; }
    public static List<Subject> GetAllSubjects()
    {
        return new List<Subject>()
        {
            new Subject { ID = 1, SubjectName = "ASP.NET"},
            new Subject { ID = 2, SubjectName = "SQL Server" },
            new Subject { ID = 5, SubjectName = "Linq"}
        };
    }
}
```

**查询语法**

```csharp
static void Main(string[] args)
{
    //Cross Join using Query Syntax
    var CrossJoinResult = from student in Student.GetAllStudents() //First Data Source
                            from subject in Subject.GetAllSubjects() //Cross Join with Second Data Source
                            //Projecting the Result to Anonymous Type
                            select new
                            {
                                StudentName = student.Name,
                                SubjectName = subject.SubjectName
                            };
    //Accessing the Elements using For Each Loop
    foreach (var item in CrossJoinResult)
    {
        Console.WriteLine($"Name : {item.StudentName}, Subject: {item.SubjectName}");
    }
    Console.ReadLine();
}
```

**方法语法**

```csharp
static void Main(string[] args)
{
    //Cross Join using SelectMany Method
    var CrossJoinResult = Student.GetAllStudents()
                .SelectMany(sub => Subject.GetAllSubjects(),
                    (std, sub) => new
                    {
                        StudentName = std.Name,
                        SubjectName = sub.SubjectName
                    });
    //Cross Join using Join Method
    var CrossJoinResult2 = Student.GetAllStudents()
                .Join(Subject.GetAllSubjects(),
                    std => true,
                    sub => true,
                    (std, sub) => new
                    {
                        StudentName = std.Name,
                        SubjectName = sub.SubjectName
                    }
                    );
    foreach (var item in CrossJoinResult2)
    {
        Console.WriteLine($"Name : {item.StudentName}, Subject: {item.SubjectName}");
    }
    Console.ReadLine();
}
```

#### `元素` 操作符

`LINQ` 中的元素操作符用于使用元素索引位置或基于谓词（即基于指定条件）从数据源返回单个元素。这些元素操作符可用于单个数据源或多个数据源的查询。

* `ElementAt`：

用于使用元素的索引或谓词（即条件）从数据源返回单个元素。这些元素操作符可用于单个数据源或多个数据源的查询。

**示例**

```csharp
List<int> numbers = new List<int>() { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

// 方法语法
int MethodSyntax = numbers.ElementAt(1);

// 查询语法
int QuerySyntax = (from num in numbers
                    select num).ElementAt(1);
```

* `ElementAtOrDefault`：

`ElementAtOrDefault` 方法与 `ElementAt` 方法完全相同，除了当数据源为空或当提供的索引值超出范围或当索引位置指定负值时，该方法不会抛出`ArgumentOutOfRangeException` 异常。在这种情况下，它将根据数据源包含的元素的数据类型返回默认值。如果数据源为 `Null` ，那么它将抛出`ArgumentNullException`。

**示例**

```csharp
List<int> numbers = new List<int>() { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

// 方法语法
int MethodSyntax = numbers.ElementAtOrDefault(-1);

// 查询语法
int QuerySyntax = (from num in numbers
                    select num).ElementAtOrDefault(-1);

// 输出：0
```

* `First`：

`First` 方法返回数据源或集合中的第一个元素。如果数据源或集合为空，或者指定了一个条件并使用该条件，且在数据源中找不到任何元素，将抛出`InvalidOperationException`。如果数据源为 `Null`，那么它将抛出`ArgumentNullException`

**示例**

```csharp
List<int> numbers = new List<int>() { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

// 方法语法
int MethodSyntax = numbers.First();

// 查询语法
int QuerySyntax = (from num in numbers
                    select num).First();

// 带查询条件的
int MethodSyntax = numbers.First(num => num % 2 == 0);
int QuerySyntax = (from num in numbers
                    select num).First(num => num % 2 == 0);
```

* `FirstOrDefault`：

`FirstOrDefault` 方法与 `First` 方法完全相同，除了当数据源为空或指定的条件与数据源中的任何元素不匹配时，该方法不会抛出`InvalidOperationException` 异常。在这种情况下，它将根据数据源的数据类型返回默认值。如果数据源为空，那么像 `First` 方法一样，它也会抛出`ArgumentNullException`。

**示例**

```csharp
List<int> numbers = new List<int>() { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };

int MethodSyntax2 = numbers.FirstOrDefault(num => num > 50);

int QuerySyntax2 = (from num in numbers
                    select num).FirstOrDefault(num => num > 50);
```

* `Last`：

`Last` 方法返回数据源或集合的最后一个元素。如果数据源或集合为空，或者指定了条件并使用该条件，且在数据源中找不到匹配的元素，将抛出`InvalidOperationException`。如果数据源为 `Null` ，那么它将抛出`ArgumentNullException`。

**示例**

```csharp
List<int> numbers = new List<int>() { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
int MethodSyntax = numbers.Last();
int QuerySyntax = (from num in numbers
                    select num).Last();
// 输出：10

// 带查询条件的
int MethodSyntax = numbers.Last(num => num % 3 == 0);
int QuerySyntax = (from num in numbers
                    select num).Last(num => num % 3 == 0);
// 输出：9
```

* `LastOrDefault`：

`LastOrDefault` 方法与 `Last` 方法完全相同，除了当数据源为空或指定的条件与数据源中的任何元素不匹配时，该方法不会抛出`InvalidOperationException` 异常。在这种情况下，它将根据数据源的数据类型返回默认值。如果数据源为空，那么像 `Last` 方法一样，它也会抛出`ArgumentNullException`。

**示例**

```csharp
List<int> numbers = new List<int>() { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
int MethodSyntax2 = numbers.LastOrDefault(num => num > 50);
int QuerySyntax2 = (from num in numbers
                    select num).LastOrDefault(num => num > 50);

// 输出：0
```

* `Single`：

`Single` 方法从数据源返回单个元素，或者可以说从序列返回单个元素。当希望序列只包含一个满足指定条件的元素时，可以使用它。如果数据源为空，或者数据源包含多个元素，则会抛出异常

**示例**

```csharp
List<int> numbers = new List<int>() { 10 };
int numberMS = numbers.Single();
int numberQS = (from num in numbers
                select num).Single();

// 带查询条件的
List<int> numbers = new List<int>() { 10, 20, 30 }; ;
int numberMS = numbers.Single(num => num == 20);
int numberQS = (from num in numbers
                select num).Single(num => num == 20);

// 返回多个元素抛出异常：
// System.InvalidOperationException: Sequence contains more than one matching element
int numberMS = numbers.Single(num => num > 10);
int numberQS = (from num in numbers
                select num).Single(num => num > 10);
```

* `SingleOrDefault`：

如果不想在序列为空或指定的条件没有从序列返回元素时抛出异常，则需要使用`SingleOrDefault` 方法。`SingleOrDefault` 方法与 `Single` 方法非常相似，除了当序列为空或没有元素满足给定条件时，该方法不会抛出异常，如果序列为空，它返回一个默认值。

**示例**

```csharp
List<int> numbers = new List<int>() {10, 20, 30 };
int number = numbers.SingleOrDefault(num => num < 10);

// 输出：0
```

* `DefaultIfEmpty`：

`DefaultIfEmpty` 方法用于值序列，如果序列为空，则返回默认值。此方法通常与其他操作符（如 `Select` 和 `SelectMany` ）一起使用，以确保查询返回结果，即使源序列不包含任何元素。 
 
如果调用 `DefaultIfEmpty` 方法的序列或数据源不为空，则将返回原始序列或数据源值。另一方面，如果序列或数据源为空，则返回一个基于数据类型的具有默认值的序列。

**示例**

```csharp
List<int> numbers = new List<int>();
IEnumerable<int> resultMS = numbers.DefaultIfEmpty();
IEnumerable<int> resultQS = (from num in numbers
                            select num).DefaultIfEmpty();
foreach (int num in resultMS)
{
    Console.Write($"{num} ");
}

// 输出：0

// 方法的重载版本提供默认值
IEnumerable<int> resultMS = numbers.DefaultIfEmpty(5);
```

* `SequenceEqual`：

`SequenceEqual` 方法检查两个序列是否相等。如果两个序列相等，则返回`true`；否则，返回 `false` 。当两个序列具有相同数量的元素和相同的值，并且应该以相同的顺序出现时，两个序列被认为是相等的。

**示例**

```csharp
List<string> cityList1 = new List<string> { "Delhi", "Mumbai", "Hyderabad" };
List<string> cityList2 = new List<string> { "Delhi", "Mumbai", "Hyderabad" };
bool IsEqualMS = cityList1.SequenceEqual(cityList2);
bool IsEqualQS = (from city in cityList1
                    select city).SequenceEqual(cityList2);

// 输出：true

// 方法的重载版本提供参数使得比较不区分大小写
List<string> cityList1 = new List<string> { "DELHI", "mumbai", "Hyderabad" };
List<string> cityList2 = new List<string> { "delhi", "MUMBAI", "Hyderabad" };
bool IsEqualMS = cityList1.SequenceEqual(cityList2, StringComparer.OrdinalIgnoreCase);
bool IsEqualQS = (from city in cityList1
                    select city).SequenceEqual(cityList2, StringComparer.OrdinalIgnoreCase);

// 输出：true
```  

#### `分区` 操作符：

分区操作符用于将一个序列（或者可以说数据源）分成两部分，然后在不改变元素位置的情况下返回其中一部分作为输出。

* `Take`：    

`Take` 方法用于从集合的开始处获取指定数量的元素，在分页等场景中特别有用。

**示例**

```csharp
List<int> numbers = new List<int>() { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
List<int> ResultMS = numbers.Take(4).ToList();
List<int> ResultQS = (from num in numbers
                        select num).Take(4).ToList();

// 输出：1 2 3 4
```

* `TakeWhile`：

`TakeWhile` 方法从数据源、序列或集合中获取所有元素，直到指定的条件为真。一旦条件失败，`TakeWhile` 方法将不会检查数据源中的其余元素，即使对于某些剩余元素条件为真。

**示例**

```csharp
List<int> numbers = new List<int>() { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
List<int> ResultMS = numbers.TakeWhile(num => num < 6).ToList();
List<int> ResultQS = (from num in numbers
                        select num).TakeWhile(num => num < 6).ToList();
// 输出：1 2 3 4 5
```

* `Skip`：

`Skip` 方法绕过序列中指定数量的元素，然后返回剩余的元素。通常用于希望跳过与前页对应的几条记录并返回当前页的下一组记录的分页场景。

**示例**

```csharp
List<int> numbers = new List<int>() { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 };
List<int> ResultMS = numbers.Skip(4).ToList();
List<int> ResultQS = (from num in numbers
                        select num).Skip(4).ToList();

// 输出：5 6 7 8 9 10
```

* `SkipWhile`：

`SkipWhile` 方法用于跳过数据源、序列或集合中的所有元素，直到指定的条件为真。一旦条件失败，则返回序列中剩余的元素作为输出。即使对于某些剩余元素，条件为真，`SkipWhile` 方法也不会检查其余元素。

**示例**

```csharp
List<int> numbers = new List<int>() { 1, 4, 5, 6, 7, 8, 9, 10, 2, 3 };
List<int> ResultMS = numbers.SkipWhile(num => num < 5).ToList();

// 输出：5 6 7 8 9 10 2 3
```

* `Range`：

`Range` 方法是 `Enumerable` 类的静态方法，用于生成指定范围内的整数序列。当需要一个简单的数字序列而无需手动初始化数组或列表时，它特别有用。

**示例**

```csharp
IEnumerable<int> numberSequence = Enumerable.Range(1, 10);
foreach (int num in numberSequence)
{
    Console.Write($"{num} ");
}
// 打印从 1 到 10 的值

// 带条件的生成
IEnumerable<int> EvenNumbers = Enumerable.Range(10, 40).Where(x => x % 2 == 0);

// 接Select方法
IEnumerable<int> EvenNumbers = Enumerable.Range(1, 5).Select(x => x * x);
// 输出：1 4 9 16 25
```

* `Repeat`：

`Repeat` 方法用于生成具有指定数量元素的序列或集合，每个元素都包含相同的值。

**示例**

```csharp
IEnumerable<string> repeatStrings = Enumerable.Repeat("Welcome", 10);
foreach (string str in repeatStrings)
{
    Console.WriteLine(str);
}

// 输出10次 "Welcome"
```

* `Empty`：

`Empty` 方法生成一个特定类型的空序列。当需要从方法返回一个空序列而不返回`null` 时，或者当需要从一个空序列开始并根据进一步的逻辑有条件地使用元素填充它时，此方法非常有用。

**示例**

```csharp
IEnumerable<string> emptyCollection1 = Enumerable.Empty<string>();
IEnumerable<Student> emptyCollection2 = Enumerable.Empty<Student>();
```

* `Append`：

`Append` 方法用于在序列的末尾追加一个值。此方法不修改序列的元素。相反，它创建一个带有新附加元素的序列副本。

**示例**

```csharp
List<int> intSequence = new List<int> { 10, 20, 30, 40 };
intSequence.Append(5);
```

* `Prepend`：

`Prepend` 方法将单个元素添加到序列的开头，与 `Append` 方法一样， `Prepend` 方法不修改序列元素。相反，它使用新元素创建序列的副本。

**示例**

```csharp
List<int> numberSequence = new List<int> { 10, 20, 30, 40 };
numberSequence.Prepend(50);
```

* `Zip`：

`Zip` 方法通过将对应的元素配对在一起，将来自两个或多个序列或集合的元素组合成单个序列。它允许创建元组或应用指定的函数来根据元素在输入序列中的位置组合元素。`Zip` 方法生成一个新序列，其长度由最短的输入序列决定。

`Zip` 方法将第一个序列的每个元素与第二个序列中具有相同索引位置的元素合并。如果两个序列不具有相同数量的元素，那么 `Zip` 方法将合并序列，直到到达包含较少元素的序列的末尾。例如，如果一个序列有5个元素，另一个序列有4个元素，那么结果序列将只有4个元素。

**示例**

```csharp
int[] numbersSequence = { 10, 20, 30, 40, 50 };
string[] wordsSequence = { "Ten", "Twenty", "Thirty", "Fourty" };
var resultSequence = numbersSequence.Zip(wordsSequence, (first, second) => first + " - " + second);
foreach (var item in resultSequence)
{
    Console.WriteLine(item);
}

// 输出：
// 10 - Ten
// 20 - Twenty
// 30 - Thirty
// 40 - Fourty

// 第一个序列包含5个元素，而第二个序列包含4个元素。
// 因此，对于第一个序列的第五个元素，在第二个序列中没有对应的第五个元素。
// 结果，Zip方法合并了这四个元素
```

* `ToList` 和 `ToArray`：

`ToList` 和 `ToArray` 方法分别用于将集合或序列转换为 `List<T>` 或数组 `T[]`

**示例**

```csharp
// 数组转List
int[] numbersArray = { 10, 22, 30, 40, 50, 60 };
List<int> numbersList = numbersArray.ToList();

// List转数组
List<int> numbersList = new List<int>()
{
    10, 22, 30, 40, 50, 60
};
int[] numbersArray = numbersList.ToArray();
```

* `ToDictionary`：

`ToDictionary` 方法是一个 `LINQ` 扩展方法，它将元素序列（例如，一个集合或查询结果）转换为一个字典。当有一个具有键值对的对象集合，并且希望创建一个字典，其中的键从集合中的元素派生时，它特别有用。

**示例**

```csharp
List<Product> listProducts = new List<Product>
{
    new Product { ID= 1001, Name = "Mobile", Price = 800 },
    new Product { ID= 1002, Name = "Laptop", Price = 900 },
    new Product { ID= 1003, Name = "Desktop", Price = 800 }
};
Dictionary<int, Product> productsDictionary = listProducts.ToDictionary(x => x.ID);
foreach (KeyValuePair<int, Product> kvp in productsDictionary)
{
    Console.WriteLine(kvp.Key + " Name : " + kvp.Value.Name + ", Price: " + kvp.Value.Price);
}

// 输出：
// 1001 Name : Mobile, Price: 800
// 1002 Name : Laptop, Price: 900
// 1003 Name : Desktop, Price: 800

// 自定义键值对的“值”
Dictionary<int, string> productsDictionary = listProducts.ToDictionary(x => x.ID, x => x.Name);
foreach (KeyValuePair<int, string> kvp in productsDictionary)
{
    Console.WriteLine("Key : " + kvp.Key + " Value : " + kvp.Value);
}
```

* `Cast`：

`Cast` 方法用于将非泛型集合的元素强制转换为指定类型。可以应用于任何实现`IEnumerable` 的类型，将其转换为 `IEnumerable<T>`，其中 `T` 是目标类型。`Cast` 方法在处理非强类型集合时特别有用。

**示例**

```csharp
ArrayList list = new ArrayList
{
    10,
    20,
    30,
};
list.Add("40");
IEnumerable<int> result = list.Cast<int>();
foreach (int i in result)
{
    Console.WriteLine(i);
}
```