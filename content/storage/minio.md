---
title: "minio 轻量级对象存储"
date: 2019-03-18T16:59:48+08:00
draft: false
toc: true 
categories: ['存储']
tags: []
---

## 简单了解

minio 完全实现了s3协议，使用简单方便。 支持多机模式，提高数据可用性和整体容量。

限制， 最多5T存储。 单个文件最大5T。 

缺点， 不能在线扩容。开发者认为扩容应该是开发人员需要解决的问题。


## 安装及简单使用

服务端
```
#下载
wget https://dl.minio.io/server/minio/release/linux-amd64/minio
mv minio /usr/local/bin/
chmod 777 /usr/local/bin/minio 

#启动服务
minio server minidata/

Endpoint:  http://10.1.88.74:9000  http://172.17.0.1:9000  http://172.19.0.1:9000  http://172.21.0.1:9000  http://172.22.0.1:9000  http://172.23.0.1:9000  http://127.0.0.1:9000      
AccessKey: ZSYLNWA109W0Q4DWDS73 
SecretKey: kuqn+i1MpR0yoE9RoT59gYjRuB5IJdz8IhIZOqP9 

Browser Access:
   http://10.1.88.74:9000  http://172.17.0.1:9000  http://172.19.0.1:9000  http://172.21.0.1:9000  http://172.22.0.1:9000  http://172.23.0.1:9000  http://127.0.0.1:9000      

Command-line Access: https://docs.minio.io/docs/minio-client-quickstart-guide
   $ mc config host add myminio http://10.1.88.74:9000 ZSYLNWA109W0Q4DWDS73 kuqn+i1MpR0yoE9RoT59gYjRuB5IJdz8IhIZOqP9

Object API (Amazon S3 compatible):
   Go:         https://docs.minio.io/docs/golang-client-quickstart-guide
   Java:       https://docs.minio.io/docs/java-client-quickstart-guide
   Python:     https://docs.minio.io/docs/python-client-quickstart-guide
   JavaScript: https://docs.minio.io/docs/javascript-client-quickstart-guide
   .NET:       https://docs.minio.io/docs/dotnet-client-quickstart-guide

#后台启动,也可做成服务模式，由systemd管理
nohup minio server minidata/ > log &
```

客户端

```
#下载
wget https://dl.minio.io/client/mc/release/linux-amd64/mc
mv mc /usr/local/bin/
chmod 777 /usr/local/bin/mc 
#根据服务启动信息配置连接
mc config host add myminio http://10.1.88.74:9000 ZSYLNWA109W0Q4DWDS73 kuqn+i1MpR0yoE9RoT59gYjRuB5IJdz8IhIZOqP9

#简单应用
mc  cp  /var/lib/pgsql/10/data/pg_wal/00000001000000C40000006A myminio/mb1
...sql/10/data/pg_wal/00000001000000C40000006A:  16.00 MB / 16.00 MB ┃▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓┃ 100.00% 96.81 MB/s 0s

```

## 服务管理

通过服务的方式管理minio

#### 添加用户，用户组
``` 
groupadd  minio:minio

useradd -g minio minio
```

#### 设置存储位置用户权限
chown minio:minio /data/ -R

#### 配置管理
vi /etc/minio.conf 
```
MINIO_VOLUMES="/data"
MINIO_OPTS="-C /minio/etc --address 192.168.6.14:9000 --console-address 192.168.6.14:19000"
MINIO_ROOT_USER='admin123'
MINIO_ROOT_PASSWORD='admin456'
```

#### 配置服务
vi /etc/systemd/system/minio.service 
```
[Unit]
Description=MinIO
Documentation=https://docs.min.io
Wants=network-online.target
After=network-online.target
AssertFileIsExecutable=/usr/local/bin/minio 
[Service]
User=minio
Group=minio
EnvironmentFile=/etc/minio.conf
ExecStart=/usr/local/bin/minio server $MINIO_OPTS $MINIO_VOLUMES
Restart=always
LimitNOFILE=65536
TimeoutStopSec=infinity
SendSIGKILL=no
[Install]
WantedBy=multi-user.target
```

#### 启动管理
```
systemctl daemon-reload
systemctl restart minio
systemctl enable minio
```

#### 压缩存储

查看可压缩类型
```
mc admin config get myminio compression
compression enable=off allow_encryption=off extensions=.txt,.log,.csv,.json,.tar,.xml,.bin mime_types=text/*,application/json,application/xml,binary/octet-stream 
```

开启压缩
```
export MINIO_COMPRESS="on"
export MINIO_COMPRESS_EXTENSIONS=".pdf,.doc"
export MINIO_COMPRESS_MIME_TYPES="application/pdf"
```

#### 查看日志
```
tail -f /var/log/
```

#### 控制台管理

在浏览器中访问 http://192.168.6.14:19000

#### 用户及权限管理

略

#### ningx 代理

负载均衡 略

## 监控

#### 健康检测

单机 Status 200
```
# curl -I http://192.168.6.14:9000/minio/health/live
HTTP/1.1 200 OK
Accept-Ranges: bytes
Content-Length: 0
Content-Security-Policy: block-all-mixed-content
Server: MinIO
Strict-Transport-Security: max-age=31536000; includeSubDomains
Vary: Origin
X-Amz-Request-Id: 169CA53FFA7B7FE6
X-Content-Type-Options: nosniff
X-Xss-Protection: 1; mode=block
```

集群
```
 /minio/health/cluster
```

#### 使用prometheus 监控

认证模式 prometheus 配置
```
scrape_configs:
- job_name: minio-job
  bearer_token: <secret>
  metrics_path: /minio/v2/metrics/cluster
  scheme: http
  static_configs:
  - targets: ['localhost:9000']

```

免认证模式 
```
#添加环境变量 /etc/minio.conf
MINIO_PROMETHEUS_AUTH_TYPE="public"
```
prometheus 配置

集群
```
scrape_configs:
- job_name: minio-job
  metrics_path: /minio/v2/metrics/cluster
  scheme: http
  static_configs:
  - targets: ['localhost:9000']
```

单机
```
scrape_configs:
- job_name: minio-job
  metrics_path: /minio/v2/metrics/node
  scheme: http
  static_configs:
  - targets: ['localhost:9000']
``` 

## 应用案例

自动同步备份数据
```
mc mirror --force --remove --watch  pgsql/data/ myminio/pgsqlbkp
```

[rclone备份应用参考](../rclone/)




