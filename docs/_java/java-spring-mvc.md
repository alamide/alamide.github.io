---
layout: post
title: SpringMVC 
categories: java
tags: Java SpringMVC
date: 2023-05-09
---
MVC 是一种软件架构的思想，将软件按照模型、视图、控制器来划分。

M：Model，模型层，指工程中的 JavaBean，作用是处理数据。JavaBean 又分为两类，一类称为实体类 Bean，专门存储业务数据的；另一类称为业务处理 Bean ，指     Service 或 Dao 对象，专门用于处理业务逻辑和数据访问

V：View，视图层，指工程中的 html 或 jsp 等页面，作用是与用户进行交互，展示数据

C：Controller，控制层，指工程中的 Servlet ，作用是接收请求和响应浏览器

SpringMVC 是 Spring 的一个后续产品，是 Spring 的一个子项目，是 Spring 为表述层开发提供的一整套完备的解决方案

本文文档地址 [在这里](https://docs.spring.io/spring-framework/docs/current/reference/html/web.html#mvc)
<!--more-->
## 1.版本依赖
Java17、Tomcat10、SpringMVC6.0.8、Servlet5.0
```xml
<properties>
    <maven.compiler.source>17</maven.compiler.source>
    <maven.compiler.target>17</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>

<dependencies>
    <dependency>
        <groupId>jakarta.servlet</groupId>
        <artifactId>jakarta.servlet-api</artifactId>
        <version>5.0.0</version>
        <scope>provided</scope>
    </dependency>

    <dependency>
        <groupId>org.springframework</groupId>
        <artifactId>spring-webmvc</artifactId>
        <version>6.0.8</version>
    </dependency>
</dependencies>
```

## 2.配置 DispatcherServlet
DispatcherServlet 有三种配置方式，web.xml、WebApplicationInitializer、AbstractAnnotationConfigDispatcherServletInitializer，这里我会选择第三种方式
### 2.1 web.xml
```xml
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">
<!--    配套配置-->
<!--    <context-param>-->
<!--        <param-name>contextConfigLocation</param-name>-->
<!--        <param-value>classpath:applicationContext.xml</param-value>-->
<!--    </context-param>-->
<!--    <listener>-->
<!--        <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>-->
<!--    </listener>-->

    <servlet>
        <servlet-name>springDispatcherServlet</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>classpath:spring-mvc.xml</param-value>
        </init-param>
    </servlet>
    <servlet-mapping>
        <servlet-name>springDispatcherServlet</servlet-name>
        <url-pattern>/*</url-pattern>
    </servlet-mapping>
</web-app>
```

`spring-mvc.xml`

```xml
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:mvc="http://www.springframework.org/schema/mvc"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd http://www.springframework.org/schema/context https://www.springframework.org/schema/context/spring-context.xsd http://www.springframework.org/schema/mvc https://www.springframework.org/schema/mvc/spring-mvc.xsd">
    <context:component-scan base-package="com.alamide.web"/>
</beans>
```

### 2.2 WebApplicationInitializer
```java
public class MyWebApplicationInitializer implements WebApplicationInitializer {

    @Override
    public void onStartup(ServletContext servletContext) {

        // Load Spring web application configuration
        AnnotationConfigWebApplicationContext context = new AnnotationConfigWebApplicationContext();
        context.register(AppConfig.class);

        // Create and register the DispatcherServlet
        DispatcherServlet servlet = new DispatcherServlet(context);
        ServletRegistration.Dynamic registration = servletContext.addServlet("app", servlet);
        registration.setLoadOnStartup(1);
        registration.addMapping("/*");
    }
    //上面的是加载注解配置类，下面是加载 xml 配置 
    // @Override
    // public void onStartup(ServletContext container) {
    //     XmlWebApplicationContext appContext = new XmlWebApplicationContext();
    //     appContext.setConfigLocation("/WEB-INF/spring/dispatcher-config.xml");

    //     ServletRegistration.Dynamic registration = container.addServlet("dispatcher", new DispatcherServlet(appContext));
    //     registration.setLoadOnStartup(1);
    //     registration.addMapping("/");
    // }
}
```

### 2.3 AbstractAnnotationConfigDispatcherServletInitializer
这里底层实际也是使用 WebApplicationInitializer 的方式来注册的，只不过官方替我们封装了一下，让我们配置更简单
```java
@ComponentScan(basePackages = {"com.alamide.web"})
public class AppConfig {

}

//Java-based Spring configuration
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

    @Override
    protected Class<?>[] getRootConfigClasses() {
        return null;
    }

    @Override
    protected Class<?>[] getServletConfigClasses() {
        return new Class<?>[] { AppConfig.class };
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/*" };
    }
}

//XML-based Spring configuration
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {

    @Override
    protected WebApplicationContext createRootApplicationContext() {
        return null;
    }

    @Override
    protected WebApplicationContext createServletApplicationContext() {
        XmlWebApplicationContext cxt = new XmlWebApplicationContext();
        cxt.setConfigLocation("classpath:spring-mvc.xml");
        return cxt;
    }

    @Override
    protected String[] getServletMappings() {
        return new String[] { "/*" };
    }
}
```

上面这段配置相当于方法一，getRootConfigClasses 相当于 context-param 中的配置，getServletConfigClasses 相当于 servlet 中配置。

## 3.配置 Filter
```java
public class MyWebAppInitializer extends AbstractDispatcherServletInitializer {
    ...
    @Override
    protected Filter[] getServletFilters() {
        return new Filter[] {
                new HiddenHttpMethodFilter(), new CharacterEncodingFilter() };
    }
}
```



