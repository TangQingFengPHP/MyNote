# 别把 JWT 当成加密令牌：Symfony LexikJWTAuthenticationBundle 实战详解

前后端分离项目里，常见做法是先用账号密码换取一个 Token，再把 Token 放进后续 API 请求。Symfony 本身负责认证和授权框架，LexikJWTAuthenticationBundle 则把 JWT 的生成、签名和校验接入 Symfony Security。

这套方式很适合 API，但 JWT 不是“加密后的用户资料”，也不会自动解决刷新、注销和权限撤销。下面以 Symfony 7.4 和 LexikJWTAuthenticationBundle 3.x 为例，从安装配置到登录、访问受保护接口，一步步搭出可运行的基本流程。

## 先认识 JWT 和 Lexik Bundle

JWT（JSON Web Token）是一种带签名的令牌格式，常见结构由三段组成：

```text
Header.Payload.Signature
```

- `Header`：令牌类型和签名算法等信息。
- `Payload`：用户标识、角色、过期时间等声明（Claim）。
- `Signature`：根据内容和密钥生成的签名，用于验证内容没有被篡改。

常见的签名 JWT **不会隐藏 Payload 内容**。拿到令牌的人通常可以解码并读取 Payload，但不能在不知道密钥的情况下伪造有效签名。因此不要把密码、身份证号、银行卡号等敏感信息放进 Claim。

LexikJWTAuthenticationBundle 负责把 JWT 和 Symfony Security 串起来：

```text
账号密码登录
    ↓
Symfony 加载用户并校验密码
    ↓
Lexik 签发 JWT
    ↓
客户端保存 Token
    ↓
后续请求携带 Authorization: Bearer <token>
    ↓
Lexik 校验签名和有效期，Symfony 再执行授权
```

需要区分三件事：

- **认证**：Token 是否有效、对应哪个用户。
- **授权**：这个用户能不能访问当前接口或数据。
- **刷新**：旧 Token 快过期时，如何换一个新的 Token。

Lexik 主要解决 JWT 的签发和认证。接口权限仍由 Symfony Security 的角色、`access_control`、`#[IsGranted]` 或 Voter 负责；刷新 Token 需要额外机制。

## 版本和准备条件

LexikJWTAuthenticationBundle 3.x 支持 PHP 8.2 及以上，并兼容 Symfony 6.4、7.x、8.x。下面示例按 Symfony 7.4 写。项目还需要启用 OpenSSL 扩展，并应通过 HTTPS 提供服务。

安装 SecurityBundle、Doctrine 和 Lexik Bundle：

```bash
composer require symfony/security-bundle
composer require symfony/orm-pack
composer require lexik/jwt-authentication-bundle
composer require --dev symfony/maker-bundle
```

Symfony Flex 通常会自动注册 Bundle。可在 `config/bundles.php` 中确认：

```php
Lexik\Bundle\JWTAuthenticationBundle\LexikJWTAuthenticationBundle::class => ['all' => true],
```

## 生成签名密钥

Lexik 默认使用 RSA 密钥对：私钥签发令牌，公钥校验令牌。生成密钥：

```bash
php bin/console lexik:jwt:generate-keypair
```

默认文件位置：

```text
config/jwt/private.pem
config/jwt/public.pem
```

私钥应只部署在需要签发 Token 的服务上，不能提交到公开仓库，也不能放进前端代码。公钥可以分发给只需要校验 Token 的 API 服务。生产环境可通过 Secret 管理工具注入密钥文件和口令。

`.env.local` 中配置：

```dotenv
JWT_SECRET_KEY=%kernel.project_dir%/config/jwt/private.pem
JWT_PUBLIC_KEY=%kernel.project_dir%/config/jwt/public.pem
JWT_PASSPHRASE=生成密钥时使用的口令
```

`config/packages/lexik_jwt_authentication.yaml`：

```yaml
lexik_jwt_authentication:
    secret_key: '%env(resolve:JWT_SECRET_KEY)%'
    public_key: '%env(resolve:JWT_PUBLIC_KEY)%'
    pass_phrase: '%env(JWT_PASSPHRASE)%'
    token_ttl: 3600
```

`token_ttl` 单位是秒，这里表示一小时。生产环境不要把口令硬编码在配置文件或提交进版本库。若私钥没有设置口令，应按所用密钥生成方式和 Bundle 配置妥善处理空口令。

生成密钥时如果文件已存在，命令会避免意外覆盖。只有明确需要替换密钥时才使用 `--overwrite`；更换签名密钥会让旧 Token 无法再通过校验。

## 创建用户实体

登录时仍然需要 Symfony 的 User 和 User Provider。Lexik 不负责创建用户，也不替代密码哈希。

生成实体：

```bash
php bin/console make:user
```

可选择 `App\Entity\User`，使用 `email` 作为登录标识，并启用密码字段。生成的实体需要实现 `UserInterface` 和 `PasswordAuthenticatedUserInterface`，核心方法类似：

```php
<?php

namespace App\Entity;

use Symfony\Component\Security\Core\User\PasswordAuthenticatedUserInterface;
use Symfony\Component\Security\Core\User\UserInterface;

class User implements UserInterface, PasswordAuthenticatedUserInterface
{
    // Doctrine 字段略：id、email、roles、password

    public function getUserIdentifier(): string
    {
        return $this->email;
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
        // 清除实体中临时保存的明文凭证（若有）。
    }
}
```

`getUserIdentifier()` 返回唯一登录标识，`getRoles()` 返回用户角色，`getPassword()` 返回数据库里保存的密码哈希。明文密码不应写进数据库。

给 User 配好哈希算法，并告诉 Symfony 从数据库按邮箱查用户。

## 配置 Symfony Security

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

        login:
            pattern: ^/api/login
            stateless: true
            provider: app_user_provider
            json_login:
                check_path: api_login_check
                username_path: email
                password_path: password
                success_handler: lexik_jwt_authentication.handler.authentication_success
                failure_handler: lexik_jwt_authentication.handler.authentication_failure

        api:
            pattern: ^/api
            stateless: true
            provider: app_user_provider
            jwt: ~

    access_control:
        - { path: ^/api/login_check$, roles: PUBLIC_ACCESS }
        - { path: ^/api, roles: IS_AUTHENTICATED_FULLY }
```

几个容易忽略的地方：

- 防火墙按配置顺序匹配，一个请求只进入第一个匹配的 Firewall。
- `login` 必须排在 `api` 前面，否则登录请求会先被 JWT 防火墙处理。
- 两个防火墙都设为 `stateless: true`，代表不使用 Session。
- `json_login` 校验 JSON 里的邮箱和密码，成功处理器返回 Lexik 生成的 Token。
- `jwt: ~` 开启 Lexik 的 JWT 认证器，默认从 `Authorization: Bearer ...` 读取 Token。
- `access_control` 同样按顺序匹配，登录检查端点先设为 `PUBLIC_ACCESS`，其余 `/api` 路径要求完成认证。

旧版本配置中常出现 `enable_authenticator_manager: true`。Symfony 7.4 不需要这项旧配置，直接使用当前 Security 配置即可。

## 声明登录路由

在 `config/routes.yaml` 声明 JSON 登录检查路径：

```yaml
api_login_check:
    path: /api/login_check
    methods: [POST]
```

这个路由通常不需要控制器动作。请求会被 `login` Firewall 拦截，交由 `json_login` 完成验证；成功后调用 Lexik 的 `authentication_success` handler 返回 Token。

## 运行登录 Demo

先确保数据库迁移已执行，且数据库中存在一个可登录用户。用户创建流程应使用 `UserPasswordHasherInterface` 生成密码哈希：

```php
use App\Entity\User;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Component\PasswordHasher\Hasher\UserPasswordHasherInterface;

public function createUser(
    EntityManagerInterface $entityManager,
    UserPasswordHasherInterface $passwordHasher,
): void {
    $user = new User();
    $user->setEmail('reader@example.com');
    $user->setPassword($passwordHasher->hashPassword($user, 'ChangeThisPassword!'));

    $entityManager->persist($user);
    $entityManager->flush();
}
```

运行本地服务：

```bash
symfony server:start
```

Symfony CLI 会显示本地访问地址；下面的请求以 `https://localhost:8000` 为例。若本地证书尚未被系统信任，`curl -k` 可用于开发环境临时跳过证书校验，生产环境不得关闭证书校验。

向登录端点提交 JSON：

```bash
curl -k -i -X POST https://localhost:8000/api/login_check \
  -H 'Content-Type: application/json' \
  -d '{"email":"reader@example.com","password":"ChangeThisPassword!"}'
```

认证成功时，响应大致如下：

```json
{
  "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9..."
}
```

生产环境必须使用 HTTPS，Token 在网络中传输时必须有 TLS 保护。

## 访问受保护接口

创建一个只返回当前登录用户信息的 API 控制器 `src/Controller/ApiMeController.php`：

```php
<?php

namespace App\Controller;

use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\Routing\Attribute\Route;

final class ApiMeController extends AbstractController
{
    #[Route('/api/me', name: 'api_me', methods: ['GET'])]
    public function me(): JsonResponse
    {
        $user = $this->getUser();

        return $this->json([
            'email' => $user?->getUserIdentifier(),
            'roles' => $user?->getRoles(),
        ]);
    }
}
```

将登录响应中的 Token 带到请求头：

```bash
curl -k -i https://localhost:8000/api/me \
  -H 'Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiJ9...'
```

Token 缺失、格式不对、签名不匹配或已经过期时，受保护接口会拒绝请求；没有认证身份通常返回 `401 Unauthorized`。身份有效但没有所需权限时，Symfony 授权层通常返回 `403 Forbidden`。

## 角色授权和接口权限

认证只说明 Token 属于哪个用户，接口能否访问还要配置授权。例如管理员端点要求 `ROLE_ADMIN`：

```php
use Symfony\Component\Routing\Attribute\Route;
use Symfony\Component\Security\Http\Attribute\IsGranted;

#[Route('/api/admin/report', methods: ['GET'])]
#[IsGranted('ROLE_ADMIN')]
public function report(): JsonResponse
{
    return $this->json(['report' => 'admin only']);
}
```

也可以把规则写在 `access_control`：

```yaml
access_control:
    - { path: ^/api/login_check$, roles: PUBLIC_ACCESS }
    - { path: ^/api/admin, roles: ROLE_ADMIN }
    - { path: ^/api, roles: IS_AUTHENTICATED_FULLY }
```

注意管理员规则必须放在一般 `/api` 规则之前，否则 `/api` 会先匹配，后面的规则不会执行。

如果授权条件涉及订单归属、租户 ID 或资源状态，角色通常不够用，可使用 Symfony Voter 按“当前用户 + 具体对象 + 操作名”决定是否放行。JWT 负责身份认证，Voter 负责业务资源授权，两者各管一段。

## Token 里放什么，怎样加自定义 Claim

Payload 通常包含用户标识和标准时间声明，例如 `iat`、`exp`。可以通过 Lexik 事件在创建 Token 时添加业务 Claim：

```php
<?php

namespace App\EventListener;

use Lexik\Bundle\JWTAuthenticationBundle\Event\JWTCreatedEvent;
use Lexik\Bundle\JWTAuthenticationBundle\Events;
use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

#[AsEventListener(event: Events::JWT_CREATED, method: 'onJWTCreated')]
final class JWTCreatedListener
{
    public function onJWTCreated(JWTCreatedEvent $event): void
    {
        $payload = $event->getData();
        $payload['user_id'] = $event->getUser()->getId();
        $event->setData($payload);
    }
}
```

Claim 会被令牌持有者读取，因此只放非敏感、确有必要且体积较小的数据。权限变更后，旧 Token 里的角色声明不会自动改变，直到令牌过期或被撤销。若接口每次都需要最新权限，可在认证过程中从数据库重新加载用户并以服务端数据为准。

## Token 过期、刷新和注销

### 过期

`token_ttl` 控制签发 Token 的默认有效期，单位为秒。过期后 JWT 防火墙会拒绝请求，客户端需要重新认证或使用刷新机制。

### 刷新

Access Token 通常设置得较短。刷新 Token 应单独设计生命周期、存储和撤销规则，常见做法是使用 GesdinetJWTRefreshTokenBundle。刷新 Token 往往是有状态凭证，需要存储在服务端并能轮换或吊销；它和 Lexik 的 JWT Access Token 不是同一个东西。

### 注销和撤销

纯无状态 JWT 发出后，服务端默认不会记住每个 Token。删除浏览器本地 Token 只能让当前客户端不再发送它，不能让已泄露的 Token 立刻失效。

需要在登出时让 Token 立即失效，可启用 Lexik 的 Token Blocklist：

```yaml
lexik_jwt_authentication:
    # 其他密钥和 TTL 配置略
    blocklist_token:
        enabled: true
        cache: cache.app
```

Bundle 会为 Token 添加 `jti`，并把登出的令牌 ID 放入缓存，缓存项保留到该 Token 过期。这样每次校验都需要查询撤销状态，会增加缓存访问开销；是否启用应结合登出语义和流量规模决定。

## 常见误区和排查

### 把 JWT 当成加密数据

签名保证内容未经篡改，不代表内容保密。不要放密码、密钥、个人敏感信息。

### 把私钥提交到 Git

私钥泄漏后，攻击者可以签发看起来可信的 Token。将私钥放在受控 Secret 存储中，限制读取权限，并规划轮换。

### 把 Token 放在 URL

URL 容易出现在访问日志、浏览器历史和 Referer 中。默认使用 `Authorization` 请求头，不启用 Query 参数提取。

### 误以为无状态等于不用做安全设计

无状态只是服务器不靠 Session 保存认证状态。HTTPS、密钥保护、Token 生命周期、CORS、权限检查和令牌泄漏处理仍然需要设计。

### Firewall 顺序写反

`login` 必须在 `api` 前，具体 API 防火墙必须在通用 `main` 防火墙前。否则登录请求可能被错误的防火墙接管。

### 登录成功却拿不到 Token

确认 `success_handler` 指向 Lexik 的 `authentication_success`，登录路由属于 `login` Firewall，登录字段与 `username_path`、`password_path` 一致，且 User Provider 能查到用户。

### Apache 或反向代理丢了 Authorization Header

若应用收不到 `Authorization`，先检查 Web Server 和代理是否透传该请求头。Apache 配置有时需要把 Authorization 映射到 `HTTP_AUTHORIZATION`。也应检查代理、网关和应用日志，避免把完整 Token 写入日志。

常用排查命令：

```bash
php bin/console debug:config security
php bin/console debug:config lexik_jwt_authentication
php bin/console debug:router
```

## 小结

LexikJWTAuthenticationBundle 把 JWT 签发和验证接进 Symfony Security：JSON 登录防火墙校验账号密码并签发 Token，API 防火墙从 Bearer Header 提取 Token 并验证。Symfony 再用角色、访问规则和 Voter 判断接口或业务资源是否可访问。

记住几个重点：JWT 是签名令牌，不是加密保险箱；签名私钥必须保密；Token 要通过 HTTPS 传输；刷新与撤销需要额外设计；角色和数据权限仍由 Symfony Security 完成。

## 参考资料

- [LexikJWTAuthenticationBundle 3.x 官方文档](https://github.com/lexik/LexikJWTAuthenticationBundle/tree/3.x/Resources/doc)
- [LexikJWTAuthenticationBundle Getting Started](https://github.com/lexik/LexikJWTAuthenticationBundle/blob/3.x/Resources/doc/index.rst)
- [LexikJWTAuthenticationBundle 配置参考](https://github.com/lexik/LexikJWTAuthenticationBundle/blob/3.x/Resources/doc/1-configuration-reference.rst)
- [Symfony 7.4 Security](https://symfony.com/doc/7.4/security.html)
- [Symfony 7.4 Access Token Authentication](https://symfony.com/doc/7.4/security/access_token.html)
