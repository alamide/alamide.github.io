---
layout: sql
title:  "MySQL 流程控制函数"
date:   2023-02-12
categories: MySQL Functions FlowControl
tags: MySQL Functions 
---
# MySQL 流程控制函数

* IF(expr1, expr2, expr3)

相当于三元表达式 `expr1 ? expr2 : expr3`

需求：编写一个 SQL 查询来交换前五条的 'F' 和 'M' 
```sql
select emp_id, emp_name, if(sex='M', 'F', 'M') as reverse_sex from t_emp where emp_id < 6;
```
<table border="1" class="sql_tab">
<tr><th>emp_id</th><th>emp_name</th><th>reverse_sex</th></tr>
<tr><td>1</td><td>李三</td><td>F</td></tr>
<tr><td>2</td><td>王丽</td><td>M</td></tr>
<tr><td>3</td><td>王强</td><td>F</td></tr>
<tr><td>4</td><td>何马</td><td>F</td></tr>
<tr><td>5</td><td>赵四海</td><td>F</td></tr>
</table>

* IFNULL(expr1, expr2)
  
```sql
select emp_id, emp_name, IFNULL(prov, '中国') as country from t_emp where prov IS NULL;
```
<table border="1" class="sql_tab">
<tr><th>emp_id</th><th>emp_name</th><th>country</th></tr>
<tr><td>12</td><td>蒋一心</td><td>中国</td></tr>
</table>

* CASE
  
`CASE WHEN condition THEN result [WHEN condition THEN result ...] [ELSE result] END`

需求：写出一个SQL 查询语句，计算每个雇员的税收，税收比例如下
1. 5000~10000 10%
2. 10000~20000 15%
3. 20000~50000 20%
4. &gt;50000 50%
   

```sql
select emp_id, emp_name, salary,
      CASE
          WHEN salary <= 10000 THEN salary * 0.1
          WHEN salary <= 20000 THEN (salary - 10000)*0.15 + 10000*0.1
          WHEN salary <= 50000 THEN (salary-20000)*0.2 + 10000*0.15 + 10000*0.1
          ELSE (salary-50000)*0.5 + 30000*0.2 + 10000*0.15 + 10000*0.1
      END
      as tax
from t_emp where emp_id < 5;
```

<table border="1" class="sql_tab">
<tr><th>emp_id</th><th>emp_name</th><th>salary</th><th>tax</th></tr>
<tr><td>1</td><td>李三</td><td>18000</td><td>2200.00</td></tr>
<tr><td>2</td><td>王丽</td><td>9000</td><td>900.00</td></tr>
<tr><td>3</td><td>王强</td><td>12000</td><td>1300.00</td></tr>
<tr><td>4</td><td>何马</td><td>8000</td><td>800.00</td></tr>
</table>

`CASE value WHEN compare_value THEN result [WHEN compare_value THEN result ...] [ELSE result] END`

需求：写出一个SQL 查询语句，M 改为 男，F 显示为 女
```sql
select emp_id, emp_name,
      CASE sex
          WHEN 'M' then '男'
          WHEN 'F' then '女'
          ELSE '未知'
      END
       as "性别"
from t_emp where emp_id < 5;
```

<table border="1" class="sql_tab">
<tr><th>emp_id</th><th>emp_name</th><th>性别</th></tr>
<tr><td>1</td><td>李三</td><td>男</td></tr>
<tr><td>2</td><td>王丽</td><td>女</td></tr>
<tr><td>3</td><td>王强</td><td>男</td></tr>
<tr><td>4</td><td>何马</td><td>男</td></tr>
</table>

* NULLIF(expr1, expr2)

Returns NULL if expr1 = expr2 is true, otherwise returns expr1.

需求：现将采购部裁撤，写出一个SQL 查询语句，查询出员工的部门名称，裁撤的部门显示为NULL
```sql
select emp_id, emp_name, NULLIF(dept_id, 5) as dept_id from t_emp where emp_id > 6;
```

<table border="1" style="border-collapse:collapse">
<tr><th>emp_id</th><th>emp_name</th><th>dept_id</th></tr>
<tr><td>7</td><td>孙月</td><td>4</td></tr>
<tr><td>8</td><td>钱金</td><td>4</td></tr>
<tr><td>9</td><td>刘红</td><td>NULL</td></tr>
<tr><td>10</td><td>蒋一心</td><td>NULL</td></tr>
<tr><td>11</td><td>刘红</td><td>6</td></tr>
<tr><td>12</td><td>蒋一心</td><td>6</td></tr>
<tr><td>13</td><td>马文博</td><td>7</td></tr>
<tr><td>14</td><td>成虎</td><td>7</td></tr>
</table>




