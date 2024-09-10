
## 1、查看集群节点的状态

```bash
## pgpool

show pool_nodes;
select client_addr,state,sync_state from pg_stat_replication;

## patroni
[postgres@wtj1vpk8sql01 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pgsql16 (7408451029595073953) -----------+----+-----------+
| Member | Host          | Role         | State     | TL | Lag in MB |
+--------+---------------+--------------+-----------+----+-----------+
| pg01   | 172.17.44.155 | Leader       | running   |  1 |           |
| pg02   | 172.17.44.156 | Replica      | running   |  1 |        16 |
| pg03   | 172.17.44.157 | Sync Standby | streaming |  1 |         0 |
+--------+---------------+--------------+-----------+----+-----------+
```


## 2、查看pg数据库所有schema的大小

```sql
SELECT
     pg_database.datname,
     pg_size_pretty(pg_database_size(pg_database.datname)) AS size
 FROM
     pg_database;
 
  datname  |  size   
-----------+---------
 postgres  | 7628 kB
 template1 | 7708 kB
 template0 | 7473 kB
(3 rows)     
```


## 3、查看核心的几个配置参数

```sql
SELECT  
	name, 
	setting, 
	unit
	-- (unit * 1024) 'bytes'
	FROM pg_settings WHERE name  in
	('shared_buffers',
	'max_connections',
	'wal_buffers',
	'effective_cache_size',
	'work_mem',
	'maintenance_work_mem'
	);

         name         | setting | unit 
----------------------+---------+------
 effective_cache_size | 524288  | 8kB
 maintenance_work_mem | 65536   | kB
 max_connections      | 100     | 
 shared_buffers       | 16384   | 8kB
 wal_buffers          | 512     | 8kB
 work_mem             | 4096    | kB
(6 rows)

```


-- create extension system_stats;

##  4、查看数据库编码字符集

```sql
SHOW server_encoding;

 server_encoding 
-----------------
 UTF8
(1 row)

SELECT current_database(), pg_encoding_to_char(encoding) 
FROM pg_database 
WHERE datname = current_database();

 current_database | pg_encoding_to_char 
------------------+---------------------
 postgres         | UTF8
(1 row)

```


## 5、查看pg数据库的启动时间以及运行时间：
```sql
SELECT
    pg_postmaster_start_time() AS start_time,
    current_timestamp - pg_postmaster_start_time() AS uptime;

	      start_time           |     uptime      
-------------------------------+-----------------
 2024-08-29 14:53:49.014673+08 | 00:40:38.490402
(1 row)
```
    
## 6、查看所有表的占用大小 
```sql
select relname, pg_size_pretty(pg_relation_size(relid)) from pg_stat_user_tables order by pg_relation_size(relid) desc;

 relname | pg_size_pretty 
---------+----------------
(0 rows)


```


# 7、查看 PostgreSQL 数据库中哪些表的写入操作（`INSERT`、`UPDATE`、`DELETE`）比较频繁

要查看 PostgreSQL 数据库中哪些表的写入操作（`INSERT`、`UPDATE`、`DELETE`）比较频繁，可以使用内置的统计视图 `pg_stat_user_tables`。该视图记录了自上次服务器启动以来每个表的各种活动统计信息，包括插入、更新和删除操作的计数。

以下是查询表写入频繁程度的 SQL 语句：

```sql
SELECT
    schemaname,
    relname,
    n_tup_ins AS inserts,
    n_tup_upd AS updates,
    n_tup_del AS deletes,
    n_tup_ins + n_tup_upd + n_tup_del AS total_writes
FROM
    pg_stat_user_tables
ORDER BY
    total_writes DESC
LIMIT 10;

```

### 解释：

- `schemaname`: 表所在的模式（schema）。
- `relname`: 表的名称。
- `n_tup_ins`: 自上次服务器启动以来的插入行数。
- `n_tup_upd`: 自上次服务器启动以来的更新行数。
- `n_tup_del`: 自上次服务器启动以来的删除行数。
- `total_writes`: 插入、更新、删除操作的总和，用于衡量表的写入频繁程度。

### 结果分析：

- 查询结果按写入操作总和（`total_writes`）降序排列。结果中，写入频率较高的表会排在前面。
- 如果需要查看更多的表，可以调整 `LIMIT` 子句的值。

通过这个查询，你可以快速识别哪些表在数据库中写入频率较高，从而有助于优化这些表的性能或进一步分析它们对系统资源的影响。