
# 在leader节点上以postgres身份执行

```bash
su - postgres

# 添加虚拟网卡ens192:2

$ [postgres@wtj1vpk8sql01 ~]$ /etc/patroni/patroni_callback.sh on_start Leader pg-sentry-cluster01

[postgres@wtj1vpk8sql01 ~]$ ip ad sh|grep 172.17.
    inet 172.17.44.155/22 brd 172.17.47.255 scope global noprefixroute ens192
    inet 172.17.44.159/22 brd 172.17.47.255 scope global ens192:2

```

# 发生故障切换后，先在原来leader节点关闭vip

```bash
$ [postgres@wtj1vpk8sql01 ~]$ /etc/patroni/patroni_callback.sh on_stop Leader pg-sentry-cluster01

```

# 再次查看集群状态，确认新的leader节点

```bash

[postgres@wtj1vpk8sql01 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg-sentry-cluster01 (7410688421450405471) ----+-----------+
| Member | Host          | Role         | State     | TL | Lag in MB |
+--------+---------------+--------------+-----------+----+-----------+
| pg01   | 172.17.44.155 | Sync Standby | streaming |  9 |           |
| pg02   | 172.17.44.156 | Leader       | running   |  9 |         0 |
| pg03   | 172.17.44.157 | Replica      | streaming |  9 |         0 |
+--------+---------------+--------------+-----------+----+-----------+

```

# 在新leader节点执行如下命令，添加vip及arp广播

```bash
[postgres@wtj1vpk8sql01 ~]$ /etc/patroni/patroni_callback.sh on_start Leader pg-sentry-cluster01

[postgres@wtj1vpk8sql01 ~]$  /sbin/arping -I ens192 -c 3 -A 172.17.44.159

```