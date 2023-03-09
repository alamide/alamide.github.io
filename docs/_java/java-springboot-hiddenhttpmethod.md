---
layout: post
title: SpringBoot 表单提交 PUT, DELETE 解决方案
categories: java
excerpt: SpringBoot 使用 HiddenHttpMethodFilter 来解决表单提交 PUT, DELETE
tags: Java SpringBoot
---

### 1.配置文件中开启
```yaml
spring:
  mvc:
    hiddenmethod:
      filter:
        enabled: true
```
### 2.表单中的配置
注意这里需要携带隐藏参数 `_method=DELETE`，这个参数名也可以修改，需要自己配置
```html
<form  action="http://localhost:8080/user" method="POST">
    <input name="_method" value="DELETE" type="hidden" />
    <input type="submit" value="提交"/>
</form>
```

### 3.原理
配置开启之后，请求经过 `HiddenHttpMethodFilter` 过滤器，执行方法 `doFilterInternal`，重新生成新的 `HttpServletRequest`

```java
public class HiddenHttpMethodFilter extends OncePerRequestFilter {

  public static final String DEFAULT_METHOD_PARAM = "_method";

	private String methodParam = DEFAULT_METHOD_PARAM;

  @Override
	protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain)
			throws ServletException, IOException {

		HttpServletRequest requestToUse = request;

		if ("POST".equals(request.getMethod()) && request.getAttribute(WebUtils.ERROR_EXCEPTION_ATTRIBUTE) == null) {
			//this.method = "_method"
      String paramValue = request.getParameter(this.methodParam);
			if (StringUtils.hasLength(paramValue)) {
				String method = paramValue.toUpperCase(Locale.ENGLISH);
				if (ALLOWED_METHODS.contains(method)) {
          //这里修改 HttpServletRequest 的 method 
					requestToUse = new HttpMethodRequestWrapper(request, method);
				}
			}
		}

		filterChain.doFilter(requestToUse, response);
	}
}
```
### 4.修改隐藏参数名
`@ConditionalOnMissingBean(HiddenHttpMethodFilter.class)` 容器中没有 `HiddenHttpMethodFilter` 时注入。
```java
public class WebMvcAutoConfiguration {
	@Bean
	@ConditionalOnMissingBean(HiddenHttpMethodFilter.class)
	@ConditionalOnProperty(prefix = "spring.mvc.hiddenmethod.filter", name = "enabled", matchIfMissing = false)
	public OrderedHiddenHttpMethodFilter hiddenHttpMethodFilter() {
		return new OrderedHiddenHttpMethodFilter();
	}
}
```
所以可以在自己的配置类中配置
```java
@Configuration
@ImportResource
@EnableConfigurationProperties(MySQLConfig.class)
public class MyConfig {
    @Bean
    public HiddenHttpMethodFilter hiddenHttpMethodFilter(){
        HiddenHttpMethodFilter methodFilter = new HiddenHttpMethodFilter();
        methodFilter.setMethodParam("_restMethod");
        return methodFilter;
    }
}
```
这样表单可以这么写
```html
<form  action="http://localhost:8080/user" method="POST">
    <input name="_restMethod" value="DELETE" type="hidden" />
    <input type="submit" value="提交"/>
</form>
```