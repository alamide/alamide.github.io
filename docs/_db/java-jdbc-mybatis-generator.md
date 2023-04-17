---
layout: post
title: MyBatis Generator
categories: db
tags: Mybatis
date: 2023-04-02
isHidden: true
excerpt: Mybatis 逆向生成工具，可以依据数据库中表结构自动生成 POJO、*Mapper.xml、*Mapper.java
---

* 官方文档在[这里](http://mybatis.org/generator/quickstart.html)

## 1. 引入插件
```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.mybatis.generator</groupId>
            <artifactId>mybatis-generator-maven-plugin</artifactId>
            <version>1.4.2</version>
            <dependencies>
                <dependency>
                    <groupId>org.mybatis</groupId>
                    <artifactId>mybatis</artifactId>
                    <version>3.5.13</version>
                </dependency>
                <dependency>
                    <groupId>mysql</groupId>
                    <artifactId>mysql-connector-java</artifactId>
                    <version>8.0.32</version>
                </dependency>
            </dependencies>
        </plugin>
    </plugins>
</build>
...
</project>
```
## 2. 配置文件
`generatorConfig.xml`
```xml
<!DOCTYPE generatorConfiguration PUBLIC
        "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>
    <context id="simple" targetRuntime="MyBatis3">
        <commentGenerator>
            <property name="suppressAllComments" value="true"/>
        </commentGenerator>
        <jdbcConnection driverClass="com.mysql.cj.jdbc.Driver"
                        connectionURL="jdbc:mysql://localhost:3306/mybatis_generator"
                        password="root"
                        userId="root"/>

        <javaTypeResolver>
            <property name="useJSR310Types" value="true"/>
            <property name="forceBigDecimals" value="true"/>
        </javaTypeResolver>

        <javaModelGenerator targetPackage="com.alamide.guli.entity" targetProject="src/main/java">
            <property name="exampleTargetPackage" value="com.alamide.generator.entity.example"/>
        </javaModelGenerator>

        <sqlMapGenerator targetPackage="mapper" targetProject="src/main/resources"/>

        <javaClientGenerator type="XMLMAPPER" targetPackage="com.alamide.generator.mapper" targetProject="src/main/java"/>

        <table tableName="edu_teacher"/>
    </context>
</generatorConfiguration>
```
## 3.配置文件属性
### 3.1 &lt;jdbcConnection/&gt;
配置 `JDBC Connection`
```xml
<jdbcConnection driverClass="com.mysql.cj.jdbc.Driver"
                connectionURL="jdbc:mysql://localhost:3306/mybatis_generator"
                password="root"
                userId="root"/>
``` 
### 3.2 &lt;commentGenerator/&gt;
注释相关，不太重要
<table>
  <tr>
    <th>Property Name</th>
    <th>Property Values</th>
  </tr>
  <tr>
    <td>suppressAllComments</td>
    <td>false or true 默认false，是否不写注释</td>
  </tr>
  <tr>
    <td>suppressDate</td>
    <td>false or true 默认 false，在注释中是否添加时间戳</td>
  </tr>
  <tr>
    <td>addRemarkComments</td>
    <td>false or true 默认 false，是否在注释中加上数据库的注释</td>
  </tr>
  <tr>
    <td>dateFormat</td>
    <td>注释中 日期的格式 yyyy-MM-dd HH:mm:ss</td>
  </tr>
</table>

```xml
<commentGenerator>
    <property name="suppressAllComments" value="true"/>
</commentGenerator>
```

### 3.3 &lt;javaModelGenerator/&gt;
`Java Model` 相关，具体的配置如下

必须属性
<table>
  <tr>
    <th>Attribute</th>
    <th>Description</th>
  </tr>
  <tr>
    <td>targetPackage</td>
    <td>生的 Java Model 包目录</td>
  </tr>
  <tr>
    <td>targetProject</td>
    <td>目标工程位置，如：src/main/java</td>
  </tr>
</table>

可选属性
<table>
  <tr>
    <th>Property Name</th>
    <th>Property Values</th>
  </tr>
  <tr>
    <td>constructorBased</td>
    <td>false or true 默认false，是否生成有参，无参构造器</td>
  </tr>
  <tr>
    <td>rootClass</td>
    <td>父类全类名，如配置最终 extends 这个类，不能实现接口，如 implements Serializable</td>
  </tr>
</table>

### 3.4 &lt;sqlMapGenerator/&gt;
`Mybatis` 映射文件配置
<table>
  <tr>
    <th>Attribute</th>
    <th>Description</th>
  </tr>
  <tr>
    <td>targetPackage</td>
    <td>生的包目录，如 mapper</td>
  </tr>
  <tr>
    <td>targetProject</td>
    <td>目标工程位置，src/main/resources</td>
  </tr>
</table>

### 3.5 &lt;javaClientGenerator/&gt;
`Mybatis Mapper` 接口
```xml
<javaClientGenerator type="XMLMAPPER" targetPackage="com.alamide.generator.mapper" targetProject="src/main/java"/>
``` 

### 3.6 &lt;table/&gt;
* `tableName` 表名

* `domainObjectName` 生成的 `Java Model` 类名，不配置则依据表名生成

#### 3.6.1 &lt;columnOverride&gt; 
可以在生成 `Java Model` 的时候，对字段进行一些转换，例如数据库的有一些表中，有 `is_deleted` 软删除字段，类型为 `TINYINT` 。默认会被转换为 `Byte` 型，
在使用的时候会有一些不便，希望转换为 `Integer` 型
```xml
<table tableName="t_example">
    <columnOverride column="is_deleted" jdbcType="TINYINT" javaType="java.lang.Integer"/>
</table>
```

### 3.7 &lt;javaTypeResolver/&gt;
类型转换器

<table>
  <tr>
    <th>Property Name</th>
    <th>Property Values</th>
  </tr>
  <tr>
    <td>forceBigDecimals</td>
    <td>默认 false，把 JDBC DECIMAL 和 NUMERIC 类型解析为 Integer，为 true 时把 JDBC DECIMAL和 NUMERIC 类型解析为 java.math.BigDecimal</td>
  </tr>
  <tr>
    <td>useJSR310Types</td>
    <td>是否使用 JSR-310 标准，默认为 false，为 true 时，DATE->java.time.LocalDate，TIME->java.time.LocalTime，TIMESTAMP->java.time.LocalDateTime</td>
  </tr>
</table>


