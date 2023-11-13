---
title: "联想搜索"
date: 2021-11-16T10:30:23+08:00
draft: false
toc: false
categories: ["python","redis"]
tags: []
---

## Redis 联想搜索实现

- 基于redis ZSET 

例子: 当输入`n`时显示所有n开头的数据，当输入`nb`时显示所有nb开头的数据。


```
# 数据录入
127.0.0.1:6379> ZADD ss 0 'n'
(integer) 1
127.0.0.1:6379> ZADD ss 0 'nb'
(integer) 1
127.0.0.1:6379> ZADD ss 0 'nba'
(integer) 1
# 搜索n
127.0.0.1:6379> ZRANK ss 'n'
(integer) 0
127.0.0.1:6379> ZRANGE ss 0 -1
1) "n"
2) "nb"
3) "nba"
# 搜索nb
127.0.0.1:6379> ZRANK ss 'nb'
(integer) 1
127.0.0.1:6379> ZRANGE ss 1 -1
1) "nb"
2) "nba"
```
