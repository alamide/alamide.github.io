---
layout: post
title: Java 基础 Thread
categories: java
tags: Java Thread
date: 2023-04-11
---
Java 基础语法 Thread，学习内容来自 《Java 编程的逻辑》
<!--more-->
## 1.开启线程
```java
Thread thread1 = new Thread(() -> {
    System.out.println(Thread.currentThread().getName() + " : start a thread");
});
Thread thread2 = new Thread(() -> {
    System.out.println(Thread.currentThread().getName() + " : start a thread");
});

thread1.start();
thread2.start();
```

## 2.join
Thread有一个join方法，可以让调用join的线程等待该线程结束
```java
Thread[] threads = new Thread[1000];
for (int i = 0; i < 1000; i++) {
    threads[i] = new Thread(() -> {
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    });
    threads[i].start();
}

//        for (Thread t : threads) {
//            t.join();
//        }

final Map<String, Long> collect = IntStream.rangeClosed(1, threads.length)
        .boxed()
        .map(i -> threads[i - 1].getState())
        .collect(Collectors.groupingBy(Enum::toString, Collectors.counting()));
System.out.println(collect);
```

output:
```
{TERMINATED=424, TIMED_WAITING=576}
```

打开注释，使用 `join`

output:
```
{TERMINATED=1000}
```

可以看到调用 `join` ，会等待该线程结束
## 3.多线程共享内存的问题
当多个线程共享数据时，可能会出现线程安全问题，Java 线程模型如下，模型图片来自深入理解 Java 虚拟机
<img src="../assets/imgs/java-thread-model.jpg" style="width:100%;margin-top:15px;margin-bottom:15px;"/>
在 Java 内存模型中，分为主内存和线程工作内存，每条线程都有自己的工作内存，线程使用共享数据时，都是先从主存拷贝到工作内存，线程对该变量的所有操作都必须在工作内存中进行，而不能直接读写主内存中的变量，操作完成后再写回主内存。不能线程之间不能互相访问对方的工作内存，必须通过主内存来完成。多线程操作同一变量时会发生数据不一致，出现线程安全问题。

```java
public class ThreadTest {
    public static int sharedInt = 0;
    @Test
    public void testSharedVariable() throws InterruptedException {
        Thread[] threads = new Thread[1000];
        for (int i = 0; i < 1000; i++) {
            threads[i] = new Thread(() -> {
                IntStream.rangeClosed(1, 10000).forEach(t -> sharedInt++);
            });
            threads[i].start();
        }

        for (Thread t : threads) {
            t.join();
        }

        System.out.println(sharedInt);
    }
}
```

output:
```
9567011
```

输出结果不是预想的 `10000000`

可以如下解决
```java
public static AtomicInteger sharedInt = new AtomicInteger(0);
@Test
public void testSharedVariable() throws InterruptedException {
    Thread[] threads = new Thread[1000];
    for (int i = 0; i < 1000; i++) {
        threads[i] = new Thread(() -> {
            IntStream.rangeClosed(1, 1000000).forEach(t -> sharedInt.addAndGet(1));
        });
        threads[i].start();
    }

    for (Thread t : threads) {
        t.join();
    }

    System.out.println(sharedInt.get());
}
```

## 4.volatile (内存可见性)
```java
private boolean shutdown = false;
@Test
public void testVolatile() throws InterruptedException {
    Thread thread = new Thread(() -> {
        while (!shutdown){

        }
        System.out.println("exit " + Thread.currentThread().getName());
    });

    thread.start();
    Thread.sleep(1000);
    shutdown = true;
    Thread.sleep(1000);

    System.out.println("exit " + Thread.currentThread().getName());
}
```

output:
```
exit main
```

当 shutdown 修改为 true 时，对 thread 线程不可见，可以改为 `private volatile boolean shutdown = false;`
```java
exit Thread-0
exit main
```

## 5.synchronized 同步
synchronized 可以用于修饰类的实例方法、静态方法和代码块

## 6.死锁
任务可以变成阻塞状态，所以可能会出现这种情况：某个任务在等待别的任务，而后者又在等待别的任务，这样一直下去，直到这个链条上的任务又在等待第一个任务释放锁。这得到
一个任务之间相互等待的连续循环，没有哪个线程能继续，这就称之为死锁。

死锁需要满足四个条件：
1. 互斥条件，任务中使用的资源至少有一个是不能共享的。

2. 至少有一个任务它必须持有一个资源且正在获取一个当前被别的任务持有的资源。

3. 资源不能被任务抢占，一个任务等待其它任务持有的资源。

4. 必须循环等待，这时，一个任务等待其它任务所持有的资源，后者又在等待另一个任务所持有的资源，这样一直下去，直到有一个任务在等待第一个任务持有的资源，使得大家都被锁住。

```java
public static class Resources {
    private static final Object resA = new Object();
    private static final Object resB = new Object();

    public static void startThreadA() throws InterruptedException {
        Thread threadA = new Thread(() -> {
            synchronized (resB) {
                try {
                    System.out.println("threadA working...");
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (resA) {
                }
            }
        });

        threadA.start();
    }

    public static void startThreadB() throws InterruptedException {
        Thread threadB = new Thread(() -> {
            synchronized (resA) {
                try {
                    System.out.println("threadB working...");
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                synchronized (resB) {
                }
            }
        });

        threadB.start();
        threadB.join();
    }
}

@Test
public void testDeadLock() throws InterruptedException {
    Resources.startThreadA();
    Resources.startThreadB();
}
```

经典死锁，哲学家就餐问题，五个哲学家，他们很穷，只能买得起五根筷子，他们围坐在桌子周围，每人之间放一根筷子。当一个哲学家就餐的时候，必须同时得到
左边和右边的筷子。如果一个哲学家左边或右边已经有人在使用筷子了，那么这个哲学家就必须等待，直到可得到必须的筷子。

```java
public static class Chopstick {
    private boolean isTaken = false;

    public synchronized void take() throws InterruptedException {
        while (isTaken) {
            wait();
        }
        isTaken = true;
    }

    public synchronized void drop() {
        notifyAll();
        isTaken = false;
    }
}

public static class Philosopher implements Runnable {
    private final Chopstick left;
    private final Chopstick right;
    private final int id;
    private final int ponderFactor;
    private final Random random = new Random(47);

    public Philosopher(Chopstick left, Chopstick right, int id, int ponderFactor) {
        this.left = left;
        this.right = right;
        this.id = id;
        this.ponderFactor = ponderFactor;
    }

    private void pause() {
        if (ponderFactor == 0) return;
        try {
            TimeUnit.MILLISECONDS.sleep(random.nextInt(ponderFactor * 250));
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }

    @Override
    public void run() {
        try {
            while (!Thread.interrupted()) {
                System.out.println("Philosopher " + id + " is thinking...");
                pause();

                System.out.println("Philosopher " + id + " taking left chopstick");
                left.take();
                System.out.println("Philosopher " + id + " take right chopstick");
                right.take();

                System.out.println("Philosopher " + id + " is eating...");
                pause();

                left.drop();
                right.drop();
            }
        } catch (InterruptedException e) {
            System.out.println("Philosopher " + id + " exiting via interrupt");
        }
    }
}

@Test
public void testPhilosopher() {
    Chopstick[] chopsticks = new Chopstick[5];
    Philosopher[] philosophers = new Philosopher[5];

    for (int i = 0; i < chopsticks.length; i++) {
        chopsticks[i] = new Chopstick();
    }
    int chopsticksNum = chopsticks.length;
    for (int i = 0; i < philosophers.length; i++) {
        philosophers[i] = new Philosopher(chopsticks[i], chopsticks[(i + 1) % chopsticksNum], i, 0);
    }

    final ExecutorService executorService = Executors.newFixedThreadPool(5);
    for (Philosopher philosopher : philosophers) {
        executorService.execute(philosopher);
    }

    try {
        TimeUnit.SECONDS.sleep(10);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
    executorService.shutdownNow();
}
```

解决死锁，破除循环等待
```java
for (int i = 0; i < philosophers.length - 1; i++) {
    philosophers[i] = new Philosopher(chopsticks[i], chopsticks[i + 1], i, 0);
}

philosophers[chopsticksNum - 1] = new Philosopher(chopsticks[0], chopsticks[chopsticksNum - 1], chopsticksNum - 1, 0);
```

## 7.线程的基本协作 wait、notify
wait 是在获取对象🔒之后才可以调用，下面的调用是错误的
```java
@Test
public void testWait() throws InterruptedException {
    wait();
}
```

正确用法，在加锁的对象上调用
```java
@Test
public void testWait() throws InterruptedException {
    Object locker = new Object();
    synchronized (locker){
        locker.wait();
    }
}
```

在 wait 期间对象锁是释放的，可以通过 notify、notifyAll，或令时间到期，从 wait 中恢复执行

线程协作
```java
public static class WaitThread extends Thread {
    private volatile boolean isPrepared = false;

    @Override
    public void run() {
        synchronized (this) {
            try {
                while (!isPrepared) {
                    wait();
                }
                System.out.println("begin execute....");
            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
    }

    public synchronized void prepared() {
        isPrepared = true;
        notify();
    }
}

@Test
public void testWait() throws InterruptedException {
    WaitThread waitThread = new WaitThread();
    waitThread.start();

    Thread.sleep(1000);
    waitThread.prepared();
    System.out.println("prepared...");
}
```

生产者消费者
```java
public static class Producer {
    private volatile boolean isConsumed = true;
    private int productId = 0;
    public synchronized void produce() {
        try {
            while (!Thread.interrupted()) {
                if (isConsumed) {
                    productId++;
                    System.out.println("producing productId: " + productId + " ...");
                    Thread.sleep(100);
                    System.out.println("produced productId: " + productId + " ...");
                    isConsumed = false;
                    notifyAll();
                } else {
                    wait();
                }
            }
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }

    public synchronized void consume(int consumerId) {
        while (isConsumed) {
            try {
                wait();

            } catch (InterruptedException e) {
                throw new RuntimeException(e);
            }
        }
        System.out.println("consumerId: " + consumerId + " consumed");
        isConsumed = true;
        notify();
    }
}

public static class Consumer implements Runnable {

    private final Producer producer;
    private final Integer id;

    public Consumer(Producer producer, Integer id) {
        this.producer = producer;
        this.id = id;
    }

    @Override
    public void run() {
        producer.consume(id);
    }
}

@Test
public void testConsume() throws IOException, InterruptedException {
    Producer producer = new Producer();

    final ExecutorService executorService = Executors.newFixedThreadPool(11);
    executorService.execute(producer::produce);

    for (int i = 0; ;i++) {
        Thread.sleep(50);
        executorService.execute(new Consumer(producer, i));
    }
}
```

## 8.异步结果
```java
Callable<String> callable = () -> {
    Thread.sleep(10000);
    return "complete!";
};

final ExecutorService executorService = Executors.newFixedThreadPool(1);
final Future<String> submit = executorService.submit(callable);
System.out.println(submit.get());
```

## 9.线程中断
很多线程的运行模式是死循环，如生产者/消费者模式中，消费者主体就是一个死循环，在程序停止时，我们需要有一种优雅的方式关闭该线程。

Thread 定义了如下线程中断的方法：
```java
public boolean isInterrupted()
public void interrupt()
public static boolean interrupted()
```

一般我们的线程使用 Executor 管理，调用 shutdown/shutdownNow。调用线程中断方法后，不一定立即结束线程。

线程的状态有：
1. RUNNABLE：线程在运行或具备运行条件只是在等待操作系统调度。

2. WAITING/TIMED_WAITING：线程在等待某个条件或超时。

3. BLOCKED：线程在等待锁，试图进入同步块。

4. NEW/TERMINATED：线程还未启动或已结束。

## 10.原子变量和CAS
对于以下形式的代码，synchronized 成本有点太高了，需要先获取锁，最后再释放锁，获取不到锁的时候还要等待，还有线程的上下文切换
```java
private int count;
public synchronized void incr() {
    count++;
}
public synchronized int getCount() {
    return count;
}
```

可以做一下性能测试对比一下，测试数据 8 线程，每个线程 1000_0000 次 incr
```java
public class SynchronizedCounter implements ICounter{
    private int count;
    @Override
    public synchronized void incr() {
        count++;
    }
    @Override
    public synchronized int getCount() {
        return count;
    }
}

public class AtomicIntegerCounter implements ICounter{
    private final AtomicInteger counter = new AtomicInteger(0);
    @Override
    public void incr(){
        counter.incrementAndGet();
    }
    @Override
    public int getCount(){
        return counter.get();
    }
}

@State(Scope.Thread)
@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.MILLISECONDS)
@Fork(value = 2, jvmArgs = {"-Xms4G", "-Xmx4G"})
public class CounterBenchMark {

    @Benchmark
    public void atomic() throws InterruptedException {
        count(new AtomicIntegerCounter());
    }
    
    @Benchmark
    public void synch() throws InterruptedException {
        count(new SynchronizedCounter());
    }

    private void count(ICounter counter) throws InterruptedException {
        final ExecutorService executorService = Executors.newFixedThreadPool(8);
        CountDownLatch countDownLatch = new CountDownLatch(8);

        for (int i = 0; i < 8; i++) {
            executorService.execute(() -> {
                IntStream.rangeClosed(1, 10000000).forEach(n -> counter.incr());
                countDownLatch.countDown();
            });
        }

        countDownLatch.await();
        executorService.shutdownNow();

        assert counter.getCount() == 8000_0000;
    }

    public static void main(String[] args) throws RunnerException {
        Options opt = new OptionsBuilder()
                .include(CounterBenchMark.class.getSimpleName())
                .warmupIterations(5)
                .measurementIterations(5)
                .build();

        new Runner(opt).run();
    }
}
```

测试结果，可以看出性能差距还是蛮大的
```
Benchmark                Mode  Cnt     Score     Error  Units
CounterBenchMark.atomic  avgt   10  2257.037 ± 244.928  ms/op
CounterBenchMark.synch   avgt   10  5586.794 ± 735.329  ms/op
```

Java 包中提供的原子变量类型有 `AtomicBoolean`、`AtomicInteger`、`AtomicLong`、`AtomicReference`

AtomicInteger 主要方法有
```java
//构造方法
public AtomicInteger(int initialValue)
public AtomicInteger()

//实例方法
//以原子方式获取旧值并设置新值
public final int getAndSet(int newValue)
//以原子方式获取旧值并给当前值加1
public final int getAndIncrement()
//以原子方式获取旧值并给当前值减1
public final int getAndDecrement()
//以原子方式获取旧值并给当前值加delta
public final int getAndAdd(int delta)
//以原子方式给当前值加1并获取新值
public final int incrementAndGet()
//以原子方式给当前值减1并获取新值
public final int decrementAndGet()
//以原子方式给当前值加delta并获取新值
public final int addAndGet(int delta)
```

所有这些方法都依赖 `compareAndSet` 简称 `CAS`，书上有写实现原理，但是在 Java17 中，已经使用本地方法实现了，所以就先不去扒源码了

## 11.显式锁
显式锁的接口为 `Lock`
```java
public interface Lock {
    //获取锁，会阻塞
    void lock();
    //获取锁，可以响应中断
    void lockInterruptibly() throws InterruptedException;
    //尝试获取锁，立即返回，不阻塞，如果成功返回 true，失败返回 false
    boolean tryLock();
    //先尝试获取锁，如果成功立即返回 true，否则阻塞等待，等待的时长由参数指定，等待的同时响应中断，如果发生中断抛出 InterruptedException
    //如果在等待时长内获取了锁，返回 true，否则返回 false
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
    //释放锁
    void unlock();
    //新建一个条件
    Condition newCondition();
}
```

Lock 的主要实现类为 ReentrantLock，其构造方法为
```java
public ReentrantLock()
//参数为是否保证公平，默认为 false，公平是指等待时间最长的线程优先获得锁，保证公平会影响性能，所以默认不保证，synchronized 为不公平
public ReentrantLock(boolean fair)
```

基本用法如下
```java
public class LockCounter implements ICounter {
    private final ReentrantLock lock = new ReentrantLock();
    private volatile int count;

    @Override
    public void incr() {
        lock.lock();
        try {
            count++;
        } finally {
            lock.unlock();
        }
    }

    @Override
    public int getCount() {
        return count;
    }
}
```

可以使用 tryLock 来避免死锁，将上面的死锁代码修改一下，就可以避免死锁了
```java
public static class Resources {
    private static final ReentrantLock resA = new ReentrantLock();
    private static final ReentrantLock resB = new ReentrantLock();

    public static void startThreadA() throws InterruptedException {
        Thread threadA = new Thread(() -> {
            resB.tryLock();
            try {
                System.out.println("threadA working...");
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                resB.unlock();
            }
            resA.tryLock();

        });

        threadA.start();
    }

    public static void startThreadB() throws InterruptedException {
        Thread threadB = new Thread(() -> {
            resA.tryLock();
            try {
                System.out.println("threadB working...");
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                resA.unlock();
            }
            resB.tryLock();
        });

        threadB.start();
        threadB.join();
    }
}
```

一般情况下能用 `synchronized` 就用 `synchronized`，不满足要求时再考虑使用 `ReentrantLock`。
