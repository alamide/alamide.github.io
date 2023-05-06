---
layout: post
title: Jackson ObjectMapper
categories: java json
excerpt: Jackson ObjectMapper 
tags: java json
date: 2023-05-06
---
引入依赖
```xml
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.14.2</version>
</dependency>
```

## 1.Java Object To JSON
```java
@Data
@ToString
@AllArgsConstructor
@NoArgsConstructor
public class Car {
    private String color;
    private String type;
    private int price;
}

@Test
public void testObjectToJSON() throws JsonProcessingException {
    Car car = new Car("red", "tesla", 250_000);
    ObjectMapper objectMapper = new ObjectMapper();
    final String string = objectMapper.writeValueAsString(car);
    System.out.println(string);
}
```

## 2.JSON To Object
```java
String json = "{ \"color\" : \"Black\", \"type\" : \"BMW\", \"price\":180000 }";
final Car readValue = objectMapper.readValue(json, Car.class);
```

还可以从文件中读取 json
data.json
```
{"color":"black","type":"byd","price":178000}
```

```java
final Car value = objectMapper.readValue(this.getClass().getClassLoader().getResourceAsStream("data.json"), Car.class);
```

可以读取 URL 中的 json
```java
Car car = objectMapper.readValue(new URL("file:src/test/resources/json_car.json"), Car.class);
```

## 3.JSON to Jackson JsonNode
```java
String json = "{ \"color\" : \"Black\", \"type\" : \"BMW\", \"price\":180000 }";
final JsonNode jsonNode = objectMapper.readTree(json);
final String color = jsonNode.get("color").asText();
```

## 4.Creating a Java List From a JSON Array String
data.json
```
[
  {"color":"black","type":"byd","price":178000},
  {"color":"red","type":"benzi","price":520000}
]
```

```java
final List<Car> cars = objectMapper.readValue(
                this.getClass().getClassLoader().getResourceAsStream("data.json"),
                new TypeReference<List<Car>>() {
                });
```

## 5.Creating Java Map From JSON String
```java
String json = "{ \"color\" : \"Black\", \"type\" : \"BMW\", \"price\":180000 }";
final Map<String, Object> objectMap = objectMapper.readValue(json, new TypeReference<Map<String, Object>>() {});
```

## 6.Configuring Serialization or Deserialization Feature
```java
String jsonString
                = "{ \"color\" : \"Black\", \"type\" : \"Fiat\", \"year\" : \"1970\" }";
//如果不设置，则会解析错误 UnrecognizedPropertyException
objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
final Car value = objectMapper.readValue(jsonString, Car.class);

//也可以使用 JsonNode
final JsonNode readTree = objectMapper.readTree(jsonString);
final String year = readTree.get("year").asText();

String jsonString
                = "{ \"color\" : \"Black\", \"type\" : \"Fiat\",\"price\" : \"null\"}";
//对于基本数据类型是否允许 NULL，默认为 false
objectMapper.configure(DeserializationFeature.FAIL_ON_NULL_FOR_PRIMITIVES, false);
final Car value = objectMapper.readValue(jsonString, Car.class);
```

## 7.Custom Serializer 
自定义序列化器
```java
public class CustomSerializer extends StdSerializer<Car> {
    protected CustomSerializer(Class<Car> t) {
        super(t);
    }
    public CustomSerializer(){
        this(null);
    }
    @Override
    public void serialize(Car value, JsonGenerator gen, SerializerProvider provider) throws IOException {
        gen.writeStartObject();
        gen.writeStringField("carBrand", value.getType());
        gen.writeEndObject();
    }
}

ObjectMapper objectMapper = new ObjectMapper();
SimpleModule module = new SimpleModule("CustomSerializer", new Version(1,0,0,null,null,null));
module.addSerializer(Car.class, new CustomSerializer());
objectMapper.registerModule(module);
final String valueAsString = objectMapper.writeValueAsString(car);
//{"carBrand":"tesla"}
System.out.println(valueAsString);
```

## 8.Custom Deserializer 
```java
public class CustomCarDeserializer extends StdDeserializer<Car> {
    protected CustomCarDeserializer(Class<?> vc) {
        super(vc);
    }
    public CustomCarDeserializer(){
        this(null);
    }

    @Override
    public Car deserialize(JsonParser p, DeserializationContext ctxt) throws IOException, JacksonException {
        Car car = new Car();
        final ObjectCodec codec = p.getCodec();
        final JsonNode node = codec.readTree(p);

        final JsonNode colorNode = node.get("colo");
        final String color = colorNode.asText();
        car.setColor(color);
        return car;
    }
}

ObjectMapper objectMapper = new ObjectMapper();
SimpleModule module = new SimpleModule("CustomSerializer", new Version(1,0,0,null,null,null));
module.addSerializer(Car.class, new CustomSerializer());
module.addDeserializer(Car.class, new CustomCarDeserializer());

objectMapper.registerModule(module);
json = "{ \"colo\" : \"Black\", \"type\" : \"BMW\" }";
final Car car = objectMapper.readValue(json, Car.class);
//Car(color=Black, type=null, price=0)
System.out.println(car);
```

## 9.Handling Date Formats
```java
@AllArgsConstructor
public class Request {
    public Car car;
    public Date datePurchased;
}

@Test
public void testDateFormat() throws JsonProcessingException {
    ObjectMapper objectMapper = new ObjectMapper();
    objectMapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

    Request request = new Request(new Car("red", null, 10000), new Date());

    final String string = objectMapper.writeValueAsString(request);
    System.out.println(string);

    String json = "{\"car\":{\"color\":\"red\",\"type\":null,\"price\":10000},\"datePurchased\":\"2023-05-06 19:20:08\"}";
    final Request value = objectMapper.readValue(json, Request.class);
    System.out.println(value);
}
```

## 10.Handling Collections
data.json
```
[
  {"color":"black","type":"byd","price":178000},
  {"color":"red","type":"benzi","price":520000}
]
```

```java
@Test
public void testCollections() throws IOException {
    ObjectMapper objectMapper = new ObjectMapper();
    // objectMapper.configure(DeserializationFeature.USE_JAVA_ARRAY_FOR_JSON_ARRAY, true); 无效
    final Car[] cars = objectMapper.readValue(
            this.getClass().getClassLoader().getResourceAsStream("data.json"),
            Car[].class);
    System.out.println(Arrays.toString(cars));

    final List<Car> carList = objectMapper.readValue(
            this.getClass().getClassLoader().getResourceAsStream("data.json"),
            new TypeReference<List<Car>>() {
            });
    System.out.println(carList);
}
```