---
layout: post
title: Redis 学习笔记
categories: db
date: 2023-06-01
excerpt: Redis 的基础使用。
tags: DB Redis 
---
## 1.Redis 的安装
```bash
docker run --name some-redis -d redis:7.0
docker exec -it some-redis bash 

redis-cli
```

## 2.一些基本的操作
### 2.1 key 操作
```bash
#查看数据库数量的 database
config get database

#选择 8 号库
select 8

#查看当前库大小
dbsize

#查看当前库所有 key
keys *

#查看某个 key 是否存在
exists key

#查看 key 类型
type key

#删除指定 key 的数据
del key

#非阻塞删除，将 key 从 keyspace 删除，后续操作异步删除
unlink key 

#设置 key 的过期时间
expire key 10

#查看还有多少秒过期，-1 永不过期，-2 表示已过期
ttl key

#清空当前库
flushdb 

#清空所有库
flushall
```

### 2.2 String
String 类型是二进制安全的。Redis 的 String 可以包含人后数据，如序列化的对象，jpg 图像。

String 类型是 Redis 最基本的类型，一个 Redis 中字符串 value 最多为 512MB
```bash
#set <key><value> 添加键值对
set username alamide

#get <key> 查询键值
get username

#append <key><value> 将指定的值添加到原值末尾
append username 001

#strlen <key> 获取值的长度
strlen username

#setnx <key><value> key 不存在时，设置
setnx username alamide

#incr <key> 将 key 中存储的值增 1，只能对数字操作，如果为空，新增值为 1
incr times

#decr <key> 将 key 中存储的值增 1，只能对数字操作，如果为空，新增值为 -1
decr times

#incrby <key><步长> 将 key 中存储的值增自定义步长
incrby times 10

#incrby <key><步长> 将 key 中存储的值减自定义步长
decrby times 10

#mset <key1><value1><key2><value2>... 设置多组值
mset n1 1 n2 2 n3 3

#mget <key1><key2>... 获取多组值
mget n1 n2 n3

#msetnx <key1><value1><key2><value2>... 当所有的 key 都不存在时才会设置成功
msetnx n1 1 n4 4

#getrange <key><起始位置><结束位置> 获得值的范围，下标从 0 开始
getrange username 1 2 #la

#setrange <key><起始位置><value> 用 value 覆盖 key 起始位置的字符串
setrange username 0 y #ylamide

#setex <key><过期时间><value> 设置键值的同时，设置过期时间
setex username 8 alamide

#getset <key><value> 以新换旧，设置新值的同时获得旧值
getset username lalala
```

### 2.3 List
```bash
#lpush <key><value1><value2><vlaue3>... 从左边插入多值
lpush arr1 v1 v2 v3 v4

#lpushx <key> <value1> <value2>... 从左边向指定列表中追加元素
lpushx arr1 v10

#rpush <key><value1><value2><vlaue3>... 从右边插入多值
rpush arr1 v1 v2 v3 v4

#rpushx <key> <value1> <value2>... 从右边向指定列表中追加元素
rpushx arr1 v100

#lrange <key><start><stop> 按照索引获得元素，从左到右
lrange arr1 0 2 #v4 v3 v2

#lrange <key> 0 -1 获取所有元素
lrange arr1 0 -1 #v4 v3 v2 v1

#lpop <key><count> 从左边吐出几个值
lpop arr1 

#rpop <key><count> 从右边吐出几个值
rpop arr1

#lindex <key><index> 获取指定位置的元素，从左到右
lindex arr1 2

#llen <key> 获取列表长度
llen arr1

#linsert <key> before/after <value><newvalue> 在 value 的前面后后面插入新值
linsert arr1 before v3 v9

#lrem <key>n<value> 从左边删除 n 个 value
lrem arr1 2 v9 #从左边删除 2 个 v9

#lset <key><index><value> 指定下标设置
lset arr1 1 v10

````

### 2.4 Set
```bash
#sadd <key><value1><value2><value3>.... 将一个或多个 value 添加到集合中，自动去重
sadd set1 v1 v2 v3 v4 v1

#smembers <key> 查询集合中所有元素
smembers set1

#sismember <key><value> 判断集合中是否存在元素 1-存在 0-不存在
sismember set1 v3

#scard <key> 返回集合中元素个数
scard set1

#srem <key><value1><value2>...
srem set1 v1 v2

#spop <key> 随机从该集合中吐出一个值
spop set1

#srandmember <key><n> 随机从该集合中取出 n 个元素，不删除
srandmember set1 2

#smove <source><destination><value> 从一个集合中移动一个元素到另一个集合
smove set1 set2 v10

#sinter <key1><key2> 返回两个集合的交集
sinter set1 set2

#sunion <key1><key2> 返回两个集合的并集
sunion set1 set2

#sdiff <key1><key2> 返回两个元素的差集 key1 中有的，key2 中没有的
sdiff set1 set2
```

### 2.5 Hash 
时候存储对象
```bash
#hset <key><field><value> 设置 key 中 field 的值
hset student1 name alamide

#hmset <key><field1><value1><filed2><value2> 设置多个值
hmset student2 name alamide age 18 country china

#hexists <key><field> 查看 key 中指定的 filed 是否存在，存在返回 1，不存在返回 0
hexists student1 country

#hkeys <key> 查看 key 中所有的字段名
hkeys student1

#hvals <key> 查看 key 中所有字段的值
hvals student1

#hincrby <key><field><increment> 指定 key 的 filed 增加 increment
hincrby student2 age 1

#hsetnx <key><field><value> 不存在 key 时，设置
hsetnx student1 name lop

#hdel <key><filed> 删除指定属性
hdel student1 name
```

### 2.6 Zset
有序集合
```bash
#zadd <key><score1><value1><score2><value2>.... 将元素及其 score 加入到有序集合中 
zadd name 1 zhangsan 2 lisi  8 nanfeng 6 delu 

#zrange <key> <start> <stop> [withscores]返回片段，[withscores] 显示分数
zrange name 0 2

#zcard <key> 获取有序集合中的元素数
zcard name

#zcount <key> <min> <max> 计算在有序集合中指定 score 区间的元素个数
zcount name 1 2

#zrangebyscore <key> <min> <max> [withscores]，返回指定 score 区间元素，[withscores] 显示分数
zrangescore name 1 4

#zrevrangebyscore <key> <max> <min> 同上，从大到小排列
zrevrangebyscore name 4 1

#zincrby <key> <increment> <value> 为元素的 score 加上增量
zincrby name 10 lisi

#zrem <key> <value> 移除指定元素
zrem name lisi

#zrank <key> <value> 返回该值在集合中的排名，从 0 开始
zrank name zhangsan
```

## 3.JRedis
在 `redis-cli` 中运行的命令，在 `JRedis` 中有类似的方法
### 3.1 引入依赖
引入依赖，注：`jredis` 版本为 `4.3.0` 时会时不时的报，所以这里使用 `2.9.3`
```
redis.clients.jedis.exceptions.JedisConnectionException: Unexpected end of stream.

	at redis.clients.jedis.util.RedisInputStream.ensureFill(RedisInputStream.java:205)
	at redis.clients.jedis.util.RedisInputStream.readByte(RedisInputStream.java:46)
    ...
```

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>2.9.3</version>
</dependency>
```

### 3.2 String 操作
```java
@Test
public void testString() {
    try (Jedis jedis = new Jedis("localhost", 6379)){
        jedis.set("username", "alamide");
        jedis.set("age", "18");
        
        final Set<String> keys = jedis.keys("*");
        System.out.println(keys);

        final boolean exists = jedis.exists("username");
        System.out.println(exists);

        final long ttl = jedis.ttl("username");
        System.out.println(ttl);

        final String s = jedis.get("username");
        System.out.println(s);

        jedis.mset("k1", "v2", "k2", "v2");
        final List<String> mget = jedis.mget("k1", "k2");
        System.out.println(mget);

        final Long setnx = jedis.setnx("k1", "v11");
        System.out.println(setnx);

        final Long counter = jedis.incr("counter");
        System.out.println(counter);

        final String username = jedis.getrange("username", 0, 1);
        System.out.println(username);
    }
}
```

### 3.3 List 操作
```java
@Test
public void testList(){
    try (Jedis jedis = new Jedis("localhost", 6379)){
        final Long lpush = jedis.lpush("arr1", "v1", "v2", "v3");
        System.out.println(lpush);

        final Long lpushx = jedis.lpushx("arr1", "v100");
        System.out.println(lpushx);

        final List<String> arr1 = jedis.lrange("arr1", 0, -1);
        System.out.println(arr1);

        final Long llen = jedis.llen("arr1");
        System.out.println(llen);

        final String arr11 = jedis.lindex("arr1", 1);
        System.out.println(arr11);

        jedis.lrem("arr1", 10, "v100");

        final List<String> arr12 = jedis.lrange("arr1", 0, -1);
        System.out.println(arr12);
    }
}
```

### 3.4 Set 操作
```java
 @Test
public void testSet(){
    try (Jedis jedis = new Jedis("localhost", 6379)){
        jedis.sadd("set1", "v1", "v2", "v3");
        final Set<String> set1 = jedis.smembers("set1");
        System.out.println(set1);

        final Long srem = jedis.srem("set1", "v2");
        System.out.println(srem);

        final Set<String> set11 = jedis.smembers("set1");
        System.out.println(set11);
    }
}
```

### 3.5 Hash 操作
```java
@Test
public void testHash(){
    try (Jedis jedis = new Jedis("localhost", 6379)){
        Map<String, String> map = new HashMap<>();
        map.put("username", "alamide");
        jedis.hmset("student", map);

        final List<String> student = jedis.hmget("student", "username");
        System.out.println(student);
    }
}
```

### 3.6 ZSet 操作
```java
@Test
public void testZSet(){
    try (Jedis jedis = new Jedis("localhost", 6379)){
        jedis.zadd("zset1", 101, "meili");
        jedis.zadd("zset1", 10, "linda");
        final Set<String> zset1 = jedis.zrange("zset1", 0, -1);
        System.out.println(zset1);
    }
}
```

## 4.SpringBoot 整合 Redis
导入依赖，SpringBoot 版本 V3.1.0
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

配置相关属性，SpringBoot V3.1.0 中，RedisProperties 的前缀是 spring.data.redis，其中 host 默认为 localhost，port 为 6379，database 为 0
```yaml
spring:
  data:
    redis:
      host: localhost
      port: 6379
      database: 0
      timeout: 60s
      lettuce:
        pool:
          max-active: 20
          max-wait: -1
          max-idle: 5
          min-idle: 0
```

配置 RedisTemplate，主要是配置 key 序列化器和 value 序列化器
```java
public class JacksonObjectMapper extends ObjectMapper {

    public static final String DEFAULT_DATE_FORMAT = "yyyy-MM-dd";
    public static final String DEFAULT_DATE_TIME_FORMAT = "yyyy-MM-dd HH:mm:ss";
    public static final String DEFAULT_TIME_FORMAT = "HH:mm:ss";

    public JacksonObjectMapper() {
        super();
        //收到未知属性时不报异常
        this.configure(FAIL_ON_UNKNOWN_PROPERTIES, false);

        //反序列化时，属性不存在的兼容处理
        this.getDeserializationConfig().withoutFeatures(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES);


        SimpleModule simpleModule = new SimpleModule()
                .addDeserializer(LocalDateTime.class, new LocalDateTimeDeserializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)))
                .addDeserializer(LocalDate.class, new LocalDateDeserializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT)))
                .addDeserializer(LocalTime.class, new LocalTimeDeserializer(DateTimeFormatter.ofPattern(DEFAULT_TIME_FORMAT)))

                .addSerializer(BigInteger.class, ToStringSerializer.instance)
                .addSerializer(Long.class, ToStringSerializer.instance)
                .addSerializer(LocalDateTime.class, new LocalDateTimeSerializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_TIME_FORMAT)))
                .addSerializer(LocalDate.class, new LocalDateSerializer(DateTimeFormatter.ofPattern(DEFAULT_DATE_FORMAT)))
                .addSerializer(LocalTime.class, new LocalTimeSerializer(DateTimeFormatter.ofPattern(DEFAULT_TIME_FORMAT)));

        //注册功能模块 例如，可以添加自定义序列化器和反序列化器
        this.registerModule(simpleModule);
    }
}
@Configuration
public class RedisTemplateConfiguration {
    //将 JacksonObjectMapper 注入
    @Bean
    public RedisTemplate<Object, Object> redisTemplate(RedisConnectionFactory factory, ObjectMapper jacksonObjectMapper) {
        RedisTemplate<Object, Object> redisTemplate = new RedisTemplate<>();
        StringRedisSerializer stringRedisSerializer = new StringRedisSerializer();
        Jackson2JsonRedisSerializer<Object> jackson2JsonRedisSerializer = new Jackson2JsonRedisSerializer<>(jacksonObjectMapper, Object.class);
        redisTemplate.setConnectionFactory(factory);
        redisTemplate.setKeySerializer(stringRedisSerializer);
        redisTemplate.setHashKeySerializer(stringRedisSerializer);
        redisTemplate.setValueSerializer(jackson2JsonRedisSerializer);
        redisTemplate.setHashValueSerializer(jackson2JsonRedisSerializer);
        return redisTemplate;
    }
}
```
