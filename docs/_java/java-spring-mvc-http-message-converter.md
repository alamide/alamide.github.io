---
layout: post
title: SpringMVC HttpMessageConverter
categories: java
tags: Java SpringMVC
date: 2023-05-16
---
SpringWeb 使用 HttpMessageConverter 来转化 HTTP 请求的 Request 和 Response.
<!--more-->
>The spring-web module contains the HttpMessageConverter contract for reading and writing the body of HTTP requests and responses through InputStream and OutputStream. HttpMessageConverter instances are used on the client side (for example, in the RestTemplate) and on the server side (for example, in Spring MVC REST controllers).

## 1.框架提供的一些转化器
框架为主要的数据类型(MIME)提供一些具体的实现类。
<table>
<tr>
<th>MessageConverter</th>
<th>Description</th>
</tr>
<tr>
<td>StringHttpMessageConverter</td>
<td>Request: Content-Type: text/*; Response: Content-Type: text/plain</td>
</tr>
<tr>
<td>FormHttpMessageConverter</td>
<td>Content-Type: application/x-www-form-urlencoded; 读取表单中的数据</td>
</tr>
<tr>
<td>MappingJackson2HttpMessageConverter</td>
<td>Request: 将 RequestBody 的 Json 转换为对象；Response: 将对象转换为 Json 字符串</td>
</tr>
<td>MappingJackson2XmlHttpMessageConverter</td>
<td>Request: 将 RequestBody 的 xml 转换为对象；Response: 将对象转换为 xml 字符串</td>
</tr>
</table>

## 2.Request Converter
标记为 @RequestBody 的 Controller Method Parameter
### 2.1 StringHttpMessageConverter
可以将请求体中的内容转换为 String，Request 的 Content-Type: text/*，只要请求的类型为 text，都可以转换为字符串

```
POST /message/string HTTP/1.1
Accept: */*
Content-Length: 23
Content-Type: text/plain
Host: localhost:8080

Payload: username=alamide&age=18
```

```java
@PostMapping("/string")
public void stringMessage(@RequestBody String content){
    //username=alamide&age=18
    log.info(content);
}
```

上面的请求指定了 Content-Type: text/plain， 所以只能转换为 String 或 byte[]，即只有 StringHttpMessageConverter 和 ByteArrayHttpMessageConverter 支持，不能转换为 UserInfo。所有 text/* 都可以转换为 String 。

### 2.2 FormHttpMessageConverter
可以将 Form 表单提交的 Content-Type: application/x-www-form-urlencoded 数据转换为 MultiValueMap<String, String>，
```
POST /message/form HTTP/1.1
Accept: */*
Content-Length: 23
Content-Type: application/x-www-form-urlencoded
Host: localhost:8080

Payload: username=alamide&age=18
```

```java
@PostMapping("/form")
public void form(@RequestBody MultiValueMap<String, String> multiValueMap) {
    //{username=[alamide], age=[20]}
    log.info(multiValueMap.toString());
}
```

### 2.3 MappingJackson2HttpMessageConverter
将请求体中的 Json 数据转换为被 @RequestBody 标记的参数对象 
```
POST /message/json HTTP/1.1
Accept: */*
Content-Length: 31
Content-Type: application/json
Host: localhost:8080

Payload: {
  "username": "alamide",
  "age": 18
}
```

```java
@PostMapping("/json")
public void json(@RequestBody UserInfo userInfo) {
    //UserInfo(username=alamide, age=18)
    log.info(userInfo.toString());
}
```

### 2.4 MappingJackson2XmlHttpMessageConverter
将请求体中的 Xml 数据转换为被 @RequestBody 标记的参数对象 
```
POST /message/xml HTTP/1.1
Accept: */*
Content-Length: 79
Content-Type: application/xml
Host: localhost:8080

Payload: 
<Pet>
    <ownerId>1</ownerId>
    <petId>2</petId>
    <name>tom</name>
</Pet>
```

```java
@PostMapping("/xml")
public void xmlDe(@RequestBody Pet pet) {
    //Pet(ownerId=1, petId=2, name=tom)
    log.info(pet.toString());
}
```

### 2.5 HttpMessageConverter 工作原理
HttpMessageConverter 是在 WebMvcConfigurationSupport 中注册的
```java
public class WebMvcConfigurationSupport implements ApplicationContextAware, ServletContextAware {
     static {
        ...
        ClassLoader classLoader = WebMvcConfigurationSupport.class.getClassLoader();
        //查看类路径下是否有目标类
        jackson2Present = ClassUtils.isPresent("com.fasterxml.jackson.databind.ObjectMapper", classLoader) && ClassUtils.isPresent("com.fasterxml.jackson.core.JsonGenerator", classLoader);
        jackson2XmlPresent = ClassUtils.isPresent("com.fasterxml.jackson.dataformat.xml.XmlMapper", classLoader);
        gsonPresent = ClassUtils.isPresent("com.google.gson.Gson", classLoader);
        ...
    }
     protected final void addDefaultHttpMessageConverters(List<HttpMessageConverter<?>> messageConverters) {
        messageConverters.add(new ByteArrayHttpMessageConverter());
        messageConverters.add(new StringHttpMessageConverter());
        messageConverters.add(new ResourceHttpMessageConverter());
        messageConverters.add(new ResourceRegionHttpMessageConverter());
        messageConverters.add(new AllEncompassingFormHttpMessageConverter());
        ...
        //如果目标类存在，则注册
        if (jackson2XmlPresent) {
            builder = Jackson2ObjectMapperBuilder.xml();
            if (this.applicationContext != null) {
                builder.applicationContext(this.applicationContext);
            }

            messageConverters.add(new MappingJackson2XmlHttpMessageConverter(builder.build()));
        } else if (jaxb2Present) {
            messageConverters.add(new Jaxb2RootElementHttpMessageConverter());
        }
        ...
        //如果目标类存在，则注册
        if (jackson2Present) {
            builder = Jackson2ObjectMapperBuilder.json();
            if (this.applicationContext != null) {
                builder.applicationContext(this.applicationContext);
            }

            messageConverters.add(new MappingJackson2HttpMessageConverter(builder.build()));
        } else if (gsonPresent) {
            messageConverters.add(new GsonHttpMessageConverter());
        } else if (jsonbPresent) {
            messageConverters.add(new JsonbHttpMessageConverter());
        }
     }
}
```

Debug 调试可以看到默认注册 7 个转换器
![converts](../assets/imgs/java-spring-mvc-http-message-convert.jpg)

```
ByteArrayHttpMessageConverter: Content-Typ: application/octet-stream, */*; ParameterType: byte[]

StringHttpMessageConverter: Content-Typ: text/plain, */*;ParameterType: String

MappingJackson2XmlHttpMessageConverter: Content-Typ: application/xml, text/xml, application/*+xml;ParameterType: objectMapper.canDeserialize

MappingJackson2HttpMessageConverter: Content-Typ: application/json, application/*+json;ParameterType: objectMapper.canDeserialize
```

请求体匹配时，按照这个顺序依次匹配，匹配的过程为
```java
public abstract class AbstractMessageConverterMethodArgumentResolver implements HandlerMethodArgumentResolver {
    @Nullable
    @SuppressWarnings({ "unchecked", "rawtypes" })
    protected <T> Object readWithMessageConverters(HttpInputMessage inputMessage, MethodParameter parameter,
            Type targetType) throws IOException, HttpMediaTypeNotSupportedException, HttpMessageNotReadableException {
        for (HttpMessageConverter<?> converter : this.messageConverters) {
            Class<HttpMessageConverter<?>> converterType = (Class<HttpMessageConverter<?>>) converter.getClass();
            GenericHttpMessageConverter<?> genericConverter =
                    (converter instanceof GenericHttpMessageConverter ghmc ? ghmc : null);
            //查看是否支持类型转换
            if (genericConverter != null ? genericConverter.canRead(targetType, contextClass, contentType) :
                    (targetClass != null && converter.canRead(targetClass, contentType))) {
                if (message.hasBody()) {
                    HttpInputMessage msgToUse =
                            getAdvice().beforeBodyRead(message, parameter, targetType, converterType);
                    body = (genericConverter != null ? genericConverter.read(targetType, contextClass, msgToUse) :
                            ((HttpMessageConverter<T>) converter).read(targetClass, msgToUse));
                    body = getAdvice().afterBodyRead(body, msgToUse, parameter, targetType, converterType);
                }
                else {
                    body = getAdvice().handleEmptyBody(null, message, parameter, targetType, converterType);
                }
                break;
            }
        }
        
    }
}
```

## 3.Response Converter
标记为 @ResponseBody 的 Controller Method ReturnValue
### 3.1 StringHttpMessageConverter
将返回的数据以 String 写回响应流
```java
@ResponseBody
@PostMapping("/string")
public String stringMessage(@RequestBody String content) {
    return content;
}
```

执行的流程，messageConverters 还是上图的那几个
```java
public abstract class AbstractMessageConverterMethodProcessor extends AbstractMessageConverterMethodArgumentResolver
		implements HandlerMethodReturnValueHandler {
    protected List<MediaType> getProducibleMediaTypes(
            HttpServletRequest request, Class<?> valueClass, @Nullable Type targetType) {
        Set<MediaType> mediaTypes =
                (Set<MediaType>) request.getAttribute(HandlerMapping.PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE);
        if (!CollectionUtils.isEmpty(mediaTypes)) {
            return new ArrayList<>(mediaTypes);
        }
        Set<MediaType> result = new LinkedHashSet<>();
        //查询支持返回数据类型的转换器 
        //StringHttpMessageConverter
        //MappingJackson2HttpMessageConverter
        //MappingJackson2XmlHttpMessageConverter
        for (HttpMessageConverter<?> converter : this.messageConverters) {
            if (converter instanceof GenericHttpMessageConverter<?> ghmc && targetType != null) {
                if (ghmc.canWrite(targetType, valueClass, null)) {
                    result.addAll(converter.getSupportedMediaTypes(valueClass));
                }
            }
            else if (converter.canWrite(valueClass, null)) {
                result.addAll(converter.getSupportedMediaTypes(valueClass));
            }
        }
        //最终 result 的类型为 下图
        return (result.isEmpty() ? Collections.singletonList(MediaType.ALL) : new ArrayList<>(result));
    }
}
```

![response-convert](../assets/imgs/java-spring-mvc-http-message-response.jpg)

```java
public abstract class AbstractMessageConverterMethodProcessor extends AbstractMessageConverterMethodArgumentResolver
		implements HandlerMethodReturnValueHandler {
    protected <T> void writeWithMessageConverters(@Nullable T value, MethodParameter returnType,
        ServletServerHttpRequest inputMessage, ServletServerHttpResponse outputMessage)
        throws IOException, HttpMediaTypeNotAcceptableException, HttpMessageNotWritableException {
        ...
        MediaType contentType = outputMessage.getHeaders().getContentType();
        boolean isContentTypePreset = contentType != null && contentType.isConcrete();
        //response 是否已经设置了 Content-Type
        if (isContentTypePreset) {
            if (logger.isDebugEnabled()) {
                logger.debug("Found 'Content-Type:" + contentType + "' in response");
            }
            selectedMediaType = contentType;
        }
        //没有设置的话，则从请求头中读取 Accept 请求头
        else {
            HttpServletRequest request = inputMessage.getServletRequest();
            List<MediaType> acceptableTypes;
            try {
                //读取 Accept 请求头信息
                acceptableTypes = getAcceptableMediaTypes(request);
            }
            catch (HttpMediaTypeNotAcceptableException ex) {
                ...
            }
            //获取 Convert 中支持返回类型的所有 MediaType，具体查找方法见上面代码块
            List<MediaType> producibleTypes = getProducibleMediaTypes(request, valueType, targetType);
            ...

            List<MediaType> compatibleMediaTypes = new ArrayList<>();
            //依据 Accept 筛选最终支持的 MediaType
            determineCompatibleMediaTypes(acceptableTypes, producibleTypes, compatibleMediaTypes);
            ...
            //重排 MediaType
            MimeTypeUtils.sortBySpecificity(compatibleMediaTypes);

            for (MediaType mediaType : compatibleMediaTypes) {
                //选择第一个符合条件的 MediaType
                if (mediaType.isConcrete()) {
                    selectedMediaType = mediaType;
                    break;
                }
                else if (mediaType.isPresentIn(ALL_APPLICATION_MEDIA_TYPES)) {
                    selectedMediaType = MediaType.APPLICATION_OCTET_STREAM;
                    break;
                }
            }
            ...
        }
        if (selectedMediaType != null) {
            selectedMediaType = selectedMediaType.removeQualityValue();
            for (HttpMessageConverter<?> converter : this.messageConverters) {
                GenericHttpMessageConverter genericConverter =
                        (converter instanceof GenericHttpMessageConverter ghmc ? ghmc : null);
                //选择第一个合适的 Converter，本次请求第一个符合的是 StringHttpMessageConverter
                if (genericConverter != null ?
                        ((GenericHttpMessageConverter) converter).canWrite(targetType, valueType, selectedMediaType) :
                        converter.canWrite(valueType, selectedMediaType)) {
                    ...
                    //向响应流中写入数据，并这只 Reponse 的 Content-Type
                    ((HttpMessageConverter) converter).write(body, selectedMediaType, outputMessage);
                    ...
                    ...
                    return;
                }
            }
        }
        ...
    }
}
```

最终选出 StringHttpMessageConverter ，向客户端写数据，并且 Response 的 Content-Type 为 text/plain
### 3.2 Content-Type 和 Accept 的灵活使用
可以依据 Accept 灵活的返回数据，依据 Content-Type 序列化请求体，下面的这个请求，可以对应四种不同类型的请求
```java
@PostMapping("/multi")
public UserInfo json(@RequestBody UserInfo userInfo) {
    return userInfo;
}
```

<hr/>

`Request:`
```
POST /message/multi HTTP/1.1
Accept: application/json
Content-Type: application/json
Host: localhost:8080

Payload: {
  "username": "alamide",
  "age": 18
}
```

`Response:`
```
Content-Type: application/json

ResponseBody:
{
    "username": "alamide",
    "age": 18
}
```

<hr/>

`Request:`
```
POST /message/multi HTTP/1.1
Accept: application/xml
Content-Type: application/json
Host: localhost:8080

Payload: {
  "username": "alamide",
  "age": 18
}
```

`Response:`
```
Content-Type: application/xml

ResponseBody:
<UserInfo>
    <username>alamide</username>
    <age>18</age>
</UserInfo>
```

<hr/>

`Request:`
```
POST /message/multi HTTP/1.1
Accept: application/xml
Content-Type: application/xml
Host: localhost:8080

Payload:
<UserInfo>
    <username>alamide</username>
    <age>18</age>
</UserInfo>
```

`Response:`
```
Content-Type: application/xml

ResponseBody:
<UserInfo>
    <username>alamide</username>
    <age>18</age>
</UserInfo>
```

<hr/>

`Request:`
```
POST /message/multi HTTP/1.1
Accept: application/json
Content-Type: application/xml
Host: localhost:8080

Payload:
<UserInfo>
    <username>alamide</username>
    <age>18</age>
</UserInfo>
```

`Response:`
```
Content-Type: application/json

ResponseBody:
{
    "username": "alamide",
    "age": 18
}
```

## 4.自定义 HttpMessageConverter
定义一个可以解析特定格式请求的 Converter
```java
public class DiyHttpMessageConverter implements HttpMessageConverter<Pet> {
    @Override
    public boolean canRead(Class<?> clazz, MediaType mediaType) {
        return clazz.isAssignableFrom(Pet.class) && MediaType.valueOf("text/pet").equalsTypeAndSubtype(mediaType);
    }

    @Override
    public boolean canWrite(Class<?> clazz, MediaType mediaType) {
        return false;
    }

    @Override
    public List<MediaType> getSupportedMediaTypes() {
        return Collections.singletonList(MediaType.valueOf("text/pet"));
    }

    @Override
    public Pet read(Class<? extends Pet> clazz, HttpInputMessage inputMessage) throws IOException, HttpMessageNotReadableException {
        final String data = StreamUtils.copyToString(inputMessage.getBody(), StandardCharsets.UTF_8);
        //ownerId=1;petId=2;name=tom;
        final String[] strings = data.split(";");
        Pet pet = null;
        try {
            pet = clazz.getDeclaredConstructor().newInstance();
            pet.setOwnerId(Long.valueOf(strings[0].split("=")[1]));
            pet.setPetId(Long.valueOf(strings[1].split("=")[1]));
            pet.setName(strings[2].split("=")[1]);
        } catch (Exception e) {
            throw new RuntimeException(e);
        }

        return pet;
    }

    @Override
    public void write(Pet pet, MediaType contentType, HttpOutputMessage outputMessage) throws IOException, HttpMessageNotWritableException {

    }
}

//扩展系统的转换器
@Configuration
@ComponentScan(basePackages = {"com.alamide.web"})
@EnableWebMvc
public class WebConfig implements WebMvcConfigurer {

    //扩展
    @Override
    public void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
        converters.add(new DiyHttpMessageConverter());
    }

    //替换
    // @Override
    // public void configureMessageConverters(List<HttpMessageConverter<?>> converters) {
    //     WebMvcConfigurer.super.configureMessageConverters(converters);
    // }

}

@PostMapping("/pet")
public Pet pet(@RequestBody Pet pet) {
    return pet;
}
```

`Request:`
```
POST /message/pet HTTP/1.1
Accept: application/json
Content-Type: text/pet
Host: localhost:8080

Payload:`
ownerId=1;petId=2;name=tom;
```

`Response:`
```
Content-Type: application/json

ResponseBody:
{
    "ownerId": 1,
    "petId": 2,
    "name": "tom"
}
```
## 5.小总结
SpringMVC 可以依据请求的 Content-Type 和 Accept 来灵活的转换数据

