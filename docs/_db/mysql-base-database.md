---
layout: post
title: MySQL-Base-Database
categories: db
excerpt: MySQL 数据库。
---
### 1.一些🍪
* 创建数据库时需要设置编码方式，一般设置为 utf8mb4(utf8 most bytes 4，可以存储 emoji，utf8最大长度为3个字节，不能存储emoji，latin1不能存储汉字)。
如不指定，编码集默认为 latin1。

* 创建表时也可以指定编码集，但是最好在创建数据库时就指定好。
  
* 当修改数据库编码集时，数据库中表的编码集并不会同时变更，所以创建数据库时要指定编码集！


查询支持的编码集
```sql
SHOW CHARACTER SET LIKE 'utf%';
```
<table border="1" style="border-collapse:collapse">
<tr><th>Charset</th><th>Description</th><th>Default collation</th><th>Maxlen</th></tr>
<tr><td>utf8</td><td>UTF-8 Unicode</td><td>utf8_general_ci</td><td>3</td></tr>
<tr><td>utf8mb4</td><td>UTF-8 Unicode</td><td>utf8mb4_general_ci</td><td>4</td></tr>
<tr><td>utf16</td><td>UTF-16 Unicode</td><td>utf16_general_ci</td><td>4</td></tr>
<tr><td>utf16le</td><td>UTF-16LE Unicode</td><td>utf16le_general_ci</td><td>4</td></tr>
<tr><td>utf32</td><td>UTF-32 Unicode</td><td>utf32_general_ci</td><td>4</td></tr>
</table>

### 2.创建数据库
****

```sql
CREATE DATABASE [数据库名];

CREATE DATABASE [数据库名] CHARACTER SET [字符集]; 

CREATE DATABASE IF NOT EXISTS [数据库名] CHARACTER SET [字符集]; 
```
### 3.使用数据库 

***
```sql
SHOW TABLES FROM [数据库名];

SHOW DATABASES;

-- 显示当前使用的库名
SELECT DATABASE();

-- 显示建表语句
SHOW CREATE DATABASE [数据库名];

USE [数据库名];
```
### 4.修改数据库

***
```sql
-- 修改编码集
ALTER DATABASE [数据库名] CHARACTER SET [字符集];-- gbk, utf8, latin1, utf8mb4

DROP DATABASE [数据库名];

DROP DATABASE IF EXISTS [数据库名];
```
### 5.代码示例

***
```sql
DROP DATABASE IF EXISTS db03;

CREATE DATABASE IF NOT EXISTS db03 CHARACTER SET gbk;

ALTER DATABASE db03 CHARACTER SET utf8mb4;

SHOW CREATE DATABASE db03;

SHOW DATABASES;

USE db03;

SELECT DATABASE();

SHOW TABLES FROM db03;
```


