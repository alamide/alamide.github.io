---
layout: default
title: MySQL-Base-Table
categories: db
tags: DB MySQL 
excerpt: MySQL 修改相关的基础知识，操作数据表。
date: 2023-03-16
---
本文所使用的表 [在这里](./mysql-base-db-ddl.html)

## 1.创建表
```sql
-- 方式一
CREATE TABLE IF NOT EXISTS t_emp
(
    emp_id   INT AUTO_INCREMENT          PRIMARY KEY,
    emp_name VARCHAR(20)                 NOT NULL,
    sex      ENUM ('M', 'F') DEFAULT 'M' NULL,
    email    VARCHAR(50)                 NULL,
    prov     VARCHAR(100)                NULL,
    age      INT                         NULL,
    salary   INT                         NULL,
    dept_id  INT                         NULL
);

-- 方式二
CREATE TABLE IF NOT EXISTS t_emp_back
AS SELECT emp_id, emp_name, salary FROM t_emp WHERE salary > 10000;

-- 查看表结构
SHOW CREATE TABLE t_emp;
```

## 2.修改表结构
### 2.1 追加一列
语法：`ALTER TABLE 表名 ADD 字段名 字段类型 [FIRST|AFTER 字段名2]`

FIRST 放在第一行、AFTER 放在指定列后
```sql
-- 添加 dept_id 列，且放在 emp_id 列后面
ALTER TABLE t_emp_back ADD dept_id INTEGER DEFAULT 0 AFTER emp_id;
```

### 2.2 修改一列
语法：`ALTER TABLE 表名 MODIFY 字段名 字段类型 [DEFAULT 默认值] [FIRST|AFTER 字段名2]`
```sql
ALTER TABLE t_emp_back MODIFY dept_id DOUBLE(9,2) DEFAULT 1.5 FIRST;
```

### 2.3 重命名一列
语法：`ALTER TABLE 表名 CHANGE 列名 新列名 新数据类型`
```sql
ALTER TABLE t_emp_back CHANGE dept_id d_id INTEGER;
```

### 2.4 删除一列
语法：`ALTER TABLE 表名 DROP 列名`
```sql
ALTER TABLE t_emp_back DROP dept_id;
```

### 2.5 更改表名
```sql
-- 方式一
RENAME TABLE t_emp_back TO t_emp_backup;

-- 方式二
ALTER TABLE t_emp_backup RENAME TO t_emp_back;
```

## 3.删除表
```sql
DROP TABLE IF EXISTS t_emp;
```

## 4.清空表
DELTE 清空表 primary_key 不会从 1 开始，TRUNCATE 会清空表，primary_key 从 1 开始
```sql
TRUNCATE TABLE t_emp;
```

## 5.插入数据
语法一：`INSERT INTO 表名 VALUES (value1, value2, value3, ...)`

语法二：`INSRT INTO 表名(column1, column2, ...) VALUES (values1, values2, ...)`

语法三：`INSERT INTO 表名 VALUES (value1, value2, value3, ...), (value1, value2, value3, ...), ...`

语法四：`INSRT INTO 表名(column1, column2, ...) VALUES (values1, values2, ...), (values1, values2, ...), ...`

语法一需要为表中的每一个字段赋值，语法二仅需为指定字段赋值

语法五：将查询的结果插入到表中
```
INSET INTO 目标表名
(tar_column1 [, tar_column2, ..., tar_columnn])
SELECT
(src_column1 [, src_column2, ..., src_columnn])
FROM 源表名
[WHERE condition]
```

```sql
INSERT INTO t_emp_back(d_id, emp_id, emp_name, salary)
SELECT dept_id, emp_id, emp_name, salary FROM t_emp WHERE salary > 10000;
```

## 6.更新数据
语法：`UPDATE 表名 SET column1=value1, column2=value2, ... [WHERE condition]`
```sql
UPDATE t_emp_back SET salary=80000 WHERE emp_id<10;
```

## 7.删除数据
语法：`DELETE FROM 表名 [WHERE condition]`



## 8.查询
### 8.1 列的别名
使用 AS ，AS 可以省略
```sql
SELECT emp_name AS name, emp_id  id FROM t_emp;
```

### 8.2 去除重复行
Distinct
```sql
SELECT DISTINCT dept_id FROM t_emp;
```

### 8.3 空值
空值不等同于 0，''，'null'，遇到 NULL 参与运算时，可以使用 IFNULL

筛选为空 IS NULL，筛选不为空 IS NOT NULL
```sql
SELECT IFNULL(prov, '未知') FROM t_emp;
```

### 8.4 着重号
表字段或数据表名与保留字相同时，使用着重号 ``
```sql
SELECT * FROM `order`;
```

### 8.5 查询常数
```sql
SELECT '科技会' AS '公司名', emp_id FROM t_emp;
```

### 8.6 一些运算符
1. BETWEEN AND

2. IN

3. NOT IN

4. LIKE （% 匹配 0 个或多个字符，_ 只能匹配一个字符）

### 8.7 排序分页
排序语法：`ORDER BY [column1, column2, ...] [ASC|DESC]` ，column1 相同则按 column2 排序，ASC 升序，DESC 降序

分页语法：`LIMIT 偏移量, 每页数` ，LIMIT 子句放在整个 SELECT 语句最后

### 8.8 多表查询
```sql
-- 笛卡尔积
SELECT * FROM t_emp, t_dept;

-- 内连接，合并表中相同值的行
SELECT * FROM t_emp, t_dept WHERE t_emp.dept_id = t_dept.dept_id;

-- 左外连接，右边表中没有匹配的行，则相应的列为 NULL，t_emp 为主表
SELECT * FROM t_emp LEFT JOIN t_dept ON t_emp.dept_id = t_dept.dept_id;

-- 右外连接，左边表中没有匹配的行，则相应的列为 NULL，t_dept 为主表
SELECT * FROM t_emp RIGHT JOIN t_dept ON t_emp.dept_id = t_dept.dept_id;
```

### 8.9 UNION
利用UNION关键字，可以给出多条SELECT语句，并将它们的结果组合成单个结果集。合并时，两个表对应的列数和数据类型必须相同，并且相互对应。
```sql
SELECT column,... FROM table1
UNION [ALL]
SELECT column,... FROM table2
```

UNION 会去除重复记录，UNION ALL 不会去除重复的记录

### 8.10 自然连接
SQL99 在 SQL92 的基础上提供了一些特殊语法，比如 NATURAL JOIN 用来表示自然连接。我们可以把 自然连接理解为 SQL92 中的等值连接。它会帮你自动查询两张连接表中 所有相同的字段 ，然后进行 等值 连接。
```sql
-- SQL92
SELECT t_emp.dept_id, dept_name, emp_id, emp_name, sex, email, prov, age, salary FROM t_emp, t_dept WHERE t_emp.dept_id = t_dept.dept_id;

-- SQL99
SELECT * FROM t_dept NATURAL JOIN t_emp;
```

### 8.11 USING
```sql
SELECT t_emp.dept_id, dept_name, emp_id, emp_name, sex, email, prov, age, salary FROM t_emp JOIN t_dept USING (dept_id);

-- 等同于
SELECT t_emp.dept_id, dept_name, emp_id, emp_name, sex, email, prov, age, salary FROM t_emp, t_dept WHERE t_emp.dept_id = t_dept.dept_id;
```

### 8.12 聚合函数
聚合函数作用于一组数据，并对一组数据返回一个值

聚合函数 `AVG` 、`SUM` 、`MAX` 、`MIN` 、`COUNT` ，COUNT(*) 返回表中记录总数，COUNT(expr) 返回 expr 不为空的记录总数，要统计总函数时使用 COUNT(*)

### 8.13 GROUP BY
将表中的数据分为若干组，SELECT 中出现的非组函数字段必须声明在 GROUP BY 中，
```sql
SELECT dept_id, max(salary) AS '最高工资' FROM t_emp GROUP BY dept_id;
```

### 8.14 HAVING
过滤分组，不能单独使用，需要结合 GROUP BY 使用
```sql
SELECT dept_id, max(salary) AS '最高工资' FROM t_emp GROUP BY dept_id HAVING max(salary) < 10000;
```

### 8.15 子查询
```sql
-- 查询人事部员工
SELECT emp_name FROM t_emp WHERE dept_id = (SELECT dept_id FROM t_dept WHERE dept_name='人事部');

-- 查询非人事部员工
SELECT emp_name FROM t_emp WHERE dept_id IN (SELECT dept_id FROM t_dept WHERE  dept_name <> '人事部');

-- 挑选工资最低的员工
SELECT emp_name, salary FROM t_emp WHERE salary <= ALL (SELECT salary FROM t_emp);

-- 查询工资非最低的员工
SELECT emp_name, salary FROM t_emp WHERE salary > ANY (SELECT salary FROM t_emp);
```
