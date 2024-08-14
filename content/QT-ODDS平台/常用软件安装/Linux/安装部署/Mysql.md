

---
title: "GitLab Documentation"
description: "How to use GitLab effectively"
author: "Your Name"
date: "2023-08-09"
---


## [单机环境安装](http://114.242.246.250:8036/linux/install/mysql.html#%E5%8D%95%E6%9C%BA%E7%8E%AF%E5%A2%83%E5%AE%89%E8%A3%85)

如果已经有Mysql8.0的环境可以直接使用，或者可以参考以下方式全新部署Mysql8.0服务。

1.卸载MariaDB：在CentOS中默认安装有MariaDB，是MySQL的一个分支，主要由开源社区维护。CentOS 7及以上版本已经不再使用MySQL数据库，而是使用MariaDB数据库。如果直接安装MySQL，会和MariaDB的文件冲突。因此，需要先卸载自带的MariaDB，再安装MySQL。

```
#查看版本：
rpm -qa|grep mariadb
#卸载：
rpm -e --nodeps 文件名
#检查是否卸载干净：
rpm -qa|grep mariadb
```

2.安装MySQL

```
#解压
tar xf mysql-8.0.X-1.el7.x86_64.rpm-bundle.tar
#安装，依次执行
rpm -ivh mysql-community-common-8.0.X-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-plugins-8.0.X-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-8.0.X-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-8.0.X-1.el7.x86_64.rpm
rpm -ivh mysql-community-icu-data-files-8.0.X-1.el7.x86_64.rpm
rpm -ivh mysql-community-server-8.0.X-1.el7.x86_64.rpm --nodeps
```

3.创建数据目录和授权

```
#根据实际系统，在数据盘中创建数据目录：
mkdir -p /hos/mysql/data

# 更改属主和数组
chown -R mysql:mysql /hos/mysql/data
```

4.编辑配置文件：/etc/my.cnf：

```
[mysqld]
port       = 3306
server-id  = 1
user       = mysql

binlog_format=mixed
expire_logs_days=30

datadir=/hos/mysql/data
socket=/hos/mysql/data/mysql.sock
log-bin=/hos/mysql/data/mysql-bin
log-error=/hos/mysql/data/mysql.log
pid-file=/hos/mysql/data/mysql.pid

character-set-server=utf8mb4
collation-server=utf8mb4_0900_ai_ci
default-authentication-plugin=mysql_native_password
explicit_defaults_for_timestamp=on
max_connections=1000
lower_case_table_names=1

[client]
port=3306
socket=/hos/mysql/data/mysql.sock
```

5.初始化数据库

```
mysqld --initialize
```

5.启动并配置MySQL

```
systemctl start mysqld
systemctl enable mysqld

#查看是否启动:
ps -ef|grep mysql

#登录验证：初始的随机密码在/hos/mysql/data/mysql.log中：
mysql -u root -p
password:输入密码

#修改root密码，将123456修改为复杂密码:
ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';

#设置允许远程登录：
mysql> use mysql
mysql> update user set user.Host='%' where user.User='root';
mysql> flush privileges;
mysql> quit
```

## [集群环境安装](http://114.242.246.250:8036/linux/install/mysql.html#%E9%9B%86%E7%BE%A4%E7%8E%AF%E5%A2%83%E5%AE%89%E8%A3%85)

Mysql高可用安装，使用mysql+keepalived方式，需要2台服务器以及1个VIP。

#### [安装Mysql](http://114.242.246.250:8036/linux/install/mysql.html#%E5%AE%89%E8%A3%85mysql)

在两台服务器上分别安装：

1.卸载MariaDB

在CentOS中默认安装有MariaDB，是MySQL的一个分支，主要由开源社区维护。CentOS 7及以上版本已经不再使用MySQL数据库，而是使用MariaDB数据库。如果直接安装MySQL，会和MariaDB的文件冲突。因此，需要先卸载自带的MariaDB，再安装MySQL。

```
#查看版本：
rpm -qa|grep mariadb
#卸载：
rpm -e --nodeps 上一步骤查到的名称
#检查是否卸载干净：
rpm -qa|grep mariadb
```

2.安装MySQL

```
#解压
tar xf mysql-8.0.30-1.el7.x86_64.rpm-bundle.tar
#安装，依次执行
rpm -ivh mysql-community-common-8.0.30-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-plugins-8.0.30-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-8.0.30-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-8.0.30-1.el7.x86_64.rpm
rpm -ivh mysql-community-icu-data-files-8.0.30-1.el7.x86_64.rpm
rpm -ivh mysql-community-server-8.0.30-1.el7.x86_64.rpm --nodeps
```

3.创建数据目录和授权

```
#根据实际系统，在数据盘中创建数据目录：
mkdir -p /hos/mysql/data

# 更改属主和数组
chown -R mysql:mysql /hos/mysql/data
```

4.编辑配置文件：/etc/my.cnf，两台服务器的配置分别为： 服务器1：

```
[mysqld]
port       = 3306
server-id  = 1
user       = mysql

binlog_format=mixed
auto_increment_increment=2
auto_increment_offset=1
expire_logs_days=30

datadir=/hos/mysql/data
socket=/hos/mysql/data/mysql.sock
log-bin=/hos/mysql/data/mysql-bin
log-error=/hos/mysql/data/mysql.log
pid-file=/hos/mysql/data/mysql.pid

character-set-server=utf8mb4
collation-server=utf8mb4_0900_ai_ci
default-authentication-plugin=mysql_native_password
explicit_defaults_for_timestamp=on
max_connections=1000
lower_case_table_names=1

[client]
socket=/hos/mysql/data/mysql.sock
```

服务器2：

```
[mysqld]
port       = 3306
server-id  = 2
user       = mysql

binlog_format=mixed
auto_increment_increment=2
auto_increment_offset=2
expire_logs_days=30

datadir=/hos/mysql/data
socket=/hos/mysql/data/mysql.sock
log-bin=/hos/mysql/data/mysql-bin
log-error=/hos/mysql/data/mysql.log
pid-file=/hos/mysql/data/mysql.pid

character-set-server=utf8mb4
collation-server=utf8mb4_0900_ai_ci
default-authentication-plugin=mysql_native_password
explicit_defaults_for_timestamp=on
max_connections=1000
lower_case_table_names=1

[client]
socket=/hos/mysql/data/mysql.sock
```

初始化：mysqld --initialize

5.启动MySQL（两台服务器都要执行）

```
systemctl start mysqld
systemctl enable mysqld

#查看是否启动:
ps -ef|grep mysql

#登录验证：初始的随机密码在/hos/mysql/data/mysql.log中：
mysql -u root -p
password:输入密码

#修改密码(2个mysql密码需要一致，修改为复杂密码):
ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';

#设置允许远程登录：登录到mysql里执行：
mysql> use mysql
mysql> update user set user.Host='%' where user.User='root';
mysql> flush privileges;
mysql> quit
```

#### [Mysql配置](http://114.242.246.250:8036/linux/install/mysql.html#mysql%E9%85%8D%E7%BD%AE)

1.确认mysql服务都已启动，分别使用root账号在两台服务器上登录各自的mysql：

```
#hosta
mysql -u root -p

#hostb
mysql -u root -p
```

2.创建复制用户，2个数据库上都要创建

```
create user 'repl'@'%' identified by 'repl';
grant replication slave on *.* to 'repl'@'%';
flush privileges;
```

3.在hosta上查看binlog信息：

```
show master status;
```

4.hostb上开启复制，以下命令在hostb上执行：

```
-- 配置复制
mysql> change master to master_host='hosta的ip',master_port=3306, master_user='repl',master_password='repl',master_log_file='上面步骤查到的file名称', master_log_pos=上面步骤查到的position值;

-- 开启复制
mysql> start slave;

-- 查看复制状态，其中Slave_IO_Running和Slave_SQL_Running都是Yes代表完成配置
mysql> show slave status \G
 *************************** 1. row ***************************
                Slave_IO_State: Waiting for master to send event
                   Master_Host: 192.168.10.11
                   Master_User: rep
                   Master_Port: 3306
                 Connect_Retry: 60
               Master_Log_File: master-bin.000001
           Read_Master_Log_Pos: 322
                Relay_Log_File: hostb-relay-bin.000002
                 Relay_Log_Pos: 417
         Relay_Master_Log_File: master-bin.000001
              Slave_IO_Running: Yes

             Slave_SQL_Running: Yes
```

5.在hostb上查看binlog信息：

```
show master status;
```

6.hosta上开启复制，以下命令在hosta上执行：

```
-- 配置复制
mysql> change master to master_host='hostb的ip',master_port=3306, master_user='repl',master_password='repl',master_log_file='上面步骤查到的file名称', master_log_pos=上面步骤查到的position值;

-- 开启复制
mysql> start slave;

-- 查看复制状态，其中Slave_IO_Running和Slave_SQL_Running都是Yes代表完成配置
mysql> show slave status \G;
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.10.12
                  Master_User: rep
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: master-bin.000001
          Read_Master_Log_Pos: 154
               Relay_Log_File: hosta-relay-bin.000002
                Relay_Log_Pos: 369
        Relay_Master_Log_File: master-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
```

6.测试双主复制：在hosta上创建数据库test1，到hostb服务器上查看数据库是否已经创建；然后在hostb上创建数据库test2，到hosta服务器上查看数据库是否已经创建。

#### [安装配置keepalived](http://114.242.246.250:8036/linux/install/mysql.html#%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AEkeepalived)

1.keepalived的安装与管理，两台服务器都需要安装。

```
#进入安装包hos-base的linux/keepalived目录，执行：
rpm -ivh *.rpm --nodeps --force

使用如下命令管理keepalived：
# 开启keepalived
systemctl start keepalived

# 配置自动启动
systemctl enable keepalived

#其他操作
# 查看keepalived运行状态
systemctl status keepalived
# 关闭keepalived
systemctl stop keepalived
# 重新启动keepalived
systemctl restart keepalived
```

2.keepalived的配置

keepalived的配置文件为：/etc/keepalived/keepalived.conf，配置文件如内容下，使用申请的虚拟VIP配置：

【hosta主机的配置文件】

```
[root@hosta keepalived]# cat keepalived.conf
global_defs {
    router_id db-1
}

vrrp_instance VI_1 {
    state MASTER
    interface ens34               #修改为服务器网卡名称
    virtual_router_id 11
    priority 100
    advert_int 1
    authentication {   
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {    
        192.168.10.10             #修改为医院提供的虚拟VIP
    }
}
```

【hostb主机的配置文件】

```
[root@hostb keepalived]# cat keepalived.conf
global_defs {
    router_id db-2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens34               #修改为服务器网卡名称
    virtual_router_id 11
    priority 90
    advert_int 1
    authentication {   
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {    
        192.168.10.10             #修改为医院提供的虚拟VIP
    }
}
```

```
#重启启动
systemctl restart keepalived

#在hosta上查看vip是否配置到网卡
ip addr
```