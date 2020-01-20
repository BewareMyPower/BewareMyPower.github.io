---
title: 'Kafka源码阅读10: 延迟操作DelayedOperation'
date: 2020-01-20 14:10:18
tags:
  - Kafka
top: 1
---
## 前言

之前阅读了 Produce 和 Fetch 请求的实现，对于需要耗时处理的网络请求，都是利用 `DelayedOperation` 和 `DelayedOperationPurgatory` 来进行异步延迟操作，防止阻塞 `KafkaRequestHandler` 线程。

比如处理 Produce 请求时，`ReplicaManager.appendRecords` 方法在 ack 为 -1，有数据发送且有至少有一个分区的 append 操作成功时：

```scala
        val delayedProduce = new DelayedProduce(timeout, produceMetadata, this, responseCallback, delayedProduceLock)
        val producerRequestKeys = entriesPerPartition.keys.map(new TopicPartitionOperationKey(_)).toSeq
        delayedProducePurgatory.tryCompleteElseWatch(delayedProduce, producerRequestKeys)
```

再比如，处理 Fetch 请求时，`ReplicaManager.fetchMessages` 方法在 timeout 大于 0，读取本地数据没出错且响应积攒的字节数足够多时：

```scala
      val delayedFetch = new DelayedFetch(timeout, fetchMetadata, this, quota, isolationLevel, responseCallback)
      val delayedFetchKeys = fetchPartitionStatus.map { case (tp, _) => new TopicPartitionOperationKey(tp) }
      delayedFetchPurgatory.tryCompleteElseWatch(delayedFetch, delayedFetchKeys)
```

用法很类似，首先创建 DelayedXXX 对象，然后对每个分区都创建 `TopicPartitionOperationKey` 对象组成 `Seq`，将两者传入 purgatory 中进行 tryXXX 操作，从命名和注释可以猜到，这个操作是尝试完全请求，就像 Scala 的 `Promise` 类的 `tryComplete` 方法一样，异步操作的常见模式就是下面两个非阻塞操作：

1. 启动一个任务异步执行，然后当前线程该干嘛干嘛；
2. 想要确认任务是否执行结束时，看一眼，如果结束了就取得结果。

不过除了确认操作（Operation）是否完成外，还会在没有完成的时候，监控（Watch）相应的延迟操作。

```scala
def tryCompleteElseWatch(operation: T, watchKeys: Seq[Any]): Boolean
```

```scala
/* used by delayed-produce and delayed-fetch operations */
case class TopicPartitionOperationKey(topic: String, partition: Int) extends DelayedOperationKey {

  def this(topicPartition: TopicPartition) = this(topicPartition.topic, topicPartition.partition)

  override def keyLabel = "%s-%d".format(topic, partition)
}
```

实际上 Key 可以是任意类型，只要实现了 `keyLabel` 方法，主题和分区组成的 Key 是用于延迟的 Produce 和 Fetch 操作，而对于其它请求/操作则用的其它类型的 Key，比如 JoinGroup 操作用 group id 和 consumer id 组成 Key。

通过前面的源码阅读可知，Produce 和 Fetch 请求全程都是按照分区去处理，也就是每个分区对应一个类型，然后对这个类型进行 map, filter 等等，所以这里传入 purgatory 的 key 可以唯一标识延迟处理的数据，比如 Fetch 操作中需要处理的 `FetchMetadata`：

```scala
case class FetchMetadata(/* 其它字段... */
                         // key: 分区; value: 获取分区的状态
                         fetchPartitionStatus: Seq[(TopicPartition, FetchPartitionStatus)]) 
```

## DelayOperation

### 主要方法

首先看看 `DelayedOperationPurgatory` 类及其 `tryCompleteElseWatch` 方法的完整签名

```scala
final class DelayedOperationPurgatory[T <: DelayedOperation](/* ... */)
        extends Logging with KafkaMetricsGroup {

  def tryCompleteElseWatch(operation: T, watchKeys: Seq[Any]): Boolean
```

参数 1 是泛型参数 `T`，该类型必须继承自 `DelayedOperation`，该类型是抽象类

```scala
abstract class DelayedOperation(override val delayMs: Long,
    lockOpt: Option[Lock] = None) extends TimerTask with Logging {
```

设置了毫秒级延时 `delayMs` 以及可选的锁 `lockOpt`。要实现一个延迟操作，也就是继承 `DelayedOperation` 类并重写（override）以下抽象方法：

```scala
  // 回调: 当延迟操作过期时执行, 因此 delayMs 到期时会强制完成
  def onExpiration(): Unit

  // 回调: 当操作完成时执行
  def onComplete(): Unit

  // 检查现在操作是否已经完成:
  // 1. 已完成, 则调用 forceComplete() 并返回 true 如果 forceComplete 返回 true;
  // 2. 否则返回 false
  def tryComplete(): Boolean
```

此外提供了原子 `Boolean` 类型的字段来标识是否完成

```scala
  private val completed = new AtomicBoolean(false)

  def isCompleted: Boolean = completed.get()

  def forceComplete(): Boolean = {
    if (completed.compareAndSet(false, true)) {
      cancel() // 调用基类 TimerTask 的方法取消当前定时任务
      onComplete() // 执行派生类自定义的回调
      true
    } else {
      false
    }
  }
```

该原子变量是在 `forceComplete` 中设置为 true 的，可能有多个线程尝试完成同一个任务，由于是执行原子 `Boolean` 的 CAS 操作，只有第一个线程会返回 true，`onComplete()` 回调只会被调用一次。

前面的注释也提过，简单查看 `DelayedProduce` 和 `DelayedFetch` 的 `tryComplete()` 实现也能看到，每个代表任务已完成的分支，都会将 `forceComplete()` 的返回值作为 `tryComplete()` 的返回值。

此外，基类 `TimerTask` 继承自 `Runnable` 接口，因此 `DelayedOperation` 可以作为线程被启动，执行 `run()` 方法：

```scala
  override def run(): Unit = {
    if (forceComplete())
      onExpiration()
  }
```

如果任务未完成，则强制完成（期间会执行 `onComplete` 回调），并执行 `onExpiration` 回调。相当于多了个对超时的处理，因此可以猜测会在过期时启动线程来执行超时回调。

### maybeTryComplete

该方法是 server 包私有的

```scala
  private[server] def maybeTryComplete(): Boolean = {
    var retry = false
    var done = false
    do {
      if (lock.tryLock()) {
        // 上锁成功, 直接调用 tryComplete
        try {
          tryCompletePending.set(false)
          done = tryComplete()
        } finally {
          lock.unlock()
        }
        // 此时可能另外一个线程调用了 `maybeTryComplete` 并将 `tryCompletePending` 置为 true(在此之前已经设为了 false)
        // 因此 retry 为 true 就代表这种情况发生, 此时会触发重试条件, 继续 while 循环
        retry = tryCompletePending.get()
      } else { // 上锁失败, 说明另一个线程持有锁
        // 如果此时 `tryCompletePending` 为 true, 那么持锁线程必然看到了 true 并且重试, 那么当前线程可以退出。
        // 否则持锁线程正在 `tryComplete`, 此时将其设为 true 因为持锁线程可能在 `tryCompletePending` 设为 true 的时候返回
        retry = !tryCompletePending.getAndSet(true)
      }
    } while (!isCompleted && retry)
    done
  }
```

看起来有点绕，分情况讨论。

1. 单线程：调用 `tryComplete` 后退出循环，因为自己将 `tryCompletePending` 置为了 false，解锁后`retry` 为false，此时和直接 `tryComplete` 无异；
2. 双线程：记为 A 和 B，假设 A 上锁成功，且在 `tryComplete` 检查完成状态的时候 B 上锁失败，那么 B 将 `tryCompletePending` 置为 true，这会导致两种情况：
   - A `tryComplete` 成功，代表 `onComplete` 已被调用，`isCompleted` 为 true，A 和 B 都会退出循环；
   - A `tryComplete` 失败，`isCompleted` 为 false，由于 `tryCompletePending` 被 B 置为 false，A 的 `retry` 为 true，而由于不存在其它等待线程，所以 B 在 `getAndSet` 时得到的值（赋给 `retry`）也是 true，A 和 B 重新争夺锁，也就是说至少会再调用一次 `tryComplete`。
3. 三个以上线程：存在 1 个持锁线程和 N 个等待线程（N > 1），`getAndSet` 是原子操作，也就是说 N 个等待线程只有 1 个等待线程的 `retry` 会被置为 true，其它线程都因为第一个调用 `getAndSet` 的等待线程将 `tryCompletePending` 置为 false 时退出，此时和双线程的情况无异。

核心就是双线程的情况，这种做法是为了针对这种场景：线程 A 检查完成状态的时候，此时还未完成，而线程 B 检查完成状态的时候，虽然实际已经完成了，但由于线程 A 正持有锁，B 不会检查状态。这种做法能让无论线程 A 还是 B，都会再调用一次 `tryComplete` 检查是否完成了。

举个例子，假设状态由 1 个 `Boolean` 表示，只有都为 `true` 时才算完成。如果不这么设计，那就可能出现下面的情况：

| 时刻 | 状态  | 线程 A                                | 线程 B               |
| ---- | ----- | ------------------------------------- | -------------------- |
| 1    | false | 上锁                                  |                      |
| 2    | false | 取得状态 false                        |                      |
| 3    | true  | 判断状态（已经是旧的状态）是否为 true | 上锁失败，不检查状态 |
| 4    | true  | 返回 false                            | 返回 false           |

而现在的做法就是在第 4 步，让线程 A 和 B 再次竞争 `tryComplete` 的机会，至少有一个线程能检查新的状态。

## TimerTask

`DelayedOperation.forceComplete` 有一个关键的 `cancel()` 调用来自于基类 `TimerTask`，从源码注释可知，这个方法是取消 timeout 计时器，即强行停止耗时超过 `delayedMs` 的延时任务。这个方法是在基类 `TimerTask` 中实现的。

```scala
  private[this] var timerTaskEntry: TimerTaskEntry = null

  def cancel(): Unit = {
    synchronized {
      if (timerTaskEntry != null) timerTaskEntry.remove()
      timerTaskEntry = null
    }
  }
```

实际上是调用了 `TimerTaskEntry.remove` 方法

```scala
private[timer] class TimerTaskEntry(val timerTask: TimerTask, val expirationMs: Long) extends Ordered[TimerTaskEntry] {

  @volatile
  var list: TimerTaskList = null

  def remove(): Unit = {
    var currentList = list
    // 如果另一个线程将当前节点从一个链表移动到另一个链表，由于 list 会被修改成新链表的引用
    // 所以 remove 会失败，因此在这里用 currentList 暂存之前链表的引用，这样就避免锁住整个 list
    // 单线程: list.remove(this) 会将 this.list 置为 null, 移除后退出循环；
    // 多线程: 如果 this.list 被其它线程修改指向了新的链表, 那么循环会继续, 将该节点从新链表移除
    // 一个罕见场景: 线程 B 将该节点从链表 A 移除加入链表 B, 但是在修改节点的 list 之前, 线程 A
    // 就移除成功了, 获取的 list 为 null, 退出循环, 之后线程 B 将节点加入链表 B
    while (currentList != null) {
      currentList.remove(this)
      currentList = list
    }
  }
}
```

`TimerTaskEntry` 是 `TimerTaskList` （定时任务链表）上的一个**节点（Entry）**，内部维护了链表的引用，调用 `remove` 即将当前节点从链表上移除。

```scala
  def remove(timerTaskEntry: TimerTaskEntry): Unit = {
    synchronized { // 保护多线程 remove 和 add
      timerTaskEntry.synchronized { // 保护单个节点的 add 和 remove
        if (timerTaskEntry.list eq this) { // 确认当前节点还在当前链表上才移除
          timerTaskEntry.next.prev = timerTaskEntry.prev
          timerTaskEntry.prev.next = timerTaskEntry.next
          timerTaskEntry.next = null
          timerTaskEntry.prev = null
          timerTaskEntry.list = null
          taskCounter.decrementAndGet()
        }
      }
    }
```

可见 `TimerTaskList` 是双向链表，移除节点时会锁住该节点，然后修改其 `prev` 和 `next`，并维护了原子变量的 `taskCounter` 记录任务节点的数量。除了 `remove` 外只有 `add` 方法会在多线程下造成 race condition，因此要加锁。

```scala
  def add(timerTaskEntry: TimerTaskEntry): Unit = {
    var done = false
    while (!done) {
      // 如果节点存在于另一个链表中, 将其删除从而保证一个节点只被一个链表拥有
      // 这里不加锁是因为 remove 也会调用 synchronized, 会造成死锁, remove 本身
      // 会反复重试直到节点的链表为 null
      timerTaskEntry.remove()

      synchronized { // 保护多线程 add
        timerTaskEntry.synchronized { // 保护单个节点的 add 和 remove
          if (timerTaskEntry.list == null) { // 确认当前节点已经成功被 remove
            val tail = root.prev
            timerTaskEntry.next = root
            timerTaskEntry.prev = tail
            timerTaskEntry.list = this
            tail.next = timerTaskEntry
            root.prev = timerTaskEntry
            taskCounter.incrementAndGet()
            done = true
          }
        }
      }
    }
  }
```

PS：这里似乎锁住单个节点没必要，因为 `add` 和 `remove` 本身都已经上锁了，大概是防止将来有其它方法直接修改节点的字段吧，毕竟节点的字段（这里主要指`list`）都不是私有的。（虽然我看到的对 `list` 的修改也就只在 `add`/`remove` 方法中）。

总之，这里单独实现的 `TimerTaskList`，除了实现了线程安全的增删操作外，主要是保证了链表上的节点（对应一个定时任务）是对应唯一的链表的，主要是为了保证节点在从链表 A 迁移到链表 B 时，不会继续留存在 A 中。

`TimerTask` 会绑定一个 `TimerTaskEntry` 类型的节点，该节点位于 `TimerTaskList` 类型的双向链表上，链表包含一个字段：`expirationMs`，即任务的毫秒级 timeout。任务链表在源码注释中也被称为 **bucket（桶）**，其本身也有一个原子类型的 `expiration` 字段代表任务链表本身的 timeout，提供了 setter 和 getter，需要注意的是 setter 返回的是 `Boolean` 而非 `Unit`，为 true 时则代表 `expiration` 发生了变化。

`TimerTask`  在绑定新的 `TimerTaskEntry` 时，如果和之前的节点不一样，也会将其移除：

```scala
  private[timer] def setTimerTaskEntry(entry: TimerTaskEntry): Unit = {
    synchronized {
      // if this timerTask is already held by an existing timer task entry,
      // we will remove such an entry first.
      if (timerTaskEntry != null && timerTaskEntry != entry)
        timerTaskEntry.remove()

      timerTaskEntry = entry
    }
  }
```

而在创建定时任务节点时，会自动和构造参数的 `TimerTask` 对象绑定：

```scala
private[timer] class TimerTaskEntry(val timerTask: TimerTask, val expirationMs: Long) extends Ordered[TimerTaskEntry] {
    
  // if this timerTask is already held by an existing timer task entry,
  // setTimerTaskEntry will remove it.
  if (timerTask != null) timerTask.setTimerTaskEntry(this)
```

## 总结

延时操作（`DelayedOperation`）是一个抽象基类，继承自定时任务（`TimerTask`），由派生类实现以下方法：

- 任务完成的回调；
- 任务超时的回调；
- 非阻塞地确认任务是否完成。

每个延时操作都是一个定时任务（`TimerTask`），对应一个定时任务节点（`TimerTaskEntry`），而每个任务节点都是存在一个 bucket（定时任务链表，`TimerTaskList`）上的。通过线程安全的 `remove` 和 `add` 操作可以让节点从一个 bucket 移动到另一个 bucket，整个过程中节点都始终对应唯一的 bucket，不可能被多个 bucket 共享。这使得任务能安全地在多个 bucket 之间迁移。也就是接下来要阅读的时间轮 `TimeWheel`。