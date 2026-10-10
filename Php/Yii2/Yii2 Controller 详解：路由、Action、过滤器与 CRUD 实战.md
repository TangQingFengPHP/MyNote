# Yii2 Controller 详解：路由、Action、过滤器与 CRUD 实战

Yii2 Controller 是 HTTP 请求进入应用后的协调层。它接收请求参数，调用 Model 或服务处理数据，再选择渲染页面、返回 JSON、重定向或其他响应。控制器不该变成“什么都做”的大类：业务规则放在 Model 或 Service，HTML 放在 View，Controller 负责把这些部分接起来。

本文以 Yii 2.0 为例，从 URL 到 Action 的映射讲起，再用文章管理 CRUD 串起请求、过滤器、参数绑定、表单提交和响应返回。

## Controller 在请求流程中的位置

一个 Yii2 Web 请求大致经过这些步骤：

```text
浏览器请求 web/index.php
        ↓
Yii 创建 Application 并解析路由
        ↓
创建 Controller 和 Action
        ↓
执行过滤器与 beforeAction()
        ↓
运行 Action，读取请求并调用 Model / Service
        ↓
渲染 View 或生成其他响应
        ↓
Response 发送给客户端
```

Controller 属于 MVC 中的 C。它负责处理请求和生成响应，不负责替代 Model 承担输入校验、领域规则或复杂业务流程。

## Controller 类和 URL 路由

普通 Web 控制器通常继承 `yii\web\Controller`：

```php
<?php

namespace app\controllers;

use yii\web\Controller;

class SiteController extends Controller
{
    public function actionIndex()
    {
        return $this->render('index');
    }
}
```

Yii 根据路由定位控制器和动作。默认路由格式为：

```text
controller-id/action-id
```

例如：

| URL 路由 | 控制器类 | Action 方法 |
| --- | --- | --- |
| `site/index` | `SiteController` | `actionIndex()` |
| `post/view` | `PostController` | `actionView()` |
| `admin/user/create` | `admin\controllers\UserController` | `actionCreate()` |

`PostController` 的控制器 ID 通常是 `post`，`actionHelloWorld()` 的动作 ID 通常是 `hello-world`。路由名称使用小写字母、数字、下划线和连字符更清晰。

开发阶段常见的查询参数路由：

```text
index.php?r=post/view&id=12
```

启用 URL 美化规则后，可以变成：

```text
/post/view?id=12
```

路由规则由应用 URL Manager 控制，Controller 类名和方法名本身不会自动决定是否启用美化 URL。

## Action：Controller 对外提供的操作

### Inline Action

最常见的 Action 是控制器中的 public 方法，方法名以 `action` 开头：

```php
class PostController extends \yii\web\Controller
{
    public function actionIndex()
    {
        return '文章列表';
    }

    public function actionView($id)
    {
        return '文章编号：' . $id;
    }
}
```

`actionView($id)` 可以从路由参数或请求参数中接收 `id`。通过类型声明或默认值可以让参数定义更清楚：

```php
public function actionView(int $id = 1)
{
    // ...
}
```

Action 参数由 Yii 从路由参数和请求参数中绑定。缺少必需参数或参数名对不上时，通常会在请求处理阶段报错。

只有 public 且以 `action` 开头的方法才会成为 Inline Action。普通 helper 方法应使用 private 或 protected，避免被当成可访问动作。

### Standalone Action

动作需要在多个控制器中复用，或希望将动作实现独立成类时，可以创建 Standalone Action：

```php
<?php

namespace app\actions;

use yii\base\Action;

class HealthAction extends Action
{
    public function run()
    {
        return 'ok';
    }
}
```

在控制器的 `actions()` 中注册：

```php
public function actions()
{
    return [
        'health' => [
            'class' => \app\actions\HealthAction::class,
        ],
    ];
}
```

访问 `site/health` 时，Yii 会创建 `HealthAction` 并调用 `run()`。框架内也有 `yii\web\ErrorAction`、`yii\web\ViewAction` 等独立动作。

## 实战 Demo：文章管理 CRUD

下面建立一个小型文章管理控制器，包含列表、详情、新建、编辑和删除。示例假设存在 `Post` ActiveRecord，数据库表有 `id`、`title`、`content` 字段。

### Post ActiveRecord

```php
<?php

namespace app\models;

use yii\db\ActiveRecord;

class Post extends ActiveRecord
{
    public static function tableName()
    {
        return '{{%post}}';
    }

    public function rules()
    {
        return [
            [['title', 'content'], 'required'],
            [['title'], 'string', 'max' => 200],
            [['content'], 'string'],
        ];
    }
}
```

### PostController

创建 `controllers/PostController.php`：

```php
<?php

namespace app\controllers;

use app\models\Post;
use Yii;
use yii\filters\AccessControl;
use yii\filters\VerbFilter;
use yii\web\Controller;
use yii\web\NotFoundHttpException;
use yii\web\Response;

class PostController extends Controller
{
    public function behaviors()
    {
        return [
            'access' => [
                'class' => AccessControl::class,
                'only' => ['create', 'update', 'delete'],
                'rules' => [
                    [
                        'allow' => true,
                        'roles' => ['@'],
                    ],
                ],
            ],
            'verbs' => [
                'class' => VerbFilter::class,
                'actions' => [
                    'delete' => ['POST'],
                ],
            ],
        ];
    }

    public function actionIndex()
    {
        $posts = Post::find()
            ->orderBy(['id' => SORT_DESC])
            ->all();

        return $this->render('index', [
            'posts' => $posts,
        ]);
    }

    public function actionView($id)
    {
        return $this->render('view', [
            'model' => $this->findModel($id),
        ]);
    }

    public function actionCreate()
    {
        $model = new Post();

        if ($model->load(Yii::$app->request->post()) && $model->save()) {
            return $this->redirect(['view', 'id' => $model->id]);
        }

        return $this->render('create', [
            'model' => $model,
        ]);
    }

    public function actionUpdate($id)
    {
        $model = $this->findModel($id);

        if ($model->load(Yii::$app->request->post()) && $model->save()) {
            return $this->redirect(['view', 'id' => $model->id]);
        }

        return $this->render('update', [
            'model' => $model,
        ]);
    }

    public function actionDelete($id)
    {
        $this->findModel($id)->delete();

        return $this->redirect(['index']);
    }

    protected function findModel($id)
    {
        $model = Post::findOne($id);
        if ($model !== null) {
            return $model;
        }

        throw new NotFoundHttpException('文章不存在');
    }
}
```

控制器里每个动作只做 HTTP 流程协调：查记录、加载输入、调用保存、选择视图或重定向。验证规则在 Post Model，认证权限由过滤器控制，视图只负责页面呈现。

### 表单视图

创建和编辑页面可以共用 `_form.php`：

```php
<?php

use yii\helpers\Html;
use yii\widgets\ActiveForm;

$form = ActiveForm::begin();
?>

<?= $form->field($model, 'title')->textInput(['maxlength' => true]) ?>
<?= $form->field($model, 'content')->textarea(['rows' => 8]) ?>

<div class="form-group">
    <?= Html::submitButton('保存', ['class' => 'btn btn-primary']) ?>
</div>

<?php ActiveForm::end(); ?>
```

`views/post/create.php` 和 `views/post/update.php` 中引入表单：

```php
<?= $this->render('_form', ['model' => $model]) ?>
```

Yii ActiveForm 默认处理表单字段名、CSRF 隐藏字段和模型错误显示。`load()` 只给安全属性赋值，`save()` 默认会验证 Model，通过后再写入数据库。

### 列表和详情视图

`views/post/index.php`：

```php
<?php

use yii\helpers\Html;

foreach ($posts as $post) {
    echo Html::tag('h2', Html::a(
        Html::encode($post->title),
        ['view', 'id' => $post->id]
    ));
}
```

`views/post/view.php`：

```php
<?php

use yii\helpers\Html;

echo Html::tag('h1', Html::encode($model->title));
echo Html::tag('div', nl2br(Html::encode($model->content)));
```

视图输出用户输入内容时应进行 HTML 编码，避免把文章内容直接当作 HTML 执行。

## `behaviors()`：给动作加过滤器

Yii2 的过滤器通过 Controller Behavior 配置，常在 Action 执行前后工作。常用过滤器包括：

- `AccessControl`：按登录状态、角色等控制访问；
- `VerbFilter`：限制动作接受的 HTTP 方法；
- `ContentNegotiator`：根据请求协商响应格式；
- `HttpCache`：处理 HTTP 缓存头；
- `PageCache`：缓存页面输出。

上面的 Demo 规定：

- `index` 和 `view` 不要求登录；
- `create`、`update`、`delete` 只允许已登录用户；
- `delete` 只能通过 POST 请求调用。

`AccessControl` 中的 `@` 表示已登录用户，`?` 表示访客。访问控制只是示例规则，实际系统还要检查用户是否有权操作目标文章，比如只能修改自己创建的记录。

删除操作使用 POST，而不是 GET，是为了避免浏览器预取链接或爬虫访问 URL 时触发数据变更。表单仍应保留 Yii 的 CSRF 保护。

## `beforeAction()` 和 `afterAction()`

少数控制器需要在多个动作前执行同一段轻量逻辑时，可以重写 `beforeAction()`：

```php
public function beforeAction($action)
{
    if (!parent::beforeAction($action)) {
        return false;
    }

    // 当前控制器动作运行前的处理
    return true;
}
```

返回 `false` 会取消动作执行。重写时要先调用父类的 `beforeAction()`，这样基类事件及相关过滤器仍能正常工作。只针对部分动作的认证、HTTP 方法限制等规则，通常写在 `behaviors()` 更直观。

`afterAction($action, $result)` 在动作执行后收到结果，可做统一收尾或调整结果。它不适合替代事务管理，也不应藏入关键业务逻辑。

## Request：读取请求数据

Yii 通过 `request` 应用组件提供请求信息：

```php
$request = Yii::$app->request;

$queryParams = $request->get();
$postParams = $request->post();
$id = $request->get('id');
$method = $request->getMethod();
$isPost = $request->isPost;
$userAgent = $request->userAgent;
```

使用 `get()`、`post()` 等接口比直接读 `$_GET`、`$_POST` 更清楚，也方便按框架方式测试和配置。

表单请求一般通过 Model 的 `load()` 加载；查询参数常用于筛选和分页。请求数据都属于外部输入，仍需验证和限制，不能因为经过 Request 组件就当成可信数据。

## Response：返回 HTML、JSON 或重定向

### 渲染页面

```php
return $this->render('index', [
    'posts' => $posts,
]);
```

`render()` 使用当前控制器对应的视图文件，通常位于 `views/<controller-id>/<view-name>.php`，并将数据传给视图。

### 重定向

```php
return $this->redirect(['view', 'id' => $model->id]);
```

`redirect()` 返回 Response 对象。使用 `return` 让 Yii 正常完成响应处理，不必手工 `header()` 或 `exit`。

### 返回 JSON

```php
use yii\web\Response;

public function actionApiView($id)
{
    Yii::$app->response->format = Response::FORMAT_JSON;

    $post = $this->findModel($id);

    return [
        'id' => $post->id,
        'title' => $post->title,
    ];
}
```

也可以使用 `$this->asJson($data)`。返回数组并设置 JSON 格式时，Response 组件负责序列化并设置相应的 Content-Type。

### 返回状态码

```php
Yii::$app->response->statusCode = 201;

return [
    'id' => $model->id,
    'message' => '创建成功',
];
```

API 应按语义使用 HTTP 状态码，例如创建成功返回 `201`、参数校验失败返回 `422`、未登录返回 `401`、没有权限返回 `403`、记录不存在返回 `404`。

## REST API Controller

普通页面控制器继承 `yii\web\Controller`。REST API 可继承 `yii\rest\Controller`，若资源基于 ActiveRecord，还可用 `yii\rest\ActiveController` 获得常见 CRUD 动作。

```php
<?php

namespace app\controllers;

use app\models\Post;
use yii\rest\ActiveController;

class PostController extends ActiveController
{
    public $modelClass = Post::class;
}
```

REST 控制器还需要按应用需求配置 URL 规则、认证方式、访问控制、字段过滤和序列化。`ActiveController` 能减少通用 CRUD 代码，但不会自动替代资源级权限检查或业务校验。

## 控制器职责边界

比较清楚的分工是：

```text
Controller：读取 HTTP 输入，选择业务操作，组织响应
Model / Form Model：验证输入和数据规则
Service：编排跨模型的业务流程
ActiveRecord / Repository：读写持久化数据
View：生成 HTML
```

如果一个 Action 同时负责复杂校验、支付流程、库存扣减、邮件发送、数据库事务和 HTML 拼接，后续修改就容易牵一发而动全身。将业务步骤拆到 Model 或 Service，Controller 会更容易阅读和测试。

## 常见问题

### 访问 URL 提示路由不存在

确认控制器文件名、命名空间、Controller 类名和 Action 方法名是否符合约定。默认 `post/view` 对应 `PostController::actionView()`。模块中的控制器还要包含模块路由段。

### Action 没有执行

Inline Action 必须是 public，并以 `action` 开头；Standalone Action 必须在 `actions()` 中注册。方法名大小写也需要匹配。

### 删除动作被拒绝

检查 `VerbFilter` 配置和表单提交方式。若动作只允许 POST，通过浏览器地址栏发送 GET 会被拒绝，这是预期行为。

### `load()` 后模型没有变化

检查表单字段名是否带有 Model 表单名、请求方法是否正确，以及属性是否属于当前场景的安全属性。复杂输入通常交给 Form Model 处理。

### 返回 JSON 却得到 HTML

检查 Response 格式是否设为 `Response::FORMAT_JSON`，以及是否有过滤器或错误处理逻辑提前返回其他响应。

## 小结

Yii2 Controller 将请求路由到 Action，经过过滤器和生命周期钩子后执行动作，再把结果交给 Response。页面控制器通常继承 `yii\web\Controller`，REST API 则使用 `yii\rest\Controller` 或 `ActiveController`。

实际开发中，Action 适合做简短的 HTTP 协调：读取请求、调用 Model 或 Service、渲染 View 或返回 JSON。过滤器负责认证、方法限制等横切规则；输入验证和业务规则应放在专门的模型或服务中。

## 参考资料

- [Yii 2.0 Controllers 官方指南](https://www.yiiframework.com/doc/guide/2.0/en/structure-controllers)
- [Yii 2.0 Controller API](https://www.yiiframework.com/doc/api/2.0/yii-web-controller)
- [Yii 2.0 Request Handling Overview](https://www.yiiframework.com/doc/guide/2.0/en/runtime-overview)
- [Yii 2.0 AccessControl API](https://www.yiiframework.com/doc/api/2.0/yii-filters-accesscontrol)
- [Yii 2.0 REST Controllers](https://www.yiiframework.com/doc/guide/2.0/en/rest-controllers)
