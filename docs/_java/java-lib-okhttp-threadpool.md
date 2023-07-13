---
layout: post
title: OkHttp 线程池
categories: java 
tags: java okhttp thread
date: 2023-07-13
---
OkHttp 中线程池的使用。

本文学习目标：

为什么 OkHttp 要使用的线程池参数 corePoolSize 为 0，为什么使用 SynchronousQueue。
<!--more-->

## 1.为什么要使用自定义 ThreadPoolExecutor
OkHttp 中使用的线程池都为这种形式，执行异步请求及清理连接池
```java
executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<>(), Util.threadFactory("OkHttp Dispatcher", false));
```

先来看一下 ThreadPoolExecutor 构造方法中的各个参数
```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory) {
}
```

corePoolSize 线程池中保持核心线程数，即使空闲也不会被销毁

maximumPoolSize 线程池中可同时存在的最大线程数，线程池中除了核心线程，在必要时还可以创建额外的线程

keepAliveTime 创建的额外线程最大存活时间

workQueue 没有空闲线程时任务存放的队列，供给有空闲线程时获取并执行

threadFactory 线程创建工厂，用来创建线程，可以设置自定义线程名


Executors 提供了一些方法来生成 ThreadPoolExecutor 如
```java
Executors.newFixedThreadPool();
Executors.newSingleThreadExecutor();
Executors.newCachedThreadPool();
``` 

那为什么 OkHttp 还要使用自定义的 ThreadPoolExecutor 呢？

先来看看这几个方法：

Executors.newFixedThreadPool() 创建固定线程数的线程池，不能满足网络请求的场景。原因为，无法确定同时网络请求数，也就无法确定核心线程数。核心线程池数不能设置太大，否则会和浪费资源。不能设置太小，否则会有请求任务在阻塞排队。所以不符合。

Executors.newSingleThreadExecutor() 同上

Executors.newCachedThreadPool() 是符合需求的，实际上 OkHttp 的自定义 ThreadPoolExecutor 与 Executors.newCachedThreadPool() 基本一致，除了 threadFactory


好了，到这里可以解释为什么要自定义的问题了，下面就是自定义的参数解析了。

OkHttp 的亮点之一就是高并发，高并发的要求是什么，有多少请求处理多少，不需要等待，所以使用 SynchronousQueue 而不是 LinkedBlockingQueue。

## 2.SynchronousQueue VS LinkedBlockingQueue
在阅读 ThreadPoolExecutor 源码之前，先来看一下 SynchronousQueue 和 LinkedBlockingQueue 的区别，目前不会去看它们的源码，有点复杂。

SynchronousQueue 和 LinkedBlockingQueue 都是 BlockingQueue，SynchronousQueue 是当有消费者等待获取值时，才能生产者才能入列成功，有点拗口，看代码吧
```java
SynchronousQueue<Integer> synchronousQueue = new SynchronousQueue<>();
final boolean offer = synchronousQueue.offer(10);
System.out.println(offer);//false
```

这里会入列失败，因为没有消费者在等待获取，
```java
SynchronousQueue<Integer> synchronousQueue = new SynchronousQueue<>();

//因为 take() 为阻塞操作，所以需要放在子线程中。
Thread thread = new Thread(() -> {
    final Integer take;
    try {
        take = synchronousQueue.take();
        System.out.println(take);
    } catch (InterruptedException e) {
        throw new RuntimeException(e);
    }
});

thread.start();

Thread.sleep(1000);
final boolean offer = synchronousQueue.offer(10);
System.out.println(offer);
```

这样就可以入列和获取成功了，SynchronousQueue 还有一个 poll(long timeout, TimeUnit unit) 方法，等待指定时间，超时则返回 null
```java
take = synchronousQueue.poll(2, TimeUnit.SECONDS);
```

可以这样理解 SynchronousQueue，生产一个消费一个，没有消费者就直接丢弃，不会额外存储。

而 LinkedBlockingQueue 则是生产多少存多少，供给消费者消费，消费不了就存储在队列中。

## 3.ThreadPoolExecutor
来看下一个 Runnable 怎么被执行的，用线程池执行一个任务，调用 ThreadPoolExecutor.execute()
```java
public class ThreadPoolExecutor extends AbstractExecutorService {
    public void execute(Runnable command) {
        if (command == null)
            throw new NullPointerException();

        int c = ctl.get();
        
        //corePooolSize=0, 不执行
        if (workerCountOf(c) < corePoolSize) {
            if (addWorker(command, true))
                return;
            c = ctl.get();
        }

        //使用 SynchronousQueue，第一次 false
        if (isRunning(c) && workQueue.offer(command)) {
            int recheck = ctl.get();
            if (! isRunning(recheck) && remove(command))
                reject(command);
            else if (workerCountOf(recheck) == 0)
                addWorker(null, false);
        }
        //执行 addWorker
        else if (!addWorker(command, false))
            reject(command);
    }

    private boolean addWorker(Runnable firstTask, boolean core) {
        Worker w = null;
        try {
            w = new Worker(firstTask);
            final Thread t = w.thread;
            if (t != null) {
                ......
                if (workerAdded) {
                    t.start();
                    workerStarted = true;
                }
            }
        } finally {
            if (! workerStarted)
                addWorkerFailed(w);
        }
        return workerStarted;
    }

    private final class Worker
        extends AbstractQueuedSynchronizer
        implements Runnable
    {
        Worker(Runnable firstTask) {
            setState(-1); // inhibit interrupts until runWorker
            this.firstTask = firstTask;
            this.thread = getThreadFactory().newThread(this);
        }

        public void run() {
            runWorker(this);
        }
    }
}
```

调用链一致到 runWorker，可以获知 Worker 是 ThreadPool 负责执行任务的类，

```java
final void runWorker(Worker w) {
    Thread wt = Thread.currentThread();
    Runnable task = w.firstTask;
    w.firstTask = null;

    while (task != null || (task = getTask()) != null) {
        task.run();
    }
}

private Runnable getTask() {
    for (;;) {
        if ((wc > maximumPoolSize || (timed && timedOut))
            && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        Runnable r = timed ?
                workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                workQueue.take();
        if (r != null)
            return r;
    }
}
```

runWork 的工作是执行具体的 Runnable，执行完之后，就从队列中继续获取，获取之后继续执行，如果队列中没有元素，就阻塞，这样就完成了 Thread 的复用。

如果超时，且 workers > maximumPoolSize && workQueue.isEmpty() 则返回 null，Worker 调用结束，销毁。

再来看执行的代码
```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();

    int c = ctl.get();
    
    //corePooolSize=0, 不执行
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }

    /**
     * 使用 SynchronousQueue
     * 第一次：workQueue.offer()=false，addWorker
     * 第二次：如果第一次请求时创建的 Worker 已经执行完任务，且未超时，则 offer 成功， Worker 继续工作；
     * 如果未执行完，则 offer 失败，走第一次的流程
     * 
     * 使用 LinkedBlockingQueue
     * 第一次：workQueue.offer()=true，workerCountOf(recheck) == 0 -> addWorker
     * 第二次：workQueue.offer()=true，workerCountOf(recheck) != 0 直接跳出，任务存放在队列中，永远只会有一个 Worker
     **/
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    //执行 addWorker
    else if (!addWorker(command, false))
        reject(command);
}
```

所以可以获知，使用 SynchronousQueue 任务永远都会被立刻执行，有空闲的 Worker 使用空闲的 Worker，没有则新创建

而 LinkedBlockingQueue，只会创建一个 Worker，任务存放在 BlockingQueue 中，等待执行

所以这里解释了，为什么使用 SynchronousQueue。

你可能会有疑问，那直接来一个执行一个好了，为什么要使用队列？使用队列是为了复用 Thread ，这也是 ThreadPool 的核心功能之一。

那为什么 corePoolSize 为 0 呢？这个我也不是太理解，按照我的理解起码线程池中应该放几个长存线程，而且网络调用也是很频繁的。我目前的理解如下，网络请求还是很频繁的，所以总会有几个 Worker 未超时，而保留在线程池中。当所有 Worker 都超时，那么用户有很大概率是退出应用了，所以设置为 0。

至于 maximumPoolSize 设置为 Integer.MAX_VALUE，可以最大程度提供并发能力。

threadFactory 没什么可说的，就是创建 Thread，主要用来设置线程名，
```java
public static ThreadFactory threadFactory(String name, boolean daemon) {
    return runnable -> {
        Thread result = new Thread(runnable, name);
        result.setDaemon(daemon);
        return result;
    };
}
```






