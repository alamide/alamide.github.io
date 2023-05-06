---
layout: post
title: Jackson
categories: java json
excerpt: Jackson 的相关笔记
tags: java json
date: 2023-05-03
---

文档[在这里](https://www.baeldung.com/jackson)

## 1.引入 Jackson
```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-core</artifactId>
    <version>2.13.3</version>
</dependency>
```

## 2.Jackson Serialization Annotations
### 2.1 @JsonAnyGetter
可以将 key/value 型的拆解

```java
public class ExtendableBean {
    public String name;
    private Map<String, String> properties = new HashMap<>();

    public ExtendableBean(String name) {
        this.name = name;
    }

    public void add(String key, String value){
        properties.put(key, value);
    }

    @JsonAnyGetter
    public Map<String, String> getProperties() {
        return properties;
    }
}

@Test
public void testJsonAnyGetter() throws IOException {

    ExtendableBean bean = new ExtendableBean("My bean");
    bean.add("attr1", "val1");
    bean.add("attr2", "val2");

    String result = new ObjectMapper().writeValueAsString(bean);
    System.out.println(result);
}
```

使用 @JsonAnyGetter 输出：

```json
{"name":"My bean","attr2":"val2","attr1":"val1"}
```

不使用 @JsonAnyGetter 或 @JsonAnyGetter(enabled = false) 输出：
```json
{"name":"My bean","properties":{"attr2":"val2","attr1":"val1"}}
```

### 2.2 @JsonGetter
```java
@AllArgsConstructor
public class MyBean {
    public int id;

    private String name;

    @JsonGetter("name")
    public String getTheName(){
        return name;
    }
}

@Test
public void testJsonGet() throws JsonProcessingException {
    MyBean myBean = new MyBean(1, "alamide");
    final String string = new ObjectMapper().writeValueAsString(myBean);
    //1.使用 JsonGetter 输出 {"id":1,"name":"alamide"}
    //2.不使用 JsonGetter 输出 {"id":1,"theName":"alamide"}
    System.out.println(string);
}
```

### 2.3 @JsonPropertyOrder
设置序列化时 key 的顺序
```java
@AllArgsConstructor
@JsonPropertyOrder({"name", "id"})
public class MyBean {
    public int id;

    public String name;
}

@Test
public void testJsonPropertyOrder() throws JsonProcessingException {
    MyBean myBean = new MyBean(1, "alamide");
    final String string = new ObjectMapper().writeValueAsString(myBean);
    System.out.println(string);
}
```

输出：
```json
{"name":"alamide","id":1}
```

### 2.4 @JsonRawValue
Json 格式的数据以什么形式输出
```java
@AllArgsConstructor
public class RawBean {
    public String name;

    @JsonRawValue
    public String json;
}

@Test
public void testJsonRawValue() throws JsonProcessingException {
    RawBean rawBean = new RawBean("My Bean", "{\"attr\":false}");
    final String string = new ObjectMapper().writeValueAsString(rawBean);
    //1.使用 @JsonRawValue {"name":"My Bean","json":{"attr":false}}
    //2.不使用 @JsonRawValue {"name":"My Bean","json":"{\"attr\":false}"}
    System.out.println(string);
}
```

### 2.5 @JsonValue
@JsonValue indicates a single method the library will use to serialize the entire instance.

```java
@AllArgsConstructor
public enum TypeEnumWithValue {
    TYPE1(1, "Type A"), TYPE2(2, "Type 2");

    private Integer id;
    private String name;

    @JsonValue
    public String getName() {
        return name;
    }
}

@Test
public void whenSerializingUsingJsonValue_thenCorrect()
        throws JsonParseException, IOException {

    String enumAsString = new ObjectMapper()
            .writeValueAsString(TypeEnumWithValue.TYPE1);
    //1.使用 @JsonValue 输出："Type A"
    //2.不使用则输出 "TYPE1"
    //3.一个 POJO 只有一个方法可使用 @JsonValue，
    System.out.println(enumAsString);
}
```

### 2.6 @JsonRootName
```java
@AllArgsConstructor
@JsonRootName(value = "user")
public class UserWithRoot {
    public int id;
    public String name;
}

@Test
public void testJsonRootName() throws JsonProcessingException {
    UserWithRoot userWithRoot = new UserWithRoot(1, "alamide");
    final ObjectMapper objectMapper = new ObjectMapper();
    objectMapper.enable(SerializationFeature.WRAP_ROOT_VALUE);
    final String string =objectMapper.writeValueAsString(userWithRoot);
    //输出：{"user":{"id":1,"name":"alamide"}}
    System.out.println(string);
}
```

### 2.7 @JsonSerialize
定义序列化的格式
```java
@AllArgsConstructor
public class EventWithSerializer {

    public String name;
    @JsonSerialize(using = CustomDateSerializer.class)
    public Date eventDate;
}

public class CustomDateSerializer extends StdSerializer<Date> {

    private final SimpleDateFormat simpleDateFormat = new SimpleDateFormat("yyyy-MM-dd HH:mm");
    protected CustomDateSerializer(Class<Date> t) {
        super(t);
    }

    public CustomDateSerializer(){
        this(null);
    }

    @Override
    public void serialize(Date value, JsonGenerator gen, SerializerProvider provider) throws IOException {
        gen.writeString(simpleDateFormat.format(value));
    }
}

@Test
public void testJsonSerialize() throws JsonProcessingException {
    EventWithSerializer eventWithSerializer = new EventWithSerializer("alamide", new Date());
    final String json = new ObjectMapper().writeValueAsString(eventWithSerializer);
    //{"name":"alamide","eventDate":"2023-05-03 22:35"}
    System.out.println(json);
}
```

## 3.Jackson Deserialization Annotations
### 3.1  @JsonCreator
当key 不匹配时使用，使用时，构造器参数都需要加上 @JsonProperty
```java
@ToString
public class BeanWithCreator {
    public int id;
    public String name;
    @JsonCreator
    public BeanWithCreator(@JsonProperty("id") int id, @JsonProperty("theName") String name) {
        this.id = id;
        this.name = name;
    }
}

@Test
public void testJsonCreator() throws JsonProcessingException {
    String json = "{\"id\":1,\"theName\":\"My bean\"}";
    final BeanWithCreator bean = new ObjectMapper().readerFor(BeanWithCreator.class).readValue(json);
    //BeanWithCreator(id=1, name=My bean)
    System.out.println(bean);
}
```

### 3.2 @JacksonInject
属性从 Inject 获取，而不是从 Json 字符串
```java
@ToString
public class BeanWithInject {
    @JacksonInject
    public int id;

    public String name;
}

@Test
public void testJsonInject() throws JsonProcessingException {
    String json = "{\"name\":\"My bean\"}";
    InjectableValues inject = new InjectableValues.Std().addValue(int.class, 100);
    final BeanWithInject bean = new ObjectMapper()
            .reader(inject)
            .forType(BeanWithInject.class)
            .readValue(json);
    //BeanWithInject(id=100, name=My bean)
    System.out.println(bean);
}
```

### 3.3 @JsonAnySetter
可以将多余的属性，放置入 key/value 中
```java
@ToString
@NoArgsConstructor
public class ExtendableBean {
    public String name;
    private Map<String, String> properties = new HashMap<>();

    public ExtendableBean(String name) {
        this.name = name;
    }


    @JsonAnySetter
    public void add(String key, String value){
        properties.put(key, value);
    }
}

@Test
public void testJsonAnySetter() throws JsonProcessingException {
    String json
            = "{\"name\":\"My bean\",\"attr2\":\"val2\",\"attr1\":\"val1\"}";
    final ExtendableBean extendableBean = new ObjectMapper().readerFor(ExtendableBean.class).readValue(json);
    //ExtendableBean(name=My bean, properties={attr2=val2, attr1=val1})
    System.out.println(extendableBean);
}
```

### 3.4 @JsonSetter
指定 key
```java
@AllArgsConstructor
@NoArgsConstructor
@JsonPropertyOrder({"name", "id"})
@ToString
public class MyBean {
    public int id;

    private String name;

    @JsonSetter("name")
    public void setTheName(String name) {
        this.name = name;
    }
}

@Test
public void testJsonSetter() throws JsonProcessingException {
    String json = "{\"id\":1,\"name\":\"My bean\"}";

    MyBean bean = new ObjectMapper()
            .readerFor(MyBean.class)
            .readValue(json);
    //MyBean(id=1, name=My bean)
    System.out.println(bean);
}
```

### 3.5 @JsonDeserialize
定制反序列化器
```java
@AllArgsConstructor
@ToString
@NoArgsConstructor
public class EventWithSerializer {

    public String name;
    @JsonSerialize(using = CustomDateSerializer.class)
    @JsonDeserialize(using = CustomDateDeserializer.class)
    public Date eventDate;
}

@Test
public void testDeserializer() throws JsonProcessingException {
    String json
            = "{\"name\":\"party\",\"eventDate\":\"20-12-2014 02:30:00\"}";
    final EventWithSerializer event = new ObjectMapper().readerFor(EventWithSerializer.class).readValue(json);
    //EventWithSerializer(name=party, eventDate=Thu Jun 06 02:30:00 CST 26)
    System.out.println(event);
}
```

### 3.6 @JsonAlias
指定多个可选名
```java
@Data
@ToString
public class AliasBean {
    @JsonAlias({"1Name", "first_name"})
    private String firstName;

    private String lastName;
}

@Test
public void testJsonAlias() throws JsonProcessingException {
    String json = "{\"1Name\": \"John\", \"lastName\": \"Green\"}";
    AliasBean aliasBean = new ObjectMapper().readerFor(AliasBean.class).readValue(json);
    System.out.println(aliasBean);
}
```
## 4.Jackson Property Inclusion Annotations
### 4.1 @JsonIgnoreProperties
忽略属性
```java
@Data
@AllArgsConstructor
@JsonIgnoreProperties({"name"})
@NoArgsConstructor
public class BeanWithIgnore {
    public int id;
    private String name;
}

@Test
public void testJsonIgnore() throws JsonProcessingException {
    BeanWithIgnore beanWithIgnore=new BeanWithIgnore(1, "alamide");
    final String string = new ObjectMapper().writeValueAsString(beanWithIgnore);
    //{"id":1}
    System.out.println(string);
}
```

### 4.2 @JsonIgnore
```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class BeanWithIgnore {
    public int id;
    @JsonIgnore
    private String name;
}

@Test
public void testJsonIgnore() throws JsonProcessingException {
    BeanWithIgnore beanWithIgnore=new BeanWithIgnore(1, "alamide");
    final String string = new ObjectMapper().writeValueAsString(beanWithIgnore);
    //{"id":1}
    System.out.println(string);
}
```

### 4.3 @JsonIgnoreType
```java
@AllArgsConstructor
public class User {
    public int id;

    public Name name;

    @JsonIgnoreType
    public static class Name {
        public String firstName;

        public String lastName;
    }
}

@Test
public void testJsonIgnoreType() throws JsonProcessingException {
    User.Name name = new User.Name();
    name.firstName = "ala";
    name.lastName = "mide";
    User user = new User(1, name);
    final String string = new ObjectMapper().writeValueAsString(user);
    //{"id":1}
    System.out.println(string);
}
```

### 4.4 @JsonInclude
```java
@AllArgsConstructor
@NoArgsConstructor
@ToString
@JsonInclude(JsonInclude.Include.NON_NULL)
public class MyBean {
    public int id;

    public String name;
}

@Test
public void testJsonInclude() throws JsonProcessingException {
    MyBean myBean = new MyBean(1, null);
    final String string = new ObjectMapper().writeValueAsString(myBean);
    //{"id":1}，如果不加JsonInclude，则输出 {"id":1,"name":null}
    System.out.println(string);
}
```

### 4.5 @JsonIncludeProperties
```java
@JsonIncludeProperties({"name"})
@AllArgsConstructor
public class BeanWithInclude {
    public int id;

    public String name;
}

@Test
public void testJsonIncludeProperties() throws JsonProcessingException {
    BeanWithInclude bean = new BeanWithInclude(1, "alamide");
    final String string = new ObjectMapper().writeValueAsString(bean);
    System.out.println(string);
}
```

### 4.6 @JsonAutoDetect
指定哪些属性是可见的、哪些不可见
```java
@AllArgsConstructor
@JsonAutoDetect(fieldVisibility = JsonAutoDetect.Visibility.PUBLIC_ONLY)
public class PrivateBean {
    private int id;
    private String name;
}

@Test
public void testJsonAutoDetect() throws JsonProcessingException {
    PrivateBean privateBean = new PrivateBean(1, "alamide");
    final String string = new ObjectMapper().writeValueAsString(privateBean);
    //{}
    System.out.println(string);
}
```

## 5.Jackson Polymorphic Type Handling Annotations
```java
@AllArgsConstructor
@NoArgsConstructor
@ToString
public class Zoo {
    public Animal animal;
    @AllArgsConstructor
    @JsonTypeInfo(
            use = JsonTypeInfo.Id.NAME,
            include = JsonTypeInfo.As.PROPERTY,
            property = "type")
    @JsonSubTypes({
            @JsonSubTypes.Type(value=Dog.class, name = "dog"),
            @JsonSubTypes.Type(value=Cat.class, name = "cat"),
    })
    public static class Animal {
        public String name;
    }
    @JsonTypeName("dog")
    public static class Dog extends Animal {
        public double barkVolume;
        public Dog(){
            super(null);
        }
        public Dog(String name, double barkVolume) {
            super(name);
            this.barkVolume = barkVolume;
        }
    }

    @JsonTypeName("cat")
    public static class Cat extends Animal {
        boolean likesCream;
        public int lives;
        public Cat(){
            super(null);
        }
        public Cat(String name, boolean likesCream, int lives) {
            super(name);
            this.likesCream = likesCream;
            this.lives = lives;
        }
    }
}

@Test
public void testType() throws JsonProcessingException {
    Zoo zoo = new Zoo(new Zoo.Dog("dog", 120));
    final String string = new ObjectMapper().writeValueAsString(zoo);
    //{"animal":{"type":"dog","name":"dog","barkVolume":120.0}}
    System.out.println(string);

    String json = "{\"animal\":{\"name\":\"lacy\",\"type\":\"cat\"}}";
    final Zoo value = new ObjectMapper().readerFor(Zoo.class).readValue(json);
    //class com.alamide.third.entities.Zoo$Cat
    System.out.println(value.animal.getClass());
}
```

## 6.Jackson General Annotations
### 6.1 @JsonProperty
指定在 Json 中的 key，可以代替 JsonSetter、JsonGetter
```java
@AllArgsConstructor
@NoArgsConstructor
@ToString
public class MyBean {
    public int id;
    private String name;
    
    @JsonProperty("diyName")
    public String getTheName() {
        return name;
    }

    @JsonProperty("diyName")
    public void setTheName(String name) {
        this.name = name;
    }
}

@Test
public void testJsonProperty() throws JsonProcessingException {
    MyBean myBean = new MyBean(1, "alamide");
    final String string = new ObjectMapper().writeValueAsString(myBean);
    //{"id":1,"diyName":"alamide"}
    System.out.println(string);
}
```

### 6.2 @JsonFormat
可以替换 @JsonSerialize、@JsonDeserialize
```java
@AllArgsConstructor
@ToString
@NoArgsConstructor
public class EventWithSerializer {

    public String name;
//    @JsonSerialize(using = CustomDateSerializer.class)
//    @JsonDeserialize(using = CustomDateDeserializer.class)
    @JsonFormat(
            shape = JsonFormat.Shape.STRING,
            pattern = "yyyy-MM-dd HH:mm"
    )
    public Date eventDate;
}

@Test
public void testJsonSerialize() throws JsonProcessingException {
    EventWithSerializer eventWithSerializer = new EventWithSerializer("alamide", new Date());
    final String json = new ObjectMapper().writeValueAsString(eventWithSerializer);
    //{"name":"alamide","eventDate":"2023-05-06 08:29"}
    System.out.println(json);
}
```

### 6.3 @JsonUnwrapped
unwrapped/flattened when serialized/deserialized
```java
@AllArgsConstructor
public class User {
    public int id;

    @JsonUnwrapped
    public Name name;

    public static class Name {
        public String firstName;

        public String lastName;
    }
}

@Test
public void testJsonIgnoreType() throws JsonProcessingException {
    User.Name name = new User.Name();
    name.firstName = "ala";
    name.lastName = "mide";
    User user = new User(1, name);
    final String string = new ObjectMapper().writeValueAsString(user);
    //不加 @JsonUnwrapped {"id":1,"name":{"firstName":"ala","lastName":"mide"}}
    //加 @JsonUnwrapped {"id":1,"firstName":"ala","lastName":"mide"}
    System.out.println(string);
}
```

### 6.4 @JsonView
indicates the View in which the property will be included for serialization/deserialization

可以指定哪些试图显示属性
```java
 public class Views {
    public static class Public {}
    public static class Internal extends Public {}
}
@AllArgsConstructor
public class Item {
    @JsonView(Views.Public.class)
    public int id;

    @JsonView(Views.Public.class)
    public String itemName;

    @JsonView(Views.Internal.class)
    public String ownerName;
}

@Test
public void testJsonView() throws JsonProcessingException {
    Item item = new Item(2, "book", "John");

    String result = new ObjectMapper()
            .writerWithView(Views.Public.class)
            .writeValueAsString(item);
    //{"id":2,"itemName":"book"}
    System.out.println(result);
}
```

### 6.5 @JsonManagedReference, @JsonBackReference
避免循环引用
```java
@AllArgsConstructor
public class ItemWithRef {
    public int id;
    public String itemName;

    @JsonManagedReference
    public UserWithRef owner;
}

@AllArgsConstructor
public class UserWithRef {
    public int id;
    public String name;

    @JsonBackReference
    public List<ItemWithRef> userItems = new ArrayList<>();

    public UserWithRef(int id, String name) {
        this.id = id;
        this.name = name;
    }
}

@Test
public void testLoop() throws JsonProcessingException {
    UserWithRef user = new UserWithRef(1, "John");
    ItemWithRef item = new ItemWithRef(2, "book", user);
    user.userItems.add(item);

    String result = new ObjectMapper().writeValueAsString(item);
    //{"id":2,"itemName":"book","owner":{"id":1,"name":"John"}}
    System.out.println(result);
}
```

### 6.6 @JsonIdentityInfo
@JsonIdentityInfo indicates that Object Identity should be used when serializing/deserializing values, like when dealing with infinite recursion types of problems

同样是解决循环引用的问题
```java
@JsonIdentityInfo(
        generator = ObjectIdGenerators.PropertyGenerator.class,
        property = "id")
public class UserWithIdentity {
    public int id;
    public String name;
    public List<ItemWithIdentity> userItems = new ArrayList<>();

    public UserWithIdentity(int id, String name) {
        this.id = id;
        this.name = name;
    }
}

@JsonIdentityInfo(
        generator = ObjectIdGenerators.PropertyGenerator.class,
        property = "id")
@AllArgsConstructor
public class ItemWithIdentity {
    public int id;
    public String itemName;
    public UserWithIdentity owner;
}

@Test
public void testLoop() throws JsonProcessingException {
    UserWithIdentity user = new UserWithIdentity(1, "John");
    ItemWithIdentity item = new ItemWithIdentity(2, "book", user);
    user.userItems.add(item);
    String result = new ObjectMapper().writeValueAsString(item);
    //{"id":2,"itemName":"book","owner":{"id":1,"name":"John","userItems":[2]}}
    System.out.println(result);
}
```

### 6.7 @JsonFilter
```java
@JsonFilter("myFilter")
@AllArgsConstructor
public class BeanWithFilter {
    public int id;
    public String name;
}

@Test
public void testJsonFilter() throws JsonProcessingException {
    BeanWithFilter bean = new BeanWithFilter(1, "My bean");

    FilterProvider filters
            = new SimpleFilterProvider().addFilter(
            "myFilter",
            SimpleBeanPropertyFilter.filterOutAllExcept("name"));

    String result = new ObjectMapper()
            .writer(filters)
            .writeValueAsString(bean);
    //{"name":"My bean"}
    System.out.println(result);
}
```

## 7.Custom Jackson Annotation
组合使用 Jackson Annotation
```java
@Retention(RetentionPolicy.RUNTIME)
@JacksonAnnotationsInside
@JsonInclude(JsonInclude.Include.NON_NULL)
@JsonPropertyOrder({ "name", "id", "dateCreated" })
public @interface CustomAnnotation {}

@CustomAnnotation
@AllArgsConstructor
public class BeanWithCustomAnnotation {
    public int id;
    public String name;
    public Date dateCreated;
}


@Test
public void testCustomAnnotation() throws JsonProcessingException {
    BeanWithCustomAnnotation bean
            = new BeanWithCustomAnnotation(1, "My bean", null);

    String result = new ObjectMapper().writeValueAsString(bean);
    //{"name":"My bean","id":1}
    System.out.println(result);
}
```

## 8.Jackson MixIn Annotations
```java
@JsonIgnoreType
public class MyMixInForIgnoreType {}

@AllArgsConstructor
public class Item {
    public int id;

    public String itemName;

    public User user;
}

@Test
public void test() throws JsonProcessingException {
    Item item = new Item(1, "book", null);

    String result = new ObjectMapper().writeValueAsString(item);
    //{"id":1,"itemName":"book","user":null}
    System.out.println(result);

    ObjectMapper mapper = new ObjectMapper();
    mapper.addMixIn(User.class, MyMixInForIgnoreType.class);

    result = mapper.writeValueAsString(item);
    //{"id":1,"itemName":"book"}
    System.out.println(result);
}
```

## 9.Disable Jackson Annotation
使所有的 Jackson 失效
```java
@AllArgsConstructor
@NoArgsConstructor
@ToString
public class MyBean {
    public int id;
    private String name;
    @JsonProperty("diyName")
    public String getTheName() {
        return name;
    }

    @JsonProperty("diyName")
    public void setTheName(String name) {
        this.name = name;
    }
}

@Test
public void testJsonGet() throws JsonProcessingException {
    MyBean myBean = new MyBean(1, "alamide");
    ObjectMapper mapper = new ObjectMapper();
    mapper.disable(MapperFeature.USE_ANNOTATIONS);
    final String string = mapper.writeValueAsString(myBean);
    //{"id":1,"theName":"alamide"}
    System.out.println(string);
}
```