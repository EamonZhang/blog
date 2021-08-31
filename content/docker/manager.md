---
title: "docker 磁盘空间管理"
date: 2019-03-18T08:58:07+08:00
draft: false
toc: true
categories: ["docker"]
---

## 查看docker占用的空间情况

```
# docker system df 
TYPE                TOTAL               ACTIVE              SIZE                RECLAIMABLE
Images              58                  36                  6.091GB             2.119GB (34%)
Containers          90                  89                  232.3MB             0B (0%)
Local Volumes       137                 16                  232.7MB             194.2MB (83%)
Build Cache         0                   0                   0B                  0B

```
四大资源尽收眼底，可回收多少资源也了然于胸

## 清除不在需要的资源

```
This will remove:
        - all stopped containers
        - all networks not used by at least one container
        - all dangling images
        - all build cach

# docker system prune  -f
```
清除一切非活跃状态，将资源还给系统

## 清除volume

```
查看
# docker volume ls

清除
# docker volume prune -f
```

## docker logs 占用磁盘问题

#### 查看
cat dock_log_size.sh 
```
#!/bin/sh

echo "======== docker containers logs file size ========"  

logs=$(find /var/lib/docker/containers/ -name *-json.log)  

for log in $logs  
        do  
             ls -lh $log   
        done  

```

#### 清理
cat clean_docker_logs.sh 
```
#!/bin/sh

echo "======== docker containers logs file size ========"  

logs=$(find /var/lib/docker/containers/ -name *-json.log)  

for log in $logs  
        do  
             cat /dev/null >  $log   
        done  

```

#### 修改docker 服务配置
 
vi /etc/docker/daemon.json
```
"log-driver":"json-file",
  "log-opts":{ "max-size" :"50m","max-file":"1"}
```


https://docs.docker.com/config/containers/logging/configure/

注意： 我们需要重新创建容器才可以实现该配置的生效。

创建好以后，通过docker inspect ，或者 docker inspect -f '{{.HostConfig.LogConfig}}' 容器名xxx 来查看是否生效了
