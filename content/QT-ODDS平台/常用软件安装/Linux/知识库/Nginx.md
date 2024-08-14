

---
title: "GitLab Documentation"
description: "How to use GitLab effectively"
author: "Your Name"
date: "2023-08-09"
---


## [支持WebSocket](http://114.242.246.250:8036/linux/kb/nginx.html#%E6%94%AF%E6%8C%81websocket)

WebSocket是HTML5下一种新的协议。它实现了浏览器与服务器全双工通信，能更好的节省服务器资源和带宽并达到实时通讯的目的。它与HTTP一样通过已建立的TCP连接来传输数据，但是它和HTTP最大不同是：

- WebSocket是一种双向通信协议。在建立连接后，WebSocket服务器端和客户端都能主动向对方发送或接收数据，就像Socket一样；
- WebSocket需要像TCP一样，先建立连接，连接成功后才能相互通信。

Nginx从「1.3」版本开始支持WebSocket，其可以作为一个反向代理和为WebSocket程序做负载均衡。允许在客户机和后端服务器之间建立隧道，Nginx支持WebSocket。对于NGINX将升级请求从客户端发送到后台服务器，必须明确设置Upgrade和Connection标题。

### [Nginx开启WebSocket代理的配置方法如下：](http://114.242.246.250:8036/linux/kb/nginx.html#nginx%E5%BC%80%E5%90%AFwebsocket%E4%BB%A3%E7%90%86%E7%9A%84%E9%85%8D%E7%BD%AE%E6%96%B9%E6%B3%95%E5%A6%82%E4%B8%8B)

注意

这里严格区分大小写，所有配置内容需要和文档保持一致！

1.编辑nginx.conf，在http区域内一定要添加下面配置：

```
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}
```

map指令的作用：该作用主要是根据客户端请求中的值，来构造改变connection_upgrade的值，即根据变量的值创建新的变量connection_upgrade， 创建的规则就是{}里面的东西。其中的规则没有做匹配，因此使用默认的，即 http_upgrade为空字符串的话，那么值就是 close。

2.编辑vhosts下虚拟主机的配置文件，在location匹配配置中添加如下内容（配置到后端服务的location中）：

```
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "$connection_upgrade";
# proxy_set_header Connection "Upgrade"; 写死为 Upgrade 也可以
```

完整的示例如下:

```
map $http_upgrade $connection_upgrade {
    default upgrade;
    '' close;
}

# 门户系统后端负载
upstream hos-portal {
        server 192.168.1.1:8000;
        server 192.168.1.2:8000;
}

# 门户系统配置
server {
    listen       8000 ssl;
    server_name  localhost;

    ssl_certificate        /hos/app/ssl/hos/hos.crt;
    ssl_certificate_key    /hos/app/ssl/hos/hos.key;

    # 门户前端静态文件
    location / {
        alias  /hos/app/hos-portal/dist/;
        try_files $uri $uri/ /index.html;
        index  index.html index.htm;
    }

    # 门户后端代理，支持websocket
    location /api/ {
        proxy_pass http://hos-portal/api/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "$connection_upgrade";
    }
}
```

### [Nginx代理webSocket经常中断的解决方法（即如何保持长连接）](http://114.242.246.250:8036/linux/kb/nginx.html#nginx%E4%BB%A3%E7%90%86websocket%E7%BB%8F%E5%B8%B8%E4%B8%AD%E6%96%AD%E7%9A%84%E8%A7%A3%E5%86%B3%E6%96%B9%E6%B3%95-%E5%8D%B3%E5%A6%82%E4%BD%95%E4%BF%9D%E6%8C%81%E9%95%BF%E8%BF%9E%E6%8E%A5)

```
http {
    server {
        location / {
            root   html;
            index  index.html index.htm;
            proxy_pass http://sre_backend;
            proxy_http_version 1.1;
            proxy_connect_timeout 5s;
            proxy_read_timeout 60s;
            proxy_send_timeout 30s;
            proxy_set_header Upgrade $http_upgrade; 
            proxy_set_header Connection "$connection_upgrade";  
        }
    }
}
```

超时参数解释：

- proxy_read_timeout参数：默认值60秒,该指令设置与代理服务器的读超时时间。它决定了nginx会等待多长时间来获得请求的响应。这个时间不是获得整个response的时间，而是两次reading操作的时间。即是服务器对你等待最大的时间，也就是说当你使用nginx转发webSocket的时候，如果60秒内没有通讯，依然是会断开的，所以，你可以按照你的需求来设定。比如说，我设置了5分钟，那么如果我5分钟内有通讯，或者5分钟内有做心跳的话，是可以保持连接不中断的。所以这个时间是看你的业务需求来调整时间长短的。
- proxy_send_timeout参数：默认值60秒,设置了发送请求给upstream服务器的超时时间。超时设置不是为了整个发送期间，而是在两次write操作期间。如果超时后，upstream没有收到新的数据，nginx会关闭连接。