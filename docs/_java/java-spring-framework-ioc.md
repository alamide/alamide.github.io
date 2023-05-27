---
layout: post
title: Spring Framework IOC
categories: java
excerpt: Spring Framework IOC, V6.0.7
tags: Java Spring SpringFramework
date: 2023-04-05
---
IOC 即 Inversion of Control 的简写，控制反转。对象的创建和管理交由容器负责，控制权交给容器。DI（Dependency Injection）：依赖注入， 实现了控制反转的思想，
指 Spring 创建对象的过程中，将对象依赖属性通过配置进行注入。
## 1.引入 Spring Framework
```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-context</artifactId>
    <version>6.0.7</version>
</dependency>
```

## 2.基于 XML 方式管理 Bean，简单类型
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="user" class="com.alamide.spring6.entities.User"/>
</beans>
```

`id` 要求唯一

获取注入的 `Java Bean` ，依据 `id` 和类型
```java
@Test
public void testXmlBean(){
    ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
    final User user = context.getBean("user", User.class);
    System.out.println(user);
}
```

依据类型
```java
final User userByClass = context.getBean( User.class);
System.out.println(userByClass);
```
### 2.1 给 Bean 对象属性赋值

`get` 、`set`
```xml
<bean id="user" class="com.alamide.spring6.entities.User">
    <property name="age" value="18"/>
    <property name="username" value="alamide"/>
</bean>
```

构造器方式
```xml
<bean id="user" class="com.alamide.spring6.entities.User">
    <constructor-arg name="age" value="18"/>
    <constructor-arg name="username" value="alamide"/>
</bean>
```

特殊值处理
1. `null`， 为属性值赋值为 `null`
   ```xml
   <property name="username">
        <null></null>
   </property>
   ```

2. 特殊值，如 `<` 要转义为 `&lt;` ，或使用 `<![CDATA[]]>`
   ```xml
   <property name="username" value="a&lt;b"/>
   <property name="username">
       <value><![CDATA[a<b]]></value>
   </property>
   ```

### 2.2 给 Bean 对象属性赋值，对象类型
在 `User` 中新增属性 `address`
```java
@Data
@ToString
@AllArgsConstructor
@NoArgsConstructor
public class User {
    private String username;
    private Integer age;
    private Address address;
}
```

可以如下注入属性
```xml
<bean id="user" class="com.alamide.spring6.entities.User">
    <property name="age" value="18"/>
    <property name="username" value="alamide"/>
    <property name="address">
        <bean class="com.alamide.spring6.entities.Address">
            <property name="province" value="suzhou"/>
        </bean>
    </property>
</bean>
```

注意上面的 `Address` 对象并不会注入到容器中

还可以有下面的方式注入
```xml
<bean id="address" class="com.alamide.spring6.entities.Address">
    <property name="province" value="suzhou"/>
</bean>
<bean id="user" class="com.alamide.spring6.entities.User">
    <property name="age" value="18"/>
    <property name="username" value="alamide"/>
    <property name="address" ref="address"/>
</bean>
```

还可以给注入的引用类型赋值
```xml
<bean id="user" class="com.alamide.spring6.entities.User">
    <property name="age" value="18"/>
    <property name="username" value="alamide"/>
    <property name="address" ref="address"/>
    <property name="address.province" value="zhongguo"/>
</bean>
```

### 2.3 给 Bean 对象属性赋值，数组类型，List 类型

`User` 类中新增 `String[] hobbies;` ，可以如下赋值，
```xml
<property name="hobbies">
    <array>
        <value>basketball</value>
        <value>game</value>
    </array>
</property>
```

如果换成 `List<String> hobbies;` 
```xml
<property name="hobbies">
    <list>
        <value>basketball</value>
        <value>game</value>
    </list>
</property>
```

### 2.4 给 Bean 对象属性赋值，Map 类型
`User` 类中新增 `Map<String, Integer> score;` ，
```xml
<property name="score">
    <map>
        <entry key="math" value="100"/>
    </map>
</property>
```

`User` 类中新增 `Map<String, Address> addressMap;` ，
```xml
<property name="addressMap">
    <map>
        <entry key="now" value-ref="address"/>
    </map>
</property>
```

还可以先定义集合类型在引用，需要先引入相应的命名空间 
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:util="http://www.springframework.org/schema/util"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/util
       https://www.springframework.org/schema/util/spring-util.xsd">

<util:list id="hobbies">
    <value>basketball</value>
    <value>game</value>
</util:list>
<util:map id="addressMap">
    <entry key="now" value-ref="address"/>
</util:map>

<property name="hobbies" ref="hobbies"/>
<property name="addressMap" ref="addressMap"/>
```

### 2.5 p 命名空间
要先引入命名空间 `xmlns:p="http://www.springframework.org/schema/p"`
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd">
<bean id="address" class="com.alamide.spring6.entities.Address" p:province="nanjing"/>
```

### 2.6 引入属性文件
引入命名空间，有配置文件 `data.properties` 其中内容有
```
data.username=zhaoxiaosi
data.age=18
```

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:contex="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
       http://www.springframework.org/schema/beans/spring-beans.xsd
       http://www.springframework.org/schema/context 
       https://www.springframework.org/schema/context/spring-context.xsd">
 <contex:property-placeholder location="classpath:data.properties"/>
 <bean id="user" class="com.alamide.spring6.entities.User">
    <property name="age" value="${data.age}"/>
    <property name="username" value="${data.username}"/>
</bean>      
```

### 2.7 Bean 的作用域
`Bean` 的作用域有 `singleton` 、`prototype` ，前者是单例，后者是每次获取 `bean` 实例时都会创建新的对象

### 2.8 FactoryBean
还可以通过 `FactoryBean` 获取 `Bean` ，这种方式可以替我们屏蔽一些封装的细节
```java
public class UserFactoryBean implements FactoryBean<User> {
    @Override
    public User getObject() throws Exception {
        return new User();
    }

    @Override
    public Class<?> getObjectType() {
        return User.class;
    }
}
```

```xml
<bean id="userByFactory" class="com.alamide.spring6.UserFactoryBean"/>
```

### 2.9 自动装配
根据指定的策略，在IOC容器中匹配某一个bean，自动为指定的bean中所依赖的类类型或接口类型属性赋值

```xml
<bean class="com.alamide.spring6.entities.Address">
    <property name="province" value="suzhou"/>
</bean>

<bean id="user" class="com.alamide.spring6.entities.User" autowire="byType">
    <property name="age" value="18"/>
    <property name="username" value="alamide"/>
</bean>
```

这样 `User` 中的属性 `address` 就会被自动装配了，因为容器中有 `Address` 类型的实例

还可以依据 `id` 自动装配
```xml
<bean id="address" class="com.alamide.spring6.entities.Address">
    <property name="province" value="suzhou"/>
</bean>

<bean id="user" class="com.alamide.spring6.entities.User" autowire="byName">
    <property name="age" value="18"/>
    <property name="username" value="alamide"/>
</bean>
```

## 3.基于注解管理 Bean
可以在类上加上注解，然后让 `Spring` 自动扫描，然后将 `Bean` 注入到容器中

扫描类，将有加上指定注解的类注入到容器中，指定的注解有 `@Component` 、 `@Repository` （数据层）、`@Service` 业务逻辑层）、`@Controller` （控制层）

使用上面几个注解时，可以指定 `bean` 的 `id` ，`@Service("userServiceImpl")` ，如不指定，则默认为类型名的首字母小写

获取 `Bean` 时使用 `@Autowired` ，依据类型注入(byType)
```xml
<contex:component-scan base-package="com.alamide.spring6"/>
```

### 3.1 属性注入
```java
public interface UserDao {
    void saveUser(User user);
}

@Slf4j
@Repository
public class UserDaoImpl implements UserDao{
    @Override
    public void saveUser(User user) {
        log.info("save success, user: {}", user);
    }
}

public interface UserService {
    void saveUser(User user);
}

@Service
public class UserServiceImpl implements UserService {
    @Autowired
    private UserDao userDao;
    @Override
    public void saveUser(User user) {
        userDao.saveUser(user);
    }
}

@Controller
public class UserController {
    @Autowired
    private UserService userService;

    public void saveUser(){
        userService.saveUser(new User("alamide", 18));
    }
}

@Test
public void testAnnotationBean(){
    ClassPathXmlApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
    final UserController userController = context.getBean(UserController.class);
    userController.saveUser();
}
```

### 3.2 方法注入
可以在方法上添加 `@Autowired` 注解，会将容器中的对象注入给方法的参数
```java
private UserService userService;

@Autowired
public void setUserService(UserService userService) {
    this.userService = userService;
}
```

和下面的功效一致
```java
@Autowired
private UserService userService;
```

### 3.3 构造器、形参注入
可以在构造器上添加 `@Autowired` 注解，会将容器中的对象注入给方法的参数，注意构造器注入时不可以有无参构造器，否则 `Spring` 只会调用无参构造器
```java
private UserDao userDao;

@Autowired
public UserServiceImpl(UserDao userDao){
    this.userDao = userDao;
}

// 下面也是可以的
// public UserServiceImpl(@Autowired UserDao userDao){
//     this.userDao = userDao;
// }

// 不加任何注解也可，只有一个构造器时
// public UserServiceImpl(UserDao userDao){
//     this.userDao = userDao;
// }
```

和下面的功效一致
```java
@Autowired
private UserDao userDao;
```

### 3.4 同一类型的实例对象有多个
如果容器中还注入了一个 `UserDaoRedisImpl` ，那么上面的代码将会执行失败
```java
@Slf4j
@Repository
public class UserDaoRedisImpl implements UserDao{
    @Override
    public void saveUser(User user) {
        log.info("redis save: {}", user);
    }
}
```

错误信息如下：
```
No qualifying bean of type 'com.alamide.spring6.UserDao' available: expected single matching bean but found 2: userDaoImpl,userDaoRedisImpl
```

在注入的属性上加上 `@Qualifier`
```java
@Autowired
@Qualifier("userDaoRedisImpl")
private UserDao userDao;
```

### 3.5 @Resource 注入
使用`@Resource` 同样可以注入， `@Resource` 注解是 `JDK` 扩展包中的，也就是说属于JDK的一部分。

@Resource注解默认根据名称装配byName，未指定name时，使用属性名作为name。通过name找不到的话会自动启动通过类型byType装配。

@Autowired注解默认根据类型装配byType，如果想根据名称装配，需要配合@Qualifier注解一起用。

@Resource注解用在属性上、setter方法上。

@Autowired注解用在属性上、setter方法上、构造方法上、构造方法参数上。

### 3.6 全注解开发
完全不使用 `xml` 配置文件，全注解

开启扫描
```java
@Configuration
@ComponentScan(basePackages = {"com.alamide.spring6"})
public class SpringConfig {
}
```

测试    
```java
@Test
public void testAnnotationBean(){
    final AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(SpringConfig.class);
    final UserController userController = context.getBean(UserController.class);
    userController.saveUser();
}
```