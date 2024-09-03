
# 查看patroni集群初始化信息

```bash
[root@wtj1vpk8sql04 ~]# etcdctl --endpoints=172.17.44.158:2379,172.17.44.68:2379,172.17.44.69:2379  get /service/pg-sentry-prod/config
/service/pg-sentry-prod/config
{"ttl":30,"loop_wait":1,"retry_timeout":14,"maximum_lag_on_failover":1048576,"master_start_timeout":300,"synchronous_mode":true,"postgresql":{"use_pg_rewind":true,"use_slots":true,"parameters":{"max_connections":8000,"superuser_reserved_connections":10,"shared_buffers":"4GB","wal_buffers":"1GB","effective_cache_size":"12GB","work_mem":"4MB","maintenance_work_mem":"512MB","max_wal_size":"2GB","min_wal_size":"256MB","idle_in_transaction_session_timeout":30000,"effective_io_concurrency":8,"parallel_workers_per_gather":8,"listen_addresses":"*","port":5432,"wal_level":"replica","hot_standby":"on","wal_keep_segments":1000,"max_wal_senders":10,"max_replication_slots":10,"wal_log_hints":"on","logging_collector":"on"}}}
```


# 查询所有key

```bash
[root@wtj1vpk8sql04 ~]#  etcdctl --endpoints=172.17.44.158:2379,172.17.44.68:2379,172.17.44.69:2379  get / --prefix --keys-only
/pgsql/pgsql16/config

/pgsql/pgsql16/initialize

/pgsql/pgsql16/status

/pgsql/pgsql16/sync

/pgsql16/pgsql16/config

/pgsql16/pgsql16/history

/pgsql16/pgsql16/initialize

/pgsql16/pgsql16/status

/pgsql16/pgsql16/sync

/service/pg-sentry-prod/config

/service/pg-sentry-prod/failover

/service/pg-sentry-prod/history

/service/pg-sentry-prod/initialize

/service/pg-sentry-prod/leader

/service/pg-sentry-prod/members/pg01

/service/pg-sentry-prod/members/pg02

/service/pg-sentry-prod/members/pg03

/service/pg-sentry-prod/status

/service/pg-sentry-prod/sync


```


# 查询具体key

```bash
[root@wtj1vpk8sql04 ~]#  etcdctl --endpoints=172.17.44.158:2379,172.17.44.68:2379,172.17.44.69:2379 get /service/pg-sentry-prod/config
/service/pg-sentry-prod/config
{"ttl":30,"loop_wait":1,"retry_timeout":14,"maximum_lag_on_failover":1048576,"master_start_timeout":300,"synchronous_mode":true,"postgresql":{"use_pg_rewind":true,"use_slots":true,"parameters":{"max_connections":8000,"superuser_reserved_connections":10,"shared_buffers":"4GB","wal_buffers":"1GB","effective_cache_size":"12GB","work_mem":"4MB","maintenance_work_mem":"512MB","max_wal_size":"2GB","min_wal_size":"256MB","idle_in_transaction_session_timeout":30000,"effective_io_concurrency":8,"parallel_workers_per_gather":8,"listen_addresses":"*","port":5432,"wal_level":"replica","hot_standby":"on","wal_keep_segments":1000,"max_wal_senders":10,"max_replication_slots":10,"wal_log_hints":"on","logging_collector":"on"}}}
```


```bash 
#查询所有key
[root@pg_node1 data]# etcdctl --endpoints='192.168.210.15:2379' get / --prefix --keys-only
/service/twpg/config
/service/twpg/history
/service/twpg/initialize
/service/twpg/leader
/service/twpg/members/pg1
/service/twpg/members/pg2
/service/twpg/members/pg3
/service/twpg/optime/leader
--查询具体key
[root@pg_node1 data]# etcdctl --endpoints='192.168.210.15:2379' get /service/twpg/config

#通过restapi查询patroni信息（主节点）
[root@pg_node1 data]# curl -s http://192.168.210.15:8008/patroni | jq
{
  "database_system_identifier": "6976142033405049133",
  "postmaster_start_time": "2021-06-21 15:33:22.073 CST",
  "timeline": 3,
  "cluster_unlocked": false,
  "patroni": {
    "scope": "twpg",
    "version": "2.0.2"
  },
  "replication": [
    {
      "sync_state": "async",
      "sync_priority": 0,
      "client_addr": "192.168.210.81",
      "state": "streaming",
      "application_name": "pg2",
      "usename": "repuser"
    },
    {
      "sync_state": "async",
      "sync_priority": 0,
      "client_addr": "192.168.210.33",
      "state": "streaming",
      "application_name": "pg3",
      "usename": "repuser"
    }
  ],
  "state": "running",
  "role": "master",
  "xlog": {
    "location": 83886408
  },
  "server_version": 130003
}
#通过restapi查询patroni信息（备节点）
[root@pg_node1 data]# curl -s http://192.168.210.81:8008/patroni | jq
{
  "database_system_identifier": "6976142033405049133",
  "postmaster_start_time": "2021-06-21 15:37:42.746 CST",
  "timeline": 3,
  "cluster_unlocked": false,
  "patroni": {
    "scope": "twpg",
    "version": "2.0.2"
  },
  "state": "running",
  "role": "replica",
  "xlog": {
    "received_location": 83886408,
    "replayed_timestamp": null,
    "paused": false,
    "replayed_location": 83886408
  },
  "server_version": 130003
}

#模拟etcd节点出现问题，停掉etcd的LEADER节点
[root@pg_node1 ~]# etcdctl endpoint status --endpoints='192.168.210.15:2379,192.168.210.81:2379,192.168.210.33:2379' -w table
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT       |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.210.15:2379 | 2baa1a77ec379977 |   3.5.0 |  1.0 MB |     false |      false |        15 |       4866 |               4866 |        |
| 192.168.210.81:2379 |  ff5595d67d21105 |   3.5.0 |  1.0 MB |      true |      false |        15 |       4866 |               4866 |        |
| 192.168.210.33:2379 | b5d9c4826815356e |   3.5.0 |  1.0 MB |     false |      false |        15 |       4866 |               4866 |        |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
[root@pg_node2 ~]# systemctl stop etcd
[root@pg_node2 ~]# etcdctl endpoint status --endpoints='192.168.210.15:2379,192.168.210.81:2379,192.168.210.33:2379' -w table
{"level":"warn","ts":"2021-06-23T09:20:00.826+0800","logger":"etcd-client","caller":"v3/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0xc0003fea80/#initially=[192.168.210.15:2379;192.168.210.81:2379;192.168.210.33:2379]","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = latest balancer error: last connection error: connection error: desc = \"transport: Error while dialing dial tcp 192.168.210.81:2379: connect: connection refused\""}
Failed to get the status of endpoint 192.168.210.81:2379 (context deadline exceeded)
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT       |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.210.15:2379 | 2baa1a77ec379977 |   3.5.0 |  1.0 MB |     false |      false |        16 |       4867 |               4867 |        |
| 192.168.210.33:2379 | b5d9c4826815356e |   3.5.0 |  1.0 MB |      true |      false |        16 |       4867 |               4867 |        |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
[root@pg_node1 ~]# etcdctl endpoint status --endpoints='192.168.210.15:2379,192.168.210.81:2379,192.168.210.33:2379' -w table
{"level":"warn","ts":"2021-06-23T09:16:37.350+0800","logger":"etcd-client","caller":"v3/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0xc00037c700/#initially=[192.168.210.15:2379;192.168.210.81:2379;192.168.210.33:2379]","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = latest balancer error: last connection error: connection error: desc = \"transport: Error while dialing dial tcp 192.168.210.81:2379: connect: connection refused\""}
Failed to get the status of endpoint 192.168.210.81:2379 (context deadline exceeded)
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT       |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.210.15:2379 | 2baa1a77ec379977 |   3.5.0 |  1.0 MB |     false |      false |        16 |       4867 |               4867 |        |
| 192.168.210.33:2379 | b5d9c4826815356e |   3.5.0 |  1.0 MB |      true |      false |        16 |       4867 |               4867 |        |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
[root@pg_node3 ~]# etcdctl endpoint status --endpoints='192.168.210.15:2379,192.168.210.81:2379,192.168.210.33:2379' -w table
{"level":"warn","ts":"2021-06-23T09:20:39.338+0800","logger":"etcd-client","caller":"v3/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0xc00013a380/#initially=[192.168.210.15:2379;192.168.210.81:2379;192.168.210.33:2379]","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = latest balancer error: last connection error: connection error: desc = \"transport: Error while dialing dial tcp 192.168.210.81:2379: connect: connection refused\""}
Failed to get the status of endpoint 192.168.210.81:2379 (context deadline exceeded)
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT       |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.210.15:2379 | 2baa1a77ec379977 |   3.5.0 |  1.0 MB |     false |      false |        16 |       4867 |               4867 |        |
| 192.168.210.33:2379 | b5d9c4826815356e |   3.5.0 |  1.0 MB |      true |      false |        16 |       4867 |               4867 |        |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
#从以上信息可以看到，etcd已重新选举了LEADER，pg_node2节点已拒绝连接
#下面再次启动pg_node2节点的etcd，节点加入集群
[root@pg_node2 ~]# systemctl start etcd
[root@pg_node2 ~]# etcdctl endpoint status --endpoints='192.168.210.15:2379,192.168.210.81:2379,192.168.210.33:2379' -w table
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT       |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.210.15:2379 | 2baa1a77ec379977 |   3.5.0 |  1.0 MB |     false |      false |        16 |       4868 |               4868 |        |
| 192.168.210.81:2379 |  ff5595d67d21105 |   3.5.0 |  1.0 MB |     false |      false |        16 |       4868 |               4868 |        |
| 192.168.210.33:2379 | b5d9c4826815356e |   3.5.0 |  1.0 MB |      true |      false |        16 |       4868 |               4868 |        |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
[root@pg_node1 ~]# etcdctl endpoint status --endpoints='192.168.210.15:2379,192.168.210.81:2379,192.168.210.33:2379' -w table
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT       |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.210.15:2379 | 2baa1a77ec379977 |   3.5.0 |  1.0 MB |     false |      false |        16 |       4868 |               4868 |        |
| 192.168.210.81:2379 |  ff5595d67d21105 |   3.5.0 |  1.0 MB |     false |      false |        16 |       4868 |               4868 |        |
| 192.168.210.33:2379 | b5d9c4826815356e |   3.5.0 |  1.0 MB |      true |      false |        16 |       4868 |               4868 |        |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
[root@pg_node3 ~]# etcdctl endpoint status --endpoints='192.168.210.15:2379,192.168.210.81:2379,192.168.210.33:2379' -w table
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT       |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.210.15:2379 | 2baa1a77ec379977 |   3.5.0 |  1.0 MB |     false |      false |        16 |       4868 |               4868 |        |
| 192.168.210.81:2379 |  ff5595d67d21105 |   3.5.0 |  1.0 MB |     false |      false |        16 |       4868 |               4868 |        |
| 192.168.210.33:2379 | b5d9c4826815356e |   3.5.0 |  1.0 MB |      true |      false |        16 |       4868 |               4868 |        |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+

#模拟etcd节点出现问题，停掉任意2个etcd节点
[root@pg_node2 ~]# systemctl stop etcd
[root@pg_node2 ~]# etcdctl endpoint status --endpoints='192.168.210.15:2379,192.168.210.81:2379,192.168.210.33:2379' -w table
{"level":"warn","ts":"2021-06-23T09:20:00.826+0800","logger":"etcd-client","caller":"v3/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0xc0003fea80/#initially=[192.168.210.15:2379;192.168.210.81:2379;192.168.210.33:2379]","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = latest balancer error: last connection error: connection error: desc = \"transport: Error while dialing dial tcp 192.168.210.81:2379: connect: connection refused\""}
Failed to get the status of endpoint 192.168.210.81:2379 (context deadline exceeded)
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT       |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.210.15:2379 | 2baa1a77ec379977 |   3.5.0 |  1.0 MB |     false |      false |        16 |       4867 |               4867 |        |
| 192.168.210.33:2379 | b5d9c4826815356e |   3.5.0 |  1.0 MB |      true |      false |        16 |       4867 |               4867 |        |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
[root@pg_node3 ~]# systemctl stop etcd
[root@pg_node3 ~]# etcdctl endpoint status --endpoints='192.168.210.15:2379,192.168.210.81:2379,192.168.210.33:2379' -w table
{"level":"warn","ts":"2021-06-23T09:27:22.438+0800","logger":"etcd-client","caller":"v3/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0xc000150000/#initially=[192.168.210.15:2379;192.168.210.81:2379;192.168.210.33:2379]","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = latest balancer error: last connection error: connection error: desc = \"transport: Error while dialing dial tcp 192.168.210.81:2379: connect: connection refused\""}
Failed to get the status of endpoint 192.168.210.81:2379 (context deadline exceeded)
{"level":"warn","ts":"2021-06-23T09:27:27.439+0800","logger":"etcd-client","caller":"v3/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0xc000150000/#initially=[192.168.210.15:2379;192.168.210.81:2379;192.168.210.33:2379]","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = latest balancer error: last connection error: connection error: desc = \"transport: Error while dialing dial tcp 192.168.210.33:2379: connect: connection refused\""}
Failed to get the status of endpoint 192.168.210.33:2379 (context deadline exceeded)
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+-----------------------+
|      ENDPOINT       |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX |        ERRORS         |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+-----------------------+
| 192.168.210.15:2379 | 2baa1a77ec379977 |   3.5.0 |  1.0 MB |     false |      false |        17 |       4869 |               4869 | etcdserver: no leader |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+-----------------------+
#从以上信息可以看到，停止3个中任意两个节点，etcd集群不再可用
#启动后集群恢复正常，健壮性非常强
[root@pg_node3 ~]# systemctl start etcd
[root@pg_node3 ~]# etcdctl endpoint status --endpoints='192.168.210.15:2379,192.168.210.81:2379,192.168.210.33:2379' -w table
{"level":"warn","ts":"2021-06-23T09:28:31.642+0800","logger":"etcd-client","caller":"v3/retry_interceptor.go:62","msg":"retrying of unary invoker failed","target":"etcd-endpoints://0xc000150a80/#initially=[192.168.210.15:2379;192.168.210.81:2379;192.168.210.33:2379]","attempt":0,"error":"rpc error: code = DeadlineExceeded desc = latest balancer error: last connection error: connection error: desc = \"transport: Error while dialing dial tcp 192.168.210.81:2379: connect: connection refused\""}
Failed to get the status of endpoint 192.168.210.81:2379 (context deadline exceeded)
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT       |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.210.15:2379 | 2baa1a77ec379977 |   3.5.0 |  1.0 MB |      true |      false |        18 |       4883 |               4883 |        |
| 192.168.210.33:2379 | b5d9c4826815356e |   3.5.0 |  1.0 MB |     false |      false |        18 |       4883 |               4883 |        |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
[root@pg_node2 ~]# systemctl start etcd
[root@pg_node2 ~]# etcdctl endpoint status --endpoints='192.168.210.15:2379,192.168.210.81:2379,192.168.210.33:2379' -w table
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT       |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.210.15:2379 | 2baa1a77ec379977 |   3.5.0 |  1.0 MB |      true |      false |        18 |       4888 |               4888 |        |
| 192.168.210.81:2379 |  ff5595d67d21105 |   3.5.0 |  1.0 MB |     false |      false |        18 |       4888 |               4888 |        |
| 192.168.210.33:2379 | b5d9c4826815356e |   3.5.0 |  1.0 MB |     false |      false |        18 |       4888 |               4888 |        |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
[root@pg_node1 ~]# etcdctl endpoint status --endpoints='192.168.210.15:2379,192.168.210.81:2379,192.168.210.33:2379' -w table
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT       |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 192.168.210.15:2379 | 2baa1a77ec379977 |   3.5.0 |  1.0 MB |      true |      false |        19 |       4905 |               4905 |        |
| 192.168.210.81:2379 |  ff5595d67d21105 |   3.5.0 |  1.0 MB |     false |      false |        19 |       4905 |               4905 |        |
| 192.168.210.33:2379 | b5d9c4826815356e |   3.5.0 |  1.0 MB |     false |      false |        19 |       4905 |               4905 |        |
+---------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+

#模拟PostgreSQL数据库出现问题
#停止主库服务
[postgres@pg_node1 ~]$ pg_ctl stop -D /opt/PostgreSQL/13/data
waiting for server to shut down.... done
server stopped
[postgres@pg_node1 ~]$ ps -ef|grep postgres
postgres  4447  4367  0 09:37 pts/0    00:00:00 grep --color=auto postgres
[postgres@pg_node1 ~]$ ps -ef|grep postgres
postgres  4451  4367  0 09:37 pts/0    00:00:00 grep --color=auto postgres
[postgres@pg_node1 ~]$ ps -ef|grep postgres
postgres  4453  4367  0 09:37 pts/0    00:00:00 grep --color=auto postgres
[postgres@pg_node1 ~]$ ps -ef|grep postgres
postgres  4459     1  1 09:37 ?        00:00:00 /opt/PostgreSQL/13/bin/postgres -D /opt/PostgreSQL/13/data --config-file=/opt/PostgreSQL/13/data/postgresql.conf --listen_addresses=0.0.0.0 --max_worker_processes=8 --max_prepared_transactions=0 --wal_level=replica --track_commit_timestamp=off --max_locks_per_transaction=64 --port=5432 --max_replication_slots=10 --max_connections=100 --hot_standby=on --cluster_name=twpg --wal_log_hints=on --max_wal_senders=10
postgres  4462  4459  0 09:37 ?        00:00:00 postgres: twpg: checkpointer 
postgres  4463  4459  0 09:37 ?        00:00:00 postgres: twpg: background writer 
postgres  4464  4459  0 09:37 ?        00:00:00 postgres: twpg: stats collector 
postgres  4472  4459  0 09:37 ?        00:00:00 postgres: twpg: twsm postgres 127.0.0.1(40298) idle
postgres  4487  4459  0 09:37 ?        00:00:00 postgres: twpg: walwriter 
postgres  4488  4459  0 09:37 ?        00:00:00 postgres: twpg: autovacuum launcher 
postgres  4489  4459  0 09:37 ?        00:00:00 postgres: twpg: logical replication launcher 
postgres  4490  4459  0 09:37 ?        00:00:00 postgres: twpg: walsender repuser 192.168.210.33(52430) streaming 0/5000668
postgres  4491  4459  0 09:37 ?        00:00:00 postgres: twpg: walsender repuser 192.168.210.81(62184) streaming 0/5000668
#PostgreSQL服务被patroni自动拉起，未发生故障转移
[root@pg_node1 log]# patronictl -c /etc/patroni/patroni.yml list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Leader  | running |  6 |           |
| pg2    | 192.168.210.81:5432 | Replica | running |  6 |       0.0 |
| pg3    | 192.168.210.33:5432 | Replica | running |  6 |       0.0 |
+--------+---------------------+---------+---------+----+-----------+
#查看下刚刚停止PG服务后的patroni日志，patroni检测到PG服务关闭，会尝试把PG服务启动
Jun 23 09:37:45 pg_node1 patroni: 2021-06-23 09:37:45,395 INFO: no action.  i am the leader with the lock
Jun 23 09:37:47 pg_node1 patroni: 2021-06-23 09:37:47.040 CST [4012] LOG:  received fast shutdown request
Jun 23 09:37:47 pg_node1 patroni: 2021-06-23 09:37:47.047 CST [4012] LOG:  aborting any active transactions
Jun 23 09:37:47 pg_node1 patroni: 2021-06-23 09:37:47.047 CST [4056] FATAL:  terminating connection due to administrator command
Jun 23 09:37:47 pg_node1 patroni: 2021-06-23 09:37:47.048 CST [4012] LOG:  background worker "logical replication launcher" (PID 4258) exited with exit code 1
Jun 23 09:37:47 pg_node1 patroni: 2021-06-23 09:37:47.049 CST [4015] LOG:  shutting down
Jun 23 09:37:47 pg_node1 patroni: 2021-06-23 09:37:47.122 CST [4012] LOG:  database system is shut down
Jun 23 09:37:55 pg_node1 patroni: 2021-06-23 09:37:55,389 WARNING: Postgresql is not running.
Jun 23 09:37:55 pg_node1 patroni: 2021-06-23 09:37:55,390 INFO: Lock owner: pg1; I am pg1
Jun 23 09:37:55 pg_node1 patroni: 2021-06-23 09:37:55,404 INFO: pg_controldata:
Jun 23 09:37:55 pg_node1 patroni: Database system identifier: 6976142033405049133
Jun 23 09:37:55 pg_node1 patroni: pg_control last modified: Wed Jun 23 09:37:47 2021
Jun 23 09:37:55 pg_node1 patroni: Blocks per segment of large relation: 131072
Jun 23 09:37:55 pg_node1 patroni: Size of a large-object chunk: 2048
Jun 23 09:37:55 pg_node1 patroni: WAL block size: 8192
Jun 23 09:37:55 pg_node1 patroni: Latest checkpoint's oldestActiveXID: 0
Jun 23 09:37:55 pg_node1 patroni: Latest checkpoint's TimeLineID: 5
Jun 23 09:37:55 pg_node1 patroni: Bytes per WAL segment: 16777216
Jun 23 09:37:55 pg_node1 patroni: Fake LSN counter for unlogged rels: 0/3E8
Jun 23 09:37:55 pg_node1 patroni: max_connections setting: 100
Jun 23 09:37:55 pg_node1 patroni: Latest checkpoint location: 0/5000510
Jun 23 09:37:55 pg_node1 patroni: Float8 argument passing: by value
Jun 23 09:37:55 pg_node1 patroni: Minimum recovery ending location: 0/0
Jun 23 09:37:55 pg_node1 patroni: track_commit_timestamp setting: off
Jun 23 09:37:55 pg_node1 patroni: Latest checkpoint's newestCommitTsXid: 0
Jun 23 09:37:55 pg_node1 patroni: Latest checkpoint's NextMultiXactId: 1
Jun 23 09:37:55 pg_node1 patroni: Maximum size of a TOAST chunk: 1996
Jun 23 09:37:55 pg_node1 patroni: Maximum data alignment: 8
Jun 23 09:37:55 pg_node1 patroni: Date/time type storage: 64-bit integers
Jun 23 09:37:55 pg_node1 patroni: Database block size: 8192
Jun 23 09:37:55 pg_node1 patroni: Data page checksum version: 1
Jun 23 09:37:55 pg_node1 patroni: Time of latest checkpoint: Wed Jun 23 09:37:47 2021
Jun 23 09:37:55 pg_node1 patroni: wal_log_hints setting: on
Jun 23 09:37:55 pg_node1 patroni: Latest checkpoint's full_page_writes: on
Jun 23 09:37:55 pg_node1 patroni: End-of-backup record required: no
Jun 23 09:37:55 pg_node1 patroni: max_prepared_xacts setting: 0
Jun 23 09:37:55 pg_node1 patroni: Latest checkpoint's NextMultiOffset: 0
Jun 23 09:37:55 pg_node1 patroni: Backup start location: 0/0
Jun 23 09:37:55 pg_node1 patroni: Latest checkpoint's oldestMultiXid: 1
Jun 23 09:37:55 pg_node1 patroni: Mock authentication nonce: 020ac2d0808d3cf471a8d90e23263e8a317b31c5f7e494fc3fba551683e5c39b
Jun 23 09:37:55 pg_node1 patroni: Latest checkpoint's NextOID: 16385
Jun 23 09:37:55 pg_node1 patroni: Maximum columns in an index: 32
Jun 23 09:37:55 pg_node1 patroni: Latest checkpoint's oldestXID: 479
Jun 23 09:37:55 pg_node1 patroni: Catalog version number: 202007201
Jun 23 09:37:55 pg_node1 patroni: max_worker_processes setting: 8
Jun 23 09:37:55 pg_node1 patroni: Maximum length of identifiers: 64
Jun 23 09:37:55 pg_node1 patroni: Min recovery ending loc's timeline: 0
Jun 23 09:37:55 pg_node1 patroni: max_locks_per_xact setting: 64
Jun 23 09:37:55 pg_node1 patroni: max_wal_senders setting: 10
Jun 23 09:37:55 pg_node1 patroni: Latest checkpoint's NextXID: 0:488
Jun 23 09:37:55 pg_node1 patroni: Latest checkpoint's REDO location: 0/5000510
Jun 23 09:37:55 pg_node1 patroni: Backup end location: 0/0
Jun 23 09:37:55 pg_node1 patroni: Database cluster state: shut down
Jun 23 09:37:55 pg_node1 patroni: pg_control version number: 1300
Jun 23 09:37:55 pg_node1 patroni: wal_level setting: replica
Jun 23 09:37:55 pg_node1 patroni: Latest checkpoint's REDO WAL file: 000000050000000000000005
Jun 23 09:37:55 pg_node1 patroni: Latest checkpoint's oldestCommitTsXid: 0
Jun 23 09:37:55 pg_node1 patroni: Latest checkpoint's oldestXID's DB: 1
Jun 23 09:37:55 pg_node1 patroni: Latest checkpoint's oldestMulti's DB: 1
Jun 23 09:37:55 pg_node1 patroni: Latest checkpoint's PrevTimeLineID: 5
Jun 23 09:37:55 pg_node1 patroni: 2021-06-23 09:37:55,406 INFO: Lock owner: pg1; I am pg1
Jun 23 09:37:55 pg_node1 patroni: 2021-06-23 09:37:55,406 INFO: Lock owner: pg1; I am pg1
Jun 23 09:37:55 pg_node1 patroni: 2021-06-23 09:37:55,418 INFO: starting as readonly because i had the session lock
Jun 23 09:37:55 pg_node1 patroni: 2021-06-23 09:37:55,420 INFO: closed patroni connection to the postgresql cluster
Jun 23 09:37:55 pg_node1 patroni: 2021-06-23 09:37:55,452 INFO: postmaster pid=4459
Jun 23 09:37:55 pg_node1 patroni: localhost:5432 - no response
Jun 23 09:37:55 pg_node1 patroni: 2021-06-23 09:37:55.485 CST [4459] LOG:  starting PostgreSQL 13.3 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-44), 64-bit
Jun 23 09:37:55 pg_node1 patroni: 2021-06-23 09:37:55.485 CST [4459] LOG:  listening on IPv4 address "0.0.0.0", port 5432
Jun 23 09:37:55 pg_node1 patroni: 2021-06-23 09:37:55.508 CST [4459] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
Jun 23 09:37:55 pg_node1 patroni: 2021-06-23 09:37:55.524 CST [4461] LOG:  database system was shut down at 2021-06-23 09:37:47 CST
Jun 23 09:37:55 pg_node1 patroni: 2021-06-23 09:37:55.525 CST [4461] WARNING:  specified neither primary_conninfo nor restore_command
Jun 23 09:37:55 pg_node1 patroni: 2021-06-23 09:37:55.525 CST [4461] HINT:  The database server will regularly poll the pg_wal subdirectory to check for files placed there.
Jun 23 09:37:55 pg_node1 patroni: 2021-06-23 09:37:55.525 CST [4461] LOG:  entering standby mode
Jun 23 09:37:55 pg_node1 patroni: 2021-06-23 09:37:55.533 CST [4461] LOG:  consistent recovery state reached at 0/5000588
Jun 23 09:37:55 pg_node1 patroni: 2021-06-23 09:37:55.534 CST [4461] LOG:  invalid record length at 0/5000588: wanted 24, got 0
Jun 23 09:37:55 pg_node1 patroni: 2021-06-23 09:37:55.535 CST [4459] LOG:  database system is ready to accept read only connections
Jun 23 09:37:56 pg_node1 patroni: localhost:5432 - accepting connections
Jun 23 09:37:56 pg_node1 patroni: localhost:5432 - accepting connections
Jun 23 09:37:56 pg_node1 patroni: this is patroni callback on_role_change replica twpg
Jun 23 09:37:56 pg_node1 patroni: 2021-06-23 09:37:56,544 INFO: Lock owner: pg1; I am pg1
Jun 23 09:37:56 pg_node1 patroni: 2021-06-23 09:37:56,545 INFO: establishing a new patroni connection to the postgres cluster
Jun 23 09:37:56 pg_node1 systemd: Started Session c31 of user root.
Jun 23 09:37:56 pg_node1 patroni: 2021-06-23 09:37:56,570 INFO: Software Watchdog activated with 25 second timeout, timing slack 15 seconds
Jun 23 09:37:56 pg_node1 patroni: 2021-06-23 09:37:56,584 INFO: promoted self to leader because i had the session lock
Jun 23 09:37:56 pg_node1 patroni: 2021-06-23 09:37:56,586 INFO: Lock owner: pg1; I am pg1
Jun 23 09:37:56 pg_node1 patroni: server promoting
Jun 23 09:37:56 pg_node1 patroni: 2021-06-23 09:37:56.593 CST [4461] LOG:  received promote request
Jun 23 09:37:56 pg_node1 patroni: 2021-06-23 09:37:56.593 CST [4461] LOG:  redo is not required
Jun 23 09:37:56 pg_node1 patroni: 2021-06-23 09:37:56,593 INFO: cleared rewind state after becoming the leader
Jun 23 09:37:56 pg_node1 patroni: 2021-06-23 09:37:56.599 CST [4461] LOG:  selected new timeline ID: 6
Jun 23 09:37:56 pg_node1 patroni: this is patroni callback on_role_change master twpg
Jun 23 09:37:56 pg_node1 patroni: 2021-06-23 09:37:56,618 INFO: updated leader lock during promote
Jun 23 09:37:56 pg_node1 systemd: Started Session c32 of user root.
Jun 23 09:37:56 pg_node1 systemd: Started Session c33 of user root.
Jun 23 09:37:56 pg_node1 patroni: 2021-06-23 09:37:56.880 CST [4461] LOG:  archive recovery complete
Jun 23 09:37:56 pg_node1 patroni: 2021-06-23 09:37:56.911 CST [4459] LOG:  database system is ready to accept connections
Jun 23 09:37:57 pg_node1 patroni: 2021-06-23 09:37:57,638 INFO: Lock owner: pg1; I am pg1
Jun 23 09:37:57 pg_node1 patroni: 2021-06-23 09:37:57,724 INFO: no action.  i am the leader with the lock

#模拟PostgreSQL数据库出现问题
#停止备库服务
[postgres@pg_node2 ~]$ pg_ctl stop -D /opt/PostgreSQL/13/data
waiting for server to shut down.... done
server stopped
[postgres@pg_node2 ~]$ ps -ef|grep postgres
postgres 26277 26180  0 09:51 pts/0    00:00:00 grep --color=auto postgres
[postgres@pg_node2 ~]$ ps -ef|grep postgres
postgres 26284     1  2 09:51 ?        00:00:00 /opt/PostgreSQL/13/bin/postgres -D /opt/PostgreSQL/13/data --config-file=/opt/PostgreSQL/13/data/postgresql.conf --listen_addresses=0.0.0.0 --max_worker_processes=8 --max_prepared_transactions=0 --wal_level=replica --track_commit_timestamp=off --max_locks_per_transaction=64 --port=5432 --max_replication_slots=10 --max_connections=100 --hot_standby=on --cluster_name=twpg --wal_log_hints=on --max_wal_senders=10
postgres 26286 26284  0 09:51 ?        00:00:00 postgres: twpg: startup recovering 000000060000000000000005
postgres 26287 26284  0 09:51 ?        00:00:00 postgres: twpg: checkpointer 
postgres 26288 26284  0 09:51 ?        00:00:00 postgres: twpg: background writer 
postgres 26289 26284  0 09:51 ?        00:00:00 postgres: twpg: stats collector 
postgres 26290 26284  0 09:51 ?        00:00:00 postgres: twpg: walreceiver 
postgres 26293 26180  0 09:51 pts/0    00:00:00 grep --color=auto postgres
#查看下刚刚停止PG服务后的patroni日志，patroni检测到PG服务关闭，会尝试把PG服务启动
Jun 23 09:51:29 pg_node2 patroni: 2021-06-23 09:51:29,043 INFO: does not have lock
Jun 23 09:51:29 pg_node2 patroni: 2021-06-23 09:51:29,045 INFO: no action.  i am a secondary and i am following a leader
Jun 23 09:51:37 pg_node2 patroni: 2021-06-23 09:51:37.498 CST [26228] LOG:  received fast shutdown request
Jun 23 09:51:37 pg_node2 patroni: 2021-06-23 09:51:37.505 CST [26228] LOG:  aborting any active transactions
Jun 23 09:51:37 pg_node2 patroni: 2021-06-23 09:51:37.505 CST [26234] FATAL:  terminating walreceiver process due to administrator command
Jun 23 09:51:37 pg_node2 patroni: 2021-06-23 09:51:37.505 CST [26241] FATAL:  terminating connection due to administrator command
Jun 23 09:51:37 pg_node2 patroni: 2021-06-23 09:51:37.507 CST [26231] LOG:  shutting down
Jun 23 09:51:37 pg_node2 patroni: 2021-06-23 09:51:37.522 CST [26228] LOG:  database system is shut down
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39,042 WARNING: Postgresql is not running.
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39,042 INFO: Lock owner: pg1; I am pg2
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39,053 INFO: pg_controldata:
Jun 23 09:51:39 pg_node2 patroni: Database system identifier: 6976142033405049133
Jun 23 09:51:39 pg_node2 patroni: pg_control last modified: Wed Jun 23 09:51:37 2021
Jun 23 09:51:39 pg_node2 patroni: Blocks per segment of large relation: 131072
Jun 23 09:51:39 pg_node2 patroni: Size of a large-object chunk: 2048
Jun 23 09:51:39 pg_node2 patroni: WAL block size: 8192
Jun 23 09:51:39 pg_node2 patroni: Latest checkpoint's oldestActiveXID: 488
Jun 23 09:51:39 pg_node2 patroni: Latest checkpoint's TimeLineID: 6
Jun 23 09:51:39 pg_node2 patroni: Bytes per WAL segment: 16777216
Jun 23 09:51:39 pg_node2 patroni: Fake LSN counter for unlogged rels: 0/3E8
Jun 23 09:51:39 pg_node2 patroni: max_connections setting: 100
Jun 23 09:51:39 pg_node2 patroni: Latest checkpoint location: 0/50006A0
Jun 23 09:51:39 pg_node2 patroni: Float8 argument passing: by value
Jun 23 09:51:39 pg_node2 patroni: Minimum recovery ending location: 0/5000750
Jun 23 09:51:39 pg_node2 patroni: track_commit_timestamp setting: off
Jun 23 09:51:39 pg_node2 patroni: Latest checkpoint's newestCommitTsXid: 0
Jun 23 09:51:39 pg_node2 patroni: Latest checkpoint's NextMultiXactId: 1
Jun 23 09:51:39 pg_node2 patroni: Maximum size of a TOAST chunk: 1996
Jun 23 09:51:39 pg_node2 patroni: Maximum data alignment: 8
Jun 23 09:51:39 pg_node2 patroni: Date/time type storage: 64-bit integers
Jun 23 09:51:39 pg_node2 patroni: Database block size: 8192
Jun 23 09:51:39 pg_node2 patroni: Data page checksum version: 1
Jun 23 09:51:39 pg_node2 patroni: Time of latest checkpoint: Wed Jun 23 09:37:57 2021
Jun 23 09:51:39 pg_node2 patroni: wal_log_hints setting: on
Jun 23 09:51:39 pg_node2 patroni: Latest checkpoint's full_page_writes: on
Jun 23 09:51:39 pg_node2 patroni: End-of-backup record required: no
Jun 23 09:51:39 pg_node2 patroni: max_prepared_xacts setting: 0
Jun 23 09:51:39 pg_node2 patroni: Latest checkpoint's NextMultiOffset: 0
Jun 23 09:51:39 pg_node2 patroni: Backup start location: 0/0
Jun 23 09:51:39 pg_node2 patroni: Latest checkpoint's oldestMultiXid: 1
Jun 23 09:51:39 pg_node2 patroni: Mock authentication nonce: 020ac2d0808d3cf471a8d90e23263e8a317b31c5f7e494fc3fba551683e5c39b
Jun 23 09:51:39 pg_node2 patroni: Latest checkpoint's NextOID: 16385
Jun 23 09:51:39 pg_node2 patroni: Maximum columns in an index: 32
Jun 23 09:51:39 pg_node2 patroni: Latest checkpoint's oldestXID: 479
Jun 23 09:51:39 pg_node2 patroni: Catalog version number: 202007201
Jun 23 09:51:39 pg_node2 patroni: max_worker_processes setting: 8
Jun 23 09:51:39 pg_node2 patroni: Maximum length of identifiers: 64
Jun 23 09:51:39 pg_node2 patroni: Min recovery ending loc's timeline: 6
Jun 23 09:51:39 pg_node2 patroni: max_locks_per_xact setting: 64
Jun 23 09:51:39 pg_node2 patroni: max_wal_senders setting: 10
Jun 23 09:51:39 pg_node2 patroni: Latest checkpoint's NextXID: 0:488
Jun 23 09:51:39 pg_node2 patroni: Latest checkpoint's REDO location: 0/5000668
Jun 23 09:51:39 pg_node2 patroni: Backup end location: 0/0
Jun 23 09:51:39 pg_node2 patroni: Database cluster state: shut down in recovery
Jun 23 09:51:39 pg_node2 patroni: pg_control version number: 1300
Jun 23 09:51:39 pg_node2 patroni: wal_level setting: replica
Jun 23 09:51:39 pg_node2 patroni: Latest checkpoint's REDO WAL file: 000000060000000000000005
Jun 23 09:51:39 pg_node2 patroni: Latest checkpoint's oldestCommitTsXid: 0
Jun 23 09:51:39 pg_node2 patroni: Latest checkpoint's oldestXID's DB: 1
Jun 23 09:51:39 pg_node2 patroni: Latest checkpoint's oldestMulti's DB: 1
Jun 23 09:51:39 pg_node2 patroni: Latest checkpoint's PrevTimeLineID: 6
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39,055 INFO: Lock owner: pg1; I am pg2
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39,081 INFO: Local timeline=6 lsn=0/5000750
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39,088 INFO: master_timeline=6
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39,089 INFO: Lock owner: pg1; I am pg2
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39,105 INFO: starting as a secondary
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39,107 INFO: closed patroni connection to the postgresql cluster
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39,130 INFO: postmaster pid=26284
Jun 23 09:51:39 pg_node2 patroni: localhost:5432 - no response
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39.172 CST [26284] LOG:  starting PostgreSQL 13.3 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-44), 64-bit
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39.172 CST [26284] LOG:  listening on IPv4 address "0.0.0.0", port 5432
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39.186 CST [26284] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39.203 CST [26286] LOG:  database system was shut down in recovery at 2021-06-23 09:51:37 CST
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39.203 CST [26286] LOG:  entering standby mode
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39.212 CST [26286] LOG:  redo starts at 0/5000668
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39.212 CST [26286] LOG:  consistent recovery state reached at 0/5000750
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39.212 CST [26286] LOG:  invalid record length at 0/5000750: wanted 24, got 0
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39.213 CST [26284] LOG:  database system is ready to accept read only connections
Jun 23 09:51:39 pg_node2 patroni: 2021-06-23 09:51:39.229 CST [26290] LOG:  started streaming WAL from primary at 0/5000000 on timeline 6
Jun 23 09:51:40 pg_node2 patroni: localhost:5432 - accepting connections
Jun 23 09:51:40 pg_node2 patroni: localhost:5432 - accepting connections
Jun 23 09:51:40 pg_node2 patroni: this is patroni callback on_start replica twpg
Jun 23 09:51:40 pg_node2 patroni: 2021-06-23 09:51:40,229 INFO: Lock owner: pg1; I am pg2
Jun 23 09:51:40 pg_node2 patroni: 2021-06-23 09:51:40,229 INFO: does not have lock
Jun 23 09:51:40 pg_node2 patroni: 2021-06-23 09:51:40,229 INFO: establishing a new patroni connection to the postgres cluster
Jun 23 09:51:40 pg_node2 patroni: 2021-06-23 09:51:40,261 INFO: no action.  i am a secondary and i am following a leader
Jun 23 09:51:50 pg_node2 patroni: 2021-06-23 09:51:50,228 INFO: Lock owner: pg1; I am pg2
Jun 23 09:51:50 pg_node2 patroni: 2021-06-23 09:51:50,228 INFO: does not have lock
Jun 23 09:51:50 pg_node2 patroni: 2021-06-23 09:51:50,238 INFO: no action.  i am a secondary and i am following a leader

#模拟patroni出现问题
#停止PG主机所在节点的patroni服务
[root@pg_node1 ~]# patronictl -c /etc/patroni/patroni.yml list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Leader  | running |  6 |           |
| pg2    | 192.168.210.81:5432 | Replica | running |  6 |       0.0 |
| pg3    | 192.168.210.33:5432 | Replica | running |  6 |       0.0 |
+--------+---------------------+---------+---------+----+-----------+
[root@pg_node1 ~]# ip -o -4 a
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
2: eth0    inet 192.168.210.15/24 brd 192.168.210.255 scope global noprefixroute dynamic eth0\       valid_lft 63429sec preferred_lft 63429sec
2: eth0    inet 192.168.210.66/24 scope global secondary eth0\       valid_lft forever preferred_lft forever
[root@pg_node1 ~]# systemctl stop patroni
[root@pg_node1 ~]# ip -o -4 a
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
2: eth0    inet 192.168.210.15/24 brd 192.168.210.255 scope global noprefixroute dynamic eth0\       valid_lft 63357sec preferred_lft 63357sec
[root@pg_node2 ~]# patronictl -c /etc/patroni/patroni.yml list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Leader  | running |  6 |           |
| pg2    | 192.168.210.81:5432 | Replica | running |  6 |       0.0 |
| pg3    | 192.168.210.33:5432 | Replica | running |  6 |       0.0 |
+--------+---------------------+---------+---------+----+-----------+
[root@pg_node2 ~]# patronictl -c /etc/patroni/patroni.yml list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Replica | stopped |    |   unknown |
| pg2    | 192.168.210.81:5432 | Replica | running |  7 |       0.0 |
| pg3    | 192.168.210.33:5432 | Leader  | running |  7 |           |
+--------+---------------------+---------+---------+----+-----------+
[root@pg_node2 ~]# patronictl -c /etc/patroni/patroni.yml list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg2    | 192.168.210.81:5432 | Replica | running |  7 |       0.0 |
| pg3    | 192.168.210.33:5432 | Leader  | running |  7 |           |
+--------+---------------------+---------+---------+----+-----------+
[root@pg_node3 ~]# patronictl -c /etc/patroni/patroni.yml list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Leader  | running |  6 |           |
| pg2    | 192.168.210.81:5432 | Replica | running |  6 |       0.0 |
| pg3    | 192.168.210.33:5432 | Replica | running |  6 |       0.0 |
+--------+---------------------+---------+---------+----+-----------+
[root@pg_node3 ~]# patronictl -c /etc/patroni/patroni.yml list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg2    | 192.168.210.81:5432 | Replica | running |  7 |       0.0 |
| pg3    | 192.168.210.33:5432 | Leader  | running |  7 |           |
+--------+---------------------+---------+---------+----+-----------+
[root@pg_node3 ~]# ip -o -4 a
1: lo    inet 127.0.0.1/8 scope host lo\       valid_lft forever preferred_lft forever
2: eth0    inet 192.168.210.33/24 brd 192.168.210.255 scope global noprefixroute dynamic eth0\       valid_lft 69980sec preferred_lft 69980sec
2: eth0    inet 192.168.210.66/24 scope global secondary eth0\       valid_lft forever preferred_lft forever
#节点3日志信息，数据库提升为主库
Jun 23 09:59:07 pg_node3 patroni: 2021-06-23 09:59:07,841 INFO: Lock owner: pg1; I am pg3
Jun 23 09:59:07 pg_node3 patroni: 2021-06-23 09:59:07,841 INFO: does not have lock
Jun 23 09:59:07 pg_node3 patroni: 2021-06-23 09:59:07,849 INFO: no action.  i am a secondary and i am following a leader
Jun 23 09:59:12 pg_node3 patroni: 2021-06-23 09:59:12.171 CST [25279] LOG:  replication terminated by primary server
Jun 23 09:59:12 pg_node3 patroni: 2021-06-23 09:59:12.171 CST [25279] DETAIL:  End of WAL reached on timeline 6 at 0/50007C8.
Jun 23 09:59:12 pg_node3 patroni: 2021-06-23 09:59:12.171 CST [25279] FATAL:  could not send end-of-streaming message to primary: no COPY in progress
Jun 23 09:59:12 pg_node3 patroni: 2021-06-23 09:59:12.172 CST [32443] LOG:  invalid record length at 0/50007C8: wanted 24, got 0
Jun 23 09:59:12 pg_node3 patroni: 2021-06-23 09:59:12.179 CST [26335] FATAL:  could not connect to the primary server: could not connect to server: Connection refused
Jun 23 09:59:12 pg_node3 patroni: Is the server running on host "192.168.210.15" and accepting
Jun 23 09:59:12 pg_node3 patroni: TCP/IP connections on port 5432?
Jun 23 09:59:13 pg_node3 patroni: 2021-06-23 09:59:13,151 WARNING: Request failed to pg1: GET http://192.168.210.15:8008/patroni (HTTPConnectionPool(host=u'192.168.210.15', port=8008): Max retries exceeded with url: /patroni (Caused by ProtocolError('Connection aborted.', error(104, 'Connection reset by peer'))))
Jun 23 09:59:13 pg_node3 patroni: 2021-06-23 09:59:13,153 INFO: Got response from pg2 http://192.168.210.81:8008/patroni: {"database_system_identifier": "6976142033405049133", "postmaster_start_time": "2021-06-23 09:51:39.194 CST", "timeline": 6, "cluster_unlocked": false, "patroni": {"scope": "twpg", "version": "2.0.2"}, "state": "running", "role": "replica", "xlog": {"received_location": 83888072, "replayed_timestamp": null, "paused": false, "replayed_location": 83888072}, "server_version": 130003}
Jun 23 09:59:13 pg_node3 patroni: 2021-06-23 09:59:13,247 INFO: Software Watchdog activated with 25 second timeout, timing slack 15 seconds
Jun 23 09:59:13 pg_node3 patroni: 2021-06-23 09:59:13,264 INFO: promoted self to leader by acquiring session lock
Jun 23 09:59:13 pg_node3 patroni: server promoting
Jun 23 09:59:13 pg_node3 patroni: 2021-06-23 09:59:13,273 INFO: Lock owner: pg3; I am pg3
Jun 23 09:59:13 pg_node3 patroni: 2021-06-23 09:59:13.276 CST [32443] LOG:  received promote request
Jun 23 09:59:13 pg_node3 patroni: 2021-06-23 09:59:13.276 CST [32443] LOG:  redo done at 0/5000750
Jun 23 09:59:13 pg_node3 patroni: 2021-06-23 09:59:13,277 INFO: cleared rewind state after becoming the leader
Jun 23 09:59:13 pg_node3 patroni: 2021-06-23 09:59:13.289 CST [32443] LOG:  selected new timeline ID: 7
Jun 23 09:59:13 pg_node3 patroni: this is patroni callback on_role_change master twpg
Jun 23 09:59:13 pg_node3 patroni: 2021-06-23 09:59:13,298 INFO: updated leader lock during promote
Jun 23 09:59:13 pg_node3 systemd: Started Session c7 of user root.
Jun 23 09:59:13 pg_node3 systemd: Started Session c8 of user root.
Jun 23 09:59:13 pg_node3 patroni: 2021-06-23 09:59:13.585 CST [32443] LOG:  archive recovery complete
Jun 23 09:59:13 pg_node3 patroni: 2021-06-23 09:59:13.650 CST [32441] LOG:  database system is ready to accept connections
Jun 23 09:59:14 pg_node3 patroni: 2021-06-23 09:59:14,318 INFO: Lock owner: pg3; I am pg3
Jun 23 09:59:14 pg_node3 patroni: 2021-06-23 09:59:14,435 INFO: no action.  i am the leader with the lock
#
Jun 23 09:59:10 pg_node2 patroni: 2021-06-23 09:59:10,228 INFO: Lock owner: pg1; I am pg2
Jun 23 09:59:10 pg_node2 patroni: 2021-06-23 09:59:10,228 INFO: does not have lock
Jun 23 09:59:10 pg_node2 patroni: 2021-06-23 09:59:10,237 INFO: no action.  i am a secondary and i am following a leader
Jun 23 09:59:12 pg_node2 patroni: 2021-06-23 09:59:12.170 CST [26290] LOG:  replication terminated by primary server
Jun 23 09:59:12 pg_node2 patroni: 2021-06-23 09:59:12.170 CST [26290] DETAIL:  End of WAL reached on timeline 6 at 0/50007C8.
Jun 23 09:59:12 pg_node2 patroni: 2021-06-23 09:59:12.170 CST [26290] FATAL:  could not send end-of-streaming message to primary: no COPY in progress
Jun 23 09:59:12 pg_node2 patroni: 2021-06-23 09:59:12.170 CST [26286] LOG:  invalid record length at 0/50007C8: wanted 24, got 0
Jun 23 09:59:12 pg_node2 patroni: 2021-06-23 09:59:12.177 CST [26671] FATAL:  could not connect to the primary server: could not connect to server: Connection refused
Jun 23 09:59:12 pg_node2 patroni: Is the server running on host "192.168.210.15" and accepting
Jun 23 09:59:12 pg_node2 patroni: TCP/IP connections on port 5432?
Jun 23 09:59:13 pg_node2 patroni: 2021-06-23 09:59:13,174 WARNING: Request failed to pg1: GET http://192.168.210.15:8008/patroni (HTTPConnectionPool(host=u'192.168.210.15', port=8008): Max retries exceeded with url: /patroni (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7f1b110fb9d0>: Failed to establish a new connection: [Errno 111] Connection refused',)))
Jun 23 09:59:13 pg_node2 patroni: 2021-06-23 09:59:13,188 INFO: Got response from pg3 http://192.168.210.33:8008/patroni: {"database_system_identifier": "6976142033405049133", "postmaster_start_time": "2021-06-21 15:43:32.837 CST", "timeline": 6, "cluster_unlocked": true, "patroni": {"scope": "twpg", "version": "2.0.2"}, "state": "running", "role": "replica", "xlog": {"received_location": 83888072, "replayed_timestamp": null, "paused": false, "replayed_location": 83888072}, "server_version": 130003}
Jun 23 09:59:13 pg_node2 patroni: 2021-06-23 09:59:13,277 INFO: Could not take out TTL lock
Jun 23 09:59:13 pg_node2 patroni: server signaled
Jun 23 09:59:13 pg_node2 patroni: 2021-06-23 09:59:13.291 CST [26284] LOG:  received SIGHUP, reloading configuration files
Jun 23 09:59:13 pg_node2 patroni: 2021-06-23 09:59:13.292 CST [26284] LOG:  parameter "primary_conninfo" changed to "user=repuser passfile=/home/postgres/pgpass host=192.168.210.33 port=5432 sslmode=prefer application_name=pg2 gssencmode=prefer channel_binding=prefer"
Jun 23 09:59:13 pg_node2 patroni: 2021-06-23 09:59:13.296 CST [26682] FATAL:  could not connect to the primary server: could not connect to server: Connection refused
Jun 23 09:59:13 pg_node2 patroni: Is the server running on host "192.168.210.15" and accepting
Jun 23 09:59:13 pg_node2 patroni: TCP/IP connections on port 5432?
Jun 23 09:59:13 pg_node2 patroni: 2021-06-23 09:59:13,330 INFO: following new leader after trying and failing to obtain lock
Jun 23 09:59:13 pg_node2 patroni: 2021-06-23 09:59:13,333 INFO: Lock owner: pg3; I am pg2
Jun 23 09:59:13 pg_node2 patroni: 2021-06-23 09:59:13,333 INFO: does not have lock
Jun 23 09:59:13 pg_node2 patroni: 2021-06-23 09:59:13,342 INFO: Local timeline=6 lsn=0/50007C8
Jun 23 09:59:13 pg_node2 patroni: 2021-06-23 09:59:13,349 INFO: master_timeline=6
Jun 23 09:59:13 pg_node2 patroni: 2021-06-23 09:59:13,354 INFO: no action.  i am a secondary and i am following a leader
Jun 23 09:59:14 pg_node2 patroni: 2021-06-23 09:59:14,373 INFO: Lock owner: pg3; I am pg2
Jun 23 09:59:14 pg_node2 patroni: 2021-06-23 09:59:14,373 INFO: does not have lock
Jun 23 09:59:14 pg_node2 patroni: 2021-06-23 09:59:14,375 INFO: no action.  i am a secondary and i am following a leader
Jun 23 09:59:14 pg_node2 patroni: 2021-06-23 09:59:14,491 INFO: Lock owner: pg3; I am pg2
Jun 23 09:59:14 pg_node2 patroni: 2021-06-23 09:59:14,492 INFO: does not have lock
Jun 23 09:59:14 pg_node2 patroni: 2021-06-23 09:59:14,500 INFO: no action.  i am a secondary and i am following a leader
Jun 23 09:59:18 pg_node2 patroni: 2021-06-23 09:59:18.306 CST [26690] LOG:  fetching timeline history file for timeline 7 from primary server
Jun 23 09:59:18 pg_node2 patroni: 2021-06-23 09:59:18.320 CST [26690] LOG:  started streaming WAL from primary at 0/5000000 on timeline 6
Jun 23 09:59:18 pg_node2 patroni: 2021-06-23 09:59:18.320 CST [26690] LOG:  replication terminated by primary server
Jun 23 09:59:18 pg_node2 patroni: 2021-06-23 09:59:18.320 CST [26690] DETAIL:  End of WAL reached on timeline 6 at 0/50007C8.
Jun 23 09:59:18 pg_node2 patroni: 2021-06-23 09:59:18.321 CST [26286] LOG:  new target timeline is 7
Jun 23 09:59:18 pg_node2 patroni: 2021-06-23 09:59:18.322 CST [26690] LOG:  restarted WAL streaming at 0/5000000 on timeline 7
Jun 23 09:59:24 pg_node2 patroni: 2021-06-23 09:59:24,492 INFO: Lock owner: pg3; I am pg2
Jun 23 09:59:24 pg_node2 patroni: 2021-06-23 09:59:24,492 INFO: does not have lock
Jun 23 09:59:24 pg_node2 patroni: 2021-06-23 09:59:24,513 INFO: no action.  i am a secondary and i am following a leader
#从以上信息可以看到，当主库所在的patroni服务不可用时，会发生故障转移（pg3选举为了主库，VIP进行漂移）

#模拟主库所在的patroni进程无响应
#kill掉patroni进程，会导致服务器reboot
[root@pg_node2 ~]# patronictl -c /etc/patroni/patroni.yml list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Replica | running | 10 |       0.0 |
| pg2    | 192.168.210.81:5432 | Leader  | running | 10 |           |
| pg3    | 192.168.210.33:5432 | Replica | running | 10 |       0.0 |
+--------+---------------------+---------+---------+----+-----------+
kill -9 `pgrep patroni`
[root@pg_node2 ~]# uptime
 14:26:01 up 1 min,  1 user,  load average: 1.35, 0.41, 0.14
[root@pg_node2 ~]# patronictl -c /etc/patroni/patroni.yml list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Replica | running | 10 |       0.0 |
| pg2    | 192.168.210.81:5432 | Replica | running | 10 |       0.0 |
| pg3    | 192.168.210.33:5432 | Leader  | running | 10 |           |
+--------+---------------------+---------+---------+----+-----------+

#模拟主机不可用（主库所在主机）
#立刻关机：poweroff
[root@pg_node3 ~]# uptime
 14:38:38 up 4 days, 22:08,  3 users,  load average: 0.03, 0.09, 0.07
[root@pg_node3 ~]# patronictl -c /etc/patroni/patroni.yml list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Replica | running | 10 |       0.0 |
| pg2    | 192.168.210.81:5432 | Replica | running | 10 |       0.0 |
| pg3    | 192.168.210.33:5432 | Leader  | running | 10 |           |
+--------+---------------------+---------+---------+----+-----------+
[root@pg_node3 ~]# poweroff
[root@pg_node2 ~]# patronictl -c /etc/patroni/patroni.yml list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Leader  | running | 11 |           |
| pg2    | 192.168.210.81:5432 | Replica | running | 11 |       0.0 |
| pg3    | 192.168.210.33:5432 | Replica | stopped |    |   unknown |
+--------+---------------------+---------+---------+----+-----------+
[root@pg_node2 ~]# patronictl -c /etc/patroni/patroni.yml list
2021-06-23 14:42:14,614 - ERROR - Request to server http://192.168.210.33:2379 failed: MaxRetryError("HTTPConnectionPool(host=u'192.168.210.33', port=2379): Max retries exceeded with url: /v3/kv/range (Caused by ConnectTimeoutError(<urllib3.connection.HTTPConnection object at 0x7feec6e0d9d0>, u'Connection to 192.168.210.33 timed out. (connect timeout=1.25)'))",)
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Leader  | running | 11 |           |
| pg2    | 192.168.210.81:5432 | Replica | running | 11 |       0.0 |
| pg3    | 192.168.210.33:5432 | Replica | stopped |    |   unknown |
+--------+---------------------+---------+---------+----+-----------+
[root@pg_node2 ~]# patronictl -c /etc/patroni/patroni.yml list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Leader  | running | 11 |           |
| pg2    | 192.168.210.81:5432 | Replica | running | 11 |       0.0 |
+--------+---------------------+---------+---------+----+-----------+
#再次启动pg3节点
[root@pg_node3 ~]# uptime
 14:46:14 up 0 min,  1 user,  load average: 0.71, 0.16, 0.05
[root@pg_node3 ~]# patronictl -c /etc/patroni/patroni.yml list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Leader  | running | 11 |           |
| pg2    | 192.168.210.81:5432 | Replica | running | 11 |       0.0 |
| pg3    | 192.168.210.33:5432 | Replica | running | 10 |       0.0 |
+--------+---------------------+---------+---------+----+-----------+

#手工switchover
[root@pg_node1 ~]# patronictl switchover
Master [pg1]: 
Candidate ['pg2', 'pg3'] []: ^CAborted!
[root@pg_node1 ~]# patronictl switchover
Master [pg1]: 
Candidate ['pg2', 'pg3'] []: pg2
When should the switchover take place (e.g. 2021-06-23T16:12 )  [now]: 
Current cluster topology
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Leader  | running | 11 |           |
| pg2    | 192.168.210.81:5432 | Replica | running | 11 |       0.0 |
| pg3    | 192.168.210.33:5432 | Replica | running | 11 |       0.0 |
+--------+---------------------+---------+---------+----+-----------+
Are you sure you want to switchover cluster twpg, demoting current master pg1? [y/N]: y  
2021-06-23 15:13:26.97922 Successfully switched over to "pg2"
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Replica | stopped |    |   unknown |
| pg2    | 192.168.210.81:5432 | Leader  | running | 11 |           |
| pg3    | 192.168.210.33:5432 | Replica | running | 11 |       0.0 |
+--------+---------------------+---------+---------+----+-----------+
[root@pg_node1 ~]# patronictl list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Replica | stopped |    |   unknown |
| pg2    | 192.168.210.81:5432 | Leader  | running | 12 |           |
| pg3    | 192.168.210.33:5432 | Replica | running | 11 |       0.0 |
+--------+---------------------+---------+---------+----+-----------+
[root@pg_node1 ~]# patronictl list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Replica | running | 12 |       0.0 |
| pg2    | 192.168.210.81:5432 | Leader  | running | 12 |           |
| pg3    | 192.168.210.33:5432 | Replica | running | 12 |       0.0 |
+--------+---------------------+---------+---------+----+-----------+
#日志信息如下：
Jun 23 15:13:16 pg_node1 patroni: 2021-06-23 15:13:16,311 INFO: Lock owner: pg1; I am pg1
Jun 23 15:13:16 pg_node1 patroni: 2021-06-23 15:13:16,316 INFO: no action.  i am the leader with the lock
Jun 23 15:13:24 pg_node1 patroni: 2021-06-23 15:13:24,854 INFO: received switchover request with leader=pg1 candidate=pg2 scheduled_at=None
Jun 23 15:13:24 pg_node1 patroni: 2021-06-23 15:13:24,879 INFO: Got response from pg2 http://192.168.210.81:8008/patroni: {"database_system_identifier": "6976142033405049133", "postmaster_start_time": "2021-06-23 14:25:40.252 CST", "timeline": 11, "cluster_unlocked": false, "patroni": {"scope": "twpg", "version": "2.0.2"}, "state": "running", "role": "replica", "xlog": {"received_location": 84186784, "replayed_timestamp": "2021-06-23 14:25:40.045 CST", "paused": false, "replayed_location": 84186784}, "server_version": 130003}
Jun 23 15:13:25 pg_node1 patroni: 2021-06-23 15:13:25,011 INFO: Lock owner: pg1; I am pg1
Jun 23 15:13:25 pg_node1 patroni: 2021-06-23 15:13:25,031 INFO: Got response from pg2 http://192.168.210.81:8008/patroni: {"database_system_identifier": "6976142033405049133", "postmaster_start_time": "2021-06-23 14:25:40.252 CST", "timeline": 11, "cluster_unlocked": false, "patroni": {"scope": "twpg", "version": "2.0.2"}, "state": "running", "role": "replica", "xlog": {"received_location": 84186784, "replayed_timestamp": "2021-06-23 14:25:40.045 CST", "paused": false, "replayed_location": 84186784}, "server_version": 130003}
Jun 23 15:13:25 pg_node1 patroni: 2021-06-23 15:13:25,134 INFO: manual failover: demoting myself
Jun 23 15:13:25 pg_node1 patroni: 2021-06-23 15:13:25.222 CST [4256] LOG:  received fast shutdown request
Jun 23 15:13:25 pg_node1 patroni: 2021-06-23 15:13:25.234 CST [4256] LOG:  aborting any active transactions
Jun 23 15:13:25 pg_node1 patroni: 2021-06-23 15:13:25.234 CST [4270] FATAL:  terminating connection due to administrator command
Jun 23 15:13:25 pg_node1 patroni: 2021-06-23 15:13:25.235 CST [4256] LOG:  background worker "logical replication launcher" (PID 4522) exited with exit code 1
Jun 23 15:13:25 pg_node1 patroni: 2021-06-23 15:13:25.237 CST [4259] LOG:  shutting down
Jun 23 15:13:25 pg_node1 patroni: 2021-06-23 15:13:25.306 CST [4256] LOG:  database system is shut down
Jun 23 15:13:26 pg_node1 patroni: this is patroni callback on_stop master twpg
Jun 23 15:13:26 pg_node1 systemd: Started Session c5 of user root.
Jun 23 15:13:26 pg_node1 patroni: 2021-06-23 15:13:26,275 INFO: Leader key released
Jun 23 15:13:26 pg_node1 patroni: 2021-06-23 15:13:26,277 INFO: Lock owner: None; I am pg1
Jun 23 15:13:26 pg_node1 patroni: 2021-06-23 15:13:26,277 INFO: not healthy enough for leader race
Jun 23 15:13:26 pg_node1 patroni: 2021-06-23 15:13:26,277 INFO: manual failover: demote in progress
Jun 23 15:13:26 pg_node1 patroni: 2021-06-23 15:13:26,278 INFO: Lock owner: None; I am pg1
Jun 23 15:13:26 pg_node1 patroni: 2021-06-23 15:13:26,279 INFO: not healthy enough for leader race
Jun 23 15:13:26 pg_node1 patroni: 2021-06-23 15:13:26,279 INFO: manual failover: demote in progress
Jun 23 15:13:26 pg_node1 patroni: 2021-06-23 15:13:26,296 INFO: Lock owner: pg2; I am pg1
Jun 23 15:13:26 pg_node1 patroni: 2021-06-23 15:13:26,297 INFO: manual failover: demote in progress
Jun 23 15:13:27 pg_node1 patroni: 2021-06-23 15:13:27,416 INFO: Lock owner: pg2; I am pg1
Jun 23 15:13:27 pg_node1 patroni: 2021-06-23 15:13:27,417 INFO: manual failover: demote in progress
Jun 23 15:13:27 pg_node1 patroni: 2021-06-23 15:13:27,532 INFO: Lock owner: pg2; I am pg1
Jun 23 15:13:27 pg_node1 patroni: 2021-06-23 15:13:27,532 INFO: manual failover: demote in progress
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28,292 INFO: Local timeline=11 lsn=0/5049750
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28,300 INFO: master_timeline=12
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28,315 INFO: master: history=8#0110/5014B48#011no recovery target specified
Jun 23 15:13:28 pg_node1 patroni: 9#0110/502CCE8#011no recovery target specified
Jun 23 15:13:28 pg_node1 patroni: 10#0110/5048A98#011no recovery target specified
Jun 23 15:13:28 pg_node1 patroni: 11#0110/50497C8#011no recovery target specified
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28,317 INFO: closed patroni connection to the postgresql cluster
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28,348 INFO: postmaster pid=4739
Jun 23 15:13:28 pg_node1 patroni: localhost:5432 - no response
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28.386 CST [4739] LOG:  starting PostgreSQL 13.3 on x86_64-pc-linux-gnu, compiled by gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-44), 64-bit
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28.386 CST [4739] LOG:  listening on IPv4 address "0.0.0.0", port 5432
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28.401 CST [4739] LOG:  listening on Unix socket "/tmp/.s.PGSQL.5432"
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28.417 CST [4741] LOG:  database system was shut down at 2021-06-23 15:13:25 CST
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28.418 CST [4741] LOG:  entering standby mode
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28.428 CST [4741] LOG:  consistent recovery state reached at 0/50497C8
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28.428 CST [4741] LOG:  invalid record length at 0/50497C8: wanted 24, got 0
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28.429 CST [4739] LOG:  database system is ready to accept read only connections
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28.447 CST [4745] LOG:  fetching timeline history file for timeline 12 from primary server
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28.460 CST [4745] LOG:  started streaming WAL from primary at 0/5000000 on timeline 11
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28.463 CST [4745] LOG:  replication terminated by primary server
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28.463 CST [4745] DETAIL:  End of WAL reached on timeline 11 at 0/50497C8.
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28.463 CST [4741] LOG:  new target timeline is 12
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28.464 CST [4745] LOG:  restarted WAL streaming at 0/5000000 on timeline 12
Jun 23 15:13:28 pg_node1 patroni: 2021-06-23 15:13:28.703 CST [4741] LOG:  redo starts at 0/50497C8
Jun 23 15:13:29 pg_node1 patroni: localhost:5432 - accepting connections
Jun 23 15:13:29 pg_node1 patroni: localhost:5432 - accepting connections
Jun 23 15:13:29 pg_node1 patroni: this is patroni callback on_role_change replica twpg
Jun 23 15:13:29 pg_node1 systemd: Started Session c6 of user root.
Jun 23 15:13:37 pg_node1 patroni: 2021-06-23 15:13:37,534 INFO: Lock owner: pg2; I am pg1
Jun 23 15:13:37 pg_node1 patroni: 2021-06-23 15:13:37,534 INFO: does not have lock
Jun 23 15:13:37 pg_node1 patroni: 2021-06-23 15:13:37,535 INFO: establishing a new patroni connection to the postgres cluster
Jun 23 15:13:37 pg_node1 patroni: 2021-06-23 15:13:37,579 INFO: no action.  i am a secondary and i am following a leader

#手工failover
[root@pg_node1 ~]# patronictl failover
Candidate ['pg1', 'pg3'] []: pg1
Current cluster topology
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Replica | running | 12 |       0.0 |
| pg2    | 192.168.210.81:5432 | Leader  | running | 12 |           |
| pg3    | 192.168.210.33:5432 | Replica | running | 12 |       0.0 |
+--------+---------------------+---------+---------+----+-----------+
Are you sure you want to failover cluster twpg, demoting current master pg2? [y/N]: y
2021-06-23 15:17:33.25244 Successfully failed over to "pg1"
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Leader  | running | 12 |           |
| pg2    | 192.168.210.81:5432 | Replica | stopped |    |   unknown |
| pg3    | 192.168.210.33:5432 | Replica | running | 12 |       0.0 |
+--------+---------------------+---------+---------+----+-----------+
[root@pg_node1 ~]# patronictl list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Leader  | running | 13 |           |
| pg2    | 192.168.210.81:5432 | Replica | stopped |    |   unknown |
| pg3    | 192.168.210.33:5432 | Replica | running | 13 |       0.0 |
+--------+---------------------+---------+---------+----+-----------+
[root@pg_node1 ~]# patronictl list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Leader  | running | 13 |           |
| pg2    | 192.168.210.81:5432 | Replica | running | 13 |       0.0 |
| pg3    | 192.168.210.33:5432 | Replica | running | 13 |       0.0 |
+--------+---------------------+---------+---------+----+-----------+
#日志信息如下：
Jun 23 15:17:27 pg_node1 patroni: 2021-06-23 15:17:27,609 INFO: Lock owner: pg2; I am pg1
Jun 23 15:17:27 pg_node1 patroni: 2021-06-23 15:17:27,609 INFO: does not have lock
Jun 23 15:17:27 pg_node1 patroni: 2021-06-23 15:17:27,612 INFO: no action.  i am a secondary and i am following a leader
Jun 23 15:17:31 pg_node1 patroni: 2021-06-23 15:17:31.544 CST [4745] LOG:  replication terminated by primary server
Jun 23 15:17:31 pg_node1 patroni: 2021-06-23 15:17:31.544 CST [4745] DETAIL:  End of WAL reached on timeline 12 at 0/504A428.
Jun 23 15:17:31 pg_node1 patroni: 2021-06-23 15:17:31.544 CST [4745] FATAL:  could not send end-of-streaming message to primary: no COPY in progress
Jun 23 15:17:31 pg_node1 patroni: 2021-06-23 15:17:31.545 CST [4741] LOG:  invalid record length at 0/504A428: wanted 24, got 0
Jun 23 15:17:31 pg_node1 patroni: 2021-06-23 15:17:31.550 CST [4810] FATAL:  could not connect to the primary server: could not connect to server: Connection refused
Jun 23 15:17:31 pg_node1 patroni: Is the server running on host "192.168.210.81" and accepting
Jun 23 15:17:31 pg_node1 patroni: TCP/IP connections on port 5432?
Jun 23 15:17:32 pg_node1 patroni: 2021-06-23 15:17:32,556 INFO: Cleaning up failover key after acquiring leader lock...
Jun 23 15:17:32 pg_node1 patroni: 2021-06-23 15:17:32,565 INFO: Software Watchdog activated with 25 second timeout, timing slack 15 seconds #激活看门狗
Jun 23 15:17:32 pg_node1 patroni: 2021-06-23 15:17:32,577 INFO: promoted self to leader by acquiring session lock
Jun 23 15:17:32 pg_node1 patroni: server promoting  #数据库提升
Jun 23 15:17:32 pg_node1 patroni: 2021-06-23 15:17:32,579 INFO: Lock owner: pg1; I am pg1
Jun 23 15:17:32 pg_node1 patroni: 2021-06-23 15:17:32.589 CST [4741] LOG:  received promote request
Jun 23 15:17:32 pg_node1 patroni: 2021-06-23 15:17:32.589 CST [4741] LOG:  redo done at 0/504A3B0
Jun 23 15:17:32 pg_node1 patroni: 2021-06-23 15:17:32,593 INFO: cleared rewind state after becoming the leader
Jun 23 15:17:32 pg_node1 patroni: 2021-06-23 15:17:32.594 CST [4741] LOG:  selected new timeline ID: 13
Jun 23 15:17:32 pg_node1 patroni: this is patroni callback on_role_change master twpg #回调VIP漂移脚本
Jun 23 15:17:32 pg_node1 patroni: 2021-06-23 15:17:32,611 INFO: updated leader lock during promote
Jun 23 15:17:32 pg_node1 systemd: Started Session c7 of user root.
Jun 23 15:17:32 pg_node1 systemd: Started Session c8 of user root.
Jun 23 15:17:32 pg_node1 patroni: 2021-06-23 15:17:32.837 CST [4741] LOG:  archive recovery complete
Jun 23 15:17:32 pg_node1 patroni: 2021-06-23 15:17:32.869 CST [4739] LOG:  database system is ready to accept connections
Jun 23 15:17:33 pg_node1 patroni: 2021-06-23 15:17:33,631 INFO: Lock owner: pg1; I am pg1
Jun 23 15:17:33 pg_node1 patroni: 2021-06-23 15:17:33,772 INFO: no action.  i am the leader with the lock

#查看集群参数
[root@pg_node1 ~]# patronictl show-config
loop_wait: 10
master_start_timeout: 300
maximum_lag_on_failover: 1048576
postgresql:
  parameters:
    hot_standby: 'on'
    listen_addresses: 0.0.0.0
    max_replication_slots: 10
    max_wal_senders: 10
    port: 5432
    wal_keep_segments: 256
    wal_level: replica
    wal_log_hints: 'on'
  use_pg_rewind: true
  use_slots: true
retry_timeout: 10
synchronous_mode: false
ttl: 30

#修改PG参数
#例如修改shared_buffers: 1GB
[root@pg_node1 ~]# patronictl edit-config twpg
--- 
+++ 
@@ -4,6 +4,7 @@
 postgresql:
   parameters:
     hot_standby: 'on'
+    shared_buffers: '1GB'
     listen_addresses: 0.0.0.0
     max_replication_slots: 10
     max_wal_senders: 10

Apply these changes? [y/N]: y
Configuration changed
[root@pg_node1 ~]# patronictl restart twpg
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+-----------------+
| Member | Host                | Role    | State   | TL | Lag in MB | Pending restart |
+--------+---------------------+---------+---------+----+-----------+-----------------+
| pg1    | 192.168.210.15:5432 | Leader  | running | 13 |           | *               |
| pg2    | 192.168.210.81:5432 | Replica | running | 13 |       0.0 | *               |
| pg3    | 192.168.210.33:5432 | Replica | running | 13 |       0.0 | *               |
+--------+---------------------+---------+---------+----+-----------+-----------------+
When should the restart take place (e.g. 2021-06-23T16:45)  [now]: 
Are you sure you want to restart members pg2, pg3, pg1? [y/N]: y
Restart if the PostgreSQL version is less than provided (e.g. 9.5.2)  []: 
Success: restart on member pg2
Success: restart on member pg3
Success: restart on member pg1
[root@pg_node2 ~]# psql -U twsm -d postgres -c 'show shared_buffers;'
 shared_buffers 
----------------
 1GB
(1 row)

#或者通过REST API修改，例如：max_connections修改为1000
curl -s -XPATCH -d '{"postgresql":{"parameters":{"max_connections":"1000"}}}' http://localhost:8008/config | jq .
#如果想reset或是删除
curl -s -XPATCH -d '{"postgresql":{"parameters":{"max_connections":null}}}' http://localhost:8008/config | jq .
#修改max_connections需要重启（Pending restart）
[root@pg_node1 ~]# patronictl list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+-----------------+
| Member | Host                | Role    | State   | TL | Lag in MB | Pending restart |
+--------+---------------------+---------+---------+----+-----------+-----------------+
| pg1    | 192.168.210.15:5432 | Replica | running | 14 |       0.0 | *               |
| pg2    | 192.168.210.81:5432 | Replica | running | 14 |       0.0 | *               |
| pg3    | 192.168.210.33:5432 | Leader  | running | 14 |           | *               |
+--------+---------------------+---------+---------+----+-----------+-----------------+
#如果想无条件地完全重写现有的动态配置
curl -s -XPUT -d '{"maximum_lag_on_failover":1048576,"retry_timeout":10,"postgresql":{"use_slots":true,"use_pg_rewind":true,"parameters":{"hot_standby":"on","wal_log_hints":"on","wal_level":"hot_standby","unix_socket_directories":".","max_wal_senders":5}},"loop_wait":3,"ttl":20}' http://localhost:8008/config | jq .

#查看历史failovers/switchovers
[root@pg_node1 ~]# patronictl history
+----+----------+------------------------------+----------------------------------+
| TL |      LSN | Reason                       | Timestamp                        |
+----+----------+------------------------------+----------------------------------+
|  1 | 25210568 | no recovery target specified | 2021-06-21T15:24:42.878129+08:00 |
|  2 | 25211144 | no recovery target specified | 2021-06-21T15:33:23.462860+08:00 |
|  3 | 83886408 | no recovery target specified | 2021-06-23T09:28:25.304246+08:00 |
|  4 | 83886920 | no recovery target specified | 2021-06-23T09:35:14.536083+08:00 |
|  5 | 83887496 | no recovery target specified | 2021-06-23T09:37:56.880226+08:00 |
|  6 | 83888072 | no recovery target specified | 2021-06-23T09:59:13.584445+08:00 |
|  7 | 83888704 | no recovery target specified | 2021-06-23T12:39:14.897591+08:00 |
|  8 | 83970888 | no recovery target specified | 2021-06-23T13:55:26.207367+08:00 |
|  9 | 84069608 | no recovery target specified | 2021-06-23T14:24:48.030224+08:00 |
| 10 | 84183704 | no recovery target specified | 2021-06-23T14:41:45.263156+08:00 |
| 11 | 84187080 | no recovery target specified | 2021-06-23T15:13:26.627326+08:00 |
| 12 | 84190248 | no recovery target specified | 2021-06-23T15:17:32.836962+08:00 |
+----+----------+------------------------------+----------------------------------+

#集群进入维护模式，防止自动故障转移
+----+----------+------------------------------+----------------------------------+
[root@pg_node1 ~]# patronictl pause
Success: cluster management is paused
[root@pg_node1 ~]# patronictl list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Leader  | running | 13 |           |
| pg2    | 192.168.210.81:5432 | Replica | running | 13 |       0.0 |
| pg3    | 192.168.210.33:5432 | Replica | running | 13 |       0.0 |
+--------+---------------------+---------+---------+----+-----------+
 Maintenance mode: on
#日志信息：
Jun 23 16:00:02 pg_node1 patroni: 2021-06-23 16:00:02,006 INFO: Lock owner: pg1; I am pg1
Jun 23 16:00:02 pg_node1 patroni: 2021-06-23 16:00:02,018 INFO: PAUSE: no action.  i am the leader with the lock
Jun 23 16:00:02 pg_node1 patroni: 2021-06-23 16:00:02,028 INFO: No PostgreSQL configuration items changed, nothing to reload.
#维护结束，恢复自动故障转移
[root@pg_node1 ~]# patronictl resume
Success: cluster management is resumed
[root@pg_node1 ~]# patronictl list
+ Cluster: twpg (6976142033405049133) ---+---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg1    | 192.168.210.15:5432 | Leader  | running | 13 |           |
| pg2    | 192.168.210.81:5432 | Replica | running | 13 |       0.0 |
| pg3    | 192.168.210.33:5432 | Replica | running | 13 |       0.0 |
+--------+---------------------+---------+---------+----+-----------+
#日志信息：
Jun 23 16:02:32 pg_node1 patroni: 2021-06-23 16:02:32,006 INFO: Lock owner: pg1; I am pg1
Jun 23 16:02:32 pg_node1 patroni: 2021-06-23 16:02:32,012 INFO: PAUSE: no action.  i am the leader with the lock
Jun 23 16:02:39 pg_node1 patroni: 2021-06-23 16:02:39,565 INFO: Lock owner: pg1; I am pg1
Jun 23 16:02:39 pg_node1 patroni: 2021-06-23 16:02:39,572 INFO: Software Watchdog activated with 25 second timeout, timing slack 15 seconds
Jun 23 16:02:39 pg_node1 patroni: 2021-06-23 16:02:39,581 INFO: no action.  i am the leader with the lock
Jun 23 16:02:39 pg_node1 patroni: 2021-06-23 16:02:39,588 INFO: No PostgreSQL configuration items changed, nothing to reload.
Jun 23 16:02:49 pg_node1 patroni: 2021-06-23 16:02:49,552 INFO: Lock owner: pg1; I am pg1
Jun 23 16:02:49 pg_node1 patroni: 2021-06-23 16:02:49,558 INFO: no action.  i am the leader with the lock

```
![[Pasted image 20240903175126.png]]