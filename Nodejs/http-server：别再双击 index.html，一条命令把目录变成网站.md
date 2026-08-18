# 别再双击 index.html：http-server 把本地目录变成网站的实战详解

前端打包结果、静态 HTML、接口 Mock 数据、图片目录，很多时候只差一个 HTTP 服务就能正常运行。直接双击 `index.html` 得到的是 `file://` 地址，而 `fetch()`、ES Module、路由回退等功能通常需要 `http://` 环境。

`http-server` 就是为这个场景准备的工具：不需要编写服务器代码，不需要配置 Nginx，执行一条命令即可把某个本地目录作为静态网站提供访问。

本文覆盖安装、目录映射、常用参数、缓存、跨域、SPA 刷新、代理、HTTPS，以及一个原生 Node.js HTTP 服务 Demo。命令以 `http-server` 14.x 的常用行为为准，具体参数仍可通过 `http-server --help` 查看。

## 一、http-server 到底解决了什么问题？

假设目录如下：

```text
demo/
├── index.html
├── app.js
├── style.css
└── data.json
```

直接打开页面时，浏览器地址类似这样：

```text
file:///Users/panfeng/demo/index.html
```

执行：

```bash
cd demo
npx http-server
```

目录会被挂载到一个 HTTP 服务上：

```text
http://localhost:8080/index.html  ->  demo/index.html
http://localhost:8080/app.js      ->  demo/app.js
http://localhost:8080/data.json   ->  demo/data.json
```

浏览器访问 `http://localhost:8080` 时，服务器优先寻找当前目录下的 `index.html`。找不到目标文件时，通常返回 404；目录列表和 404 页面也可以按参数或特殊文件进行定制。

一句话概括：

```text
http-server = 本地目录 + HTTP 访问入口
```

它主要提供静态文件，不负责数据库、用户登录、业务路由和数据写入。需要这些能力时，应使用 Express、Fastify、NestJS 或其他后端框架。

## 二、安装与启动

### 1. 临时运行：适合快速预览

无需把命令安装到全局环境：

```bash
npx http-server ./dist
```

首次执行可能下载 npm 包。项目打包后预览时，`./dist`、`./build` 是最常见的目标目录。

### 2. 全局安装：适合经常使用

```bash
npm install --global http-server
http-server --version
```

安装完成后，任意目录都可以执行：

```bash
http-server
```

路径参数默认为当前目录；若当前目录下存在 `public` 文件夹，官方 CLI 会优先把 `public` 作为服务目录。

### 3. 写进项目脚本

```bash
npm install --save-dev http-server
```

`package.json`：

```json
{
  "scripts": {
    "preview": "http-server ./dist -p 4173 -c-1"
  }
}
```

执行：

```bash
npm run preview
```

`-c-1` 表示关闭缓存，特别适合反复修改打包文件后检查结果的场景。

## 三、从零完成一个可访问页面

创建目录：

```bash
mkdir http-server-demo
cd http-server-demo
```

创建 `index.html`：

```html
<!doctype html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <title>http-server Demo</title>
  <link rel="stylesheet" href="/style.css">
</head>
<body>
  <main>
    <h1>静态页面已经跑起来了</h1>
    <button id="loadButton">读取 JSON</button>
    <pre id="result">等待请求……</pre>
  </main>
  <script src="/app.js"></script>
</body>
</html>
```

创建 `style.css`：

```css
body { margin: 0; background: #f5f7fb; font: 16px sans-serif; }
main { max-width: 680px; margin: 80px auto; padding: 32px; background: white; }
button { padding: 8px 14px; cursor: pointer; }
pre { padding: 16px; background: #f0f2f5; }
```

创建 `data.json`：

```json
{
  "project": "http-server",
  "status": "running",
  "port": 8080
}
```

创建 `app.js`：

```javascript
document.querySelector('#loadButton').addEventListener('click', async () => {
  const result = document.querySelector('#result');

  try {
    const response = await fetch('/data.json');
    if (!response.ok) throw new Error(`HTTP ${response.status}`);

    const data = await response.json();
    result.textContent = JSON.stringify(data, null, 2);
  } catch (error) {
    result.textContent = `请求失败：${error.message}`;
  }
});
```

启动：

```bash
http-server . -p 8080
```

打开 `http://localhost:8080`，点击按钮即可看到浏览器通过 HTTP 读取 `data.json` 的结果。这个例子也说明了静态服务器的边界：JSON 文件是提前准备好的，`http-server` 只是把它原样返回，并没有执行查询或生成数据。

## 四、常用参数：真正经常用到的是这些

| 参数 | 作用 | 示例 |
| --- | --- | --- |
| `-p`、`--port` | 指定端口 | `-p 3000` |
| `-a` | 指定监听地址 | `-a 127.0.0.1` |
| `-c` | 设置缓存秒数 | `-c 60` |
| `-c-1` | 关闭缓存 | `-c-1` |
| `-o` | 启动后打开浏览器 | `-o /index.html` |
| `-s`、`--silent` | 不输出访问日志 | `-s` |
| `-d` | 显示目录列表 | `-d` |
| `--cors` | 增加允许跨域访问的响应头 | `--cors` |
| `-e`、`--ext` | 请求没有扩展名时补默认扩展名 | `-e html` |
| `-P`、`--proxy` | 将本地找不到的请求转发到目标地址 | `-P http://localhost:3000` |
| `-S`、`--ssl` | 开启 HTTPS | `-S -C cert.pem -K key.pem` |
| `--no-dotfiles` | 不展示隐藏文件 | `--no-dotfiles` |
| `--log-ip` | 输出客户端 IP | `--log-ip` |

端口被占用时：

```bash
http-server ./dist -p 3000
```

临时寻找可用端口，可以使用：

```bash
http-server ./dist -p 0
```

只允许本机访问时：

```bash
http-server ./dist -a 127.0.0.1
```

局域网设备需要访问时，通常监听 `0.0.0.0`，再使用电脑的局域网 IP 访问：

```text
http://192.168.1.20:8080
```

防火墙、路由器隔离和 Wi-Fi 网络策略也可能影响访问。`0.0.0.0` 是监听地址，不是浏览器里应该输入的目标地址。

## 五、缓存：页面明明改了，浏览器为什么还是旧的？

`http-server` 默认会设置缓存时间，默认值通常为 3600 秒。浏览器可能继续使用旧的 CSS、JavaScript 或 JSON，于是出现“文件已经修改但页面没有变化”。

开发调试直接关闭缓存：

```bash
http-server ./dist -c-1
```

想模拟短缓存：

```bash
http-server ./dist -c 10
```

生产预览则可以保留缓存，并为静态资源增加文件指纹，例如 `app.3f2a1.js`。文件名变化后，浏览器会把它当成新资源，缓存策略更容易控制。

## 六、跨域 Demo：为什么加上 `--cors`？

假设页面运行在 `http://localhost:8080`，接口运行在 `http://localhost:3000`。端口不同，浏览器就会把它们视为不同源。

启动静态目录：

```bash
http-server ./dist -p 8080 --cors
```

`--cors` 会为响应增加允许跨域的响应头，适合本地联调和临时测试。

需要注意：CORS 不是服务器之间的通信限制，而是浏览器的安全策略。命令行工具、服务端程序和 Postman 不会按同样方式拦截。开发环境可以使用 `--cors`，正式环境仍应根据域名、请求方法和请求头精确配置跨域规则，不能把“允许所有来源”当成权限控制。

## 七、单页应用刷新 404：用 `404.html` 做回退

Vue、React 等单页应用常使用前端路由：

```text
/                 -> index.html
/user/profile     -> 前端路由
```

首次从首页进入时，JavaScript 接管路由没有问题；直接刷新 `/user/profile` 时，服务器会尝试寻找 `user/profile` 文件，找不到就返回 404。

一个简单做法是把入口页复制成 `404.html`：

```bash
cp dist/index.html dist/404.html
http-server dist -p 4173
```

找不到静态文件时，`404.html` 会返回入口页面，前端路由随后接管地址。页面中的资源路径建议使用正确的相对路径或构建工具的 `base` 配置，否则深层路径可能导致 CSS、JS 的地址解析错误。

也可以使用代理回退：

```bash
http-server dist --proxy http://localhost:4173?
```

URL 末尾的 `?` 是这个用法里的关键写法。生产环境更适合交给 Nginx、Caddy 或应用服务器配置 `try_files`，因为它们对压缩、缓存、日志和进程管理的控制更完整。

## 八、目录列表、默认文件和特殊文件

目录请求的常见处理顺序可以理解为：

```text
请求 /docs/
    ├─ 有 docs/index.html  -> 返回 index.html
    ├─ 没有入口文件且允许目录列表 -> 展示文件列表
    └─ 找不到目标 -> 返回 404.html 或 404
```

关闭目录列表：

```bash
http-server ./public -d false
```

关闭自动索引展示：

```bash
http-server ./public -i false
```

目录列表在临时文件共享时很方便，但公开访问时可能暴露文件名、构建产物和配置文件。服务前应确认目录中没有 `.env`、私钥、备份文件和内部文档；也可以使用：

```bash
http-server ./public --no-dotfiles
```

## 九、压缩文件、代理与 HTTPS

### 1. 提供预压缩资源

已有 `.gz` 或 `.br` 文件时，可以让服务器根据浏览器的 `Accept-Encoding` 返回预压缩版本：

```bash
http-server ./dist --gzip --brotli
```

这不是实时压缩，而是优先读取已经生成的压缩文件。构建流程需要提前产出对应文件，并保证内容和原文件同步。

### 2. 代理未命中的请求

静态目录里找不到的请求，可以转发到后端：

```bash
http-server ./dist -P http://localhost:3000
```

适合简单的本地联调。复杂的路径重写、鉴权、限流、WebSocket 和多服务路由，建议使用专门的反向代理或开发服务器。

### 3. 开启 HTTPS

准备证书和私钥后：

```bash
http-server ./dist \
  -S \
  -C ./cert.pem \
  -K ./key.pem \
  -p 8443
```

访问：

```text
https://localhost:8443
```

自签名证书通常会触发浏览器警告，只适合本地测试。证书、私钥和包含敏感信息的目录不应随意暴露到公网。

## 十、用 Node.js 原生模块理解 HTTP 服务本质

`http-server` 适合“把现成文件发布出来”。需要动态响应时，可以直接使用 Node.js 内置的 `node:http`：

创建 `server.mjs`：

```javascript
import http from 'node:http';

const server = http.createServer((req, res) => {
  const url = new URL(req.url, `http://${req.headers.host}`);

  if (req.method === 'GET' && url.pathname === '/api/health') {
    const body = JSON.stringify({ ok: true, time: new Date().toISOString() });

    res.writeHead(200, {
      'Content-Type': 'application/json; charset=utf-8',
      'Cache-Control': 'no-store'
    });
    res.end(body);
    return;
  }

  res.writeHead(404, { 'Content-Type': 'application/json; charset=utf-8' });
  res.end(JSON.stringify({ error: 'Not Found' }));
});

server.listen(3000, '127.0.0.1', () => {
  console.log('API running at http://127.0.0.1:3000');
});
```

运行：

```bash
node server.mjs
curl http://127.0.0.1:3000/api/health
```

代码里的几个关键点：

* `req.method` 表示请求方法，例如 `GET`、`POST`。
* `req.url` 是请求路径和查询字符串，需要用 `URL` 解析。
* `res.writeHead()` 设置状态码和响应头。
* `res.end()` 结束响应；没有调用它，请求可能一直处于等待状态。
* 动态接口、鉴权和业务逻辑属于应用服务器职责，不是 `http-server` 的目标。

Node.js 官方 HTTP API 也提供 `request`、`response`、`server.listen()` 等底层能力。`http-server` 可以看成在这些能力之上封装了静态文件查找、MIME 类型、目录展示、缓存、代理和 TLS 等常见工作。

## 十一、常见问题排查

### 1. `command not found: http-server`

没有全局安装，或 npm 的全局 bin 目录不在 `PATH` 中。临时使用：

```bash
npx http-server .
```

也可以改用项目脚本，避免依赖全局环境。

### 2. `EADDRINUSE: address already in use`

端口已被占用。换一个端口：

```bash
http-server . -p 8081
```

macOS 或 Linux 可以查看占用进程：

```bash
lsof -i :8080
```

### 3. 浏览器显示旧文件

先关闭缓存：

```bash
http-server . -c-1
```

再进行强制刷新，并检查开发者工具的 Network 面板是否命中了 `from disk cache`。

### 4. 页面能打开，JavaScript 却加载失败

检查大小写、文件路径和打包后的目录结构。Linux 区分 `App.js` 与 `app.js`，本地 macOS 开发环境中不明显的问题，部署到 Linux 后可能暴露出来。

### 5. 局域网设备访问不了

确认监听地址、操作系统防火墙和网络是否允许设备互通：

```bash
http-server . -a 0.0.0.0 -p 8080
```

浏览器访问电脑的局域网 IP，不要访问 `0.0.0.0`。

## 十二、什么时候适合使用？

适合：

```text
本地预览静态页面
检查前端打包产物
测试 fetch、ES Module 和 Service Worker
临时共享局域网文件
学习 HTTP 请求与响应
```

不适合单独承担：

```text
用户登录与权限控制
数据库读写
复杂业务 API
高并发生产网关
完整的进程守护、监控和日志体系
```

生产环境使用时，至少应明确服务目录、监听地址、缓存策略、目录列表、认证方式和暴露范围。对外提供网站时，通常还需要反向代理、HTTPS 证书、压缩、访问日志、健康检查和进程管理。

## 总结

`http-server` 的价值不在于功能繁多，而在于把“本地文件”和“HTTP 网站”之间的距离缩短到一条命令：

```bash
npx http-server ./dist -p 4173 -c-1
```

记住几个最常用的组合：

```bash
# 预览打包目录
http-server ./dist

# 修改文件时关闭缓存
http-server ./dist -c-1

# 局域网访问
http-server ./dist -a 0.0.0.0 -p 8080

# 本地联调跨域接口
http-server ./dist --cors -P http://localhost:3000
```

只是静态文件预览，`http-server` 足够轻便；开始出现动态数据、用户权限和生产运维要求时，应升级到真正的应用服务器或反向代理方案。

## 参考资料

* [http-server npm 官方说明](https://www.npmjs.com/package/http-server)
* [http-party/http-server GitHub 仓库](https://github.com/http-party/http-server)
* [Node.js HTTP 官方文档](https://nodejs.org/api/http.html)
