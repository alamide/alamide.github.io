---
layout: post
title: JDBC 数据库连接
categories: db
tags: jdbc 
date: 2023-03-17
---
`JDBC(Java Database Connectivity)` 是 `Java` 与 `Database` 之间的桥梁，是 `Java` 官方定制的一系列规范，对应的具体实现由各数据库厂商完成。这种机制可以使我们在切换数据库时，
几乎不需要改变原有代码。各个 数据库框架也是基于 `JDBC` 实现的，如 `MyBatis` 、 `Hibernate` 、 `JPA` 等。
<!--more-->
* 本文使用的数据库数据 [在这里](mysql-base-db-ddl.html)
  
## 1.安装数据库
docker 安装
```shell
docker pull mysql:8.0

docker run -p 3306:3306 --name MySQL8 -e MYSQL_ROOT_PASSWORD=root -d mysql:8.0

docker exec -it MySQL8 /bin/bash
```

创建数据库
```sql
CREATE DATABASE IF NOT EXISTS db_jdbc;
```

## 2.配置 MySQL 驱动
```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.32</version>
</dependency>
```

## 3.简单使用流程
### 3.1 建立连接
* 当加载 `DriverManager.class` 时，会自动装配 `com.mysql.cj.jdbc.Driver` （[自动装配原理](../java/java-jdbc-auto-register.html)），所以不需要自己注册 `mysql` 驱动。

* `url` 有常用可选属性 `serverTimezone=Asia/Shanghai` 、 `useUnicode=true` 、`characterEncoding=utf8` 、 `useSSL=true` 、`rewriteBatchedStatements=true`

```java
//DriverManager.registerDriver(new Driver()); //会导致被注册两次
final Connection connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/db_jdbc", "root", "root");
```

### 3.2.执行简单查询
```java
final Statement statement = connection.createStatement();
String querySQL = "select * from t_dept";
final ResultSet resultSet = statement.executeQuery(querySQL);
```
### 3.3.处理结果集
```java
while (resultSet.next()){
    Integer deptId = resultSet.getInt("dept_id");
    String deptName = resultSet.getString("dept_name");
    log.info("deptId={}, deptName={}", deptId, deptName);
}
```
### 3.4.释放资源
```java
resultSet.close();
statement.close();
connection.close();
```

## 4.Statement
`Statement` 一般只用于无动态值的查询，否则会有发生 `注入攻击` 的危险，可以用 `PreparedStatement` 预编译解决。

```java
public static void testStatement() throws SQLException {
    final Connection connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/db_jdbc", "root", "root");
    final Statement statement = connection.createStatement();
    //注入攻击
    String deptId = "1" + " or 1=1";
    String querySQL = "select * from t_dept where dept_id="+deptId;//会查出数据库中所有数据
    final ResultSet resultSet = statement.executeQuery(querySQL);

    while (resultSet.next()){
        Integer dId = resultSet.getInt("dept_id");
        String deptName = resultSet.getString("dept_name");
        log.info("deptId={}, deptName={}", dId, deptName);
    }

    resultSet.close();
    statement.close();
    connection.close();
}
```

## 5.PreparedStatement
### 5.1 预编译，防止注入式攻击
```java
public static void testPreparedStatement() throws SQLException {
    final Connection connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/db_jdbc", "root", "root");
    String querySQL = "select * from t_dept where dept_id=?";
    String deptId = "1" + " or 1=1";
    final PreparedStatement preparedStatement = connection.prepareStatement(querySQL);

    preparedStatement.setObject(1, deptId);
    final ResultSet resultSet = preparedStatement.executeQuery();
    while (resultSet.next()){
        Integer dId = resultSet.getInt("dept_id");
        String deptName = resultSet.getString("dept_name");
        log.info("deptId={}, deptName={}", dId, deptName);
    }
    resultSet.close();
    preparedStatement.close();
    connection.close();
}
```
开始时很奇怪，按照一开始的理解是不会有满足条件的数据的，但是却查出了 `dept_id=1` 的行，
实际发送到数据库的查询语句为 `select * from t_dept where dept_id='1 or 1=1'` ，应该是 `MySQL` 将 `'1 or 1=1'` 转为 `1` 了，在官方文档中没找到相应的文档，以后看到再补上具体的转换规则。

```sql
SELECT 1 + '10or w', 1 + '1 or 1=1', 1 + 'asss';
```
out:
<table border="1">
  <tr><th>1 + &#39;10or w&#39;</th><th>1 + &#39;1 or 1=1&#39;</th><th>1 + &#39;asss&#39;</th></tr>
  <tr><td>11</td><td>2</td><td>1</td></tr>
</table>

### 5.2 数据插入
#### 5.2.1 普通插入
每次插入一条数据，插入大量数据时，此方法效率极低
```java
@Test
public void testInsert() throws SQLException {
    final Connection connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/db_jdbc", "root", "root");
    String insertSQL = "insert into t_dept (dept_name) values (?)";
    final PreparedStatement preparedStatement = connection.prepareStatement(insertSQL);

    long start = System.currentTimeMillis();
    for (int i = 0; i < 10000; i++) {
        preparedStatement.setObject(1, "公关部" + i);
        preparedStatement.executeUpdate();
    }
    long end = System.currentTimeMillis();

    log.info("cost {} ms", (end - start));//cost 34758 ms
    preparedStatement.close();
    connection.close();
}
```
#### 5.2.2 批量插入
批量插入，大量数据时效率高，对比插入 `10000` 条数据耗时不到普通循环插入的 `1/10` 。注意使用批量删除时需要配置属性 `rewriteBatchedStatements=true` ，否则无效。
```java
@Test
public void testBatchInsert() throws SQLException {
    final Connection connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/db_jdbc?rewriteBatchedStatements=true", "root", "root");
    String insertSQL = "insert into t_dept (dept_name) values (?)";
    final PreparedStatement preparedStatement = connection.prepareStatement(insertSQL);

    long start = System.currentTimeMillis();
    for (int i = 0; i < 10000; i++) {
        preparedStatement.setObject(1, "公关部" + i);
        preparedStatement.addBatch();
    }
    preparedStatement.executeBatch();
    long end = System.currentTimeMillis();
    
    log.info("cost {} ms", (end - start));//cost 217 ms
    preparedStatement.close();
    connection.close();
}
```
### 5.3 数据删除
```java
@Test
public void testDelete() throws SQLException {
    final Connection connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/db_jdbc", "root", "root");
    String deleteSQL = "delete from t_dept where dept_id > ?";
    final PreparedStatement preparedStatement = connection.prepareStatement(deleteSQL);
    preparedStatement.setObject(1, 8);
    final int deleteCount = preparedStatement.executeUpdate();

    log.info("delete {} rows", deleteCount);

    preparedStatement.close();
    connection.close();
}
```
### 5.4 数据更新
```java
@Test
public void testUpdate() throws SQLException {
    final Connection connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/db_jdbc", "root", "root");
    String updateSQL = "update t_dept set dept_name=? where dept_id=?";
    final PreparedStatement preparedStatement = connection.prepareStatement(updateSQL);
    preparedStatement.setObject(1, "公关部");
    preparedStatement.setObject(2, 8);
    final int update = preparedStatement.executeUpdate();

    log.info("effect {} rows", update);

    preparedStatement.close();
    connection.close();
}
```
### 5.5 数据查询
`ResultSetMetaData metaData = resultSet.getMetaData()` 中含有列的信息，配合 `while (resultSet.next())` 可以获取表全部信息。
```java
@Test
public void testQuery() throws SQLException {
    final Connection connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/db_jdbc", "root", "root");
    String querySQL = "select * from t_emp where emp_id < ?";
    final PreparedStatement preparedStatement = connection.prepareStatement(querySQL);
    preparedStatement.setObject(1, 5);
    final ResultSet resultSet = preparedStatement.executeQuery();

    final ResultSetMetaData metaData = resultSet.getMetaData();
    List<Map<String, Object>> items = new ArrayList<Map<String, Object>>();
    while (resultSet.next()){
        HashMap<String, Object> item = new HashMap<String, Object>();
        for(int i=1; i <= metaData.getColumnCount(); i++){
            item.put(metaData.getColumnLabel(i), resultSet.getObject(i));
        }
        items.add(item);
    }

    log.info(items.toString());
    resultSet.close();
    preparedStatement.close();
    connection.close();
}
```