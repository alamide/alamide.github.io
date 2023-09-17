---
layout: post
title: Docker 镜像自动发布到阿里云
categories: docker
tags: docker container
date: 2023-09-17
---
使用 Shell 脚本，自动发布镜像到阿里云仓库。
<!--more-->
```shell
#!/bin/bash

#需要制作镜像的容器 ID
container_id="e3fe6eb64c0a"
#目标镜像名
image_name="cts"
#制作镜像时提交的信息
default_commit_message="make new image"
#作者名
author="alamide"
#远程仓库地址
remote_repo="registry.cn-hangzhou.aliyuncs.com/alamide/test"

tag=`docker images -a | grep $image_name | awk '{print $2}' | sort -r | sed -n '1p' | awk -F '.' '{print($1"."($2+1))}'`

read -p "commit message, default($default_commit_message): " input_commit_msg
read -p "tag, default($tag): " input_tag

if [ -z "$default_commit_message" ] ;then
    input_commit_msg=$default_commit_message
fi

if [ -z "$input_tag" ] ;then
    input_tag=$tag
fi

docker commit -m="$input_commit_msg" -a="$author" $container_id "$image_name":"$input_tag"

image_id=`docker images -a | grep $image_name | grep $input_tag | awk '{print $3}' | sed -n '1p'`

docker tag "$image_id" "$remote_repo:$input_tag"
docker push "$remote_repo:$input_tag"
echo "execute complete"
```