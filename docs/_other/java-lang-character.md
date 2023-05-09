---
layout: post
title: JDK java.lang.Character
categories: java jdk
excerpt: JDK Character 类库方法使用
tags: java jdk
date: 2023-03-08
---

## 1.获取一个字符的 Unicode 编码
`public static int codePointAt(CharSequence seq, int index)`
```java
@Test
public void testCharacter(){
    final int point = Character.codePointAt("😄", 0);
    System.out.println(point);//128516
    final char[] chars = Character.toChars(point);
    System.out.println(chars.length);//2
    System.out.println(new String(chars));//😄
}
```
## 2.由 Unicode 编码，查看 占用 `char` 数
```java
final int charCount = Character.charCount(128516);
System.out.println(charCount);//2
```

```java
public static final int MIN_SUPPLEMENTARY_CODE_POINT = 0x010000;

public static int charCount(int codePoint) {
  return codePoint >= MIN_SUPPLEMENTARY_CODE_POINT ? 2 : 1;
}
```

## 3.判断是否为空
空白字符包括 '\n' 、' ' '\t' 等
```java
Character.isWhitespace(c);
```

## 4.是否为字母
```java
Character.isAlphabetic(a);
```