---
layout: post
title: JavaPoet Java 源文件生成工具
categories: java
tags: Java 
date: 2023-06-14
---
Java 源代码生成工具。
<!--more-->

JavaPoet 官方文档，[在这里](https://github.com/square/javapoet)

依据 JavaPoet 的设计模式，我们可以把 Java 类分为四个部分
1. JavaFile/Package 对应于 JavaFile

2. Class 类定义，对应于 TypeSpec

3. Method 方法、方法体，对应于 MethodSpec

4. Field 字段，对应于 FieldSpec

每个部分各司其职，都使用 build 构建方式。TypeSpec 负责组合 Field 和 Method，JavaFile 负责组合 Package 和 Class。
## 1.依赖引入
```xml
<dependency>
  <groupId>com.squareup</groupId>
  <artifactId>javapoet</artifactId>
  <version>1.13.0</version>
</dependency>
```

```groovy
implementation 'com.squareup:javapoet:1.13.0'
```
## 2.引用
### 2.1 $L
字面量引用
```java
int a = 0;
final MethodSpec methodSpec = MethodSpec
        .methodBuilder("main")
        .returns(void.class)
        .addStatement("$T.out.println($L)", System.class, a)
        .build();
```

### 2.2 $S
字符串引用，自动加上双引号
```java
final MethodSpec methodSpec = MethodSpec
        .methodBuilder("main")
        .returns(void.class)
        .addStatement("$T.out.println($S)", System.class, "Hallo")
        .build();
```

### 2.3 $T
类的引用，会帮我们自动导入所需的包
```java
final MethodSpec methodSpec = MethodSpec
                .methodBuilder("main")
                .returns(Date.class)
                .addStatement("return new $T()", Date.class)
                .build();

package com.alamide.lib;
//自动导入
import java.util.Date;

public final class HelloWorld {
  Date main() {
    return new Date();
  }
}
```

对于不存在的类(可能是后续生成)，可以使用 Class.forName
```java
final ClassName futureClass = ClassName.get("com.alamide.future", "FutureClass");
final MethodSpec methodSpec = MethodSpec
        .methodBuilder("main")
        .returns(futureClass)
        .addStatement("return new $T()", futureClass)
        .build();

package com.alamide.lib;

import com.alamide.future.FutureClass;

public final class HelloWorld {
  FutureClass main() {
    return new FutureClass();
  }
}
```

范型及 ClassName 的重要使用场景
```java
final ClassName futureClass = ClassName.get("com.alamide.future", "FutureClass");
final ClassName list = ClassName.get("java.util", "List");
final ClassName arrayList = ClassName.get("java.util", "ArrayList");
final ParameterizedTypeName parameterizedTypeName = ParameterizedTypeName.get(list, futureClass);
final MethodSpec methodSpec = MethodSpec
        .methodBuilder("beyond")
        .returns(parameterizedTypeName)
        .addStatement("$T result = new $T<>()", parameterizedTypeName, arrayList)
        .addStatement("result.add(new $T())", futureClass)
        .addStatement("result.add(new $T())", futureClass)
        .addStatement("result.add(new $T())", futureClass)
        .addStatement("return result")
        .build();

List<FutureClass> beyond() {
    List<FutureClass> result = new ArrayList<>();
    result.add(new FutureClass());
    result.add(new FutureClass());
    result.add(new FutureClass());
    return result;
}
```

静态导入
```java
final ClassName futureClass = ClassName.get("com.alamide.future", "FutureClass");
final ClassName list = ClassName.get("java.util", "List");
final ClassName arrayList = ClassName.get("java.util", "ArrayList");
final ParameterizedTypeName parameterizedTypeName = ParameterizedTypeName.get(list, futureClass);
final MethodSpec methodSpec = MethodSpec
        .methodBuilder("beyond")
        .returns(parameterizedTypeName)
        .addStatement("$T result = new $T<>()", parameterizedTypeName, arrayList)
        .addStatement("result.add(new $T())", futureClass)
        .addStatement("result.add(new $T())", futureClass)
        .addStatement("result.add(new $T())", futureClass)
        .addStatement("$T.sort(result)", Collections.class)
        .addStatement("return result")
        .build();
 TypeSpec typeSpec = TypeSpec.classBuilder("HelloWorld")
        .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
        .addMethod(methodSpec)
        .build();
final JavaFile javaFile = JavaFile
        .builder("com.alamide.lib", typeSpec)
        .addStaticImport(Collections.class, "*")
        .build();

package com.alamide.lib;

import static java.util.Collections.*;

import com.alamide.future.FutureClass;
import java.util.ArrayList;
import java.util.List;

public final class HelloWorld {
  List<FutureClass> beyond() {
    List<FutureClass> result = new ArrayList<>();
    result.add(new FutureClass());
    result.add(new FutureClass());
    result.add(new FutureClass());
    sort(result);
    return result;
  }
}
```

### 2.4 $N
对另一个 MethodSpec 的引用
```java
final MethodSpec methodSpec1 = MethodSpec
        .methodBuilder("sum")
        .addModifiers(Modifier.PUBLIC)
        .returns(int.class)
        .addParameter(int.class, "a")
        .addParameter(int.class, "b")
        .addStatement("return a + b")
        .build();

final MethodSpec methodSpec2 = MethodSpec.methodBuilder("example")
        .addModifiers(Modifier.PUBLIC)
        .returns(void.class)
        .addStatement("$T.out.println($N(1, 2))", System.class, methodSpec1)
        .addStatement("$T.out.println($N(3, 5))", System.class, methodSpec1)
        .build();

public int sum(int a, int b) {
    return a + b;
}

public void example() {
    System.out.println(sum(1, 2));
    System.out.println(sum(3, 5));
}
```

### 2.5 Constructor
```java
final MethodSpec methodSpec1 = MethodSpec
        .constructorBuilder()
        .addModifiers(Modifier.PUBLIC)
        .build();
```

### 2.6 Parameter
```java
public int sum(final int a, final int b) {
    return a + b;
}

final ParameterSpec p1 = ParameterSpec.builder(int.class, "a", Modifier.FINAL).build();
final ParameterSpec p2 = ParameterSpec.builder(int.class, "b", Modifier.FINAL).build();
final MethodSpec methodSpec1 = MethodSpec
        .methodBuilder("sum")
        .addModifiers(Modifier.PUBLIC)
        .returns(int.class)
        .addParameter(p1)
        .addParameter(p2)
        .addStatement("return a + b")
        .build();
```

### 2.7 Interface
```java
TypeSpec typeSpec = TypeSpec.interfaceBuilder("HelloWorld")
        .addModifiers(Modifier.PUBLIC)
        .addField(FieldSpec.builder(int.class, "A")
                .addModifiers(Modifier.PUBLIC, Modifier.STATIC, Modifier.FINAL)
                .initializer("$L", 1).build())
        .build();

public interface HelloWorld {
  int A = 1;
}
```

### 2.8 Enums
```java
TypeSpec typeSpec = TypeSpec.enumBuilder("Version")
        .addModifiers(Modifier.PUBLIC)
        .addEnumConstant("VERSION_1")
        .addEnumConstant("VERSION_2")
        .build();

public enum Version {
  VERSION_1,

  VERSION_2
}
```

### 2.9 Anonymous Inner Classes
```java
final TypeSpec anonymous = TypeSpec.anonymousClassBuilder("")
        .addSuperinterface(ParameterizedTypeName.get(Comparator.class, String.class))
        .addMethod(MethodSpec.methodBuilder("compare")
                .addAnnotation(Override.class)
                .addModifiers(Modifier.PUBLIC)
                .addParameter(String.class, "a")
                .addParameter(String.class, "b")
                .returns(int.class)
                .addStatement("return a.length() - b.length()")
                .build())
        .build();

final TypeSpec typeSpec = TypeSpec.classBuilder("HelloWorld")
        .addMethod(MethodSpec.methodBuilder("sortByLength")
                .addParameter(ParameterizedTypeName.get(List.class, String.class), "strings")
                .addStatement("$T.sort($N, $L)", Collections.class, "strings", anonymous)
                .returns(void.class).build())
        .build();

void sortByLength(List<String> strings) {
    Collections.sort(strings, new Comparator<String>() {
        @Override
        public int compare(String a, String b) {
        return a.length() - b.length();
        }
    });
}
```

### 2.10 Annotation
```java
@Route(
    path = {
        "/main",
        "/home"
    },
    isNeedLogin = true,
    desc = "主页"
)
class HelloWorld {
}

final TypeSpec typeSpec = TypeSpec.classBuilder("HelloWorld")
        .addAnnotation(AnnotationSpec
                .builder(ClassName.get("com.alamide.route", "Route"))
                .addMember("path", "$S", "/main")
                .addMember("path", "$S", "/home")
                .addMember("isNeedLogin", "$N", "true")
                .addMember("desc", "$S", "主页")
                .build())
        .build();
```

### 2.11 Javadoc
```java
/**
 * This is a test class, you can see {@link Route}
 */
@Route(
    path = {
        "/main",
        "/home"
    },
    isNeedLogin = true,
    desc = "主页"
)
class HelloWorld {
}

final TypeSpec typeSpec = TypeSpec.classBuilder("HelloWorld")
        .addAnnotation(AnnotationSpec
                .builder(ClassName.get("com.alamide.route", "Route"))
                .addMember("path", "$S", "/main")
                .addMember("path", "$S", "/home")
                .addMember("isNeedLogin", "$N", "true")
                .addMember("desc", "$S", "主页")
                .build())
        .addJavadoc("This is a test class, you can see {@link $T}", ClassName.get("com.alamide.route", "Route"))
        .build();
```
## 3.Example
### 3.1 Example-1
目标类
```java
package com.alamide.lib;
import java.lang.String;
import java.lang.System;

public final class HelloWorld {
  public static void main(String[] args) {
    System.out.println("Hello JavaPoet!");
  }
}
```

```java
MethodSpec methodSpec = MethodSpec.methodBuilder("main")
        .addModifiers(Modifier.PUBLIC, Modifier.STATIC)
        .returns(void.class)
        .addParameter(String[].class, "args")
        .addStatement("$T.out.println($S)", System.class, "Hello JavaPoet!")
        .build();
TypeSpec typeSpec = TypeSpec.classBuilder("HelloWorld")
        .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
        .addMethod(methodSpec)
        .build();
final JavaFile javaFile = JavaFile.builder("com.alamide.lib", typeSpec).build();
javaFile.writeTo(Path.of("./src/main/java").toAbsolutePath());
```

### 3.2 Example-2
目标类
```java
void main() {
  int total = 0;
  for (int i = 0; i < 10; i++) {
    total += i;
  }
}
```

```java
final MethodSpec methodSpec = MethodSpec.methodBuilder("main")
        .returns(void.class)
        .addCode(
                "  int total = 0;\n" +
                        "  for (int i = 0; i < 10; i++) {\n" +
                        "    total += i;\n" +
                        "  }\n")
        .build();
System.out.println(methodSpec);

final MethodSpec methodSpec = MethodSpec.methodBuilder("main")
        .returns(void.class)
        .addStatement("int total=0")
        .beginControlFlow("for(int i=0; i<10; i++)")
        .addStatement("total += i")
        .endControlFlow()
        .build();
System.out.println(methodSpec);
```

### 3.3 Example-3
目标类
```java
void main() {
  long now = System.currentTimeMillis();
  if (System.currentTimeMillis() < now)  {
    System.out.println("Time travelling, woo hoo!");
  } else if (System.currentTimeMillis() == now) {
    System.out.println("Time stood still!");
  } else {
    System.out.println("Ok, time still moving forward");
  }
}
```

```java
final MethodSpec methodSpec = MethodSpec
        .methodBuilder("main")
        .returns(void.class)
        .addStatement("long now = $T.currentTimeMillis()", System.class)
        .beginControlFlow("if($T.currentTimeMillis() < now)", System.class)
        .addStatement("$T.out.println($S)", System.class, "Time travelling, woo hoo!")
        .nextControlFlow("else if($T.currentTimeMillis() == now)", System.class)
        .addStatement("$T.out.println($S)", System.class, "Time travelling, woo hoo!")
        .nextControlFlow("else")
        .addStatement("$T.out.println($S)", System.class, "Ok, time still moving forward")
        .endControlFlow()
        .build();
```

### 3.4 Example-4
目标类
```java
void main() {
  try {
    throw new Exception("Failed");
  } catch (Exception e) {
    throw new RuntimeException(e);
  }
}
```

```java
final MethodSpec methodSpec = MethodSpec
            .methodBuilder("main")
            .returns(void.class)
            .beginControlFlow("try")
            .addStatement("throw new $T($S)", Exception.class, "Failed")
            .nextControlFlow("catch($T e)", Exception.class)
            .addStatement("throw new $T(e)", RuntimeException.class)
            .endControlFlow()
            .build();
```

## 4.万能例子
这个例子有些地方是不合法的，如 `? extends String` ，`String` 不可被继承
```java
package com.alamide.lib;

import java.io.File;
import java.io.IOException;
import java.io.Serializable;
import java.lang.Comparable;
import java.lang.Exception;
import java.lang.Integer;
import java.lang.Object;
import java.lang.Override;
import java.lang.Runnable;
import java.lang.RuntimeException;
import java.lang.String;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public abstract class Clazz<T> extends Object implements Serializable, Comparable<String>, Map<T, ? extends String> {
  static {
  }

  private int mInt;

  private int[] mArr;

  private File mRef;

  private T mT;

  private List<String> mParameterizedField;

  private List<? extends String> mWildcardField;

  {
  }

  public Clazz() {
  }

  @Override
  public <T> int method(String string, T t, Map<Integer, ? extends T> map) throws IOException,
      RuntimeException {
    int foo = 1;
    String bar = "a string";
    Object obj = new HashMap<Integer, ? extends T>(5);
    baz(new Runnable(String param) {
      @Override
      void run() {
      }
    });
    for (int i = 0; i < 5; i++) {
    }
    while (false) {
    }
    do {
    } while (false);
    if (false) {
    } else if (false) {
    } else {
    }
    try {
    } catch (Exception e) {
      e.printStackTrace();
    } finally {
    }
    return 0;
  }

  class InnerClass {
  }
}
```

```java
final FieldSpec mInt = FieldSpec.builder(int.class, "mInt", Modifier.PRIVATE).build();
final FieldSpec mArr = FieldSpec.builder(int[].class, "mArr", Modifier.PRIVATE).build();
final FieldSpec mT = FieldSpec.builder(TypeVariableName.get("T"), "mT", Modifier.PRIVATE).build();
final FieldSpec mRef = FieldSpec.builder(File.class, "mRef", Modifier.PRIVATE).build();
final FieldSpec mParameterizedField = FieldSpec.builder(ParameterizedTypeName.get(List.class, String.class), "mParameterizedField", Modifier.PRIVATE).build();
final FieldSpec mWildcardField = FieldSpec.builder(ParameterizedTypeName.get(ClassName.get(List.class), WildcardTypeName.subtypeOf(String.class)), "mWildcardField", Modifier.PRIVATE).build();

final MethodSpec constructor = MethodSpec.constructorBuilder().addModifiers(Modifier.PUBLIC).build();

final MethodSpec methodSpec = MethodSpec.methodBuilder("method")
        .addAnnotation(Override.class)
        .addModifiers(Modifier.PUBLIC)
        .addTypeVariable(TypeVariableName.get("T"))
        .returns(int.class)
        .addParameter(String.class, "string")
        .addParameter(TypeVariableName.get("T"), "t")
        .addParameter(ParameterizedTypeName.get(ClassName.get(Map.class), ClassName.get(Integer.class), WildcardTypeName.subtypeOf(TypeVariableName.get("T"))), "map")
        .addException(IOException.class)
        .addException(RuntimeException.class)
        .addStatement("int foo = $N", "1")
        .addStatement("String bar = $S", "a string")
        .addStatement("$T obj = new $T<$T, ? extends T>($N)", Object.class, HashMap.class, Integer.class, "5")
        .addStatement("baz($L)",
                TypeSpec.anonymousClassBuilder("$T params", String.class)
                        .addSuperinterface(Runnable.class)
                        .addMethod(
                                MethodSpec
                                        .methodBuilder("run")
                                        .returns(void.class)
                                        .addAnnotation(Override.class)
                                        .build()).build())
        .beginControlFlow("for(int i=0; i<5; i++)")
        .endControlFlow()
        .beginControlFlow("while(false)")
        .endControlFlow()
        .beginControlFlow("do")
        .endControlFlow("while(false)")
        .beginControlFlow("if(false)")
        .nextControlFlow("else if(false)")
        .nextControlFlow("else")
        .endControlFlow()
        .beginControlFlow("try")
        .nextControlFlow("catch($T e)", Exception.class)
        .addStatement("e.printStackTrace()")
        .nextControlFlow("finally")
        .endControlFlow()
        .addStatement("return 0").build();

final TypeSpec innerClass = TypeSpec.classBuilder("InnerClass").build();

final TypeSpec typeSpec = TypeSpec.classBuilder("Clazz").addTypeVariable(TypeVariableName.get("T"))
        .addModifiers(Modifier.PUBLIC, Modifier.ABSTRACT)
        .superclass(String.class)
        .addSuperinterface(Serializable.class)
        .addSuperinterface(ParameterizedTypeName.get(Comparable.class, String.class))
        .addSuperinterface(ParameterizedTypeName.get(ClassName.get(Map.class), TypeVariableName.get("T"), WildcardTypeName.subtypeOf(String.class)))
        .addStaticBlock(CodeBlock.builder().build())
        .addField(mInt)
        .addField(mArr)
        .addField(mT)
        .addField(mRef)
        .addField(mParameterizedField)
        .addField(mWildcardField)
        .addInitializerBlock(CodeBlock.builder().build())
        .addType(innerClass)
        .addMethod(constructor)
        .addMethod(methodSpec)
        .build();


final JavaFile javaFile = JavaFile
        .builder("com.alamide.lib", typeSpec)
        .build();
javaFile.writeTo(Path.of("./src/main/java").toAbsolutePath());
```