### 简介

将原文件复制到目标位置，可以一次性复制多个文件

#### 一般用法

```shell
cp abc.txt a/
```

#### 复制多个文件

```shell
cp abc.txt def.txt a/
```

#### 递归复制

```shell
cp -r abc/ a/
```

#### 复制到指定目录，并改名

```shell
cp abc.txt abc/def.txt
```