---
title: "数据库备份和恢复wal-g 应用"
date: 2018-10-30T10:18:57+08:00
categories: ["postgres"]
toc : true
draft: false
---

## 存储服务 minio

设置用户名和密码
```
docker run -d -p 9000:9000 -e MINIO_ACCESS_KEY=xxxxx(changeme) -e MINIO_SECRET_KEY=kkkkk(changeme)  -v /data/minio/:/data  minio/minio server /data 
```
创建 bucket

```
mc mb local/buecket003
```

## wal-g 下载 

```
wget https://github.com/wal-g/wal-g/releases/download/v0.2.9/wal-g.linux-amd64.tar.gz

tar -zxvf wal-g.linux-amd64.tar.gz 
```

## 设置环境变量

minio

cat wal-g.env 
```
export PGDATA=/var/lib/pgsql/10/data/
export WALG_S3_PREFIX=s3://bucket003/
export PGPORT=5432
export PGUSER=postgres
export AWS_SECRET_ACCESS_KEY=xxxxx(changeme)
export AWS_REGION=us-east-1
export AWS_ACCESS_KEY_ID=kkkkk(changeme)
export AWS_ENDPOINT=http://localhost:9000
export AWS_S3_FORCE_PATH_STYLE=true
```

swift

```
export PGDATA=
export WALG_SWIFT_PREFIX=swift://buckt003/
export PGPORT=
export PGUSER=
export OS_USERNAME=
export OS_PASSWORD=
export OS_AUTH_URL=http://ip:port/auth/v1.0
```

## 全备份

```
source mydir/wal-g.env &&  wal-g  backup-push $PGDATA
```

## wal 备份

```
wal_level = archive
archive_mode = on ## 从库 always
archive_command = 'source mydir/wal-g.env &&  wal-g wal-push %p'
archive_timeout = 60
```

## 恢复数据

查看所有全备份

```
wal-g backup-list
name                          last_modified        wal_segment_backup_start
base_000000020000001E000000CB 2019-11-07T01:34:08Z 000000020000001E000000CB
base_000000020000001E000000CD 2019-11-07T01:37:03Z 000000020000001E000000CD
base_000000020000001E000000CF 2019-11-07T02:23:34Z 000000020000001E000000CF
base_000000020000001E000000D1 2019-11-07T02:31:00Z 000000020000001E000000D1
base_000000020000001E000000D3 2019-11-07T02:38:29Z 000000020000001E000000D3
base_000000020000001E000000DA 2019-11-07T06:08:19Z 000000020000001E000000DA
base_000000020000001E000000DD 2019-11-07T06:30:24Z 000000020000001E000000DD
base_000000020000001E000000DF 2019-11-07T08:45:30Z 000000020000001E000000DF
```

下载一个全备份 最近的一个全备份可用 LATEST 表示
```
wal-g backup-fetch /var/lib/pgsql/10/data-restore/ base_000000020000001E000000CB
```

实时恢复

cat recover.conf
```
restore_command = 'source mydir/wal-g.env && wal-g wal-fetch %f %p'
recovery_target_time='2019-09-10 09:51:55.794813+08'
recovery_target_timeline='latest'
```

关闭数据库pause状态
```
select pg_wal_replay_resume();
```

## 清理存储

保留最近的10个备份及wal
```
wal-g delete  retain  FULL  10 (试删)

wal-g delete  retain  FULL  10  --confirm （真删） 
```

删除某个备份前的备份

```
wal-g delete before backup_name
```

## 将现有的所有wal上传

cat wal-push-all.sh
```
#!/bin/bash
#print the directory and file
 
for file in $PGDATA/pg_wal/*
do
if [ -f "$file" ]
then 
  wal-g wal-push $file
fi
done
```

## 定期全备份及清理

cat /etc/cron.weekly/pg_backup_retain.sh 例如每周一个全备份，保留近半年数据

```
source mydir/wal-g.env &&  wal-g  backup-push $PGDATA
source mydir/wal-g.env &&  wal-g delete retain FULL 26 --confirm
```

## 注意事项

1 需要先进行wal日志的备份在进行全备份。否则在恢复的时候可能会遗漏期间的wal日志。

2 全备份需要等待当前wal日志发生切换才能完成。如果是写入慢或暂无写入数据库可执行select pg_switch_wal() 进行手动触发。

3 全备份不包括pg_wal目录下的wal日志文件

## 思考

归档备份wal日志 会比生产系统的数据库滞后一个wal文件 。 是当wal日志写满或切换写新wal日志的时候触发的归档 。

如果需要使用归档文件恢复数据库时需要考虑时候可以找到最近的wal日志文件，比如在从库中。


## 其他有用脚本

下载一段连续范围内的wal日志文件 到目录 walbackup 目录中，防止下载过程中出现网络问题等。可重复多次执行

cat refetch_wal.sh
```#! /bin/bash

walfiles=$(python calc_wal.py $1 $2)

source mydir/wal-g.env

for wal_seq in $walfiles; do
 if [[ ! -f walbackup/"$wal_seq" ]];then
   wal-g wal-fetch $wal_seq walbackup/$wal_seq
   echo "fetch wal file " $wal_seq
 fi
done
```

cat calc_wal.py
```
import sys


def next_str(start):
    s_8 = start[:8]
    s_16 = start[8:16]
    s_24 = start[16:]
    if s_24.endswith('FF'):
        s_24 = hex(int('01',base=16))[2:].zfill(8)
        s_16 = hex(int(s_16, base=16) + 1)[2:].zfill(8)
    else:
        s_24 = hex(int(s_24, base=16) + 1)[2:].zfill(8)
    return ''.join([s_8, s_16, s_24]).upper()


def get_all(start, end):
    start = start.upper()
    end = end.upper()
    new_seq = None
    print(start)
    while new_seq != end:
        new_seq = next_str(new_seq if new_seq is not None else start)
        print(new_seq)


if __name__ == "__main__":
    get_all(sys.argv[1], sys.argv[2])
```

使用方法

refetch_wal.sh wal开始文件名 wal结束文件名
