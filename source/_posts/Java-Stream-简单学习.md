---
title: Java Stream 简单学习
date: 2022-02-27 01:23:57
tags:
  - Java
top: 1
---

[toc]

## 背景

最近对项目做了一个重构，其中有一部分是把代码风格变得很函数式了。Java 8 引入了 lambda 表达式和 `Stream` 后使得函数式编程就更容易了。这里不谈及比较理论的东西，只谈下我认为的优缺点。

优点：

- 表达简单直观（虽然使用 Java 还是比较啰嗦）
- 提倡使用 immutable 对象，实现线程安全更容易
- 能够通过将 stream 改成 parallel stream 将现有代码改成并行处理

缺点：

- 性能有一定损失。可以用 [JMH](https://github.com/openjdk/jmh) 进行性能测试，大多数情况这点性能损失不会有明显影响，除非代码处在关键路径上。
- 调试相对比较麻烦。其实很多时候可以通过测试驱动的编码方式避免写了一大串代码后再去排查是不是链式调用的某一块出问题了。

## 了解 Stream

举个例子，给出下面这段处理代码：

```java
        final List<Integer> list1 = new ArrayList<>(Arrays.asList(1, 2, 3));
        final List<Integer> list2 = new ArrayList<>(list1.size());
        list1.forEach(x -> list2.add(x + 100));
        final List<String> list3 = new ArrayList<>(list2.size());
        list2.forEach(x -> list3.add("i-" + x));
```

从函数式思维的角度，我们实际上是依次对 `list1` 的每个元素执行以下运算：

1. 加上 100
2. 转换成字符串，并加上 `i-` 前缀。

理想的函数映射表达是：

```java
final List<String> list3 = list1.map(x -> x + 100).map(x -> "i-" + x);
```

但是在实际的 Java 代码中，`map` 这样的操作不能直接作用于容器，而必须作用于容器对应的 stream。

```java
final List<String> list3 = list1.stream().map(x -> x + 100).map(x -> "i-" + x).collect(Collectors.toList());
```

容器必须先调用 `stream()` 方法转换成流（`Stream` 接口）。其实从设计的角度也可以理解，假如对 `List` 扩充 `map` 接口：

```java
<R> List<R> map(Function<? super T, ? extends R> mapper)
```

那么每次 `map` 都会创建新的 `List` 对象。而 `Stream` 是可以理解为它只记录了对流中每个元素的**函数**，每次 `map` 调用返回的只有记录了新的函数（`Function`）的 `Stream`，不会将容器中地元素进行拷贝。只有在 `collect` 调用时才会将元素从流中取出，放入新的容器中。

流操作会被复合成一个**流管道（stream pipeline）**，它包含：

- 源（source）：可以是集合，数组或者函数生成器，或者 I/O 管道。
- 中间操作（intermediate operation）：将流转换成另一个流，比如前文的 `map`。
- 终端操作（terminal operation）：产生结果或副作用，比如前文的 `collect`。

流是惰性的（lazy），只有在终端操作才会对源数据进行计算，并且仅在需要时消耗源元素。

Java 容器都提供了 `stream()` 方法取得对应的流。类似的，用 `parallelStream()` 方法得到并行流，并行流会使用线程池来计算，本文只讨论顺序流。对于数组 `T[]` 可以用 `Arrays#stream` 静态方法将其转换成流，对于可变参数列表 `T...` 可以用 `Streams#of` 静态方法将其转换成流，对于两个整型表示的左闭右开区间，可以用 `IntStream#range` 或者 `LongStream#range` 将其转换成流。比如：

```java
        Arrays.stream(new int[]{0, 1, 2});
        Stream.of(1, 2, 3);
        IntStream.range(0, 3);
        LongStream.range(0L, 3L);
```

上述四行代码都是将序列 0，1，2 转换成流。

流也是一次性的，在遍历一遍后，流就会处于关闭状态。比如：

```java
        Stream<Integer> stream = Stream.of(1, 2, 3);
        stream.map(x -> x + 10);
        stream.map(x -> x + 10);
```

在对同一个流第二次调用 `map` 时会报出异常：

> java.lang.IllegalStateException: stream has already been operated upon or closed

因为无法对同一个流进行两次中间操作（如果允许的话，就会流就会出现两个分支），因此在进行一个中间操作后，这个流就无法再使用，必须对返回的流添加新的中间操作。注意到错误提示里还有个 or closed 描述，因为流也可以调用 `close()` 主动关闭，一般是对于 I/O 管道的流才需要这些操作。

## 常用的流操作示例

在对流有了一个基础认识后，这一节偏实用性，以 `List` 容器（列表）作为源，介绍一些常见的操作。

### reduce

比如最简单的求和：

```java
        final List<Integer> list = Arrays.asList(1, 2, 3);
        int sum = 0;
        for (Integer x : list) {
            sum += x;
        }
```

实际上可以看作有一个初始值 0（`identity`），然后依次对流的每个元素 `value` 进行求和运算（`identity + value`）。此时可以用 `reduce` 中间操作：

```java
        final int sum = Stream.of(1, 2, 3).reduce(0, Integer::sum);
```

如果初始值的类型和流的元素类型不一致，那么需要第三个参数 `combiner`，它是函数 `(T, T) -> T`，其中 `T` 为初始值的类型，该函数必须满足：

```java
combiner.apply(u, accumulator.apply(identity, t)) == accumulator.apply(u, t)
```

其中 `accumulator` 为 `reduce` 的第二个参数。如果流不是并行的，这个参数不起作用，该函数不会被调用，这个时候传入一个简单的 `(x, y) -> x` 函数即可。对 combiner 感兴趣的可以阅读 [这个讨论](https://stackoverflow.com/questions/24308146/why-is-a-combiner-needed-for-reduce-method-that-converts-type-in-java-8)。

```java
        final List<Integer> list = Arrays.asList(1, 2, 3);
        final String result = list.stream().reduce("prefix" , (s, i) -> s + "-" + i, (x, y) -> x);
        System.out.println(result)
```

上述代码得到的 `result` 是字符串 `prefix-1-2-3`。如果要在并行场景下也能工作，那么需要特别设计 combiner，比如：

```java
        final String result = list.parallelStream().reduce("prefix" , (s, i) -> s + "-" + i,
                (s1, s2) -> s1 + s2.substring("prefix".length()));
```

类似地，假如要合并多个列表，比如：

```java
        final List<Integer> list1 = Arrays.asList(1, 2, 3);
        final List<Integer> list2 = Arrays.asList(4, 5);
        final List<Integer> list3 = Arrays.asList(6, 7, 8, 9);
```

直接的方式：

```java
        final List<Integer> list = new ArrayList<>(list1);
        list.addAll(list2);
        list.addAll(list3);
```

基于 `reduce` 的方式：

```java
        final List<Integer> list = Stream.of(list1, list2, list3).reduce(new ArrayList<>(), (lhs, rhs) -> {
            lhs.addAll(rhs);
            return lhs;
        });
```

### flatMap

接着前一节的示例，合并多个列表，虽然用 `reduce` 是自然的，但并不是最适合的。这种场景实际上应该用 `flatMap`。

`map` 是将 `Stream<T>` 映射到 `Stream<R>`。但列表合并的需求，实际上是将 `Stream<List<T>>` 转换成 `Stream<T>`，无法直接将 `List<T>` 通过函数得到 `T`，而且两者关系也不是一对一，比如第 1 个 `List<T>` 包含 3 个元素，那么我们想要在返回的流中添加 3 个 `T` 。在这种数量发生斌华的场景，则是 `flatMap` 使用的时机。

```java
    <R> Stream<R> flatMap(Function<? super T, ? extends Stream<? extends R>> mapper);
```

它接收的函数是将 `T` 映射到 `Stream<R>`。如果 `T` 是 `List<E>`，那么很自然地就能得到 `Stream<E>`。因此基于 `flatMap` 可以更自然地合并多个列表：

```java
        final List<Integer> list = Stream.of(list1, list2, list3).flatMap(List::stream).collect(Collectors.toList());
```

可见，`flatMap` 特别适合于流的元素（比如 `List<T>`）本身就可以转换成流（`Stream<T>`）的场景。对于列表嵌套，只需要多调用几次 `flatMap` 即可，比如：

```java
        final List<List<Integer>> list1 = Arrays.asList(Arrays.asList(1, 2, 3), Arrays.asList(4, 5));
        final List<List<Integer>> list2 = Collections.singletonList(Arrays.asList(6, 7, 8, 9));
        final List<Integer> list = Stream.of(list1, list2)
                .flatMap(Collection::stream)
                .flatMap(Collection::stream)
                .collect(Collectors.toList());
```

### groupingBy

这个需求常见于将流进行分类。比如将整数流按照奇数和偶数分成两部分。这个操作本身是将列表分成多个列表，但是并没有对应的中间操作，原因在于，流不支持分叉成多个流。此时要使用终端操作 `Collectors.groupingBy`。

```java
    public static <T, K> Collector<T, ?, Map<K, List<T>>>
    groupingBy(Function<? super T, ? extends K> classifier) {
        return groupingBy(classifier, toList());
    }
```

它将流的元素 `T` 映射到结果 `K`，最终得到的结果是 `Map<K, List<T>>`。假如要将整数流分成奇数和偶数，那么我们可以用 `Boolean` 表示元素 `Integer` 是否为奇数。

```java
        final List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);
        final Map<Boolean, List<Integer>> map = list.stream().collect(Collectors.groupingBy(i -> i % 2 != 0));
        System.out.println(map.get(true)); // 奇数
        System.out.println(map.get(false)); // 偶数
```

打印结果：

```
[1, 3, 5]
[2, 4]
```

这里需要重点注意的是，`get(true)` 和 `get(false)` 可能得到的是 `null`，对应的分别是流中没有偶数和没有奇数。因此需要谨慎地进行  null check。

### filter

将整数流分成奇数和偶数有一个更慢的方法，那就是实用 `filter` 遍历两次，第一次过滤出奇数，第二次过滤出偶数：

```java
        final List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);
        final List<Integer> odds = list.stream().filter(x -> x % 2 != 0).collect(Collectors.toList());
        final List<Integer> evens = list.stream().filter(x -> x % 2 == 0).collect(Collectors.toList());
        System.out.println(odds);
        System.out.println(evens);
```

如果我们只想对奇数或者偶数处理，那么 `filter` 则是最适合的。

## Map 容器的流处理

比如对 `Map` 容器本身无法得到流，因为 `Map` 并不是单个元素的流，只能对 `entrySet()` 方法返回的 `Set` 得到流 `Stream<Map.Entry<K, V>>`，然后对每个元素 `Map.Entry<K, V>` 进行处理。如果要在终端操作中重新得到新的 map，比如 `Map<K, V2>` 或者 `Map<K2, V2>`，那么则需要用 `Collector.toMap` 方法。

举个例子，下列 Java 代码将 `Map<Integer, Integer>` 分别对 value 和 entry 做映射得到 `Map<Integer, String>` 和 `Map<String, String>`。

```java
        final Map<Integer, Integer> map = new HashMap<>();
        map.put(1, 100);
        map.put(2, 200);
        System.out.println(map.entrySet().stream().collect(Collectors.toMap(
                Map.Entry::getKey,
                e -> "value-" + e.getValue())
        ));
        System.out.println(map.entrySet().stream().collect(Collectors.toMap(
                e -> "key-" + e.getKey(),
                e -> "value-" + e.getValue())
        ));
```

打印结果：

```
{1=value-100, 2=value-200}
{key-1=value-100, key-2=value-200}
```

可见 `Collectors.toMap` 中分别接收对 key 和 value 的映射，因此支持 key 和 value 不同的中间操作次数。

相比而言，Scala 在函数式编程上简洁太多，比如实现上述功能，在 Scala 中只需要如下所示：

```scala
  val map = Map(1 -> 100, 2 -> 200)
  println(map.map(e => (e._1, "value-" + e._2)))
  println(map.map(e => ("key-" + e._1, "value-" + e._2)))
```

## Collectors

前面我们使用了三种 `Collectors` 的静态方法，这里我们来看看 `collect` 到底做了什么。

```java
    <R, A> R collect(Collector<? super T, A, R> collector)
```

首先 `collect` 方法接受的是 `Collector<T, A, R>` 类型，其中：

- T 是流的元素类型
- A 是 `Collector` 进行中间运算的类型
- R 是结果类型

以 `Collectors#toList` 方法为例：

```java
    public static <T>
    Collector<T, ?, List<T>> toList() {
        return new CollectorImpl<>((Supplier<List<T>>) ArrayList::new, List::add,
                                   (left, right) -> { left.addAll(right); return left; },
                                   CH_ID);
    }
```

其中 `CollectorImpl` 实现了 `Collector` 接口，接收四个参数，最主要的是：

| 字段名        | `toList` 传入的参数 | 含义                                         |
| ------------- | ------------------- | -------------------------------------------- |
| `supplier`    | `ArrayList::new`    | 创建 `ArrayList` 存放收集的元素              |
| `accumulator` | `List::add`         | 以流的元素作为参数对 `supplier` 调用该方法， |

其余的 `combiner` 和 `characteristics` 比较复杂，这里就不深入研究了。

如果有特殊要求，我们也可以模仿 `CollectorImpl` 类自行实现 `Collector` 接口。

## 总结

本文主要是最近一次重构中对 Java 基于 `Stream` 的函数式编程的一点学习笔记，从了解 `Stream` 开始到常用的几个中间操作，以及针对 `Map` 容器的特殊处理，最后看了下终端操作 `collect`。更多内容可以参考 [Java SE 8 Documents](https://docs.oracle.com/javase/8/docs/api/)。
