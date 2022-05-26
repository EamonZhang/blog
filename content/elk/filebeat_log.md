---
title: "filebeat 自身log输出位置问题"
date: 2022-05-26T10:13:30+08:00
draft: false
toc: false
categories: []
tags: []
---

## file beat 日志输出管理

默认情况 filebeat日志输出位置为系统日志文件 `/var/log/messages` 中

## 重定向日志文件输出处位置

cat /usr/lib/systemd/system/filebeat.service
```
[Unit]
Description=Filebeat sends log files to Logstash or directly to Elasticsearch.
Documentation=https://www.elastic.co/products/beats/filebeat
Wants=network-online.target
After=network-online.target

[Service]

Environment="BEAT_LOG_OPTS="
Environment="BEAT_CONFIG_OPTS=-c /etc/filebeat/filebeat.yml"
Environment="BEAT_PATH_OPTS=-path.home /usr/share/filebeat -path.config /etc/filebeat -path.data /var/lib/filebeat -path.logs /var/log/filebeat"
ExecStart=/usr/share/filebeat/bin/filebeat -environment systemd $BEAT_LOG_OPTS $BEAT_CONFIG_OPTS $BEAT_PATH_OPTS
Restart=always

[Install]
WantedBy=multi-user.target
```

filebeat.yml
```
logging.level: info
logging.to_files: true
logging.files:
  path: /var/log/filebeat
  name: filebeat
  keepfiles: 7
  permissions: 0644

```

