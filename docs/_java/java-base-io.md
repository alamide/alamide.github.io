---
layout: post
title: Java 基础 IO
categories: java
tags: Java IO
date: 2023-04-11
---
Java 基础语法 IO，学习内容来自 《Java 编程的逻辑》
<!--more-->
## 1.InputStream、OutputStream
读写一个文件
```java
try(FileInputStream is = new FileInputStream("data.txt")) {
    byte[] bytes = new byte[1024];
    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    int read = 0;
    while ((read = is.read(bytes)) != -1){
        baos.write(bytes, 0, read);
    }
    System.out.println(baos.toString(StandardCharsets.UTF_8));
}catch (Exception e){
    e.printStackTrace();
}

try (OutputStream out = new FileOutputStream("data.txt", true)) {
    out.write("\n".getBytes(StandardCharsets.UTF_8));
    out.write("Hello World".getBytes(StandardCharsets.UTF_8));
} catch (Exception e) {
    e.printStackTrace();
}
```

复制文件
```java
try (FileInputStream is = new FileInputStream("data.txt");OutputStream out = new FileOutputStream("data_backup.txt", true)) {
    is.transferTo(out);
} catch (Exception e) {
    e.printStackTrace();
}
```
## 2.ByteArrayInputStream、ByteArrayOutputStream
```java
ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(new byte[]{});
```
## 3.DataInputStream、DataOutputStream
基本数据读取
```java
@Test
public void readStudents() {
    try (DataInputStream dis = new DataInputStream(new FileInputStream("object.txt"))) {
        final int size = dis.readInt();
        System.out.println(size);
        List<Student> students = new ArrayList<>(size);
        for (int i = 0; i < size; i++) {
            Student student = new Student();
            student.setName(dis.readUTF());
            student.setId(dis.readUTF());
            student.setAge(dis.readInt());
            int length = dis.readInt();
            String[] hobbies = new String[length];
            for (int j = 0; j < length; j++) {
                hobbies[j] = dis.readUTF();
            }
            student.setHobbies(hobbies);
            student.setHeight(dis.readDouble());
            student.setWeight(dis.readDouble());
            students.add(student);
        }
        students.forEach(System.out::println);
    } catch (Exception e) {
        e.printStackTrace();
    }
}

@Test
public void saveStudents() {
    try (DataOutputStream dos = new DataOutputStream(new FileOutputStream("object.txt"))) {
        final List<Student> students = IntStream.rangeClosed(1, 10)
                .boxed()
                .map(i -> new Student("stud" + i, "10000" + i, 16 + i % 3, new String[]{"football", "game"}, 178.3, 65.3))
                .toList();
        dos.writeInt(students.size());
        for (Student student : students) {
            dos.writeUTF(student.getName());
            dos.writeUTF(student.getId());
            dos.writeInt(student.getAge());
            dos.writeInt(student.getHobbies().length);
            for (String hobby : student.getHobbies()) {
                dos.writeUTF(hobby);
            }
            dos.writeDouble(student.getHeight());
            dos.writeDouble(student.getWeight());
        }

        dos.flush();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```
## 4.BufferedInputStream、BufferedOutputStream
InputStream 的包装类，可以提高效率
```java
ByteArrayOutputStream baos = new ByteArrayOutputStream();

try (FileInputStream is = new FileInputStream("data.txt")) {
    byte[] bytes = new byte[1024];
    int read = 0;
    while ((read = is.read(bytes)) != -1) {
        baos.write(bytes, 0, read);
    }
} catch (Exception e) {
    e.printStackTrace();
}

long start = System.currentTimeMillis();

try (FileOutputStream out = new FileOutputStream("data.txt")) {
    byte[] bytes = baos.toByteArray();
    for (byte b : bytes) {
        out.write(b);
    }
} catch (IOException e) {
    throw new RuntimeException(e);
}
log.info("FileOutputStream cost time: {}ms", (System.currentTimeMillis() - start));

start = System.currentTimeMillis();
try (BufferedOutputStream bos = new BufferedOutputStream(new FileOutputStream("data.txt"))) {
    byte[] bytes = baos.toByteArray();
    for (byte b : bytes) {
        bos.write(b);
    }
    bos.flush();
} catch (IOException e) {
    throw new RuntimeException(e);
}
log.info("BufferedOutputStream cost time: {}ms", (System.currentTimeMillis() - start));
```

output:
```
08:25:08.711 [main] INFO com.alamide.base.FileTest - FileOutputStream cost time: 9ms
08:25:08.718 [main] INFO com.alamide.base.FileTest - BufferedOutputStream cost time: 1ms
```
可以看到同样的数据，在按字节写时，使用 `Buffer` 的效率要高出很多倍

## 5.InputStreamReader、OutputStreamWriter
可以以字符串的形式写
```java
@Test
public void testStreamReader() {
    try (Writer writer = new OutputStreamWriter(new FileOutputStream("data.txt", true))) {
        writer.write("\ntestStreamReader");
    } catch (IOException e) {
        throw new RuntimeException(e);
    }

    try (Reader reader = new InputStreamReader(new FileInputStream("data.txt"))) {
        char[] chars = new char[1024];
        int count = 0;
        StringBuilder stringBuilder = new StringBuilder();
        while ((count = reader.read(chars)) != -1){
            stringBuilder.append(Arrays.copyOfRange(chars, 0, count));
        }
        System.out.println(stringBuilder.toString());
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```

```java
try (Writer writer = new FileWriter("data.txt", true)) {
    writer.write("\ntestStreamReader");
} catch (IOException e) {
    throw new RuntimeException(e);
}

try (Reader reader = new FileReader("data.txt")) {
    char[] chars = new char[1024];
    int count = 0;
    StringBuilder stringBuilder = new StringBuilder();
    while ((count = reader.read(chars)) != -1){
        stringBuilder.append(Arrays.copyOfRange(chars, 0, count));
    }
    System.out.println(stringBuilder.toString());
} catch (IOException e) {
    throw new RuntimeException(e);
}
```

## 6.FileReader、FileWriter
```java
try (Writer writer = new FileWriter("data.txt", true)) {
    writer.write("\ntestStreamReader");
} catch (IOException e) {
    throw new RuntimeException(e);
}

try (Reader reader = new FileReader("data.txt")) {
    char[] chars = new char[1024];
    int count = 0;
    StringBuilder stringBuilder = new StringBuilder();
    while ((count = reader.read(chars)) != -1){
        stringBuilder.append(Arrays.copyOfRange(chars, 0, count));
    }
    System.out.println(stringBuilder.toString());
} catch (IOException e) {
    throw new RuntimeException(e);
}
```

## 7.CharArrayReader、CharArrayWriter
类似于 `ByteArrayInputStream` 、`ByteArrayOutputStream`
```java
try (Reader reader = new FileReader("data.txt")) {
    char[] chars = new char[1024];
    int count = 0;
    CharArrayWriter charArrayWriter = new CharArrayWriter();
    while ((count = reader.read(chars)) != -1){
        charArrayWriter.write(chars, 0, count);
    }
    System.out.println(charArrayWriter.toString());
} catch (IOException e) {
    throw new RuntimeException(e);
}
```

## 8.StringReader、StringWriter
```java
StringWriter stringWriter = new StringWriter();
stringWriter.write("Hello world!");
System.out.println(stringWriter);

StringReader stringReader = new StringReader("Hello world");
```

## 9.BufferedReader、BufferedWriter
可以按行存取数据
```java
@Test
public void testBufferedWriter(){
    try(BufferedWriter bufferedWriter = new BufferedWriter(new FileWriter("data.txt"))){
        for (int integer = 1; integer <= 10; integer++) {
            bufferedWriter.newLine();
            bufferedWriter.write("Hello World" +integer);
        }
        bufferedWriter.flush();
    } catch (IOException e) {
        throw new RuntimeException(e);
    }

    try(BufferedReader bufferedReader = new BufferedReader(new FileReader("data.txt"))){
        String line;
        while ((line = bufferedReader.readLine()) != null){
            System.out.println(line);
        }
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```

## 10.PrintWriter
```java
@Test
public void testPrintWriter(){
    try(PrintWriter printWriter = new PrintWriter(new FileWriter("data.txt", true))) {
        printWriter.printf("this is a %s", "test");
        printWriter.println("testPrintWriter");
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```

## 11.Scanner
Scanner是一个单独的类，它是一个简单的文本扫描器，能够分析基本类型和字符串，它需要一个分隔符来将不同数据区分开来，默认是使用空白符，可以通过useDelimiter()方法进行指定。
```java
@Test
public void testScanner(){
    try(BufferedReader bufferedReader = new BufferedReader(new StringReader("2,suzhou,china,38.55,false"))){
        String line;
        while ((line = bufferedReader.readLine()) != null){
            Scanner scanner = new Scanner(line).useDelimiter(",");
            System.out.println(scanner.nextInt());
            System.out.println(scanner.next());
            System.out.println(scanner.next());
            System.out.println(scanner.nextDouble());
            System.out.println(scanner.nextBoolean());
        }
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```

从控制台读取数据
```java
Scanner scanner = new Scanner(System.in);
int nextInt = scanner.nextInt();
System.out.println(nextInt);
String next = scanner.next();
System.out.println(next);
```

标准流重定向
```java
@Test
public void testStreamRedirect() throws FileNotFoundException {
    System.setIn(new FileInputStream("data.txt"));
    System.setOut(new PrintStream("out.txt"));
    System.setErr(new PrintStream("error.txt"));
    try {
        Scanner scanner = new Scanner(System.in);
        System.out.println(scanner.nextLine());
        System.out.println(scanner.nextLine());
        System.out.println(scanner.nextLine());
    }catch (Exception e){
        System.err.println(e.getMessage());
    }
}
```
## 12.Properties
文件不能直接处理中文，非 ASCII 字符要使用 Unicode 编码
```java
Properties properties = new Properties();
properties.load(getClass().getClassLoader().getResourceAsStream("config.properties"));
final String property = properties.getProperty("jdbc.username");
System.out.println(property);
```

## 13.Serializable
```java
@Test
public void testSerializable(){
    try (ObjectOutputStream dos = new ObjectOutputStream(new FileOutputStream("object.txt"))) {
        final List<Student> students = IntStream.rangeClosed(1, 10)
                .boxed()
                .map(i -> new Student("stud" + i, "10000" + i, 16 + i % 3, new String[]{"football", "game"}, 178.3, 65.3))
                .toList();
        dos.writeObject(students);
    } catch (Exception e) {
        e.printStackTrace();
    }

    try (ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream("object.txt"))){
        List<Student> students = (List<Student>) objectInputStream.readObject();
        students.forEach(System.out::println);
    } catch (IOException | ClassNotFoundException e) {
        throw new RuntimeException(e);
    }
}
```