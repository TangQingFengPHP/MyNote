### 简介

`nslookup` 和 `dig` （Domain Information Groper）命令用于查询 `DNS` （Domain Name System）服务器，获取域名、IP地址等DNS记录信息。

### `nslookup` 命令

`nslookup`（名称服务器查找）命令是用于查询 `DNS` 记录的旧工具。虽然仍然可用，但已弃用，取而代之的是 `dig`。

#### 基础用法

```shell
nslookup [domain-name]
```

**示例用法**

```shell
nslookup google.com
```

**示例输出**

```shell
Server:		192.168.1.1
Address:	192.168.1.1#53

Non-authoritative answer:
Name: google.com
Address: 142.250.183.110
```

#### 查询指定DNS服务器

使用自定义 DNS 服务器而不是系统默认的 DNS 服务器

```shell
nslookup google.com 8.8.8.8
```

#### 查询 MX（邮件交换 Mail Exchange）记录

```shell
nslookup -query=MX google.com
```

#### 反向 DNS 查找（从 IP 获取域名）

```shell
nslookup 142.250.183.110
```

### `dig` 命令

`dig` 命令是一个更强大、更灵活的查询DNS信息的工具

#### 基础用法

```shell
dig [options] [domain-name] [record-type]
```

#### 从域名中查询 IP 地址

```shell
dig google.com
```

**示例输出**

```shell
;; ANSWER SECTION:
google.com.		299	IN	A	142.250.183.110

# google.com 的 IP 地址（A 记录）是 142.250.183.110
```

#### 查询特定 DNS 服务器

```shell
dig @8.8.8.8 google.com
```

#### 查询 MX（邮件交换）记录

```shell
dig google.com MX
```

**示例输出**

```shell
google.com.		3599	IN	MX	50 alt4.aspmx.l.google.com.
google.com.		3599	IN	MX	40 alt3.aspmx.l.google.com.
```

#### 反向查询 DNS

```shell
dig -x 142.250.183.110
```

#### 仅获取简短的答复

```shell
dig +short google.com
```

#### 查询所有 DNS 记录

```shell
dig google.com ANY
```

### nslookup 与 dig 的区别

|  `特点`  |  `nslookup`   |  `dig`   |
| --- | --- | --- |
| 输出   |  更简单，但细节较少   |  详细输出   |
|  支持查询特定 DNS 服务器   |  支持   |  支持   |
|  反向查找   |  支持    |  支持   |
|  支持现代功能   |  已弃用    |  推荐使用   |