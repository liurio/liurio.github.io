---
logout: post
title: Future和CompletableFuture的区别
tags: [java,thread,all]
---

### 为什么引入CompletableFutrue?

#### 回调

回调函数的机制：

- 定义一个回调函数
- 提供函数实现的一方在初始化时候，将回调函数的函数指针注册给调用者
- 当特定的事件或条件发生的时候，调用者使用函数指针调用回调函数对事件进行处理

#### 回调方式的异步编程

所谓异步调用其实就是实现一个可无需等待被调用函数的返回值而让操作继续运行的方法。在 Java 语言中，简单的讲就是另启一个线程来完成调用中的部分计算，使调用继续运行或返回，而不需要等待计算结果。但调用者仍需要取线程的计算结果。

JDK5新增了Future接口，用于描述一个异步计算的结果。虽然 Future 以及相关使用方法提供了异步执行任务的能力，但是对于结果的获取却是很不方便，只能通过阻塞或者轮询的方式得到任务的结果。阻塞的方式显然和我们的异步编程的初衷相违背，轮询的方式又会耗费无谓的 CPU 资源，而且也不能及时地得到计算结果。

以前我们获取一个异步任务的结果可能是这样写的：

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    System.out.println(Thread.currentThread().getName());
    ExecutorService executorService = Executors.newCachedThreadPool();
    Future<String> future = executorService.submit(() -> {
        return "new thread";
    });
    executorService.shutdown();
    System.out.println(future.get());
}
```

#### 组合计算

Future接口可以构建异步应用，但依然有其局限性。它很难直接表述多个Future 结果之间的依赖性。实际开发中，我们经常需要达成以下目的：

- 将多个异步计算的结果合并为一个
- 等待Future集合中的所有任务都完成
- Future完成事件 (即，任务完成以后触发执行动作)

### CompletableFuture

在Java8中，CompletableFuture提供了非常强大的Future的扩展功能，可以帮助我们简化异步编程的复杂性，并且提供了函数式编程的能力，可以通过回调的方式处理计算结果，也提供了转换和组合 CompletableFuture 的方法。

它可能代表一个明确完成的Future，也有可能代表一个完成阶段（ CompletionStage ），它支持在计算完成以后触发一些函数或执行某些动作。

CompletionStage代表异步计算过程中的某一个阶段，一个阶段完成以后可能会触发另外一个阶段，一个阶段的计算执行可以是一个Function，Consumer或者Runnable。比如：stage.thenApply(x -> square(x)).thenAccept(x -> System.out.print(x)).thenRun(() -> System.out.println())。

它实现了Future和CompletionStage接口：

```java
public class CompletableFuture<T> implements Future<T>, CompletionStage<T>
```

#### CompletableFuture的创建

CompletableFuture 提供了四个静态方法来创建一个异步操作。

```JAVA
public static CompletableFuture<Void> runAsync(Runnable runnable)
public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor)
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier)
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor)
```

没有指定Executor的方法会使用ForkJoinPool.commonPool() 作为它的线程池执行异步代码。如果指定线程池，则使用指定的线程池运行。以下所有的方法都类同。

- runAsync方法不支持返回值
- supplyAsync可以支持返回值

#### CompletableFuture的组合操作

##### thenApply方法

当一个线程依赖另一个线程时，可以使用 thenApply 方法来把这两个线程串行化。有返回值

```java
public <U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn)
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor)
/**
	Function<? super T,? extends U>
    T：上一个任务返回结果的类型
    U：当前任务的返回值类型
*/
```

demo:

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    System.out.println(Thread.currentThread().getName());
    ExecutorService executorService = Executors.newCachedThreadPool();
    CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
        return 1;
    }).thenApplyAsync(v -> v + 2);
    executorService.shutdown();
    System.out.println(future.get());
}
/**
	3
*/
```

##### handle方法

handle 是执行任务完成时对结果的处理。
handle 方法和 thenApply 方法处理方式基本一样。不同的是 handle 是在任务完成后再执行，还可以处理异常的任务。thenApply 只可以执行正常的任务，任务出现异常则不执行 thenApply 方法。

```java
public <U> CompletionStage<U> handle(BiFunction<? super T, Throwable, ? extends U> fn);
public <U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn);
public <U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn,Executor executor);
```

demo:

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    System.out.println(Thread.currentThread().getName());
    ExecutorService executorService = Executors.newCachedThreadPool();
    CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
        int i = 1/0;
        return 1;
    }).handleAsync((v,throwable) -> {
        int result = 2;
        if (throwable == null) {
            result = v+result;
        } else {
            System.out.println(throwable.getMessage());
        }
        return result;
    });
    executorService.shutdown();
    System.out.println(future.get());
}
/**
	main
    java.lang.ArithmeticException: / by zero
    2
*/
```

从示例中可以看出，在 handle 中可以根据任务是否有异常来进行做相应的后续处理操作。而 thenApply 方法，如果上个任务出现错误，则不会执行 thenApply 方法。

##### thenAccept方法

接收任务的处理结果，并消费处理，无返回结果。和thenApply方法一致，只是该方法没有返回值。

```java
public CompletionStage<Void> thenAccept(Consumer<? super T> action);
public CompletionStage<Void> thenAcceptAsync(Consumer<? super T> action);
public CompletionStage<Void> thenAcceptAsync(Consumer<? super T> action,Executor executor);
```

demo

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    System.out.println(Thread.currentThread().getName());
    ExecutorService executorService = Executors.newCachedThreadPool();
    CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> {
        return 1;
    }).thenAcceptAsync(v -> {
        System.out.println(v);
    });
    executorService.shutdown();
    System.out.println(future.get());
}
/**
	main
    1
    null
*/
```

##### thenRun方法

跟 thenAccept 方法不一样的是，不关心任务的处理结果。只要上面的任务执行完成，就开始执行 thenAccept 。

```java
public CompletionStage<Void> thenRun(Runnable action);
public CompletionStage<Void> thenRunAsync(Runnable action);
public CompletionStage<Void> thenRunAsync(Runnable action,Executor executor);
```

demo

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    System.out.println(Thread.currentThread().getName());
    ExecutorService executorService = Executors.newCachedThreadPool();
    CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> {
        return 1;
    }).thenRunAsync(() -> {
        System.out.println("thenRun...");
    });
    executorService.shutdown();
    System.out.println(future.get());
}
/**
	main
    thenRun...
    null
*/
```

##### thenCombine方法

thenCombine 会把 两个 CompletionStage 的任务都执行完成后，把两个任务的结果一块交给 thenCombine 来处理。

```java
public <U,V> CompletionStage<V> thenCombine(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn);
public <U,V> CompletionStage<V> thenCombineAsync(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn);
public <U,V> CompletionStage<V> thenCombineAsync(CompletionStage<? extends U> other,BiFunction<? super T,? super U,? extends V> fn,Executor executor);
```

demo:

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    System.out.println(Thread.currentThread().getName());
    ExecutorService executorService = Executors.newCachedThreadPool();
    CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
        System.out.println(Thread.currentThread().getName());
        return 1;
    },executorService);
    CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> {
        System.out.println(Thread.currentThread().getName());
        return 2;
    },executorService);
    CompletableFuture<Integer> future = future1.thenCombineAsync(future2,(v1,v2) -> {
        System.out.println(Thread.currentThread().getName());
        return v1 + v2;
    },executorService);
    executorService.shutdown();
    System.out.println(future.get());
}
/**
	main
    pool-1-thread-1
    pool-1-thread-2
    pool-1-thread-2
    3
*/
```

##### applyToEither方法

两个CompletionStage，谁计算的快，就用那个CompletionStage的结果进行下一步的处理。

```java
public <U> CompletionStage<U> applyToEither(CompletionStage<? extends T> other,Function<? super T, U> fn);
public <U> CompletionStage<U> applyToEitherAsync(CompletionStage<? extends T> other,Function<? super T, U> fn);
public <U> CompletionStage<U> applyToEitherAsync(CompletionStage<? extends T> other,Function<? super T, U> fn,Executor executor);
```

demo

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    System.out.println(Thread.currentThread().getName());
    ExecutorService executorService = Executors.newCachedThreadPool();
    CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
        System.out.println("1 -- "+Thread.currentThread().getName());
        try {
            Thread.sleep(100);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return 1;
    },executorService).applyToEitherAsync(CompletableFuture.supplyAsync( () -> {
        System.out.println("2 -- "+Thread.currentThread().getName());
        try {
            Thread.sleep(300);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return 2;
    },executorService), v -> {
        System.out.println("3 -- "+Thread.currentThread().getName());
        return v;
    });
    executorService.shutdown();
    System.out.println(future.get());
}
/**
	main
    1 -- pool-1-thread-1
    2 -- pool-1-thread-2
    3 -- ForkJoinPool.commonPool-worker-1
    1
*/
```

#### CompletableFuture的完成操作

当CompletableFuture的计算结果完成，或者抛出异常的时候，可以执行特定的Action。主要是下面的方法：

```java
public CompletableFuture<T> whenComplete(BiConsumer<? super T,? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action)
public CompletableFuture<T> whenCompleteAsync(BiConsumer<? super T,? super Throwable> action, Executor executor)
public CompletableFuture<T> exceptionally(Function<Throwable,? extends T> fn)
```

demo

```java
public static void main(String[] args) throws ExecutionException, InterruptedException {
    System.out.println(Thread.currentThread().getName());
    ExecutorService executorService = Executors.newCachedThreadPool();
    CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
        System.out.println("1 -- "+Thread.currentThread().getName());
        return 1/0;
    }).whenComplete((v,throwable)->{
        System.out.println("run end...");
    }).exceptionally(throwable -> {
        System.out.println("exist exception...");
        return null;
    });
    executorService.shutdown();
    System.out.println(future.get());
}
/**
	main
    1 -- ForkJoinPool.commonPool-worker-1
    run end...
    exist exception...
    null
*/
```

#### CompletableFuture的返回值获取

获取结果的方式主要有四种

```java
//同步获取结果
public T    get()
public T    get(long timeout, TimeUnit unit)
public T    getNow(T valueIfAbsent)
public T    join()
```

`getNow`有点特殊，如果结果已经计算完则返回结果或者抛出异常，否则返回给定的`valueIfAbsent`值。`join()`与`get()`区别在于`join()`返回计算的结果或者抛出一个unchecked异常(CompletionException)，而`get()`返回一个具体的异常。用`get()`方式若结果未返回，则堵塞当前线程，但为了防止长时间等待，可以设置一个超时时间。

参考: 

[CompletableFuture基本用法](https://www.cnblogs.com/cjsblog/p/9267163.html)

[JDK8新特性之CompletableFuture详解](https://www.jianshu.com/p/547d2d7761db)

[通过实例理解 JDK8 的 CompletableFuture](https://www.ibm.com/developerworks/cn/java/j-cf-of-jdk8/index.html)



