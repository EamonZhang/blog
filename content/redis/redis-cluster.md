---
title: "Redis Cluster"
date: 2021-03-09T17:00:05+08:00
draft: false 
toc: true
categories: ['redis']
tags: []
---

## 主从复制

原理说明参考 https://www.cnblogs.com/daofaziran/p/10978628.html

#### 基本原理

主要是Redis没有wal日志机制。aof是先执行命令再记录。主从同步不是依赖aof日志。

通过单独的进程完成主从的同步。

初始时全量复制，SYNC。 然后进行增量复制PSYNC。在主库内存中维护一个偏移量`master run id` 。 当断开重新连接上比较偏移量，尝试增量同步，如果增量失败进行全量同步。

#### 从库只读

Redis 从库支持写操作。 容易产生冲突,可通过配置进行设置

```
replica-read-only yes
```

#### 无盘复制

当新添加从节点的时候，会在主库先拷贝一份全量数据放在本地磁盘，如果主库磁盘有限制或IO增大可能对业务造成影响。可选择无盘复制，主从将直接通过socket通信。

```
repl-diskless-sync

repl-diskless-sync-delay
```

#### 多个从库同步成功后主可写

为了增加整体的数据一致性。可设定N个从库同步成功后主库可写功能。

```
min-slaves-to-write（最小从服务器数）
min-slaves-max-lag（从服务器最大确认延迟）
```

#### 级联主从

Redis 支持级联复制模式。从库的复制对主库会有一定的性能影响。

为了消除或减少复制时对主库的影响。可采取级联复制

#### 主从配置

配置文件中加入，或执行命令
```
slaveof  # 旧版本命令
replicaof # 新版本命令
```

#### 状态查看

查看命令
```
INFO replication
```


## Redis Sentinel 

Sentinel 可以帮助我们自动的管理Redis 的主从复制。能够做到

- Monitoring
- Notification
- Automatic failover
- Configuration provider


#### 一个简单配置

一个Sentinel 集群可以管理多个 Redis 主从。 通过名称 monitor mymaster 来区分。

```
port 5000
sentinel monitor mymaster 127.0.0.1 6379 2 # 2 表示有2个或以上sentinel 连接不到redis时认为redis挂掉。
sentinel down-after-milliseconds mymaster 60000 # 连接中断时间大于 xx 认为超时。
sentinel failover-timeout mymaster 180000 # failover过期时间。当failover开始后，在此时间内仍然没有触发任何failover操作，当前sentinel将会认为此次failoer失败。
sentinel parallel-syncs mymaster 1 # 当发生failover时 统一时刻可以有几个从库从主库同步数据

sentinel monitor resque 192.168.1.3 6380 3
sentinel down-after-milliseconds resque 10000
sentinel failover-timeout resque 180000
sentinel parallel-syncs resque 5
```

#### Sentinel 通信

Sentinel 在配置的时候只指定了master Redis的地址。多个Sentinel如何是如何形成集群的呢？

Sentinel 通过Redis的pub/sub 广播自动发现，形成网络拓扑。


#### 运行时管理命令

redis-cli -p 5000
```
SENTINEL CONFIG GET 
SENTINEL CONFIG SET 
SENTINEL CKQUORUM <master name>
SENTINEL FLUSHCONFIG 
```

#### python 连接Sentinel

```
# import redis
from redis.sentinel import Sentinel
 
"""
1、通过访问Sentinel服务的方式，获取redis的master、slave节点信息
2、向master redis写入数据
3、从slave redis读取数据
"""
 
# 连接哨兵服务器(主机名也可以用域名)
sentinel = Sentinel([('192.168.196.129', 26379),
                     ('192.168.196.132', 26379)
             ],
                    socket_timeout=0.5)
 
# 获取主服务器地址
master = sentinel.discover_master('mymaster')
print(master)
# 输出：('192.168.196.132', 6379)
 
 
# 获取从服务器地址
slave = sentinel.discover_slaves('mymaster')
print(slave)
# 输出：[('192.168.196.129', 6379)]
 
 
# 获取主服务器进行写入
master = sentinel.master_for('mymaster', socket_timeout=0.5, password='newpwd', db=0)
w_ret = master.set('foo', 'bar')
# 输出：True
 
# 获取从服务器进行读取（默认是round-roubin,随机从多个slave服务中读取数据）
slave = sentinel.slave_for('mymaster', socket_timeout=0.5, password='newpwd', db=0)
r_ret = slave.get('foo')
print(r_ret)
# 输出：bar
```
