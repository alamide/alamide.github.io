---
layout: post
title: SpringBoot Request 
categories: java
tags: Java Request DispatchServlet
date: 2023-03-25
---



`SpringBoot` 中一般我们的请求都是通过 `DispatchServlet` 处理转发的，我这里追踪的起点是 `doService` 方法。`doService` 中会传入已经封装过的参数
`HttpServletRequest request` 

![doService](/assets/imgs/java-springboot-request-param-doservice.png)

底层会将 `Http` 请求封装到 `HttpServletRequest` 中传入，封装的内容包括 `请求路径` 、`请求方法` 、`请求参数` 、`Headers` 等必要的信息。


`SpringBoot` 请求参数支持的注解有 `@PathVariable` 、 `@RequestHeader` 、 `@ModelAttribute` 、 `@RequestParam` 、 `@MatrixVariable` 、 `@CookieValue` 、 `@RequestBody` ，及参数解析原理，自定义参数解析