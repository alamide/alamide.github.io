---
layout: post
title: 字符编码
categories: java
excerpt: 字符编码，Unicode，UTF-8，UTF-16
tags: code
---
字符在计算机中最终是以二进制的方式存储的，编解码时字符集要一致，否则会出现乱码。常见的字符集有 ASCII，GBK，Unicode等。
* 一个 ASCII 字符占用一个字节，共有 128 个字符，字节最高位为 0，缺点是不能用于非英语国家字符编码
* GBK 即汉字内码扩展规范，用于汉字编码，长度为两个字节
* 早期每个不同语种的国家基本上都有自己的一套编码，这样就会产生乱码问题。为了这个问题 Unicode 应运而生，Unicode 将所有语言都统一到一套编码中，这样就不会产生
  乱码问题。 

摘抄自 `The Go Program Language`
>Long ago, life was simple and there was, at least in a parochial view, only one character set to
deal with: ASCII, the American Stand ard Code for Information Interchange . ASCII, or more
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

>`We could represent a sequence of runes as a sequence of int32 values. In this represent ation,
which is calle d UTF-32 or UCS-4, the encoding of each Unicode code point has the same size,
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

`0xxxxxxx runes 0−127 (ASCII)`

`110xxxxx 10xxxxxx 128−2047 (values <128 unused)`

`1110xxxx 10xxxxxx 10xxxxxx 2048−65535 (values <2048 unused)`

`11110xxx 10xxxxxx 10xxxxxx 10xxxxxx 65536−0x10ffff (other values unused)`

### 小实验
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

* Java 中 中文采用3个 `byte` 存储一个汉字，1个 `byte` 存储 `ascii` 字符，4个 `byte` 存储 `emoji`
* 一个 `char` 表示一个中文，一个 `char` 表示一个 `ascaii`，两个 `char` 表示一个 `emoji`

### 自定义编解码
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
### 小进阶

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



