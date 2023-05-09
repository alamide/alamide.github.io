---
layout: post
title: Java 基准测试 JMH
categories: java
tags: JMH
date: 2023-05-07
---
JMH，全称 Java Microbenchmark Harness（微基准测试框架），是专门用于 Java 代码微基准测试的一套测试工具 API，是由 OpenJDK/Oracle 官方发布的工具。
<!--more-->

官方 [Samples](https://hg.openjdk.org/code-tools/jmh/file/2be2df7dbaf8/jmh-samples/src/main/java/org/openjdk/jmh/samples)

引入依赖
```xml
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-core</artifactId>
    <version>1.17.4</version>
</dependency>
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-generator-annprocess</artifactId>
    <version>1.17.4</version>
</dependency>
```

一些测试
```java
@State(Scope.Thread)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@Fork(value = 2, jvmArgs = {"-Xms4G", "-Xmx4G"})//4GB 的堆，执行两次
public class BoxAndUnBoxBenchMark {
    private static final int  N = 100_000_000;
    @Benchmark
    public long primitive(){
        long sum = 0;
        for (int i=0; i< N; i++){
            sum += i;
        }
        return sum;
    }

    @Benchmark
    public Long box(){
        Long sum = 0L;
        for (int i=0; i< N; i++){
            sum += i;
        }
        return sum;
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(BoxAndUnBoxBenchMark.class.getSimpleName())
                .warmupIterations(5)
                .measurementIterations(10)
                .build();

        new Runner(opt).run();
    }
}
```

output:
```
Benchmark                       Mode  Cnt    Score    Error  Units
BoxAndUnBoxBenchMark.box        avgt   20  300.532 ± 30.118  ms/op
BoxAndUnBoxBenchMark.primitive  avgt   20   35.137 ±  2.719  ms/op
```

可以看到装箱箱操作比较耗费资源，计算过程中应避免装箱

测试字符串拼接
```java
@State(Scope.Thread)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@Fork(value = 2, jvmArgs = {"-Xms4G", "-Xmx4G"})
public class StringAppendBenchMark {
    private static final int N = 10_0000;
    @Benchmark
    public String overload(){
        String result = "";
        for (int i=0; i<N; i++){
            result += i;
        }
        return result;
    }
    @Benchmark
    public String stringBuilder(){
        StringBuilder result = new StringBuilder();
        for (int i=0; i<N; i++){
            result.append(i);
        }
        return result.toString();
    }

    @Benchmark
    public String stringBuffer(){
        StringBuffer result = new StringBuffer();
        for (int i=0; i<N; i++){
            result.append(i);
        }
        return result.toString();
    }

    @TearDown(Level.Invocation)
    public void tearDown(){
        System.gc();
    }
    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(StringAppendBenchMark.class.getSimpleName())
                .warmupIterations(5)
                .measurementIterations(10)
                .build();

        new Runner(opt).run();
    }
}
```

output:
```
Benchmark                            Mode  Cnt     Score    Error  Units
StringAppendBenchMark.overload       avgt   20  2464.098 ± 55.441  ms/op
StringAppendBenchMark.stringBuffer   avgt   20     1.699 ±  0.054  ms/op
StringAppendBenchMark.stringBuilder  avgt   20     1.682 ±  0.045  ms/op
```

大量字符串拼接时不要使用 += 重载符，太消耗资源