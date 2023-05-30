---
layout: post
title:  LeetCode1768 合并后的字符串
date:   2023-05-30
categories: LeetCode
tags: LeetCode 
---
给你两个字符串 word1 和 word2 。请你从 word1 开始，通过交替添加字母来合并字符串。如果一个字符串比另一个字符串长，就将多出来的字母追加到合并后字符串的末尾。返回合并后的字符串。
<!--more-->

初始版本，写的有点复杂了，有点轴，不灵活，还是要多练。
```java
public String mergeAlternately(String word1, String word2) {
    char[] arr1 = word1.toCharArray();
    char[] arr2 = word2.toCharArray();
    char[] resArr = new char[arr2.length + arr1.length];

    for (int i = 0; i < arr1.length && i < arr2.length; i++) {
        resArr[i * 2] = arr1[i];
        resArr[i * 2 + 1] = arr2[i];
    }

    if (arr1.length < arr2.length) {
        System.arraycopy(arr2, arr1.length, resArr, 2 * arr1.length, arr2.length - arr1.length);
    } else {
        System.arraycopy(arr1, arr2.length, resArr, 2 * arr2.length, arr1.length - arr2.length);
    }

    return new String(resArr);
}
```

优化后版本，代码更简洁，性能的话，前面的那种解法性能可能会更优，尤其当两个字符串长度相差较大时，因为采用的是整块复制。
```java
public String mergeAlternately2(String word1, String word2) {
    char[] arr1 = word1.toCharArray();
    char[] arr2 = word2.toCharArray();
    char[] resArr = new char[arr2.length + arr1.length];

    int i = 0, j = 0, k = 0, n = arr1.length, m = arr2.length;
    while (i < n || j < m) {
        if (i < n) resArr[k++] = arr1[i++];
        if (j < m) resArr[k++] = arr2[j++];
    }

    return new String(resArr);
}
```