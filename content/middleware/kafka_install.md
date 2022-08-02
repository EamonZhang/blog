
title: "Kafka集群安装"
date: 2022-08-02T15:54:04+08:00
draft: false
toc: false
categories: ["kafka"]
tags: []
---

## 环境

- 操作系统 centos7
- openjdk11

##  集群规划

| IP地址           |            |      |
| ---------------- | ---------- | ---- |
| 10.10.2.11/node0 | zk & kafka |      |
| 10.10.2.12/node1 | zk & kafka |      |
| 10.10.2.13/node2 | zk & kafka |      |

## 准备阶段

### 软件准备

- kafka  下载地址 https://dlcdn.apache.org/kafka/3.2.0/kafka_2.13-3.2.0.tgz 

- connetor plugins

  jdbc  下载地址  https://d1i4a15mxbxib1.cloudfront.net/api/plugins/confluentinc/kafka-connect-jdbc/versions/10.5.1/confluentinc-kafka-connect-jdbc-10.5.1.zip

  debezium 下载地址 https://repo1.maven.org/maven2/io/debezium/debezium-connector-postgres/1.9.5.Final/debezium-connector-postgres-1.9.5.Final-plugin.tar.gz

- 安装jdk 

  ```
  yum install java-11-openjdk-devel yum install java-11-openjdk -y
  ```

 				   jdk 下载地址 https://www.oracle.com/java/technologies/downloads/	  

### 系统用户

``` 
 group add kafka
 useradd kafka -g kafka
```

## 配置信息

**hosts 配置**

```
#vi /etc/hosts
10.10.2.11      node0
10.10.2.12      node1
10.10.2.13      node2
```

**zk  配置**

```
# vi zookeeper.properties
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/tmp/zookeeper
clientPort=2181
server.1=node0:2888:3888
server.2=node1:2888:3888
server.3=node2:2888:3888
# myid
echo "1" > /tmp/zookeeper/myid
echo "2" > /tmp/zookeeper/myid
echo "3" > /tmp/zookeeper/myid
```

**broker 配置**

```
# vi config/server.properties
broker.id=1   # 每个节点唯一编号
log.dirs=/tmp/kafka-logs #数据存储位置
zookeeper.connect=node0:2181,node1:2181,node2:2181/kafka # 连接zk信息

############################# Internal Topic Settings  #############################
# The replication factor for the group metadata internal topics "__consumer_offsets" and "__transaction_state"
# For anything other than development testing, a value greater than 1 is recommended to ensure availability such as 3.
offsets.topic.replication.factor=3
transaction.state.log.replication.factor=3
transaction.state.log.min.isr=3
```

**环境变量**

```
#vi /etc/profile.d/kafka.sh
KAFKA_HOME=/opt/kafka
PATH=$PATH:$KAFKA_HOME/bin

#vi  /etc/profile.d/java.sh
export JAVA_HOME=/usr/local/jdk-18.0.2
export PATH=$PATH:$JAVA_HOME/bin
```

## 集群管理

### 启动集群

​       **启动zk**

```
bin/zookeeper-server-start.sh -daemon config/zookeeper.properties
```

​       **启动kafka**

```
bin/kafka-server-start.sh -daemon config/server.properties
```

### 关闭集群

​      **关闭 kafka**

    bin/kafka-server-stop.sh

​      **关闭zk**

```
bin/zookeeper-server-start.sh
```

### connector 服务管理

​      **配置管理**

```
#vi config/connect-distributed.properties
bootstrap.servers=node0:9092,node1:9092,node2:9092
plugin.path=/opt/kafka/plugins
```

​     将准备阶段下载的插件文件拷贝到 `/opt/kafka/plugins`  中

​	 **启动 connector**

   ```
bin/connect-distributed.sh -daemon config/connect-distributed.properties
   ```

​     **查看plugins**

```
# curl -s -X GET localhost:8083/connector-plugins | jq
[
  {
    "class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "type": "sink",
    "version": "10.5.1"
  },
  {
    "class": "io.confluent.connect.jdbc.JdbcSourceConnector",
    "type": "source",
    "version": "10.5.1"
  },
  {
    "class": "io.debezium.connector.postgresql.PostgresConnector",
    "type": "source",
    "version": "1.9.5.Final"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorCheckpointConnector",
    "type": "source",
    "version": "3.2.0"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorHeartbeatConnector",
    "type": "source",
    "version": "3.2.0"
  },
  {
    "class": "org.apache.kafka.connect.mirror.MirrorSourceConnector",
    "type": "source",
    "version": "3.2.0"
  }
]
```

## 优化

```
#vi kafka-server-start.sh
...
if [ "x$KAFKA_HEAP_OPTS" = "x" ]; then
    export KAFKA_HEAP_OPTS="-server -Xms2G -Xmx2G -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:ParallelGCThreads=8 -XX:ConcGCThreads=5 -XX:InitiatingHeapOccupancyPercent=70"
    export JMX_PORT="9999"
fi

#vi config/server.properties
auto.create.topics.enable = true
auto.create.topics.enable = false

log.dirs=        # 数据存放位置
num.partitions=1  # 分区数

log.retention.hours= # 保留时长

zookeeper.connect=node0:2181,node1:2181,node2:2181/kafka
#vi config/log4j.properties
```

## 安全

### 授权机制

### 访问限制

## kafka web

### kafka-eagle

​     软件下载地址 https://github.com/smartloli/kafka-eagle-bin/archive/v2.1.0.tar.gz

```
--单机版
tar -zxvf efak-xxx-bin.tar.gz
rm -rf efak
mv efak-xxx efak

vi /etc/profile.d/efak.sh
export KE_HOME=/data/soft/efak
export PATH=$PATH:$KE_HOME/bin

cd ${KE_HOME}/conf
vi system-config.properties
cluster1.zk.list=zk01:2181,zk02:2181,zk03:2181/kafka   # 重点配置

cd ${KE_HOME}/bin
chmod +x ke.sh 
ke.sh start
ke.sh restart
ke.sh stop

-- 集群版配置
1. 各节点之间可相互ssh 免密访问
2. 配置conf/works 填写集群每个节点的ip
10.10.2.11
10.10.2.12
10.10.2.13
3. vi system-config.properties

######################################
# EFAK enable distributed
######################################
efak.distributed.enable=true
efak.cluster.mode.status=master # 其他节点 配置为 slave
efak.worknode.master.host=10.10.2.11 # 主节点IP
efak.worknode.port=8085

集群状态管理
ke.sh cluster start
ke.sh cluster restart
ke.sh cluster stop

--访问
http:xxxx:8048  admin:123456  # 修改默认用户密码
```

### kafka ui

```
#cat docker-compose.yaml

version: '3'
services:
  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    environment:
      - KAFKA_CLUSTERS_0_NAME=mycluster1
      - KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS=kakfa_host:9092
      - KAFKA_CLUSTERS_0_ZOOKEEPER=zk_host:2181
      - KAFKA_CLUSTERS_0_KAFKACONNECT_0_NAME=kc1
      - KAFKA_CLUSTERS_0_KAFKACONNECT_0_ADDRESS=http://connect_host:8083
    ports:
      - "9080:8080"
    extra_hosts:
      zk01: 10.10.2.11
      zk02: 10.10.2.12
      zk03: 10.10.2.13
```

## 待解决问题

- ksql 查询时 topic null


