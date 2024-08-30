### 出现问题：PANIC:  could not locate a valid checkpoint record

```bash
[root@wtj1vpk8sql02 log]# tail -f postgresql-2024-08-29_163804.log
2024-08-29 16:38:04.850 CST [11829] LOG:  starting PostgreSQL 16.3 [By gg] on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-44), 64-bit
2024-08-29 16:38:04.850 CST [11829] LOG:  listening on IPv4 address "172.17.44.156", port 5432
2024-08-29 16:38:04.879 CST [11829] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
2024-08-29 16:38:04.881 CST [11834] LOG:  database system was shut down in recovery at 2024-08-29 16:03:01 CST
2024-08-29 16:38:04.881 CST [11834] LOG:  entering standby mode
2024-08-29 16:38:04.881 CST [11834] LOG:  invalid resource manager ID in checkpoint record
2024-08-29 16:38:04.881 CST [11834] PANIC:  could not locate a valid checkpoint record
2024-08-29 16:38:04.897 CST [11829] LOG:  startup process (PID 11834) was terminated by signal 6: Aborted
2024-08-29 16:38:04.897 CST [11829] LOG:  aborting startup due to startup process failure
2024-08-29 16:38:04.898 CST [11829] LOG:  database system is shut down
```

### 解决办法：以用户postgres身份，运行pg_resetwal进行修复

```bash
[root@wtj1vpk8sql02 ~]# pg_resetwal -f /acdata/data/pg16/data/
pg_resetwal: error: cannot be executed by "root"
pg_resetwal: hint: You must run pg_resetwal as the PostgreSQL superuser.


[root@wtj1vpk8sql02 ~]# su - postgres
Last login: Thu Aug 29 16:31:14 CST 2024 on pts/0
[postgres@wtj1vpk8sql02 ~]$  pg_resetwal -f /acdata/data/pg16/data/
Write-ahead log reset
```

### 重新启动pg

```bash
[postgres@wtj1vpk8sql02 ~]$ patroni /etc/patroni/patroni.yml
2024-08-29 17:58:10,492 INFO: Lock owner: pg03; I am pg02
2024-08-29 17:58:10,534 INFO: starting as a secondary
2024-08-29 17:58:10,988 INFO: Lock owner: pg03; I am pg02
2024-08-29 17:58:11,031 INFO: restarting after failure in progress
2024-08-29 17:58:11.051 CST [16167] LOG:  redirecting log output to logging collector process
2024-08-29 17:58:11.051 CST [16167] HINT:  Future log output will appear in directory "log".
2024-08-29 17:58:11,071 INFO: postmaster pid=16167
172.17.44.156:5432 - no response
2024-08-29 17:58:11,992 INFO: Lock owner: pg03; I am pg02
2024-08-29 17:58:12,075 INFO: failed to start postgres
2024-08-29 17:58:13,488 WARNING: Postgresql is not running.
2024-08-29 17:58:13,488 INFO: Lock owner: pg03; I am pg02
2024-08-29 17:58:13,491 INFO: pg_controldata:
  pg_control version number: 1300
  Catalog version number: 202307071
  Database system identifier: 7408451029595073953
  Database cluster state: in archive recovery
  pg_control last modified: Thu Aug 29 17:58:11 2024
  Latest checkpoint location: 0/6000028
  Latest checkpoint's REDO location: 0/6000028
  Latest checkpoint's REDO WAL file: 000000010000000000000006
  Latest checkpoint's TimeLineID: 1
  Latest checkpoint's PrevTimeLineID: 1
  Latest checkpoint's full_page_writes: on
  Latest checkpoint's NextXID: 0:745
  Latest checkpoint's NextOID: 24576
  Latest checkpoint's NextMultiXactId: 1
  Latest checkpoint's NextMultiOffset: 0
  Latest checkpoint's oldestXID: 723
  Latest checkpoint's oldestXID's DB: 1
  Latest checkpoint's oldestActiveXID: 0
  Latest checkpoint's oldestMultiXid: 1
  Latest checkpoint's oldestMulti's DB: 1
  Latest checkpoint's oldestCommitTsXid: 0
  Latest checkpoint's newestCommitTsXid: 0
  Time of latest checkpoint: Thu Aug 29 17:52:12 2024
  Fake LSN counter for unlogged rels: 0/3E8
  Minimum recovery ending location: 0/6000028
  Min recovery ending loc's timeline: 1
  Backup start location: 0/0
  Backup end location: 0/0
  End-of-backup record required: no
  wal_level setting: minimal
  wal_log_hints setting: off
  max_connections setting: 100
  max_worker_processes setting: 8
  max_wal_senders setting: 10
  max_prepared_xacts setting: 0
  max_locks_per_xact setting: 64
  track_commit_timestamp setting: off
  Maximum data alignment: 8
  Database block size: 8192
  Blocks per segment of large relation: 131072
  WAL block size: 8192
  Bytes per WAL segment: 16777216
  Maximum length of identifiers: 64
  Maximum columns in an index: 32
  Maximum size of a TOAST chunk: 1996
  Size of a large-object chunk: 2048
  Date/time type storage: 64-bit integers
  Float8 argument passing: by value
  Data page checksum version: 1
  Mock authentication nonce: 6aecb3a15270954f1891dd3aecc6b3e552e483635bd3d1ac2db70cf38873ec18

```
### 查看日志

```bash
[postgres@wtj1vpk8sql02 log]$ tail -f postgresql-2024-08-29_175355.log
2024-08-29 17:53:55.301 CST [15088] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
2024-08-29 17:53:55.302 CST [15094] LOG:  database system was interrupted while in recovery at log time 2024-08-29 17:52:12 CST
2024-08-29 17:53:55.302 CST [15094] HINT:  If this has occurred more than once some data might be corrupted and you might need to choose an earlier recovery target.
2024-08-29 17:53:55.310 CST [15094] LOG:  entering standby mode
2024-08-29 17:53:55.311 CST [15094] FATAL:  WAL was generated with wal_level=minimal, cannot continue recovering
2024-08-29 17:53:55.311 CST [15094] DETAIL:  This happens if you temporarily set wal_level=minimal on the server.
2024-08-29 17:53:55.311 CST [15094] HINT:  Use a backup taken after setting wal_level to higher than minimal.
2024-08-29 17:53:55.311 CST [15088] LOG:  startup process (PID 15094) exited with exit code 1
2024-08-29 17:53:55.311 CST [15088] LOG:  aborting startup due to startup process failure
2024-08-29 17:53:55.312 CST [15088] LOG:  database system is shut down



2024-08-29 17:58:16.941 CST [16191] LOG:  starting PostgreSQL 16.3 [By gg] on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-44), 64-bit
2024-08-29 17:58:16.941 CST [16191] LOG:  listening on IPv4 address "172.17.44.156", port 5432
2024-08-29 17:58:16.941 CST [16191] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
2024-08-29 17:58:16.943 CST [16195] LOG:  database system was interrupted while in recovery at log time 2024-08-29 17:52:12 CST
2024-08-29 17:58:16.943 CST [16195] HINT:  If this has occurred more than once some data might be corrupted and you might need to choose an earlier recovery target.
2024-08-29 17:58:16.954 CST [16195] LOG:  entering standby mode
2024-08-29 17:58:16.955 CST [16195] FATAL:  WAL was generated with wal_level=minimal, cannot continue recovering
2024-08-29 17:58:16.955 CST [16195] DETAIL:  This happens if you temporarily set wal_level=minimal on the server.
2024-08-29 17:58:16.955 CST [16195] HINT:  Use a backup taken after setting wal_level to higher than minimal.
2024-08-29 17:58:16.955 CST [16191] LOG:  startup process (PID 16195) exited with exit code 1
2024-08-29 17:58:16.955 CST [16191] LOG:  aborting startup due to startup process failure
2024-08-29 17:58:16.955 CST [16191] LOG:  database system is shut down
```

### 查看3个pg节点服务器的时间同步

```bash
[root@wtj1vpk8sql04 ~]#  for i in {155..157};do ssh 172.17.44.$i "hostname;date";done
wtj1vpk8sql01
Thu Aug 29 18:06:56 CST 2024
wtj1vpk8sql02
Thu Aug 29 18:06:56 CST 2024
wtj1vpk8sql03
Thu Aug 29 18:06:56 CST 2024
```


通过日志内容`2024-08-29 17:53:55.311 CST [15094] FATAL:  WAL was generated with wal_level=minimal, cannot continue recovering`，不难发现问题出现wal_level=minimal。
集群中配置的wal_level=replica并没有生效。

### 解决办法：（修改配置文件/etc/patroni/patroni.yml）


```bash
[root@wtj1vpk8sql02 ~]# cat /etc/patroni/patroni.yml 
scope: pgsql16
#namespace: /pgsql/
namespace: /pgsql16/
name: pg02

restapi:
  listen: 0.0.0.0:8008
  connect_address: 172.17.44.156:8008

etcd3:
  hosts: 172.17.44.155:2379,172.17.44.156:2379,172.17.44.157:2379

bootstrap:
 # this section will be written into Etcd:/<namespace>/<scope>/config after initializing new cluster and all other cluster members will use it as a `global configuration`
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 30
    maximum_lag_on_failover: 1048576
    master_start_timeout: 300
    synchronous_mode: true
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
        listen_addresses: "*"
        port: 5432
        wal_level: "replica"
        hot_standby: "on"
        wal_keep_segments: 1000
        max_wal_senders: 10
        max_replication_slots: 10
        wal_log_hints: "on"
        logging_collector: "on"
#        archive_mode: "on"
#        archive_timeout: 1800s
#        archive_command: cp %p /acdata/backup/pgwalarchive/%f
#      recovery_conf:
#        restore_command: cp /acdata/backup/pgwalarchive/%f %p

  initdb:
   - encoding: UTF8
   - locale: C
   - lc-ctype: zh_CN.UTF-8
   - data-checksums

#  pg_hba:
#   - host replication repuser 172.17.44.0/21 md5
#   - host all all 172.17.0.0/16 md5

  pg_hba:
  - host replication repuser 0.0.0.0/0 md5
  - host all all 0.0.0.0/0 md5

  users:
      admin:
          password: admin
          options:
                - createrole
                - createdb

postgresql:
  listen: 172.17.44.156:5432
  connect_address: 172.17.44.156:5432
  data_dir: /acdata/data/pg16/data
  bin_dir: /acdata/data/pg16/pgsoft/bin

  authentication:
    replication:
      username: repuser
      password: '123456'
    superuser:
      username: postgres
      password: postgres

basebackup:
    max-rate: 100M
    checkpoint: fast

#callbacks:
#    on_start: /etc/patroni/patroni_callback.sh
#    on_stop:  /etc/patroni/patroni_callback.sh
#    on_role_change: /etc/patroni/patroni_callback.sh
#
#watchdog:
#  mode: automatic       # Allowed values: off, automatic, required
#  device: /dev/watchdog
#  safety_margin: 5

tags:
   nofailover: false
   noloadbalance: false
   clonefrom: false
   nosync: false
```


### 参数设置如下：

```bash
wal_level: "replica"
```


## 再次重启pg02
```bash
patroni /etc/patroni/patroni.yml
```

## 查看集群状态

```bash
[root@wtj1vpk8sql01 ~]# patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pgsql16 (7408451029595073953) -----------+----+-----------+
| Member | Host          | Role         | State     | TL | Lag in MB |
+--------+---------------+--------------+-----------+----+-----------+
| pg01   | 172.17.44.155 | Sync Standby | streaming |  2 |         0 |
| pg02   | 172.17.44.156 | Replica      | running   |  1 |         0 |
| pg03   | 172.17.44.157 | Leader       | running   |  2 |           |
+--------+---------------+--------------+-----------+----+-----------+

```


在PostgreSQL中，`wal_level` 是一个非常重要的配置参数，用于控制写前日志（WAL）的详细程度。它影响数据库在数据恢复、复制、备份等方面的能力。根据不同的应用场景，可以设置不同的 `wal_level`。

### 可用的 `wal_level` 值

PostgreSQL 16中，`wal_level` 有三个主要的可选值：

1. **`minimal`**：
    
    - 只记录最少的WAL信息，主要用于灾难恢复。
    - 适用于不需要流复制、逻辑复制或PITR（Point-In-Time Recovery）的情况。
    - 使用这种设置会有最低的WAL开销，但会限制高可用性和备份恢复的选项。
2. **`replica`**：
    
    - 记录的WAL信息足够支持流复制和备份（PITR）。
    - 这是大多数生产环境的推荐设置，因为它提供了足够的信息用于物理复制（即主从复制）。
    - 这也是PostgreSQL 9.6及以后版本的默认值。
3. **`logical`**：
    
    - 在 `replica` 的基础上增加了更多的WAL信息，以支持逻辑复制。
    - 适用于需要逻辑复制（即表级别或行级别的复制），或者需要基于触发器、数据变更捕获的场景。
    - 逻辑复制允许将数据从PostgreSQL流式传输到不同的目标，包括不同的PostgreSQL实例或其他系统。

### 配置和修改 `wal_level`

你可以在 `postgresql.conf` 配置文件中设置 `wal_level` 参数，也可以通过运行SQL命令在会话中查看当前的 `wal_level` 值：

```sql
SHOW wal_level;
```

要更改 `wal_level`，需要编辑 `postgresql.conf` 文件：

`wal_level = replica`

在修改后，需要重启PostgreSQL服务使更改生效：

`sudo systemctl restart postgresql`

### `wal_level` 的选择建议

- **使用`minimal`**：当你只关心简单的数据恢复而不需要任何形式的复制时（例如在开发环境中），可以选择`minimal`来减少磁盘I/O。
    
- **使用`replica`**：适用于大多数生产环境，尤其是需要流复制、备份恢复的场景。
    
- **使用`logical`**：当你需要逻辑复制或需要与外部系统集成的场景下，选择`logical`。
    

### 相关参数

- **`max_wal_senders`**：这个参数指定了允许多少个并发的WAL发送器进程，它在启用流复制时非常重要。
- **`max_replication_slots`**：指定了最大复制槽的数量，尤其在使用逻辑复制时非常重要。
- **`wal_keep_size`**：设置要保留的WAL日志的大小，以防止在从节点滞后时主节点删除这些WAL。

通过合理配置`wal_level`，可以有效平衡数据库的性能和数据复制/恢复功能。