---
layout: post
title: Docker Dockerfile Compose
categories: docker
tags: docker
date: 2023-05-01
isHidden: true
---
Dockerfile 是用来构建 Docker 镜像的文本文件，是由一条条构建镜像所需的指令和参数构成的脚本。

Compose 实现对 docker 容器群的快速编排。
<!--more-->

## 1.Dockerfile
Dockerfile 常用构建保留字

1. `FROM` 基础镜像，当前镜像是基于哪个镜像的，指定一个已存在的镜像作为基础镜像，第一条必须是 FROM

2. `MAINTAINER` 镜像维护者的姓名和邮箱

3. `RUN` 构建容器时需要执行的命令， `RUN yum -y install vim` ，相当于直接在命令行输入命令

4. `EXPOSE` 容器向外暴露的端口，例如 tomcat Dockerfile 为 EXPOSE 8080

5. `WORKDIR` 指定在创建容器后，登录进容器的工作目录

6. `USER` 指定镜像以什么样的用户执行，默认为 root

7. `ENV` 构建镜像过程中设置环境变量

8. `ADD` 将宿主目录下文件拷贝进容器中，会自动处理 URL 和解压 tar 文件

9. `COPY` 与 ADD 类似，拷贝文件到镜像中 COPY src dest

10. `VOLUME` 数据容器卷

11. `CMD` 指定容器启动后执行的命令，可以有多个 CMD ，但是只有最后一个生效，CMD 会被 docker run 之后的参数替换，RUN 是构建时运行

12. `ENTRYPOINT` 指定容器启动时执行的命令，类似于 CMD ，也是只有最后一个生效，但是不会被 docker run 后面的命令覆盖，`ENTRYPOINT["java", "-jar", "aa.jar"]`

制作 JAVA 环境
```
FROM centos:centos7
MAINTAINER alamide<alamide@163.com>

WORKDIR /opt/softwear

RUN yum -y install wget

RUN wget https://download.oracle.com/java/17/latest/jdk-17_linux-x64_bin.tar.gz

RUN tar -xzf jdk-17_linux-x64_bin.tar.gz 

RUN rm -f jdk-17_linux-x64_bin.tar.gz

ENV JAVA_HOME /opt/softwear/jdk-17.0.7

ENV JRE_HOME $JAVA_HOME/jre

ENV CLASSPATH $JAVA_HOME/lib/dt.jar:$JAVA_HOME/tools.jar:$JRE_HOME/lib:$CLASSPATH

ENV PATH $JAVA_HOME/bin:$PATH

RUN wget https://dlcdn.apache.org/maven/maven-3/3.8.8/binaries/apache-maven-3.8.8-bin.tar.gz --no-check-certificate

RUN tar -xzf apache-maven-3.8.8-bin.tar.gz

RUN rm -f apache-maven-3.8.8-bin.tar.gz

ENV MVN_HOME /opt/softwear/apache-maven-3.8.8

ENV PATH $MVN_HOME/bin:$PATH

EXPOSE 8080

CMD /bin/bash
```

简洁版 SpringBoot 镜像
```
FROM openjdk:17

COPY ./target/DataStatistics-1.0-SNAPSHOT.jar /root/app/DataStatistics-1.0-SNAPSHOT.jar

WORKDIR /root/app

#修改时区
RUN cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

#从命令行指定配置文件的前提是  SpringApplication.run(MainApplication.class, args); 要传入 args，否则无效，我踩的一个小坑
ENTRYPOINT ["java", "-jar", "DataStatistics-1.0-SNAPSHOT.jar" ,"--spring.profiles.active=prod"]

EXPOSE 8080
```

构建
```bash
docker build -t [镜像名称]:[版本] .   
```

## 2.Compose
来批量启动容器，可以快速部署集群

文件名 `docker-compose.yml` 

编排 `docker compose up -d`
```yaml
version: "1"

services:
  nginx:
    image: nginx:1.24
    container_name: nginx80
    ports:
      - "80:80"
      - “443:443”
    volumes:
      - /opt/docker-volume/nginx/logs:/var/log/nginx
      - /opt/docker-volume/nginx/conf:/etc/nginx/conf.d:ro
      - /opt/docker-volume/nginx/www:/usr/share/nginx/www:ro
    networks:
      - server-network
    depends_on:
      - server

  server:
    image: registry.cn-hangzhou.aliyuncs.com/alamide/alamide
    container_name: server8010
    ports:
      - "8010:8010"
    networks:
      - server-network
    depends_on:
      - mysql
  
  mysql:
    image: mysql:5.7
    container_name: mysql3306
    environment:
      MYSQL_ROOT_PASSWORD: '123456'
      MYSQL_ALLOW_EMPTY_PASSWORD: 'no'
      MYSQL_DATABASE: 'db03'
      MYSQL_USER: 'alamide'
      MYSQL_PASSWORD: 'alamide123'
    ports:
      - "3306:3306"
    volumes:
      - /opt/docker-volume/mysql/db:/var/lib/mysql
      - /opt/docker-volume/mysql/conf/my.cnf:/etc/my.cnf
      - /opt/docker-volume/mysql/init:/docker-entrypoint-initdb.d
    networks:
      - server-network

networks:
  server-network:
    name: server-network
    driver: bridge
```