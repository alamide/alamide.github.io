---
layout: post
title: SpringBoot 基础笔记
categories: java
excerpt: SpringBoot 基础笔记
tags: Java SpringBoot
---

### 1.文档位置 
SpringBoot 文档 [https://docs.spring.io/spring-boot/docs/current/reference/html/](https://docs.spring.io/spring-boot/docs/current/reference/html/)

### 2.@ConditionalOnBean
符合条件才会向容器中注入对象，一个 `Configuration` 中，Spring 向容器中注入对象是按代码顺序注入的。将下面的两个 `bean` 交换顺序，`user` 对象将不会注入。
```java
@Configuration
public class MyConfig {

    @Bean(name = "tom")
    public Pet getPet(){
        return new Pet("tomcat");
    }

    @Bean("user")
    @ConditionalOnBean(name = "tom")
    public User getUser(){
        User user = new User("cherry", 20);
        user.setPet(getPet());
        return user;
    }
}
```

### 3.@ImportResource

向容器中注入 `xml` 中配置的 `bean` ，将 `SpringMVC` 升级到 `SpringBoot` 时可能会用到这种引入方式。
```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="pet" class="com.zhaosiyuan.springboot.pojo.Pet">
        <property name="name" value="tom"/>
    </bean>
    <bean id="user" class="com.zhaosiyuan.springboot.pojo.User">
        <property name="name" value="Miki"/>
        <property name="age" value="20"/>
        <property name="pet" ref="pet"/>
    </bean>
</beans>
```

```java
@Configuration
@ImportResource("classpath:beans.xml")
public class ConfigImportResource {

}
```
### 4.配置绑定，从 `properties` 中读取配置属性
`application.yml`
```yaml
mysqlconfig:
  username: root
  password: root
  url: jdbc:mysql://localhost:3306/db03
  driver-name: com.mysql.jdbc.Driver
```
两种配置方式
* 1.@EnableConfigurationProperties + @ConfigurationProperties

```java
@Configuration
@ImportResource
@EnableConfigurationProperties(MySQLConfig.class)
public class MyConfig {
}

@Data
@AllArgsConstructor
@ToString
@NoArgsConstructor
@ConfigurationProperties(prefix = "mysqlconfig")
public class MySQLConfig {
    private String driverName;
    private String username;
    private String password;
    private String url;
}
```
* 2.@Component + @ConfigurationProperties
  
```java
@Data
@AllArgsConstructor
@ToString
@NoArgsConstructor
@Component
@ConfigurationProperties(prefix = "mysqlconfig")
public class MySQLConfig {
    private String driverName;
    private String username;
    private String password;
    private String url;
}
```
### 5.加载到容器
每个子工程中的 `Bean` 是如何加载到 `Spring` 容器中的

`classpath` 下 `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` 文件中读取

如 `org.springframework.boot.autoconfigure.web.servlet.DispatcherServletAutoConfiguration` 在这个类中将相关的 `Bean` 装载到容器中

### 6.静态资源访问
* SpringBoot 默认静态资源存放路径为 `CLASSPATH_RESOURCE_LOCATIONS` ，见代码

```java
@ConfigurationProperties(prefix = "spring.resources", ignoreUnknownFields = false)
public class ResourceProperties {

	private static final String[] CLASSPATH_RESOURCE_LOCATIONS = { "classpath:/META-INF/resources/",
			"classpath:/resources/", "classpath:/static/", "classpath:/public/" };
  private String[] staticLocations = CLASSPATH_RESOURCE_LOCATIONS;

  public String[] getStaticLocations() {
		return this.staticLocations;
	}

	public void setStaticLocations(String[] staticLocations) {
		this.staticLocations = appendSlashIfNecessary(staticLocations);
	}
}
```
可以在配置文件中自定义配置

```yaml
spring:
  resources:
    static-locations: [classpath:photo]
```
* SpringBoot 默认静态资源访问路径

```java
@ConfigurationProperties(prefix = "spring.mvc")
public class WebMvcProperties {
  private String staticPathPattern = "/**";

  public void setStaticPathPattern(String staticPathPattern) {
		this.staticPathPattern = staticPathPattern;
	}
}
```
同样可以自定义，配置后访问路径为 `http:localhost:8080/res/bug.jpeg`

```yaml
spring:
  mvc:
    static-path-pattern: /res/**
```
### 7.读取 URL 矩阵参数
形如 `http:localhost:8080/user/info;name=alamide;age=18` 读取方法

```java
@RestController
public class MethodController {
    @RequestMapping("/user/{path}")
    public Map<String, Object> userInfo(
            @PathVariable("path") String path,
            @MatrixVariable("name") String name,
            @MatrixVariable("age") Integer age
    ){
        Map<String, Object> infoMap = new HashMap<>();
        infoMap.put("path", path);//path=info
        infoMap.put("name", name);//name=alamide
        infoMap.put("age", age);//age=18
        return infoMap;
    }
}

@Configuration
@ImportResource
@EnableConfigurationProperties(MySQLConfig.class)
public class MyConfig {
    @Bean
    public WebMvcConfigurer webMvcConfigurer(){
        return new WebMvcConfigurer() {
            @Override
            public void configurePathMatch(PathMatchConfigurer configurer) {
                UrlPathHelper urlPathHelper = new UrlPathHelper();
                urlPathHelper.setRemoveSemicolonContent(false);
                configurer.setUrlPathHelper(urlPathHelper);
            }
        };
    }
}
```
SpringBoot 中默认在 `WebMvcAutoConfiguration` 中实现
```java
@Configuration(proxyBeanMethods = false)
@Import(EnableWebMvcConfiguration.class)
@EnableConfigurationProperties({ WebMvcProperties.class, ResourceProperties.class })
@Order(0)
public static class WebMvcAutoConfigurationAdapter implements WebMvcConfigurer {
  @Override
  @SuppressWarnings("deprecation")
  public void configurePathMatch(PathMatchConfigurer configurer) {
    configurer.setUseSuffixPatternMatch(this.mvcProperties.getPathmatch().isUseSuffixPattern());
    configurer.setUseRegisteredSuffixPatternMatch(
        this.mvcProperties.getPathmatch().isUseRegisteredSuffixPattern());
    this.dispatcherServletPath.ifAvailable((dispatcherPath) -> {
      String servletUrlMapping = dispatcherPath.getServletUrlMapping();
      if (servletUrlMapping.equals("/") && singleDispatcherServlet()) {
        UrlPathHelper urlPathHelper = new UrlPathHelper();
        urlPathHelper.setAlwaysUseFullPath(true);
        configurer.setUrlPathHelper(urlPathHelper);
      }
    });
  }
}
```
`WebMvcConfigurer` 可以有多个实现类，最终会存储在一个 `List` 集合中，按序执行 `configurePathMatch` 方法，注意 `WebMvcAutoConfigurationAdapter` 中有
注解 `@Order(0)` ，这个注解能保证我们的配置可以生效。

```java
class WebMvcConfigurerComposite implements WebMvcConfigurer {

	private final List<WebMvcConfigurer> delegates = new ArrayList<>();


	public void addWebMvcConfigurers(List<WebMvcConfigurer> configurers) {
		if (!CollectionUtils.isEmpty(configurers)) {
			this.delegates.addAll(configurers);
		}
	}


	@Override
	public void configurePathMatch(PathMatchConfigurer configurer) {
		for (WebMvcConfigurer delegate : this.delegates) {
			delegate.configurePathMatch(configurer);
		}
	}
}
```

