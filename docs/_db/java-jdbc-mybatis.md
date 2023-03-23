---
layout: post
title: MyBatis 使用文档
categories: db
tags: jdbc mybatis
date: 2023-03-20
excerpt: MyBatis 的基本使用方法，及如何整合进工程中。
---

## 1.简单描述
> `MyBatis` 是一款优秀的持久层框架，它支持自定义 `SQL` 、存储过程以及高级映射。`MyBatis` 免除了几乎所有的 `JDBC` 代码以及设置参数和获取结果集的工作。`MyBatis` 可以通过简单的 `XML` 或注解来配置和映射原始类型、接口和 `Java POJO（Plain Old Java Objects，普通老式 Java 对象）` 为数据库中的记录。

* 官方文档地址 [https://mybatis.org/mybatis-3/zh/index.html](https://mybatis.org/mybatis-3/zh/index.html)

* `Mybatis` 是对 `JDBC` 的进一步封装，能够让我们更为方便的操作数据库，提升开发效率


## 2.简单使用 MyBatis
`MyBatis` 可以分为配置文件、映射文件、映射接口，每个部分各司其职。
* 配置（`mybatis-config.xml`），主要是负责全局信息，包括数据库连接信息、映射文件存放位置等

* 映射文件是负责 `SQL` 语句的部分，及结果映射

* 映射接口与映射文件对应，是 `Java` 程序操作的部分

### 2.1 引入 Mybatis

```xml
<dependency>
    <groupId>org.mybatis</groupId>
    <artifactId>mybatis</artifactId>
    <version>3.5.13</version>
</dependency>
```
### 2.2 配置文件
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "https://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <!--外部引入属性-->
    <properties resource="jdbc.properties"/>
    <settings>
        <setting name="mapUnderscoreToCamelCase" value="true"/>
    </settings>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.driver}"/>
                <property name="url" value="${jdbc.url}"/>
                <property name="username" value="${jdbc.username}"/>
                <property name="password" value="${jdbc.password}"/>
            </dataSource>
        </environment>
    </environments>
    <mappers>
        <mapper resource="mappers/DeptMapper.xml"/>
    </mappers>
</configuration>
```

`jdbc.properties`

```
jdbc.driver=com.mysql.cj.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/db_jdbc
jdbc.username=root
jdbc.password=root
```

### 2.3 映射接口
```java
package com.alamide.jdbc.mybatis.mapper;
import com.alamide.jdbc.mybatis.entities.Dept;

public interface DeptMapper {
    public Dept getDept(Integer deptId);
}
```

### 2.4 映射文件
映射文件的地址需要在配置文件中配置，在 `<mappers>` 下，来告诉 `MyBatis` 到哪里去找到这些 `SQL` 语句。

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "https://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.alamide.jdbc.mybatis.mapper.DeptMapper">
    <select id="getDept" resultType="com.alamide.jdbc.mybatis.entities.Dept">
        select * from t_dept where dept_id = #{deptId}
    </select>
</mapper>
```

### 2.5 查询
`Mybatis` 提供了工具类 `Resources` 来读取配置文件，注意在使用 `SqlSession` 时，需要提交事务，否则更新操作无法生效。`openSession` 时传入 `true` 可以自动提交。

```java
private SqlSession getSqlSession(boolean autoCommit) throws IOException {
    final InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
    final SqlSessionFactory build = new SqlSessionFactoryBuilder().build(inputStream);
    inputStream.close();
    return build.openSession(autoCommit);
}

@Test
public void testQuery() throws IOException {
    final SqlSession sqlSession = getSqlSession(true);
    final DeptMapper mapper = sqlSession.getMapper(DeptMapper.class);
    final HashMap dept = mapper.getDept(1);
    log.info(dept.toString());
    sqlSession.close();
}
```

## 3.配置文件
### 3.1 &lt;settings/&gt;
这是 MyBatis 中极为重要的调整设置，它们会改变 MyBatis 的运行时行为。下面记录一些常用到的，
<table border="1">
<tr><th>设置名</th><th>描述</th><th>有效值</th><th>默认值</th></tr>
<tr><td>cacheEnabled</td><td>全局性地开启或关闭所有映射器配置文件中已配置的任何缓存。</td><td>	true | false</td><td>true</td></tr>
<tr><td>lazyLoadingEnabled</td><td>延迟加载的全局开关。当开启时，所有关联对象都会延迟加载。 特定关联关系中可通过设置 fetchType 属性来覆盖该项的开关状态。</td><td>true | false</td><td>false</td></tr>
<tr><td>autoMappingBehavior</td><td>定 MyBatis 应如何自动映射列到字段或属性。 NONE 表示关闭自动映射；PARTIAL 只会自动映射没有定义嵌套结果映射的字段。 FULL 会自动映射任何复杂的结果集（无论是否嵌套）。</td><td>NONE, PARTIAL, FULL</td><td>PARTIAL</td></tr>
<tr><td>mapUnderscoreToCamelCase</td><td>是否开启驼峰命名自动映射，即从经典数据库列名 A_COLUMN 映射到经典 Java 属性名 aColumn。</td><td>true | false</td><td>False</td></tr>
</table>

### 3.2 &lt;mappers/&gt;
映射文件的存放位置，有四种使用方式，常用的有三种
```xml
<!-- 使用相对于类路径的资源引用 -->
<mappers>
  <mapper resource="mappers/DeptMapper.xml"/>
</mappers>

<!-- 这里要 DeptMapper.xml 的存放路径和 DeptMapper.java 的存放路径一致才可以 -->
<mappers>
  <mapper class="com.alamide.jdbc.mybatis.mapper.DeptMapper"/>
</mappers>

<!-- 这里同样要是路径一致才可生效 -->
<mappers>
  <package name="com.alamide.jdbc.mybatis.mapper"/>
</mappers>
```
### 3.3 &lt;typeAliases/&gt; 
类型别名可为 Java 类型设置一个缩写名字。 它仅用于 XML 配置，意在降低冗余的全限定类名书写，`类型别名使用时不区分大小写` 。
```xml
<typeAliases>
  <typeAlias alias="dept" type="com.alamide.jdbc.mybatis.entities.Dept" />
</typeAliases>

<typeAliases>
  <package name="com.alamide.jdbc.mybatis.entities"/>
</typeAliases>
```

`Mybatis` 有一些内建的类型别名，同样是不区分大小写。还可以自定义类型处理器，具体见官方文档。
<table border="1">
<tr><th>别名</th><th>类型映射</th></tr>
<tr><td>int</td><td>Integer</td></tr>
<tr><td>integer</td><td>Integer</td></tr>
<tr><td>map</td><td>Map</td></tr>
<tr><td>hashmap</td><td>HashMap</td></tr>
</table>

### 3.4 &lt;plugins/&gt;
`MyBatis` 允许你在映射语句执行过程中的某一点进行拦截调用。我们在向 `Mybatis` 映射文件中传入参数时，传入的参数名是什么呢？即我们该怎么样获取接口中传入的参数？

1. 不指定参数名(`@Param`)时可以使用 `param1, param2...` 或 `arg0, arg1` 来获取

2. 指定参数名(`@Param`)时可以使用 `指定参数名` 或 `param1, param2...` 来获取

下面通过 `plugin` 来验证一下

配置 `plugin` 插件
```xml
<plugins>
    <plugin interceptor="com.alamide.jdbc.mybatis.LogPlugin"/>
</plugins>
```
`Plugin` 的具体实现，`@Signature` 用来指定需要拦截的方法名，下面的注解即表示拦截 `Executor` 接口中 `query(MappedStatement ms, Object parameter, RowBounds rowBounds, ResultHandler resultHandler)` 方法，可以配置多个
```java
@Slf4j
@Intercepts(@Signature(type = Executor.class, method = "query",
        args = {MappedStatement.class, Object.class, RowBounds.class, ResultHandler.class}))
public class LogPlugin implements Interceptor {
    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        log.info("begin...........");
        final Object proceed = invocation.proceed();
        final Object[] args = invocation.getArgs();
        log.info(Arrays.toString(args));
        final Method method = invocation.getMethod();
        final Object target = invocation.getTarget();
        log.info("method={}, target={}, proceed={}", method.getName(), target.getClass().getCanonicalName(), proceed);
        log.info("end...........");        
        return proceed;
    }
}
```
#### 3.4.1 不指定参数名
即不使用 `@Param` 注解
```java
public interface EmpMapper {
    List<Emp> getEmpByGenderAndDeptId(String gender, Integer deptId);
}
```

```xml
<select id="getEmpByGenderAndDeptId" resultType="Emp">
    select emp_id, emp_name, sex as gender, email, prov, age, salary, dept_id
    from t_emp
    where sex=#{param1} and dept_id=#{param2}
</select>
```
开始测试
```java
@Test
public void testQueryByGenderAndDeptId() throws IOException {
    final InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
    final SqlSessionFactory build = new SqlSessionFactoryBuilder().build(inputStream);
    inputStream.close();

    final SqlSession sqlSession = build.openSession(true);
    final EmpMapper mapper = sqlSession.getMapper(EmpMapper.class);
    final List<Emp> m = mapper.getEmpByGenderAndDeptId("M", 1);

    sqlSession.close();

    log.info(m.toString());
}
```
output:
```
c.a.j.m.LogPlugin - begin...........
c.a.j.m.LogPlugin - [org.apache.ibatis.mapping.MappedStatement@111610e6, {arg1=1, arg0=M, param1=M, param2=1}, org.apache.ibatis.session.RowBounds@4ad4936c, null]
c.a.j.m.LogPlugin - method=query, target=org.apache.ibatis.executor.CachingExecutor, proceed=[Emp(empId=1, empName=李三, gender=M, email=lisan@emp.com, prov=江苏, age=32, salary=18000, deptId=1, dept=null)]
c.a.j.m.LogPlugin - end...........
```
可以看到是以 `arg0, arg1` 和 `param1, param2` 来传入的

#### 3.4.2 指定参数名
下面来试试在接口方法参数加上 `@Param` 注解
```java
public interface EmpMapper {
    List<Emp> getEmpByGenderAndDeptId(@Param("gender") String gender, @Param("deptId") Integer deptId);
}
```

再来测试一下 ouput:
```
c.a.j.m.LogPlugin - begin...........
c.a.j.m.LogPlugin - [org.apache.ibatis.mapping.MappedStatement@111610e6, {gender=M, deptId=1, param1=M, param2=1}, org.apache.ibatis.session.RowBounds@4ad4936c, null]
c.a.j.m.LogPlugin - method=query, target=org.apache.ibatis.executor.CachingExecutor, proceed=[Emp(empId=1, empName=李三, gender=M, email=lisan@emp.com, prov=江苏, age=32, salary=18000, deptId=1, dept=null)]
c.a.j.m.LogPlugin - end...........
```

可以看到这时是按指定的参数名和 `param1, param2` 传入的
### 3.5 &lt;environments/&gt;
`environments` 中 `default` 配合 `environment` 中 `id` 可以快速切换数据库环境，如线上、生产环境

```xml
<environments default="online">
    <environment id="offline">
        <transactionManager type="JDBC"/>
        <dataSource type="POOLED">
            <property name="driver" value="${jdbc.driver}"/>
            <property name="url" value="jdbc:mysql://localhost:3306/db_jdbc"/>
            <property name="username" value="${jdbc.username}"/>
            <property name="password" value="${jdbc.password}"/>
        </dataSource>
    </environment>

    <environment id="online">
        <transactionManager type="JDBC"/>
        <dataSource type="POOLED">
            <property name="driver" value="${jdbc.driver}"/>
            <property name="url" value="jdbc:mysql://localhost:3366/db03"/>
            <property name="username" value="${jdbc.username}"/>
            <property name="password" value="${jdbc.password}"/>
        </dataSource>
    </environment>
</environments>
```

#### 3.5.1 &lt;transactionManager/&gt;
管理数据库事务，可以关闭自动事务提交
```xml
<transactionManager type="JDBC">
  <property name="skipSetAutoCommitOnClose" value="true"/>
</transactionManager>
```
阻止自动关闭连接
```xml
<transactionManager type="MANAGED">
  <property name="closeConnection" value="false"/>
</transactionManager>
```

#### 3.5.2 &lt;dataSource/&gt;
配置数据库相关信息，这里可以配置第三方 `DataSource` 
```java
public class DruidDataSourceFactory extends PooledDataSourceFactory {
    public DruidDataSourceFactory(){
        this.dataSource = new DruidDataSource();
    }
}
```

```xml
<environments default="development">
    <environment id="development">
        <transactionManager type="JDBC"/>
        <dataSource type="com.alamide.jdbc.mybatis.DruidDataSourceFactory">
            <!--这里 driver 要改成 driverClassName，Druid 中使用 driverClassName-->
            <property name="driverClassName" value="${jdbc.driver}"/>
            <property name="url" value="${jdbc.url}"/>
            <property name="username" value="${jdbc.username}"/>
            <property name="password" value="${jdbc.password}"/>
        </dataSource>
    </environment>
</environments>
```

## 4.映射文件
`namespace` 映射接口的全类名
```xml
<mapper namespace="com.alamide.jdbc.mybatis.mapper.DeptMapper">
</mapper>
```

### 4.1 &lt;select/&gt;
`DeptMapper.java`
```java
public interface DeptMapper {
    HashMap getDept(Integer deptId);
}
```

`DeptMapper.xml`
```xml
<select id="getDept" parameterType="int" resultType="hashmap" >
    select * from t_dept where dept_id = #{deptId}
</select>
```
测试代码
```java
@Test
public void testQuery() throws IOException {
    final SqlSession sqlSession = getSqlSession(true);
    final DeptMapper mapper = sqlSession.getMapper(DeptMapper.class);
    final HashMap dept = mapper.getDept(1);
    log.info(dept.toString());
    sqlSession.close();
}
```
常用属性
<table border="1">
  <tr><th>属性</th><th>描述</th></tr>
  <tr><td>id</td><td>命名空间中唯一的标识，与映射接口中方法名对应</td></tr>
  <tr><td>parameterType</td><td>传入参数类型，不常用</td></tr>
  <tr><td>resultType</td><td>返回结果的全限定类名或别名（别名不区分大小写），如果返回的是集合，那么就设置为集合所包含的数据类型</td></tr>
  <tr><td>resultMap</td><td>对外部 resultMap 的命名引用，Mybatis 的强大之处，可以自己处理返回结果</td></tr>
  <tr><td>timeout</td><td>这个设置是在抛出异常之前，驱动程序等待数据库返回请求结果的秒数</td></tr>
</table>

### 4.2 &lt;insert/&gt;
<table border="1">
  <tr><th>属性</th><th>描述</th></tr>
  <tr><td>id</td><td>命名空间中唯一的标识，与映射接口中方法名对应</td></tr>
  <tr><td>parameterType</td><td>传入参数类型，不常用</td></tr>
  <tr><td>useGeneratedKeys</td><td>生成数据库内部生成的主键</td></tr>
  <tr><td>keyProperty</td><td>唯一识别的对象的属性，返回插入数据新生成的主键值时用到</td></tr>
  <tr><td>keyColumn</td><td>设置生成键值在表中的列名，应用场景？</td></tr>
</table>

#### 4.2.1 插入单条数据
`DeptMapper.java`
```java
public interface DeptMapper {
    int saveDept(Dept dept);
}
```

`DeptMapper.xml`
```xml
<insert id="saveDept" useGeneratedKeys="true" keyProperty="deptId">
    insert into t_dept values (null, #{deptName});
</insert>
```

生成的 `key` 会被赋值到 `dept` 中
```java
@Test
public void testInsert() throws IOException {
    final SqlSession sqlSession = getSqlSession(true);
    final DeptMapper mapper = sqlSession.getMapper(DeptMapper.class);
    Dept dept = new Dept(null, "保洁部");
    final int effectedRows = mapper.saveDept(dept);
    log.info("effectedRows = {}, deptId = {}", effectedRows, dept.getDeptId());
    sqlSession.close();
}
```

#### 4.2.2 插入多条数据
`DeptMapper.java`
```java
public interface DeptMapper {
    int saveDeptMulti(@Param("depts") List<Dept> depts);
}
```

`DeptMapper.xml`
```xml
<insert id="saveDeptMulti" useGeneratedKeys="true" keyProperty="deptId">
    insert into t_dept values
    <foreach item="dept" collection="depts" separator=",">
        (null, #{dept.deptName})
    </foreach>
</insert>
```

生成的 `key` 会被赋值到 `dept` 中
```java
@Test
public void testInsertMulti() throws IOException {
    final SqlSession sqlSession = getSqlSession(true);
    final DeptMapper mapper = sqlSession.getMapper(DeptMapper.class);
    List<Dept> depts = new ArrayList<>(5);
    for(int i=0; i<5; i++){
        Dept dept = new Dept(null, "保洁部" + i);
        depts.add(dept);
    }

    final int effectedRows = mapper.saveDeptMulti(depts);
    final String collect = depts
            .stream()
            .map(dept -> String.valueOf(dept.getDeptId()))
            .collect(Collectors.joining(",", "{", "}"));
    log.info("effectedRows = {}, generatedKeys = {}", effectedRows, collect);
    sqlSession.close();
}
```
### 4.3 &lt;update/&gt;
属性和 `<insert/>` 类似

### 4.4 &lt;delete/&gt;
属性和 `<insert/>` 类似

### 4.5 &lt;sql/&gt;
可以用来定义可重用的 SQL 代码片段，以便在其它语句中使用。
`EmpMapper.java`
```java
public interface EmpMapper {
    HashMap getEmpById(Integer empId);
}
```

`EmpMapper.xml`
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "https://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.alamide.jdbc.mybatis.mapper.EmpMapper">
    <sql id="empColumns">
        emp_name, sex, email, age
    </sql>
    <select id="getEmpById" resultType="hashmap">
        select
        <include refid="empColumns"/>
        from t_emp
        where emp_id=#{empId}
    </select>
</mapper>
```
测试代码
```java
@Test
public void testQuery() throws IOException {
    final SqlSession sqlSession = getSqlSession(true);
    final EmpMapper mapper = sqlSession.getMapper(EmpMapper.class);
    final HashMap hashMap = mapper.getEmpById(1);
    log.info(hashMap.toString());
    sqlSession.close();
}
```

### 4.5 参数
* 所有参数传入都必须使用 `@Param`, 强制要求！使用默认的 `arg` 、 `param` 会造成混乱，不利于代码维护。

* `${}` 是直接替换， `#{}` 相当于占位符 `?` ， `select * from t_emp where emp_id>#{empId} order by ${columnName}`
   
#### 4.5.1 参数为 POJO
传入 `POJO` 时，会查找对象中的相关属性。

`DeptMapper.java`
```java
public interface DeptMapper {
    int saveDept(Dept dept);
}
```
`DeptMapper.xml`
```xml
<insert id="saveDept" useGeneratedKeys="true" keyProperty="deptId">
    insert into t_dept values (null, #{deptName});
</insert>
```

传入的参数类型为 `Dept` ，`#{deptName}` 为 `dept.getDeptName()`

#### 4.5.2 参数为基本类型
* 传入一个参数时，在不加 `@Param` 的情况下，`#{}` 中为任意值都可以，`empId` , `param1` , `id` 都可以

```java
public interface EmpMapper {
    Emp getEmpById(Integer empId);
}
```
```xml
<select id="getEmpById" resultType="Emp">
    select * from t_emp where emp_id=#{empId}
</select>
<!--下面效果和上面一样-->
<select id="getEmpById" resultType="Emp">
    select * from t_emp where emp_id=#{param1}
</select>
```


* 传入多个参数时，在不加 `@Param` 的情况下，用 `#{param1}` , `#{param2}` ... 或  `#{arg0}` , `#{arg1}` 来获取传入的参数

```java
public interface EmpMapper {
    List<Emp> getEmpByGenderAndDeptId(String gender, Integer deptId);
}
```
```xml
<select id="getEmpByGenderAndDeptId" resultType="Emp">
    select * from t_emp where sex=#{param1} and dept_id=#{param2}
</select>
<!--下面效果和上面一样-->
<select id="getEmpByGenderAndDeptId" resultType="Emp">
    select * from t_emp where sex=#{arg0} and dept_id=#{arg1}
</select>
```

#### 4.5.3 参数使用 @Param 
对接口中方法的参数添加 `@Param` 时，就可以按照标记的参数名获取参数了，这也是推荐的方式

`EmpMapper.java`
```java
public interface EmpMapper {
    List<Emp> getEmpByGenderAndDeptId(@Param("gender") String gender, @Param("deptId") Integer deptId);
}

```
`EmpMapper.xml`
```xml
<select id="getEmpByGenderAndDeptId" resultType="Emp">
    select * from t_emp where sex=#{gender} and dept_id=#{deptId}
</select>
```

### 4.6 结果映射（重要）
由于这个部分很重要所以单独摘出来，[在这里](./java-jdbc-mybatis-resultmap.html)

### 4.7 动态 SQL
这个特性也很强大，详见[这里](./java-jdbc-mybatis-dynamic-sql.html)

## 5.Java API
### 5.1 注解
在执行一些简单的查询或更新时，去创建一个 `xml`  映射文件会显得有点繁琐，可以直接使用注解
```java
public interface EmpMapper {
    @Select("select * from t_emp where emp_id=#{empId}")
    Emp getEmpById(@Param("empId") Integer empId);
}
```

```java
public interface EmpMapper {
    @Update("update t_emp set emp_name=#{empName} where emp_id=#{empId}")
    int updateEmpNameById(Emp emp);
}
```