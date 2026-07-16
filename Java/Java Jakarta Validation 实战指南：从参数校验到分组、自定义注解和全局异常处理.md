### 简介

Jakarta Validation 是 Java 生态里用于数据校验的标准规范。

它最常见的使用方式，就是在请求对象、实体对象、方法参数上加注解：

```java
public record CreateUserRequest(
        @NotBlank(message = "用户名不能为空")
        String username,

        @Email(message = "邮箱格式不正确")
        String email,

        @Min(value = 18, message = "年龄不能小于18岁")
        Integer age
) {
}
```

然后在接口入口处触发校验：

```java
@PostMapping("/users")
public Long create(@Valid @RequestBody CreateUserRequest request) {
    return userService.create(request);
}
```

请求参数不符合规则时，Spring 会在进入业务方法前抛出校验异常。

它的价值很直接：把参数规则写在模型上，把错误返回集中处理，让业务代码少写一堆重复的 `if` 判断。

### Jakarta Validation、Hibernate Validator、Spring Boot 的关系

这几个名字经常一起出现，但角色不一样。

| 名称 | 角色 | 说明 |
| --- | --- | --- |
| Jakarta Validation | 规范 | 定义注解、API、校验模型 |
| Hibernate Validator | 实现 | Jakarta Validation 的常见参考实现 |
| Spring Boot | 集成方 | 自动配置 Validator，并在 Web、Service 等场景触发校验 |

简单理解：

```text
Jakarta Validation 定规则
Hibernate Validator 负责执行规则
Spring Boot 负责把校验接到 Web 请求、方法调用和异常处理流程里
```

包名也有一个历史变化：

| 时代 | 包名 |
| --- | --- |
| Java EE / Bean Validation 老项目 | `javax.validation.*` |
| Jakarta EE / Spring Boot 3+ | `jakarta.validation.*` |

Spring Boot 3 开始整体切到 Jakarta 命名空间，所以代码里通常使用：

```java
import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
```

### Maven 依赖

Spring Boot 项目通常直接引入 validation starter：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

它会带上 Jakarta Validation API 和 Hibernate Validator。

如果是普通 Java 项目，可以直接引入 Hibernate Validator：

```xml
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>${hibernate-validator.version}</version>
</dependency>
```

部分 Java SE 场景还需要表达式语言实现，用于处理更复杂的消息插值：

```xml
<dependency>
    <groupId>org.glassfish.expressly</groupId>
    <artifactId>expressly</artifactId>
    <version>${expressly.version}</version>
</dependency>
```

Spring Boot 项目优先交给 Boot 的依赖管理，不建议在业务项目里随意手写 Hibernate Validator 版本。

### 第一个 Spring Boot Demo

先准备一个创建用户请求对象。

```java
package com.example.user.dto;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Pattern;
import jakarta.validation.constraints.Size;

public record CreateUserRequest(

        @NotBlank(message = "用户名不能为空")
        @Size(min = 2, max = 20, message = "用户名长度需要在2到20之间")
        String username,

        @NotBlank(message = "邮箱不能为空")
        @Email(message = "邮箱格式不正确")
        String email,

        @NotNull(message = "年龄不能为空")
        @Min(value = 18, message = "年龄不能小于18岁")
        @Max(value = 100, message = "年龄不能大于100岁")
        Integer age,

        @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
        String phone
) {
}
```

Controller：

```java
package com.example.user.controller;

import com.example.user.dto.CreateUserRequest;
import com.example.user.dto.UserCreateResponse;
import com.example.user.service.UserService;
import jakarta.validation.Valid;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @PostMapping
    public UserCreateResponse create(@Valid @RequestBody CreateUserRequest request) {
        Long userId = userService.create(request);
        return new UserCreateResponse(userId);
    }
}
```

响应对象：

```java
package com.example.user.dto;

public record UserCreateResponse(Long userId) {
}
```

Service：

```java
package com.example.user.service;

import com.example.user.dto.CreateUserRequest;
import org.springframework.stereotype.Service;

@Service
public class UserService {

    public Long create(CreateUserRequest request) {
        return System.currentTimeMillis();
    }
}
```

请求参数不合法时，Controller 方法不会继续执行，Spring 会抛出绑定异常。

### 常用注解

常用约束可以按类型记。

| 注解 | 适用场景 | 说明 |
| --- | --- | --- |
| `@NotNull` | 任意对象 | 不能为 `null` |
| `@NotEmpty` | 字符串、集合、数组、Map | 不能为 `null`，长度或大小不能为 0 |
| `@NotBlank` | 字符串 | 不能为 `null`，去掉空白后不能是空字符串 |
| `@Size` | 字符串、集合、数组、Map | 限制长度或大小 |
| `@Min` / `@Max` | 整数、长整数等数字 | 限制最小值和最大值 |
| `@DecimalMin` / `@DecimalMax` | `BigDecimal` 等数字 | 适合金额、比例 |
| `@Positive` | 数字 | 需要大于 0 |
| `@PositiveOrZero` | 数字 | 需要大于等于 0 |
| `@Negative` | 数字 | 需要小于 0 |
| `@Digits` | 数字 | 限制整数位和小数位 |
| `@Email` | 字符串 | 邮箱格式 |
| `@Pattern` | 字符串 | 正则表达式 |
| `@Past` | 日期时间 | 应为过去时间 |
| `@PastOrPresent` | 日期时间 | 应为过去或当前时间 |
| `@Future` | 日期时间 | 应为未来时间 |
| `@FutureOrPresent` | 日期时间 | 应为未来或当前时间 |
| `@AssertTrue` | 布尔值 | 应为 `true` |
| `@AssertFalse` | 布尔值 | 应为 `false` |

几个注解很容易混：

| 注解 | `null` | 空字符串 `""` | 空白字符串 `"   "` |
| --- | --- | --- | --- |
| `@NotNull` | 不通过 | 通过 | 通过 |
| `@NotEmpty` | 不通过 | 不通过 | 通过 |
| `@NotBlank` | 不通过 | 不通过 | 不通过 |

用户名、标题、备注这类字符串必填字段，通常用 `@NotBlank`。

集合必填并且至少有一个元素，通常用 `@NotEmpty`。

数字、日期、对象必填，通常用 `@NotNull`。

### @Valid 和 @Validated 的区别

`@Valid` 来自 Jakarta Validation：

```java
import jakarta.validation.Valid;
```

`@Validated` 来自 Spring：

```java
import org.springframework.validation.annotation.Validated;
```

常见区别如下：

| 注解 | 来源 | 常见用途 |
| --- | --- | --- |
| `@Valid` | Jakarta Validation | 触发对象校验、级联校验 |
| `@Validated` | Spring | 触发分组校验、方法参数校验 |

JSON 请求体校验通常这样写：

```java
@PostMapping
public UserCreateResponse create(@Valid @RequestBody CreateUserRequest request) {
    return new UserCreateResponse(1001L);
}
```

如果要使用分组校验，通常用 `@Validated`：

```java
@PostMapping
public UserCreateResponse create(
        @Validated(CreateGroup.class) @RequestBody UserRequest request
) {
    return new UserCreateResponse(1001L);
}
```

如果要校验 `@RequestParam`、`@PathVariable` 这种普通参数，Controller 类或方法通常需要加 `@Validated`：

```java
package com.example.user.controller;

import jakarta.validation.constraints.Min;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@Validated
@RestController
@RequestMapping("/api/users")
public class UserQueryController {

    @GetMapping("/{id}")
    public String getById(@PathVariable @Min(value = 1, message = "用户ID需要大于0") Long id) {
        return "user-" + id;
    }
}
```

### 全局异常处理

参数校验失败后，默认错误响应通常不适合直接给前端使用。

可以用 `@RestControllerAdvice` 统一包装错误结果。

先定义统一错误对象：

```java
package com.example.common;

import java.util.List;

public record ApiErrorResponse(
        String code,
        String message,
        List<FieldErrorItem> errors
) {
}
```

字段错误对象：

```java
package com.example.common;

public record FieldErrorItem(
        String field,
        String message
) {
}
```

全局异常处理器：

```java
package com.example.common;

import jakarta.validation.ConstraintViolation;
import jakarta.validation.ConstraintViolationException;
import org.springframework.http.HttpStatus;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.method.annotation.HandlerMethodValidationException;

import java.util.ArrayList;
import java.util.List;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiErrorResponse handleRequestBodyValidException(MethodArgumentNotValidException ex) {
        List<FieldErrorItem> errors = ex.getBindingResult()
                .getFieldErrors()
                .stream()
                .map(this::toFieldErrorItem)
                .toList();

        return new ApiErrorResponse("VALIDATION_FAILED", "参数校验失败", errors);
    }

    @ExceptionHandler(ConstraintViolationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiErrorResponse handleConstraintViolationException(ConstraintViolationException ex) {
        List<FieldErrorItem> errors = ex.getConstraintViolations()
                .stream()
                .map(this::toFieldErrorItem)
                .toList();

        return new ApiErrorResponse("VALIDATION_FAILED", "参数校验失败", errors);
    }

    @ExceptionHandler(HandlerMethodValidationException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ApiErrorResponse handleHandlerMethodValidationException(HandlerMethodValidationException ex) {
        List<FieldErrorItem> errors = new ArrayList<>();

        ex.visitResults(new HandlerMethodValidationException.Visitor() {
            @Override
            public void other(org.springframework.validation.method.ParameterValidationResult result) {
                result.getResolvableErrors().forEach(error -> {
                    String field = result.getMethodParameter().getParameterName();
                    errors.add(new FieldErrorItem(field, error.getDefaultMessage()));
                });
            }
        });

        return new ApiErrorResponse("VALIDATION_FAILED", "参数校验失败", errors);
    }

    private FieldErrorItem toFieldErrorItem(FieldError error) {
        return new FieldErrorItem(error.getField(), error.getDefaultMessage());
    }

    private FieldErrorItem toFieldErrorItem(ConstraintViolation<?> violation) {
        return new FieldErrorItem(
                violation.getPropertyPath().toString(),
                violation.getMessage()
        );
    }
}
```

常见异常来源：

| 异常 | 常见来源 |
| --- | --- |
| `MethodArgumentNotValidException` | `@RequestBody` 对象校验失败 |
| `ConstraintViolationException` | Service 方法参数校验、部分普通参数校验 |
| `HandlerMethodValidationException` | Spring MVC 方法参数校验失败 |

不同 Spring 版本和注解位置可能抛出不同异常，统一处理时可以一起覆盖。

### 分组校验

新增用户和修改用户经常有不同规则。

比如新增时没有 `id`，修改时需要传 `id`。

先定义两个分组接口：

```java
package com.example.user.validation;

public interface CreateGroup {
}
```

```java
package com.example.user.validation;

public interface UpdateGroup {
}
```

请求对象：

```java
package com.example.user.dto;

import com.example.user.validation.CreateGroup;
import com.example.user.validation.UpdateGroup;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;

public record UserRequest(

        @NotNull(message = "用户ID不能为空", groups = UpdateGroup.class)
        Long id,

        @NotBlank(message = "用户名不能为空", groups = {CreateGroup.class, UpdateGroup.class})
        @Size(min = 2, max = 20, message = "用户名长度需要在2到20之间", groups = {CreateGroup.class, UpdateGroup.class})
        String username,

        @NotBlank(message = "邮箱不能为空", groups = CreateGroup.class)
        @Email(message = "邮箱格式不正确", groups = {CreateGroup.class, UpdateGroup.class})
        String email
) {
}
```

Controller：

```java
package com.example.user.controller;

import com.example.user.dto.UserRequest;
import com.example.user.validation.CreateGroup;
import com.example.user.validation.UpdateGroup;
import org.springframework.validation.annotation.Validated;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api/users")
public class UserGroupController {

    @PostMapping
    public String create(@Validated(CreateGroup.class) @RequestBody UserRequest request) {
        return "创建成功";
    }

    @PutMapping
    public String update(@Validated(UpdateGroup.class) @RequestBody UserRequest request) {
        return "修改成功";
    }
}
```

分组适合规则差异比较明确的场景。

如果新增和修改字段差异很大，拆成 `CreateUserRequest` 和 `UpdateUserRequest` 两个 DTO 通常更清楚。

### 级联校验

对象里嵌套对象时，需要在嵌套字段上加 `@Valid`。

地址对象：

```java
package com.example.order.dto;

import jakarta.validation.constraints.NotBlank;

public record AddressRequest(

        @NotBlank(message = "省份不能为空")
        String province,

        @NotBlank(message = "城市不能为空")
        String city,

        @NotBlank(message = "详细地址不能为空")
        String detail
) {
}
```

订单对象：

```java
package com.example.order.dto;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;

public record CreateOrderRequest(

        @NotBlank(message = "订单号不能为空")
        String orderNo,

        @Valid
        @NotNull(message = "收货地址不能为空")
        AddressRequest address
) {
}
```

没有 `@Valid` 时，只会校验 `address` 是否为 `null`，不会继续校验 `AddressRequest` 里面的 `province`、`city`、`detail`。

### 集合和泛型元素校验

Jakarta Validation 支持容器元素校验。

比如订单明细列表：

```java
package com.example.order.dto;

import jakarta.validation.Valid;
import jakarta.validation.constraints.NotEmpty;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Positive;

import java.util.List;

public record SubmitOrderRequest(

        @NotEmpty(message = "订单明细不能为空")
        List<@Valid OrderItemRequest> items,

        List<@NotNull(message = "优惠券ID不能为空") Long> couponIds
) {
}
```

明细对象：

```java
package com.example.order.dto;

import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Positive;

public record OrderItemRequest(

        @NotNull(message = "商品ID不能为空")
        Long skuId,

        @NotNull(message = "购买数量不能为空")
        @Positive(message = "购买数量需要大于0")
        Integer count
) {
}
```

几个细节：

| 写法 | 含义 |
| --- | --- |
| `@NotEmpty List<OrderItemRequest> items` | 列表本身不能为空，且至少有一个元素 |
| `List<@Valid OrderItemRequest> items` | 校验列表里每个对象 |
| `List<@NotNull Long> couponIds` | 校验列表里每个 ID 不能为 `null` |

### Service 方法参数校验

校验不只发生在 Controller。

Service 方法参数也可以校验，常见于内部服务、定时任务、消息消费入口。

```java
package com.example.user.service;

import jakarta.validation.Valid;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import org.springframework.stereotype.Service;
import org.springframework.validation.annotation.Validated;

@Validated
@Service
public class UserQueryService {

    public String getUsername(@Min(value = 1, message = "用户ID需要大于0") Long userId) {
        return "user-" + userId;
    }

    public void changeNickname(
            @Min(value = 1, message = "用户ID需要大于0") Long userId,
            @NotBlank(message = "昵称不能为空") String nickname
    ) {
        // 更新昵称
    }

    public void create(@Valid CreateUserCommand command) {
        // 创建用户
    }
}
```

命令对象：

```java
package com.example.user.service;

import jakarta.validation.constraints.NotBlank;

public record CreateUserCommand(
        @NotBlank(message = "用户名不能为空")
        String username
) {
}
```

Service 类上加 `@Validated` 后，Spring 会通过代理触发方法校验。

同一个类内部直接调用本类方法时，可能绕过代理，方法校验不会触发。需要把被校验的方法放到另一个 Spring Bean，或者通过代理对象调用。

### 返回值校验

方法返回值也可以加约束。

```java
package com.example.user.service;

import jakarta.validation.constraints.NotNull;
import org.springframework.stereotype.Service;
import org.springframework.validation.annotation.Validated;

@Validated
@Service
public class UserProfileService {

    @NotNull(message = "用户资料不能为空")
    public UserProfile findProfile(Long userId) {
        return new UserProfile(userId, "Tom");
    }
}
```

返回对象：

```java
package com.example.user.service;

public record UserProfile(
        Long userId,
        String username
) {
}
```

返回值校验更适合内部服务契约，不适合滥用。很多业务场景下，返回为空本身可能就是合法结果。

### 自定义校验注解

内置注解覆盖不了所有业务规则。

比如手机号格式，可以封装成一个注解。

先定义注解：

```java
package com.example.validation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Documented
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PhoneNumberValidator.class)
public @interface PhoneNumber {

    String message() default "手机号格式不正确";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

再写校验器：

```java
package com.example.validation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;

import java.util.regex.Pattern;

public class PhoneNumberValidator implements ConstraintValidator<PhoneNumber, String> {

    private static final Pattern PHONE_PATTERN = Pattern.compile("^1[3-9]\\d{9}$");

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null || value.isBlank()) {
            return true;
        }
        return PHONE_PATTERN.matcher(value).matches();
    }
}
```

使用：

```java
package com.example.user.dto;

import com.example.validation.PhoneNumber;
import jakarta.validation.constraints.NotBlank;

public record BindPhoneRequest(

        @NotBlank(message = "手机号不能为空")
        @PhoneNumber
        String phone
) {
}
```

自定义校验器里通常把 `null` 当作通过。

原因是是否必填应该交给 `@NotNull`、`@NotBlank` 这类注解表达。这样一个 `@PhoneNumber` 既可以用于必填手机号，也可以用于非必填手机号。

### 类级别校验

有些规则需要同时看多个字段。

比如开始时间不能晚于结束时间。

先定义注解：

```java
package com.example.validation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;

import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Documented
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = DateRangeValidator.class)
public @interface ValidDateRange {

    String message() default "开始时间不能晚于结束时间";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

请求对象：

```java
package com.example.activity.dto;

import com.example.validation.ValidDateRange;
import jakarta.validation.constraints.FutureOrPresent;
import jakarta.validation.constraints.NotNull;

import java.time.LocalDateTime;

@ValidDateRange
public record ActivityRequest(

        @NotNull(message = "开始时间不能为空")
        @FutureOrPresent(message = "开始时间不能早于当前时间")
        LocalDateTime startTime,

        @NotNull(message = "结束时间不能为空")
        @FutureOrPresent(message = "结束时间不能早于当前时间")
        LocalDateTime endTime
) {
}
```

校验器：

```java
package com.example.validation;

import com.example.activity.dto.ActivityRequest;
import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;

public class DateRangeValidator implements ConstraintValidator<ValidDateRange, ActivityRequest> {

    @Override
    public boolean isValid(ActivityRequest value, ConstraintValidatorContext context) {
        if (value == null || value.startTime() == null || value.endTime() == null) {
            return true;
        }
        return !value.startTime().isAfter(value.endTime());
    }
}
```

类级别校验适合跨字段规则。单字段规则仍然放在字段注解上更直观。

### 手动校验

脱离 Spring Web 时，也可以手动使用 `Validator`。

```java
package com.example.validation;

import jakarta.validation.ConstraintViolation;
import jakarta.validation.Validation;
import jakarta.validation.Validator;
import jakarta.validation.ValidatorFactory;

import java.util.Set;

public class ManualValidationDemo {

    public static void main(String[] args) {
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        Validator validator = factory.getValidator();

        RegisterCommand command = new RegisterCommand("", "bad-email");

        Set<ConstraintViolation<RegisterCommand>> violations = validator.validate(command);

        for (ConstraintViolation<RegisterCommand> violation : violations) {
            System.out.println(violation.getPropertyPath() + " -> " + violation.getMessage());
        }
    }
}
```

命令对象：

```java
package com.example.validation;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;

public record RegisterCommand(

        @NotBlank(message = "用户名不能为空")
        String username,

        @Email(message = "邮箱格式不正确")
        String email
) {
}
```

输出大概是：

```text
username -> 用户名不能为空
email -> 邮箱格式不正确
```

### 错误消息和国际化

注解里的 `message` 可以直接写中文：

```java
@NotBlank(message = "用户名不能为空")
private String username;
```

也可以写消息 key：

```java
@NotBlank(message = "{user.username.notBlank}")
private String username;
```

然后在 `src/main/resources/ValidationMessages.properties` 中配置：

```properties
user.username.notBlank=用户名不能为空
user.email.invalid=邮箱格式不正确
```

Spring Boot 也会结合应用的 `MessageSource` 解析消息。项目已经有 `messages.properties`、`messages_zh_CN.properties` 这类国际化文件时，可以把校验文案纳入统一管理。

### Fail Fast：只返回第一个错误

默认情况下，一个对象里多个字段不合法，会返回多个错误。

有些接口只想返回第一个错误，可以开启 Hibernate Validator 的 fail fast。

Spring Boot 里可以注册一个配置：

```java
package com.example.config;

import org.hibernate.validator.HibernateValidatorConfiguration;
import org.springframework.boot.autoconfigure.validation.ValidationConfigurationCustomizer;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class ValidationConfig {

    @Bean
    public ValidationConfigurationCustomizer validationConfigurationCustomizer() {
        return configuration -> {
            if (configuration instanceof HibernateValidatorConfiguration hibernateConfiguration) {
                hibernateConfiguration.failFast(true);
            }
        };
    }
}
```

多错误返回适合表单场景，一次性展示所有字段问题。

单错误返回适合移动端弹窗、命令式接口、对响应体大小比较敏感的场景。

### 和 JPA 实体校验的关系

Jakarta Validation 可以放在 JPA 实体上。

```java
import jakarta.persistence.Entity;
import jakarta.persistence.GeneratedValue;
import jakarta.persistence.Id;
import jakarta.validation.constraints.NotBlank;

@Entity
public class UserEntity {

    @Id
    @GeneratedValue
    private Long id;

    @NotBlank(message = "用户名不能为空")
    private String username;
}
```

Hibernate ORM 在持久化生命周期中也可以触发校验。

不过 Web 接口里更推荐使用专门的请求 DTO：

```text
CreateUserRequest 负责接口入参
UserEntity 负责数据库映射
```

接口字段和数据库字段经常不完全一致。把校验规则全部堆到实体上，后期容易被不同接口场景互相影响。

### 常见问题

#### 引入了注解但校验没有生效

常见原因：

| 原因 | 处理方式 |
| --- | --- |
| 缺少 `spring-boot-starter-validation` | 添加 validation starter |
| `@RequestBody` 前没有 `@Valid` 或 `@Validated` | 在参数前添加触发注解 |
| 普通参数校验缺少 `@Validated` | 在 Controller 或 Service 类上添加 `@Validated` |
| Service 内部调用本类方法 | 通过 Spring 代理调用，或拆到另一个 Bean |
| 嵌套对象缺少 `@Valid` | 在嵌套字段或集合元素上添加 `@Valid` |

#### int 加了 @NotNull 仍然没效果

`int` 是基本类型，默认值是 `0`，不会是 `null`。

如果要表达必填，使用包装类型：

```java
@NotNull(message = "年龄不能为空")
private Integer age;
```

#### @NotBlank 用在 Integer 上报错

`@NotBlank` 只能用于字符串。

数字必填用 `@NotNull`，数值范围用 `@Min`、`@Max`、`@Positive` 等注解。

```java
@NotNull(message = "数量不能为空")
@Positive(message = "数量需要大于0")
private Integer count;
```

#### @Valid 和 BindingResult 的位置

Spring MVC 支持在被校验参数后面紧跟 `BindingResult` 手动处理错误。

```java
@PostMapping
public String create(@Valid @RequestBody CreateUserRequest request,
                     BindingResult bindingResult) {
    if (bindingResult.hasErrors()) {
        return bindingResult.getFieldError().getDefaultMessage();
    }
    return "创建成功";
}
```

这种写法适合少量特殊接口。

大多数 REST API 更适合用全局异常处理器统一返回错误结构。

### 实践建议

| 场景 | 建议 |
| --- | --- |
| Web JSON 入参 | DTO 上写约束，Controller 参数使用 `@Valid` |
| 普通参数 | Controller 类或 Service 类使用 `@Validated` |
| 新增和修改规则差异小 | 使用分组校验 |
| 新增和修改字段差异大 | 拆成不同请求 DTO |
| 嵌套对象 | 嵌套字段上加 `@Valid` |
| 集合元素 | 使用 `List<@Valid Item>` 或 `List<@NotNull Long>` |
| 业务格式规则 | 封装自定义注解 |
| 错误返回 | 使用 `@RestControllerAdvice` 统一处理 |
| JPA 实体 | 避免把所有接口规则都压到实体上 |

### 小结

Jakarta Validation 的核心是声明式校验。

简单字段规则用内置注解，跨字段规则用类级别自定义注解，业务格式规则用自定义约束，接口错误返回交给全局异常处理器。

Spring Boot 项目中，常见组合是：

```text
spring-boot-starter-validation
DTO 约束注解
Controller @Valid / @Validated
Service @Validated
@RestControllerAdvice
```

这样参数校验、业务逻辑和错误返回会分得比较清楚，代码也更容易维护。
