---
layout: post
title: Java 基础 DateTime
categories: java
tags: Java DateTime
date: 2023-04-16
---
Java 基础语法 日期和时间，学习内容来自 《Java 实战（第二版）》
<!--more-->

Java 基础之日期时间相关，Java1.0 使用的是 `java.util.Date` ，这个工具类有一些缺陷，无法表示日期，且只能以毫秒精度表示时间，
更糟糕的是它的易用性，由于某些原因未知的设计决策，这个类的易用性被深深地损害了，比如：年份的起始选择是1900年，月份的起始从0开始。
这个类会让人很困惑，Date 是表示日期的意思，是不连续的，但是这里却用来表示时间。

Java1.1 中，Date 类中很多方法被废弃了，取而代之的是 `java.util.Calendar` ，但是也有缺陷，很困惑到底使用哪一个。

Java8 之后提供了新的时间和日期 API，LocalDate、LocalTime、LocalDateTime、Instant、Duration、Period。

## 1.时间与日期
### 1.1 LocalDate
```java
final LocalDate localDate = LocalDate.of(2023, 4, 16);
final int dayOfWeek = localDate.getDayOfWeek().getValue();
final int dayOfYear = localDate.getDayOfYear();
final int dayOfMonth = localDate.getDayOfMonth();
final int monthValue = localDate.getMonthValue();
final int lengthOfMonth = localDate.lengthOfMonth();//获取这一月天数
final int lengthOfYear = localDate.lengthOfYear();//获取这一年天数
```

### 1.2 LocalTime
```java
LocalTime localTime = LocalTime.of(10, 0, 0);
```

### 1.3 LocalDateTime
```java
LocalTime localTime = LocalTime.now();
LocalDate localDate = LocalDate.now();
final LocalDateTime localDateTime = LocalDateTime.of(localDate, localTime);
final LocalDateTime localDateTime1 = LocalDateTime.of(2023, Month.APRIL, 16, 11, 15);
final LocalDateTime localDateTime2 = localDate.atTime(11, 15, 15);
final LocalDateTime localDateTime3 = localDate.atTime(localTime);
final LocalDateTime localDateTime4 = localTime.atDate(localDate);
```

### 1.4 Instant
```java
final Instant now = Instant.now();
final Instant instant1 = Instant.ofEpochSecond(3);
final Instant instant2 = Instant.ofEpochSecond(3, 0);
final Instant instant3 = Instant.ofEpochSecond(2, 1_000_000_000);
final Instant instant4 = Instant.ofEpochSecond(4, -1_000_000_000);
```

## 2.时间和日期间隔
### 2.1 Duration
Duration类主要用于以秒和纳秒衡量时间的长短，所以不能仅向between方法传递一个LocalDate对象做参数
```java
final Instant instant1 = Instant.now();
final Instant instant2 = Instant.now();
final Duration duration = Duration.between(instant1, instant2);
final long nanos = duration.get(ChronoUnit.NANOS);

final LocalDateTime localDateTime1 = LocalDateTime.of(2023, 4, 15, 11,11,11);
final LocalDateTime localDateTime2 = LocalDateTime.of(2023, 4, 16,11,11);
final Duration between = Duration.between(localDateTime1, localDateTime2);
final long seconds = between.get(ChronoUnit.SECONDS);
```

### 2.2 Period
以年、月或者日的方式对多个时间单位建模，那么可以使用Period类
```java
final LocalDate localDate1 = LocalDate.of(2023, 3, 3);
final LocalDate localDate2 = LocalDate.of(2024, 4, 10);
final Period period = Period.between(localDate1, localDate2);
```

## 3.操作、解析和格式化日期
### 3.1 修改属性
修改属性会创建一个新的对象，并不会修改原来的对象
```java
final LocalDate localDate = startDate.withYear(2012);
final LocalDate localDate1 = localDate.with(ChronoField.MONTH_OF_YEAR, 2);
```

### 3.2 TemporalAdjuster
```java
//下一个周日
final LocalDate nextSunDay = LocalDate.now().with(TemporalAdjusters.nextOrSame(DayOfWeek.SUNDAY));
//年末
final LocalDate lastDayOfYear = LocalDate.now().with(TemporalAdjusters.lastDayOfYear());

//下一个工作日
final LocalDate nextWorkDate = LocalDate.now().with(temporal -> {
    final DayOfWeek dayOfWeek = DayOfWeek.of(temporal.get(ChronoField.DAY_OF_WEEK));
    int dayAdd = 1;
    if (dayOfWeek == DayOfWeek.FRIDAY) {
        dayAdd = 3;
    } else if (dayOfWeek == DayOfWeek.SATURDAY) {
        dayAdd = 2;
    }
    return temporal.plus(dayAdd, ChronoUnit.DAYS);
});
```

### 3.3 格式化
```java
//20230416
final String format1 = LocalDate.now().format(DateTimeFormatter.BASIC_ISO_DATE);
//2023-04-16
final String format2 = LocalDate.now().format(DateTimeFormatter.ISO_LOCAL_DATE);
//2023-04-16
final LocalDate parseDate = LocalDate.parse("20230416", DateTimeFormatter.BASIC_ISO_DATE);

final DateTimeFormatter dateTimeFormatter = DateTimeFormatter.ofPattern("dd/MM/yyyy");
//16/04/2023
final String format3 = LocalDate.now().format(dateTimeFormatter);

final DateTimeFormatter dateTimeFormatter1 = new DateTimeFormatterBuilder()
        .appendText(ChronoField.MONTH_OF_YEAR)
        .appendLiteral("%")
        .appendText(ChronoField.DAY_OF_MONTH)
        .appendLiteral("*")
        .appendText(ChronoField.YEAR)
        .parseCaseInsensitive()
        .toFormatter(Locale.CHINESE);
//四月%16*2023
final String format4 = LocalDate.now().format(dateTimeFormatter1);
```

### 3.4 时区
```java
final Instant instant1 = Instant.now();
System.out.println(instant1);
final ZonedDateTime zonedDateTime = instant1.atZone(ZoneId.of("Asia/Shanghai"));
System.out.println(zonedDateTime);

final Instant instant = LocalDateTime.now().toInstant(ZoneOffset.UTC);
final LocalDateTime localDateTime = LocalDateTime.ofInstant(Instant.now(), ZoneId.of("America/Anchorage"));
final LocalDateTime localDateTime1 = LocalDateTime.ofInstant(Instant.now(), ZoneId.of("UTC+7"));
final LocalDateTime localDateTime2 = LocalDateTime.ofInstant(Instant.now(), ZoneId.systemDefault());
```

## 4.Snippets
计算预产期及孕期信息
```java
final LocalDate startDate = LocalDate.of(2023, 1, 16);
final LocalDate goalDate = startDate.plusDays(280);
final LocalDate now = LocalDate.now();
final long weeks = ChronoUnit.WEEKS.between(startDate, now);
final long days = ChronoUnit.DAYS.between(startDate.plusWeeks(weeks), now);

log.info("预产期为：{}, 当前为：孕 {} 周 + {} 天", goalDate, weeks, days);
```

新年倒计时
```java
final LocalDateTime goalDate = LocalDate.now().plusYears(1).withDayOfYear(1).atStartOfDay();
        
Timer timer = new Timer();
timer.scheduleAtFixedRate(new TimerTask() {
    @Override
    public void run() {
        final long gapSeconds = ChronoUnit.SECONDS.between(LocalDateTime.now(), goalDate);
        final long days = Duration.ofSeconds(gapSeconds).toDaysPart();
        final long hours = Duration.ofSeconds(gapSeconds).toHoursPart();
        final long minutes = Duration.ofSeconds(gapSeconds).toMinutesPart();
        final long seconds = Duration.ofSeconds(gapSeconds).toSecondsPart();

        log.info("距离新的一年还剩：{} 天 {} 时 {} 分 {} 秒", String.format("%03d", days), String.format("%02d", hours), String.format("%02d", minutes), String.format("%02d", seconds));
    }
}, 0, 1000);

System.in.read();
```
