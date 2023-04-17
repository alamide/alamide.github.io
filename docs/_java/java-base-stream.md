---
layout: post
title: Java 基础 Stream
categories: java
tags: Java Stream
date: 2023-03-27
---
Java 基础语法 Stream，学习内容来自 《Java 实战（第二版）》
<!--more-->
## 1.使用流进行函数式数据处理
> 数据就像一条河流：它可以被观测，被过滤，被操作，或者为新的消费者与另外一条流合并为一条新的流（这是别人写的，忘记出处了，是一篇 RxJava 的博客）。

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class Dish {
    private String name;
    private boolean vegetarian;
    private int calories;
    private Type type;

    @Override
    public String toString() {
        return name;
    }

    public enum Type {MEAT, FISH, OTHER}

    public static List<Dish> getDishes() {
        return Arrays.asList(
                new Dish("pork", false, 800, Type.MEAT),
                new Dish("beef", false, 700, Type.MEAT),
                new Dish("chicken", false, 400, Type.MEAT),
                new Dish("french fries", true, 530, Type.OTHER),
                new Dish("rice", true, 350, Type.OTHER),
                new Dish("season fruit", true, 120, Type.OTHER),
                new Dish("pizza", true, 550, Type.OTHER),
                new Dish("prawns", false, 300, Type.FISH),
                new Dish("salmon", false, 450, Type.FISH));
    }
}
```

### 1.1 筛选
谓词筛选，筛选蔬菜
```java
Dish.getDishes().stream()
                .filter(Dish::isVegetarian)
                .forEach(System.out::println);
```

去重
```java
Stream.of(1, 2, 1, 3, 4, 2, 10, 10)
                .filter(i -> i % 2 == 0)
                .distinct()
                .forEach(System.out::println);
```

取前两个
```java
IntStream.rangeClosed(1, 20)
                .limit(2)
                .forEach(System.out::println);
```

### 1.2 流的切片 
对于有序的列表，可以使用 `takeWhile` （遇到第一个不符合，停止遍历，返回前面的元素）和 `dropWhile` （遇到第一个符合，返回余下的元素）高效的选择流中的元素，两个方法互补

获取热量小于 600，使用 filter 效率不高，需要遍历每一个元素 
```java
Dish.getDishes().stream()
                .sorted(Comparator.comparingInt(Dish::getCalories))
                .takeWhile(d -> d.getCalories() < 600)
                .forEach(System.out::println);
```

获取热量大于 600
```java
Dish.getDishes().stream()
                .sorted(Comparator.comparingInt(Dish::getCalories))
                .dropWhile(d -> d.getCalories() < 600)
                .forEach(System.out::println);
```

截短流
```java
IntStream.rangeClosed(1, 20)
                .limit(2)
                .forEach(System.out::println);
```

跳过元素，skip，与 limit 互补
```java
IntStream.rangeClosed(1, 20)
                .skip(2)
                .forEach(System.out::println);
```

### 1.3 映射
流支持map方法，它会接受一个函数作为参数。这个函数会被应用到每个元素上，并将其映射成一个新的元素。
```java
Dish.getDishes().stream()
                .map(Dish::getName)
                .forEach(System.out::println);
```

返回每道菜的菜名长度
```java
Dish.getDishes().stream()
                .map(Dish::getName)
                .map(String::length)
                .forEach(System.out::println);
```

### 1.4 流的扁平化
将数组中的字符串拆解为字母，flatMap 将流合并
```java
Stream.of("Hello", "World")
                .map(s -> s.split(""))
                .flatMap(Arrays::stream)
                .toList();
```

给定两个数字列表，如何返回所有的数对呢？例如，给定列表[1, 2, 3]和列表[3, 4]，应该返回[(1, 3), (1, 4), (2, 3), (2, 4), (3, 3), (3, 4)]。为简单起见，你可以用有两个元素的数组来代表数对。
```java
Stream.of(1, 2, 3)
                .flatMap(i -> Stream.of(3, 4).map(j -> new int[]{i, j}))
                .forEach(ints -> System.out.println(Arrays.toString(ints)));
```

### 1.5 查找和匹配
检查谓词是否至少匹配一个元素
```java
final boolean anyMatch = Dish.getDishes().stream().anyMatch(Dish::isVegetarian);
System.out.println(anyMatch);
```

检查谓词是否匹配所有元素
```java
final boolean anyMatch = Dish.getDishes().stream().allMatch(Dish::isVegetarian);
System.out.println(anyMatch);

final boolean anyMatch = Dish.getDishes().stream().noneMatch(d -> !d.isVegetarian());
System.out.println(anyMatch);
```

查找元素
```java
final Optional<Dish> any = Dish.getDishes().stream()
                .filter(Dish::isVegetarian)
                .findAny();
any.ifPresent(System.out::println);

//findFirst 查找第一个
final Optional<Dish> any = Dish.getDishes().stream()
                .filter(Dish::isVegetarian)
                .findFirst();
any.ifPresent(System.out::println);
```

### 1.6 规约
元素求和，相乘，最大，最小
```java
int sum = Stream.of(1, 10, 3, 4, 5).reduce(0, Integer::sum);

int multi = Stream.of(1, 10, 3, 4, 5).reduce(1, (a, b) -> a * b);

Optional<Integer> reduce = Stream.of(1, 10, 3, 4, 5).reduce((a, b) -> a * b);

Optional<Integer> max = Stream.of(1, 10, 3, 4, 5).reduce(Integer::max);

Optional<Integer> min = Stream.of(1, 10, 3, 4, 5).reduce(Integer::min);
```

### 1.7 原始类型流特化
要记住的是，这些特化的原因并不在于流的复杂性，而是装箱造成的复杂性——即类似int和Integer之间的效率差异。
```java
IntStream intStream = Transaction.getTransaction().stream()
                .mapToInt(Transaction::getValue);

Stream<Integer> integerStream = intStream.boxed();
```

`OptionalInt`，使用 `OptionalInt` 的原因是可能 `Stream` 中没有元素
```java
final OptionalInt max = Transaction.getTransaction().stream()
                .mapToInt(Transaction::getValue)
                .max();
max.ifPresent(System.out::println);
```

数值范围
```java
IntStream.rangeClosed(1, 3).forEach(System.out::println);
IntStream.range(1, 3).forEach(System.out::println);
```

### 1.8 构建流
```java
Stream.of(1, 2, 3).forEach(System.out::println);
Stream<String> empty = Stream.empty();

Stream<String> values =
                Stream.of("config", "home", "user")
                        .flatMap(key -> Stream.ofNullable(System.getProperty(key)));
values.forEach(System.out::println);

int[] numbers = {1, 2, 3, 4, 5};
final IntStream stream = Arrays.stream(numbers);
```

由文件生成流
```java
try (Stream<String> lines = Files.lines(Paths.get("data.txt"))) {
    lines.map(s -> s.split(" ")).flatMap(Arrays::stream).distinct().forEach(System.out::println);
} catch (IOException e) {

}
```

### 1.9 由函数生成流：创建无限流
```java
Stream.iterate(0, n -> n + 2)
                .limit(10)
                .forEach(System.out::println);

//也可以使用下面代码
Stream.iterate(0, n -> n + 2)
                .takeWhile(n -> n < 20)
                .forEach(System.out::println);

// 不能使用如下代码，无法停止
// Stream.iterate(0, n -> n < 20, n -> n + 2)
//                 .filter(n -> n < 20)
//                 .forEach(System.out::println);

//斐波那契数列
String collect = Stream.iterate(new int[]{0, 1}, ints -> new int[]{ints[1], ints[0] + ints[1]})
                .limit(10)
                .map(ints -> String.valueOf(ints[0]))
                .collect(Collectors.joining(", "));
System.out.println(collect);

//小于 20 停止
Stream.iterate(0, n -> n < 20, n -> n + 2)
                .limit(10)
                .forEach(System.out::println);

//调用方法生成流
Stream.generate(Math::random)
                .limit(10)
                .forEach(System.out::println);

//斐波那契数列
IntSupplier fib = new IntSupplier() {
    private int previous = 0;
    private int current = 1;

    @Override
    public int getAsInt() {
        int res = previous;
        previous = current;
        current += res;
        return res;
    }
};

IntStream.generate(fib).limit(10).forEach(System.out::println);

```

## 2.用流收集数据
### 2.1 查找流中最大值和最小值
```java
final Optional<Dish> max = Dish.getDishes().stream().max(Comparator.comparing(Dish::getCalories));
max.ifPresent(System.out::println);

// final Integer sum = Dish.getDishes().stream().collect(Collectors.summingInt(Dish::getCalories));
// final Integer sum = Dish.getDishes().stream().mapToInt(Dish::getCalories).sum();
// final double average = Dish.getDishes().stream().collect(Collectors.averagingInt(Dish::getCalories));

final IntSummaryStatistics summaryStatistics = Dish.getDishes().stream().collect(Collectors.summarizingInt(Dish::getCalories));
final long sum = summaryStatistics.getSum();
final double average = summaryStatistics.getAverage();
final int max = summaryStatistics.getMax();
final int min = summaryStatistics.getMin();
final long count = summaryStatistics.getCount();
log.info("sum: {}, average: {}, max: {}, min: {}, count: {}", sum, average, max, min, count);
```

### 2.2 连接字符串
```java
//拼接成字符串
final String shortMenu = Dish.getDishes().stream().map(Dish::getName).collect(Collectors.joining(", "));
```

### 2.3 分组
```java
//分组
final Map<Dish.Type, List<Dish>> typeListMap = Dish.getDishes().stream().collect(Collectors.groupingBy(Dish::getType));


public enum CaloricLevel { DIET, NORMAL, FAT }

Map<CaloricLevel, List<Dish>> dishesByCaloricLevel = Dish.getDishes().stream().collect(
                Collectors.groupingBy(dish -> {
                    if (dish.getCalories() <= 400) return CaloricLevel.DIET;
                    else if (dish.getCalories() <= 700) return CaloricLevel.NORMAL;
                    else return CaloricLevel.FAT;
                } ));


//{MEAT=[pork, beef], OTHER=[french fries, pizza]}
final Map<Dish.Type, List<Dish>> collect = Dish.getDishes().stream()
                .filter(dish -> dish.getCalories() > 500)
                .collect(groupingBy(Dish::getType));

//{FISH=[], MEAT=[pork, beef], OTHER=[french fries, pizza]}
final Map<Dish.Type, List<Dish>> typeListMap = Dish.getDishes().stream()
        .collect(
                Collectors.groupingBy(
                        Dish::getType,
                        Collectors.filtering(dish -> dish.getCalories() > 500, Collectors.toList())
                )
        );

Map<Dish.Type, List<String>> dishNamesByType =
                Dish.getDishes().stream()
                        .collect(groupingBy(Dish::getType, mapping(Dish::getName, toList())));


Map<String, List<String>> dishTags = new HashMap<>();
dishTags.put("pork", asList("greasy", "salty"));
dishTags.put("beef", asList("salty", "roasted"));
dishTags.put("chicken", asList("fried", "crisp"));
dishTags.put("french fries", asList("greasy", "fried"));
dishTags.put("rice", asList("light", "natural"));
dishTags.put("season fruit", asList("fresh", "natural"));
dishTags.put("pizza", asList("tasty", "salty"));
dishTags.put("prawns", asList("tasty", "roasted"));
dishTags.put("salmon", asList("delicious", "fresh"));

final Map<Dish.Type, Set<String>> collect = Dish.getDishes().stream()
        .collect(groupingBy(Dish::getType, flatMapping(dish -> dishTags.get(dish.getName()).stream(), toSet())));

```
### 2.4 多级分组
```java
//多级分组
final Map<Dish.Type, Map<CaloricLevel, List<String>>> collect = Dish.getDishes().stream()
                .collect(groupingBy(Dish::getType,
                                groupingBy(dish -> {
                                    if (dish.getCalories() <= 400) return CaloricLevel.DIET;
                                    else if (dish.getCalories() <= 700) return CaloricLevel.NORMAL;
                                    else return CaloricLevel.FAT;
                                }, mapping(Dish::getName, toList()))
                        )
                );

//maxBy
final Map<Dish.Type, Optional<Dish>> collect = Dish.getDishes().stream()
                .collect(groupingBy(Dish::getType, maxBy(Comparator.comparing(Dish::getCalories))));

//获取 name
final Map<Dish.Type, String> collect = Dish.getDishes().stream()
                .collect(groupingBy(Dish::getType,
                        collectingAndThen(maxBy(Comparator.comparing(Dish::getCalories)), op -> op.orElseThrow().getName())
                ));

final Map<Dish.Type, Integer> collect = Dish.getDishes().stream()
                .collect(groupingBy(Dish::getType, summingInt(Dish::getCalories)));

final Map<Dish.Type, Set<CaloricLevel>> collect = Dish.getDishes().stream()
                .collect(groupingBy(Dish::getType,
                        mapping(dish -> {
                            if (dish.getCalories() <= 400) return CaloricLevel.DIET;
                            else if (dish.getCalories() <= 700) return CaloricLevel.NORMAL;
                            else return CaloricLevel.FAT;
                        }, toCollection(HashSet::new))));

```
### 2.5 分区
```java          
//分区
final Map<Boolean, Set<String>> collect = Dish.getDishes().stream()
        .collect(partitioningBy(Dish::isVegetarian, mapping(Dish::getName, toSet())));

final Map<Boolean, Map<Dish.Type, List<String>>> collect = Dish.getDishes().stream()
                .collect(partitioningBy(Dish::isVegetarian, groupingBy(Dish::getType, mapping(Dish::getName, toList()))));

final Map<Boolean, String> collect = Dish.getDishes().stream()
                .collect(partitioningBy(Dish::isVegetarian,
                                collectingAndThen(
                                        maxBy(Comparator.comparing(Dish::getCalories)),
                                        op -> op.get().getName())
                        )
                );
```

### 2.6 求质数与自定义收集器
```java
@Test
public void test(){
    long fastest = Integer.MAX_VALUE;
    for (int i = 0; i < 10; i++) {
        long start = System.currentTimeMillis();
        IntStream.rangeClosed(2, 1_000_000).boxed().collect(new PrimeNumberCollector());
        long duration = System.currentTimeMillis() - start;
        if (fastest > duration) fastest = duration;
    }

    System.out.println(fastest);

    fastest = Integer.MAX_VALUE;

    for (int i = 0; i < 10; i++) {
        long start = System.currentTimeMillis();
        partitionPrimes(1_000_000);
        long duration = System.currentTimeMillis() - start;
        if (fastest > duration) fastest = duration;
    }
    System.out.println(fastest);
}


//求质数
public Map<Boolean, List<Integer>> partitionPrimes(int n){
    return IntStream.rangeClosed(2, n)
            .boxed()
            .collect(partitioningBy(this::isPrime));
}

public boolean isPrime(int candidate){
    int candidateRoot = (int) Math.sqrt(candidate);
    return IntStream.rangeClosed(2, candidateRoot).noneMatch( i -> candidate % i == 0);
}

public static class PrimeNumberCollector implements Collector<Integer, Map<Boolean, List<Integer>>, Map<Boolean, List<Integer>>> {

    @Override
    public Supplier<Map<Boolean, List<Integer>>> supplier() {
        return () -> {
            HashMap<Boolean, List<Integer>> hashMap = new HashMap<>();
            hashMap.put(true, new ArrayList<>());
            hashMap.put(false, new ArrayList<>());
            return hashMap;
        };
    }

    @Override
    public BiConsumer<Map<Boolean, List<Integer>>, Integer> accumulator() {
        return (Map<Boolean, List<Integer>> acc, Integer candidate) -> {
            acc.get(isPrime(acc.get(true), candidate)).add(candidate);
        };
    }

    @Override
    public BinaryOperator<Map<Boolean, List<Integer>>> combiner() {
        return (Map<Boolean, List<Integer>> map1, Map<Boolean, List<Integer>> map2) -> {
            map1.get(true).addAll(map2.get(true));
            map1.get(false).addAll(map2.get(false));
            return map1;
        };
    }

    @Override
    public Function<Map<Boolean, List<Integer>>, Map<Boolean, List<Integer>>> finisher() {
        return Function.identity();
    }

    @Override
    public Set<Characteristics> characteristics() {
        return Collections.unmodifiableSet(EnumSet.of(Characteristics.IDENTITY_FINISH));
    }
}

public static boolean isPrime(List<Integer> primes, int candidate) {
    int candidateRoot = (int) Math.sqrt(candidate);
    return primes.stream()
            .takeWhile(i -> i <= candidateRoot)
            .noneMatch(i -> candidate % i == 0);
}
```

output:
```
PrimeNumberCollector average cost time: 294ms
Normal average cost time: 469ms
```

### 2.7 求质数与自定义收集器