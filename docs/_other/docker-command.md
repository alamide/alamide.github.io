---
layout: post
title: Docker 常用命令
categories: docker
tags: docker container
date: 2023-04-23
isHidden: true
---
Docker 一些常用的常用命令
<!--more-->

## 1.帮助类启动命令
```bash
systemctl start docker

systemctl stop docker

systemctl restart docker

systemctl status docker

#开机启动
systemctl enable docker

#查看 docker 简要信息
docker info

docker --help

#具体命令帮助文档
docker run --help
```

## 2.镜像命令
### 2.1 docker images
列出本地镜像
```bash
docker images

#列出本地所有镜像
docker images -a

#只显示镜像 id
docker images -q
```

### 2.2 docker search
搜索某个镜像，[镜像仓库地址](https://hub.docker.com)
```shell
docker serach mysql

#列出前五个
docker search --limit 5 mysql
```

### 2.3 docker pull
下载镜像 `docker pull 镜像名字[:TAG]`
```shell
#获取 mysql 最新版
docker pull mysql

#获取 mysql5.7 版本
docker pull mysql:5.7
```

### 2.4 docker commit
提交容器副本，使其成为一个新的镜像
```shell
#运行 centos7
docker run -it centos:centos7 /bin/bash

#安装 vim
yum install vim

#Ctrl + p + q 退出

#制作新的镜像，新的镜像带有 vim 功能 -m 提交信息 -a 作者  cts 镜像名 1.2 版本号
docker commit -m="add vim " -a="alamide" 1faf600796bc cts:1.2

#运行并登入，可以看到可以直接使用 vim
docker run -it cts:1.2 /bin/bash
```

### 2.5 docker system df 
查看镜像/容器/数据卷所占的空间
```shell
docker system df
```

### 2.6 docker rmi
删除某个镜像
```shell
#删除 mysql 最新版
docker rmi mysql

#删除特定版本
docker rmi mysql:5.7

#强制删除，创建容器实例后，需要加上 f
docker rmi -f mysql:5.7

#删除多个
docker rmi -f redis redis:6.0.8

#全部删除
docker rmi -f $(docker images -qa)
```

## 3.容器命令
先获取 centos7 镜像
```shell
docker pull centos:centos7
```

### 3.1 docker run 
新建并启动 docker，`docker run [OPTIONS] IMAGE`
```shell
#/bin/bash 是命令 [-i interactive 交互式操作] [-t Allocate a pseudo-TTY 终端]
docker run -it centos:centos7 /bin/bash

#后台运行
docker run -d redis
```

### 3.2 docker ps 
列出当前所有容器
```shell
#列出所有正在运行的容器
docker ps

#列出所有正在运行+历史上运行过的容器
docker ps -a

#列出最近创建的容器，所有状态的
docker ps -l

#列出最近的 3 个容器 -n 
docker ps -n 3

#只显示容器编号
docker ps -q
```

### 3.3 退出容器
exit `docker run -it centos:centos7 /bin/bash` ，以这种方式登入容器，退出，容器停止

Ctrl + p + q `docker run -it centos:centos7 /bin/bash` ，以这种方式登入容器，退出，容器不停止

### 3.4 docker start
启动已停止的容器 docker start 容器ID 或 容器名
```shell
docker start a9cb7535c278
```

### 3.5 docker restart
重启容器 docker restart 容器ID 或 容器名
```shell
docker restart a9cb7535c278
```

### 3.6 docker stop
停止容器 docker stop 容器ID 或 容器名
```shell
docker stop a9cb7535c278
```

### 3.7 docker kill
强制停止容器 docker kill 容器ID 或 容器名
```shell
docker kill a9cb7535c278
```

### 3.8 docker rm 
删除 docker rm 容器ID 或 容器名
```shell
#删除已停止容器
docker rm a7d01a5cbebb

#强制删除容器
docker rm -f elastic_lumiere

#强制删除所有容器
docker rm -f $(docker ps -aq)

#强制删除所有容器
docker ps -a -q|xargs docker rm -f
```

### 3.9 与正在运行的容器交互
启动一个 redis ，并后台运行
```shell
docker run -d redis:6.0.8
```

exec 登入，启动新的进程连接，exit 后，容器不会停止
```shell
#登入 exec 是执行容器命令 docker exec [OPTIONS] CONTAINER COMMAND [ARG...]
#docker exec nginx01 nginx -s reload 容器内 nginx 重新加载配置文件
docker exec -it 15f363022ca6 /bin/bash

#登入并执行 redis-cli
docker exec -it 15f363022ca6 redis-cli
```

attach 登入，exit 后，容器停止
```shell
docker attach 15f363022ca6
```

### 3.10 容器和主机文件互传
docker cp 容器ID:容器内路径 目的主机路径
```shell
#容器内文件拷贝到本地
docker cp dac0e6def6d4:/home/hello.txt /root/data/

#本地拷贝到容器内
docker cp /root/data/abc.txt dac0e6def6d4:/home/abc.txt 

#导出容器内容作为一个归档文件
docker export dac0e6def6d4 > ctos.tar.gz

#删除所有镜像
docker rmi -f $(docker images -qa)

#从包内容创建一个新的文件系统再导入为镜像
cat ctos.tar.gz | docker import - alamide/centos:centos7

#登入，可以看到 /home/abc.txt /home/hello.txt
docker run -it alamide/centos:centos7 /bin/bash
```

### 3.11 一些其它
以下都可以使用容器 ID 或 容器名
```bash
#查看容器日志
docker logs 6e9f053094ce
docker logs awesome_darwin

#查看容器内进程
docker top 6e9f053094ce

#查看容器内部细节
docker inspect 6e9f053094ce
```

## 4.数据卷
Docker 可以通过容器卷来完成主机和容器数据共享

通过 -v 来实现，-v 主机目录:容器目录

```bash
docker run \
    --name nginx80 \
    --network inner-network \
    -v /opt/docker-volume/nginx/logs:/var/log/nginx \
    -v /opt/docker-volume/nginx/conf:/etc/nginx/conf.d:ro \
    -v /opt/docker-volume/nginx/www:/usr/share/nginx/www:ro \
    -p 80:80 \
    -d nginx:1.24
```

## 5.Docker 网络
Docker 容器间通信可以使用 IP 来通信，但是因为 IP 地址是 Docker 默认分配的，可能会变更，所以可以使用 Docker 的 network 来解决

创建一个网络
```bash
docker network create inner-network
```

使用已定义的网络创建一个 MySQL 容器
```bash
docker run -p 3306:3306 \
    --network inner-network \
    --name mysql3306\
    -e MYSQL_ROOT_PASSWORD=root \
    -e MYSQL_DATABASE=db03 \
    -d mysql:5.7
docker exec -it mysql3306 bash

mysql -uroot -proot

use db03;
create table test (id int primary key auto_increment, name varchar(20));
insert into test values (null, "zhangsan"), (null, "lisi");
```

创建微服务，可以使用 mysql3306 来访问数据库
```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: root
    password: root
    url: jdbc:mysql://mysql3306:3306/db03
```

启动微服务
```bash
docker run --name server8007 --network inner-network -p 8007:8007 -d cc57abef6ed8
```

可以访问数据库成功

查看网络
```bash
#查看
docker network ls

#查看
docker network inspect inner-network

#移除
docker network rm inner-network
```

