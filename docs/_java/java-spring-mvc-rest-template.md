---
layout: post
title: SpringMVC RestTemplate
categories: java
tags: Java SpringMVC
date: 2023-05-22
---
SpringMVC RestTemplate，可以实现多个应用之间 Restful 风格接口调用
<!--more-->
## 1.引入依赖
```xml
<dependency>
    <groupId>com.squareup.okhttp3</groupId>
    <artifactId>okhttp</artifactId>
    <version>3.14.5</version>
</dependency>
```

## 2.配置 RestTemplate
```java
@Bean
public RestTemplate restTemplate() {
    OkHttp3ClientHttpRequestFactory factory = new OkHttp3ClientHttpRequestFactory();
    factory.setConnectTimeout(3000);
    factory.setReadTimeout(3000);
    return new RestTemplate(factory);
}
```

## 3.Request
### 3.1 getForObject
```java
final UserInfo forObject = restTemplate.getForObject("http://localhost:8080/parameter/non-simple?username=alamide&age=18", UserInfo.class);

final String forObject = restTemplate.getForObject("http://localhost:8080/parameter/owners/{ownerId}/pets/{petId}/edit", String.class, 10086, 10001);
```

### 3.2 postForObject
POST 请求
```java
String res = restTemplate.postForObject("http://localhost:8080/request", new Pet(1L, 2L, "tom"), String.class);
System.out.println(res);
```

### 3.3 Headers
POST请求带有 Body 时，需要使用 RequestEntity，设置 Content-Type、Header
```java
RequestEntity<String> requestEntity = RequestEntity
        .post("http://localhost:8080/request")
        .header("Accept", "text/plain")
        .contentType(MediaType.APPLICATION_FORM_URLENCODED)
        .body("name=alamide&age=18");
final ResponseEntity<String> response = restTemplate.exchange(requestEntity, String.class);
System.out.println(response.getBody());
System.out.println(response.getHeaders().getContentType());
```

### 3.4 MultiPart
Content-Type 依据 MultiValueMap 来自动设置
>If the MultiValueMap contains at least one non-String value, the Content-Type is set to multipart/form-data by the FormHttpMessageConverter. If the MultiValueMap has String values the Content-Type is defaulted to application/x-www-form-urlencoded. If necessary the Content-Type may also be set explicitly.

```java
MultiValueMap<String, Object> parts = new LinkedMultiValueMap<>();
parts.add("username", "alamide");
ClassPathResource classPathResource = new ClassPathResource("a.jpeg");
parts.add("avatar", classPathResource);
final String res = restTemplate.postForObject("http://localhost:8080/multipart/form", parts, String.class);
System.out.println(res);
```

### 3.5 File Download
文件下载
```java
final ResponseEntity<org.springframework.core.io.Resource> response = restTemplate.getForEntity("http://localhost:8080/page/upload.html", org.springframework.core.io.Resource.class);
if(HttpStatus.OK.equals(response.getStatusCode())){
    InputStream inputStream = response.getBody().getInputStream();
    OutputStream out = new FileOutputStream(dirPath + "upload.html");
    FileCopyUtils.copy(inputStream, out);
}
```