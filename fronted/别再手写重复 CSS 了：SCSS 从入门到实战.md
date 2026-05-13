### 简介

`SCSS` 是 `Sass` 的一种语法格式，可以理解成“加强版 CSS”。

普通 CSS 能写的内容，SCSS 基本都能直接写；在此基础上，SCSS 又加了变量、嵌套、混合、函数、循环、模块拆分等能力。浏览器并不直接执行 SCSS，项目构建时会先把 `.scss` 文件编译成普通 `.css` 文件，然后再交给浏览器使用。

一句话概括：

```text
SCSS 负责让样式代码更好写、更好拆、更好维护，最终产物还是 CSS。
```

### SCSS 和 Sass 的区别

很多文章会把 `Sass` 和 `SCSS` 混着说，实际可以这样区分：

| 名称 | 文件后缀 | 写法特点 |
| --- | --- | --- |
| Sass 缩进语法 | `.sass` | 不写大括号和分号，靠缩进表示层级 |
| SCSS 语法 | `.scss` | 和 CSS 写法很像，有大括号和分号 |

示例：

```scss
.box {
  color: red;
}
```

上面这种就是 SCSS。由于 SCSS 和 CSS 更接近，所以现在项目里更常见。

### 安装与编译

现在推荐使用 `Dart Sass`，命令行包名就是 `sass`。

```shell
npm install -D sass
```

如果只是想在命令行里直接使用：

```shell
npm install -g sass
```

单文件编译：

```shell
sass input.scss output.css
```

监听文件变化：

```shell
sass --watch input.scss:output.css
```

监听整个目录：

```shell
sass --watch src/scss:dist/css
```

在 `Vue`、`React`、`Vite`、`Webpack` 等项目中，一般只要安装 `sass`，构建工具就能处理 `.scss` 文件。

### 变量

变量用 `$` 开头，适合保存颜色、字号、间距、圆角、阴影等经常复用的值。

```scss
$primary-color: #1677ff;
$danger-color: #ff4d4f;
$text-color: #222;
$radius: 6px;
$space: 16px;

.button {
  color: #fff;
  border-radius: $radius;
  background: $primary-color;
  padding: $space;
}

.button-danger {
  background: $danger-color;
}
```

变量最大的价值不是少写几个字符，而是统一管理。比如主色从蓝色改成绿色，只改变量即可。

#### 变量默认值

`!default` 表示：如果前面没有定义过这个变量，就使用当前值；如果已经定义过，就不覆盖。

```scss
$primary-color: #1677ff !default;

.link {
  color: $primary-color;
}
```

组件库、主题包里很常见这种写法。外部可以先定义变量，内部再读取默认值。

### 嵌套

普通 CSS 写层级选择器时，经常重复父选择器：

```css
.nav {
  height: 56px;
}

.nav .item {
  color: #333;
}

.nav .item:hover {
  color: #1677ff;
}
```

换成 SCSS，可以写成：

```scss
.nav {
  height: 56px;

  .item {
    color: #333;

    &:hover {
      color: #1677ff;
    }
  }
}
```

其中 `&` 表示当前父选择器。上面的 `&:hover` 编译后就是 `.nav .item:hover`。

#### 常见的 `&` 用法

```scss
.card {
  padding: 16px;

  &:hover {
    box-shadow: 0 8px 24px rgba(0, 0, 0, 0.12);
  }

  &.active {
    border-color: #1677ff;
  }

  &__title {
    font-size: 18px;
  }

  &__body {
    color: #666;
  }
}
```

编译后：

```css
.card:hover {}
.card.active {}
.card__title {}
.card__body {}
```

这种写法很适合配合 `BEM` 命名。

### 嵌套不要写太深

SCSS 的嵌套很方便，但不能无限套。层级太深会生成又长又重的选择器。

不推荐：

```scss
.page {
  .main {
    .content {
      .list {
        .item {
          .title {
            color: #222;
          }
        }
      }
    }
  }
}
```

生成结果会变成：

```css
.page .main .content .list .item .title {
  color: #222;
}
```

这样的选择器可读性差，覆盖也麻烦。实际项目里一般控制在 2 到 3 层比较舒服。

### 混合 Mixin

`mixin` 用来封装一段可复用的样式。写一次，多处 `@include`。

```scss
@mixin flex-center {
  display: flex;
  align-items: center;
  justify-content: center;
}

.avatar {
  @include flex-center;
  width: 40px;
  height: 40px;
}
```

编译后：

```css
.avatar {
  display: flex;
  align-items: center;
  justify-content: center;
  width: 40px;
  height: 40px;
}
```

#### 带参数的 mixin

```scss
@mixin size($width, $height: $width) {
  width: $width;
  height: $height;
}

.avatar {
  @include size(40px);
}

.banner {
  @include size(100%, 240px);
}
```

`$height: $width` 表示第二个参数有默认值。不传高度时，高度等于宽度。

#### 使用 @content

`@content` 可以把 `@include` 里面的样式插入到 mixin 内部，常用在媒体查询、主题、状态封装里。

```scss
@use "sass:map";

$breakpoints: (
  sm: 576px,
  md: 768px,
  lg: 1024px
);

@mixin respond($name) {
  $width: map.get($breakpoints, $name);

  @media (min-width: $width) {
    @content;
  }
}

.container {
  padding: 12px;

  @include respond(md) {
    padding: 24px;
  }
}
```

编译后：

```css
.container {
  padding: 12px;
}

@media (min-width: 768px) {
  .container {
    padding: 24px;
  }
}
```

### 继承 Extend

`@extend` 可以让一个选择器继承另一个选择器的样式。

```scss
%message {
  padding: 12px 16px;
  border-radius: 6px;
  border: 1px solid #ddd;
}

.message-success {
  @extend %message;
  color: #237804;
  border-color: #b7eb8f;
  background: #f6ffed;
}

.message-error {
  @extend %message;
  color: #a8071a;
  border-color: #ffa39e;
  background: #fff1f0;
}
```

`%message` 是占位选择器，自己不会生成 CSS，只有被 `@extend` 使用时才会输出。

编译后大概是：

```css
.message-success,
.message-error {
  padding: 12px 16px;
  border-radius: 6px;
  border: 1px solid #ddd;
}

.message-success {
  color: #237804;
  border-color: #b7eb8f;
  background: #f6ffed;
}

.message-error {
  color: #a8071a;
  border-color: #ffa39e;
  background: #fff1f0;
}
```

`@extend` 会合并选择器，适合提取一组高度相似的静态样式。参数化样式更适合用 `mixin`。

### 运算

SCSS 支持数字运算，适合处理尺寸、间距、栅格宽度等。

```scss
$base-space: 8px;
$sidebar-width: 240px;

.layout {
  gap: $base-space * 2;
}

.content {
  width: calc(100% - #{$sidebar-width});
}
```

注意：涉及 `100% - 240px` 这种不同单位的计算，建议交给 CSS 的 `calc()`，并用 `#{$variable}` 插值。

#### 除法写法

早期 SCSS 常写：

```scss
.box {
  width: 100px / 2;
}
```

现在更推荐使用模块函数：

```scss
@use "sass:math";

.box {
  width: math.div(100px, 2);
}
```

这样可以避免 `/` 和 CSS 原生斜杠语法冲突。

### 插值

插值语法是 `#{}`，可以把变量拼进选择器、属性名、字符串里。

```scss
$name: primary;
$property: color;

.text-#{$name} {
  #{$property}: #1677ff;
}
```

编译后：

```css
.text-primary {
  color: #1677ff;
}
```

循环批量生成类名时，插值很常用。

### 列表和 Map

列表类似数组，Map 类似键值对。

```scss
$sizes: 4, 8, 12, 16, 24;

@each $size in $sizes {
  .mt-#{$size} {
    margin-top: #{$size}px;
  }
}
```

编译后：

```css
.mt-4 { margin-top: 4px; }
.mt-8 { margin-top: 8px; }
.mt-12 { margin-top: 12px; }
.mt-16 { margin-top: 16px; }
.mt-24 { margin-top: 24px; }
```

Map 示例：

```scss
$theme-colors: (
  primary: #1677ff,
  success: #52c41a,
  warning: #faad14,
  danger: #ff4d4f
);

@each $name, $color in $theme-colors {
  .btn-#{$name} {
    background: $color;
    border-color: $color;
  }
}
```

这类写法适合生成工具类、主题类、按钮状态类。

### 条件语句

条件语句使用 `@if`、`@else if`、`@else`。

```scss
@mixin button-variant($type) {
  @if $type == primary {
    background: #1677ff;
    color: #fff;
  } @else if $type == danger {
    background: #ff4d4f;
    color: #fff;
  } @else {
    background: #f5f5f5;
    color: #333;
  }
}

.btn-primary {
  @include button-variant(primary);
}

.btn-danger {
  @include button-variant(danger);
}
```

条件语句适合封装有分支的 mixin，但不建议把样式写成复杂业务逻辑。SCSS 的职责还是生成 CSS。

### 循环

SCSS 常用三种循环：`@for`、`@each`、`@while`。

#### @for

```scss
@for $i from 1 through 4 {
  .col-#{$i} {
    width: $i * 25%;
  }
}
```

`through` 包含结束值，`to` 不包含结束值。

```scss
@for $i from 1 to 4 {
  .item-#{$i} {
    z-index: $i;
  }
}
```

上面只会生成 `.item-1`、`.item-2`、`.item-3`。

#### @each

```scss
$directions: top, right, bottom, left;

@each $dir in $directions {
  .border-#{$dir} {
    border-#{$dir}: 1px solid #ddd;
  }
}
```

#### @while

```scss
$i: 1;

@while $i <= 3 {
  .z-#{$i} {
    z-index: $i;
  }

  $i: $i + 1;
}
```

`@while` 用得不多，能用 `@for` 或 `@each` 解决时，优先使用更清晰的写法。

### 函数

`@function` 用来返回一个值，和 `mixin` 不一样。`mixin` 输出一段样式，函数输出一个计算结果。

```scss
@use "sass:math";

@function rem($px, $base: 16px) {
  @return math.div($px, $base) * 1rem;
}

.title {
  font-size: rem(24px);
}

.desc {
  font-size: rem(14px);
}
```

编译后：

```css
.title {
  font-size: 1.5rem;
}

.desc {
  font-size: 0.875rem;
}
```

### 颜色处理

Sass 内置了颜色相关模块。现代写法推荐 `@use "sass:color"`。

```scss
@use "sass:color";

$primary: #1677ff;

.button {
  background: $primary;

  &:hover {
    background: color.adjust($primary, $lightness: -8%);
  }

  &:disabled {
    background: color.adjust($primary, $lightness: 28%);
  }
}
```

以前常见的 `lighten()`、`darken()` 仍能在很多项目里看到，但新项目更推荐使用模块化函数。

### 模块化：@use 和 @forward

老项目里经常看到 `@import`：

```scss
@import "./variables";
@import "./mixins";
```

现在 Sass 更推荐 `@use` 和 `@forward`。原因是 `@import` 容易造成全局变量污染、重复加载、来源不清楚等问题。

#### _variables.scss

```scss
$primary: #1677ff;
$radius: 6px;
$space: 16px;
```

#### _mixins.scss

```scss
@mixin flex-center {
  display: flex;
  align-items: center;
  justify-content: center;
}
```

#### index.scss

```scss
@forward "variables";
@forward "mixins";
```

#### button.scss

```scss
@use "./index" as ui;

.button {
  @include ui.flex-center;
  border-radius: ui.$radius;
  background: ui.$primary;
}
```

`@use "./index" as ui;` 的意思是：把 `index.scss` 里的内容放到 `ui` 命名空间下。这样变量和 mixin 来源很清楚。

如果嫌命名空间太长，也可以：

```scss
@use "./index" as *;
```

这样可以直接使用 `$primary`、`flex-center`。不过大型项目里更建议保留命名空间，避免重名。

### 常见目录结构

中小项目可以这样拆：

```text
src/
└── styles/
    ├── abstracts/
    │   ├── _variables.scss
    │   ├── _mixins.scss
    │   ├── _functions.scss
    │   └── index.scss
    ├── base/
    │   ├── _reset.scss
    │   └── _common.scss
    ├── components/
    │   ├── _button.scss
    │   └── _card.scss
    └── main.scss
```

说明：

* `abstracts`：只放变量、函数、mixin，不直接生成大段 CSS
* `base`：放重置样式、全局基础样式
* `components`：放按钮、卡片、表单等组件样式
* `main.scss`：统一入口

`main.scss` 示例：

```scss
@use "./base/reset";
@use "./base/common";
@use "./components/button";
@use "./components/card";
```

### 完整 Demo：商品卡片

下面用一个商品卡片示例串一下 SCSS 的常用能力：变量、Map、mixin、函数、嵌套、状态类、响应式。

#### HTML

```html
<section class="product-list">
  <article class="product-card">
    <div class="product-card__cover">
      <span class="product-card__tag product-card__tag--hot">热卖</span>
    </div>

    <div class="product-card__body">
      <h3 class="product-card__title">机械键盘 K87</h3>
      <p class="product-card__desc">三模连接，热插拔轴体，适合办公和游戏。</p>

      <div class="product-card__footer">
        <strong class="product-card__price">￥399</strong>
        <button class="btn btn-primary">加入购物车</button>
      </div>
    </div>
  </article>

  <article class="product-card product-card--disabled">
    <div class="product-card__cover">
      <span class="product-card__tag product-card__tag--new">新品</span>
    </div>

    <div class="product-card__body">
      <h3 class="product-card__title">人体工学鼠标 M2</h3>
      <p class="product-card__desc">轻量化设计，长时间使用更省力。</p>

      <div class="product-card__footer">
        <strong class="product-card__price">￥159</strong>
        <button class="btn btn-disabled">暂时缺货</button>
      </div>
    </div>
  </article>
</section>
```

#### SCSS

```scss
@use "sass:color";
@use "sass:math";

$primary: #1677ff;
$text: #1f2329;
$muted: #646a73;
$border: #e5e6eb;
$surface: #fff;
$radius: 8px;
$space: 8px;

$tag-colors: (
  hot: #ff4d4f,
  new: #52c41a
);

@function rem($px, $base: 16px) {
  @return math.div($px, $base) * 1rem;
}

@mixin flex-between {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

@mixin button-base {
  border: 0;
  height: 36px;
  padding: 0 14px;
  cursor: pointer;
  font-size: rem(14px);
  border-radius: $radius - 2px;
}

@mixin respond-md {
  @media (min-width: 768px) {
    @content;
  }
}

.product-list {
  display: grid;
  gap: $space * 2;
  padding: $space * 2;
  background: #f7f8fa;

  @include respond-md {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}

.product-card {
  overflow: hidden;
  border: 1px solid $border;
  border-radius: $radius;
  background: $surface;
  transition: transform 0.2s ease, box-shadow 0.2s ease;

  &:hover {
    transform: translateY(-2px);
    box-shadow: 0 10px 24px rgba(31, 35, 41, 0.1);
  }

  &--disabled {
    opacity: 0.6;

    &:hover {
      transform: none;
      box-shadow: none;
    }
  }

  &__cover {
    position: relative;
    height: 160px;
    background:
      linear-gradient(135deg, rgba(22, 119, 255, 0.18), rgba(82, 196, 26, 0.16)),
      #eef3ff;
  }

  &__tag {
    position: absolute;
    top: $space;
    left: $space;
    color: #fff;
    font-size: rem(12px);
    line-height: 1;
    padding: 5px 8px;
    border-radius: 999px;
  }

  @each $name, $color in $tag-colors {
    &__tag--#{$name} {
      background: $color;
    }
  }

  &__body {
    padding: $space * 2;
  }

  &__title {
    margin: 0;
    color: $text;
    font-size: rem(18px);
  }

  &__desc {
    margin: $space 0 0;
    color: $muted;
    font-size: rem(14px);
    line-height: 1.7;
  }

  &__footer {
    @include flex-between;
    gap: $space;
    margin-top: $space * 2;
  }

  &__price {
    color: #f5222d;
    font-size: rem(20px);
  }
}

.btn {
  @include button-base;

  &-primary {
    color: #fff;
    background: $primary;

    &:hover {
      background: color.adjust($primary, $lightness: -8%);
    }
  }

  &-disabled {
    color: #999;
    cursor: not-allowed;
    background: #f0f0f0;
  }
}
```

这个 demo 里出现了几个典型场景：

* `$primary`、`$radius`、`$space` 统一管理主题值
* `rem()` 函数统一处理字号单位转换
* `flex-between` 和 `button-base` 复用样式片段
* `@each` 根据 `$tag-colors` 自动生成标签样式
* `&__title`、`&--disabled` 配合 BEM 命名组织组件样式
* `respond-md` 把响应式媒体查询封装起来

### 编译后的部分 CSS

SCSS 最终还是会生成普通 CSS。以上 demo 中，部分编译结果大概如下：

```css
.product-list {
  display: grid;
  gap: 16px;
  padding: 16px;
  background: #f7f8fa;
}

@media (min-width: 768px) {
  .product-list {
    grid-template-columns: repeat(2, minmax(0, 1fr));
  }
}

.product-card__tag--hot {
  background: #ff4d4f;
}

.product-card__tag--new {
  background: #52c41a;
}

.btn-primary {
  color: #fff;
  background: #1677ff;
}
```

写 SCSS 时，脑子里要一直有一件事：最后产物是 CSS。只要生成的 CSS 清晰、体积合理、覆盖关系不混乱，SCSS 才算用得合适。

### SCSS 和 CSS 变量怎么选

SCSS 变量和 CSS 变量不是一回事。

| 对比项 | SCSS 变量 | CSS 变量 |
| --- | --- | --- |
| 写法 | `$primary` | `--primary` |
| 生效时间 | 编译时 | 浏览器运行时 |
| 能否被 JS 动态修改 | 不行 | 可以 |
| 是否能响应主题切换 | 编译后固定 | 很适合 |
| 适合场景 | 编译期计算、生成类名、统一常量 | 动态主题、运行时换肤 |

SCSS 变量示例：

```scss
$primary: #1677ff;

.btn {
  background: $primary;
}
```

CSS 变量示例：

```scss
:root {
  --primary: #1677ff;
}

.btn {
  background: var(--primary);
}
```

实际项目里可以混用：布局尺寸、断点、工具类生成适合 SCSS 变量；主题色、暗黑模式、用户自定义皮肤适合 CSS 变量。

### 常见坑

#### @import 不推荐继续大量使用

老项目可以继续维护，新项目更建议使用 `@use` 和 `@forward`。`@use` 有命名空间，变量来源更明确。

#### 嵌套太深会让 CSS 变重

SCSS 的嵌套不是为了照抄 HTML 结构，而是为了组织组件内部关系。层级太深会导致选择器难覆盖。

#### mixin 会复制代码

每次 `@include` 都会把 mixin 里的样式复制一份。如果一段样式被大量重复输出，CSS 体积会变大。

#### extend 会合并选择器

`@extend` 输出结果有时不如预期，尤其跨文件、跨层级使用时更明显。公共组件样式优先考虑清晰的类名和 mixin。

#### 不要把 SCSS 当编程语言滥用

条件、循环、函数都很好用，但样式文件不是业务逻辑文件。能直接写清楚的 CSS，不一定要封装成复杂工具。

### 实用总结

SCSS 最值得掌握的内容：

* 变量：统一管理颜色、间距、圆角、层级等基础值
* 嵌套：让组件内部结构更清楚，配合 `&` 写状态和 BEM
* mixin：封装可复用样式，尤其是响应式、布局、按钮基础样式
* 函数：处理单位转换、尺寸计算等返回值场景
* Map 和循环：批量生成主题类、间距类、按钮状态类
* @use 和 @forward：让样式文件模块化，避免全局污染

SCSS 的核心价值不是“让 CSS 看起来像代码”，而是把重复、分散、难维护的样式整理成更稳定的结构。小页面可以少用，复杂项目越大，变量、模块、mixin、Map 的价值越明显。
