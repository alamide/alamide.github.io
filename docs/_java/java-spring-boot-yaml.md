---
layout: post
title: SpringBoot YAML
categories: java
tags: Java Spring SpringBoot
date: 2023-05-23
---
SpringBoot 中 YAML 配置相关
<!--more-->
## 1.SpringBoot 属性加载顺序
配置文件属性加载的顺序，后面定义的 Property 会覆盖前面定义的 Property，下面列举一些常见的获取 Property 的顺序。
### 1.1 SpringApplication.setDefaultProperties 
```java
@Component
public class MainConfig {

    @Value("${message}")
    private String message;

    public String sayHello(){
        return message;
    }
}

@RestController
@SpringBootApplication
public class MainApplication {

    @Autowired
    private MainConfig mainConfig;

    @RequestMapping("/")
    String hello(){
        return mainConfig.sayHello();
    }
    public static void main(String[] args) {
        SpringApplication springApplication = new SpringApplication(MainApplication.class);
        final Properties properties = new Properties();
        properties.put("message", "setDefaultProperties");
        springApplication.setDefaultProperties(properties);
        springApplication.run(args);
    }
}
```

`curl http://localhost:8080`

output: `setDefaultProperties`

### 1.2 @PropertySource

`app.properties`

```
message=@PropertySource
```

```java
@RestController
@SpringBootApplication
@PropertySource("classpath:app.properties")
public class MainApplication {
    //同上
}
```

`curl http://localhost:8080`

output: `@PropertySource`

### 1.3 application.properties

`application.yaml`

```yaml
message: this is a message
```

Java 部分代码不变

`curl http://localhost:8080`

output: `this is a message`

### 1.4 System.getProperties

```java
@RestController
@SpringBootApplication
@PropertySource("classpath:app.properties")
public class MainApplication {
    public static void main(String[] args) {
        System.setProperty("message", "System.setProperty");
        //同上
        ....
    }
}
```

`curl http://localhost:8080`

output: `System.setProperty`

### 1.5 Command line arguments

打成可执行 Jar 包执行，`java -jar SpringBootSS-1.0-SNAPSHOT.jar --message="Command Line Message"`

`curl http://localhost:8080`

output: `Command Line Message`


配置文件加载的顺序是：
1. JAR 包中的 application.properties

2. JAR 包中的 application-{profile}.properties and YAML variants

3. JAR 包外的 application.properties

4. JAR 包外的 application-{profile}.properties and YAML variants

同目录中同时存在 .yaml 和 .properties 时，.properties 优先

## 2.SpringBoot 加载配置文件位置及顺序
1. classpath 下加载

2. classpath 下 /config 目录下加载

3. 当前目录下，即 jar 存放的目录

4. 当前目录下，config/

5. 当前目录下，config/ 的子目录

加载顺序如图，序号大的覆盖序号小的

![Config-Order](../assets/imgs/java-spring-boot-config-order.jpg)

也可以自己指定配置文件名及配置文件存放路径，不常用，这里就不再介绍了，用到的时候再看文档

## 3.Multi-Document Files
在同一个文件可以定义多个逻辑配置文件，后面定义的属性覆盖前面定义的属性。 .yaml 文件使用 `---` 分割，.properties 使用 `#---` 或 `!---` 分割。
```yml
spring:
  application:
    name: "Application1"
---
spring:
  application:
    name: "Application2"
```

最终 `spring.application.name=Application2` 

## 4.YAML 随机数据
```yaml
my:
  secret: "${random.value}"
  number: "${random.int}"
  bignumber: "${random.long}"
  uuid: "${random.uuid}"
  number-less-than-ten: "${random.int(10)}"
  number-int-range: "${random.int[1024,65536]}"
```

## 5.将配置文件中属性绑定到 JavaBean
可以将配置文件中属性与配置 JavaBean 绑定起来，也是 SpringBoot 重要特性之一，配置 > 约定 > 编码
```yaml
my:
  service:
    enabled: true
    remote-address: 127.0.0.1
    security:
      username: alamide
      password: 123456
      roles:
        - Admin
        - Manager
```

```java
@Data
@Component
@ConfigurationProperties("my.service")
@ToString
public class MyProperties {

    private boolean enabled;

    private InetAddress remoteAddress;

    private final Security security = new Security();

    @Data
    @ToString
    public static class Security {

        private String username;

        private String password;

        private List<String> roles = new ArrayList<>(Collections.singleton("USER"));
    }
}
```

JavaBean 有多个构造器时，需要给默认构造起加上 @ConstructorBinding，当配置问价中没有指定属性时，对应的属性会被设置为 null，如果需要默认值则可以使用    @DefaultValue 。使用 @ConstructorBinding 时需要使用 @EnableConfigurationProperties 来使绑定成功。
```java
@Data
@ToString
@ConfigurationProperties("my.service")
public class MyProperties {
    //fields
    public MyProperties(boolean enabled) {
        this.enabled = enabled;
    }

    @ConstructorBinding
    public MyProperties(boolean enabled, InetAddress remoteAddress, @DefaultValue Security security) {
        this.enabled = enabled;
        this.remoteAddress = remoteAddress;
        this.security = security;
    }
}
```

当 JavaBean 有有参构造器时，不能使用 @Component 将 JavaBean 注入到容器中，需要使用 @EnableConfigurationProperties
```java
//false Lite 模式，提高 SpringBoot 启动速度；true Full 模式，单例，每次获取都是同一个 Bean
@Configuration(proxyBeanMethods = false)
@EnableConfigurationProperties(SomeProperties.class)
public class MyConfiguration {

}
```

还可以在启动类上使用 `@ConfigurationPropertiesScan` 注解
```java
@SpringBootApplication
@ConfigurationPropertiesScan({"com.alamide.springboot.config"})
public class MainApplication {
}
```

对不受控制的第三方组件绑定属性，如导入第三方的 JAR 包，对 JAR 包中的组件配置属性，你不可能去修改 JAR 包中的类，在类上加上 `@ConfigurationProperties` 注解
```java
@Data
@ToString
public class MainConfig {
    private String message;
    private Integer number;
}

@Configuration(proxyBeanMethods = false)
public class MyConfiguration {
    @Bean
    @ConfigurationProperties(prefix = "my")
    public MainConfig mainConfig(){
        return new MainConfig();
    }
}
```

绑定时会自动将连字符型和下划线转为驼峰，一下都可以绑定到 `private String firstName;`
```
my.main-project.person.first-name
my.main-project.person.firstName
my.main-project.person.first_name
```

## 6.Binding Maps
如果 key 没有用 [] 包裹起来，所有非字母数字，- 或 . 的字符都会被移除
```yaml
my:
  map:
    "/key3": "value3"
    "[/key2]": "value2"
```

```java
@Data
@ToString
@ConfigurationProperties(prefix = "my")
public class MainConfig {
    private Map<String, String> map;
}
```

打印 map 为 `map={key3=value3, /key2=value2}` ，key3 前面的 / 被移除

对于接收 Map 为 Map<String, String> 时，将会保留 . ，而为 Map<String, Object> 时，需要使用 [] 包裹
```yaml
my:
  map:
    "/key3": "value3"
    "[/key2]": "value2"
    "a.b": "c"
```

Map<String, String> 接收，map 中内容为 `map={key3=value3, /key2=value2, a.b=c}`

Map<String, Object> 接收，map 中内容为 `map={key3=value3, /key2=value2, a={b=c}}`

## 7.Merging Complex Types
如果 list 在多个配置文件中配置，被选中配置文件中 list ，会覆盖前面的，map 也是一样的。
```yaml
my:
  list:
    - name: "my name"
      description: "my description"
---
spring:
  config:
    activate:
      on-profile: "dev"
my:
  list:
    - name: "my another name"
```

```java
@Data
public class MyPojo {
    private String name;
    private String description;
}

@Data
@ToString
@ConfigurationProperties(prefix = "my")
public class MainConfig {
    private List<MyPojo> list;
}
```

如果 dev 没有激活，那么 list 中含有一个 `MyPojo` ，内容为 `MyPojo(name=my name, description=my description)`

如果 dev 被激活，那么 list 仍会含有一个 `MyPojo` ，内容为 `MyPojo(name=my another name, description=null)`

list 并不会合并，而是会覆盖

## 8.Converting Durations
可以转换 Duration ，支持的单位有，`ns` 、`us` 、`ms` 、`s` 、`m` 、`h` 、`d` ，默认的单位是 `ms`
```yaml
time:
  sessionTimeout: 10h
  readTimeout: 1100ms
```

```java
@Data
@ToString
@Component
@ConfigurationProperties("time")
public class MyProperties {
    private Duration sessionTimeout;
    private Duration readTimeout;
}
```

MyProperties 为 `MyProperties(sessionTimeout=PT10H, readTimeout=PT1.1S)`

可以为 JavaBean 的属性设置默认单位，在配置文件中不设置单位时，则会使用默认的单位，下面的 `sessionTimeout` 为 10d
```yaml
time:
  sessionTimeout: 10
  readTimeout: 1100ms
```

```java
@Data
@ToString
@Component
@ConfigurationProperties("time")
public class MyProperties {
    @DurationUnit(ChronoUnit.DAYS)
    private Duration sessionTimeout;
    private Duration readTimeout;
}
```

对于 Contructor 的绑定，可以使用 `@DefaultValue`
```java
@Data
@ToString
@ConfigurationProperties("time")
public class MyProperties {
    private Duration sessionTimeout;
    private Duration readTimeout;

    public MyProperties(@DurationUnit(ChronoUnit.DAYS) @DefaultValue("50d") Duration sessionTimeout,
                        @DefaultValue("100ms") Duration readTimeout) {
        this.sessionTimeout = sessionTimeout;
        this.readTimeout = readTimeout;
    }
}
```

## 9.Converting Periods
和 Duration 一样，支持的单位为 `y` 、`m` 、`w` 、`d` ，默认的单位是 `d`
```yaml
time:
  period: 10y
```

```java
@Data
@ToString
@ConfigurationProperties("time")
public class MyProperties {
    private Period period;
}
```

## 10.Converting Data Sizes
和 Duration 类似，支持的单位有 `B` 、`KB` 、`MB` 、`GB` 、`TB` ，默认的单位是 `bytes(B)`
```yaml
data:
  buffer-size: 10MB
```

```java
@Data
@ToString
@ConfigurationProperties("data")
public class MyProperties {
    private DataSize bufferSize;
}
```

## 11.Profiles
可以使用 @Profile 来使指定组件被加载
```java
//spring.profiles.active=dev 时生效
@Configuration(proxyBeanMethods = false)
@Profile("dev")
public class DevConfiguration {
    @Bean
    public DatabaseConfig dataBaseConfig() {
        DatabaseConfig dataBaseConfig = new DatabaseConfig();
        dataBaseConfig.setDriverClassName("com.mysql.cj.jdbc.Driver");
        dataBaseConfig.setUrl("jdbc:mysql://localhost:3306/db01");
        dataBaseConfig.setUsername("root");
        dataBaseConfig.setPassword("root");
        return dataBaseConfig;
    }
}

//spring.profiles.active=prod 时生效
@Configuration(proxyBeanMethods = false)
@Profile("prod")
public class ProductionConfiguration {
    @Bean
    public DatabaseConfig dataBaseConfig() {
        DatabaseConfig dataBaseConfig = new DatabaseConfig();
        dataBaseConfig.setDriverClassName("oracle.jdbc.driver.OracleDrive");
        dataBaseConfig.setUrl("jdbc:oracle:thin:@192.168.221.205:1521:orcl");
        dataBaseConfig.setUsername("root");
        dataBaseConfig.setPassword("root");
        return dataBaseConfig;
    }
}
```

没有指定 `spring.profiles.active` 时，会加载默认的配置文件，可以自行指定
```yaml
spring:
  profiles:
    default: "dev"
```

`spring.profiles.active` 和 `spring.profiles.default` 只能在 `non-profile` 文档中使用
```yaml
# this document is valid
spring:
  profiles:
    active: "prod"
---
# this document is invalid，会报错
spring:
  config:
    activate:
      on-profile: "prod"
  profiles:
    active: "metrics"
```

增加配置文件而不是替换，如下的配置，在指定 `spring.profiles.active` 时，不会替换
```yaml
spring:
  profiles:
    include:
      - "common"
      - "local"
```

指定一组配置文件，指定 `spring.profiles.active=production` 时，`production` 、`proddb` 、`prodmq` 三个配置文件生效 
```yaml
spring:
  profiles:
    group:
      production:
      - "proddb"
      - "prodmq"
```