## 一、什么是Composer？

Composer 是 PHP 中的依赖管理工具。它允许声明项目所依赖的库，并且它将为您管理（安装/更新）它们。

## 二、如何安装？

> Linux 系统和 MacOS 系统

直接下载最新稳定版：

![alt text](/images/composer-image.png)

然后执行下列命令，放到/usr/local/bin下面

```shell
sudo mv composer.phar /usr/local/bin/composer

sudo chmod +x /usr/local/bin/composer

执行composer -V 查看是否安装成功
```

> Windows系统

第一步也是下载最新稳定版，然后基于 composer.phar文件创建 composer.bat文件

* 使用cmd时，执行：

```shell
echo @php "%~dp0composer.phar" %*>composer.bat
```

* 使用PowerShell时，执行：

```shell
Set-Content composer.bat '@php "%~dp0composer.phar" %*'
```

composer.bat文件所在的目录要配到环境变量中，然后才能执行 composer -V，所以一般建议可以直接放到php的可执行目录中。

## 三、composer.json VS composer.lock

### 什么是composer.json？

composer.json文件是composer使用json格式描述项目依赖的配置文件，其中包含一些项目元数据及其依赖配置等。

### 什么是composer.lock？

composer.lock文件是composer安装完依赖后，生成的版本锁定文件，以确保在项目中工作的每个人的包版本一致。

### 两者应用场景

当项目第一次初始化时，需要运行 **update** 命令，composer从composer.json文件查找依赖项，获取依赖的版本并写入到composer.lock文件中，最后隐式调用 **install** 命令，下载所有依赖包默认放到项目根目录的vendor目录中，composer.lock文件应该提交到版本控制系统中，例如：git。

当项目已经初始化过，其他人员拉取项目，安装依赖，只需要执行 **install** 命令，composer会从composer.lock中获取依赖项来下载依赖包。

如果想要更新依赖包版本，需要执行 **update** 命令，composer会更新composer.json和composer.lock文件，并下载最新的依赖包。

## 四、什么是Autoloading（自动加载）？

对于指定自动加载信息的库，Composer 会生成一个vendor/autoload.php 文件，然后引入依赖包时，只需要使用 `require __DIR__ . '/vendor/autoload.php';`，不用 **require** 单独引入使用的包。

例如：

```php
require __DIR__ . '/vendor/autoload.php';

$log = new Monolog\Logger('name');
$log->pushHandler(new Monolog\Handler\StreamHandler('app.log', Monolog\Logger::WARNING));
$log->warning('Foo');
```

## 五、什么是PSR0、PSR4、classmap？

### PSR0示例：

![alt text](/images/composer-image-1.png)

**lib/abc/src/Test2.php** 文件命名空间是 **abc\src**

那么在 **composer.json** 文件中的 **autoload** 项 **psr-0** 节点要把命名空间配在键上面，值要配最外层目录，此处为 **lib**，一般 **composer** 包的话是 **vendor**；

```php
{
  "autoload": {
    "psr-0": {
      "abc\\src": "lib/"
    }
  }
}
```

**autoload** 加载规则是拼接 **lib** 为路径前缀，即：**/lib/abc/src**；

**composer.json** 加上配置后，要使用 `composer dump-autoload` 生成位于 **vendor** 目录中的 **psr0** 自动加载文件

![alt text](/images/composer-image-3.png)

注意：**psr0** 生成的是 **autoload_namespaces.php** ，**psr4** 生成的是 **autoload_psr4.php**，然后就可以从其他文件中访问了。

例如：访问 **abc/src/Test2.php** ，即查找 **/lib/abc/src/Test2.php** 。

如图在 **app/sms/src/Test.php** 要想访问 **Test2.php** ，首先用 **require_once**加载 **autoload.php**，然后就可以访问了。

![alt text](/images/composer-image-2.png)

### PSR4示例：

![alt text](/images/composer-image-4.png)

**lib2/abc/src/Test3.php** 命名空间是 **Lib2\Abc**

那么在 **composer.json** 文件的 **autoload** 项 **psr-4** 节点要把命名空间配到键上面，文件全路径配到值上面，则 **composer** 在解析 **Lib2/Abc** 时，会找到 **lib2/abc/src** 里面的文件。

![alt text](/images/composer-image-5.png)

再次执行 `composer dump-autoload` 生成 **autoload_psr4.php** 文件内容

![alt text](/images/composer-image-7.png)

从 **Test.php** 中访问 **Test3.php** 中的方法：

![alt text](/images/composer-image-8.png)

以上示例基本就能理解PSR-0、PSR-4到底是怎么用了，PSR官方的文档一言难尽。

### classmap示例：

`classmap` 是 `composer` 使用的简单类映射，用于在没有遵循PSR-0/4规范的模块或类文件时，通过配置自动加载。

![alt text](/images/composer-image-18.png)

`classmap` 还支持通配符的路径方式，例如：

```json
{
    "autoload": {
        "classmap": ["src/addons/*/lib/", "3rd-party/*", "Something.php"]
    }
}
```

直接在 `autoload` 的 `classmap` 属性键下配置需要自动加载的目录或文件，当然也需要再次执行 `composer dump-autoload`，会在 **autoload_classmap.php** 文件中生成映射内容。

![alt text](/images/composer-image-19.png)

然后就可以在其他类直接使用了：

![alt text](/images/composer-image-20.png)


## 六、常用命令

* 列出完整命令列表

```shell
composer 或

composer list
```

* `--verbose (-v)`：打印详细信息

示例：

```shell
# 一个v正常打印输出，两个v打印更多信息，三个v为debug模式
composer update -v

composer update -vv

composer update -vvv
```

* `--help (-h)`：打印帮助信息

示例：

```shell
composer -h
```

* `--quiet (-q)`：静音模式，不输出任何信息

示例：

```shell
composer update -q
```

* `--no-interaction (-n)`：非交互式模式，不询问任何问题

示例：

```shell
composer update -n
```

* `-no-plugins`：禁用插件

示例：

```shell
composer install --no-plugins
```

* `--no-scripts`：跳过定义在composer.json中的脚本执行

示例：

```shell
composer install --no-scripts
```

* `--no-cache`：禁用使用缓存目录

示例：

```shell
composer install --no-cache
```

* `--profile`：显示时间和内存使用信息

示例：

```shell
composer install --profile
```

* `--version (-V)`：打印版本信息

示例：

```shell
composer -V
```

* `init`：初始化composer.json文件，交互式询问填充的值

示例：

```shell
composer init
```

* `install或i`：安装依赖包

如果没有composer.lock文件，则composer读取composer.json文件，下载并安装依赖包到vendor目录，最后创建composer.lock文件。

如果已存在composer.lock文件，则composer从composer.lock文件中读取依赖版本进行安装，以此保证每个人安装的包版本是一致的。

示例：

```shell
composer install
```

> 常用的选项：

1. `--prefer-install`：选择从可发布版的包（dist）还是从源码（source）下载，默认是dist，下载源码一般用于贡献开源、bug修复等。

示例：

```shell
composer install --prefer-install=source
```

2. `--dry-run`：模拟下载安装过程，不实际安装包

示例：

```shell
composer install --dry-run
```

![alt text](/images/composer-image-9.png)

实际上是做了一个包兼容性检查，最后少了实际下载安装过程。

3. `--dev`：也安装列在 **require-dev** 键指定的开发所需的包，此为默认行为

4. `--no-dev`：跳过安装在 **require-dev** 键指定的开发所需的包。

示例：

```shell
composer i --no-dev
```

* `update 或 u 或 upgrade`：

更新依赖包版本，然后更新composer.json和composer.lock文件。

示例：

```shell
# 更新所有包
composer update

# 更新指定的包，可以一次性指定多个
composer update monolog/monolog guzzlehttp/guzzle
```

* `require 或 r`

添加依赖包，如果composer.json存在，则添加依赖包版本约束；如果composer.json不存在，则自动创建。

示例：
```shell
# 如果require后面不指定依赖包名，则composer会提示你输入包名来搜索
composer require

# 指定依赖包名即版本约束，注意版本带点："."，要用双引号括起来，否则命令行会解析失败
composer require "monolog/monolog:3.0.*" "symfony/console:7.0.*"

# 只指定依赖包名，不指定版本约束，composer会基于可用的包版本、已安装的其他依赖包、php版本来选择合适的版本。
composer require monolog/monolog symfony/console
```

> 常用的选项：

1. `--dev`：添加包到 `require-dev` 指令项。

2. `--dry-run`：同上。

3. `--prefer-install`：同上。

* `remove或rm或uninstall`：

移除在composer.json中指定的依赖包，同时更新composer.lock文件，并卸载/删除包文件。

示例：

```shell
composer remove monolog/monolog symfony/console
```

> 常用的选项：

1. `--unused`：移除未使用的依赖包

2. `--dev`：仅仅移除 `require-dev` 指定的依赖包

3. `--dry-run`：同上。

* `reinstall`：重新安装依赖包

一般用于依赖包源文件被修改了，想复原，或者想安装发布版本或源码版本。

示例：

```shell
composer reinstall monolog/monolog symfony/console
```

* `check-platform-reqs`：检查PHP版本和扩展是否匹配依赖包的版本

示例：

```shell
composer check-platform-reqs
```

> 常用的选项：

1. `--lock`：仅仅从composer.lock文件中获取依赖包

2. `--no-dev`：禁用检查 `require-dev` 指令项列出的包

3. `--format或-f`：格式化输出，默认是text，可以为json。

* `global`：对包进行全局管理，例如：安装、更新、移除等。

安装的包在 `COMPOSER_HOME` composer家目录下面的vendor目录

示例：

```shell
composer global require monolog/monolog symfony/console
```

* `search`：指定包名搜索依赖包，默认从packagist仓库搜索结果。

示例：

```shell
composer search monolog/monolog symfony/console
```

* `show或info`：列出项目已安装的所有可用的包

示例：

```shell
# 显示所有
composer show

# 指定包
composer show monolog/monolog symfony/console
```

> 常用的选项：

1. `--self或-s`：仅仅列出根包的信息，即当前的composer.json文件中的信息。

2. `--tree或-t`：以树形结构显示包的依赖关系。

![alt text](/images/composer-image-12.png)

3. `--latest或-l`：列出所有安装的包和最新的版本。

4. `--outdated或-o`：仅仅列出有新版本可用的包。 

* `outdated`：列出具有可用更新的已安装软件包的列表，显示出当前的版本和最新的版本，此命令为：composer show -lo的别名。

示例：

```shell
composer outdated
```

![alt text](/images/composer-image-10.png)

输出内容的颜色有如下含义：

`green (=)`：依赖项当前版本就是最新版本
`yellow (~)`：依赖项有一个可用的新版本，其中包括根据语义的向后兼容性中断，因此请尽可能升级。
`red (!)`：依赖项有一个语义兼容的新版本，应该要升级了。

* `browse或home`：调用浏览器打开依赖包的仓库地址或主页。

示例：

```shell
composer browse
```

* `suggests`：列出所有包或指定包的的建议。

示例：

```shell
# 默认列出所有包的建议
composer suggests

# 列出指定包的建议
composer suggests monolog/monolog
```

![alt text](/images/composer-image-13.png)

![alt text](/images/composer-image-14.png)

* `depends / why`：哪些其他包依赖于指定的包。

> 常用选项：

1. `--recursive或-r`：递归解析到根包。

2. `--tree或-t`：打印树状结构，使用-t时会隐式调用-r。

示例：

```shell
# 正常输出被依赖的包
composer depends psr/log

# 以树形结构输出被依赖的包
composer depends -t psr/log 
```

![alt text](/images/composer-image-15.png)

![alt text](/images/composer-image-16.png)

* `prohibits或why-not`：查找是否能安装指定版本的包，可能有其他包依赖此包的当前版本，所以不能升级到新版本。

> 常用选项：

1. `--recursive或-r`：递归解析到根包。

2. `--tree或-t`：打印树状结构，使用-t时会隐式调用-r。

示例：

```shell
composer prohibits symfony/symfony 3.1 

laravel/framework v5.2.16 requires symfony/var-dumper (2.8.*|3.0.*)
```

* `validate`：检查 `composer.json` 文件是否有json语法错误。

示例：

```shell
composer validate
```

* `status`：如果依赖包是从源码安装的，则在本地对依赖包的源码进行了更改，则使用 `status` 可以检测出代码有哪些改动，类似于 `git` 的 `status`。

示例：

```shell
# 简单输出改动的地方
composer status

# 详细输出改动的信息，使用-v或--verbose
composer status -v

You have changes in the following dependencies:
vendor/seld/jsonlint:
    M README.mdown
```

* `self-update或selfupdate`：更新composer本身到最新版本，或者指定一个版本进行更新。

> 常用的选项：

1. `--rollback或-r`：回退到上一个版本

2. `--clean-backups`：删除所有旧版本的备份，缓存中现仅存当前版本，所以不能回退了。

示例：

```shell
composer self-update
```

* `config`：列出 `composer` 的配置项或修改配置。

修改配置的话建议直接编辑配置文件，在命令行多个配置项连在一块容易混乱。

全局配置文件路径：`$COMPOSER_HOME/config.json`，`$COMPOSER_HOME` 表示 `composer` 的目录，所以真实路径一般为： `~/.composer/config.json`

项目单个配置文件，则为 `composer.json` 了。

> 常用的选项：

1. `--list或-l`：列出当前的配置项，包括当前项目和全局的，如果指定了 `--global` 则只列出全局的配置项。

2. `--unset`：移除指定的配置项。

示例：

```shell
#列出所有配置项
composer config --list

# 移除指定的配置项
composer config --unset repositories
```

*  `create-project`：下载依赖包作为项目来开发，其实质意义与 `git clone` 拉取源代码大致相同。

示例：

```shell
# 依赖包后面可以指定安装后的项目名，此处为：doctrine_orm，也可以在项目名后指定版本，不指定默认安装最新版本。
composer create-project doctrine/orm doctrine_orm
```

* `dump-autoload 或 dumpautoload`：在自定义添加了 `PSR-0或PSR-4` 的类映射后，需要执行此命令重新生成在 `vendor` 目录中的 `autoload_namespaces.php` 或 `autoload_psr4.php` 文件。

示例：

```shell
composer dump-autoload
```

* `clear-cache 或 clearcache 或 cc`：删除 `composer` 缓存目录中的所有内容。

示例：

```shell
composer clear-cache
```

* `run-script 或 run`：运行用户定义的脚本。

> 常用选项：

1. `--list 或 -l`：列出用户定义的脚本

示例：

```shell
# 列出用户定义的脚本
composer run-script -l

# 运行指定的脚本
composer run-script script1
```

* `diagnose`：诊断 `composer` 出现的bug和一些奇怪的问题。

示例：

```shell
composer diagnose
```

![alt text](/images/composer-image-17.png)

* `archive`：把项目打成压缩包，且把忽略的文件排除在外。

> 常用的选项：

1. `--format 或 -f`：指定压缩包的格式，有：tar, tar.gz, tar.bz2, zip，默认是tar。

2. `--dir`：指定压缩包写入的目录名，默认是当前目录：`.`。

3. `--file`：指定压缩包的名称，默认是依赖包的名称。

示例：

```shell
composer archive vendor/package 2.0.21 --format=zip
```

* `help`：打印帮助信息。

> 常用选项：

1. `--format`：指定格式，有txt, xml, json, md可用，默认是txt

示例：

```shell
composer help

# 指定子命令
composer help install

# 指定输出的格式
composer help install md
```

## 七、composer.json 常用属性(或者叫字段)解析

* `name`：项目名或包名，例如：monolog/monolog。

包名的规则是：厂商名/包名，全小写，多个单词用 `-`、`.` 或 `_` 分割。

如果当前包已经发布过了，则name属性必须填写。

* `description`：包名的描述。

* `version`：包的版本，一般建议不写，composer可以从包仓库中自动推断。

版本规则：`X.Y.Z` 或 `vX.Y.Z`，可接的后缀有：-dev, -patch (-p), -alpha (-a), -beta (-b), -RC

示例：

```shell
1.0.2

1.0.2-dev

v.2.0.2-dev
```

* `type`：包的类型，默认是：`library`。

1. `library`：表示包是类库项目，会安装到 `vendor` 目录

2. `project`：表示包是标准应用项目。

* `keywords`：包相关的关键词数组，用于后续搜索可过滤，`可选`。

示例：

```json
"keywords": [
  "yii2",
  "api",
  "framework",
  "templating"
]
```

* `homepage`：项目网站URL，或 packagist 上的URL位置，`可选`。

* `license`：包的许可声明，可以是单个字符串为或字符串数组。

常见的license有：

1. Apache-2.0

2. BSD-2-Clause

3. BSD-3-Clause

4. BSD-4-Clause

5. GPL-2.0-only / GPL-2.0-or-later

6. GPL-3.0-only / GPL-3.0-or-later

7. LGPL-2.1-only / LGPL-2.1-or-later

8. LGPL-3.0-only / LGPL-3.0-or-later

9. MIT

示例：

```json
// 单个license
{
  "license": "MIT"
}

// 多个license，或的关系，通过数组指定
{
  "license": [
      "LGPL-2.1-only",
      "GPL-3.0-or-later"
  ]
}

// 多个license，或的关系，也可以用()加or来指定
{
  "license": "(LGPL-2.1-only or GPL-3.0-or-later)"
}

// 如果多个license，是且的关系，则用and来指定
{
  "license": "(LGPL-2.1-only and GPL-3.0-or-later)"
}
```

* `authors`：指定包的作者，此属性是一个对象数组，每个对象可包括以下属性：

1. `name`：作者名字

2. `email`：邮箱

3. `homepage`：站点主页

4. `role`：作者在这个项目担任的角色

示例：

```json
{
    "authors": [
        {
            "name": "Nils Adermann",
            "email": "naderman@naderman.de",
            "homepage": "https://www.naderman.de",
            "role": "Developer"
        },
        {
            "name": "Jordi Boggiano",
            "email": "j.boggiano@seld.be",
            "homepage": "https://seld.be",
            "role": "Developer"
        }
    ]
}
```

* `require`：依赖的包。

示例：

```json
{
    "require": {
        "monolog/monolog": "1.0.*"
    }
}
```

* `require-dev`：仅开发期间或运行测试时所需的依赖包。

```json
{
    "require-dev": {
        "monolog/monolog": "1.0.*"
    }
}
```

* `conflict`：指定与依赖包所冲突的包，composer不会安装此处指定的包。

示例：

```json
{
    "conflict": {
        "monolog/monolog": "1.0.*"
    }
}
```

* `suggest`：可以增强此软件包或与此软件包配合良好的建议软件包，在安装包后显示，以提示用户可以添加更多包，即使它们不是严格要求的。

示例：

```json
{
    "suggest": {
        "monolog/monolog": "Allows more advanced logging of the application flow",
        "ext-xml": "Needed to support XML format in class Foo"
    }
}
```

* `autoload`：自动加载映射，包括主要的 `PSR-0` 、 `PSR-4` 、`classmap`，如上所述。

还包括：

`exclude-from-classmap`：指定自动加载排除的目录或文件

示例：

```json
{
    "autoload": {
        "exclude-from-classmap": ["/Tests/", "/test/", "/tests/"]
    }
}
```

* `autoload-dev`：定义仅在开发阶段的自动加载规则

示例：

```json
{
    "autoload-dev": {
        "psr-4": { "MyLibrary\\Tests\\": "tests/" }
    }
}
```

* `repositories`：指定包的仓库，`composer` 默认是使用 `packagist` 中心仓库。

仓库支持三种类型：

1. `composer`：类似 `packagist` 的仓库。

2. `vcs`：版本控制系统仓库，如：git。

3. `package`：当加载的是不支持 `composer` 的类库项目，使用此选项。

示例：

```json
{
    "repositories": [
        {
            "type": "composer",
            "url": "http://packages.example.com"
        },
        {
            "type": "vcs",
            "url": "https://github.com/Seldaek/monolog"
        },
        {
            "type": "package",
            "package": {
                "name": "smarty/smarty",
                "version": "3.1.7",
                "dist": {
                    "url": "https://www.smarty.net/files/Smarty-3.1.7.zip",
                    "type": "zip"
                }
            }
        }
    ]
}
```

也可以使用json对象提供仓库名称的方式来指定，例如：

```json
{
    "repositories": {
        "foo": {
            "type": "composer",
            "url": "http://packages.foo.com"
        },

        // 通过指定仓库名为：packagist，使用镜像来覆盖默认的仓库地址
        "packagist": {
            "type": "composer",
            "url": "https://mirrors.aliyun.com/composer/"
        }
    }
}
```

* `config`：项目的配置项。

示例：

```json
{
    // 启动指定的插件
    "config": {
        "allow-plugins": {
            "third-party/required-plugin": true,
            "my-organization/*": true,
            "unnecessary/plugin": false
        }
    }

    // 禁用所有插件
    "config": {
        "allow-plugins": false
    }

    // 启用所有插件
    "config": {
        "allow-plugins": true
    }
}
```

* `scripts`：composer执行命令过程中的可执行一些脚本做辅助性的工作。

> 脚本可执行的条件要满足以下条件：

1. 脚本可包含PHP回调函数和命令行可执行命令。

2. 包含已定义回调的 PHP 类和命令必须可通过 Composer 的自动加载功能自动加载。

3. 回调只能从 psr-0、psr-4 和 classmap 定义自动加载类。如果定义的回调依赖于类外部定义的函数，则回调本身负责加载包含这些函数的文件。


> 脚本是需要 `composer` 预定义的事件来触发的，常见的事件有：

1. `pre-install-cmd`：如果 `composer.lock` 文件存在，则在执行 `install` 命令之前被触发。

2. `post-install-cmd`：如果 `composer.lock` 文件存在，则在执行 `install` 命令之后被触发。

3. `pre-update-cmd`：在执行 `update` 命令之前触发，或者当 `composer.lock` 文件不存在时，执行了 `install` 命令也会触发。

4. `post-update-cmd`：在执行 `update` 命令之后触发，或者当 `composer.lock` 文件不存在时，执行了 `install` 命令也会触发。

5. `pre-autoload-dump`：当执行了 `install/update` 期间，或者执行了 `dump-autoload` 命令，在自动加载重新写入配置文件之前触发。

6. `post-autoload-dump`：当执行了 `install/update` 期间，或者执行了 `dump-autoload` 命令，在自动加载重新写入配置文件之后触发。

7. `post-root-package-install`：在执行了 `create-project` 时，在根包已经安装完成后且正常的依赖没有安装之前时触发。

8. `post-create-project-cmd`：在执行了 `create-project` 命令后触发。

参考学习 `laravel` 使用 `scripts` 的示例：

```json
    "scripts": {
        "post-autoload-dump": [
            "Illuminate\\Foundation\\ComposerScripts::postAutoloadDump",
            "@php artisan package:discover --ansi"
        ],
        "post-update-cmd": [
            // 发布静态资源
            "@php artisan vendor:publish --tag=laravel-assets --ansi --force"
        ],
        "post-root-package-install": [
            // 在执行了 create-project 后判断有没有.env文件，没有则从 .env.example 复制一份为 .env文件
            "@php -r \"file_exists('.env') || copy('.env.example', '.env');\""
        ],
        "post-create-project-cmd": [
            "@php artisan key:generate --ansi",
            // 生成数据库配置文件并执行迁移命令
            "@php -r \"file_exists('database/database.sqlite') || touch('database/database.sqlite');\"",
            "@php artisan migrate --graceful --ansi"
        ]
    }
```

## 八、版本约束解析

### 指定具体的版本号

`composer` 会按照执行的版本进行安装。

示例：

```json
{
    "require": {
        "monolog/monolog": "3.6.0"
    }
}
```

### 普通指定版本的范围

有效的操作符为：`>，>=, <, <=, !=`

不同的操作符可以联合在一起使用，如果是逻辑与的关系，使用空格或逗号(`,`)隔开，如果是逻辑或的关系，使用 `||` 隔开，逻辑与的优先级要高于逻辑或。

示例：

```json
>=1.0

>=1.0 <2.0

>=1.0,<1.1 || >=1.2
```

### 带连字符的版本范围：-

* 如果是 `X.Y` 的格式，即只有一位小数点，左闭右开。

示例：`1.0 - 2.0`，2.0 等于 2.0.*，即2.0后面可以为任意数字，即小于2.1，合并一起表示：`>=1.0.0 < 2.1`

* 如果是 `X.Y.Z` 的格式，即有两位小数点，左右皆闭合。

示例：`1.0.0 - 2.1.0`，表示：`>=1.0.0 <=2.1.0`

### 带通配符的版本范围：*

左闭右开

上面 2.0.* 就用到了通配符，即为：`>=2.0 < 2.1`

### 波浪线版本范围：~

* 如果是 `X.Y` 的格式，即只有一位小数点，最大兼容版本可上升到主版本(即整数部分)+1以下，左闭右开。

示例：`~1.2` 表示：`>=1.2 <2.0.0`，因为 `1.2` 只有一位小数，即可小于 `2.0.0`，此处的1表示主版本。

* 如果是 `X.Y.Z` 的格式，即有两位小数，最大兼容版本只能上升到次要版本(即第一个小数)+1以下，左闭右开。

示例：`~1.2.3` 表示：`>=1.2.3 <1.3.0`

### 插入符号版本范围：^

* 如果整数部分和所有小数部分都大于0，则最大兼容版本可上升到主版本(即整数部分)+1以下，左闭右开。 

示例：`^1.2.3` 表示：`>=1.2.3 <2.0.0`，主版本是1，所以可以小于 `2.0.0`

* 如果整数部分等于0，且第一个小数部分大于0，则最大兼容版本可上升到次要版本(即第一个小数)+1以下，左闭右开。

示例：`^0.3` 表示：`>=0.3.0 <0.4.0`，次要版本是第一个小数为3，所以可以小于 `0.4.0`

* 如果整数部分和第一个小数都等于0，则最大兼容版本可上升到第二个小数（即最后一个小数）+1以下，左闭右开。

示例：`^0.0.3` 表示：`>=0.0.3 <0.0.4`

### 版本约束后缀标志解析

例如，后缀带有：`-stable, -dev`。

明确指定后缀的，则应用后缀的标志。

示例：

* `1.2.3`：明确指定了版本，则内部表示为：`1.2.3-stable`

* `>1.2`：大于版本的范围，则内部表示为：`>1.2.0-stable`

* `>=1.2`：大于等于版本的范围，则内部表示为：`>=1.2.0-dev`

* `>=1.2-stable`：明确指定了 `-stable` 后缀的范围，则内部表示为：`>=1.2.0-stable`

* `<1.3`：小于版本的范围，则内部表示为：`<1.3.0-dev`

* `<=1.3`：小于等于版本的范围，则内部表示为：`<=1.3.0-stable`

* `1.0 - 2.0`：连字符指定1到2的范围，则内部表示为：`>=1.0.0-dev < 2.1.0-dev`

* `~1.3`：用波浪线操作符指定的范围，则内部表示为：`>=1.3.0-dev < 2.0.0-dev`

* `^1.2.3`：用插入符号指定的范围，则内部表示为：`>=1.2.3-dev <2.0.0-dev`

* `1.4.*`：用通配符指定的范围，则内部表示为：`>=1.4.0-dev <1.5.0-dev`


## 九、怎么验证依赖包所需的版本？

有一个用爱发电的 [Packagist Semver Checker](https://semver.madewithlove.com/) 的网站提供了版本检查验证功能，只要提供依赖包版本，它会自动标识出有哪些版本的包可用。

如图，提供依赖：`madewithlove/htaccess-cli: ^1.3.0`，则最大版本不超过2.0。

![alt text](/images/composer-image-21.png)

## 十、如何发布包到Packagist上？

1. 新建一个项目，或者已有项目，包含 `composer.json` 文件，包含至少以下配置项：

```json
{
    // 包名必须存在
    "name": "your-vendor-name/package-name",

    // 描述信息必须存在
    "description": "A short description of what your package does",

    // 依赖项必须存在
    "require": {
        "php": ">=8.2",
        "another-vendor/package": "1.*"
    }
}
```

2. 验证 `composer.json` 文件的语法格式是否正确

```shell
composer validate
```

3. 提交代码到 `github` 或其他版本控制系统仓库。

4. 进入 `packagist` 官方网站，创建账号并登录，点击 `Submit` 按钮。

![alt text](/images/composer-image-22.png)

5. 在提交页面填入仓库地址，系统会定期自动抓取包。

![alt text](/images/composer-image-23.png)

## 十一、资源地址

* [composer官方网站](https://getcomposer.org/)

* [composer命令文档](https://getcomposer.org/doc/03-cli.md)

* [composer.json文件结构解析文档](https://getcomposer.org/doc/04-schema.md)

* [composer版本约束解析文档](https://getcomposer.org/doc/articles/versions.md)

* [packagist官方网站](https://packagist.org/)

* [composer源码地址](https://github.com/composer/composer)

* [Packagist Semver Checker源码地址](https://github.com/madewithlove/semver)
