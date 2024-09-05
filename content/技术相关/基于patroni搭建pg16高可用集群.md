

>本文基于patroni搭建postgresql 16的高可用集群。

  

# Patroni介绍

  

Patroni是一个基于Python的用于实现PostgreSQL HA解决方案的框架。为了最大程度的兼容性，它支持多种分布式配置存储，包括ZooKeeper、etcd、Consul或Kubernetes。旨在帮助数据库工程师、DBA、DevOps工程师和SRE快速部署数据中心（或任何地方）的HA PostgreSQL环境。

  

当前支持的PostgreSQL版本从9.3到16。支持自动化故障转移、物理复制和逻辑复制、提供 RESTful API 接口，允许外部应用或运维工具直接操作 PostgreSQL 集群，进行如启停、迁移等操作，与 Linux watchdog集成，以避免脑裂现象。

# Patroni 优势

- 支持自动 failover 和按需 switchover
- 支持一个和多个备节点
- 支持级联复制
- 支持同步复制，异步复制
- 支持同步复制下备库故障时自动降级为异步复制（功效类似于 MySQL 的半同步，但是更加智能）
- 支持控制指定节点是否参与选主，是否参与负载均衡以及是否可以成为同步备机
- 支持通过pg_rewind自动修复旧主
- 支持多种方式初始化集群和重建备机，包括pg_basebackup和支持wal_e，pgBackRest，barman等备份工具的自定义脚本
- 支持自定义外部 callback 脚本
- 支持 REST API
- 支持通过 watchdog 防止脑裂
- 支持 k8s，docker 等容器化环境部署
- 支持多种常见 DCS(Distributed Configuration Store)存储元数据，包括 etcd，ZooKeeper，Consul，Kubernetes

因此，除非只有 2 台机器没有多余机器部署 [DCS](https://so.csdn.net/so/search?q=DCS&spm=1001.2101.3001.7020) 的情况，Patroni 是一款非常值得推荐的 PostgreSQL 高可用工具。

项目地址：https://github.com/zalando/patroni/

  

# Etcd介绍

  

etcd 是一个分布式键值存储数据库，支持跨平台，拥有强大的社区。etcd 的 Raft 算法，提供了可靠的方式存储分布式集群涉及的数据。etcd 广泛应用在微服务架构和 Kubernates 集群中，不仅可以作为服务注册与发现，还可以作为键值对存储的中间件。从业务系统 Web 到 Kubernetes 集群，都可以很方便地从 etcd 中读取、写入数据。

etcd完整的cluster（集群）至少有三台，这样才能选举出一个master节点，两个slave节点。如果小于 3 台则无法进行选举,造成集群不可用。Etcd使用2379和2380端口。

2379端口：提供HTTP API服务，和etcdctl交互

2380端口：集群中节点间通讯

项目地址：https://github.com/etcd-io/etcd

  

# 环境说明


| 主机名   | ip地址          | OS版本      | 内存、CPU    | 数据库端口     | 部署组件            |
| ----- | ------------- | --------- | --------- | --------- | --------------- |
| node1 | 172.17.44.155 | Centos7.9 | 8C16G200G | 8008,5432 | patroni+pg01    |
| node2 | 172.17.44.156 | Centos7.9 | 8C16G200G | 8008,5432 | patroni+pg02    |
| node3 | 172.17.44.157 | Centos7.9 | 8C16G200G | 8008,5432 | patroni+pg03    |
| node4 | 172.17.44.158 | Centos7.9 | 4C8G200G  | 2379      | etcd-01         |
| node5 | 172.17.44.68  | Centos7.9 | 4C8G200G  | 2379      | etcd-02,archery |
| node6 | 172.17.44.68  | Centos7.9 | 4C8G200G  | 2379      | etcd-03,archery |

**vip：172.17.44.159**


## 添加hosts解析

>**node1~node6上操作**

```bash
[root@wtj1vpk8sql04 pg_cluster_source]# vim /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
172.17.44.155 pg01
172.17.44.156 pg02
172.17.44.157 pg03
172.17.44.158 etcd01 
172.17.44.68 etcd02 
172.17.44.69 etcd03
```
 
## 注意：

>防止避免出现pg服务挂掉时，重启服务器，引起etcd服务的意外，建议服务器资源富余的情况下，优先考虑单独分开部署方案。


# Etcd部署

  

## 安装etcd

  

```Plain
cd /soft/
tar -zxvf etcd-v3.5.13-linux-amd64.tar.gz
cd /soft/etcd-v3.5.13-linux-amd64
cp -r etcd* /usr/bin
cp etcdctl /usr/bin

--查看版本
# etcdctl version
etcdctl version: 3.5.13
API version: 3.3
```

  

## 配置etcd

  

注意：每个节点配置不同。

  

### node4节点操作如下:

  

```bash
mkdir -p /etc/etcd/
mkdir -p /acdata/data/etcd/

vi /etc/etcd/etcd.conf
name: etcd-1
data-dir: /acdata/data/etcd/
listen-client-urls: http://172.17.44.158:2379
advertise-client-urls: http://172.17.44.158:2379
listen-peer-urls: http://172.17.44.158:2380
initial-advertise-peer-urls: http://172.17.44.158:2380
initial-cluster: "etcd-1=http://172.17.44.158:2380,etcd-2=http://172.17.44.68:2380,etcd-3=http://172.17.44.69:2380"
initial-cluster-token: etcd-cluster
initial-cluster-state: new
```

  

### node5节点操作如下:

  

```bash
mkdir -p /etc/etcd/
mkdir -p /acdata/data/etcd/

vi /etc/etcd/etcd.conf

name: etcd-2
data-dir: /acdata/data/etcd/
listen-client-urls: http://172.17.44.68:2379
advertise-client-urls: http://172.17.44.68:2379
listen-peer-urls: http://172.17.44.68:2380
initial-advertise-peer-urls: http://172.17.44.68:2380
initial-cluster: "etcd-1=http://172.17.44.158:2380,etcd-2=http://172.17.44.68:2380,etcd-3=http://172.17.44.69:2380"
initial-cluster-token: etcd-cluster
initial-cluster-state: new

```

  

### node6节点操作如下:

  

```bash
mkdir -p /etc/etcd/
mkdir -p /acdata/data/etcd/
vim /etc/etcd/etcd.conf

name: etcd-3
data-dir: /acdata/data/etcd/
listen-client-urls: http://172.17.44.69:2379
advertise-client-urls: http://172.17.44.69:2379
listen-peer-urls: http://172.17.44.69:2380
initial-advertise-peer-urls: http://172.17.44.69:2380
initial-cluster: "etcd-1=http://172.17.44.158:2380,etcd-2=http://172.17.44.68:2380,etcd-3=http://172.17.44.69:2380"
initial-cluster-token: etcd-cluster
initial-cluster-state: new

```

  

## 配置etcd service 

>**node4,node5,node6上操作**
>
  

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

  >**node4,node5,node6上操作**

```bash
systemctl daemon-reload
systemctl restart etcd
systemctl status etcd
systemctl enable etcd
```

  

## 查看etcd集群的状态

   >**node4,node5,node6上任意一个节点上操作** 
### 查看etcd服务状态

```bash

[root@wtj1vpk8sql04 pg_cluster_source]# systemctl status etcd
● etcd.service - Etcd Server
   Loaded: loaded (/usr/lib/systemd/system/etcd.service; static; vendor preset: disabled)
   Active: active (running) since Fri 2024-08-30 17:14:19 CST; 8min ago
 Main PID: 24372 (etcd)
   CGroup: /system.slice/etcd.service
           └─24372 /usr/bin/etcd --config-file=/etc/etcd/etcd.conf

Aug 30 17:17:57 wtj1vpk8sql04 etcd[24372]: {"level":"warn","ts":"2024-08-30T17:17:57.869982+0800","caller":"rafthtt...
Aug 30 17:17:58 wtj1vpk8sql04 etcd[24372]: {"level":"warn","ts":"2024-08-30T17:17:58.172218+0800","caller":"ra...3d8"}
Aug 30 17:17:58 wtj1vpk8sql04 etcd[24372]: {"level":"info","ts":"2024-08-30T17:17:58.913844+0800","caller":"ra...age"}
Aug 30 17:17:58 wtj1vpk8sql04 etcd[24372]: {"level":"info","ts":"2024-08-30T17:17:58.913889+0800","caller":"ra...3d8"}
Aug 30 17:17:58 wtj1vpk8sql04 etcd[24372]: {"level":"info","ts":"2024-08-30T17:17:58.913905+0800","caller":"ra...3d8"}
Aug 30 17:17:58 wtj1vpk8sql04 etcd[24372]: {"level":"info","ts":"2024-08-30T17:17:58.917337+0800","caller":"ra... v2"}
Aug 30 17:17:58 wtj1vpk8sql04 etcd[24372]: {"level":"warn","ts":"2024-08-30T17:17:58.917360+0800","caller":"ra...3d8"}
Aug 30 17:17:58 wtj1vpk8sql04 etcd[24372]: {"level":"info","ts":"2024-08-30T17:17:58.917370+0800","caller":"ra...3d8"}
Aug 30 17:17:58 wtj1vpk8sql04 etcd[24372]: {"level":"info","ts":"2024-08-30T17:17:58.970290+0800","caller":"ra...3d8"}
Aug 30 17:17:58 wtj1vpk8sql04 etcd[24372]: {"level":"info","ts":"2024-08-30T17:17:58.970415+0800","caller":"ra...3d8"}


```

### 查看集群状态

```bash

[root@node4 pg_cluster_source]# etcdctl  --endpoints=172.17.44.158:2379,172.17.44.68:2379,172.17.44.69:2379 endpoint status -w table
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
|      ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER | IS LEARNER | RAFT TERM | RAFT INDEX | RAFT APPLIED INDEX | ERRORS |
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+
| 172.17.44.158:2379 | 8ae877675b454848 |  3.5.15 |   90 kB |      true |      false |         5 |        230 |                230 |        |
|  172.17.44.68:2379 | 1ddf9fc3680e97d7 |  3.5.15 |   90 kB |     false |      false |         5 |        230 |                230 |        |
|  172.17.44.69:2379 | 9b775eb3953be3d8 |  3.5.15 |   90 kB |     false |      false |         5 |        230 |                230 |        |
+--------------------+------------------+---------+---------+-----------+------------+-----------+------------+--------------------+--------+

[root@wtj1vpk8sql04 ~]# etcdctl --endpoints=172.17.44.158:2379,172.17.44.68:2379,172.17.44.69:2379  endpoint health
172.17.44.158:2379 is healthy: successfully committed proposal: took = 1.881018ms
172.17.44.69:2379 is healthy: successfully committed proposal: took = 1.996743ms
172.17.44.68:2379 is healthy: successfully committed proposal: took = 2.241606ms
[root@wtj1vpk8sql04 ~]# etcdctl --endpoints=172.17.44.158:2379,172.17.44.68:2379,172.17.44.69:2379  endpoint status
172.17.44.158:2379, 8ae877675b454848, 3.5.15, 111 kB, true, false, 5, 325, 325, 
172.17.44.68:2379, 1ddf9fc3680e97d7, 3.5.15, 111 kB, false, false, 5, 325, 325, 
172.17.44.69:2379, 9b775eb3953be3d8, 3.5.15, 111 kB, false, false, 5, 325, 325, 


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

  

```bash
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
  ```bash

Aug 28 16:06:21 wtj1vpk8sql01 patroni: FATAL: Patroni requires psycopg2>=2.5.4, psycopg2-binary, or psycopg>=3.0.0
Aug 28 16:06:21 wtj1vpk8sql01 systemd: patroni.service: main process exited, code=exited, status=1/FAILURE
Aug 28 16:06:21 wtj1vpk8sql01 systemd: Unit patroni.service entered failed state.
Aug 28 16:06:21 wtj1vpk8sql01 systemd: patroni.service failed.
```

```bash
vi /etc/profile.conf

export PGHOME=/acdata/data/pg16/pgsoft
export PGDATA=/acdata/data/pg16/data
export PATH=$PGHOME/bin:$PATH:.
export LD_LIBRARY_PATH=$PGHOME/lib:$LD_LIBRARY_PATH
export PGPORT=5432
export PGUSER=postgres
export PGDATABASE=postgres  
```

## 创建pg16软件安装目录

  

```bash
mkdir -p /acdata/data/pg16/pgsoft
chmod 755 -R /acdata/data/pg16/pgsoft
chown -R postgres:postgres /acdata/data/pg16/pgsoft
```

  

## 创建数据文件存放目录

  

```bash
mkdir -p /acdata/data/pg16/data
chmod -R 700 /acdata/data/pg16/data
chown -R postgres:postgres /acdata/data/pg16/data
```

  

## 解压软件并授权

  

```bash
tar -zxvf /soft/postgresql-16.3.tar.gz 
chown -R postgres:postgres /soft/postgresql-16.3/
```

  

## 编译安装pg16

  

```bash
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

  > 使用watchdog为防止出现脑裂，如果Leader节点异常导致patroni进程无法及时更新watchdog，会在Leader key过期的前5秒触发重启。重启如果在5秒之内完成，Leader节点有机会再次获得Leader锁，否则Leader key过期后，由备库通过选举选出新的Leader。Patroni会在将PostgreSQL提升为master之前尝试激活watchdog。如果看watchdog激活失败并且watchdog模式是required那么节点将拒绝成为主节点。在决定参加领导者选举时，Patroni还将检查watchdog配置是否允许它成为领导者。在将PostgreSQL降级后（例如由于手动故障转移），Patroni将再次禁用watchdog。当 Patroni处于暂停状态时，watchdog也将被禁用。正常停止Patroni服务，也会将watchdog禁用。

> watchdog防止脑裂。Patroni支持通过Linux的watchdog监视patroni进程的运行，当patroni进程无法正常往watchdog设备写入心跳时，由watchdog触发Linux重启。

  

```bash
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

```bash
[postgres@wtj1vpk8sql02 ~]$  pip3 list
Package            Version
------------------ -----------
click              8.0.4
dnspython          2.2.1
importlib-metadata 4.8.3
patroni            3.3.2
pip                21.3.1
prettytable        2.5.0
psutil             6.0.0
psycopg2-binary    2.9.8
python-dateutil    2.9.0.post0
python-etcd        0.4.5
PyYAML             6.0.1
setuptools         39.2.0
six                1.16.0
typing_extensions  4.1.1
urllib3            1.26.19
wcwidth            0.2.13
wheel              0.37.1
ydiff              1.3
zipp               3.6.0
```



```
  
### 报错处理：

```bash
[root@wtj1vpk8sql01 pg_cluster_source]# python3 get-pip.py 
WARNING: Retrying (Retry(total=4, connect=None, read=None, redirect=None, status=None)) after connection broken by 'NewConnectionError('<pip._vendor.urllib3.connection.HTTPSConnection object at 0x7f1535a96898>: Failed to establish a new connection: [Errno 101] Network is unreachable',)': /simple/pip/ 
WARNING: Retrying (Retry(total=3, connect=None, read=None, redirect=None, status=None)) after connection broken by 'NewConnectionError('<pip._vendor.urllib3.connection.HTTPSConnection object at 0x7f1535a96390>: Failed to establish a new connection: [Errno 101] Network is unreachable',)': /simple/pip/ 
```

### 解决办法：
>在 CentOS 7.9 下，你可以通过以下几种方法来更换 `pip` 的源。替换 `pip` 源可以加速 Python 包的安装过程，特别是在国内使用时，推荐使用国内的镜像源如阿里云、清华大学等。

### **方法 1：全局更换 `pip` 源**

要全局更换 `pip` 的源，可以配置 `pip` 的配置文件。这样以后所有的 `pip` 操作都会使用新的源。

1. **创建或编辑 `pip` 配置文件**对于 CentOS 7.9，`pip` 的配置文件通常位于 `~/.pip/pip.conf`（对于当前用户）或 `/etc/pip.conf`（全局配置）。创建目录并编辑文件：

```bash
mkdir -p ~/.pip 
vi ~/.pip/pip.conf
```

或者编辑全局配置文件：

```bash
sudo vi /etc/pip.conf
```

2. **配置国内镜像源**在文件中添加以下内容（根据你的选择替换成对应的源）：

- **阿里云**

```bash
[global] 
index-url = https://mirrors.aliyun.com/pypi/simple/ 

[install] 
trusted-host = mirrors.aliyun.com

```

3. **保存文件并退出**
4. **测试新的 `pip` 源**你可以安装一个包来测试新的源是否生效：
```bash
pip install <package-name>
```

>安装速度应当有所提升，并且在安装日志中可以看到它是从你指定的镜像源下载的包。
>
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


## 注意：

```bash
配置文件/etc/patroni/patroni_env.conf,可以使用如果命令生成：
# touch /etc/patroni/patroni_env.conf
# chown -R postgres.postgres /etc/patroni/patroni_env.conf

$ su - postgres
$ env > /etc/patroni/patroni_env.conf

$ cat /etc/patroni/patroni_env.conf

XDG_SESSION_ID=533
HOSTNAME=wtj1vpk8sql03
SHELL=/bin/bash
TERM=xterm
HISTSIZE=1000
USER=postgres
PGPORT=5432
LD_LIBRARY_PATH=/acdata/data/pg16/pgsoft/lib:
LS_COLORS=rs=0:di=01;34:ln=01;36:mh=00:pi=40;33:so=01;35:do=01;35:bd=40;33;01:cd=40;33;01:or=40;31;01:mi=01;05;37;41:su=37;41:sg=30;43:ca=30;41:tw=30;42:ow=34;42:st=37;44:ex=01;32:*.tar=01;31:*.tgz=01;31:*.arc=01;31:*.arj=01;31:*.taz=01;31:*.lha=01;31:*.lz4=01;31:*.lzh=01;31:*.lzma=01;31:*.tlz=01;31:*.txz=01;31:*.tzo=01;31:*.t7z=01;31:*.zip=01;31:*.z=01;31:*.Z=01;31:*.dz=01;31:*.gz=01;31:*.lrz=01;31:*.lz=01;31:*.lzo=01;31:*.xz=01;31:*.bz2=01;31:*.bz=01;31:*.tbz=01;31:*.tbz2=01;31:*.tz=01;31:*.deb=01;31:*.rpm=01;31:*.jar=01;31:*.war=01;31:*.ear=01;31:*.sar=01;31:*.rar=01;31:*.alz=01;31:*.ace=01;31:*.zoo=01;31:*.cpio=01;31:*.7z=01;31:*.rz=01;31:*.cab=01;31:*.jpg=01;35:*.jpeg=01;35:*.gif=01;35:*.bmp=01;35:*.pbm=01;35:*.pgm=01;35:*.ppm=01;35:*.tga=01;35:*.xbm=01;35:*.xpm=01;35:*.tif=01;35:*.tiff=01;35:*.png=01;35:*.svg=01;35:*.svgz=01;35:*.mng=01;35:*.pcx=01;35:*.mov=01;35:*.mpg=01;35:*.mpeg=01;35:*.m2v=01;35:*.mkv=01;35:*.webm=01;35:*.ogm=01;35:*.mp4=01;35:*.m4v=01;35:*.mp4v=01;35:*.vob=01;35:*.qt=01;35:*.nuv=01;35:*.wmv=01;35:*.asf=01;35:*.rm=01;35:*.rmvb=01;35:*.flc=01;35:*.avi=01;35:*.fli=01;35:*.flv=01;35:*.gl=01;35:*.dl=01;35:*.xcf=01;35:*.xwd=01;35:*.yuv=01;35:*.cgm=01;35:*.emf=01;35:*.axv=01;35:*.anx=01;35:*.ogv=01;35:*.ogx=01;35:*.aac=01;36:*.au=01;36:*.flac=01;36:*.mid=01;36:*.midi=01;36:*.mka=01;36:*.mp3=01;36:*.mpc=01;36:*.ogg=01;36:*.ra=01;36:*.wav=01;36:*.axa=01;36:*.oga=01;36:*.spx=01;36:*.xspf=01;36:
PGUSER=postgres
PGDATABASE=postgres
MAIL=/var/spool/mail/postgres
PATH=/acdata/data/pg16/pgsoft/bin:/usr/local/bin:/usr/local/bin:/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/usr/bin/python3:/usr/local/bin/pip3:/home/postgres/bin:.:/home/postgres/.local/bin:/home/postgres/bin
PWD=/home/postgres
LANG=en_US.UTF-8
PGHOME=/acdata/data/pg16/pgsoft
HISTCONTROL=ignoredups
SHLVL=1
HOME=/home/postgres
LOGNAME=postgres
PGDATA=/acdata/data/pg16/data
LESSOPEN=||/usr/bin/lesspipe.sh %s
PATRONICTL_CONFIG_FILE=/etc/patroni/patroni.yml
_=/bin/env
```

## 配置patroni

  

```bash

# **不同节点需要修改对应的IP信息，以及开头的name信息**

# node1节点:

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
  hosts: 172.17.44.158:2379,172.17.44.68:2379,172.17.44.69:2379

bootstrap:
 # this section will be written into Etcd:/<namespace>/<scope>/config after initializing new cluster and all other cluster members will use it as a `global configuration`
  dcs:
    ttl: 30
    loop_wait: 10             # 循环更新领导者密钥过程中的休眠时间
    retry_timeout: 30         # etcd和PostgreSQL操作重试的超时时间（以秒为单位）
    maximum_lag_on_failover: 1048576  # 如果Master和Replicate之间的字节数延迟大于此值，那么Replicate将不参与新的领导者选举
    master_start_timeout: 300
    synchronous_mode: true    # 是否打开同步复制模式
    postgresql:               # PostgreSQL的配置，是否使用pg_rewind，是否使用复制插槽，还有PostgreSQL参数等信息
      use_pg_rewind: true
      use_slots: true
      parameters:
        max_connections: 8000
        superuser_reserved_connections: 10
        shared_buffers: 4GB
        wal_buffers: 1GB
        effective_cache_size:  12GB
        work_mem: 4MB
        maintenance_work_mem: 512MB
        max_wal_size: 2GB
        min_wal_size: 256MB
        idle_in_transaction_session_timeout: 30000 #默认值为 0（禁用）。设置事务空闲超时时间，有助于防止长时间的空闲事务占用资源
        effective_io_concurrency: 8  #默认值为 1。用于设置并发 IO 请求的数量。对于 SSD 或高速磁盘阵列，建议调高该值（如 200）以提高 IO 性能
        parallel_workers_per_gather: 8 #默认值为 2。控制查询时用于并行执行的 worker 数量。根据查询复杂度和系统资源，适当增加并行度以提高查询性能
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

  initdb:                 # 定义了在引导过程中要传递给initdb的选项
   - encoding: UTF8
   - locale: C
   - lc-ctype: zh_CN.UTF-8
   - data-checksums

#  pg_hba:                # 定义了集群初始化后，pg_hba.conf中该设置的条目
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
  listen: 172.17.44.155:5432
  connect_address: 172.17.44.155:5432
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
callbacks:
    on_start: /etc/patroni/patroni_callback.sh
    on_stop:  /etc/patroni/patroni_callback.sh
    on_role_change: /etc/patroni/patroni_callback.sh

watchdog:
  mode: automatic       # Allowed values: off, automatic, required
  device: /dev/watchdog
  safety_margin: 5

tags:
   nofailover: false       # 不参与选主
   noloadbalance: false    # 不参与负载均衡
   clonefrom: false
   nosync: false           # 也不作为同步备库



scope: pg-sentry-cluster01
namespace: /service/
name: pg01

restapi:
  listen: 0.0.0.0:8008
  connect_address: 172.17.44.155:8008

etcd3:
  hosts: 172.17.44.158:2379,172.17.44.68:2379,172.17.44.69:2379

bootstrap:
 # this section will be written into Etcd:/<namespace>/<scope>/config after initializing new cluster and all other cluster members will use it as a `global configuration`
  dcs:
    ttl: 30
    loop_wait: 10
    retry_timeout: 10
    maximum_lag_on_failover: 1048576
    master_start_timeout: 300
    synchronous_mode: true
    postgresql:
      use_pg_rewind: true
      use_slots: true
      parameters:
         #listen_addresses: '*'
         #port: 5432
         max_connections: 8000
         superuser_reserved_connections: 10
         shared_buffers: 4GB
         wal_buffers: 1GB
         effective_cache_size:  12GB
         work_mem: 4MB
         maintenance_work_mem: 512MB
         max_wal_size: 2GB
         min_wal_size: 256MB
         idle_in_transaction_session_timeout: 30000 
         effective_io_concurrency: 8 
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
  listen: 0.0.0.0:5432
  connect_address: 172.17.44.155:5432
  data_dir: /acdata/data/pg16/data
  bin_dir: /acdata/data/pg16/pgsoft/bin

  parameters:
    listen_addresses: '*'
    port: 5432

  authentication:
    replication:
      username: repuser
      password: '123456'
    superuser:
      username: postgres
      password: 'FEHdSQQEB8' 
      #password: postgres 

basebackup:
    max-rate: 100M
    checkpoint: fast

callbacks:
    on_start: /bin/bash /etc/patroni/patroni_callback.sh
    on_stop:  /bin/bash /etc/patroni/patroni_callback.sh
    on_role_change: /bin/bash /etc/patroni/patroni_callback.sh

#watchdog:
#  mode: automatic       # Allowed values: off, automatic, required
#  device: /dev/watchdog
#  safety_margin: 5

tags:
   nofailover: false
   noloadbalance: false
   clonefrom: false
   nosync: false





# node2节点:

scope: pgsql16
namespace: /pgsql/
name: pg02

restapi:
  listen: 0.0.0.0:8008
  connect_address: 172.17.44.156:8008

etcd3:
  hosts: 172.17.44.158:2379,172.17.44.68:2379,172.17.44.69:2379

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
   
  
# node3节点:

scope: pgsql16
namespace: /pgsql/
name: pg03

restapi:
  listen: 0.0.0.0:8008
  connect_address: 172.17.44.157:8008

etcd3:
  hosts: 172.17.44.158:2379,172.17.44.68:2379,172.17.44.69:2379

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
  listen: 172.17.44.157:5432
  connect_address: 172.17.44.157:5432
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
```


### 注意：
- 第一次启动初始化失败后，再次启动无法重新初始化，需要删除/acdata/data/pg16/data下的所有文件，以及更换namespace: /pgsql/为新的名称，如：namespace: /pgsql16/
- 再重新启动，进行初始化操作即可成功

  

## 创建patroni的callbak脚本

  

```bash

cat >> /etc/patroni/patroni_callback.sh <<EOF

#!/bin/bash

readonly OPERATION=$1
readonly ROLE=$2
readonly SCOPE=$3

VIP='172.17.44.159'
PREFIX='22'
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

  > **注意：**
  > 第一次需要手动运行如下命令，添加一下vip，后续切换不需要手动干预。
  
  ```bash
 [root@wtj1vpk8sql01 ~]# sh /etc/patroni/patroni_callback.sh on_start master pgsql16
2024-09-03 14:59:11 +0800 This is patroni callback on_start master pg-sentry-prod
2024-09-03 14:59:11 +0800 VIP 172.17.44.159 added


[root@wtj1vpk8sql01 ~]# ip ad sh|grep 172.17
    inet 172.17.44.155/22 brd 172.17.47.255 scope global noprefixroute ens192
    inet 172.17.44.159/22 brd 172.17.47.255 scope global secondary ens192:1

```


## 3台节点启动patroni集群

  

```bash

# 方法一:直接启动：
su - postgres
patroni /etc/patroni/patroni.yml

# 方法二：服务启动
systemctl start patroni
```

  

## 查看patroni集群状态

  

```bash
[root@wtj1vpk8sql01 ~]# patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pgsql16 (7408855315163097790) -----------+----+-----------+
| Member | Host          | Role         | State     | TL | Lag in MB |
+--------+---------------+--------------+-----------+----+-----------+
| pg01   | 172.17.44.155 | Leader       | running   |  1 |           |
| pg02   | 172.17.44.156 | Sync Standby | streaming |  1 |         0 |
| pg03   | 172.17.44.157 | Replica      | streaming |  1 |         0 |
+--------+---------------+--------------+-----------+----+-----------+
```

### 主节点pg01上的启动日志：

```bash
024-08-29 15:08:59,731 INFO: no action. I am (pg01), the leader with the lock
2024-08-29 15:09:00,731 INFO: no action. I am (pg01), the leader with the lock
2024-08-29 15:09:01,731 INFO: no action. I am (pg01), the leader with the lock
2024-08-29 15:09:02,731 INFO: no action. I am (pg01), the leader with the lock
2024-08-29 15:09:03,733 INFO: no action. I am (pg01), the leader with the lock
2024-08-29 15:09:04,731 INFO: no action. I am (pg01), the leader with the lock
```
### 从节点pg02和pg03上的启动日志：

```bash

**pg02节点：**

2024-08-29 15:12:52,706 INFO: no action. I am (pg02), a secondary, and following a leader (pg01)
2024-08-29 15:12:54,687 INFO: no action. I am (pg02), a secondary, and following a leader (pg01)
2024-08-29 15:12:54,688 WARNING: Loop time exceeded, rescheduling immediately.
2024-08-29 15:12:54,689 INFO: no action. I am (pg02), a secondary, and following a leader (pg01)
2024-08-29 15:12:56,230 INFO: no action. I am (pg02), a secondary, and following a leader (pg01)
2024-08-29 15:12:57,189 INFO: no action. I am (pg02), a secondary, and following a leader (pg01)
2024-08-29 15:12:58,231 INFO: no action. I am (pg02), a secondary, and following a leader (pg01)


**pg03节点：**
2024-08-29 15:13:10,318 INFO: no action. I am (pg03), a secondary, and following a leader (pg01)
2024-08-29 15:13:11,276 INFO: no action. I am (pg03), a secondary, and following a leader (pg01)
2024-08-29 15:13:12,319 INFO: no action. I am (pg03), a secondary, and following a leader (pg01)
2024-08-29 15:13:13,277 INFO: no action. I am (pg03), a secondary, and following a leader (pg01)
2024-08-29 15:13:14,319 INFO: no action. I am (pg03), a secondary, and following a leader (pg01)
2024-08-29 15:13:15,277 INFO: no action. I am (pg03), a secondary, and following a leader (pg01)
2024-08-29 15:13:15,912 INFO: no action. I am (pg03), a secondary, and following a leader (pg01)

```

# 主节点node1上查看复制状态

  

```bash

# psql

postgres=# select * from pg_stat_replication;
 pid  | usesysid | usename | application_name |  client_addr  | client_hostname | client_port |         backend_start 
        | backend_xmin |   state   | sent_lsn  | write_lsn | flush_lsn | replay_lsn | write_lag | flush_lag | replay_l
ag | sync_priority | sync_state |          reply_time           
------+----------+---------+------------------+---------------+-----------------+-------------+-----------------------
--------+--------------+-----------+-----------+-----------+-----------+------------+-----------+-----------+---------
---+---------------+------------+-------------------------------
 2805 |    16384 | repuser | pg02             | 172.17.44.156 |                 |       50462 | 2024-08-30 17:06:59.06
6348+08 |              | streaming | 0/5000148 | 0/5000148 | 0/5000148 | 0/5000148  |           |           |         
   |             1 | sync       | 2024-08-30 18:03:52.644567+08
 2811 |    16384 | repuser | pg03             | 172.17.44.157 |                 |       40864 | 2024-08-30 17:08:20.94
7125+08 |              | streaming | 0/5000148 | 0/5000148 | 0/5000148 | 0/5000148  |           |           |         
   |             0 | async      | 2024-08-30 18:03:48.894924+08
(2 rows)
```

  ![[Pasted image 20240830184606.png]]


[root@wtj1vpk8sql01 ~]# sh /etc/patroni/patroni_callback.sh on_start master pgsql
2024-09-02 09:51:23 +0800 This is patroni callback on_start master pgsql
2024-09-02 09:51:23 +0800 VIP 172.17.44.159 added


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

## 模拟pg02的pg故障，发现无法正常启动

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

通过日志内容`2024-08-29 17:53:55.311 CST [15094] FATAL:  WAL was generated with wal_level=minimal, cannot continue recovering`，不难发现问题出现wal_level=minimal。
集群中配置的wal_level=replica并没有生效。

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

#### Patroni 进阶设定

##### Patroni 故障自动修复

| 故障位置 | 场景                            | Patroni 的动作                                 |
| ---- | ----------------------------- | ------------------------------------------- |
| 备库   | 备库 PG 停止                      | 停止备库 PG                                     |
| 备库   | 停止备库 Patroni                  | 停止备库 PG                                     |
| 备库   | 强杀备库 Patroni（或 Patroni crash） | 无操作                                         |
| 备库   | 备库无法连接 etcd                   | 无操作                                         |
| 备库   | 非 Leader 角色但是 PG 处于生产模式       | 重启 PG 并切换到恢复模式作为备库运行                        |
| 主库   | 主库 PG 停止                      | 重启 PG，重启超过 master_start_timeout 设定时间，进行主备切换 |
| 主库   | 停止主库 Patroni                  | 停止主库 PG，并触发 failover                        |
| 主库   | 强杀主库 Patroni（或 Patroni crash） | 触发 failover，此时可能出现双主                        |
| 主库   | 主库无法连接 etcd                   | 将主库降级为备库，并触发 failover                       |
| -    | etcd 集群故障                     | 将主库降级为备库，此时集群中全部都是备库                        |
| -    | 同步模式下无可用同步备库                  | 临时切换主库为异步复制，在恢复为同步复制之前自动 failover 暂不生效      |


# 遗留问题：

>**patroni的callback脚本调用失败，vip不能自动生成，以及自动漂移。**
>

# 参考文档

  

https://patroni.readthedocs.io/en/latest/README.html

https://blog.csdn.net/lsx_3/article/details/131072594

  