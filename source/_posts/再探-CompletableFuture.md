---
title: 再探 CompletableFuture
date: 2021-05-09 00:58:55
tags:
  - Java
top: 1
---

[toc]

## 前言

上次在 Github 上面写博客都是大半年前了，这段时间一直相对较忙，没抽出时间来认真写博客，大多只是一些零散的笔记记录在个人的有道云笔记上，这次姑且尝试继续写博客。

在之前的博客 [Java CompletableFuture 学习](https://bewaremypower.github.io/2020/10/07/Java-CompletableFuture-%E5%AD%A6%E4%B9%A0/) 中，我们了解了 `CompletableFuture` 用于异步编程的基本套路：
- 使用 `supplyAsync` 在 `ExecutorService` 中启动异步任务并返回 future
- 使用 `thenApply` / `thenCompose` / `exceptionally` 方法进行 future 之间的链式映射
- 使用 `whenComplete` 方法来指定异步任务完成时的回调

然而在这段时间阅读 [Pulsar](https://github.com/apache/pulsar) 和 [KoP](https://github.com/streamnative/kop) 的源码过程中，发现其实对 `CompletableFuture` 的使用还是存在一些误区的，本文将个人的一些经验整合一下。

本文的示例从一个简单的例子开始（其中 `print` 方法会在后文中复用）：

```java
    private static void print(final String msg) {
        System.out.println(Thread.currentThread().getName() + " " + msg);
    }

    public static void main(String[] args) throws ExecutionException, InterruptedException {
        final CompletableFuture<Integer> intFuture = CompletableFuture.supplyAsync(() -> {
            print("in supplyAsync()");
            return 0;
        });
        print("intFuture returns: " + intFuture.get());
    }
```

输出结果：

```
ForkJoinPool.commonPool-worker-9 in supplyAsync()
main intFuture returns: 0
```

可见 `supplyAsync` 默认用的是 [`ForkJoinPool`](https://docs.oracle.com/javase/8/docs/api/java/util/concurrent/ForkJoinPool.html) 这个 Java 线程池。由于本人是半路出家的 Javaer，这里先记下这个知识点，之后再系统学习下。

## thenApply/exceptionally vs. whenComplete

### 针对不会异常完成的 future

这个其实很多人都用错了使用场景，因为大多数时候，其实我们只是想给 future 加一个回调，并不想继续链式调用。其实这种简单场景下，`whenComplete` 才是应该用的。比如，我们想对奇数进行操作，偶数则抛出异常，可以是：

```java
        final CompletableFuture<Integer> firstFuture = CompletableFuture.supplyAsync(() -> {
            final int x = new Random().nextInt();
            print("in firstFuture, x = " + x);
            return x;
        });
        final CompletableFuture<Integer> secondFuture = new CompletableFuture<>();
        firstFuture.whenComplete((integer, throwable) -> {
            assert throwable != null;
            print("in secondFuture, firstFuture returns " + integer);
            if (integer % 2 != 0) {
                secondFuture.complete(integer / 2);
            } else {
                secondFuture.completeExceptionally(new Exception("firstFuture returns an even number: " + integer));
            }
        });
        try {
            print("secondFuture returns: " + secondFuture.get());
        } catch (ExecutionException e) {
            print("secondFuture failed: " + e.getCause());
        }
```

以下是两种典型输出：

```
ForkJoinPool.commonPool-worker-9 in firstFuture, x = -1634409183
main in secondFuture, firstFuture returns -1634409183
main secondFuture returns: -817204591
```

```
ForkJoinPool.commonPool-worker-9 in firstFuture, x = 1243845674
main in secondFuture, firstFuture returns 1243845674
main secondFuture failed: java.lang.Exception: firstFuture returns an even number: 1243845674
```

这里我们可以先留意一下，`firstFuture` 的 `whenComplete` 方法是在主线程内执行的。

但实际上由于 `firstFuture` 永远不会异常结束，因此这个时候其实可以简化 `secondFuture` 的构造为：

```java
        final CompletableFuture<Integer> secondFuture = firstFuture.thenApply(integer -> {
            print("in secondFuture, firstFuture returns " + integer);
            if (integer % 2 != 0) {
                return integer / 2;
            } else {
                // 注意 thenApply 内部只能抛出 unchecked exception
                throw new RuntimeException("firstFuture returns an even number: " + integer);
            }
        });
```

此时两者本质上是一样的，但是 `thenApply` 更简洁。而且其实可以将上述代码简化为：

```java
final CompletableFuture<Integer> future = CompletableFuture.supplyAsync(/* ... */).thenApply(/* ... */);
```

### 针对可能异常完成的 future

TL; DR 使用 `whenComplete` 就是最好的，不要链式使用 `thenApply` 和 `exceptionally`。 

假如 `firstFuture` 就在偶数的时候抛出异常：

```java
        final CompletableFuture<Integer> firstFuture = CompletableFuture.supplyAsync(() -> {
            final int x = new Random().nextInt();
            print("in firstFuture, x = " + x);
            if (x % 2 != 0) {
                return x;
            } else {
                throw new RuntimeException("firstFuture returns an even number: " + x);
            }
        });
```

那么基于 `whenComplete` 实现的 `secondFuture` 变成了：

```java
        final CompletableFuture<Integer> secondFuture = new CompletableFuture<>();
        print("in secondFuture, firstFuture returns " + integer);
        firstFuture.whenComplete((integer, throwable) -> {
            if (throwable != null) {
                secondFuture.completeExceptionally(throwable);
            } else {
                secondFuture.complete(integer / 2);
            }
        });
```

这个实现和上一节的是等价的。但此时，如果基于 `thenApply`，很多人会进一步用 `exceptionally` 进行两层链式调用：

```java
        final CompletableFuture<Integer> secondFuture = firstFuture.thenApply(integer -> {
            print("in secondFuture thenApply, x = " + integer);
            return integer;
        }).exceptionally(e -> {
            print("in secondFuture exception: " + e);
            return null;
        });
```

注意，这其实是错误的实现，并且和基于 `whenComplete` 的实现不是等价的。原因在于 `exceptionally`，注意我们返回的是 `null`，因此 `secondFuture` 在 `firstFuture` 异常完成时，是返回 null 而非异常完成。

比如下面这组 `firstFuture` 异常完成时的输出：

```
ForkJoinPool.commonPool-worker-9 in firstFuture, x = 784152994
main in secondFuture exception: java.util.concurrent.CompletionException: java.lang.RuntimeException: firstFuture returns an even number: 784152994
main secondFuture returns: null
```

这里有两个重点：
1. `secondFuture` 是以 null 正常完成（而不是继续以 `firstFuture` 的异常正常完成）；
2. `exceptionally` 的 `e` 的类型不是 `RuntimeException`，而是 `CompletionException`，它的 cause 才是 `RuntimeException`。

第一点我们刚才提到了，是 `return null` 导致的，第二点则尤为致命。之前 Pulsar 这边有个问题是，大量的 HTTP 处理的错误码都是 500 internal error，而不是正确的错误码，原因就在于大量处理都是这样的：

```java
final CompletableFuture<Object> future = new CompletableFuture<>();
asyncDoSomething()
    .thenApply(result -> future.complete(result))
    .exceptionally(e -> {
        future.completeExceptionally(e);
        return null;
    };
```

而在之后处理 `future` 的异常时，直接把 `e` 当成 `e.getCause()` 的类型来使用了。

## thenApply 的实现

造成上一节里 `exceptionally` 的 `e` 不再是最初 future 的异常，而是包裹后的 `CompletionException` 的原因其实很简单，那就是某个 `CompletableFuture` 调用 `thenApply` 返回后的 `CompletableFuture`，已经不是它自己了。这里我们简单看下源码。

```java
public class CompletableFuture<T> implements Future<T>, CompletionStage<T> {

    public <U> CompletableFuture<U> thenApply(
        Function<? super T,? extends U> fn) {
        // 这里 fn 即我们传入的 lambda 表达式，接收 T 类型的参数，返回 U 类型
        return uniApplyStage(null, fn);
    }
```

注意 `thenApply` 其实是将 `CompletableFuture<T>` 映射到 `CompletableFuture<U>` 的，因此很显然前后两个 future 类型不一样。那为什么不继续传播前一个 future 的异常呢？继续看下去。

```java
    private <V> CompletableFuture<V> uniApplyStage(
        Executor e, Function<? super T,? extends V> f) {
        // f 即用户传入的 T -> U 的 lambda 表达式，这里泛型参数是 V 而不是 U
        if (f == null) throw new NullPointerException();
        CompletableFuture<V> d =  new CompletableFuture<V>();
        // 注意这里的 e 是 Executor 对象，而不是异常，因为 f 并没有执行
        if (e != null || !d.uniApply(this, f, null)) {
            UniApply<T,V> c = new UniApply<T,V>(e, d, this, f);
            push(c);
            c.tryFire(SYNC); // SYNC 是常量 0
        }
        return d;
    }
```

构造了 `UniApply` 对象，然后调用了 `tryFire` 方法：

```java
        final CompletableFuture<V> tryFire(int mode) {
            CompletableFuture<V> d; CompletableFuture<T> a;
            if ((d = dep) == null ||
                !d.uniApply(a = src, fn, mode > 0 ? null : this))
                return null;
            dep = null; src = null; fn = null;
            return d.postFire(a, mode);
        }
```

对返回的 future `d` 调用了 `uniApply` 方法：

```java
    final <S> boolean uniApply(CompletableFuture<S> a,
                               Function<? super S,? extends T> f,
                               UniApply<S,T> c) {
        Object r; Throwable x;
        if (a == null || (r = a.result) == null || f == null)
            return false;
        tryComplete: if (result == null) {
            if (r instanceof AltResult) {
                /* ... */
            }
            try {
                if (c != null && !c.claim())
                    return false;
                @SuppressWarnings("unchecked") S s = (S) r;
                // 注意，在这里执行我们传入的 lambda 表达式 f
                completeValue(f.apply(s));
            } catch (Throwable ex) {
                // 如果抛出异常，则调用 `completeThrowable` 方法
                completeThrowable(ex);
            }
        }
        return true;
    }
```

终于看到捕获异常的位置了，最后我们只需要看看 `completeThrowable` 做了什么。

```java
    static AltResult encodeThrowable(Throwable x) {
        // 用 x 作为 cause 构造 CompletionException，这里还进行了类型判断避免 CompletionException 嵌套
        return new AltResult((x instanceof CompletionException) ? x :
                             new CompletionException(x));
    }

    final boolean completeThrowable(Throwable x) {
        // 使用 native CAS 方法将偏移量 RESULT 对应的字段（其实就是 result）设置为 encodeThrowable(x)，如果 result 之前是 null
        return UNSAFE.compareAndSwapObject(this, RESULT, null,
                                           encodeThrowable(x));
    }
```

至此，我们知道为什么 `thenApply` 返回的 future 若异常完成，其异常是 `CompletionException` 而非原来的 future 的异常了。这里多说一句，设置的 `result` 字段为：

```java
    volatile Object result;       // Either the result or boxed AltResult
```

也就是 `get()` / `getNow()` 等方法会尝试返回的字段。进一步的代码阅读就不继续了，总之，我们已经知道了，并且知道为什么，将 `thenApply` 和 `exceptionally` 结合起来并不是一种合适的取代 `whenComplete` 的方法。

虽然我们一般用 `thenApply` 的返回值比较多，但实际上 `exceptionally` 和 `whenComplete` 也有返回值。

```java
    public CompletableFuture<T> exceptionally(
        Function<Throwable, ? extends T> fn) {
        return uniExceptionallyStage(fn);
    }

    public CompletableFuture<T> whenComplete(
        BiConsumer<? super T, ? super Throwable> action) {
        return uniWhenCompleteStage(null, action);
    }
```

初学者一个常见的使用错误或困惑是，在 `exceptionally` 末尾忘记去 `return null`，因为在 `whenComplete` 里不用 `return`。从方法签名就能看出来，原因是 `exceptionally` 接收的是 `(Throwable) -> T` 的函数，而 `whenComplete` 接收的是 `(T, Throwable) -> void` 的函数，没有返回值。而由于 `exceptionally` 接收的参数里没有 `T` 类型，因此只能返回一个常量，绝大多时候就是 null 了。

当然，它们返回的虽然和当前 future 对象是一个类型： `CompletableFuture<T>`，但实际上已经不是一个对象了。依然是举个例子：

```java
        CompletableFuture.supplyAsync(() -> {
            final int x = new Random().nextInt();
            print("supplyAsync, x = " + x);
            if (x % 2 != 0) {
                return x;
            } else {
                throw new RuntimeException("firstFuture returns an even number: " + x);
            }
        }).whenComplete((integer, throwable) -> {
            print("whenComplete integer=" + integer + ", throwable=" + throwable);
        }).get();
```

两个示例输出：

```
ForkJoinPool.commonPool-worker-9 supplyAsync, x = 1489321475
main whenComplete integer=1489321475, throwable=null
```

```
ForkJoinPool.commonPool-worker-9 supplyAsync, x = -1124011744
main whenComplete integer=null, throwable=java.util.concurrent.CompletionException: java.lang.RuntimeException: firstFuture returns an even number: -1124011744
```

我们可以注意到由于 `get()` 方法是 `whenComplete` 返回的 future 而不是原来的 future 调用的，抛出异常时，也是 `CompletionException`，其中的 cause 才是真正的异常。这里就不去看源码了，其实可以猜到，处理和 `thenApply` 是类似的。

## getNow vs. thenApply/whenComplete

有一些场景是，判断当前 future 是否已经完成，若完成则直接处理结果，否则加入队列等待。亦或者，是用 `CompletableFuture.all` 来等待多个 future 结束时再做处理，比如：

```java
    private static void sleep(long millis) {
        try {
            Thread.sleep(millis);
        } catch (InterruptedException ignored) {
        }
    }
```

```java
        final CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
            sleep(1000);
            print("future1 returns");
            return 1;
        });
        final CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> {
            sleep(1200);
            print("future2 returns");
            return 2;
        });
        // allOf 返回的是 CompletableFuture<Void>，而非 Scala 里 Future.sequence 那样返回一个 List 的 future
        CompletableFuture.allOf(future1, future2).get();
        print(future1.getNow(null) + ", " + future2.getNow(null));
```

示例输出：

```
ForkJoinPool.commonPool-worker-9 future1 returns
ForkJoinPool.commonPool-worker-2 future2 returns
main 1, 2
```

这里我们用的 `getNow`，实际上用 `get` 方法也一样，因为此时 future 已经完成了，`get` 会立刻返回。但由于 `get` 的方法签名中包含 checked exception，因此需要你捕获。

实际上 `getNow` 从表达上也不太好，因为 future 已经完成，这个默认值 null 其实是多余的。从语义表达的角度，在这里不对 `getNow(null)` 的返回值进行 null 检查的话，就得在注释中写明此时保证 future 已经完成。但实际上有可能前面 `allOf` 写错了，比如写漏了一个 future，导致其实有的 future 在这里是并没有完成的，那么排查问题就变得麻烦。

有时候在这里也会选择 `join` 方法，它和 `get` 的不同在于，future 异常完成时，它抛出的是 unchecked exception，这样就不需要捕获。

但实际上其实也可以用 `thenApply`：

```java
        future1.thenApply(result1 -> {
            future2.thenApply(result2 -> {
                print(result1 + ", " + result2);
                return null;
            });
            return null;
        });
```

虽然在这里略显麻烦，并且出现了 future 嵌套。但其实反而这种嵌套从表达上更好，因为这里我们的处理，result2 是依赖于 result1 的，这种嵌套表达了依赖关系。

另外，对于已经完成的 future，`thenApply` 的处理全部是在主线程进行的。同理，`whenComplete` 也是。因此假如我们在对一个已经完成的 future 进行处理时，可以直接用 `whenComplete`。

现在来看看另一个场景，那就是在 future 完成之前这么做会怎样？

```java
        final CompletableFuture<Void> future = new CompletableFuture<>();
        future1.thenApply(result1 -> {
            print("result1: " + result1);
            future2.thenApply(result2 -> {
                print(result1 + ", " + result2);
                future.complete(null);
                return null;
            });
            return null;
        });
        future.get();
```

示例输出：

```
ForkJoinPool.commonPool-worker-9 future1 returns
ForkJoinPool.commonPool-worker-9 result1: 1
ForkJoinPool.commonPool-worker-2 future2 returns
ForkJoinPool.commonPool-worker-2 1, 2
```

可见每个 future 的回调都是接着当前 future 所在线程的。然而，假如我们调整一下，让 future2 只 sleep 200 ms（也就是说 future2 在 future1 之前完成），输出变成了：

```
ForkJoinPool.commonPool-worker-2 future2 returns
ForkJoinPool.commonPool-worker-9 future1 returns
ForkJoinPool.commonPool-worker-9 result1: 1
ForkJoinPool.commonPool-worker-9 1, 2
```

一个显著变化是，future2 的回调不再是在 future2 所在线程执行，而是在 future1 所在线程执行。原因其实和刚才的一样，在 future1 回调中去给 future2 再注册新的回调时，future2 已经完成了，因此 future2 的回调直接在 **当前线程** 执行。其实这个原理在前面贴的代码中 `uniApply` 中实现的：

```java
    final <S> boolean uniApply(CompletableFuture<S> a,
                               Function<? super S,? extends T> f,
                               UniApply<S,T> c) {
        Object r; Throwable x;
        // 如果当前 future 未完成，a.result 则为 null，此时 uniApply 会返回 false，f 被加入队列
        if (a == null || (r = a.result) == null || f == null)
            return false;
        // 如果 a.result != null，则后面会调用 completeValue 执行 f.apply，这里就不重复贴代码了
```

至此可见使用 `thenApply` 注册回调，比起 `getNow` 等方法更灵活，而且在 future 已经完成的场景下，两者其实是等价的，前者并不会带来更多的开销。当然，从代码简洁性的角度，有时候用 `getNow(null)` 其实还是可以的，取决于个人。

## 总结

本文主要探讨了两个方面，一个是设置回调时结合 `thenApply` 和 `exceptionally` 和直接使用 `whenComplete` 的对比，另一个是对于 future 已经完成的场合，`thenApply` 的使用。此外，还附带着看了下 `thenApply` 的实现。总的来说，`CompletableFuture` 这个基础设施还是简单易用，但还是有一些细节需要注意才能写出更好的代码。
