### 简介

`Java 1.5` 引入了注解，现在它在 `Java EE` 框架（如 `Hibernate`、`Jersey` 和 `Spring` ）中被大量使用。`Java` 注释是该语言的一个强大特性，用于向 `Java` 代码中添加元数据。它们不直接影响程序逻辑，但可以由工具、库或框架处理，以执行特定的任务（例如代码生成、验证或自定义处理）。

### 元注解

> 目前有五种类型的元注解

#### `@Documented`：

指示使用此注解的元素应该由 `javadoc` 和类似的工具记录。该类型应该用于注解类型的声明，这些类型的注解会影响其客户端对已注解元素的使用。如果一个类型声明用 `Documented` 进行了注解，那么它的注解将成为被注解元素的公共API的一部分。

#### `@Target`：

表示注解类型适用的程序元素种类。一些可能的值是 `TYPE、METHOD、CONSTRUCTOR、FIELD` 等。如果不存在目标元注解，则注解可用于任何程序元素。

**元素类型对应可以应用的位置：**

* `TYPE`：类、接口、枚举、记录

* `FIELD`：字段、枚举常量

* `METHOD`：方法

* `CONSTRUCTOR`：构造器

* `LOCAL_VARIABLE`：本地变量

* `ANNOTATION_TYPE`：注解接口声明

* `PARAMETER`：参数声明

* `PACKAGE`：包声明

* `TYPE_PARAMETER`：类型参数声明

* `TYPE_USE`：类型的使用

* `MODULE`：模块声明

* `RECORD_COMPONENT`：记录组件声明

**用法示例**

```java
@Target({ElementType.TYPE, ElementType.FIELD, ElementType.METHOD})  
@interface MyAnnotation{  
    int value1();  
    String value2();  
}  
```

#### `@Inherited`：

表示自动继承注解类型。如果用户查询类声明上的注解类型，而类声明没有此类型的注解，则将自动查询该类的超类以获取注解类型。这个过程将重复进行，直到找到该类型的注解，或者到达类层次结构的顶端（Object）

#### `@Retention`：

表示注解类型的注解要保留多长时间。它采用 `RetentionPolicy` 枚举中的参数，其可能值为 `SOURCE、CLASS 和 RUNTIME`。

**保留策略可用的值：**

* `RetentionPolicy.SOURCE`：源代码可用，在编译期间被丢弃，它在编译后的类中不可用。

* `RetentionPolicy.CLASS`：`java` 编译器可以访问，但 `JVM` 无法访问，它包含在 `class` 文件中。

* `RetentionPolicy.RUNTIME`：运行时可用，可供 `Java` 编译器和 `JVM` 使用。

#### `@Repeatable`：

用于指示其注解的类型声明是可重复的。

### 内建的注解

* `@Override`：

当想要重写一个 `Superclass` 的方法时，应该使用这个注解来通知编译器我们正在重写一个方法。因此，当父类方法被删除或更改时，编译器将显示错误消息。

* `@Deprecated`：

当我们想让编译器知道某个方法已被弃用时，我们应该使用此注释。

* `@SuppressWarnings`：

这只是为了告诉编译器忽略它们产生的特定警告，例如在 `Java` 泛型中使用原始类型。它的保留策略是 `SOURCE`，它会被编译器丢弃。

* `@FunctionalInterface`：

此注释是在 `Java 8` 中引入的，用于表明该接口旨在成为一个功能接口。

* `@SafeVarargs`：

程序员断言，注解方法或构造函数的主体不会对其 `varargs` 参数执行潜在的不安全操作。

### 使用元注解和内建注解示例

```java
public class AnnotationExample {
	public static void main(String[] args) {
	}

	@Override
	@MethodInfo(author = "Pankaj", comments = "Main method", date = "Nov 17 2012", revision = 1)
	public String toString() {
		return "Overriden toString method";
	}

	@Deprecated
	@MethodInfo(comments = "deprecated method", date = "Nov 17 2012")
	public static void oldMethod() {
		System.out.println("old method, don't use it.");
	}

	@SuppressWarnings({ "unchecked", "deprecation" })
	@MethodInfo(author = "Pankaj", comments = "Main method", date = "Nov 17 2012", revision = 10)
	public static void genericsTest() throws FileNotFoundException {
		List l = new ArrayList();
		l.add("abc");
		oldMethod();
	}
}
```

### 注解的类型

* 标记注释

> 没有方法的注解称为标记注解，例如：@Override 和 @Deprecated 是标记注释

```java
@interface MyAnnotation{} 
```

* 单值注解

> 只有一个方法的注解称为单值注解

```java
@interface MyAnnotation{  
    int value() default 0;  
} 

// 应用注解
@MyAnnotation(value=10)  
```

* 多值注解

> 具有多个方法的注释称为多值注释

```java
@interface MyAnnotation{  
    int value1() default 1;  
    String value2() default "";  
    String value3() default "abc";  
}  

// 应用注解
@MyAnnotation(value1=10,value2="Arun Kumar",value3="Ghaziabad")  
```

### 自定义注解示例

**先定义三个注解**

* `JsonSerializable`

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface JsonSerializable {
}
```

* `JsonElement`

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.FIELD)
public @interface JsonElement {
    String key() default "";
}
```

* `Init`

```java
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface Init {
}
```

**接着定义Person实体**

```java
import com.redmine.annotationexample.annotation.Init;
import com.redmine.annotationexample.annotation.JsonElement;
import com.redmine.annotationexample.annotation.JsonSerializable;

@JsonSerializable
public class Person {

    @JsonElement
    private String firstName;

    @JsonElement
    private String lastName;

    @JsonElement(key = "personAge")
    private int age;

    private String address;

    @Init
    private void initNames() {
        this.firstName = this.firstName.substring(0, 1).toUpperCase() + this.firstName.substring(1);
        this.lastName = this.lastName.substring(0, 1).toUpperCase() + this.lastName.substring(1);
    }

    public Person(String firstName, String lastName, int age) {
        this.firstName = firstName;
        this.lastName = lastName;
        this.age = age;
    }
}
```

**定义一个异常类**

```java
public class JsonSerializationException extends RuntimeException {
    public JsonSerializationException(String message) {
        super(message);
    }
}
```

**编写注解处理器**

```java
import com.redmine.annotationexample.exception.JsonSerializationException;

import java.lang.annotation.Annotation;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;
import java.util.Objects;
import java.util.stream.Collectors;

public class ObjectToJsonConverter {

    public String convertObjectToJson(Object obj) throws JsonSerializationException {
        try {
            checkIfSerializable(obj);
            initializeObject(obj);
            return getJsonString(obj);
        } catch (Exception e) {
            throw new JsonSerializationException(e.getMessage());
        }
    }

    private void checkIfSerializable(Object obj) {
        if (Objects.isNull(obj)) {
            throw new JsonSerializationException("Object is null");
        }

        Class<?> clazz = obj.getClass();
        if (!clazz.isAnnotationPresent(JsonSerializable.class)) {
            throw new JsonSerializationException("The class" + clazz.getName() + " is not annotated with JsonSerializable");
        }
    }

    private void initializeObject(Object obj) throws Exception {
        Class<?> clazz = obj.getClass();
        for (Method method : clazz.getDeclaredMethods()) {
            if (method.isAnnotationPresent(Init.class)) {
                method.setAccessible(true);
                method.invoke(obj);
            }
        }
    }

    private String getJsonString(Object obj) throws Exception {
        Class<?> clazz = obj.getClass();
        Map<String, String> jsonElementsMap = new HashMap<>();
        for (Field field : clazz.getDeclaredFields()) {
            field.setAccessible(true);
            if (field.isAnnotationPresent(JsonElement.class)) {
                JsonElement annotation = field.getAnnotation(JsonElement.class);
                String mapKey = !Objects.equals(annotation.key(), "") ? annotation.key() : field.getName();
                jsonElementsMap.put(mapKey, field.get(obj).toString());
            }
        }

        return jsonElementsMap.entrySet()
                .stream()
                .map(entry -> "\"" + entry.getKey() + "\":\"" + entry.getValue() + "\"")
                .collect(Collectors.joining(",", "{", "}"));
    }
}
```

**单元测试运行试下**

```java
@SpringBootTest
class AnnotationExampleApplicationTests {

    @Test
    void contextLoads() {
    }

    @Test
    void givenObjectSerializedTheTrueReturned() throws JsonSerializationException {
        Person person = new Person("soufiane", "cheouati", 34);
        ObjectToJsonConverter serializer = new ObjectToJsonConverter();
        String jsonString = serializer.convertObjectToJson(person);
        Assertions.assertEquals("{\"personAge\":\"34\",\"firstName\":\"Soufiane\",\"lastName\":\"Cheouati\"}", jsonString);
    }
}
```

### JDK 注解定义

![alt text](/images/Java/注解-image.png)