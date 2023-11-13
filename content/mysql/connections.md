---
title: "连接数管理"
date: 2021-09-24T17:18:25+08:00
draft: false
toc: false
categories: ["mysql"]
tags: []
---

## 连接池
```
show variables like "thread%";
+-------------------+---------------------------+
| Variable_name     | Value                     |
+-------------------+---------------------------+
| thread_cache_size | 9                         |
| thread_handling   | one-thread-per-connection |
| thread_stack      | 286720                    |
+-------------------+---------------------------+
3 rows in set (0.01 sec)

```


## 全局设置

```
-- 查看全局连接
 show variables like "%connections%";
+------------------------+-------+
| Variable_name          | Value |
+------------------------+-------+
| max_connections        | 151   |
| max_user_connections   | 0     |
| mysqlx_max_connections | 100   |
+------------------------+-------+
```

```
-- 超级用户连接
show variables like "admin_%";
+------------------------+-------------------------------+
| Variable_name          | Value                         |
+------------------------+-------------------------------+
| admin_address          | 127.0.0.1                     |
| admin_port             | 33062                         |

```

```
-- 运行状态
show status like "thread%";
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| Threads_cached    | 0     |
| Threads_connected | 1     |
| Threads_created   | 1     |
| Threads_running   | 2     |
+-------------------+-------+
```


## user 级别设置

```
[mysqld]
datadir=/var/lib/mysql
socket=/var/lib/mysql/mysql.sock
user=mysql
max_connections = 10000
max_user_connections=2000

2、通过其它方式处理一下，针对指定用户设置的，比如：
方式1
GRANT USAGE ON *.* TO test_user@localhost MAX_USER_CONNECTIONS 2000;
方式2
UPDATE mysql.user SET max_user_connections = 2000 WHERE user='test_user' AND host='localhost'; FLUSH PRIVILEGES

```
