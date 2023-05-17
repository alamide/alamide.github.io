---
layout: post
title: Centos7 常用命令
categories: centos
tags: centos
date: 2023-05-12
---
Centos7 一些常用的常用命令
<!--more-->

### 1.设置 ssh 连接时长
```bash
vim /etc/ssh/sshd_config
```

修改文件
```
#多久向客户端发送请求，300表示300秒即5分钟；
ClientAliveInterval 300
#超过多少次请求没被客户端响应，结束连接；
ClientAliveCountMax 10
```

重启 sshd 
```bash
service sshd restart
```
