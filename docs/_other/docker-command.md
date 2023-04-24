---
layout: post
title: Docker 常用命令
categories: docker
tags: docker container
date: 2023-04-23
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
#登入
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


version: "1"
 

services:

  dataService:

    image: mvn17:1.1

    container_name: data01

    ports:

      - "8080:8080"

    volumes:

      - /root/app/dataService:/data

    networks: 

      - alamide_net 

    depends_on: 

      - nginx

      - mysql

 

  nginx:

    image: nginx:1.24

    container_name: nginx124

    ports:

      - "80:80"

    volumes:

      - /root/app/nginx/conf.d:/etc/nginx/conf.d

    networks: 

      - alamide_net
  

  mysql:

    image: mysql:5.7

    container_name: mysql57

    environment:

      MYSQL_ROOT_PASSWORD: '123456'

      MYSQL_ALLOW_EMPTY_PASSWORD: 'no'

      MYSQL_DATABASE: 'blog_statistics'

      MYSQL_USER: 'alamide'

      MYSQL_PASSWORD: 'alamide123'

    ports:

       - "3306:3306"

    volumes:

       - /root/app/mysql/db:/var/lib/mysql

       - /root/app/mysql/conf/my.cnf:/etc/my.cnf

       - /root/app/mysql/init:/docker-entrypoint-initdb.d

    networks:

      - alamide_net

    command: --default-authentication-plugin=mysql_native_password #解决外部无法访问

 

networks: 

   alamide_net: 