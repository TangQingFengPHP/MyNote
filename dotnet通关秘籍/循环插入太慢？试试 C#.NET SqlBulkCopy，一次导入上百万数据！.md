### 简介

`SqlBulkCopy` 是 `.NET` 中针对 `SQL Server` 的高性能批量数据导入类，通过最小化网络往返和利用 `SQL Server` 的批量加载机制，实现远超传统 `INSERT` 语句的性能（通常快 10-100 倍）。它通过利用 `SQL Server` 的批量插入机制（`BCP，Bulk Copy Protocol`），显著提高了数据导入的效率，特别适合大数据量场景。

#### 背景和作用

在 `.NET` 应用中，插入大量数据到 `SQL Server` 数据库时，传统的逐行插入（如通过 `EF Core` 的 `Add` 和 `SaveChanges`）效率低下，容易导致性能瓶颈。`SqlBulkCopy` 解决了以下问题：

* 高性能批量插入：通过批量操作，减少数据库往返，提升插入速度。

* 大吞吐量支持：适合导入数千到数百万行数据。

* 灵活的数据源：支持从 `DataTable、DataReader` 或其他实现 `IDataReader` 的数据源导入。

* 事务支持：允许在事务中执行批量插入，确保数据一致性。

### 安装与配置

`SqlBulkCopy` 是 `SQL Server` 提供的一个类，位于 `System.Data.SqlClient` 中，所以需要确保在项目中引用了相应的包。

* `NuGet` 包

如果项目是 `.NET Core` 或 `.NET 5+`，可以使用 `Microsoft.Data.SqlClient` 包。

```shell
Install-Package Microsoft.Data.SqlClient
```

* 命名空间引用

```csharp
using Microsoft.Data.SqlClient;
```

* `SQL Server` 环境

需要 `SQL Server` 数据库支持 `SqlBulkCopy`，并且操作需要数据库连接字符串和相关表结构。

### 核心功能

| 功能                               | 描述                                           |
| ---------------------------------- | ---------------------------------------------- |
| `SqlBulkCopy.WriteToServer`        | 将数据从内存写入到 SQL Server 数据库           |
| `SqlBulkCopy.BulkCopyTimeout`      | 设置执行超时。默认 30 秒，超过会抛出异常       |
| `SqlBulkCopy.BatchSize`            | 批量插入的行数。控制一次性提交的行数，提升性能 |
| `SqlBulkCopy.DestinationTableName` | 指定目标数据库表名                             |
| `SqlBulkCopy.ColumnMappings`       | 设置源列到目标列的映射关系                     |
| `SqlBulkCopy.NotifyAfter`          | 设置每处理指定行数时触发 `SqlRowsCopied` 事件  |
| `SqlBulkCopy.SqlRowsCopied`        | 捕获批量插入的数据统计信息（如行数）           |

### 主要 API 用法

#### 基本用法

```csharp
using Microsoft.Data.SqlClient;

public void BulkInsert()
{
    // 连接到数据库
    string connectionString = "Your_Connection_String";
    using var connection = new SqlConnection(connectionString);
    connection.Open();

    // 使用 SqlBulkCopy 执行批量插入
    using var bulkCopy = new SqlBulkCopy(connection)
    {
        DestinationTableName = "TargetTable"  // 目标表名
    };

    // 创建一个 DataTable 或 IDataReader
    var dataTable = GetDataTable(); // 从数据库或其他数据源获取数据
    bulkCopy.WriteToServer(dataTable); // 执行插入操作

    Console.WriteLine("批量插入成功！");
}
```

#### 从 IDataReader 插入数据

从另一个数据库读取数据并插入：

```csharp
using Microsoft.Data.SqlClient;

class Program
{
    static void Main()
    {
        string sourceConnString = "Server=source;Database=sourcedb;Trusted_Connection=True;";
        string destConnString = "Server=localhost;Database=testdb;Trusted_Connection=True;";

        using var sourceConn = new SqlConnection(sourceConnString);
        using var destConn = new SqlConnection(destConnString);
        sourceConn.Open();
        destConn.Open();

        using var command = new SqlCommand("SELECT Id, Name FROM SourceUsers", sourceConn);
        using var reader = command.ExecuteReader();

        using var bulkCopy = new SqlBulkCopy(destConn)
        {
            DestinationTableName = "Users",
            BatchSize = 1000
        };

        // 列映射（如果列名不匹配）
        bulkCopy.ColumnMappings.Add("Id", "UserId");
        bulkCopy.ColumnMappings.Add("Name", "UserName");

        bulkCopy.WriteToServer(reader);
        Console.WriteLine("Data copied from source to destination");
    }
}
```

#### 异步插入

使用异步方法提升性能：

```csharp
using Microsoft.Data.SqlClient;
using System.Data;
using System.Threading.Tasks;

class Program
{
    static async Task Main()
    {
        string connectionString = "Server=localhost;Database=testdb;Trusted_Connection=True;";
        var dataTable = new DataTable("Users");
        dataTable.Columns.Add("Id", typeof(int));
        dataTable.Columns.Add("Name", typeof(string));

        for (int i = 1; i <= 10000; i++)
        {
            dataTable.Rows.Add(i, $"User{i}");
        }

        using var connection = new SqlConnection(connectionString);
        await connection.OpenAsync();

        using var bulkCopy = new SqlBulkCopy(connection)
        {
            DestinationTableName = "Users",
            BatchSize = 1000
        };

        await bulkCopy.WriteToServerAsync(dataTable);
        Console.WriteLine("Inserted 10,000 rows asynchronously");
    }
}
```

#### 设置映射关系

如果源数据列与目标表列名称不同，可以通过 `ColumnMappings` 来映射它们。

```csharp
using var bulkCopy = new SqlBulkCopy(connection)
{
    DestinationTableName = "TargetTable"
};
bulkCopy.ColumnMappings.Add("SourceColumn1", "DestinationColumn1");
bulkCopy.ColumnMappings.Add("SourceColumn2", "DestinationColumn2");

bulkCopy.WriteToServer(dataTable);
```

#### 设置批量大小与超时

```csharp
using var bulkCopy = new SqlBulkCopy(connection)
{
    DestinationTableName = "TargetTable",
    BatchSize = 1000,          // 每次提交 1000 行数据
    BulkCopyTimeout = 600      // 设置超时时间为 600 秒
};
bulkCopy.WriteToServer(dataTable);
```

#### 使用事件通知批量插入进度

```csharp
using var bulkCopy = new SqlBulkCopy(connection)
{
    DestinationTableName = "TargetTable",
    NotifyAfter = 1000       // 每处理 1000 行数据时触发事件
};

bulkCopy.SqlRowsCopied += (sender, e) =>
{
    Console.WriteLine($"已插入 {e.RowsCopied} 行数据");
};

bulkCopy.WriteToServer(dataTable);
```

#### 事务与错误处理

```csharp
using (var transaction = connection.BeginTransaction())
{
    try
    {
        bulkCopy.BatchSize = 5000;
        bulkCopy.BulkCopyTimeout = 120;
        bulkCopy.WriteToServer(data);
        transaction.Commit();
    }
    catch (SqlException ex)
    {
        transaction.Rollback();
        // 处理错误（如违反约束）
        foreach (SqlError error in ex.Errors)
        {
            Console.WriteLine($"错误: {error.Message}");
        }
    }
}
```

### 性能优化

#### 批量大小（BatchSize）

设置合适的批量大小（`BatchSize`）可以提高性能。默认情况下，每次插入 10000 行。如果批量大小过大，可能导致内存消耗过高；太小则会影响性能。一般来说，批量大小在 1000-10000 行之间最为合适。

#### 禁用或减少日志

如果不需要插入时的日志记录，可以通过设置 `SqlBulkCopyOptions` 来禁用：

```csharp
using var bulkCopy = new SqlBulkCopy(connection, SqlBulkCopyOptions.KeepNulls);
```

#### 禁用外键约束

批量插入时，外键约束会导致性能下降。如果可以禁用外键约束，插入会更快：

```sql
ALTER TABLE TargetTable NOCHECK CONSTRAINT ALL;
```

在插入后，可以重新启用约束：

```sql
ALTER TABLE TargetTable CHECK CONSTRAINT ALL;
```

#### 关闭索引

插入大量数据时，可以暂时禁用索引，插入完成后再重建索引，性能将显著提高。

```sql
ALTER INDEX ALL ON TargetTable DISABLE;
-- 插入数据
ALTER INDEX ALL ON TargetTable REBUILD;
```

### 使用示例

#### 示例 1：将 `List<T>` 数据批量插入到数据库

```csharp
public class Employee
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Department { get; set; }
}

public void BulkInsertEmployees(List<Employee> employees)
{
    using var connection = new SqlConnection("Your_Connection_String");
    connection.Open();

    // 将 List 转为 DataTable
    var dataTable = ConvertToDataTable(employees);

    using var bulkCopy = new SqlBulkCopy(connection)
    {
        DestinationTableName = "Employees",
        BatchSize = 5000,
        NotifyAfter = 1000
    };

    bulkCopy.SqlRowsCopied += (sender, e) =>
    {
        Console.WriteLine($"已插入 {e.RowsCopied} 行数据");
    };

    bulkCopy.WriteToServer(dataTable);
}

public DataTable ConvertToDataTable(List<Employee> employees)
{
    var table = new DataTable();
    table.Columns.Add("Id", typeof(int));
    table.Columns.Add("Name", typeof(string));
    table.Columns.Add("Department", typeof(string));

    foreach (var employee in employees)
    {
        table.Rows.Add(employee.Id, employee.Name, employee.Department);
    }

    return table;
}
```

#### 示例 2：处理 CSV 文件批量插入

```csharp
public void BulkInsertFromCsv(string filePath)
{
    using var connection = new SqlConnection("Your_Connection_String");
    connection.Open();

    var dataTable = new DataTable();
    dataTable.Columns.Add("Id", typeof(int));
    dataTable.Columns.Add("Name", typeof(string));
    dataTable.Columns.Add("Age", typeof(int));

    var lines = File.ReadLines(filePath);
    foreach (var line in lines.Skip(1))  // 假设第一行是表头
    {
        var values = line.Split(',');
        dataTable.Rows.Add(int.Parse(values[0]), values[1], int.Parse(values[2]));
    }

    using var bulkCopy = new SqlBulkCopy(connection)
    {
        DestinationTableName = "People",
        BatchSize = 5000
    };

    bulkCopy.WriteToServer(dataTable);
}
```

### 使用场景与限制

#### 理想应用场景

* 数据迁移：从旧系统导入历史数据

* `ETL` 处理：数据仓库定期加载

* 实时数据流：传感器/`IoT` 设备批量上传

* 报表生成：预计算数据批量存储

* 缓存预热：初始化内存数据库

#### 功能限制

* 仅限 `SQL Server`：不直接支持其他数据库

* 无更新能力：仅插入新数据（需配合临时表实现更新）

* 触发器影响：默认触发插入触发器（可通过选项控制）

* 标识列处理：需显式设置 `KeepIdentity` 选项

### 资源和文档

* 官方文档：

    * `Microsoft Learn`：https://learn.microsoft.com/en-us/dotnet/api/microsoft.data.sqlclient.sqlbulkcopy

    * `Microsoft.Data.SqlClient`：https://learn.microsoft.com/en-us/dotnet/api/microsoft.data.sqlclient


* `NuGet` 包：https://www.nuget.org/packages/Microsoft.Data.SqlClient

* `GitHub`：https://github.com/dotnet/SqlClient