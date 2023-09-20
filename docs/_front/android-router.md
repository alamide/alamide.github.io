---
layout: post
title: Android 路由
categories: android
tags: android router
date: 2023-06-19
---
Android 组件化的基石，路由。其中主要思路学习自阿里 `Arouter`
<!--more-->

## 1.设计目标
设计的目标，实现不同模块之间在不相互依赖的前提下，可以相互跳转，在跳转的时可以进行一些额外处理，如判断是否登录，未登录需要先跳转到登录页。Activity 页面跳转，一般有显示调用和隐式调用，隐式调用需要使用 `IntentFilter` ，匹配的规则信息需要在 `AndroidManifest.xml` 声明，这种实现方式太繁琐，不够灵活。所以我们采用显式调用，一般调用的方法如下：
```java
Intent intent = new Intent(this, MainActivity.class);
startActivity(intent);
```

因为项目中每个模块是相互独立的，所以彼此之间是不可见的，也即打包前无法获取其它模块的 `Activity` 类。虽然在项目中的模块相互独立，但其实最终还是会打包进同一个 APK 包中，也即在运行时“彼此可见”。所以可以修改一下上面的代码，
```java
Intent intent = new Intent(this, Class.forName("com.alamide.rrrandroid.MainActivity"));
startActivity(intent);
```

这样就可以打包成功了。

## 2.阶段一
所以借着这个思路，我们可以设计一个路由模块，模块的核心功能是保存每个模块 `Activity` 的全类名，再用一个 `Path` 与之绑定（方便分类、管理），设计如下：
```java
public class RouterCenter {

    private static final HashMap<String, String> router = new HashMap<>();

    public static void navigateTo(Context context, String path) {
        try {
            String fullClassName = router.get(path);
            if (fullClassName == null) {
                return;
            }
            Intent intent = new Intent(context, Class.forName(fullClassName));
            context.startActivity(intent);
        } catch (ClassNotFoundException e) {
            throw new RuntimeException(e);
        }
    }
}
```

接下来我们只需在应用启动时，把每个 `Activity` 全类名和对应的 `Path` 存储到 `router` 中就可以了，接下来要解决的问题是这些信息如何存储。

直接自己手写，在工程规模较小时或许可行，但随着工程规模增长时，维护的难度会非常的大，想想在几百个 `.put(xxx, xxx)` 中删除或修改一个指定的 `Activity` 映射信息，那肯定是非常酸爽的。所以手工的方式是不可行的，可以采用注解标记的方式。设计如下注解类：
```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface Route {
    String path() default "";
}
```

在 `Activity` 上加 `Router` 标记
```java
@Route(path = "/app/main")
public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
}
```

在 App 启动时，扫描所有的类，找出带有 `@Route` 注解的类，并获取 `path` 信息，再添加到 `RouterCenter.router` 中，以下 `ClassUtils.getFileNameByPackageName` 的代码，来自于阿里的 `ARouter`，[在这里](https://github.com/alibaba/ARouter/blob/develop/arouter-api/src/main/java/com/alibaba/android/arouter/utils/ClassUtils.java)
```java
 private void findByClass(Context context){
    try {
        Set<String> fileNameByPackageName = ClassUtils.getFileNameByPackageName(context, "com.alamide");

        for (String className : fileNameByPackageName) {
            Class<?> clazz = Class.forName(className);
            Router annotation = clazz.getAnnotation(Router.class);
            if (annotation != null) {
                router.put(annotation.path(), new NavigatorItem(annotation.desc(), annotation.path(), className));
            }
        }
    } catch (PackageManager.NameNotFoundException | InterruptedException | ClassNotFoundException | IOException e) {
        e.printStackTrace();
    }
}
```

`RouterCenter` 优化后代码如下：
```java
public class RouterCenter {

    private static final HashMap<String, String> router = new HashMap<>();

    public static void init(Context context) {
        findByClass(context);
    }

    public static void navigateTo(Context context, String path) {
        try {
            String fullClassName = router.get(path);
            if (fullClassName == null) {
                return;
            }
            Intent intent = new Intent(context, Class.forName(fullClassName));
            context.startActivity(intent);
        } catch (ClassNotFoundException e) {
            throw new RuntimeException(e);
        }
    }

    private static void findByClass(Context context){
        try {
            Set<String> fileNameByPackageName = ClassUtils.getFileNameByPackageName(context, "com.alamide");

            for (String className : fileNameByPackageName) {
                Class<?> clazz = Class.forName(className);
                Router annotation = clazz.getAnnotation(Router.class);
                if (annotation != null) {
                    router.put(annotation.path(), className);
                }
            }
        } catch (PackageManager.NameNotFoundException | InterruptedException | ClassNotFoundException | IOException e) {
            e.printStackTrace();
        }
    }
}
```

以上的代码基本能实现页面之间的跳转，还需要实现是否需要登录
## 3.阶段二
扩展一下 `@Route` ，使其可以多携带一些信息，其中 `isLogin` 使用来设置页面是否需要登录，`desc` 是一些额外的描述信息，为了简洁，在 `RouteCenter` 中新增字段 `isLogin` 来记录是否已登录，扩展后代码。
```java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface Route {
    String path() default "";

    String desc() default "";

    boolean isNeedLogin() default false;
}

public class RouteMeta {
    private String path;
    private String fullClassName;
    private String desc;
    private boolean isNeedLogin;
}

public class RouterCenter {
    private static final HashMap<String, RouteMeta> router = new HashMap<>();

    public static boolean isLogin = false;

    public static void init(Context context) {
        findByClass(context);
    }

    public static void navigateTo(Context context, String path) {
        try {
            RouteMeta routeMeta = router.get(path);
            if (routeMeta == null) {
                return;
            }
            Intent intent;
            if (routeMeta.isNeedLogin() && !isLogin) {
                intent = new Intent(context, Class.forName(router.get("/app/login").getFullClassName()));
                //传入登录后要跳转的地址
                intent.putExtra("nextPath", path);
            } else {
                intent = new Intent(context, Class.forName(routeMeta.getFullClassName()));
            }
            context.startActivity(intent);
        } catch (ClassNotFoundException e) {
            throw new RuntimeException(e);
        }
    }

    private static void findByClass(Context context){
        .......
    }
}

@Route(path = "/app/login")
public class LoginActivity extends AppCompatActivity {

    private Button btnLogin;

    @Override
    protected void onCreate(@Nullable Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_login);

        String nextPath = getIntent().getStringExtra("nextPath");

        btnLogin = findViewById(R.id.btn_login);

        btnLogin.setOnClickListener(v -> {
            Toast.makeText(v.getContext(), "Login Success", Toast.LENGTH_SHORT).show();
            RouterCenter.isLogin = true;
            RouterCenter.navigateTo(LoginActivity.this, nextPath);
            LoginActivity.this.finish();
        });
    }
}
```

以上代码是可以成功实现我们的预期目标的，但是还有些问题。就是扫描所有类是非常消耗资源的，尤其是在项目启动时，可能会造成卡顿。所以需要在优化一下，不要扫描那么多非目标类，只处理标记 `@Route` 的类。

## 4.阶段三
优化的方法是使用 `APT` 处理 `@Route` 标记的类，生成辅助类，例如 对于 `FirstActivity` 会生成 `FirstActivity_RouteRegister` 
```java
@Route(path = "/app/first", desc = "第一个页面")
public class FirstActivity extends AppCompatActivity {
}

//Generated by apt
public final class FirstActivity_RouteRegister {
  public void loadInto(HashMap<String, RouteMeta> router) {
    RouteMeta routeMeta = new RouteMeta("/app/first", "com.alamide.app.FirstActivity", "第一个页面", false);
    router.put("/app/first", routeMeta);
  }
}
```

最后在 `RouterCenter` 中注册
```java
public class RouterCenter {
    ......
    private static void loadMap() {
        ......
        new FirstActivity_RouteRegister().loadInto(router);
        ......
    }

    public static void init(Context context) {
        loadMap();
    }
    ......
}
```

以上的思路是可行的，现在的关键点在于如何插入那么多的 `_RouteRegister` ，肯定不能手工，也不能应用启动时扫描全部类。既然不能启动时扫描，那么可以在打包时扫描呀，这里 `Transform` 的作用就体现出来了，[Transform 的笔记在这里](./android-transform.html)

可以在 `Transform` 中进行处理，找到 `RouterCenter` 和所有的辅助类，最后利用 `ASM` 来插入所需的代码，插入前的代码如下：
```java
private static void loadMap() {
    registerByPlugin = false;
}
```

目标代码
```java
private static void loadMap() {
    registerByPlugin = false;
    new FirstActivity_RouteRegister().loadInto(router);
    new SecondActivity_RouteRegister().loadInto(router);
    new ThirdActivity_RouteRegister().loadInto(router);
    new LoginActivity_RouteRegister().loadInto(router);
    new MainAnno_RouteRegister().loadInto(router);
    new MainActivity_RouteRegister().loadInto(router);
    registerByPlugin = true;
}
```

`Transform` 的转换代码如下：
```java
public class RegisterTransform extends Transform {

    //为 RouterCenter 所在的 Jar 包
    private static File targetFile;
    //被 @Router 标记类生成的辅助类，如：FirstActivity_RouteRegister
    private static List<String> targetClassNames = new ArrayList<>();

    @Override
    public String getName() {
        return "com.alamide.router";
    }

    @Override
    public Set<QualifiedContent.ContentType> getInputTypes() {
        return TransformManager.CONTENT_CLASS;
    }

    @Override
    public Set<? super QualifiedContent.Scope> getScopes() {
        return TransformManager.SCOPE_FULL_PROJECT;
    }

    @Override
    public boolean isIncremental() {
        return false;
    }

    @Override
    public void transform(TransformInvocation transformInvocation) {
        Collection<TransformInput> inputs = transformInvocation.getInputs();
        TransformOutputProvider outputProvider = transformInvocation.getOutputProvider();

        if (!transformInvocation.isIncremental()) {
            try {
                outputProvider.deleteAll();
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }

        System.out.println("===============>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>");
        inputs.forEach(v -> {

            v.getJarInputs().forEach(jarInput -> {
                try (JarFile jarFile = new JarFile(jarInput.getFile())) {
                    File outputProviderContentLocation = outputProvider.getContentLocation(jarInput.getName(), jarInput.getContentTypes(), jarInput.getScopes(), Format.JAR);

                    jarFile.stream().forEach(jarEntry -> {
                        String name = jarEntry.getName();
                        if (name.startsWith("com.alamide.android.router".replace(".", "/"))) {
                            System.out.println(name);
                            if (name.contains("_RouteRegister")) {
                                try (InputStream inputStream = jarFile.getInputStream(jarEntry)) {
                                    readRouterClassNames(inputStream);
                                } catch (IOException e) {
                                    throw new RuntimeException(e);
                                }
                            }
                        }

                        if (name.contains("com/alamide/router/RouterCenter.class")) {
                            targetFile = outputProviderContentLocation;
                        }
                    });


                    FileUtils.copyFile(jarInput.getFile(), outputProviderContentLocation);
                } catch (IOException e) {
                    throw new RuntimeException(e);
                }
            });

            v.getDirectoryInputs().forEach(directoryInput -> {
                try {
                    File outputProviderContentLocation = outputProvider.getContentLocation(directoryInput.getName(), directoryInput.getContentTypes(), directoryInput.getScopes(), Format.DIRECTORY);
                    readTargetFile(directoryInput);
                    readRouterClassNames(directoryInput);
                    FileUtils.copyDirectory(directoryInput.getFile(), outputProviderContentLocation);
                } catch (IOException e) {
                    throw new RuntimeException(e);
                }
            });
        });


        System.out.println("===============>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>");
        System.out.println(targetClassNames);
        System.out.println(targetFile);

        //开始插入代码
        if (targetFile != null) {
            File optJar = new File(targetFile.getParent(), targetFile.getName() + ".opt");
            if (optJar.exists()) {
                optJar.delete();
            }

            try (JarFile jarFile = new JarFile(targetFile); JarOutputStream jarOutputStream = new JarOutputStream(Files.newOutputStream(optJar.toPath()));) {
                Enumeration<JarEntry> entries = jarFile.entries();
                while (entries.hasMoreElements()) {
                    JarEntry jarEntry = entries.nextElement();
                    String entryName = jarEntry.getName();
                    jarOutputStream.putNextEntry(new ZipEntry(entryName));
                    if ("com/alamide/router/RouterCenter.class".equals(entryName)) {
                        byte[] bytes = insertCode(jarFile.getInputStream(jarEntry));
                        jarOutputStream.write(bytes);
                    } else {
                        jarOutputStream.write(IOUtils.toByteArray(jarFile.getInputStream(jarEntry)));
                    }
                }

                optJar.renameTo(targetFile);
            } catch (IOException e) {
                throw new RuntimeException(e);
            }
        }


    }

    private byte[] insertCode(InputStream inputStream) throws IOException {
        ClassReader cr = new ClassReader(inputStream);

        ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);
        ClassVisitor cv = new MethodExitVisitor(Opcodes.ASM5, cw);

        cr.accept(cv, ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES);
        return cw.toByteArray();

    }

    private static class MethodExitVisitor extends ClassVisitor {

        private static String className;

        public MethodExitVisitor(int api, ClassVisitor classVisitor) {
            super(api, classVisitor);
        }

        @Override
        public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
            super.visit(version, access, name, signature, superName, interfaces);
            className = name;
        }

        @Override
        public MethodVisitor visitMethod(int access, String name, String descriptor, String signature, String[] exceptions) {
            MethodVisitor mv = super.visitMethod(access, name, descriptor, signature, exceptions);

            if (mv != null && name.equals("loadMap")) {
                mv = new MethodAdapter(api, mv);
            }

            return mv;
        }

        private static class MethodAdapter extends MethodVisitor {
            public MethodAdapter(int api, MethodVisitor methodVisitor) {
                super(api, methodVisitor);
            }

            @Override
            public void visitInsn(int opcode) {

                if (opcode >= Opcodes.IRETURN && opcode <= Opcodes.RETURN) {
                    for (String clazz : targetClassNames) {
                        super.visitTypeInsn(Opcodes.NEW, clazz);
                        super.visitInsn(Opcodes.DUP);
                        super.visitMethodInsn(Opcodes.INVOKESPECIAL, clazz, "<init>", "()V", false);
                        super.visitFieldInsn(Opcodes.GETSTATIC, className, "router", "Ljava/util/HashMap;");
                        super.visitMethodInsn(Opcodes.INVOKEVIRTUAL, clazz, "loadInto", "(Ljava/util/HashMap;)V", false);
                    }

                    super.visitInsn(Opcodes.ICONST_1);
                    super.visitFieldInsn(Opcodes.PUTSTATIC, className, "registerByPlugin", "Z");
                }
                super.visitInsn(opcode);
            }
        }
    }


    private void readTargetFile(DirectoryInput directoryInput) {
        FileUtils.listFiles(directoryInput.getFile(), null, true).forEach(file -> {
            if (file.getName().contains("RouterCenter")) {
                targetFile = file;
            }
        });
    }

    private void readRouterClassNames(DirectoryInput directoryInput) {
        FileUtils.listFiles(directoryInput.getFile(), null, true).forEach(file -> {
            try (FileInputStream inputStream = new FileInputStream(file)) {
                readRouterClassNames(inputStream);
            } catch (IOException e) {
                throw new RuntimeException();
            }
        });
    }

    private void readRouterClassNames(InputStream inputStream) throws IOException {

        ClassReader cr = new ClassReader(inputStream);

        ClassWriter cw = new ClassWriter(ClassWriter.COMPUTE_FRAMES);

        ClassVisitor cv = new ClassVisitor(Opcodes.ASM5, cw) {
            @Override
            public void visit(int version, int access, String name, String signature, String superName, String[] interfaces) {
                super.visit(version, access, name, signature, superName, interfaces);
                if (name.endsWith("_RouteRegister")) {
                    targetClassNames.add(name);
                }
            }
        };

        cr.accept(cv, ClassReader.SKIP_DEBUG | ClassReader.SKIP_FRAMES);
    }
}
```

这样就可以实现我们的目标了，其中使用 `Transform + ASM` 实现代码插桩，是一个很好的思路。