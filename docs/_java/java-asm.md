---
layout: post
title: Java 字节码操作 ASM
categories: java
tags: Java ASM
date: 2023-06-13
---
ASM 操作 Java 字节码的类库。
<!--more-->

官方地址，[在这里](https://asm.ow2.io/index.html)

学习的 Blog，[在这里](https://lsieun.github.io/java/asm/java-asm-season-01.html)

## 1.引入依赖
```xml
<dependency>
    <groupId>org.ow2.asm</groupId>
    <artifactId>asm</artifactId>
    <version>9.5</version>
</dependency>
<dependency>
    <groupId>org.ow2.asm</groupId>
    <artifactId>asm-util</artifactId>
    <version>9.5</version>
</dependency>
```

## 2.生成类
生成类的 ASM 代码可以使用 ASM 提供的工具类来辅助生成，也可以使用 IDEA 插件 `ASM ByteCode Viewer`

字段描述符
<table>
<tr><th>Java 类型</th><th>ClassFile 描述符</th></tr>
<tr><td>boolean</td><td>Z</td></tr>
<tr><td>byte</td><td>B</td></tr>
<tr><td>char</td><td>C</td></tr>
<tr><td>short</td><td>S</td></tr>
<tr><td>int</td><td>I</td></tr>
<tr><td>float</td><td>F</td></tr>
<tr><td>long</td><td>J</td></tr>
<tr><td>double</td><td>D</td></tr>
<tr><td>void</td><td>V</td></tr>
<tr><td>non-array reference</td><td>L&lt;InternetName&gt;;</td></tr>
<tr><td>array reference</td><td>[</td></tr>
</table>

方法描述符举例
1. int add(int a, int b): (II)I

2. void test(int a, int b): (II)V

3. boolean compare(Object o): (Ljava/lang/Object;)Z

4. void main(String[] args): ([Ljava/lang/String;)V

### 2.1 生成一个空接口
生成的目标类如下
```java
public interface HelloWorld{
}
```

```java
private static byte[] dump(){
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    cw.visit(
            Opcodes.V1_8,
            Opcodes.ACC_PUBLIC | Opcodes.ACC_ABSTRACT | Opcodes.ACC_INTERFACE,
            "com/alamide/asm/HelloWorld",
            null,
            "java/lang/Object",
            null);

    cw.visitEnd();

    return cw.toByteArray();
}
```

### 2.2 生成字段 + 字段 + 方法
生成目标类如下
```java
public interface HelloWorld extends Serializable {
    int N = 1;
    String TAG = "tag";
    double D = 1.0;
    void sayHello();
    String saySome(String some);
}
```

```java
private static byte[] dump() {
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    cw.visit(
            Opcodes.V1_8,
            Opcodes.ACC_PUBLIC | Opcodes.ACC_ABSTRACT | Opcodes.ACC_INTERFACE,
            "com/alamide/asm/HelloWorld",
            null,
            "java/lang/Object",
            new String[]{"java/io/Serializable"});

    {
        final FieldVisitor fv1 = cw.visitField(
                Opcodes.ACC_PUBLIC | Opcodes.ACC_STATIC | Opcodes.ACC_FINAL,
                "N",
                "I",
                null,
                1
        );

        fv1.visitEnd();
    }

    {
        FieldVisitor fv2 = cw.visitField(
                Opcodes.ACC_PUBLIC | Opcodes.ACC_STATIC | Opcodes.ACC_FINAL,
                "TAG",
                "Ljava/lang/String;",
                null,
                "tag"
        );

        fv2.visitEnd();
    }

    {
        FieldVisitor fv3 = cw.visitField(
                Opcodes.ACC_PUBLIC | Opcodes.ACC_STATIC | Opcodes.ACC_FINAL,
                "D",
                "D",
                null,
                1.0
        );
        fv3.visitEnd();
    }

    {
        final MethodVisitor mv1 = cw.visitMethod(
                Opcodes.ACC_PUBLIC | Opcodes.ACC_ABSTRACT,
                "saySome",
                "(Ljava/lang/String;)Ljava/lang/String;",
                null,
                null
        );
        mv1.visitEnd();
    }

    {
        final MethodVisitor mv2 = cw.visitMethod(
                Opcodes.ACC_PUBLIC | Opcodes.ACC_ABSTRACT,
                "sayHello",
                "()V",
                null,
                null
        );
        mv2.visitEnd();
    }


    cw.visitEnd();

    return cw.toByteArray();
}
```

### 2.3 生成类
目标类
```java
public class HelloWorld {
}
```

```java
private static byte[] dump() {
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    cw.visit(
            Opcodes.V1_8,
            Opcodes.ACC_PUBLIC | Opcodes.ACC_SUPER,
            "com/alamide/asm/HelloWorld",
            null,
            "java/lang/Object",
            null);
    {
        final MethodVisitor mv1 = cw.visitMethod(
                Opcodes.ACC_PUBLIC,
                "<init>",
                "()V",
                null,
                null
        );

        mv1.visitCode();
        mv1.visitVarInsn(Opcodes.ALOAD, 0);
        mv1.visitMethodInsn(Opcodes.INVOKESPECIAL, "java/lang/Object", "<init>", "()V", false);
        mv1.visitInsn(Opcodes.RETURN);
        mv1.visitMaxs(1, 1);
        mv1.visitEnd();
    }
    cw.visitEnd();
    
    return cw.toByteArray();
}
```

### 2.4 生成的字段加注解
目标类
```java
@Retention(RetentionPolicy.CLASS)
@Target({ElementType.FIELD})
public @interface Version {
    double value();
}
```

```java
private static byte[] dump() {
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

    cw.visit(
            Opcodes.V1_8,
            Opcodes.ACC_PUBLIC | Opcodes.ACC_INTERFACE | Opcodes.ACC_ABSTRACT,
            "com/alamide/asm/HelloWorld",
            null,
            "java/lang/Object",
            null);

    {
        final FieldVisitor fv1 = cw.visitField(
                Opcodes.ACC_PUBLIC | Opcodes.ACC_STATIC | Opcodes.ACC_FINAL,
                "TAG",
                "Ljava/lang/String;",
                null,
                "tag"
        );

        {
            final AnnotationVisitor av1 = fv1.visitAnnotation("Lcom/alamide/asm/Version;", false);
            av1.visit("value", 1.0);
            av1.visitEnd();
        }

        fv1.visitEnd();
    }
    
    cw.visitEnd();

    return cw.toByteArray();
}
```

## 3.修改类
### 3.1 修改类的版本
```java
private static byte[] dump() throws IOException {
    byte[] readBytes = Files.readAllBytes(Path.of("./target/classes/com/alamide/asm/HelloWorld.class"));

    ClassReader cr = new ClassReader(readBytes);
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    int api = Opcodes.ASM9;
    ClassVisitor cv = new ClassVisitor(api, cw) {
        @Override
        public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
            super.visit(Opcodes.V1_7, access, name, signature, superName, interfaces);
        }
    };

    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    return cw.toByteArray();
}
```

### 3.2 修改类的接口
目标类没有实现 Serializable，通过 ASM 使其实现这个接口
```java
private static byte[] dump() throws IOException {
    byte[] readBytes = Files.readAllBytes(Path.of("./target/classes/com/alamide/asm/HelloWorld.class"));

    ClassReader cr = new ClassReader(readBytes);
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    int api = Opcodes.ASM9;
    ClassVisitor cv = new ClassVisitor(api, cw) {
        @Override
        public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
            super.visit(version, access, name, signature, superName, new String[]{"java/io/Serializable"});
        }
    };

    int parsingOptions = ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES;
    cr.accept(cv, parsingOptions);

    return cw.toByteArray();
}
```

### 3.3 删除字段
被操作类
```java
public class HelloWorld {
    private String hello;
    private static final String TAG = "tag";
}
```

```java
private static byte[] dump() throws IOException {
    byte[] readBytes = Files.readAllBytes(Path.of("./target/classes/com/alamide/asm/HelloWorld.class"));

    ClassReader cr = new ClassReader(readBytes);
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    int api = Opcodes.ASM9;

    ClassVisitor cv1 = new ClassRemoveFieldVisitor(api, cw, "hello", "Ljava/lang/String;");

    ClassVisitor cv2 = new ClassRemoveFieldVisitor(api, cv1, "TAG", "Ljava/lang/String;");
    cr.accept(cv2, ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES);

    return cw.toByteArray();
}

private static class ClassRemoveFieldVisitor extends ClassVisitor {
    private final String fieldName;
    private final String fieldDescriptor;

    protected ClassRemoveFieldVisitor(int api, ClassVisitor classVisitor, String fieldName, String fieldDescriptor) {
        super(api, classVisitor);
        this.fieldName = fieldName;
        this.fieldDescriptor = fieldDescriptor;
    }

    @Override
    public FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value) {
        if (name.equals(fieldName) && descriptor.equals(fieldDescriptor)) {
            return null;
        }
        return super.visitField(access, name, descriptor, signature, value);
    }
}
```

### 3.4 添加字段
```java
private static byte[] dump() throws IOException {
    byte[] readBytes = Files.readAllBytes(Path.of("./target/classes/com/alamide/asm/HelloWorld.class"));

    ClassReader cr = new ClassReader(readBytes);
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    int api = Opcodes.ASM9;

    ClassVisitor cv = new ClassAddFieldVisitor(api, cw, Opcodes.ACC_PRIVATE, "newField", "I");

    cr.accept(cv, ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES);

    return cw.toByteArray();
}

public class ClassAddFieldVisitor extends ClassVisitor {
    private final int fieldAccess;
    private final String fieldName;
    private final String fieldDescriptor;

    private boolean isPresent;

    protected ClassAddFieldVisitor(int api, ClassVisitor classVisitor, int fieldAccess, String fieldName, String fieldDescriptor) {
        super(api, classVisitor);
        this.fieldAccess = fieldAccess;
        this.fieldName = fieldName;
        this.fieldDescriptor = fieldDescriptor;
    }

    @Override
    public FieldVisitor visitField(int access, String name, String descriptor, String signature, Object value) {
        if(name.equals(fieldName) && descriptor.equals(fieldDescriptor)){
            isPresent = true;
        }
        return super.visitField(access, name, descriptor, signature, value);
    }

    @Override
    public void visitEnd() {
        if(!isPresent){
            final FieldVisitor fv = super.visitField(fieldAccess, fieldName, fieldDescriptor, null, null);
            if(fv != null){
                fv.visitEnd();
            }
        }
        super.visitEnd();
    }
}
```

### 3.5 删除方法
操作类
```java
public class HelloWorld {
    public void saySomething(String something){
        System.out.println(something);
    }
}
```

```java
 private static byte[] dump() throws IOException {
    byte[] readBytes = Files.readAllBytes(Path.of("./target/classes/com/alamide/asm/HelloWorld.class"));

    ClassReader cr = new ClassReader(readBytes);
    ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
    int api = Opcodes.ASM9;

    ClassVisitor cv = new ClassRemoveMethodVisitor(api, cw, "saySomething", "(Ljava/lang/String;)V");

    cr.accept(cv, ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES);

    return cw.toByteArray();
}

public class ClassRemoveMethodVisitor extends ClassVisitor {
    private final String methodName;
    private final String methodDescriptor;

    protected ClassRemoveMethodVisitor(int api, ClassVisitor classVisitor, String methodName, String methodDescriptor) {
        super(api, classVisitor);
        this.methodName = methodName;
        this.methodDescriptor = methodDescriptor;
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
        if (name.equals(methodName) && descriptor.equals(methodDescriptor)) {
            return null;
        }
        return super.visitMethod(access, name, descriptor, signature, exceptions);
    }
}
```

### 3.6 添加方法
可以使用插件来辅助生成
```java
public class HelloWorld {
    public HelloWorld() {
    }

    //新增
    public int add(int var1, int var2) {
        return var1 + var2;
    }
}
```

```java
public abstract class ClassAddMethodVisitor extends ClassVisitor {
    private boolean isPresent;
    private final int methodAccess;
    private final String methodName;
    private final String methodDescriptor;
    private final String methodSignature;
    private final String[] methodExceptions;

    protected ClassAddMethodVisitor(int api, ClassVisitor classVisitor,
                                    int methodAccess, String methodName, String methodDescriptor,
                                    String methodSignature, String[] methodExceptions) {
        super(api, classVisitor);
        this.methodAccess = methodAccess;
        this.methodName = methodName;
        this.methodDescriptor = methodDescriptor;
        this.methodSignature = methodSignature;
        this.methodExceptions = methodExceptions;
    }


    @Override
    public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
        if(name.equals(methodName) && descriptor.equals(methodDescriptor)){
            isPresent = true;
        }
        return super.visitMethod(access, name, descriptor, signature, exceptions);
    }

    @Override
    public void visitEnd() {
        if(!isPresent){
            final MethodVisitor mv = super.visitMethod(methodAccess, methodName, methodDescriptor, methodSignature, methodExceptions);
            if(mv != null){
                generateMethodBody(mv);
            }
        }
        super.visitEnd();
    }

    public abstract void generateMethodBody(MethodVisitor mv);
}


ClassVisitor cv = new ClassAddMethodVisitor(api, cw, Opcodes.ACC_PUBLIC, "add", "(II)I", null, null) {
    @Override
    public void generateMethodBody(MethodVisitor mv) {
        mv.visitCode();
        mv.visitVarInsn(Opcodes.ILOAD, 1);
        mv.visitVarInsn(Opcodes.ILOAD, 2);
        mv.visitInsn(Opcodes.IADD);
        mv.visitInsn(Opcodes.IRETURN);
        mv.visitMaxs(2, 3);
        mv.visitEnd();
    }
};
```

## 4.修改已有的方法
给方法新增 System.out.println("")，代码可以用插件辅助生成

方法开始前添加
```java
public class MethodEnterVisitor extends ClassVisitor {
    protected MethodEnterVisitor(int api, ClassVisitor classVisitor) {
        super(api, classVisitor);
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
        MethodVisitor mv =  super.visitMethod(access, name, descriptor, signature, exceptions);
        if(mv != null && !"<init>".equals(name)){
            mv = new MethodEnterAdapter(api, mv);
        }
        return mv;
    }

    private static class MethodEnterAdapter extends MethodVisitor {

        protected MethodEnterAdapter(int api, MethodVisitor methodVisitor) {
            super(api, methodVisitor);
        }

        @Override
        public void visitCode() {
            super.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
            super.visitLdcInsn("Method Begin!");
            super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
            super.visitCode();
        }
    }
}
```

方法结束前添加
```java
public class MethodExitVisitor extends ClassVisitor {
    protected MethodExitVisitor(int api, ClassVisitor classVisitor) {
        super(api, classVisitor);
    }

    @Override
    public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
        MethodVisitor mv =  super.visitMethod(access, name, descriptor, signature, exceptions);
        if(mv != null && !"<init>".equals(name)){
            mv = new MethodExitAdapter(api, mv);
        }
        return mv;
    }

    private static class MethodExitAdapter extends MethodVisitor {

        protected MethodExitAdapter(int api, MethodVisitor methodVisitor) {
            super(api, methodVisitor);
        }

        @Override
        public void visitInsn(int opcode) {
            if(opcode == Opcodes.ATHROW || (opcode >= Opcodes.IRETURN && opcode <= Opcodes.RETURN)){
                super.visitFieldInsn(Opcodes.GETSTATIC, "java/lang/System", "out", "Ljava/io/PrintStream;");
                super.visitLdcInsn("Method Exit!");
                super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, "java/io/PrintStream", "println", "(Ljava/lang/String;)V", false);
            }
            super.visitInsn(opcode);
        }
    }
}
```