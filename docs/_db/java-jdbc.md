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
#### 5.2.3 插入数据返回自增长主键
有些时候我们需要获取插入数据自动生成的主键值
```java
@Test
public void testReturnAutoIncrementPrimaryKey() throws SQLException {
    final Connection connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/db_jdbc?rewriteBatchedStatements=true", "root", "root");
    String insertSQL = "insert into t_dept (dept_name) values (?)";
    final PreparedStatement preparedStatement = connection.prepareStatement(insertSQL, Statement.RETURN_GENERATED_KEYS);
    preparedStatement.setObject(1, "法务部");
    int update = preparedStatement.executeUpdate();

    final ResultSet generatedKeys = preparedStatement.getGeneratedKeys();
    if(generatedKeys.next()){
        log.info("generated primary key = {}", generatedKeys.getInt(1));
    }

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

## 6.事务
> 事务就是一组原子性的SQL查询，或者说一个独立的工作单元。如果数据库引擎能够成功地对数据库应用该组查询的全部语句，那么就执行该组查询。如中有任何一条语句因为崩溃或其他原因无法执行，那么所有语句都不执行。也就是说，事务内的语句，要么全部执行，要么执行失败。（《高性能mysql》）

* 简单理解就是要么都执行，要么都不执行。MySQL 默认是开启自动提交事务的，可以调用 `SET autocommit=off; COMMIT/ROLLBACK;` 来关闭，`JDBC` 中使用 `connection.setAutoCommit(false);` ， `connection.commit();` ，`connection.rollback();`

* 事务的开启需要与提交，要是同一个 `Connection` 。
### 6.1 未开启事务可能引发的问题
数据库如下，`balance` 类型为 `INTEGER UNSIGNED`
<table border="1">
<tr><th>id</th><th>account</th><th>balance</th></tr>
<tr><td>1</td><td>8859-1</td><td>2000</td></tr>
<tr><td>2</td><td>8859-2</td><td>2000</td></tr>
</table>

```java
@Test
public void testAutoCommit() throws SQLException {
    final Connection con = DriverManager.getConnection("jdbc:mysql://localhost:3306/db_jdbc", "root", "root");
    addMoney(con, "8859-1", 2500);
    subMoney(con, "8859-2", 2500);
    con.close();
}

public void addMoney(Connection con, String account, int money) throws SQLException {
    final PreparedStatement preparedStatement = con.prepareStatement("update t_account set balance=balance+? where account=?");
    preparedStatement.setObject(1, money);
    preparedStatement.setObject(2, account);
    preparedStatement.executeUpdate();
    preparedStatement.close();
}

public void subMoney(Connection con, String account, int money) throws SQLException {
    final PreparedStatement preparedStatement = con.prepareStatement("update t_account set balance=balance-? where account=?");
    preparedStatement.setObject(1, money);
    preparedStatement.setObject(2, account);
    preparedStatement.executeUpdate();
    preparedStatement.close();
}
```
可以看到程序出错终止，因为 `balance` 为 `INTEGER UNSIGNED` 。而数据库已经被修改为
<table border="1">
<tr><th>id</th><th>account</th><th>balance</th></tr>
<tr><td>1</td><td>8859-1</td><td>4500</td></tr>
<tr><td>2</td><td>8859-2</td><td>2000</td></tr>
</table>
这是严重的错误

### 6.2 开启事务
开启事务之后会解决上述问题
```java
@Test
public void testCommit() throws SQLException {
    Connection con = DriverManager.getConnection("jdbc:mysql://localhost:3306/db_jdbc", "root", "root");
    try {
        con.setAutoCommit(false);
        addMoney(con, "8859-1", 2500);
        subMoney(con, "8859-2", 2500);
        con.commit();
    } catch (Exception exception) {
        con.rollback();
    }finally {
        con.close();
    }
}
```

## 7.数据库连接池
* 数据库连接的创建和关闭会消耗大量资源，所以应该尽可能的复用数据库连接，数据库连接池就是这样一种技术，可以重复使用已创建的数据库连接。

* Java 官方定制了 `javax.sql.DataSource` 接口规范，市场上有多种实现，有 `C3P0` 、`DBCP` 、`Hikari` 、`Druid` 等。其中 `Hikari` 性能较优， 阿里的 `Druid` 较为全面。这里我们使用阿里的 `Druid` 。

* `Druid` [https://github.com/alibaba/druid](https://github.com/alibaba/druid)

### 7.1 引入 Druid
```xml
<dependency>
  <groupId>com.alibaba</groupId>
  <artifactId>druid</artifactId>
  <version>1.2.16</version>
</dependency>
```
### 7.2 使用 Druid
#### 7.2.1 硬编码
```java
@Test
public void testHardDruid() throws SQLException {
    final DruidDataSource dataSource = new DruidDataSource();
    
    dataSource.setUrl("jdbc:mysql://localhost:3306/db_jdbc");
    dataSource.setUsername("root");
    dataSource.setPassword("root");
    dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");

    final Connection connection = dataSource.getConnection();
    connection.close();
}
```
#### 7.2.2 软编码（推荐）
jdbc.properties

```
driverClassName=com.mysql.cj.jdbc.Driver
url=jdbc:mysql://localhost:3306/db_jdbc
username=root
password=root
```

```java
@Test
public void testSoftDruid() throws Exception {

    Properties properties = new Properties();
    final InputStream inputStream = this.getClass().getClassLoader().getResourceAsStream("jdbc.properties");
    properties.load(inputStream);
    final DataSource dataSource = DruidDataSourceFactory.createDataSource(properties);
    final Connection connection = dataSource.getConnection();

    inputStream.close();
    connection.close();
}
```
## 8.自定义工具类
所有对数据库的操作可以分为两类，一种是查询，另一种是更新。
* 查询返回的是 `List<T>` ，一条数据 `List` 的 `size` 即为 `1`，无数据 `size` 为 `0` ，类型 `T` 的实例对象可以依据 `MetaData` 来创建。

* 更新包括插入、删除、更新，返回的是影响数据的行数，或返回自增长的主键值。可以用一个对象封装

### 8.1 初步
```java
public  abstract class BaseDao {
    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public static class UpdateInfo {
        private Integer effectedRows;
        private Integer autoIncrementPrimaryKey;
    }

    protected <T> T queryForBean(Connection con, Class<T> clazz, String sql, Object... args) throws Exception {
        final PreparedStatement preparedStatement = con.prepareStatement(sql);
        final List<T> queryList = query(con, clazz, sql, args);

        if (queryList.size() > 0) {
            return queryList.get(0);
        }

        return null;
    }

    protected <T> List<T> query(Connection con, Class<T> clazz, String sql, Object... args) throws Exception {
        final PreparedStatement preparedStatement = con.prepareStatement(sql);
        for (int i = 1; i <= args.length; i++) {
            preparedStatement.setObject(i, args[i - 1]);
        }
        final ResultSet resultSet = preparedStatement.executeQuery();
        final ResultSetMetaData metaData = resultSet.getMetaData();

        List<T> queryList = new ArrayList<>();
        while (resultSet.next()) {
            T item = clazz.newInstance();
            for (int i = 1; i <= metaData.getColumnCount(); i++) {
                final Field declaredField = clazz.getDeclaredField(StringUtils.underScoreToCamel(metaData.getColumnLabel(i)));
                declaredField.setAccessible(true);
                declaredField.set(item, resultSet.getObject(i));
            }
            queryList.add(item);
        }

        return queryList;
    }

    protected int update(Connection con, String sql, Object... args) throws SQLException {
        final PreparedStatement preparedStatement = con.prepareStatement(sql);
        for (int i = 1; i <= args.length; i++) {
            preparedStatement.setObject(i, args[i - 1]);
        }
        return preparedStatement.executeUpdate();
    }

    protected UpdateInfo updateForGeneratedKey(Connection con, String sql, Object... args) throws SQLException {
        final PreparedStatement preparedStatement = con.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS);
        for (int i = 1; i <= args.length; i++) {
            preparedStatement.setObject(i, args[i - 1]);
        }

        UpdateInfo updateInfo = new UpdateInfo();
        int effectedRows = preparedStatement.executeUpdate();
        updateInfo.setEffectedRows(effectedRows);
        final ResultSet generatedKeys = preparedStatement.getGeneratedKeys();
        if (generatedKeys.next()) {
            updateInfo.setAutoIncrementPrimaryKey(generatedKeys.getInt(1));
        }

        return updateInfo;
    }
}
```
`StringUtils` 临时写的，可能有 `Bug` 
```java
public class StringUtils {
    public static String underScoreToCamel(String underScore) {
        if (underScore == null || !underScore.contains("_")) {
            return underScore;
        }

        StringBuilder stringBuilder = new StringBuilder();
        char[] charArray = underScore.toCharArray();
        for (int i = 0; i < underScore.length(); ) {
            if (charArray[i] == '_') {
                if (i + 1 < underScore.length()) {
                    if (charArray[i + 1] == '_') {
                        i += 1;
                    } else {
                        stringBuilder.append(Character.toUpperCase(charArray[i + 1]));
                        i += 2;
                    }
                } else {
                    i += 1;
                }
            } else {
                stringBuilder.append(charArray[i]);
                i += 1;
            }

        }

        return stringBuilder.toString();
    }
}
```

### 8.2 优化
不传入 `Connection` 参数，因为一次事务操作肯定是在同一个线程内，所以可以采用 `ThreadLocal` 来保存 `Connection` 对象。
`JDBCUtils`
```java
public class JDBCUtils {
    private static final ThreadLocal<Connection> threadLocal = new ThreadLocal<>();
    private static DataSource dataSource = null;

    static {
        Properties properties = new Properties();
        final InputStream inputStream = JDBCUtils.class.getClassLoader().getResourceAsStream("jdbc.properties");
        try {
            properties.load(inputStream);
            dataSource = DruidDataSourceFactory.createDataSource(properties);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }

    public static Connection getConnection() throws Exception {
        Connection con = threadLocal.get();
        if (con == null) {
            con = dataSource.getConnection();
            threadLocal.set(con);
        }
        return con;
    }

    public static void freeConnection() throws SQLException {
        Connection connection = threadLocal.get();
        if(connection != null){
            threadLocal.remove();
            connection.setAutoCommit(true);
            connection.close();
        }
    }
}
```
`BaseDao`
```java
public abstract class BaseDao {

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public static class UpdateInfo {
        private Integer effectedRows;
        private Integer autoIncrementPrimaryKey;
    }

    protected <T> T queryForBean(Class<T> clazz, String sql, Object... args) throws Exception {
        Connection con = JDBCUtils.getConnection();
        final PreparedStatement preparedStatement = con.prepareStatement(sql);
        final List<T> queryList = query(clazz, sql, args);

        if (queryList.size() > 0) {
            return queryList.get(0);
        }

        return null;
    }

    protected <T> List<T> query(Class<T> clazz, String sql, Object... args) throws Exception {
        Connection con = JDBCUtils.getConnection();
        final PreparedStatement preparedStatement = con.prepareStatement(sql);
        for (int i = 1; i <= args.length; i++) {
            preparedStatement.setObject(i, args[i - 1]);
        }
        final ResultSet resultSet = preparedStatement.executeQuery();
        final ResultSetMetaData metaData = resultSet.getMetaData();

        List<T> queryList = new ArrayList<>();
        while (resultSet.next()) {
            T item = clazz.newInstance();
            for (int i = 1; i <= metaData.getColumnCount(); i++) {
                final Field declaredField = clazz.getDeclaredField(StringUtils.underScoreToCamel(metaData.getColumnLabel(i)));
                declaredField.setAccessible(true);
                declaredField.set(item, resultSet.getObject(i));
            }
            queryList.add(item);
        }

        return queryList;
    }

    protected int update(String sql, Object... args) throws Exception {
        Connection con = JDBCUtils.getConnection();
        final PreparedStatement preparedStatement = con.prepareStatement(sql);
        for (int i = 1; i <= args.length; i++) {
            preparedStatement.setObject(i, args[i - 1]);
        }
        return preparedStatement.executeUpdate();
    }

    protected UpdateInfo updateForGeneratedKey(String sql, Object... args) throws Exception {
        Connection con = JDBCUtils.getConnection();
        final PreparedStatement preparedStatement = con.prepareStatement(sql, Statement.RETURN_GENERATED_KEYS);
        for (int i = 1; i <= args.length; i++) {
            preparedStatement.setObject(i, args[i - 1]);
        }

        UpdateInfo updateInfo = new UpdateInfo();
        int effectedRows = preparedStatement.executeUpdate();
        updateInfo.setEffectedRows(effectedRows);
        final ResultSet generatedKeys = preparedStatement.getGeneratedKeys();
        if (generatedKeys.next()) {
            updateInfo.setAutoIncrementPrimaryKey(generatedKeys.getInt(1));
        }

        return updateInfo;
    }
}
```