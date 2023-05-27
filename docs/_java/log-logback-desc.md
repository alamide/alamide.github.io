---
layout: post
title: Java 日志框架 简介
categories: java log
tags: java log slfj
date: 2023-04-03
excerpt: 日志系统在软件程序中占有非常重要的地位，日志文件是排查程序问题的主要工具，是程序调试的利器。 
---
* `SLF4J` 官方文档 [在这里](https://www.slf4j.org/index.html)

* `Logback` 官方文档 [在这里](https://logback.qos.ch/index.html)

`Logback` 是 `SLF4J` 的一个具体实现
>The Simple Logging Facade for Java (SLF4J) serves as a simple facade or abstraction for various logging frameworks, such as java.util.logging, log4j 1.x, reload4j and logback. 

## 1.What is logback?
Logback is intended as a successor to the popular log4j project. It was designed by Ceki Gülcü, log4j's founder. 

别人问你为什么要选择 `logback` ，而不是 `log4j` 。这个回答就足够了。它是 `log4j` 的继任者，`log4j` 的创造者说的。

## 2.简单使用
### 2.1 引入 logback-classic
```xml
<dependency> 
  <groupId>ch.qos.logback</groupId>
  <artifactId>logback-classic</artifactId>
  <version>1.3.6</version>
</dependency>
```
会自动引入 `logback-core` 及 `slf4j-api` ，当然我们也可以显示引入这两个 `jar` 包
>Note that explicitly declaring a dependency on logback-core-1.4.6 or slf4j-api-2.0.7.jar is not wrong and may be necessary to impose the correct version of said artifacts by virtue of Maven's "nearest definition" dependency mediation rule.

### 2.2 打印日志
```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

public class LogbackTest {
    private static Logger logger = LoggerFactory.getLogger(LogbackTest.class);
    @Test
    public void testSimple() {
        logger.info("这是一条测试信息");
    }
}
```

可以看到，没有使用到 `logback` 中的具体实现类，只是使用到 `slf4j` 中定义的接口，接口和具体实现分离，随时可以替换为 `log4j` ，而不用修改代码，多态的威力。

>Launching the application will output a single line on the console. By virtue of logback's default configuration policy, when no default configuration file is found, logback will add a `ConsoleAppender` to the root logger.

在 `logback` 发生问题的时候，打印内部状态会很有用
>Note that in the above example we have instructed logback to print its internal state by invoking the StatusPrinter.print() method. Logback's internal status information can be very useful in diagnosing logback-related problems.

```java
@Test
public void testSimple() {
    logger.info("这是一条测试信息");
    LoggerContext lc = (LoggerContext) LoggerFactory.getILoggerFactory();
    StatusPrinter.print(lc);
}
```

output:
```
17:21:48.439 [main] INFO com.alamide.third.LogbackTest -- 这是一条测试信息
17:21:48,101 |-INFO in ch.qos.logback.classic.LoggerContext[default] - This is logback-classic version 1.3.6
17:21:48,128 |-INFO in ch.qos.logback.classic.LoggerContext[default] - Could NOT find resource [logback-test.xml]
17:21:48,128 |-INFO in ch.qos.logback.classic.LoggerContext[default] - Could NOT find resource [logback.xml]
17:21:48,130 |-INFO in ch.qos.logback.classic.BasicConfigurator@7fc2413d - Setting up default configuration.
```

## 3.架构
`logback` 分为三个模块，`logback-core` 、`logback-classic` 、`logback-access` ，其中 `logback-core` 是其它两个模块的基础。
我们一般只会用到 `logback-classic` ，`logback-access` 用在 `Servlet containers` 。
>The third module called access integrates with Servlet containers to provide HTTP-access log functionality.

`Logback` 主要的三个类是 `Logger` 、`Appender` 、`Layout` ，三个类各司其职。

`Logger` 是 `logback-classic` 中类，`Appender` 、`Layout` 是 `logback-core` 中的类。

### 3.1 Logger
#### 3.1.1 Logger 的继承机制
`Logger` 的名字是有继承机制的，如 `"com.foo" is a parent of the logger named "com.foo.Bar"`

>Every single logger is attached to a LoggerContext which is responsible for manufacturing loggers as well as arranging them in a tree like hierarchy.

>`Loggers are named entities. Their names are case-sensitive and they follow the hierarchical naming rule:`

>A logger is said to be an ancestor of another logger if its name followed by a dot is a prefix of the descendant logger name. A logger is said to be a parent of a child logger if there are no ancestors between itself and the descendant logger.

>For example, the logger named "com.foo" is a parent of the logger named "com.foo.Bar". Similarly, "java" is a parent of "java.util" and an ancestor of "java.util.Vector". This naming scheme should be familiar to most developers.

那么为什么要引入继承机制呢？可以更好的控制输出。看下面这个例子：
```java
@Test
public void testHierarchy(){
    ch.qos.logback.classic.Logger loggerParent = (ch.qos.logback.classic.Logger) LoggerFactory.getLogger("com.alamide");
    loggerParent.setLevel(Level.INFO);

    //继承先代的级别为 info 级别
    Logger loggerChild = LoggerFactory.getLogger("com.alamide.third.LogbackTest");

    loggerParent.info("parent info level");
    //没有输出 debug < info
    loggerChild.debug("child debug level");
}
```

output:
```
18:01:22.011 [main] INFO com.alamide -- parent info level
```

可以看到，只会打印 `parent` 的 `info` ，而不会打印 `child` 的 `debug` 。依据 `Logger` 的继承机制，`loggerChild` 是 `loggerParent` 后代，会继承打印级别，即只会输出 `info level` 之上的信息。 

当然，`loggerChild` 也可以设置自己的级别。

`logger` 如果未设置自己的日志级别，则会继承最相邻先代的级别。

`root logger` 的默认级别为 `DEBUG` 。


#### 3.1.2 Logger Level
日志的级别控制着输出，哪可以输出，哪些不可以输出
* 级别顺序如下：`TRACE < DEBUG < INFO <  WARN < ERROR`

* 规则如下 `A log request of level p issued to a logger having an effective level q, is enabled if p >= q.`

也就是说，只有更高级别的信息才会被输出，这也解释了为什么上面 `debug` 无法输出的原因（level debug < level info）。

#### 3.1.3 Retrieving Loggers
Calling the LoggerFactory.getLogger method with the same name will always return a reference to the exact same logger object.

### 3.2 Appender
`Appender` 控制日志输出到哪里？可以是控制台、文件、数据库等等。
>In logback speak, an output destination is called an appender.

一个 `Logger` 可以有多个 `Appender` ，`Appender` 同样可以被继承，例如我们上面的 `Logger` 并没有指定 `Appender` ，它会继承 `Root` 的 `console appender` 。

`Appender Additivity` 
>The output of a log statement of logger L will go to all the appenders in L and its ancestors. This is the meaning of the term "appender additivity".

>However, if an ancestor of logger L, say P, has the additivity flag set to false, then L's output will be directed to all the appenders in L and its ancestors up to and including P but not the appenders in any of the ancestors of P.

>Loggers have their additivity flag set to true by default.

<table>
  <tr><th>Logger Name</th><th>Attached Appenders</th><th>Additivity Flag</th><th>Output Targets</th></tr>
  <tr><td>root</td><td>A1</td><td>not applicable</td><td>A1</td></tr>
  <tr><td>x</td><td>A-x1, A-x2</td><td>true</td><td>A1, A-x1, A-x2</td></tr>
  <tr><td>x.y</td><td>none</td><td>true</td><td>A1, A-x1, A-x2</td></tr>
  <tr><td>x.y.z</td><td>A-xyz1</td><td>true</td><td>A1, A-x1, A-x2, A-xyz1</td></tr>
  <tr><td>security</td><td>A-sec</td><td>false</td><td>A-sec</td></tr>
  <tr><td>security.access</td><td>none</td><td>true</td><td>A-sec</td></tr>
</table>

### 3.3 Layout
`Layout` 负责输出的格式
>The layout is responsible for formatting the logging request according to the user's wishes, whereas an appender takes care of sending the formatted output to its destination. 

### 3.4 Parameterized logging
我们输出的内容大部分时候是动态的，需要拼接，你可能会这样写：
```java
logger.debug("Entry number: " + i + " is " + String.valueOf(entry[i]));
```
这样写，无论信息是否输出，都会有 `int` 转换成 `String` ，这在日志不输出时，会造成一定的资源浪费，可以如下改进

```java
if(logger.isDebugEnabled()) { 
  logger.debug("Entry number: " + i + " is " + String.valueOf(entry[i]));
}
```

这里还有一些问题，输出日志的时候会做两次判断，一次是外层的 `logger.isDebugEnabled()` ，另一次是 `logger.debug` 内部，当然这点消耗几乎可以忽略不计。

更雅观的写法如下：
```java
logger.debug("Entry number: {} is {}", i, String.valueOf(entry[i]));
```
不用自己拼接，只会在日志输出时进行类型转换，且更清晰明了，推荐使用，因为我们日志输出是巨量的，可能已百万计、千万计，每一点点小的消耗，到最后都是巨大的
