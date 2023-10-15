### 简介

删除文件或目录

#### 一般用法

```shell
rm abc.txt
```

#### 删除目录

> `-r`或`-R`或`--recursive`

```shell
rm -r abc/
```

#### 只删除空目录

> `-d`或`--dir`

```shell
rm -d abc/
```

#### 在每次删除前提示

> `-i`

```shell
rm -i abc/
```

#### 当删除的文件数超过三个或者是递归删除目录时给出提示

> `-I`

```shell
rm -I abc/
```