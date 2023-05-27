---
layout: post
title: SpringMVC Config
categories: java
tags: Java SpringMVC
date: 2023-05-18
---
SpringMVC 的相关配置
<!--more-->
## 1.Enable MVC Configuration
使配置生效
```java
@Configuration
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
}
```

## 2.Type Conversion
可以使用 `public interface Converter<S, T>` 接口，实现类型转换，可用于在 Controller Method 中，例如客户端传来账号 Id，服务端接收数据后自动转换为 Account
```java
public class StringToAccountConverter implements Converter<String, Account> {
    @Override
    public Account convert(String source) {
        return getAccount(source);
    }

    private Account getAccount(String accountId){
        Account account = new Account();
        account.setAccountId(accountId);
        account.setBalance(accountId.length());
        return account;
    }
}

@Configuration
@ComponentScan(basePackages = {"com.alamide.web"})
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(new StringToAccountConverter());
    }
}
```

## 3.Interceptors
配置 Interceptor
```java
@Configuration
@ComponentScan(basePackages = {"com.alamide.web"})
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        WebMvcConfigurer.super.addInterceptors(registry);
    }
}
```

## 4.ViewResolver
@Controller 的 Method 返回的 String，映射到具体的 View 页面
```java
@Configuration
@ComponentScan(basePackages = {"com.alamide.web"})
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configureViewResolvers(ViewResolverRegistry registry) {
        final UrlBasedViewResolver viewResolver = new UrlBasedViewResolver();
        UrlBasedViewResolverRegistration resolverRegistration = new UrlBasedViewResolverRegistration(viewResolver);
        resolverRegistration.prefix("/WEB-INF/").suffix(".jsp").viewClass(InternalResourceView.class);
        registry.viewResolver(viewResolver);
    }
}
```

## 4.Static Resources
设置静态资源映射
```java
@Configuration
@ComponentScan(basePackages = {"com.alamide.web"})
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/page/**")
                .addResourceLocations("classpath:/static/");
    }
}
```