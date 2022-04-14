---
title: "Prometheus 服务发现"
date: 2022-04-14T15:26:53+08:00
draft: false
toc: true 
categories: [" 监控"]
tags: []
---

## 基于consul 的服务发现


prometheus.yaml
```
## 操作系统监控 动态发现
  - job_name: 'os_system'
    #metrics_path: "/metrics"
    consul_sd_configs:
      - server: 127.0.0.1:8500
        services:
          - os_system
        scheme: http
        tags:
          - "node"
    relabel_configs:
    - source_labels: [__meta_consul_tags]
      regex: .*node.*
      action: keep
    - regex: __meta_consul_service_metadata_(.+)
      action: labelmap
```

#### 注册服务

方式一 curl
```
## prometheus 与 consul 配置之间的对应关系
## tags - tags
## service - name
## label - meta
##
curl -X PUT -d '{"id": "node-1","name": "os_system","address": "10.1.x.x","port": 9100, "tags": ["node"],"meta":{"idc":"XX机房","instance":"node1"} ,"checks": [{"http": "http://10.1.x.x:9100/","interval": "5s"}]}'  http://127.0.0.1:8500/v1/agent/service/register?replace-existing-checks=1
```

方式二 consul service register ，没找到如何设置 check
```
consul services register -id=node-1  -name=os_system -address=10.1.x.x -port=9100 -tag=node -meta=idc="xx机房" -meta=instance="node1"
```

方式三 consul service register json
```
consul services register -name=web
```

cat web.json
```
{
  "Service": {
    "Name": "web"
  }
}
```

## 基于文件的服务发现

prometheus.yml
```
- job_name: "nodes"

    file_sd_configs:

      - files:

          - "./targets/*.yaml"

        refresh_interval: 1m
```

targets/nodes.yaml
```
- targets: ['10.1.xx:9100']
      labels:
        idc: 'XX机房'
        instance: 'node-2'
```
