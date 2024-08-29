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

