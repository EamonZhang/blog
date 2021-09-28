---
title: "MySQL常用性能分析命令"
date: 2019-01-29T14:06:55+08:00
draft: false
toc: false
categories: ["mysql"]
tags: []
---

## MySQL常用性能突发事件分析命令：

1. SHOW PROCESSLIST; —当前MySQL数据库的运行的所有线程
 
2. INNODB_TRX; — 当前运行的所有事务

3. INNODB_LOCKS; — 当前出现的锁

4. INNODB_LOCK_WAITS; — 锁等待的对应关系计
 
5. SHOW OPEN TABLES where In_use >0; — 当前打开表

6. SHOW ENGINE INNODB STATUS  \G; —Innodb状态

7. SHOW STATUS LIKE  'innodb_row_lock_%'; — 锁性能状态

8. SQL语句EXPLAIN; — 查询优化器

## 数据库size查看

```
-- databases size
select table_schema as '数据库', sum(table_rows) as '记录数', sum(truncate(data_length/1024/1024, 2)) as '数据容量(MB)', sum(truncate(index_length/1024/1024, 2)) as '索引容量(MB)' from information_schema.tables group by table_schema order by sum(data_length) desc, sum(index_length) desc;
```

```
-- table size
select table_schema as '数据库', table_name as '表名', table_rows as '记录数', truncate(data_length/1024/1024, 2) as '数据容量(MB)', truncate(index_length/1024/1024, 2) as '索引容量(MB)' from information_schema.tables order by data_length desc, index_length desc;
```
