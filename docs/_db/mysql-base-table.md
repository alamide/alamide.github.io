---
layout: default
title: MySQL-Base-Modify
categories: db
tags: DB MySQL 
excerpt: MySQL 修改相关的基础知识，修改数据库、表、列等。
date: 2023-03-16
---

### 1.数据类型
#### 1.1 整数类型

<table>
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
