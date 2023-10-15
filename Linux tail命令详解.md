### 简介

查看文件最后几行数据，一般用于查看实时日志

#### 一般用法

```shell
tail abc.log

默认查看最后十行数据
```

#### 指定查看的行数

> `-n`或`--lines=[+]NUM`

```shell
tail -n 5 abc.log

tail -5 abc.log

以上两种方式等同
```

#### 指定查看从哪一行开始到结尾的行

```shell
tail -n +5 abc.log

表示从第5行开始一直到结尾
```

#### 指定字节数查看

> `-c`或`--bytes`

```shell
tail -c 100 abc.log
```

#### 根据文件描述符查看实时更新的日志

> `-f`或`--follow[={name|descriptor}]`

```shell
tail -f abc.log

以上--follow的默认值是descriptor，即只要文件改名或删除即停止跟踪
```

#### 根据文件名查看实时更新的日志

> `-F`或`--follow=name --retry`

```shell
tail -F abc.log

即该文件被删除或改名后，如果再次创建相同的文件名，会继续追踪
```

#### 指定监视文件变化时间的间隔秒数

> `-s`或`--sleep-interval`

```shell
tail -f -s 2 abc.log

默认是1秒刷新一次
```

#### 查看多个文件时，不打印文件名

> `-q`或`--quiet, --silent`

```shell
tail -q abc.txt def.txt
```

#### tail与head同时使用

```shell
head -10 abc.txt | tail -5

表示先取文件的前10行，再在10行里面取最后5行
```

