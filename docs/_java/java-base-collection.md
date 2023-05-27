---
layout: post
title: Java 基础 Collection 增强
categories: java
tags: Java Collection
date: 2023-05-09
---
Java 基础语法 Collection 增强，学习内容来自 《Java 实战（第二版）》
<!--more-->
## 1.集合工厂

### 1.1 List.of
不支持 set 和 add

>可能你会问能不能使用Stream API而不是新的集合工厂方法来创建这种列表。毕竟前面章节里曾经使用收集器的Collectors.toList()方法将流转换为了列表。我的建议是除非你需要进行某种形式的数据处理并对数据进行转换，否则应该尽量使用工厂方法。工厂方法使用起来更简单，实现也更容易，并且在大多数情况下就够用了。
```java
final List<String> list = List.of("a", "b", "c");
```

### 1.2 Set.of
创建不可变集合
```java
final Set<String> set = Set.of("a", "b", "c");
```

### 1.3 Map.of
```java
final Map<String, Integer> map = Map.of("tom", 20, "jack", 40);
final Map<String, Integer> entries = Map.ofEntries(Map.entry("tom", 20), Map.entry("jack", 40));
```

## 2.使用 List 和 Set
### 2.1 removeIf
移除集合中匹配指定谓词的元素
```java
List<String> stringList = new ArrayList<>(List.of("a", "b", "c"));

for (String s: stringList){
    if(s.equals("a")){
        stringList.remove(s);
    }
}
```

使用上述方法移除元素时，会报 java.util.ConcurrentModificationException，可以使用如下方法
```java
final Iterator<String> iterator = stringList.iterator();
while (iterator.hasNext()){
    final String next = iterator.next();
    if(next.equals("a")) iterator.remove();
}
```

还是有点繁琐的，Java8 之后可以使用如下方法
```java
stringList.removeIf(next -> next.equals("a"));
```

### 2.2 replaceAll
```java
//java8 之前
for (int i=0; i< stringList.size(); i++){
    stringList.set(i, stringList.get(i).toUpperCase());
}
//java8 之后
stringList.replaceAll(String::toUpperCase);
```

## 3.Map
### 3.1 forEach
```java
//java8 之前
HashMap<String, Integer> map = new HashMap<>(Map.ofEntries(Map.entry("tom", 20), Map.entry("jack", 40)));
for (Map.Entry<String, Integer> entry: map.entrySet()){
    System.out.println(entry.getKey() + "," +entry.getValue());
}

//java8 之后
map.forEach((s, integer) -> System.out.println(s + "," + integer));
```

### 3.2 排序
```java
//依据 key 排序
map.entrySet().stream().sorted(Map.Entry.comparingByKey()).forEachOrdered(System.out::println);
//依据 value 排序
map.entrySet().stream().sorted(Map.Entry.comparingByValue()).forEachOrdered(System.out::println);
```

## 3.3 其它
```java
//如果 Map 中不存在这个值或为空，那么使用该键计算，并添加到 Map 中
map.computeIfAbsent("h", s -> 100);

//存在则计算，并添加到 map 中
map.computeIfPresent("tom", (s, integer) -> integer+100);

//计算并存储，不存在就添加，存在就修改
map.compute("jjj" , (s, integer) -> 290);

//移除 key=tom 且 value=120
map.remove("tom", 120);

//替换
map.replace("jack", 180);
map.replace("jack", 180, 100);
map.replaceAll((s, integer) -> 20);

//合并map，下面的合并会丢失信息，tom 18888888888 丢失
Map<String, String> phone1 = Map.of("tom", "18888888888", "hack", "19999999999");
Map<String, String> phone2 = Map.of("tom", "18855558888", "jack", "17799998888");
HashMap<String, String> hashMap = new HashMap<>(phone1);
hashMap.putAll(phone2);

//合并
phone2.forEach((k, v) -> hashMap.merge(k, v, (oldP, newP) -> oldP + "&" + newP));

//统计字符，书上的例子好像错了 不应该是 count + 1，应该是 k + 1，而且变量名起的不好
char[] chars = readString().toCharArray();
Map<String, Long> charCount = new HashMap<>();

for (char c : chars) {
    charCount.merge(String.valueOf(c), 1L, (oldValue, newValue) -> oldValue + 1L);
}

//移除
charCount.entrySet().removeIf(stringLongEntry -> stringLongEntry.getValue() > 1);
```

## 4.ConcurrentHashMap
线程安全，提供一些其它的操作方法，serachKeys、searchValues 等等

计数 
```java
ConcurrentHashMap<String, Long> concurrentHashMap = new ConcurrentHashMap<>(charCount);
final long count = concurrentHashMap.mappingCount();
System.out.println(count);

```