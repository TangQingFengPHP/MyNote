# Yii2 Applications 详解：应用生命周期、组件配置与多入口实战

Yii2 项目里经常出现 `Yii::$app->request`、`Yii::$app->db`、`Yii::$app->user`。这些对象都由 Application 统一管理。理解 Application，才能看明白 Yii 从入口脚本启动、加载配置、创建组件，到解析请求并发送响应的完整过程。

本文以 Yii 2.0 为例，讲解 Web Application 和 Console Application 的区别、入口脚本、应用配置、组件、模块、启动组件和生命周期，并用一套小型 Web/Console 配置示例把概念连起来。

## Application 是什么？

Application 是 Yii 应用运行时的总协调对象。每个应用系统通常创建一个 Application 实例，用来管理当前运行环境、组件、模块、事件和请求生命周期。

Yii 有两种常见 Application：

| 类 | 处理内容 | 常见入口 |
| --- | --- | --- |
| `yii\web\Application` | HTTP 请求 | `web/index.php` |
| `yii\console\Application` | 命令行任务 | 项目根目录的 `yii` |

两者都继承 `yii\base\Application`。Application 又继承 Module 和 Component，因此也拥有模块、组件、行为和事件等能力。

```text
yii\base\Component
        ↑
yii\base\Module
        ↑
yii\base\Application
     ↙          ↘
Web Application  Console Application
```

`Yii::$app` 是访问当前 Application 的全局入口。Web 请求中通常指向 `yii\web\Application`；执行控制台命令时则指向 `yii\console\Application`。它是静态访问方式，不意味着所有运行场景都共享同一个对象。

## Application 负责哪些事情？

Application 的主要工作包括：

- 读取并应用启动配置；
- 注册和管理应用组件，例如 `request`、`db`、`cache`、`user`；
- 管理模块和控制器路由；
- 运行启动阶段需要执行的模块或组件；
- 触发应用级事件；
- Web 场景下处理 HTTP 请求并发送响应，Console 场景下解析并运行命令。

应用结构可以这样理解：

```text
Entry Script
    ↓ 创建 Application
Application
    ├── Components：db、cache、request、user、log...
    ├── Modules：admin、api、debug...
    ├── Params / Aliases / Environment
    └── Lifecycle：启动、处理请求、发送响应
```

## 从入口脚本启动 Web Application

标准 Yii2 Web 入口通常是 `web/index.php`：

```php
<?php

defined('YII_DEBUG') or define('YII_DEBUG', true);
defined('YII_ENV') or define('YII_ENV', 'dev');

require __DIR__ . '/../vendor/autoload.php';
require __DIR__ . '/../vendor/yiisoft/yii2/Yii.php';

$config = require __DIR__ . '/../config/web.php';

(new yii\web\Application($config))->run();
```

入口脚本主要做四件事：

1. 定义调试和环境常量；
2. 加载 Composer 自动加载器和 Yii 核心类；
3. 读取 Web Application 配置；
4. 创建 Application 并调用 `run()`。

Web 服务器的 DocumentRoot 应指向 `web/`，这样配置、源码、运行文件和依赖目录不会直接暴露在公网。

## 创建一份 Web Application 配置

配置通常放在 `config/web.php`。下面是一个精简示例：

```php
<?php

use yii\caching\FileCache;
use yii\db\Connection;

$params = require __DIR__ . '/params.php';

return [
    'id' => 'shop-web',
    'basePath' => dirname(__DIR__),
    'controllerNamespace' => 'app\controllers',
    'language' => 'zh-CN',
    'timeZone' => 'Asia/Shanghai',

    'aliases' => [
        '@uploads' => '@app/web/uploads',
    ],

    'bootstrap' => ['log'],

    'components' => [
        'request' => [
            // 本地开发示例值；正式环境应使用独立随机密钥并从环境配置读取。
            'cookieValidationKey' => 'replace-with-a-long-random-secret',
            'parsers' => [
                'application/json' => 'yii\web\JsonParser',
            ],
        ],
        'response' => [
            'format' => yii\web\Response::FORMAT_HTML,
            'charset' => 'UTF-8',
        ],
        'cache' => [
            'class' => FileCache::class,
        ],
        'db' => [
            'class' => Connection::class,
            'dsn' => 'mysql:host=127.0.0.1;dbname=shop',
            'username' => 'app',
            'password' => 'change-me',
            'charset' => 'utf8mb4',
        ],
        'urlManager' => [
            'enablePrettyUrl' => true,
            'showScriptName' => false,
            'enableStrictParsing' => false,
            'rules' => [],
        ],
        'user' => [
            'identityClass' => 'app\models\User',
            'enableAutoLogin' => true,
        ],
        'errorHandler' => [
            'errorAction' => 'site/error',
        ],
    ],

    'modules' => [
        'admin' => [
            'class' => 'app\modules\admin\Module',
        ],
    ],

    'params' => $params,
];
```

实际项目中，数据库账号、Cookie 验证密钥等不应使用示例里的占位值。可以拆到 `db.php`、环境变量或部署系统的密钥配置中，再由 `web.php` 合并配置。

### 常见 Application 配置项

| 属性 | 作用 |
| --- | --- |
| `id` | 应用标识，通常用于区分不同 Application |
| `basePath` | 应用根目录，很多默认别名和路径从这里派生 |
| `controllerNamespace` | 默认控制器命名空间 |
| `components` | 注册数据库、缓存、请求等应用服务 |
| `modules` | 注册带独立 MVC 结构的功能模块 |
| `bootstrap` | 指定启动时立即初始化的组件、模块或类 |
| `params` | 保存普通应用参数，通过 `Yii::$app->params` 读取 |
| `aliases` | 定义路径别名，例如 `@app`、`@runtime`、`@uploads` |
| `defaultRoute` | 未指定路由时使用的默认控制器/动作 |
| `catchAll` | Web 维护模式下把所有请求转到指定路由 |

Web Application 至少需要 `id` 和 `basePath`。Console Application 也需要自己的标识和根路径，但组件和默认控制器命名空间可能不同。

## `Yii::$app->db`：Application Component 和 Service Locator

`components` 是 Application 最常用的配置区域。每个键都是组件 ID，值可以是类名或配置数组：

```php
'components' => [
    'cache' => [
        'class' => yii\caching\FileCache::class,
    ],
    'db' => [
        'class' => yii\db\Connection::class,
        'dsn' => 'mysql:host=127.0.0.1;dbname=shop',
        'username' => 'app',
        'password' => 'change-me',
    ],
],
```

之后可通过组件 ID 访问：

```php
$db = Yii::$app->db;
$cache = Yii::$app->cache;
$request = Yii::$app->request;
```

应用组件通常是**按需创建**：第一次访问 `Yii::$app->db` 时，Yii 根据配置创建 Connection；同一个 Application 后续访问会复用该实例。需要在启动阶段就初始化的组件，可以放进 `bootstrap`。

Application 在这里充当 Service Locator（服务定位器）：根据 ID 返回对应服务。使用全局定位器很方便，但业务类大量直接读取 `Yii::$app` 会让依赖藏在方法内部，不利于单元测试。复用性高的服务可以通过构造函数显式接收依赖；控制器和少量框架入口代码使用 `Yii::$app` 则很常见。

## 组件、模块和参数有什么区别？

- **Component** 提供一项服务，例如数据库连接、缓存、邮件发送器。
- **Module** 是一块相对独立的应用结构，通常有自己的控制器、模型、视图和子模块，例如 `admin`、`api`。
- **Params** 是一组普通配置值，不是组件，也不会提供服务方法。

示例：

```php
$language = Yii::$app->language;
$pageSize = Yii::$app->params['pageSize'];
$cache = Yii::$app->cache;
$adminModule = Yii::$app->getModule('admin');
```

模块可以通过路由前缀访问，例如 `admin/user/index`。是否提前加载模块取决于它是否需要参与应用启动阶段；只有需要注册 URL 规则、事件监听器等启动行为时才放进 `bootstrap`。

## `bootstrap`：哪些内容需要应用启动时就加载？

`bootstrap` 列表中的组件或模块会在 Application 初始化阶段被创建。常见用途是注册路由规则、事件监听器或全请求都需要的基础设施。

```php
'bootstrap' => [
    'log',
    'admin',
    app\bootstrap\AppBootstrap::class,
],
```

自定义启动类可以实现 `yii\base\BootstrapInterface`：

```php
<?php

namespace app\bootstrap;

use Yii;
use yii\base\BootstrapInterface;

class AppBootstrap implements BootstrapInterface
{
    public function bootstrap($app)
    {
        $app->on(\yii\base\Application::EVENT_BEFORE_REQUEST, function () {
            Yii::info('开始处理请求', 'application');
        });
    }
}
```

BootstrapInterface 的 `bootstrap()` 会在每次应用实例启动时调用。Web PHP-FPM 传统模式下一次请求通常创建一次 Application；常驻应用服务器中则可能复用进程，因此启动阶段和请求阶段的状态边界需要格外清楚。

启动列表不是“所有组件清单”。过多的启动组件会让每次请求都多做初始化工作；没有必须提前运行的服务，应保持懒加载。

## Web Application 生命周期

调用 `(new yii\web\Application($config))->run()` 后，主要过程如下：

```text
应用对象创建
   ↓
应用配置写入对象属性
   ↓
初始化扩展和 bootstrap 组件
   ↓
触发 EVENT_BEFORE_REQUEST
   ↓
解析路由并执行 Controller Action
   ↓
触发 EVENT_AFTER_REQUEST
   ↓
发送 Response
```

Application 事件包括 `EVENT_BEFORE_REQUEST` 和 `EVENT_AFTER_REQUEST`。可以通过事件处理器记录请求级信息或执行轻量的全局处理：

```php
use yii\base\Application;

$config['on ' . Application::EVENT_BEFORE_REQUEST] = function ($event) {
    Yii::info('请求开始', 'application');
};

$config['on ' . Application::EVENT_AFTER_REQUEST] = function ($event) {
    Yii::info('请求处理结束', 'application');
};
```

事件处理器也可以绑定到 Application 实例：

```php
$app = new yii\web\Application($config);
$app->on(yii\base\Application::EVENT_BEFORE_REQUEST, function ($event) {
    Yii::info('请求开始', 'application');
});
$app->run();
```

入口脚本中绑定的事件必须在 `run()` 之前完成。若处理器需要参与早期启动，例如注册 URL 规则，应该使用 BootstrapInterface，而不是等请求开始事件才处理。

## Console Application：另一套入口和配置

Yii 的命令行入口通常是项目根目录的 `yii`：

```php
#!/usr/bin/env php
<?php

defined('YII_DEBUG') or define('YII_DEBUG', true);
defined('YII_ENV') or define('YII_ENV', 'dev');

require __DIR__ . '/vendor/autoload.php';
require __DIR__ . '/vendor/yiisoft/yii2/Yii.php';

$config = require __DIR__ . '/config/console.php';

$application = new yii\console\Application($config);
exit($application->run());
```

`config/console.php` 通常复用 Web 配置中的数据库和业务参数，但不会配置 `request`、`response`、`user` 这类 Web 专属服务：

```php
<?php

$db = require __DIR__ . '/db.php';
$params = require __DIR__ . '/params.php';

return [
    'id' => 'shop-console',
    'basePath' => dirname(__DIR__),
    'controllerNamespace' => 'app\commands',
    'components' => [
        'db' => $db,
        'cache' => [
            'class' => yii\caching\FileCache::class,
        ],
    ],
    'params' => $params,
];
```

因此，同一个进程入口不会同时把 `Yii::$app` 当作 Web 和 Console Application。高级模板会拆出 frontend、backend 和 console 多个应用，它们可共享 common 目录中的代码，但各自拥有独立配置和 Application 实例。

## 环境配置：把共同配置和差异拆开

项目配置可以组合多个 PHP 数组，减少 Web、Console 和环境之间的重复：

```php
<?php

$common = require __DIR__ . '/common.php';
$web = require __DIR__ . '/web-local.php';

return yii\helpers\ArrayHelper::merge($common, $web);
```

常见拆分方式：

```text
config/
├── common.php       # Web 和 Console 共用的组件与参数
├── web.php          # Web Application 配置
├── console.php      # Console Application 配置
├── web-local.php    # 本机或部署环境的 Web 覆盖项
├── console-local.php
├── db.php           # 数据库连接配置
└── params.php       # 普通应用参数
```

基础模板和高级模板的目录安排不同，配置合并方式也会随模板变化。环境差异应放在本地配置或环境变量中；密码、密钥等敏感值不应提交到公共版本库。

## `params`、`aliases` 和环境常量

### `params`

`params` 用于保存一般配置值，例如分页大小、客服邮箱或业务开关：

```php
// config/params.php
return [
    'pageSize' => 20,
    'supportEmail' => 'support@example.com',
];
```

读取方式：

```php
$pageSize = Yii::$app->params['pageSize'];
```

参数不是密钥保险箱。数据库密码、签名密钥等敏感内容应从部署环境的秘密配置中读取。

### `aliases`

路径别名让代码不必依赖绝对路径：

```php
'aliases' => [
    '@uploads' => '@app/web/uploads',
],
```

后续可使用 `Yii::getAlias('@uploads')` 解析为真实路径。框架内置 `@app`、`@vendor`、`@runtime` 等常用别名。

### `YII_DEBUG` 和 `YII_ENV`

入口脚本通常定义环境常量：

```php
defined('YII_DEBUG') or define('YII_DEBUG', false);
defined('YII_ENV') or define('YII_ENV', 'prod');
```

开发环境可启用详细错误和调试模块；生产环境应关闭调试信息，避免堆栈和配置细节暴露。`YII_ENV` 常用于区分配置分支，例如只在开发环境加载 Gii 和 Debug 模块。

## Yii2 Advanced Template 的多个 Application

高级模板通常包含 frontend、backend、console 三个应用：

- `frontend` 面向普通访问者；
- `backend` 面向管理后台；
- `console` 负责计划任务、队列和部署命令；
- `common` 放多个应用共享的 Model、组件或配置。

这不是一个 Application 根据 URL 自动切换模式，而是多份入口脚本分别创建不同 Application。Web 前端、管理后台、Console 可以共享业务代码，同时分别配置路由、认证、组件和环境参数。

## 一个控制器中的 Application 使用示例

Application 提供的组件在控制器和服务中都可以访问：

```php
<?php

namespace app\controllers;

use Yii;
use yii\web\Controller;

class SiteController extends Controller
{
    public function actionStatus()
    {
        $db = Yii::$app->db;
        $cache = Yii::$app->cache;
        $requestId = Yii::$app->request->get('requestId');

        Yii::info('状态接口被调用', 'application');

        return $this->asJson([
            'application' => Yii::$app->id,
            'environment' => YII_ENV,
            'request_id' => $requestId,
            'database_ready' => $db->isActive,
            'cache_enabled' => $cache !== null,
        ]);
    }
}
```

这里展示的是组件入口，不代表控制器应当直接承担所有服务编排。复杂业务可以拆到 Service，再通过应用容器配置或构造函数注入维护依赖边界。

## 常见问题

### 为什么 `Yii::$app->db` 第一次访问比较慢？

应用组件默认延迟创建。首次读取组件时，Yii 要根据配置创建对象并完成初始化；后续访问同一个 Application 中的该组件会复用实例。数据库连接也可能在第一次真正执行 SQL 时才建立。

### 为什么 bootstrap 里不建议放很多组件？

启动组件会在 Application 初始化时提前创建或运行。Web 请求通常每次都要创建 Application，启动列表越重，请求开销越大。只把需要参与整个应用生命周期的模块或组件放入列表。

### `Yii::$app->params` 和 `Yii::$app->components` 有什么区别？

`params` 是普通键值配置；`components` 是按 ID 注册的服务对象，可能有初始化过程、方法和状态。

### Web 和 Console 能使用同一个配置吗？

可以共享数据库连接、缓存、日志和业务参数等共同配置，但 Web 专属组件和 Console 专属组件应分别配置。通常做法是拆出公共配置，再由各自入口组合。

### Advanced Template 为什么有多个 `Yii::$app`？

frontend、backend 和 console 是不同入口创建的不同 Application。每个进程执行时只有当前入口对应的 Application 作为 `Yii::$app`。

## 小结

Yii2 Application 管理应用配置、组件、模块和生命周期。Web 与 Console 使用不同 Application 类和入口脚本；`Yii::$app` 提供当前 Application 的访问入口；`components` 按需提供服务；`bootstrap` 负责少量必须在启动时运行的内容。

掌握这几个部分后，应用如何从 `index.php` 进入 Controller、如何读取全局服务、如何拆分环境配置，以及 Advanced Template 为何有多个应用入口，就能串成一条完整链路。

## 参考资料

- [Yii 2.0 Applications 官方指南](https://www.yiiframework.com/doc/guide/2.0/en/structure-applications)
- [Yii 2.0 Application Components](https://www.yiiframework.com/doc/guide/2.0/en/structure-application-components)
- [Yii 2.0 Bootstrapping](https://www.yiiframework.com/doc/guide/2.0/en/runtime-bootstrapping)
- [Yii 2.0 Application API](https://www.yiiframework.com/doc/api/2.0/yii-base-application)
- [Yii2 Advanced Template 应用结构](https://www.yiiframework.com/extension/yiisoft/yii2-app-advanced/doc/guide/2.0/en/structure-applications)
