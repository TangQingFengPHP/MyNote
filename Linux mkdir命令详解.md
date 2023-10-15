### 简介

创建不存在的目录

#### 一般用法

```shell
mkdir abc
```

#### 递归创建目录

> `-p`或`--parents`

```shell
mkdir -p abc/def
```

#### 创建目录并添加权限

> `-m`或`--mode`

```shell
mkdir -m 744 abc/
此选项在使用-p时不起作用
```