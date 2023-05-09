---
layout: post
title: Java 基础 Lambda
categories: java
tags: Java Lambda
date: 2023-04-11
isHidden: true
---
Java 基础语法 Lambda，学习内容来自 《Java 实战（第二版）》
<!--more-->
## 1.一些术语
* 谓词（ `predicate` ）在数学上常常用来代表类似于函数的东西，它接受一个参数值，并返回 `true` 或 `false`

* 行为参数化就是一个方法接受多个不同的行为作为参数，并在内部使用它们，完成不同行为的能力。和多态类似

* 函数接口：一言以蔽之，函数式接口就是只定义一个抽象方法的接口。哪怕有很多默认方法，只要接口只定义了一个抽象方法，它就仍然是一个函数式接口



## 2.Lambda 的引入
引入 `Lambda` 的目标之一就是简化代码，让代码看起来更简洁。

有这么一个需求，要求依据不同的条件，筛选出符合要求的苹果，我能写到的最简式是这样
```java
public class Lambda {
    public interface IFilter<T> {
        boolean filter(T t);
    }

    @AllArgsConstructor
    @ToString
    @Data
    public static class Apple {
        private String color;
        private Integer weight;
        private String country;
    }

    public static List<Apple> filterApples(List<Apple> apples, IFilter<Apple>... filters) {
        List<Apple> result = new ArrayList<>();
        for (Apple apple : apples) {
            boolean pass = true;
            for (IFilter<Apple> filter : filters) {
                pass = pass && filter.filter(apple);
            }
            if (pass) {
                result.add(apple);
            }
        }
        return result;
    }

    public static void main(String[] args) {
        List<Apple> apples = Arrays.asList(
                new Apple("green", 23, "china"),
                new Apple("red", 25, "china"),
                new Apple("green", 30, "US"),
                new Apple("red", 50, "china"),
                new Apple("white", 18, "UK"),
                new Apple("black", 19, "HK"),
                new Apple("red", 90, "TW"),
                new Apple("yellow", 21, "china"));
        //下面的代码好啰嗦
        final List<Apple> filterApples = filterApples(apples,
                new IFilter<Apple>() {
                    @Override
                    public boolean filter(Apple apple) {
                        return Objects.equals(apple.color, "red");
                    }
                },
                new IFilter<Apple>() {
                    @Override
                    public boolean filter(Apple apple) {
                        return apple.weight > 30;
                    }
                },
                new IFilter<Apple>() {
                    @Override
                    public boolean filter(Apple apple) {
                        return Objects.equals("china", apple.country);
                    }
                });

        System.out.println(filterApples);
    }
}
```

代码中的匿名内部类，确实显的十分啰嗦，重复且啰嗦。其实 `IFilter` 中必要的就是参数和返回结果，其它都是重复且次要的(其它部分 Java 编译器，完全可以自行推断)，要是我们能有一个表达式
是传入参数，返回结果就好了。`Lambda` 正是这为了解决这个问题而生，

使用 `Lambda` 简化一下

```java
final List<Apple> filterApples = filterApples(apples,
apple -> Objects.equals(apple.color, "red"),
apple -> apple.weight > 30,
apple -> Objects.equals("china", apple.country));
```
清爽了很多，`apple` 表示传入的参数， `->` 后面是返回的结果，只写必要的。

### 2.1 基本使用

基本语法
```
(parameters) -> expression
(parameters) -> { statements; }
```

* 为什么叫 `Lambda` ？其实它起源于学术界开发出的一套用来描述计算的 `λ` 演算法

* 哪里可以使用 `Lambda` ? 你可以在函数式（只定义一个抽象方法的接口）接口上使用 `Lambda` 表达式

* `@FunctionalInterface` 这个标注用于表示该接口会设计成一个函数式接口，因此对文档来说非常有用

### 2.2 类库中已定义的函数式接口
`Java8` 中已经有一些已定义的函数式接口了
#### 2.2.1 java.util.function.Predicate&lt;T&gt;
`T -> boolean`
```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);
}
```

#### 2.2.2 java.util.function.Consumer&lt;T&gt;
`T -> Void`
```java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
}
```

#### 2.2.3 java.util.function.Function&lt;T, R&gt;
`T -> R`

相当于类型转换，输入一个对象，再转换封装成另一个对象
```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);
}
```

#### 2.2.4 java.util.function.Supplier&lt;T&gt;
`() -> T`
```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
```

#### 2.2.5 java.util.function.BiFunction&lt;T, U, R&gt;
`(T, U) -> R`
```java
@FunctionalInterface
public interface BiFunction<T, U, R> {
    R apply(T t, U u);
}
```

#### 2.2.6 java.util.function.IntBinaryOperator
`(int, int) -> int`
```java
@FunctionalInterface
public interface IntBinaryOperator {
    int applyAsInt(int left, int right);
}
```

### 2.3 基本类型特化
基本数据类型在拆箱、装箱时是需要付出代价的，所以 `Java 8` 为前面所说的函数式接口带来了一个专门的版本，以便在输入和输出都是基本类型时避免自动装箱的操作。

有 `DoublePredicate` 、`DoublePredicate` 、`IntConsumer` 、`LongBinaryOperator` 、`IntFunction` 等

```java
IntPredicate predicate1 = (int i) -> i % 2 == 0;
Predicate<Integer> predicate2 = (Integer i) -> i % 2 == 0;
```

### 2.4 同样的Lambda，不同的函数式接口
```java
Callable<Integer> c = () -> 42;
PrivilegedAction<Integer> p = () -> 42;
```
上面同一个 `lambda` 可以表示不同的函数式接口

二义性
```java
public void execute(Runnable runnable) {
    runnable.run();
}
public void execute(Action<T> action) {
    action.act();
}
@FunctionalInterface
interface Action {
    void act();
}
```
当调用 `execute(() -> {})` 会产生二义性，`() -> {}` 既能符合 `Runnable` 的函式接口，又符合 `Action` ，为了消除这个二义性，可以这么解决
```java
execute((Action) () -> { });
```

### 2.5 方法引用
>方法引用可以被看作仅仅调用特定方法的Lambda的一种快捷写法。它的基本思想是，如果一个Lambda代表的只是“直接调用这个方法”，那最好还是用名称来调用它，而不是去描述如何调用它。

>当你需要使用方法引用时，目标引用放在分隔符：：前，方法的名称放在后面。例如，Apple::getWeight就是引用了Apple类中定义的方法getWeight。请记住，getWeight后面不需要括号，因为你没有实际调用这个方法，只是引用了它的名称。方法引用就是Lambda表达式(Apple apple) -> apple.getWeight()的快捷写法。

方法引用主要有三类
1. 指向静态方法的方法引用（例如Integer的parseInt方法，写作Integer::parseInt）。

2. 指向任意类型实例方法的方法引用（例如String的length方法，写作String::length）。

3. 指向现存对象或表达式实例方法的方法引用（假设你有一个局部变量expensive Transaction保存了Transaction类型的对象，它提供了实例方法getValue，那你就可以这么写expensive-Transaction::getValue）。

```java
public class Lambda {

    public static void main(String[] args) {
        List<String> stringList = new ArrayList<>();
        final List<String> filterString = filterString(stringList, Lambda::isLongString);
        System.out.println(filterString);
    }

    public static boolean isLongString(String s){
        return s.length() > 10;
    }

    public static List<String> filterString(List<String> strings, Predicate<String> predicate){
        List<String> result = new ArrayList<>();
        for (String str: strings){
            if(predicate.test(str)){
                result.add(str);
            }
        }
        return result;
    }
}
```

`Lambda::isLongString` 相当于 `s -> s.length > 10` ，也就是相当于引用了 `isLongString` 方法，

```java
List<String> stringList = new ArrayList<>();
//java8 之前写法
stringList.sort(new Comparator<String>() {
    @Override
    public int compare(String o1, String o2) {
        return o1.compareToIgnoreCase(o2);
    }
});

//lambda 写法
stringList.sort((o1, o2) -> o1.compareToIgnoreCase(o2));

//方法引用，暂时还是有点不适应
stringList.sort(String::compareToIgnoreCase);
```

### 2.6 构造函数引用
对于一个现有构造函数，你可以利用它的名称和关键字new来创建它的一个引用：ClassName::new。

无参构造
```java
Supplier<Apple> supplier = Apple::new;
System.out.println(supplier.get());
```
等价于
```java
Supplier<Apple> supplier = () -> new Apple();
```

多参构造函数
```java
//BiFunction<String, Integer, Apple> c = (color, weight) -> new Apple(color, weight)
BiFunction<String, Integer, Apple> c = Apple::new;
System.out.println(c.apply("dark", 30));
```

### 2.7 复合Lambda表达式
你可以把多个简单的Lambda复合成复杂的表达式，两个谓词之间可以 or 或 and

#### 2.7.1 比较器复合
按 `weight` 逆序排序，如果相等时比较 `country`
```java
apples.sort(
        Comparator
                .comparing(Apple::getWeight)
                .reversed()
                .thenComparing(Apple::getCountry)
);
```

#### 2.7.2 谓词复合
```java
//红色苹果
Predicate<Apple> redApple = apple -> Objects.equals("red", apple.getColor());
//非红色苹果
Predicate<Apple> notRedApple = redApple.negate();
//非红色且重的苹果
Predicate<Apple> notReadAndHeavy = notRedApple
        .and(apple -> apple.weight > 30);
```

#### 2.7.3 函数复合
```java
Function<Integer, Integer> f = x -> x + 1;
Function<Integer, Integer> g = x -> x * 2;
// (10+1) * 2 g(f(x))
int result = f.andThen(g).apply(10);
System.out.println("result = " + result);

// 10*2 + 1 f(g(x))
result = f.compose(g).apply(10);
System.out.println("result = " + result);
```