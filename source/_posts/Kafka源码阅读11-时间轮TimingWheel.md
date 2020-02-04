---
title: 'Kafka源码阅读11: 时间轮TimingWheel'
date: 2020-02-04 21:40:43
tags:
  - Kafka
top: 1
---
## 前言

前一章阅读了各种延迟操作类的基类 `DelayedOperation`，而延迟操作对象会传入 `DelayedOperationPurgatory`，查看其构造参数：

```scala
purgatoryName: String,
timeoutTimer: Timer,
brokerId: Int = 0,
purgeInterval: Int = 1000,
reaperEnabled: Boolean = true,
timerEnabled: Boolean = true
```

在 `ReplicaManager` 中是调用 `apply` 方法构造的，这里的 `timer` 使用 `util.timer.SystemTimer`。

```scala
val timer = new SystemTimer(purgatoryName)
```

`SystemTimer`内部一个重要字段就是时间轮 `TimingWheel` 对象：

```scala
  private[this] val timingWheel = new TimingWheel(
    tickMs = tickMs,
    wheelSize = wheelSize,
    startMs = startMs,
    taskCounter = taskCounter,
    delayQueue
  )
```

## 设计思路

实现在 `utils/timer/TimingWheel.scala` 中，这是 Kafka 精心设计的时间轮，因此关于该类的说明有长达 70 多行，这里首先阅读其设计思路。

### 简单时间轮

简单时间轮通常是时间任务桶的循环链表。令 u 为时间单元，一个大小为 n 的时间轮有 n 个桶，能够持有 `n * u` 个时间间隔的定时任务。

每个桶持有进入相应时间范围的定时任务。最开始，第一个桶持有 `[0, u)` 范围的任务，第二个桶持有 `[u, 2u)` 范围的任务……第 n 个桶持有 `[u * (n - 1), u * n)` 范围的任务。每过一个时间单元 u，定时器会 tick 并移动到下个桶，然后其中所有的定时任务都会过期。由于任务已经过期，此时定时器不会插入任务到当前桶中。定时器会立刻运行过期的任务。因为空桶在下一轮是可用的，所以如果当前的桶对应时间 t，那么它会在 tick 后变成 `[t + u * n, t + (n + 1) * u)` 的桶。

时间轮的插入/删除（即启动/停止定时器）的时间复杂度是 `O(1)`，而基于优先队列的定时器，比如 `java.util.concurrent.DelayQueue` 和 `java.util.Timer` 插入/删除的时间复杂度是 `O(log n)`。

----

本质上时间轮就是个哈希表，因此插入/删除的时间复杂度是 `O(1)`，而哈希表的 value 类型是链表，插入/删除的时间复杂度也是 `O(1)`，因此将定时任务 `TimerTaskEntry` 插入到时间轮/从时间轮中删除的时间复杂度也是 `O(1)`。

### 分层时间轮

简单时间轮的主要缺点是它假设定时器请求是在从当前时刻开始的 `n * u` 时间间隔内，如果定时器请求超出了这个间隔就会产生溢出。分层时间轮会处理这种溢出，它以层次来组织时间轮，最底层的精度更高，层数越高，表示的精度更低。如果某一层时间轮的精度是 u，大小是 n，则更高一层的精度是 `n * u`。每一层的溢出会被委托给更高层的时间轮。当更高层的时间轮 tick 时，它会把定时任务插入到更底层。溢出的时间轮会按照需求来创建。当溢出的时间轮的桶过期时，其中所有任务会重新递归地插入到定时器中，之后这些任务会被移动到精度更高的时间轮中或者被执行。设 m 是时间轮的数量，则插入（启动定时器）的时间复杂度是 `O(m)`，相比起系统中请求的数量，通常是小很多的。而删除（停止定时器）的时间复杂度仍然是 `O(1)`。

----

像时钟就是一个典型的三层时间轮，秒针能表示 0 到 59 秒，但是对 60 秒以上则需要分针进一步表示，再进一步即时针，一共能表示的时间范围为 0 到 43199 秒，精度为 1 秒。从秒针到分针到时针，表示精度是依次降低的，秒针精度为 1 秒，有 60 格，因此分针精度是 `1 * 60 = 60` 秒，类似地，时钟精度是 3600 秒。而上文用到的 tick 一词，则对应秒针/分针/时针的走动。

时间轮的每个时间间隔都对应了一个桶（bucket），即定时任务链表 `TimerTaskList`。根据每个定时任务的 timeout（过期时间），决定将任务分配给那个桶。

### 示例

令 `u = 1, n = 3`，设起始时刻是 c，则各层次的桶为

| 层次 | 桶                                   | 精度 |
| ---- | ------------------------------------ | ---- |
| 1    | `[c,c]` `[c+1,c+1]` `[c+2,c+2]`      | 1    |
| 2    | `[c,c+2]` `[c+3,c+5]` `[c+6,c+8]`    | 3    |
| 3    | `[c,c+8]` `[c+9,c+17]` `[c+18,c+26]` | 9    |

PS：这里沿用了代码注释里的表示，即闭区间，而前面讲述原理时都是左闭右开区间，两者是等价的，只是表示不一致。

在 `c+1` 时刻，桶 `[c,c]`、`[c,c+2]`、`[c,c+8]`过期了，之后：

- 1 层的时钟移动到 `c+1`，并且创建新的桶 `[c+3,c+3]`；
- 2、3 层的时钟仍然在 c 处，因为他们的精度是 3 和 9。

此时各层次的桶为：

| 层次 | 桶                                   | 精度 |
| ---- | ------------------------------------ | ---- |
| 1    | `[c+1,c+1]` `[c+2,c+2]` `[c+3,c+3]`  | 1    |
| 2    | `[c,c+2]` `[c+3,c+5]` `[c+6,c+8]`    | 3    |
| 3    | `[c,c+8]` `[c+9,c+17]` `[c+18,c+26]` | 9    |

注意，桶 `[c,c+2]` 不会接收任何任务，因为此时时刻是 `c+1`，只有 timeout 为 `c+1` 和 `c+2` 才会被分配到该桶，然而 1 层的两个桶 `[c+1,c+1]` `[c+2,c+2]` 会优先接收任务。类似地，3 层的 `[c+1,c+8]` 也不会接收任何任务，因为这个范围被 2 层的桶覆盖了。

依次类推，在  `c+3` 时刻，2 层也会创建新的桶，各层次的桶为：

| 层次 | 桶                                   | 精度 |
| ---- | ------------------------------------ | ---- |
| 1    | `[c+3,c+3]` `[c+4,c+4]` `[c+5,c+5]`  | 1    |
| 2    | `[c+3,c+5]` `[c+6,c+8]` `[c+9,c+11]` | 3    |
| 3    | `[c,c+8]` `[c+9,c+17]` `[c+18,c+26]` | 9    |

PS：这里源码的注释说 3 层的第 3 个桶是 `[c+8,c+11]`，看了下，大概是注释错误？

## 实现

前一节花了不少篇幅描述的设计思路（主要翻译源码注释，补充自己理解）暂时只说了如何分层，但是对于一个指定了过期时间的任务，如何添加进时间轮，还有前文提到的任务如何从高层时间轮降到底层时间轮：

> 当更高层的时间轮 tick 时，它会把定时任务插入到更底层。

这些都需要通过源码进一步了解实现。

### `TimeWheel` 的字段

主构造器的字段

| 名称          | 类型                        | 说明                                         |
| ------------- | --------------------------- | -------------------------------------------- |
| `tickMs`      | `Long`                      | 每 tick 一次经过的毫秒数，即前文的时间单元 u |
| `wheelSize`   | `Int`                       | 时间轮大小，即前文的桶数 n                   |
| `startMs`     | `Long`                      | 毫秒级时间戳                                 |
| `taskCounter` | `AtomicInteger`             | 任务数量，即所有桶（链表）中的节点数量之和   |
| `queue`       | `DelayQueue[TimerTaskList]` |                                              |

注意到这里还有个 `DelayQueue` 作为辅助，具体作用之后再看。

通过上述主构造参数可以计算出以下私有字段（`private[this]`，可以被包内其他类访问）

```scala
  // 当前时间轮的整个时间跨度，即更高一层时间轮的 tickMs
  private[this] val interval = tickMs * wheelSize
  // 创建 wheelSize 个桶（定时任务链表）
  private[this] val buckets = Array.tabulate[TimerTaskList](wheelSize) { _ => new TimerTaskList(taskCounter) }

  // 向下取整，使起始时间戳能被 tickMs 整除
  private[this] var currentTime = startMs - (startMs % tickMs) // rounding down to multiple of tickMs

  // 高一层时间轮，用来保存超过 interval 的任务
  @volatile private[this] var overflowWheel: TimingWheel = null
```

注意这里做了取整，因此左闭右开区间 `[currentTime, currentTime + tickMs)`  即时间轮第一个桶的范围。

通过 `addOverflowWheel` 创建高一层时间轮：

```scala
  private[this] def addOverflowWheel(): Unit = {
    synchronized {
      if (overflowWheel == null) { // Double-Checked Locking 模式
        overflowWheel = new TimingWheel(
          tickMs = interval, // 仅有此参数和之前不同，见分层时间轮一节的解释
          wheelSize = wheelSize,
          startMs = currentTime,
          taskCounter = taskCounter,
          queue
        )
      }
    }
  }
```

### 添加定时任务

```scala
  def add(timerTaskEntry: TimerTaskEntry): Boolean = {
    // 定时任务的过期时间戳
    val expiration = timerTaskEntry.expirationMs

    if (timerTaskEntry.cancelled) {
      // Entry 绑定的 TimerTask 调用了 cancel() 方法主动将 Entry 从链表中移除
      false
    } else if (expiration < currentTime + tickMs) {
      // 过期时间在第一个桶的范围内，表示已经过期，此时无需加入时间轮
      false
    } else if (expiration < currentTime + interval) {
      // 过期时间在当前时间轮能表示的时间范围内，加入到其中一个桶
      // 注意按照这个算法，第一个桶的时间范围是 [c+u,c+u*2)，因为 [c,c+u) 范围内被视为已过期
      // 而且第一个桶对应 buckets 的下标并不一定是 0，因为数组只是作为循环队列的存储方式，起始下标无所谓
      val virtualId = expiration / tickMs
      val bucket = buckets((virtualId % wheelSize.toLong).toInt)
      bucket.add(timerTaskEntry)

      // 设置过期时间，这里也取整了，即可以被 tickMs 整除
      if (bucket.setExpiration(virtualId * tickMs)) { // 仅在新的过期时间和之前的不同才返回 true
        // 由于进行了取整，同一个 bucket 所有节点的过期时间都相同，因此仅在 bucket 的第一个节点加入时才会进入此 if 块
        // 因此保证了每个桶只会被加入一次到 queue 中，queue 存放所有包含定时任务节点的 bucket
        // 借助 DelayQueue 来检测 bucket 是否过期，bucket 时遍历即可取出所有节点
        queue.offer(bucket)
      }
      true
    } else {
      // 过期时间在当前时间轮表示的范围之外，即溢出，需要创建高一层时间轮来加入
      if (overflowWheel == null) addOverflowWheel() // 双重检查上锁的第一层检查
      overflowWheel.add(timerTaskEntry) // 注意高一层时间轮也可能无法容纳，因此可能会递归创建更高层级的时间轮
    }
  }
```

主要知识点在前面的设计思路中都讲到了，可以看到 `DelayQueue` 对象 `queue` 在时间轮的作用是，保存包含定时任务节点的桶，桶可以来自不同层次的时间轮，当然，所有层次时间轮也共享这个队列。

`TimeWheel` 本身没有实现 tick 功能，而是借助延迟队列 `DelayQueue` 来实现时间的推移，假设有 M 个定时任务分布在 N 个桶中，那么插入的时间复杂度为 `O(M + N * log N)`，其中 `M >= N`。如果把任务全存到延迟队列中，那么插入的时间复杂度为 `O(M * log M)`，因此 Kafka 时间轮的优化是有意义的。

比如对于 1 层时间轮的 3 个桶：`[0,4)`，`[4,8)`，`[8,12)`，有以下过期时间的定时任务：

```
1,2,3,8,9,10,11
```

那么会向 `queue` 中插入 2 个桶，然后利用 `queue` 依次弹出 2 个桶，通过遍历弹出每个桶的节点：

- 时刻 0：弹出节点 1,2,3；
- 时刻 8：弹出节点 8,9,10,11。

### 删除定时任务

再再再次回顾，延迟操作 `DelayedOperation` 对象，继承自定时任务 `TimerTask` 接口，而 `TimerTask` 会绑定一个 `TimerTaskEntry` 节点，每个节点位于唯一对应的链表 `TimerTaskList` （即 bucket）上。

定时任务的删除即调用 `TimerTaskList.remove` 方法（`TimerTaskEntry.remove` 也会调用该方法），有以下几种可能：

- 延时操作对象主动调用 `cancel` 和节点解绑，解绑后的节点也无法加入到 bucket 中；
- 当前 bucket 上的节点被另一个 bucket 调用 `add` 方法，此时会先从当前 bucket 上移除该节点。

## 总结

本文叙述了 Kafka 分层时间轮的设计思路，并阅读了其源码实现，在 Kafka 这种需要处理大量异步任务（延时请求、定时任务，都可以视为等价的概念）的系统上，基于优先级队列的 `DelayQueue` 性能不够高，因此 Kafka 借助了时间轮的思想，将同一个时间范围内的异步任务放到一个桶中，进一步将桶放入优先级队列。核心思想是同一个时间区间范围的多个任务，只需要加入一次到优先级队列中。

顺便，本文留下了一个问题，那就是 `queue` 调用了 `offer` 方法将 bucket 加入到队列中，但是在 `TimeWheel.scala` 源码中，没有看到 `queue` 调用 `poll` 方法弹出 bucket。其实是在 `SystemTimer` 中实现的，也就是下一篇文章要阅读的部分。