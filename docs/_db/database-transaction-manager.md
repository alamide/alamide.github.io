---
layout: post
title: 数据库事务四大特性及隔离级别
categories: db
tags: DB 
date: 2023-04-07
excerpt: 数据库事务四大特性及隔离级别
---
数据库事务( `transaction` )是访问并可能操作各种数据项的一个数据库操作序列，这些操作要么全部执行，要么全部不执行，是一个不可分割的工作单位。事务由事务开始与事务结束之间执行的全部数据库操作组成。

## 1.事务的特性
1. 原子性 ( `Atomicity` )：一个事务中的所有操作，要么全部完成，要么全部不完成，不会结束在中间某个环节。事务在执行的过程中发生错误，会回滚到事务开始时的状态，就像这个事务从来没执行过一下。

2. 一致性 ( `Consistency` )：一致性是指事务必须从一个状态变换到另一个状态，一个事务执行之前和执行之后都必须处于一致性状态。例如，A账户和B账户余额加起来一共100元，那么不管A、B之间不管转账几次，事务结束后最终余额加起来还是100元。

3. 隔离性 ( `Isolation` )：隔离性是当多个用户并发访问数据库时，比如操作同一张表时，数据库为每一个用户开启的事务，不能被其他事务的操作所干扰，多个并发事务之间要相互隔离。

4. 持久性 (`Durability` )：持久性是指一旦事务提交，那么对数据库中数据的改变就是永久性的，即使系统崩溃，数据库还能恢复到事务成功结束时的状态。

## 2.事务的隔离级别
`MySQL InnoDB` 的默认隔离级别是 `REPEATABLE_READ`。
### 2.1 数据库脏读(Dirty Read)
脏读又称无效数据的读出，是指在数据库访问中，事务T1将某一值修改，然后事务T2读取该值，此后T1因为某种原因撤销对该值的修改，这就导致了T2所读取到的数据是无效的，值得注意的是，脏读一般是针对于update操作的。

情景模拟，A、B 之间互相转账
```java
public interface AccountDao {
    void sub(String account, int money);

    void add(String account, int money);

    List<Account> getAccount();

    void updateMoney(String account, int money);
}

@Slf4j
@Repository
public class AccountDaoImpl implements AccountDao {
    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    public void sub(String account, int money) {
        String sql = "update t_account set balance=balance-? where account=?";
        jdbcTemplate.update(sql, money, account);
    }

    @Override
    public void add(String account, int money) {
        String sql = "update t_account set balance=balance+? where account=?";
        jdbcTemplate.update(sql, money, account);
    }

    @Override
    public List<Account> getAccount() {
        String sql = "select * from t_account";
        return jdbcTemplate.query(sql, new BeanPropertyRowMapper<>(Account.class));
    }

    @Override
    public void updateMoney(String account, int money) {
        String sql = "update t_account set balance=? where account=?";
        jdbcTemplate.update(sql, money, account);
    }
}

@Slf4j
@Component
public class DBTransaction {
    @Autowired
    private AccountDao accountDao;

    @Transactional(timeout = 10, isolation = Isolation.READ_UNCOMMITTED)
    public void updateSlow() {
        List<Account> originalAccounts = accountDao.getAccount();
        accountDao.updateMoney("8859-2", originalAccounts.get(1).getBalance() - 100);

        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        int i = 1 / 0;
        accountDao.updateMoney("8859-1", originalAccounts.get(0).getBalance() + 100);
    }

    @Transactional(timeout = 10, isolation = Isolation.READ_UNCOMMITTED)
    public void updateFast() {
        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        final List<Account> originalAccounts = accountDao.getAccount();

        accountDao.updateMoney("8859-1", originalAccounts.get(0).getBalance() - 200);
        accountDao.updateMoney("8859-2", originalAccounts.get(1).getBalance() + 200);
    }
}

@Slf4j
@SpringJUnitConfig(classes = {SpringConfig.class})
class DBTransactionTest {
    @Autowired
    private DBTransaction dbTransaction;

    @Autowired
    private AccountDao accountDao;

    @Test
    public void testDirtyRead() throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(2);

        final List<Account> originalAccounts = accountDao.getAccount();
        log.info("original accounts: ");
        originalAccounts.forEach((account -> log.info(account.toString())));

        final ExecutorService executorService = Executors.newFixedThreadPool(2);
        executorService.execute(() -> {
            try {
                dbTransaction.updateSlow();
            } catch (Exception e) {
                log.error("exception is {}", e.getMessage());
            } finally {
                countDownLatch.countDown();
            }
        });
        executorService.execute(() -> {
            dbTransaction.updateFast();
            countDownLatch.countDown();
        });
        
        countDownLatch.await();

        final List<Account> newAccounts = accountDao.getAccount();
        log.info("now accounts: ");
        newAccounts.forEach((account -> log.info(account.toString())));
    }
}
```

output:
```
22:05:30.233 [main] INFO  com.alamide.spring6.jdbc.DBTransactionTest -- original accounts: 
22:05:30.236 [main] INFO  com.alamide.spring6.jdbc.DBTransactionTest -- Account(id=1, account=8859-1, balance=1000)
22:05:30.236 [main] INFO  com.alamide.spring6.jdbc.DBTransactionTest -- Account(id=2, account=8859-2, balance=1000)
22:05:35.276 [pool-1-thread-1] ERROR com.alamide.spring6.jdbc.DBTransactionTest -- exception is / by zero
22:05:35.281 [main] INFO  com.alamide.spring6.jdbc.DBTransactionTest -- now accounts: 
22:05:35.281 [main] INFO  com.alamide.spring6.jdbc.DBTransactionTest -- Account(id=1, account=8859-1, balance=800)
22:05:35.281 [main] INFO  com.alamide.spring6.jdbc.DBTransactionTest -- Account(id=2, account=8859-2, balance=1100)
```

可以看到违背数据一致性，有数据库脏读出现

在测试过程中，有一个 `bug` ，搞了很久，而且难以出现数据库脏读现象，在这里记录一下，发生问题的代码如下
```java
@Slf4j
@Component
public class DBTransaction {
    @Autowired
    private AccountDao accountDao;

    @Transactional(timeout = 10, isolation = Isolation.READ_UNCOMMITTED)
    public void updateSlow() {
//        List<Account> originalAccounts = accountDao.getAccount();
//        accountDao.updateMoney("8859-2", originalAccounts.get(1).getBalance() - 100);
        accountDao.sub("8859-2", 100);
        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
//        int i = 1 / 0;
//        accountDao.updateMoney("8859-1", originalAccounts.get(0).getBalance() + 100);
        accountDao.add("8859-1", 100);
    }

    @Transactional(timeout = 10, isolation = Isolation.READ_UNCOMMITTED)
    public void updateFast() {
        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
//        final List<Account> originalAccounts = accountDao.getAccount();
//        accountDao.updateMoney("8859-1", originalAccounts.get(0).getBalance() - 200);
//        accountDao.updateMoney("8859-2", originalAccounts.get(1).getBalance() + 200);
        accountDao.sub("8859-1", 200);
        accountDao.add("8859-2", 200);
    }
}
```

这样执行测试，会报 `Deadlock found when trying to get lock; try restarting transaction` ，而且不会发生 `Dirty Read` 。最开始网上搜答案是，由于数据库加了索引，也有的说是没加索引，我试了都无法解决这个问题。最后通过研究日志，猜测是与 `update t_account set balance=balance-? where account=?` ，这里的 `balance=balance-?` 有关，我的理解是使用这条语句时会给 `balance` 加锁，也就是给这一行记录加锁，所以两个事务发生死锁问题。改为直接更新，测试通过，发生 `Dirty Read` 。

### 2.2 数据库不可重复读(Non-repeatable Read)
在一个事务内，多次读同一个数据。在这个事务还没有结束时，另一个事务也访问该同一数据并修改数据。那么，在第一个事务的两次读数据之间。由于另一个事务的修改，那么第一个事务两次读到的数据可能不一样，这样就发生了在一个事务内两次读到的数据是不一样的，因此称为不可重复读，即原始读取不可重复。

不可重复读和脏读的区别是，脏读是某一事务读取了另一个事务未提交的脏数据，而不可重复读则是读取了前一事务提交的数据。
```java
@Component
@Slf4j
public class DBTransaction {
    @Autowired
    private AccountDao accountDao;

    @Transactional(timeout = 10, isolation = Isolation.READ_UNCOMMITTED)
    public void updateSlow() {
        Account originalAccount = accountDao.getAccountByAcc("8859-2");
        log.info("updateSlow------------->firstRead--------------->");
        log.info(originalAccount.toString());

        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        
        originalAccount = accountDao.getAccountByAcc("8859-2");
        log.info("updateSlow------------->secondRead--------------->");
        log.info(originalAccount.toString());
    }

    @Transactional(timeout = 10, isolation = Isolation.READ_UNCOMMITTED)
    public void updateFast() {
        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        final List<Account> originalAccounts = accountDao.getAccount();

        accountDao.updateMoney("8859-1", originalAccounts.get(0).getBalance() - 200);
        accountDao.updateMoney("8859-2", originalAccounts.get(1).getBalance() + 200);
    }
}

@Test
public void testNonRepeatableRead() throws InterruptedException {
    CountDownLatch countDownLatch = new CountDownLatch(2);

    final ExecutorService executorService = Executors.newFixedThreadPool(2);
    executorService.execute(() -> {
        try {
            dbTransaction.updateSlow();
        } catch (Exception e) {
            log.error("exception is {}", e.getMessage());
        } finally {
            countDownLatch.countDown();
        }
    });
    executorService.execute(() -> {
        dbTransaction.updateFast();
        countDownLatch.countDown();
    });

    countDownLatch.await();
}
```

output:
```
10:01:16.865 [pool-1-thread-1] INFO  com.alamide.spring6.jdbc.DBTransaction -- updateSlow------------->firstRead--------------->
10:01:16.868 [pool-1-thread-1] INFO  com.alamide.spring6.jdbc.DBTransaction -- Account(id=2, account=8859-2, balance=2000)
10:01:21.874 [pool-1-thread-1] INFO  com.alamide.spring6.jdbc.DBTransaction -- updateSlow------------->secondRead--------------->
10:01:21.875 [pool-1-thread-1] INFO  com.alamide.spring6.jdbc.DBTransaction -- Account(id=2, account=8859-2, balance=2200)
```

可以看到同一个事务中同一个数据两次读取的数据不一致， `updateSlow` 读到了 `updateFast` 修改提交的事务，出现了不可重复读。

### 2.3 数据库不可重复读(Phantom Read)
幻读是指当事务不是独立执行时发生的一种现象，例如第一个事务对一个表中的数据进行了修改，比如这种修改涉及到表中的“全部数据行”。同时，第二个事务也修改这个表中的数据，这种修改是向表中插入“一行新数据”。那么，以后就会发生操作第一个事务的用户发现表中还存在没有修改的数据行，就好象发生了幻觉一样。

幻读和不可重复读的异同：都是读取了另一条已经提交的事务（这点就脏读不同），所不同的是不可重复读查询的都是同一个数据项，而幻读针对的是一批数据整体（比如数据的个数）。
```java
@Component
@Slf4j
public class DBTransaction {
    @Autowired
    private AccountDao accountDao;

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Transactional(timeout = 10, isolation = Isolation.REPEATABLE_READ)
    public void updateSlow() {
        String sql = "select * from t_account";
        List<Account> originalAccounts = jdbcTemplate.query(sql, new BeanPropertyRowMapper<>(Account.class));;
        log.info("updateSlow------------->firstRead--------------->");
        int sum = originalAccounts.stream().mapToInt(Account::getBalance).sum();
        log.info("total money = {}", sum);

        try {
            TimeUnit.SECONDS.sleep(5);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }

        sql = "select * from t_account for update";
        originalAccounts = jdbcTemplate.query(sql, new BeanPropertyRowMapper<>(Account.class));
        log.info("updateSlow------------->secondRead--------------->");
        sum = originalAccounts.stream().mapToInt(Account::getBalance).sum();
        log.info("total money = {}", sum);
    }

    @Transactional(timeout = 10, isolation = Isolation.REPEATABLE_READ)
    public void updateFast() {
        try {
            TimeUnit.SECONDS.sleep(3);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }

        accountDao.saveAccount(new Account(null, "8859-4", 1000));
    }
}
```

output:
```
10:45:39.389 [pool-1-thread-1] INFO  com.alamide.spring6.jdbc.DBTransaction -- updateSlow------------->firstRead--------------->
10:45:39.391 [pool-1-thread-1] INFO  com.alamide.spring6.jdbc.DBTransaction -- total money = 4000
10:45:44.397 [pool-1-thread-1] INFO  com.alamide.spring6.jdbc.DBTransaction -- updateSlow------------->secondRead--------------->
10:45:44.398 [pool-1-thread-1] INFO  com.alamide.spring6.jdbc.DBTransaction -- total money = 5000
```
同一事务中，两次计算的总金额却不一致，好像出现了幻觉。这里使用的隔离级别是 `Isolation.REPEATABLE_READ` ，“不能解决”这里的幻读。我这里的第二次查询使用了
`select * from t_account for update` ，这样才会出现幻读现象。`MySQL` 的 `REPEATABLE_READ` 是可以解决部分幻读问题的。具体的解答 [MySQL 是如何解决幻读的？ - 一灯架构的回答 - 知乎](https://www.zhihu.com/question/437140633/answer/2692819760)。

## 3.代码所用依赖及配置文件
```xml
 <dependencies>
    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-context</artifactId>
        <version>6.0.7</version>
    </dependency>

    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-test</artifactId>
        <version>6.0.7</version>
    </dependency>
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

    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aop</artifactId>
        <version>6.0.7</version>
    </dependency>

    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-aspects</artifactId>
        <version>6.0.7</version>
    </dependency>
    <dependency>
        <groupId>org.junit.jupiter</groupId>
        <artifactId>junit-jupiter-api</artifactId>
        <version>5.9.2</version>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.12</version>
    </dependency>
    <dependency>
        <groupId>ch.qos.logback</groupId>
        <artifactId>logback-classic</artifactId>
        <version>1.3.6</version>
    </dependency>
</dependencies>
```

注解管理 `Bean`
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
        dataSource.setMaxActive(10);
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
```

`logback-test.xml`
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration>

<configuration>
    <import class="ch.qos.logback.classic.encoder.PatternLayoutEncoder"/>
    <import class="ch.qos.logback.core.ConsoleAppender"/>

    <appender name="STDOUT" class="ConsoleAppender">
        <encoder class="PatternLayoutEncoder">
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{50} -%kvp- %msg%n</pattern>
        </encoder>
    </appender>
    
    <root level="ERROR">
        <appender-ref ref="STDOUT"/>
    </root>
    <logger level="INFO" name="com.alamide.spring6.jdbc"/>
</configuration>
```