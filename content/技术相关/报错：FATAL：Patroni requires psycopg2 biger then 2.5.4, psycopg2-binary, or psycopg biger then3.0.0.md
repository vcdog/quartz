
#  使用 systemctl start patroni.service启动报错：

>FATAL: Patroni requires psycopg2>=2.5.4, psycopg2-binary, or psycopg>=3.0.0


# 详细报错：

```bash
[root@wtj1vpk8sql02 ~]# systemctl start patroni.service
[root@wtj1vpk8sql02 ~]# ps -ef|grep patro
root      2351  1769  0 14:23 pts/0    00:00:00 grep --color=auto patro
[root@wtj1vpk8sql02 ~]# 
[root@wtj1vpk8sql02 ~]# systemctl status patroni.service -l 
● patroni.service - Runners to orchestrate a high-availability PostgreSQL
   Loaded: loaded (/etc/systemd/system/patroni.service; enabled; vendor preset: disabled)
   Active: failed (Result: exit-code) since Mon 2024-09-02 14:23:18 CST; 13s ago
  Process: 2346 ExecStart=/usr/local/bin/patroni /etc/patroni/patroni.yml (code=exited, status=1/FAILURE)
  Process: 2341 ExecStartPre=/usr/bin/sudo /bin/chown postgres /dev/watchdog (code=exited, status=0/SUCCESS)
  Process: 2335 ExecStartPre=/usr/bin/sudo /sbin/modprobe softdog (code=exited, status=0/SUCCESS)
 Main PID: 2346 (code=exited, status=1/FAILURE)

Sep 02 14:23:18 wtj1vpk8sql02 systemd[1]: Starting Runners to orchestrate a high-availability PostgreSQL...
Sep 02 14:23:18 wtj1vpk8sql02 sudo[2335]: postgres : TTY=unknown ; PWD=/ ; USER=root ; COMMAND=/sbin/modprobe softdog
Sep 02 14:23:18 wtj1vpk8sql02 sudo[2341]: postgres : TTY=unknown ; PWD=/ ; USER=root ; COMMAND=/bin/chown postgres /dev/watchdog
Sep 02 14:23:18 wtj1vpk8sql02 systemd[1]: Started Runners to orchestrate a high-availability PostgreSQL.
Sep 02 14:23:18 wtj1vpk8sql02 patroni[2346]: FATAL: Patroni requires psycopg2>=2.5.4, psycopg2-binary, or psycopg>=3.0.0
Sep 02 14:23:18 wtj1vpk8sql02 systemd[1]: patroni.service: main process exited, code=exited, status=1/FAILURE
Sep 02 14:23:18 wtj1vpk8sql02 systemd[1]: Unit patroni.service entered failed state.
Sep 02 14:23:18 wtj1vpk8sql02 systemd[1]: patroni.service failed.
```

# 经过排查，是因为环境变量的问题。

```bash
[root@wtj1vpk8sql01 ~]# cat /usr/lib/systemd/system/patroni.service
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
ExecReload=/bin/kill -s HUP $MAINPID
KillMode=process
TimeoutSec=30
Restart=no
  
[Install]
WantedBy=multi-user.target

```

# 生成postgres用户配置文件


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

# 重启patroni

```bash
[root@wtj1vpk8sql02 ~]# systemctl start patroni
[root@wtj1vpk8sql02 ~]# systemctl status patroni -l
● patroni.service - Runners to orchestrate a high-availability PostgreSQL
   Loaded: loaded (/etc/systemd/system/patroni.service; enabled; vendor preset: disabled)
   Active: active (running) since Mon 2024-09-02 15:10:00 CST; 7s ago
  Process: 3003 ExecStartPre=/usr/bin/sudo /bin/chown postgres /dev/watchdog (code=exited, status=0/SUCCESS)
  Process: 2998 ExecStartPre=/usr/bin/sudo /sbin/modprobe softdog (code=exited, status=0/SUCCESS)
 Main PID: 3007 (patroni)
   CGroup: /system.slice/patroni.service
           ├─3007 /usr/bin/python3 /usr/local/bin/patroni /etc/patroni/patroni.yml
           ├─3035 /acdata/data/pg16/pgsoft/bin/postgres -D /acdata/data/pg16/data --config-file=/acdata/data/pg16/data/postgresql.conf --listen_addresses=172.17.44.156 --port=5432 --cluster_name=pgsql16 --wal_level=replica --hot_standby=on --max_connections=300 --max_wal_senders=10 --max_prepared_transactions=0 --max_locks_per_transaction=64 --track_commit_timestamp=off --max_replication_slots=10 --max_worker_processes=8 --wal_log_hints=on
           ├─3037 postgres: pgsql16: logger                                      
           ├─3038 postgres: pgsql16: checkpointer                                
           ├─3039 postgres: pgsql16: background writer           
           ├─3040 postgres: pgsql16: startup recovering 000000040000000000000005            ├─3047 postgres: pgsql16: walreceiver streaming 0/5001958                        └─3049 postgres: pgsql16: postgres postgres 172.17.44.156(49480) idle                                                                                                                                                                  
Sep 02 15:10:04 wtj1vpk8sql02 patroni[3007]: 2024-09-02 15:10:04,310 INFO: Lock owner: pg01; I am pg02
Sep 02 15:10:04 wtj1vpk8sql02 patroni[3007]: 2024-09-02 15:10:04,353 INFO: restarting after failure in progress
Sep 02 15:10:04 wtj1vpk8sql02 patroni[3007]: 172.17.44.156:5432 - accepting connections
Sep 02 15:10:04 wtj1vpk8sql02 patroni[3007]: 2024-09-02 15:10:04,557 INFO: Lock owner: pg01; I am pg02
Sep 02 15:10:04 wtj1vpk8sql02 patroni[3007]: 2024-09-02 15:10:04,558 INFO: establishing a new patroni heartbeat connection to postgres
Sep 02 15:10:06 wtj1vpk8sql02 patroni[3007]: 2024-09-02 15:10:06,586 INFO: Dropped unknown replication slot 'pg03'
Sep 02 15:10:06 wtj1vpk8sql02 patroni[3007]: 2024-09-02 15:10:06,587 INFO: Dropped unknown replication slot 'pg01'
Sep 02 15:10:06 wtj1vpk8sql02 patroni[3007]: 2024-09-02 15:10:06,673 INFO: no action. I am (pg02), a secondary, and following a 

leader (pg01)
Sep 02 15:10:06 wtj1vpk8sql02 patroni[3007]: 2024-09-02 15:10:06,674 WARNING: Loop time exceeded, rescheduling immediately.
Sep 02 15:10:06 wtj1vpk8sql02 patroni[3007]: 2024-09-02 15:10:06,676 INFO: no action. I am (pg02), a secondary, and following a leader (pg01)

```

# patroni常用命令


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