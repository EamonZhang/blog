---
title: "逻辑复制实现数据迁移"
date: 2022-07-01T16:36:52+08:00
draft: false
toc: true
categories: ['postgres']
tags: []
---

# DTS 数据迁移服务

## 实现目标

​		平滑将现有在线业务数据库数据迁移到新数据库中。

​		 如数据库大版本升级、原pg数据库迁移到citus集群、 多数据源汇总等业务场景。

## 迁移前原库检查

- 配置检查

  ```
  -- 源端
  wal_level = logical
  max_replication_slots = 大于1 
  max_wal_senders = 
  max_worker_processes 
  -- 目标端
  max_replication_slots，大于等于该实例总共需要创建的订阅数
  max_logical_replication_workers，大于等于该实例总共需要创建的订阅数
  max_worker_processes
  ```

- 表必须存在主键或唯一约束，作为复制标识。

   ```
  -- 没有主键或索引的表
  select relname as table_name from pg_stat_user_tables where relid not in (select pc.oid  from pg_class pc join pg_index pi  on pc.oid = pi.indrelid join pg_stat_user_tables psut on psut.relid = pc.oid  where indisunique = 't' and  indisprimary = 't');
   ```

  ```
  -- 特殊表的发布处理
  alter table table_name
      REPLICA IDENTITY { DEFAULT | USING INDEX index_name | FULL | NOTHING }
  ```

- 迁移过程中避免DDL操作

- 原库发生故障切换，迁移失败。11版本前不具备failover slot

- 原数据必须预留足够的磁盘空间存储迁移过程中产生的wal日志

- 发布端需为数据库主节点

- 数据库用户及权限

- 源数据库连接

## 其他注意事项

- Sequence不会按照源库的Sequence最大值作为初始值去递增，需要在业务切换前，在源库中查询对应Sequence的最大值，然后在目标库中将其作为对应Sequence的初始值。
- FLOAT或DOUBLE的列的迁移精度是否符合业务预期
- 迁移过程中的数据变更仅支持DML，如 INSERT、UPDATE、DELETE操作
- DDL 操作不会同步，可使用触发器完成

## 迁移过程

- 数据结构迁移

  ```
  -- 备份出原始数据结构
  pg_dump \
     --format=plain \
     --no-owner \
     --schema-only \
     --file=schema.sql \
     --schema=target_schema \
     postgres://user:pass@host:5432/db
  
  -- 在目标端创建表结构
  \i schema.sql
  -- 根据实际业务调整表结构
  
  ```

- 存量迁移

- 增量迁移

  ```
  在创建slot时，在源数据库端创建SNAPSHOT快照，基于快照完成全量及增量数据迁移
  -- 发布
  select pg_create_logical_replication_slot('logical_slot_name001','pgoutput');
  create publication pub1 for all tables
  -- 详细语法
  CREATE PUBLICATION name
      [ FOR TABLE [ ONLY ] table_name [ * ] [, ...]
        | FOR ALL TABLES ]
      [ WITH ( publication_parameter [= value] [, ... ] ) ]
  URL: https://www.postgresql.org/docs/12/sql-createpublication.html
  
  -- 订阅
  create  subscription sub1 connection 'hostaddr= xxx port=xxx user=xxx dbname=xxx' 
   publication pub1 with(create_slot='false',slot_name='logical_slot_name001'); 
  -- 详细语法
  CREATE SUBSCRIPTION subscription_name
      CONNECTION 'conninfo'
      PUBLICATION publication_name [, ...]
      [ WITH ( subscription_parameter [= value] [, ... ] ) ]
  
  参数说明
  subscription_parameter
  copy_data :  The default is true ,存量数据是否迁移
  create_slot: The default is true. 创建slot
  enabled : The default is true 是否马上开启复制
  slot_name: synchronous_commit (enum) The default value is off.
  connect (boolean) : 
  URL: https://www.postgresql.org/docs/12/sql-createsubscription.html
  
  --禁止或开启订阅
  ALTER SUBSCRIPTION mysub DISABLE;
  ALTER SUBSCRIPTION mysub ENABLE
  ALTER SUBSCRIPTION name REFRESH PUBLICATION 
  -- 如果发布端修改，订阅端刷新订阅
  ALTER SUBSCRIPTION name REFRESH PUBLICATION 
  ```



## 过程监控

### 发布端

```
-- 发布端
select * from pg_stat_replication ;
select * from pg_replication_slots ;
select * from pg_publication;
select * from pg_publication_tables ;

select pg_size_pretty(pg_wal_location_diff(pg_current_wal_insert_location(), sent_location)), pg_size_pretty(pg_wal_location_diff(pg_current_wal_insert_location(), replay_location)), * from pg_stat_replication ;
```

### 订阅端

```
-- 订阅端
select * from pg_subscription ;
select * from pg_stat_subscription ;
select * from pg_replication_origin_status ;
select pg_size_pretty(pg_wal_location_diff(received_lsn, latest_end_lsn)), * from pg_stat_subscription ;
```

## 结果检验

- 表结构，表数量，索引数量
- 表中数据量、数据条数
- 数据内容

## 迁移后处理

- 新数据库序列处理，业务切割前处理

  ```
  -- sql 参考
  do language plpgsql $$
  declare
    nsp name;
    rel name;
    val int8;
  begin
    for nsp,rel in select nspname,relname from pg_class t2 , pg_namespace t3 where t2.relnamespace=t3.oid and t2.relkind='S'
    loop
      execute format($_$select last_value from %I.%I$_$, nsp, rel) into val;
      raise notice '%',
      format($_$select setval('%I.%I'::regclass, %s);$_$, nsp, rel, val+1);
    end loop;
  end;
  $$;
  ```

- 删除逻辑复制slot ，发布订阅。业务切割后处理

  ```
  DROP PUBLICATION mypublication;
  DROP SUBSCRIPTION mysub;
  select pg_drop_replication_slot('myslot');
  ```

## 业务切割

​	略

## 失败撤回

- 增量回流 

  主要是考虑如果新数据库遇到问题需要切换回原来的数据库场景。
