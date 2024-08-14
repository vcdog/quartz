---
title: "GitLab Documentation"
description: "How to use GitLab effectively"
author: "Your Name"
date: "2023-08-09"
---


```
# 1.解压文件并重命名
cd /hos/app
tar zxf jdk-8u231-linux-x64.tar.gz
mv jdk1.8.0_231 jdk

#2.配置环境变量：编辑/etc/profile，在文件最后增加以下内容并保存（安装目录修改为实际路径）：
export JAVA_HOME=/hos/app/jdk
export PATH=$PATH:$JAVA_HOME/bin

#执行命令使之生效：
source /etc/profile

#3.验证，输出版本信息：
java -version
```