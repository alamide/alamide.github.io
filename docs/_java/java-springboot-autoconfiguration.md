---
layout: post
title: SpringBoot 自动加载机制
categories: java
excerpt: SpringBoot 自定加载机制原理
tags: Java SpringBoot
---

```java

@SpringBootApplication(scanBasePackages = "com.zhaosiyuan")
public class MainApplication {
}

@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
}

@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
}

public class AutoConfigurationImportSelector implements DeferredImportSelector, BeanClassLoaderAware,
		ResourceLoaderAware, BeanFactoryAware, EnvironmentAware, Ordered {

}

```

最终是去这个文件加载

`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`