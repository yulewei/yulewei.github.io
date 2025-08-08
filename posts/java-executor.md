---
title: Java Executor 框架笔记 
date: 2017-03-23 15:09:51
categories: Java
tags: [Java, 并发]
---

本文整理 Java 并发框架 Executor 的用法，并对结合 JDK 相关的实现源码作简单分析。

# 任务与线程池

先来看下 Executor 框架的 javadoc 描述 [ [ref1](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/package-summary.html#package_description) [ref2](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/package-summary.html#package.description) ]

<!--more-->

接口：

* [Executor](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/Executor.html) 是一个简单的标准化接口，用于定义类似于线程的自定义子系统，包括线程池、异步 IO 和轻量级任务框架。根据所使用的具体 Executor 类的不同，可能在新创建的线程中，现有的任务执行线程中，或者调用 execute() 的线程中执行任务，并且可能顺序或并发执行。
* [ExecutorService](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/ExecutorService.html) 提供了多个完整的异步任务执行框架。ExecutorService 管理任务的排队和安排，并允许受控制的关闭。[ScheduledExecutorService](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/ScheduledExecutorService.html) 子接口及相关的接口添加了对延迟的和定期任务执行的支持。ExecutorService 提供了安排异步执行的方法，可执行由 [Callable](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/Callable.html) 表示的任何函数，结果类似于 Runnable。
* [Future](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/Future.html) 返回函数的结果，允许确定执行是否完成，并提供取消执行的方法。[RunnableFuture](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/RunnableFuture.html) 是拥有 run 方法的 Future，run 方法执行时将设置其结果。

实现：

* 类 [ThreadPoolExecutor](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/ThreadPoolExecutor.html) 和 [ScheduledThreadPoolExecutor](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/ScheduledThreadPoolExecutor.html) 提供可调的、灵活的线程池。
* [Executors](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/Executors.html) 类提供大多数 Executor 的常见类型和配置的工厂方法，以及使用它们的几种实用工具方法。
* 其他基于 Executor 的实用工具包括具体类 [FutureTask](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/FutureTask.html)，它提供 Future 的常见可扩展实现，以及 [ExecutorCompletionService](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/ExecutorCompletionService.html)，它有助于协调对异步任务组的处理。 

涉及到的类与接口的层次结构，如下图所示：

<img width="400" alt="Executor 相关类" src="https://static.nullwy.me/java-executor.png">
<img width="400" alt="任务相关类" src="https://static.nullwy.me/java-future.png">

## 线程池

Executor，此接口提供一种将任务提交与每个任务将如何运行的机制（包括线程使用的细节、调度等）分离开来的方法。接口只定义唯一一个方法：

``` java
void execute(Runnable command)
    在未来某个时间执行给定的命令
```

实现 Executor 接口，就是定义某种运行任务的机制。最简单的运行任务的机制是，在调用者的线程中立即运行已提交的任务，如下：

``` java
class DirectExecutor implements Executor {
    public void execute(Runnable r) {
        r.run();
    }
}
```

或者，以下实现将为每个任务生成一个新线程：

``` java
class ThreadPerTaskExecutor implements Executor {
    public void execute(Runnable r) {
        new Thread(r).start();
    }
}
```

更典型的执行任务的方式是，使用线程池（[Thread pool](https://en.wikipedia.org/wiki/Thread_pool)）。

JDK 下的 [Executors](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/Executors.html) 类提供创建线程池的静态工厂方法：

1. [newFixedThreadPool](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/Executors.html#newFixedThreadPool%28int%29)：固定大小线程池，创建一个可重用固定线程数的线程池，以共享的无界队列方式来运行这些线程
2. [newCachedThreadPool](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/Executors.html#newCachedThreadPool%28%29)：无界线程池，创建一个可根据需要创建新线程的线程池，但是在以前构造的线程可用时将重用它们
3. [newSingleThreadExecutor](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/Executors.html#newSingleThreadExecutor%28%29)：单个后台线程池，创建一个使用单个 worker 线程的 Executor，以无界队列方式来运行该线程
4. [newScheduledThreadPool](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/Executors.html#newScheduledThreadPool%28int%29)：任务调度线程池，创建一个线程池，它可安排在给定延迟后运行命令或者定期地执行

这 4 个工厂方法返回的类型是 [ExecutorService](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/ExecutorService.html)，该接口扩展自 Executor。ExecutorService 提供了管理终止的方法，以及可为跟踪一个或多个异步任务执行状况而生成 [Future](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/Future.html) 的方法。

现在让我们来看看创建线程池的静态工厂方法，对应的实现源码 [ [src](https://github.com/dmlloyd/openjdk/blob/jdk8u/jdk8u/jdk/src/share/classes/java/util/concurrent/Executors.java) ]：

``` java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>());
}
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60L, TimeUnit.SECONDS, new SynchronousQueue<Runnable>());
}
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1, 0L, TimeUnit.MILLISECONDS, new LinkedBlockingQueue<Runnable>()));
}
public static ScheduledExecutorService newScheduledThreadPool(int corePoolSize) {
    return new ScheduledThreadPoolExecutor(corePoolSize);
}
```

可以看到 `newFixedThreadPool`、`newCachedThreadPool` 和 `newSingleThreadExecutor` 内部都是通过类 `ThreadPoolExecutor` 实现。[ThreadPoolExecutor](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/ThreadPoolExecutor.html) 的构造方法的 javadoc 如下：

``` java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue)

    用给定的初始参数和默认的线程工厂及被拒绝的执行处理程序创建新的 ThreadPoolExecutor。使用 Executors 工厂方法之一比使用此通用构造方法方便得多。

    参数：
        corePoolSize - 池中所保存的线程数，包括空闲线程。
        maximumPoolSize - 池中允许的最大线程数。
        keepAliveTime - 当线程数大于核心时，此为终止前多余的空闲线程等待新任务的最长时间。
        unit - keepAliveTime 参数的时间单位。
        workQueue - 执行前用于保持任务的队列。此队列仅保持由 execute 方法提交的 Runnable 任务。 
```

ThreadPoolExecutor 的处理流程如下图所示（参考自《Java并发编程的艺术》第9章 Java中的线程池）：
<img width="700" alt="线程池的主要处理流程" src="https://static.nullwy.me/java-thread-pool-executor.jpg">
<img width="500" alt="ThreadPoolExecutor 执行示意图" src="https://static.nullwy.me/java-thread-pool-executor2.jpg">

基本上预定义的三个线程池已经满足常见的使用需求，若有特殊需求也可以，特殊构造 ThreadPoolExecutor 实例。此类提供 protected 的 [beforeExecute](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/ThreadPoolExecutor.html#beforeExecute(java.lang.Thread,%20java.lang.Runnable)) 和 [afterExecute](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/ThreadPoolExecutor.html#afterExecute(java.lang.Runnable,%20java.lang.Throwable)) 钩子 (hook) 方法，就是预留扩展用的。

[ScheduledExecutorService](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/ScheduledExecutorService.html) 类提供的方法：

``` java
ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit)
    创建并执行在给定延迟后启用的 ScheduledFuture
ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit)
    创建并执行在给定延迟后启用的一次性操作
ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit)
    创建并执行一个在给定初始延迟后首次启用的定期操作，后续操作具有给定的周期
ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit)
    创建并执行一个在给定初始延迟后首次启用的定期操作，随后，在每一次执行终止和下一次执行开始之间都存在给定的延迟
```

用法就如文档所示，不展开描述。

## 提交任务

若直接使用线程运行任务，则典型的做法是创建 [Runnable](http://www.cjsdn.net/Doc/JDK60/java/lang/Runnable.html) 接口实例，然后启动线程，如下：

``` java
Runnable r = ...
Thread worker = new Thread(r);
worker.start();
worker.join();
int result = getSavedValue();
```

Runnable 实例是没有直接的办法获取运行结果的返回值的，若要获取，需要添加额外的代码，如示例中 `getSavedValue`。

Executor 框架下，使用 ExecutorService 的 [submit](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/ExecutorService.html#submit(java.util.concurrent.Callable)) 方法提交任务。

``` java
<T> Future<T> submit(Callable<T> task)
    提交一个返回值的任务用于执行，返回一个表示任务的未决结果的 Future。该 Future 的 get 方法在成功完成时将会返回该任务的结果。 
```

Callable 接口类似于 Runnable，两者都是为那些其实例可能被另一个线程执行的类设计的。但是 Callable 会返回结果，并且可以抛出经过检查的异常。 submit 方法也可以接受 Runnable 参数，但阅读内部实现代码的话，就可以看到，最终还是会通过 Executors 类的 [callable](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/Executors.html#callable(java.lang.Runnable)) 方法，将 Runnable 转换成 Callable [ [src](https://github.com/dmlloyd/openjdk/blob/jdk8u/jdk8u/jdk/src/share/classes/java/util/concurrent/AbstractExecutorService.java#L73) [src](https://github.com/dmlloyd/openjdk/blob/jdk8u/jdk8u/jdk/src/share/classes/java/util/concurrent/FutureTask.java#L139) ]。Runnable 是为线程设计的，Callable 是为任务设计。任务和线程概念上分离，这样线程如何运用任务，即运行任务的机制，就可以按具体情况定义了。

上文提到 ExecutorService 扩展自 Executor 接口，那么现在就看下 submit 方法实现<span id="submit-src">源码</span> [ [src](https://github.com/dmlloyd/openjdk/blob/jdk8u/jdk8u/jdk/src/share/classes/java/util/concurrent/AbstractExecutorService.java#L127) ]：

``` java
public <T> Future<T> submit(Callable<T> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<T> ftask = newTaskFor(task);
    execute(ftask);
    return ftask;
}
```

可以看到内部实现其实就是调用 execute，但传入参数和返回结果包裹了 Callable 和 Future。execute 方法跟具体实现有关，对于 ThreadPoolExecutor 的实现逻辑，代码会根据线程大小，以及任务队列 [workQueue](https://github.com/dmlloyd/openjdk/blob/jdk8u/jdk8u/jdk/src/share/classes/java/util/concurrent/ThreadPoolExecutor.java#L433) 和工作线程 [worker](https://github.com/dmlloyd/openjdk/blob/jdk8u/jdk8u/jdk/src/share/classes/java/util/concurrent/ThreadPoolExecutor.java#L461) 状况，做出相应选择：[ [doc](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/ThreadPoolExecutor.html) [src](https://github.com/dmlloyd/openjdk/blob/jdk8u/jdk8u/jdk/src/share/classes/java/util/concurrent/ThreadPoolExecutor.java#L1318) ]：

 - 如果运行的线程少于 corePoolSize，则 Executor 始终首选添加新的线程，而不进行排队。
 - 如果运行的线程等于或多于 corePoolSize，则 Executor 始终首选将请求加入队列，而不添加新的线程。
 - 如果无法将请求加入队列，则创建新的线程，除非创建此线程超出 maximumPoolSize，在这种情况下，任务将被拒绝。

## 使用 ExecutorService

`ExecutorService` 示例：

``` java
package concurrent;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;

public class ExecutorMain {

    public static void main(String[] args) throws InterruptedException, ExecutionException, TimeoutException {
        ExecutorService pool = Executors.newFixedThreadPool(10);

        List<Future<Integer>> futures = new ArrayList<>();
        futures.add(pool.submit(() -> {
            Thread.sleep(2000);
            System.out.println("return 1");
            return 1;
        }));
        futures.add(pool.submit(() -> {
            Thread.sleep(1000);
            System.out.println("return 2");
            return 2;
        }));
        futures.add(pool.submit(() -> {
            Thread.sleep(3000);
            System.out.println("return 3");
            return 3;
        }));

        for (Future<Integer> future : futures) {
            Integer result = future.get();
            System.out.println("get " + result);
        }
        pool.shutdown();
    }
}
```

输出结果：

``` txt
return 2
return 1
get 1
get 2
return 3
get 3
```

为什么输出结果是这样呢？查文档知道，Future 的 [get](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/Future.html#get()) 方法，若已经完成，则直接返回，否则会等待计算完成，然后获取其结果。在示例代码中，任务1 需要 2 秒完成，任务2 需要 1 秒完成，任务3 需要 3 秒完成。自然，在线程池中完成次序是，任务2 - 任务1 - 任务3。main 主线程，先是获取 get 任务1 的执行结果，需要等待 2 秒，在去获取 get 任务2 的执行结果，此时该任务已经完成，直接返回，接下来是获取 get 任务3 的执行结果，任务3 耗时 3 秒，此时时间线是第 2 秒，所以需等 1 秒，才能获取 get 执行结果。

使用 ExecutorService，若想按任务完成次序获取执行结果，可以将代码中 for 循环，修改为轮询方式，就不断地在调用 get 前，先用 isDone 判断是否任务已经完成。代码如下：

``` java
while (true) {
    Iterator<Future<Integer>> iter = futures.iterator();
    if (!iter.hasNext()) break;
    while (iter.hasNext()) {
        Future<Integer> future = iter.next();
        if (future.isDone()) {
            Integer result = future.get();
            System.out.println("get " + result);
            iter.remove();
        }
    }
}
```

输出结果：

``` txt
return 2
get 2
return 1
get 1
return 3
get 3
```

这种实现方式，虽然可行，但相当繁琐。幸运的是，JDK 还提供一种更好的方法，[CompletionService](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/CompletionService.html)。

## 使用 CompletionService

[CompletionService](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/CompletionService.html) 将 [Executor](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/Executor.html) 和 [BlockingQueue](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/BlockingQueue.html) 的功能融合在一起。你可以将 Callable 任务提交给它来执行，然后使用类似于队列操作的 [take](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/CompletionService.html#take()) 和 [poll](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/CompletionService.html#poll()) 等方法来获得已完成的结果，而这些结果会在完成时将被封装为 Future。[ExecutorCompletionService](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/ExecutorCompletionService.html) 实现了 CompletionService。ExecutorCompletionService 的实现非常简单。在构造函数中创建一个 BlockingQueue 来保存计算完成的结果。当计算完成时，调用 FutureTask 中的 done 方法。当提交某个任务时，该任务将首先包装为一个 QueueingFuture，这是 FutureTask 一个子类，然后再改写子类的 done 方法，并将结果放入 BlockingQueue 中，take 和 poll 方法委托给了 BlockingQueue，这些方法会在得出结果之前阻塞。[ Goetz 2006, 6.3.5 ]

可以看出整个实现逻辑核心在于重写 FutureTask 的 done 方法，正如这个方法的 javadoc 所述，定义该方法的目的就是如此：

``` java
protected void done()
    当此任务转换到状态 isDone（不管是正常地还是通过取消）时，调用受保护的方法。默认实现不执行任何操作。子类可以重写此方法，以调用完成回调或执行簿记
```

再来看下，BlockingQueue 的实现源码 [ [src](https://github.com/dmlloyd/openjdk/blob/jdk8u/jdk8u/jdk/src/share/classes/java/util/concurrent/ExecutorCompletionService.java#L112) ]，以及 submit 任务是对应的实现源码 [ [src](https://github.com/dmlloyd/openjdk/blob/jdk8u/jdk8u/jdk/src/share/classes/java/util/concurrent/ExecutorCompletionService.java#L178) ]：

``` java
public Future<V> submit(Callable<V> task) {
    if (task == null) throw new NullPointerException();
    RunnableFuture<V> f = newTaskFor(task);
    executor.execute(new QueueingFuture(f));
    return f;
}
```

``` java
private class QueueingFuture extends FutureTask<Void> {
    QueueingFuture(RunnableFuture<V> task) {
        super(task, null);
        this.task = task;
    }
    protected void done() { completionQueue.add(task); }
    private final Future<V> task;
}
```

然后再看下 ExecutorCompletionService 的 take 和 poll 方法的实现源码 [ [src](https://github.com/dmlloyd/openjdk/blob/jdk8u/jdk8u/jdk/src/share/classes/java/util/concurrent/ExecutorCompletionService.java#L192) ]：

``` java
public Future<V> take() throws InterruptedException {
    return completionQueue.take();
}

public Future<V> poll() {
    return completionQueue.poll();
}
```

整个实现逻辑正如上面文字所述。

上文示例的 `ExecutorMain`，现在用 `CompletionService` 重现实现如下：

``` java
package concurrent;

import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.*;

public class CompletionServiceMain {

    public static void main(String[] args) throws InterruptedException, ExecutionException {
        ExecutorService pool = Executors.newFixedThreadPool(10);
        CompletionService<Integer> service = new ExecutorCompletionService<>(pool);

        List<Future<Integer>> futures = new ArrayList<>();
        futures.add(service.submit(() -> {
            Thread.sleep(2000);
            System.out.println("return 1");
            return 1;
        }));
        futures.add(service.submit(() -> {
            Thread.sleep(1000);
            System.out.println("return 2");
            return 2;
        }));
        futures.add(service.submit(() -> {
            Thread.sleep(3000);
            System.out.println("return 3");
            return 3;
        }));

        for (int i = 0; i < futures.size(); i++) {
            Integer result = service.take().get();
            System.out.println("get " + result);
        }
        pool.shutdown();
    }
}
```

运行结果，就是依次输出，任务2 - 任务1 - 任务3：

``` txt
return 2
get 2
return 1
get 1
return 3
get 3
```

## Guava 的 ListenableFuture

CompletionService 的逻辑是，任务完成后，依次添加到完成队列中，然后让主线程主动去获取这些已经完成的任务。其实这种主线程主动模式，也可以修为被动模式，即任务完成后主动以事件回调的形式通知主线程，主动方从主线程换成了任务本身。这种方式就消除了等待任务完成的过程。可惜的是直到 Java 8 新添加的 [CompletableFuture](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html) 才支持实现这种机制。在 Java 8 之前，需要借助第三方库，比如 Guava 提供的 [ListenableFuture](http://google.github.io/guava/releases/20.0/api/docs/com/google/common/util/concurrent/ListenableFuture.html)。使用 CompletableFuture 需要了解 lambda 以及函数式编程相关知识，本文暂不展开，本文只讨论 ListenableFuture。来看下 Guava 的 ListenableFuture 主要涉及到的类 [ [doc](https://github.com/google/guava/wiki/ListenableFutureExplained) ]：

* [ListeningExecutorService](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/ListeningExecutorService.html)：扩展自 ExecutorService 接口，但不同的是，提交 [submit](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/ListeningExecutorService.html#submit-java.util.concurrent.Callable-) 过去的任务返回的是 ListenableFuture 实例。
* [ListenableFuture](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/ListenableFuture.html)：扩展自 Future 接口，唯一新增的方法是 [addListener](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/ListenableFuture.html#addListener-java.lang.Runnable-java.util.concurrent.Executor-)，用于添加 Future 完成后的回调监听器。
* [MoreExecutors](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/MoreExecutors.html)：工具类，类似于 JDK 的 Executors，主要提供 Executor 的桥接、转换和构造的工具方法。
* [Futures](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/Futures.html)：工具类，提供 Future 接口相关的静态工具方法，如静态方法 [addCallback](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/Futures.html#addCallback-com.google.common.util.concurrent.ListenableFuture-com.google.common.util.concurrent.FutureCallback-) 用于添加任务执行的回调监听器，包括对任务执行成功的 onSuccess 事件和执行失败（抛出异常或[被取消](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/CancellationException.html)）的 onFailure 事件。
* [ListenableFutureTask](http://google.github.io/guava/releases/snapshot/api/docs/com/google/common/util/concurrent/ListenableFutureTask.html)：实现 ListenableFuture 接口的 FutureTask。

这些类如何使用呢？还是先来看下示例代码吧。

``` java
package concurrent;

import com.google.common.util.concurrent.*;
import java.util.concurrent.Executors;

public class ListenableFutureMain {

    public static void main(String[] args) {
        ListeningExecutorService service = MoreExecutors.listeningDecorator(Executors.newFixedThreadPool(10));

        ListenableFuture<Integer> task = service.submit(() -> {
            Thread.sleep(2000);
            return 42;
        });
        Futures.addCallback(task, new FutureCallback<Integer>() {
            public void onSuccess(Integer result) {
                System.out.println("onSuccess " + result);
            }
            public void onFailure(Throwable t) {
                System.out.println("onFailure");
            }
        });

        service.shutdown();
    }
}
```

简单解释下。MoreExecutors 工具类的 listeningDecorator 方法将 JDK 创建的 ExecutorService 装饰为 ListeningExecutorService。接下来，创建的任务，并提交 submit 到这个 ExecutorService 里。submit 方法返回 ListenableFuture 接口实例（内部实现其实就是 ListenableFutureTask 对象）。Futures 工具类的 addCallback 方法用于添加在任务执行回调的监听器。

类似于 JDK 的 CompletionService，实现 Guava 的 ListeningExecutorService 也是通过重写 FutureTask 的 done 方法完成的。FutureTask 对应的子类就是 ListenableFutureTask（事实上，Guava 版本 19 开始 ListenableFutureTask 类替换成立了 TrustedListenableFutureTask 类 [ [github](https://github.com/google/guava/commit/99d2816f0a099430f28b69639e9f6f5aae388812) ]，这里暂不展开分析）。实现 ListenableFutureTask 的核心代码如下 [ [src](https://github.com/google/guava/blob/v18.0/guava/src/com/google/common/util/concurrent/ListenableFutureTask.java) ]：

``` java
private final ExecutionList executionList = new ExecutionList();

@Override
public void addListener(Runnable listener, Executor exec) {
    executionList.add(listener, exec);
}

@Override
protected void done() {
    executionList.execute();
}
```

之前查看 submit 方法实现[源码](#submit-src)时，看到该方法内部会调用 newTaskFor 方法创建 FutureTask。相应的 JDK 的 AbstractExecutorService 的 [newTaskFor](http://www.cjsdn.net/Doc/JDK60/java/util/concurrent/AbstractExecutorService.html#newTaskFor(java.util.concurrent.Callable)) 也 Guava 的 AbstractListeningExecutorService 被重写为 [ [src](https://github.com/google/guava/blob/v18.0/guava/src/com/google/common/util/concurrent/AbstractListeningExecutorService.java#L45) ]：

``` java
@Override protected final <T> ListenableFutureTask<T> newTaskFor(Callable<T> callable) {
    return ListenableFutureTask.create(callable);
}
```

这样在线程池中运行任务的不再是 FutureTask，而变成了 ListenableFutureTask。任务完成后调用 done 方法，而 done 方法就去执行该任务关联的监听器列表，即 `executionList.execute()`。

再来看下 Futures 工具类的 addCallback 方法的实现 [ [src](https://github.com/google/guava/blob/v18.0/guava/src/com/google/common/util/concurrent/Futures.java#L1261) ]：

``` java
public static <V> void addCallback(final ListenableFuture<V> future,
    final FutureCallback<? super V> callback, Executor executor) {
  Preconditions.checkNotNull(callback);
  Runnable callbackListener = new Runnable() {
    @Override
    public void run() {
      final V value;
      try {
        // TODO(user): (Before Guava release), validate that this
        // is the thing for IE.
        value = getUninterruptibly(future);
      } catch (ExecutionException e) {
        callback.onFailure(e.getCause());
        return;
      } catch (RuntimeException e) {
        callback.onFailure(e);
        return;
      } catch (Error e) {
        callback.onFailure(e);
        return;
      }
      callback.onSuccess(value);
    }
  };
  future.addListener(callbackListener, executor);
}
```

ListenableFuture API 功能的完整介绍参见 [ListenableFutureExplained](https://github.com/google/guava/wiki/ListenableFutureExplained)，本文不再展开。

# Fork/Join 框架

一个大的任务可能会由多个子任务组成，比如整个任务A，由任务B 和任务C 组成，而任务B 又可以被分解为任务D 和任务E，如下图所示。任务分解组合问题，利用 ListenableFuture 的回调将子任务组合起来是一种解决办法，但还是不够优雅简洁。Java 7 引入的 Fork/Join 框架，就是以这种方式设计的。Fork/Join 框架编程的风格就是，将任务分解为多个子任务，并行执行，然后将结果组合起来，即分而治之。ExecutorService 适合解决相互独立的任务，而 Fork/Join 框架适合解决任务分解组合的情况 [ [ref](https://stackoverflow.com/q/21156599) ]。

<img width="300" alt="任务的分治" src="https://static.nullwy.me/java-task.png">

其实整个 java.util.concurrent 包是由 [JSR-166](http://g.oswego.edu/dl/concurrency-interest/) [规范](https://www.jcp.org/en/jsr/detail?id=166)引入的，Fork/Join 框架就是其中的 jsr166y。JSR-166 由 [Doug Lea](https://en.wikipedia.org/wiki/Doug_Lea) 主导，是主要设计和代码实现者，而 jsr166y 最初源自他发表于 2000 年的论文“A Java Fork/Join Framework”（[msa](https://academic.microsoft.com/#/detail/1975579741) [pdf](http://gee.cs.oswego.edu/dl/papers/fj.pdf)）。

## 工作窃取

Fork/Join 框架采用工作窃取（[work stealing](https://en.wikipedia.org/wiki/Work_stealing)）的任务调度机制 [ [ref](http://ifeve.com/a-java-fork-join-framework-3-2/) ]：

 - 每一个工作线程维护自己的调度队列中的可运行任务。
 - 队列以双端队列 deque 的形式被维护，不仅支持 LIFO（last-in-first-out 后进先出）的 push 和 pop 操作，还支持 FIFO（first-in-first-out 先进先出）的 take 操作。
 - 对于一个给定的工作线程来说，任务所产生的子任务将会被放入到工作者自己的双端队列 deque 中。
 - 工作线程使用 LIFO（最早的优先）的顺序，通过弹出任务来处理队列中的任务。
 - 当一个工作线程的本地没有任务去运行的时候，它将使用 FIFO 的规则尝试随机的从别的工作线程中拿（“窃取 steal”）一个任务去运行。
 - 当一个工作线程触及了 join 操作，如果可能的话它将处理其他任务，直到目标任务被告知已经结束（通过 isDone 方法）。所有的任务都会无阻塞的完成。
 - 当一个工作线程无法再从其他线程中获取任务和失败处理的时候，它就会退出（通过 yields, sleeps, 和/或者优先级调整）并经过一段时间之后再度尝试直到所有的工作线程都被告知他们都处于空闲的状态。在这种情况下，他们都会阻塞直到其他的任务再度被上层调用。

<img width="350" alt="work stealing" src="https://static.nullwy.me/java-work-stealing.png">

使用 LIFO 规则来处理每个工作线程的自己任务，窃取别的工作线程的任务却使用 FIFO 规则，这是一种被广泛使用的进行递归 fork/join 设计的一种调优手段。这种模式有以下两个优点：它通过窃取工作线程队列反方向的任务减少了竞争。同时，它利用了递归的分治算法越早的产生大任务这一特点。因此，更早期被窃取的任务有可能会提供一个更大的单元任务，从而使得窃取线程能够在将来进行递归分解。

这些规则的结果是，拥有相对细粒度的基本任务，比那些仅仅使用粗粒度划分或没有使用递归分解的任务运行更快。

## 使用示例

[ForkJoinPool](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html) 类是用于执行 ForkJoinTask 的 ExecutorService。在构造过程中，可以在构造函数中指定线程池的大小。如果使用的是默认的无参构造函数，那么会创建大小等同于可用处理器数量的线程池。ForkJoinPool 类的典型方法：

``` java
public void execute(ForkJoinTask<?> task)
  异步，不需要等待计算结果，只是将任务提交给线程池
public <T> T invoke(ForkJoinTask<T> task)
  同步，等待计算结束，并返回计算结果值
public <T> ForkJoinTask<T> submit(ForkJoinTask<T> task)
  异步，不需要等待计算结果，只是将任务提交给线程池
```

提交到 ForkJoinPool 中的任务由 [ForkJoinTask](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinTask.html) 抽象类表示。ForkJoinTask 的实现子类有 [RecursiveAction](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/RecursiveAction.html) 和 [RecursiveTask](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/RecursiveTask.html)，以及 Java 8 新增的 [CountedCompleter](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CountedCompleter.html)。RecursiveAction 用于没有返回结果的任务，RecursiveTask 用于返回结果的任务。CountedCompleter 用于完成动作会触发另外一个动作的任务 [ [javadoc](http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinTask.html) ]。

计算斐波那契数（[Fibonacci number](https://en.wikipedia.org/wiki/Fibonacci_number)）是一个经典的分而治之的问题。Doug Lea 的论文以及 RecursiveTask 的 javadoc 都以斐波那契数为例进行说明。斐波那契数定义公式为：

$$F_{n}=F_{n-1}+F_{n-2}$$
$$F_{0}=0, F_{1}=1$$

使用 Fork/Join 框架计算斐波那契数的示例代码如下：

``` java
package concurrent;

import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.RecursiveTask;

public class ForkJoinMain {

    public static class Fibonacci extends RecursiveTask<Integer> {
        final int n;

        public Fibonacci(int n) {
            this.n = n;
        }

        protected Integer compute() {
            if (n <= 1)
                return n;
            Fibonacci f1 = new Fibonacci(n - 1);
            Fibonacci f2 = new Fibonacci(n - 2);
            f1.fork();
            return f2.compute() + f1.join();
        }
    }

    public static void main(String[] args) throws Exception {
        ForkJoinPool pool = new ForkJoinPool();
        Fibonacci fib = new Fibonacci(3);
        pool.invoke(fib); // 等待计算完成
        Integer result = fib.get();
        System.out.println(result);
    }
}
```

这个计算过程如下图所示：

![fork-join](https://static.nullwy.me/java-fork-join.png)



# 参考资料

1. Java 并发编程实战，Goetz，2006；第6章 任务执行 <https://book.douban.com/subject/10484692/>
2. Java并发编程的艺术，方腾飞，2015 <https://book.douban.com/subject/26591326/>
2. ExecutorService vs ExecutorCompletionService in Java <https://dzone.com/articles/executorservice-vs>
3. Java's Fork/Join vs ExecutorService - when to use which? <https://stackoverflow.com/q/21156599>
4. Java Fork Join框架 (三) 设计 <http://ifeve.com/a-java-fork-join-framework-3-2/>



