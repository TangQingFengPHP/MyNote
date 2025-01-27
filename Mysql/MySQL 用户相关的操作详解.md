### MySQL 5.x 用户操作

#### 创建用户

> 在 `MySQL` 5.x 中，使用 `GRANT` 语句创建用户并授权：

**语法**

```sql
GRANT ALL PRIVILEGES ON *.* TO 'username'@'host' IDENTIFIED BY 'password';
```

* `username`：用户名

* `host`：指定用户可访问的主机，例如 `localhost` 或 `%`（表示任何主机）

* `password`：用户的密码

**示例**

```sql
GRANT ALL PRIVILEGES ON *.* TO 'testuser'@'localhost' IDENTIFIED BY 'testpassword';
```

#### 授权

> 再次使用 `GRANT` 语句即可修改用户权限：

```sql
GRANT SELECT, INSERT ON database_name.* TO 'username'@'host';
```

#### 刷新权限

> 需要刷新权限以生效：

```sql
FLUSH PRIVILEGES;
```

#### 修改密码

> 修改用户密码使用 `SET PASSWORD` 或 `GRANT` 语句：

```sql
SET PASSWORD FOR 'username'@'host' = PASSWORD('newpassword');
```

**示例**

```sql
SET PASSWORD FOR 'testuser'@'localhost' = PASSWORD('newpassword');
```

#### 删除用户

> 使用 `DROP USER` 删除用户：

```sql
DROP USER 'username'@'host';
```

**示例**

```sql
DROP USER 'testuser'@'localhost';
```

### MySQL 8.x 用户操作

#### 创建用户

> 在 `MySQL 8.x` 中，推荐使用 `CREATE USER`：

```sql
CREATE USER 'username'@'host' IDENTIFIED BY 'password';
```

**示例**

```sql
CREATE USER 'testuser'@'localhost' IDENTIFIED BY 'testpassword';
```

#### 授权

> 授权使用 `GRANT` 语句：
>
> `MySQL 8.x` 默认不需要手动执行 `FLUSH PRIVILEGES`，授权会立即生效。

```sql
GRANT SELECT, INSERT ON database_name.* TO 'username'@'host';
```

**示例**

```sql
GRANT ALL PRIVILEGES ON *.* TO 'testuser'@'localhost';
```

#### 修改密码

> 修改密码使用 `ALTER USER` 语句：

```sql
ALTER USER 'username'@'host' IDENTIFIED BY 'newpassword';
```

**示例**

```sql
ALTER USER 'testuser'@'localhost' IDENTIFIED BY 'newpassword';
```

#### 删除用户

> 删除用户同样使用 `DROP USER`：

```sql
DROP USER 'username'@'host';
```

**示例**

```sql
DROP USER 'testuser'@'localhost';
```

#### 重命名用户

> `MySQL 8.x` 支持直接重命名用户：

```sql
RENAME USER 'old_username'@'host' TO 'new_username'@'host';
```

**示例**

```sql
RENAME USER 'testuser'@'localhost' TO 'newuser'@'localhost';
```

### 通用操作

#### 查看用户

> 查看所有用户：

```sql
SELECT User, Host FROM mysql.user;
```

#### 查看用户权限

> 查看特定用户的权限：

```sql
SHOW GRANTS FOR 'username'@'host';
```

**示例**

```sql
SHOW GRANTS FOR 'testuser'@'localhost';
```

#### 撤销权限

> 使用 `REVOKE` 语句撤销权限：

```sql
REVOKE privilege_type ON database_name.* FROM 'username'@'host';
```

**示例**

```sql
REVOKE SELECT ON testdb.* FROM 'testuser'@'localhost';
```

### `PRIVILEGES` 权限

> 在 `MySQL` 中，`PRIVILEGES` 是指权限，用于控制用户可以对数据库执行哪些操作，分为全局权限和基于数据库、表或列的权限。

#### 全局权限

> 全局权限适用于整个 `MySQL` 服务器，通常授予 `*.*`（所有数据库和表）

**权限列表及说明：**

* `ALL PRIVILEGES`：授予用户所有权限（不包括 `GRANT OPTION`）

* `GRANT OPTION`：允许用户将其拥有的权限授予其他用户

* `CREATE USER`：允许用户创建、删除、修改其他用户

* `FILE`：允许用户读取和写入服务器上的文件

* `PROCESS`：允许用户查看其他用户的查询或线程（通过 `SHOW PROCESSLIST`）

* `RELOAD`：允许用户执行刷新操作，例如 `FLUSH` 命令

* `SHOW DATABASES`：允许用户查看所有数据库（非仅限其拥有权限的数据库）

* `SHUTDOWN`：允许用户关闭 `MySQL` 服务器

* `SUPER`：允许用户执行高级管理操作，例如修改全局变量、杀死线程等

* `REPLICATION SLAVE`：允许用户配置和管理 `MySQL` 复制从库

* `REPLICATION CLIENT`：允许用户查看主库和从库的状态

#### 数据库级权限

> 数据库级权限适用于单个数据库的所有表

**权限列表及说明：**

* `CREATE`：允许用户创建新数据库或表

* `DROP`：允许用户删除数据库或表

* `EVENT`：允许用户创建、修改和删除事件

* `INDEX`：允许用户在表中创建或删除索引

* `ALTER`：允许用户修改表结构，例如添加或删除列

* `SHOW VIEW`：允许用户查看视图定义

* `TRIGGER`：允许用户创建或删除触发器

#### 表级权限

> 表级权限适用于某个数据库中的特定表

**权限列表及说明：**

* `SELECT`：允许用户读取表中的数据（查询）

* `INSERT`：允许用户向表中插入数据

* `UPDATE`：允许用户更新表中的数据

* `DELETE`：允许用户删除表中的数据

* `REFERENCES`：允许用户创建外键约束

* `LOCK TABLES`：允许用户锁定表，通常与事务处理相关

#### 列级权限

> 列级权限是表级权限的进一步细化，适用于特定列

**权限列表及说明：**

* `SELECT`：允许用户查询特定列的数据

* `INSERT`：允许用户向特定列插入数据

* `UPDATE`：允许用户更新特定列的数据

**示例**

```sql
GRANT SELECT (column1, column2) ON mydb.mytable TO 'username'@'host';
```

#### 存储程序权限

> 这些权限适用于存储过程和存储函数

**权限列表及说明：**

* `CREATE ROUTINE`：允许用户创建存储过程和存储函数

* `ALTER ROUTINE`：允许用户修改存储过程和存储函数

* `EXECUTE`：允许用户执行存储过程和存储函数

#### 动态权限（`MySQL 8.x` 新增）

> 动态权限适用于特定场景，例如管理数据复制、安全性和审计功能

**权限列表及说明：**

* `BACKUP_ADMIN`：允许用户进行备份操作

* `GROUP_REPLICATION_ADMIN`：允许用户管理组复制设置

* `RESOURCE_GROUP_ADMIN`：允许用户管理资源组

* `RESOURCE_GROUP_USER`：允许用户使用资源组

* `SYSTEM_VARIABLES_ADMIN`：允许用户修改系统变量

* `PERSIST_RO_VARIABLES_ADMIN`：允许用户管理持久化只读变量

#### 查看权限

* 查看当前用户的权限

```sql
SHOW GRANTS;
```

* 查看其他用户的权限

```sql
SHOW GRANTS FOR 'username'@'host';
```