---
layout: post
title: Docker 镜像发布到仓库
categories: docker
tags: docker container
date: 2023-04-24
---
镜像发步到仓库，如阿里云、私有库，这样就可以更好的管理、获取自己的镜像了
<!--more-->
## 1.发布到阿里云
需要自己去申请 [文档](https://cr.console.aliyun.com/cn-hangzhou/instances)

```bash
#1.制作新的镜像，新的镜像带有 vim 功能 -m 提交信息 -a 作者  cts 镜像名 1.2 版本号
docker commit -m="add vim " -a="alamide" 1faf600796bc cts:1.2

#2.登录
docker login --username=alamide registry.cn-hangzhou.aliyuncs.com

#3.Create a tag TARGET_IMAGE that refers to SOURCE_IMAGE
docker tag 91ea7a97fd33 registry.cn-hangzhou.aliyuncs.com/alamide/alamide:1.0

#4.推送
docker push registry.cn-hangzhou.aliyuncs.com/alamide/alamide:1.0

#5.拉取
docker pull registry.cn-hangzhou.aliyuncs.com/alamide/alamide:1.0
```