---
title: Scala 笔记 - 函数式编程风格初探
date: 2020-01-07 10:54:08
tags:
  - Scala
top: 1
---

## 前言

最近一段时间写 Scala 比较多，虽然用传统的风格写代码也没问题，但 Scala 既然提供了比较方便的函数式编程方式，那么还是入乡随俗，好好利用比较好。

目前写下来几个感受最深的还是：尽量避免使用 `var` 和 `mutable` 数据结构。

上周末抽空看了下 **Java 并发编程实战**，基于锁和原子变量的线程安全实现虽然用起来很方便，但是一旦逻辑复杂了，锁住哪些变量，粒度多大，都会使代码变得比较复杂。而使用不可变（immutable）数据结构则能很大程度简化线程安全的实现（当然，锁和原子变量一定程度上还是需要的），因为不可变数据结构本身是线程安全的。

此外，Scala 不提供原生的 `break` 和 `continue` 来进行流程控制，虽然可以通过导入模块的方式使用，但尽量还是避免。

不过作为 C++er，有些地方从 C++ 转过来还是不太适应，因此记录一些笔记。

## 编程习惯的改变

Scala 使用 `()` 来使用下标访问集合，而其它大多数语言都是使用 `[]`，比如用 C 风格代码遍历数组：

```scala
  val array = Array(1, 2, 3)
  for (i <- array.indices)
    println(array(i)) // array(i) 而非 array[i]
```

原因很简单，因为 Scala 使用 `[]` 来指定泛型，而其它语言比如 C++ 和 Java 都是使用 `<>`，Scala 的 `<` 和 `>` 都有各自用法：

- `<-`：用于 for 循环，左边的引用为右边集合的遍历；
- `->`：二元元组的定义，左边为 key，右边为 value；
- `=>`：定义**函数**（而非方法），左边是参数列表，右边是函数体；
- `<=`：嗯？这是小于或等于的意思。笑话出自[Scala 是一门怎样的语言，具有哪些优缺点？ - 颜巍的回答 - 知乎](https://www.zhihu.com/question/19748408/answer/264461050)

Scala 有比其它大部分语言更为强大的 for 循环，比如：

```scala
  val map = Map(1 -> "Java", 2 -> "C", 3 -> "Python", 4 -> "C++")
  for ((key, value) <- map if value.matches("C.*"))
    println(s"$key => $value")
```

打印出 value 以 C 开头的键值对，当然，Scala 也可以使用传统函数式编程常用的 `map`，`filter` 等方法构成调用链：

```scala
  map.filter(_._2.matches("C.*")).foreach { case (key, value) =>
    println(s"$key => $value")
  }
```

注意这里的 `_2` 是取得元组（前面说 `->` 提到了二元元组）的第 2 个元素。两者是基于 Scala 强大的模式匹配（`match`）的，并且都用语法糖省略了多余代码。

这里由于仅仅是处理每个元素，因此用了 `foreach`，如果需要得到结果，比如将 `println` 打印的字符串构成数组或链表，就可以用 `map`，然后用 `mkString` 将换行符作为分隔符即可实现一样的功能：

```scala
  val list = map.filter(_._2.matches("C.*")).map { case (key, value) => s"$key => $value" }
  println(list.mkString("\n"))
```

因此 `map` 可谓最常用的方法了，前面说了，多线程访问共享的可变数据结构时存在 race condition，而如果使用 `map` 将共享的不可变数据结构映射为线程内部可见的另一个不可变数据结构时，则避免了 race condition，也不用麻烦地去加锁来解决，加锁还要考虑锁的粒度，还要谨慎思考粒度太小会不会导致线程不安全了。

当然，作为 C++er 的一个坏习惯就是过早优化，比如担心拷贝和内存多次分配的开销会不会太大。

然而实践起来，很多时候性能的瓶颈是网络延迟，磁盘 I/O，算法处理速度这些因素，而不是拷贝和内存分配。有些拷贝也是必要的，用 C++ 写，不加锁要实现线程安全，也得拷贝一份，至于内存分配，JVM 的 GC 经过一代代发展已经相当强大了。

这里不是说性能不重要，而是说，性能导致问题之前，编写不易出 BUG 的代码优先级更高。

## 流程控制

比如，从 N 台副本服务器上取数据，只要成功取得一份就返回，因此不需要并行操作，而是循环依次去访问服务器，这里给几个模拟实际场景的变量和方法：

```scala
  val servers = Seq("Server1", "Server2", "Server3")

  type DataType = String

  def getDataFromServer(server: String): DataType = {
    if (server.contains("2")) "Ok" else ""
  }
```

### return vs break

使用 `return` 单独写个方法：

```scala
  def getData: DataType = {
    for (server <- servers) {
      val data = getDataFromServer(server)
      if (data.nonEmpty)
        return data
    }
    ""
  }
```

这里的 `for (server <- servers) {` 也可以换成 `servers.foreach { server =>`，看个人喜好。

当然也可以用 `break`，首先 `import scala.util.control.Breaks`，然后代码改成：

```scala
  var data: DataType = ""
  breakable {
    servers.foreach { server =>
      data = getDataFromServer(server)
      if (data.nonEmpty)
        break
    }
  }
```

**这种方式仅仅是举例，实际写代码应该避免**。除了少写一个函数外，没任何好处。

1. 是破坏了不用`var` 的原则，虽然在这里并没有什么影响，但很容易让人逐渐依赖于 `var`;

2. Scala 使用 `break` 本来就比其它语言的 `break` 要复杂，多了外层的 `breakable` 块；

3. 底层是用异常来实现的：

   ```scala
   private class BreakControl extends ControlThrowable
   
   class Breaks {
     private val breakException = new BreakControl
   
     def breakable(op: => Unit) {
       try {
         op
       } catch {
         case ex: BreakControl =>
           if (ex ne breakException) throw ex
       }
     }
   
     def break(): Nothing = { throw breakException }
   }
   ```

   也就是说，如果在 `breakable` 块中处理异常时，还得额外捕获 `ControlThrowable`：

   ```scala
   breakable {
       try {
           // 调用了 break 的代码（略）
       } catch {
           case e: ControlThrowable => throw e // 抛给外层来流程控制
           case e: Throwable => // TODO: 处理真正的异常
       }
   }
   ```

总之，`break` 仅仅是让习惯了其它语言的用户方便上手而已。

### 尾递归

之前 `foreach` 的做法还是避免不了 return 跳出循环，实际上 Scala 提供了尾递归优化，这里不详述，简单说就是将特殊条件下函数的递归调用优化替换成循环调用，并且无法优化的场景利用 `tailrec` 注解会抛出异常说明此处无法尾递归优化。

```scala
  @scala.annotation.tailrec
  def getData(servers: Seq[String]): DataType = {
    servers match {
      case Seq() => ""
      case Seq(server, rest@_*) =>
        val data = getDataFromServer(server)
        if (data.nonEmpty)
          data
        else
          getData(rest.toSeq)
    }
  }
```

上述代码利用了 Scala 强大的模式匹配能力。

第一行的 `case Seq()` 匹配空 `Seq`，也就是递归终止条件。

第二行的需要说明的是 `rest@_*`，进行匹配的是 `_*`，`_` 匹配类型，而 `*` 在 Scala 中则是匹配可变参数列表。前面在通过 `@` 运算符将匹配到的可变参数列表绑定到变量 `reset` 上。

这里的主要好处还是略去了 `return`，在其它语言中 `return` 的一个好处是提前返回来避免 N 层缩进的难以阅读的代码。不过另一方面，`return` 的滥用会导致程序流程不是那么清晰，因为代码太长的话不知道前面哪里会直接 `return`了。不过其实说起来，这里的 `return` 是用在单独的方法中，相当于被隐藏了，也不会导致跳出方法外层的循环，可读性并不受影响。

一开始我是认为尾递归的方式比 `return` 更好，当时最大原因错以为 `return` 会抛出 `ControlThrowable` 来进行流程控制导致外层的 `try-catch` 需要额外处理这个异常，后来发现并不会抛出。因此在这里使用尾递归+模式匹配某种程度上有点炫技的意味。

另外，如果 `servers` 类型是 `List` 则可以用这种尾递归方式：

```scala
  @scala.annotation.tailrec
  def getData(servers: List[String]): DataType = {
    servers match {
      case Nil => ""
      case server :: rest =>
        val data = getDataFromServer(server)
        if (data.nonEmpty)
          data
        else
          getData(rest) // tail recursion
    }
  }
```

## Future 的处理

Scala 可以用 Java 的线程设施来编写多线程程序，但是内置的 `Future` 一般情况下够用和好用了，最近用到的 [play framework WS 模块](https://www.playframework.com/documentation/2.8.x/ScalaWS)，`Post` 和 `Get` 方法返回的都是 `Future`。

一般情况下 `Future` 用 `onComplete` 方法，利用回调函数来处理正常返回或者异常发送的场景：

```scala
  val s = "hello"
  val future = Future {
    if (s.length % 2 == 0)
      s.length
    else
      throw new Exception(s""""$s"'s length is not even""")
  }
  future.onComplete {
    case Failure(e) => // TODO: 处理 future 抛出的异常
    case Success(result) => // TODO: 处理 future 的结果，这里即字符串长度
  }
```

注意 `Future` 底层还是使用线程池的，因此需要导入 `ExecutionContext`，一般用默认的就行：

```scala
import scala.concurrent.ExecutionContext.Implicits.global
```

但是有些时候还是需要等待 `Future` 完成的，这里不回顾 `Future` 的语法，而是谈一些同步方面的处理。

### 等待多个 Future 结束

比如最常见的分块计算，然后将结果汇总：

```scala
  val array = (1 to 10).toArray
  val blockSize = 3

  val blocks = (array.indices by blockSize).map { i => (i, i + blockSize) }.map { pair =>
    if (pair._2 <= array.length) pair else pair._1 -> array.length
  }
  val futures = blocks.map { case (from, to) =>
    Future(array.slice(from, to).sum)
  } // Seq[Future[Int]]
```

现在得到了 `Future[Int]` 的序列，可以用回调函数的方法将结果存入一个 `ConcurrentHashMap`

````scala
  // 修改了 Future 的定义，加入了 from 作为 key
  val futures = blocks.map { case (from, to) =>
    Future(from -> array.slice(from, to).sum)
  }
  val results = new ConcurrentHashMap[Int, Int]()
  futures.foreach { future =>
    future.onComplete {
      case Failure(e) => // TODO: 处理单个计算错误的异常
      case Success((from, sum)) => results.put(from, sum)
    }
  }
  Thread.sleep(100) // 模拟主线程做其它操作
  while (futures.size != blocks.size) { // 无事可做了，轮询等待
    Thread.sleep(100)
  }
  val sum = results.entrySet().asScala.map(_.getValue).sum
````

注意 `asScala` 将 Java 的集合类型转换成 Scala 对应的集合类型，在 Scala 2.13 之前需要：

```scala
import scala.collection.JavaConverters._
```

从 Scala 2.13 开始则是导入另一个包：

```scala
import scala.jdk.CollectionConverters._
```

PS：本文默认都是 Scala 2.8 以上。

回正题，如果主线程不需要做其他操作，就只想等阿迪，那么这种基于回调的方式就未免过于麻烦，不如直接等待。但是 `Await.result` 或者 `Await.ready` 只能等待单个 `Future`，于是得 for 循环等待，然后还是得将结果一个个存入 `HashMap` 或者其它容器中。

`Future` 类提供了 `sequence()` 方法来简化这个操作，直接一行搞定：

```scala
val sum = Await.result(Future.sequence(futures), 2.seconds).sum
```

注意 `seconds` 需要导入相关包：

```scala
import scala.concurrent.duration._
```

`sequence` 方法将 `Future[A]` 的集合转换成 `A` 的集合的 `Future`，在这里即将`Seq[Future[Int]]` 转换成了 `Future[Seq[Int]]`，这样直接等待就行了。

### 任务的顺序执行

有时候一个任务需要等待另一个任务完成后才能执行，此时可以用 `Future` 的 `map` 或 `flatMap` 方法：

```scala
  val f1 = Future { 1 }
  val f2 = f1.map { _ * 2}
  val f3 = f2.map { _ * 3}
  val result = Await.result(f3, 10.milliseconds)
```

`map` 方法接收的 block 是将结果类型映射到结果类型，但是 `flatMap` 方法是将结果类型映射到结果类型的 `Future`，有时候外部方法返回的是 `Future` 类型，此时就得用 `flatMap`：

```scala
  // 模拟外部的接口，比如 PlayWS
  def getResponse(x: Int) = Future {
    x * 2
  }

  val f1 = Future { 1 }
  val f2 = f1.flatMap(getResponse)
  val result = Await.result(f2, 10.milliseconds)
  println(result)
```

假如任务数量不确定，也就回到前文提到的类型，多个服务器，取到一个就退出：

```scala
  val servers = List("Server1", "Server2", "Server3")

  def getResponseFromServer(server: String) = Future {
    if (server.contains("2")) "Ok" else ""
  }
```

这里可以用类似前文尾递归的方法得到新 `Future`：

```scala
  def getResponse(servers: List[String]): Future[String] = servers match {
    case Nil => Future { "" }
    case server :: rest =>
      getResponseFromServer(server).flatMap { result =>
        if (result.nonEmpty)
          Future { result }
        else
          getResponse(rest)
      }
  }

  val result = Await.result(getResponse(servers), 10.milliseconds)
```

这里无法用尾递归，因为递归调用发生在 `flatMap` 接收的 block 中，而非当前方法末尾。

如果把该方法做成同步调用就行了，因为反正要等待，不如直接每次 `Future` 都等待：

```scala
  @scala.annotation.tailrec
  def getResponse(servers: List[String]): String = servers match {
    case Nil => ""
    case server :: rest =>
      val result = Await.result(getResponseFromServer(server), 10.milliseconds)
      if (result.nonEmpty)
        result
      else
        getResponse(rest)
  }

  val result = getResponse(servers)
```

这也是我在项目里实际采用的做法，这种做法有个缺点就是不便于扩展，如果返回 `Future`，那么如果以后要用到 `getResponse` 的结果，直接 `map` 或 `flatMap` 即可，但是现在的话，就必须同步等待 `getResponse` 完成了。

但是考虑到基本上没有进一步扩展的需求，目前就保持这样了。

## 总结

本文算是最近写 Scala 时的一些笔记，其实学 Scala 主要是为了看 Kafka 源码，Kafka 的 Scala 代码其实很多还是并不那么函数式的，毕竟很大一块基础设施还是 Java 写的然后 Scala 来调用，当然，不否认不少地方也用了函数式编程来节省代码和增加可读性。

比如在 1.1.0 版本的 `ReplicaManager.scala` 中，有一段代码：

```scala
    val errorReadingData = logReadResultValues.foldLeft(false) ((errorIncurred, readResult) =>
      errorIncurred || (readResult.error != Errors.NONE))
```

`foldLeft` 之前只用过一遍，所以看到这里根本不知道什么意思，实际上等价于

```scala
    var errorReadingData = false
    logReadResultValues.foreach { readResult =>
      errorReadingData = (errorReadingData || (readResult.error != Errors.NONE))
    }
```

也就是 `logReadResultValues` 中存在一个元素的 `error` 字段为 `Errors.NONE` 则为 false。当然，其实存在一个就可以 break 或 return 了，但后面继续循环也不会有什么显著性开销，所以用 `foldLeft` 非常简洁有效地实现了功能。重要的还是没有使用 `var`。



