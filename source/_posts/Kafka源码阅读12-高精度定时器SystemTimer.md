---
title: 'Kafka源码阅读12: 高精度计时器SystemTimer'
date: 2020-02-05 19:22:54
tags:
  - Kafka
top: 1
---
## 前言

前一篇阅读了时间轮 `TimingWheel` 的实现，遗留了两个重要问题：

1. 时间轮中被插入延迟队列的桶，何时被移除？
2. 高层时间轮运转时，定时任务何时被插入低层时间轮？

实际上，在 `kafka.utils.timer` 包的类中，真正暴露给其它包的只有 `SystemTimer`，而且注解为 `@threadsafe`（线程安全），时间轮 `TimingWheel` 只不过是它的一个字段，本身注解也是 `@nonthreadsafe`（非线程安全）。`SystemTimer` 实现了接口 `Timer`，是基于 Kafka 时间轮设计的高性能定时器。

## 构造

### 字段含义

```scala
class SystemTimer(executorName: String,
                  tickMs: Long = 1,
                  wheelSize: Int = 20,
                  startMs: Long = Time.SYSTEM.hiResClockMs) extends Timer {

  // 线程池（任务执行器），固定大小为 1，也就是同时只能执行最多一个任务
  private[this] val taskExecutor = Executors.newFixedThreadPool(1, new ThreadFactory() {
    // 自定义线程创建方式：非守护线程，指定 executorName 作为线程名后缀
    def newThread(runnable: Runnable): Thread =
      KafkaThread.nonDaemon("executor-"+executorName, runnable)
  })

  private[this] val delayQueue = new DelayQueue[TimerTaskList]()
  private[this] val taskCounter = new AtomicInteger(0)
  private[this] val timingWheel = new TimingWheel(
    tickMs = tickMs,
    wheelSize = wheelSize,
    startMs = startMs,
    taskCounter = taskCounter,
    delayQueue
  )

  // 读写锁，保护时间轮运转（tick）时的相关数据结构
  private[this] val readWriteLock = new ReentrantReadWriteLock()
  private[this] val readLock = readWriteLock.readLock()
  private[this] val writeLock = readWriteLock.writeLock()
```

主构造器 4 个参数第一个用于指定线程名称，后面三个用于构造时间轮。此外，所有时间轮（包括各个桶）共享一个延迟队列和任务计数器。**多层时间轮共享的延迟队列**就是这里的 `delayQueue`，调用 `poll` 时会将过期的桶弹出队列。

### 细节 1：高精度时间戳计时

注意到 `startMs` 是通过 `System.nanoTime()` 转换得到的高精度纳秒级时间戳：

```java
public interface Time {

    Time SYSTEM = new SystemTime();
```

```java
public class SystemTime implements Time {

    @Override
    public long milliseconds() {
        return System.currentTimeMillis();
    }

    @Override
    public long hiResClockMs() {
        return TimeUnit.NANOSECONDS.toMillis(nanoseconds());
    }

    @Override
    public long nanoseconds() {
        return System.nanoTime();
    }
```

之所以使用 `nanoTime` 是为了高精度计时，但是由于纳秒级时间戳超过了 64位 整型能表达的上限，所以得到的是溢出值（还有可能为负数），只能用于计算两个时间戳的时间间隔，而不能用作时间戳。因此在记录时间戳（比如 `Produce` 请求得到 `LogAppendTime` 时）以及对时间间隔精确性不敏感的地方都是用的 `currentMilliseconds` 方法计时。

### 细节 2：`KafkaThread`

Java 线程池屏蔽了线程的细节，用户只要提供了实现 `Runnable` 的类，即可通过 `execute` 或 `submit` 方法创建线程。出于灵活性考虑，Java 线程池也支持用户自定义 `ThreadFactory` 接口，实现 `newThread` 通过 `Runnable` 对象创建线程的方法。

```java
    public static KafkaThread nonDaemon(final String name, Runnable runnable) {
        return new KafkaThread(name, runnable, false);
    }

    public KafkaThread(final String name, Runnable runnable, boolean daemon) {
        super(runnable, name);
        configureThread(name, daemon);
    }

    private void configureThread(final String name, boolean daemon) {
        setDaemon(daemon);
        setUncaughtExceptionHandler(new UncaughtExceptionHandler() {
            public void uncaughtException(Thread t, Throwable e) {
                log.error("Uncaught exception in thread '{}':", name, e);
            }
        });
    }
```

这里是通过 `KafkaThread` 类（位于 `org.apache.kafka.common.utils` 包下）的工厂方法创建的，关键的是设置了异常处理器，当线程函数中抛出意想不到的异常时，将其写入错误日志。

但是，仅当 `Runnable` 对象由 `execute` 执行时才会调用这个处理器，因为 `submit` 执行 `Runnable` 会返回 `Future<?>` 对象，只有调用 `Future` 对象的 `get` 方法时才会触发异常，这样用户就可以手动 `try-catch` 捕获异常，而不用自定义异常处理器。

而在 `SystemTimer` 中，任务是使用 `submit` 执行的，并且未处理返回的 `Future` 对象：

```scala
taskExecutor.submit(timerTaskEntry.timerTask)
```

因此，虽然 `KafkaThread` 设置了异常处理器，但是在这里，定时任务抛出的异常实际上被忽略了。

## Timer 接口实现

### 接口概览

`SystemTimer` 是基于 `TimingWheel` 实现的定时器，对外提供的功能即它所实现的接口 `Timer`:

```scala
trait Timer {
  /**
    * 添加新的任务到当前执行器（线程池），在任务过期后会执行任务。
    * @param timerTask 待添加的任务
    */
  def add(timerTask: TimerTask): Unit

  /**
    * 推进内部时钟，执行任何在走过的时间间隔内过期的任务
    * @param timeoutMs
    * @return 是否有任务被执行
    */
  def advanceClock(timeoutMs: Long): Boolean

  // 取得待执行的任务数量
  def size: Int

  // 关闭定时器服务，待执行的任务将不会被执行
  def shutdown(): Unit
}
```

其中 `size` 和 `shutdown` 的实现很简单，分别是取得 `taskCounter` 的值以及关闭线程池。

```scala
  def size: Int = taskCounter.get

  override def shutdown() {
    taskExecutor.shutdown()
  }
```

### `add`

```scala
def add(timerTask: TimerTask): Unit = {
  readLock.lock()
  try {
    // 通过任务的延时加上当前时间得到延时的具体时刻，作为定时任务的过期时间
    addTimerTaskEntry(new TimerTaskEntry(timerTask, timerTask.delayMs + Time.SYSTEM.hiResClockMs))
  } finally {
    readLock.unlock()
  }
}

private def addTimerTaskEntry(timerTaskEntry: TimerTaskEntry): Unit = {
  // 尝试将任务加入时间轮
  if (!timingWheel.add(timerTaskEntry)) {
    // 仅当 任务已经过期 或者 任务主动取消 才会进入此分支
    if (!timerTaskEntry.cancelled) // 任务过期则执行任务
      taskExecutor.submit(timerTaskEntry.timerTask)
  }
}
```

其实就是将任务扔进时间轮中，添加失败只有可能是过期或者主动取消，这里额外判断了是否任务主动取消。

唯一值得注意的是这里用了读锁，按照常理，`add` 并不是读操作而是写操作，为什么是读锁呢？读锁意味着可以多线程同时调用 `add` 时无需上锁。这是因为 `TimingWheel.add` 是线程安全的，回顾下时间轮添加任务的流程：

1. 判断任务是否被取消

   任务绑定的 `Entry` 是 `private[this]` 修饰的，也就是仅有当前对象能访问。因此只要不是两个相同任务，那么这个判断是线程安全的

2. 判断过期时间处于那个桶，是否需要加入更高一级时间轮

   所有桶的时间范围由 `currentTime`（即取整后的 `startMs`）、`tickMs`、`wheelSize` 决定的，而 `add` 方法并不会修改它们

3. 将任务添加进桶：`TimeTaskList.add` 内部用内置锁保护了，线程安全；

4. 设置桶的过期时间：调用原子变量的 `getAndSet` 方法，也是线程安全的。

保证线程安全的策略是要么不修改内部状态，要么调用那些线程安全的方法，因此允许并发地 `add`。

### `advanceClock`

```scala
  def advanceClock(timeoutMs: Long): Boolean = {
    // 尝试在 timeoutMs 内取出完成的任务
    var bucket = delayQueue.poll(timeoutMs, TimeUnit.MILLISECONDS)
    if (bucket != null) { // 取出了过期的 bucket
      writeLock.lock()
      try {
        while (bucket != null) {
          // 推进当前时间轮，内部可能会递归推进更高一层时间轮，currentTime 被修改
          timingWheel.advanceClock(bucket.getExpiration())
          // 取出 bucket 所有任务节点，将其传入 reinsert 方法
          bucket.flush(reinsert)
          // 非阻塞地取出任务，将当前时点所有过期的 bucket 全部取出
          bucket = delayQueue.poll()
        }
      } finally {
        writeLock.unlock()
      }
      true
    } else { // 没有 bucket 过期
      false
    }
  }
```

推进 `timeoutMs` 毫秒，尽可能取出此时所有过期的 bucket（**问题 1 解决**），然后调用 `flush` 将 bucket 中所有任务节点传入 `reinsert`。

```scala
  def flush(f: (TimerTaskEntry)=>Unit): Unit = {
    synchronized {
      var head = root.next
      while (head ne root) {
        remove(head)
        f(head)
        head = root.next
      }
      expiration.set(-1L)
    }
  }
```

`TimerTaskList.flush` 方法很简单，用内置锁保护，然后依次删除链表（bucket）所有节点，并应用到函数上，最后重置 `expiration` 以保证下次有任务加入该 bucket 时，该 bucket 会被加入延迟队列。

```scala
private[this] val reinsert = (timerTaskEntry: TimerTaskEntry) => addTimerTaskEntry(timerTaskEntry)
```

`reinsert` 则是尝试将这些从 bucket 中删除的节点重新加入时间轮。

这里需要注意， bucket 过期时，内部节点也都过期了，因为 bucket 的过期时间是所有内部过期时间取整后得到的被 `tickMs` 整除的值。那为什么要这么做呢？

回顾我们开始提出的问题 2，如果取出的这个 bucket 是属于高层时间轮的，由于高层时间轮精度不够，此时 bucket 可能并未过期。

举个两层时间轮的例子（单位：毫秒）：

| 层次 | buckets         |
| ---- | --------------- |
| 1    | `[0,1)` `[1,2)` |
| 2    | `[0,2)` `[2,4)` |

初始状态下，延时为 3 的任务被加入 `[2,4)`，调用 `advanceClock(2)` 后，时间轮变成了

| 层次 | buckets         |
| ---- | --------------- |
| 1    | `[2,3)` `[3,4)` |
| 2    | `[2,4)` `[4,6)` |

第 2 层的 `[2,4)` 被取出，然后延时为 3 的任务被取出，此时调用 `reinsert` 就会将其加入第 1 层的 `[3,4)`，而不是立刻判断它过期。至此，**问题 2 解决**，从高层时间轮降级到底层时间轮被隐藏在了这句不起眼的 `bucket.flush(reinsert)` 中。

## 总结

本章阅读了 Kafka 高精度定时器 `SystemTimer` 的实现，它管理了延迟队列和时间轮，每次加入定时任务将任务扔进时间轮中，并将任务节点所在的 bucket 扔进延迟队列中。它本身的推进是通过延迟队列进行的，每次推进一段时间，尽可能取出到期的 bucket，并依次取出 bucket 的所有任务节点。通过将取出的任务节点重新加入到时间轮中，可能会将高层时间轮中过期任务转移到底层时间轮中。

此外，对于到期的任务，`SystemTimer` 使用仅包含单线程的线程池执行，若推进时又多个任务节点被取出，会等待前一个任务对应的线程完成后才会继续执行该任务（复用这个线程）。