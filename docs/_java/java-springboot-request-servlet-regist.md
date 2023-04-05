---
layout: post
title: SpringBoot Request DispatchServlet 注册
categories: java
tags: Java Request DispatchServlet
date: 2023-03-24
---
总在好奇发出的 `Http` 请求， `Spring` 是如何精确的定位到具体的方法的，请求的参数又如何解析的？
<!--more-->
* `Spring Framework` 文档版本 `5.3.26`
* `Spring Boot` 版本 `2.3.4.RELEASE`

## 1.DispatchServlet 何时注册
早期写 `JavaWeb` 的时候，每一个 `Servlet` 都需要在 `web.xml` 中注册，而在 `SpringBoot` 中找了很久却没有找到 `web.xml` ，调试发现 `SpringBoot` 是
采用代码注册的。后面查看文档，发现 `Spring Framework>Web Servlet>Spring MVC>DispatcherServlet` 文档中有写
>The DispatcherServlet, as any Servlet, needs to be declared and mapped according to the Servlet specification by using Java configuration or in web.xml. In turn, the DispatcherServlet uses Spring configuration to discover the delegate components it needs for request mapping, view resolution, exception handling

### 1.1 代码跟踪
在 `DispatcherServletAutoConfiguration` 中向 `Spring` 容器注入 `DispatcherServletRegistrationBean`
```java

public class DispatcherServletAutoConfiguration {

    protected static class DispatcherServletConfiguration {
        @Bean(name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
        public DispatcherServlet dispatcherServlet(WebMvcProperties webMvcProperties) {
          ...
          return dispatcherServlet;
        }
    }

    protected static class DispatcherServletRegistrationConfiguration {
        @Bean(name = DEFAULT_DISPATCHER_SERVLET_REGISTRATION_BEAN_NAME)
        @ConditionalOnBean(value = DispatcherServlet.class, name = DEFAULT_DISPATCHER_SERVLET_BEAN_NAME)
        public DispatcherServletRegistrationBean dispatcherServletRegistration(DispatcherServlet dispatcherServlet,
            WebMvcProperties webMvcProperties, ObjectProvider<MultipartConfigElement> multipartConfig) {
          ...
          return registration;
        }
    }
}
```

`DispatcherServletRegistrationBean` 实现了 `ServletContextInitializer` 接口

```java
public interface ServletContextInitializer {
	  void onStartup(ServletContext servletContext) throws ServletException;
}
```

可以这么理解， `Spring` 容器会在合适的时机，来通知 `DispatcherServletRegistrationBean` 可以向 `Servlet` 容器中注册 `Servlet` 了

`DispatcherServletRegistrationBean` 的继承关系如下
```java
public class DispatcherServletRegistrationBean extends ServletRegistrationBean<DispatcherServlet>
		implements DispatcherServletPath {
}

public class ServletRegistrationBean<T extends Servlet> extends DynamicRegistrationBean<ServletRegistration.Dynamic> {

    @Override
    protected ServletRegistration.Dynamic addRegistration(String description, ServletContext servletContext) {
        String name = getServletName();
        // Web 容器中注册 Servlet
        return servletContext.addServlet(name, this.servlet);
    }
}

public abstract class DynamicRegistrationBean<D extends Registration.Dynamic> extends RegistrationBean {
    @Override
    protected final void register(String description, ServletContext servletContext) {
      //开始注册
        D registration = addRegistration(description, servletContext);
        if (registration == null) {
          logger.info(StringUtils.capitalize(description) + " was not registered (possibly already registered?)");
          return;
        }
        configure(registration);
    }
}

public abstract class RegistrationBean implements ServletContextInitializer, Ordered {
    /*回调注册*/
    @Override
    public final void onStartup(ServletContext servletContext) throws ServletException {
        String description = getDescription();
        if (!isEnabled()) {
            logger.info(StringUtils.capitalize(description) + " was not registered (disabled)");
            return;
        }
        //注册
        register(description, servletContext);
    }

    protected abstract void register(String description, ServletContext servletContext);
}

public interface ServletContextInitializer {
    void onStartup(ServletContext servletContext) throws ServletException;
}
```
大概向容器中注册 `Servlet` 的流程是这样的

那么何时注册呢？具体又是怎么注册的呢？

在 `run` 启动类后
```java
SpringApplication.run(MainApplication.class, args);

public class SpringApplication {
    public static ConfigurableApplicationContext run(Class<?>[] primarySources, String[] args) {
        return new SpringApplication(primarySources).run(args);
    }

    public ConfigurableApplicationContext run(String... args) {
        ...
        refreshContext(context);
        ...
    }

    private void refreshContext(ConfigurableApplicationContext context) {
        refresh((ApplicationContext) context);
        ...
    }
}
```

经过一系列的 `refresh` 后最终会来到父类的 `onRefresh`

```java
public class ServletWebServerApplicationContext extends GenericWebApplicationContext
		implements ConfigurableWebServerApplicationContext {
    protected void onRefresh() {
        ...
        createWebServer();
        ...
    }

    private void createWebServer() {
        ...
        //这里会获取 WebServerFactory，因为工程引入的是 Tomcat，所以这里为 TomcatServletWebServerFactory
        ServletWebServerFactory factory = getWebServerFactory();
        this.webServer = factory.getWebServer(getSelfInitializer());
        ...
    }
}

public class TomcatServletWebServerFactory extends AbstractServletWebServerFactory
		implements ConfigurableTomcatWebServerFactory, ResourceLoaderAware {
    @Override
	  public WebServer getWebServer(ServletContextInitializer... initializers) { 
        ...
        return getTomcatWebServer(tomcat);
    }
    protected TomcatWebServer getTomcatWebServer(Tomcat tomcat) {
        return new TomcatWebServer(tomcat, getPort() >= 0, getShutdown());
    }
}
```

再来到 `TomcatWebServer` 

```java
public class TomcatWebServer implements WebServer {
    public TomcatWebServer(Tomcat tomcat, boolean autoStart, Shutdown shutdown) {
        ...
        initialize();
    }

    private void initialize() throws WebServerException {
        ...
        this.tomcat.start();
        ...
    }
}

public class Tomcat {
    public void start() throws LifecycleException {
        ...
        server.start();
    }
}
```
经过一系列的方法最终回调到 `TomcatStarter` 的 `onStartup` 方法
```java
/**
 * {@link ServletContainerInitializer} used to trigger {@link ServletContextInitializer
 * ServletContextInitializers} and track startup errors.
 *
 * @author Phillip Webb
 * @author Andy Wilkinson
 */
class TomcatStarter implements ServletContainerInitializer {
    @Override
    public void onStartup(Set<Class<?>> classes, ServletContext servletContext) throws ServletException {
        ...
        for (ServletContextInitializer initializer : this.initializers) {
            initializer.onStartup(servletContext);
        }
        ...
    }
}

public class ServletWebServerApplicationContext extends GenericWebApplicationContext
		implements ConfigurableWebServerApplicationContext {

    private void selfInitialize(ServletContext servletContext) throws ServletException {
        ...
        WebApplicationContextUtils.registerEnvironmentBeans(getBeanFactory(), servletContext);
        //查找类型为 ServletContextInitializer 的 JavaBean
        for (ServletContextInitializer beans : getServletContextInitializerBeans()) {
            beans.onStartup(servletContext);
        }
    }
}

```
可以看到最终会去 `Spring` 容器中查找类型为 `ServletContextInitializer` 的 `JavaBean`，然后调用其 `onStartup` 方法， 而我们的开头提到的 `DispatcherServletRegistrationBean` 正是实现了 `ServletContextInitializer` 接口

### 1.2 注册自己的 Servlet
查看源码，可以知道，要注册自己的 `Servlet` 除了使用原生的注入外，还可以在容器中注入一个 继承 `ServletRegistrationBean` 的 `Java Bean`

```java
public class MyServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        resp.getWriter().write("hallo ya");
    }
}

@Configuration
public class MyConfig {
    @Bean
    public ServletRegistrationBean<MyServlet> myServlet(){
        MyServlet myServlet = new MyServlet();
        return new ServletRegistrationBean<>(myServlet, "/my");
    }
}
/**************************原生方式**************************/
@WebServlet(urlPatterns = "/raw")
public class RawServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
        resp.getWriter().write("raw servlet");
    }
}

@RestController
@ServletComponentScan(basePackages = "com.zhaosiyuan.springboot.servlet")//不要忘记这个注解
@SpringBootApplication(scanBasePackages = "com.zhaosiyuan")
public class MainApplication {
}
```

### 1.3 小结
1. `SpringBoot` 经过一系列的初始化，上下文环境准备完成之后，会发出 `ServletContext` 的初始化通知（即开始回调实现 `ServletContextInitializer` 接口的类），
之后开始向 `Web Server` 中注册 `Servlet` 

2. 如果想修改 `DispatchServlet` 的访问路径，可以通过在 `application.yml` 中修改 `spring.mvc.servlet.path` 来实现

3. 注册自己的 `Servlet` 除了使用原生的注入外，还可以在容器中注入一个 继承 `ServletRegistrationBean` 的 `Java Bean`
