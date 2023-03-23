---
layout: post
title: MyBatis 动态SQL
categories: db
tags: jdbc mybatis resultMap
date: 2023-03-23
---
动态 SQL 是 MyBatis 的强大特性之一。如果你使用过 JDBC 或其它类似的框架，你应该能理解根据不同条件拼接 SQL 语句有多痛苦，例如拼接时要确保不能忘记添加必要的空格，还要注意去掉列表最后一个列名的逗号。利用动态 SQL，可以彻底摆脱这种痛苦。
<!--more-->

## 1.场景引入
有这样一个张表
`t_emp`
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
</table>

对于业务有这样一个需求，要求可以依据多个字段查询员工信息。如果使用原生的 `JDBC` ，那么 `SQL` 语句写起来将十分繁琐。大概可以这么写，`BaseDao.query()` [在这里](./java-jdbc.html) 拉到最后面。 
```java
@Test
public void testQueryEmpByMultiParam() throws Exception {
    Emp params = new Emp();
    params.setAge(20);
    params.setProv("江苏");
    params.setSex("M");

    StringBuilder sqlBuilder = new StringBuilder("select * from t_emp");
    List<Object> args = new ArrayList<>();
    appendParam(params.getEmpId(), "emp_id", args, args.size() > 0, sqlBuilder);
    appendParam(params.getEmpName(), "emp_name", args, args.size() > 0, sqlBuilder);
    appendParam(params.getAge(), "age", args, args.size() > 0, sqlBuilder);
    appendParam(params.getSex(), "sex", args, args.size() > 0, sqlBuilder);
    appendParam(params.getProv(), "prov", args, args.size() > 0, sqlBuilder);
    appendParam(params.getEmail(), "email", args, args.size() > 0, sqlBuilder);
    appendParam(params.getDeptId(), "dept_id", args, args.size() > 0, sqlBuilder);
    appendParam(params.getSalary(), "salary", args, args.size() > 0, sqlBuilder);

    log.info("sql = {}, args = {}", sqlBuilder, args);

    final List<Emp> empList = BaseDao.query(Emp.class, sqlBuilder.toString(), args.toArray());
    log.info(empList.toString());
}

public void appendParam(Object prop, String columnName, List<Object> args, boolean insertedWhere, StringBuilder sqlBuilder) {
    if (prop == null) return;
    args.add(prop);
    sqlBuilder
            .append(insertedWhere ? " and " : " where ")
            .append(columnName)
            .append("=")
            .append("?");
}
```
可以看到代码写起来还是有点繁琐的，而且可拓展性不高，太硬，不灵活。算了再优化一下吧。。。
```java
@Test
public void testQueryEmpByMultiParam2() throws Exception {
    Emp params = new Emp();
    params.setAge(20);
    params.setProv("江苏");
    params.setSex("M");
    
    final List<Emp> empList = queryByMultiParam(params, "t_emp");
    log.info(empList.toString());
}

public <T> List<T> queryByMultiParam(T param, String tableName) throws Exception {
    StringBuilder sqlBuilder = new StringBuilder("select * from ").append(tableName);
    List<Object> args = new ArrayList<>();

    final Field[] fields = param.getClass().getDeclaredFields();
    for (Field field : fields) {
        field.setAccessible(true);
        appendParam(field.get(param), StringUtils.camelToUnderScore(field.getName()), args, args.size() > 0, sqlBuilder);
    }

    log.info("sql = {}, args = {}", sqlBuilder, args);

    return BaseDao.query((Class<T>) param.getClass(), sqlBuilder.toString(), args.toArray());
}

public static String camelToUnderScore(String camel) {
    StringBuilder stringBuilder = new StringBuilder();
    final char[] chars = camel.toCharArray();
    for (Character c : chars) {
        if (Character.isUpperCase(c)) {
            stringBuilder.append("_").append(Character.toLowerCase(c));
        } else {
            stringBuilder.append(c);
        }
    }
    return stringBuilder.toString();
}
```
嗯，稍微好了一点，但是 `SQL` 写在代码里还是很不爽，而且灵活性还是不够，多表连接查询时就又要重写了，当然还可以继续优化，写一个我们自己的 `Mybatis` 。但为什么要重复造轮子呢？下面来用 `Mybatis` 的动态语句来解决需求。

## 2.动态 SQL
使用 `<where/>` 标签，来解决上面的需求
```xml
<select id="getEmpByMultiParam" resultType="emp">
    select * from t_emp 
    <where>
        <if test="empId != null">
            emp_id=#{empId}
        </if>
        <if test="empName != null">
            and emp_name=#{empName}
        </if>
        <if test="gender != null">
            and sex=#{gender}
        </if>
        <if test="age != null">
            and age=#{age}
        </if>
        <if test="deptId != null">
            and dept_id=#{deptId}
        </if>
        <if test="email != null">
            and email=#{email}
        </if>
        <if test="prov != null">
            and prov=#{prov}
        </if>
        <if test="salary != null">
            and salary=#{salary}
        </if>
    </where>
</select>
```
很优雅，`<where/>` 会在有子句的时候才会插入，而且会去掉多余的 `and` 或 `or` 

### 2.1 &lt;if/&gt;
`<if/>` 标签含有属性 `test`，`test` 为真时，会将标签内的内容拼接到 `SQL` 语句中。
```xml
<select id="getEmpByMultiParam" resultType="emp">
    select * from t_emp
    <where>
        <if test="empId != null">
            emp_id=#{empId}
        </if>
    </where>
</select>
```
如果传入的 `empId` 不为空时再将 `emp_id=?` 拼接到 `SQL` 语句中

### 2.2 &lt;where/&gt;
`<where/>` 会在需要的时候才会拼接到 `SQL` 语句中，而且还会去掉多余的 `and` 或 `or` 。

### 2.3 &lt;choose/&gt;
`<choose/>` 会从众多条件中选取一个，和 `Java` 中的 `switch` 语句类似。`choose` 对应 `switch` ， `when` 对应 `case` ， `otherwise` 对应于 `default` ，清晰明了。
```xml
<select id="getEmpByOrderedParam" resultType="emp">
    select * from t_emp
    <where>
        <choose>
            <when test="empId != null">
                emp_id=#{empId}
            </when>
            <when test="empName != null">
                and emp_name=#{empName}
            </when>
            <when test="gender != null">
                and sex=#{gender}
            </when>
            <when test="age != null">
                and age=#{age}
            </when>
            <when test="deptId != null">
                and dept_id=#{deptId}
            </when>
            <when test="email != null">
                and email=#{email}
            </when>
            <when test="prov != null">
                and prov=#{prov}
            </when>
            <otherwise>
                and salary=#{salary}
            </otherwise>
        </choose>
    </where>
</select>
```
会依据固定的顺序来拼接，满足拼接之后就跳出，不再向下判断
### 2.4 &lt;trim/&gt;
用 `<trim/>` 同样可以实现 `<where/>` 的功能
```xml
<select id="getEmpByMultiParam" resultType="emp">
    select * from t_emp
    <trim prefix="where" prefixOverrides="and">
        <if test="empId != null">
            and emp_id=#{empId}
        </if>
        <if test="empName != null">
            and emp_name=#{empName}
        </if>
        <if test="gender != null">
            and sex=#{gender}
        </if>
        <if test="age != null">
            and age=#{age}
        </if>
        <if test="deptId != null">
            and dept_id=#{deptId}
        </if>
        <if test="email != null">
            and email=#{email}
        </if>
        <if test="prov != null">
            and prov=#{prov}
        </if>
        <if test="salary != null">
            and salary=#{salary}
        </if>
    </trim>
</select>
```
上面 `<trim/>` 原理就是字符串替换，如果拼接后 `and sex=? and salary=?` ，那么 `trim` 后会替换为 `where sex=? and salary=?` ，即 `where` replace `and` 。

`<trim/>` 标签还有 `suffix` ，`suffixOverrides` 属性，和 `prefix` 用法类似。

### 2.5 &lt;set/&gt;
和 `<where/>` 类似，`<set/>` 会去掉多余的 `,` 及拼接上 `set`
```xml
<update id="updateEmpById">
    update t_emp
    <set>
        <if test="empName != null">
            emp_name=#{empName},
        </if>
        <if test="gender != null">
            sex=#{gender},
        </if>
        <if test="age != null">
            age=#{age},
        </if>
        <if test="deptId != null">
            dept_id=#{deptId},
        </if>
        <if test="email != null">
            email=#{email},
        </if>
        <if test="prov != null">
            prov=#{prov},
        </if>
        <if test="salary != null">
            salary=#{salary},
        </if>
    </set>
    where emp_id=#{empId}
</update>
```
同样也可以用 `<trim/>` 标签实现同样的功能
```xml
<update id="updateEmpById">
    update t_emp
    <trim prefix="set" suffixOverrides=",">
        <if test="empName != null">
            emp_name=#{empName},
        </if>
        ...
    </trim>
</update>
```

### 2.6 &lt;foreach/&gt;
对集合进行遍历时，需要用到 `<foreach/>` ，例如批量插入数据、构建 `in` 语句时
```xml
<insert id="saveDeptMulti" useGeneratedKeys="true" keyProperty="deptId">
    insert into t_dept values
    <foreach item="dept" collection="depts" separator="," index="index">
        (null, #{dept.deptName})
    </foreach>
</insert>

<select id="getEmpByIds" resultType="emp">
    select * from t_emp
    <where>
        <foreach item="empId" collection="empIds"
                  open="emp_id in (" separator="," close=")" nullable="true">
            #{empId}
        </foreach>
    </where>
</select>
```

1. 属性 `collection` 事传入的集合参数名，

2. 属性 `separator` 会在各语句之间拼接上 `,`

3. 属性 `index` 表示索引

4. 属性 `open` 相当于 `prefix`

5. 属性 `close` 相当于 `subffix`

6. 属性 `nullable` 是否允许传入空值

### 2.7 &lt;script/&gt;
也可以在注解中使用动态 `SQL` ，很简单，[详见官方文档](https://mybatis.org/mybatis-3/zh/dynamic-sql.html#script)

### 2.8 &lt;bind/&gt;
也可以创建变量，并绑定到上下文，[详见官方文档](https://mybatis.org/mybatis-3/zh/dynamic-sql.html#bind)
