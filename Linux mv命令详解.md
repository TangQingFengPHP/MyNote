### 简介

移动文件到目录、重命名文件名

#### 一般用法

```shell
mv abc.txt def/
```

#### 移动多个文件

```shell
mv dif abc.txt /def
```

#### 如果文件已经存在，强制覆盖不提示

> `-f`或`--force`

```shell
mv -f abc.txt def/
```

#### 移动前给出交互式提示询问

> `-i`或`--interactive`

```shell
mv -i abc.txt def/
```

#### 目标文件存在则不覆盖

> `-n`或`--no-clobber`

```shell
mv -n abc.txt def/
```

#### 重命名文件

```shell
mv abc.txt def.txt
```

#### 重命名目录

```shell
mv abc/ def/
```

#### 打印调试信息

> `-v`或`--verbose`

```shell
mv -v abc.txt def/
```

#### 查看帮助信息

> `--help`

```shell
mv --help
```

#### 查看版本信息

> `--version`

```shell
mv --version
```