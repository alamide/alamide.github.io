---
layout: post
title: SpringBoot Core
categories: java
tags: Java Spring SpringBoot
date: 2023-05-23
---
SpringBoot 的一些特性
<!--more-->

## 1.Banner
可以自定义 `Banner`，在类路径下新增 `banner.txt`，写入自定义的 `banner`。如果不在类路径下，则可以使用 `spring.banner.location` 来指定 `banner.txt` 的位置
```
-------------------------------------------------------------------------------------------
       :\     /;               _                                                          |
      ;  \___/  ;             ; ;                                                         |
     ,:-"'   `"-:.            / ;                                                         |
_   /,---.   ,---.\   _     _; /                                                          |
_:>((  |  ) (  |  ))<:_ ,-""_,"                                                           |
    \`````   `````/""""",-""                                                              |
     '-.._ v _..-'      )                                                                 |
       / ___   ____,..  \                                                                 |
      / /   | |   | ( \. \                                                                |
ctr  / /    | |    | |  \ \                                                               |
     `"     `"     `"    `"                                                               |
-------------------------------------------------------------------------------------------
SpringBoot-Version: ${spring-boot.formatted-version}
```

`banner.txt` 中可以写一写变量，如 `${application.version}` 、`spring-boot.formatted-version` 等

还可以关闭 `Banner` 输出
```java
@SpringBootApplication
public class MainApplication {
    public static void main(String[] args) {
        SpringApplication springApplication = new SpringApplication(MainApplication.class);
        springApplication.setBannerMode(Banner.Mode.OFF);
        springApplication.run(args);
    }
}
```



