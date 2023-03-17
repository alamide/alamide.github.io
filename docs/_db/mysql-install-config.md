---
layout: default
title: MySQL-Install-Config
categories: db
tags: DB MySQL 
excerpt: MySQL 安装配置
date: 2023-02-15
---
## 1.安装数据库
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
5. 登入数据库
   ```shell
   mysql -uroot -proot
   ```

## 2.配置数据库
1. 开启日志记录，开启之后可以查看数据库的访问记录，包括查询等。
```shell
mysql> show variables like '%general_log%';
```
<table border="1">
   <tr><th>Variable_name</th><th>Value</th></tr>
   <tr><td>general_log</td><td>ON</td></tr>
   <tr><td>general_log_file</td><td>/var/lib/mysql/0e8f412b5f18.log</td></tr>
</table>


```shell
mysql> set global general_log=on;
```