### 脚本目的：

`init` 脚本主要的作用是：从 `environments` 目录中复制配置文件，确保应用适配不同环境（例如开发、生产环境等）。

### 工作流程：

* 获取 `$_SERVER` 的 `argv` 参数 

* 加载 `environments/index.php` 文件，拿到不同环境配置指定的配置文件关系。

* 如果执行 `init` 脚本时提供了 `--env` 选项，例如：`--env=Development` 则直接应用此环境，否则会被提示需要选择一个环境来初始化。

* 获取 `environments` 对应环境下的所有文件。

* 因为上一步获取到的所有文件是带有文件全路径的，所以这一步直接复制文件到对应的路径，如：`frontend/config/params-local.php`。

* 从 `environments/index.php` 文件中获取到对应环境所配置的需要设置可写、可执行的目录来执行操作。

### 代码详解：

* 解析命令行参数：

![alt text](/images/Yii2-init-初始化脚本分析-image-1.png)

* 检查命令行参数：

![alt text](/images/Yii2-init-初始化脚本分析-image-2.png)

* 获取 `environments` 中的文件列表：

![alt text](/images/Yii2-init-初始化脚本分析-image-3.png)

* 复制文件：

```php
function copyFile($root, $source, $target, &$all, $params)
{
    // 检查源文件是否存在
    if (!is_file($root . '/' . $source)) {
        echo "       skip $target ($source not exist)\n";
        return true;
    }
    // 检查目标文件是否存在
    if (is_file($root . '/' . $target)) {
        if (file_get_contents($root . '/' . $source) === file_get_contents($root . '/' . $target)) {
            echo "  unchanged $target\n";
            return true;
        }
        // 如果$all为true，输出信息并直接进行覆盖。
        // 否则，提示用户目标文件已存在，并询问是否覆盖（选择“是”、“否”、“全部”或“退出”）。
        if ($all) {
            echo "  overwrite $target\n";
        } else {
            echo "      exist $target\n";
            echo "            ...overwrite? [Yes|No|All|Quit] ";


            // 通过命令行接收用户输入。如果$params['overwrite']不为空，使用该值；否则，等待用户输入。
            $answer = !empty($params['overwrite']) ? $params['overwrite'] : trim(fgets(STDIN));

            // 根据用户输入执行相应操作：
            // 如果输入“q”或“Q”，返回false以退出操作。
            // 如果输入“y”或“Y”，输出覆盖信息并继续。
            // 如果输入“a”或“A”，输出覆盖信息并设置$all为true，以便后续文件均自动覆盖。
            // 其他输入则跳过目标文件。
            if (!strncasecmp($answer, 'q', 1)) {
                return false;
            } else {
                if (!strncasecmp($answer, 'y', 1)) {
                    echo "  overwrite $target\n";
                } else {
                    if (!strncasecmp($answer, 'a', 1)) {
                        echo "  overwrite $target\n";
                        $all = true;
                    } else {
                        echo "       skip $target\n";
                        return true;
                    }
                }
            }
        }
        file_put_contents($root . '/' . $target, file_get_contents($root . '/' . $source));
        return true;
    }
    // 如果目标文件不存在，输出信息并进行复制。
    echo "   generate $target\n";
    // 使用@mkdir创建目标文件的目录（如果不存在），并设置目录权限为0777。
    @mkdir(dirname($root . '/' . $target), 0777, true);
    file_put_contents($root . '/' . $target, file_get_contents($root . '/' . $source));
    return true;
}
```

* 执行回调方法：

```php
$callbacks = ['setCookieValidationKey', 'setWritable', 'setExecutable', 'createSymlink'];
foreach ($callbacks as $callback) {
    if (!empty($env[$callback])) {
        $callback($root, $env[$callback]);
    }
}
// 读取 environments/index.php 文件的配置：
'Development' => [
    'path' => 'dev',
    'setWritable' => [ // runtime目录设置为可写
        'backend/runtime',
        'console/runtime',
        'frontend/runtime',
    ],
    'setExecutable' => [ // yii、yii_test文件设置为可执行
        'yii',
        'yii_test',
    ],
    'setCookieValidationKey' => [
        'backend/config/main-local.php',
        'common/config/codeception-local.php',
        'frontend/config/main-local.php',
    ],
],

// 执行具体的回调方法：
// 设置文件可写
function setWritable($root, $paths)
{
    foreach ($paths as $writable) {
        if (is_dir("$root/$writable")) {
            if (@chmod("$root/$writable", 0777)) {
                echo "      chmod 0777 $writable\n";
            } else {
                printError("Operation chmod not permitted for directory $writable.");
            }
        } else {
            printError("Directory $writable does not exist.");
        }
    }
}

// 设置文件可执行
function setExecutable($root, $paths)
{
    foreach ($paths as $executable) {
        if (file_exists("$root/$executable")) {
            if (@chmod("$root/$executable", 0755)) {
                echo "      chmod 0755 $executable\n";
            } else {
                printError("Operation chmod not permitted for $executable.");
            }
        } else {
            printError("$executable does not exist.");
        }
    }
}

function setCookieValidationKey($root, $paths)
{
    foreach ($paths as $file) {
        echo "   generate cookie validation key in $file\n";
        $file = $root . '/' . $file;
        $length = 32;
        $bytes = openssl_random_pseudo_bytes($length);
        $key = strtr(substr(base64_encode($bytes), 0, $length), '+/=', '_-.');
        $content = preg_replace('/(("|\')cookieValidationKey("|\')\s*=>\s*)(""|\'\')/', "\\1'$key'", file_get_contents($file));
        file_put_contents($file, $content);
    }
}
```
