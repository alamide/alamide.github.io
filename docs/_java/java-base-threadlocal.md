---
layout: post
title: Java ThreadLocal
categories: java
tags: Java Thread
date: 2023-07-03
---
Java ThreadLocal 的使用，及实现原理，本次学习目标：
1. ThreadLocal 如何使用

2. ThreadLocal 是如何实现同一个线程中数据存储、获取的
<!--more-->

## 1.如何使用
ThreadLocal 可以用于传递线程执行的上下文信息，可以避免上下文参数传递的痛点。先通过一个小示例来展示  ThreadLocal 的基本使用
```java
@Data
@AllConstructorArgs
@ToString
public class UserInfo {
    private String name;
    private int age;
}

public class BaseContext {
    private static final ThreadLocal<String> t_name = new ThreadLocal<>();

    private static final ThreadLocal<UserInfo> t_user_info = new ThreadLocal<>();

    private static final ThreadLocal<String> t_d_value = ThreadLocal.withInitial(() -> "defaultValue");

    public static void putThreadName(String name){
        t_name.set(name);
    }

    public static String getThreadName(){
        return t_name.get();
    }

    public static void setUserInfo(UserInfo userInfo){
        t_user_info.set(userInfo);
    }

    public static UserInfo getUserInfo(){
        return t_user_info.get();
    }

    public static String getDefaultValue(){
        return t_d_value.get();
    }

    public static void setDefaultValue(String value){
        t_d_value.set(value);
    }
}

public class MainThread {
    public static void main(String[] args) {
        final ExecutorService executorService = Executors.newFixedThreadPool(3);
        for (int i = 0; i < 3; i++) {
            int finalI = i;
            executorService.execute(() -> {
                BaseContext.putThreadName("Thread-name " + finalI);
                BaseContext.setUserInfo(new UserInfo("name0" + finalI, 18 + finalI));
                
                if(finalI % 2 == 1){
                    BaseContext.setDefaultValue("Not default");
                }

                step1();
            });
        }
    }

    public static void step1() {
        step2();
    }

    public static void step2() {
        step3();
    }

    public static void step3() {
        final UserInfo userInfo = BaseContext.getUserInfo();
        System.out.println(BaseContext.getThreadName() + ": " + userInfo);
    }
}
```

output:
```
Thread-name 1: UserInfo{name='name01', age=19}, Not default
Thread-name 2: UserInfo{name='name02', age=20}, defaultValue
Thread-name 0: UserInfo{name='name00', age=18}, defaultValue
```

这样可以实现进程上下文之间代码的传递，使用的场景有很多，JavaWeb 中可以使用 ThreadLocal 来传递 user_id，在 Filter 中验证获取、储存，供给后续使用。以及在 Android 的 Handler 通信机制中，用来存储 Looper。
## 2.实现原理
下面就来看看 ThreadLocal 如何实现数据在线程的上下文传递的，首先要知道的是：存储使用 ThreadLocal.set()，获取使用 ThreadLocal.get()，先来看看 set()

```java
public class ThreadLocal<T> {

    ThreadLocal.ThreadLocalMap threadLocals = null;

    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            map.set(this, value);
        } else {
            createMap(t, value);
        }
    }

    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
}
```

`Thread.currentThread()` 是一个 `native` 方法，即获取当前线程对象，代码的执行是在线程中执行的，所以每一段执行的代码必依附于一个线程。

刚开始 `map=null`，所以需要创建 `ThreadLocalMap`，创建的 `Map` 对象存储在当前线程的 `threadLocals` 成员变量中，来看一下 `ThreadLocalMap` 的具体代码

```java
static class ThreadLocalMap {

    static class Entry extends WeakReference<ThreadLocal<?>> {
        /** The value associated with this ThreadLocal. */
        Object value;

        Entry(ThreadLocal<?> k, Object v) {
            super(k);
            value = v;
        }
    }

    private Entry[] table;
    private static final int INITIAL_CAPACITY = 16;

    ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
        table = new Entry[INITIAL_CAPACITY];
        int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
        table[i] = new Entry(firstKey, firstValue);
        size = 1;
        setThreshold(INITIAL_CAPACITY);
    }
}
```

`ThreadLocalMap` 中使用 `Entry[]` 来存储数据，初始化大小为 16，到这里可以知道，`ThreadLocal` 中存储的数据最终是存储到当前 `Thread` 的 `threadLocals` 中，每个线程都有自己的 `threadLocals` ，当然没有存储数据时为 `null`。`threadLocals` 的数据结构为 `Entry[]`，`Entry` 的 `key` 为 `ThreadLocal`，`value` 为存储的具体数据，可以存储任意类型。

再来看看具体 `map != null` 时，数据如何存储的，
```java
public class ThreadLocal<T> {
    public void set(T value) {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            map.set(this, value);
        } else {
            createMap(t, value);
        }
    }

    static class ThreadLocalMap {
        private void set(ThreadLocal<?> key, Object value) {

            Entry[] tab = table;
            int len = tab.length;
            int i = key.threadLocalHashCode & (len-1);

            //来查看是否已经存储过 key 
            for (Entry e = tab[i];
                 e != null;
                 //如果被占用就向后移一位
                 e = tab[i = nextIndex(i, len)]) {
                //存储过，替换数据
                if (e.refersTo(key)) {
                    e.value = value;
                    return;
                }
                //当前 Entry 存在，但是 key == null，可能是由于内存紧张，被回收？
                if (e.refersTo(null)) {
                    replaceStaleEntry(key, value, i);
                    return;
                }
            }

            //不存在，直接存储
            tab[i] = new Entry(key, value);
            int sz = ++size;
            //调整数组大小等
            if (!cleanSomeSlots(i, sz) && sz >= threshold)
                rehash();
        }
        //后移一位
        private static int nextIndex(int i, int len) {
            return ((i + 1 < len) ? i + 1 : 0);
        }
    }
}
```

存储数据时，需要先查看是否已经存储过当前 `ThreadLocal`，如果存储过就需要替换数据。如果当前存储的位置已被占用，就向后移一位。

再看看数据是如何获取的，
```java
public class ThreadLocal<T> {
    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null) {
                @SuppressWarnings("unchecked")
                T result = (T)e.value;
                return result;
            }
        }
        //初次获取时，如果存储过数据，则使用 initialValue 来初始化
        return setInitialValue();
    }

    private T setInitialValue() {
        T value = initialValue();
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            map.set(this, value);
        } else {
            createMap(t, value);
        }
        if (this instanceof TerminatingThreadLocal) {
            TerminatingThreadLocal.register((TerminatingThreadLocal<?>) this);
        }
        return value;
    }

    static class ThreadLocalMap {
        private Entry getEntry(ThreadLocal<?> key) {
            int i = key.threadLocalHashCode & (table.length - 1);
            Entry e = table[i];
            //查看是否直接命中
            if (e != null && e.refersTo(key))
                return e;
            else
                //继续向后查询，可能会有 hash 碰撞，所以存储时会后移
                return getEntryAfterMiss(key, i, e);
        }
    }
}
```

获取数据时直接依据当前 `ThreadLocal` 来获取。

每一个线程都有自己的成员变量 `threadLocals`，可以用来存储一些需要线程上下文传递的数据，实现的原理如下：
1. 获取当前执行代码的线程

2. 获取成员变量 `threadLocals` ，其是 `ThreadLocalMap` 的一个具体实例，数据结构为 `Entry[]`，其中 `Entry` 的 `key` 为 `ThreadLocal`，`value` 为具体存储的数据

3. 存储数据 or 获取数据


ThreadLocal 是一个辅助类，其并不负责存储数据。

线程是通过 `ThreadLocal` 来存储数据的，一个 `ThreadLocal` 只能存储一个数据（对象），要存储多个数据时，需要使用多个 `ThreadLocal`。

