---
title: "nginx"
date: 2019-04-09T15:42:15+08:00
draft: false
toc: true 
categories: ["nginx"]
tags: []
---

## 性能优化

- CPU 亲和性

```
worker_processes 4; # 比如物理CPU的数量为4
worker_cpu_affinity 0001 0010 0100 1000;
```

- Nginx最大打开文件数

```
worker_rlimit_nofile 65535; ulimit -n的值保持一致
```

- GZIP 压缩

节约带宽，加快传输速度。需要CPU资源

压缩： 文本，js，html，css，不压缩: 图片，视频，flash
```
gzip on; # 开启
gzip_min_length 2k; # 小于 2k 不压缩
gzip_buffers   4 32k; # 压缩缓冲区
gzip_http_version 1.1;
gzip_comp_level 6; # 压缩比例 1-9 ,9 压缩的比例最大，需要的cpu也最多
gzip_types text/plain text/css text/javascript application/json application/javascript application/x-javascriptapplication/xml; # 压缩类型，都压缩哪些内容
gzip_vary on; # 代理服务器 ，例如用Squid缓存经过nginx压缩的数据
gzip_proxied any;
```

- expires 缓存调优

缓存，主要针对于图片，css，js等元素更改机会比较少的情况
```
location ~* \.(ico|jpe?g|gif|png|bmp|swf|flv)$ {
  expires 30d;
  #log_not_found off;
  access_log off;
}

location ~* \.(js|css)$ {
  expires 7d;
  log_not_found off;
  access_log off;
} 
```

- 启用 HTTP2 协议

在 Nginx1.9.5 及更高版本中已经支持了 HTTP/2 协议,HTTP/2 是用于服务网页的下一代协议，旨在更好地利用网络和主机服务器
```
listen 443 ssl http2;
```
验证
https://http2.pro

- 日志部分性能优化

 禁用页面资源请求的日志记录
```
location ~* \.(?:jpg|jpeg|gif|png|ico|woff2|js|css)$ {
    access_log off;
}
```

 启用IO写缓存

填满 512KB 缓冲区或自上次刷新以来已过了 1 分钟
```
access_log /var/log/nginx/access.log buffer=512k flush=1m;
```

- 下载限速

限制下载速度会留下更多带宽，以使应用程序的关键部分保持响应，这是硬件制造商使用的非常受欢迎的解决方案。
```
location /down {
    limit_rate_after 500k;
    limit_rate 50k;
}
```

- 开启高效传输模式

开启高效文件传输模式，sendfile指令指定nginx是否调用sendfile函数来输出文件，对于普通应用设为 on，如果用来进行下载等应用磁盘IO重负载应用，可设置为off，以平衡磁盘与网络I/O处理速度，降低系统的负载。注意：如果图片显示不正常把这个改成off
```
http {
  sendfile on;
  tcp_nopush on
}
```

- 防盗链

防止别人直接从你网站引用图片等链接，消耗了你的资源和网络流量
```
location ~*^.+\.(jpg|gif|png|swf|flv|wma|wmv|asf|mp3|mmf|zip|rar)$ {
valid_referers none blocked zhangeamon.top doc.zhangeamon.top localhost;
if($invalid_referer) {
  return 404;
  break;
}
access_log off;
}
```

- 内核参数优化

```
fs.file-max = 999999
net.ipv4.ip_forward = 0
net.ipv4.conf.default.rp_filter = 1
net.ipv4.conf.default.accept_source_route = 0
kernel.sysrq = 0
kernel.core_uses_pid = 1
net.ipv4.tcp_syncookies = 1
kernel.msgmnb = 65536
kernel.msgmax = 65536
kernel.shmmax = 68719476736
kernel.shmall = 4294967296
net.ipv4.tcp_max_tw_buckets = 6000
net.ipv4.tcp_sack = 1
net.ipv4.tcp_window_scaling = 1
net.ipv4.tcp_rmem = 10240 87380 12582912
net.ipv4.tcp_wmem = 10240 87380 12582912
net.core.wmem_default = 8388608
net.core.rmem_default = 8388608
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.netdev_max_backlog = 262144
net.core.somaxconn = 40960
net.ipv4.tcp_max_orphans = 3276800
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_timestamps = 0
net.ipv4.tcp_synack_retries = 1
net.ipv4.tcp_syn_retries = 1
net.ipv4.tcp_tw_recycle = 1
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_mem = 94500000 915000000 927000000
net.ipv4.tcp_fin_timeout = 1
net.ipv4.tcp_keepalive_time = 30
net.ipv4.ip_local_port_range = 1024 65000
```

[//]: #[性能优化](https://mp.weixin.qq.com/s/YoZDzY4Tmj8HpQkSgnZLvA)

## 常见错误

- 错误码 502   ， error.log 中错误信息 [error] 236#236: *8371899 upstream sent too big header while reading response header from upstream,

问题 header 过大
```
proxy_buffer_size 64k;
proxy_buffers   4 32k;
proxy_busy_buffers_size 64k;
```
[官网说明](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_buffer_size)

[基本配置](https://www.cnblogs.com/dongye95/p/11096785.html)

## 设置用户登陆认证

如下举例设置用户访问kibana时登陆认证
```
server {
   listen 80;
   server_name kibana.×××.com;
   location / {
      auth_basic "secret";
      auth_basic_user_file /etc/nginx/db/passwd.db;
      proxy_pass http://****:5601;
      proxy_set_header Host $host:5601;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header Via "nginx";
   }
   access_log off;
}
```

2、配置登录用户名(admin)，密码

```
htpasswd -c /etc/nginx/db/passwd.db admin
New password: 
Re-type new password: 
Adding password for user admin
```
htpasswd是apache自带的小工具，如果找不到该命令，尝试用yum install httpd安装

```
cat db/passwd.db 
admin:$apr1$Jc.x0rme$BWrmulBqUj.g6BeeoEM79/
```

[访问频率控制](https://www.cnblogs.com/zhangeamon/p/9341807.html)

[backend 后端服务健康检测](https://www.cnblogs.com/zhangeamon/p/9341788.html)


[日志分析goaccess](https://cloud.tencent.com/developer/article/1449085)

nginx + consul + upsync 实现动态负载均衡 
