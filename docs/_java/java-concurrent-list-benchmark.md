---
layout: post
title: Vector VS Collections.synchronizedList VS CopyOnWriteArrayList
categories: java
tags: Java Stream
date: 2023-05-30
---
Java 线程安全有序集合性能测试。
<!--more-->

本次测试参赛选手：
1. ArrayList

2. Vector

3. Collections.synchronizedList

4. CopyOnWriteArrayList

5. DiyConcurrentList（自定义线程安全容器，读写锁实现）

## 1.Read 测试
只读取数据，测试参数：`8` 线程，每个线程读取 `1000_0000` 次，初始容器含有 `10000` 条数据。测试结果如下：
```
Benchmark                                            Mode  Cnt      Score      Error  Units
ConcurrentListBenchMark.collectionsSynchronizedRead  avgt   10   9448.708 ±  863.695  ms/op
ConcurrentListBenchMark.copyOnWriteArrayListRead     avgt   10    156.831 ±    7.969  ms/op
ConcurrentListBenchMark.diyConcurrentListRead        avgt   10  16145.034 ± 1162.401  ms/op
ConcurrentListBenchMark.noLockRead                   avgt   10    156.482 ±   10.386  ms/op
ConcurrentListBenchMark.vectorRead                   avgt   10   9970.584 ±  795.218  ms/op
```

可以看到，CopyOnWriteArrayList 的读取性能非常优越，与非线程安全容器 ArrayList 的差距可以忽略不计。Vector 与 Collections.synchronizedList 属于同一级别，Collections.synchronizedList 小优。自定义 List 最差，这一点有点吃惊，毕竟书上介绍，读锁可以并发。。。后又想了一下，也在网上搜了案例，应该本场景操作时间太短，所以发挥不出读写锁的优势。

## 2.Write 测试
本测试参赛选手将 ArrayList 剔除，因为会发生线程安全问题。测试参数：`4` 线程，初始容器含有 `10000` 条数据。，本系列测试，将会修改每个线程操作次数参数。因为 CopyOnWriteArrayList 采用 Copy-Write 的方式写数据，所以性能是最差的。
### 2.1 每个线程插入 1000 条数据，合计 4000
```
Benchmark                                             Mode  Cnt   Score   Error  Units
ConcurrentListBenchMark.collectionsSynchronizedWrite  avgt   10   0.806 ± 0.075  ms/op
ConcurrentListBenchMark.copyOnWriteArrayListWrite     avgt   10  30.687 ± 2.703  ms/op
ConcurrentListBenchMark.diyConcurrentListWrite        avgt   10   0.640 ± 0.085  ms/op
ConcurrentListBenchMark.vectorWrite                   avgt   10   0.734 ± 0.347  ms/op
```

在这个量级可以看到，CopyOnWriteArrayList 性能最差，DiyConcurrentListWrite 最优
### 2.2 每个线程插入 10000 条数据，合计 40000
```
Benchmark                                             Mode  Cnt    Score    Error  Units
ConcurrentListBenchMark.collectionsSynchronizedWrite  avgt   10    4.508 ±  1.350  ms/op
ConcurrentListBenchMark.copyOnWriteArrayListWrite     avgt   10  663.622 ± 49.699  ms/op
ConcurrentListBenchMark.diyConcurrentListWrite        avgt   10    2.309 ±  0.175  ms/op
ConcurrentListBenchMark.vectorWrite                   avgt   10    3.764 ±  0.249  ms/op
```

### 2.3 每个线程插入 100000 条数据，合计 40_0000，由于 CopyOnWriteArrayList 性能下降太快，剔除
```
Benchmark                                             Mode  Cnt   Score    Error  Units
ConcurrentListBenchMark.collectionsSynchronizedWrite  avgt   10  44.401 ± 13.473  ms/op
ConcurrentListBenchMark.diyConcurrentListWrite        avgt   10  22.595 ±  0.626  ms/op
ConcurrentListBenchMark.vectorWrite                   avgt   10  42.482 ±  2.753  ms/op
```

### 2.4 每个线程插入 100_0000 条数据，合计 400_0000
```
Benchmark                                             Mode  Cnt    Score     Error  Units
ConcurrentListBenchMark.collectionsSynchronizedWrite  avgt   10  446.667 ± 114.173  ms/op
ConcurrentListBenchMark.diyConcurrentListWrite        avgt   10  244.116 ±  10.168  ms/op
ConcurrentListBenchMark.vectorWrite                   avgt   10  407.695 ±  48.510  ms/op
```

上面的几个测试可以得知，读写锁的写锁最优
## 3.Read And Write 测试
8_0000:8_0000 左边是读操作总数，右边是写操作总数
### 3.1 8_0000:8_0000
```
Benchmark                                        Mode  Cnt     Score     Error  Units
ConcurrentListBenchMark.collectionsSynchronized  avgt   10    18.312 ±   2.714  ms/op
ConcurrentListBenchMark.copyOnWriteArrayList     avgt   10  2027.385 ± 108.147  ms/op
ConcurrentListBenchMark.diyConcurrentList        avgt   10    20.053 ±   0.653  ms/op
ConcurrentListBenchMark.vector                   avgt   10    18.120 ±   0.630  ms/op
```

### 3.2 8_0000:80_0000
```
Benchmark                                        Mode  Cnt     Score     Error  Units
ConcurrentListBenchMark.collectionsSynchronized  avgt   10    94.961 ±  13.850  ms/op
ConcurrentListBenchMark.copyOnWriteArrayList     avgt   10  2050.149 ± 164.038  ms/op
ConcurrentListBenchMark.diyConcurrentList        avgt   10   165.166 ±   6.744  ms/op
ConcurrentListBenchMark.vector                   avgt   10   101.591 ±   3.040  ms/op
```

### 3.3 8_0000:800_0000
```
Benchmark                                        Mode  Cnt     Score     Error  Units
ConcurrentListBenchMark.collectionsSynchronized  avgt   10   978.502 ±  77.386  ms/op
ConcurrentListBenchMark.copyOnWriteArrayList     avgt   10  2033.593 ± 137.201  ms/op
ConcurrentListBenchMark.diyConcurrentList        avgt   10  1540.791 ±  95.221  ms/op
ConcurrentListBenchMark.vector                   avgt   10   996.089 ±  76.670  ms/op
```

### 3.4 8_0000:8000_0000
```
Benchmark                                        Mode  Cnt      Score      Error  Units
ConcurrentListBenchMark.collectionsSynchronized  avgt   10   9553.112 ±  736.881  ms/op
ConcurrentListBenchMark.copyOnWriteArrayList     avgt   10   1995.871 ±  112.006  ms/op
ConcurrentListBenchMark.diyConcurrentList        avgt   10  15362.669 ±  907.706  ms/op
ConcurrentListBenchMark.vector                   avgt   10  10331.707 ± 1204.202  ms/op
```

### 3.5 800_0000:8_0000
```
Benchmark                                        Mode  Cnt    Score     Error  Units
ConcurrentListBenchMark.collectionsSynchronized  avgt   10  999.705 ± 191.238  ms/op
ConcurrentListBenchMark.diyConcurrentList        avgt   10  514.118 ±  14.622  ms/op
ConcurrentListBenchMark.vector                   avgt   10  967.352 ±  78.785  ms/op
```

### 3.6 小结
1. CopyOnWriteArrayList 在读次数远远多于写次数时，优势较大

2. DiyConcurrentList 在写次数远远多于读次数时，优势较大

## 4.小结
1. CopyOnWriteArrayList 在读取数据时，性能与读取 ArrayList 一致，但是在写数据时由于是采用 Copy-Write 的模式，所以写性能较差，所以适合写少读多的需求。由于采用 Copy-Write 所以在数据量巨大的时候，写性能很糟糕。如列表中有 10亿数据，那么每写一条数据，都要耗费巨多资源，所以使用的时候还要仔细考量。

2. Vector 与 Collections.synchronizedList 性能几乎持平，且在读写量增多时，耗费时间线性增长，属于中庸的选择。Collections.synchronizedList 的优点是可以用于 LinkedList ，Vector 做不到，使用 Collections.synchronizedList 会让代码更灵活。

3. 读写锁实现的自定义容器，写性能占优，适用于写多读少的场景。

4. 在没有确定把握的情况下请使用 Collections.synchronizedList 。

## 5.测试的代码
`DiyConcurrentList.java`
```java
public class DiyConcurrentList<E> extends ArrayList<E> {
    private ReentrantReadWriteLock lock = new ReentrantReadWriteLock();
    private final Lock readLock = lock.readLock();
    private final Lock writeLock = lock.writeLock();

    public DiyConcurrentList() {
    }

    public DiyConcurrentList(Collection<? extends E> c) {
        super(c);
    }

    public boolean add(E e) {
        try {
            writeLock.lock();
            super.add(e);
            return true;
        } finally {
            writeLock.unlock();
        }
    }

    public E get(int index) {
        try {
            readLock.lock();
            return super.get(index);
        } finally {
            readLock.unlock();
        }
    }

    public int size() {
        try {
            readLock.lock();
            return super.size();
        } finally {
            readLock.unlock();
        }
    }
}
```

读测试
```java
@State(Scope.Thread)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@Fork(value = 2, jvmArgs = {"-Xms4G", "-Xmx4G"})
public class ConcurrentListBenchMark {

    private final List<Integer> list = new ArrayList<>();
    private static final int THREAD_COUNT = 8;
    public static final int OP_COUNT_PER_COUNT = 10000000;
    public static final int INIT_LIST_SIZE = 10000;

    @Setup
    public void prepare() {
        for (int i = 0; i < INIT_LIST_SIZE; i++) {
            list.add(i);
        }
    }

    @Benchmark
    public void noLockRead() throws InterruptedException {
        opRead(new ArrayList<>(list), THREAD_COUNT, OP_COUNT_PER_COUNT);
    }

    @Benchmark
    public void vectorRead() throws InterruptedException {
        opRead(new Vector<>(list), THREAD_COUNT, OP_COUNT_PER_COUNT);
    }

    @Benchmark
    public void collectionsSynchronizedRead() throws InterruptedException {
        opRead(Collections.synchronizedList(list), THREAD_COUNT, OP_COUNT_PER_COUNT);
    }

    @Benchmark
    public void copyOnWriteArrayListRead() throws InterruptedException {
        opRead(new CopyOnWriteArrayList<>(list), THREAD_COUNT, OP_COUNT_PER_COUNT);
    }

    @Benchmark
    public void diyConcurrentListRead() throws InterruptedException {
        opRead(new DiyConcurrentList<>(list), THREAD_COUNT, OP_COUNT_PER_COUNT);
    }

    private void opRead(List<Integer> list, int threadCount, int opCountPerThread) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(threadCount);
        final ExecutorService executorService = Executors.newFixedThreadPool(threadCount);
        for (int i = 0; i < threadCount; i++) {
            executorService.execute(() -> {
                int size = list.size();
                Random random = new Random();
                for (int j = 0; j < opCountPerThread; j++) {
                    list.get(random.nextInt(0, size));
                }
                countDownLatch.countDown();
            });
        }

        countDownLatch.await();
        executorService.shutdownNow();
    }

    @TearDown(Level.Invocation)
    public void tearDown() {
        System.gc();
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(ConcurrentListBenchMark.class.getSimpleName())
                .warmupIterations(5)
                .measurementIterations(5)
                .build();

        new Runner(opt).run();
    }
}
```

读写测试
```java
@State(Scope.Thread)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@Fork(value = 2, jvmArgs = {"-Xms4G", "-Xmx4G"})
public class ConcurrentListBenchMark {
    private final List<Integer> list = new ArrayList<>();
    private static final int READ_THREAD_COUNT = 8;
    private static final int WRITE_THREAD_COUNT = 8;
    //每个读线程执行读操作数，可自行修改测试
    public static final int OP_READ_COUNT_PER_COUNT = 10000;
    //每个读线程执行写操作数，可自行修改测试
    public static final int OP_WRITE_COUNT_PER_COUNT = 1000000;
    public static final int INIT_LIST_SIZE = 10000;

    @Setup
    public void prepare() {
        for (int i = 0; i < INIT_LIST_SIZE; i++) {
            list.add(i);
        }
    }

    @Benchmark
    public void vector() throws InterruptedException {
        opReadAndWrite(new Vector<>(list));
    }

    @Benchmark
    public void collectionsSynchronized() throws InterruptedException {
        opReadAndWrite(Collections.synchronizedList(new ArrayList<>(list)));
    }

    @Benchmark
    public void copyOnWriteArrayList() throws InterruptedException {
        opReadAndWrite(new CopyOnWriteArrayList<>(list));
    }

    @Benchmark
    public void diyConcurrentList() throws InterruptedException {
        opReadAndWrite(new DiyConcurrentList<>(list));
    }

    private void opReadAndWrite(List<Integer> list) throws InterruptedException {
        CountDownLatch countDownLatch = new CountDownLatch(ConcurrentListBenchMark.READ_THREAD_COUNT + ConcurrentListBenchMark.WRITE_THREAD_COUNT);
        final ExecutorService executorService = Executors.newFixedThreadPool(ConcurrentListBenchMark.READ_THREAD_COUNT + ConcurrentListBenchMark.WRITE_THREAD_COUNT);
        opRead(list, ConcurrentListBenchMark.READ_THREAD_COUNT, ConcurrentListBenchMark.OP_READ_COUNT_PER_COUNT, countDownLatch, executorService);
        opWrite(list, ConcurrentListBenchMark.WRITE_THREAD_COUNT, ConcurrentListBenchMark.OP_WRITE_COUNT_PER_COUNT, countDownLatch, executorService);

        countDownLatch.await();
        executorService.shutdownNow();
    }

    private void opRead(List<Integer> list, int threadCount, int opCountPerThread, CountDownLatch countDownLatch, ExecutorService executorService) throws InterruptedException {
        for (int i = 0; i < threadCount; i++) {
            executorService.execute(() -> {
                int size = list.size();
                Random random = new Random();
                for (int j = 0; j < opCountPerThread; j++) {
                    list.get(random.nextInt(0, size));
                }
                countDownLatch.countDown();
            });
        }
    }

    private void opWrite(List<Integer> list, int threadCount, int opCountPerThread, CountDownLatch countDownLatch, ExecutorService executorService) throws InterruptedException {
        for (int i = 0; i < threadCount; i++) {
            executorService.execute(() -> {
                Random random = new Random();
                for (int j = 0; j < opCountPerThread; j++) {
                    list.add(random.nextInt(1, 10000));
                }
                countDownLatch.countDown();
            });
        }
    }
    
    @TearDown(Level.Invocation)
    public void tearDown() {
        System.gc();
    }


    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(ConcurrentListBenchMark.class.getSimpleName())
                .warmupIterations(5)
                .measurementIterations(5)
                .build();

        new Runner(opt).run();
    }
}
```