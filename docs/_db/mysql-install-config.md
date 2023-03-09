---
layout: default
title: MySQL-Install-Config
categories: db
tags: DB MySQL 
excerpt: MySQL 安装配置。
---
### 1.安装数据库
安装使用 docker 
1. 在docker中心搜索MySQL镜像 [https://hub.docker.com](https://hub.docker.com)
   
2. 获取镜像 
   ```shell
   docker pull mysql:5.7
   ```
3. 创建容器 
   ```shell
   # -p 端口映射 docker 容器端口号:本地主机端口号
   # --name 容器名
   # -e 环境参数
   # -d 后台运行
   docker run -p 3306:3306 --name mysqldb003 -e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=db03 -d mysql:5.7
   ```
4. 登录容器
   ```shell
   docker exec -it mysqldb003 /bin/bash
   ```