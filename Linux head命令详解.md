### 简介

默认将每个文件的前10行打印到标准输出

#### 一般用法

```shell
head abc.txt
```

#### 指定打印显示前几行

> `-n`

```shell
head -n 5 abc.txt
```

#### 指定显示前几个字节

> `-c`或`--bytes`

```shell
head -c 5 abc.txt
```

#### 同时打印多个文件

```shell
head abc.txt def.txt
显示的内容会以==> [文件名1] <==格式分开
```