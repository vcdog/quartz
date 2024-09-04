
```bash

# 单独启动vip

# su - postgres
$ /bin/bash /etc/patroni/patroni_callback.sh on_start master pg-sentry-cluster03

# 单独关闭vip
$ /bin/bash /etc/patroni/patroni_callback.sh on_stop master pg-sentry-cluster03
2024-09-04 14:14:51 +0800 This is patroni callback on_stop master pg-sentry-cluster03
2024-09-04 14:14:51 +0800 VIP 172.17.44.159 removed

[postgres@wtj1vpk8sql01 ~]$ ifconfig
ens192: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.44.155  netmask 255.255.252.0  broadcast 172.17.47.255
        inet6 fe80::be74:66c7:d817:9c72  prefixlen 64  scopeid 0x20<link>
        inet6 fe80::cbc2:84a7:c175:3ba  prefixlen 64  scopeid 0x20<link>
        inet6 fe80::572a:d307:7644:b8e1  prefixlen 64  scopeid 0x20<link>
        ether 00:50:56:98:e0:6c  txqueuelen 1000  (Ethernet)
        RX packets 541870  bytes 1693303615 (1.5 GiB)
        RX errors 0  dropped 48  overruns 0  frame 0
        TX packets 433559  bytes 7632921655 (7.1 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 44361  bytes 31476048 (30.0 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 44361  bytes 31476048 (30.0 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

[postgres@wtj1vpk8sql01 ~]$ 
```

# 可以查看命令的使用说明

patronictl –help

```bash

[root@wtj1vpk8sql02 ~]# patronictl –help
Usage: patronictl [OPTIONS] COMMAND [ARGS]...

  Command-line interface for interacting with Patroni.

Options:
  -c, --config-file TEXT     Configuration file
  -d, --dcs-url, --dcs TEXT  The DCS connect url
  -k, --insecure             Allow connections to SSL sites without certs
  --help                     Show this message and exit.

Commands:
  dsn          Generate a dsn for the provided member, defaults to a dsn...
  edit-config  Edit cluster configuration
  failover     Failover to a replica
  flush        Discard scheduled events
  history      Show the history of failovers/switchovers
  list         List the Patroni members for a given Patroni
  pause        Disable auto failover
  query        Query a Patroni PostgreSQL member
  reinit       Reinitialize cluster member
  reload       Reload cluster member configuration
  remove       Remove cluster from DCS
  restart      Restart cluster member
  resume       Resume auto failover
  show-config  Show cluster configuration
  switchover   Switchover to a replica
  topology     Prints ASCII topology for given cluster
  version      Output version of patronictl command or a running Patroni...
```

# 查看版本号

patronictl -c /etc/patroni/patroni.yml version

```bash
[root@wtj1vpk8sql02 ~]# patronictl -c /etc/patroni/patroni.yml version
patronictl version 3.3.2
```

# 编辑配置文件

>修改最大连接数后需要重启才能生效，因此Patroni会在相关的节点状态中设置一个`Pending restart`标志。

patronictl -c /etc/patroni/patroni.yml edit-config cluster_name

```bash
[postgres@wtj1vpk8sql02 ~]$ patronictl -c /etc/patroni/patroni.yml edit-config pgsql16
--- 
+++ 
@@ -6,7 +6,7 @@
     hot_standby: 'on'
     listen_addresses: '*'
     logging_collector: 'on'
-    max_connections: 300
+    max_connections: 10000
     max_replication_slots: 10
     max_wal_senders: 10
     port: 5432

Apply these changes? [y/N]: y
Configuration changed

[postgres@wtj1vpk8sql02 ~]$ patronictl -c /etc/patroni/patroni.yml show-config pgsql16
loop_wait: 1
master_start_timeout: 300
maximum_lag_on_failover: 1048576
postgresql:
  parameters:
    hot_standby: 'on'
    listen_addresses: '*'
    logging_collector: 'on'
    max_connections: 10000
    max_replication_slots: 10
    max_wal_senders: 10
    port: 5432
    wal_keep_segments: 1000
    wal_level: replica
    wal_log_hints: 'on'
  use_pg_rewind: true
  use_slots: true
retry_timeout: 14
synchronous_mode: true
ttl: 30

[postgres@wtj1vpk8sql02 ~]$ 

[postgres@wtj1vpk8sql02 ~]$ patronictl -c /etc/patroni/patroni.yml reload pgsql16
+ Cluster: pgsql16 (7408855315163097790) -----------+----+-----------+-----------------+-----------------------------+
| Member | Host          | Role         | State     | TL | Lag in MB | Pending restart | Pending restart reason      |
+--------+---------------+--------------+-----------+----+-----------+-----------------+-----------------------------+
| pg01   | 172.17.44.155 | Replica      | streaming |  5 |         0 | *               | max_connections: 300->10000 |
| pg02   | 172.17.44.156 | Leader       | running   |  5 |           | *               | max_connections: 300->10000 |
| pg03   | 172.17.44.157 | Sync Standby | streaming |  5 |         0 | *               | max_connections: 300->10000 |
+--------+---------------+--------------+-----------+----+-----------+-----------------+-----------------------------+
Are you sure you want to reload members pg01, pg02, pg03? [y/N]: y
Reload request received for member pg01 and will be processed within 1 seconds
Reload request received for member pg02 and will be processed within 1 seconds
Reload request received for member pg03 and will be processed within 1 seconds

[postgres@wtj1vpk8sql02 ~]$ patronictl -c /etc/patroni/patroni.yml show-config pgsql16
loop_wait: 1
master_start_timeout: 300
maximum_lag_on_failover: 1048576
postgresql:
  parameters:
    hot_standby: 'on'
    listen_addresses: '*'
    logging_collector: 'on'
    max_connections: 10000
    max_replication_slots: 10
    max_wal_senders: 10
    port: 5432
    wal_keep_segments: 1000
    wal_level: replica
    wal_log_hints: 'on'
  use_pg_rewind: true
  use_slots: true
retry_timeout: 14
synchronous_mode: true
ttl: 30

[postgres@wtj1vpk8sql02 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pgsql16 (7408855315163097790) -----------+----+-----------+-----------------+-----------------------------+
| Member | Host          | Role         | State     | TL | Lag in MB | Pending restart | Pending restart reason      |
+--------+---------------+--------------+-----------+----+-----------+-----------------+-----------------------------+
| pg01   | 172.17.44.155 | Replica      | streaming |  5 |         0 | *               | max_connections: 300->10000 |
| pg02   | 172.17.44.156 | Leader       | running   |  5 |           | *               | max_connections: 300->10000 |
| pg03   | 172.17.44.157 | Sync Standby | streaming |  5 |         0 | *               | max_connections: 300->10000 |
+--------+---------------+--------------+-----------+----+-----------+-----------------+-----------------------------+
[postgres@wtj1vpk8sql02 ~]$ 

[postgres@wtj1vpk8sql02 ~]$ patronictl -c /etc/patroni/patroni.yml restart pgsql16
+ Cluster: pgsql16 (7408855315163097790) -----------+----+-----------+-----------------+-----------------------------+
| Member | Host          | Role         | State     | TL | Lag in MB | Pending restart | Pending restart reason      |
+--------+---------------+--------------+-----------+----+-----------+-----------------+-----------------------------+
| pg01   | 172.17.44.155 | Replica      | streaming |  5 |         0 | *               | max_connections: 300->10000 |
| pg02   | 172.17.44.156 | Leader       | running   |  5 |           | *               | max_connections: 300->10000 |
| pg03   | 172.17.44.157 | Sync Standby | streaming |  5 |         0 | *               | max_connections: 300->10000 |
+--------+---------------+--------------+-----------+----+-----------+-----------------+-----------------------------+
When should the restart take place (e.g. 2024-09-02T19:04)  [now]: 
Are you sure you want to restart members pg01, pg02, pg03? [y/N]: y
Restart if the PostgreSQL version is less than provided (e.g. 9.5.2)  []: 
Success: restart on member pg01
Success: restart on member pg02
Success: restart on member pg03

```

# 查看所有成员信息

patronictl -c /etc/patroni/patroni.yml list

```bash
[root@wtj1vpk8sql02 ~]# patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pgsql16 (7408855315163097790) -----------+----+-----------+
| Member | Host          | Role         | State     | TL | Lag in MB |
+--------+---------------+--------------+-----------+----+-----------+
| pg01   | 172.17.44.155 | Leader       | running   |  4 |           |
| pg02   | 172.17.44.156 | Sync Standby | streaming |  4 |         0 |
| pg03   | 172.17.44.157 | Replica      | streaming |  4 |         0 |
+--------+---------------+--------------+-----------+----+-----------+
```

# 重新加载配置

patronictl -c /etc/patroni/patroni.yml reload cluster_name

```bash
[root@wtj1vpk8sql02 ~]# patronictl -c /etc/patroni/patroni.yml reload pgsql16
+ Cluster: pgsql16 (7408855315163097790) -----------+----+-----------+
| Member | Host          | Role         | State     | TL | Lag in MB |
+--------+---------------+--------------+-----------+----+-----------+
| pg01   | 172.17.44.155 | Leader       | running   |  4 |           |
| pg02   | 172.17.44.156 | Sync Standby | streaming |  4 |         0 |
| pg03   | 172.17.44.157 | Replica      | streaming |  4 |         0 |
+--------+---------------+--------------+-----------+----+-----------+
Are you sure you want to reload members pg01, pg02, pg03? [y/N]: y
Reload request received for member pg01 and will be processed within 1 seconds
Reload request received for member pg02 and will be processed within 1 seconds
Reload request received for member pg03 and will be processed within 1 seconds
```



# 移除集群，重新配置的时候使用

patronictl -c /etc/patroni/patroni.yml  remove cluster_name

```bash

[root@wtj1vpk8sql01 ~]# patronictl remove pgsql16
+ Cluster: pgsql16 (7408855315163097790) ----+----+-----------+
| Member | Host          | Role    | State   | TL | Lag in MB |
+--------+---------------+---------+---------+----+-----------+
| pg01   | 172.17.44.155 | Replica | stopped |    |   unknown |
| pg02   | 172.17.44.156 | Replica | stopped |    |   unknown |
| pg03   | 172.17.44.157 | Replica | stopped |    |   unknown |
+--------+---------------+---------+---------+----+-----------+
Please confirm the cluster name to remove: pgsql16
You are about to remove all information in DCS for pgsql16, please type: "Yes I am aware": Yes I am aware

```


# 重启数据库集群

patronictl  -c /etc/patroni/patroni.yml restart cluster_name

```bash

[root@wtj1vpk8sql02 ~]# patronictl -c /etc/patroni/patroni.yml restart pgsql16
+ Cluster: pgsql16 (7408855315163097790) -----------+----+-----------+
| Member | Host          | Role         | State     | TL | Lag in MB |
+--------+---------------+--------------+-----------+----+-----------+
| pg01   | 172.17.44.155 | Leader       | running   |  4 |           |
| pg02   | 172.17.44.156 | Sync Standby | streaming |  4 |         0 |
| pg03   | 172.17.44.157 | Replica      | streaming |  4 |         0 |
+--------+---------------+--------------+-----------+----+-----------+
When should the restart take place (e.g. 2024-09-02T18:45)  [now]: 
Are you sure you want to restart members pg01, pg02, pg03? [y/N]: y
Restart if the PostgreSQL version is less than provided (e.g. 9.5.2)  []: 
Success: restart on member pg01
Success: restart on member pg02
Success: restart on member pg03
```

# 切换 Leader，将一个 slave 切换成 leader。

patronictl  -c /etc/patroni/patroni.yml switchover

```bash

[root@wtj1vpk8sql02 ~]# patronictl -c /etc/patroni/patroni.yml switchover
Current cluster topology
+ Cluster: pgsql16 (7408855315163097790) -----------+----+-----------+
| Member | Host          | Role         | State     | TL | Lag in MB |
+--------+---------------+--------------+-----------+----+-----------+
| pg01   | 172.17.44.155 | Leader       | running   |  4 |           |
| pg02   | 172.17.44.156 | Sync Standby | streaming |  4 |         0 |
| pg03   | 172.17.44.157 | Replica      | streaming |  4 |         0 |
+--------+---------------+--------------+-----------+----+-----------+
Primary [pg01]: 
Candidate ['pg02', 'pg03'] []: pg02                                                                    
When should the switchover take place (e.g. 2024-09-02T18:50 )  [now]: 
Are you sure you want to switchover cluster pgsql16, demoting current leader pg01? [y/N]: y
2024-09-02 17:50:29.66974 Successfully switched over to "pg02"
+ Cluster: pgsql16 (7408855315163097790) ----+----+-----------+
| Member | Host          | Role    | State   | TL | Lag in MB |
+--------+---------------+---------+---------+----+-----------+
| pg01   | 172.17.44.155 | Replica | stopped |    |   unknown |
| pg02   | 172.17.44.156 | Leader  | running |  4 |           |
| pg03   | 172.17.44.157 | Replica | running |  4 |         0 |
+--------+---------------+---------+---------+----+-----------+
[root@wtj1vpk8sql02 ~]# patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pgsql16 (7408855315163097790) -----------+----+-----------+
| Member | Host          | Role         | State     | TL | Lag in MB |
+--------+---------------+--------------+-----------+----+-----------+
| pg01   | 172.17.44.155 | Replica      | streaming |  5 |         0 |
| pg02   | 172.17.44.156 | Leader       | running   |  5 |           |
| pg03   | 172.17.44.157 | Sync Standby | streaming |  5 |         0 |
+--------+---------------+--------------+-----------+----+-----------+
[root@wtj1vpk8sql02 ~]# 
```


#  修改PostgreSQL参数

修改个别节点的参数，可以执行`ALTER SYSTEM SET ...` SQL命令，比如临时打开某个节点的debug日志。对于需要统一配置的参数应该通过`patronictl edit-config`设置，确保全局一致，比如修改最大连接数。

```bash
patronictl edit-config -p 'max_connections=300'
```

修改最大连接数后需要重启才能生效，因此Patroni会在相关的节点状态中设置一个`Pending restart`标志。

```bash

[root@wtj1vpk8sql03 ~]# patronictl list
+ Cluster: pgsql16 (7408855315163097790) -----------+----+-----------+
| Member | Host          | Role         | State     | TL | Lag in MB |
+--------+---------------+--------------+-----------+----+-----------+
| pg01   | 172.17.44.155 | Replica      | streaming | 10 |         0 |
| pg02   | 172.17.44.156 | Sync Standby | streaming | 10 |         0 |
| pg03   | 172.17.44.157 | Leader       | running   | 10 |           |
+--------+---------------+--------------+-----------+----+-----------+
```

重启集群中所有PG实例后，参数生效。

```bash
 patronictl restart pgsql16
```