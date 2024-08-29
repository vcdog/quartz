
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
