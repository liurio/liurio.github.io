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

## 线程池的创建

线程池的创建一般有两种方式：***Executors***和***ThreadPoolExecutor***

### Executors的方式

![img](https://gitee.com/liurio/image_save/raw/master/flink/Executors.png)

使用Executors创建线程池有两个弊端：

> - FixedThreadPool和SingleThreadExecutor: 允许请求的队列长度为 Integer.MAX_VALUE,可能堆积大量的请求，从而导致OOM。
> - CachedThreadPool 和 ScheduledThreadPool：允许创建的线程数量为 Integer.MAX_VALUE ，可能会创建大量线程，从而导致OOM。

### ThreadPoolExecutor的方式

![img](https://gitee.com/liurio/image_save/raw/master/flink/executor接口图.png)))

ThreadPoolExecutor的构造函数如下：

```java
public ThreadPoolExecutor(int corePoolSize, //线程池核心池大小
                          int maximumPoolSize,//线程池最大线程数量
                          long keepAliveTime,//当线程数大于核心时，此为终止前多余的空闲线程等待新任务的最长时间。
                          TimeUnit unit,//上一个参数的单位
                          BlockingQueue<Runnable> workQueue,//用来储存等待执行任务的队列。
                          ThreadFactory threadFactory //线程工厂
                         ) {
    ....
}
```

ExecutorService是ThreadPoolExecutor的顶层接口，我们创建线程池时尽量通过ThreadPoolExecutor创建。各个参数的解释如下：

- corePoolSize: 核心池的大小，在创建了线程池后，线程池中的线程数为0，当有任务来之后，就会创建一个线程去执行任务，当线程池中的线程数目达到corePoolSize后，就会把到达的任务放到缓存队列当中;

- maximumPoolSize: 线程池最大线程数，它表示在线程池中最多能创建多少个线程；这个参数是跟后面的阻塞队列联系紧密的；只有当阻塞队列满了，如果还有任务添加到线程池的话，会尝试new 一个Thread的进行救急处理，立马执行对应的runnable任务；如果继续添加任务到线程池，且线程池中的线程数已经达到了maximumPoolSize，那么线程就会就会执行reject操作.

- keepAliveTime: 表示线程没有任务执行时最多保持多久时间会终止；默认情况下，只有当线程池中的线程数大于corePoolSize时，keepAliveTime才会起作用；即当线程池中的线程数大于corePoolSize时，如果一个线程空闲的时间达到keepAliveTime，则会终止，直到线程池中的线程数不超过corePoolSize。但是如果调用了allowCoreThreadTimeOut(boolean)方法并设置了参数为true，在线程池中的线程数不大于corePoolSize时，keepAliveTime参数也会起作用，直到线程池中的阻塞队列大小为0；

- unit: 参数keepAliveTime的时间单位，有7种取值，在TimeUnit类中有7种静态属性（时间单位）

- workQueue: 一个阻塞队列，用来存储等待执行的任务，这个参数的选择也很重要，会对线程池的运行过程产生重大影响，一般来说，这里的阻塞队列有以下几种选择　　

  　　ArrayBlockingQueue;

    　　LinkedBlockingQueue;

    　　SynchronousQueue;

    　　ArrayBlockingQueue和PriorityBlockingQueue使用较少，一般使用LinkedBlockingQueue和Synchronous。线程池的排队策略与BlockingQueue有关

- threadFactory: 线程工厂，主要用来创建线程：默认值 DefaultThreadFactory；（可以自定义设定线程名称）

- handler：表示当拒绝处理任务时的策略，就是上面提及的reject操作；有以下四种取值：

  　　ThreadPoolExecutor.AbortPolicy:丢弃任务并抛出RejectedExecutionException异常。（默认handle）

    　　ThreadPoolExecutor.DiscardPolicy：也是丢弃任务，但是不抛出异常。

    　　ThreadPoolExecutor.DiscardOldestPolicy：丢弃队列最前面的任务，然后重新尝试执行任务（重复此过程）

    　　ThreadPoolExecutor.CallerRunsPolicy：由调用线程处理该任务

```java
ThreadPoolExecutor executor =
    new ThreadPoolExecutor(
    1,
    1,
    60,
    TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(1),
    new ThreadPoolExecutor.DiscardOldestPolicy());
```

## ThreadPoolExecutor

ExecutorService是ThreadPoolExecutor的顶层接口，使用线程池中的线程执行每个提交任务，通常我们使用Executors的工厂方法来创建ExecutorService。线程池创建之后返回的都是ThreadPoolExecutor对象。

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

![img](https://gitee.com/liurio/image_save/raw/master/flink/threadpoolexecutors执行流程.jpg)

当创建线程任务时，调用addWorker方法，可分为两部分

> - 原子操作，判断是否可以创建worker。通过自旋、CAS、ctl 等操作，判断继续创建还是返回false，自旋周期一般很短。
> - 第二部分：同步创建worker，并启动线程。

第二部分创建Worker，Worker是ThreadPoolExecutor的内部类，实现了 AbstractQueuedSynchronizer 并继承了 Runnable。Worker实现了简单的非重入互斥锁，主要是为了控制线程是否可以interrupt，以及其他监控，如线程是否active。runWorker的主要任务就是一直loop循环，来一个任务处理一个任务，没有任务就去getTask()，getTask()可能会阻塞。

## 参考

[https://www.jianshu.com/p/23cb8b903d2c](https://www.jianshu.com/p/23cb8b903d2c)