---
layout: post
title: SpringMVC Request
categories: java
tags: Java SpringMVC
date: 2023-05-10
---
## 1.@Controller
`SpringMVC` 使用 `@Controller` 和 `@RestController` 来表示一个请求映射、请求输入、异常处理。要使 `@Controller` 生效，需要将被注解的类加入到 Spring 容器中。
```java
@Configuration
@ComponentScan(basePackages = {"com.alamide.web"})
public class WebConfig {

}
``` 

## 2.@RequestMapping
### 2.1 PathPattern
用来将请求映射到具体的 `Controller` 方法上，可以用在类上，也可以用在方法上。用在类上表示共享路径。

`http://localhost:8080/persons/get` 将会映射到 getPerson() 
```java
@Controller
@RequestMapping("/persons")
public class HelloController {

    @ResponseBody
    @RequestMapping("/get")
    public String getPerson(){
        return "getPerson";
    }
}
```

`@RequestMapping` 使用 URL Pattern 的模式映射，有 PathPattern、AntPathMatcher，官方推荐 PathPattern，匹配的规则如下：
1. /ima?e.png 匹配 所有 ima_e.png 请求路径，_为任意字符

2. /*.png 匹配所有以 .png 结尾的请求，注意 image.png 优先匹配 ima?e.png

3. /start/{action}/end 匹配以 start 开头， end 结尾的路径，action 为 URI 变量，可以传入方法的参数

4. /start/{action:[a-z]+}/end 匹配以 start 开头， end 结尾的路径，且中间为字母，action 为 URI 变量，可以传入方法的参数

5. /** 匹配任意路径 /xxx/xxx/xxx 优先级最低

具体映射例子如下
```java
@Controller
@RequestMapping("/persons")
public class HelloController {
    //http://localhost:8080/persons/imaxe.png
    //http://localhost:8080/persons/image.png
    //http://localhost:8080/persons/ima1e.png
    //以上都会映射到这个方法 http://localhost:8080/persons/imae.png 不能映射
    @ResponseBody
    @RequestMapping("/ima?e.png")
    public String pattern1(HttpServletRequest request){
        return "pattern1 : " + request.getRequestURI();
    }

    //http://localhost:8080/persons/ima1e.png 会映射到 pattern1()，而不是 pattern2()


    //http://localhost:8080/persons/imae.png
    //http://localhost:8080/persons/imassse.png
    @ResponseBody
    @RequestMapping("/*.png")
    public String pattern2(HttpServletRequest request){
        return "pattern2 : " + request.getRequestURI();
    }

    //http://localhost:8080/persons/put/10
    @ResponseBody
    @RequestMapping("/**")
    public String pattern3(HttpServletRequest request){
        return "pattern3 : " + request.getRequestURI();
    }

    //http://localhost:8080/persons/start/10/end
    @ResponseBody
    @RequestMapping("/start/{action}/end")
    public String pattern4(HttpServletRequest request){
        return "pattern4 : " + request.getRequestURI();
    }

    //http://localhost:8080/persons/start/aaaaa/end
    @ResponseBody
    @RequestMapping("/start/{action:[a-z]+}/end")
    public String pattern5(HttpServletRequest request){
        return "pattern5 : " + request.getRequestURI();
    }
}
```

URI 变量可以同时在类和方法上声明
```java
@Controller
@RequestMapping("/owners/{ownerId}")
public class OwnerController {

    @GetMapping("/pets/{petId}")
    public Pet findPet(@PathVariable Long ownerId, @PathVariable Long petId) {
        // ...
    }
}
```

URI 变量会自动转换为合适的类型，系统默认支持一些简单类型的转换，如：(int、long、Date、String) 等。

URI 变量的接收可以显示接收，如果方法的变量名和 URI 变量名一致，且代码编译时带有 -parameters，则可以省略

```java
@ResponseBody
@RequestMapping("/start/{action}/end")
public String pattern4(HttpServletRequest request, @PathVariable("action") String action) {
    return "pattern4 : " + request.getRequestURI() + "?action=" + action;
}

//{action} 与 String action 变量名一致，可以不声明
@ResponseBody
@RequestMapping("/start/{action}/end")
public String pattern4(HttpServletRequest request, @PathVariable String action) {
    return "pattern4 : " + request.getRequestURI() + "?action=" + action;
}
```

URI {varName:regex} 模式的小应用，获取名字、版本、扩展名
```java
///spring-web-3.0.5.jar
@GetMapping("/{name:[a-z-]+}-{version:\\d\\.\\d\\.\\d}{ext:\\.[a-z]+}")
public void handle(@PathVariable String name, @PathVariable String version, @PathVariable String ext) {
    // ...
}
```

### 2.2 依据 Content-Type 匹配
pattern6 为 POST 请求，且 Content-Type=application/json 才会匹配到，pattern7 为非 application/json 或不带有 Content-Type 的请求匹配到，consumes 也可以用在顶层 Class 上，如果 Class 和 Method 同时有，则 Method 覆盖 Class
```java
//Content-Type: application/json
@ResponseBody
@PostMapping(path = "/put", consumes = MediaType.APPLICATION_JSON_VALUE)
public String pattern7(HttpServletRequest request){
    return "pattern6 : " + request.getRequestURI();
}

//Content-Type: application/xml
@ResponseBody
@PostMapping(path = "/put", consumes = "!application/json")
public String pattern8(HttpServletRequest request){
    return "pattern7 : " + request.getRequestURI();
}
```

### 2.3 依据 Accept 匹配
Accept 为浏览告知服务器需要返回的数据类型，Accept = application/json 匹配 pattern8，Accept = application/xml 匹配 pattern9，同样可以在 Class 级声明
```java
//Accept: application/json
@ResponseBody
@PostMapping(path = "/put", produces = MediaType.APPLICATION_JSON_VALUE)
public String pattern8(HttpServletRequest request) {
    return "pattern8 : " + request.getRequestURI();
}

//Accept: application/xml
@ResponseBody
@PostMapping(path = "/put", produces = MediaType.APPLICATION_XML_VALUE)
public String pattern9(HttpServletRequest request) {
    return "pattern9 : " + request.getRequestURI();
}
```

### 2.4 依据 Parameters 匹配
匹配的规则为(myParam=myValue)，或 (!myParam)
```java
//http://localhost:8080/persons/get?id=10086
@ResponseBody
@GetMapping(path = "/get", params = {"id=10086"})
public String pattern10(HttpServletRequest request) {
    return "pattern10 : " + request.getRequestURI();
}

//http://localhost:8080/persons/get?id=10001
@ResponseBody
@GetMapping(path = "/get", params = {"id=10001"})
public String pattern11(HttpServletRequest request) {
    return "pattern11 : " + request.getRequestURI();
}

//http://localhost:8080/persons/get?name=yidong
@ResponseBody
@GetMapping(path = "/get", params = {"!id"})
public String pattern12(HttpServletRequest request) {
    return "pattern12 : " + request.getRequestURI();
}
``` 

### 2.5 依据 Headers 匹配
与依据 Parameters 匹配一样
```java
//Headers 中带有自定义 Data-Type: DiyType
@ResponseBody
@PostMapping(path = "/put", headers = {"Data-Type=DiyType"})
public String pattern13(HttpServletRequest request) {
    return "pattern13 : " + request.getRequestURI();
}
```

### 2.6 注册 Handler Methods
官方的例子不能运行，需要配置 PatternParser
```java
@Component
public class UserHandler {
    @ResponseBody
    public String getUser(@PathVariable Long userId) {
        return "UserHandler.getUser() = " + userId;
    }
}

@Configuration
@ComponentScan(basePackages = {"com.alamide.web"})
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    @Autowired
    public void setHandlerMapping(RequestMappingHandlerMapping mapping, UserHandler handler)
            throws NoSuchMethodException {

        RequestMappingInfo.BuilderConfiguration config = new RequestMappingInfo.BuilderConfiguration();
        config.setPatternParser(mapping.getPatternParser());

        RequestMappingInfo info = RequestMappingInfo
                .paths("/user/{userId}")
                .options(config)
                .methods(RequestMethod.GET).build();

        Method method = UserHandler.class.getMethod("getUser", Long.class);
        mapping.registerMapping(info, handler, method);
    }
}
```

## 3.Controller Method Arguments
### 3.1 支持的参数类型
Controller 方法中支持传入的参数，这里记录一些常用的，更多见文档
<table>
  <tr>
    <th>Controller method argument</th>
    <th>Description</th>
  </tr>
  <tr>
    <td>jakarta.servlet.ServletRequest</td>
    <td>可以注入具体的类型，如 HttpServletRequest、MultipartRequest、MultipartHttpServletRequest</td>
  </tr> 
  <tr>
    <td>jakarta.servlet.ServletResponse</td>
    <td>可以注入具体的类型，如 HttpServletResponse</td>
  </tr> 
  <tr>
    <td>jakarta.servlet.http.HttpSession</td>
    <td>非线程安全，具体见文档</td>
  </tr> 
  <tr>
    <td>HttpMethod</td>
    <td>The HTTP method of the request.</td>
  </tr> 
  <tr>
    <td>java.io.InputStream、java.io.Reader</td>
    <td>For access to the raw request body as exposed by the Servlet API.</td>
  </tr> 
  <tr>
    <td>java.io.OutputStream, java.io.Writer</td>
    <td>For access to the raw response body as exposed by the Servlet API.</td>
  </tr> 
  <tr>
    <td>@PathVariable</td>
    <td>获取 URI Pattern 中的变量</td>
  </tr> 
  <tr>
    <td>@MatrixVariable</td>
    <td>获取 URI 中的矩阵变量</td>
  </tr> 
  <tr>
    <td>@RequestParam</td>
    <td>获取请求中的参数变量，包括 POST(username=alamide&email=alamide@163.com) 和 GET(?username=alamide) </td>
  </tr> 
  <tr>
    <td>@RequestHeader</td>
    <td>获取请求的请求头</td>
  </tr> 
  <tr>
    <td>@CookieValue</td>
    <td>获取请求携带的 Cookie</td>
  </tr> 
  <tr>
    <td>@RequestBody</td>
    <td>获取请求体，并将携带的内容转换为具体的类型，通过 HttpMessageConverter 来转换</td>
  </tr> 
</table>

### 3.2 类型实例
#### 3.2.1 @MatrixVariable
如果想获取 URL 中包含矩阵变量，则必须用 URI 变量来获取，即必须以 /{var} 这种形式来获取。

官网讲要配置，矩阵变量才会生效，但是我没做任何配置也生效了，有点奇怪，后面如果遇到解析失败再回头来配置吧
>Note that you need to enable the use of matrix variables. In the MVC Java configuration, you need to set a UrlPathHelper with removeSemicolonContent=false through Path Matching. In the MVC XML namespace, you can set `<mvc:annotation-driven enable-matrix-variables="true"/>`

```java
@Controller
@RequestMapping("/parameter")
public class ParameterController {
    //http://localhost:8080/parameter/pets/12;q=1;q=2;r=100
    //可以得到 petId = 12, q = 1
    @ResponseBody
    @GetMapping("/pets/{petId}")
    public String matrixVar(@PathVariable String petId, @MatrixVariable int q) {
        return "petId = " + petId + ", q = " + q;
    }
}
```

路径多个片段中包含矩阵变量
```java
//http://localhost:8080/parameter/owners/42;q=11/pets/21;q=22
@ResponseBody
@GetMapping("/owners/{ownerId}/pets/{petId}")
public String findPet(
        @MatrixVariable(name="q", pathVar="ownerId") int q1,
        @MatrixVariable(name="q", pathVar="petId") int q2) {
    //q1 = 11, q2 = 22
    return "q1 = " + q1 + ", q2 = " + q2;
}
```

矩阵变量可选及设置默认值，没有矩阵变量时即为默认值，有矩阵变量时为传入的矩阵变量
```java
//http://localhost:8080/parameter/pets/12
@ResponseBody
@GetMapping("/pets/{petId}")
public String findPet(@MatrixVariable(required=false, defaultValue="1") int q) {
    //q = 1
    return "q = " + q;
}
```

获取所有的矩阵变量，一个路径片段中可能会有多个矩阵变量，如 `/pets/12;q=1;h=2;r=100` 第二个片段中含有三个矩阵变量 q、h、r
```java
//http://localhost:8080/parameter/owners/42;q=11;r=12/pets/21;q=22;s=23
@ResponseBody
@GetMapping("/owners/{ownerId}/pets/{petId}")
public String findPet(
        @MatrixVariable MultiValueMap<String, String> matrixVars,
        @MatrixVariable(pathVar="petId") MultiValueMap<String, String> petMatrixVars) {
    //{q=[22, 11], s=[23], r=[12]}, {q=[22], s=[23]}
    return matrixVars.toString() + ", " + petMatrixVars.toString();
}
```

如果不加 pathVar ，则为获取所有的矩阵变量，获取某一片段中的矩阵变量，需要加上 pathVar

#### 3.2.2 @RequestParam
可以在 Controller 的方法参数上加上 @RequestParam 来获取具体的请求参数（Get 或 Form Data）
```java
@Controller
@RequestMapping("/parameter")
public class ParameterController {

    //http://localhost:8080/parameter?id=10
    @ResponseBody
    @GetMapping
    public String get(@RequestParam("id") String id) {
        //10
        return id;
    }
}
```

注意 URL 中的参数 id 不能省略，否则会请求失败，如果允许缺失，可以配置 `@RequestParam(value = "id", required = false) String id` 或 `@RequestParam("id") Optional<String> id`

可以用 `Map<String, String>` 或 `MultiValueMap<String, String>` 获取所有参数
```java
//http://localhost:8080/parameter/all?id=10&username=alamide&email=alamide@163.com&username=alami
@ResponseBody
@GetMapping("/all")
public String getAll(@RequestParam MultiValueMap<String, String> params){
    //{id=[10], username=[alamide, alami], email=[alamide@163.com]}
    return params.toString();
}
```

#### 3.2.3 @RequestHeader
可以获取 HTTP 请求的请求头信息
```java
//http://localhost:8080/parameter/header
@ResponseBody
@GetMapping("/header")
public String getHeader(@RequestHeader("User-Agent") String userAgent) {
    //Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/16.4 Safari/605.1.15
    return userAgent;
}
```

获取所有请求头信息
```java
@ResponseBody
@GetMapping("/header")
public String getHeader(@RequestHeader("User-Agent") String userAgent, @RequestHeader MultiValueMap<String, String> headers) {
    return userAgent + "<br/>" + headers;
}
```

如果请求头中含有的信息可以分解成数组，还可以这么获取
```java
@ResponseBody
@GetMapping("/header")
public String getHeader(@RequestHeader("User-Agent") String userAgent,
                        @RequestHeader MultiValueMap<String, String> headers,
                        @RequestHeader("Accept") String[] accepts) {
    return userAgent + "<br/>" + headers + "<br/>" + Arrays.toString(accepts);
}
```

#### 3.2.3 @CookieValue
获取请求带有的 Cookie 信息
```java
//http://localhost:8080/parameter/cookie
@ResponseBody
@GetMapping("/cookie")
public String Cookie(@CookieValue("JSESSIONID") String sessionId) {
    //A73B9D2263AF2B16622EFA33A1790C62
    return sessionId;
}
```

#### 3.2.4 @ModelAttribute
>You can use the @ModelAttribute annotation on a method argument to access an attribute from the model or have it be instantiated if not present. The model attribute is also overlain with values from HTTP Servlet request parameters whose names match to field names. This is referred to as data binding, and it saves you from having to deal with parsing and converting individual query parameters and form fields.

大概就是在 @Controller 的方法参数上使用 @ModelAttribute ，表示从 Model 对象中获取相应的对象，如果 Model 中不存在，则实例化一个对象。对象中属性有请求参数中对应的值覆盖。
```java
@Data
@ToString
@AllArgsConstructor
@NoArgsConstructor
public class Pet {
    private Long ownerId;
    private Long petId;
    private String name;
}

@Controller
@RequestMapping("/model")
public class ModelAttributeController {
    //http://localhost:8080/model/owners/85758/pets/378/edit
    @ResponseBody
    @GetMapping("/owners/{ownerId}/pets/{petId}/edit")
    public String processSubmit(@ModelAttribute Pet pet) {
        //Pet(ownerId=85758, petId=378, name=tom)
        return pet.toString();
    }

    @ModelAttribute
    public Pet pet(){
        return new Pet(1L, 2L, "tom");
    }
}
```

可以得知 Pet 对象先是由 Model 中获取，获取之后再赋值，@ModelAttribute 在方法上定义，则这个方法在 @Controller 的所有方法之前执行。

>One alternative to using a @ModelAttribute method to supply it or relying on the framework to create the model attribute, is to have a Converter<String, T> to provide the instance.

这段看了好久，主要是英文水平不济，断句没断好，还有就是官网给的例子有点迷惑性。上面这段英文的意思是，除了 @ModelAttribute 标记方法和框架创建实例外，还可以使用 Converter<String, T> 来提供实例
```java
public class StringToAccountConverter implements Converter<String, Account> {
    @Override
    public Account convert(String source) {
        return getAccount(source);
    }

    //mock get Account from db
    private Account getAccount(String acc){
        Account account = new Account();
        account.setAccount(acc);
        account.setBalance(acc.length());
        return account;
    }
}
//注册 Converter
@Configuration
@ComponentScan(basePackages = {"com.alamide.web"})
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addFormatters(FormatterRegistry registry) {
        registry.addConverter(new StringToAccountConverter());
    }
}

@ResponseBody
@GetMapping("/accounts/{account}")
public String save(@ModelAttribute("account") Account account) {
    return account.toString();
}
```

迷惑的地方在于即使 @ModelAttribute 不加 name ，同样是使用 Converter 来转化。开始有点多此一举的感觉。
```java
//同等效果
@ResponseBody
@GetMapping("/accounts/{account}")
public String save(@ModelAttribute Account account) {
    return account.toString();
}
```

查看 @ModelAttribute 源码之后，原来不指定 name 时，`The default model attribute name is inferred from the declared attribute type.` 名字为类型的首字母小写，所以上面的 @ModelAttribute 的 name 为 account，加与不加没什么区别

把上面的例子改成下面这样或许会清晰一点
```java
//加上 name，且 name 与 PathVariable 同名，Account 实例由 Converter 创建，且 PathVariable 会传递给 convert(String source)
//http://localhost:8080/model/accounts/alamide
@ResponseBody
@GetMapping("/accounts/{accountId}")
public String save(@ModelAttribute("accountId") Account account) {
    //Account(account=alamide, balance=7)
    return account.toString();
}


//不加 name，Account 实例由 Framework 或 @ModelAttribute method 创建
//http://localhost:8080/model/accounts/alamide
@ResponseBody
@GetMapping("/accounts/{accountId}")
public String save(@ModelAttribute Account account) {
    //Account(account=alamide, balance=null)
    return account.toString();
}

//加 name，且 name 与 PathVariable 同名，Account 实例由 Framework 或 @ModelAttribute method 创建
//http://localhost:8080/model/accounts/alamide
@ResponseBody
@GetMapping("/accounts/{accountId}")
public String save(@ModelAttribute("account") Account account) {
    //Account(account=alamide, balance=null)
    return account.toString();
}
```

数据绑定时可能会发生错误，可以使用 BindingResult 来接收
```java
@Data
@ToString
@AllArgsConstructor
@NoArgsConstructor
public class Pet {
    private Long ownerId;
    private Long petId;
    private String name;
}

//http://localhost:8080/model/owners/weddd/pets/378/edit
//类型转换错误，ownerId weddd 不能转换为 Long
@ResponseBody
@GetMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute Pet pet, BindingResult result) {
    if (result.hasErrors()) {
        return "Bind Error";
    }
    return pet.toString();
}
```

In some cases, you may want access to a model attribute without data binding. For such cases, you can inject the Model into the controller and access it directly or, alternatively, set @ModelAttribute(binding=false)
```java
//http://localhost:8080/model/owners/2378/pets/378/edit
@ResponseBody
@GetMapping("/owners/{ownerId}/pets/{petId}/edit")
public String processSubmit(@ModelAttribute(binding = false) Pet pet, BindingResult result) {
    if (result.hasErrors()) {
        return "Bind Error";
    }
    //Pet(ownerId=1, petId=2, name=tom)
    return pet.toString();
}

@ModelAttribute
public Pet pet() {
    return new Pet(1L, 2L, "tom");
}
```

在方法上应用 @ModelAttribute，下面这三个方法是等价的，加上这个注解的方法会在所有 Controller 方法之前执行，向 Model 中添加属性
```java
@ModelAttribute
public Pet pet() {
    return new Pet(1L, 2L, "tom");
}

@ModelAttribute("pet")
public Pet pet() {
    return new Pet(1L, 2L, "tom");
}

@ModelAttribute
public void pet(Model model) {
    final Pet pet = new Pet(1L, 2L, "tom");
    model.addAttribute("pet", pet);
}
```

docker run \
    --name nginx80 \
    --network inner-network \
    -v /opt/docker-volume/nginx/logs:/var/log/nginx \
    -v /opt/docker-volume/nginx/conf:/etc/nginx/conf.d:ro \
    -v /opt/docker-volume/nginx/www:/usr/share/nginx/www:ro \
    -p 80:80 \
    -p 443:443 \
    -d nginx:1.24


