### 简介

`Newtonsoft.Json`（又称 `Json.NET`）是 `.NET` 生态中最流行的 `JSON` 序列化/反序列化库，支持 `.NET Framework、.NET Core、Mono、Xamarin` 等多种平台。

功能丰富：自动映射对象、`LINQ to JSON、JSchema` 验证、自定义转换、性能可调等

### 核心功能与基础使用

#### 序列化与反序列化

```csharp
using Newtonsoft.Json;

// 模型类
public class Person {
    public string Name { get; set; }
    public int Age { get; set; }
    public DateTime BirthDate { get; set; }
}

// 序列化对象到 JSON 字符串
Person person = new Person { Name = "John", Age = 30, BirthDate = new DateTime(1990, 1, 1) };
string json = JsonConvert.SerializeObject(person);
// 输出: {"Name":"John","Age":30,"BirthDate":"1990-01-01T00:00:00"}

// 反序列化 JSON 字符串到对象
Person deserializedPerson = JsonConvert.DeserializeObject<Person>(json);

// 反序列化为匿名类型
var anonymous = JsonConvert.DeserializeAnonymousType(json, new { Id = 0, Name = "", BirthDate = DateTime.MinValue });
```

#### 处理集合与嵌套对象

```csharp
// 序列化集合
List<Person> people = new List<Person> { person };
string jsonArray = JsonConvert.SerializeObject(people);
// 输出: [{"Name":"John","Age":30,"BirthDate":"1990-01-01T00:00:00"}]

// 嵌套对象
public class Company {
    public string Name { get; set; }
    public Person CEO { get; set; }
}

Company company = new Company { 
    Name = "Acme Corp", 
    CEO = person 
};
string jsonWithNested = JsonConvert.SerializeObject(company);
// 输出: {"Name":"Acme Corp","CEO":{"Name":"John","Age":30,"BirthDate":"1990-01-01T00:00:00"}}
```

#### LINQ to JSON：通过 JObject、JArray 动态操作 JSON

```csharp
JObject jo = JObject.Parse(json);
var name = jo["name"]?.ToString();
```

* 动态解析 `JSON`

```csharp
string json = @"{""Color"":{""Red"":0.8}}";
JObject obj = JObject.Parse(json);
decimal red = obj["Color"]["Red"].Value<decimal>(); // 0.8
```

支持路径查询（如 `obj.SelectToken("Color.Red")`）

* 动态构建与修改 `JSON`

```csharp
JObject root = new JObject();
root["Users"] = new JArray();
(root["Users"] as JArray).Add(new JObject { ["Name"] = "Bob" });
root["Users"][0]["Age"] = 25; // 修改值
root.Remove("Users"); // 删除节点
```

### 序列化设置与高级配置

#### 格式化输出

```csharp
string formattedJson = JsonConvert.SerializeObject(person, Formatting.Indented);
// 输出:
// {
//   "Name": "John",
//   "Age": 30,
//   "BirthDate": "1990-01-01T00:00:00"
// }
```

#### 忽略空值属性

```csharp
var settings = new JsonSerializerSettings {
    NullValueHandling = NullValueHandling.Ignore
};

Person personWithNull = new Person { Name = "Alice", Age = 0, BirthDate = null };
string jsonWithoutNulls = JsonConvert.SerializeObject(personWithNull, settings);
// 输出: {"Name":"Alice"}
```

#### 日期格式自定义

```csharp
var settings = new JsonSerializerSettings {
    DateFormatString = "yyyy-MM-dd"
};

string jsonWithCustomDate = JsonConvert.SerializeObject(person, settings);
// 输出: {"Name":"John","Age":30,"BirthDate":"1990-01-01"}
```

#### 属性重命名（CamelCase）

```csharp
var settings = new JsonSerializerSettings {
    ContractResolver = new CamelCasePropertyNamesContractResolver()
};

string jsonWithCamelCase = JsonConvert.SerializeObject(person, settings);
// 输出: {"name":"John","age":30,"birthDate":"1990-01-01T00:00:00"}
```

### 属性特性与自定义序列化

#### 常用属性特性

```csharp
public class Product {
    [JsonProperty("product_id")]  // 自定义 JSON 属性名
    public int Id { get; set; }
    
    [JsonIgnore]  // 忽略该属性
    public string Secret { get; set; }
    
    [JsonRequired]  // 反序列化时必须存在该属性
    public string Name { get; set; }
    
    [JsonConverter(typeof(StringEnumConverter))]  // 自定义转换器
    public ProductType Type { get; set; }

    [JsonExtensionData] // 捕获额外字段
    public Dictionary<string, JToken> Extra { get; set; }
}

public enum ProductType {
    Book,
    Electronics,
    Clothing
}
```

#### 自定义转换器

```csharp
// 自定义日期转换器
public class CustomDateTimeConverter : JsonConverter<DateTime> {
    public override DateTime ReadJson(JsonReader reader, Type objectType, DateTime existingValue, bool hasExistingValue, JsonSerializer serializer) {
        return DateTime.Parse(reader.Value.ToString());
    }

    public override void WriteJson(JsonWriter writer, DateTime value, JsonSerializer serializer) {
        writer.WriteValue(value.ToString("yyyy-MM-dd"));
    }
}

// 使用自定义转换器
public class Order {
    [JsonConverter(typeof(CustomDateTimeConverter))]
    public DateTime OrderDate { get; set; }
}
```

### 高性能与流式处理

#### 使用 JsonTextReader/JsonTextWriter

```csharp
using (var stringWriter = new StringWriter())
using (var jsonWriter = new JsonTextWriter(stringWriter)) {
    jsonWriter.Formatting = Formatting.Indented;
    
    jsonWriter.WriteStartObject();
    jsonWriter.WritePropertyName("Name");
    jsonWriter.WriteValue("John");
    jsonWriter.WritePropertyName("Age");
    jsonWriter.WriteValue(30);
    jsonWriter.WriteEndObject();
    
    string json = stringWriter.ToString();
}

// 流式读取
using (var stringReader = new StringReader(json))
using (var jsonReader = new JsonTextReader(stringReader)) {
    while (jsonReader.Read()) {
        Console.WriteLine($"Token: {jsonReader.TokenType}, Value: {jsonReader.Value}");
    }
}
```

#### 使用预编译序列化器

避免频繁创建 `JsonSerializerSettings`：复用设置实例

```csharp
var serializer = JsonSerializer.CreateDefault();
serializer.TypeNameHandling = TypeNameHandling.Auto;

using (var writer = new StringWriter()) {
    serializer.Serialize(writer, person);
    string json = writer.ToString();
}
```

#### 缓存 JsonSerializerSettings 或 JsonSerializer

```csharp
private static readonly JsonSerializerSettings Settings = new JsonSerializerSettings
{
    NullValueHandling = NullValueHandling.Ignore
};
```

### 处理复杂场景

#### 处理循环引用

```csharp
public class Employee {
    public string Name { get; set; }
    public Employee Manager { get; set; }
    public List<Employee> Subordinates { get; set; }
}

// 设置引用处理
var settings = new JsonSerializerSettings {
    ReferenceLoopHandling = ReferenceLoopHandling.Ignore
};

string json = JsonConvert.SerializeObject(employee, settings);
```

#### 多态序列化

```csharp
public abstract class Animal {
    public string Name { get; set; }
}

public class Dog : Animal {
    public string Breed { get; set; }
}

// 使用 TypeNameHandling
var settings = new JsonSerializerSettings {
    TypeNameHandling = TypeNameHandling.Auto
};

string json = JsonConvert.SerializeObject(dog, settings);
// 输出包含类型信息: {"$type":"YourNamespace.Dog, YourAssembly","Name":"Buddy","Breed":"Labrador"}
```

#### 部分序列化与反序列化

```csharp
// 只序列化部分属性
var settings = new JsonSerializerSettings {
    ContractResolver = new DynamicContractResolver(new[] { "Name", "Age" })
};

string partialJson = JsonConvert.SerializeObject(person, settings);
// 输出: {"Name":"John","Age":30}
```

#### 枚举值序列化为字符串

```csharp
[JsonConverter(typeof(StringEnumConverter))]
public enum OrderStatus { Pending, Shipped }
```

#### 多态类型处理

```csharp
[JsonConverter(typeof(JsonSubtypes), "Type")]
[JsonSubtypes.KnownSubType(typeof(Cat), "cat")]
[JsonSubtypes.KnownSubType(typeof(Dog), "dog")]
public class Animal { public string Type { get; set; } }
```

### 全局配置与 JsonSerializerSettings

通过 `JsonConvert.DefaultSettings` 或在调用 `SerializeObject/DeserializeObject` 时传入 `JsonSerializerSettings`，可定制全局行为：

```csharp
var settings = new JsonSerializerSettings
{
    Formatting = Formatting.Indented,               // 缩进输出
    NullValueHandling = NullValueHandling.Ignore,   // 忽略 null 值
    ReferenceLoopHandling = ReferenceLoopHandling.Ignore, // 循环引用忽略
    MissingMemberHandling = MissingMemberHandling.Error, // 遇到多余字段抛异常
    DateFormatString = "yyyy-MM-dd HH:mm:ss",       // 全局日期格式
    ContractResolver = new CamelCasePropertyNamesContractResolver(), // 驼峰命名
    Converters = new List<JsonConverter>
    {
        new StringEnumConverter(),                  // 枚举按字符串处理
        new IsoDateTimeConverter()                  // ISO 格式日期
    }
};

string json = JsonConvert.SerializeObject(obj, settings);
var obj2 = JsonConvert.DeserializeObject<MyType>(json, settings);
```

常用设置一览：

| 设置项                       | 说明                                                                  |
| ---------------------------- | --------------------------------------------------------------------- |
| `Formatting`                 | `None` / `Indented`                                                   |
| `NullValueHandling`          | `Include` / `Ignore`                                                  |
| `DefaultValueHandling`       | 控制默认值（如 `0`、`false`）是否输出                                 |
| `ReferenceLoopHandling`      | 循环引用时 `Error` / `Ignore` / `Serialize`                           |
| `PreserveReferencesHandling` | 保留对象引用关系（`$id` / `$ref`）                                    |
| `TypeNameHandling`           | 序列化 `$type`，方便反序列化到正确类型                                |
| `ContractResolver`           | 控制属性命名、包含规则（如 `CamelCasePropertyNamesContractResolver`） |
| `Converters`                 | 自定义或内置转换器列表                                                |

#### 属性特性

可以在模型上通过属性微调序列行为：

| 特性                                     | 用途                           |
| ---------------------------------------- | ------------------------------ |
| `[JsonProperty("json_name", Order = n)]` | 指定 JSON 字段名、序列顺序     |
| `[JsonIgnore]`                           | 忽略该属性或字段               |
| `[JsonConverter(typeof(MyConverter))]`   | 为该成员或类型指定转换器       |
| `[JsonRequired]`                         | 反序列化时如果缺少该字段则报错 |
| `[JsonExtensionData]`                    | 捕获多余或未知字段到字典       |
| `[JsonConstructor]`                      | 指定用哪个构造器进行反序列化   |

**示例**

```csharp
public class Person
{
    [JsonProperty("id")]
    public int Id { get; set; }

    [JsonProperty("full_name", Order = 1)]
    public string Name { get; set; }

    [JsonIgnore]
    public int InternalFlag { get; set; }

    [JsonExtensionData]
    public Dictionary<string, JToken> Extra { get; set; }
}
```

### 定制转换器（JsonConverter）

#### 自定义简单转换器

```csharp
public class UnixDateTimeConverter : JsonConverter<DateTime>
{
    public override void WriteJson(JsonWriter writer, DateTime value, JsonSerializer serializer)
    {
        long unix = new DateTimeOffset(value).ToUnixTimeSeconds();
        writer.WriteValue(unix);
    }
    public override DateTime ReadJson(JsonReader reader, Type objectType, DateTime existingValue, bool hasExistingValue, JsonSerializer serializer)
    {
        long unix = (long)reader.Value;
        return DateTimeOffset.FromUnixTimeSeconds(unix).DateTime;
    }
}
```

注册使用：

```csharp
settings.Converters.Add(new UnixDateTimeConverter());
```

或属性级别：

```csharp
public class Event
{
    [JsonConverter(typeof(UnixDateTimeConverter))]
    public DateTime Timestamp { get; set; }
}
```

### 常见问题

#### 日期格式问题

使用 `DateTimeZoneHandling` 设置时区处理方式

```csharp
var settings = new JsonSerializerSettings {
    DateTimeZoneHandling = DateTimeZoneHandling.Utc
};
```

#### 反序列化时忽略额外属性

```csharp
var settings = new JsonSerializerSettings {
    MissingMemberHandling = MissingMemberHandling.Ignore
};
```

#### 处理特殊字符

```csharp
var settings = new JsonSerializerSettings {
    StringEscapeHandling = StringEscapeHandling.EscapeNonAscii
};
```

#### 错误处理

```csharp
var settings = new JsonSerializerSettings
{
    Error = (sender, args) =>
    {
        Console.WriteLine($"Error: {args.ErrorContext.Error.Message}");
        args.ErrorContext.Handled = true; // 标记为已处理
    }
};

string invalidJson = @"{""Id"":1,""Name"":123}"; // Name 应为字符串
User user = JsonConvert.DeserializeObject<User>(invalidJson, settings);
```

### Newtonsoft.Json 与现代化技术栈整合

#### ASP.NET Core 集成

```csharp
// Startup.cs
services.AddControllers().AddNewtonsoftJson(options => 
{
    options.SerializerSettings.ContractResolver = new CamelCaseContractResolver();
    options.SerializerSettings.Converters.Add(new StringEnumConverter());
});
```

#### EF Core 值转换器

```csharp
modelBuilder.Entity<Product>()
    .Property(p => p.Metadata)
    .HasConversion(
        v => JsonConvert.SerializeObject(v),
        v => JsonConvert.DeserializeObject<Dictionary<string, object>>(v)
    );
```