### 简介

在 `Spring` 中，`ResponseEntity` 是 `HTTP` 响应的包装器。它允许自定义响应的各个方面：

* HTTP 状态码

* 响应主体

* HTTP 请求头

使用 `ResponseEntity` 允许完全控制 `HTTP` 响应，并且它通常用于 `RESTful Web` 服务中从控制器方法返回响应。

### 基本语法

```java
ResponseEntity<T> response = new ResponseEntity<>(body, headers, status);
```

* `T`：响应主体的类型

* `body`：想要作为响应主体发送的对象（如果不想返回主体，则可以为空）

* `headers`：想要包含的任何其他 `HTTP` 请求头

* `status`：HTTP 状态代码（如 `HttpStatus.OK、HttpStatus.CREATED`等）

### 示例用法

#### 基本用法：返回简单响应

```java
@RestController
@RequestMapping("/api/posts")
public class PostController {

    @GetMapping("/{id}")
    public ResponseEntity<Post> getPost(@PathVariable Long id) {
        Post post = postService.findById(id);
        if (post != null) {
            return new ResponseEntity<>(post, HttpStatus.OK);  // 200 OK
        } else {
            return new ResponseEntity<>(HttpStatus.NOT_FOUND);  // 404 Not Found
        }
    }
}
```

#### 返回带有请求头的 `ResponseEntity`

```java
@GetMapping("/custom-header")
public ResponseEntity<String> getWithCustomHeader() {
    HttpHeaders headers = new HttpHeaders();
    headers.add("Custom-Header", "CustomValue");

    return new ResponseEntity<>("Hello with custom header!", headers, HttpStatus.OK);
}
```

#### 返回具有创建状态的 ResponseEntity

> 创建新资源时，通常希望返回 `201 Created` 状态代码

```java
@PostMapping("/create")
public ResponseEntity<Post> createPost(@RequestBody Post post) {
    Post createdPost = postService.save(post);
    URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(createdPost.getId())
            .toUri();

    return ResponseEntity.created(location).body(createdPost);
}
```

#### 返回没有内容的 ResponseEntity

> 当成功处理一个请求但不需要返回任何内容（例如，一个 `DELETE` 请求）时，可以使用 `204 No Content`

```java
@DeleteMapping("/{id}")
public ResponseEntity<Void> deletePost(@PathVariable Long id) {
    boolean isDeleted = postService.delete(id);
    if (isDeleted) {
        return new ResponseEntity<>(HttpStatus.NO_CONTENT);  // 204 No Content
    } else {
        return new ResponseEntity<>(HttpStatus.NOT_FOUND);   // 404 Not Found
    }
}
```

#### 使用带有异常处理的 ResponseEntity

> 可以在全局异常处理程序或控制器中使用 ResponseEntity 来处理异常

```java
@ExceptionHandler(PostNotFoundException.class)
public ResponseEntity<String> handlePostNotFound(PostNotFoundException ex) {
    return new ResponseEntity<>(ex.getMessage(), HttpStatus.NOT_FOUND);
}
```

#### 使用 Map 返回 ResponseEntity（例如，对于 JSON 响应）

```java
@GetMapping("/user/{id}")
public ResponseEntity<Map<String, Object>> getUser(@PathVariable Long id) {
    Map<String, Object> response = new HashMap<>();
    User user = userService.findById(id);

    if (user != null) {
        response.put("status", "success");
        response.put("data", user);
        return new ResponseEntity<>(response, HttpStatus.OK);
    } else {
        response.put("status", "error");
        response.put("message", "User not found");
        return new ResponseEntity<>(response, HttpStatus.NOT_FOUND);
    }
}
```

#### 具有泛型类型的 ResponseEntity

```java
@GetMapping("/posts/{id}")
public ResponseEntity<Post> getPostById(@PathVariable Long id) {
    Post post = postService.findById(id);
    if (post != null) {
        return ResponseEntity.ok(post);  // 200 OK with Post object as body
    }
    return ResponseEntity.status(HttpStatus.NOT_FOUND).build();  // 404 Not Found with no body
}

// ResponseEntity.ok(post) 是 new ResponseEntity<>(post, HttpStatus.OK) 的简写
```

#### 返回验证错误的 ResponseEntity

```java
@PostMapping("/validate")
public ResponseEntity<Map<String, String>> validateUser(@RequestBody User user, BindingResult result) {
    if (result.hasErrors()) {
        Map<String, String> errorResponse = new HashMap<>();
        result.getFieldErrors().forEach(error -> errorResponse.put(error.getField(), error.getDefaultMessage()));
        return new ResponseEntity<>(errorResponse, HttpStatus.BAD_REQUEST);  // 400 Bad Request
    }
    userService.save(user);
    return new ResponseEntity<>(HttpStatus.CREATED);  // 201 Created
}
```

#### 使用统一的响应对象

* 定义统一响应对象

```java
public class ApiResponse<T> {

    private String status;
    private String message;
    private T data;
    private ErrorDetails error;

    // Constructor for success response
    public ApiResponse(String status, String message, T data) {
        this.status = status;
        this.message = message;
        this.data = data;
    }

    // Constructor for error response
    public ApiResponse(String status, String message, ErrorDetails error) {
        this.status = status;
        this.message = message;
        this.error = error;
    }

    // Getters and setters
}

class ErrorDetails {
    private String timestamp;
    private int status;
    private String error;
    private String path;

    // Getters and setters
}
```

* 在控制器方法中使用统一响应

```java
@GetMapping("/posts/{id}")
public ResponseEntity<ApiResponse<Post>> getPostById(@PathVariable Long id) {
    Post post = postService.findById(id);
    if (post != null) {
        ApiResponse<Post> response = new ApiResponse<>(
            "success", 
            "Post retrieved successfully", 
            post
        );
        return new ResponseEntity<>(response, HttpStatus.OK);
    } else {
        return getErrorResponse(HttpStatus.NOT_FOUND, "Post not found", "/api/posts/" + id);
    }
}
```

```java
private ResponseEntity<ApiResponse<Post>> getErrorResponse(HttpStatus status, String message, String path) {
    ErrorDetails errorDetails = new ErrorDetails();
    errorDetails.setTimestamp(LocalDateTime.now().toString());
    errorDetails.setStatus(status.value());
    errorDetails.setError(status.getReasonPhrase());
    errorDetails.setPath(path);

    ApiResponse<Post> response = new ApiResponse<>(
        "error",
        message,
        errorDetails
    );

    return new ResponseEntity<>(response, status);
}
```

**响应数据结构示例**

* `Success`

```json
{
  "status": "success",
  "message": "Post retrieved successfully",
  "data": {
    "id": 1,
    "title": "Hello World",
    "content": "This is my first post"
  }
}
```

* `Error`

```json
{
  "status": "error",
  "message": "Post not found",
  "error": {
    "timestamp": "2025-02-07T06:43:41.111+00:00",
    "status": 404,
    "error": "Not Found",
    "path": "/api/posts/1"
  }
}
```

* 使用 `@ControllerAdvice` 全局统一处理异常

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    // Handle all exceptions
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ApiResponse<Object>> handleGeneralException(Exception ex) {
        ErrorDetails errorDetails = new ErrorDetails();
        errorDetails.setTimestamp(LocalDateTime.now().toString());
        errorDetails.setStatus(HttpStatus.INTERNAL_SERVER_ERROR.value());
        errorDetails.setError("Internal Server Error");
        errorDetails.setPath("/api/posts");

        ApiResponse<Object> response = new ApiResponse<>("error", ex.getMessage(), errorDetails);
        return new ResponseEntity<>(response, HttpStatus.INTERNAL_SERVER_ERROR);
    }
}
```

### 常用的 HTTP 状态码

* `HttpStatus.OK`：200 OK

* `HttpStatus.CREATED`：201 Created

* `HttpStatus.NO_CONTENT`：204 No Content

* `HttpStatus.BAD_REQUEST`：400 Bad Request

* `HttpStatus.UNAUTHORIZED`：401 Unauthorized

* `HttpStatus.FORBIDDEN`：403 Forbidden

* `HttpStatus.NOT_FOUND`：404 Not Found

* `HttpStatus.INTERNAL_SERVER_ERROR`：500 Internal Server Error