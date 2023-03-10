---
layout: post
title: 字符编码
categories: java
excerpt: 字符编码，Unicode，UTF-8，UTF-16，数据加密，数据压缩
tags: Code ByteBuffer NIO
date: 2023-03-10
---
Unicode 官网PDF地址 [https://www.unicode.org/charts/](https://www.unicode.org/charts/)

摘抄自 `Unicode` 官网
>High Surrogate Area<br/>
Range: D800-DBFF<br/>
Isolated surrogate code points have no interpretation; consequently, no character code charts or names lists are provided for
this range.<br/>

>Low Surrogate Area<br/>
Range: DC00-DFFF<br/>
Isolated surrogate code points have no interpretation; consequently, no character code charts or names lists are provided for
this range.

Java 平台是采用 `UTF-16` ，下文摘抄自 `java.lang.Character`
>The set of characters from U+0000 to U+FFFF is sometimes referred to as the Basic Multilingual Plane (BMP). Characters whose code points are greater than U+FFFF are called supplementary characters. `The Java platform uses the UTF-16 representation in char arrays and in the String and StringBuffer classes.` In this representation, supplementary characters are represented as a pair of char values, the first from the high-surrogates range, (\uD800-\uDBFF), the second from the low-surrogates range (\uDC00-\uDFFF).

### 1.缩写展开
1. `Universal Multiple-Octet Coded Character Set` 简称 `UCS` 俗称 `Unicode`
2. `ASCII(American Standard Code for Information Interchange)`
3. `UTF（UCS Transfer Format）`

### 2.简单描述
字符在计算机中最终是以二进制的方式存储的，编解码时字符集要一致，否则会出现乱码。常见的字符集有 ASCII，GBK，Unicode等。
* 一个 `ASCII(American Standard Code for Information Interchange)` 字符占用一个字节，共有 128 个字符，字节最高位为 0，缺点是不能用于非英语国家字符编码。

* GBK 即汉字内码扩展规范，用于汉字编码，长度为两个字节。

* 早期每个不同语种的国家基本上都有自己的一套编码，这样就会产生乱码问题。为了这个问题 Unicode 应运而生，Unicode 将所有语言都统一到一套编码中，这样就不会产生
  乱码问题。 

* Unicode 只是字符集，UTF-8、UTF-16、UTF-32 是真正的字符编码规则。

* UTF-8 变长 `1 byte` `2 bytes` `3 bytes` `4 bytes`

* UTF-16 变长，为 `2 bytes` 或 `4 bytes` 

* UTF-32 定长 `4 bytes`

摘抄自 `The Go Program Language`
>Long ago, life was simple and there was, at least in a parochial view, only one character set to
deal with: ASCII, the American Standard Code for Information Interchange . ASCII, or more
precisely US-ASCII, uses 7bits to represent 128 ‘‘characters’’: the upper-and lower-case letters
of English, digits, and a variety of punctuation and device-control characters. `For much of the
early days of computing , this was adequate, but it left a very large fraction of the world’s
population unable to use their own writing systems in computers.` With the growth of the
Internet, data in myriad languages has become much more common. How can this rich variety be dealt with at all and, if possible, efficiently?

>The answer is `Unicode `(unicode.org), which collects `all of the characters` in all of the world’s
writing systems, plus accents and other diacritical marks, control codes like tab and carriage
return, and plenty of esoterica, and assigns each one a standard number called a Unicode code
point or, in Go terminology, a rune.

>Unicode version8 defines code points for over 120,000 characters in well over 100 languages
and scripts. How are these represented in computer programs and data? `The natural data
type to hold a single rune is int32`, and that’s what Go uses; it has the synonym rune for
precisely this purpose.

>`We could represent a sequence of runes as a sequence of int32 values. In this representation,
which is called UTF-32 or UCS-4, the encoding of each Unicode code point has the same size,
32 bits. This is simple and uniform, but it uses much more space than necessary since most
computer-readable text is in ASCII, which requires only 8bits or 1 byte per character. All the
characters in widespread use still number fewer than 65,536, which would fit in 16 bits. Can
we do better?` 

>UTF-8 is `avariable-length` encoding of Unicode code points as bytes. UTF-8 was invented by
Ken Thompson and Rob Pike, two of the creators of Go, and is now `a Unicode standard` . It
uses between 1 and 4 bytes to represent each rune, but only `1 byte for ASCII characters`, and
`only 2 or 3 bytes for most runes` in common use. `The high-order bits of the first byte of the
encoding for a rune indicate how many bytes follow. A high-order 0 indicates 7-bit ASCII,
where each rune takes only 1 byte, so it is identical to conventional ASCII. A high-order 110
indicates that the rune takes 2 bytes; the second byte begins with 10.` Larger runes have analogous encodings.

>`0xxxxxxx runes 0−127 (ASCII)`<br/>
`110xxxxx 10xxxxxx 128−2047 (values <128 unused)`<br/>
`1110xxxx 10xxxxxx 10xxxxxx 2048−65535 (values <2048 unused)`<br/>
`11110xxx 10xxxxxx 10xxxxxx 10xxxxxx 65536−0x10ffff (other values unused)`


### 3.编码规则
#### UTF-8 编码规则
解析时就可按照首字节来判断字符所占字节数
<table border="1" style="border-collapse:collapse">
  <tr><th>Unicode</th><th>UTF-8</th></tr>
  <tr><td>0x00-0x7F</td><td>0xxxxxxx</td></tr>
  <tr><td>0x80-0x7FF</td><td>110xxxxx 10xxxxxx</td></tr>
  <tr><td>0x800-0xFFFF</td><td>1110xxxx 10xxxxxx 10xxxxxx </td></tr>
  <tr><td>0x10000-0x10FFFF</td><td>11110xxx 10xxxxxx 10xxxxxx 10xxxxxx</td></tr>
</table>

#### UTF-16 编码规则
<table border="1" style="border-collapse:collapse">
  <tr><th>Unicode</th><th>UTF-16</th></tr>
  <tr><td>0x0000-0xFFFF</td><td>xxxxxxxx xxxxxxxx</td></tr>
  <tr><td>0x10000-0x10FFFF</td><td>110110xx xxxxxxxx 110111xx xxxxxxxx</td></tr>
</table>
UTF-16 是如何判断字符是占用 `2 bytes` or `4 bytes`？

由于 `Unicode` 规定 `D800-DBFF` 为保留区段（见本文开头摘抄），所以可以据此来判断。参见 `java.lang.Character`
```java

public static final char MIN_HIGH_SURROGATE = '\uD800';

public static final char MAX_HIGH_SURROGATE = '\uDBFF';

public static final char MIN_LOW_SURROGATE  = '\uDC00';

public static final char MAX_LOW_SURROGATE  = '\uDFFF';

public static final int MIN_SUPPLEMENTARY_CODE_POINT = 0x010000;

/*计算 Unicode 码点*/
public static int toCodePoint(char high, char low) {
    return ((high << 10) + low) + (MIN_SUPPLEMENTARY_CODE_POINT
                                    - (MIN_HIGH_SURROGATE << 10)
                                    - MIN_LOW_SURROGATE);
}

public static int codePointAt(CharSequence seq, int index) {
    char c1 = seq.charAt(index);
    if (isHighSurrogate(c1) && ++index < seq.length()) {
        char c2 = seq.charAt(index);
        if (isLowSurrogate(c2)) {
            return toCodePoint(c1, c2);
        }
    }
    return c1;
}

public static boolean isHighSurrogate(char ch) {
    return ch >= MIN_HIGH_SURROGATE && ch < (MAX_HIGH_SURROGATE + 1);
}

public static boolean isLowSurrogate(char ch) {
    return ch >= MIN_LOW_SURROGATE && ch < (MAX_LOW_SURROGATE + 1);
}
```

#### UTF-32 编码规则
<table border="1" style="border-collapse:collapse">
  <tr><th>Unicode</th><th>UTF-32</th></tr>
  <tr><td>0x000000-0x10FFFF</td><td>xxxxxxxx xxxxxxxx xxxxxxxx xxxxxxxx</td></tr>
</table>

### 4.小实验
```java
@Test
public void testRuneLength() {
    String character1 = "A";
    String character2 = "中";
    String character3 = "😄";

    printCharacterLengthInfo(character1);//chars 1, bytes: 1
    printCharacterLengthInfo(character2);//chars 1, bytes: 3
    printCharacterLengthInfo(character3);//chars 2, bytes: 4

    String content = "今天天气真不错😄Nice";
    printRuneLength(content.getBytes(StandardCharsets.UTF_8));//字节数组中共含 12 个 UTF8 字符
    System.out.println("content 中字符长度为 "+ content.length());//content 中字符长度为 13
}

public void printCharacterLengthInfo(String str) {
    System.out.println("chars " + str.toCharArray().length + ", bytes: " + str.getBytes(StandardCharsets.UTF_8).length);
}

/**
  * 依据字节流，判断字节流中字符个数
  * 
  * @param bytes
  */
public void printRuneLength(byte[] bytes) {
    int count = 0;
    for (int i = 0; i < bytes.length; ) {
        count++;
        if (((bytes[i] & 0XFF) >>> 7) == 0) {
            i += 1;
        } else if (((bytes[i] & 0XFF) >>> 5) == 0b110) {
            i += 2;
        } else if (((bytes[i] & 0xFF) >>> 4) == 0b1110) {
            i += 3;
        } else if (((bytes[i] & 0xFF) >>> 3) == 0b11110) {
            i += 4;
        } else {
            throw new RuntimeException("编码错误");
        }
    }
    System.out.println("字节数组中共含 " + count + " 个 UTF8 字符");
}
```

### 5.自定义编解码
将字符串编码成 ,xxxx形式，再解码出原内容
```java
@Test
public void testStringEncodeDiy() {
    final String l = stringToDiy("中华😄aaabb,圣爱大厦", "%");
    System.out.println(l);
    //out: %e4b8ad%e58d8e%f09f9884%61%61%61%62%62%2c%e59ca3%e788b1%e5a4a7%e58ea6
    final String diyDecode = diyDecode(l, "%");
    System.out.println(diyDecode);
    //out: 中华😄aaabb,圣爱大厦
}

public void appendCh(byte[] bytes, int start, int end, StringBuilder out) {
    for (int i = start; i < end; i++) {
        char ch = Character.forDigit(bytes[i] >> 4 & 0xF, 16);
        out.append(ch);
        ch = Character.forDigit(bytes[i] & 0xF, 16);
        out.append(ch);
    }

}

public String diyDecode(String diyEncodedStr, String separator) {
    final String[] runes = diyEncodedStr.split(separator);
    ByteArrayOutputStream out = new ByteArrayOutputStream();
    for (String rune : runes) {
        for (int i = 0; i < rune.length(); ) {
            final byte b = (byte) (Integer.parseInt(rune.substring(i, i + 2), 16) & 0xFF);
            out.write(b);
            i += 2;
        }
    }
    return out.toString(StandardCharsets.UTF_8);
}

public String stringToDiy(String str, String separator) {
    final byte[] bytes = str.getBytes(StandardCharsets.UTF_8);
    StringBuilder out = new StringBuilder();
    for (int i = 0; i < bytes.length; ) {
        out.append(separator);
        if (((bytes[i] & 0XFF) >>> 7) == 0) {
            appendCh(bytes, i, i + 1, out);
            i += 1;
        } else if (((bytes[i] & 0XFF) >>> 5) == 0b110) {
            appendCh(bytes, i, i + 2, out);
            i += 2;
        } else if (((bytes[i] & 0xFF) >>> 4) == 0b1110) {
            appendCh(bytes, i, i + 3, out);
            i += 3;
        } else if (((bytes[i] & 0xFF) >>> 3) == 0b11110) {
            appendCh(bytes, i, i + 4, out);
            i += 4;
        } else {
            throw new RuntimeException("编码错误");
        }
    }
    return out.toString();
}
```
### 6.小进阶

* 把我们所需要传输的数据以特定的格式（有一定的加密效果）生成，再以特定的格式解析。

```java
public class DiyData {

    public static void main(String[] args) throws UnsupportedEncodingException, IOException {
        TransData transData = new TransData();
        byte[] bytes = transData
                .writeInt(165)
                .writeDouble(165.2)
                .writeString("hello world!")
                .writeIntArray(new int[]{85, 89, 56})
                .writeStringArray(new String[]{"zhongguo", "nanjing", "Java"})
                .getData();

        ByteBuffer byteBuffer = ByteBuffer.wrap(bytes);

        while (byteBuffer.hasRemaining()) {
            switch (byteBuffer.get()) {
                case TransData.TYPE_INT:
                    System.out.println(byteBuffer.getInt());
                    break;
                case TransData.TYPE_DOUBLE:
                    System.out.println(byteBuffer.getDouble());
                    break;
                case TransData.TYPE_STRING:
                    int length = byteBuffer.getInt();
                    System.out.println(new String(byteBuffer.array(), byteBuffer.position(), length, "UTF-8"));
                    byteBuffer.position(byteBuffer.position() + length);
                    break;
                case TransData.TYPE_ARRAY_INT:
                    int len = byteBuffer.getInt();
                    int[] results = new int[len];
                    for (int i = 0; i < len; i++) {
                        results[i] = byteBuffer.getInt();
                    }
                    System.out.println(Arrays.toString(results));
                    break;
                case TransData.TYPE_ARRAY_STRING:
                    int stringsLength = byteBuffer.getInt();
                    String[] strings = new String[stringsLength];
                    for (int i = 0; i < stringsLength; i++) {
                        int stringLen = byteBuffer.getInt();
                        strings[i] = new String(byteBuffer.array(), byteBuffer.position(), stringLen, "UTF-8");
                        byteBuffer.position(byteBuffer.position() + stringLen);
                    }
                    System.out.println(Arrays.toString(strings));
                    break;
            }
        }

    }

    /**
     * |type|length|content| type 1 byte length 4 byte 基本类型不需要 length
     *
     * @author zhaoxiaosi
     */
    static class TransData {
        private static final byte TYPE_INT = 1;
        private static final byte TYPE_DOUBLE = 2;
        private static final byte TYPE_STRING = 3;

        private static final byte TYPE_ARRAY_STRING = 4;
        private static final byte TYPE_ARRAY_INT = 5;
        private static final byte TYPE_ARRAY_DOUBLE = 6;

        private ByteArrayOutputStream out = new ByteArrayOutputStream();

        public TransData writeInt(int content) {
            out.write(TYPE_INT);
            out.write(content >>> 24);
            out.write(content >>> 16);
            out.write(content >>> 8);
            out.write(content);
            return this;
        }

        public TransData writeIntArray(int[] ints) {
            out.write(TYPE_ARRAY_INT);
            writeLength(ints.length);
            for (int i : ints) {
                out.write(i >>> 24);
                out.write(i >>> 16);
                out.write(i >>> 8);
                out.write(i);
            }
            return this;
        }

        public TransData writeStringArray(String[] strings) throws IOException {
            out.write(TYPE_ARRAY_STRING);
            writeLength(strings.length);
            for (String string : strings) {
                byte[] bytes = string.getBytes("UTF-8");
                writeLength(bytes.length);
                out.write(bytes);
            }
            return this;
        }

        public TransData writeDouble(double content) {
            long value = Double.doubleToRawLongBits(content);
            out.write(TYPE_DOUBLE);
            out.write((byte) (int) value >>> 56);
            out.write((byte) (int) value >>> 48);
            out.write((byte) (int) value >>> 40);
            out.write((byte) (int) value >>> 32);
            out.write((byte) (int) value >>> 24);
            out.write((byte) (int) value >>> 16);
            out.write((byte) (int) value >>> 8);
            out.write((byte) (int) value);
            return this;
        }


        public TransData writeString(String content) throws UnsupportedEncodingException, IOException {
            out.write(TYPE_STRING);
            byte[] bytes = content.getBytes("UTF-8");
            writeLength(bytes.length);
            out.write(bytes);
            return this;
        }

        private void writeLength(int length) {
            out.write(length >>> 24);
            out.write(length >>> 16);
            out.write(length >>> 8);
            out.write(length);
        }

        public byte[] getData() {
            return out.toByteArray();
        }

    }
}
```



