---
title: 'Kafka源码阅读06: Produce请求(1): 写入本地日志'
date: 2019-11-06 18:43:19
tags:
  - Kafka
top: 1
---
## 回顾

前一节介绍了`Message`的格式及其实现，本来是继续阅读`MessageSet`，但后来发现在Kafka 0.11.0之后`Message`和`MessageSet`（消息集）发生了较大改变，详细参考[Kafka Protocol - Messagesets](https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol#AGuideToTheKafkaProtocol-Messagesets)，实际和生产者消费者交互的是`RecordBatch`而非`MessageSet`，原来的`MessageSet`只是简单地在若干`Message`之前加入offset字段和消息数量字段，现在的`RecordBatch`多了不少字段，比如`ProducerId`/`ProducerEpoch`等。目前脱离了对API协议的实际处理过程去看这些数据结构的实现很难明白其实际意义，因此先阅读API请求。

本文就先阅读生产者的请求，其类型为**Produce**，对应`KafkaApis.handle()`中的下列分支：

```scala
case ApiKeys.PRODUCE => handleProduceRequest(request)
```

## Produce协议

本节参考[Produce API](https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol#AGuideToTheKafkaProtocol-ProduceAPI)。Produce API使用通用的消息集格式，由于在发送时无法确定消息的offset，因此生产者可以随意填充该字段。

### 请求格式

Kafka 1.1使用的Produce请求是v2（实际上和v0及v1相同）：

```
ProduceRequest => RequiredAcks Timeout [TopicName [Partition MessageSetSize MessageSet]]
  RequiredAcks => int16
  Timeout => int32
  Partition => int32
  MessageSetSize => int32
```

这种描述格式是Kafka wiki的标准请求格式，`field => type`代表字段`field`是`type`类型，`field => [type]`代表`field`字段包含若干个`type`类型，也就是`[]`代表数组。

因此这里的消息请求的格式，可以看作包含1个2字节整型`RequiredAcks`，1个4字节整型`Timeout`，接下来是N个结构，每个结构都有1个`TopicName`，以及若干个子结构，每个子结构由1个`Partition`/`MessageSetSize`/`MessageSet`组成。

然后介绍官方对上述参数的定义：

- `RequiredAcks`（下文简称acks）

  指定服务端在响应请求之前应该受到多少确认（ack）:

  - 0：服务器不发送任何响应（这是服务器不回复请求的唯一情况）；
  - 1：服务器在等待数据写入本地日志后才会发送响应；
  - -1：服务器在等待所有同步副本提交消息之后才发送响应。

  0和1很好理解，0就是生产者发完就不管了，1就是等待消息被写入本地日志之后再返回，这里涉及到**同步副本（isr，in-sync replicas）**这个概念。这里简单介绍下。用Kafka自带脚本创建topic时会指定`--replication-factor`，也就是消息日志的复制数量，此时会创建多个**副本（replicas）**来保存消息日志，在**Leader**写入消息日志到本地时，副本也会从Leader取得消息，写入到自己的消息日志。暂且不提其同步过程，可以认为目前存活且消息写入跟上Leader的副本就是同步副本。

- `Timeout`

  服务器可以等待`RequiredAcks`指定数量的确认所用的最长时间，单位：ms。这个参数并不是请求时间的确切限制，因为：

  1. 网络传输延迟不包含在内；
  2. 计时器在处理请求时才开始，因此如果很多请求正在排队等待处理，那么这个等待时间不包含在内；
  3. 我们不会终止本地写操作，因此如果本地写入时间超时，将不予考虑，要获得这种类型的超时，客户端应该使用socket的超时。

- `TopicName`：发布数据的目标主题；

- `Partition`：发布数据的目标分区；

- `MessageSetSize`：紧接着的`MessageSet`字段的字节数；

- `MessageSet`：消息集的标准格式，参考[Protocol - Messagesets](https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol#AGuideToTheKafkaProtocol-Messagesets)，注意Kafka 1.1使用的是v2版本的RecordBatch。

### 响应格式

Kafka 1.1使用的是0.10.0后支持的v2版本，因此v0版本和0.9.0后支持的v1版本就先无视了。

```
ProduceResponse => [TopicName [Partition ErrorCode Offset Timestamp]] ThrottleTime
  TopicName => string
  Partition => int32
  ErrorCode => int16
  Offset => int64
  Timestamp => int64
  ThrottleTime => int32
```

- `Topic`：响应对应的主题；

- `Partition`：响应对应的分区；

- `ErrorCode`：当前分区的错误码；

  错误码是基于分区的，因为指定分区可能不可用或者无法在其他主机上维护而其他分区可能成功接受了Produce请求；

- `Offset`：赋值给消息集中第1条消息的offset；

- `Timestamp`：从UTC epoch至今的毫秒数，根据时间戳类型有不同的设定：

  - 时间戳类型为`LogAppendTime`，则为broker赋值给该消息集的时间戳，消息集内的所有内部消息都拥有同一个时间戳；
  - 时间戳类型为`CreateTime`，则该字段总是-1。

  如果没有错误码返回，那么生产者可以认为Produce请求的时间戳已被broker接受。

- `ThrottleTime`：由于超过了quota（限额）而导致请求被限流的时间间隔，单位：毫秒。

## handleProduceRequest

```scala
  def handleProduceRequest(request: RequestChannel.Request) {
    // 将 ByteBuffer 类型的请求解析成 ProduceRequest 类型
    val produceRequest = request.body[ProduceRequest]
    // 取得请求的总字节数, 包含 header 和 body
    val numBytesAppended = request.header.toStruct.sizeOf + request.sizeOfBodyInBytes
    // ...
  }
```

其中`header`和`sizeofBodyInBytes`在`network.RequestChannel`类中定义

```scala
  class Request(/* ... */
                val context: RequestContext,
                /* ... */
                @volatile private var buffer: ByteBuffer,
                /* ... */) extends BaseRequest {
    private val bodyAndSize: RequestAndSize = context.parseRequest(buffer)

    def header: RequestHeader = context.header
    def sizeOfBodyInBytes: Int = bodyAndSize.size
    // ...
  }
```

请求头之前在[网络层阅读之RequestChannel](https://bewaremypower.github.io/2019/09/23/Kafka源码阅读03-网络层阅读之RequestChannel/)中提过，这里简单回顾下。`RequestHeader`为Java类，定义在`org.apache.kafka.common.requests`包中，包含以下字段

```java
    private final ApiKeys apiKey;    // 请求类型
    private final short apiVersion;  // API版本
    private final String clientId;   // 用户指定的客户端ID
    private final int correlationId; // 用户提供的整数值，将和响应一起返回
```

对应[消息协议](https://kafka.apache.org/protocol.html#protocol_messages)的Headers：

```
Request Header => api_key api_version correlation_id client_id 
  api_key => INT16
  api_version => INT16
  correlation_id => INT32
  client_id => NULLABLE_STRING
```

在`Processor`处理客户端的请求字节序列时，会调用`RequestHeader.parse`方法构造请求头，然后和字节序列`buffer`一起发送给`RequestChannel`，`Handler`线程从中取得请求发送给`KafkaApis`处理。

后面是一些认证相关的代码，调用了`authorize`方法，由于不影响主要流程，所以暂且跳过，最后会进入以下分支：

```scala
      val internalTopicsAllowed = request.header.clientId == AdminUtils.AdminClientId

      // call the replica manager to append messages to the replicas
      // 传入replicaManager来添加消息到副本上
      replicaManager.appendRecords(
        timeout = produceRequest.timeout.toLong, // Produce请求的timeout字段
        requiredAcks = produceRequest.acks, // Produce请求的acks字段
        internalTopicsAllowed = internalTopicsAllowed, // client id是否为__admin_client
        isFromClient = true, // 这里是处理客户端的Produce请求，所以为true
        entriesPerPartition = authorizedRequestInfo, // 通过认证的请求信息
        responseCallback = sendResponseCallback, // 发送响应的回调函数
        processingStatsCallback = processingStatsCallback) // 处理stats的回调函数

      // if the request is put into the purgatory, it will have a held reference and hence cannot be garbage collected;
      // hence we clear its data here in order to let GC reclaim its memory since it is already appended to log
      produceRequest.clearPartitionRecords() // 简单将Produce请求的partitionRecords置为null
```

留意最后的操作，提到了**purgatory**这个概念：如果请求被放入purgatory，那么就会被（purgatory）持有引用，因此将其置为`null`防止被垃圾收集。也是之后涉及再看。

其中，`entriesPerPartition`是之前认证过程得到的信息：

```scala
    val authorizedRequestInfo = mutable.Map[TopicPartition, MemoryRecords]()

    // 从 produceRequest.partitionRecords 取得所有 TopicPartiton 和 MemoryRecords
    // ***OrFail 方法仅仅检查 partitionRecords 字段是否为 null, 若为 null 则抛出异常
    for ((topicPartition, memoryRecords) <- produceRequest.partitionRecordsOrFail.asScala) {
      if (!authorize(request.session, Write, new Resource(Topic, topicPartition.topic)))
        unauthorizedTopicResponses += topicPartition -> new PartitionResponse(Errors.TOPIC_AUTHORIZATION_FAILED)
      else if (!metadataCache.contains(topicPartition.topic))
        nonExistingTopicResponses += topicPartition -> new PartitionResponse(Errors.UNKNOWN_TOPIC_OR_PARTITION)
      else // 通过了 authorize 方法认证, 并且 metadataCache 包含该 topic
        authorizedRequestInfo += (topicPartition -> memoryRecords)
    }

```

### Idea调试

追根刨底去看`metadataCache`的构造和读取略麻烦，而且偏离了我们这篇文章的核心目的（了解Kafka怎么处理Produce请求）这里就利用Intellij Idea调试先看看里面到底是什么，也是阅读源码以来第1次调试。

首先`zkServer`命令启动Zookeeper服务端，然后在Idea中在定义`authorizedRequestInfo`处设断点，调试模式启动Kafka的core模块（即Kafka服务端），然后启动Kafka客户端，向`test`主题发送字符串`hello`，此时可以看到`metadataCache`的结构：

- `brokerId` = 0
- `cache` = "HashMap" size = 2
  - 0 = ...
    - _1 = "__consumer_offsets"
      - `value` = {char[18]@5303}
      - `hash` = -970371369
    - _2 = "HashMap" size = 50
  - 1 = ...
    - _1 = "test"
      - `value` = {char[4]@5410}
      - `hash` = 3556498
    - _2 = "HashMap" size = 1

可见其`cache`字段为`HashMap`类型，包含了所有的topic，一个是我们创建的`test`主题，一个是用来管理消费者提交的offset的`__consumer_offsets`。

因此保证了`authorizedRequestInfo`，也就是传入`appendRecords`的`entriesPerPartition`参数，它的topic都是目前现有的。

## ReplicaManager.appendRecords

> 将消息添加到分区的首领副本，等待它们被复制到其他副本。无论是timeout或者acks的条件被满足，都会触发回调函数。如果回调函数本身已经在某个对象上被同步，那么传递这个对象来避免死锁。

```scala
  def appendRecords(timeout: Long,
                    requiredAcks: Short,
                    internalTopicsAllowed: Boolean,
                    isFromClient: Boolean,
                    entriesPerPartition: Map[TopicPartition, MemoryRecords],
                    responseCallback: Map[TopicPartition, PartitionResponse] => Unit,
                    delayedProduceLock: Option[Lock] = None,
                    processingStatsCallback: Map[TopicPartition, RecordsProcessingStats] => Unit = _ => ()) {
    if (isValidRequiredAcks(requiredAcks)) { // acks只能为-1，0，1
      // ...
    } else {
      // acks 在可接受的范围外, 则客户端肯定出错了, 仅仅返回错误, 而不用处理请求
      // 具体处理: 对每个 TopicPartition 对象, 构造相应的 PartitionResponse 对象组成新的 Map
      //  其中包含 error, baseOffset, logAppendTime, logStartOffset 等字段,
      //  除了 error 字段标明为 acks不合法 外, 其余字段都随意设置
      val responseStatus = entriesPerPartition.map { case (topicPartition, _) =>
        topicPartition -> new PartitionResponse(Errors.INVALID_REQUIRED_ACKS,
          LogAppendInfo.UnknownLogAppendInfo.firstOffset, RecordBatch.NO_TIMESTAMP, LogAppendInfo.UnknownLogAppendInfo.logStartOffset)
      }
      // 调用传入的回调 responseCallback 将返回值发送回去
      responseCallback(responseStatus)
    }

```

先看else分支，可以得知，传入的`entriesPerPartition`为`TopicPartition`到`MemoryRecords`（消息）的`Map`而传入的`responseCallback`为发送响应给客户端的回调函数，响应类型也是`Map`，key也是`TopicPartition`，只不过value变成了`PartitionResponse`。也就是说，无论是请求还是响应，都是以分区为单位的，对于错误的响应，只有`error`字段起作用，而正确的响应是包含`baseOffset`，`logAppendTime`和`logStartOffset`等字段，前2个字段在上一篇[消息协议阅读](https://bewaremypower.github.io/2019/10/14/Kafka源码阅读05-消息协议阅读之Message/)中简单提过，分别是消息日志中第1个offset以及发送的消息被写入消息日志的时间戳，现在具体阅读acks合法时的处理流程。

### time字段

首先取得毫秒级的`time`：

```scala
      val sTime = time.milliseconds

```

其中`time`为`replicaManager`的构造参数，而`replicaManager`也是`KafkaApis`的构造参数：

```scala
class ReplicaManager(val config: KafkaConfig,
                     metrics: Metrics,
                     time: Time,

```

```scala
class KafkaApis(val requestChannel: RequestChannel,
                val replicaManager: ReplicaManager,

```

`KafkaApis`对象是在`KafkaServer`的`startup`方法中创建的，层层追溯如下：

```scala
apis = new KafkaApis(socketServer.requestChannel, replicaManager, /* ... */)

```

```scala
replicaManager = createReplicaManager(isShuttingDown)

```

```scala
  protected def createReplicaManager(isShuttingDown: AtomicBoolean): ReplicaManager =
    new ReplicaManager(config, metrics, time, /* ... */)

```

```scala
class KafkaServer(val config: KafkaConfig, time: Time = Time.SYSTEM,

```

```java
public interface Time {
    Time SYSTEM = new SystemTime();

```

可见`time`的`SystemTime`对象，作为计时器，包含以下常用方法：

- `milliseconds`：取得毫秒级时间戳；
- `nanoseconds`：取得纳秒级时间戳；
- `sleep(long ms)`：当前线程休眠指定毫秒数。

因此Kafka中一切用到计时器的类都会使用该对象，回过头看`appendRecords`代码：

```scala
      val sTime = time.milliseconds // 取得当前毫秒级时间戳
      val localProduceResults = appendToLocalLog(internalTopicsAllowed = internalTopicsAllowed,
        isFromClient = isFromClient, entriesPerPartition, requiredAcks)
      // 调试信息: 再次取得时间戳, 相减得到 appendToLocalLog 的用时
      debug("Produce to local log in %d ms".format(time.milliseconds - sTime))

```

也就是说首先会调用`appendToLocalLog`方法

### appendToLocalLog

> 将消息添加到本地副本日志中

```scala
  private def appendToLocalLog(internalTopicsAllowed: Boolean,
                               isFromClient: Boolean,
                               entriesPerPartition: Map[TopicPartition, MemoryRecords],
                               requiredAcks: Short): Map[TopicPartition, LogAppendResult] = {
    trace(s"Append [$entriesPerPartition] to local log")
    // 遍历所有客户端请求写入的 topicPartition 以及对应消息 records
    entriesPerPartition.map { case (topicPartition, records) =>
      // 更新topicStats，暂时略去不看
      brokerTopicStats.topicStats(topicPartition.topic).totalProduceRequestRate.mark()
      brokerTopicStats.allTopicsStats.totalProduceRequestRate.mark()
      if (Topic.isInternal(topicPartition.topic) && !internalTopicsAllowed) {
        // topic是内部主题: __consumer_offsets 或 __transaction_state, 且 internalTopicsAllowed 为 false
        //  (在 KafkaApis.handleProduceRequest 中, 只有请求的 clientId 为 AdminClientId 时才为 true)
        // 也就是如果不是 Admin 客户端, 尝试写入内部主题则会返回 写入不合法 的 LogAppendResult
        (topicPartition, LogAppendResult(
          LogAppendInfo.UnknownLogAppendInfo,
          Some(new InvalidTopicException(s"Cannot append to internal topic ${topicPartition.topic}"))))
      } else { // 非内部主题, 可以写入
        try {
          // ...
        } catch {
          // 异常处理(略)，会将处理客户端请求的异常信息写入返回结果中
          // 注意，对于用于流程控制的Throwable异常，会单独处理，这里后面再看
        }
      }
    }

```

首先是区分了消费主题是否为内部主题，比如`__consumer_offsets`，这种主题并不是存储生产/消费的消息的，因此只允许Admin客户端读写。至于`brokerTopicStats`也是度量指标相关的，暂且略过。

```scala
// 从内部的 allPartitions 中找到 topicPartition, PS: allPartitions 是从本地消息日志中读取的
val partitionOpt = getPartition(topicPartition)
val info = partitionOpt match {
  case Some(partition) =>
    // 找到的是 OfflinePartition (当前broker不在分区的ISR列表上) 则会(通过异常处理)返回错误信息
    // https://issues.apache.org/jira/browse/KAFKA-6796 在 Kafka 2.0 中对这种行为进行了修复
    // 比如在分区重分配期间, 客户的Produce请求在本地副本被删除后到达, 此时不应该返回分区不存在的错误
    // 因此2.0中抛出的是 NotLeaderForPartitionException, 会强制让客户端更新元数据来找到新的分区位置
    if (partition eq ReplicaManager.OfflinePartition)
      throw new KafkaStorageException(s"Partition $topicPartition is in an offline log directory on broker $localBrokerId")
    // 添加记录到leader副本上
    partition.appendRecordsToLeader(records, isFromClient, requiredAcks)

  // 若找不到目标 topicPartition, 则代表生产者向一个未知的分区生产消息, 返回表示分区不存在的结果
  case None => throw new UnknownTopicOrPartitionException("Partition %s doesn't exist on %d"
    .format(topicPartition, localBrokerId))
}

// 略去更新brokerTopicStats的代码

(topicPartition, LogAppendResult(info))

```

处理了2种错误：分区是离线的（Offline）和分区是未知，而对于已知分区，则是将`appendRecordsToLeader`方法返回的`info`来构造该分区对应的`LogAppendResult`作为返回结果。

这里通过`getPartition`返回的`partition`类型是`Partition`，位于`cluster`包中：

```scala
class Partition(val topic: String,
                val partitionId: Int,
                time: Time,
                replicaManager: ReplicaManager,
                val isOffline: Boolean = false)

```

除了主题名`topic`和分区号`partitionId`外，还会引用`replicaManager`用于将信息写入副本中。还通过`isOffline`来区分分区是否在副本broker上。

### Partition.appendRecordsToLeader

```scala
  def appendRecordsToLeader(records: MemoryRecords, isFromClient: Boolean, requiredAcks: Int = 0): LogAppendInfo = {
    // 用读锁保护, 可以多线程添加记录到 leader副本 上, 但是如果ISR更新过程会获取写锁, 此时要等待ISR更新完毕
    val (info, leaderHWIncremented) = inReadLock(leaderIsrUpdateLock) {
      // 若leader副本的id为本地的broker id, 则返回对应的 Replica对象
      leaderReplicaIfLocal match {
        case Some(leaderReplica) => // leader副本
          val log = leaderReplica.log.get // append-only的Log对象
          val minIsr = log.config.minInSyncReplicas // min.insync.replicas 配置
          val inSyncSize = inSyncReplicas.size // ISR数量

          // acks为-1时, 客户端会等待所有ISR确认收到消息时才返回, 此时配置 min.insync.replicas
          // 指定了这个ISR的最小数量, 因此ISR数量不够时会抛出ISR副本太少的异常
          if (inSyncSize < minIsr && requiredAcks == -1) {
            throw new NotEnoughReplicasException("Number of insync replicas for partition %s is [%d], below required minimum [%d]"
              .format(topicPartition, inSyncSize, minIsr))
          }

          // 将消息集写入日志, 分配 offsets 和 分区leader的epoch
          val info = log.appendAsLeader(records, leaderEpoch = this.leaderEpoch, isFromClient)
          // probably unblock some follower fetch requests since log end offset has been updated
          // 因为 LEO(log end offset) 已经更新了, 所以某些 follower 的 fetch请求可能解除阻塞了, 于是
          // replicaManager.delayedFetchPurgatory 尝试完成该分区的延迟的fetch请求, 因为 LEO(log end offset)已经跟新
          replicaManager.tryCompleteDelayedFetch(TopicPartitionOperationKey(this.topic, this.partitionId))
          // 因为 ISR 可能只剩1个, 因此可能需要增加HW (high watermark)
          (info, maybeIncrementLeaderHW(leaderReplica))

        case None => // 非leader副本
          throw new NotLeaderForPartitionException("Leader not local for partition %s on broker %d"
            .format(topicPartition, localBrokerId))
      }
    }

    // some delayed operations may be unblocked after HW changed
    // 一些延迟操作可能因为 HW 的改变而解除阻塞, 因此尝试完成这些延迟请求
    if (leaderHWIncremented)
      tryCompleteDelayedRequests()

    info // log.AppendAsLeader返回的结果
  }

```

这里有几个方法暂时没看细节，将其列出（对于`server`包之外的标注包名），之后有可能的话单独阅读：

- `log.Log.appendAsLeader`：将消息集，分配的offset，leader副本的epoch写入本地消息日志；
- `DelayedOperation.checkAndComplete(key: Any)`：检查某些**延迟操作(delayed operations)**用给定的key能否完成，若能则完成；
- `cluster.Partition.maybeIncrementLeaderHW`：检查并且可能增加分区的HW，仅当分区ISR改变或者任意副本的LEO改变时才更新。

由于本小节涉及到分区的操作，来回顾一些基本概念，每个分区都有多个broker来保存，实现消息的冗余备份，这些broker称为该分区的**副本（replica）**。对每个分区，存在唯一的**leader副本**（通过选举产生），与客户端进行直接读写，而其他副本为**follower**，不断地从leader复制最新的消息。与leader保持同步的follower被称为**ISR(in-sync replica)**，而某些follower会因为某些原因复制速度较慢或者和leader断开连接（通过某种规则判断），此时会从ISR中移除，直到重新跟上进度会重新加入ISR。

**HW(high watermark, 高水位)**即最新**已提交的（committed）**消息的offset，即所有ISR的分区日志上都写入了该消息，消费者无法拉取比HW更大的offset，从而保证leader一旦不可用，消费者之前消费的消息存在于任意ISR的消息日志中。

**LEO(log end offset)**是所有副本都会维护的offset，即当前副本最后一个消息的offset+1，也就是如果有新的消息写入，那么它的offset即之前的LEO，而副本将消息写入消息日志后，LEO会递增。

至于**epoch**这个概念是Kafka 0.11引入的，暂时还不清楚具体功能，之后再提。

## appendToLocalLog总结

在之前将客户端发送的请求解析成了**分区**到**消息集**的映射，而返回值是**分区**到`LogAppendResult`的映射，因此只对遍历整个`Map`，对每对分区消息集进行处理得到`LogAppendResult`即可：

1. 对`__consumer_offsets`这样的内部主题，验证请求头的client id是否为管理员（admin）的id，否则返回*Cannot append to internal topic*的错误；
2. 在`ReplicaManager`维护的当前broker上的分区列表中找到对应的分区；
3. 若查找失败则返回*Partition ... doesn't exist on ...*的错误；
4. 若分区不可用，则返回*Partition ... is in an offline log directory on broker ...*的错误；
5. 若当前broker不是分区的leader，则返回*Leader not local for partition ... on broker ...*的错误；
6. 若acks字段为-1，且ISR数量小于`min.insync.replicas`配置的数量，则返回*Number of insync replicas for partition ... is ... below required minimum*的错误；
7. 将消息集写入本地日志，并给当前分区分配offsets和leader epoch；
8. 处理延后处理的Fetch请求，可能更新HW；
9. 若更新HW，则处理延后处理的请求。

前面的流程都是一些合法性判断，主要是7~9这几步，待深入阅读的内容：

1. 对指定分区，写入日志后如何分配offsets和leader epoch？
2. 延后处理是怎么实现的？

关于延后处理，主要是`ReplicaManager`的以下字段

```scala
val delayedProducePurgatory: DelayedOperationPurgatory[DelayedProduce],
val delayedFetchPurgatory: DelayedOperationPurgatory[DelayedFetch],
val delayedDeleteRecordsPurgatory: DelayedOperationPurgatory[DelayedDeleteRecords]

```

都是`Purgatory`（炼狱），在辅助构造器中进行默认构造：

```scala
      DelayedOperationPurgatory[DelayedProduce](
        purgatoryName = "Produce", brokerId = config.brokerId,
        purgeInterval = config.producerPurgatoryPurgeIntervalRequests),
      DelayedOperationPurgatory[DelayedFetch](
        purgatoryName = "Fetch", brokerId = config.brokerId,
        purgeInterval = config.fetchPurgatoryPurgeIntervalRequests),
      DelayedOperationPurgatory[DelayedDeleteRecords](
        purgatoryName = "DeleteRecords", brokerId = config.brokerId,
        purgeInterval = config.deleteRecordsPurgatoryPurgeIntervalRequests),

```

都是泛型类`DelayedOperationPurgatory`，类型参数不同。

## 总结

本篇开始阅读Produce请求的处理，首先从官网阅读了Kafka 1.1对应的Produce请求和响应协议，然后阅读`KafkaApis`类的处理方法`handleProduceRequest`。

跳过了加密/认证的部分，实际上是由`ReplicaManager`来处理，调用`appendRecords`方法，接受了客户端Produce请求中的acks和timeout两个关键字段。

首先验证acks是否合法（-1, 0 or 1），对不合法acks发送`INVALID_REQUIRED_ACKS`响应。

然后调用`appendToLocalLog`方法，也是本篇主要阅读的部分。

之后的处理，以及`appendRecords`接收的回调函数（比如如何发送响应）的实现，日志的写入，分区的offsets和leader epoch的更新，以及如何延迟处理将在之后进行阅读。