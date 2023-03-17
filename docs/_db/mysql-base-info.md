---
layout: post
title: MySQL 基础信息
categories: db
date: 2023-03-16
excerpt: MySQL 的一些规则、规范、数据类型。
tags: DB MySQL 
---
## 1.一些🍪
* 导入 SQL 文件 `mysql> source [dir]/xx.sql`

* 每条命令以 `;` or `\g` or `\G` 结束

* 列的别名，尽量使用双引号（" "），而且不建议省略as

* 数据库名、表名、表别名、字段名、字段别名等都小写

* SQL 关键字、函数名、绑定变量等都大写

* 开发中使用 DECIMAL

* `DDL(Data Definition Language)` ，如 `CREATE` 、`ALTER` 、`DROP` 、`TRUNCATE`

* `DML(Data Manipulation Language)` ，如 `INSERT` 、`UPDATE` 、`DELETE`

* `DCL(Data Control Language)` ，如常见的授权、取消授权、回滚、提交

* `DQL(Data Query Language)` ，如 `SELECT`

## 2.数据类型
### 2.1 整数类型
* 可选项 `xxINT[(M)] [UNSIGNED] [ZEROFILL]`  `M` 表示显示宽度，不会对插入的数据有任何影响

* `BIT(M) 	1 <= M <= 64  约为(M + 7)/8个字节`
<table border="1" class="sql_tab">
  <tr>
    <th>Type</th>
    <th>Storage (Bytes)</th>
    <th>Minimum Value Signed</th>
    <th>Minimum Value Unsigned</th>
    <th>Maximum Value Signed</th>
    <th>Maximum Value Unsigned</th>
  </tr>
  <tr>
    <td>TINYINT</td>
    <td>1</td>
    <td>-128</td>
    <td>0</td>
    <td>127</td>
    <td>255</td>
  </tr>
  <tr>
    <td>SMALLINT</td>
    <td>2</td>
    <td>-32768</td>
    <td>0</td>
    <td>32767</td>
    <td>65535</td>
  </tr>
  <tr>
    <td>MEDIUMINT</td>
    <td>3</td>
    <td>-8388608</td>
    <td>0</td>
    <td>8388607</td>
    <td>16777215</td>
  </tr>
  <tr>
    <td>INT</td>
    <td>4</td>
    <td>-2147483648</td>
    <td>0</td>
    <td>2147483647</td>
    <td>4294967295</td>
  </tr>
  <tr>
    <td>BIGINT</td>
    <td>8</td>
    <td>-2<sup>63</sup></td>
    <td>0</td>
    <td>2<sup>63</sup>-1</td>
    <td>2<sup>64</sup>-1</td>
  </tr>
</table>

### 2.2 浮点数
* `FLOAT(M,D)` (M,D)中 M=整数位+小数 位，D=小数位。 D<=M<=255，0<=D<=30

* `DOUBLE(M,D)` (M,D)中 M=整数位+小数 位，D=小数位。 D<=M<=255，0<=D<=30

### 2.3 定点数
* `DECIMAL(M,D)`  M+2 字节 `Standard SQL requires that DECIMAL(5,2) be able to store any value with five digits and two decimals, so values that can be stored in the salary column range from -999.99 to 999.99.`

### 2.4 文本
* 设置文本类型时可以设置编码 `c1 VARCHAR(10) CHARACTER SET latin1`
* 可选项 `NATIONAL] CHAR[(M)] [CHARACTER SET charset_name] [COLLATE collation_name]`

#### 2.4.1 CHARACTER AND COLLATE
> `A character set is a set of symbols and encodings. A collation is a set of rules for comparing characters in a character set. Let's make the distinction clear with an example of an imaginary character set.`

> `Suppose that we have an alphabet with four letters: A, B, a, b. We give each letter a number: A = 0, B = 1, a = 2, b = 3. The letter A is a symbol, the number 0 is the encoding for A, and the combination of all four letters and their encodings is a character set.`

> `Suppose that we want to compare two string values, A and B. The simplest way to do this is to look at the encodings: 0 for A and 1 for B. Because 0 is less than 1, we say A is less than B. What we've just done is apply a collation to our character set. The collation is a set of rules (only one rule in this case): “compare the encodings.” We call this simplest of all possible collations a binary collation.`

> `But what if we want to say that the lowercase and uppercase letters are equivalent? Then we would have at least two rules: (1) treat the lowercase letters a and b as equivalent to A and B; (2) then compare the encodings. We call this a case-insensitive collation. It is a little more complex than a binary collation.`

> `In real life, most character sets have many characters: not just A and B but whole alphabets, sometimes multiple alphabets or eastern writing systems with thousands of characters, along with many special symbols and punctuation marks. Also in real life, most collations have many rules, not just for whether to distinguish lettercase, but also for whether to distinguish accents (an “accent” is a mark attached to a character as in German Ö), and for multiple-character mappings (such as the rule that Ö = OE in one of the two German collations).`

指定 `CHARACTER SET` 之后，会有默认的 `COLLATE` 

#### 2.4.2 ENUM
```sql
CREATE TABLE shirts (
    name VARCHAR(40),
    size ENUM('x-small', 'small', 'medium', 'large', 'x-large')
);
```
#### 2.4.3 SET
>`A SET is a string object that can have zero or more values, each of which must be chosen from a list of permitted values specified when the table is created. SET column values that consist of multiple set members are specified with members separated by commas (,). A consequence of this is that SET member values should not themselves contain commas.`

```sql
CREATE TABLE myset (col SET('a', 'b', 'c', 'd'));
INSERT INTO myset (col) VALUES ('a,d'), ('d,a'), ('a,d,a'), ('a,d,d'), ('d,a,d');

SELECT col FROM myset;
```
out:
```shell
+------+
| col  |
+------+
| a,d  |
| a,d  |
| a,d  |
| a,d  |
| a,d  |
+------+
```
### 2.5 时间
#### 2.5.1 数据类型

<table border="1" class="sql_tab">
  <tr>
    <th>类型</th>
    <th>字节</th>
    <th>日期格式</th>
    <th>最小值</th>
    <th>最大值</th>
  </tr>
  <tr>
    <td>YEAR</td>
    <td>1</td>
    <td>YYYY或YY</td>
    <td>1901</td>
    <td>2155</td>
  </tr>
  <tr>
    <td>TIME</td>
    <td>3</td>
    <td>HH:MM:SS</td>
    <td>-838:59:59</td>
    <td>838:59:59</td>
  </tr>
  <tr>
    <td>DATE</td>
    <td>3</td>
    <td>YYYY-MM-DD</td>
    <td>1000-01-01</td>
    <td>9999-12-03</td>
  </tr>
  <tr>
    <td>DATETIME</td>
    <td>8</td>
    <td>YYYY-MM-DD HH:MM:SS</td>
    <td>1000-01-01 00:00:00</td>
    <td>9999-12-31 23:59:59</td>
  </tr>
  <tr>
    <td>TIMESTAMP</td>
    <td>4</td>
    <td>YYYY-MM-DD HH:MM:SS</td>
    <td>1970-01-01 00:00:00 UTC</td>
    <td>2038-01-19 03:14:07 UTC</td>
  </tr>
</table>

#### 2.5.2 TIMESTAMP AND DATETIME

> `MySQL converts TIMESTAMP values from the current time zone to UTC for storage, and back from UTC to the current time zone for retrieval. (This does not occur for other types such as DATETIME.) By default, the current time zone for each connection is the server's time. The time zone can be set on a per-connection basis. As long as the time zone setting remains constant, you get back the same value you store. If you store a TIMESTAMP value, and then change the time zone and retrieve the value, the retrieved value is different from the value you stored. This occurs because the same time zone was not used for conversion in both directions. The current time zone is available as the value of the time_zone system variable. `

```sql
CREATE TABLE t_date (
    id INTEGER PRIMARY KEY AUTO_INCREMENT,
    `year` YEAR,
    `time` TIME,
    `date` DATE,
    `date_time` DATETIME,
    `time_stamp` TIMESTAMP
);

INSERT INTO t_date (id, year, time, date, date_time, time_stamp)
VALUES  (1, 2023, '21:21:03', '2023-03-15', '2023-03-15 21:21:03', '2023-03-15 21:21:03');

SELECT * FROM t_date;
```

<table border="1" class="sql_tab">
  <tr><th>id</th><th>year</th><th>time</th><th>date</th><th>date_time</th><th>time_stamp</th></tr>
  <tr><td>1</td><td>2023</td><td>21:21:03</td><td>2023-03-15</td><td>2023-03-15 21:21:03</td><td>2023-03-15 21:21:03</td></tr>
</table>

修改时区
```sql
SET time_zone = 'US/Eastern'; -- 改变当前 Session 的时区，并不会影响其它 Session (SET GLOBAL time_zone = timezone; 改变全局)
SELECT * FROM t_date;
```
<table border="1" style="border-collapse:collapse">
<tr><th>id</th><th>year</th><th>time</th><th>date</th><th>date_time</th><th>time_stamp</th></tr>
<tr><td>1</td><td>2023</td><td>21:21:03</td><td>2023-03-15</td><td>2023-03-15 21:21:03</td><td>2023-03-15 17:21:03</td></tr>
</table>

