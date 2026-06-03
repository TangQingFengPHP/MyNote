### 简介

`Doctrine ORM` 是 PHP 生态里非常常见的对象关系映射框架，Symfony 项目中用得尤其多。

它的作用可以用一句话概括：

```text
把数据库表映射成 PHP 对象，通过操作对象完成增删改查，最后由 Doctrine 生成并执行 SQL。
```

传统写法通常是直接写 SQL：

```php
$stmt = $pdo->prepare('INSERT INTO product (name, price) VALUES (?, ?)');
$stmt->execute(['机械键盘', 39900]);
```

使用 Doctrine 后，代码会变成操作实体对象：

```php
$product = new Product();
$product->setName('机械键盘');
$product->setPrice(39900);

$em->persist($product);
$em->flush();
```

表面上只是写法变了，底层做的事情更多：

* `Product` 类映射到 `product` 表
* `$product->setName()` 修改对象属性
* `persist()` 把新对象交给 Doctrine 管理
* `flush()` 统一生成并执行 SQL
* 修改已有对象时，Doctrine 会做变更追踪

Doctrine 适合实体结构比较清晰、业务围绕对象展开、需要迁移、关联查询、事务管理的 PHP 项目。

### Doctrine、DBAL、ORM 的关系

Doctrine 不是只有 ORM。

常见几个概念如下：

| 名称 | 作用 |
| --- | --- |
| `Doctrine DBAL` | 数据库抽象层，封装连接、SQL 执行、参数绑定、不同数据库差异 |
| `Doctrine ORM` | 对象关系映射，把 PHP 类和数据库表关联起来 |
| `Doctrine Migrations` | 数据库迁移工具，根据实体变化生成表结构变更 |
| `Doctrine Bundle` | Symfony 集成包，把 Doctrine 接入 Symfony 配置、容器和命令行 |

在 Symfony 项目中，常见调用链大致是：

```text
Controller
  |
  v
Service
  |
  v
Repository
  |
  v
EntityManager
  |
  v
Doctrine ORM
  |
  v
Doctrine DBAL
  |
  v
MySQL / PostgreSQL / SQLite
```

其中最核心的对象是 `EntityManager`。

它负责管理实体对象的生命周期，包括新增、修改、删除、查询、事务提交等。

### 安装 Doctrine

Symfony 项目中一般直接安装 `orm-pack`：

```shell
composer require symfony/orm-pack
```

开发环境常用代码生成器：

```shell
composer require --dev symfony/maker-bundle
```

如果需要查看 SQL、请求耗时、数据库查询次数，可以安装 Profiler：

```shell
composer require --dev symfony/profiler-pack
```

安装完成后，可以查看 Doctrine 相关包：

```shell
composer show doctrine/*
```

### 配置数据库连接

`.env` 中配置 `DATABASE_URL`。

MySQL 示例：

```env
DATABASE_URL="mysql://root:123456@127.0.0.1:3306/doctrine_demo?serverVersion=8.0&charset=utf8mb4"
```

PostgreSQL 示例：

```env
DATABASE_URL="postgresql://app:123456@127.0.0.1:5432/doctrine_demo?serverVersion=16&charset=utf8"
```

常见格式：

```text
数据库类型://用户名:密码@主机:端口/数据库名?参数
```

创建数据库：

```shell
php bin/console doctrine:database:create
```

如果数据库已经存在，这一步可以跳过。

### 准备一个商品订单 Demo

下面用一个小型商品订单场景演示 Doctrine。

实体关系如下：

```text
Category 1 ---- N Product
Product  1 ---- N OrderItem
Order    1 ---- N OrderItem
```

也就是：

* 一个分类下有多个商品
* 一个订单有多条订单明细
* 一条订单明细对应一个商品

这比单表 CRUD 更接近真实项目，也能把关联关系、查询、事务串起来。

### 创建 Category 实体

可以使用命令生成：

```shell
php bin/console make:entity Category
```

字段示例：

```text
name string 100 not null
```

实体代码可以写成下面这样：

```php
<?php

namespace App\Entity;

use App\Repository\CategoryRepository;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity(repositoryClass: CategoryRepository::class)]
class Category
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 100)]
    private ?string $name = null;

    /**
     * @var Collection<int, Product>
     */
    #[ORM\OneToMany(mappedBy: 'category', targetEntity: Product::class)]
    private Collection $products;

    public function __construct()
    {
        $this->products = new ArrayCollection();
    }

    public function getId(): ?int
    {
        return $this->id;
    }

    public function getName(): ?string
    {
        return $this->name;
    }

    public function setName(string $name): static
    {
        $this->name = $name;

        return $this;
    }

    /**
     * @return Collection<int, Product>
     */
    public function getProducts(): Collection
    {
        return $this->products;
    }
}
```

这里有几个重点：

* `#[ORM\Entity]` 表示这个类是实体
* `#[ORM\Id]` 表示主键
* `#[ORM\GeneratedValue]` 表示主键由数据库生成
* `#[ORM\Column]` 表示普通字段
* `#[ORM\OneToMany]` 表示一对多关系

`products` 不是数据库中的普通字段，而是关联对象集合。

### 创建 Product 实体

商品实体包含名称、价格、库存、创建时间、所属分类。

价格建议用整数保存，单位为分，避免浮点数精度问题。

```php
<?php

namespace App\Entity;

use App\Repository\ProductRepository;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\DBAL\Types\Types;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity(repositoryClass: ProductRepository::class)]
#[ORM\Table(name: 'product')]
class Product
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 150)]
    private ?string $name = null;

    #[ORM\Column]
    private int $price = 0;

    #[ORM\Column]
    private int $stock = 0;

    #[ORM\Column(type: Types::TEXT, nullable: true)]
    private ?string $description = null;

    #[ORM\Column]
    private \DateTimeImmutable $createdAt;

    #[ORM\ManyToOne(inversedBy: 'products')]
    #[ORM\JoinColumn(nullable: false)]
    private ?Category $category = null;

    /**
     * @var Collection<int, OrderItem>
     */
    #[ORM\OneToMany(mappedBy: 'product', targetEntity: OrderItem::class)]
    private Collection $orderItems;

    public function __construct()
    {
        $this->createdAt = new \DateTimeImmutable();
        $this->orderItems = new ArrayCollection();
    }

    public function getId(): ?int
    {
        return $this->id;
    }

    public function getName(): ?string
    {
        return $this->name;
    }

    public function setName(string $name): static
    {
        $this->name = $name;

        return $this;
    }

    public function getPrice(): int
    {
        return $this->price;
    }

    public function setPrice(int $price): static
    {
        $this->price = $price;

        return $this;
    }

    public function getStock(): int
    {
        return $this->stock;
    }

    public function setStock(int $stock): static
    {
        $this->stock = $stock;

        return $this;
    }

    public function getDescription(): ?string
    {
        return $this->description;
    }

    public function setDescription(?string $description): static
    {
        $this->description = $description;

        return $this;
    }

    public function getCreatedAt(): \DateTimeImmutable
    {
        return $this->createdAt;
    }

    public function getCategory(): ?Category
    {
        return $this->category;
    }

    public function setCategory(?Category $category): static
    {
        $this->category = $category;

        return $this;
    }
}
```

`ManyToOne` 是业务开发中最常见的关系。

```php
#[ORM\ManyToOne(inversedBy: 'products')]
#[ORM\JoinColumn(nullable: false)]
private ?Category $category = null;
```

这段代码表示：

```text
多个商品属于同一个分类，product 表中会有 category_id 外键。
```

### 创建 Order 和 OrderItem 实体

订单实体：

```php
<?php

namespace App\Entity;

use App\Repository\OrderRepository;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity(repositoryClass: OrderRepository::class)]
#[ORM\Table(name: '`order`')]
class Order
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\Column(length: 50)]
    private string $orderNo;

    #[ORM\Column]
    private int $totalAmount = 0;

    #[ORM\Column(length: 20)]
    private string $status = 'pending';

    #[ORM\Column]
    private \DateTimeImmutable $createdAt;

    /**
     * @var Collection<int, OrderItem>
     */
    #[ORM\OneToMany(mappedBy: 'order', targetEntity: OrderItem::class, cascade: ['persist'], orphanRemoval: true)]
    private Collection $items;

    public function __construct()
    {
        $this->orderNo = date('YmdHis') . random_int(1000, 9999);
        $this->createdAt = new \DateTimeImmutable();
        $this->items = new ArrayCollection();
    }

    public function getId(): ?int
    {
        return $this->id;
    }

    public function getOrderNo(): string
    {
        return $this->orderNo;
    }

    public function getTotalAmount(): int
    {
        return $this->totalAmount;
    }

    public function setTotalAmount(int $totalAmount): static
    {
        $this->totalAmount = $totalAmount;

        return $this;
    }

    public function getStatus(): string
    {
        return $this->status;
    }

    public function setStatus(string $status): static
    {
        $this->status = $status;

        return $this;
    }

    /**
     * @return Collection<int, OrderItem>
     */
    public function getItems(): Collection
    {
        return $this->items;
    }

    public function addItem(OrderItem $item): static
    {
        if (!$this->items->contains($item)) {
            $this->items->add($item);
            $item->setOrder($this);
        }

        return $this;
    }
}
```

订单明细实体：

```php
<?php

namespace App\Entity;

use App\Repository\OrderItemRepository;
use Doctrine\ORM\Mapping as ORM;

#[ORM\Entity(repositoryClass: OrderItemRepository::class)]
class OrderItem
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\ManyToOne(inversedBy: 'items')]
    #[ORM\JoinColumn(nullable: false)]
    private ?Order $order = null;

    #[ORM\ManyToOne(inversedBy: 'orderItems')]
    #[ORM\JoinColumn(nullable: false)]
    private ?Product $product = null;

    #[ORM\Column]
    private int $quantity = 1;

    #[ORM\Column]
    private int $unitPrice = 0;

    public function getId(): ?int
    {
        return $this->id;
    }

    public function getOrder(): ?Order
    {
        return $this->order;
    }

    public function setOrder(?Order $order): static
    {
        $this->order = $order;

        return $this;
    }

    public function getProduct(): ?Product
    {
        return $this->product;
    }

    public function setProduct(?Product $product): static
    {
        $this->product = $product;

        return $this;
    }

    public function getQuantity(): int
    {
        return $this->quantity;
    }

    public function setQuantity(int $quantity): static
    {
        $this->quantity = $quantity;

        return $this;
    }

    public function getUnitPrice(): int
    {
        return $this->unitPrice;
    }

    public function setUnitPrice(int $unitPrice): static
    {
        $this->unitPrice = $unitPrice;

        return $this;
    }

    public function getAmount(): int
    {
        return $this->unitPrice * $this->quantity;
    }
}
```

`Order` 表名使用了反引号：

```php
#[ORM\Table(name: '`order`')]
```

原因是 `order` 在 SQL 中经常和排序关键字 `ORDER BY` 撞名。实际项目中也可以直接使用 `orders` 作为表名。

### 生成迁移并创建表

实体写好后，需要生成迁移文件：

```shell
php bin/console make:migration
```

执行迁移：

```shell
php bin/console doctrine:migrations:migrate
```

迁移文件通常位于：

```text
migrations/Versionxxxxxxxxxxxx.php
```

迁移文件记录表结构变化，例如创建表、添加字段、创建索引、添加外键等。

常用迁移命令：

```shell
# 查看迁移状态
php bin/console doctrine:migrations:status

# 执行迁移
php bin/console doctrine:migrations:migrate

# 回滚到上一个版本，版本号按实际情况填写
php bin/console doctrine:migrations:migrate prev

# 生成 SQL，不直接执行
php bin/console doctrine:migrations:migrate --dry-run
```

实体类只是 PHP 代码，数据库不会自动多出表。迁移执行成功后，表结构才会真正落到数据库中。

### 新增数据：persist 和 flush

创建一个控制器用于添加分类和商品：

```php
<?php

namespace App\Controller;

use App\Entity\Category;
use App\Entity\Product;
use Doctrine\ORM\EntityManagerInterface;
use Symfony\Bundle\FrameworkBundle\Controller\AbstractController;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\Routing\Attribute\Route;

class ProductDemoController extends AbstractController
{
    #[Route('/demo/products/seed', name: 'demo_products_seed')]
    public function seed(EntityManagerInterface $em): Response
    {
        $category = new Category();
        $category->setName('数码配件');

        $keyboard = new Product();
        $keyboard->setName('机械键盘')
            ->setPrice(39900)
            ->setStock(100)
            ->setDescription('青轴、有线、白色')
            ->setCategory($category);

        $mouse = new Product();
        $mouse->setName('无线鼠标')
            ->setPrice(12900)
            ->setStock(200)
            ->setDescription('蓝牙双模')
            ->setCategory($category);

        $em->persist($category);
        $em->persist($keyboard);
        $em->persist($mouse);
        $em->flush();

        return new Response('初始化完成');
    }
}
```

`persist()` 和 `flush()` 很容易混在一起。

简单理解：

```text
persist()：把对象放进 Doctrine 管理队列
flush()：统一计算变化并执行 SQL
```

上面代码执行时，大致会生成：

```sql
INSERT INTO category (name) VALUES ('数码配件');
INSERT INTO product (name, price, stock, description, created_at, category_id)
VALUES ('机械键盘', 39900, 100, '青轴、有线、白色', '...', 1);
INSERT INTO product (name, price, stock, description, created_at, category_id)
VALUES ('无线鼠标', 12900, 200, '蓝牙双模', '...', 1);
```

`flush()` 之后，数据库生成的主键会回填到实体对象：

```php
$id = $keyboard->getId();
```

### 查询数据：Repository 基础方法

Doctrine 会为实体提供 Repository。

常见方法如下：

```php
$repo = $em->getRepository(Product::class);

// 按主键查询
$product = $repo->find(1);

// 查询单条
$product = $repo->findOneBy(['name' => '机械键盘']);

// 按条件查询多条
$products = $repo->findBy(
    ['category' => $category],
    ['id' => 'DESC'],
    10,
    0
);

// 查询全部
$products = $repo->findAll();
```

也可以直接注入 `ProductRepository`：

```php
use App\Repository\ProductRepository;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\Routing\Attribute\Route;

#[Route('/demo/products', name: 'demo_products')]
public function list(ProductRepository $repository): JsonResponse
{
    $products = $repository->findBy([], ['id' => 'DESC'], 20);

    $data = array_map(static function (Product $product): array {
        return [
            'id' => $product->getId(),
            'name' => $product->getName(),
            'price' => $product->getPrice(),
            'stock' => $product->getStock(),
            'category' => $product->getCategory()?->getName(),
        ];
    }, $products);

    return new JsonResponse($data);
}
```

`findBy()` 适合简单等值查询。

比如：

```php
$repository->findBy(['status' => 'pending']);
```

复杂条件，例如大于、小于、模糊查询、聚合统计、连表查询，更适合使用 `QueryBuilder`。

### 修改数据：脏检查

Doctrine 管理的对象发生属性变化后，不需要再次 `persist()`。

```php
#[Route('/demo/products/{id}/change-price', name: 'demo_products_change_price')]
public function changePrice(
    int $id,
    ProductRepository $repository,
    EntityManagerInterface $em
): Response {
    $product = $repository->find($id);

    if ($product === null) {
        return new Response('商品不存在', 404);
    }

    $product->setPrice(35900);

    $em->flush();

    return new Response('价格修改完成');
}
```

这就是 Doctrine 的变更追踪，也叫脏检查。

执行流程大致如下：

```text
1. 从数据库查出 Product
2. Product 进入 EntityManager 管理状态
3. setPrice() 修改对象属性
4. flush() 对比原始值和当前值
5. 发现 price 变化，生成 UPDATE
```

类似 SQL：

```sql
UPDATE product SET price = 35900 WHERE id = 1;
```

### 删除数据：remove 和 flush

删除商品：

```php
#[Route('/demo/products/{id}/delete', name: 'demo_products_delete')]
public function delete(
    int $id,
    ProductRepository $repository,
    EntityManagerInterface $em
): Response {
    $product = $repository->find($id);

    if ($product === null) {
        return new Response('商品不存在', 404);
    }

    $em->remove($product);
    $em->flush();

    return new Response('删除完成');
}
```

`remove()` 只是标记删除，真正执行 SQL 仍然发生在 `flush()`。

### QueryBuilder：复杂查询更常用

在 `src/Repository/ProductRepository.php` 中添加自定义查询：

```php
<?php

namespace App\Repository;

use App\Entity\Product;
use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
use Doctrine\Persistence\ManagerRegistry;

/**
 * @extends ServiceEntityRepository<Product>
 */
class ProductRepository extends ServiceEntityRepository
{
    public function __construct(ManagerRegistry $registry)
    {
        parent::__construct($registry, Product::class);
    }

    /**
     * @return Product[]
     */
    public function searchAvailableProducts(?string $keyword, int $minPrice, int $maxPrice): array
    {
        $qb = $this->createQueryBuilder('p')
            ->innerJoin('p.category', 'c')
            ->addSelect('c')
            ->andWhere('p.stock > 0')
            ->andWhere('p.price BETWEEN :minPrice AND :maxPrice')
            ->setParameter('minPrice', $minPrice)
            ->setParameter('maxPrice', $maxPrice)
            ->orderBy('p.id', 'DESC')
            ->setMaxResults(20);

        if ($keyword !== null && $keyword !== '') {
            $qb->andWhere('p.name LIKE :keyword')
                ->setParameter('keyword', '%' . $keyword . '%');
        }

        return $qb->getQuery()->getResult();
    }
}
```

调用：

```php
#[Route('/demo/products/search', name: 'demo_products_search')]
public function search(ProductRepository $repository): JsonResponse
{
    $products = $repository->searchAvailableProducts('键盘', 10000, 50000);

    return new JsonResponse(array_map(static function (Product $product): array {
        return [
            'id' => $product->getId(),
            'name' => $product->getName(),
            'price' => $product->getPrice(),
            'category' => $product->getCategory()?->getName(),
        ];
    }, $products));
}
```

`QueryBuilder` 最常见的几个方法：

| 方法 | 作用 |
| --- | --- |
| `createQueryBuilder('p')` | 创建查询构造器，`p` 是实体别名 |
| `andWhere()` | 添加查询条件 |
| `setParameter()` | 绑定参数 |
| `innerJoin()` | 内连接 |
| `leftJoin()` | 左连接 |
| `addSelect()` | 额外查询关联对象 |
| `orderBy()` | 排序 |
| `setMaxResults()` | 限制数量 |
| `setFirstResult()` | 设置偏移量 |
| `getResult()` | 返回多条实体 |
| `getOneOrNullResult()` | 返回一条或 null |

`setParameter()` 很重要，它会使用参数绑定，避免字符串拼接带来的 SQL 注入风险。

### DQL：面向对象的查询语言

Doctrine 还有一种查询方式叫 `DQL`。

它看起来像 SQL，但查询对象是实体类和实体属性，不是表名和字段名。

```php
public function findLowStockProducts(int $stock): array
{
    return $this->getEntityManager()
        ->createQuery(
            'SELECT p FROM App\Entity\Product p WHERE p.stock < :stock ORDER BY p.id DESC'
        )
        ->setParameter('stock', $stock)
        ->getResult();
}
```

注意这里写的是：

```text
App\Entity\Product
p.stock
```

不是：

```text
product
p.stock_column
```

DQL 适合表达稳定的复杂查询。动态条件很多时，`QueryBuilder` 通常更顺手。

### 分页查询

普通分页可以这样写：

```php
public function findPage(int $page, int $pageSize): array
{
    $page = max(1, $page);
    $offset = ($page - 1) * $pageSize;

    return $this->createQueryBuilder('p')
        ->orderBy('p.id', 'DESC')
        ->setFirstResult($offset)
        ->setMaxResults($pageSize)
        ->getQuery()
        ->getResult();
}
```

同时查询总数：

```php
public function countAll(): int
{
    return (int) $this->createQueryBuilder('p')
        ->select('COUNT(p.id)')
        ->getQuery()
        ->getSingleScalarResult();
}
```

列表页常见返回结构：

```php
return new JsonResponse([
    'items' => $items,
    'page' => $page,
    'pageSize' => $pageSize,
    'total' => $total,
]);
```

数据量很大时，深分页会越来越慢。比如 `OFFSET 100000 LIMIT 20`，数据库需要先跳过大量记录。

这类场景可以用基于游标的分页：

```php
public function findNextPage(?int $lastId, int $pageSize): array
{
    $qb = $this->createQueryBuilder('p')
        ->orderBy('p.id', 'DESC')
        ->setMaxResults($pageSize);

    if ($lastId !== null) {
        $qb->andWhere('p.id < :lastId')
            ->setParameter('lastId', $lastId);
    }

    return $qb->getQuery()->getResult();
}
```

这种方式适合信息流、后台滚动加载、导出数据等场景。

### 关联加载：懒加载和主动加载

Doctrine 的关联默认通常是懒加载。

例如：

```php
$product = $productRepository->find(1);
$categoryName = $product->getCategory()->getName();
```

第一行只查商品。

访问 `getCategory()` 时，Doctrine 才查询分类。

如果列表里有 20 个商品，每个商品再访问一次分类，可能变成：

```text
1 次查询商品列表
20 次查询分类
```

这就是常见的 `N + 1` 查询问题。

可以通过 `join + addSelect` 提前加载关联对象：

```php
public function findLatestWithCategory(): array
{
    return $this->createQueryBuilder('p')
        ->innerJoin('p.category', 'c')
        ->addSelect('c')
        ->orderBy('p.id', 'DESC')
        ->setMaxResults(20)
        ->getQuery()
        ->getResult();
}
```

这样商品和分类会在一条查询里取出来，后续访问分类时不用再额外查一次。

### 创建订单：事务实战

下单需要同时做几件事：

* 查询商品
* 判断库存
* 扣减库存
* 创建订单
* 创建订单明细
* 保存总金额

这些操作要么全部成功，要么全部失败，所以需要事务。

可以在 Service 中封装下单逻辑：

```php
<?php

namespace App\Service;

use App\Entity\Order;
use App\Entity\OrderItem;
use App\Repository\ProductRepository;
use Doctrine\ORM\EntityManagerInterface;

class OrderService
{
    public function __construct(
        private readonly EntityManagerInterface $em,
        private readonly ProductRepository $productRepository,
    ) {
    }

    /**
     * @param array<int, array{productId: int, quantity: int}> $items
     */
    public function createOrder(array $items): Order
    {
        return $this->em->wrapInTransaction(function () use ($items): Order {
            $order = new Order();
            $totalAmount = 0;

            foreach ($items as $itemData) {
                $product = $this->productRepository->find($itemData['productId']);

                if ($product === null) {
                    throw new \RuntimeException('商品不存在');
                }

                $quantity = $itemData['quantity'];

                if ($product->getStock() < $quantity) {
                    throw new \RuntimeException('商品库存不足');
                }

                $product->setStock($product->getStock() - $quantity);

                $orderItem = new OrderItem();
                $orderItem->setProduct($product);
                $orderItem->setQuantity($quantity);
                $orderItem->setUnitPrice($product->getPrice());

                $order->addItem($orderItem);
                $totalAmount += $orderItem->getAmount();
            }

            $order->setTotalAmount($totalAmount);

            $this->em->persist($order);

            return $order;
        });
    }
}
```

控制器调用：

```php
use App\Service\OrderService;
use Symfony\Component\HttpFoundation\JsonResponse;
use Symfony\Component\Routing\Attribute\Route;

#[Route('/demo/orders/create', name: 'demo_orders_create')]
public function createOrder(OrderService $orderService): JsonResponse
{
    $order = $orderService->createOrder([
        ['productId' => 1, 'quantity' => 2],
        ['productId' => 2, 'quantity' => 1],
    ]);

    return new JsonResponse([
        'id' => $order->getId(),
        'orderNo' => $order->getOrderNo(),
        'totalAmount' => $order->getTotalAmount(),
    ]);
}
```

`wrapInTransaction()` 会自动开启事务。

闭包正常结束时提交事务；闭包抛出异常时回滚事务。

这段代码还有一个细节：

```php
#[ORM\OneToMany(mappedBy: 'order', targetEntity: OrderItem::class, cascade: ['persist'], orphanRemoval: true)]
private Collection $items;
```

`cascade: ['persist']` 表示保存 `Order` 时，关联的 `OrderItem` 也会一起保存。

所以 Service 中只需要：

```php
$this->em->persist($order);
```

不需要对每个 `OrderItem` 单独 `persist()`。

### flush 的执行时机

Doctrine 不会每调用一次 setter 就执行一次 SQL。

它会先记录对象状态变化，等到 `flush()` 时统一处理。

例如：

```php
$product->setName('新名称');
$product->setPrice(29900);
$product->setStock(50);

$em->flush();
```

通常只会生成一次 `UPDATE`。

这也是 Doctrine 的工作单元模式，也叫 `Unit of Work`。

简单理解：

```text
EntityManager 像一个临时记账本。
persist、remove、setter 都是在记账。
flush 才是结账。
```

### detach、clear 和实体状态

Doctrine 管理的实体有几种常见状态：

| 状态 | 含义 |
| --- | --- |
| `new` | 新创建的对象，还没有交给 EntityManager |
| `managed` | 已经被 EntityManager 管理 |
| `removed` | 已标记删除 |
| `detached` | 曾经被管理，后来脱离了 EntityManager |

示例：

```php
$product = new Product();
$product->setName('显示器');

$em->persist($product); // 进入 managed
$em->flush();           // 写入数据库

$em->detach($product);  // 脱离管理

$product->setName('曲面显示器');
$em->flush();           // 不会更新这个 product
```

`clear()` 会清空 EntityManager 当前管理的全部实体：

```php
$em->clear();
```

批量导入数据时经常会用到它，避免内存持续上涨。

### 批量写入示例

一次导入几万条数据时，不适合每一条都 `flush()`。

可以分批提交：

```php
public function importProducts(Category $category, array $rows, EntityManagerInterface $em): void
{
    foreach ($rows as $index => $row) {
        $product = new Product();
        $product->setName($row['name']);
        $product->setPrice((int) $row['price']);
        $product->setStock((int) $row['stock']);
        $product->setCategory($category);

        $em->persist($product);

        if (($index + 1) % 500 === 0) {
            $em->flush();
            $em->clear();
        }
    }

    $em->flush();
    $em->clear();
}
```

这里的关键点：

```text
每 500 条 flush 一次，把 SQL 分批提交。
每次提交后 clear，释放 EntityManager 对已管理对象的引用。
```

如果 `Category` 在 `clear()` 后还要继续使用，需要重新查询或改成只保存 `category_id` 的批处理写法。

### 常用字段类型

Doctrine 常见字段类型：

| 类型 | PHP 类型 | 数据库含义 |
| --- | --- | --- |
| `string` | `string` | VARCHAR |
| `text` | `string` | TEXT |
| `integer` | `int` | INT |
| `bigint` | `string/int` | BIGINT，不同平台表现略有差异 |
| `boolean` | `bool` | BOOLEAN / TINYINT |
| `decimal` | `string` | DECIMAL，常用于金额 |
| `float` | `float` | 浮点数 |
| `datetime` | `DateTimeInterface` | 可变日期时间 |
| `datetime_immutable` | `DateTimeImmutable` | 不可变日期时间 |
| `json` | `array` | JSON |

示例：

```php
use Doctrine\DBAL\Types\Types;

#[ORM\Column(type: Types::DECIMAL, precision: 10, scale: 2)]
private string $amount = '0.00';

#[ORM\Column(type: Types::JSON)]
private array $extra = [];

#[ORM\Column(type: Types::DATETIME_IMMUTABLE)]
private \DateTimeImmutable $createdAt;
```

金额字段有两种常见处理方式：

```text
1. 使用 integer 保存分，计算简单，适合大多数业务。
2. 使用 decimal 保存小数，注意 Doctrine 中通常映射为 string。
```

### 常用映射属性

| 属性 | 作用 |
| --- | --- |
| `#[ORM\Entity]` | 声明实体类 |
| `#[ORM\Table]` | 指定表名、索引、唯一约束 |
| `#[ORM\Id]` | 声明主键 |
| `#[ORM\GeneratedValue]` | 主键生成策略 |
| `#[ORM\Column]` | 字段映射 |
| `#[ORM\ManyToOne]` | 多对一 |
| `#[ORM\OneToMany]` | 一对多 |
| `#[ORM\OneToOne]` | 一对一 |
| `#[ORM\ManyToMany]` | 多对多 |
| `#[ORM\JoinColumn]` | 外键列配置 |
| `#[ORM\JoinTable]` | 多对多中间表配置 |

索引和唯一约束示例：

```php
#[ORM\Entity(repositoryClass: ProductRepository::class)]
#[ORM\Table(name: 'product')]
#[ORM\Index(columns: ['created_at'], name: 'idx_product_created_at')]
#[ORM\UniqueConstraint(columns: ['name'], name: 'uniq_product_name')]
class Product
{
}
```

### 生命周期回调

生命周期回调适合处理创建时间、更新时间这类通用字段。

```php
#[ORM\Entity]
#[ORM\HasLifecycleCallbacks]
class Article
{
    #[ORM\Id]
    #[ORM\GeneratedValue]
    #[ORM\Column]
    private ?int $id = null;

    #[ORM\Column]
    private \DateTimeImmutable $createdAt;

    #[ORM\Column(nullable: true)]
    private ?\DateTimeImmutable $updatedAt = null;

    #[ORM\PrePersist]
    public function onPrePersist(): void
    {
        $this->createdAt = new \DateTimeImmutable();
    }

    #[ORM\PreUpdate]
    public function onPreUpdate(): void
    {
        $this->updatedAt = new \DateTimeImmutable();
    }
}
```

常见回调：

| 回调 | 时机 |
| --- | --- |
| `PrePersist` | 新实体写入前 |
| `PostPersist` | 新实体写入后 |
| `PreUpdate` | 实体更新前 |
| `PostUpdate` | 实体更新后 |
| `PreRemove` | 删除前 |
| `PostRemove` | 删除后 |
| `PostLoad` | 实体从数据库加载后 |

如果逻辑比较复杂，更适合使用事件监听器或订阅器，避免实体类变得过重。

### 原生 SQL 查询

Doctrine 不限制使用原生 SQL。

复杂报表、特殊数据库函数、性能要求很高的查询，可以直接使用 DBAL：

```php
$connection = $em->getConnection();

$rows = $connection->fetchAllAssociative(
    'SELECT category_id, COUNT(*) AS total_count
     FROM product
     WHERE stock > :stock
     GROUP BY category_id',
    ['stock' => 0]
);
```

返回结果是数组，不是实体对象。

如果查询结果只是统计报表，数组通常更直接。

### 乐观锁示例

并发更新同一条数据时，可以加版本字段。

```php
#[ORM\Column]
#[ORM\Version]
private int $version = 1;
```

当两个请求同时读取同一个商品并修改库存时，Doctrine 会在更新时检查版本。

版本不一致时，会抛出乐观锁异常。

这种方式适合低冲突并发场景。高并发扣库存还需要结合数据库锁、队列、Redis、库存冻结等方案。

### 常见问题

#### 修改了实体，数据库表没有变化

实体类只是映射定义，不会自动改表。

需要执行：

```shell
php bin/console make:migration
php bin/console doctrine:migrations:migrate
```

#### 修改对象后没有更新数据库

常见原因是没有调用：

```php
$em->flush();
```

还有一种情况是对象已经脱离 EntityManager 管理，例如执行过 `clear()` 或 `detach()`。

#### findBy 只能写等值条件

例如：

```php
$repository->findBy(['price' => 10000]);
```

这表示 `price = 10000`。

如果需要 `price > 10000`，使用 `QueryBuilder`：

```php
$repository->createQueryBuilder('p')
    ->andWhere('p.price > :price')
    ->setParameter('price', 10000)
    ->getQuery()
    ->getResult();
```

#### 关联对象访问时查询次数很多

列表场景中，如果循环访问关联对象，容易出现大量额外查询。

可以使用：

```php
->innerJoin('p.category', 'c')
->addSelect('c')
```

提前把关联对象查出来。

#### 批量导入内存越来越高

EntityManager 会持有已管理实体引用。

批量写入时建议分批：

```php
if (($index + 1) % 500 === 0) {
    $em->flush();
    $em->clear();
}
```

### Doctrine 的使用边界

Doctrine 很适合：

* 标准业务 CRUD
* 实体关系清晰的项目
* 后台管理系统
* 需要数据库迁移管理的项目
* 需要 Repository 封装查询逻辑的项目
* 需要事务包裹多个实体变更的业务

以下场景可以混合使用 DBAL 或原生 SQL：

* 大型报表统计
* 极复杂 SQL
* 大批量更新
* 对 SQL 执行计划要求很细的查询
* 只需要返回数组、不需要实体对象的查询

比较实用的做法是：

```text
常规业务用 ORM。
复杂查询用 QueryBuilder 或 DQL。
报表和批处理用 DBAL 或原生 SQL。
```

### 总结

Doctrine 的核心不是把 SQL 藏起来，而是把数据库操作组织成更稳定的对象模型。

最重要的几条主线：

* `Entity` 负责描述表和字段
* `Repository` 负责封装查询
* `EntityManager` 负责管理对象状态
* `persist()` 用于登记新对象
* `flush()` 才是真正提交变更
* `Migration` 负责把实体结构同步到数据库
* `QueryBuilder` 适合复杂条件查询
* 事务适合包裹一组必须同时成功的业务操作

掌握这些主线后，Doctrine 就不只是一个 CRUD 工具，而是 PHP 项目里组织数据访问层的一套完整方案。
