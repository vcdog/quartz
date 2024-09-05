
# 在leader节点上以postgres身份执行

```bash
su - postgres

# 添加虚拟网卡ens192:2

$ [postgres@wtj1vpk8sql01 ~]$ /etc/patroni/patroni_callback.sh on_start Leader pg-sentry-cluster01

[postgres@wtj1vpk8sql01 ~]$ ip ad sh|grep 172.17.
    inet 172.17.44.155/22 brd 172.17.47.255 scope global noprefixroute ens192
    inet 172.17.44.159/21 brd 172.17.47.255 scope global ens192:2

```

# 发生故障切换后，先在原来