---
title: "长轮询"
date: 2021-11-16T13:56:39+08:00
draft: false
toc: false
categories: ["python","redis"]
tags: []
---

## 长轮询

实现原理： 与传统的轮询方式不同的是，当服务端接收到客户端的请求的时候，如果没有最新消息时不是立刻返回请求，而是等待一个最大超时时间。如果等待期间有最新消息则立刻返回。

相对于暴力的轮询，长轮询能够很大程度的减少客户端与服务端的连接进而缓解服务端的压力。

利用长轮询模拟推送服务，还可以得到实时的消息。避免轮询的时间间隔带来的延迟。

优点： 缓解服务端压力，消息实时。

## 利用redis 堵塞队列实现长轮询


#### 服务端发布消息,

由于消息具有实时性。并且避免如果客户端没有消费队列中的消息造成队列积压。在向队列中添加消息的同时更新队列的过期时间
```
127.0.0.1:6379> RPUSH channel-queue 1
(integer) 1
127.0.0.1:6379> EXPIRE channel-queue 100
(integer) 1
127.0.0.1:6379> RPUSH channel-queue 3
(integer) 1
127.0.0.1:6379> EXPIRE channel-queue 100
(integer) 1

```

#### 消费端获取消息

采用堵塞的方式读取队列中的消息，主要是考虑队列为空的情况。不是立刻返回、而是等待一个最大超时时间。

```
BLPOP channel-queue 10
1) "channel-queue"
2) "3"
(5.10s)
```

## python 实现

```
#!/usr/bin/env python

import redis
import time
import random

redis_pool = redis.ConnectionPool(host="192.168.6.15",port="6379")
redis_client = redis.Redis(connection_pool=redis_pool)

#模拟向客户端发送消息
def sendMessages():
    for i in range(10):
        redis_client.rpush("channel-queue",i)
        redis_client.expire("channel-queue",100)
        time.sleep(random.randint(5,60))
        print("send message : %d at %s" % (i,time.asctime(time.localtime(time.time()))))


# 模拟客户端轮询获取消息
def readMessage():
    message = redis_client.blpop("channel-queue",10)
    if message is None:
        print("get nothing...")
    else:
        print("receive message : %s at %s " % (message[1],time.asctime(time.localtime(time.time()))))
    readMessage()

if __name__ == '__main__':
    # sendMessages()
    readMessage()
```

注意观察消息发送后，客户端可及时接受到消息，并没有延迟。
