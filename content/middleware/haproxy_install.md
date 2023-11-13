---
title: "Haproxy  编译安装配置 "
date: 2022-03-25T15:31:27+08:00
draft: false
toc: true
categories: ["haproxy"]
tags: []
---
## 编译安装Haproxy 2.4

#### 下载预备

```
sudo wget https://www.haproxy.org/download/2.4/src/haproxy-2.4.15.tar.gz -O /usr/local/src/haproxy-2.4.15.tar.gz
sudo wget http://www.lua.org/ftp/lua-5.3.5.tar.gz -O /usr/local/src/lua-5.3.5.tar.gz
sudo yum  install make gcc build-essential libssl-devel zlib1g-devel pcre3 pcre3-devel systemd-devel readline-devel openssl openssl-devel -y 
```

#### 编译安装

lua 
```
cd /usr/local/src && tar -zxvf lua-5.3.5.tar.gz
cd /usr/local/src/lua-5.3.5 && make linux
src/lua -v
```

haproxy
```
cd /usr/local/src && tar -zxvf haproxy-2.4.15.tar.gz
cd /usr/local/src/haproxy-2.4.15 
make -j `lscpu |awk 'NR==4{print $2}'` ARCH=x86_64 TARGET=linux-glibc USE_PCRE=1 USE_ZLIB=1 USE_SYSTEMD=1 USE_CPU_AFFINITY=1 USE_LUA=1 LUA_INC=/usr/local/src/lua-5.3.5/src/ LUA_LIB=/usr/local/src/lua-5.3.5/src/ PREFIX=/usr/local/haproxy
make install PREFIX=/usr/local/haproxy
/usr/local/haproxy/sbin/haproxy -v
```

如果想添加 openssl 支持参考
```
In order to link OpenSSL statically against HAProxy, first download OpenSSL
from https://www.openssl.org/ then build it with the "no-shared" keyword and
install it to a local directory, so your system is not affected :

  $ export STATICLIBSSL=/tmp/staticlibssl
  $ ./config --prefix=$STATICLIBSSL no-shared
  $ make && make install_sw

Then when building haproxy, pass that path via SSL_INC and SSL_LIB :

  $ make TARGET=generic \
    USE_OPENSSL=1 SSL_INC=$STATICLIBSSL/include SSL_LIB=$STATICLIBSSL/lib
```

更多编译参数信息请参考 INSTALL 文件,在刚刚下载的源码中

## 服务管理

```
mkdir /etc/haproxy
mkdir /var/lib/haproxy
```

```
cat >> /lib/systemd/system/haproxy.service << EOF

[Unit]
Description=HAProxy Load Balancer
After=syslog.target network.target

[Service]
ExecStartPre=/usr/local/haproxy/sbin/haproxy -f /etc/haproxy/haproxy.cfg -c
ExecStart=/usr/local/haproxy/sbin/haproxy -Ws -f /etc/haproxy/haproxy.cfg -p /var/lib/haproxy/haproxy.pid
ExecReload=/bin/kill -USR2 $MAINPID

[Install]
WantedBy=multi-user.target

EOF
``` 

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
检查配置

```
usr/local/haproxy/sbin/haproxy -f /etc/haproxy/haproxy.cfg -c
```


#### 健康检查 check 配置

- inter <delay>：设定健康状态检查的时间间隔，单位为毫秒，默认为2000；也可以使用fastinter和downinter来根据服务器端状态优化此时间延迟；
- fastinter <delay>：过渡上架、过渡下架的检查时间间隔。
- downinter <delay>：当后端服务器下架后，检查的时间间隔。
- rise <count>：设定健康状态检查中，某离线的server从离线状态转换至正常状态需要成功检查的次数；
- fall <count>：确认server从正常状态转换为不可用状态需要检查的次数。


配置文档 http://cbonte.github.io/haproxy-dconv/


#### 动态更新

socat

#### Ubuntu 参见


https://www.yisu.com/zixun/4383.html
