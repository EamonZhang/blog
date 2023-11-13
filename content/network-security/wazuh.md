---
title: "Wazuh 功能简介"
date: 2021-11-01T09:34:45+08:00
draft: false
toc: true 
categories: [" 安全"]
tags: []
---

## 日志收集

客户端配置，指定需要收集系统日志及日志格式。默认如下

```
<ossec_config>
  <localfile>
    <log_format>audit</log_format>
    <location>/var/log/audit/audit.log</location>
  </localfile>
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/ossec/logs/active-responses.log</location>
  </localfile>
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/messages</location>
  </localfile>
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/secure</location>
  </localfile>
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/log/maillog</location>
  </localfile>
```

## 系统命令


客户端配置，设置系统监控执行命令及频率。默认如下
```
# 系统文件
<localfile>
    <log_format>command</log_format>
    <command>df -P</command>
    <frequency>360</frequency>
  </localfile>
# 端口
  <localfile>
    <log_format>full_command</log_format>
    <command>netstat -tulpn | sed 's/\([[:alnum:]]\+\)\ \+[[:digit:]]\+\ \+[[:digit:]]\+\ \+\(.*\):\([[:digit:]]*\)\ \+\([0-9\.\:\*]\+\).\+\ \([[:digit:]]*\/[[:alnum:]\-]*\).*/\1 \2 == \3 =
= \4 \5/' | sort -k 4 -g | sed 's/ == \(.*\) ==/:\1/' | sed 1,2d</command>
    <alias>netstat listening ports</alias>
    <frequency>360</frequency>
  </localfile>
# 登陆日志
  <localfile>
    <log_format>full_command</log_format>
    <command>last -n 20</command>
    <frequency>360</frequency>
  </localfile>
```

## 文件完整性监控

File integrity monitoring， FIM  。

对指定目录下的文件完整性进行监控，包括对文件的新增，删除，修改。

基本原理，对文件进行md5，sha1 等摘要提取。判断文件是否被篡改过。

实现， 集成 osquery
```
  <!-- File integrity monitoring -->
  <syscheck>
    <disabled>no</disabled>

    <!-- Frequency that syscheck is executed default every 12 hours -->
    <frequency>43200</frequency>

    <scan_on_start>yes</scan_on_start>

    <!-- Directories to check  (perform all possible verifications) -->
    <directories>/etc,/usr/bin,/usr/sbin</directories>
    <directories>/bin,/sbin,/boot</directories>

    <!-- Files/directories to ignore -->
    <ignore>/etc/mtab</ignore>
    <ignore>/etc/hosts.deny</ignore>
    <ignore>/etc/mail/statistics</ignore>
    <ignore>/etc/random-seed</ignore>
    <ignore>/etc/random.seed</ignore>
    <ignore>/etc/adjtime</ignore>
    <ignore>/etc/httpd/logs</ignore>
    <ignore>/etc/utmpx</ignore>
    <ignore>/etc/wtmpx</ignore>
    <ignore>/etc/cups/certs</ignore>
    <ignore>/etc/dumpdates</ignore>
    <ignore>/etc/svc/volatile</ignore>

    <!-- File types to ignore -->
    <ignore type="sregex">.log$|.swp$</ignore>

    <!-- Check the file, but never compute the diff -->
    <nodiff>/etc/ssl/private.key</nodiff>

    <skip_nfs>yes</skip_nfs>
    <skip_dev>yes</skip_dev>
    <skip_proc>yes</skip_proc>
    <skip_sys>yes</skip_sys>

    <!-- Nice value for Syscheck process -->
    <process_priority>10</process_priority>

    <!-- Maximum output throughput -->
    <max_eps>100</max_eps>

    <!-- Database synchronization settings -->
    <synchronization>
      <enabled>yes</enabled>
      <interval>5m</interval>
      <max_interval>1h</max_interval>
      <max_eps>10</max_eps>
    </synchronization>
  </syscheck>
```

## 安全配置评估

Security configuration assessment，SCA

监控系统的配置文件是否合乎安全配置基线
```
  <sca>
    <enabled>yes</enabled>
    <scan_on_start>yes</scan_on_start>
    <interval>12h</interval>
    <skip_nfs>yes</skip_nfs>
  </sca>
```

检测规则 `/var/ossec/ruleset/sca/cis_centos7_linux.yml`

## 资产清单

System inventory

如操作系统版本，硬件，端口等。扫描结果保持在本地sqlite

```
 <!-- System inventory -->
  <wodle name="syscollector">
    <disabled>no</disabled>
    <interval>1h</interval>
    <scan_on_start>yes</scan_on_start>
    <hardware>yes</hardware>
    <os>yes</os>
    <network>yes</network>
    <packages>yes</packages>
    <ports all="no">yes</ports>
    <processes>yes</processes>

    <!-- Database synchronization settings -->
    <synchronization>
      <max_eps>10</max_eps>
    </synchronization>
  </wodle>
```

## 安全策略监控

Policy monitoring

```
<!-- Policy monitoring -->
  <rootcheck>
    <disabled>no</disabled>
    <check_files>yes</check_files>
    <check_trojans>yes</check_trojans>
    <check_dev>yes</check_dev>
    <check_sys>yes</check_sys>
    <check_pids>yes</check_pids>
    <check_ports>yes</check_ports>
    <check_if>yes</check_if>

    <!-- Frequency that rootcheck is executed - every 12 hours -->
    <frequency>43200</frequency>

    <rootkit_files>etc/shared/rootkit_files.txt</rootkit_files>
    <rootkit_trojans>etc/shared/rootkit_trojans.txt</rootkit_trojans>

    <skip_nfs>yes</skip_nfs>
  </rootcheck>
```

## 恶意软件检测

Malware detection

VirusTotal is an online portal, owned by Google, that uses many antivirus engines to check for viruses and malware

默认不开启

VirusTotal，是一个提供免费的可疑文件分析服务的网站， 只能扫描提交的文件。

```
<ossec_config>
  <integration>
    <name>virustotal</name>
    <api_key>${your_virustotal_api_key}</api_key>
    <rule_id>100200,100201</rule_id>
    <alert_format>json</alert_format>
  </integration>
</ossec_config>
```

## 漏洞检测

名称解释： 

CVE （Common Vulnerabilities & Exposures） 漏洞信息库

NVD  (National Vulnerability Database) 美国国家漏洞库

vulnerability detector

`WARNING: vulnerability-detector configuration is only set in the manager`

在管理端配置

```
 <vulnerability-detector>
    <enabled>yes</enabled>
    <interval>5m</interval>
    <ignore_time>6h</ignore_time>
    <run_on_start>yes</run_on_start>

    <!-- Ubuntu OS vulnerabilities -->
    <provider name="canonical">
      <enabled>no</enabled>
      <os>trusty</os>
      <os>xenial</os>
      <os>bionic</os>
      <os>focal</os>
      <update_interval>1h</update_interval>
    </provider>

    <!-- Debian OS vulnerabilities -->
    <provider name="debian">
      <enabled>no</enabled>
      <os>stretch</os>
      <os>buster</os>
      <update_interval>1h</update_interval>
    </provider>

    <!-- RedHat OS vulnerabilities -->
    <provider name="redhat">
      <enabled>yes</enabled>
      <os>5</os>
      <os>6</os>
      <os>7</os>
      <os>8</os>
      <update_interval>1h</update_interval>
    </provider>

    <!-- Windows OS vulnerabilities -->
    <provider name="msu">
      <enabled>yes</enabled>
      <update_interval>1h</update_interval>
    </provider>

    <!-- Aggregate vulnerabilities -->
    <provider name="nvd">
      <enabled>yes</enabled>
      <update_from_year>2010</update_from_year>
      <update_interval>1h</update_interval>
    </provider>

  </vulnerability-detector>

```

基本流程： 

1. agent 扫描并上报本地信息到 manager 

2. manager 将上报的每个agent信息分别存储到各自sqlite , 存储目录`/var/ossec/queue/db/`

3. manager 从网络上下载更新最新的病毒库。然后利用病毒库进行检测。

注意： 由于病毒库需要从网络下载，在网络环境不能保证时采用离线模式。手动定期更新病毒库。 [参见](https://documentation.wazuh.com/current/user-manual/capabilities/vulnerability-detection/offline_update.html)

配置后需求重启服务

```
/var/ossec/bin/wazuh-control restart 
```

## 自动响应

Active response

```
  <!-- Active response -->
  <active-response>
    <disabled>no</disabled>
    <ca_store>etc/wpk_root.pem</ca_store>
    <ca_verification>yes</ca_verification>
  </active-response>
```

## 容器监控

默认配置中没有如下配置，有需求自行添加
```
<wodle name="docker-listener">
    <interval>10m</interval>
    <attempts>5</attempts>
    <run_on_start>no</run_on_start>
    <disabled>no</disabled>
</wodle>
```

## 其他

略

## agent 管理

在 manager 端，管理命令

可以对`agent`进行查看,添加，删除操作
```
/var/ossec/bin/agent_control
/var/ossec/bin/manage_agents
```

## 配置说明

以上配置正确姿势

在web管理界面中，Management -> Configuration 。 生成模版

## agent 与 mangager 通信

需要端口 1515 认证 1514 通信 

## 扩展 

利用 hydra 进行暴力破解，查看Wazuh监控情况。

