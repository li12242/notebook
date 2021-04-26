---
layout: post
title: MySQL常用命令
date: 2021-4-23
categories: tools
---

# MySQL配置

使用yum安装MySQL相关服务，命令如下：

```bash
$ yum install mariadb # install mysql environment

$ yum install mariadb-server # install mysql service
```

安装完成后可以启动mysql服务

```bash
$ systemctl start mariadb
```

设置root账户密码


设置密码后使用密码登录mysql账户

```
$ mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 19
Server version: 5.5.56-MariaDB MariaDB Server

Copyright (c) 2000, 2017, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
```

# MySQL基本命令

MySQL服务器启动之后，可以采用软件或命令行与数据库交互，这里主要介绍命令行方法。

除了用户自定义的变量名，MySQL语句是不区分大小写的，但是习惯上把MySQL关键词都用大写，而小写字母表示数据表和数据库的名称。MySQL语句结束必须用分号

当首先进入MySQL时候，可能没有数据库存放数据，用下面命令创建

```mysql
>CREATE DATABASE testing;
```

因为MySQL中可以有多个数据库，所以使用某个数据库之前需要指定数据库名称：

```mysql
>USE testing;
Database changed
```

创建好数据库并使用后，可以在其中创建数据表。创建MySQL数据表时必须有一列（数据），因此使用 `CREATE TABLE` 命令时候，必须把字段的定义放进一个带括号的、内部由逗号分隔的表中：

```mysql
>CREATE TABLE pages (id BIGINT(7) NOT NULL AUTO_INCREMENT, title VARCHAR(200), content VARCHAR(10000), created TIMESTAMP DEFAULT CURRENT_TIMESTAMP, PRIMARY KEY(id));
```

每个字段的定义由三部分组成：

* 名称（id, title, created）
* 数据类型（BIGINT(7)，VARCHAR，TIMESTAMP）
* 其他属性（NOT NULL AUTO_INCREMENT）

在字段定义列表的最后，还定义了一个主键（key），主要用于组织表的内容，便于后面快速查询。
语句执行后，可以查看数据表是否已经建立：

```mysql
>SHOW TABLES;
+-------------------+
| Tables_in_testing |
+-------------------+
| pages             |
+-------------------+
1 row in set (0.00 sec)
```

随后可以用DESCRIBE关键字查看数据表结构：

```mysql
>DESCRIBE pages;
+---------+----------------+------+-----+-------------------+----------------+
| Field   | Type           | Null | Key | Default           | Extra          |
+---------+----------------+------+-----+-------------------+----------------+
| id      | bigint(7)      | NO   | PRI | NULL              | auto_increment |
| title   | varchar(200)   | YES  |     | NULL              |                |
| content | varchar(10000) | YES  |     | NULL              |                |
| created | timestamp      | NO   |     | CURRENT_TIMESTAMP |                |
+---------+----------------+------+-----+-------------------+----------------+
4 rows in set (0.00 sec)
```

pages表内虽然有4个数据列，但是实际只需要两个数据即可（`title` 和 `content`）。因为 `id` 是自动递增的，而`created` 数据类型是 `timestamp`，默认就是数据加入的时间戳。
此时这个表内容仍是空的，可以用下面命令插入一些测试数据：

```mysql
>INSERT INTO pages (title, content) VALUES ("Test page title", "This is ome test page content. It can be up to 10000 characters long.");
```

此时表内有数据了，可以用下面语句选择数据：

```mysql
>SELECT * FROM pages WHERE id = 1;
+----+-----------------+-----------------------------------------------------------------------+---------------------+
| id | title           | content                                                               | created             |
+----+-----------------+-----------------------------------------------------------------------+---------------------+
|  1 | Test page title | This is ome test page content. It can be up to 10000 characters long. | 2021-04-24 21:16:37 |
+----+-----------------+-----------------------------------------------------------------------+---------------------+
1 row in set (0.00 sec)
``` 

其中 `WHERE` 限制了选择范围，当缺少此范围时，会显示表中所有内容。删除数据的话使用DELETE语句：

```mysql
>DELETE FROM pages WHERE id = 1;
```

注意，由于数据删除后不能恢复，所以在执行DELETE语句之前，建议用 `SELECT` 确认要删除的数据，然后把 `SELECT *` 换成 `DELETE` 就可以了。

尽管将 `id = 1` 的数据删除了，但是下次插入数据时 `id` 仍会递增，此时再次插入数据表，查询表内容如下所示：

```mysql
> INSERT INTO pages (title, content) VALUES ("Test page title", "This is ome test page content.");
Query OK, 1 row affected (0.00 sec)

> SELECT * FROM pages WHERE id = 2;
+----+-----------------+--------------------------------+---------------------+
| id | title           | content                        | created             |
+----+-----------------+--------------------------------+---------------------+
|  2 | Test page title | This is ome test page content. | 2021-04-24 21:20:29 |
+----+-----------------+--------------------------------+---------------------+
1 row in set (0.00 sec)
```

# Python 操作 MySQL

Python 没有内置的 MySQL 支持工具，但是可以使用很多开源库与 MySQL 进行交互，这里推荐使用的就是pymysql。
可以使用conda或者pip命令安装：

```
$ conda install pymysql
```

安装完成后就可以使用pymysql包与MySQL进行交互了。启动MySQL服务后，可以将下面命令保存为文件 `mysql_test.py` 执行：

```python
import pymysql
conn = pymysql.connect(user='root', passwd='111111')

cur = conn.cursor()
cur.execute('USE testing')
cur.execute("SELECT * FROM pages")
print(cur.fetchone())

cur.close()
conn.close()
```

执行后会显示 `pages` 数据库内所有数据表。

```bash
$ python mysql_test.py
(3, 'Test page title', 'This is ome test page content.', datetime.datetime(2021, 4, 24, 21, 47, 19))
```

在上述python代码中有两个对象：连接对象（conn）和光标对象（cur）。
连接/光标模式是数据库编程的常见模式，链接模式除了要链接数据库之外，还有发送数据库信息，处理回滚操作，创建新的光标对象等。
一个连接可以有多个光标，一个光标跟踪一种状态信息。当使用多个数据库，并且需要向所有数据库写内容时，就需要多个光标进行处理。光标还会包含最后一次查询结果，通过函数 `fetchone()` 可以查看。
在使用完光标和连接之后，需要使用 `close()` 函数将其关闭，否则会导致连接泄露（connection leak），造成连接不可用但是却仍耗费数据库资源。
