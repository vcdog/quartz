

---
title: "GitLab Documentation"
description: "How to use GitLab effectively"
author: "Your Name"
date: "2023-08-09"
---


## [麒麟系统启动失败问题](http://114.242.246.250:8036/linux/kb/keepalived.html#%E9%BA%92%E9%BA%9F%E7%B3%BB%E7%BB%9F%E5%90%AF%E5%8A%A8%E5%A4%B1%E8%B4%A5%E9%97%AE%E9%A2%98)

- 问题：安装Keepalived后启动失败，报错缺少librpm.so.8/librpmio.so.8库文件
- 原因：麒麟系统安装的是librpm.so.9版本
- 方案：手动上传librpm.so.8/librpmio.so.8文件，步骤如下：

1. 将安装包中的lib64目录下的librpm.so.8.2.0和librpmio.so.8.2.0文件上传到服务器的/usr/lib64/中；
2. 增加权限：

```
cd /usr/lib64
chmod +x librpm.so.8.2.0 librpmio.so.8.2.0
```

3. 创建软链接：

```
ln -s librpm.so.8.2.0 librpm.so.8
ln -s librpmio.so.8.2.0 librpmio.so.8
```

4. 重新启动keepalived服务即可。