

---
title: "GitLab Documentation"
description: "How to use GitLab effectively"
author: "Your Name"
date: "2023-08-09"
---


## [单机环境安装](http://114.242.246.250:8036/linux/install/nginx.html#%E5%8D%95%E6%9C%BA%E7%8E%AF%E5%A2%83%E5%AE%89%E8%A3%85)

进入安装包中的nginx目录，执行：rpm -ivh nginx-1.20.2-1.el7.ngx.x86_64.rpm

该rpm包为centos/rhel7使用，其他系统的[下载地址open in new window](http://nginx.org/packages/)。

- 启动服务：systemctl start nginx
- 查看状态：systemctl status nginx
- 配置自启动：systemctl enable nginx

访问验证：浏览器输入服务器ip地址，显示nginx信息表示安装完成。

## [集群环境安装](http://114.242.246.250:8036/linux/install/nginx.html#%E9%9B%86%E7%BE%A4%E7%8E%AF%E5%A2%83%E5%AE%89%E8%A3%85)

使用vip+nginx+keepalived方式搭建负载均衡高可用环境，准备2台服务器。

#### [安装nginx](http://114.242.246.250:8036/linux/install/nginx.html#%E5%AE%89%E8%A3%85nginx)

两台服务器分别安装nginx：

进入安装包中的nginx目录，执行：

rpm -ivh nginx-1.20.2-1.el7.ngx.x86_64.rpm

- 启动服务：systemctl start nginx
- 查看状态：systemctl status nginx
- 配置自启动：systemctl enable nginx

访问验证：浏览器输入服务器ip地址，出现nginx提示表示安装完成：

#### [安装配置keepalived](http://114.242.246.250:8036/linux/install/nginx.html#%E5%AE%89%E8%A3%85%E9%85%8D%E7%BD%AEkeepalived)

主服务器配置：

```
1.安装keepalived
#进入安装包hos-base的linux/keepalived目录，执行：
rpm -ivh *.rpm --nodeps

2.修改配置文件
mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
vi /etc/keepalived/keepalived.conf，填入以下内容：

global_defs {
   router_id hos-1
}

vrrp_instance VI_1 {
    state MASTER
        # 该实例绑定的网卡名称，根据实际修改
    interface ens33
    virtual_router_id 21
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
        # 修改为医院提供的虚拟vip
    virtual_ipaddress {
        192.168.100.149
    }
}

3.启动、开机自启keepalived服务
systemctl start keepalived
systemctl enable keepalived

4.查看vip服务状态和vip
systemctl status keepalived
ip addr
```

从服务器配置： 1：安装keepalived，参考主服务器

2：修改配置文件

```
mv /etc/keepalived/keepalived.conf /etc/keepalived/keepalived.conf.bak
vi /etc/keepalived/keepalived.conf，填入以下内容：

global_defs {
   router_id hos-2
}

vrrp_instance VI_1 {
    state BACKUP
        # 该实例绑定的网卡名称，根据实际修改
    interface ens33
        # 和主节点一致
    virtual_router_id 21
    priority 90
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
        # 修改为医院提供的虚拟vip，和主节点一致
    virtual_ipaddress {
        192.168.100.149
    }
}
```

3：启动、开机自启keepalived服务,参考主服务器

4：查看keepalived服务状态,参考主服务器