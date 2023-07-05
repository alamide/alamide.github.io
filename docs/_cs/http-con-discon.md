---
layout: post
title: TCP 的连接与释放，及 HTTP 报文格式
categories: http
tags: http
date: 2023-07-05
---
Http 的连接与释放，报文格式，及一些常用的请求头。本次学习目标：

1. 为什么 TCP 连接需要三次握手

2. 为什么 TCP 释放连接需要四次挥手

3. HTTP 报文协议的格式

4. HTTP 一些常用的请求头

5. 使用 Socket 请求来完成一次网络请求

6. 复用 Socket，完成对同一个 HOST 多次请求

<!--more-->
## 1.为什么 TCP 连接需要三次握手
看了很多资料都是在画图、画箭头、发响应码，真的是难懂，最终还是找到了比较认可且易懂的回答，也有了自己的理解。一直将三次握手带入到 TCP、具体的细节上是错误的，应该脱离 TCP 来思考这个问题。

思考这样一个问题：确定一个信道可通信最少需要几次通信（A <----> B）？

第一次信息发送：A 端发送一条信息到 B 端，A 等待收到 B 的回复， A 不知道 B 是否能收到信息，所以需要等待回信确认

第二次信息发送：B 端收到信息，发送已收到给 A 端，等待 A 的回复，B 不知道 A 是否能收到信息，所以需要等待回信确认 

第三次信息发送：A 端收到 B 的信息，确认可以通信，A 准备通信，并给 B 端发送确认信息；B 端收到，确认可以信道可用，B 准备通信

三次握手是确认信道是否双向可用最少通信次数，注意这里是可用而不是可靠，因为不论多少次握手都不能保证信道可靠。

所以为什么 TCP 连接需要三次握手？因为这不是 TCP 本身的需求，而是确定一条信道双向可用的要求。

## 2.为什么 TCP 释放连接需要四次挥手
释放连接需要双方确认，具体流程如下：

第一次：客户端数据发送完毕，给服务器发信息，表明客户端数据已发送完毕，需要等待服务端收到信息的确认信息

第二次：服务端收到客户端发送的数据已发送完毕的消息，回复客户端，已确认收到信息

第三次：服务端数据发送完毕，给客户端发送信息，表明服务端数据已发送完毕，需要等待客户端确认收到消息的回复

第四次：客户端收到服务端数据发送完毕的信息，向服务端发信息表明已收到信息，客户端释放连接；服务端收到信息，也释放连接。至此，两端连接释放完毕

其中第二和第三次信息不能合并，因为信息的传输是双向的，服务端收到客户端数据发送完毕的时候，服务端向客户端发送的数据，可能还没有传输完毕

## 3.HTTP 报文协议的格式
HTTP 是一种传输协议，既然是协议，那么肯定需要遵循一定的规则。HTTP 报文为请求报文和响应报文，都遵循一定的规则。
### 3.1 请求报文
一个简单请求报文如下
```
GET /cs/http-con-discon.html HTTP/1.1
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7
Accept-Encoding: gzip, deflate, br
Accept-Language: zh-CN,zh;q=0.9,en;q=0.8
Cache-Control: no-cache
Connection: keep-alive
Host: 127.0.0.1:4000
Pragma: no-cache
Referer: http://127.0.0.1:4000/
```

可以将请求报文分为三个部分：请求行、请求头、请求体。下面就逐步拆解出这三个部分：
#### 3.1.1 请求行
请求行包含请求的方法、请求的 URI 和 HTTP 版本
```
GET /cs/http-con-discon.html HTTP/1.1
```

请求方法：GET 称为请求方法，除了 GET， 还有 POST、PUT、DELETE 等

请求 URI: /cs/http-con-discon.html 为请求 URI 表示请求资源的定位符

HTTP 版本：具体的 HTTP 协议版本


#### 3.1.2 请求头
请求条件和属性的各类首部，常用的请求头有：
1. HOST 表明请求的目标主机地址

2. Accept 表明希望服务器返回的数据类型，如 `application/xml` 表明希望服务端返回 xml 格式的数据

3. Content-Type 表明请求体的数据类型，比如 `application/json` 表明请求体中携带的数据类型为 json

4. Connection 表明希望维持长连接，即双方通信完毕后不立即释放连接

5. User-Agent 简称 UA，它是一个特殊字符串头，使得服务器能够识别客户使用的操作系统及版本、CPU 类型、浏览器及版本、浏览器渲染引擎、浏览器语言、浏览器插件等。在 UI 适配时很有用

6. Referer 表明访问来源信息

#### 3.1.3 请求体
用来存放请求时携带的额外数据，下面写 Socket 请求时，再具体的解析

### 3.2 响应报文
一个简单的响应报文如下
```
HTTP/1.1 200 OK
Etag: 3043b4e81-24cb-64a4d58d
Content-Type: text/html; charset=utf-8
Content-Length: 9419
Last-Modified: Wed, 05 Jul 2023 02:29:33 GMT
Cache-Control: private, max-age=0, proxy-revalidate, no-store, no-cache, must-revalidate
Server: WEBrick/1.8.1 (Ruby/3.2.2/2023-03-30)
Date: Wed, 05 Jul 2023 02:29:51 GMT
Connection: Keep-Alive
```

与请求报文类似，可以将请求报文分为，状态行、响应头、响应体

#### 3.2.1 状态行
包含表明响应结果的状态码，原因短语和HTTP版本
```
HTTP/1.1 200 OK
```

#### 3.2.1 响应头
包含响应的各类信息，

Content-Type 表明响应体的数据类型，即表明客户端应该如何解析返回的数据

Content-Length 响应体的字节数

Connection 是否支持长连接

Transfer-Encoding 传输编码，如值为 chunked，表示报文的响应体分块编码，此时 Content-Length 的值不准确

#### 3.2.3 响应体
具体响应的数据

## 4.Socket 完成网络通信
### 4.1 简单的 GET 请求
上文已经知晓 HTTP 请求体和响应体的格式了，首先来请求服务器，发送请求体
```java
public class NetworkTest {

    public static void main(String[] args) throws IOException {
        urlConnection();
    }

    private static void urlConnection() throws IOException {
        try (Socket socket = new Socket("localhost", 8080)) {
            BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
            //写入请求行
            bw.write("GET /user HTTP/1.1\r\n");
            //写请求头
            bw.write("Host: localhost:8080\r\n");
            //表明报文结束
            bw.write("\r\n");
            bw.flush();
            bw.close();
        }
    }
}
```

上面就可以像服务器发送一条 HTTP 请求了，没什么特别之处，就是按照约定的格式写数据就可以了

再来读取服务器返回的数据，读取服务器返回的数据，会有一点点需要注意的地方。因为读完服务器返回的信息后，服务器不一定会立即释放连接，所以 `read()` 会发生阻塞现象，我们要自己判断服务器是否数据传输完毕，有两种情况处理。

1. 响应体的编码方式不为 chunked 时，可以直接依据响应头 Content-Length 来判断

2. 为 chunked 时，要依次读取块内容，直到读到块长度为 0

这里为简单，只处理非 chunked，这里使用 Okio 来读取服务器返回的数据
```java
public class NetworkTest {
    public static void main(String[] args) throws IOException {
        urlConnection();
    }

    private static void urlConnection() throws IOException {
        try (Socket socket = new Socket("localhost", 8080)) {
            BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
            bw.write("GET /user HTTP/1.1\r\n");// \r\n 表示 CR + LF
            bw.write("Host: localhost:8080\r\n");
            bw.write("\r\n");
            bw.flush();

            final BufferedSource buffer = Okio.buffer(Okio.source(socket.getInputStream()));
            
            //读取状态行
            String statusLine = buffer.readUtf8Line();
            System.out.println(statusLine);

            Map<String, String> headers = new HashMap<>();

            //读取请求头
            String header;
            while ((header = buffer.readUtf8Line()).length() != 0) {
                System.out.println(header);
                final String[] strings = header.split(": ");
                headers.put(strings[0], strings[1]);
            }

            //读取请求体，需要指定读取长度，否则会一直阻塞
            int contentLength = Integer.parseInt(headers.get("Content-Length"));
            final String responseBody = buffer.readUtf8(contentLength);
            System.out.println(responseBody);
            
            buffer.close();
            bw.close();
        }
    }
}
```

output:
```
HTTP/1.1 200 
Content-Type: text/plain;charset=UTF-8
Content-Length: 8
Date: Wed, 05 Jul 2023 05:44:02 GMT
Get User
```

### 4.2 简单的 POST 请求
POST 请求携带简单参数，传递简单参数时，请求体的 Content-Type 要设置为 application/x-www-form-urlencoded，这样服务器就可以解析我们的参数了，
```java
public class NetworkTest {
    public static void main(String[] args) throws IOException {
        postSimpleData();
    }

    private static void postSimpleData() throws IOException {
        try (Socket socket = new Socket("localhost", 8080)) {
            BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
            bw.write("POST /user HTTP/1.1\r\n");// \r\n 表示 CR + LF
            bw.write("Host: localhost:8080\r\n");
            //写入请求体长度
            final long size = Utf8.size("username=alamide");
            bw.write("Content-Length: " + size + "\r\n");
            //form 表单提交
            bw.write("Content-Type: application/x-www-form-urlencoded\r\n");
            bw.write("\r\n");
            //写入参数
            bw.write("username=alamide");
            bw.flush();

            //去读响应体，封装上面的方法
            readResponseBody(socket.getInputStream());
            bw.close();
        }
    }
}
```

### 4.3 POST MultiPart
向服务端发送用户昵称和头像信息，Content-Type 为 multipart/form-data。当类型为 MultiPart 时需要给每一个 Part 设置 Boundary，方便服务器拆分
```java
public class NetworkTest {
    public static void main(String[] args) throws IOException {
        postMultiPartData();
    }

    private static void postMultiPartData() throws IOException {
        final String boundary = ByteString.encodeUtf8(UUID.randomUUID().toString()).utf8();

        try (Socket socket = new Socket("localhost", 8080)) {

            //先将请求体中内容写入缓冲池，主要是为了计算请求体内容的大小
            final Buffer sink = new Buffer();
            //写参数
            writeParam("username", "alamide", boundary, sink);
            writeParam("age", "18", boundary, sink);
            //写图片
            writeImage("avatar", "1.jpeg", new File(file), boundary, sink);

            //写结束标志
            sink.writeUtf8("--" + boundary + "--\r\n");

            //开始向服务端写请求报文
            final BufferedSink buffer = Okio.buffer(Okio.sink(socket.getOutputStream()));
            buffer.writeUtf8("POST /user HTTP/1.1\r\n");// \r\n 表示 CR + LF
            buffer.writeUtf8("Host: localhost:8080\r\n");
            buffer.writeUtf8("Content-Type: multipart/form-data; boundary=" + boundary + "\r\n");
            //写请求体长度
            buffer.writeUtf8("Content-Length: " + sink.size()+"\r\n");
            //表示请求头数据已写完
            buffer.writeUtf8("\r\n");

            //写之前已经处理的请求体
            buffer.write(sink.readByteArray());

            buffer.flush();
            //读响应体
            readResponseBody(socket.getInputStream());
            buffer.close();
        }
    }

    private static void writeImage(String name, String filename, File file, String boundary, BufferedSink buffer) throws IOException {
        buffer.writeUtf8("--" + boundary + "\r\n");
        buffer.writeUtf8("Content-Disposition: form-data; name=\"" + name + "\"; filename=\"" + filename + "\"\r\n");
        buffer.writeUtf8("Content-Type: image/jpeg\r\n");
        buffer.writeUtf8("\r\n");

        buffer.writeAll(Okio.source(file));
        buffer.writeUtf8("\r\n");
    }

    private static void writeParam(String name, String value, String boundary, BufferedSink buffer) throws IOException {
        buffer.writeUtf8("--" + boundary + "\r\n");
        buffer.writeUtf8("Content-Disposition: form-data; name=\"" + name + "\"\r\n");
        buffer.writeUtf8("\r\n");
        buffer.writeUtf8(value + "\r\n");
    }

    private static void readResponseBody(InputStream is) throws IOException {
        System.out.println("read response");
        final BufferedSource buffer = Okio.buffer(Okio.source(is));

        String statusLine = buffer.readUtf8Line();
        System.out.println(statusLine);

        Map<String, String> headers = new HashMap<>();

        String header;
        while ((header = buffer.readUtf8Line()).length() != 0) {
            System.out.println(header);
            final String[] strings = header.split(": ");
            headers.put(strings[0], strings[1]);
        }

        int contentLength = Integer.parseInt(headers.get("Content-Length"));
        final String responseBody = buffer.readUtf8(contentLength);
        System.out.println(responseBody);

        buffer.close();
    }
}
```

## 5.复用 Socket
前面有分析过 TCP 连接的三次握手和断开连接的四次挥手，可以看出 TCP 通信还是挺耗费资源的，如果处理的数据一次通信就发送完毕，那耗损比是非常高的，八次通信，只有一次用来传递数据，太浪费了。

可以考虑复用已建立的连接，在断开连接之前，两个主机之间的信道是否可用，只需要一次验证就可以了。下面来测试一下，复用 Socket 和未复用 Socket 的时间，

当前复用的前提是服务器支持长连接，即 Connection: keep-alive

```java
@State(Scope.Thread)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@Fork(value = 2, jvmArgs = {"-Xms4G", "-Xmx4G"})
public class NetworkTest {

    private static final Integer REQUEST_TIMES = 50;
    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(NetworkTest.class.getSimpleName())
                .warmupIterations(5)
                .measurementIterations(5)
                .build();

        new Runner(opt).run();
    }
    @Benchmark
    public static void testDisposableSocket() {
        for (int i = 0; i < REQUEST_TIMES; i++) {
            try (Socket socket = new Socket("www.zhaosiyuan.com", 80)) {
                BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
                bw.write("GET / HTTP/1.1\r\n");// \r\n 表示 CR + LF
                bw.write("Host: www.zhaosiyuan.com\r\n");
                bw.write("Connection: keep-alive\r\n");
                bw.write("\r\n");
                bw.flush();

                readResponseBody(socket.getInputStream());

                bw.close();
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }
    }
    @Benchmark
    public static void testReuseSocket() {
        try (Socket socket = new Socket("www.zhaosiyuan.com", 80)) {
            for (int i = 0; i < REQUEST_TIMES; i++) {
                BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(socket.getOutputStream()));
                bw.write("GET / HTTP/1.1\r\n");// \r\n 表示 CR + LF
                bw.write("Host: www.zhaosiyuan.com\r\n");
                bw.write("Connection: keep-alive\r\n");
                bw.write("\r\n");
                bw.flush();
                readResponseBody(socket.getInputStream());
            }

        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

请求资源 50 次，平均耗时为：
```
Benchmark                         Mode  Cnt      Score      Error  Units
NetworkTest.testDisposableSocket  avgt   10  14367.734 ± 1343.908  ms/op
NetworkTest.testReuseSocket       avgt   10   6382.347 ± 1967.409  ms/op
```

可以看到复用 Socket ，耗时还是比较少的，请求短时间访问同一资源越频繁，复用 Socket 越高效。

注意复用 Socket 时，不能主动关闭输入输出流，否则会导致 Socket 关闭

## 6.小总结
TCP 的三次握手与四次挥手，脱离 TCP 来理解会比较好理解，就是确认信道双向可用的最少通信次数；通信双方完全接收对方信息后关闭。

HTTP 协议就是一种协议，不要想的太复杂，就是一种规则、一种操作手册、一种说明书。按照指定的规则来通信就可以了。请求报文和响应报文格式都类似，请求行/状态行，头信息，请求体/响应体。其中请求体复杂一点，对与多类型数据需要用 boundary 来分隔各项。以及读数据时需要先读取 Content-Length，再读取指定长度数据，写数据时写 Content-Length。

由于 TCP 连接和断开连接需要多次确认，所以可以复用已建立的连接。节省资源。




