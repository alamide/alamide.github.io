---
layout: post
title: Android ADB
categories: android
tags: android adb
date: 2026-06-30
---
Android ADB 的一些常用命令。
<!--more-->
1. 获取已安装 Apk 的安装包

```
adb shell pm list packages | grep "关键词" 查找目标应用的包名

adb shell pm path <包名> 获取 Apk 安装的完整路径

adb pull <APK路径> <本地保存路径>
```