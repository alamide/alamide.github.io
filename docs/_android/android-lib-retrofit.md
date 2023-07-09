---
layout: post
title: Retrofit Restful 风格的网络请求框架 
categories: android 
tags: android retrofit
date: 2023-07-09
---
Retrofit 是一个 Restful 风格的网络请求框架，对网络请求高度封装，使得网络请求变得简单。

本文学习目标：
1. Retrofit 的基本使用

2. Retrofit 的如何做到使用注解，封装请求的信息的

3. Retrofit 与 OkHttp 的对接

4. Retrofit 的转换器
<!--more-->

## 1.Retrofit 的基本使用
### 1.1 GET 请求
GET 请求直接给请求接口的方法加上 @GET，参数使用 @Query 标记
```java
public interface NetService {
    @GET("/user")
    Call<String> getUser(@Query("username") String username);
}

final static Retrofit retrofit;
static {
    retrofit = new Retrofit.Builder()
            .baseUrl("http://localhost:8080")
            .addConverterFactory(ScalarsConverterFactory.create())
            .build();
}

private static void simpleGet() throws IOException {
    final NetService netService = retrofit.create(NetService.class);

    final Call<String> user = netService.getUser("alamide....");
    final String body = user.execute().body();
    System.out.println(body);
}
``` 

### 1.2 POST 请求，Form
POST 请求，给方法上加伤 @POST，如果是纯表单提交，加上 @FormUrlEncoded，参数使用 @Field 标记
```java
public interface NetService {
    @POST("/user")
    @FormUrlEncoded
    Call<String> postUser(@Field("username") String username, @Field("age") Integer age);
}

private static void postForm() throws IOException {
    final NetService netService = retrofit.create(NetService.class);
    final Call<String> wenbanyama = netService.postUser("wenbanyama", 19);
    final String body = wenbanyama.execute().body();
    System.out.println(body);
}
```

### 1.3 POST，Multipart
Multipart 提交数据加上 @Multipart，每个参数都要加上 @Part 注解，文件类型使用构造器生成
```java
public interface NetService {
    @Multipart
    @POST("/user/avatar")
    Call<String> postUser(@Part("username") String username, @Part("age") Integer age, @Part MultipartBody.Part avatar);
}

private static void postMultipart() throws IOException {
    final NetService netService = retrofit.create(NetService.class);
    final RequestBody requestBody = RequestBody.create(
            MediaType.parse("image/jpeg"),
            new File(file)
    );

    final MultipartBody.Part avatar = MultipartBody.Part.createFormData("avatar", "avatar.jpeg", requestBody);
    final Call<String> wenbanyama = netService.postUser(
            "wenbanyama",
            19,
            avatar
    );
    final String body = wenbanyama.execute().body();
    System.out.println(body);
}
```

### 1.4 Header 
要给请求加上请求头信息时，使用 @Header。如获取 User 信息时，希望服务器返回 Xml 格式的数据
```java
public interface NetService {
    @Headers("Accept: application/xml")
    @GET("/user")
    Call<String> getUser(@Query("username") String username);
}
```

### 1.5 Converter
Retrofit 提供转换器接口，可以使用转换器来转换请求体中的数据，以及响应数据。例如可以使用 Jackson 转换器，将响应 Json 格式数据，转换为 JavaBean
```java
retrofit = new Retrofit.Builder()
            .baseUrl("http://localhost:8080")
            .addConverterFactory(ScalarsConverterFactory.create())
            .addConverterFactory(JacksonConverterFactory.create())
            .build();
```

例如这个例子：

请求体要求使用 Json 格式的数据，可以写成这样
```java
@POST("/user/json")
Call<User> postUserWithJson(@Body User user);
```

这样提交的 User 在传输前会被转换成 Json，不用自己去生成，返回的 Json 字符串也会被转换为 User
## 2.Retrofit 如何解析注解
Retrofit 使用 Proxy 来代理请求接口，利用反射读取注解信息，之后进一步封装，之后将封装后的信息传递给 OkHttp，

从最简单的 POST 请求看起吧，
```java
public interface NetService {
    @POST("/user")
    @FormUrlEncoded
    Call<String> postUser(@Field("username") String username, @Field("age") Integer age);
}

private static void postForm() throws IOException {
    final NetService netService = retrofit.create(NetService.class);
    final Call<String> wenbanyama = netService.postUser("wenbanyama", 19);
    final String body = wenbanyama.execute().body();
    System.out.println(body);
}
```

先通过 Retrofit.create() 来创建代理对象
```java
public final class Retrofit {
    public <T> T create(final Class<T> service) {
        validateServiceInterface(service);
        return (T)
            Proxy.newProxyInstance(
                service.getClassLoader(),
                new Class<?>[] {service},
                new InvocationHandler() {
                    private final Platform platform = Platform.get();
                    private final Object[] emptyArgs = new Object[0];

                    @Override
                    public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
                        throws Throwable {
                        if (method.getDeclaringClass() == Object.class) {
                            return method.invoke(this, args);
                        }
                        args = args != null ? args : emptyArgs;
                        return platform.isDefaultMethod(method)
                            ? platform.invokeDefaultMethod(method, service, proxy, args)
                            : loadServiceMethod(method).invoke(args);
                    }
                }
            );
    }
}
```

Proxy.newProxyInstance() 这就是 Retrofit 的核心了，

接下来就是调用具体的方法了，调用 netService.postUser()，执行 invoke
```java
public @Nullable Object invoke(Object proxy, Method method, @Nullable Object[] args)
    throws Throwable {
    //Object 的方法不做任何处理
    if (method.getDeclaringClass() == Object.class) {
        return method.invoke(this, args);
    }
    args = args != null ? args : emptyArgs;
    //platform.isDefaultMethod(method)，是否为接口的默认实现，Java8 以后接口类方法可以有默认实现
    return platform.isDefaultMethod(method)
        ? platform.invokeDefaultMethod(method, service, proxy, args)
        : loadServiceMethod(method).invoke(args);
}
```

需要关注的是 loadServiceMethod()，这个方法会读取封装具体的 method 方法
```java
ServiceMethod<?> loadServiceMethod(Method method) {
    //缓存方法，避免多次解析，浪费资源
    ServiceMethod<?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    synchronized (serviceMethodCache) {
        result = serviceMethodCache.get(method);
        if (result == null) {
            //开始读取注解信息
            result = ServiceMethod.parseAnnotations(this, method);
            serviceMethodCache.put(method, result);
        }
    }
    return result;
}
```

ServiceMethod.parseAnnotations() 会将注解的信息封装，
```java
abstract class ServiceMethod<T> {
    static <T> ServiceMethod<T> parseAnnotations(Retrofit retrofit, Method method) {
    //读取注解信息，封装
    RequestFactory requestFactory = RequestFactory.parseAnnotations(retrofit, method);
    ......
    return HttpServiceMethod.parseAnnotations(retrofit, method, requestFactory);
  }
}
```

### 2.1 Request
先来看一下如何解析封装 Reqeust

使用 RequestFactory 来解析封装请求信息，看一下 RequestFactory 封装了哪些信息
```java
final class RequestFactory {
    private final Method method;

    //retrofit 设置的 baseUrl
    private final HttpUrl baseUrl;
    //本次请求的请求方法 GET/POST/PUT/DELETE 等
    final String httpMethod;
    //相对 url，最终请求的路径为 baseUrl + relativeUrl
    private final @Nullable String relativeUrl;
    //请求头
    private final @Nullable Headers headers;
    //Content-Type
    private final @Nullable MediaType contentType;
    //是否含有请求体，如 POST 请求带有请求体
    private final boolean hasBody;
    //是否为 Form 表单提交
    private final boolean isFormEncoded;
    //是否为 multipart/form-data
    private final boolean isMultipart;
    //对请求方法参数的封装，这是是先封装存储，后面执行方法时再传入具体的值
    private final ParameterHandler<?>[] parameterHandlers;
    final boolean isKotlinSuspendFunction;
}
```

封装的信息主要就是请求行和 ParameterHandler，ParameterHandler 用来后续生成请求体，来看看这些信息如何获取的

<hr/>

```java
private final HttpUrl baseUrl;
```

这个赋值很简单，就是直接从 Retrofit 获取
```java
RequestFactory(Builder builder) {
    baseUrl = builder.retrofit.baseUrl;
}
```
<hr/>
下面这几个属性，都是由方法上添加的注解解析出的，解析使用 parseMethodAnnotation()

```java
private final HttpUrl baseUrl;
final String httpMethod;
private final @Nullable String relativeUrl;
private final @Nullable Headers headers;
private final @Nullable MediaType contentType;
private final boolean hasBody;
private final boolean isFormEncoded;
private final boolean isMultipart;
```

具体的解析过程如下
```java
final class RequestFactory {

    static RequestFactory parseAnnotations(Retrofit retrofit, Method method) {
        return new Builder(retrofit, method).build();
    }

    RequestFactory(Builder builder) {
        httpMethod = builder.httpMethod;
        relativeUrl = builder.relativeUrl;
        headers = builder.headers;
        contentType = builder.contentType;
        hasBody = builder.hasBody;
        isFormEncoded = builder.isFormEncoded;
        isMultipart = builder.isMultipart;
    }
    static final class Builder {
        final Annotation[] methodAnnotations;

        Builder(Retrofit retrofit, Method method) {
            this.methodAnnotations = method.getAnnotations();
        }

        RequestFactory build() {
            //测试的方法中只 @POST ，@FormUrlEncoded
            for (Annotation annotation : methodAnnotations) {
                parseMethodAnnotation(annotation);
            }
        }

        private void parseMethodAnnotation(Annotation annotation) {
            if (annotation instanceof POST) {
                parseHttpMethodAndPath("POST", ((POST) annotation).value(), true);
            }else if (annotation instanceof FormUrlEncoded) {
                if (isMultipart) {
                    throw methodError(method, "Only one encoding annotation is allowed.");
                }
                //为 form 表单提交
                isFormEncoded = true;
            }
        }

        private void parseHttpMethodAndPath(String httpMethod, String value, boolean hasBody) {
            //HttpMethod 为 POST
            this.httpMethod = httpMethod;
            //POST 请求默认有请求体
            this.hasBody = hasBody;
            //由@POST("/user") 获取，relativeUrl 为 /user
            this.relativeUrl = value;
        }
    }
}
```

parseMethodAnnotation() 主要负责解析，请求头相关的信息
<hr/>
参数的解析，是整个解析的重点，接口方法传递的参数，也是最终封装到请求体中的信息，封装的信息放在 parameterHandlers 中

```java
private final ParameterHandler<?>[] parameterHandlers;
```

我们测试的网络请求 `postUser(@Field("username") String username, @Field("age") Integer age)` 会用两个 ParameterHandler 来存储，
```java
static final class Builder {
    ParameterHandler<?>[] parameterHandlers;

    Builder(Retrofit retrofit, Method method) {
        this.parameterTypes = method.getGenericParameterTypes();
        this.parameterAnnotationsArray = method.getParameterAnnotations();
    }

    RequestFactory build() {
        int parameterCount = parameterAnnotationsArray.length;
        parameterHandlers = new ParameterHandler<?>[parameterCount];
        for (int p = 0, lastParameter = parameterCount - 1; p < parameterCount; p++) {
            parameterHandlers[p] =
                parseParameter(p, parameterTypes[p], parameterAnnotationsArray[p], p == lastParameter);
        }
    }

    private @Nullable ParameterHandler<?> parseParameter(
        int p, Type parameterType, @Nullable Annotation[] annotations, boolean allowContinuation) {
        if (annotations != null) {
            for (Annotation annotation : annotations) {
                ParameterHandler<?> annotationAction =
                    parseParameterAnnotation(p, parameterType, annotations, annotation);
                ......
                result = annotationAction;
            }
        }  
    }

    private ParameterHandler<?> parseParameterAnnotation(
        int p, Type type, Annotation[] annotations, Annotation annotation) {
        if (annotation instanceof Field) {
            validateResolvableType(p, type);
            if (!isFormEncoded) {
                throw parameterError(method, p, "@Field parameters can only be used with form encoding.");
            }
            Field field = (Field) annotation;
            String name = field.value();
            boolean encoded = field.encoded();

            gotField = true;
           
            Converter<?, String> converter = retrofit.stringConverter(type, annotations);
            return new ParameterHandler.Field<>(name, converter, encoded);
        } 
    }
}

abstract class ParameterHandler<T> {
    static final class Field<T> extends ParameterHandler<T> {
        private final String name;
        private final Converter<T, String> valueConverter;
        private final boolean encoded;

        Field(String name, Converter<T, String> valueConverter, boolean encoded) {
            this.name = Objects.requireNonNull(name, "name == null");
            this.valueConverter = valueConverter;
            this.encoded = encoded;
        }

        @Override
        void apply(RequestBuilder builder, @Nullable T value) throws IOException {
            if (value == null) return; // Skip null values.

            String fieldValue = valueConverter.convert(value);
            if (fieldValue == null) return; // Skip null converted values

            builder.addFormField(name, fieldValue, encoded);
        }
    }
}
```

最终将参数信息封装到 ParameterHandler.Field 中，封装的信息包括参数名、转换器

到这里为止，postUser() 的所以注解全部解析完毕了，请求头的信息已经封装好，ParameterHandler 还待处理。

再来看看如何使用 ParameterHandler 来构建请求体的，在调用方法时，最终会调用 RequestFactory.create() 来构建请求体
```java
final class RequestFactory {
    okhttp3.Request create(Object[] args) throws IOException {
        ParameterHandler<Object>[] handlers = (ParameterHandler<Object>[]) parameterHandlers;

        RequestBuilder requestBuilder =
        new RequestBuilder(
            httpMethod,
            baseUrl,
            relativeUrl,
            headers,
            contentType,
            hasBody,
            isFormEncoded,
            isMultipart);
        
        List<Object> argumentList = new ArrayList<>(argumentCount);
        for (int p = 0; p < argumentCount; p++) {
            //存放 args
            argumentList.add(args[p]);
            //请求参数写入请求体
            handlers[p].apply(requestBuilder, args[p]);
        }

        return requestBuilder.get().tag(Invocation.class, new Invocation(method, argumentList)).build();
    }
}
```

其中 `handlers[p].apply(requestBuilder, args[p]);` 将参数写入到请求体中，本次测试使用的 POST Form 表单提交，
```java
static final class Field<T> extends ParameterHandler<T> {
    @Override
    void apply(RequestBuilder builder, @Nullable T value) throws IOException {
        if (value == null) return; // Skip null values.

        String fieldValue = valueConverter.convert(value);
        if (fieldValue == null) return; // Skip null converted values
        //写入 form 表单参数
        builder.addFormField(name, fieldValue, encoded);
    }
}

final class RequestBuilder {

    RequestBuilder(
        String method,
        HttpUrl baseUrl,
        @Nullable String relativeUrl,
        @Nullable Headers headers,
        @Nullable MediaType contentType,
        boolean hasBody,
        boolean isFormEncoded,
        boolean isMultipart) {
        this.method = method;
        this.baseUrl = baseUrl;
        this.relativeUrl = relativeUrl;
        this.requestBuilder = new Request.Builder();
        this.contentType = contentType;
        this.hasBody = hasBody;

        if (headers != null) {
            headersBuilder = headers.newBuilder();
        } else {
            headersBuilder = new Headers.Builder();
        }

        if (isFormEncoded) {
            // form 表单提交使用
            formBuilder = new FormBody.Builder();
        } else if (isMultipart) {
            // Will be set to 'body' in 'build'.
            multipartBuilder = new MultipartBody.Builder();
            multipartBuilder.setType(MultipartBody.FORM);
        }
    }

    void addFormField(String name, String value, boolean encoded) {
        if (encoded) {
            formBuilder.addEncoded(name, value);
        } else {
            formBuilder.add(name, value);
        }
    }

    Request.Builder get() {
        RequestBody body = this.body;
        if (body == null) {
            // Try to pull from one of the builders.
            if (formBuilder != null) {
                body = formBuilder.build();
            } 
        }
        //构建请求体
        return requestBuilder.url(url).headers(headersBuilder.build()).method(method, body);
    }
}
```

FormBody 是 OKHttp 的类，这里就与 OkHttp 对接了

小结一下：
1. 先读取方法上所添加的注解，这里的注解是静态的，一般没有什么需要特殊处理，所以逻辑也简单，就是读取、存储，读取的信息存储在 RequestFactory 中

2. 再读取方法参数上的注解，注解读取，使用 ParameterHandler[] 来存储，

3. 调用 RequestFactory.create() 来正式将之前读取的信息，来和 OkHttp 对接，依据 formBuilder 和 multipartBuilder 来赋值响应的请求体 FormBody、MultipartBody

4. 使用 `okhttp3.Request.Builder` 构造 Request

### 2.2 Response
响应的代码简单一点，因为不需要解析注解，直接返回请求的结果就可以了
```java
abstract class HttpServiceMethod<ResponseT, ReturnT> extends ServiceMethod<ReturnT> {
    final @Nullable ReturnT invoke(Object[] args) {
        Call<ResponseT> call = new OkHttpCall<>(requestFactory, args, callFactory, responseConverter);
        return adapt(call, args);
    }
}

final class OkHttpCall<T> implements Call<T> {
    public Response<T> execute() throws IOException {
        okhttp3.Call call;

        synchronized (this) {
        if (executed) throw new IllegalStateException("Already executed.");
            executed = true;

            call = getRawCall();
        }

        if (canceled) {
            call.cancel();
        }
        //call.execute() 调用 OkHttp 层
        return parseResponse(call.execute());
    }
}
```

### 2.3 Converter
Converter 用来转换请求和响应的信息，如：postUserWithJson(@Body User user)，可以将 User 序列化为 Json 字符串传递给服务器。

Call<User> getUser() 可以将服务器返回的 Json 字符串反序列化为 User。

使用转换器时，要预先注册，注册 Jackson 转换器
```java
retrofit = new Retrofit.Builder()
        .baseUrl("http://localhost:8080")
        .addConverterFactory(ScalarsConverterFactory.create())
        .addConverterFactory(JacksonConverterFactory.create())
        .build();
```

#### 2.3.1 Request
请求时，会在使用 @Body 注解时进行转化，以 postUserWithJson(@Body User user) 为例
```java
if (annotation instanceof Body) {
    Converter<?, RequestBody> converter;
    converter = retrofit.requestBodyConverter(type, annotations, methodAnnotations);
    return new ParameterHandler.Body<>(method, p, converter);
}

public <T> Converter<T, RequestBody> requestBodyConverter(
    Type type, Annotation[] parameterAnnotations, Annotation[] methodAnnotations) {
    return nextRequestBodyConverter(null, type, parameterAnnotations, methodAnnotations);
}

public <T> Converter<T, RequestBody> nextRequestBodyConverter(...){
    int start = converterFactories.indexOf(skipPast) + 1;
    for (int i = start, count = converterFactories.size(); i < count; i++) {
        Converter.Factory factory = converterFactories.get(i);
        Converter<?, RequestBody> converter =
            factory.requestBodyConverter(type, parameterAnnotations, methodAnnotations, this);
        //获取合适的 Converter
        if (converter != null) {
            return (Converter<T, RequestBody>) converter;
        }
    }
}
```

converterFactories 中存储已注册的 Converter，本次测试中其含有四个 
```
BuiltInConverter
ScalarsConverterFactory
JacksonConverterFactory
OptionalConverterFactory
```

在构建 Retrofit 时初始化
```java
public Retrofit build() {
    List<Converter.Factory> converterFactories =
          new ArrayList<>(
              1 + this.converterFactories.size() + platform.defaultConverterFactoriesSize());
    //这是默认的处理器，当返回的类型为 RequestBody 时，使用这个转换器
    converterFactories.add(new BuiltInConverters());
    //自行注册的 Converter
    converterFactories.addAll(this.converterFactories);
    //Java8 之后 Call<Optional>
    converterFactories.addAll(platform.defaultConverterFactories());
}
```

自行注册的 Converter 为 ScalarsConverterFactory、JacksonConverterFactory，在使用转换器时遍历 converterFactories
1. 遍历到 BuiltInConverter，由于类型不为 RequestBody，所以返回 null，继续遍历

2. 遍历到 ScalarsConverterFactory，类型不为基础数据类型，返回 null，继续遍历

3. 遍历到 JacksonConverterFactory，符合，返回 JacksonRequestBodyConverter

```java
final class JacksonRequestBodyConverter<T> implements Converter<T, RequestBody> {
    private static final MediaType MEDIA_TYPE = MediaType.get("application/json; charset=UTF-8");

    private final ObjectWriter adapter;

    JacksonRequestBodyConverter(ObjectWriter adapter) {
        this.adapter = adapter;
    }

    @Override
    public RequestBody convert(T value) throws IOException {
        //将 User 转化为 Json
        byte[] bytes = adapter.writeValueAsBytes(value);
        //创建请求体
        return RequestBody.create(MEDIA_TYPE, bytes);
    }
}
```

#### 2.3.2 Response
服务器返回的结果，可以通过 Converter 转换，以 `Call<User> getUser()` 为例
```java
final class OkHttpCall<T> implements Call<T> {
    public Response<T> execute() throws IOException {
        return parseResponse(call.execute());
    }

    Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
        ......
        T body = responseConverter.convert(catchingBody);
        return Response.success(body, rawResponse);
    }
}

final class JacksonResponseBodyConverter<T> implements Converter<ResponseBody, T> {
    public T convert(ResponseBody value) throws IOException {
        try {
            return adapter.readValue(value.charStream());
        } finally {
            value.close();
        }
    }
}
```

将读取的 Response 使用 Jackson 转换为 User
### 2.4 CallAdapter
CallAdapter 负责对 OkHttp 返回的数据做进一步处理，可以用 RxJava、LiveData 等进行处理，这里看一下默认的 CallAdapter，学习一下是如何转换的
```java
abstract class HttpServiceMethod<ResponseT, ReturnT> extends ServiceMethod<ReturnT> {
    static <ResponseT, ReturnT> HttpServiceMethod<ResponseT, ReturnT> parseAnnotations(
        Retrofit retrofit, Method method, RequestFactory requestFactory) {
        //DefaultCallAdapterFactory 中创建的匿名内部类
        CallAdapter<ResponseT, ReturnT> callAdapter =
            createCallAdapter(retrofit, method, adapterType, annotations);

        okhttp3.Call.Factory callFactory = retrofit.callFactory;
        return new CallAdapted<>(requestFactory, callFactory, responseConverter, callAdapter);
    }

    private static <ResponseT, ReturnT> CallAdapter<ResponseT, ReturnT> createCallAdapter(...){
        return (CallAdapter<ResponseT, ReturnT>) retrofit.callAdapter(returnType, annotations);
    }

    static final class CallAdapted<ResponseT, ReturnT> extends HttpServiceMethod<ResponseT, ReturnT> {
        private final CallAdapter<ResponseT, ReturnT> callAdapter;

        CallAdapted(
            RequestFactory requestFactory,
            okhttp3.Call.Factory callFactory,
            Converter<ResponseBody, ResponseT> responseConverter,
            CallAdapter<ResponseT, ReturnT> callAdapter) {
            super(requestFactory, callFactory, responseConverter);
            this.callAdapter = callAdapter;
        }

        @Override
        protected ReturnT adapt(Call<ResponseT> call, Object[] args) {
            return callAdapter.adapt(call);
        }
    }
}
```

CallAdapter 在构建 Retrofit 默认注册，
```java
public Retrofit build() {
    Executor callbackExecutor = this.callbackExecutor;
    if (callbackExecutor == null) {
        //为 null
        callbackExecutor = platform.defaultCallbackExecutor();
    }

    // Make a defensive copy of the adapters and add the default Call adapter.
    List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
    callAdapterFactories.addAll(platform.defaultCallAdapterFactories(callbackExecutor));
}

class Platform {
    List<? extends CallAdapter.Factory> defaultCallAdapterFactories(
        @Nullable Executor callbackExecutor) {
        DefaultCallAdapterFactory executorFactory = new DefaultCallAdapterFactory(callbackExecutor);
        //CompletableFutureCallAdapterFactory 为 Call<CompletableFuture> 时使用
        return hasJava8Types
            ? asList(CompletableFutureCallAdapterFactory.INSTANCE, executorFactory)
            : singletonList(executorFactory);
    }
}

final class DefaultCallAdapterFactory extends CallAdapter.Factory {
    public @Nullable CallAdapter<?, ?> get(
        Type returnType, Annotation[] annotations, Retrofit retrofit) {

            //最终返回的 Adapter
            return new CallAdapter<Object, Call<?>>() {
                @Override
                public Type responseType() {
                    return responseType;
                }

                @Override
                public Call<Object> adapt(Call<Object> call) {
                    //executor = null，直接返回 call
                    return executor == null ? call : new ExecutorCallbackCall<>(executor, call);
                }
            };
        }
}
```

本次测试中 parseAnnotations 直接返回 CallAdapted，最终调用 loadServiceMethod(method).invoke(args) 后，直接调用 CallAdapter 的 adapt，返回 OkHttpCall，再调用 execute
```java
final class OkHttpCall<T> implements Call<T> {
    public Response<T> execute() throws IOException {
        return parseResponse(call.execute());
    }
}
```

下面就使用 Converter 转换 Response 携带的信息了

## 3.小结
现在放开 Retrofit ，让我们在 OkHttp 的基础上设计一个使用注解请求网络的框架，该如何设计，
1. 设计注解，包括请求方法 @GET、@POST 等，

2. 设计注解，表明请求的 Content-Type ，如 @FormUrlEncoded、@Multipart 等，

3. 设计参数上使用的注解，包括 @Query、@Field、@Body 等，

4. 设计注解处理工厂，读取被注解标记的元素的信息，将读取的结果保存，在后续调用 execute 时，将读取的信息写入 OkHttp

5. 设计 Call 的处理器，将响应的信息进一步处理

Retrofit 的处理流程大致是这样的，其中使用 ParameterHandler 是一个非常值得借鉴的点，Http 请求包含请求行、请求头、请求体，请求行、请求头的相关信息，由方法上添加注解表明

方法中的参数，一般是写入请求体中的内容，请求体有 Form、MultiPart 等类型，使用 ParameterHandler[] 来存储方法参数的信息，每一个 ParameterHandler 都应该知道自己以何种形式写入 OkHttp，

最后就是 CallAdapter ，可以将返回的数据再做进一步处理，




