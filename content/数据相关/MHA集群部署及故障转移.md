# 一、环境配置
本环境共有四个节点， 其角色分配如下（实验机器均为centos 7.x）

| 机器名称    | IP配置          | 服务角色       |
| ------- | ------------- | ---------- |
| manager | 172.17.44.144 | manager控制器 |
| master  | 172.17.44.45  | 数据库主服务器    |
| slave1  | 172.17.44.44  | 数据库从服务器    |
| slave2  | 172.17.44.143 | 数据库从服务器    |
为了方便我们后期的操作，我们在各节点的/etc/hosts，配置内容中添加如下内容：
```bash

[root@wtj1vpztmysql02 ~]# cat /etc/hosts
#mha manager and node
172.17.44.45 mha-master-01
172.17.44.143 mha-master-02
172.17.44.44 mha-slave-01
172.17.44.144 mha-manager-01
```

# 二、安装主从复制集群

# 1、下载安装脚本及mysql安装包

```Bash
# mkdir /acdata/dba_tools/
-- 上传压缩包到/acdata/dba_tools，并解压缩

# tar xvfz deploy_mysql_install_mysql_init_mult_version_news.2024-06-20.tar.gz
# cd deploy_mysql
# wget https://cdn.mysql.com/archives/mysql-8.0/mysql-8.0.28-linux-glibc2.12-x86_64.tar.xz
# ls
install_mysql_init_mult_version_news.sh  mysql_centos_ubuntu_conf  usr mysql-8.0.28-linux-glibc2.12-x86_64.tar.xz
```

# 2、安装数据库

```YAML
# bash install_mysql_init_mult_version_news.sh 
Illegal Syntax: install_mysql_init_mult_version_news.sh :usage: [-s server_id]  [-m mem_size] [-c core_num]  [-P port] [-v my_version]
Illegal Syntax: install_mysql_init_mult_version_news.sh :usage: [-d drop_instance]  [-P port]
建议选择安装mysql-5.7.20~5.7.39,或8.0.16~8.0.28版本

参数说明：
[-s server_id]     -- 指定server_id，范围1-65535
[-m mem_size]      -- 指定内存大小，单位：G，例如：4
[-c core_num]      -- 指定cpu核心数，数字，例如：8
[-P port]          -- 指定数据库的端口号，例如：3306或3307等
[-v my_version]    -- 指定数据库的版本号，例如：8.0.28

安装的例子：
# bash install_mysql_init_mult_version_news.sh  -s 1001 -m 4 -c 8 -P 3307 -v 8.0.28
```

# 3、安装成功后，登录测试

安装成功后，会打印如下信息：

```SQL
##########################################
db_host:172.17.44.xx
db_name:all databases
db_port:3307
db_user:root
db_pwd:xxxxxxxx
##########################################
```

进行登录测试验证：

```SQL
# mysql -h 172.17.44.xx -u root -p'xxxxxx' -P 3307
```

# 4、主从库节点的配置文件
## `主库：172.17.44.45`

```bash


[root@wtj1vpztmysql02 ~]# cat /etc/my_3307.cnf |sed  "/^$/d"
[client]
port=3307
socket=/acdata/data/mysqlsoft3307/mysql.sock
[mysqld]
port=3307
mysqlx_port=33070
admin_port=33072
report_host="172.17.44.45"
   
server-id=1045
datadir=/acdata/data/mysqldb3307
basedir=/acdata/data/mysqlsoft3307
socket=/acdata/data/mysqlsoft3307/mysql.sock
   
 
mysqlx_socket=/acdata/data/mysqlsoft3307/mysqlx_3307.sock
   
user=mysql
#symbolic-links=0
default_storage_engine=innodb
default_time_zone = '+8:00'
character_set_server=utf8mb4
collation_server = utf8mb4_unicode_ci
explicit_defaults_for_timestamp=1
# 默认使用mysql_native_password插件认证
default_authentication_plugin=mysql_native_password
#用来控制select语句的最大执行时间，单位是毫秒
max_execution_time=300000
#innodb_flush_log_at_trx_commit && sync_binlog
innodb_flush_log_at_trx_commit=2
sync_binlog=1000
#innodb_flush_log_at_trx_commit=1
#sync_binlog=1
#add slow_query_log
slow_query_log=1
slow_query_log_file=/acdata/data/mysqlsoft3307/slowlog/slowquery_3307.log
long_query_time=0.5
#log_queries_not_using_indexes = 1
log_queries_not_using_indexes = 0
log_slow_admin_statements = 1
   
#mysql-8.0
log_slow_replica_statements=1
#log_throttle_queries_not_using_indexes = 10
   
max_connections=5000
max_user_connections=1800
max_connect_errors = 100
back_log=500
max_allowed_packet =1G
#innodb configure
innodb_flush_method = O_DIRECT
innodb_open_files=65535
open_files_limit =65535
innodb_file_per_table=1
innodb_buffer_pool_size=2G
innodb_log_buffer_size = 16M
#innodb_lock_wait_timeout
innodb_lock_wait_timeout=20
#close query cache
#query_cache_type=0
#query_cache_size =0
sort_buffer_size=4M
join_buffer_size=4M
read_buffer_size=8M
read_rnd_buffer_size=8M
tmpdir=/dev/shm
tmp_table_size=1G
#innodb_logfile
innodb_log_file_size=128M
#transaction_isolation=READ-COMMITTED
thread_cache_size=1000
innodb_read_io_threads=64
innodb_write_io_threads=64
innodb_io_capacity=600
#thread_concurrency=64
#table_cache = 512
key_buffer_size =128M
#log-bin=mysql-bin
replicate-ignore-db=mysql
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
replicate-ignore-db=test
#master-connect-retry=60
#rpl_semi_sync_master_enabled=1 
#rpl_semi_sync_master_timeout=3000  
#rpl_semi_sync_slave_enabled=1
log_bin_trust_function_creators = 1
   
#mysql-8.0
skip_replica_start
#mysql-8.0
#replica_skip_errors=1032,1062
#mysql-8.0
log_replica_updates
#mysql-8.0
replica_skip_errors = ddl_exist_errors
########Ignore upper_lower case and set sql_mode
lower_case_table_names=1
   
skip-name-resolve
########Ignore upper_lower case and set sql_mode
#lower_case_table_names=1
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
########replication settings########
#master_info_repository = TABLE
#relay_log_info_repository = TABLE
log_bin=mysql-bin-sunac_3307
#sync_binlog = 1
gtid_mode = on
enforce_gtid_consistency = 1
binlog_format = row 
#relay_log = relay.log
relay_log_recovery = 1
binlog_gtid_simple_recovery = 1
#Secure File path
#secure_auth=1
#secure_file_priv=/acdata/data/backup
########semi sync replication settings########
plugin_dir=/acdata/data/mysqlsoft3307/lib/plugin
   
 
#mysql-8.0
plugin_load = "rpl_semi_sync_source=semisync_source.so;rpl_semi_sync_replica=semisync_replica.so"
#mysql-8.0
loose_rpl_semi_sync_source_enabled = 1
loose_rpl_semi_sync_replica_enabled = 1
loose_rpl_semi_sync_source_timeout = 3000
###############slave_parallel################
#mysql-8.0
replica_parallel_type=LOGICAL_CLOCK
replica_parallel_workers=2
   
 
# binlog_transaction_dependency_tracking
# COMMIT_ORDER：    默认值，它使用 MySQL 5.7 中可用的默认机制。
# WRITESET：        能实现更好的并行化，并且在主库的二进制日志中存储writeset数据。
# WRITESET_SESSION：确保事务在从库中按顺序执行，并且消除了从库中看到主库从未出现过的数据库状态的问题。
#                   降低了并行化程度���但是仍然提供了比默认设置更好的吞吐量。
# master node add this option
binlog_transaction_dependency_tracking=WRITESET_SESSION
#############################################
#[mysqld-5.7]
innodb_buffer_pool_dump_pct = 40
innodb_page_cleaners = 4
innodb_undo_log_truncate = 1
innodb_max_undo_log_size = 2G
innodb_purge_rseg_truncate_frequency = 128
binlog_gtid_simple_recovery=1
log_timestamps=system
#记录事务的算法,官网建议设置该参数使用 XXHASH64 算法
   
  
#mysql-8.0 
transaction_write_set_extraction=XXHASH64
   
  
#relay_log
relay_log=/acdata/data/mysqldb3307/mysql_3307_relay_bin
relay_log_index=/acdata/data/mysqldb3307/mysql_3307_relay_bin.index
#binlog_format
binlog_format=ROW
max_binlog_size=512M
   
  
#mysql-8.0 
binlog_expire_logs_seconds = 1209600
   
  
#add timeout configure
wait_timeout = 172800
interactive-timeout = 172800
[mysqld_safe]
log-error=/acdata/data/mysqldb3307/mysqld_3307.err
pid-file=/acdata/data/mysqldb3307/mysqld_3307.pid
```


## `从库：172.17.44.44`

```bash

[root@wtj1vpztmysql01 deploy_mysql]# cat /etc/my_3307.cnf |sed "/^$/d"
[client]
port=3307
socket=/acdata/data/mysqlsoft3307/mysql.sock
[mysqld]
port=3307
mysqlx_port=33070
admin_port=33072
report_host="172.17.44.44"
   
server-id=1044
datadir=/acdata/data/mysqldb3307
basedir=/acdata/data/mysqlsoft3307
socket=/acdata/data/mysqlsoft3307/mysql.sock
   
 
mysqlx_socket=/acdata/data/mysqlsoft3307/mysqlx_3307.sock
   
user=mysql
#symbolic-links=0
default_storage_engine=innodb
default_time_zone = '+8:00'
character_set_server=utf8mb4
collation_server = utf8mb4_unicode_ci
explicit_defaults_for_timestamp=1
# 默认使用mysql_native_password插件认证
default_authentication_plugin=mysql_native_password
#用来控制select语句的最大执行时间，单位是毫秒
max_execution_time=300000
#innodb_flush_log_at_trx_commit && sync_binlog
innodb_flush_log_at_trx_commit=2
sync_binlog=1000
#innodb_flush_log_at_trx_commit=1
#sync_binlog=1
#add slow_query_log
slow_query_log=1
slow_query_log_file=/acdata/data/mysqlsoft3307/slowlog/slowquery_3307.log
long_query_time=0.5
#log_queries_not_using_indexes = 1
log_queries_not_using_indexes = 0
log_slow_admin_statements = 1
   
#mysql-8.0
log_slow_replica_statements=1
#log_throttle_queries_not_using_indexes = 10
   
max_connections=5000
max_user_connections=1800
max_connect_errors = 10000
back_log=500
max_allowed_packet =1G
#innodb configure
innodb_flush_method = O_DIRECT
innodb_open_files=65535
open_files_limit =65535
innodb_file_per_table=1
innodb_buffer_pool_size=2G
innodb_log_buffer_size = 16M
#innodb_lock_wait_timeout
innodb_lock_wait_timeout=20
#close query cache
#query_cache_type=0
#query_cache_size =0
sort_buffer_size=4M
join_buffer_size=4M
read_buffer_size=8M
read_rnd_buffer_size=8M
tmpdir=/dev/shm
tmp_table_size=1G
#innodb_logfile
innodb_log_file_size=128M
#transaction_isolation=READ-COMMITTED
thread_cache_size=1000
innodb_read_io_threads=64
innodb_write_io_threads=64
innodb_io_capacity=600
#thread_concurrency=64
#table_cache = 512
key_buffer_size =128M
#log-bin=mysql-bin
replicate-ignore-db=mysql
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
replicate-ignore-db=test
#master-connect-retry=60
#rpl_semi_sync_master_enabled=1 
#rpl_semi_sync_master_timeout=3000  
#rpl_semi_sync_slave_enabled=1
log_bin_trust_function_creators = 1
   
#mysql-8.0
skip_replica_start
#mysql-8.0
#replica_skip_errors=1032,1062
#mysql-8.0
log_replica_updates
#mysql-8.0
replica_skip_errors = ddl_exist_errors
########Ignore upper_lower case and set sql_mode
lower_case_table_names=1
   
skip-name-resolve
########Ignore upper_lower case and set sql_mode
#lower_case_table_names=1
sql_mode=STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION
########replication settings########
#master_info_repository = TABLE
#relay_log_info_repository = TABLE
log_bin=mysql-bin-sunac_3307
#sync_binlog = 1
gtid_mode = on
enforce_gtid_consistency = 1
binlog_format = row 
#relay_log = relay.log
relay_log_recovery = 1
binlog_gtid_simple_recovery = 1
#Secure File path
#secure_auth=1
#secure_file_priv=/acdata/data/backup
########semi sync replication settings########
plugin_dir=/acdata/data/mysqlsoft3307/lib/plugin
   
 
#mysql-8.0
plugin_load = "rpl_semi_sync_source=semisync_source.so;rpl_semi_sync_replica=semisync_replica.so"
#mysql-8.0
loose_rpl_semi_sync_source_enabled = 1
loose_rpl_semi_sync_replica_enabled = 1
loose_rpl_semi_sync_source_timeout = 3000
###############slave_parallel################
#mysql-8.0
replica_parallel_type=LOGICAL_CLOCK
replica_parallel_workers=4
   
 
# binlog_transaction_dependency_tracking
# COMMIT_ORDER：    默认值，它使用 MySQL 5.7 中可用的默认机制。
# WRITESET：        能实现更好的并行化，并且在主库的二进制日志中存储writeset数据。
# WRITESET_SESSION：确保事务在从库中按顺序执行，并且消除了从库中看到主库从未出现过的数据库状态的问题。
#                   降低了并行化程度，但是仍然提供了比默认设置更好的吞吐���。
# master node add this option
binlog_transaction_dependency_tracking=WRITESET_SESSION
#############################################
#[mysqld-5.7]
innodb_buffer_pool_dump_pct = 40
innodb_page_cleaners = 4
innodb_undo_log_truncate = 1
innodb_max_undo_log_size = 2G
innodb_purge_rseg_truncate_frequency = 128
binlog_gtid_simple_recovery=1
log_timestamps=system
#记录事务的算法,官网建议设置该参数使用 XXHASH64 算法
   
  
#mysql-8.0 
transaction_write_set_extraction=XXHASH64
   
  
#relay_log
relay_log=/acdata/data/mysqldb3307/mysql_3307_relay_bin
relay_log_index=/acdata/data/mysqldb3307/mysql_3307_relay_bin.index
#binlog_format
binlog_format=ROW
max_binlog_size=512M
   
  
#mysql-8.0 
binlog_expire_logs_seconds = 1209600
   
  
#add timeout configure
wait_timeout = 172800
interactive-timeout = 172800
[mysqld_safe]
log-error=/acdata/data/mysqldb3307/mysqld_3307.err
pid-file=/acdata/data/mysqldb3307/mysqld_3307.pid
```

## `从库：172.17.44.143`
```bash
[root@wtj1vpztmysql03 deploy_mysql]#  cat /etc/my_3307.cnf |sed "/^$/d"
[client]
port=3307
socket=/acdata/data/mysqlsoft3307/mysql.sock
[mysqld]
port=3307
mysqlx_port=33070
admin_port=33072
report_host="172.17.44.143"
   
server-id=10143
datadir=/acdata/data/mysqldb3307
basedir=/acdata/data/mysqlsoft3307
socket=/acdata/data/mysqlsoft3307/mysql.sock
   
 
mysqlx_socket=/acdata/data/mysqlsoft3307/mysqlx_3307.sock
   
user=mysql
#symbolic-links=0
default_storage_engine=innodb
default_time_zone = '+8:00'
character_set_server=utf8mb4
collation_server = utf8mb4_unicode_ci
explicit_defaults_for_timestamp=1
# 默认使用mysql_native_password插件认证
default_authentication_plugin=mysql_native_password
#用来控制select语句的最大执行时间，单位是毫秒
max_execution_time=300000
#innodb_flush_log_at_trx_commit && sync_binlog
innodb_flush_log_at_trx_commit=2
sync_binlog=1000
#innodb_flush_log_at_trx_commit=1
#sync_binlog=1
#add slow_query_log
slow_query_log=1
slow_query_log_file=/acdata/data/mysqlsoft3307/slowlog/slowquery_3307.log
long_query_time=0.5
#log_queries_not_using_indexes = 1
log_queries_not_using_indexes = 0
log_slow_admin_statements = 1
   
#mysql-8.0
log_slow_replica_statements=1
#log_throttle_queries_not_using_indexes = 10
   
max_connections=5000
max_user_connections=1800
max_connect_errors = 100
back_log=500
max_allowed_packet =1G
#innodb configure
innodb_flush_method = O_DIRECT
innodb_open_files=65535
open_files_limit =65535
innodb_file_per_table=1
innodb_buffer_pool_size=2G
innodb_log_buffer_size = 16M
#innodb_lock_wait_timeout
innodb_lock_wait_timeout=20
#close query cache
#query_cache_type=0
#query_cache_size =0
sort_buffer_size=4M
join_buffer_size=4M
read_buffer_size=8M
read_rnd_buffer_size=8M
tmpdir=/dev/shm
tmp_table_size=1G
#innodb_logfile
innodb_log_file_size=128M
#transaction_isolation=READ-COMMITTED
thread_cache_size=1000
innodb_read_io_threads=64
innodb_write_io_threads=64
innodb_io_capacity=600
#thread_concurrency=64
#table_cache = 512
key_buffer_size =128M
#log-bin=mysql-bin
replicate-ignore-db=mysql
replicate-ignore-db=information_schema
replicate-ignore-db=performance_schema
replicate-ignore-db=test
#master-connect-retry=60
#rpl_semi_sync_master_enabled=1 
#rpl_semi_sync_master_timeout=3000  
#rpl_semi_sync_slave_enabled=1
log_bin_trust_function_creators = 1
   
#mysql-8.0
skip_replica_start
#mysql-8.0
#replica_skip_errors=1032,1062
#mysql-8.0
log_replica_updates
#mysql-8.0
replica_skip_errors = ddl_exist_errors
   
skip-name-resolve
########Ignore upper_lower case and set sql_mode
lower_case_table_names=1
sql_mode=STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_ENGINE_SUBSTITUTION
########replication settings########
#master_info_repository = TABLE
#relay_log_info_repository = TABLE
log_bin=mysql-bin-sunac_3307
#sync_binlog = 1
gtid_mode = on
enforce_gtid_consistency = 1
binlog_format = row 
#relay_log = relay.log
relay_log_recovery = 1
binlog_gtid_simple_recovery = 1
#Secure File path
#secure_auth=1
#secure_file_priv=/acdata/data/backup
########semi sync replication settings########
plugin_dir=/acdata/data/mysqlsoft3307/lib/plugin
   
 
#mysql-8.0
plugin_load = "rpl_semi_sync_source=semisync_source.so;rpl_semi_sync_replica=semisync_replica.so"
#mysql-8.0
loose_rpl_semi_sync_source_enabled = 1
loose_rpl_semi_sync_replica_enabled = 1
loose_rpl_semi_sync_source_timeout = 3000
###############slave_parallel################
#mysql-8.0
replica_parallel_type=LOGICAL_CLOCK
replica_parallel_workers=4
   
 
# binlog_transaction_dependency_tracking
# COMMIT_ORDER：    默认值，它使用 MySQL 5.7 中可用的默认机制。
# WRITESET：        能实现更好的并行化，并且在主库的二进制日志中存储writeset数据。
# WRITESET_SESSION：确保事务在从库中按顺序执行，并且消除了从库中看到主库从未出现过的数据库状态的问题。
# 降低了并行化程度，但是仍然提供了比默认设置更好的吞吐量。
# master node add this option
binlog_transaction_dependency_tracking=WRITESET_SESSION
#############################################
#[mysqld-5.7]
innodb_buffer_pool_dump_pct = 40
innodb_page_cleaners = 4
innodb_undo_log_truncate = 1
innodb_max_undo_log_size = 2G
innodb_purge_rseg_truncate_frequency = 128
binlog_gtid_simple_recovery=1
log_timestamps=system
#记录事务的算法,官网建议设置该参数使用 XXHASH64算法
   
  
#mysql-8.0 
transaction_write_set_extraction=XXHASH64
   
  
#relay_log
relay_log=/acdata/data/mysqldb3307/mysql_3307_relay_bin
relay_log_index=/acdata/data/mysqldb3307/mysql_3307_relay_bin.index
#binlog_format
binlog_format=ROW
max_binlog_size=512M
   
  
#mysql-8.0 
binlog_expire_logs_seconds = 1209600
   
  
#add timeout configure
wait_timeout = 172800
interactive-timeout = 172800
[mysqld_safe]
log-error=/acdata/data/mysqldb3307/mysqld_3307.err
pid-file=/acdata/data/mysqldb3307/mysqld_3307.pid
```

# 三、配置Mysql半同步复制

## 在主库加载插件semisync_master.so，从库加载插件semisync_slave.so

```sql
########semi sync replication settings########
plugin_dir=/acdata/data/mysqlsoft3307/lib/plugin
   
#mysql-8.0
plugin_load = "rpl_semi_sync_source=semisync_source.so;rpl_semi_sync_replica=semisync_replica.so"

#mysql-8.0
loose_rpl_semi_sync_source_enabled = 1
loose_rpl_semi_sync_replica_enabled = 1
loose_rpl_semi_sync_source_timeout = 3000

###############slave_parallel################
#mysql-8.0
replica_parallel_type=LOGICAL_CLOCK
replica_parallel_workers=2
   
 
# binlog_transaction_dependency_tracking
# COMMIT_ORDER：    默认值，它使用 MySQL 5.7 中可用的默认机制。
# WRITESET：        能实现更好的并行化，并且在主库的二进制日志中存储writeset数据。
# WRITESET_SESSION：确保事务在从库中按顺序执行，并且消除了从库中看到主库从未出现过的���据库状态的问题。
#                   降低了并行化程度，但是仍然提供了比默认设置更好的吞吐量。
# master node add this option
binlog_transaction_dependency_tracking=WRITESET_SESSION
#############################################
```

# 四、创建主从复制账号：

## 1、运行脚本

```bash
# sh create_reset_mult_slave.sh -h1 172.17.44.45 -p1 3307 -h2 172.17.44.44 -p2 3307
# sh create_reset_mult_slave.sh -h1 172.17.44.45 -p1 3307 -h2 172.17.44.143 -p2 3307
```

## 2、脚本内容
```bash 
[root@wtj1vpztmysql01 deploy_mysql]# cat create_reset_mult_slave.sh 
#!/bin/bash

#######color code########
RED="31m"
GREEN="32m"
YELLOW="33m"
BLUE="36m"
FUCHSIA="35m"

func_color_echo(){
    COLOR=$1
    echo -e "\033[${COLOR}${@:2}\033[0m"
}


func_global_init(){

    m_host="${maste_host}"
    m_user="rpl_slave"
    m_pwd="BzCFegx05x1Qdxqga_2022"
    m_port="${maste_port}"
    
    s_host="${slave_host}"
    s_user="rpl_slave"
    s_pwd="BzCFegx05x1Qdxqga_2022"
    s_port="${slave_port}"

}

func_init_repl(){
echo "#############$m_host,$m_user######################"
# 0.主库上创建同步复制账号：
/acdata/data/mysqlsoft${maste_port}/bin/mysql -h ${maste_host} -u root -p'pC52liycziojMi_2022' -P ${maste_port} << sqleof
DROP USER IF EXISTS 'rpl_slave'@'172.17.%';
CREATE USER IF NOT EXISTS 'rpl_slave'@'172.17.%' IDENTIFIED WITH mysql_native_password BY 'BzCFegx05x1Qdxqga_2022';
GRANT RELOAD, PROCESS, LOCK TABLES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'rpl_slave'@'172.17.%';
GRANT SELECT ON performance_schema.threads TO 'rpl_slave'@'172.17.%';
FLUSH PRIVILEGES;
sqleof

echo ""
echo "############create repl_slave user for master:$m_host:$m_port -- $m_user is ok.##########"

}


func_init_slave(){

/acdata/data/mysqlsoft${slave_port}/bin/mysql -h localhost -P ${slave_port} -uroot -p'pC52liycziojMi_2022' -S /acdata/data/mysqlsoft${slave_port}/mysql.sock << sqleof
stop slave;
reset master;
reset slave all;

-- drop user if exists 'root'@'127.0.0.1';
-- drop user if exists 'root'@'%';
-- DROP USER IF EXISTS 'dba_manager'@'172.17.%';

-- 普通模式复制
-- change master to master_host='172.17.44.66',master_port=${maste_port},master_user='rpl_slave',master_password='BzCFegx05x1Qdxqga_2022',MASTER_LOG_FILE='bin.000265', MASTER_LOG_POS=66665373;

select @@server_id;
-- GTID模式复制
CHANGE MASTER TO
MASTER_HOST='$m_host',
MASTER_USER='$m_user',
MASTER_PORT=$m_port,
MASTER_PASSWORD='$m_pwd',
-- MASTER_LOG_FILE='$m_log_file',
-- MASTER_LOG_POS=$m_log_pos;
master_auto_position=1;
start slave;
select sleep(3);
show slave status\G;
sqleof
echo ""
echo "############Finished replication for master:$m_host:$m_port -- slave:$s_host:$s_port ##########"

}

#func_init_repl
#func_init_slave


func_usage() {
        func_color_echo $FUCHSIA "Err Syntax: `basename $0` :usage: [-h1 --maste-host]  [-p1 maste_port] [-h2 slave_host]  [-p2 slave_port]"
}


# print usage
if [[ $# -lt 2 ]] || [[ $# -gt 8 ]]
then
        echo ""
        func_usage
        echo ""
        echo ""
        exit 0
fi

 
# read input
while [[ $# -gt 0 ]]
  do
        case "$1" in
               -h1|--maste-host)
               shift
               maste_host=$1
        ;;
               -h2|--slave-host)
               shift
               slave_host=$1
        ;;
               -p1|--maste-port)
               shift
               maste_port=$1
        ;;
               -p2|--slave-port)
               shift
               slave_port=$1
        ;;


               *)
               shift
               func_usage
               exit
        ;;
        esac
        shift
 done

echo "##########$m_host $m_port $s_host $s_port" 


if [ ! -z "${maste_host}" ] && [ ! -z "${slave_host}" ] && [ ! -z "${maste_port}" ] && [ ! -z "${slave_port}" ];then
      echo ""
      func_global_init
      #func_init_repl
      func_init_slave
      echo ""
else
     func_color_echo $RED "################### These variables is not allowed.#####################"
     func_usage
fi
```

## 3、创建mha管理账号：(主库上操作即可)
```mysql
    create user mha_admin@'172.17.%' identified WITH mysql_native_password by 'Mha_sunac_2024';
    grant all privileges on *.* to mha_admin@'172.17.%'; 
```

## 4、查看复制状态

 `在从库主机上使用 show slave status查看复制状态，当Slave_IO_Running和Slave_SQL_Running都是Yes的时候说明主从复制状态是正常的，此时可以在主库上操作数据，然后在从库上验证数据是否会同步过来。`

 至此，mysql基于GTID的主从复制搭建完毕，下面就剩下mha软件的搭建。

# 五、 安装配置MHA

> mha node角色需要部署在每台主机上面，mha控制节点需要部署mha manager和mha_node。

## 1、mha node节点部署

mha是由perl语言开发，所以需要使用perl的依赖，推荐使用yum进行安装，此软件需要安装在每台服务器上。

```js
# rpm -ivh rpmforge-release-0.5.3-1.el7.rf.x86_64.rpm
# yum install -y  perl-DBD-MySQL \
                  perl-Config-Tiny \
                  perl-Log-Dispatch \
                  perl-Parallel-ForkManager \
                  perl-Time-HiRes \
                  perl-ExtUtils-CBuilder \
                  perl-ExtUtils-MakeMaker


# rpm -q perl-DBD-MySQL perl-Config-Tiny perl-Log-Dispatch perl-Parallel-ForkManager  perl-Time-HiRes perl-ExtUtils-CBuilder perl-ExtUtils-MakeMaker
```

## 2、安装mha node

```js

# tar xvfz mha4mysql-node-0.58.tar.gz 
# cd mha4mysql-node-0.58
# perl Makefile.PL 
# make && make install
```

### node主要工具

 Node工具包（这些工具通常由MHA Manager的脚本触发，无需人为操作）主要包括以下几个工具： 

>`save_binary_logs                保存和复制master的二进制日志`
>`apply_diff_relay_logs            识别差异的中继日志事件并将其差异的事件应用于其他的slave`
>`filter_mysqlbinlog               去除不必要的ROLLBACK事件（MHA已不再使用这个工具）`
>`purge_relay_logs                清除中继日志（不会阻塞SQL线程）`

## 3、mha manager 节点部署（需要访问公网）

Mha manager控制节点单独一台服务器，部署在172.17.44.144服务器上。

由于已经安装mha node，所以相关依赖的perl模块已经安装，可以直接安装mha manager软件.

```js

# tar xvfz mha4mysql-manager-0.58.tar.gz
# cd mha4mysql-manager-0.58
# perl Makefile.PL
# make && make install
```

### manager主要工具



> `masterha_check_ssh              检查MHA的SSH配置状况`
> `masterha_check_repl             检查MySQL复制状况`
> `masterha_manger                 启动MHA`
> `masterha_check_status            检测当前MHA运行状态`
> `masterha_master_monitor          检测master是否宕机`
> `masterha_master_switch           控制故障转移（自动或者手动）`
> `masterha_conf_host               添加或删除配置的server信息`

## 4、配置ssh信任

需要配置SSH登陆无密码验证功能，因为mha切换的时候需要到主机上执行命令，各主机之间应当都是免密登陆。

需要注意的是不能禁止password登陆，否则会出现错误.

  Mha manager主机需要登陆到三台node节点主机，在172.17.44.144上执行：

```bash
# ssh-keygen    #一路回车
# cd ~/.ssh
# ssh-copy-id -i ./id_rsa.pub root@172.17.44.44
# ssh-copy-id -i ./id_rsa.pub root@172.17.44.45
# ssh-copy-id -i ./id_rsa.pub root@172.17.44.143
# ssh-copy-id -i ./id_rsa.pub root@172.17.44.144
```

在172.17.44.44/45/143上生成密钥对，然后互相打通ssh密钥登陆：
```bash

44.44上执行命令:

# ssh-copy-id -i ./id_rsa.pub root@172.17.44.45
# ssh-copy-id -i ./id_rsa.pub root@172.17.44.143
# ssh-copy-id -i ./id_rsa.pub root@172.17.44.144

44.45上执行命令:

# ssh-copy-id -i ./id_rsa.pub root@172.17.44.44
# ssh-copy-id -i ./id_rsa.pub root@172.17.44.143
# ssh-copy-id -i ./id_rsa.pub root@172.17.44.144

44.143上执行命令:
# ssh-copy-id -i ./id_rsa.pub root@172.17.44.44
# ssh-copy-id -i ./id_rsa.pub root@172.17.44.45
# ssh-copy-id -i ./id_rsa.pub root@172.17.44.144
```

## 5、 设置从库只读（可选）

两台slave服务器设置read_only（从库对外提供读服务，只所以没有写进配置文件，是因为随时slave会提升为master） 

> # 设置只读的指令，需要在两个从库44/143上执行
> # mysql3307 -e 'set global read_only=1'


## 6、 设置relaylog清理（可选）

MHA在发生切换的过程中，从库的恢复过程中依赖于relay log的相关信息，所以这里要将relay log的自动清除设置为OFF，

采用手动清除relay log的方式。在默认情况下，从服务器上的中继日志会在SQL线程执行完毕后被自动删除。

> `# 设置关闭relay_log自动清理的指令，需要在两个从库87/88上执行`
 >`# mysql3307 -e 'set global relay_log_purge=0'`
 
 在MHA环境中，这些中继日志在恢复其他从服务器时可能会被用到，因此需要禁用中继日志的自动删除功能。定期清除中继日志需要考虑到复制延时的问题。在ext3的文件系统下，删除大的文件需要一定的时间，会导致严重的复制延时。为了避免复制延时，需要暂时为中继日志创建硬链接，因为在linux系统中通过硬链接删除大文件速度会很快。（在mysql数据库中，删除大表时，通常也采用建立硬链接的方式）

 MHA节点中包含了pure_relay_logs命令工具，它可以为中继日志创建硬链接，执行SET GLOBAL relay_log_purge=1,等待几秒钟以便SQL线程切换到新的中继日志，再执行SET GLOBAL relay_log_purge=0.这是此工具的原理.
### Pure_relay_log脚本介绍

```bash
--user mysql                      用户名
--password mysql                  密码
--port                            端口号
--workdir                         指定创建relay log的硬链接的位置，默认是/var/tmp，由于系统不同分区创建硬链接文件会失败，故需要执行硬链接具体位置，成功执行脚本后，硬链接的中继日志文件被删除
--disable_relay_log_purge         默认情况下，如果relay_log_purge=1，脚本会什么都不清理，自动退出，通过设定这个参数，当relay_log_purge=1的情况下会将relay_log_purge设置为0。清理relay log之后，最后将参数设置为OFF。
```

两台从服务器上设置relay脚本定期清除.

```bash

# crontab -l

0 4 * * * /bin/bash /data/scripts/purge_relay_log.sh

# cat /data/scripts/purge_relay_log.sh   # 清理脚本

#!/bin/bash

. /etc/profile
. ~/.bash_profile
. ~/.bashrc

user=root
passwd=dbpass123
port=3306

log_dir=/masterha/app2/log
work_dir=/masterha/app2
purge=/usr/local/bin/purge_relay_logs

if [ ! -d $log_dir ];then
 mkdir  -p $log_dir
fi

$purge --user=$user --password=$passwd --disable_relay_log_purge --port=$port --workdir=$work_dir >> $log_dir /purge_relay_logs .log 2>&1`
```

## 在mha集群所有节点创建目录：
```js
# for i in {44,45,143,144};do ssh root@172.17.44.${i} "mkdir -p /masterha/app2";done
[root@wtj1vpztmysql04 ~]# for i in {44,45,143,144};do ssh root@172.17.44.${i} "hostname;ls -l /masterha";done
wtj1vpztmysql01
total 0
drwxr-xr-x 2 root root 6 Jul 30 14:54 app1
drwxr-xr-x 2 root root 6 Aug  1 17:50 app2
wtj1vpztmysql02
total 0
drwxr-xr-x 2 root root 6 Jul 30 14:54 app1
drwxr-xr-x 2 root root 6 Aug  1 17:50 app2
wtj1vpztmysql03
total 0
drwxr-xr-x 2 root root 6 Jul 30 14:54 app1
drwxr-xr-x 2 root root 6 Aug  1 17:50 app2
wtj1vpztmysql04
total 0
drwxr-xr-x. 2 root root  6 Jul 30 14:51 app1
drwxr-xr-x. 2 root root 60 Aug  1 17:30 app2
```

# 创建mha_manager服务
```bash
[root@wtj1vpztmysql04 masterha]# cat  /etc/init.d/mha_manager
#!/bin/bash
# chkconfig: 35 80 20
# description: MHA management script.
 
STARTEXEC="/usr/local/bin/masterha_manager --conf"
STOPEXEC="/usr/local/bin/masterha_stop --conf"
CONF="/etc/masterha/app1.cnf"
process_count=$(ps -ef |grep -w masterha_manager|grep -v grep|wc -l)
PARAMS="--ignore_last_failover"
 
case "$1" in
  start)
      if [ $process_count -gt 1 ];then
              echo "masterha_manager exists, process is already running"
      else
              echo "Starting Masterha Manager"
              $STARTEXEC $CONF $PARAMS < /dev/null > /acdata/data/masterha/app1/manager_app1.log 2>&1 &
      fi
      ;;
  stop)
      if [ $process_count -eq 0 ];then
              echo "Masterha Manager does not exist, process is not running"
      else
              echo "Stopping ..."
              $STOPEXEC $CONF
              while(true)
              do
                  process_count=`ps -ef |grep -w masterha_manager|grep -v grep|wc -l`
                  if [ $process_count -gt 0 ];then
                      sleep 1
                  else
                      break
                  fi
              done
              echo "Master Manager stopped"
      fi
      ;;
  *)
      echo "Please use start or stop as first argument"
      ;;
esac

[root@wtj1vpztmysql04 masterha]# chmod +x /etc/init.d/mha_manager 
[root@wtj1vpztmysql04 masterha]# chkconfig --add mha_manager
[root@wtj1vpztmysql04 masterha]# chkconfig mha_manager on

[root@wtj1vpztmysql04 masterha]# systemctl daemon-reload
[root@wtj1vpztmysql04 masterha]# systemctl start mha_manager
[root@wtj1vpztmysql04 masterha]# systemctl status mha_manager
● mha_manager.service - SYSV: MHA management script.
   Loaded: loaded (/etc/rc.d/init.d/mha_manager; bad; vendor preset: disabled)
   Active: active (running) since Wed 2024-08-07 14:38:37 CST; 26s ago
     Docs: man:systemd-sysv-generator(8)
   CGroup: /system.slice/mha_manager.service
           └─1668 perl /usr/local/bin/masterha_manager --conf /etc/masterha/app1.cnf --ignore_last_failover

Aug 07 14:38:37 wtj1vpztmysql04 systemd[1]: Starting SYSV: MHA management script....
Aug 07 14:38:37 wtj1vpztmysql04 mha_manager[1662]: Starting Masterha Manager
Aug 07 14:38:37 wtj1vpztmysql04 systemd[1]: Started SYSV: MHA management script..


```
# 目录结构
```bash

mha_manager目录结构：

[root@wtj1vpztmysql04 masterha]# ll /etc/masterha_default.cnf 
-rw-r--r--. 1 root root 340 Aug  1 12:05 /etc/masterha_default.cnf
[root@wtj1vpztmysql04 masterha]# ll /etc/masterha/
total 12
-rw-r--r--. 1 root root 1421 Aug  6 18:13 app1.cnf
-rw-r--r--. 1 root root 1421 Aug  5 17:59 app2.cnf
-rw-------. 1 root root 1776 Aug  1 18:22 nohup.out
[root@wtj1vpztmysql04 masterha]# ll /script/masterha/
total 36
-rwxr-xr-x. 1 root root  2703 Aug  6 18:15 master_ip_failover.3306
-rwxr-xr-x. 1 root root  2703 Aug  1 18:32 master_ip_failover.3307
-rwxr-xr-x. 1 root root 11691 Aug  6 18:16 master_ip_online_change.3306
-rwxr-xr-x. 1 root root 11691 Aug  5 17:59 master_ip_online_change.3307
-rw-------. 1 root root   296 Aug  6 18:17 nohup.out

[root@wtj1vpztmysql04 masterha]# ll /masterha
total 0
drwxr-xr-x. 2 root root 63 Aug  6 18:17 app1
drwxr-xr-x. 2 root root 63 Aug  6 16:01 app2

mha_node目录结构：
[root@wtj1vpztmysql04 masterha]# ll /masterha
total 0
drwxr-xr-x. 2 root root 63 Aug  6 18:17 app1
drwxr-xr-x. 2 root root 63 Aug  6 16:01 app2
```
## 全局配置文件：
```js
[root@wtj1vpztmysql04 masterha]# cat /etc/masterha_default.cnf 

[server default]
manager_workdir=/masterha/app1
manager_log=/masterha/app1/manager.log

#mha管理用户
#user=mha_admin
#mha管理密码
#password=Mha_sunac_2024

#设置ssh的端口号
ssh_port=22
#设置ssh的登陆用户
ssh_user=root

#repl同步用户
#repl_user=rpl_slave
#repl_password=BzCFegx05x1Qdxqga_2022

#ping_interval=1

#log_format
#log_level=info
log_level=debug
```


## 单个集群配置：
```js

[root@wtj1vpztmysql04 masterha]# cat /etc/masterha/app2.cnf 
[server default]
port=3307
manager_workdir=/masterha/app2
manager_log=/masterha/app2/manager_app2.log
remote_workdir=/masterha/app2

#mha管理用户
user=mha_admin
#mha管理密码
password=Mha_sunac_2024

#设置ssh的端口号
ssh_port=22
#设置ssh的登陆用户
ssh_user=root

#repl同步用户和密码
repl_user=rpl_slave
repl_password=BzCFegx05x1Qdxqga_2022

# ping间隔时长
ping_interval=1
 
 
secondary_check_script=/usr/local/bin/masterha_secondary_check -s 172.17.44.44 -s 172.17.44.143 --user=root --master_host=mha-master-01 --master_ip=172.17.44.45 --master_port=3307
 
master_ip_failover_script=/script/masterha/master_ip_failover.3307
master_ip_online_change_script=/script/masterha/master_ip_online_change.3307
#shutdown_script= /script/masterha/power_manager
#report_script= /script/masterha/send_master_failover_mail
 
[server1]
hostname=172.17.44.45
ssh_port=22
port=3307
master_binlog_dir=/acdata/data/mysqldb3307
candidate_master=1
 
[server2]
hostname=172.17.44.143
ssh_port=22
port=3307
master_binlog_dir=/acdata/data/mysqldb3307
candidate_master=1
#no_master=1
 
 
[server3]
hostname=172.17.44.44
ssh_port=22
port=3307
master_binlog_dir=/acdata/data/mysqldb3307
#candidate_master=1
no_master=1

```
## 检查ssh用户信任：
```bash

[root@wtj1vpztmysql04 masterha]# masterha_check_ssh --conf=/etc/masterha/app2.cnf
Thu Aug  1 17:05:10 2024 - [info] Reading default configuration from /etc/masterha_default.cnf..
Thu Aug  1 17:05:10 2024 - [info] Reading application default configuration from /etc/masterha/app2.cnf..
Thu Aug  1 17:05:10 2024 - [info] Reading server configuration from /etc/masterha/app2.cnf..
Thu Aug  1 17:05:10 2024 - [info] Starting SSH connection tests..
Thu Aug  1 17:05:11 2024 - [debug] 
Thu Aug  1 17:05:10 2024 - [debug]  Connecting via SSH from root@172.17.44.45(172.17.44.45:22) to root@172.17.44.143(172.17.44.143:22)..
Thu Aug  1 17:05:11 2024 - [debug]   ok.
Thu Aug  1 17:05:11 2024 - [debug]  Connecting via SSH from root@172.17.44.45(172.17.44.45:22) to root@172.17.44.44(172.17.44.44:22)..
Thu Aug  1 17:05:11 2024 - [debug]   ok.
Thu Aug  1 17:05:12 2024 - [debug] 
Thu Aug  1 17:05:11 2024 - [debug]  Connecting via SSH from root@172.17.44.143(172.17.44.143:22) to root@172.17.44.45(172.17.44.45:22)..
Thu Aug  1 17:05:11 2024 - [debug]   ok.
Thu Aug  1 17:05:11 2024 - [debug]  Connecting via SSH from root@172.17.44.143(172.17.44.143:22) to root@172.17.44.44(172.17.44.44:22)..
Thu Aug  1 17:05:11 2024 - [debug]   ok.
Thu Aug  1 17:05:13 2024 - [debug] 
Thu Aug  1 17:05:11 2024 - [debug]  Connecting via SSH from root@172.17.44.44(172.17.44.44:22) to root@172.17.44.45(172.17.44.45:22)..
Thu Aug  1 17:05:12 2024 - [debug]   ok.
Thu Aug  1 17:05:12 2024 - [debug]  Connecting via SSH from root@172.17.44.44(172.17.44.44:22) to root@172.17.44.143(172.17.44.143:22)..
Thu Aug  1 17:05:12 2024 - [debug]   ok.
Thu Aug  1 17:05:13 2024 - [info] All SSH connection tests passed successfully.
Use of uninitialized value in exit at /usr/local/bin/masterha_check_ssh line 44.
```

```bash

[root@wtj1vpztmysql04 masterha]# masterha_check_repl --conf=/etc/masterha/app2.cnf
Thu Aug  1 17:05:22 2024 - [info] Reading default configuration from /etc/masterha_default.cnf..
Thu Aug  1 17:05:22 2024 - [info] Reading application default configuration from /etc/masterha/app2.cnf..
Thu Aug  1 17:05:22 2024 - [info] Reading server configuration from /etc/masterha/app2.cnf..
Thu Aug  1 17:05:22 2024 - [info] MHA::MasterMonitor version 0.58.
Thu Aug  1 17:05:22 2024 - [debug] Connecting to servers..
Thu Aug  1 17:05:23 2024 - [debug]  Connected to: 172.17.44.45(172.17.44.45:3307), user=mha_admin
Thu Aug  1 17:05:23 2024 - [debug]  Number of slave worker threads on host 172.17.44.45(172.17.44.45:3307): 2
Thu Aug  1 17:05:23 2024 - [debug]  Connected to: 172.17.44.143(172.17.44.143:3307), user=mha_admin
Thu Aug  1 17:05:23 2024 - [debug]  Number of slave worker threads on host 172.17.44.143(172.17.44.143:3307): 4
Thu Aug  1 17:05:23 2024 - [debug]  Connected to: 172.17.44.44(172.17.44.44:3307), user=mha_admin
Thu Aug  1 17:05:23 2024 - [debug]  Number of slave worker threads on host 172.17.44.44(172.17.44.44:3307): 2
Thu Aug  1 17:05:23 2024 - [debug]  Comparing MySQL versions..
Thu Aug  1 17:05:23 2024 - [debug]   Comparing MySQL versions done.
Thu Aug  1 17:05:23 2024 - [debug] Connecting to servers done.
Thu Aug  1 17:05:23 2024 - [info] GTID failover mode = 1
Thu Aug  1 17:05:23 2024 - [info] Dead Servers:
Thu Aug  1 17:05:23 2024 - [info] Alive Servers:
Thu Aug  1 17:05:23 2024 - [info]   172.17.44.45(172.17.44.45:3307)
Thu Aug  1 17:05:23 2024 - [info]   172.17.44.143(172.17.44.143:3307)
Thu Aug  1 17:05:23 2024 - [info]   172.17.44.44(172.17.44.44:3307)
Thu Aug  1 17:05:23 2024 - [info] Alive Slaves:
Thu Aug  1 17:05:23 2024 - [info]   172.17.44.143(172.17.44.143:3307)  Version=8.0.28 (oldest major version between slaves) log-bin:enabled
Thu Aug  1 17:05:23 2024 - [info]     GTID ON
Thu Aug  1 17:05:23 2024 - [debug]    Relay log info repository: TABLE
Thu Aug  1 17:05:23 2024 - [info]     Replicating from 172.17.44.45(172.17.44.45:3307)
Thu Aug  1 17:05:23 2024 - [info]     Primary candidate for the new Master (candidate_master is set)
Thu Aug  1 17:05:23 2024 - [info]   172.17.44.44(172.17.44.44:3307)  Version=8.0.28 (oldest major version between slaves) log-bin:enabled
Thu Aug  1 17:05:23 2024 - [info]     GTID ON
Thu Aug  1 17:05:23 2024 - [debug]    Relay log info repository: TABLE
Thu Aug  1 17:05:23 2024 - [info]     Replicating from 172.17.44.45(172.17.44.45:3307)
Thu Aug  1 17:05:23 2024 - [info]     Not candidate for the new Master (no_master is set)
Thu Aug  1 17:05:23 2024 - [info] Current Alive Master: 172.17.44.45(172.17.44.45:3307)
Thu Aug  1 17:05:23 2024 - [info] Checking slave configurations..
Thu Aug  1 17:05:23 2024 - [info]  read_only=1 is not set on slave 172.17.44.143(172.17.44.143:3307).
Thu Aug  1 17:05:23 2024 - [info]  read_only=1 is not set on slave 172.17.44.44(172.17.44.44:3307).
Thu Aug  1 17:05:23 2024 - [info] Checking replication filtering settings..
Thu Aug  1 17:05:23 2024 - [info]  binlog_do_db= , binlog_ignore_db= 
Thu Aug  1 17:05:23 2024 - [info]  Replication filtering check ok.
Thu Aug  1 17:05:23 2024 - [info] GTID (with auto-pos) is supported. Skipping all SSH and Node package checking.
Thu Aug  1 17:05:23 2024 - [info] Checking SSH publickey authentication settings on the current master..
Thu Aug  1 17:05:23 2024 - [debug] SSH connection test to 172.17.44.45, option -o StrictHostKeyChecking=no -o PasswordAuthentication=no -o BatchMode=yes -o ConnectTimeout=5, timeout 5
Thu Aug  1 17:05:23 2024 - [info] HealthCheck: SSH to 172.17.44.45 is reachable.
Thu Aug  1 17:05:23 2024 - [info] 
172.17.44.45(172.17.44.45:3307) (current master)
 +--172.17.44.143(172.17.44.143:3307)
 +--172.17.44.44(172.17.44.44:3307)

Thu Aug  1 17:05:23 2024 - [info] Checking replication health on 172.17.44.143..
Thu Aug  1 17:05:23 2024 - [info]  ok.
Thu Aug  1 17:05:23 2024 - [info] Checking replication health on 172.17.44.44..
Thu Aug  1 17:05:23 2024 - [info]  ok.
Thu Aug  1 17:05:23 2024 - [info] Checking master_ip_failover_script status:
Thu Aug  1 17:05:23 2024 - [info]   /script/masterha/master_ip_failover.3307 --command=status --ssh_user=root --orig_master_host=172.17.44.45 --orig_master_ip=172.17.44.45 --orig_master_port=3307 


IN SCRIPT TEST====/sbin/ifconfig ens192:1 down==/sbin/ifconfig ens192:1 172.17.44.145/24===

Checking the Status of the script.. OK 
Thu Aug  1 17:05:23 2024 - [info]  OK.
Thu Aug  1 17:05:23 2024 - [warning] shutdown_script is not defined.
Thu Aug  1 17:05:23 2024 - [debug]  Disconnected from 172.17.44.45(172.17.44.45:3307)
Thu Aug  1 17:05:23 2024 - [debug]  Disconnected from 172.17.44.143(172.17.44.143:3307)
Thu Aug  1 17:05:23 2024 - [debug]  Disconnected from 172.17.44.44(172.17.44.44:3307)
Thu Aug  1 17:05:23 2024 - [info] Got exit code 0 (Not master dead).

MySQL Replication Health is OK.

```

## 故障漂移vip脚本：
```js

[root@wtj1vpztmysql04 masterha]# cat /script/masterha/m
master_ip_failover.3306       master_ip_online_change.3306  mysql-mult-mha.zip            
master_ip_failover.3307       master_ip_online_change.3307  
[root@wtj1vpztmysql04 masterha]# cat /script/masterha/master_ip_failover.3307 
#!/usr/bin/env perl 
use strict; 
use warnings FATAL =>'all'; 

use Getopt::Long; 

my ( 
$command,          $ssh_user,        $orig_master_host, $orig_master_ip, 
$orig_master_port, $new_master_host, $new_master_ip,    $new_master_port 
); 

###########################################################################  
my $vip = '172.17.44.145/21';  
my $key = "1";
my $ssh_start_vip = "/sbin/ifconfig ens192:$key $vip";  
my $ssh_stop_vip = "/sbin/ifconfig ens192:$key $vip down";  
my $ssh_Bcast_arp= "/sbin/arping -I ens192 -c 3 -A 172.17.44.145";
###########################################################################  

GetOptions( 
'command=s'          => \$command, 
'ssh_user=s'         => \$ssh_user, 
'orig_master_host=s' => \$orig_master_host, 
'orig_master_ip=s'   => \$orig_master_ip, 
'orig_master_port=i' => \$orig_master_port, 
'new_master_host=s'  => \$new_master_host, 
'new_master_ip=s'    => \$new_master_ip, 
'new_master_port=i'  => \$new_master_port, 
); 

exit &main(); 

sub main { 

print "\n\nIN SCRIPT TEST====$ssh_stop_vip==$ssh_start_vip===\n\n"; 

if ( $command eq "stop" || $command eq "stopssh" ) { 

        # $orig_master_host, $orig_master_ip, $orig_master_port are passed. 
        # If you manage master ip address at global catalog database, 
        # invalidate orig_master_ip here. 
my $exit_code = 1; 
        eval { 
            print "Disabling the VIP on old master: $orig_master_host \n"; 
&stop_vip(); 
            $exit_code = 0; 
        }; 
        if ($@) { 
            warn "Got Error: $@\n"; 
            exit $exit_code; 
        } 
        exit $exit_code; 
} 
elsif ( $command eq "start" ) { 

        # all arguments are passed. 
        # If you manage master ip address at global catalog database, 
        # activate new_master_ip here. 
        # You can also grant write access (create user, set read_only=0, etc) here. 
my $exit_code = 10; 
        eval { 
###########################################################################  
    print "Enabling the VIP - $vip on the new master - $new_master_host \n"; 
            &start_vip(); 
            &start_arp();
###########################################################################
            $exit_code = 0; 
        }; 
        if ($@) { 
            warn $@; 
            exit $exit_code; 
        } 
        exit $exit_code; 
} 
elsif ( $command eq "status" ) { 
        print "Checking the Status of the script.. OK \n"; 
        `ssh $ssh_user\@$orig_master_host \" $ssh_start_vip \"`; 
        exit 0; 
} 
else { 
&usage(); 
        exit 1; 
} 
} 

# A simple system call that enable the VIP on the new master 
###########################################################################  
sub start_vip() { 
    `ssh $ssh_user\@$new_master_host \" $ssh_start_vip \"`; 
} 
# A simple system call that disable the VIP on the old_master 
sub stop_vip() { 
    `ssh $ssh_user\@$orig_master_host \" $ssh_stop_vip \"`; 
} 
sub start_arp() {
    `ssh $ssh_user\@$new_master_host \" $ssh_bcast_arp \"`;
} 
###########################################################################
sub usage { 
print 
"Usage: master_ip_failover –command=start|stop|stopssh|status –orig_master_host=host –orig_master_ip=ip –orig_master_port=port –new_master_host=host –new_master_ip=ip –new_master_port=port\n"; 
} 

```

## 故障切换脚本：
```js
[root@wtj1vpztmysql04 masterha]# cat /script/masterha/master_ip_online_change.3307 
#!/usr/bin/env perl

#  Copyright (C) 2011 DeNA Co.,Ltd.
#
#  This program is free software; you can redistribute it and/or modify
#  it under the terms of the GNU General Public License as published by
#  the Free Software Foundation; either version 2 of the License, or
#  (at your option) any later version.
#
#  This program is distributed in the hope that it will be useful,
#  but WITHOUT ANY WARRANTY; without even the implied warranty of
#  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#  GNU General Public License for more details.
#
#  You should have received a copy of the GNU General Public License
#   along with this program; if not, write to the Free Software
#  Foundation, Inc.,
#  51 Franklin Street, Fifth Floor, Boston, MA  02110-1301  USA

## Note: This is a sample script and is not complete. Modify the script based on your environment.

use strict;  
use warnings FATAL => 'all';  
use Getopt::Long;  
use MHA::DBHelper;  
use MHA::NodeUtil;  
use Time::HiRes qw( sleep gettimeofday tv_interval );  
use Data::Dumper;  
my $_tstart;  
my $_running_interval = 0.1;  
my (  
  $command,              $orig_master_is_new_slave, $orig_master_host,  
  $orig_master_ip,       $orig_master_port,         $orig_master_user,  
  $orig_master_password, $orig_master_ssh_user,     $new_master_host,  
  $new_master_ip,        $new_master_port,          $new_master_user,  
  $new_master_password,  $new_master_ssh_user,  
);  
  
###########################################################################  
my $vip = '172.17.44.145';  
my $key = "1";
my $ssh_start_vip = "/sbin/ifconfig ens192:$key $vip";  
my $ssh_stop_vip = "/sbin/ifconfig ens192:$key $vip down"; 
my $ssh_Bcast_arp= "/sbin/arping -I ens192 -c 3 -A 172.17.44.145";
###########################################################################  
  
GetOptions(  
  'command=s'                => \$command,  
  'orig_master_is_new_slave' => \$orig_master_is_new_slave,  
  'orig_master_host=s'       => \$orig_master_host,  
  'orig_master_ip=s'         => \$orig_master_ip,  
  'orig_master_port=i'       => \$orig_master_port,  
  'orig_master_user=s'       => \$orig_master_user,  
  'orig_master_password=s'   => \$orig_master_password,  
  'orig_master_ssh_user=s'   => \$orig_master_ssh_user,  
  'new_master_host=s'        => \$new_master_host,  
  'new_master_ip=s'          => \$new_master_ip,  
  'new_master_port=i'        => \$new_master_port,  
  'new_master_user=s'        => \$new_master_user,  
  'new_master_password=s'    => \$new_master_password,  
  'new_master_ssh_user=s'    => \$new_master_ssh_user,  
);  
exit &main();  
sub current_time_us {  
  my ( $sec, $microsec ) = gettimeofday();  
  my $curdate = localtime($sec);  
  return $curdate . " " . sprintf( "%06d", $microsec );  
}  
sub sleep_until {  
  my $elapsed = tv_interval($_tstart);  
  if ( $_running_interval > $elapsed ) {  
    sleep( $_running_interval - $elapsed );  
  }  
}  
sub get_threads_util {  
  my $dbh                    = shift;  
  my $my_connection_id       = shift;  
  my $running_time_threshold = shift;  
  my $type                   = shift;  
  $running_time_threshold = 0 unless ($running_time_threshold);  
  $type                   = 0 unless ($type);  
  my @threads;  
  my $sth = $dbh->prepare("SHOW PROCESSLIST");  
  $sth->execute();  
  while ( my $ref = $sth->fetchrow_hashref() ) {  
    my $id         = $ref->{Id};  
    my $user       = $ref->{User};  
    my $host       = $ref->{Host};  
    my $command    = $ref->{Command};  
    my $state      = $ref->{State};  
    my $query_time = $ref->{Time};  
    my $info       = $ref->{Info};  
    $info =~ s/^\s*(.*?)\s*$/$1/ if defined($info);  
    next if ( $my_connection_id == $id );  
    next if ( defined($query_time) && $query_time < $running_time_threshold );  
    next if ( defined($command)    && $command eq "Binlog Dump" );  
    next if ( defined($user)       && $user eq "system user" );  
    next  
      if ( defined($command)  
      && $command eq "Sleep"  
      && defined($query_time)  
      && $query_time >= 1 );  
    if ( $type >= 1 ) {  
      next if ( defined($command) && $command eq "Sleep" );  
      next if ( defined($command) && $command eq "Connect" );  
    }  
    if ( $type >= 2 ) {  
      next if ( defined($info) && $info =~ m/^select/i );  
      next if ( defined($info) && $info =~ m/^show/i );  
    }  
    push @threads, $ref;  
  }  
  return @threads;  
}  
sub main {  
  if ( $command eq "stop" ) {  
    ## Gracefully killing connections on the current master  
    # 1. Set read_only= 1 on the new master  
    # 2. DROP USER so that no app user can establish new connections  
    # 3. Set read_only= 1 on the current master  
    # 4. Kill current queries  
    # * Any database access failure will result in script die.  
    my $exit_code = 1;  
    eval {  
      ## Setting read_only=1 on the new master (to avoid accident)  
      my $new_master_handler = new MHA::DBHelper();  
      # args: hostname, port, user, password, raise_error(die_on_error)_or_not  
      $new_master_handler->connect( $new_master_ip, $new_master_port,  
        $new_master_user, $new_master_password, 1 );  
      print current_time_us() . " Set read_only on the new master.. ";  
      $new_master_handler->enable_read_only();  
      if ( $new_master_handler->is_read_only() ) {  
        print "ok.\n";  
      }  
      else {  
        die "Failed!\n";  
      }  
      $new_master_handler->disconnect();  
      # Connecting to the orig master, die if any database error happens  
      my $orig_master_handler = new MHA::DBHelper();  
      $orig_master_handler->connect( $orig_master_ip, $orig_master_port,  
        $orig_master_user, $orig_master_password, 1 );  
      ## Drop application user so that nobody can connect. Disabling per-session binlog beforehand  
      $orig_master_handler->disable_log_bin_local();  
      print current_time_us() . " Drpping app user on the orig master..\n";  
###########################################################################  
      #FIXME_xxx_drop_app_user($orig_master_handler);  
###########################################################################  
      ## Waiting for N * 100 milliseconds so that current connections can exit  
      my $time_until_read_only = 15;  
      $_tstart = [gettimeofday];  
      my @threads = get_threads_util( $orig_master_handler->{dbh},  
        $orig_master_handler->{connection_id} );  
      while ( $time_until_read_only > 0 && $#threads >= 0 ) {  
        if ( $time_until_read_only % 5 == 0 ) {  
          printf  
"%s Waiting all running %d threads are disconnected.. (max %d milliseconds)\n",  
            current_time_us(), $#threads + 1, $time_until_read_only * 100;  
          if ( $#threads < 5 ) {  
            print Data::Dumper->new( [$_] )->Indent(0)->Terse(1)->Dump . "\n"  
              foreach (@threads);  
          }  
        }  
        sleep_until();  
        $_tstart = [gettimeofday];  
        $time_until_read_only--;  
        @threads = get_threads_util( $orig_master_handler->{dbh},  
          $orig_master_handler->{connection_id} );  
      }  
      ## Setting read_only=1 on the current master so that nobody(except SUPER) can write  
      print current_time_us() . " Set read_only=1 on the orig master.. ";  
      $orig_master_handler->enable_read_only();  
      if ( $orig_master_handler->is_read_only() ) {  
        print "ok.\n";  
      }  
      else {  
        die "Failed!\n";  
      }  
      ## Waiting for M * 100 milliseconds so that current update queries can complete  
      my $time_until_kill_threads = 5;  
      @threads = get_threads_util( $orig_master_handler->{dbh},  
        $orig_master_handler->{connection_id} );  
      while ( $time_until_kill_threads > 0 && $#threads >= 0 ) {  
        if ( $time_until_kill_threads % 5 == 0 ) {  
          printf  
"%s Waiting all running %d queries are disconnected.. (max %d milliseconds)\n",  
            current_time_us(), $#threads + 1, $time_until_kill_threads * 100;  
          if ( $#threads < 5 ) {  
            print Data::Dumper->new( [$_] )->Indent(0)->Terse(1)->Dump . "\n"  
              foreach (@threads);  
          }  
        }  
        sleep_until();  
        $_tstart = [gettimeofday];  
        $time_until_kill_threads--;  
        @threads = get_threads_util( $orig_master_handler->{dbh},  
          $orig_master_handler->{connection_id} );  
      }  
###########################################################################  
      print "disable the VIP on old master: $orig_master_host \n";  
      &stop_vip();  
      &start_arp();
###########################################################################  
      ## Terminating all threads  
      print current_time_us() . " Killing all application threads..\n";  
      $orig_master_handler->kill_threads(@threads) if ( $#threads >= 0 );  
      print current_time_us() . " done.\n";  
      $orig_master_handler->enable_log_bin_local();  
      $orig_master_handler->disconnect();  
      ## After finishing the script, MHA executes FLUSH TABLES WITH READ LOCK  
      $exit_code = 0;  
    };  
    if ($@) {  
      warn "Got Error: $@\n";  
      exit $exit_code;  
    }  
    exit $exit_code;  
  }  
  elsif ( $command eq "start" ) {  
    ## Activating master ip on the new master  
    # 1. Create app user with write privileges  
    # 2. Moving backup script if needed  
    # 3. Register new master's ip to the catalog database  
    my $exit_code = 10;  
    eval {  
      my $new_master_handler = new MHA::DBHelper();  
      # args: hostname, port, user, password, raise_error_or_not  
      $new_master_handler->connect( $new_master_ip, $new_master_port,  
        $new_master_user, $new_master_password, 1 );  
      ## Set read_only=0 on the new master  
      $new_master_handler->disable_log_bin_local();  
      print current_time_us() . " Set read_only=0 on the new master.\n";  
      $new_master_handler->disable_read_only();  
      ## Creating an app user on the new master  
      print current_time_us() . " Creating app user on the new master..\n";  
###########################################################################  
      #FIXME_xxx_create_app_user($new_master_handler);  
###########################################################################  
      $new_master_handler->enable_log_bin_local();  
      $new_master_handler->disconnect();  
      ## Update master ip on the catalog database, etc  
###############################################################################  
      print "enable the VIP: $vip on the new master: $new_master_host \n ";  
      &start_vip();  
###############################################################################  
      $exit_code = 0;  
    };  
    if ($@) {  
      warn "Got Error: $@\n";  
      exit $exit_code;  
    }  
    exit $exit_code;  
  }  
  elsif ( $command eq "status" ) {  
    # do nothing  
    exit 0;  
  }  
  else {  
    &usage();  
    exit 1;  
  }  
}  
###########################################################################  
sub start_vip() {  
    `ssh $new_master_ssh_user\@$new_master_host \" $ssh_start_vip \"`;  
}  
sub stop_vip() {  
    `ssh $orig_master_ssh_user\@$orig_master_host \" $ssh_stop_vip \"`;  
}  
sub start_arp() {
    `ssh $new_master_ssh_user\@$new_master_host \" $ssh_Bcast_arp \"`;
}
###########################################################################    
sub usage {  
  print  
"Usage: master_ip_online_change --command=start|stop|status --orig_master_host=host --orig_master_ip=ip --orig_master_port=port --new_master_host=host --new_master_ip=ip --new_master_port=port\n";  
  die;  
} 
```

# 飞书告警脚本

```bash

[root@wtj1vpztmysql04 masterha]# cat /script/masterha/send_master_failover_mail 
#!/usr/bin/perl

#  Copyright (C) 2011 DeNA Co.,Ltd.
#
#  This program is free software; you can redistribute it and/or modify
#  it under the terms of the GNU General Public License as published by
#  the Free Software Foundation; either version 2 of the License, or
#  (at your option) any later version.
#
#  This program is distributed in the hope that it will be useful,
#  but WITHOUT ANY WARRANTY; without even the implied warranty of
#  MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
#  GNU General Public License for more details.
#
#  You should have received a copy of the GNU General Public License
#   along with this program; if not, write to the Free Software
#  Foundation, Inc.,
#  51 Franklin Street, Fifth Floor, Boston, MA  02110-1301  USA

## Note: This is a sample script and is not complete. Modify the script based on your environment.

use strict;
use warnings FATAL => 'all';
use Getopt::Long;
use LWP::UserAgent;
use JSON;

# new_master_host and new_slave_hosts are set only when recovering master succeeded
my ($dead_master_host, $new_master_host, $new_slave_hosts, $subject, $body);
my $webhook_url = 'https://open.feishu.cn/open-apis/bot/v2/hook/ec84bade-58d3-4852-91cf-983c0bac2411';
GetOptions(
  'orig_master_host=s' => \$dead_master_host,
  'new_master_host=s'  => \$new_master_host,
  'new_slave_hosts=s'  => \$new_slave_hosts,
  'subject=s'          => \$subject,
  'body=s'             => \$body,
);

sendToFeishu($webhook_url, $subject, $body);

sub sendToFeishu {
    my ($webhook_url, $title, $text) = @_;
    my $ua = LWP::UserAgent->new;
    my %payload = (
        msg_type => 'text',
        content  => {
            text => "$title\n$text"
        }
    );
    my $response = $ua->post(
        $webhook_url,
        'Content-Type' => 'application/json',
        Content        => encode_json(\%payload)
    );

    if ($response->is_success) {
        print "Message sent successfully\n";
    } else {
        die "Failed to send message: " . $response->status_line;
    }
}

# Do whatever you want here

exit 0;
```

## 启动mha进程：
```js
[root@wtj1vpztmysql04 masterha]# nohup masterha_manager --conf=/etc/masterha/app2.cnf 2>&1 &
[1] 30604
[root@wtj1vpztmysql04 masterha]# nohup: ignoring input and appending output to ‘nohup.out’
[root@wtj1vpztmysql04 masterha]# masterha_check_status --conf=/etc/masterha/app2.cnf
app2 (pid:30604) is running(0:PING_OK), master:172.17.44.45
```

> [!NOTE] **无需手工添加vip**
> 不需要手工到主库上，进行虚拟vip网卡ip的添加，启动mha进程后，会在主库服务器自动启动vip。

## 在主库服务器查看vip
```js
[root@wtj1vpztmysql02 ~]# ifconfig
ens192: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.44.45  netmask 255.255.252.0  broadcast 172.17.47.255
        inet6 fe80::aca1:4d91:5d6f:75d4  prefixlen 64  scopeid 0x20<link>
        inet6 fe80::5388:16ac:e3ca:4969  prefixlen 64  scopeid 0x20<link>
        inet6 fe80::9b3f:61d2:54f5:d453  prefixlen 64  scopeid 0x20<link>
        ether 00:50:56:98:98:86  txqueuelen 1000  (Ethernet)
        RX packets 1140928122  bytes 6451134289579 (5.8 TiB)
        RX errors 0  dropped 100118  overruns 0  frame 0
        TX packets 1014019132  bytes 5304036383637 (4.8 TiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

ens192:1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.44.145  netmask 255.255.255.0  broadcast 172.17.44.255
        ether 00:50:56:98:98:86  txqueuelen 1000  (Ethernet)

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 330838978  bytes 786532734239 (732.5 GiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 330838978  bytes 786532734239 (732.5 GiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```

## 验证故障自动转移

### 关闭主库服务
```js
[root@wtj1vpztmysql02 ~]# service mysql3307 stop
Shutting down MySQL............ SUCCESS! 
[root@wtj1vpztmysql02 ~]# 
```

# 查看mha_manger日志
```js
[root@wtj1vpztmysql04 masterha]# tail -f /masterha/app2/manager_app2.log 
Thu Aug  1 17:22:02 2024 - [debug]  Disconnected from 172.17.44.143(172.17.44.143:3307)
Thu Aug  1 17:22:02 2024 - [debug]  Disconnected from 172.17.44.44(172.17.44.44:3307)
Thu Aug  1 17:22:02 2024 - [debug] SSH check command: exit 0
Thu Aug  1 17:22:02 2024 - [info] Set master ping interval 1 seconds.
Thu Aug  1 17:22:02 2024 - [info] Set secondary check script: /usr/local/bin/masterha_secondary_check -s 172.17.44.44 -s 172.17.44.143 --user=root --master_host=mha-master-01 --master_ip=172.17.44.45 --master_port=3307
Thu Aug  1 17:22:02 2024 - [info] Starting ping health check on 172.17.44.45(172.17.44.45:3307)..
Thu Aug  1 17:22:02 2024 - [debug] Connected on master.
Thu Aug  1 17:22:02 2024 - [debug] Set short wait_timeout on master: 2 seconds
Thu Aug  1 17:22:02 2024 - [debug] Trying to get advisory lock..
Thu Aug  1 17:22:02 2024 - [info] Ping(SELECT) succeeded, waiting until MySQL doesn't respond..





Thu Aug  1 17:30:04 2024 - [warning] Got error on MySQL select ping: 1053 (Server shutdown in progress)
Thu Aug  1 17:30:04 2024 - [info] Executing secondary network check script: /usr/local/bin/masterha_secondary_check -s 172.17.44.44 -s 172.17.44.143 --user=root --master_host=mha-master-01 --master_ip=172.17.44.45 --master_port=3307  --user=root  --master_host=172.17.44.45  --master_ip=172.17.44.45  --master_port=3307 --master_user=mha_admin --master_password=Mha_sunac_2024 --ping_type=SELECT
Thu Aug  1 17:30:04 2024 - [info] Executing SSH check script: exit 0
Thu Aug  1 17:30:04 2024 - [debug] SSH connection test to 172.17.44.45, option -o StrictHostKeyChecking=no -o PasswordAuthentication=no -o BatchMode=yes -o ConnectTimeout=5, timeout 5
Thu Aug  1 17:30:04 2024 - [info] HealthCheck: SSH to 172.17.44.45 is reachable.
Monitoring server 172.17.44.44 is reachable, Master is not reachable from 172.17.44.44. OK.
Monitoring server 172.17.44.143 is reachable, Master is not reachable from 172.17.44.143. OK.
Thu Aug  1 17:30:05 2024 - [info] Master is not reachable from all other monitoring servers. Failover should start.
Thu Aug  1 17:30:05 2024 - [warning] Got error on MySQL connect: 2003 (Can't connect to MySQL server on '172.17.44.45' (111))
Thu Aug  1 17:30:05 2024 - [warning] Connection failed 2 time(s)..
Thu Aug  1 17:30:06 2024 - [warning] Got error on MySQL connect: 2003 (Can't connect to MySQL server on '172.17.44.45' (111))
Thu Aug  1 17:30:06 2024 - [warning] Connection failed 3 time(s)..
Thu Aug  1 17:30:07 2024 - [warning] Got error on MySQL connect: 2003 (Can't connect to MySQL server on '172.17.44.45' (111))
Thu Aug  1 17:30:07 2024 - [warning] Connection failed 4 time(s)..
Thu Aug  1 17:30:07 2024 - [warning] Master is not reachable from health checker!
Thu Aug  1 17:30:07 2024 - [warning] Master 172.17.44.45(172.17.44.45:3307) is not reachable!
Thu Aug  1 17:30:07 2024 - [warning] SSH is reachable.
Thu Aug  1 17:30:07 2024 - [info] Connecting to a master server failed. Reading configuration file /etc/masterha_default.cnf and /etc/masterha/app2.cnf again, and trying to connect to all servers to check server status..
Thu Aug  1 17:30:07 2024 - [info] Reading default configuration from /etc/masterha_default.cnf..
Thu Aug  1 17:30:07 2024 - [info] Reading application default configuration from /etc/masterha/app2.cnf..
Thu Aug  1 17:30:07 2024 - [info] Reading server configuration from /etc/masterha/app2.cnf..
Thu Aug  1 17:30:07 2024 - [debug] Skipping connecting to dead master 172.17.44.45(172.17.44.45:3307).
Thu Aug  1 17:30:07 2024 - [debug] Connecting to servers..
Thu Aug  1 17:30:08 2024 - [debug]  Connected to: 172.17.44.143(172.17.44.143:3307), user=mha_admin
Thu Aug  1 17:30:08 2024 - [debug]  Number of slave worker threads on host 172.17.44.143(172.17.44.143:3307): 4
Thu Aug  1 17:30:08 2024 - [debug]  Connected to: 172.17.44.44(172.17.44.44:3307), user=mha_admin
Thu Aug  1 17:30:08 2024 - [debug]  Number of slave worker threads on host 172.17.44.44(172.17.44.44:3307): 2
Thu Aug  1 17:30:08 2024 - [debug]  Comparing MySQL versions..
Thu Aug  1 17:30:08 2024 - [debug]   Comparing MySQL versions done.
Thu Aug  1 17:30:08 2024 - [debug] Connecting to servers done.
Thu Aug  1 17:30:08 2024 - [info] GTID failover mode = 1
Thu Aug  1 17:30:08 2024 - [info] Dead Servers:
Thu Aug  1 17:30:08 2024 - [info]   172.17.44.45(172.17.44.45:3307)
Thu Aug  1 17:30:08 2024 - [info] Alive Servers:
Thu Aug  1 17:30:08 2024 - [info]   172.17.44.143(172.17.44.143:3307)
Thu Aug  1 17:30:08 2024 - [info]   172.17.44.44(172.17.44.44:3307)
Thu Aug  1 17:30:08 2024 - [info] Alive Slaves:
Thu Aug  1 17:30:08 2024 - [info]   172.17.44.143(172.17.44.143:3307)  Version=8.0.28 (oldest major version between slaves) log-bin:enabled
Thu Aug  1 17:30:08 2024 - [info]     GTID ON
Thu Aug  1 17:30:08 2024 - [debug]    Relay log info repository: TABLE
Thu Aug  1 17:30:08 2024 - [info]     Replicating from 172.17.44.45(172.17.44.45:3307)
Thu Aug  1 17:30:08 2024 - [info]     Primary candidate for the new Master (candidate_master is set)
Thu Aug  1 17:30:08 2024 - [info]   172.17.44.44(172.17.44.44:3307)  Version=8.0.28 (oldest major version between slaves) log-bin:enabled
Thu Aug  1 17:30:08 2024 - [info]     GTID ON
Thu Aug  1 17:30:08 2024 - [debug]    Relay log info repository: TABLE
Thu Aug  1 17:30:08 2024 - [info]     Replicating from 172.17.44.45(172.17.44.45:3307)
Thu Aug  1 17:30:08 2024 - [info]     Not candidate for the new Master (no_master is set)
Thu Aug  1 17:30:08 2024 - [info] Checking slave configurations..
Thu Aug  1 17:30:08 2024 - [info]  read_only=1 is not set on slave 172.17.44.143(172.17.44.143:3307).
Thu Aug  1 17:30:08 2024 - [info]  read_only=1 is not set on slave 172.17.44.44(172.17.44.44:3307).
Thu Aug  1 17:30:08 2024 - [info] Checking replication filtering settings..
Thu Aug  1 17:30:08 2024 - [info]  Replication filtering check ok.
Thu Aug  1 17:30:08 2024 - [info] Master is down!
Thu Aug  1 17:30:08 2024 - [info] Terminating monitoring script.
Thu Aug  1 17:30:08 2024 - [debug]  Disconnected from 172.17.44.143(172.17.44.143:3307)
Thu Aug  1 17:30:08 2024 - [debug]  Disconnected from 172.17.44.44(172.17.44.44:3307)
Thu Aug  1 17:30:08 2024 - [info] Got exit code 20 (Master dead).
Thu Aug  1 17:30:08 2024 - [info] MHA::MasterFailover version 0.58.
Thu Aug  1 17:30:08 2024 - [info] Starting master failover.
Thu Aug  1 17:30:08 2024 - [info] 
Thu Aug  1 17:30:08 2024 - [info] * Phase 1: Configuration Check Phase..
Thu Aug  1 17:30:08 2024 - [info] 
Thu Aug  1 17:30:08 2024 - [debug] Skipping connecting to dead master 172.17.44.45.
Thu Aug  1 17:30:08 2024 - [debug] Connecting to servers..
Thu Aug  1 17:30:09 2024 - [debug]  Connected to: 172.17.44.143(172.17.44.143:3307), user=mha_admin
Thu Aug  1 17:30:09 2024 - [debug]  Number of slave worker threads on host 172.17.44.143(172.17.44.143:3307): 4
Thu Aug  1 17:30:09 2024 - [debug]  Connected to: 172.17.44.44(172.17.44.44:3307), user=mha_admin
Thu Aug  1 17:30:09 2024 - [debug]  Number of slave worker threads on host 172.17.44.44(172.17.44.44:3307): 2
Thu Aug  1 17:30:09 2024 - [debug]  Comparing MySQL versions..
Thu Aug  1 17:30:09 2024 - [debug]   Comparing MySQL versions done.
Thu Aug  1 17:30:09 2024 - [debug] Connecting to servers done.
Thu Aug  1 17:30:09 2024 - [info] GTID failover mode = 1
Thu Aug  1 17:30:09 2024 - [info] Dead Servers:
Thu Aug  1 17:30:09 2024 - [info]   172.17.44.45(172.17.44.45:3307)
Thu Aug  1 17:30:09 2024 - [info] Checking master reachability via MySQL(double check)...
Thu Aug  1 17:30:09 2024 - [info]  ok.
Thu Aug  1 17:30:09 2024 - [info] Alive Servers:
Thu Aug  1 17:30:09 2024 - [info]   172.17.44.143(172.17.44.143:3307)
Thu Aug  1 17:30:09 2024 - [info]   172.17.44.44(172.17.44.44:3307)
Thu Aug  1 17:30:09 2024 - [info] Alive Slaves:
Thu Aug  1 17:30:09 2024 - [info]   172.17.44.143(172.17.44.143:3307)  Version=8.0.28 (oldest major version between slaves) log-bin:enabled
Thu Aug  1 17:30:09 2024 - [info]     GTID ON
Thu Aug  1 17:30:09 2024 - [debug]    Relay log info repository: TABLE
Thu Aug  1 17:30:09 2024 - [info]     Replicating from 172.17.44.45(172.17.44.45:3307)
Thu Aug  1 17:30:09 2024 - [info]     Primary candidate for the new Master (candidate_master is set)
Thu Aug  1 17:30:09 2024 - [info]   172.17.44.44(172.17.44.44:3307)  Version=8.0.28 (oldest major version between slaves) log-bin:enabled
Thu Aug  1 17:30:09 2024 - [info]     GTID ON
Thu Aug  1 17:30:09 2024 - [debug]    Relay log info repository: TABLE
Thu Aug  1 17:30:09 2024 - [info]     Replicating from 172.17.44.45(172.17.44.45:3307)
Thu Aug  1 17:30:09 2024 - [info]     Not candidate for the new Master (no_master is set)
Thu Aug  1 17:30:09 2024 - [info] Starting GTID based failover.
Thu Aug  1 17:30:09 2024 - [info] 
Thu Aug  1 17:30:09 2024 - [info] ** Phase 1: Configuration Check Phase completed.
Thu Aug  1 17:30:09 2024 - [info] 
Thu Aug  1 17:30:09 2024 - [info] * Phase 2: Dead Master Shutdown Phase..
Thu Aug  1 17:30:09 2024 - [info] 
Thu Aug  1 17:30:09 2024 - [info] Forcing shutdown so that applications never connect to the current master..
Thu Aug  1 17:30:09 2024 - [info] Executing master IP deactivation script:
Thu Aug  1 17:30:09 2024 - [info]   /script/masterha/master_ip_failover.3307 --orig_master_host=172.17.44.45 --orig_master_ip=172.17.44.45 --orig_master_port=3307 --command=stopssh --ssh_user=root  
Thu Aug  1 17:30:09 2024 - [debug]  Stopping IO thread on 172.17.44.143(172.17.44.143:3307)..
Thu Aug  1 17:30:09 2024 - [debug]  Stopping IO thread on 172.17.44.44(172.17.44.44:3307)..
Thu Aug  1 17:30:09 2024 - [debug]  Stop IO thread on 172.17.44.143(172.17.44.143:3307) done.
Thu Aug  1 17:30:09 2024 - [debug]  Stop IO thread on 172.17.44.44(172.17.44.44:3307) done.


IN SCRIPT TEST====/sbin/ifconfig ens192:1 down==/sbin/ifconfig ens192:1 172.17.44.145/24===

Disabling the VIP on old master: 172.17.44.45 
Thu Aug  1 17:30:09 2024 - [info]  done.
Thu Aug  1 17:30:09 2024 - [warning] shutdown_script is not set. Skipping explicit shutting down of the dead master.
Thu Aug  1 17:30:09 2024 - [info] * Phase 2: Dead Master Shutdown Phase completed.
Thu Aug  1 17:30:09 2024 - [info] 
Thu Aug  1 17:30:09 2024 - [info] * Phase 3: Master Recovery Phase..
Thu Aug  1 17:30:09 2024 - [info] 
Thu Aug  1 17:30:09 2024 - [info] * Phase 3.1: Getting Latest Slaves Phase..
Thu Aug  1 17:30:09 2024 - [info] 
Thu Aug  1 17:30:09 2024 - [debug] Fetching current slave status..
Thu Aug  1 17:30:09 2024 - [debug]  Fetching current slave status done.
Thu Aug  1 17:30:09 2024 - [info] The latest binary log file/position on all slaves is mysql-bin-sunac_3307.000001:5870
Thu Aug  1 17:30:09 2024 - [info] Retrieved Gtid Set: a4164c09-4fbb-11ef-9e31-005056989886:1-24
Thu Aug  1 17:30:09 2024 - [info] Latest slaves (Slaves that received relay log files to the latest):
Thu Aug  1 17:30:09 2024 - [info]   172.17.44.143(172.17.44.143:3307)  Version=8.0.28 (oldest major version between slaves) log-bin:enabled
Thu Aug  1 17:30:09 2024 - [info]     GTID ON
Thu Aug  1 17:30:09 2024 - [debug]    Relay log info repository: TABLE
Thu Aug  1 17:30:09 2024 - [info]     Replicating from 172.17.44.45(172.17.44.45:3307)
Thu Aug  1 17:30:09 2024 - [info]     Primary candidate for the new Master (candidate_master is set)
Thu Aug  1 17:30:09 2024 - [info]   172.17.44.44(172.17.44.44:3307)  Version=8.0.28 (oldest major version between slaves) log-bin:enabled
Thu Aug  1 17:30:09 2024 - [info]     GTID ON
Thu Aug  1 17:30:09 2024 - [debug]    Relay log info repository: TABLE
Thu Aug  1 17:30:09 2024 - [info]     Replicating from 172.17.44.45(172.17.44.45:3307)
Thu Aug  1 17:30:09 2024 - [info]     Not candidate for the new Master (no_master is set)
Thu Aug  1 17:30:09 2024 - [info] The oldest binary log file/position on all slaves is mysql-bin-sunac_3307.000001:5870
Thu Aug  1 17:30:09 2024 - [info] Retrieved Gtid Set: a4164c09-4fbb-11ef-9e31-005056989886:1-24
Thu Aug  1 17:30:09 2024 - [info] Oldest slaves:
Thu Aug  1 17:30:09 2024 - [info]   172.17.44.143(172.17.44.143:3307)  Version=8.0.28 (oldest major version between slaves) log-bin:enabled
Thu Aug  1 17:30:09 2024 - [info]     GTID ON
Thu Aug  1 17:30:09 2024 - [debug]    Relay log info repository: TABLE
Thu Aug  1 17:30:09 2024 - [info]     Replicating from 172.17.44.45(172.17.44.45:3307)
Thu Aug  1 17:30:09 2024 - [info]     Primary candidate for the new Master (candidate_master is set)
Thu Aug  1 17:30:09 2024 - [info]   172.17.44.44(172.17.44.44:3307)  Version=8.0.28 (oldest major version between slaves) log-bin:enabled
Thu Aug  1 17:30:09 2024 - [info]     GTID ON
Thu Aug  1 17:30:09 2024 - [debug]    Relay log info repository: TABLE
Thu Aug  1 17:30:09 2024 - [info]     Replicating from 172.17.44.45(172.17.44.45:3307)
Thu Aug  1 17:30:09 2024 - [info]     Not candidate for the new Master (no_master is set)
Thu Aug  1 17:30:09 2024 - [info] 
Thu Aug  1 17:30:09 2024 - [info] * Phase 3.3: Determining New Master Phase..
Thu Aug  1 17:30:09 2024 - [info] 
Thu Aug  1 17:30:09 2024 - [debug] Checking replication delay on 172.17.44.143(172.17.44.143:3307).. 
Thu Aug  1 17:30:09 2024 - [debug]  ok.
Thu Aug  1 17:30:09 2024 - [info] Searching new master from slaves..
Thu Aug  1 17:30:09 2024 - [info]  Candidate masters from the configuration file:
Thu Aug  1 17:30:09 2024 - [info]   172.17.44.143(172.17.44.143:3307)  Version=8.0.28 (oldest major version between slaves) log-bin:enabled
Thu Aug  1 17:30:09 2024 - [info]     GTID ON
Thu Aug  1 17:30:09 2024 - [debug]    Relay log info repository: TABLE
Thu Aug  1 17:30:09 2024 - [info]     Replicating from 172.17.44.45(172.17.44.45:3307)
Thu Aug  1 17:30:09 2024 - [info]     Primary candidate for the new Master (candidate_master is set)
Thu Aug  1 17:30:09 2024 - [info]  Non-candidate masters:
Thu Aug  1 17:30:09 2024 - [info]   172.17.44.44(172.17.44.44:3307)  Version=8.0.28 (oldest major version between slaves) log-bin:enabled
Thu Aug  1 17:30:09 2024 - [info]     GTID ON
Thu Aug  1 17:30:09 2024 - [debug]    Relay log info repository: TABLE
Thu Aug  1 17:30:09 2024 - [info]     Replicating from 172.17.44.45(172.17.44.45:3307)
Thu Aug  1 17:30:09 2024 - [info]     Not candidate for the new Master (no_master is set)
Thu Aug  1 17:30:09 2024 - [info]  Searching from candidate_master slaves which have received the latest relay log events..
Thu Aug  1 17:30:09 2024 - [info] New master is 172.17.44.143(172.17.44.143:3307)
Thu Aug  1 17:30:09 2024 - [info] Starting master failover..
Thu Aug  1 17:30:09 2024 - [info] 
From:
172.17.44.45(172.17.44.45:3307) (current master)
 +--172.17.44.143(172.17.44.143:3307)
 +--172.17.44.44(172.17.44.44:3307)

To:
172.17.44.143(172.17.44.143:3307) (new master)
 +--172.17.44.44(172.17.44.44:3307)
Thu Aug  1 17:30:09 2024 - [info] 
Thu Aug  1 17:30:09 2024 - [info] * Phase 3.3: New Master Recovery Phase..
Thu Aug  1 17:30:09 2024 - [info] 
Thu Aug  1 17:30:09 2024 - [info]  Waiting all logs to be applied.. 
Thu Aug  1 17:30:09 2024 - [info]   done.
Thu Aug  1 17:30:09 2024 - [debug]  Stopping slave IO/SQL thread on 172.17.44.143(172.17.44.143:3307)..
Thu Aug  1 17:30:10 2024 - [debug]   done.
Thu Aug  1 17:30:10 2024 - [info] Getting new master's binlog name and position..
Thu Aug  1 17:30:10 2024 - [info]  mysql-bin-sunac_3307.000001:6632
Thu Aug  1 17:30:10 2024 - [info]  All other slaves should start replication from here. Statement should be: CHANGE MASTER TO MASTER_HOST='172.17.44.143', MASTER_PORT=3307, MASTER_AUTO_POSITION=1, MASTER_USER='rpl_slave', MASTER_PASSWORD='xxx';
Thu Aug  1 17:30:10 2024 - [info] Master Recovery succeeded. File:Pos:Exec_Gtid_Set: mysql-bin-sunac_3307.000001, 6632, a4164c09-4fbb-11ef-9e31-005056989886:1-24,
a5aa9e7b-4fce-11ef-8641-0050569859cb:1-3
Thu Aug  1 17:30:10 2024 - [info] Executing master IP activate script:
Thu Aug  1 17:30:10 2024 - [info]   /script/masterha/master_ip_failover.3307 --command=start --ssh_user=root --orig_master_host=172.17.44.45 --orig_master_ip=172.17.44.45 --orig_master_port=3307 --new_master_host=172.17.44.143 --new_master_ip=172.17.44.143 --new_master_port=3307 --new_master_user='mha_admin'   --new_master_password=xxx
Unknown option: new_master_user
Unknown option: new_master_password


IN SCRIPT TEST====/sbin/ifconfig ens192:1 down==/sbin/ifconfig ens192:1 172.17.44.145/24===

Enabling the VIP - 172.17.44.145/24 on the new master - 172.17.44.143 
Thu Aug  1 17:30:10 2024 - [info]  OK.
Thu Aug  1 17:30:10 2024 - [info] ** Finished master recovery successfully.
Thu Aug  1 17:30:10 2024 - [info] * Phase 3: Master Recovery Phase completed.
Thu Aug  1 17:30:10 2024 - [info] 
Thu Aug  1 17:30:10 2024 - [info] * Phase 4: Slaves Recovery Phase..
Thu Aug  1 17:30:10 2024 - [info] 
Thu Aug  1 17:30:10 2024 - [info] 
Thu Aug  1 17:30:10 2024 - [info] * Phase 4.1: Starting Slaves in parallel..
Thu Aug  1 17:30:10 2024 - [info] 
Thu Aug  1 17:30:10 2024 - [info] -- Slave recovery on host 172.17.44.44(172.17.44.44:3307) started, pid: 31148. Check tmp log /masterha/app2/172.17.44.44_3307_20240801173008.log if it takes time..
Thu Aug  1 17:30:11 2024 - [info] 
Thu Aug  1 17:30:11 2024 - [info] Log messages from 172.17.44.44 ...
Thu Aug  1 17:30:11 2024 - [info] 
Thu Aug  1 17:30:10 2024 - [info]  Resetting slave 172.17.44.44(172.17.44.44:3307) and starting replication from the new master 172.17.44.143(172.17.44.143:3307)..
Thu Aug  1 17:30:10 2024 - [debug]  Stopping slave IO/SQL thread on 172.17.44.44(172.17.44.44:3307)..
Thu Aug  1 17:30:10 2024 - [debug]   done.
Thu Aug  1 17:30:10 2024 - [info]  Executed CHANGE MASTER.
Thu Aug  1 17:30:10 2024 - [debug]  Starting slave IO/SQL thread on 172.17.44.44(172.17.44.44:3307)..
Thu Aug  1 17:30:10 2024 - [debug]   done.
Thu Aug  1 17:30:10 2024 - [info]  Slave started.
Thu Aug  1 17:30:10 2024 - [info]  gtid_wait(a4164c09-4fbb-11ef-9e31-005056989886:1-24,
a5aa9e7b-4fce-11ef-8641-0050569859cb:1-3) completed on 172.17.44.44(172.17.44.44:3307). Executed 4 events.
Thu Aug  1 17:30:11 2024 - [info] End of log messages from 172.17.44.44.
Thu Aug  1 17:30:11 2024 - [info] -- Slave on host 172.17.44.44(172.17.44.44:3307) started.
Thu Aug  1 17:30:11 2024 - [info] All new slave servers recovered successfully.
Thu Aug  1 17:30:11 2024 - [info] 
Thu Aug  1 17:30:11 2024 - [info] * Phase 5: New master cleanup phase..
Thu Aug  1 17:30:11 2024 - [info] 
Thu Aug  1 17:30:11 2024 - [info] Resetting slave info on the new master..
Thu Aug  1 17:30:11 2024 - [debug]  Clearing slave info..
Thu Aug  1 17:30:11 2024 - [debug]  Stopping slave IO/SQL thread on 172.17.44.143(172.17.44.143:3307)..
Thu Aug  1 17:30:11 2024 - [debug]   done.
Thu Aug  1 17:30:11 2024 - [debug]  SHOW SLAVE STATUS shows new master does not replicate from anywhere. OK.
Thu Aug  1 17:30:11 2024 - [info]  172.17.44.143: Resetting slave info succeeded.
Thu Aug  1 17:30:11 2024 - [info] Master failover to 172.17.44.143(172.17.44.143:3307) completed successfully.
Thu Aug  1 17:30:11 2024 - [debug]  Disconnected from 172.17.44.143(172.17.44.143:3307)
Thu Aug  1 17:30:11 2024 - [debug]  Disconnected from 172.17.44.44(172.17.44.44:3307)
Thu Aug  1 17:30:11 2024 - [info] 

----- Failover Report -----

app2: MySQL Master failover 172.17.44.45(172.17.44.45:3307) to 172.17.44.143(172.17.44.143:3307) succeeded

Master 172.17.44.45(172.17.44.45:3307) is down!

Check MHA Manager logs at wtj1vpztmysql04:/masterha/app2/manager_app2.log for details.

Started automated(non-interactive) failover.
Invalidated master IP address on 172.17.44.45(172.17.44.45:3307)
Selected 172.17.44.143(172.17.44.143:3307) as a new master.
172.17.44.143(172.17.44.143:3307): OK: Applying all logs succeeded.
172.17.44.143(172.17.44.143:3307): OK: Activated master IP address.
172.17.44.44(172.17.44.44:3307): OK: Slave started, replicating from 172.17.44.143(172.17.44.143:3307)
172.17.44.143(172.17.44.143:3307): Resetting slave info succeeded.
Master failover to 172.17.44.143(172.17.44.143:3307) completed successfully.

```

## 新的主库查询
```bash

mysql> show slave hosts;
+-----------+--------------+------+-----------+--------------------------------------+
| Server_id | Host         | Port | Master_id | Slave_UUID                           |
+-----------+--------------+------+-----------+--------------------------------------+
|      1044 | 172.17.44.44 | 3307 |     10143 | 6053e59e-4fee-11ef-8631-005056986de2 |
|      1045 | 172.17.44.45 | 3307 |     10143 | a4164c09-4fbb-11ef-9e31-005056989886 |
+-----------+--------------+------+-----------+--------------------------------------+
2 rows in set, 1 warning (0.00 sec)
```



## 重新监听进程
```js
[root@wtj1vpztmysql04 masterha]# nohup masterha_manager --conf=/etc/masterha/app2.cnf 2>&1 &
nohup: ignoring input and appending output to ‘nohup.out’
[root@wtj1vpztmysql04 masterha]# masterha_check_status --conf=/etc/masterha/app2.cnf
app2 is stopped(2:NOT_RUNNING).

-- 检查发现，进程没有启动，需要删除app2.failover.complete
[root@wtj1vpztmysql04 masterha]# ls /masterha/app2/
app2.failover.complete  manager_app2.log        
[root@wtj1vpztmysql04 masterha]# rm -rf /masterha/app2/app2.failover.complete
```

## 停止新主库的服务
```js
[root@wtj1vpztmysql03 deploy_mysql]# service mysql3307 stop
Shutting down MySQL........... SUCCESS! 
```

# 手工切换测试

```bash
[root@wtj1vpztmysql04 ~]# masterha_master_switch --master_state=alive --conf=/etc/masterha/app2.cnf
Tue Aug  6 14:23:49 2024 - [info] MHA::MasterRotate version 0.58.
Tue Aug  6 14:23:49 2024 - [info] Starting online master switch..
Tue Aug  6 14:23:49 2024 - [info] 
Tue Aug  6 14:23:49 2024 - [info] * Phase 1: Configuration Check Phase..
Tue Aug  6 14:23:49 2024 - [info] 
Tue Aug  6 14:23:49 2024 - [info] Reading default configuration from /etc/masterha_default.cnf..
Tue Aug  6 14:23:49 2024 - [info] Reading application default configuration from /etc/masterha/app2.cnf..
Tue Aug  6 14:23:49 2024 - [info] Reading server configuration from /etc/masterha/app2.cnf..
Tue Aug  6 14:23:49 2024 - [debug] Connecting to servers..
Tue Aug  6 14:23:50 2024 - [debug]  Connected to: 172.17.44.45(172.17.44.45:3307), user=mha_admin
Tue Aug  6 14:23:50 2024 - [debug]  Number of slave worker threads on host 172.17.44.45(172.17.44.45:3307): 2
Tue Aug  6 14:23:50 2024 - [debug]  Connected to: 172.17.44.143(172.17.44.143:3307), user=mha_admin
Tue Aug  6 14:23:50 2024 - [debug]  Number of slave worker threads on host 172.17.44.143(172.17.44.143:3307): 4
Tue Aug  6 14:23:50 2024 - [debug]  Connected to: 172.17.44.44(172.17.44.44:3307), user=mha_admin
Tue Aug  6 14:23:50 2024 - [debug]  Number of slave worker threads on host 172.17.44.44(172.17.44.44:3307): 4
Tue Aug  6 14:23:50 2024 - [debug]  Comparing MySQL versions..
Tue Aug  6 14:23:50 2024 - [debug]   Comparing MySQL versions done.
Tue Aug  6 14:23:50 2024 - [debug] Connecting to servers done.
Tue Aug  6 14:23:50 2024 - [info] GTID failover mode = 1
Tue Aug  6 14:23:50 2024 - [info] Current Alive Master: 172.17.44.45(172.17.44.45:3307)
Tue Aug  6 14:23:50 2024 - [info] Alive Slaves:
Tue Aug  6 14:23:50 2024 - [info]   172.17.44.143(172.17.44.143:3307)  Version=8.0.28 (oldest major version between slaves) log-bin:enabled
Tue Aug  6 14:23:50 2024 - [info]     GTID ON
Tue Aug  6 14:23:50 2024 - [debug]    Relay log info repository: TABLE
Tue Aug  6 14:23:50 2024 - [info]     Replicating from 172.17.44.45(172.17.44.45:3307)
Tue Aug  6 14:23:50 2024 - [info]     Primary candidate for the new Master (candidate_master is set)
Tue Aug  6 14:23:50 2024 - [info]   172.17.44.44(172.17.44.44:3307)  Version=8.0.28 (oldest major version between slaves) log-bin:enabled
Tue Aug  6 14:23:50 2024 - [info]     GTID ON
Tue Aug  6 14:23:50 2024 - [debug]    Relay log info repository: TABLE
Tue Aug  6 14:23:50 2024 - [info]     Replicating from 172.17.44.45(172.17.44.45:3307)
Tue Aug  6 14:23:50 2024 - [info]     Not candidate for the new Master (no_master is set)

It is better to execute FLUSH NO_WRITE_TO_BINLOG TABLES on the master before switching. Is it ok to execute on 172.17.44.45(172.17.44.45:3307)? (YES/no): Yes
Tue Aug  6 14:24:23 2024 - [info] Executing FLUSH NO_WRITE_TO_BINLOG TABLES. This may take long time..
Tue Aug  6 14:24:23 2024 - [info]  ok.
Tue Aug  6 14:24:23 2024 - [info] Checking MHA is not monitoring or doing failover..
Tue Aug  6 14:24:23 2024 - [info] Checking replication health on 172.17.44.143..
Tue Aug  6 14:24:23 2024 - [info]  ok.
Tue Aug  6 14:24:23 2024 - [info] Checking replication health on 172.17.44.44..
Tue Aug  6 14:24:23 2024 - [info]  ok.
Tue Aug  6 14:24:23 2024 - [info] Searching new master from slaves..
Tue Aug  6 14:24:23 2024 - [info]  Candidate masters from the configuration file:
Tue Aug  6 14:24:23 2024 - [info]   172.17.44.45(172.17.44.45:3307)  Version=8.0.28 log-bin:enabled
Tue Aug  6 14:24:23 2024 - [info]     GTID ON
Tue Aug  6 14:24:23 2024 - [info]   172.17.44.143(172.17.44.143:3307)  Version=8.0.28 (oldest major version between slaves) log-bin:enabled
Tue Aug  6 14:24:23 2024 - [info]     GTID ON
Tue Aug  6 14:24:23 2024 - [debug]    Relay log info repository: TABLE
Tue Aug  6 14:24:23 2024 - [info]     Replicating from 172.17.44.45(172.17.44.45:3307)
Tue Aug  6 14:24:23 2024 - [info]     Primary candidate for the new Master (candidate_master is set)
Tue Aug  6 14:24:23 2024 - [info]  Non-candidate masters:
Tue Aug  6 14:24:23 2024 - [info]   172.17.44.44(172.17.44.44:3307)  Version=8.0.28 (oldest major version between slaves) log-bin:enabled
Tue Aug  6 14:24:23 2024 - [info]     GTID ON
Tue Aug  6 14:24:23 2024 - [debug]    Relay log info repository: TABLE
Tue Aug  6 14:24:23 2024 - [info]     Replicating from 172.17.44.45(172.17.44.45:3307)
Tue Aug  6 14:24:23 2024 - [info]     Not candidate for the new Master (no_master is set)
Tue Aug  6 14:24:23 2024 - [info]  Searching from candidate_master slaves which have received the latest relay log events..
Tue Aug  6 14:24:23 2024 - [info] 
From:
172.17.44.45(172.17.44.45:3307) (current master)
 +--172.17.44.143(172.17.44.143:3307)
 +--172.17.44.44(172.17.44.44:3307)

To:
172.17.44.143(172.17.44.143:3307) (new master)
 +--172.17.44.44(172.17.44.44:3307)

Starting master switch from 172.17.44.45(172.17.44.45:3307) to 172.17.44.143(172.17.44.143:3307)? (yes/NO): yes
Tue Aug  6 14:24:33 2024 - [info] Checking whether 172.17.44.143(172.17.44.143:3307) is ok for the new master..
Tue Aug  6 14:24:33 2024 - [info]  ok.
Tue Aug  6 14:24:33 2024 - [info] ** Phase 1: Configuration Check Phase completed.
Tue Aug  6 14:24:33 2024 - [info] 
Tue Aug  6 14:24:33 2024 - [debug]  Disconnected from 172.17.44.45(172.17.44.45:3307)
Tue Aug  6 14:24:33 2024 - [info] * Phase 2: Rejecting updates Phase..
Tue Aug  6 14:24:33 2024 - [info] 
Tue Aug  6 14:24:33 2024 - [info] Executing master ip online change script to disable write on the current master:
Tue Aug  6 14:24:33 2024 - [info]   /script/masterha/master_ip_online_change.3307 --command=stop --orig_master_host=172.17.44.45 --orig_master_ip=172.17.44.45 --orig_master_port=3307 --orig_master_user='mha_admin' --new_master_host=172.17.44.143 --new_master_ip=172.17.44.143 --new_master_port=3307 --new_master_user='mha_admin' --orig_master_ssh_user=root --new_master_ssh_user=root   --orig_master_password=xxx --new_master_password=xxx
Tue Aug  6 14:24:33 2024 787156 Set read_only on the new master.. ok.
Tue Aug  6 14:24:33 2024 789413 Drpping app user on the orig master..
Tue Aug  6 14:24:33 2024 789689 Waiting all running 3 threads are disconnected.. (max 1500 milliseconds)
{'Time' => '74826','db' => undef,'Id' => '5','User' => 'event_scheduler','State' => 'Waiting on empty queue','Command' => 'Daemon','Info' => undef,'Host' => 'localhost'}
{'Time' => '73283','db' => undef,'Id' => '36','User' => 'rpl_slave','State' => 'Source has sent all binlog to replica; waiting for more updates','Command' => 'Binlog Dump GTID','Info' => undef,'Host' => '172.17.44.44:33741'}
{'Time' => '73003','db' => undef,'Id' => '39','User' => 'rpl_slave','State' => 'Source has sent all binlog to replica; waiting for more updates','Command' => 'Binlog Dump GTID','Info' => undef,'Host' => '172.17.44.143:14468'}
Tue Aug  6 14:24:34 2024 290549 Waiting all running 3 threads are disconnected.. (max 1000 milliseconds)
{'Time' => '74826','db' => undef,'Id' => '5','User' => 'event_scheduler','State' => 'Waiting on empty queue','Command' => 'Daemon','Info' => undef,'Host' => 'localhost'}
{'Time' => '73283','db' => undef,'Id' => '36','User' => 'rpl_slave','State' => 'Source has sent all binlog to replica; waiting for more updates','Command' => 'Binlog Dump GTID','Info' => undef,'Host' => '172.17.44.44:33741'}
{'Time' => '73003','db' => undef,'Id' => '39','User' => 'rpl_slave','State' => 'Source has sent all binlog to replica; waiting for more updates','Command' => 'Binlog Dump GTID','Info' => undef,'Host' => '172.17.44.143:14468'}
Tue Aug  6 14:24:34 2024 791097 Waiting all running 3 threads are disconnected.. (max 500 milliseconds)
{'Time' => '74827','db' => undef,'Id' => '5','User' => 'event_scheduler','State' => 'Waiting on empty queue','Command' => 'Daemon','Info' => undef,'Host' => 'localhost'}
{'Time' => '73284','db' => undef,'Id' => '36','User' => 'rpl_slave','State' => 'Source has sent all binlog to replica; waiting for more updates','Command' => 'Binlog Dump GTID','Info' => undef,'Host' => '172.17.44.44:33741'}
{'Time' => '73004','db' => undef,'Id' => '39','User' => 'rpl_slave','State' => 'Source has sent all binlog to replica; waiting for more updates','Command' => 'Binlog Dump GTID','Info' => undef,'Host' => '172.17.44.143:14468'}
Tue Aug  6 14:24:35 2024 291437 Set read_only=1 on the orig master.. ok.
Tue Aug  6 14:24:35 2024 292411 Waiting all running 3 queries are disconnected.. (max 500 milliseconds)
{'Time' => '74827','db' => undef,'Id' => '5','User' => 'event_scheduler','State' => 'Waiting on empty queue','Command' => 'Daemon','Info' => undef,'Host' => 'localhost'}
{'Time' => '73284','db' => undef,'Id' => '36','User' => 'rpl_slave','State' => 'Source has sent all binlog to replica; waiting for more updates','Command' => 'Binlog Dump GTID','Info' => undef,'Host' => '172.17.44.44:33741'}
{'Time' => '73004','db' => undef,'Id' => '39','User' => 'rpl_slave','State' => 'Source has sent all binlog to replica; waiting for more updates','Command' => 'Binlog Dump GTID','Info' => undef,'Host' => '172.17.44.143:14468'}
disable the VIP on old master: 172.17.44.45 
SIOCSIFNETMASK: Cannot assign requested address
Tue Aug  6 14:24:36 2024 010051 Killing all application threads..
Tue Aug  6 14:24:36 2024 027935 done.
Tue Aug  6 14:24:36 2024 - [info]  ok.
Tue Aug  6 14:24:36 2024 - [info] Locking all tables on the orig master to reject updates from everybody (including root):
Tue Aug  6 14:24:36 2024 - [info] Executing FLUSH TABLES WITH READ LOCK..
Tue Aug  6 14:24:36 2024 - [info]  ok.
Tue Aug  6 14:24:36 2024 - [info] Orig master binlog:pos is mysql-bin-sunac_3307.000002:1261.
Tue Aug  6 14:24:36 2024 - [debug] Fetching current slave status..
Tue Aug  6 14:24:36 2024 - [debug]  Fetching current slave status done.
Tue Aug  6 14:24:36 2024 - [info]  Waiting to execute all relay logs on 172.17.44.143(172.17.44.143:3307)..
Tue Aug  6 14:24:36 2024 - [info]  master_pos_wait(mysql-bin-sunac_3307.000002:1261) completed on 172.17.44.143(172.17.44.143:3307). Executed 0 events.
Tue Aug  6 14:24:36 2024 - [info]   done.
Tue Aug  6 14:24:36 2024 - [debug]  Stopping SQL thread on 172.17.44.143(172.17.44.143:3307)..
Tue Aug  6 14:24:36 2024 - [debug]   done.
Tue Aug  6 14:24:36 2024 - [info] Getting new master's binlog name and position..
Tue Aug  6 14:24:36 2024 - [info]  mysql-bin-sunac_3307.000005:702
Tue Aug  6 14:24:36 2024 - [info]  All other slaves should start replication from here. Statement should be: CHANGE MASTER TO MASTER_HOST='172.17.44.143', MASTER_PORT=3307, MASTER_AUTO_POSITION=1, MASTER_USER='rpl_slave', MASTER_PASSWORD='xxx';
Tue Aug  6 14:24:36 2024 - [info] Executing master ip online change script to allow write on the new master:
Tue Aug  6 14:24:36 2024 - [info]   /script/masterha/master_ip_online_change.3307 --command=start --orig_master_host=172.17.44.45 --orig_master_ip=172.17.44.45 --orig_master_port=3307 --orig_master_user='mha_admin' --new_master_host=172.17.44.143 --new_master_ip=172.17.44.143 --new_master_port=3307 --new_master_user='mha_admin' --orig_master_ssh_user=root --new_master_ssh_user=root   --orig_master_password=xxx --new_master_password=xxx
Tue Aug  6 14:24:36 2024 171859 Set read_only=0 on the new master.
Tue Aug  6 14:24:36 2024 172383 Creating app user on the new master..
enable the VIP: 172.17.44.145/21 on the new master: 172.17.44.143 
 Tue Aug  6 14:24:36 2024 - [info]  ok.
Tue Aug  6 14:24:36 2024 - [info] 
Tue Aug  6 14:24:36 2024 - [info] * Switching slaves in parallel..
Tue Aug  6 14:24:36 2024 - [info] 
Tue Aug  6 14:24:36 2024 - [info] -- Slave switch on host 172.17.44.44(172.17.44.44:3307) started, pid: 9165
Tue Aug  6 14:24:36 2024 - [info] 
Tue Aug  6 14:24:37 2024 - [info] Log messages from 172.17.44.44 ...
Tue Aug  6 14:24:37 2024 - [info] 
Tue Aug  6 14:24:36 2024 - [info]  Waiting to execute all relay logs on 172.17.44.44(172.17.44.44:3307)..
Tue Aug  6 14:24:36 2024 - [info]  master_pos_wait(mysql-bin-sunac_3307.000002:1261) completed on 172.17.44.44(172.17.44.44:3307). Executed 0 events.
Tue Aug  6 14:24:36 2024 - [info]   done.
Tue Aug  6 14:24:36 2024 - [debug]  Stopping SQL thread on 172.17.44.44(172.17.44.44:3307)..
Tue Aug  6 14:24:36 2024 - [debug]   done.
Tue Aug  6 14:24:36 2024 - [info]  Resetting slave 172.17.44.44(172.17.44.44:3307) and starting replication from the new master 172.17.44.143(172.17.44.143:3307)..
Tue Aug  6 14:24:36 2024 - [debug]  Stopping slave IO/SQL thread on 172.17.44.44(172.17.44.44:3307)..
Tue Aug  6 14:24:36 2024 - [debug]   done.
Tue Aug  6 14:24:36 2024 - [info]  Executed CHANGE MASTER.
Tue Aug  6 14:24:36 2024 - [debug]  Starting slave IO/SQL thread on 172.17.44.44(172.17.44.44:3307)..
Tue Aug  6 14:24:36 2024 - [debug]   done.
Tue Aug  6 14:24:36 2024 - [info]  Slave started.
Tue Aug  6 14:24:37 2024 - [info] End of log messages from 172.17.44.44 ...
Tue Aug  6 14:24:37 2024 - [info] 
Tue Aug  6 14:24:37 2024 - [info] -- Slave switch on host 172.17.44.44(172.17.44.44:3307) succeeded.
Tue Aug  6 14:24:37 2024 - [info] Unlocking all tables on the orig master:
Tue Aug  6 14:24:37 2024 - [info] Executing UNLOCK TABLES..
Tue Aug  6 14:24:37 2024 - [info]  ok.
Tue Aug  6 14:24:37 2024 - [info] All new slave servers switched successfully.
Tue Aug  6 14:24:37 2024 - [info] 
Tue Aug  6 14:24:37 2024 - [info] * Phase 5: New master cleanup phase..
Tue Aug  6 14:24:37 2024 - [info] 
Tue Aug  6 14:24:37 2024 - [debug]  Clearing slave info..
Tue Aug  6 14:24:37 2024 - [debug]  Stopping slave IO/SQL thread on 172.17.44.143(172.17.44.143:3307)..
Tue Aug  6 14:24:37 2024 - [debug]   done.
Tue Aug  6 14:24:37 2024 - [debug]  SHOW SLAVE STATUS shows new master does not replicate from anywhere. OK.
Tue Aug  6 14:24:37 2024 - [info]  172.17.44.143: Resetting slave info succeeded.
Tue Aug  6 14:24:37 2024 - [info] Switching master to 172.17.44.143(172.17.44.143:3307) completed successfully.
Tue Aug  6 14:24:37 2024 - [debug]  Disconnected from 172.17.44.45(172.17.44.45:3307)
Tue Aug  6 14:24:37 2024 - [debug]  Disconnected from 172.17.44.143(172.17.44.143:3307)
Tue Aug  6 14:24:37 2024 - [debug]  Disconnected from 172.17.44.44(172.17.44.44:3307)
[root@wtj1vpztmysql04 ~]# 
```


# 主从库均开启GTID模式，切换没问题

```js
所有节点，均使用gtid模式，进行故障转移，以及主从库切换。

```

# 两个主库开启gtid模式，1个从库使用POS复制

```js
此种情况下，在主库出现故障时，vip会切换到新的主库。使用pos位复制的从库，也同时会被开启GTID复制模式，和新的主库进行同步。
```

# 从库执行同步操作

![[Pasted image 20240806150512.png]]

```sql
mysql> change master to master_host='172.17.44.45',master_port=3307,master_user='rpl_slave',master_password='BzCFegx05x1Qdxqga_20
22',MASTER_LOG_FILE='mysql-bin-sunac_3307.000002', MASTER_LOG_POS=1261;
Query OK, 0 rows affected, 8 warnings (0.01 sec)

mysql> start slave;
Query OK, 0 rows affected, 1 warning (0.00 sec)

mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: 172.17.44.45
                  Master_User: rpl_slave
                  Master_Port: 3307
                Connect_Retry: 60
              Master_Log_File: mysql-bin-sunac_3307.000002
          Read_Master_Log_Pos: 1261
               Relay_Log_File: mysql_3307_relay_bin.000002
                Relay_Log_Pos: 337
        Relay_Master_Log_File: mysql-bin-sunac_3307.000002
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: mysql,information_schema,performance_schema,test
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1261
              Relay_Log_Space: 552
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 1045
                  Master_UUID: a4164c09-4fbb-11ef-9e31-005056989886
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: a4164c09-4fbb-11ef-9e31-005056989886:1-4,
a5aa9e7b-4fce-11ef-8641-0050569859cb:1-3
                Auto_Position: 0
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
       Master_public_key_path: 
        Get_master_public_key: 0
            Network_Namespace: 
1 row in set, 1 warning (0.01 sec)

ERROR: 
No query specified
```
# 模拟主库故障
![[Pasted image 20240806150228.png]]

# 查看从库同步状态
```sql
mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: 172.17.44.143
                  Master_User: rpl_slave
                  Master_Port: 3307
                Connect_Retry: 60
              Master_Log_File: mysql-bin-sunac_3307.000006
          Read_Master_Log_Pos: 237
               Relay_Log_File: mysql_3307_relay_bin.000002
                Relay_Log_Pos: 395
        Relay_Master_Log_File: mysql-bin-sunac_3307.000006
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: mysql,information_schema,performance_schema,test
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 237
              Relay_Log_Space: 610
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 10143
                  Master_UUID: a5aa9e7b-4fce-11ef-8641-0050569859cb
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: a4164c09-4fbb-11ef-9e31-005056989886:1-4,
a5aa9e7b-4fce-11ef-8641-0050569859cb:1-3
                Auto_Position: 1
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
       Master_public_key_path: 
        Get_master_public_key: 0
            Network_Namespace: 
1 row in set, 1 warning (0.01 sec)

ERROR: 
No query specified

mysql> 
```

# 恢复原来故障主库为新的从库

```sql

mysql> CHANGE MASTER TO MASTER_HOST='172.17.44.143', MASTER_PORT=3307, MASTER_AUTO_POSITION=1, MASTER_USER='rpl_slave', MASTER_PASSWORD='BzCFegx05x1Qdxqga_2022';
Query OK, 0 rows affected, 7 warnings (0.00 sec)

mysql> start slave;
Query OK, 0 rows affected, 1 warning (0.01 sec)

mysql> show warnings;
+---------+------+-----------------------------------------------------------------------------------------------------+
| Level   | Code | Message                                                                                             |
+---------+------+-----------------------------------------------------------------------------------------------------+
| Warning | 1287 | 'STOP SLAVE' is deprecated and will be removed in a future release. Please use STOP REPLICA instead |
+---------+------+-----------------------------------------------------------------------------------------------------+
1 row in set (0.00 sec)

-- mysql-8.0版本及以后，启动和关闭复制命令如下：

mysql> stop replica;
Query OK, 0 rows affected (0.00 sec)

mysql> start replica;
Query OK, 0 rows affected (0.00 sec)

mysql> show slave status\G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: 172.17.44.143
                  Master_User: rpl_slave
                  Master_Port: 3307
                Connect_Retry: 60
              Master_Log_File: mysql-bin-sunac_3307.000006
          Read_Master_Log_Pos: 237
               Relay_Log_File: mysql_3307_relay_bin.000002
                Relay_Log_Pos: 395
        Relay_Master_Log_File: mysql-bin-sunac_3307.000006
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: 
          Replicate_Ignore_DB: mysql,information_schema,performance_schema,test
           Replicate_Do_Table: 
       Replicate_Ignore_Table: 
      Replicate_Wild_Do_Table: 
  Replicate_Wild_Ignore_Table: 
                   Last_Errno: 0
                   Last_Error: 
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 237
              Relay_Log_Space: 610
              Until_Condition: None
               Until_Log_File: 
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File: 
           Master_SSL_CA_Path: 
              Master_SSL_Cert: 
            Master_SSL_Cipher: 
               Master_SSL_Key: 
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 10143
                  Master_UUID: a5aa9e7b-4fce-11ef-8641-0050569859cb
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind: 
      Last_IO_Error_Timestamp: 
     Last_SQL_Error_Timestamp: 
               Master_SSL_Crl: 
           Master_SSL_Crlpath: 
           Retrieved_Gtid_Set: 
            Executed_Gtid_Set: a4164c09-4fbb-11ef-9e31-005056989886:1-4,
a5aa9e7b-4fce-11ef-8641-0050569859cb:1-3
                Auto_Position: 1
         Replicate_Rewrite_DB: 
                 Channel_Name: 
           Master_TLS_Version: 
       Master_public_key_path: 
        Get_master_public_key: 0
            Network_Namespace: 
1 row in set, 1 warning (0.01 sec)

ERROR: 
No query specified
```
