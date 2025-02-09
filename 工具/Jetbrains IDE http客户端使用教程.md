### 简介

`JetBrains IDE`（如IntelliJ IDEA， WebStorm， PhpStorm和PyCharm）自带一个内置的HTTP客户端，允许直接从IDE发送HTTP请求，而无需使用第三方工具，如Postman或cURL。

### JetBrains IDE 中的 HTTP 客户端是什么？

`JetBrains IDE` 中的HTTP客户端是一个轻量级但功能强大的功能，它允许开发人员直接从IDE中发送HTTP请求（GET， POST， PUT， DELETE等）。
它支持REST API测试、GraphQL请求、WebSocket通信和环境变量。

**主要特点**

* 在 `.http` 或 `.rest` 文件中编写并执行 HTTP 请求

* 支持 `REST API` 请求（GET、POST、PUT、DELETE 等）

* 支持 `GraphQL` 和 `WebSocket` 请求

* 允许设置自定义请求头、请求参数和请求体

* 支持身份验证、文件上传和环境变量

* 保存请求历史记录以供调试

### 基本语法

```http
### Comment (optional)
HTTP_METHOD URL [QUERY_PARAMS]
HEADER_1: VALUE
HEADER_2: VALUE
(Empty line)
BODY (optional)
```

### 示例用法

#### 创建简单的 GET 请求

```http
### Fetch data from a public API
GET https://jsonplaceholder.typicode.com/posts/1
Accept: application/json
```

* `GET`：查询数据的 `HTTP` 方法

* `https://jsonplaceholder.typicode.com/posts/1`：API 接口

* `Accept: application/json`：可接受的响应类型

#### 带有 JSON 主体的 POST 请求

```http
### Create a new post
POST https://jsonplaceholder.typicode.com/posts
Content-Type: application/json

{
  "title": "My New Post",
  "body": "This is the content of the post.",
  "userId": 1
}
```

* `POST`：发送数据的 `HTTP` 方法

* `Content-Type: application/json`：指定请求主体是 `JSON`

* `JSON body`：发送到 API 的实际数据

#### PUT 请求（更新数据）

```http
### Update a post
PUT https://jsonplaceholder.typicode.com/posts/1
Content-Type: application/json

{
  "id": 1,
  "title": "Updated Title",
  "body": "Updated content.",
  "userId": 1
}
```

#### DELETE 请求

```http
### Delete a post
DELETE https://jsonplaceholder.typicode.com/posts/1
```

#### 在 .http 文件中定义变量

> 可以在 .env 文件中或直接在 .http 请求文件中定义变量

```http
### Use variables in the request
GET {{baseUrl}}/posts/1
Accept: application/json

> {%
    baseUrl = "https://jsonplaceholder.typicode.com"
%}
```

* `{{baseUrl}}`：使用定义的变量而不是硬编码 URL

* `{% %}`：语法块定义变量 baseUrl

#### 使用环境变量

> 不需要直接在 .http 文件中定义变量，而是可以将变量存储在单独的文件中

* 创建一个 `.env` 文件（例如，项目根目录中的 `http-client.env.json`）

* 定义环境和变量

```json
{
  "development": {
    "baseUrl": "https://jsonplaceholder.typicode.com",
    "authToken": "Bearer my-secret-token"
  },
  "production": {
    "baseUrl": "https://api.example.com",
    "authToken": "Bearer prod-secret-token"
  }
}
```

* 在 `.http` 请求中使用环境

```http
### Fetch posts using environment variables
GET {{baseUrl}}/posts
Authorization: {{authToken}}
```

* 切换环境：单击 `.http` 文件右上角的环境下拉菜单，然后选择开发或生产

#### 使用 Bearer Token 身份验证

```http
GET https://api.example.com/protected-resource
Authorization: Bearer my-secret-token
```

#### 使用基本身份验证

```http
GET https://api.example.com/private-data
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

#### 发送表单数据

```http
POST https://api.example.com/login
Content-Type: application/x-www-form-urlencoded

username=myUser&password=myPass
```

#### 上传文件

```http
POST https://api.example.com/upload
Content-Type: multipart/form-data; boundary=boundary

--boundary
Content-Disposition: form-data; name="file"; filename="myfile.txt"
Content-Type: text/plain

< ./myfile.txt
--boundary--
```

#### 使用 GraphQL 请求

```http
### GraphQL query request
POST https://api.example.com/graphql
Content-Type: application/json

{
  "query": "query { user(id: 1) { id, name, email } }"
}
```

#### 打开 WebSocket 连接

```http
GET ws://echo.websocket.org
```

### 图示

![切换环境](/images/工具/http-client-image.png)
![请求示例](/images/工具/http-client-image-1.png)