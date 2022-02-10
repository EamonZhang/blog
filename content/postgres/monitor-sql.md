---
title: "查看数据信息常用sql整理"
date: 2022-02-10T10:21:07+08:00
draft: false
toc: true
categories: ['postgres']
tags: []
---

## 库内存命中率
```
SELECT	
'index hit rate' AS name,
(sum(idx_blks_hit)) / nullif(sum(idx_blks_hit + idx_blks_read),0) AS ratio
FROM pg_statio_user_indexes
UNION ALL
SELECT
'table hit rate' AS name,
sum(heap_blks_hit) / nullif(sum(heap_blks_hit) + sum(heap_blks_read),0) AS ratio
FROM pg_statio_user_tables;
```
## 表内存命中率
```
SELECT relname AS relation, heap_blks_read AS heap_read, heap_blks_hit AS heap_hit,
((heap_blks_hit*100) / NULLIF((heap_blks_hit + heap_blks_read), 0)) AS ratio 
FROM pg_statio_user_tables order by ratio;
```
## 读IO内存占比
磁盘中读取和在内存中直接读取之间的数字和比
```
SELECT relname AS "relation", heap_blks_read AS heap_read, heap_blks_hit AS heap_hit, 
( (heap_blks_hit*100) / NULLIF((heap_blks_hit + heap_blks_read), 0)) AS ratioFROM pg_statio_user_tables;
```

## 表中索引命中率
```
SELECT relname,	
CASE idx_scan
WHEN 0 THEN NULL
ELSE round(100.0 * idx_scan / (seq_scan + idx_scan), 5)
END percent_of_times_index_used,
n_live_tup rows_in_table
FROM
pg_stat_user_tables
ORDER BY
n_live_tup DESC;
```

## 索引利用率
```
SELECT	
schemaname || '.' || relname AS table,
indexrelname AS index,
pg_size_pretty(pg_relation_size(i.indexrelid)) AS index_size,
idx_scan as index_scans
FROM pg_stat_user_indexes ui
JOIN pg_index i ON ui.indexrelid = i.indexrelid
WHERE NOT indisunique
AND idx_scan < 50
AND pg_relation_size(relid) > 5 * 8192
ORDER BY pg_relation_size(i.indexrelid) / nullif(idx_scan, 0) DESC NULLS FIRST,
pg_relation_size(i.indexrelid) DESC;
```

## 表空间大小 
```

```
## 表膨胀率
```

```
## 索引膨胀率
```

```

## 表中锁状态
```
SELECT count(pg_stat_activity.pid) AS number_of_queries,  substring(trim(LEADING FROM regexp_replace(pg_stat_activity.query, '[\n\r]+'::text,' '::text, 'g'::text))
FROM 0 FOR 200) AS query_name,  max(age(CURRENT_TIMESTAMP, query_start)) AS max_wait_time, wait_event, usename, locktype, mode, granted 
FROM pg_stat_activity LEFT JOIN pg_locks ON pg_stat_activity.pid = pg_locks.pid  WHERE query != '<IDLE>' AND query NOT ILIKE '%pg_%' 
AND query NOT ILIKE '%application_name%' AND query NOT ILIKE '%inet%' AND age(CURRENT_TIMESTAMP, query_start) > '5 milliseconds'::interval  GROUP BY query_name,
wait_event, usename, locktype, mode, granted  ORDER BY max_wait_time DESC;
```

## auto vacuum 预测
```
pg_class.reltuples*autovacuum_vacuum_scale_factor+autovacuum_vacuum_threshold
```

## auto analyze 预测
```
pg_class.reltuples*autovacuum_vacuum_scale_factor+autovacuum_vacuum_threshold
```
