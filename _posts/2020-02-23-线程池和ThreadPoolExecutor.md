---
logout: post
title: 线程池和ThreadPoolExecutor
tags: [java,all]
---

## 为什么要用线程池？

- 提高程序的执行效率

如果程序中有大量短时间任务的线程任务，由于创建和销毁线程需要和底层操作系统交互，大量时间都耗费在创建和销毁线程上，因而比较浪费时间，系统效率很低
而线程池里的每一个线程任务结束后，并不会死亡，而是再次回到线程池中成为空闲状态，等待下一个对象来使用，因而借助线程池可以提高程序的执行效率

- 控制线程的数量，防止程序崩溃

如果不加限制地创建和启动线程很容易造成程序崩溃，比如高并发1000W个线程，JVM就需要有保存1000W个线程的空间，这样极易出现内存溢出
线程池中线程数量是一定的，可以有效避免出现内存溢出

## 线程池的分类

- **newCachedThreadPool** 创建一个具有缓存功能的线程池，系统根据需要创建线程，这些线程将会被缓存在线程池中；

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

- **newFixedThreadPool** 创建一个可重用的、具有固定线程数的线程池；

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

- **newSingleThreadExecutor** 创建一个只有单线程的线程池

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

- **newScheduledThreadPool** 创建具有指定线程数的线程池，它可以在指定延迟后执行线程任务。

```java
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
```

## ThreadPoolExecutor

ExecutorService是ThreadPoolExecutor的顶层接口，使用线程池中的线程执行每个提交任务，通常我们使用Executors的工厂方法来创建ExecutorService。

```java
ExecutorService executorService = Executors.newFixedThreadPool(2);
executorService.submit(new Thread());
executorService.shutdown();
```

### ThreadPoolExecutor状态控制

线程池使用一个AtomicInteger的ctl变量将`workerCount`(工作线程数量)和`runState`(运行状态)两个字段压缩在一起。这种做法在在java源码里经常有出现，比如ReentrantReadWriteLock里将一个int分成高16位和低16位分别表示读锁和写锁。

ThreadPoolExecutor用3个比特位表示runState， 29个比特位表示workerCount。

| 状态       | 解释                                                         |
| ---------- | ------------------------------------------------------------ |
| RUNNING    | 运行态，可处理新任务并执行队列中的任务                       |
| SHUTDOWN   | 关闭态，不接受新任务，但处理队列中的任务                     |
| STOP       | 停止态，不接受新任务，不处理队列中任务，且打断运行中任务     |
| TIDYING    | 整理态，所有任务已经结束，workerCount = 0 ，将执行terminated()方法 |
| TERMINATED | 结束态，terminated() 方法已完成                              |

### ThreadPoolExecutor执行原理

整体上流程如下：

![img](https://upload-images.jianshu.io/upload_images/11183270-a01aea078d7f4178.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)

当创建线程任务时，调用addWorker方法，可分为两部分

> - 原子操作，判断是否可以创建worker。通过自旋、CAS、ctl 等操作，判断继续创建还是返回false，自旋周期一般很短。
> - 第二部分：同步创建worker，并启动线程。

第二部分创建Worker，Worker是ThreadPoolExecutor的内部类，实现了 AbstractQueuedSynchronizer 并继承了 Runnable。Worker实现了简单的非重入互斥锁，主要是为了控制线程是否可以interrupt，以及其他监控，如线程是否active。runWorker的主要任务就是一直loop循环，来一个任务处理一个任务，没有任务就去getTask()，getTask()可能会阻塞。

## 参考

[https://www.jianshu.com/p/23cb8b903d2c](https://www.jianshu.com/p/23cb8b903d2c)