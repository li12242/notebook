---
layout: post
title: MySQL服务mariadb安装与配置
date: 2022-2-7
categories: tools
---

# MySQL 环境配置

## mariadb 简介

## mariadb 部署

以 Ubuntu 为例，可以直接使用 apt-get 命令安装 mariadb 服务

```bash
apt-get install mariadb-server
```

安装完成后，直接运行如下命令启动 mysql 服务

```
/etc/init.d/mysql start
```

## MySQL 登录设置

安装完成 MySQL 服务后，直接在 python 环境中通过 pymysql 操作数据库，会出现如下错误。

```python
pymysql.err.InternalError: (1698, "Access denied for user 'root'@'localhost'")
```

报错原因是由于未设置root账户对应密码。
为了能够直接使用 pymysql 模块操作 MySQL 数据库，登录 mysql 界面后，选择 mysql 数据库。

```
mysql -u root

MariaDB [(none)]> use mysql;
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
```

直接使用如下命令修改 root 账户密码，注意使用 `FLUSH PRIVILEGES;` 命令更新表格信息。

```
MariaDB [mysql]> update user set Password=password("111111") where User='root';
Query OK, 0 rows affected (0.00 sec)
Rows matched: 1  Changed: 0  Warnings: 0

MariaDB [mysql]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)
```

此时，对 plugin 字段进行查看，当对应值为 `unix_socket` 时，仍会出现登录问题。

```
MariaDB [(none)]> select user, plugin from mysql.user;
+------+-------------+
| user | plugin      |
+------+-------------+
| root | unix_socket |
+------+-------------+
1 row in set (0.00 sec)
```

将此字段使用如下命令修改为 `mysql_native_password`，此时再次使用 pymysql 即可正常操作 MySQL 数据库。

```
MariaDB [mysql]> update user set plugin='mysql_native_password' where User='root';
Query OK, 1 row affected (0.000 sec)
Rows matched: 1  Changed: 1  Warnings: 0

MariaDB [mysql]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.00 sec)
```
