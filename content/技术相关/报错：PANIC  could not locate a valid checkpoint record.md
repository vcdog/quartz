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
```