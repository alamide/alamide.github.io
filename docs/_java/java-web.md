---
layout: post
title: Java Web 
categories: java
tags: Java JavaWeb
date: 2023-05-09
---
* JavaWeb 三大组件 Servlet、Filter、Listener

* JavaWeb 四大作用域 pageContext(仅限 JSP 页面)、request(HttpServletRequest)、session(HttpSession)、application(ServletContext)
<!--more-->

## 1.依赖引入
后续开发都是使用 SpringBoot 直接打成 jar 包，这里就不深究 Tomcat 的配置了

* 使用Servlet<=4.0时，选择Tomcat 9.x或更低版本；

* 使用Servlet>=5.0时，选择Tomcat 10.x或更高版本。
  
```xml
<dependency>
    <groupId>jakarta.servlet</groupId>
    <artifactId>jakarta.servlet-api</artifactId>
    <version>5.0.0</version>
    <scope>provided</scope>
</dependency>
```

## 2.Servlet
Servlet 是 JavaEE 规范（接口）之一，是运行在服务器上的一个 Java 程序，可以接收客户端发来的请求，并响应数据给客户端

### 2.1 声明 Servlet
声明一个 Servlet ，Servelt3.0 之前需要在 web.xml 中声明，Servlet3.0 之后直接使用注解 @WebServlet 即可 
```java
@WebServlet("/")
public class HelloServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        resp.getWriter().write("Hello Servlet!");
        resp.getWriter().flush();
    }
}
```

Servelt 的生命周期
```java
@Slf4j
@WebServlet("/")
public class HelloServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        resp.getWriter().write("Hello Servlet!");
    }

    @Override
    public void init() throws ServletException {
        super.init();
        log.info("HelloServlet.init()");
    }

    @Override
    protected void service(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        super.service(req, resp);
        log.info("HelloServlet.service()");
    }

    @Override
    public void destroy() {
        super.destroy();
        log.info("HelloServlet.destroy()");
    }
}
```

第一次访问
```
[http-nio-8080-exec-10] INFO com.alamide.web.HelloServlet - HelloServlet.init()
[http-nio-8080-exec-10] INFO com.alamide.web.HelloServlet - HelloServlet.service()
```

第二次访问
```
[http-nio-8080-exec-10] INFO com.alamide.web.HelloServlet - HelloServlet.service()
```

关闭服务器
```
[main] INFO com.alamide.web.HelloServlet - HelloServlet.destroy()
```

Servlet 实例只会在初次访问时创建，destory 只会在容器停止时执行，service 每次访问时执行

配置初始化参数
```java
@Slf4j
@WebServlet(value = "/", initParams = {@WebInitParam(name = "username", value = "alamide")})
public class HelloServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        resp.getWriter().write("Hello Servlet!");
        final String username = getInitParameter("username");
        log.info("username is {}", username);
    }
}
```

自己注册 Servrlet 及配置 Servlet 中参数，ServletContext 全局唯一
```java
@Slf4j
@HandlesTypes({})
public class MyServletContainerInitializer implements ServletContainerInitializer {
    @Override
    public void onStartup(Set<Class<?>> c, ServletContext ctx) throws ServletException {
        ctx.setInitParameter("password", "123456");
        final ServletRegistration.Dynamic dynamic = ctx.addServlet("registerServlet", RegisterServlet.class);
        dynamic.addMapping("/register");
        log.info("MyServletContainerInitializer.onStartup()");
    }
}
```

需要在创建 resources/META/services/jakarta.servlet.ServletContainerInitializer文件， 并在文件中加上 com.alamide.web.MyServletContainerInitializer，才会生效

### 2.2 HttpServletRequest
每次只要有请求进⼊ Tomcat 服务器，Tomcat 服务器就会把请求发来的 HTTP 协议信息解析好封装到 Request 对象中，然后传递到 service ⽅法中 (调⽤ doGet 或 doPost ⽅法) 供编程⼈员使⽤，编程⼈员通过 HttpServletRequest 对象，可以获取到请求的所有信息
```java
@Override
protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    resp.getWriter().write("Hello Servlet!");
    final String username = getInitParameter("username");
    log.info("username is {}", username);
    //获取请求的资源路径
    log.info("requestURI: {}", req.getRequestURI());
    //获取请求的绝对路径
    log.info("getRequestURL: {}", req.getRequestURL());
    //获取客户端 IP 地址
    log.info("getRemoteHost: {}", req.getRemoteHost());
    //获取请求头
    req.getHeaderNames().asIterator().forEachRemaining(s -> log.info("Header {}", s));
    //获取请求方法
    log.info("getMethod: {}", req.getMethod());
}
```

getAttribute() 获取域数据
setAttribute(key, value) 设置域数据

请求转发，浏览器栏地址不变，可以在 request 域中设置数据，在转发后的 servlet 中获取
```java
final RequestDispatcher requestDispatcher = req.getRequestDispatcher("/register");
requestDispatcher.forward(req, resp);
```

### 2.3 HttpServletResponse
#### 2.3.1 简单使用
每次只要有请求进⼊ Tomcat 服务器，Tomcat 服务器就会创建⼀个 Response 对象传递给 Servlet 程序。HttpServletResponse 表示所有响应的信息 (HttpServletRequest 表示请求发来的信息)，可以通过 HttpServletResponse 对象设置返回给客户端的信息 
```java
//可以避免乱码
resp.setContentType("text/html;charset=UTF-8");
resp.getWriter().write("你好 Servlet!");
```

getOutputStream() 用于传递二进制数据，getWriter 用于传递字符流，不可同时使用

请求重定向，通知客户端去访问新的地址
```java
resp.sendRedirect("http://localhost:8080/javaweb/register");
```

#### 2.3.2 乱码问题

POST 请求乱码
```java
req.setCharacterEncoding("utf-8");
```

响应乱码
```java
resp.setContentType("text/html;charset=UTF-8");
```

其它方法 addCookie、addHeader、setHeader、setStatus

#### 2.3.3 Session

在Web应用程序中，我们经常要跟踪用户身份。当一个用户登录成功后，如果他继续访问其他页面，Web程序如何才能识别出该用户身份？

因为HTTP协议是一个无状态协议，即Web应用程序无法区分收到的两个HTTP请求是否是同一个浏览器发出的。为了跟踪用户状态，服务器可以向浏览器分配一个唯一ID，并以Cookie的形式发送到浏览器，浏览器在后续访问时总是附带此Cookie，这样，服务器就可以识别用户身份。

我们把这种基于唯一ID识别用户身份的机制称为Session。每个用户第一次访问服务器后，会自动获得一个Session ID。如果用户在一段时间内没有访问服务器，那么Session会自动失效，下次即使带着上次分配的Session ID访问，服务器也认为这是一个新用户，会分配新的Session ID。
```java
protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
    final HttpSession session = req.getSession();
    log.info("session is {}, isNew={}, id={}", session, session.isNew(), session.getId());
    Cookie cookie = new Cookie("userId", UUID.randomUUID().toString());
    cookie.setPath("/");
    cookie.setMaxAge(Math.toIntExact(ChronoUnit.DAYS.getDuration().toSeconds()));
    resp.addCookie(cookie);
    resp.getWriter().write("你好呀！");
    resp.getWriter().flush();
}
```

Session 销毁可以调用 `session.invalidate();` ，设置默认过期时间 `servletContext.setSessionTimeout(10);` ，设置超时时间 
`session.setMaxInactiveInterval(1800);`
## 3.Filter
三大组件之一，拦截请求、过滤响应，常用来权限检查、事务管理、日志操作等
```java
//响应编码
@Slf4j
@WebFilter(filterName = "responseFilter", urlPatterns = {"/*"})
public class HelloFilter implements Filter {
    //Web 工程启动时执行
    @Override
    public void init(FilterConfig filterConfig) throws ServletException {
        log.info("HelloFilter.init()");
    }

    //每次请求符合时
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain) throws IOException, ServletException {
        log.info("HelloFilter.doFilter()");
        response.setContentType("text/html;charset=utf-8");
        chain.doFilter(request, response);
    }

    //关闭工程时
    @Override
    public void destroy() {
        log.info("HelloFilter.destroy()");
    }
}

//登录检查
@Slf4j
@WebFilter(filterName = "loginCheckFilter", urlPatterns = {"/*"})
public class LoginCheckFilter implements Filter {
    private static final AntPathMatcher PATH_MATCHER = new AntPathMatcher();

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {

        HttpServletRequest request = (HttpServletRequest) servletRequest;
        HttpServletResponse response = (HttpServletResponse) servletResponse;

        final String requestURI = request.getRequestURI();

        log.info("request uri: {}", requestURI);
        String[] urls = new String[]{
                "/backend/**",
                "/front/**",
                "/employee/login",
                "/employee/logout",
                "/user/sendMsg",
                "/user/login"
        };

        if (isMatch(urls, requestURI)) {
            log.info("本次请求不需要处理 {}", requestURI);
            filterChain.doFilter(request, response);
            return;
        }

        final Long employeeId = (Long) request.getSession().getAttribute("employeeId");
        if (employeeId != null) {
            log.info("后台用户: {}，已登录", employeeId);
            filterChain.doFilter(request, response);
            return;
        }

        final Long userId = (Long) request.getSession().getAttribute("userId");

        if (userId != null) {
            log.info("前台用户：{}, 已登录", userId);
            BaseContext.setCurrentId(userId);
            filterChain.doFilter(request, response);
            return;
        }

        response.getWriter().write(new ObjectMapper().writeValueAsString(R.error("NOTLOGIN")));
    }

    private boolean isMatch(String[] urls, String requestURI) {
        for (String url : urls) {
            if (PATH_MATCHER.match(url, requestURI)) {
                return true;
            }
        }
        return false;
    }
}
```

一些匹配规则，`*.html` (以 .html 结尾)、`*.jsp` (以 .jsp 结尾)、`/admin/*`

## 4.ServletContextListener
ServletContextListener 监听器可以监听 ServletContext 对象的创建和销毁 (web ⼯程启动时创建，停⽌时销毁)，监听到创建和销毁之后都会调⽤ ServletContextListener 监听器的⽅法进⾏反馈。可以初始化WebApp，例如打开数据库连接池等。
```java
@Slf4j
@WebListener
public class HelloListener implements ServletContextListener {
    @Override
    public void contextInitialized(ServletContextEvent sce) {
        log.info("HelloListener.contextInitialized() {}", sce.getServletContext());
    }


    @Override
    public void contextDestroyed(ServletContextEvent sce) {
        log.info("HelloListener.contextDestroyed() {}", sce);
    }
}
```