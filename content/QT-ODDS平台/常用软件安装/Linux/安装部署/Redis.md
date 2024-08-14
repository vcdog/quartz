

---
title: "GitLab Documentation"
description: "How to use GitLab effectively"
author: "Your Name"
date: "2023-08-09"
---


- 门户系统和统一用户认证系统目前**支持使用同一套redis服务**，所以我们只需要部署1套redis即可。
- 其他产品则根据实际情况选择部署方式、数量。

## [单机环境安装](http://114.242.246.250:8036/linux/install/redis.html#%E5%8D%95%E6%9C%BA%E7%8E%AF%E5%A2%83%E5%AE%89%E8%A3%85)

1.安装

进入安装目录的redis目录，执行命令安装：

```
rpm -ivh redis/redis-6.0.16-1.el7.remi.x86_64.rpm
```

2.配置:编辑配置文件：/etc/redis.conf，找到以下配置并根据实际需求修改：

```
# 远程连接配置
bind 0.0.0.0
# redis端口，根据实际修改
port 6379
# redis 访问密码
requirepass passwrod
# 关闭保护模式
protected-mode no
# 开启 appendonly 备份模式
appendonly yes
daemonize yes
# 指定数据目录，根据实际服务器环境指定
dir /hos/redis/data
```

3.创建数据目录并授权

```
#创建数据目录，和上一步骤指定的数据目录一致
mkdir -p /hos/redis/data
#授权
chown -R redis:redis /hos/redis/data
```

4.启动

- 启动：systemctl start redis
- 开机自启动：systemctl enable redis
- 停止：systemctl stop redis
- 重启：systemctl restart redis

5.验证 客户端连接命令：redis-cli -a password,其中password为上面设置的密码。

## [集群环境安装](http://114.242.246.250:8036/linux/install/redis.html#%E9%9B%86%E7%BE%A4%E7%8E%AF%E5%A2%83%E5%AE%89%E8%A3%85)

- 集群需要6个redis服务；
- 默认至少3台服务器，如果有更多服务器则根据实际数量调整服务分配；
- 每台服务器安装运行2个redis服务。

#### [安装Redis](http://114.242.246.250:8036/linux/install/redis.html#%E5%AE%89%E8%A3%85redis)

在3台服务器进行以下操作：

```
#1.安装：将/linux/redis/中的安装文件上传到服务器，其中：
#redis-6.0.16-1.el7.remi.x86_64.rpm适用于centos/redhat7.x系统
#redis-6.0.18-1.el8.remi.x86_64.rpm适用于centos/redhat8.x系统
#执行命令安装：
rpm -ivh 文件名

#2.修改配置文件：/linux/redis/conf中有2个配置文件，根据实际修改其中的端口/密码等信息：
#然后将其上传到/etc/目录下

#3.创建数据目录
mkdir /hos/redis/data-1
mkdir /hos/redis/data-2

#4.配置服务：/linux/redis/service中有2个服务文件，用于启动redis。
#将其拷贝到/etc/systemd/system/下

#5.启动
#加载服务：
systemctl daemon-reload

#启动：
systemctl start redis-1
systemctl start redis-2

#开机自启动：
systemctl enable redis-1
systemctl enable redis-2

#停止命令：systemctl stop redis-x
#重启命令：systemctl restart redis-x
```

#### [配置集群](http://114.242.246.250:8036/linux/install/redis.html#%E9%85%8D%E7%BD%AE%E9%9B%86%E7%BE%A4)

redis集群需要三主三从6个服务，所以从3台服务器中各使用2个redis服务来组成一套集群。 在任意一台服务器执行：

```
#建立集群：
redis-cli --cluster create ip1:8021 ip1:8022 ip2:8021 ip2:8022 ip3:8021 ip3:8022 --cluster-replicas 1 -a password

#其中：ip分别为各服务器ip地址，端口修改为实际端口，password为配置文件中设置的主从同步密码
#执行后会自动配置集群，输入yes完成集群创建。
```