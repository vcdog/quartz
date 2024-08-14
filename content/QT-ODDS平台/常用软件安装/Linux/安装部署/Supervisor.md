

---
title: "GitLab Documentation"
description: "How to use GitLab effectively"
author: "Your Name"
date: "2023-08-09"
---


Supervisor (http://supervisord.org) 是一个用 Python 写的进程管理工具，可以很方便的用来启动、重启、关闭进程。这里我们用来管理应用后端服务的启动、停止等。

## [安装并启动](http://114.242.246.250:8036/linux/install/supervisor.html#%E5%AE%89%E8%A3%85%E5%B9%B6%E5%90%AF%E5%8A%A8)

**注意：需要根据不同操作系统选择对应的安装流程。**

如果是`Redhat/CentOS`系统，参考如下流程：

```
#1.进入安装包的supervisor目录
cd supervisor

#2.可以看到两个rpm包，安装supervisor：
rpm -ivh *.rpm

#3.设置supervisor 开机自动启动：
systemctl enable supervisord

#4.启动supervisor 服务、查看supervisor 服务状态、查看是否存在supervisor 进程：
systemctl start supervisord
systemctl status supervisord
```

如果是`统信UOS`或`麒麟`系统，参考如下流程：

```
#1.将安装包的supervisor/uos/supervisor-uos.tgz上传到服务器（注意上传前不能解压，否则会出现权限问题）

#2.解压安装包supervisor-uos.tgz，进入解压后的目录
tar zxf supervisor-uos.tgz
cd supervisor-uos

#3.查看python版本
ll /usr/bin/python3*
#统信系统默认安装python3.7版本。安装包使用的也是3.7版本。
#如果是其他版本，则需要修改以下四个文件的第一行的python版本信息：
#echo_supervisord_conf、pidproxy、 supervisorctl、 supervisord

#4.拷贝文件
cp echo_supervisord_conf pidproxy supervisorctl supervisord /usr/bin/
cp -rp supervisor supervisor-4.2.2-py3.6.egg-info /usr/lib/python3.7/site-packages/
cp supervisord.conf /etc/
cp supervisord.service /usr/lib/systemd/system/

#5.加载并启动服务
systemctl daemon-reload
#设置supervisor 开机自动启动：
systemctl enable supervisord
#启动supervisor 服务：
systemctl start supervisord
#查看supervisor 服务状态
systemctl status supervisord

#启动服务时如果报错类似：supervisord[36131]: Error: The directory named as part of the path /var/log/superviso
#需要手动创建supervisor的日志目录
mkdir /var/log/supervisor
systemctl start supervisord
```

## [启用Web控制台](http://114.242.246.250:8036/linux/install/supervisor.html#%E5%90%AF%E7%94%A8web%E6%8E%A7%E5%88%B6%E5%8F%B0)

可选，调试时可以开启，**生产环境建议跳过本步骤**

启用Web控制台后，可以在浏览器中对服务进行启动/关闭等管理操作，也可以查看日志信息。具体开启方式如下：

- 修改配置文件,vim /etc/supervisord.conf,修改控制台web页面的端口、账号、密码信息：
    
    ![img_2.png](http://114.242.246.250:8036/assets/linux-3-BnN6DwjJ.png)
    
- 加载配置：supervisorctl reload
    
- 访问验证：使用服务器IP:9001，访问web页面。
    

## [配置项说明](http://114.242.246.250:8036/linux/install/supervisor.html#%E9%85%8D%E7%BD%AE%E9%A1%B9%E8%AF%B4%E6%98%8E)

/etc/supervisord.conf配置文件中部分配置项说明：

```
[unix_http_server]          ; 如果配置文件没有[unix_http_server]部分，则不会启动UNIX域套接字HTTP服务器
file=/tmp/supervisor.sock   ; 一个指向UNIX域套接字的路径，supervisor将在该套接字上侦听HTTP/XML-RPC请求
;chmod=0700                 ; 在启动时将UNIX域套接字的UNIX权限模式位更改为此值 (默认 0700)
;chown=nobody:nogroup       ; 将套接字文件的用户和组更改为此值，可以是UNIX用户名（例如chrism）或由冒号分隔的UNIX用户名和组（例如chrism:wheel）（默认 启动用户名和组）
;username=user              ; 对此 HTTP 服务器进行身份验证所需的用户名 
;password=123               ; 验证此 HTTP 服务器所需的密码 
 
;[inet_http_server]         ; 默认情况下不启用 inet HTTP 服务器。 inet HTTP服务器是通过取消注释[inet_http_server]部分启用 
;port=127.0.0.1:9001        ; 侦听的TCP主机端口值，supervisor将在其上侦听HTTP/XML-RPC请求
;username=user              ; 对此 HTTP 服务器进行身份验证所需的用户名 
;password=123               ; 对此 HTTP 服务器进行身份验证所需的密码 
 
[supervisord]
logfile=/tmp/supervisord.log ; supervisord进程的活动日志的路径 (默认 $CWD/supervisord.log)
logfile_maxbytes=500MB       ; 活动日志文件在轮换之前可能消耗的最大字节数，将此值设置为0以指示无限的日志大小（默认 50MB） 
logfile_backups=20           ; 由于活动日志文件轮换而保留的备份数量，如果设置为 0，则不会保留任何备份（默认 10）
loglevel=info                ; 日志级别（默认 info）其他级别包括: debug,warn,trace
pidfile=/tmp/supervisord.pid ; supervisord保存其pid文件的位置 
nodaemon=false               ; 如果为true，supervisord将在前台启动而不是守护进程 
silent=false                 ; 如果为true且未守护，日志将不会被定向到标准输出stdout 
minfds=65535                 ; 在supervisord成功启动之前必须可用的最小文件描述符数（默认 1024）
minprocs=1000                ; 在supervisord成功启动之前必须可用的最小进程描述符数（默认 200） 
;umask=022                   ; supervisord进程的umask（默认 022） 
;user=supervisord            ; 指示supervisord将用户切换到此UNIX用户帐户启动，只有以root用户启动supervisord才能切换用户 
;identifier=supervisor       ; 此主管进程的标识符字符串，由 RPC 接口使用（默认 'supervisor'）
;directory=/tmp              ; 当supervisord守护进程时，切换到这个目录 
;nocleanup=true              ; 防止supervisord在启动时清除任何现有的AUTO子日志文件，通常用于调试（默认 false） 
;childlogdir=/tmp            ; 用于AUTO子日志文件的目录（默认 Python的tempfile.gettempdir()的值） 
;environment=KEY="value"     ; KEY="val",KEY2="val2"形式的键/值对列表，将放置在所有子进程的环境中，这不会改变supervisord本身的环境 
;strip_ansi=false            ; 从子日志文件中去除所有 ANSI 转义序列（默认 false） 
 
[supervisorctl]
serverurl=unix:///tmp/supervisor.sock ; 用于访问supervisord服务器的URL，例如http://localhost:9001，对于 UNIX 域套接字，请使用unix:///absolute/path/to/file.sock（默认 http://localhost:9001） 
;username=chris              ; 传递给supervisord服务器以用于身份验证的用户名 
;password=123                ; 传递给supervisord服务器以用于身份验证的密码 
;prompt=mysupervisor         ; 用作supervisorctl提示的字符串 (默认 "supervisor")
;history_file=~/.sc_history  ; 用作readline持久历史文件的路径（默认 无） 
 
;[program:theprogramname]      ; 配置文件必须包含一个或多个program段，以便supervisord知道它应该启动和控制哪些程序
;command=/bin/cat              ; 该程序启动时将运行的命令（命令可以是绝对的或相对的） 
;process_name=%(program_name)s ; 用于组成此进程的主管进程名称 (默认 %(program_name)s)
;numprocs=1                    ; Supervisor将启动与numprocs命名的该程序的多个实例（默认1），当此配置大于1时，process_name表达式中必须包含%(process_num)s 
;directory=/tmp                ; 一个文件路径，代表一个目录， 在执行子进程之前，supervisord应该临时 chdir到该目录
;umask=022                     ; 表示进程的 umask 的八进制数（例如 002、022）（默认 none） 
;priority=999                  ; 程序在启动和关闭顺序中的相对优先级，较低的优先级表示程序在启动时以及在各种客户端中使用聚合命令（例如“全部启动”/“全部停止”）时首先启动并最后关闭（默认999）
;autostart=true                ; 如果为 true，则该程序将在启动supervisord时自动启动（默认 true） 
;startsecs=1                   ; 启动后程序需要保持运行以认为启动成功（将进程从STARTING状态移动到RUNNING状态）的总秒数（默认 1）
;startretries=3                ; 在放弃进程并将进程置于FATAL状态之前尝试启动程序时，supervisord将允许的串行失败尝试次数（默认 3） 
;autorestart=unexpected        ; 如果一个进程在处于RUNNING状态时退出，supervisord是否应该自动重新启动它(默认 unexpected)
;exitcodes=0                   ; 与autorestart一起使用的此程序的“预期”退出代码列表（默认 0）
;stopsignal=TERM               ; 请求停止时用于终止程序的信号。这可以是 TERM、HUP、INT、QUIT、KILL、USR1 或 USR2 中的任何一个（默认 TERM） 
;stopwaitsecs=10               ; 在程序发送停止信号后，等待操作系统将 SIGCHLD 返回给supervisord的秒数，超过将尝试用SIGKILL杀死它（默认 10） 
;stopasgroup=false             ; 如果为true，该标志会导致supervisor向整个进程组发送停止信号并暗示killasgroup（默认 false） 
;killasgroup=false             ; 如果为true，则当诉诸于向程序发送SIGKILL以终止它时，将其发送到其整个进程组，同时照顾其子进程，例如对于使用multiprocessing的Python 程序很有用 
;user=chrism                   ; 指示supervisord使用此 UNIX 用户帐户作为运行程序的帐户。如果supervisord以 root 用户身份运行，则只能切换 用户。如果supervisord 不能切换到指定的用户，程序将不会启动 
;redirect_stderr=true          ; 如果为 true，则将进程的 stderr 输出发送回 其 stdout 文件描述符上的supervisord（在 UNIX shell 术语中，这相当于执行/the/program 2>&1）（默认 false） 
;stdout_logfile=/a/path        ; 将进程 stdout 输出放在此文件中（如果 redirect_stderr 为真，也将 stderr 输出放在此文件中）（默认 AUTO） 
;stdout_logfile_maxbytes=200MB ; stdout_logfile在轮换之前可能消耗的最大字节数，将此值设置为 0 以指示无限的日志大小（默认 50MB） 
;stdout_logfile_backups=10     ; stdout_logfile备份保持周围从工艺标准输出日志文件旋转而产生的数量，如果设置为 0，则不会保留任何备份 （默认 10）
;stdout_capture_maxbytes=0     ; 当进程处于“stdout 捕获模式”时写入捕获 FIFO 的最大字节数（默认 0） 
;stdout_events_enabled=false   ; 如果为 true，则在进程写入其 stdout 文件描述符时将发出 PROCESS_LOG_STDOUT 事件。仅当文件描述符在接收数据时未处于捕获模式时才会发出事件（默认 0） 
;stdout_syslog=false           ; 如果为 true，stdout 将与进程名称一起定向到 syslog 
;stderr_logfile=/a/path        ; 除非redirect_stderr为真，否则将进程stderr 输出放在此文件中（默认 AUTO）
;stderr_logfile_maxbytes=1MB   ; stderr_logfile 的日志文件轮换之前的最大字节数 （默认 50M）
;stderr_logfile_backups=10     ; 由进程 stderr 日志文件轮换产生的要保留的备份数（默认 10） 
;stderr_capture_maxbytes=1MB   ; 当进程处于“stderr 捕获模式”时写入捕获 FIFO 的最大字节数（默认 0） 
;stderr_events_enabled=false   ; 如果为 true，则在进程写入其 stderr 文件描述符时将发出 PROCESS_LOG_STDERR 事件。仅当文件描述符在接收数据时未处于捕获模式时才会发出事件（默认 false）
;stderr_syslog=false           ; 如果为 true，stderr 将与进程名称一起定向到 syslog
;environment=A="1",B="2"       ; KEY="val",KEY2="val2"形式的键/值对列表，将放置在子进程的环境中
;serverurl=AUTO                ; 在环境中传递给子进程进程的 URL 作为SUPERVISOR_SERVER_URL（请参阅supervisor.childutils）以允许子进程 轻松与内部 HTTP 服务器通信
```