---
layout: post
title: 数据库测试数据
categories: db
tags: mysql 
date: 2023-03-17
---
一些数据库建表语句，方便快速创建、测试。
<!--more-->
## 建表语句
```sql
CREATE DATABASE IF NOT EXISTS db_jdbc CHARACTER SET utf8mb4;

DROP TABLE IF EXISTS t_emp;

CREATE TABLE t_emp
(
    emp_id   INT AUTO_INCREMENT
        PRIMARY KEY,
    emp_name VARCHAR(20)                 NOT NULL,
    sex      ENUM ('M', 'F') DEFAULT 'M' NULL,
    email    VARCHAR(50)                 NULL,
    prov     VARCHAR(100)                NULL,
    age      INT                         NULL,
    salary   INT                         NULL,
    dept_id  INT                         NULL
);

INSERT INTO t_emp (emp_id, emp_name, sex, email, prov, age, salary, dept_id) VALUES (1, '李三', 'M', 'lisan@emp.com', '江苏', 32, 18000, 1);
INSERT INTO t_emp (emp_id, emp_name, sex, email, prov, age, salary, dept_id) VALUES (2, '王丽', 'F', 'wangli@emp.com', '山东', 25, 9000, 1);
INSERT INTO t_emp (emp_id, emp_name, sex, email, prov, age, salary, dept_id) VALUES (3, '王强', 'M', 'wangqiang@emp.com', '北京', 42, 12000, 2);
INSERT INTO t_emp (emp_id, emp_name, sex, email, prov, age, salary, dept_id) VALUES (4, '何马', 'M', 'hema@emp.com', '江苏', 32, 8000, 2);
INSERT INTO t_emp (emp_id, emp_name, sex, email, prov, age, salary, dept_id) VALUES (5, '赵四海', 'M', 'zhaosihai@emp.com', '河北', 32, 17000, 3);
INSERT INTO t_emp (emp_id, emp_name, sex, email, prov, age, salary, dept_id) VALUES (6, '程兴', 'M', 'chenxing@emp.com', '武汉', 32, 16000, 3);
INSERT INTO t_emp (emp_id, emp_name, sex, email, prov, age, salary, dept_id) VALUES (7, '孙月', 'F', 'sunyue@emp.com', '山东', 22, 10000, 4);
INSERT INTO t_emp (emp_id, emp_name, sex, email, prov, age, salary, dept_id) VALUES (8, '钱金', 'M', 'qianjin@emp.com', '北京', 52, 28000, 4);
INSERT INTO t_emp (emp_id, emp_name, sex, email, prov, age, salary, dept_id) VALUES (9, '刘红', 'F', 'liuhong@emp.com', '安徽', 22, 8000, 5);
INSERT INTO t_emp (emp_id, emp_name, sex, email, prov, age, salary, dept_id) VALUES (10, '蒋一心', 'F', 'jinagyixin@emp.com', '山西', 28, 9000, 5);
INSERT INTO t_emp (emp_id, emp_name, sex, email, prov, age, salary, dept_id) VALUES (11, '刘红', 'F', 'liuhong@emp.com', '河南', 22, 8000, 6);
INSERT INTO t_emp (emp_id, emp_name, sex, email, prov, age, salary, dept_id) VALUES (12, '蒋一心', 'F', 'jinagyixin@emp.com', null, 28, 9000, 6);
INSERT INTO t_emp (emp_id, emp_name, sex, email, prov, age, salary, dept_id) VALUES (13, '马文博', 'M', 'mawenbo@emp.com', '河南', 22, 28000, 7);
INSERT INTO t_emp (emp_id, emp_name, sex, email, prov, age, salary, dept_id) VALUES (14, '成虎', 'M', 'chenhu@emp.com', '新疆', 38, 36000, 7);

DROP TABLE IF EXISTS t_dept;

CREATE TABLE t_dept
(
    dept_id   INT AUTO_INCREMENT
        PRIMARY KEY,
    dept_name VARCHAR(50) NULL
);

INSERT INTO t_dept (dept_id, dept_name) VALUES (1, '人事部');
INSERT INTO t_dept (dept_id, dept_name) VALUES (2, '产品部');
INSERT INTO t_dept (dept_id, dept_name) VALUES (3, '技术部');
INSERT INTO t_dept (dept_id, dept_name) VALUES (4, '运维部');
INSERT INTO t_dept (dept_id, dept_name) VALUES (5, '财务部');
INSERT INTO t_dept (dept_id, dept_name) VALUES (6, '采购部');
INSERT INTO t_dept (dept_id, dept_name) VALUES (7, '研发部');

```

<h2>Table t_emp</h2>
<table border="1" class="sql_tab">
  <tr>
    <th>emp_id</th>
    <th>emp_name</th>
    <th>sex</th>
    <th>email</th>
    <th>prov</th>
    <th>age</th>
    <th>salary</th>
    <th>dept_id</th>
  </tr>
  <tr>
    <td>1</td>
    <td>李三</td>
    <td>M</td>
    <td>lisan@emp.com</td>
    <td>江苏</td>
    <td>32</td>
    <td>18000</td>
    <td>1</td>
  </tr>
  <tr>
    <td>2</td>
    <td>王丽</td>
    <td>F</td>
    <td>wangli@emp.com</td>
    <td>山东</td>
    <td>25</td>
    <td>9000</td>
    <td>1</td>
  </tr>
  <tr>
    <td>3</td>
    <td>王强</td>
    <td>M</td>
    <td>wangqiang@emp.com</td>
    <td>北京</td>
    <td>42</td>
    <td>12000</td>
    <td>2</td>
  </tr>
  <tr>
    <td>4</td>
    <td>何马</td>
    <td>M</td>
    <td>hema@emp.com</td>
    <td>江苏</td>
    <td>32</td>
    <td>8000</td>
    <td>2</td>
  </tr>
  <tr>
    <td>5</td>
    <td>赵四海</td>
    <td>M</td>
    <td>zhaosihai@emp.com</td>
    <td>河北</td>
    <td>32</td>
    <td>17000</td>
    <td>3</td>
  </tr>
  <tr>
    <td>6</td>
    <td>程兴</td>
    <td>M</td>
    <td>chenxing@emp.com</td>
    <td>武汉</td>
    <td>32</td>
    <td>16000</td>
    <td>3</td>
  </tr>
  <tr>
    <td>7</td>
    <td>孙月</td>
    <td>F</td>
    <td>sunyue@emp.com</td>
    <td>山东</td>
    <td>22</td>
    <td>10000</td>
    <td>4</td>
  </tr>
  <tr>
    <td>8</td>
    <td>钱金</td>
    <td>M</td>
    <td>qianjin@emp.com</td>
    <td>北京</td>
    <td>52</td>
    <td>28000</td>
    <td>4</td>
  </tr>
  <tr>
    <td>9</td>
    <td>刘红</td>
    <td>F</td>
    <td>liuhong@emp.com</td>
    <td>安徽</td>
    <td>22</td>
    <td>8000</td>
    <td>5</td>
  </tr>
  <tr>
    <td>10</td>
    <td>蒋一心</td>
    <td>F</td>
    <td>jinagyixin@emp.com</td>
    <td>山西</td>
    <td>28</td>
    <td>9000</td>
    <td>5</td>
  </tr>
  <tr>
    <td>11</td>
    <td>刘红</td>
    <td>F</td>
    <td>liuhong@emp.com</td>
    <td>河南</td>
    <td>22</td>
    <td>8000</td>
    <td>6</td>
  </tr>
  <tr>
    <td>12</td>
    <td>蒋一心</td>
    <td>F</td>
    <td>jinagyixin@emp.com</td>
    <td>NULL</td>
    <td>28</td>
    <td>9000</td>
    <td>6</td>
  </tr>
  <tr>
    <td>13</td>
    <td>马文博</td>
    <td>M</td>
    <td>mawenbo@emp.com</td>
    <td>河南</td>
    <td>22</td>
    <td>28000</td>
    <td>7</td>
  </tr>
  <tr>
    <td>14</td>
    <td>成虎</td>
    <td>M</td>
    <td>chenhu@emp.com</td>
    <td>新疆</td>
    <td>38</td>
    <td>36000</td>
    <td>7</td>
  </tr>
</table>

<h2>Table t_dept</h2>
<table border="1" class="sql_tab">
  <tr>
    <th>dept_id</th>
    <th>dept_name</th>
  </tr>
  <tr>
    <td>1</td>
    <td>人事部</td>
  </tr>
  <tr>
    <td>2</td>
    <td>产品部</td>
  </tr>
  <tr>
    <td>3</td>
    <td>技术部</td>
  </tr>
  <tr>
    <td>4</td>
    <td>运维部</td>
  </tr>
  <tr>
    <td>5</td>
    <td>财务部</td>
  </tr>
  <tr>
    <td>6</td>
    <td>采购部</td>
  </tr>
  <tr>
    <td>7</td>
    <td>研发部</td>
  </tr>
</table>