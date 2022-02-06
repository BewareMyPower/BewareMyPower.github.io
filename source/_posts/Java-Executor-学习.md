---
title: Java Executor 学习
date: 2022-02-06 21:25:00
tags:
  - Java
top: 1
---

[toc]

## Executor 简介

在 Java 中，更偏向于使用 `Executor` 而非 `Thread` 来执行任务。`Executor` 接口定义如下：

```java
public interface Executor {
    void execute(Runnable command);
}
```

类似于 `Thread`，它可以用来在线程中执行一个 `Runnable`（任务），具体的执行策略取决于具体实现。比如以下实现就是每个任务都新建一个线程来执行：

```java
Executor executor = command -> new Thread(command).start()
```

在标准库中，`Executors` 类提供了若干 static 方法可用于构造不同类型的 `Executor`，它们在线程池中取出线程来执行任务，对于**已提交（调用 `execute` 或 `submit` 方法执行）**的任务，一般有三种状态：

- 已经完成：任务已经由池中的某个线程执行完毕，对应线程已返回给池中
- 运行中：已经分配了线程执行任务
- 等待执行：由于线程分配策略限制（比如限制了同时运行的线程数量上限），任务被缓存到内部队列，等待被执行。

## ExecutorService

### Executor 的生命周期

JVM 会在所有非守护线程全部终止后才会退出，比如以下代码：

```java
final Executor executor = Executors.newSingleThreadExecutor(); // 该线程池仅包含一个可用线程
executor.execute(() -> System.out.println(Thread.currentThread().getName() + " done"))
```

打印 `pool-1-thread-1 done` 后会卡住，查看线程栈会看到：

```
"pool-1-thread-1" #11 prio=5 os_prio=31 tid=0x00007fa04392a000 nid=0xa603 waiting on condition [0x0000700004bd6000]
   java.lang.Thread.State: WAITING (parking)
	at sun.misc.Unsafe.park(Native Method)
	- parking to wait for  <0x000000076b1778a0> (a java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject)
	at java.util.concurrent.locks.LockSupport.park(LockSupport.java:175)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer$ConditionObject.await(AbstractQueuedSynchronizer.java:2039)
	at java.util.concurrent.LinkedBlockingQueue.take(LinkedBlockingQueue.java:442)
	at java.util.concurrent.ThreadPoolExecutor.getTask(ThreadPoolExecutor.java:1074)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1134)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
```

进入 `ThreadPoolExecutor#getTask` 内部看到相应代码：

```java
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take(); // 卡在这里
```

`workQueue` 也就是前文所说的保存任务的工作队列：

```java
    private final BlockingQueue<Runnable> workQueue;
```

可见线程池 `ThreadPoolExecutor` 都会尝试从 `workQueue` 中取出任务然后分配给线程来执行，见 `runWorker` 方法：

```java
    final void runWorker(Worker w) {
        /* ... */
        try {
            while (task != null || (task = getTask()) != null) {
```

`Executors` 类创建的实际上是 `ExecutorService` 类型，它继承自 `Executor` 接口，提供了对 `Executor` 生命周期的管理。当然，此外还提供了 `submit` 接口来基于 `Future` 对任务进行管理。

### shutdown

```java
void shutdown();
```

该方法会使 executor 等待所有已提交的任务运行完成。包括在工作队列中的任务。注意该方法并不会等待。

```java
        ExecutorService executor = Executors.newSingleThreadExecutor();
        for (int i = 0; i < 3; i++) {
            final int index = i;
            executor.execute(() -> {
                println(index + " started...");
                sleep(1000);
                println(index + " stopped.");
            });
        }
        executor.shutdown();
        println("Executor shutdown: " + executor.isShutdown());
        try {
            executor.execute(() -> {});
        } catch (RejectedExecutionException e) {
            println("Failed to execute task after shutdown: " + e.getMessage());
        }
```

> 注：上述代码使用了我自省添加的 `println` 方法来打印时间戳和线程名，以及 `sleep` 方法吞掉 `InterruptedException`。
>
> ```java
>     private static void println(String x) {
>         System.out.println(System.currentTimeMillis() + " " + Thread.currentThread().getName() + " | " + x);
>     }
>     
>     private static void sleep(long millis) {
>         try {
>             Thread.sleep(millis);
>         } catch (InterruptedException ignored) {
>         }
>     }
> ```

运行结果：

```
1644147267909 pool-1-thread-1 | 0 started...
1644147267909 main | Executor shutdown: true
1644147267911 main | Failed to execute task after shutdown: Task jcip.Main$$Lambda$2/598446861@619a5dff rejected from java.util.concurrent.ThreadPoolExecutor@1ed6993a[Shutting down, pool size = 1, active threads = 1, queued tasks = 2, completed tasks = 0]
1644147268913 pool-1-thread-1 | 0 stopped.
1644147268913 pool-1-thread-1 | 1 started...
1644147269915 pool-1-thread-1 | 1 stopped.
1644147269916 pool-1-thread-1 | 2 started...
1644147270916 pool-1-thread-1 | 2 stopped.
```

可以看到 `shutdown()` 立刻返回了，但是 JVM 进程是等待所有线程退出后才结束。但由于 `shutdown` 方法被调用，executor  会拒绝接受新的任务，因此在调用 `execute` 时会抛出 `RejectedExecutionException` 异常，包含了线程池 `ThreadPoolExecutor` 的具体信息：

```
Shutting down, pool size = 1, active threads = 1, queued tasks = 2, completed tasks = 0
```

线程池已经关闭，池的大小为 1，活跃线程数为 1，排队的任务为 2，已经完成的任务数量为 0。

### awaitTermination

由于 `shutdown` 并不会等待 executor 关闭，因此 `ExecutorService` 还提供了 `awaitTermination` 方法进行等待。

```java
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;
```

若在 timeout 范围内 executor 已经停止，则返回 true。否则返回 false，即等待超时。因此可以轮询调用该方法来等待 executor 停止。这里修改前一节调用 `shutdown` 方法之后的代码如下所示：

```java
        println("Executor terminated: " + executor.isTerminated());
        while (!executor.awaitTermination(10000, TimeUnit.MILLISECONDS)) {
            // No ops
        }
        println("Executor terminated: " + executor.isTerminated());
```

相关打印信息：

```
1644148189276 main | Executor terminated: false
1644148192288 main | Executor terminated: true
```

从时间戳之差（3008 毫秒）可见等待不到 10 秒（timeout）就完成了。注意这里还调用了 `isTerminated` 方法，当所有任务都结束时该方法会返回 true。

### shutdownNow

`shutdown` 是优雅的关闭，如果担心有的任务是有 bug 的，会一直卡住，导致 JVM 进程无法终止，此时需要用一种粗暴的关闭方式，也就是 `shutdownNow`。

```java
    List<Runnable> shutdownNow();
```

~~我初看这个方法时比较迷惑，以为 `shutdownNow` 是异步关闭，而 `shutdown` 是同步关闭。~~实际上，`shutdown` 会等待所有任务完成，只不过不再接受新的任务。而 `shutdownNow` 则是**取消**所有运行中的任务，并且不再启动等待中的任务。

```java
        ExecutorService executor = Executors.newSingleThreadExecutor();
        for (int i = 0; i < 3; i++) {
            final int index = i;
            executor.execute(() -> {
                println(index + " started...");
                try {
                    Thread.sleep(1000);
                    println(index + " stopped.");
                } catch (InterruptedException e) {
                    println(index + " is cancelled");
                }
            });
        }
        executor.shutdownNow();
        executor.awaitTermination(1, TimeUnit.MINUTES);
```

运行结果：

```
1644148971844 pool-1-thread-1 | 0 started...
1644148971844 pool-1-thread-1 | 0 is cancelled
```

中断状态的线程会被取消，因此抛出 `InterruptedException`，而排队的两个任务则干脆没执行。

## 线程池 ThreadPoolExecutor

前面一直使用了单线程的线程池，它创建的 executor 类型实际上是 `ThreadPoolExecutor`。

```java
    public static ExecutorService newSingleThreadExecutor() {
        return new FinalizableDelegatedExecutorService // 仅仅是个 wrapper，finalize() 方法会调用 shutdown()
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>()));
    }
```

```java
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue) {
        this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
             Executors.defaultThreadFactory(), defaultHandler);
    }
```

### 线程池的配置

包括前两节提到的 `ThreadFactory` 和 `RejectedExecutionHandler`，`ThreadPoolExecutor` 构造参数及其作用如下所示：

| 参数名            | 类型                       | 含义                                                         |
| ----------------- | -------------------------- | ------------------------------------------------------------ |
| `corePoolSize`    | `int`                      | 核心线程数，即保持运行的线程数，即使处于空闲状态，除非 `allowCoreThreadTimeout` 被设置 |
| `maximumPoolSize` | `int`                      | 线程池最大允许创建的线程数                                   |
| `keepAliveTime`   | `long`                     | 当线程数超过核心线程数数量时，闲置线程在终止前等待新任务的最大时间 |
| `unit`            | `TimeUnit`                 | `keepAliveTime` 对应的时间单位                               |
| `workQueue`       | `BlockingQueue<Runnable>`  | 持有待执行的任务的队列                                       |
| `threadFactory`   | `ThreadFactory`            | 提供 `newThread` 接口，可在通过 `Runnable`创建线程时设置线程的一些信息（比如名字） |
| `handler`         | `RejectedExecutionHandler` | 当线程池到达上限时，新任务到来的处理策略。默认是抛出 `RejectedExecutionException` |

常见的几种快速创建线程池（用静态方法去掉 `new` 前缀）的参数：

|                        | `corePoolSize` | `maximumPoolSize`   | `keepAliveTime` | `workQueue`           |
| ---------------------- | -------------- | ------------------- | --------------- | --------------------- |
| `FixedThreadPool`      | nThreads       | nThreads            | 0               | `LinkedBlockingQueue` |
| `SingleThreadExecutor` | 1              | 1                   | 0               | `LinkedBlockingQueue` |
| `CachedThreadPool`     | 0              | `Integer.MAX_VALUE` | 60 s            | `SynchronousQueue`    |

上述几种典型的配置各有缺陷，比如 `CachedThreadPool` 无法限制线程数量，容易导致线程创建太多而 OOM。而另外两种配置则限制死了线程的最大数量。

最核心的参数是 `corePoolSize` 和 `maximumPoolSize`。当线程池的线程数不大于 `corePoolSize` 时，对于新的任务都会创建线程。但是如果正在运行的线程数量达到了 `corePoolSize`，新任务到来时则会根据 `maximumPoolSize` 和 `workQueue` 来决定行为：

- `workQueue` 未满：加入队列
- `workQueue` 已满：
  - 若正在运行的线程数小于 `maximumPoolSize`，则创建新的线程执行任务
  - 否则抛出 `RejectedExecutionException`

这里给出一段示例配置的行为表现：

```java
        final ExecutorService executor = new ThreadPoolExecutor(
                1, // corePoolSize
                3, // maximumPoolSize
                1, // keepAliveTime
                TimeUnit.SECONDS, // unit
                new ArrayBlockingQueue<>(1) // workQueue
                );
        for (int i = 0; i < 5; i++) {
            final int index = i;
            try {
                executor.execute(() -> {
                    println(index + " started...");
                    sleep(2000);
                    println(index + " stopped.");
                });
            } catch (RejectedExecutionException e) {
                println(index + " rejected: " + e.getMessage());
            }
        }
        executor.shutdown();
        while (!executor.awaitTermination(1, TimeUnit.SECONDS)) {
            // No ops
        }
```

运行结果：

```
1644155259100 pool-1-thread-1 | 0 started...
1644155259101 main | 4 rejected: Task jcip.Main$$Lambda$1/758705033@4534b60d rejected from java.util.concurrent.ThreadPoolExecutor@3fa77460[Running, pool size = 3, active threads = 3, queued tasks = 1, completed tasks = 0]
1644155259100 pool-1-thread-3 | 3 started...
1644155259100 pool-1-thread-2 | 2 started...
1644155261104 pool-1-thread-1 | 0 stopped.
1644155261104 pool-1-thread-2 | 2 stopped.
1644155261104 pool-1-thread-3 | 3 stopped.
1644155261104 pool-1-thread-1 | 1 started...
1644155263107 pool-1-thread-1 | 1 stopped.
```

因为 `println` 不是线程安全的，因此打印法航了乱序，但从时间戳来看，顺序为：

1. 提交任务 0，新建线程 `pool-1-thread-1` 来执行，此时活跃线程数到达了 `corePoolSize`。
2. 提交任务 1，进入大小为 1 的 `ArrayBlockingQueue` 中，此时队列已满。
3. 提交任务 2，新建线程 `pool-1-thread-2` 来执行。
4. 提交任务 3，新建线程 `pool-1-thread-3` 来执行，此时活跃线程数到达了 `maximumPoolSize`。
5. 提交任务 4，抛出 `RejectedExecutionException`。
6. 2 秒后，任务 0，2，3 执行完毕，队列中的任务 1 弹出，由线程 `pool-1-thread-1` 执行。

### Keep Alive

`keepAliveTime` 和 `unit` 参数更多的是控制闲置的非核心线程数的等待时间。闲置线程即没有执行任务的线程，非核心线程，则是相对于核心线程而言的。比如若 `corePoolSize` 为 1，`maximumPoolSize` 为 3，那么如果因为提交任务数量比较多，导致创建了 3 个线程，这多出的 2 个线程都是非核心线程。

这里仅从代码角度分析见 `ThreadPoolExecutor#getTask`：

```java
            // 必须对 ThreadPoolExecutor 调用 allowCoreThreadTimeOut 方法将同名参数设为 true 才会启用 keepAliveTime 检查
            // wc 为工作线程数量，因此只有大于 corePoolSize 才会检查，因为只有超过核心运行线程数量的线程才会被当成多余的线程
            boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

            if ((wc > maximumPoolSize || (timed && timedOut))
                && (wc > 1 || workQueue.isEmpty())) {
                if (compareAndDecrementWorkerCount(c)) // [2] 回收多余线程，减少工作线程数量
                    return null;
                continue;
            }

            try {
                Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
                if (r != null)
                    return r;
                timedOut = true; // [1] 在 keepAliveTime 内没有任务，则置为 true
            } catch (InterruptedException retry) {
                timedOut = false;
            }
```

注意这里的 `keepAliveTime` 在构造时已经转换成了纳秒单位：

```java
        this.keepAliveTime = unit.toNanos(keepAliveTime);
```

上面代码有个比较诡异的地方，就是什么时候 `wc` 会超过最大线程数量。查看方法注释，可知如果 `setMaximumPoolSize` 被调用，那么 `maximumPoolSize` 会发生动态变化，此时会导致 worker 数超过了  `maximumPoolSize`。

### ThreadFactory

```java
public interface ThreadFactory {
    Thread newThread(Runnable r);
}
```

在 `ThreadPoolExecutor` 中，默认的线程工厂是：

```java
    public static ThreadFactory defaultThreadFactory() {
        return new DefaultThreadFactory();
    }
```

```java
        DefaultThreadFactory() {
            SecurityManager s = System.getSecurityManager();
            group = (s != null) ? s.getThreadGroup() :
                                  Thread.currentThread().getThreadGroup();
            namePrefix = "pool-" +
                          poolNumber.getAndIncrement() +
                         "-thread-";
        }

        public Thread newThread(Runnable r) {
            Thread t = new Thread(group, r,
                                  namePrefix + threadNumber.getAndIncrement(),
                                  0);
            if (t.isDaemon())
                t.setDaemon(false); // 对于守护线程，也要将其改成用户线程
            if (t.getPriority() != Thread.NORM_PRIORITY)
                t.setPriority(Thread.NORM_PRIORITY); // 默认线程优先级是 NORM
            return t;
        }
```

它首先通过系统的 `SecurityManager` 是否存在来设置线程组。线程名则是 `pool-<id>-thread-<thread-id>`，其中 `id` 为线程池 id，从 1 开始递增，`thread-id` 为线程 id，也从 1 开始递增。

运行以下代码：

```java
        ExecutorService executor1 = Executors.newSingleThreadExecutor();
        ExecutorService executor2 = Executors.newSingleThreadExecutor();
        executor1.execute(() -> println("executor 1 thread 1"));
        executor2.execute(() -> println("executor 2 thread 1"));
        executor2.execute(() -> println("executor 2 thread 2"));
        executor1.shutdown();
        executor2.shutdown();
```

运行结果：

```
1644149652981 pool-1-thread-1 | executor 1 thread 1
1644149652981 pool-2-thread-1 | executor 2 thread 1
1644149652981 pool-2-thread-1 | executor 2 thread 2
```

从打印出的线程名就可以看出对应关系。常见的自定义 `ThreadFactory` 的场景往往是要单独给某些 executor 的线程进行标识，从而在调试的时候区分线程。当然，由于 `newThread` 方法是可定制的，也可以定制更多线程策略。

在 `Executors` 的方法中，一般都会有重载形式来支持指定 `ThreadFactory`，比如：

```java
    public static ExecutorService newSingleThreadExecutor(ThreadFactory threadFactory) {
        return new FinalizableDelegatedExecutorService
            (new ThreadPoolExecutor(1, 1,
                                    0L, TimeUnit.MILLISECONDS,
                                    new LinkedBlockingQueue<Runnable>(),
                                    threadFactory));
    }
```

### RejectedExecutionHandler

在前文提到 `shutdown` 方法时，我们知道对于一个已关闭的 executor，若强行执行新的任务会抛出 `RejectedExecutionException` 异常。这里 `RejectedExecutionHandler` 则是支持定制化此时的行为。默认策略是 `AbortPolicy`。

```java
    private static final RejectedExecutionHandler defaultHandler =
        new AbortPolicy();
```

```java
    public static class AbortPolicy implements RejectedExecutionHandler {
        public void rejectedExecution(Runnable r, ThreadPoolExecutor e) {
            throw new RejectedExecutionException("Task " + r.toString() +
                                                 " rejected from " +
                                                 e.toString());
        }
    }
```

不过 `Executors`的方法并没有支持自定义 `RejectedExecutionHandler`，因为抛出异常的处理方式基本上是很痛用了，一般没必要定制。

## 使用 submit 执行任务

`ExecutorService` 在 `Executor` 的基础上增加了 `submit` 方法，它会返回一个 `Future`，因此我们可以用 `Future` 来管理任务状态以及任务的返回值。对于有返回值的任务，`submit` 方法往往更实用。

```java
    private static final List<Integer> DATA = prepareData();

    private static List<Integer> prepareData() {
        final List<Integer> data = new ArrayList<>();
        for (int i = 0; i < 100000; i++) {
            data.add(i);
        }
        return data;
    }

    public static void main(String[] args) throws InterruptedException, ExecutionException {
        final int numThreads = 5;
        final int sizePerThread = DATA.size() / numThreads;
        final List<Future<Long>> futures = new ArrayList<>();
        final ExecutorService executor = Executors.newFixedThreadPool(numThreads);
        for (int i = 0; i < numThreads; i++) {
            final int startIndex = i * sizePerThread;
            final int endIndex = (i < numThreads - 1) ? (startIndex + sizePerThread) : DATA.size();
            futures.add(executor.submit(() -> {
                long sum = 0;
                for (int j = startIndex; j < endIndex; j++) {
                    sum += DATA.get(j);
                }
                return sum; // 该 lambda 返回 Long，对应 Callable<Long>，因此 submit 返回的是 Future<Long>
            }));
        }
        long sum = 0;
        for (Future<Long> future : futures) {
            sum += future.get();
        }
        System.out.println(sum);
        executor.shutdown();
    }
```

上述代码则是将 100000 个数据的 `DATA` 分成 5 块，每块由一个任务来求和，每个任务对应一个 `Future`，最后将结果汇总求和。

## 总结和展望

本文主要从 `Executor` 谈到最基本的线程池（`ThreadPoolExecutor`）的使用。

除此之外，以下相关内容并不属于本文的讨论范畴，但简单提及下。

- Java 7 引入了基于 work-stealing 的 executor（使用 `newWorkStealingPool` 创建，类型为 `ForkJoinPool`），比起普通的线程池并行化更高，它的原理是用双端队列（deque）保存任务，并且可以有多个队列保存任务，这样在一个队列任务执行完毕后，还可以从其他队列的尾端来**窃取**工作线程。
- 对于延时任务或者定时任务，应该使用 `ScheduledThreadPoolExecutor` 和 `schedule()` 方法。相比起用 `Timer` 和 `schedule()` 而言，不受系统时钟变化的影响，而且能够正确处理抛出 unchecked exception 的任务。
