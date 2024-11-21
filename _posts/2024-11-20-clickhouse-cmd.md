---
title:  "Clickhouse常用运维命令"
categories: 
  - 大数据
  - 运维
tags:
  - 大数据
  - 基础组件
  - 运维
  - clickhouse
toc: true
---

## Clickhouse常用运维命令

#### 在clickhouse环境查看分区目录底层存储

```shell
#项目运行在k8s pod中
kubectl exec -it -n component clickhouse-olap-group0-0 bash

#查看表分区目录
cd /var/lib/clickhouse/data/bdp/tracks_local && ls -lt --time=ctime

#查看新的分区目录
ll | grep "_0/"

#连接clickhouse
clickhouse-client  -h localhost -u $CLICKHOUSE_ADMIN_USERNAME --password $CLICKHOUSE_ADMIN_PASSWORD --port 9000

```

#### 使用sql排查问题

```sql
-- 查询不同表占用存储大小
select database, table, formatReadableSize(sum(bytes_on_disk)) as disk_space from system.parts group by database, table order by disk_space desc;

-- 查询不同表数据量,压缩前大小, 压缩后大小和压缩率
select database, table, sum(rows) as row,formatReadableSize(sum(data_uncompressed_bytes)) as ysq,formatReadableSize(sum(data_compressed_bytes)) as ysh, round(sum(data_compressed_bytes) / sum(data_uncompressed_bytes) * 100, 0) ys_rate from system.parts  group by database, table;

-- 查询表的分区数目
SELECT database, table, COUNT(DISTINCT partition) AS partition_count FROM system.parts GROUP BY database, table ORDER BY database, table;

-- 查询系统关于硬盘的监控
SELECT * FROM system.metrics WHERE metric LIKE '%Disk%';

-- 查询当前正在合并的分区目录
SELECT * FROM system.merges ORDER BY elapsed DESC;

-- 统计不同的表有多少在合并
SELECT database,table, count(*)FROM system.merges group by database, table;

-- 查询表正在进行的mutation操作
SELECT * FROM system.mutations where table = 'xxx' ORDER BY create_time DESC;

-- 查询merge_with_ttl_timeout参数值
select * from system.merge_tree_settings where name='merge_with_ttl_timeout';

-- 查询没有完成的mutation
SELECT * FROM system.mutations where  is_done != 1;

-- 杀掉无用的mutation操作
KILL MUTATION WHERE database = 'your_database' AND table = 'your_table' AND mutation_id = 'mutation_id';
```