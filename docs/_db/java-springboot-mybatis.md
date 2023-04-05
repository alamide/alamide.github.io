---
layout: post
title: SpringBoot 整合 Mybatis
categories: db
excerpt: SpringBoot 整合 Mybatis
tags: Mybatis 
date: 2023-04-01
isHidden: true
---
* 文档 [在这里](https://github.com/mybatis/spring-boot-starter/blob/master/mybatis-spring-boot-autoconfigure/src/site/zh/markdown/index.md)

## 1.引入 mybatis-spring-boot-starter
这里 `3.0` 的版本，要求 `jdk17` 及以上，详见参考文档
```xml
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
    <artifactId>mybatis-spring-boot-starter</artifactId>
    <version>2.3.0</version>
</dependency>
```

## 2.基本使用
### 2.1 配置 application.yml
配置 `DataSource` ，及 `*Mapper.xml` 所在的目录
```yml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    username: root
    password: root
    url: jdbc:mysql://localhost:3306/guli_edu
mybatis:
  mapper-locations: classpath:mapper/*Mapper.xml
```
### 2.2 扫描 Mapper 接口
```java
@Configuration
@MapperScan(basePackages = {"com.alamide.guli.mapper"})
public class MybatisConfig {
}
```

也可以不使用 `@MapperScan` ，而在 `Mapper` 接口上添加 `@Mapper` 注解

```java
@Mapper
public interface EduTeacherMapper {
}
```
### 2.3 使用测试
```java
@Slf4j
@SpringBootTest
class MainApplicationTest {
    @Autowired
    private EduTeacherMapper mapper;

    @Test
    public void testMybatis(){
        final List<EduTeacher> eduTeachers = mapper.selectByExample(null);
        log.info(eduTeachers.toString());
    }
}
```

### 2.4 Mybatis 配置
`SpringBoot` 中 `Mybatis` 的配置可以在 `application.yml` 中定义，也可以使用代码

配置文件
```yml
mybatis:
  mapper-locations: classpath:mapper/*Mapper.xml
  configuration:
    map-underscore-to-camel-case: true
    lazy-loading-enabled: true
```

代码配置
```java
@Configuration
@MapperScan(basePackages = {"com.alamide.guli.mapper"})
public class MybatisConfig {
    @Bean
    ConfigurationCustomizer mybatisConfigurationCustomizer() {
        return new ConfigurationCustomizer() {
            @Override
            public void customize(org.apache.ibatis.session.Configuration configuration) {
                configuration.setMapUnderscoreToCamelCase(true);
                configuration.setCacheEnabled(true);
            }
        };
    }
}
```

## 3. 自动注册
`MyBatis-Spring-Boot-Starter` 会自动探测存在的 `DataSource` ，之后注册 `SqlSessionTemplate`
```java
//MybatisAutoConfiguration.java
@Bean
@ConditionalOnMissingBean
public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
    SqlSessionFactoryBean factory = new SqlSessionFactoryBean();
    factory.setDataSource(dataSource);
}

@Bean
@ConditionalOnMissingBean
public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
}
```

指定 `MapperScan` 的包
```java
@SpringBootApplication
@MapperScan(basePackages = {"com.alamide.guli.mapper"})
public class MainApplication {
             
}

@Import(MapperScannerRegistrar.class)
@Repeatable(MapperScans.class)
public @interface MapperScan {}
```

最终一系列操作之后会来到这个类 `ClassPathMapperScanner` 
```java
public class ClassPathMapperScanner extends ClassPathBeanDefinitionScanner {
    
    private Class<? extends MapperFactoryBean> mapperFactoryBeanClass = MapperFactoryBean.class;

    private void processBeanDefinitions(Set<BeanDefinitionHolder> beanDefinitions) {
        definition.setBeanClass(this.mapperFactoryBeanClass);
    }
}
```
`MapperFactoryBean` 是 `FactoryBean` 的一个实现类

```java
public class MapperFactoryBean<T> extends SqlSessionDaoSupport implements FactoryBean<T> {
    @Override
    public T getObject() throws Exception {
        return getSqlSession().getMapper(this.mapperInterface);
    }
}
```

所以我们只需要配置 `DataSource` 然后就可以直接获取 `Mapper` 了，其它的事 `Mybatis` 都替我们做了