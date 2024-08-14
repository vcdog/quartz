

---
title: "GitLab Documentation"
description: "How to use GitLab effectively"
author: "Your Name"
date: "2023-08-09"
---


## [文件句柄数量配置](http://114.242.246.250:8036/linux/kb/linux.html#%E6%96%87%E4%BB%B6%E5%8F%A5%E6%9F%84%E6%95%B0%E9%87%8F%E9%85%8D%E7%BD%AE)

常见问题：打开的文件过多 Too many open files

Linux中所有的事物或资源都是以文件的形式存在，比如消息、共享内存、连接等，句柄可以理解为指向这些文件的指针。对于这些句柄，Linux是有数量限制的，单个进程默认可以打开的句柄数上限，可以用以下命令来查看：

```
#一般默认是1024
ulimit -n
```

修改最大文件数量：

```
#系统级别的限制，默认数值较大，一般不需要修改
cat /proc/sys/fs/file-max

#进程级别的限制，修改/etc/security/limits.conf文件
vi /etc/security/limits.conf
#在文件末尾增加以下内容，其中65535根据实际调整，保存并重新登录系统或重启程序
* soft nofile 65535
* hard nofile 65535
```

**相关说明：** linux对能够打开的文件句柄的数量做了限制。限制是分为三个级别：

- fs.file-max （系统级别参数）：该参数描述了整个系统可以打开的最大文件数量。但是root用户不会受该参数限制（比如：现在整个系统打开的文件描述符数量已达到fs.file-max ，此时root用户仍然可以使用ps、kill等命令或打开其他文件描述符）。
- soft nofile（用户级别参数）：限制单个进程上可以打开的最大文件数。只能在Linux上配置一次，不能针对不同用户配置不同的值。
- fs.nr_open（进程级别参数）：限制单个进程上可以打开的最大文件数。可以针对不同用户配置不同的值。

这三个参数之间还有耦合关系，所以配置值的时候还需要注意以下三点：

- 如果想加大soft nofile，那么hard nofile参数值也需要一起调整。如果因为hard nofile参数值设置的低，那么soft nofile参数的值设置的再高也没有用，实际生效的值会按照二者最低的来。
- 如果增大了hard nofile，那么fs.nr_open也都需要跟着一起调整（fs.nr_open参数值一定要大于hard nofile参数值）。如果不小心把hard nofile的值设置的比fs.nr_open还大，那么后果比较严重。会导致该用户无法登录，如果设置的是*，那么所有用户都无法登录。
- 如果加大了fs.nr_open，但是是用的echo "xxx" > ../fs/nr_open命令来修改的fs.nr_open的值，那么刚改完可能不会有问题，但是只要机器一重启，那么之前通过echo命令设置的fs.nr_open值便会失效，用户还是无法登录。所以非常不建议使用echo的方式修改内核参数！！！

## [系统时间配置](http://114.242.246.250:8036/linux/kb/linux.html#%E7%B3%BB%E7%BB%9F%E6%97%B6%E9%97%B4%E9%85%8D%E7%BD%AE)

### [时区设置](http://114.242.246.250:8036/linux/kb/linux.html#%E6%97%B6%E5%8C%BA%E8%AE%BE%E7%BD%AE)

```
# 查看时区信息
timedatectl

# 查看支持的全部时区列表
timedatectl list-timezones  

# 设置时区为亚洲/上海，即中国标准时间
timedatectl set-timezone Asia/Shanghai
```

![img.png](http://114.242.246.250:8036/assets/linux-timezone-B_oYkGid.png)

### [时间同步设置](http://114.242.246.250:8036/linux/kb/linux.html#%E6%97%B6%E9%97%B4%E5%90%8C%E6%AD%A5%E8%AE%BE%E7%BD%AE)

Chrony是一个开源的自由软件，在CentOS/RHEL 7或更高版本操作系统，已经是默认服务，默认配置文件在 /etc/chrony.conf 它能保持系统时间与时间服务器（NTP）同步，让时间始终保持同步。相对NTP时间同步软件，速度更快、配置和依赖都更简单。

1.查看chrony是否已经安装：

```
rpm -qa | grep chrony
```

2.如果没有安装：

```
# 在线安装
yum install chrony
# 离线安装，根据对应系统下载安装包：https://www.rpmfind.net/linux/rpm2html/search.php?query=chrony
rpm -ivh chrony-***.rpm
```

3.配置chrony，编辑/etc/chrony.conf，注释掉第一部分的所有server，增加新的server，其他配置不需要修改：

```
#server 0.centos.pool.ntp.org iburst
#server 1.centos.pool.ntp.org iburst
#server 2.centos.pool.ntp.org iburst
#server 3.centos.pool.ntp.org iburst
server 192.168.1.1 iburst
```

4.启动或重启chrony：

```
systemctl start chronyd
systemctl restart chronyd
```

5.配置自启动：

```
systemctl enable chronyd
```

### [手动修改时间](http://114.242.246.250:8036/linux/kb/linux.html#%E6%89%8B%E5%8A%A8%E4%BF%AE%E6%94%B9%E6%97%B6%E9%97%B4)

相关信息

手动修改时间前，需要先关闭时间同步服务：`systemctl stop chronyd`

修改当前时间：

```
timedatectl set-time 15:57:24
```

修改当前日期：

```
timedatectl set-time 2023-12-12
```