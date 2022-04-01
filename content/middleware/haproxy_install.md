---
title: "Haproxy  编译安装配置 "
date: 2022-03-25T15:31:27+08:00
draft: false
toc: false
categories: ["haproxy"]
tags: []
---
## 编译安装2.4

https://www.yisu.com/zixun/4383.html



## 配置

#### pg数据库负载均衡demo
```
global
    maxconn 10000
    #安全目录
    chroot /usr/local/haproxy
    stats socket /var/lib/haproxy/haproxy.sock mode 600 level admin
    uid  99
    gid  99
    daemon
    #开启工作进程数
    #nbproc 1
    #一个进程开启线程数，单进程时可用
    nbthread 1
    #spread-checks 后端server状态check。随机百分百检测时间误差范围 2-5 （20%-50%）之间
    pidfile /var/lib/haproxy/haproxy.pid
    log 127.0.0.1 local3 info

defaults
    # 后端服务挂掉后重新分发
    option redispatch
    option http-keep-alive
    #客户端真实IP到后端服务地址
    option forwardfor
    maxconn 100000
    mode http
    #超时时间
    timeout connect 30s
    timeout client  30s
    timeout server  30s
    timeout check   5s

listen stats
    bind  :9009
    stats enable
    stats uri /status
    stats auth admin:123456
    stats realm HAPorxy\ Stats\ Page

listen pg_master
    bind 0.0.0.0:15432
    mode tcp
    balance roundrobin # leastconn
#   后端pg为patroni 
#    option httpchk
#    option http-keep-alive
#   http-check send meth OPTION uri /primary
#   http-check expect status 200
    default-server inter 3s fastinter 1s downinter 5s rise 3 fall 3 on-marked-down shutdown-sessions slowstart 30s maxconn 1000 maxqueue 128 weight 100
    server pg1 127.0.0.1:5432 check port 5432 weight 100;
    server pg2 127.0.0.1:5432 check port 5432 weight 100;   
listen pg_standby
    bind 0.0.0.0:25432
    mode tcp
    server pg1 127.0.0.1:5432 maxconn 997 check addr 127.0.0.1 port 5432 inter 3s fall 3 rise 5 weight 100
    server pg2 127.0.0.1:5432 maxconn 997 check addr 127.0.0.1 port 5432 inter 3s fall 3 rise 5 weight 100 backup

```

#### 健康检查 check 配置

- inter <delay>：设定健康状态检查的时间间隔，单位为毫秒，默认为2000；也可以使用fastinter和downinter来根据服务器端状态优化此时间延迟；
- fastinter <delay>：过渡上架、过渡下架的检查时间间隔。
- downinter <delay>：当后端服务器下架后，检查的时间间隔。
- rise <count>：设定健康状态检查中，某离线的server从离线状态转换至正常状态需要成功检查的次数；
- fall <count>：确认server从正常状态转换为不可用状态需要检查的次数。


配置文档 http://cbonte.github.io/haproxy-dconv/
