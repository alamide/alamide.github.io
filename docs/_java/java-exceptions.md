---
layout: post
title: 一些遇到的异常
categories: java
tags: Java Exceptions
isHidden: true
---

## 1.org.apache.ibatis.binding.BindingException: Invalid bound statement (not found)
这是由于 `*Mapper.xml` 没有被扫描到，导致无法绑定，查看是否正确配置 `mapper-locations`