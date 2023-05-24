---
layout: post
title: SpringMVC Response
categories: java
tags: Java SpringMVC
date: 2023-05-16
isHidden: true
---
`SpringMVC Response` 相关
<!--more-->
## 1.Controller 支持的返回数据类型
<table>
    <tr>
        <th>Controller method return value</th>
        <th>Description</th>
    </tr>
    <tr>
        <td>@ResponseBody</td>
        <td>返回的数据经由 HttpMessageConverter 转换，会依据 Content-Type 来转换</td>
    </tr>
    <tr>
        <td>HttpEntity&lt;B&gt;, ResponseEntity&lt;B&gt;</td>
        <td>返回由 HttpMessageConverter 转换，且包含 Header 信息</td>
    </tr>
    <tr>
        <td>HttpHeaders</td>
        <td>返回 Header 信息，没有响应体</td>
    </tr>
    <tr>
        <td>String</td>
        <td>视图名</td>
    </tr>
</table>

## 2.@ResponseBody
`@ResponseBody` 可以加载类上和方法上，加在类上表示对所有方法都生效，加在类上可以用 `@RestController(=@Controller + @ResponseBody)` 代替，方法返回的数据通过 `HttpMessageConverter` 来转化

转换规则 [HttpMessageConverter](./java-spring-mvc-http-message-converter.html)

## 3.ResponseEntity
比 `@ResponseBody` 多返回一个头信息和 `Status`，这个返回值可以解决使用 `@ResponseBody` 的一个小问题。使用 `@ResponseBody` 时，如下代码示例
```java
@ResponseBody
@PostMapping("/json")
public UserInfo json(@RequestBody UserInfo userInfo) {
    return userInfo;
}
```

如上代码，我们想返回的内容是 `UserInfo` 的 `json`，如果客户端指定 `Accept: application/json`，那么返回值没有问题，框架会依据 `Accept` 来挑选 `MappingJackson2HttpMessageConverter` 来转换。如果客户端没有指定 `Accept`，那么返回的内容就不可控了，会优先返回符合条件，排在最前面的 `Converter`，本例会返回 `xml` 格式的数据。你可能会想到指定 `response.setContentType("application/json")`，这里不会生效。
```java
@PostMapping("/json")
public UserInfo json(@RequestBody UserInfo userInfo, HttpServletResponse response) {
    //无效，仍会返回 xml 格式的数据
    response.setContentType("application/json");
    return userInfo;
}
```

原因是，框架在封装 `response` 时，并没有将设置的 `Content-Type` 放到 `Headers` 中，所以无法正确的转换
```java
public abstract class AbstractMessageConverterMethodProcessor extends AbstractMessageConverterMethodArgumentResolver
		implements HandlerMethodReturnValueHandler {
    ...
    protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
    ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
    throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
        ...
        //为 null，无法获取到正确的 ContentType
        MediaType contentType = outputMessage.getHeaders().getContentType();
        ...
    }
    ...
}
public abstract class AbstractMessageConverterMethodProcessor extends AbstractMessageConverterMethodArgumentResolver
		implements HandlerMethodReturnValueHandler {
    //创建 OutputMessage 即 ServletServerHttpResponse
    protected ServletServerHttpResponse createOutputMessage(NativeWebRequest webRequest) {
        HttpServletResponse response = webRequest.getNativeResponse(HttpServletResponse.class);
        Assert.state(response != null, "No HttpServletResponse");
        //包装 response
        return new ServletServerHttpResponse(response);
    }
}

public class ServletServerHttpResponse implements ServerHttpResponse {

	private final HttpServletResponse servletResponse;

	private final HttpHeaders headers;

	private boolean headersWritten = false;

	private boolean bodyUsed = false;

	@Nullable
	private HttpHeaders readOnlyHeaders;

	//headers 为 empty
	public ServletServerHttpResponse(HttpServletResponse servletResponse) {
		Assert.notNull(servletResponse, "HttpServletResponse must not be null");
		this.servletResponse = servletResponse;
		this.headers = new ServletResponseHttpHeaders();
	}
    ...

    @Override
	public HttpHeaders getHeaders() {
		if (this.readOnlyHeaders != null) {
			return this.readOnlyHeaders;
		}
		else if (this.headersWritten) {
			this.readOnlyHeaders = HttpHeaders.readOnlyHttpHeaders(this.headers);
			return this.readOnlyHeaders;
		}
		else {
            //empty
			return this.headers;
		}
	}
    ...
}
```

现在使用 `ResponseEntity` 就可以很好的解决这个问题了
```java
@GetMapping("/pet")
public ResponseEntity<Pet> handle(){
    Pet pet = new Pet(1L, 3L, "tom");
    return ResponseEntity.ok().contentType(MediaType.APPLICATION_JSON).body(pet);
}
```

原理
```java
public class HttpEntityMethodProcessor extends AbstractMessageConverterMethodProcessor {
    ...
    @Override
    public void handleReturnValue(@Nullable Object returnValue, MethodParameter returnType,
            ModelAndViewContainer mavContainer, NativeWebRequest webRequest) throws Exception {
        ...
        ServletServerHttpRequest inputMessage = createInputMessage(webRequest);
		ServletServerHttpResponse outputMessage = createOutputMessage(webRequest);
        ...
        HttpHeaders outputHeaders = outputMessage.getHeaders();
        HttpHeaders entityHeaders = httpEntity.getHeaders();
        //将设置的 Content-Type 复制到 outputMessage 的 Header 中，最终解析时可以获取到 MediaType 为 application/json，可以转换为目标类型
        if (!entityHeaders.isEmpty()) {
            entityHeaders.forEach((key, value) -> {
            if (HttpHeaders.VARY.equals(key) && outputHeaders.containsKey(HttpHeaders.VARY)) {
                List<String> values = getVaryRequestHeadersToAdd(outputHeaders, entityHeaders);
                if (!values.isEmpty()) {
                    outputHeaders.setVary(values);
                }
                }
                else {
                    outputHeaders.put(key, value);
                }
            });
        }
    }
    ...
}
```

## 4.Jackson JSON
使用 `@JsonView` ，屏蔽一些敏感信息
```java
@AllArgsConstructor
@NoArgsConstructor
public class User {
    public interface WithoutPassword{}
    public interface WithPassword extends WithoutPassword{}

    private String username;

    private String password;

    @JsonView(WithPassword.class)
    public String getPassword() {
        return password;
    }

    @JsonView(WithoutPassword.class)
    public String getUsername() {
        return username;
    }
}

//屏蔽密码信息
@JsonView(User.WithoutPassword.class)
@ResponseBody
@GetMapping("/user")
public User getUser(){
    return new User("alamide", "123456");
}
```

## 5.HttpHeaders
返回响应头信息，而没有响应体
```java
@GetMapping("/header")
public HttpHeaders header(){
    HttpHeaders httpHeaders = new HttpHeaders();
    httpHeaders.add(HttpHeaders.CONTENT_TYPE, "application/json");
    return httpHeaders;
}
```

## 6.String 视图
SpringMVC 使用 ViewResolver 来实现视图映射，通过视图名映射到具体的页面，要实现映射要注册具体的映射器。现在前后端分离的场景居多，所以这里简略学习一下。
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

映射
```java
@GetMapping("/view")
public String jsp(Model model){
    model.addAttribute("message", "this is a message!");
    return "hello";
}
```

`hello.jsp`
```jsp
<%@ page contentType="text/html;charset=UTF-8" language="java" %>
<html>
  <head>
    <title>Title</title>
  </head>
  <body>
    <!--从 Model 中直接获取-->
    ${message}
  </body>
</html>
```

最终返回具体的页面