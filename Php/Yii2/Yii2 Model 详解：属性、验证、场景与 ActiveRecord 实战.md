# Yii2 Model 详解：属性、验证、场景与 ActiveRecord 实战

Yii2 里的 Model 不只是“数据库表对应的类”。它还能表示登录表单、注册数据、搜索条件和 API 请求参数，负责组织属性、验证规则、场景与错误信息。真正负责数据库读写的是 ActiveRecord；ActiveRecord 继承自 Model，所以两者有不少相同能力，也容易被混为一谈。

这篇文章从一个注册表单 Demo 入手，再逐步说明 Model 属性、批量赋值、验证规则、场景、错误信息和 ActiveRecord 的分工。

## Yii2 Model 能做什么？

Yii2 中最基础的模型类是 `yii\base\Model`。它提供的能力包括：

- 定义和读取属性；
- 声明验证规则并执行验证；
- 按场景启用不同规则和属性；
- 安全地把请求数据批量写入对象；
- 收集并返回字段错误；
- 为字段提供界面标签。

Model 本身不要求关联数据库。比如登录表单只需要接收账号和密码，不需要对应一张表：

```php
<?php

namespace app\models;

use yii\base\Model;

class LoginForm extends Model
{
    public $email;
    public $password;
}
```

`yii\db\ActiveRecord` 则在 Model 的属性和验证能力上，增加了表映射、查询、插入、更新和删除等数据库能力。

```text
yii\base\Model
    ├── 表单模型、搜索模型、请求 DTO
    └── yii\db\ActiveRecord
            └── 数据库记录映射
```

| 类型 | 是否默认关联数据库 | 常见用途 |
| --- | --- | --- |
| `yii\base\Model` | 否 | 登录、注册、搜索表单、API 参数、临时数据 |
| `yii\base\DynamicModel` | 否 | 字段和规则需要在运行时动态生成 |
| `yii\db\ActiveRecord` | 是 | 对应数据库表，读取和保存记录 |

## 属性是怎么来的？

Model 的属性通常声明为 public 成员变量：

```php
class ContactForm extends \yii\base\Model
{
    public $name;
    public $email;
    public $message;
}
```

Yii 会把这些成员当作模型属性，支持对象属性访问，也支持数组形式访问：

```php
$form->email = 'reader@example.com';
echo $form['email'];
```

也可以通过 `attributes()` 自定义属性列表，适用于属性并非简单 public 成员的模型。不过普通表单 Model 用 public 属性最直观。

## 注册 Demo：Model 接收并验证表单数据

这个 Demo 做一个常见注册流程：校验用户名、邮箱、密码和确认密码，然后把验证通过的数据交给 User ActiveRecord 保存。

### 1. 创建表单模型 `SignupForm`

先准备 User 表。以下迁移创建一个最小用户表：

```php
<?php

use yii\db\Migration;

class m260109_000001_create_user_table extends Migration
{
    public function safeUp(): void
    {
        $this->createTable('{{%user}}', [
            'id' => $this->primaryKey(),
            'username' => $this->string(30)->notNull(),
            'email' => $this->string(255)->notNull(),
            'password_hash' => $this->string(255)->notNull(),
        ]);
        $this->createIndex('ux_user_email', '{{%user}}', 'email', true);
    }

    public function safeDown(): void
    {
        $this->dropTable('{{%user}}');
    }
}
```

执行迁移后，创建对应的 `models/User.php` ActiveRecord：

```php
<?php

namespace app\models;

use yii\db\ActiveRecord;

class User extends ActiveRecord
{
    public static function tableName(): string
    {
        return '{{%user}}';
    }

    public function rules(): array
    {
        return [
            [['username', 'email', 'password_hash'], 'required'],
            [['username'], 'string', 'max' => 30],
            [['email'], 'string', 'max' => 255],
            [['email'], 'email'],
            [['email'], 'unique'],
            [['password_hash'], 'string', 'max' => 255],
        ];
    }
}
```

注册输入模型通过 `User::save()` 保存时，这些 ActiveRecord 规则也会执行。

创建 `models/SignupForm.php`：

```php
<?php

namespace app\models;

use Yii;
use yii\base\Model;

class SignupForm extends Model
{
    public $username;
    public $email;
    public $password;
    public $passwordRepeat;

    public function rules(): array
    {
        return [
            [['username', 'email', 'password', 'passwordRepeat'], 'required'],
            [['username'], 'trim'],
            [['username'], 'string', 'min' => 3, 'max' => 30],
            [['email'], 'trim'],
            [['email'], 'email'],
            [['password'], 'string', 'min' => 8],
            [['passwordRepeat'], 'compare', 'compareAttribute' => 'password'],
            [['email'], 'validateEmailUnique'],
        ];
    }

    public function validateEmailUnique(string $attribute): void
    {
        if (!$this->hasErrors($attribute)
            && User::find()->where(['email' => $this->$attribute])->exists()) {
            $this->addError($attribute, '这个邮箱已经注册');
        }
    }

    public function attributeLabels(): array
    {
        return [
            'username' => '用户名',
            'email' => '邮箱',
            'password' => '密码',
            'passwordRepeat' => '确认密码',
        ];
    }

    public function signup(): ?User
    {
        if (!$this->validate()) {
            return null;
        }

        $user = new User();
        $user->username = $this->username;
        $user->email = $this->email;
        $user->password_hash = Yii::$app->security->generatePasswordHash($this->password);

        return $user->save() ? $user : null;
    }
}
```

这里的 `SignupForm` 不是数据库实体，只负责注册输入相关的规则和流程：

- `rules()` 声明必填、长度、邮箱格式、密码确认和邮箱唯一性规则；
- `validateEmailUnique()` 是内联验证器，用数据库检查邮箱是否已占用；
- `attributeLabels()` 提供表单界面上显示的中文字段名；
- `signup()` 校验通过后才创建 User，并用 Yii 的安全组件生成密码哈希。

`User` 类要继承 ActiveRecord，并映射到包含 `username`、`email`、`password_hash` 等字段的用户表。`User::save()` 会再执行 User 自身的规则，成功时写入数据库。

> 邮箱唯一性还应由数据库唯一索引兜底。应用层先查重可以返回友好提示，但并发请求仍可能同时通过查询，数据库约束才是最终保障。

### 2. 控制器加载、验证并处理结果

```php
<?php

namespace app\controllers;

use app\models\SignupForm;
use Yii;
use yii\web\Controller;
use yii\web\Response;

class AccountController extends Controller
{
    public function actionSignup(): Response|string
    {
        $model = new SignupForm();

        if ($model->load(Yii::$app->request->post())) {
            $user = $model->signup();

            if ($user !== null) {
                Yii::$app->session->setFlash('success', '注册成功');
                return $this->redirect(['site/index']);
            }
        }

        return $this->render('signup', [
            'model' => $model,
        ]);
    }
}
```

`load()` 负责按表单名从请求数组里取值并批量赋值；它不代表校验成功。`signup()` 内部调用 `validate()`，校验失败时错误会留在 Model 中，控制器把模型交回视图显示即可。

### 3. 用 ActiveForm 显示表单和错误

创建 `views/account/signup.php`：

```php
<?php

use yii\widgets\ActiveForm;
use yii\helpers\Html;

$form = ActiveForm::begin();
?>

<?= $form->field($model, 'username') ?>
<?= $form->field($model, 'email') ?>
<?= $form->field($model, 'password')->passwordInput() ?>
<?= $form->field($model, 'passwordRepeat')->passwordInput() ?>

<div class="form-group">
    <?= Html::submitButton('注册', ['class' => 'btn btn-primary']) ?>
</div>

<?php ActiveForm::end(); ?>
```

项目若使用 Yii2 Bootstrap 扩展，也可以改用扩展提供的 ActiveForm 控件。页面字段名默认按 `SignupForm[email]` 这样的结构生成，正好对应 `load()` 默认读取的模型数据。

## `load()`：批量赋值的入口

假设请求数据如下：

```php
$_POST = [
    'SignupForm' => [
        'username' => 'alex',
        'email' => 'alex@example.com',
        'password' => '********',
    ],
];
```

执行：

```php
$model->load(Yii::$app->request->post());
```

Yii 默认通过 `formName()` 返回的名称（通常是类名）找到 `SignupForm` 子数组，再把其中安全的属性写入模型。可显式指定表单名：

```php
$model->load($data, 'SignupForm');
```

若传入的数据本身就是字段数组，没有模型名这一层，可传空字符串：

```php
$model->load($data, '');
```

`load()` 返回 `true` 只代表找到对应表单数据并尝试赋值，不代表数据通过验证。后续仍要调用 `validate()`。

## 安全属性：防止用户改写不该改的字段

批量赋值不会无条件修改模型全部属性。Yii 根据当前场景的 active attributes 确定哪些属性可以安全地批量赋值。

默认场景下，出现在验证规则里的属性通常会成为安全属性。因此用户表单中不要给 `role`、`is_admin` 等敏感字段添加 `safe` 规则，否则恶意请求可能把这些字段一并提交。

```php
// 不应把权限字段标记为普通表单安全字段
[['username', 'email'], 'safe'],
```

`safe` 验证器的用途是允许批量赋值，但不做实际校验。字段仍需有对应的类型、格式和业务规则。安全属性控制批量赋值边界，不等同于“输入可信”。

需要验证某属性但禁止批量赋值时，可在场景属性名前加 `!`，例如 `!role`。此类字段只能由服务端代码显式设置。

## 验证规则 `rules()` 怎么执行？

每条规则通常由属性列表、验证器名称和选项组成：

```php
public function rules(): array
{
    return [
        [['title', 'content'], 'required'],
        [['title'], 'string', 'max' => 120],
        [['status'], 'in', 'range' => [0, 1, 2]],
    ];
}
```

调用 `$model->validate()` 时，Yii 会根据当前场景选择相关规则，执行校验，并把错误放入模型：

```php
if (!$model->validate()) {
    $errors = $model->getErrors();
    $titleError = $model->getFirstError('title');
}
```

### 常见验证器

| 规则 | 用途 | 示例 |
| --- | --- | --- |
| `required` | 必填 | `[['name'], 'required']` |
| `string` | 字符串和长度 | `[['name'], 'string', 'max' => 50]` |
| `integer` | 整数 | `[['age'], 'integer', 'min' => 0]` |
| `number` | 数值 | `[['price'], 'number', 'min' => 0]` |
| `email` | 邮箱格式 | `[['email'], 'email']` |
| `boolean` | 布尔值 | `[['enabled'], 'boolean']` |
| `in` | 枚举范围 | `[['status'], 'in', 'range' => [0, 1]]` |
| `compare` | 两字段比较 | `[['passwordRepeat'], 'compare', 'compareAttribute' => 'password']` |
| `unique` | 数据库唯一值 | `[['email'], 'unique']`（通常用于 ActiveRecord） |
| `safe` | 允许批量赋值但不验证 | `[['description'], 'safe']` |

不少验证器可以通过 `skipOnEmpty`、`skipOnError` 和 `when` 调整执行条件。确认验证规则时，应留意验证顺序和场景限制；比如格式检查失败后，数据库查询类验证通常可以跳过。

### 自定义验证器

简单规则可写成 Model 的内联方法：

```php
public function rules(): array
{
    return [
        [['endDate'], 'validateDateRange'],
    ];
}

public function validateDateRange(string $attribute, array $params): void
{
    if ($this->startDate && $this->endDate
        && strtotime($this->endDate) < strtotime($this->startDate)) {
        $this->addError($attribute, '结束时间不能早于开始时间');
    }
}
```

跨多个模型复用的校验逻辑更适合做成独立 Validator 类，避免 Model 变成大量校验代码的集合。

## 场景：同一个 Model 服务于不同流程

注册和资料修改都涉及用户信息，但必填字段不同：注册时需要密码；修改资料时不应要求重新填写密码。场景可以让同一个 Model 按流程切换属性和规则。

```php
class UserForm extends \yii\base\Model
{
    public const SCENARIO_REGISTER = 'register';
    public const SCENARIO_PROFILE = 'profile';

    public $username;
    public $email;
    public $password;

    public function rules(): array
    {
        return [
            [['username', 'email', 'password'], 'required', 'on' => self::SCENARIO_REGISTER],
            [['username', 'email'], 'required', 'on' => self::SCENARIO_PROFILE],
            [['username'], 'string', 'min' => 3, 'max' => 30],
            [['email'], 'email'],
            [['password'], 'string', 'min' => 8, 'on' => self::SCENARIO_REGISTER],
        ];
    }
}
```

创建模型时指定场景：

```php
$registerForm = new UserForm(['scenario' => UserForm::SCENARIO_REGISTER]);
$profileForm = new UserForm(['scenario' => UserForm::SCENARIO_PROFILE]);
```

场景会影响两件事：哪些规则参与验证，以及哪些属性允许批量赋值。默认场景由 `rules()` 中出现的属性推导；需要完全控制时可重写 `scenarios()`：

```php
public function scenarios(): array
{
    return [
        self::SCENARIO_REGISTER => ['username', 'email', 'password'],
        self::SCENARIO_PROFILE => ['username', 'email'],
    ];
}
```

重写时若还需要保留父类规则推导出的场景，可先调用 `parent::scenarios()` 再调整结果。场景是输入边界控制的一部分，不能只把它当作“切换必填项”的开关。

## Model 和 ActiveRecord 怎么配合？

注册 Demo 中的 `SignupForm` 负责收集和验证外部输入；`User` ActiveRecord 负责把用户数据映射到数据库表。两者分工清晰：

```text
请求数据
   ↓ load()
SignupForm（输入规则、场景、错误）
   ↓ validate()
User ActiveRecord（数据库映射、保存）
   ↓ save()
数据库
```

ActiveRecord 也有 `rules()`、`validate()` 和 `errors`，因为它继承自 Model。保存时默认会先验证：

```php
$user->save();       // 验证通过后写入数据库
$user->save(false);  // 跳过验证，需确保调用处已经完整校验
```

`save(false)` 不是性能优化的通用写法。它会跳过 ActiveRecord 的验证，错误使用容易让脏数据进入数据库。表单模型验证和 ActiveRecord 规则可以关注不同边界：前者校验用户输入流程，后者保护持久化数据约束。

## 错误信息如何查看和显示？

Model 常用错误接口：

```php
$model->hasErrors();
$model->hasErrors('email');
$model->getErrors();
$model->getFirstErrors();
$model->getFirstError('email');
$model->addError('email', '邮箱已被占用');
```

ActiveForm 会读取 Model 的错误并显示在对应字段旁。API 场景可以把错误转换为 JSON：

```php
if (!$model->validate()) {
    Yii::$app->response->statusCode = 422;
    return $model->getErrors();
}
```

如果表单只提示错误而没有展示字段名，可以通过 `attributeLabels()` 给属性设置更适合用户阅读的标签。

## `DynamicModel`：规则需要运行时决定时

字段在代码里固定时，优先定义普通 Model 类；字段和规则来自运行时配置时，可以用 `DynamicModel`：

```php
use yii\base\DynamicModel;

$model = new DynamicModel(['width' => 12, 'height' => 8]);
$model->addRule(['width', 'height'], 'integer', ['min' => 1]);
$model->addRule(['width', 'height'], 'required');

if ($model->validate()) {
    // 参数通过校验
}
```

`DynamicModel` 适合临时数据验证或配置驱动的表单。若规则长期固定且业务语义明确，独立 Model 类更方便维护、测试和复用。

## 常见问题

### `load()` 返回 true，为什么字段还是空？

检查请求参数是否带有模型名，例如 `SignupForm[email]`；检查 `formName()` 是否被重写；再检查该属性是否属于当前场景的安全属性。无模型名的数据应使用 `$model->load($data, '')`。

### `validate()` 为什么没有检查某个字段？

确认字段是否出现在当前场景有效的规则中，规则是否使用了 `on` 限定场景，以及 `validate()` 是否传入了指定属性列表。

### `save()` 为什么没有保存？

ActiveRecord 默认会先执行验证。检查 `$model->getErrors()`，也要确认数据库连接、表名和字段映射。调试时不要一上来改成 `save(false)`，先找出验证失败原因。

### 用户提交了 `is_admin`，为什么没有改权限？

这是预期的安全行为：只有当前场景允许批量赋值的属性才会被 `load()` 设置。权限字段应由服务端授权逻辑显式修改，不能放进普通用户表单的安全属性集合。

## 小结

Yii2 的 Model 把属性、验证、场景、安全批量赋值和错误管理放在一个对象里。它适合表示输入数据和业务数据；ActiveRecord 在这些能力之上增加数据库读写。

常见流程是：`load()` 接收请求数据，`validate()` 检查规则，读取错误或把数据交给 ActiveRecord 保存。场景决定当前流程使用哪些规则和安全属性，规则配置也会影响批量赋值边界。把表单 Model 和数据库 ActiveRecord 按职责拆开，注册、搜索、登录和 API 参数会更容易维护。

## 参考资料

- [Yii 2.0 Model 官方指南](https://www.yiiframework.com/doc/guide/2.0/en/structure-models)
- [Yii 2.0 Model API](https://www.yiiframework.com/doc/api/2.0/yii-base-model)
- [Yii 2.0 输入验证指南](https://www.yiiframework.com/doc/guide/2.0/en/input-validation)
- [Yii 2.0 表单入门](https://www.yiiframework.com/doc/guide/2.0/en/start-forms)
