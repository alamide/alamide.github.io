---
layout: post
title: Docker Install
categories: docker
tags: docker
date: 2023-04-23
---
Docker 在 Centos7 上安装，使用
<!--more-->
文档地址 [在这里](https://docs.docker.com/engine/install/centos/)

## 1.安装必要库
```shell
yum -y install gcc
yum -y install gcc-c++
```

## 2.卸载旧版本
```shell
sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
```

## 3.安装必要软件包
```shell
yum install -y yum-utils
```

## 4.设置镜像仓库
推荐使用阿里云的镜像仓库，官网给的速度可能比较慢， [仓库地址](https://developer.aliyun.com/article/110806)
```shell
sudo yum-config-manager --add-repo http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
#更新软件包索引
sudo yum makecache fast
```

## 5.安装
```shell
sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## 6.启动
```shell
sudo systemctl start docker
```

## 7.测试
```shell
sudo docker run hello-world
```

## 8.镜像加速
使用阿里云镜像加速，需要去阿里云申请 [在这里](https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors)
```shell
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://***.mirror.aliyuncs.com"]
}
EOF
sudo systemctl daemon-reload
sudo systemctl restart docker

#查看是否配置成功
sudo docker info
```

## 9.卸载
```shell
sudo systemctl stop docker

sudo yum remove docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin docker-ce-rootless-extras

sudo rm -rf /var/lib/docker

sudo rm -rf /var/lib/containerd

```