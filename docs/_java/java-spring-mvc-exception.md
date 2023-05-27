---
layout: post
title: SpringMVC Exception
categories: java
tags: Java SpringMVC
date: 2023-05-22
---
SpringMVC 异常处理相关。
<!--more-->
## 1.Exceptions
处理 @Controller Method 抛出的异常，只能捕获同一个 Controller Method 中抛出的异常
```java
@Controller
public class ExceptionController {

    @ExceptionHandler
    public ResponseEntity<String> handle(ArithmeticException arithmeticException){
        return ResponseEntity.ok("ArithmeticException");
    }

    @ExceptionHandler({IOException.class})
    public ResponseEntity<String> handle2(){
        return ResponseEntity.ok("IOException");
    }

    @ResponseBody
    @GetMapping("/error1")
    public void error(){
        int i = 1/0;
        System.out.println(i);
    }

    @ResponseBody
    @GetMapping("/error2")
    public void error2() throws IOException {
        throw new IOException();
    }
}
```

## 2.Controller Advice
可以捕获指定类、包中的异常。还可以使用 @RestControllerAdvice(@ControllerAdvice + @ResponseBody)，返回的数据使用 message conversion，而不是 html views
```java
@ControllerAdvice(annotations = RestController.class)
public class AdviceController {
    //error.jsp
    @ExceptionHandler(Exception.class)
    public String handle(){
        return "error";
    }

    //error.jsp
    @ExceptionHandler(ArithmeticException.class)
    public String handle2(){
        return "error";
    }
}
```

## 3.Error Page
### 3.1 SimpleMappingExceptionResolver
简单映射，具体异常映射到具体的 view 页面
```java
@Configuration
@ComponentScan(basePackages = {"com.alamide.web"})
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
   @Override
    public void configureHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers) {
        SimpleMappingExceptionResolver resolver = new SimpleMappingExceptionResolver();
        resolver.setDefaultErrorView("error");
        Properties properties = new Properties();
        properties.putIfAbsent(ArithmeticException.class.getCanonicalName(), "error");
        properties.putIfAbsent(IOException.class.getCanonicalName(), "error2");
        resolver.setExceptionMappings(properties);
        resolvers.add(resolver);
    }
}
```

### 3.2 自定义异常处理器
```java
public class CustomExceptionHandler implements HandlerExceptionResolver {
    @Override
    public ModelAndView resolveException(HttpServletRequest request, HttpServletResponse response, Object handler, Exception ex) {
        ModelAndView errorPage = new ModelAndView("errorPage");
        final String message = ex.toString();
        errorPage.addObject("errorMsg", message);
        return errorPage;
    }
}

@Configuration
@ComponentScan(basePackages = {"com.alamide.web"})
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void configureHandlerExceptionResolvers(List<HandlerExceptionResolver> resolvers) {
        resolvers.add(new CustomExceptionHandler());
    }
}
```

### 3.3 未被处理的 Error
配置 error Page，配置全局 error 处理器，配置没有被 HandlerExceptionResolver 处理的 Error
```java
@Controller
public class ErrorController {
    @RequestMapping(value = "/errors", method = RequestMethod.GET)
    public ModelAndView renderErrorPage(HttpServletRequest httpRequest) {
        ModelAndView errorPage = new ModelAndView("errorPage");
        String errorMsg = "";
        int httpErrorCode = getErrorCode(httpRequest);

        switch (httpErrorCode) {
            case 400 -> errorMsg = "Http Error Code: 400. Bad Request";
            case 401 -> errorMsg = "Http Error Code: 401. Unauthorized";
            case 404 -> errorMsg = "Http Error Code: 404. Resource not found";
            case 500 -> errorMsg = "Http Error Code: 500. Internal Server Error";
        }
        errorPage.addObject("errorMsg", errorMsg);
        errorPage.addObject("reason", getErrorMsg(httpRequest));
        return errorPage;
    }

    private int getErrorCode(HttpServletRequest httpRequest) {
        return (Integer) httpRequest.getAttribute("jakarta.servlet.error.status_code");
    }

    private String getErrorMsg(HttpServletRequest httpRequest) {
        return (String) httpRequest.getAttribute("jakarta.servlet.error.message");
    }
}
```

`web.xml`
```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app>
    <error-page>
        <location>/errors</location>
    </error-page>
</web-app>
```