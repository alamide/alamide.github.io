---
layout: post
title: APT Java 注解处理器
categories: java
tags: Java APT
date: 2023-06-15
---
APT(Annotation Processing Tool) Java 注解处理器，可以用来在编译时扫描和处理注解。通过 APT 可以获取到注解和被注解对象的相关信息，在拿到这些信息之后我们可以根据需求生成一些代码。
<!--more-->

## 1.配置
本实验是在 AndroidStudio 中进行，涉及三个模块：`main` 、`router-annotation` 、`router-compiler` 。其中 `router-annotation` 为定义注解的模块，`router-complier` 为处理注解的模块，`main` 为具体使用注解的模块。

### 1.1 router-annotation
注解的定义
```java
package com.alamide.router;


@Target({ElementType.TYPE, ElementType.METHOD, ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.CLASS)
public @interface Route {
    String path() default "";

    String desc() default "";

    boolean isNeedLogin() default false;
}
```

### 1.2 main
```java
package com.alamide.rrrandroid;

@Route(path = "/path2")
public class MainAnno extends Activity {

    @Route(desc = "field name")
    private String name = "alamide";

    @Route(desc = "method")
    public void sayHello(){
        saySomething("Hello!");
    }

    public void saySomething(@Route(desc = "parameter") String something){
        System.out.println(something);
    }
}
```

要使注解生效，需要在 gradle 中进行如下配置
```groovy
dependencies {
    annotationProcessor project(':router_annotaion_processor')
    implementation project(":router_annotation")
}
```

### 1.3 router-compiler
```java
package com.alamide.router.processor;

@SupportedAnnotationTypes({"com.alamide.router.Route"})
public class RouterProcessor extends AbstractProcessor {

    private Messager messager;
    private Filer filer;
    private Elements elements;
    private Types types;


    @Override
    public synchronized void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        this.messager = processingEnv.getMessager();
        this.filer = processingEnv.getFiler();
        this.elements = processingEnv.getElementUtils();
        this.types = processingEnv.getTypeUtils();
        messager.printMessage(Diagnostic.Kind.NOTE, "<<<<<<<<<<<<<<init>>>>>>>>>>>>>>>");

    }

    @Override
    public SourceVersion getSupportedSourceVersion() {
        return SourceVersion.latestSupported();
    }

    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        messager.printMessage(Diagnostic.Kind.NOTE, "process>>>>>>>>>>>>>>>");
        return false;
    }

    private void print(Object content) {
        messager.printMessage(Diagnostic.Kind.NOTE, content.toString());
    }
}
```

要使注解生效，需要在 `resources/META-INF/services/javax.annotation.processing.Processor` 文件中添加如下语句
```
com.alamide.router.processor.RouterProcessor
```

## 2.一些 API 使用方法
在注解处理器的体系中，一个类的每个成员都被看作 `element` 。`Package` 看作 `PackageElement` ，`Class` 看作 `TypeElement` ，`Field/Parameter` 看作 `VariableElement` ，`Method/Constructor` 看作 `ExecutableElement` ，范型类或方法看作 `TypeParameterElement` ，所有这些类都是 `Element` 的子类。

### 2.1 获取所有被注解的元素
```java
@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    messager.printMessage(Diagnostic.Kind.NOTE, "process>>>>>>>>>>>>>>>");
    Set<? extends Element> rootElements = roundEnv.getElementsAnnotatedWith(Route.class);
}
```

### 2.2 Element
```java
public interface Element extends javax.lang.model.AnnotatedConstruct {
    /**
     * 获取当前元素所标记的注解，例如
     * @Route(desc = "field name")
     * private String name = "alamide";
     * 
     * Route annotation = element.getAnnotation(Route.class);
     * print(annotation.desc());
     * 
     * 输出：field name
     */
    @Override
    <A extends Annotation> A getAnnotation(Class<A> annotationType);

     /**
     * 获取当前元素的类别，有如下常用类别
     * CLASS
     * FIELD
     * METHOD
     * PARAMETER
     */
    ElementKind getKind();

    /**
     * 获取当前元素的修饰符，有就返回，没有就不返回，对于 Parameter 可以有 final 作为修饰符
     */
    Set<Modifier> getModifiers();

    /**
     * 获取当前元素的名称，包括类名、字段名、方法名、参数名，只返回名字，不返回修饰符
     */
    Name getSimpleName();

    /**
     * 获取当前元素的最近的父元素
     * CLASS     的父元素 PACKAGE
     * FIELD     的父元素 CLASS
     * METHOD    的父元素 CLASS
     * PARAMETER 的父元素 METHOD
     */
    Element getEnclosingElement();

    /**
     * 获取当前元素的所有子元素，只有 CLASS 和 PACKAGE 级别的元素有子元素，例子：
     * 对于 main 模块的 MainAnno 类上的注解 @Route
     *  List<? extends Element> enclosedElements = element.getEnclosedElements();
     *  print(enclosedElements.stream().map(Element::getKind).collect(Collectors.toList()));
     *
     *  输出：[CONSTRUCTOR, FIELD, METHOD, METHOD]
     */
    List<? extends Element> getEnclosedElements();

    /**
     * 获取当前元素的类型
     */
    TypeMirror asType();
}
```

### 2.3 Elements
```java
public interface Elements {

    /**
     * 获取当前元素所在的包，所有元素都可以获取，可以获取当前类所在的包信息，例如：
     * PackageElement packageElement = elements.getPackageOf(element);
     * print(packageElement.getQualifiedName());
     * 
     * 输出：com.alamide.rrrandroid
     */
    PackageElement getPackageOf(Element type);

     /**
     * 获取类元素下所有的子元素，包括当前类的 private 和 父类的 public 和 protected，例子：
     * @Route(path = "/main")
     * public class MainAnno {
     *
     *     @Route(desc = "field name")
     *     private String name = "alamide";
     *
     *     @Route(desc = "method")
     *     public void sayHello(){
     *         saySomething("Hello!");
     *     }
     *
     *     public void saySomething(@Route(desc = "parameter") final String something){
     *         System.out.println(something);
     *     }
     * }
     *
     * if(element.getKind() == ElementKind.CLASS){
     *     List<? extends Element> allMembers = elements.getAllMembers((TypeElement) element);
     *     print(allMembers.stream()
     *             .flatMap((Function<Element, Stream<Modifier>>) element1 -> element1.getModifiers().stream())
     *             .filter(m -> m.ordinal() < Modifier.ABSTRACT.ordinal())
     *             .collect(Collectors.toSet()));
     *
     *     print(allMembers.stream()
     *             .map(Element::getKind)
     *             .collect(Collectors.toSet()));
     * }
     *
     * 输出：[protected, private, public]，[FIELD, CONSTRUCTOR, METHOD]
     * 结果会包含父类 Object 中的非 private 的方法、构造方法及字段
     */
    List<? extends Element> getAllMembers(TypeElement type);

    /**
     * 由全类名获取 TypeElement，例子：
     * TypeMirror typeMirror = elements.getTypeElement("android.app.Activity").asType();
     * print(typeMirror);
     */
    TypeElement getTypeElement(CharSequence name);
}
```

### 2.4 Types
```java
public interface Types {

    /**
     * 擦除范型信息，例子：
     * TypeMirror t1 = elements.getTypeElement(ArrayList.class.getCanonicalName()).asType();
     * print(t1);
     * print(types.erasure(t1));
     * 
     * 输出：java.util.ArrayList<E>、java.util.ArrayList
     */
    TypeMirror erasure(TypeMirror t);


    /**
     * t1 是否为 t2 的子类，t1 可以认为是 t1 的子类，范型类需要擦除范型信息，例子：
     * Set<? extends Element> rootElements = roundEnv.getElementsAnnotatedWith(Route.class);
     * TypeMirror typeActivity = elements.getTypeElement("android.app.Activity").asType();
     * for (Element element : rootElements) {
     *     TypeMirror tm = element.asType();
     *     if(element.getKind() == ElementKind.CLASS && types.isSubtype(tm, typeActivity)){
     *         print(element);
     *     }
     * }
     * 
     * 输出：com.alamide.rrrandroid.MainActivity、com.alamide.rrrandroid.MainAnno
     * 
     */
    boolean isSubtype(TypeMirror t1, TypeMirror t2);

    /**
     * t1, t2 是否为同一个类，范型类需要擦除范型信息
     */
    boolean isSameType(TypeMirror t1, TypeMirror t2);

    /**
     * t1 是否可以分配给 t2 类型，即 t1 是否为为 t2 的子类，范型类需要擦除范型信息，例子：
     * TypeMirror t1 = elements.getTypeElement(String.class.getCanonicalName()).asType();
     * TypeMirror t2 = elements.getTypeElement(Object.class.getCanonicalName()).asType();
     * print(types.isAssignable(t1, t2));
     *
     * 输出：true
     *
     * 测试 List 与 ArrayList 不成立，是范型的原因，可以擦出范型再做比较
     * TypeMirror t1 = elements.getTypeElement(ArrayList.class.getCanonicalName()).asType();
     * TypeMirror t2 = elements.getTypeElement(List.class.getCanonicalName()).asType();
     * print(types.isAssignable(types.erasure(t1), types.erasure(t2)));
     * 
     * 输出：true
     */
    boolean isAssignable(TypeMirror t1, TypeMirror t2);

    /**
     * 生成范型类，例子：
     * WildcardType wildcardType1 = types.getWildcardType(elements.getTypeElement("com.alamide.rrrandroid.MainAnno").asType(), null);
     * WildcardType wildcardType2 = types.getWildcardType(null, elements.getTypeElement("com.alamide.rrrandroid.MainAnno").asType());
     * print(wildcardType1);
     * print(wildcardType2);
     *
     * 输出：? extends com.alamide.rrrandroid.MainAnno、? super com.alamide.rrrandroid.MainAnno
     */
    WildcardType getWildcardType(TypeMirror extendsBound, TypeMirror superBound);

}
```

## 3.小例子
利用 `JavaPoet`，生成路由注册类，[JavaPoet 笔记](./java-lib-javapoet.html)
```java
@Override
public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
    messager.printMessage(Diagnostic.Kind.NOTE, "process>>>>>>>>>>>>>>>");
    Set<? extends Element> rootElements = roundEnv.getElementsAnnotatedWith(Route.class);
    TypeMirror activityType = elements.getTypeElement("android.app.Activity").asType();


    for (Element element : rootElements) {
        TypeMirror tm = element.asType();
        if(element.getKind() == ElementKind.CLASS && types.isAssignable(tm, activityType)){
            Route route = element.getAnnotation(Route.class);
            String packageStr = elements.getPackageOf(element).getQualifiedName().toString();

            MethodSpec methodSpec = MethodSpec.methodBuilder("loadInto")
                    .addModifiers(Modifier.PUBLIC)
                    .returns(void.class)
                    .addParameter(ParameterizedTypeName.get(HashMap.class, String.class, String.class), "router")
                    .addStatement("router.put($S, $S)", route.path(), ((TypeElement) element).getQualifiedName())
                    .build();

            TypeSpec typeSpec = TypeSpec.classBuilder(element.getSimpleName() + "_RouteRegister")
                    .addModifiers(Modifier.PUBLIC, Modifier.FINAL)
                    .addMethod(methodSpec)
                    .build();

            JavaFile javaFile = JavaFile.builder(packageStr, typeSpec)
                    .build();

            try {
                javaFile.writeTo(filer);
            } catch (IOException e) {
                e.printStackTrace();
            }

        }
    }
    return true;
}
```

