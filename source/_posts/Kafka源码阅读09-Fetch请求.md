---
title: 'Kafka源码阅读09: Fetch请求'
date: 2020-01-13 12:55:45
tags:
  - Kafka
top: 1
---
## Fetch协议

Fetch API用于为某些分区获取日志，逻辑上它指定**主题**，**分区**和**起始offset**来取得消息，消息格式参考[The Messages Fetch](https://kafka.apache.org/protocol.html#The_Messages_Fetch)

## KafkaApis.handleFetchRequest

```scala
    val versionId = request.header.apiVersion
    val clientId = request.header.clientId
    val fetchRequest = request.body[FetchRequest]
    val fetchContext = fetchManager.newContext(fetchRequest.metadata(),
      fetchRequest.fetchData(),
      fetchRequest.toForget(),
      fetchRequest.isFromFollower()) // replicaId >= 0, 即非负id代表fetch请求来自follower

    val erroneous = mutable.ArrayBuffer[(TopicPartition, FetchResponse.PartitionData)]() // 分区 -> 响应
    val interesting = mutable.ArrayBuffer[(TopicPartition, FetchRequest.PartitionData)]() // 分区 -> 请求
    if (fetchRequest.isFromFollower()) { // fetch 请求来自于 follower
      if (authorize(request.session, ClusterAction, Resource.ClusterResource)) {
        // 认证成功，判断请求的每个分区是否存在，若存在则将分区对应的请求加入 interesting 中
        // 否则则构造错误响应加入 erroneous
        fetchContext.foreachPartition((part, data) => {
          if (!metadataCache.contains(part.topic)) {
            erroneous += part -> new FetchResponse.PartitionData(Errors.UNKNOWN_TOPIC_OR_PARTITION,
              FetchResponse.INVALID_HIGHWATERMARK, FetchResponse.INVALID_LAST_STABLE_OFFSET,
              FetchResponse.INVALID_LOG_START_OFFSET, null, MemoryRecords.EMPTY)
          } else {
            interesting += (part -> data)
          }
        })
      } else { // 认证失败，对所有分区都构造错误响应加入 erroneous
        fetchContext.foreachPartition((part, data) => {
          erroneous += part -> new FetchResponse.PartitionData(Errors.TOPIC_AUTHORIZATION_FAILED,
            FetchResponse.INVALID_HIGHWATERMARK, FetchResponse.INVALID_LAST_STABLE_OFFSET,
            FetchResponse.INVALID_LOG_START_OFFSET, null, MemoryRecords.EMPTY)
        })
      }
    } else { // fetch 请求来自于客户端（消费者），和之前处理一样，认证失败或者分区不存在则构造错误响应
      fetchContext.foreachPartition((part, data) => {
        if (!authorize(request.session, Read, new Resource(Topic, part.topic)))
          erroneous += part -> new FetchResponse.PartitionData(Errors.TOPIC_AUTHORIZATION_FAILED,
            FetchResponse.INVALID_HIGHWATERMARK, FetchResponse.INVALID_LAST_STABLE_OFFSET,
            FetchResponse.INVALID_LOG_START_OFFSET, null, MemoryRecords.EMPTY)
        else if (!metadataCache.contains(part.topic))
          erroneous += part -> new FetchResponse.PartitionData(Errors.UNKNOWN_TOPIC_OR_PARTITION,
            FetchResponse.INVALID_HIGHWATERMARK, FetchResponse.INVALID_LAST_STABLE_OFFSET,
            FetchResponse.INVALID_LOG_START_OFFSET, null, MemoryRecords.EMPTY)
        else
          interesting += (part -> data)
      })
    }
```

可见主要是调用`authorize`方法进行 ACL 认证，以及查询`metadataCache`判断请求的分区是否存在。对于 follower，认证是基于整个请求的，操作是`ClusterAction`；对于 consumer，认证是基于每个分区的，类型是`Read`。

只有经过认证且存在于`metadataCache`的分区对应的请求会加入`interesting`中，其它分区会构造一个默认的不合法响应加入`erroneous`中。

接下来定义了如下回调函数：

```scala
def convertedPartitionData(tp: TopicPartition, data: FetchResponse.PartitionData): FetchResponse.PartitionData

def processResponseCallback(responsePartitionData: Seq[(TopicPartition, FetchPartitionData)])
```

然后调用`ReplicaManager.fetchMessages`方法对 `interesting` 请求进行处理：

```scala
    if (interesting.isEmpty)
      processResponseCallback(Seq.empty)
    else {
      replicaManager.fetchMessages(
        fetchRequest.maxWait.toLong, // 最大等待时间，毫秒
        fetchRequest.replicaId, // 副本 id，客户端为 Consumer 则为 -1
        fetchRequest.minBytes, // 响应中积攒的最小字节数
        fetchRequest.maxBytes, // 响应中积攒的最大字节数
        versionId <= 2, // maxBytes 字段是从 V3 才引入的，因此判断 API 版本以兼容旧版本请求
        interesting, // 通过认证且分区存在的请求
        replicationQuota(fetchRequest),
        processResponseCallback, // 处理响应的回调
        fetchRequest.isolationLevel)
```

## ReplicaManager.fetchMessages

### 主要实现

方法说明：从 leader 副本取得消息，等待足够数据可以获取。一旦超时或者请求条件被满足则回调被调用。

```scala
  def fetchMessages(timeout: Long,
                    replicaId: Int,
                    fetchMinBytes: Int,
                    fetchMaxBytes: Int,
                    hardMaxBytesLimit: Boolean,
                    fetchInfos: Seq[(TopicPartition, PartitionData)],
                    quota: ReplicaQuota = UnboundedQuota,
                    responseCallback: Seq[(TopicPartition, FetchPartitionData)] => Unit,
                    isolationLevel: IsolationLevel) {
    val isFromFollower = Request.isValidBrokerId(replicaId) // replicaId >= 0 (Follower) 则为 true
    // replica id 不为 -2 (debugging) 和 -3 (future local) 则为 true, 即正常 Fetch 请求都只从 leader 获取
    val fetchOnlyFromLeader = replicaId != Request.DebuggingConsumerId && replicaId != Request.FutureLocalReplicaId
    // replica id 为 -1 (Consumer) 且不为 -3 (future local) 则为 true, 即 Consumer 仅获取已提交的 offsets
    val fetchOnlyCommitted = !isFromFollower && replicaId != Request.FutureLocalReplicaId

    def readFromLog(): Seq[(TopicPartition, LogReadResult)] = { /* ... */ }

    // 从本地消息日志读取结果
    val logReadResults = readFromLog() // Seq[(TopicPartition, LogReadResult)]

    // 所有分区的 LogReadResult 组成的 Seq
    val logReadResultValues = logReadResults.map { case (_, v) => v }
    // 总共读取的字节数
    val bytesReadable = logReadResultValues.map(_.info.records.sizeInBytes).sum
    // 如果存在 LogReadResult 的 error 字段不为 NONE 则为 true, 即存在读取错误
    val errorReadingData = logReadResultValues.foldLeft(false) ((errorIncurred, readResult) =>
      errorIncurred || (readResult.error != Errors.NONE))

    if (timeout <= 0 || fetchInfos.isEmpty || bytesReadable >= fetchMinBytes || errorReadingData) {
      // 请求不想等待 or 请求消息为空 or 读取的总字节数超过了最小积攒字节数 or 存在读取错误
      // 此时直接生成结果给回调函数处理
      val fetchPartitionData = logReadResults.map { case (tp, result) =>
        tp -> FetchPartitionData(result.error, result.highWatermark, result.leaderLogStartOffset, result.info.records,
          result.lastStableOffset, result.info.abortedTransactions)
      }
      responseCallback(fetchPartitionData)
    } else {
      // Map 类型, key 为 TopicPartition, value 为 FetchPartitionStatus
      val fetchPartitionStatus = logReadResults.map { case (topicPartition, result) =>
        // 对每个 LogReadResult, 从 fetchInfos 中找到第一个分区相同的 PartitionData, 若找不到分区, 则抛出 RuntimeException
        // PartitionData 包含以下字段：
        //   fetchOffset: Long     要获取的消息 offset
        //   logStartOffset: Long  follower 第一个可用 offset, V5 新增字段
        //   maxBytes: Long        响应中累积的最大字节数, V3 新增字段
        val fetchInfo = fetchInfos.collectFirst {
          case (tp, v) if tp == topicPartition => v
        }.getOrElse(sys.error(s"Partition $topicPartition not found in fetchInfos"))
        // fetchOffsetMetadata: LogOffsetMetadata 来自从本地日志读取的信息
        // fetchInfo: PartitionData 来自客户端的请求字段, 利用 FetchContext 得到的
        (topicPartition, FetchPartitionStatus(result.info.fetchOffsetMetadata, fetchInfo))
      }
      // 转发输入参数构造 DelayedFetch 对象
      val fetchMetadata = FetchMetadata(fetchMinBytes, fetchMaxBytes, hardMaxBytesLimit, fetchOnlyFromLeader,
        fetchOnlyCommitted, isFromFollower, replicaId, fetchPartitionStatus)
      val delayedFetch = new DelayedFetch(timeout, fetchMetadata, this, quota, isolationLevel, responseCallback)

      // 构造 (topic, partition) 键值对作为延迟 fetch 操作的 key
      val delayedFetchKeys = fetchPartitionStatus.map { case (tp, _) => new TopicPartitionOperationKey(tp) }

      // 尝试立刻完成请求, 否则将其放入 purgatory, 因为每次创建延迟 fetch 操作时, 新的请求可能到达并使其可完成
      delayedFetchPurgatory.tryCompleteElseWatch(delayedFetch, delayedFetchKeys)
    }
  }
```

1. 从本地日志文件中读取得到请求的每个分区的结果（`LogReadResult`）；
2. 若出现以下错误，则立刻将读取结果构造成 `FetchPartitionData` 交给回调函数处理；
   - timeout（对应请求的 `max_wait_time`字段）小于0，即客户端不想等待；
   - 读取结果为空，即客户端请求的任何分区都无法从本地读到结果；
   - 读取字节数不小于 `fetchMinBytes`（对应请求的 `min_bytes` 字段）；
   - 在读取某个请求的分区的结果时存在错误。
3. 否则，遍历每个分区的读取结果，和请求中同一分区的请求字段一起构造 `FetchPartitionStatus`；
4. 构造 `DelayedFetch` 对象，尝试完成请求，否则将其放入 `delayedFetchPurgatory` 中延迟处理。

关键的部分就是 `readFromLog()` 函数和延迟处理的部分。延迟处理相关设施（purgatory，`DelayedOperation`）在之后去阅读，本篇最后阅读 `readFromLog()` 和发送响应的回调函数的实现。

### responseCallback

即 `KafkaApis.handleFetchRequest` 方法中定义的回调函数 `processResponseCallback`，用来在处理请求完成，构造响应后将响应发送给客户端。

这部分不细看了，因为有不少逻辑是为了实现事务以及配置限额的，这不是目前我阅读源码的重点。核心处理分为两步：

1. 通过 `convertedPartitionData` 将 `PartitionData` 转换成和兼容旧版本的响应结构；
2. 调用 `KafkaApis.sendResponse` 发送响应，在之前的  [Produce 请求(2): 发送响应](https://bewaremypower.github.io/2019/11/07/Kafka源码阅读07-Produce请求-2-发送响应/)  中都看过这个方法，简单回顾下，实际上就是把响应加入 `Processor` 的响应队列，之后的发送由 `Processor` 处理，参考  [网络层阅读之 Acceptor 和 Processor](https://bewaremypower.github.io/2019/09/20/Kafka源码阅读02-网络层阅读之Acceptor和Processor/) 的 4.2 节。

## readFromLog

```scala
    def readFromLog(): Seq[(TopicPartition, LogReadResult)] = {
      val result = readFromLocalLog(
        replicaId = replicaId, // 副本 id, 客户端为 Consumer 则为 -1
        fetchOnlyFromLeader = fetchOnlyFromLeader,
        readOnlyCommitted = fetchOnlyCommitted,
        fetchMaxBytes = fetchMaxBytes, // max_bytes 字段
        hardMaxBytesLimit = hardMaxBytesLimit, // 请求版本 >= V3 则为 true, 此时请求有 max_bytes 字段
        readPartitionInfo = fetchInfos, // 通过认证且分区存在的分区信息
        quota = quota,
        isolationLevel = isolationLevel)
      if (isFromFollower) updateFollowerLogReadResults(replicaId, result)
      else result
    }
```

调用 `readFromLocalLog`，如果 Fetch 请求来自 follower 则还需要调用 `updateFollowerLogReadResults` 更新 follower 的结果。

### readFromLocalLog

首先看看内部定义的 `read` 函数：

```scala
    def read(tp: TopicPartition, fetchInfo: PartitionData, limitBytes: Int, minOneMessage: Boolean): LogReadResult = {
      val offset = fetchInfo.fetchOffset
      val partitionFetchSize = fetchInfo.maxBytes
      val followerLogStartOffset = fetchInfo.logStartOffset

      try {
        // 决定是否仅从 leader 获取, 然而无论是 Consumer 还是 Follower 都会从 leader 获取
        val localReplica = if (fetchOnlyFromLeader)
          getLeaderReplicaIfLocal(tp) // 分区的 leader 副本
        else
          getReplicaOrException(tp)

        val initialHighWatermark = localReplica.highWatermark.messageOffset
        val lastStableOffset = if (isolationLevel == IsolationLevel.READ_COMMITTED)
          Some(localReplica.lastStableOffset.messageOffset)
        else
          None

        // decide whether to only fetch committed data (i.e. messages below high watermark)
        val maxOffsetOpt = if (readOnlyCommitted)
          Some(lastStableOffset.getOrElse(initialHighWatermark))
        else
          None

        /* 在读取日志之前首先读取 LogOffsetMetadata, 它能判断指定副本是否同步
         * 在读取之后再使用 LEO 会导致 race condition, 比如在副本完成消费后, 数据立刻添加到了日志末尾,
         * 这可能导致副本一直被判断不同步
         */
        val initialLogEndOffset = localReplica.logEndOffset.messageOffset // 在读取操作之前取得 LEO
        val initialLogStartOffset = localReplica.logStartOffset
        val fetchTimeMs = time.milliseconds // 当前时间戳
        val logReadInfo = localReplica.log match {
          case Some(log) =>
            // 取得 partition_max_bytes (分区本身的最大读取字节数) 和 max_bytes 的较小值作为 fetch 字节数上限
            val adjustedFetchSize = math.min(partitionFetchSize, limitBytes)

            // 读取 offset 开始的最多 adjustedFetchSize 个字节, 若 minOneMessage 为 true, 则即使第一条消息大小
            // 超过了 adjustedFetchSize 也会返回这条消息, 返回类型: FetchDataInfo
            val fetch = log.read(offset, adjustedFetchSize, maxOffsetOpt, minOneMessage, isolationLevel)

            // 该分区正在被限速, 即限制访问该分区, 清空消息
            if (shouldLeaderThrottle(quota, tp, replicaId))
              FetchDataInfo(fetch.fetchOffsetMetadata, MemoryRecords.EMPTY)
            // V3 版本开始 hardMaxBytesLimit 为 false, 如果第一条消息大小超过了 max_bytes 限制也会读取
            // 为了防止客户端报错 RecordToolLargeException, 此时将过大的消息替换成空消息
            else if (!hardMaxBytesLimit && fetch.firstEntryIncomplete)
              FetchDataInfo(fetch.fetchOffsetMetadata, MemoryRecords.EMPTY)
            else fetch

          case None => // leader 副本在该分区不存在本地日志
            error(s"Leader for partition $tp does not have a local log")
            FetchDataInfo(LogOffsetMetadata.UnknownOffsetMetadata, MemoryRecords.EMPTY)
        }

        LogReadResult(info = logReadInfo, // localReplica.log 调用 read 方法的返回值
                      // localReplica 在内存中维护的 HW, LogStartOffset, LEO
                      highWatermark = initialHighWatermark,
                      leaderLogStartOffset = initialLogStartOffset,
                      leaderLogEndOffset = initialLogEndOffset,
                      // 请求中 follower 的 LogStartOffset, 客户端为 Consumer 则为 -1
                      followerLogStartOffset = followerLogStartOffset,
                      fetchTimeMs = fetchTimeMs, // 从本地读取数据之前记录的时间戳
                      readSize = partitionFetchSize, // NOTE: 这里是请求的 max_bytes 字段，而非实际读取字节数
                      lastStableOffset = lastStableOffset, // LSO, 用于事务实现
                      exception = None)
      } catch {
        // (...) 异常处理, 返回一个 exception 字段为捕获的异常, 其它字段都不合法的 LogReadResult
      }
    }
```

流程：首先取得本地副本（实际上对 Consumer 和 Follower 而言都是 Leader 副本），然后取得 HW，LEO 等字段，记录时间戳，然后通过本地副本读取本地数据。这里还利用了 V3 版本请求的 `max_bytes` 字段，限制读取的字节数上限，但如果第一条消息长度就超出上限的话，仍然会返回整条消息（此时读取字节数超过了 `max_bytes`）。

注意 `LogReadResult` 的第一个字段是从本地日志读取的信息：

```scala
case class FetchDataInfo(fetchOffsetMetadata: LogOffsetMetadata, // offset 元数据, 包括:
                         // offset; Segment 的基础 offset; 相对于 Segment 的物理偏移字节数
                         records: Records, // 消息集
                         firstEntryIncomplete: Boolean = false,
                         abortedTransactions: Option[List[AbortedTransaction]] = None
```

主要是前两个字段，消息集就不说了，元数据的作用是记录了 offset 对应消息相对本地 Segment 的实际偏移量。这里回顾一个基本概念，Kafka 的每个分区都用本地文件记录消息，为了防止单个文件过大，会根据文件大小和写入时间分成多个文件，单个文件被称为 **Segment**（对应代码中的 `LogSegment` 类），而 `Log` 类则是管理这些 Segment。因此，记录消息的物理偏移量，便于在从本地 Segment 中快速通过 offset 定位到对应消息。

接着看 `readFromLocalLog` 的逻辑：

```scala
    var limitBytes = fetchMaxBytes // 初始值为 max_bytes 字段, 整个响应(多条消息)中可积累的最大字节数
    val result = new mutable.ArrayBuffer[(TopicPartition, LogReadResult)]
    var minOneMessage = !hardMaxBytesLimit
    readPartitionInfo.foreach { case (tp, fetchInfo) =>
      val readResult = read(tp, fetchInfo, limitBytes, minOneMessage) // 指定分区的读取结果
      val recordBatchSize = readResult.info.records.sizeInBytes // 实际读取字节数
      // 读取了至少一条消息, 那么以后严格遵守 max_bytes 的限制
      if (recordBatchSize > 0)
        minOneMessage = false
      limitBytes = math.max(0, limitBytes - recordBatchSize)
      result += (tp -> readResult)
    }
    result
```

可见，每个分区都对应一条读取结果（`LogReadResult`），包含 offset 对应消息，还有 HW/LEO 等信息 。V3 开始外部的 `max_bytes` 字段限制所有消息的最大字节数，而每个分区都有自己的 `partition_max_bytes` 限制单条消息的最大字节数。

读完这部分代码后，可以回顾 Fetch 请求的协议（V3 版本），并附上注释说明：

```
Fetch Request (Version: 3) => replica_id max_wait_time min_bytes max_bytes [topics] 
  replica_id => INT32 // -1: Consumer, >= 0: Follower
  max_wait_time => INT32 // 延迟请求中的 timeout，用于构造 DelayedFetch 对象
  min_bytes => INT32 // 响应字节数超过则立刻发送响应，见ReplicaManager.fetchMessages
  max_bytes => INT32 // 整个响应的最大字节数
  topics => topic [partitions] 
    topic => STRING
    partitions => partition fetch_offset partition_max_bytes 
      partition => INT32
      fetch_offset => INT64
      partition_max_bytes => INT32 // 每个分区消息的最大字节数
```

其中 fetch_offset 可由 `FetchContext` 的相关方法取得：

```scala
trait FetchContext extends Logging {
  /**
    * Get the fetch offset for a given partition.
    */
  def getFetchOffset(part: TopicPartition): Option[Long]
```

### updateFollowerLogReadResults

当 replica id 大于 0 时，代表客户端为 Follower，在从本地日志读取信息后，会调用该方法更新 Follower 的 fetch 状态，并更新读取结果。

```scala
  private def updateFollowerLogReadResults(replicaId: Int,
                                           readResults: Seq[(TopicPartition, LogReadResult)]): Seq[(TopicPartition, LogReadResult)] = {
    readResults.map { case (topicPartition, readResult) =>
      var updatedReadResult = readResult
      nonOfflinePartition(topicPartition) match {
        case Some(partition) =>
          partition.getReplica(replicaId) match {
            case Some(replica) =>
            case Some(replica) =>
              // 首先更新分区上的 follower 的状态, 若 LW 或 HW 增加则返回 true
              if (partition.updateReplicaLogReadResult(replica, readResult))
                // 将 leader 副本的信息 (HW, LogStartOffset, LEO) 更新到读取结果上
                partition.leaderReplicaIfLocal.foreach { leaderReplica =>
                  updatedReadResult = readResult.updateLeaderReplicaInfo(leaderReplica)
                }
            case None => // 当前副本不是分区的副本, 清空读取结果的 records 字段并将 metadata 标记为未知
              // 略去日志...
              updatedReadResult = readResult.withEmptyFetchInfo
          }
        case None => // 分区不可用（即 offline 分区）, 打印警告日志, 不修改读取结果
          warn(s"While recording the replica LEO, the partition $topicPartition hasn't been created.")
      }
      topicPartition -> updatedReadResult
    }
  }
```

然后对读取结果调用 `updateLeaderReplicaInfo` 更新为 leader 副本的信息：

```scala
  def updateLeaderReplicaInfo(leaderReplica: Replica): LogReadResult =
    copy(highWatermark = leaderReplica.highWatermark.messageOffset,
      leaderLogStartOffset = leaderReplica.logStartOffset,
      leaderLogEndOffset = leaderReplica.logEndOffset.messageOffset)
```

利用 Scala case 类的 `copy` 方法，返回更新对应字段后的对象。这里将读取结果的 HW，LogStartOffset，LEO 更新为 leader 副本维护的相应信息。因为 follower 副本发送 Fetch 请求时，leader 副本可能更新 HW（如果之前 follower 没有同步到最新），因此需要把更新后的 HW 发送给 follower。

顺带提下这里涉及到的另一个概念：低水位（LW, Low Watermark）。LW 即所有副本中最小的 LogStartOffset，一般情况下 LW 都是 0，但是如果服务端收到了 `DeleteRecords` 请求，删除日志文件的部分记录（消息）时，会更新 LW。

## 总结

本篇阅读了 Fetch 请求的处理流程，主要根据 replica id 字段分 Consumer 和 Follower 来处理：

1. 会话认证，判断请求分区是否存在，将没有问题的分区对应的请求构成 Map 由 `ReplicaManager` 处理；
2. `ReplicaManager` 对每个分区，找到其 leader 副本；
3. leader 副本从本地读取请求的 offset 开始的若干消息（由全局的以及各分区的 `max_bytes` 字段来限制读取最大字节数），和维护的其它信息构成读取结果；
4. 对 follower 副本的请求，还会将 leader 副本的 HW，LEO，LogStartOffset 更新到读取结果中；
5. 根据读取结果和请求的相关字段判断是否立刻发送响应，比如读取没问题时，读取字节数超过了 `min_bytes` 即可发送；
6. 否则，构造 `DelayedFetch` 对象传入 `DelayedFetchPurgatory` 对象中，此时 purgatory 还会判断一次处理是否完成，若已完成则不用延迟处理。

主要区别还是第 4 步，因为 follower 的 Fetch 请求是用来与 leader 同步的，因此需要将 HW 记录在结果中让 follower 更新自己的 HW。