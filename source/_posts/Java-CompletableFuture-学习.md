---
title: Java CompletableFuture 学习
date: 2020-10-07 21:35:08
tags:
  - Java
top: 1
---

[toc]

## 前言

看了一段时间 Pulsar 代码，令人印象最深的就是整个异步处理都是基于 Java 8 引入的 `CompletableFuture`。作为 C++er 相对而言更为熟悉普通的 Future 和 Promise。Future 表示一个异步任务，即在 **未来（future）** 某个时间点完成的任务，而 Promise 则表示这个 `Future` 的完成状态（是否完成，以及完成结果正常的还是异常的）。因此 Promise 是绑定于某个 Future 的，一般而言，即可以等待 Future 自然完成，也可以提前操作 Promise 使得 Future 提前完成。

Java 1.5 引入了 `Future`  接口，但过于简单。Java 8 提供`CompletableFuture`（下文简称 future）用起来则更像 Future 和 Promise 的结合，本文将通过简单的例子来稍微系统地学习下 `CompletableFuture`。

## 状态的访问和设置

首先给出一个主线程使用 `CompletableFuture` 的代码：

```java
        // 1. 创建一个 future，其返回结果是 Integer
        CompletableFuture<Integer> future = new CompletableFuture<>();
        System.out.println(future.getNow(-1)); // -1
        // 2. 使这个 future 完成
        future.complete(10);
        System.out.println(future.getNow(-1)); // 10
```

涉及到的方法：

```java
public T getNow(T valueIfAbsent);
public boolean complete(T value);
```

其中，`getNow()` 方法表示取得 Future 完成结果（如果未完成则返回 `valueIfAbsent`），默认创建的 future 是未完成的，因此第一次打印结果是 -1，第二次打印时由于调用了 `complete()` 方法使 future 完成了，因此 `getNow()` 取得了 `value` 并返回。

对比一般的 `Future`，`CompletableFuture` 实现了完成状态的访问和设置，而 `Future` 只能通过 `get` 方法被动等待 future 完成（可以无限等待也可以设置超时），无法设置状态，要取得瞬时状态只能调用 `isDone()` 来判断是否完成，若完成，则可以安心调用 `get()` 避免阻塞。

另外，除了设置返回结果（正常状态）外，还可以抛出异常（异常状态）：

```java
        CompletableFuture<Integer> future = new CompletableFuture<>();
        // 主动设置 future 为异常完成状态
        future.completeExceptionally(new RuntimeException("failed"));
        try {
            System.out.println(future.getNow(-1));
        } catch (Exception e) {
            System.out.println("Caused by " + e.getCause());
        }
```

涉及到的方法：

```java
    public boolean completeExceptionally(Throwable ex);
```

## 在多线程的使用

一般的异步任务都会借助多线程来实现，避免主线程阻塞。类似于 `ExecutorService` 的  `execute()` 和 `submit()` 方法，`CompletableFuture` 也提供两种方法来分别执行无返回值和有返回值的任务：

```java
    public static CompletableFuture<Void> runAsync(Runnable runnable) {
        return asyncRunStage(asyncPool, runnable);
    }

    public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
        return asyncSupplyStage(asyncPool, supplier);
    }
```

其中  `Supplier` 和 `Callable` 类似，都是实现了 `T get()` 的接口，区别在于前者没有异常声明。

注意到这里都用到了 `asyncPool`：

```java
    private static final boolean useCommonPool =
        (ForkJoinPool.getCommonPoolParallelism() > 1);

    /**
     * Default executor -- ForkJoinPool.commonPool() unless it cannot
     * support parallelism.
     */
    private static final Executor asyncPool = useCommonPool ?
        ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();
```

在合适的时候会使用 `ForkJoinPool` 框架去调度 future 对应的异步任务，也就是 future 和 thread 不一定是一对一的关系，从而支持了大量异步任务的调度。

PS： `ForkJoinPool` 并不是本文的重点，之后再去学习。

而 `runAsync` 和 `supplyAsync` 也都支持自定义 executor 的形式，即根据合适的场景提供自定义的线程池：

```java
    public static CompletableFuture<Void> runAsync(Runnable runnable,
                                                   Executor executor);
    public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier,
                                                       Executor executor);
```

给出示例：

```java
        CompletableFuture<Void> future1 = CompletableFuture.runAsync(() -> System.out.println("future1 done"));
        CompletableFuture<Void> future2 = CompletableFuture.supplyAsync(() -> {
            System.out.println("future2 done");
            return null;
        });
        CompletableFuture<Integer> future3 = CompletableFuture.supplyAsync(() -> 100);
```

这里特地给出 `supplyAsync` 创建 `CompletableFuture<Void>` 的情况，由于 `Supplier` 接口的方法具有返回类型 `T`，因此必须显式返回 `null`。这也是我从 Scala 转向 Java 时一个比较不适应的地方，因为 Scala 不需要 `return` 语句就能返回结果（即最后一个表达式的返回值），而 Java 由于语法限制，这里必须显式返回。当然，直接创建 `Completable<Void>` 的时候用 `runAsync` 就好了，但是对于多个 future 的链式调用，有时候必须传入 `Supplier`，因此了解这点也很重要。

另一个常见用法是兼容旧接口，比如原来的 API 是通过回调来表示任务完成的，比如：

```java
public class AsyncTask {
    public void run(Consumer<String> callback) {
        final String result = compute();
        if (callback != null) {
            callback.accept(result);
        }
    }

    private String compute() {
        /* 耗时操作... */
        return "hello";
    }
}
```

那么，我们只要在 `Runnable` 中设置 future 状态即可：

```java
        ExecutorService executor = Executors.newSingleThreadExecutor();
        CompletableFuture<String> future = new CompletableFuture<>();
        executor.execute(() -> new AsyncTask().run(future::complete));
```

## 多任务链式调用

### thenApply

相比直接创建线程并结合 `join()`/`detach()` 这种方法来执行异步任务，future 最为灵活的一点就是支持多个任务的组合，比如现在有个任务：

1. 进行服务发现；
2. 连接刚才发现的服务。

第二步依赖于第一步的成功返回，执行同步任务时直接顺序写下来就行，但执行异步任务时，比如：

```java
CompletableFuture<String> serviceDiscoveryFuture = CompletableFuture.supplyAsync(() -> {
    /* 进行服务发现... */
    return "http://localhost:8080"; // 返回发现的 URL
});
```

我们不能在这里直接等待 `serviceDiscoveryFuture` 完成，因为这样相当于异步操作就变成同步操作了，可能阻塞当前线程。

`CompletableFuture` 提供了一系列方法来进行链式调用，比如上面的任务可以写成：

```java
        CompletableFuture<String> connectServiceFuture = CompletableFuture.supplyAsync(() -> {
            /* 1. 服务发现... */
            return "http://localhost:8080";
        }).thenApply((serverUrl) -> {
            /* 2. 连接服务... */
            System.out.println("Connected to " + serverUrl);
            return null;
        });
```

这样，只有第一个异步任务成功返回时才会将返回结果作为输入参数应用于第二个任务。

看看 `thenApply` 的方法签名：

```java
    public <U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn);
```

传入的是 `Function` 对象，也就是输入参数类型可以是 `T`（当前 future 的返回类型）的派生类，而返回类型则是 `U` 的派生类，通过 lambda 表达式中 `return` 的结果进行推导。

### thenCompose

在一些异步任务已经被封装成返回 future 的方法时，`thenApply` 无法复用这些方法。

在前一节的连接服务为例的基础上，用一个整型状态码表示连接的服务（返回 0 代表成功）：

```java
    public static CompletableFuture<String> discoverServiceAsync() {
        /* 服务发现... */
        return CompletableFuture.completedFuture("http://localhost:8080");
    }

    public static CompletableFuture<Integer> connectServiceAsync(String serviceUrl) {
        /* 连接服务... */
        System.out.println("Connected to " + serviceUrl);
        return CompletableFuture.completedFuture(1);
    }
```

PS：这里用了静态方法 `completedFuture` 来得到已完成的 future（默认构造的 future 是未完成的）。

如果我们要添加第三步：对状态码进行处理，由于现有的方法都是返回 `CompletableFuture<T>` 而非 `T`，因此 `thenApply` 此时无能为力：

```java
        CompletableFuture<Void> future = discoverServiceAsync().thenAccept(serviceUrl -> {
            CompletableFuture<Integer> statusFuture = connectServiceAsync(serviceUrl);
            // NOTE: 无法处理 statusFuture
        });
```

此时 `thenCompose` 就有用武之地了：

```java
        CompletableFuture<Integer> future = discoverServiceAsync().thenCompose((serviceUrl) -> {
            /* ... */
            return connectServiceAsync(serviceUrl);
        });
```

再看看其方法签名：

```java
    public <U> CompletableFuture<U> thenCompose(
        Function<? super T, ? extends CompletionStage<U>> fn) 
```

`CompletionStage` 接口表示异步处理的一个 **阶段（stage）**，而 `CompletableFuture` 实现了该接口（另一个实现的接口是 `Future`），因此该 `Function` 可以传入返回 `CompletableFuture` 的 lambda 表达式。

也就是说 `thenCompose` 可以将返回的 stage 作为当前 future 的新 stage。上面得到的 `future` 会等到 `connectServiceAsync` 返回的 `CompletableFuture` 完成时才算完成。

### 异常处理

前两节都是讨论 future 正常完成时的链式调用，虽然异常完成时在 future 调用 `get()` 时会抛出异常，但是有时候会对异常进行处理，比如：

1. 进行服务发现；
2. 若发现成功，连接服务；
3. 否则使用本地 HTTP 服务 `http://localhost:8080`。

示例代码：

```java
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        final String name = (args.length > 0) ? args[0] : "unknown";
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            if (name.equals("pulsar")) {
                return "pulsar://localhost:6650";
            } else {
                throw new RuntimeException("Unknown server name: " + name);
            }
        }).exceptionally(e -> {
            System.out.println(e.getCause() + ", use HTTP service");
            return "http://localhost:8080";
        });
        System.out.println(future.get());
    }
```

还是看看方法签名：

```java
    public CompletableFuture<T> exceptionally(Function<Throwable, ? extends T> fn) 
```

输入参数是抛出的异常，而返回类型则仍然是 future 泛型 `T` 的派生类。

如果并不想对异常进行一些挽救的处理，只想打印下日志，那么可以返回 `null`，因为任何类型都可以被赋值为 `null`，此时推断的类型仍为 `T`。但对于这种情况，除了结合 `thenApply` 和 `exceptionally`，也可以使用 `whenComplete`：

```java
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            if (name.equals("pulsar")) {
                return "pulsar://localhost:6650";
            } else {
                throw new RuntimeException("Unknown server name: " + name);
            }
        }).whenComplete((serviceUrl, e) -> {
            if (e == null) {
                System.out.println("Connected to " + serviceUrl);
            } else {
                System.out.println("Failed to connect to " + name + ": " + e.getCause());
            }
        });
```

看看方法签名：

```java
    public CompletableFuture<T> whenComplete(
        BiConsumer<? super T, ? super Throwable> action)
```

传入的是接受返回结果和异常的 `BiConsumer`，缺点是无法针对异常进行挽救，future 调用 `get()` 方法仍然会抛出异常，而在 `exceptionally` 中对异常进行处理并返回 `null` 的话，之后 future 调用 `get()` 不会抛出异常，而只是返回 `null`。

## 合并并行异步任务的结果

链式调用解决的是串行异步任务，也就是一个异步任务的启动要依赖另一个异步任务的结果，但有时候会遇到并行的异步任务，比如对数组分块求和：

```java
        int[] array = {1, 2, 3, 4, 5, 6};
        int[] indexes = {0, array.length / 2, array.length};
        List<CompletableFuture<Integer>> futures = new ArrayList<>();
        for (int i = 0; i < indexes.length - 1; i++) {
            final int start = indexes[i];
            final int end = indexes[i + 1];
            futures.add(CompletableFuture.supplyAsync(() -> {
                int sum = 0;
                for (int j = start; j < end; j++) {
                    sum += array[j];
                }
                return sum;
            }));
        }
```

现在要等待多个 `future` 完成，如果我们设置了超时时间比如 1 秒，那么要得到的结果实际上是 `List<Integer>`，此时可以用 `allOf` 方法将 future 列表转换成结果列表：

```java
CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).get();
List<Integer> results =
    futures.stream().map(future -> future.getNow(-1)).collect(Collectors.toList());
// results: [6, 15]
```

由于 `allOf` 返回的是 `CompletableFuture<Void>`，因此只能用来等待所有 future 完成或者其中一个失败。将结果重新组合成 `List` 还是得另外执行，看起来 Java 没有 Scala 的 `Future#sequence` 这样方便的方法。

类似地，`anyOf` 则是等待任意一个 future 完成，这里就不给例子了。

## 总结

本文简单学习了 Java 中 `CompletableFuture` 常用的使用场景，由于语法本身的限制，比起 Scala 的 `Future` 还是略显麻烦，但至少 `CompletableFuture` 的出现使得异步编程变得更加容易。