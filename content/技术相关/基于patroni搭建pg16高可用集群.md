

>本文基于patroni搭建postgresql 16的高可用集群。

  

# Patroni介绍

  

Patroni是一个基于Python的用于实现PostgreSQL HA解决方案的框架。为了最大程度的兼容性，它支持多种分布式配置存储，包括ZooKeeper、etcd、Consul或Kubernetes。旨在帮助数据库工程师、DBA、DevOps工程师和SRE快速部署数据中心（或任何地方）的HA PostgreSQL环境。

  

当前支持的PostgreSQL版本从9.3到16。支持自动化故障转移、物理复制和逻辑复制、提供 RESTful API 接口，允许外部应用或运维工具直接操作 PostgreSQL 集群，进行如启停、迁移等操作，与 Linux watchdog集成，以避免脑裂现象。

  

项目地址：https://github.com/zalando/patroni/

  

# Etcd介绍

  

etcd 是一个分布式键值存储数据库，支持跨平台，拥有强大的社区。etcd 的 Raft 算法，提供了可靠的方式存储分布式集群涉及的数据。etcd 广泛应用在微服务架构和 Kubernates 集群中，不仅可以作为服务注册与发现，还可以作为键值对存储的中间件。从业务系统 Web 到 Kubernetes 集群，都可以很方便地从 etcd 中读取、写入数据。

etcd完整的cluster（集群）至少有三台，这样才能选举出一个master节点，两个slave节点。如果小于 3 台则无法进行选举,造成集群不可用。Etcd使用2379和2380端口。

2379端口：提供HTTP API服务，和etcdctl交互

2380端口：集群中节点间通讯

项目地址：https://github.com/etcd-io/etcd

  

# 环境说明

  

|   |   |   |   |   |
|---|---|---|---|---|
|主机名|ip地址|OS版本|内存、CPU|数据库端口|
|node1|172.17.44.155|Centos7.9|8C16G200G|5432|
|node2|172.17.44.156|Centos7.9|8C16G200G|5432|
|node3|172.17.44.157|Centos7.9|8C16G200G|5432|
|pgw+keepalived|172.17.44.158|Centos7.9|4C8G200G|5432|

  

**vip：172.17.44.159**

  

# Etcd部署

  

## 安装etcd

  

```Plain
cd /soft/
tar -zxvf etcd-v3.5.13-linux-amd64.tar.gz
cd /soft/etcd-v3.5.13-linux-amd64
cp etcd /usr/bin
cp etcdctl /usr/bin

--查看版本
# etcdctl version
etcdctl version: 3.5.13
API version: 3.3
```

  

## 配置etcd

  

注意：每个节点配置不同。

  

node1:

  

```Plain
mkdir -p /etc/etcd/
mkdir -p /acdata/data/etcd/
vi /etc/etcd/etcd.conf
name: etcd-1
data-dir: /acdata/data/etcd/
listen-client-urls: http://172.17.44.155:2379,http://127.0.0.1:2379
advertise-client-urls: http://172.17.44.155:2379,http://127.0.0.1:2379
listen-peer-urls: http://172.17.44.155:2380
initial-advertise-peer-urls: http://172.17.44.155:2380
initial-cluster: etcd-1=http://172.17.44.155:2380,etcd-2=http://172.17.44.156:2380,etcd-3=http://172.17.44.157:2380
initial-cluster-token: etcd-cluster
initial-cluster-state: new
```

  

node2:

  

```Plain
mkdir -p /etc/etcd/
mkdir -p /acdata/data/etcd/
vi /etc/etcd/etcd.conf
name: etcd-2
data-dir: /acdata/data/etcd/
listen-client-urls: http://172.17.44.156:2379,http://127.0.0.1:2379
advertise-client-urls: http://172.17.44.156:2379,http://127.0.0.1:2379
listen-peer-urls: http://172.17.44.156:2380
initial-advertise-peer-urls: http://172.17.44.156:2380
initial-cluster: etcd-1=http://172.17.44.155:2380,etcd-2=http://172.17.44.156:2380,etcd-3=http://172.17.44.157:2380
initial-cluster-token: etcd-cluster
initial-cluster-state: new
```

  

node3:

  

```Plain
mkdir -p /etc/etcd/
mkdir -p /acdata/data/etcd/
/etc/etcd/etcd.conf
name: etcd-3
data-dir: /acdata/data/etcd/
listen-client-urls: http://172.17.44.157:2379,http://127.0.0.1:2379
advertise-client-urls: http://172.17.44.157:2379,http://127.0.0.1:2379
listen-peer-urls: http://172.17.44.157:2380
initial-advertise-peer-urls: http://172.17.44.157:2380
initial-cluster: etcd-1=http://172.17.44.155:2380,etcd-2=http://172.17.44.156:2380,etcd-3=http://172.17.44.157:2380
initial-cluster-token: etcd-cluster
initial-cluster-state: new
```

  

## 配置etcd service

  

```Plain
vi /usr/lib/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
User=root
Type=notify
WorkingDirectory=/acdata/data/etcd/
ExecStart=/usr/bin/etcd --config-file=/etc/etcd/etcd.conf
Restart=on-failure
LimitNOFILE=65536
```

  

## 管理etcd集群

  

```Plain
systemctl daemon-reload
systemctl restart etcd
systemctl status etcd
systemctl enable etcd
```

  

## 查看etcd集群的状态

  

```Plain
--查看etcd服务状态
[root@node1 soft]# systemctl status etcd
● etcd.service - Etcd Server
   Loaded: loaded (/usr/lib/systemd/system/etcd.service; enabled; vendor preset: disabled)
   Active: active (running) since Wed 2024-05-29 19:59:13 CST; 6s ago
 Main PID: 2792 (etcd)
    Tasks: 9
   CGroup: /system.slice/etcd.service
           └─2792 /usr/bin/etcd --config-file=/etc/etcd/etcd.conf

May 29 19:59:13 node1 etcd[2792]: ready to serve client requests
May 29 19:59:13 node1 etcd[2792]: serving insecure client requests on 127.0.0.1:2379, this is strongly discouraged!
May 29 19:59:13 node1 etcd[2792]: ready to serve client requests
May 29 19:59:13 node1 etcd[2792]: serving insecure client requests on 172.17.44.155:2379, this is strongly discouraged!
May 29 19:59:13 node1 systemd[1]: Started Etcd Server.
May 29 19:59:13 node1 etcd[2792]: established a TCP streaming connection with peer cd4811b8f06b87 (stream MsgApp v2 writer)
May 29 19:59:13 node1 etcd[2792]: established a TCP streaming connection with peer cd4811b8f06b87 (stream Message writer)
May 29 19:59:13 node1 etcd[2792]: 8530747d0edb6666 initialzed peer connection; fast-forwarding 8 ticks (election ticks 10) with 2 active peer(s)
May 29 19:59:14 node1 etcd[2792]: updated the cluster version from 3.0 to 3.3
May 29 19:59:14 node1 etcd[2792]: enabled capabilities for version 3.3

--查看集群状态
[root@wtj1vpk8sql01 ~]# etcdctl  --endpoints=172.17.44.155:2379,172.17.44.156:2379,172.17.44.157:2379 endpoint status -w table
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 172.17.44.155:2379 | a06f5cdc3087e5ea |  3.5.15 |   20 kB |     false |      false |         5 |         15 |                 15 |        |
| 172.17.44.156:2379 | bed073fe9333880e |  3.5.15 |   20 kB |      true |      false |         5 |         15 |                 15 |        |
| 172.17.44.157:2379 | 116da65d9b8fc137 |  3.5.15 |   20 kB |     false |      false |         5 |         15 |                 15 |        |
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
```

  

# 配置ntp时间同步

  

```Plain
yum install -y ntpdate

--每天凌晨2点进行同步
crontab -e
0 2 * * * /usr/sbin/ntpdate  time.windows.com >> /var/log/ntpdate.log 2>&1
```

  

# PG16软件部署

  

## PG下载

  

```Plain
地址：https://www.postgresql.org/ftp/source/v16.3/
文件：postgresql-16.3.tar.gz
```

  

## 关闭SeLinux

  

`sed -i 's/^SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config`

`setenforce 0`

  

## 关闭防火墙

  

```Bash
 systemctl stop firewalld
 systemctl disable firewalld
 systemctl status firewalld
```

  

## 关闭RemoveIPC

  

```Bash
vi /etc/systemd/logind.conf
RemoveIPC=no

systemctl daemon-reload
systemctl restart systemd-logind
```

  

  

## 安装依赖包

  

```Plain
yum install -y readline readline-devel flex bison openssl openssl-devel git 
yum install -y gcc gcc-c++  epel-release llvm5.0 llvm5.0-devel clang libicu-devel perl-ExtUtils-Embed zlib-devel openssl openssl-devel pam-devel libxml2-devel libxslt-devel openldap-devel systemd-devel tcl-devel python-devel python3-devel lz4 lz4-devel uuid libuuid-devel
```

  

## 创建组和用户

  

```Plain
groupadd -g 2000 postgres
useradd  -g 2000 -u 2000 postgres
echo postgres|passwd --stdin postgres
```

  

## 设置postgres用户的环境变量

  

```Plain
vi .bash_profile
export PGHOME=/acdata/data/pg16/pgsoft
export PGDATA=/acdata/data/pg16/data
export PATH=$PGHOME/bin:$PATH:.
export LD_LIBRARY_PATH=$PGHOME/lib:$LD_LIBRARY_PATH
export PGPORT=5432
export PGUSER=postgres
export PGDATABASE=postgres  
```

##  环境变量最好放在全局配置文件/etc/profile中

否则，在启动服务时，可能会遇到如下问题：
  ```js

  Aug 28 16:06:21 wtj1vpk8sql01 patroni: FATAL: Patroni requires psycopg2>=2.5.4, psycopg2-binary, or psycopg>=3.0.0
Aug 28 16:06:21 wtj1vpk8sql01 systemd: patroni.service: main process exited, code=exited, status=1/FAILURE
Aug 28 16:06:21 wtj1vpk8sql01 systemd: Unit patroni.service entered failed state.
Aug 28 16:06:21 wtj1vpk8sql01 systemd: patroni.service failed.
```

## 创建pg16软件安装目录

  

```Plain
mkdir -p /acdata/data/pg16/pgsoft
chmod 755 -R /acdata/data/pg16/pgsoft
chown -R postgres:postgres /acdata/data/pg16/pgsoft
```

  

## 创建数据文件存放目录

  

```Plain
mkdir -p /acdata/data/pg16/data
chmod -R 700 /acdata/data/pg16/data
chown -R postgres:postgres /acdata/data/pg16/data
```

  

## 解压软件并授权

  

```Plain
tar -zxvf /soft/postgresql-16.3.tar.gz 
chown -R postgres:postgres /soft/postgresql-16.3/
```

  

## 编译安装pg16

  

```Plain
su - postgres
cd /soft/postgresql-16.3/
./configure  --prefix=/acdata/data/pg16/pgsoft --with-pgport=5432  --with-extra-version=" [By gg]" --with-perl  --with-libxml  --with-libxslt
gmake world && gmake install-world
```

  

# 配置sudo权限

  

  

```Bash
cat >> /etc/sudoers <<EOF
postgres        ALL=(root)        NOPASSWD: ALL
EOF

```

  

# watchdog部署

  

watchdog防止脑裂。Patroni支持通过Linux的watchdog监视patroni进程的运行，当patroni进程无法正常往watchdog设备写入心跳时，由watchdog触发Linux重启。

  

```Plain
# 安装软件，linux内置功能
yum install -y watchdog
# 初始化watchdog字符设备
modprobe softdog
# 修改/dev/watchdog设备权限
chmod 666 /dev/watchdog
# 启动watchdog服务
systemctl start watchdog
systemctl enable watchdog
```

  

# Patroni部署

  

## 安装patroni

  

```Plain
curl https://bootstrap.pypa.io/pip/3.6/get-pip.py -o get-pip.py
python3 get-pip.py
pip3 install --upgrade pip
pip3 install psycopg2-binary
pip3 install patroni[etcd]
pip3 install patroni
```

  

## 检查patroni

  

```Plain
#which patroni
/usr/local/bin/patroni
# patroni --version
patroni 3.3.2
```

  

## 创建patroni系统服务

  

```Plain
cat >> /usr/lib/systemd/system/patroni.service <<EOF
[Unit]
Description=Runners to orchestrate a high-availability PostgreSQL
After=syslog.target network.target
  
[Service]
Type=simple
User=postgres
Group=postgres
EnvironmentFile=-/etc/patroni/patroni_env.conf
# 使用watchdog进行服务监控
ExecStartPre=-/usr/bin/sudo /sbin/modprobe softdog
ExecStartPre=-/usr/bin/sudo /bin/chown postgres /dev/watchdog
# 注意patroni命令的路径的正确性
ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml 
ExecReload=/bin/kill -s HUP \$MAINPID
KillMode=process
TimeoutSec=30
Restart=no
  
[Install]
WantedBy=multi-user.target
EOF

#重新加载systemd服务
systemctl daemon-reload
```

  

## 配置patroni

  

```Plain
# 不同节点需要修改对应的IP信息，以及开头的name信息
node1:
# root用户执行
mkdir -p /etc/patroni
vi /etc/patroni/patroni.yml 

scope: pgsql16
namespace: /pgsql/
name: pg01

restapi:
  listen: 0.0.0.0:8008
  connect_address: 172.17.44.155:8008

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
        wal_level: replica
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

  pg_hba:
   - host replication repuser 172.17.44.0/21 md5
   - host all all 172.17.0.0/16 md5

postgresql:
  listen: 172.17.44.155:5432
  connect_address: 172.17.44.155:5432
  data_dir: /acdata/data/pg16/data
  bin_dir: /acdata/data/pg16/pgsoft/bin

  authentication:
    replication:
      username: repuser
      password: 123456
    superuser:
      username: postgres
      password: postgres

basebackup:
    #max-rate: 100M
    checkpoint: fast
callbacks:
    on_start: /etc/patroni/patroni_callback.sh
    on_stop:  /etc/patroni/patroni_callback.sh
    on_role_change: /etc/patroni/patroni_callback.sh

watchdog:
  mode: automatic       # Allowed values: off, automatic, required
  device: /dev/watchdog
  safety_margin: 5

tags:
   nofailover: false
   noloadbalance: false
   clonefrom: false
   nosync: false
   
# node2修改:
name: pg02
restapi:
  listen: 0.0.0.0:8008
  connect_address: 172.17.44.156:8008

postgresql:
  listen: 172.17.44.156:5432
  connect_address: 172.17.44.156:5432
  
# node3修改
name: pg03
restapi:
  listen: 0.0.0.0:8008
  connect_address: 172.17.44.157:8008

postgresql:
  listen: 172.17.44.157:5432
  connect_address: 172.17.44.157:5432  
```

  

## 创建patroni的callbak脚本

  

```Plain
cat >> /etc/patroni/patroni_callback.sh <<EOF

#!/bin/bash

readonly OPERATION=$1
readonly ROLE=$2
readonly SCOPE=$3
VIP='172.17.44.159'
PREFIX='21'
BRD='172.17.47.255'
INF='ens192'

function usage() {
 echo "Usage: $0 <on_start|on_stop|on_role_change> <ROLE> <SCOPE>";
 exit 1;
}

echo "$(date "+%Y-%m-%d %H:%M:%S %z") This is patroni callback $OPERATION $ROLE $SCOPE"

case $OPERATION in
 on_stop)
 sudo ip addr del ${VIP}/${PREFIX} dev ${INF} label ${INF}:1
 echo "$(date "+%Y-%m-%d %H:%M:%S %z") VIP ${VIP} removed"
 ;;

 on_start | on_restart | on_role_change)
 if [[ $ROLE == 'master' || $ROLE == 'standby_leader' ]]; then
 sudo ip addr add ${VIP}/${PREFIX} brd ${BRD} dev ${INF} label ${INF}:1
 sudo arping -q -A -c 1 -I ${INF} ${VIP}
 echo "$(date "+%Y-%m-%d %H:%M:%S %z") VIP ${VIP} added"

 else
 sudo ip addr del ${VIP}/${PREFIX} dev ${INF} label ${INF}:1
 echo "$(date "+%Y-%m-%d %H:%M:%S %z") VIP ${VIP} removed"
 fi
 ;;

 *)
 usage
 ;;
esac
EOF
```

  

## patroni的callbak脚本添加执行权限

  

```Plain
chmod +x /etc/patroni/patroni_callback.sh
chown postgres:postgres /etc/patroni/patroni_callback.sh
```

  

## 3台节点启动patroni集群

  

```Plain
--方法一:直接启动：
su - postgres
patroni /etc/patroni/patroni.yml

--方法二：服务启动
systemctl start patroni
```

  

## 查看patroni集群状态

  

```Plain
patronictl -c /etc/patroni/patroni.yml list

[root@node3 pg16]# patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pgsql16 (7376595871806153493) -----+-----------+----+-----------+
| Member | Host                | Role         | State     | TL | Lag in MB |
+--------+---------------------+--------------+-----------+----+-----------+
| pg01   | 172.17.44.155:5432 | Leader       | running   |  7 |           |
| pg02   | 172.17.44.156:5432 | Sync Standby | streaming |  7 |         0 |
| pg03   | 172.17.44.157:5432 | Replica      | streaming |  7 |         0 |
+--------+---------------------+--------------+-----------+----+-----------+
```

  

# 主节点node1上查看复制状态

  

```Plain
postgres=# select * from pg_stat_replication;
  pid  | usesysid | usename | application_name |  client_addr   | client_hostname | client_port |         backend_start         | backend_xmin |   state   | 
sent_lsn  | write_lsn | flush_lsn | replay_lsn | write_lag | flush_lag | replay_lag | sync_priority | sync_state |          reply_time           
-------+----------+---------+------------------+----------------+-----------------+-------------+-------------------------------+--------------+-----------+-
----------+-----------+-----------+------------+-----------+-----------+------------+---------------+------------+-------------------------------
 27254 |    16385 | repuser | pg02             | 172.17.44.156 |                 |       41246 | 2024-06-04 19:33:29.367051+08 |              | streaming | 
0/6000148 | 0/6000148 | 0/6000148 | 0/6000148  |           |           |            |             1 | sync       | 2024-06-04 20:21:24.373685+08
 27309 |    16385 | repuser | pg03             | 172.17.44.157 |                 |       55470 | 2024-06-04 19:38:30.09852+08  |              | streaming | 
0/6000148 | 0/6000148 | 0/6000148 | 0/6000148  |           |           |            |             0 | async      | 2024-06-04 20:21:24.398927+08
(2 rows)
```

  

# 高可用测试

  

## 主节点手工切换测试（pg01->pg02）

  

```Plain
[postgres@node2 ~]$ patronictl -c /etc/patroni/patroni.yml switchover
Current cluster topology
+ Cluster: pgsql16 (7376595871806153493) -----+-----------+----+-----------+
| Member | Host                | Role         | State     | TL | Lag in MB |
+--------+---------------------+--------------+-----------+----+-----------+
| pg01   | 172.17.44.155:5432 | Leader       | running   |  7 |           |
| pg02   | 172.17.44.156:5432 | Sync Standby | streaming |  7 |         0 |
| pg03   | 172.17.44.157:5432 | Replica      | streaming |  7 |         0 |
+--------+---------------------+--------------+-----------+----+-----------+
Primary [pg01]:           
Candidate ['pg02', 'pg03'] []: 
When should the switchover take place (e.g. 2024-06-04T23:32 )  [now]: 
Are you sure you want to switchover cluster pgsql16, demoting current leader pg01? [y/N]: y
2024-06-04 22:33:08.16573 Successfully switched over to "pg02"
+ Cluster: pgsql16 (7376595871806153493) +---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg01   | 172.17.44.155:5432 | Replica | stopped |    |   unknown |
| pg02   | 172.17.44.156:5432 | Leader  | running |  7 |           |
| pg03   | 172.17.44.157:5432 | Replica | running |  7 |         0 |
+--------+---------------------+---------+---------+----+-----------+
[postgres@node2 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pgsql16 (7376595871806153493) -----+-----------+----+-----------+
| Member | Host                | Role         | State     | TL | Lag in MB |
+--------+---------------------+--------------+-----------+----+-----------+
| pg01   | 172.17.44.155:5432 | Replica      | streaming |  8 |         0 |
| pg02   | 172.17.44.156:5432 | Leader       | running   |  8 |           |
| pg03   | 172.17.44.157:5432 | Sync Standby | streaming |  8 |         0 |
+--------+---------------------+--------------+-----------+----+-----------+
[postgres@node2 ~]$ 
```

  

## 同步从节点（pg03）关机测试

  

```Plain
[postgres@node2 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pgsql16 (7376595871806153493) -----+-----------+----+-----------+
| Member | Host                | Role         | State     | TL | Lag in MB |
+--------+---------------------+--------------+-----------+----+-----------+
| pg01   | 172.17.44.155:5432 | Replica      | streaming |  8 |         0 |
| pg02   | 172.17.44.156:5432 | Leader       | running   |  8 |           |
| pg03   | 172.17.44.157:5432 | Sync Standby | streaming |  8 |         0 |
+--------+---------------------+--------------+-----------+----+-----------+

--node3关机
[root@node3 pg16]# shutdown -h 0
Shutdown scheduled for Tue 2024-06-04 22:58:16 CST, use 'shutdown -c' to cancel.

--同步从节点切换到node1
[postgres@node2 ~]$ patronictl -c /etc/patroni/patroni.yml list
2024-06-05 03:55:49,202 - ERROR - Failed to get list of machines from http://172.17.44.157:2379/v3beta: MaxRetryError("HTTPConnectionPool(host='172.17.44.157', port=2379): Max retries exceeded with url: /version (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7f971f1637b8>: Failed to establish a new connection: [Errno 113] No route to host',))",)
2024-06-05 03:55:50,514 - ERROR - Request to server http://172.17.44.157:2379 failed: MaxRetryError("HTTPConnectionPool(host='172.17.44.157', port=2379): Max retries exceeded with url: /v3/kv/range (Caused by ConnectTimeoutError(<urllib3.connection.HTTPConnection object at 0x7f971f163cf8>, 'Connection to 172.17.44.157 timed out. (connect timeout=1.25)'))",)
2024-06-05 03:55:51,774 - ERROR - Failed to get list of machines from http://172.17.44.157:2379/v3: MaxRetryError("HTTPConnectionPool(host='172.17.44.157', port=2379): Max retries exceeded with url: /v3/cluster/member/list (Caused by ConnectTimeoutError(<urllib3.connection.HTTPConnection object at 0x7f971f16f240>, 'Connection to 172.17.44.157 timed out. (connect timeout=1.25)'))",)
+ Cluster: pgsql16 (7376595871806153493) -----+-----------+----+-----------+
| Member | Host                | Role         | State     | TL | Lag in MB |
+--------+---------------------+--------------+-----------+----+-----------+
| pg01   | 172.17.44.155:5432 | Sync Standby | streaming |  8 |         0 |
| pg02   | 172.17.44.156:5432 | Leader       | running   |  8 |           |
+--------+---------------------+--------------+-----------+----+-----------+
```

  

## 主节点（pg02）关机测试

  

```Plain
[postgres@node2 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pgsql16 (7376595871806153493) -----+-----------+----+-----------+
| Member | Host                | Role         | State     | TL | Lag in MB |
+--------+---------------------+--------------+-----------+----+-----------+
| pg01   | 172.17.44.155:5432 | Sync Standby | streaming |  8 |         0 |
| pg02   | 172.17.44.156:5432 | Leader       | running   |  8 |           |
| pg03   | 172.17.44.157:5432 | Replica      | streaming |  8 |         0 |
+--------+---------------------+--------------+-----------+----+-----------+
--node2（pg02）关闭
[root@node2 data]# shutdown -hr 0
Shutdown scheduled for Wed 2024-06-05 04:25:44 CST, use 'shutdown -c' to cancel.

--node1(pg01)变成了主节点
[root@node1 pg16]#  patronictl -c /etc/patroni/patroni.yml list
2024-06-05 04:30:15,767 - ERROR - Failed to get list of machines from http://172.17.44.156:2379/v3beta: MaxRetryError("HTTPConnectionPool(host='172.17.44.156', port=2379): Max retries exceeded with url: /version (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7f26e3d76860>: Failed to establish a new connection: [Errno 111] Connection refused',))",)
2024-06-05 04:30:15,816 - ERROR - Request to server http://172.17.44.156:2379 failed: MaxRetryError("HTTPConnectionPool(host='172.17.44.156', port=2379): Max retries exceeded with url: /v3/kv/range (Caused by NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7f26e3d76be0>: Failed to establish a new connection: [Errno 111] Connection refused',))",)
+ Cluster: pgsql16 (7376595871806153493) +---------+----+-----------+
| Member | Host                | Role    | State   | TL | Lag in MB |
+--------+---------------------+---------+---------+----+-----------+
| pg01   | 172.17.44.155:5432 | Leader  | running |  9 |           |
| pg03   | 172.17.44.157:5432 | Replica | running |  9 |         0 |
+--------+---------------------+---------+---------+----+-----------+

--node2（pg02）重启后，自动加入集群
[root@node1 pg16]#  patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pgsql16 (7376595871806153493) -----+-----------+----+-----------+
| Member | Host                | Role         | State     | TL | Lag in MB |
+--------+---------------------+--------------+-----------+----+-----------+
| pg01   | 172.17.44.155:5432 | Leader       | running   |  9 |           |
| pg02   | 172.17.44.156:5432 | Replica      | streaming |  9 |         0 |
| pg03   | 172.17.44.157:5432 | Sync Standby | streaming |  9 |         0 |
+--------+---------------------+--------------+-----------+----+-----------+
[root@node1 pg16]# 
```

  

# 错误处理

  

patronictl list报错

  

```Plain
[postgres@node3 ~]$ patronictl -c /etc/patroni/patroni.yml list
2024-06-02 17:44:02,366 - ERROR - Request to server http://172.17.44.157:2379 failed: MaxRetryError('HTTPConnectionPool(host=\'172.17.44.157\', port=2379): Max retries exceeded with url: /v2/keys/pgsql/pgsql16/?recursive=true&quorum=true (Caused by ReadTimeoutError("HTTPConnectionPool(host=\'172.17.44.157\', port=2379): Read timed out. (read timeout=2.499634733001585)",))',)
解决办法：增加/etc/patroni/patroni.yml中retry_timeout的值
```

  

# 参考文档

  

https://patroni.readthedocs.io/en/latest/README.html

https://blog.csdn.net/lsx_3/article/details/131072594

  

# 总结

  

整个安装、测试过程还是非常丝滑顺利的~