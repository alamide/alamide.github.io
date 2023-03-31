---
layout: post
title: MyBatis ResultMap
categories: db
tags: jdbc mybatis resultMap
date: 2023-03-22
---

`resultMap` 元素是 `MyBatis` 中**最重要最强大**的元素。它可以让你从 `90%` 的 `JDBC ResultSets` 数据提取代码中解放出来，并在一些情形下允许你进行一些 `JDBC` 不支持的操作。实际上，在为一些比如连接的复杂语句编写映射代码的时候，一份 `resultMap` 能够代替实现同等功能的数千行代码。`ResultMap` 的设计思想是，对简单的语句做到零配置，对于复杂一点的语句，只需要描述语句之间的关系就行了。
<!--more-->
## 1.相关的 `POJO` , `Table`
`Emp.java` , `Dept.java`
```java
@Data
@AllArgsConstructor
@NoArgsConstructor
@ToString
public class Emp implements Serializable{
    private Integer empId;

    private String empName;

    private String sex;

    private String email;

    private String prov;

    private Integer age;

    private Integer salary;

    private Integer deptId;
}

@Data
@AllArgsConstructor
@NoArgsConstructor
@ToString
public class Dept implements Serializable{
    private Integer deptId;

    private String deptName;
}
```
`t_emp`
<table>
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
</table>

`t_dept`
<table>
  <tr>
    <th>dept_id</th>
    <th>dept_name</th>
  </tr>
</table>

## 2.自动映射
参见文档 [https://mybatis.org/mybatis-3/zh/sqlmap-xml.html#Auto-mapping](https://mybatis.org/mybatis-3/zh/sqlmap-xml.html#Auto-mapping)
>在简单的场景下，MyBatis 可以为你自动映射查询结果。但如果遇到复杂的场景，你需要构建一个结果映射
>当自动映射查询结果时，MyBatis 会获取结果中返回的列名并在 Java 类中查找相同名字的属性（忽略大小写）
>通常数据库列使用大写字母组成的单词命名，单词间用下划线分隔；而 Java 属性一般遵循驼峰命名法约定。为了在这两种命名方式之间启用自动映射，需要将 mapUnderscoreToCamelCase 设置为 true。
>甚至在提供了结果映射后，自动映射也能工作。在这种情况下，对于每一个结果映射，在 ResultSet 出现的列，如果没有设置手动映射，将被自动映射。在自动映射处理完毕后，再处理手动映射。
自动映射可以在配置文件中 `<setting name="autoMappingBehavior">` 来设置，默认为 `PARTIAL`

1. `NONE` 禁用自动映射。仅对手动映射的属性进行映射。

2. `PARTIAL` 对除在内部定义了嵌套结果映射（也就是连接的属性）以外的属性进行映射

3. `FULL` 自动映射所有属性。

要慎用 `FULL` ，无论设置的自动映射等级是哪种，你都可以通过在结果映射上设置 autoMapping 属性来为指定的结果映射设置启用/禁用自动映射。
## 3.简单结果映射 resultType
简单结果是指不含嵌套数据，不含对其它 `POJO` 的引用，类似于 `Emp.java` 
### 3.1 表字段名和 `POJO` 属性名对应
表字段名使用下划线时，可以通过配置文件来转为驼峰型， `<settings>` 中设置 `<setting name="mapUnderscoreToCamelCase" value="true"/>`
```xml
<mapper namespace="com.alamide.jdbc.mybatis.mapper.EmpMapper">
    <select id="getEmpById" resultType="Emp">
        select * from t_emp where emp_id=#{empId}
    </select>
</mapper>
```
### 3.2 表字段名和 POJO 属性名不对应
* 不对应时可以使用表字段别名 `as` ，这种方法不推荐，推荐使用高级映射 `resultMap`

对于 `Emp` 这个 `POJO` ，`sex` 字段开发人员感觉不贴切，用 `private String gender;` 来替代，那么映射文件可以这样修改
```xml
<mapper namespace="com.alamide.jdbc.mybatis.mapper.EmpMapper">
    <select id="getEmpById" resultType="Emp">
        select emp_id, emp_name, sex as gender, email, prov, age, salary, dept_id
        from t_emp
        where emp_id=#{empId}
    </select>
</mapper>
```

## 4.高级结果映射 resultMap
当 `Java Bean` 与表字段不对应或有嵌套数据时，使用 `ResultMap`

### 4.1 对于 3.2 问题的另一种解决方案

```xml
<mapper namespace="com.alamide.jdbc.mybatis.mapper.EmpMapper">
    <!--autoMapping 为 false 时只会给 POJO 映射下面配置的元素，为 true 时，会映射所有的，配置的元素会单独映射-->
    <!--可以通过 <setting name="autoMappingBehavior" value="FULL"/> 来设置映射任何复杂结果，默认为 PARTIAL，只会自动映射没有定义嵌套结果映射的字段-->
    <!--Mybatis 官方建议慎用-->
    <resultMap id="empResultMap" type="Emp" autoMapping="true">
        <result property="gender" column="sex"/>
    </resultMap>
    <select id="getEmpById" resultMap="empResultMap">
        select * from t_emp where emp_id=#{empId}
    </select>
</mapper>
```
### 4.2 &lt;association/&gt;
#### 4.2.1 小例子
现要查找员工的部门信息（一对一），这里就需要使用到关联查询了，`Emp` 中添加属性 `dept`
```java
@Data
@AllArgsConstructor
@NoArgsConstructor
@ToString
public class Emp implements Serializable{
    ...
    private Dept dept;
}
```  
```xml
<mapper namespace="com.alamide.jdbc.mybatis.mapper.EmpMapper">
    <resultMap id="empResultMap" type="Emp" autoMapping="true">
        <result property="gender" column="sex"/>
        <association property="dept" javaType="Dept" autoMapping="true"/>
    </resultMap>
    <select id="getEmpById" resultMap="empResultMap">
        select *
        from t_emp
        left join t_dept on t_emp.dept_id=t_dept.dept_id
        where emp_id=#{empId}
    </select>
</mapper>
```
#### 4.2.2 关联的嵌套 Select 查询
上面的查询还可以用 `association` 的 `select` 属性来替换。相当于执行两次查询，先查出 `Emp` ，再依据查出的 `dept_id` 查出 `Dept` 。
这种方法会产生 `N+1` 问题，效率不佳。还是推荐使用 `SQL` 连接查询语句。
```xml
<resultMap id="empResultMap" type="Emp" autoMapping="true">
    <result property="gender" column="sex"/>
    <!--这里的 column 是传递给 select 的参数值，可以传递多个值，column="{prop1=col1,prop2=col2}"-->
    <association property="dept" javaType="Dept" column="dept_id" select="selectDept"/>
</resultMap>
<select id="getEmpById" resultMap="empResultMap">
    select *
    from t_emp
    where emp_id=#{empId}
</select>
```

### 4.3 &lt;collection/&gt;
#### 4.3.1 小例子
现要查找部门下的员工（一对多），`Dept` 中添加属性 `emps`
```java
@Data
@AllArgsConstructor
@NoArgsConstructor
@ToString
public class Dept implements Serializable{
    private Integer deptId;
    private String deptName;
    private List<Emp> emps;
}
```
```xml
<mapper namespace="com.alamide.jdbc.mybatis.mapper.DeptMapper">
    <resultMap id="deptEmpResultMap" type="Dept">
        <id property="deptId" column="dept_id"/>
        <result property="deptName" column="dept_name"/>
        <collection property="emps" ofType="Emp" autoMapping="true">
            <id property="empId" column="emp_id"/>
            <result property="gender" column="sex"/>
        </collection>
    </resultMap>
    <select id="getEmpOfDept" resultMap="deptEmpResultMap">
        select *
        from t_dept 
        left join t_emp on t_dept.dept_id = t_emp.dept_id
        where t_dept.dept_id = #{deptId}
    </select>
</mapper>
```
也可以这么写，可以使用已有的映射
```xml
<mapper namespace="com.alamide.jdbc.mybatis.mapper.DeptMapper">
    <resultMap id="empResultMap" type="Emp" autoMapping="true">
        <result property="gender" column="sex"/>
    </resultMap>
    <resultMap id="deptEmpResultMap" type="Dept">
        <id property="deptId" column="dept_id"/>
        <result property="deptName" column="dept_name"/>
        <collection property="emps" ofType="Emp" resultMap="empResultMap"/>
    </resultMap>
    <select id="getEmpOfDept" resultMap="deptEmpResultMap">
        select *
        from t_dept left join t_emp
        on t_dept.dept_id = t_emp.dept_id
        where t_dept.dept_id = #{deptId}
    </select>
</mapper>
```
#### 4.3.2 集合的嵌套 Select 查询
同样 `collection` 也有 `select` 属性
```xml
<mapper namespace="com.alamide.jdbc.mybatis.mapper.DeptMapper">
    <resultMap id="empResultMap" type="Emp" autoMapping="true">
        <result property="gender" column="sex"/>
    </resultMap>
    <resultMap id="deptEmpResultMap" type="Dept">
        <id property="deptId" column="dept_id"/>
        <result property="deptName" column="dept_name"/>
        <collection property="emps" column="dept_id" ofType="Emp" javaType="ArrayList" select="getEmp"/>
    </resultMap>
    <select id="getEmp" resultMap="empResultMap">
        select * from t_emp where dept_id=#{deptId}
    </select>
    <select id="getEmpOfDept" resultMap="deptEmpResultMap">
        select *
        from t_dept
        where dept_id = #{deptId}
    </select>
</mapper>
```

## 5.缓存
* 映射语句文件中的所有 select 语句的结果将会被缓存

* 映射语句文件中的所有 insert、update 和 delete 语句会刷新缓存

* 缓存会被视为读/写缓存，这意味着获取到的对象并不是共享的，可以安全地被调用者修改，而不干扰其他调用者或线程所做的潜在修改

* 对二级缓存，单条语句可以使用 `useCache` 来设置是否可以缓存

### 5.1 本地的会话缓存(一级缓存)
默认情况下，只启用了本地的会话缓存，它仅仅对一个会话中的数据进行缓存。
```java
@Test
public void testQueryByGenderAndDeptId() throws IOException {
    final SqlSession sqlSession = getSqlSession(true);
    final EmpMapper mapper = sqlSession.getMapper(EmpMapper.class);
    final List<Emp> m = mapper.getEmpByGenderAndDeptId("M", 1);
    final List<Emp> m2 = mapper.getEmpByGenderAndDeptId("M", 1);
    log.info(m.toString());
    log.info(m2.toString());
    sqlSession.close();
}
```
output:
```
c.a.j.m.m.E.getEmpByGenderAndDeptId - ==>  Preparing: select emp_id, emp_name, sex as gender, email, prov, age, salary, dept_id from t_emp where sex=? and dept_id=?
c.a.j.m.m.E.getEmpByGenderAndDeptId - ==> Parameters: M(String), 1(Integer)
c.a.j.m.m.E.getEmpByGenderAndDeptId - <==    Columns: emp_id, emp_name, gender, email, prov, age, salary, dept_id
c.a.j.m.m.E.getEmpByGenderAndDeptId - <==        Row: 1, 李三, M, lisan@emp.com, 江苏, 32, 18000, 1
c.a.j.m.m.E.getEmpByGenderAndDeptId - <==      Total: 1
c.a.j.EmpMyBatisTest - [Emp(empId=1, empName=李三, gender=M, email=lisan@emp.com, prov=江苏, age=32, salary=18000, deptId=1, dept=null)]
c.a.j.EmpMyBatisTest - [Emp(empId=1, empName=李三, gender=M, email=lisan@emp.com, prov=江苏, age=32, salary=18000, deptId=1, dept=null)]
```
可以看到只执行了一次查询

现在稍微修改一下代码
```java
@Test
public void testQueryByGenderAndDeptId() throws IOException {
    final InputStream inputStream = Resources.getResourceAsStream("mybatis-config.xml");
    final SqlSessionFactory build = new SqlSessionFactoryBuilder().build(inputStream);
    inputStream.close();

    final SqlSession sqlSession = build.openSession(true);
    final EmpMapper mapper = sqlSession.getMapper(EmpMapper.class);
    final List<Emp> m = mapper.getEmpByGenderAndDeptId("M", 1);
    sqlSession.commit();
    sqlSession.close();

    final SqlSession sqlSession2 = build.openSession(true);
    final EmpMapper mapper2 = sqlSession2.getMapper(EmpMapper.class);
    final List<Emp> m2 = mapper2.getEmpByGenderAndDeptId("M", 1);
    sqlSession2.commit();
    sqlSession2.close();

    log.info(m.toString());
    log.info(m2.toString());
}
```
output:
```
c.a.j.m.m.E.getEmpByGenderAndDeptId - ==>  Preparing: select emp_id, emp_name, sex as gender, email, prov, age, salary, dept_id from t_emp where sex=? and dept_id=?
c.a.j.m.m.E.getEmpByGenderAndDeptId - ==> Parameters: M(String), 1(Integer)
c.a.j.m.m.E.getEmpByGenderAndDeptId - <==    Columns: emp_id, emp_name, gender, email, prov, age, salary, dept_id
c.a.j.m.m.E.getEmpByGenderAndDeptId - <==        Row: 1, 李三, M, lisan@emp.com, 江苏, 32, 18000, 1
c.a.j.m.m.E.getEmpByGenderAndDeptId - <==      Total: 1
c.a.j.m.m.E.getEmpByGenderAndDeptId - ==>  Preparing: select emp_id, emp_name, sex as gender, email, prov, age, salary, dept_id from t_emp where sex=? and dept_id=?
c.a.j.m.m.E.getEmpByGenderAndDeptId - ==> Parameters: M(String), 1(Integer)
c.a.j.m.m.E.getEmpByGenderAndDeptId - <==    Columns: emp_id, emp_name, gender, email, prov, age, salary, dept_id
c.a.j.m.m.E.getEmpByGenderAndDeptId - <==        Row: 1, 李三, M, lisan@emp.com, 江苏, 32, 18000, 1
c.a.j.m.m.E.getEmpByGenderAndDeptId - <==      Total: 1
c.a.j.EmpMyBatisTest - [Emp(empId=1, empName=李三, gender=M, email=lisan@emp.com, prov=江苏, age=32, salary=18000, deptId=1, dept=null)]
c.a.j.EmpMyBatisTest - [Emp(empId=1, empName=李三, gender=M, email=lisan@emp.com, prov=江苏, age=32, salary=18000, deptId=1, dept=null)]
```
可以看到缓存失效，执行了两次查询
### 5.2 全局二级缓存
只需要在 `SQL` 映射文件中添加一行 `<cache/>` 
* 注意 `SqlSession` 是要从同一个 `SqlSessionFactory` 打开的，否则缓存无效，自己踩的一个小坑。

* `POJO` 要实现 `Serializable` 接口，要可以序列化

* 单条语句可以是用 `useCache` 来决定是否可以缓存

```xml
<mapper namespace="com.alamide.jdbc.mybatis.mapper.EmpMapper">
    <cache/>
    <!--<select id="getEmpByGenderAndDeptId" resultType="Emp" useCache="false"> 可以禁用缓存-->
    <select id="getEmpByGenderAndDeptId" resultType="Emp">
        select emp_id, emp_name, sex as gender, email, prov, age, salary, dept_id
        from t_emp
        where sex=#{gender} and dept_id=#{deptId}
    </select>
</mapper>
```

再来执行一下上面的 `java` 方法 `testQueryByGenderAndDeptId()` output:
```
c.a.j.m.m.EmpMapper - Cache Hit Ratio [com.alamide.jdbc.mybatis.mapper.EmpMapper]: 0.0
c.a.j.m.m.E.getEmpByGenderAndDeptId - ==>  Preparing: select emp_id, emp_name, sex as gender, email, prov, age, salary, dept_id from t_emp where sex=? and dept_id=?
c.a.j.m.m.E.getEmpByGenderAndDeptId - ==> Parameters: M(String), 1(Integer)
c.a.j.m.m.E.getEmpByGenderAndDeptId - <==    Columns: emp_id, emp_name, gender, email, prov, age, salary, dept_id
c.a.j.m.m.E.getEmpByGenderAndDeptId - <==        Row: 1, 李三, M, lisan@emp.com, 江苏, 32, 18000, 1
c.a.j.m.m.E.getEmpByGenderAndDeptId - <==      Total: 1
c.a.j.m.m.EmpMapper - Cache Hit Ratio [com.alamide.jdbc.mybatis.mapper.EmpMapper]: 0.5
c.a.j.EmpMyBatisTest - [Emp(empId=1, empName=李三, gender=M, email=lisan@emp.com, prov=江苏, age=32, salary=18000, deptId=1, dept=null)]
c.a.j.EmpMyBatisTest - [Emp(empId=1, empName=李三, gender=M, email=lisan@emp.com, prov=江苏, age=32, salary=18000, deptId=1, dept=null)]
```
可以看到只执行了一次数据库连接

## 6.懒加载
懒加载即只执行必要的 `SQL` 语句

当有嵌套 `Select` 查询时，可以使用懒加载来让嵌套的 `select` 语句，只在需要时被执行，开启方法如下
* 配置文件中 `<settings>` `<setting name="lazyLoadingEnabled" value="true"/>`

* `<association/>` ，`<collection/>` 中使用属性 `fetchType="lazy"` 来开启，这里开启后将在映射中忽略全局配置参数 `lazyLoadingEnabled`，使用属性的值

* `fetchType` 有效值为 `lazy` (懒加载)，`eager` (立即生效)，默认为 `eager`

有查询如下
```xml
<resultMap id="empResultMap" type="Emp" autoMapping="true">
    <result property="gender" column="sex"/>
    <association property="dept" javaType="Dept" column="dept_id" select="selectDept" fetchType="lazy"/>
</resultMap>
<select id="getEmpById" resultMap="empResultMap">
    select *
    from t_emp
    where emp_id=#{empId}
</select>
```
获取指定员工的姓名
```java
@Test
public void testQueryEmpOfDept() throws IOException {
    final SqlSession sqlSession = getSqlSession(true);
    final DeptMapper mapper = sqlSession.getMapper(DeptMapper.class);
    final Dept dept = mapper.getEmpOfDept(2);
    log.info(dept.toString());
    sqlSession.close();
}
```
output:
```
c.a.j.m.m.E.getEmpById - ==>  Preparing: select * from t_emp where emp_id=?
c.a.j.m.m.E.getEmpById - ==> Parameters: 1(Integer)
c.a.j.m.m.E.getEmpById - <==    Columns: emp_id, emp_name, sex, email, prov, age, salary, dept_id
c.a.j.m.m.E.getEmpById - <==        Row: 1, 李三, M, lisan@emp.com, 江苏, 32, 18000, 1
c.a.j.m.m.E.getEmpById - <==      Total: 1
c.a.j.EmpMyBatisTest - 李三
```
可以看到并没有执行查询部门信息的语句，懒加载生效

再来测试下获取员工部门信息
```java
@Test
public void testQuery() throws IOException {
    final SqlSession sqlSession = getSqlSession(true);
    final EmpMapper mapper = sqlSession.getMapper(EmpMapper.class);
    final Emp emp = mapper.getEmpById(1);
    log.info(emp.getDept().getDeptName());
    sqlSession.close();
}
```
output:
```
c.a.j.m.m.E.getEmpById - ==>  Preparing: select * from t_emp where emp_id=?
c.a.j.m.m.E.getEmpById - ==> Parameters: 1(Integer)
c.a.j.m.m.E.getEmpById - <==    Columns: emp_id, emp_name, sex, email, prov, age, salary, dept_id
c.a.j.m.m.E.getEmpById - <==        Row: 1, 李三, M, lisan@emp.com, 江苏, 32, 18000, 1
c.a.j.m.m.E.getEmpById - <==      Total: 1
c.a.j.m.m.E.selectDept - ==>  Preparing: select * from t_dept where dept_id=?
c.a.j.m.m.E.selectDept - ==> Parameters: 1(Integer)
c.a.j.m.m.E.selectDept - <==    Columns: dept_id, dept_name
c.a.j.m.m.E.selectDept - <==        Row: 1, 编辑部
c.a.j.m.m.E.selectDept - <==      Total: 1
c.a.j.EmpMyBatisTest - 编辑部
```
由于需要获取部门信息，所以执行 `select * from t_dept where dept_id=?`