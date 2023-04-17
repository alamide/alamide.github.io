---
layout: post
title: MyBatis PageHelper
categories: db
tags: Mybatis
date: 2023-04-14
isHidden: true
---
Mybatis 分页插件，[在这里](https://github.com/pagehelper/Mybatis-PageHelper)
<!--more-->
## 1.引入依赖
```xml
<dependency>
  <groupId>com.github.pagehelper</groupId>
  <artifactId>pagehelper-spring-boot-starter</artifactId>
  <version>1.4.6</version>
</dependency>
```

## 2.使用
```java
PageHelper.startPage(page, pageSize);

final List<Employee> employeesByLikeUsername = employeeService.getEmployeesByLikeName(name);
PageInfo<Employee> pageInfo = new PageInfo<>(employeesByLikeUsername);
HashMap<String, Object> data = new HashMap<>();
data.put("records", pageInfo.getList());
data.put("total", pageInfo.getTotal());
```