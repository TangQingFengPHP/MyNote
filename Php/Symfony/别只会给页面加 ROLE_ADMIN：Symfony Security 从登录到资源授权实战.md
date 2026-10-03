# 别只会给页面加 ROLE_ADMIN：Symfony Security 从登录到资源授权实战

登录成功，只能说明账号凭证通过了验证。后台页面、订单、文章、文件能不能访问，还得由授权规则继续判断。Symfony Security 把这些事情放进一套相对完整的机制里：用户从哪里加载、密码如何校验、哪些请求需要登录、具体资源能否被当前用户操作，都可以集中配置和复用。

这篇文章以 Symfony 7.4 为例，从表单登录搭起一条完整链路，再加入角色限制和 Voter，让“只能编辑自己的文章”这类规则也能落到代码里。

## 先把几个词说清楚

- **认证（Authentication）**：确认当前请求对应哪个用户。比如校验邮箱和密码。
- **授权（Authorization）**：判断这个用户能不能执行某个操作。比如能否进入后台、能否编辑某篇文章。
- **User**：代表应用里的用户身份，通常是 Doctrine 实体。
- **User Provider**：按用户名、邮箱等标识从数据库或其他存储中加载 User。
- **Firewall**：匹配请求并决定使用哪些认证方式、是否建立会话。每个请求只会使用第一个匹配的防火墙。
- **Role**：粗粒度权限标签，例如 `ROLE_ADMIN`。
- **Voter**：针对某个具体对象和动作作授权判断，例如“当前用户能否编辑这篇文章”。

可以把请求过程看成下面这条链：

```text
HTTP 请求
   ↓
匹配 Firewall
   ↓
从 Session / Cookie / Token 识别用户
   ↓
User Provider 加载用户并校验身份
   ↓
角色规则、access_control 或 Voter 作授权判断
   ↓
放行，或返回登录跳转 / 403
```

认证回答“是谁”，授权回答“能做什么”。这两件事不要混成一个判断。

## 准备 Symfony 项目

已有 Symfony 项目时，安装 SecurityBundle：

```bash
composer require symfony/security-bundle
```

下文的完整 Demo 使用 Doctrine 保存用户和文章。若项目还没有 Doctrine、MakerBundle：

```bash
composer require symfony/orm-pack
composer require --dev symfony/maker-bundle
```

用户实体可以由 Maker 生成：

```bash
php bin/console make:user
```

向导中选择 `App\Entity\User`，用户标识使用 `email`，并启用密码字段。生成的实体应实现 `UserInterface` 和 `PasswordAuthenticatedUserInterface`，其中最关键的方法如下：

```php
<?php

namespace App\Entity;

use Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface;
use Symfony\Component\Security\Core\User\UserInterface;

class User implements UserInterface, PasswordAuthenticatedUserInterface
{
    // Doctrine 属性略：id、email、roles、password

    public function getUserIdentifier(): string
    {
        return (string) $this->email;
    }

    public function getRoles(): array
    {
        $roles = $this->roles;
        $roles[] = 'ROLE_USER';

        return array_unique($roles);
    }

    public function getPassword(): ?string
    {
        return $this->password;
    }

    public function eraseCredentials(): void
    {
        // 若实体临时保存了明文密码，在这里清理。
    }
}
```

实际项目中使用 Maker 生成的完整实体，并补齐 Doctrine 字段和 getter/setter。`getUserIdentifier()` 必须返回稳定且唯一的登录标识；`getRoles()` 建议默认补上 `ROLE_USER`。

## 配置密码哈希、用户来源和 Firewall

编辑 `config/packages/security.yaml`：

```yaml
security:
    password_hashers:
        App\Entity\User: 'auto'

    providers:
        app_user_provider:
            entity:
                class: App\Entity\User
                property: email

    firewalls:
        dev:
            pattern: ^/(_(profiler|wdt)|css|images|js)/
            security: false

        main:
            lazy: true
            provider: app_user_provider
            form_login:
                login_path: app_login
                check_path: app_login
                username_parameter: email
                password_parameter: password
                enable_csrf: true
            logout:
                path: app_logout
                enable_csrf: true
                csrf_token_id: logout
                target: app_login

    access_control:
        - { path: ^/login$, roles: PUBLIC_ACCESS }
        - { path: ^/admin, roles: ROLE_ADMIN }
        - { path: ^/account, roles: ROLE_USER }
```

配置里有几个容易踩坑的点：

- `password_hashers` 中的 `auto` 让 Symfony 选择合适的密码哈希器，并支持后续升级。
- `providers` 告诉 Symfony 按 `email` 字段从 `App\Entity\User` 加载用户。
- `main` 防火墙负责普通网站请求。`lazy: true` 表示请求确实需要安全上下文时再初始化相关状态。
- `form_login` 会接管发往 `check_path` 的登录 POST，不需要在控制器里手写密码比对。
- `access_control` 按顺序匹配，**只使用第一条匹配规则**。公开登录页要放在更宽泛的受保护路径之前。
- `PUBLIC_ACCESS` 表示匿名访问；已登录用户默认会有 `ROLE_USER`。

`security.yaml` 里的 `dev` 防火墙应该排在 `main` 前面，否则开发工具静态资源可能先被主防火墙接管。

## 登录页面：控制器展示表单，Firewall 校验凭证

使用 Maker 生成登录控制器和 Twig 模板：

```bash
php bin/console make:security:form-login
```

向导生成的控制器主要负责显示表单和回显错误。Symfony 会从登录 POST 中读取 `email`、`password`，根据 User Provider 加载用户，再用 Password Hasher 验证密码。

精简后的控制器大致如下：

```php
<?php

namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Component\Security\Http\Authentication\AuthenticationUtils;

class SecurityController extends AbstractController
{
    #[Route('/login', name: 'app_login', methods: ['GET', 'POST'])]
    public function login(AuthenticationUtils $authenticationUtils): Response
    {
        return $this->render('security/login.html.twig', [
            'last_username' => $authenticationUtils->getLastUsername(),
            'error' => $authenticationUtils->getLastAuthenticationError(),
        ]);
    }

    #[Route('/logout', name: 'app_logout', methods: ['POST'])]
    public function logout(): never
    {
        throw new \LogicException('该方法由 Symfony Security Firewall 接管。');
    }
}
```

登出路由通常不需要实际执行控制器方法：请求到达后会被 Firewall 拦截，清除认证状态并结束会话。

登录模板 `templates/security/login.html.twig`：

```twig
{% if error %}
    <div class="error">邮箱或密码不正确</div>
{% endif %}

<form method="post" action="{{ path('app_login') }}">
    <label for="email">邮箱</label>
    <input id="email" type="email" name="email" value="{{ last_username }}" required autofocus>

    <label for="password">密码</label>
    <input id="password" type="password" name="password" required>

    <input type="hidden" name="_csrf_token" value="{{ csrf_token('authenticate') }}">
    <button type="submit">登录</button>
</form>
```

CSRF Token 的 ID 默认为 `authenticate`，字段名默认为 `_csrf_token`。表单参数名必须和 `username_parameter`、`password_parameter` 配置一致。Symfony 官方文档也建议登录表单启用 CSRF 防护。

登录成功后，默认通过 Session 维持认证状态。之后同一浏览器发来的请求会带上 Session Cookie，Symfony 再恢复对应用户。

## 生成密码哈希，别把明文塞进数据库

注册用户时，原始密码只在请求处理期间短暂存在。写入数据库前调用 `UserPasswordHasherInterface`：

```php
use App\Entity\User;
use Symfony\Component\PasswordHasher\Hasher\UserPasswordHasherInterface;

public function register(UserPasswordHasherInterface $passwordHasher): Response
{
    $user = new User();
    $user->setEmail('reader@example.com');

    $hashedPassword = $passwordHasher->hashPassword($user, 'ChangeThisPassword!');
    $user->setPassword($hashedPassword);

    // $entityManager->persist($user);
    // $entityManager->flush();

    return new Response('用户创建完成');
}
```

密码哈希不是可逆加密。登录时 Symfony 会校验输入密码和哈希是否匹配；代码中不要直接比较明文，也不要自行设计哈希格式。`auto` 配合密码哈希器还能在用户成功登录后逐步升级旧哈希。

创建数据库表和迁移可使用：

```bash
php bin/console make:migration
php bin/console doctrine:migrations:migrate
```

## 第一层授权：路由和角色

最简单的限制可以放在 `access_control`：

```yaml
access_control:
    - { path: ^/login$, roles: PUBLIC_ACCESS }
    - { path: ^/admin, roles: ROLE_ADMIN }
    - { path: ^/account, roles: ROLE_USER }
```

也可以在控制器动作上声明角色：

```php
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Component\Security\Http\Attribute\IsGranted;

#[Route('/admin/users', name: 'admin_users')]
#[IsGranted('ROLE_ADMIN')]
public function users(): Response
{
    return $this->render('admin/users.html.twig');
}
```

Twig 模板也可以按权限显示界面元素：

```twig
{% if is_granted('ROLE_ADMIN') %}
    <a href="{{ path('admin_users') }}">用户管理</a>
{% endif %}
```

模板判断只是界面展示，不能代替后端授权。即使菜单隐藏，路由和业务操作仍需要服务端检查。

角色支持继承，例如管理员也拥有普通用户权限：

```yaml
security:
    role_hierarchy:
        ROLE_ADMIN: ROLE_USER
        ROLE_SUPER_ADMIN: [ROLE_ADMIN, ROLE_ALLOWED_TO_SWITCH]
```

角色适合“用户属于什么组”这类粗粒度判断。若判断涉及某条订单、文章的作者、所属租户或数据状态，应该使用 Voter。

## 第二层授权：用 Voter 判断资源权限

假设博客规则是：管理员可以编辑所有文章，普通用户只能编辑自己写的文章。先准备一个 `Post` 实体，至少包含 `author` 关联和 `title` 字段。再创建 `src/Security/PostVoter.php`：

```php
<?php

namespace App\Security;

use App\Entity\Post;
use App\Entity\User;
use Symfony\Component\Security\Core\Authentication\Token\TokenInterface;
use Symfony\Component\Security\Core\Authorization\Voter\Voter;

final class PostVoter extends Voter
{
    public const EDIT = 'POST_EDIT';
    public const VIEW = 'POST_VIEW';

    protected function supports(string $attribute, mixed $subject): bool
    {
        return $subject instanceof Post
            && in_array($attribute, [self::EDIT, self::VIEW], true);
    }

    protected function voteOnAttribute(
        string $attribute,
        mixed $subject,
        TokenInterface $token,
    ): bool {
        $user = $token->getUser();
        if (!$user instanceof User) {
            return false;
        }

        /** @var Post $post */
        $post = $subject;

        if (in_array('ROLE_ADMIN', $user->getRoles(), true)) {
            return true;
        }

        return match ($attribute) {
            self::EDIT => $post->getAuthor() === $user,
            self::VIEW => $post->isPublic() || $post->getAuthor() === $user,
            default => false,
        };
    }
}
```

Symfony 默认服务配置会自动发现 `src/` 下服务，并给 Voter 添加正确标签，无需再手工注册。

控制器中检查权限：

```php
use App\Entity\Post;
use App\Security\PostVoter;
use Symfony\Component\Routing\Attribute\Route;

#[Route('/posts/{id}/edit', name: 'post_edit', methods: ['GET', 'POST'])]
public function edit(Post $post): Response
{
    $this->denyAccessUnlessGranted(PostVoter::EDIT, $post);

    // 创建或处理文章编辑表单
    return $this->render('post/edit.html.twig', ['post' => $post]);
}
```

Symfony 会把 `POST_EDIT` 和 `$post` 传给支持这组参数的 Voter。用户不是作者且不是管理员时，授权失败并返回 403。也可以通过 `#[IsGranted('POST_EDIT', subject: 'post')]` 实现相同检查。

Voter 的两个方法分工明确：

- `supports()` 判断这个 Voter 是否关心当前权限名和对象类型。
- `voteOnAttribute()` 读取当前用户和资源，返回允许或拒绝。

这让权限规则集中在一处，列表、详情、编辑、API 都能复用同一套判断。

## 多个 Firewall：网站和 API 分开处理

一个应用可以同时有网页和 API。通常把更具体的 API 防火墙写在通用网站防火墙前面：

```yaml
security:
    firewalls:
        dev:
            pattern: ^/(_(profiler|wdt)|css|images|js)/
            security: false

        api:
            pattern: ^/api
            stateless: true
            provider: app_user_provider
            access_token:
                token_handler: App\Security\ApiTokenHandler

        main:
            lazy: true
            provider: app_user_provider
            form_login:
                login_path: app_login
                check_path: app_login
                username_parameter: email
                password_parameter: password
                enable_csrf: true
            logout:
                path: app_logout
```

`stateless: true` 表示 API 防火墙不使用 Session。Symfony 内置 `access_token` authenticator 从默认的 `Authorization` 请求头读取 Token，但仍需要提供 `token_handler` 把 Token 映射到用户。API Token 的签发、存储、吊销策略要按项目需要实现。

若使用 JWT，可采用 LexikJWTAuthenticationBundle 等生态方案；不要把“能解析 JWT”误当成“已经验证安全”。应验证签名、发行者、受众、有效期，并设计密钥轮换和令牌撤销策略。访问令牌通过 `Authorization: Bearer ...` 头发送，避免放进 URL，防止访问日志、浏览器历史和 Referer 泄露。

## CSRF、Session 和登出

浏览器会自动带上 Cookie，因此基于 Session Cookie 的写操作需要防 CSRF。表单登录开启 `enable_csrf: true` 后，需要在表单放入 `csrf_token('authenticate')`。修改、删除等表单也应使用 Symfony Form 的 CSRF 防护，或显式校验 Token。

登出同样可以校验 CSRF：

```yaml
logout:
    path: app_logout
    enable_csrf: true
    csrf_token_id: logout
```

对应模板中使用 POST 表单：

```twig
<form method="post" action="{{ path('app_logout') }}">
    <input type="hidden" name="_csrf_token" value="{{ csrf_token('logout') }}">
    <button type="submit">退出登录</button>
</form>
```

认证 Cookie 应只通过 HTTPS 传输，并设置合理的 `HttpOnly`、`Secure`、`SameSite` 属性。Symfony 的 Session Cookie 有相关配置项，应结合 HTTPS 终止位置、子域和第三方登录流程检查最终响应头。

## 常用检查命令

```bash
# 查看安全配置
php bin/console debug:config security

# 查看路由以及登录/登出路由是否存在
php bin/console debug:router

# 检查服务和依赖注入配置
php bin/console lint:container
```

## 常见问题

### 登录页自己也跳回登录页

检查 `access_control` 是否先写了 `^/` 或 `^/login` 的受保护规则。规则从上到下只取第一条匹配项，登录页需要 `PUBLIC_ACCESS`。

### 登录 POST 返回 404 或控制器收到了密码

确认登录表单的 `action` 对应 `check_path`，该路径属于处理表单登录的同一个 Firewall，且表单字段名与 `username_parameter`、`password_parameter` 一致。正常情况下，POST 凭证由 Firewall 接管，不需要在登录控制器里自行校验。

### 明明有角色却被拒绝

Symfony 角色名称通常以 `ROLE_` 开头。确认 User 实体的 `getRoles()` 返回了预期数组，并留意 `access_control` 的顺序和 Firewall 匹配顺序。

### 修改了用户角色但旧会话仍保留原权限

会话中的用户身份可能要到下次重新加载时才刷新。高安全要求场景需要规划会话失效、用户版本字段或重新认证策略；角色变更后，也可以要求用户重新登录。

## 小结

Symfony Security 的常见使用路径可以概括为：User 表示身份，Provider 负责加载用户，Firewall 接管请求认证，Password Hasher 校验密码，`access_control` 和角色保护路由，Voter 判断当前用户能否操作具体资源。

简单的后台入口用角色限制即可；一旦权限跟数据归属、租户、订单状态有关，就把判断放进 Voter，并在实际操作前检查。这样权限逻辑不会散落在模板和控制器各处，也不容易因为只隐藏了按钮就留下接口漏洞。

## 参考资料

- [Symfony 7.4 Security 主文档](https://symfony.com/doc/7.4/security.html)
- [Symfony 7.4 Security 配置参考](https://symfony.com/doc/7.4/reference/configuration/security.html)
- [Symfony 7.4 Voter 实战文档](https://symfony.com/doc/7.4/security/voters.html)
- [Symfony 7.4 Access Token 认证](https://symfony.com/doc/7.4/security/access_token.html)
- [Symfony 密码哈希和验证](https://symfony.com/doc/current/security/passwords.html)
