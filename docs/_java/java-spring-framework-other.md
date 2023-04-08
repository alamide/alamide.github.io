---
layout: post
title: Spring Framework Other
categories: java
excerpt: Spring Framework Other, V6.0.7
tags: Java Spring SpringFramework
date: 2023-04-06
isHidden: true
---
## 1.单元测试
引入依赖
```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-test</artifactId>
    <version>6.0.7</version>
</dependency>
```

使用
```java
@Slf4j
//@SpringJUnitConfig(classes = {SpringConfig.class}) //注解管理bean
@SpringJUnitConfig(locations = {"classpath:beans.xml"})//配置文件
class CalculatorImpTest {
    @Autowired
    private User user;
    
    @Test
    public void testCalculator(){
        log.info(user.toString());
    }
}
```

## 2.数据库事务
* 数据库的事务 [在这里](../db/database-transaction-manager.html)

### 2.1 JDBC Template
引入依赖
```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>6.0.7</version>
</dependency>

<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.32</version>
</dependency>

<dependency>
    <groupId>com.alibaba</groupId>
    <artifactId>druid</artifactId>
    <version>1.2.16</version>
</dependency>
```

配置 `JavaBean`
```xml
<contex:property-placeholder location="classpath:jdbc.properties"/>
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
    <property name="username" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>
    <property name="url" value="${jdbc.url}"/>
    <property name="driverClassName" value="${jdbc.driverClass}"/>
</bean>

<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
    <property name="dataSource" ref="dataSource"/>
</bean>
```

测试 `JDBCTemplate`
```java
String sql = "select * from t_dept where dept_id=?";
final Dept dept = jdbcTemplate.queryForObject(sql, new BeanPropertyRowMapper<>(Dept.class), 1);
log.info(dept.toString());
```

### 2.2 配置事务
有一段转账的业务代码
```java
public interface AccountDao {
    void sub(String account, int money);

    void add(String account, int money);
}

@Repository
@Slf4j
public class AccountDaoImpl implements AccountDao {
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    public void sub(String account, int money) {
        String sql = "update t_account set balance=balance-? where account=?";
        final int update = jdbcTemplate.update(sql, money, account);
        log.info("sub: {}, account={}, money={}, effected {} rows", sql, account, money, update);
    }

    @Override
    public void add(String account, int money) {
        String sql = "update t_account set balance=balance+? where account=?";
        final int update = jdbcTemplate.update(sql, money, account);
        log.info("add: {}, account={}, money={}, effected {} rows", sql, account, money, update);
    }
}

public interface AccountService {
    void transMoney(String accountA, String accountB, int money);
}

@Service
public class AccountServiceImpl implements AccountService {

    @Autowired
    private AccountDao accountDao;

    @Override
    public void transMoney(String accountA, String accountB, int money) {
        accountDao.sub(accountA, money);
        int i = 1 / 0;
        accountDao.add(accountB, money);
    }
}

@Controller
public class AccountController {
    @Autowired
    private AccountService accountService;

    public void transMoney(String accountA, String accountB, int money){
        accountService.transMoney(accountA, accountB, money);
    }
}

@SpringJUnitConfig(locations = {"classpath:beans.xml"})
class AccountControllerTest {
    @Autowired
    private AccountController accountController;

    @Test
    public void testTrans(){
        accountController.transMoney("8859-1", "8859-2", 100);
    }
}
```

上面的代码会发生问题，
```java
accountDao.sub(accountA, money);
int i = 1 / 0;
accountDao.add(accountB, money);
```

会发生 `accountA` 用户账户扣掉金额，而 `accountB` 用户账户却不能增加，这违背事务的一致性

可以配置事务来解决

配置 `JavaBean`
```xml
<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
<tx:annotation-driven transaction-manager="transactionManager"/>
```

给方法上加上 `@Transactional` 注解（ `@Transactional` 也可以加在类上，加在类上表示所有方法都加上事务控制）
```java
@Override
@Transactional
public void transMoney(String accountA, String accountB, int money) {
    accountDao.sub(accountA, money);
    int i = 1 / 0;
    accountDao.add(accountB, money);
}
```

`@Transactional` 还有一些其它的属性可以配置

1. `timeout` 超时回滚，释放资源

2. `rollbackFor` 

3. `noRollbackFor` 

4. `isolation` 隔离级别，有 `READ UNCOMMITTED` 、`READ COMMITTED` 、 `REPEATABLE READ` 、 `SERIALIZABLE`

5. `propagation` 传播行为，执行过程中多个方法上加上事务，事务是如何传递的？`REQUIRED` 没有就新建，有就加入；`REQUIRES_NEW` 每次都会新建一个事务；


全注解配置事务
```java
@Configuration
@ComponentScan(basePackages = {"com.alamide.spring6"})
@EnableTransactionManagement
public class SpringConfig {
    @Bean("dataSource")
    public DataSource getDataSource(){
        DruidDataSource dataSource = new DruidDataSource();
        dataSource.setPassword("root");
        dataSource.setUsername("root");
        dataSource.setUrl("jdbc:mysql://localhost:3306/db_jdbc");
        dataSource.setDriverClassName("com.mysql.cj.jdbc.Driver");
        return dataSource;
    }

    @Bean("jdbcTemplate")
    public JdbcTemplate getJdbcTemplate(DataSource dataSource){
        JdbcTemplate jdbcTemplate = new JdbcTemplate();
        jdbcTemplate.setDataSource(dataSource);
        return jdbcTemplate;
    }

    @Bean("dataSourceTransactionManager")
    public DataSourceTransactionManager getDataSourceTransactionManager(DataSource dataSource){
        DataSourceTransactionManager dataSourceTransactionManager = new DataSourceTransactionManager();
        dataSourceTransactionManager.setDataSource(dataSource);
        return dataSourceTransactionManager;
    }
}

@Slf4j
@SpringJUnitConfig(classes = {SpringConfig.class})
public class JDBCTemplateTest {
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Test
    public void test(){
        String sql = "select * from t_dept where dept_id=?";
        final Dept dept = jdbcTemplate.queryForObject(sql, new BeanPropertyRowMapper<>(Dept.class), 1);
        log.info(dept.toString());
    }
}
```

基于 `XML` 声明事务，利用 `Spring AOP`
```xml
<contex:property-placeholder location="classpath:jdbc.properties"/>
<bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource">
    <property name="username" value="${jdbc.username}"/>
    <property name="password" value="${jdbc.password}"/>
    <property name="url" value="${jdbc.url}"/>
    <property name="driverClassName" value="${jdbc.driverClass}"/>
</bean>

<bean id="jdbcTemplate" class="org.springframework.jdbc.core.JdbcTemplate">
    <property name="dataSource" ref="dataSource"/>
</bean>

<bean id="transactionManager" class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
    <property name="dataSource" ref="dataSource"/>
</bean>
<tx:annotation-driven transaction-manager="transactionManager"/>

<aop:config>
    <aop:advisor advice-ref="txAdvice" pointcut="execution(* com.alamide.spring6.jdbc.*ServiceImpl.*(..))"/>
</aop:config>
<tx:advice id="txAdvice" transaction-manager="transactionManager">
    <tx:attributes>
        <tx:method name="get*" read-only="true"/>
        <tx:method name="query*" read-only="true"/>
        <tx:method name="save*" read-only="false" propagation="REQUIRES_NEW"/>
        <tx:method name="update*" read-only="false" rollback-for="java.lang.Exception" propagation="REQUIRES_NEW"/>
        <tx:method name="delete*" read-only="false" rollback-for="java.lang.Exception" propagation="REQUIRES_NEW"/>
    </tx:attributes>
</tx:advice>
```

## 3.Resources
包路径是 `org.springframework.core.io.Resources` 

1. `UrlResource` ，`loadUrlResources("http://www.zhaosiyuan.com");` ，访问网络资源

2. `ClassPathResource` ，`loadClasspathResource("beans.xml");` ，访问类路径下文件

3. `FileSystemResource` ，`loadFileSystemResource("pom.xml");` ，相对路径，当前文件夹

```java
public void loadClasspathResource(String fileName) throws IOException {
    ClassPathResource resource = new ClassPathResource(fileName);
    log.info(resource.getFilename());
    log.info(resource.getDescription());
    BufferedReader buffer = new BufferedReader(new InputStreamReader(resource.getInputStream()));
    buffer.lines().forEach(log::info);
    buffer.close();
}
```

### 3.1 ResourceLoader
```java
//获取类路径下的文件
ApplicationContext applicationContext = new ClassPathXmlApplicationContext();
final Resource resource = applicationContext.getResource("beans.xml");

//获取文件系统中文件
ApplicationContext applicationContext = new FileSystemXmlApplicationContext();
final Resource resource = applicationContext.getResource("pom.xml");

//依据前缀，选择使用的 Resource
final Resource resource = applicationContext.getResource("classpath:beans.xml");
final Resource resource = applicationContext.getResource("file:beans.xml");
final Resource resource = applicationContext.getResource("http://www.zhaosiyuan.com");
```

### 3.2 classpath 通配符使用
`classpath*:beans.xml` 搜索所有名为 `beans.xml` 文件，最后合并成一个 `ApplicationContext`，如果是 `classpath*:beans.xml` 则只会加载第一个符合的 `XML` 文件
```java
ApplicationContext ctx = new ClassPathXmlApplicationContext("classpath*:beans.xml");
```

加载多个配置文件，以 `bean` 开头的 `XML`
```java
ApplicationContext ctx = new ClassPathXmlApplicationContext("classpath:bean*.xml");
```

可以结合使用
```java
ApplicationContext ctx = new ClassPathXmlApplicationContext("classpath*:bean*.xml");
```