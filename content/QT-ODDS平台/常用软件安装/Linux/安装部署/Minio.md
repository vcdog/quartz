

---
title: "GitLab Documentation"
description: "How to use GitLab effectively"
author: "Your Name"
date: "2023-08-09"
---


## [单机环境安装](http://114.242.246.250:8036/linux/install/minio.html#%E5%8D%95%E6%9C%BA%E7%8E%AF%E5%A2%83%E5%AE%89%E8%A3%85)

安装文件：minio/minio-20221005145827.0.0.x86_64.rpm

1.安装

执行命令：

rpm -ivh minio-20221005145827.0.0.x86_64.rpm

![img.png](http://114.242.246.250:8036/assets/linux-1-D0SutATx.png)

2.配置

1）修改/etc/systemd/system/minio.service

将用户和组修改为root：

![img_1.png](http://114.242.246.250:8036/assets/linux-2-BKaBQyU2.png)

新建文件：vim /etc/default/minio，输入并修改以下内容：

```
# 数据存储目录，根据实际数据盘目录修改
MINIO_VOLUMES="/hos/minio/data"
# 服务端口和控制台web端口
MINIO_OPTS="--address :9000 --console-address :9001"
# 账号密码
MINIO_ACCESS_KEY=admin
MINIO_SECRET_KEY=password
```

创建数据目录：

```
mkdir -p /hos/minio/data
```

3.启动

```
# 刷新配置：
systemctl daemon-reload
# 启动服务：
systemctl start minio
# 查看状态：
systemctl status minio
# 设置开机自启动：
systemctl enable minio
```

4.访问验证

访问地址：服务器IP:9001，输入上面配置的账号密码：

## [集群环境安装](http://114.242.246.250:8036/linux/install/minio.html#%E9%9B%86%E7%BE%A4%E7%8E%AF%E5%A2%83%E5%AE%89%E8%A3%85)

#### [准备](http://114.242.246.250:8036/linux/install/minio.html#%E5%87%86%E5%A4%87)

Minio官方推荐的最小集群环境为四台服务器，同时使用nginx配置负载均衡。本文使用四台服务器演示搭建。注意：集群的搭建和单点的方式不一致，搭建集群时，请确保未安装minio。

- 确保数据存储目录是一个空的挂载卷，不能是跟目录或root目录的挂载点
- 确保四台服务器的时间同步
- 需开启9001、9000端口的防火墙(默认端口，可以修改)

示例机器IP：

- 机器1：192.168.0.1
- 机器2：192.168.0.2
- 机器3：192.168.0.3
- 机器4：192.168.0.4

#### [集群配置](http://114.242.246.250:8036/linux/install/minio.html#%E9%9B%86%E7%BE%A4%E9%85%8D%E7%BD%AE)

```
# 创建安装目录和数据目录
mkdir -p /hos/minio/data
cd /hos/minio

# 将安装包中的linux/minio/minio.RELEASE.2022-10-05T14-58-27Z上传到安装目录，修改名称
mv minio.RELEASE.2022-10-05T14-58-27Z minio

# 授权
chmod +x minio

然后编辑如下脚本/hos/minio/run.sh：
#!/bin/bash
export MINIO_ROOT_USER=admin
export MINIO_ROOT_PASSWORD=password

/hos/minio/minio server --address ":9000" --console-address ":9001" http://192.168.0.1/hos/minio/data http://192.168.0.2/hos/minio/data http://192.168.0.3/hos/minio/data http://192.168.0.4/hos/minio/data

其中：
“MINIO_ROOT_USER”为用户名
“MINIO_ROOT_PASSWORD”为密码，密码不能设置过于简单，不然minio会启动失败
http://192.168.0.1/hos/minio/data ip地址和具体目录修改为实际目录
*注意里面的#!/bin/bash，不加的话后续使用服务启动会直接结束，会出现main process exited, code=exited, status=203/EXEC

授权：
chmod +x /hos/minio/run.sh
```

#### [注册开机自启](http://114.242.246.250:8036/linux/install/minio.html#%E6%B3%A8%E5%86%8C%E5%BC%80%E6%9C%BA%E8%87%AA%E5%90%AF)

编写启动脚本，创建服务文件：

```
vim /etc/systemd/system/minio.service

内容：
[Unit]
Description=Minio service
Documentation=https://docs.minio.io/

[Service]
WorkingDirectory=/hos/minio/
ExecStart=/hos/minio/run.sh

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

其中：
“WorkingDirectory”为minio服务目录
“ExecStart”为指定启动脚本目录

启动服务，设置开机自启。注意：以下命令需以此在每台服务器执行，得保证4台机器都处于启动状态，并通信正常。
# 重新刷新
systemctl daemon-reload
# 启动
systemctl start minio
# 配置开机自启
systemctl enable minio
# 查看状态
systemctl status minio
#重新启动命令
systemctl restart minio

#如果启动失败可以手动执行/hos/minio/run.sh查看日志。
```

#### [验证](http://114.242.246.250:8036/linux/install/minio.html#%E9%AA%8C%E8%AF%81)

四台机器全部正常启动，在浏览器输入http://192.168.0.xx:9001(任意一台的IP)即可访问。