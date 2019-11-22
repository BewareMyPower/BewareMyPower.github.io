---
title: 'Kafka源码阅读08: 写入本地日志的具体实现'
date: 2019-11-22 11:36:37
tags:
  - Kafka
top: 1
---
## 回顾

在[06: Produce请求之写入本地日志](https://bewaremypower.github.io/2019/11/06/Kafka源码阅读06-Produce请求-1-写入本地日志/)中，对`ReplicaManager`类的`appendToLocalLog`方法的阅读，主要集中在对异常场景的处理：

- 非admin客户端写入`__consumer_offsets`等特殊主题；
- 找不到请求的主题+分区；
- 请求的是离线分区；
- 当前broker不是请求分区的leader；
- 请求的`acks`字段不合法，或者为-1（all）但ISR数量小于`min.insync.replicas`配置。

会抛出异常被捕获后生成`LogAppendResult`对象（见server/ReplicaManager.scala）

```scala
case class LogAppendResult(info: LogAppendInfo, exception: Option[Throwable] = None) {
  def error: Errors = exception match {
    case None => Errors.NONE
    case Some(e) => Errors.forException(e)
  }
}
```

对上述异常场景，`LogAppendResult.info`被置为无效值：

```scala
object LogAppendInfo {
  val UnknownLogAppendInfo = LogAppendInfo(-1, -1, RecordBatch.NO_TIMESTAMP, -1L, RecordBatch.NO_TIMESTAMP, -1L,
    RecordsProcessingStats.EMPTY, NoCompressionCodec, NoCompressionCodec, -1, -1, offsetsMonotonic = false)
}
```

`appendToLocalLog`返回的`LogAppendResult`在 [07: Produce请求之发送响应](https://bewaremypower.github.io/2019/11/07/Kafka源码阅读07-Produce请求-2-发送响应/)  中会用来生成`PartitionResponse`对象和对应主题分区构成`Map`传给发送响应给客户端的回调函数中。

也就是说，最关键的部分我们之前暂且跳过了，也就是在正常清空下如何写入本地日志文件，然后生成`LogAppendInfo`。

## Log.append代码分析

在`cluster`包的`Partition.scala`中，将当前分区的`leaderEpoch`字段传入了`appendAsLeader`。

```scala
val info = log.appendAsLeader(records, leaderEpoch = this.leaderEpoch, isFromClient)
```

`log`为`Log`对象，位于`log`包下的`Log.scala`。该方法会调用`append`：

```scala
  def appendAsLeader(records: MemoryRecords, leaderEpoch: Int, isFromClient: Boolean = true): LogAppendInfo = {
    append(records, isFromClient, assignOffsets = true, leaderEpoch)
  }
```

这里只考虑来自客户端的请求，因此接下来阅读时**默认`isFromClient`和`assignOffsets`为true**。

```scala
  private def append(records: MemoryRecords, isFromClient: Boolean, assignOffsets: Boolean, leaderEpoch: Int): LogAppendInfo = {
    maybeHandleIOException(s"Error while appending records to $topicPartition in dir ${dir.getParent}") {
      // ...
    }
  }
```

```scala
  private def maybeHandleIOException[T](msg: => String)(fun: => T): T = {
    try {
      fun
    } catch {
      case e: IOException =>
        logDirFailureChannel.maybeAddOfflineLogDir(dir.getParent, msg, e)
        throw new KafkaStorageException(msg, e)
    }
  }
```

`maybeHandleIOException`捕获`fun`可能抛出的`IOException`，进一步抛出`KafkaStorageException`会被上层捕获生成`LogAppendResult`。

```scala
      // 校验消息的CRC以及消息长度(字节数)是否合法(不超过配置的 max.message.bytes), 并且会设置以下字段:
      // - firstOffset: 第1条消息的offset, V2版本可以从header的firstOffset字段直接取得
      // - lastOffset: 最后1条消息的offset, V2版本可以从header的firstOffset + lastOffsetDelta得到
      // - shallowCount: 消息集的数量，shallow即浅层，以消息集为单位
      // - validBytes: 所有长度合法的消息的长度之和
      // - offsetsMonotic: 消息offset是否单调递增，利用每个消息集的lastOffset判断
      // - sourceCodec: 生产者消息集的编码方式
      val appendInfo = analyzeAndValidateRecords(records, isFromClient = isFromClient)

      if (appendInfo.shallowCount == 0) // 没有合法消息则直接返回
        return appendInfo

      // 截断records中不合法的字节数, 然而按照前面的逻辑, 如果 analyzeAndValidateRecords 不抛出异常,
      // appendInfo.validBytes 和 records.sizeInBytes 是相等的, 猜测是遗留方法?
      var validRecords = trimInvalidBytes(records, appendInfo)

      // 将 validRecords 插入到日志中, 由于可能其他处理线程也在将消息写入本地文件, 所以用锁保护
      lock synchronized {
        // 检查内存映射的缓冲区是否关闭, 比如在 closeHandlers() 会导致其关闭
        // 若关闭, 则表示无法写入, 抛出 KafkaStorageException
        checkIfMemoryMappedBufferClosed()
        if (assignOffsets) {
          // assign offsets to the message set
          // 生产者发送的消息集的offset为0,1,...,n, nextOffsetMetadata记录了本地日志
          // 下一条消息的offset, 将其置为新的firstOffset, 也就是绝对offset
          val offset = new LongRef(nextOffsetMetadata.messageOffset)
          appendInfo.firstOffset = offset.value
          val now = time.milliseconds // 当前时间戳, 即LogAppendTime类型的时间戳
          // 重新验证/解压/压缩得到更新内部offset后的validRecords
          val validateAndOffsetAssignResult = try {
            // 更新消息集的offset, 对于V1版本以上的消息, 可能因为时间戳类型字段来覆盖时间戳
            LogValidator.validateMessagesAndAssignOffsets(validRecords,
              offset, // 会更新为最后1条消息的绝对offset+1, 即下一次写入本地日志的消息的offset
              time,
              now,
              appendInfo.sourceCodec,
              appendInfo.targetCodec,
              config.compact,
              config.messageFormatVersion.messageFormatVersion.value,
              config.messageTimestampType,
              config.messageTimestampDifferenceMaxMs,
              leaderEpoch,
              isFromClient)
          } catch {
            case e: IOException => throw new KafkaException("Error in validating messages while appending to log '%s'".format(name), e)
          }
          validRecords = validateAndOffsetAssignResult.validatedRecords
          // 设置 appendInfo 的以下字段：
          // - maxTimestamp: 消息集的最大时间戳
          // - offsetOfMaxTimestamp: 最大时间戳对应消息的绝对offset
          // - lastOffset: 最后1条消息的offset
          // - logAppendTime: 如果时间戳类型为LOG_APPEND_TIME, 则设为刚刚获取的时间戳
          appendInfo.maxTimestamp = validateAndOffsetAssignResult.maxTimestamp
          appendInfo.offsetOfMaxTimestamp = validateAndOffsetAssignResult.shallowOffsetOfMaxTimestamp
          appendInfo.lastOffset = offset.value - 1
          appendInfo.recordsProcessingStats = validateAndOffsetAssignResult.recordsProcessingStats
          if (config.messageTimestampType == TimestampType.LOG_APPEND_TIME)
            appendInfo.logAppendTime = now

          // 重新验证消息大小是否超过max.message.size, 因为修改字段后重新压缩可能导致消息大小改变
          if (validateAndOffsetAssignResult.messageSizeMaybeChanged) {
            for (batch <- validRecords.batches.asScala) {
              if (batch.sizeInBytes > config.maxMessageSize) {
                // 更新stats(略)
                throw new RecordTooLargeException("Message batch size is %d bytes which exceeds the maximum configured size of %d."
                  .format(batch.sizeInBytes, config.maxMessageSize))
              }
            }
          }
        } else {
          // assignOffsets为false的处理(略)
        }

        // TODO: 对V2以上版本的消息集, 更新 leader epoch cache
        validRecords.batches.asScala.foreach { batch =>
          if (batch.magic >= RecordBatch.MAGIC_VALUE_V2)
            _leaderEpochCache.assign(batch.partitionLeaderEpoch, batch.baseOffset)
        }

        // 检查消息集的总大小是否超过配置的segment.bytes, 即每个日志文件的大小
        if (validRecords.sizeInBytes > config.segmentSize) {
          throw new RecordBatchTooLargeException("Message batch size is %d bytes which exceeds the maximum configured segment size of %d."
            .format(validRecords.sizeInBytes, config.segmentSize))
        }

        // now that we have valid records, offsets assigned, and timestamps updated, we need to
        // validate the idempotent/transactional state of the producers and collect some metadata
        // TODO: 验证生产者的 幂等性/事务 状态, 并收集一些元数据
        val (updatedProducers, completedTxns, maybeDuplicate) = analyzeAndValidateProducerState(validRecords, isFromClient)
        maybeDuplicate.foreach { duplicate =>
          appendInfo.firstOffset = duplicate.firstOffset
          appendInfo.lastOffset = duplicate.lastOffset
          appendInfo.logAppendTime = duplicate.timestamp
          appendInfo.logStartOffset = logStartOffset
          return appendInfo
        }

        // 如果必要, 执行日志回滚策略, 将当前日志文件备份, 并创建空文件作为当前日志文件
        val segment = maybeRoll(messagesSize = validRecords.sizeInBytes,
          maxTimestampInMessages = appendInfo.maxTimestamp,
          maxOffsetInMessages = appendInfo.lastOffset)

        val logOffsetMetadata = LogOffsetMetadata(
          messageOffset = appendInfo.firstOffset,
          segmentBaseOffset = segment.baseOffset,
          relativePositionInSegment = segment.size)

        segment.append(firstOffset = appendInfo.firstOffset,
          largestOffset = appendInfo.lastOffset,
          largestTimestamp = appendInfo.maxTimestamp,
          shallowOffsetOfMaxTimestamp = appendInfo.offsetOfMaxTimestamp,
          records = validRecords)

        // 更新生产者状态
        for ((producerId, producerAppendInfo) <- updatedProducers) {
          producerAppendInfo.maybeCacheTxnFirstOffsetMetadata(logOffsetMetadata)
          producerStateManager.update(producerAppendInfo)
        }

        // update the transaction index with the true last stable offset. The last offset visible
        // to consumers using READ_COMMITTED will be limited by this value and the high watermark.
        // TODO: 用最新的稳定offset更新事务
        for (completedTxn <- completedTxns) {
          val lastStableOffset = producerStateManager.completeTxn(completedTxn)
          segment.updateTxnIndex(completedTxn, lastStableOffset)
        }

        producerStateManager.updateMapEndOffset(appendInfo.lastOffset + 1)

        // 更新 nextOffsetMetadata 为 lastOffset+1, 回顾之前if (assignOffsets)分支
        // 在下一批消息到达时, 该offset即新消息集的第1个消息的绝对offset
        updateLogEndOffset(appendInfo.lastOffset + 1)

        // TODO: update the first unstable offset (which is used to compute LSO)
        updateFirstUnstableOffset()
        
        // trace日志(略)

        // 若未冲刷的消息数量(利用LEO减去recoverPoint得到)达到了配置的"flush.messages"则冲刷消息
        if (unflushedMessages >= config.flushInterval)
          flush()

        appendInfo
      }
    }
```

注释中标出TODO的部分暂时还不了解原理，包括且不限于：

- leader epoch；
- 对事务的支持；
- stable offset（似乎也是用于事务？）

## 流程总结

首先是代码逻辑的大体流程：

1. `records.batches`为一组Record Batch，对每个batch都校验CRC是否合法，字节数是否超过配置`max.message.bytes`；
2. 若存在不合法的batch，则会抛出异常最终被`ReplicaManager.appendToLocalLog`捕获（仅限于Produce请求处理的情况），生成包含错误的响应；
3. 利用records计算出第1条消息和最后1条消息的offset，消息集的数量，合法batch的字节数之和，消息offset是否单调递增，以及消息集的编码方式，构造要返回的`LogAppendInfo`对象，记为`info`；
4. 验证合法消息的数量，并截断不合法的字节数，得到`validRecords`；（**TODO：此处实现似乎不合理，因为存在不合法的batch直接就抛异常了，但当前最新版本2.3的Kafka源码也是这么处理的**）
5. 检测内存映射缓存是否被关闭；
6. 将LEO赋给`info.firstOffset`，并取得当前时间戳`now`；
7. 更新`validRecords`的offset为绝对offset，若batch是压缩的则重新压缩，将最后1条消息的offset赋给`info.lastOffset`，并设置`info`的消息集最大时间戳及对应消息的offset；
8. 若时间戳类型为`LOG_APPEND_TIME`，将`now`赋给`info.logAppendTime`（默认为-1）；
9. 若重新压缩的`validRecords`字节数发生变化，重新检查每个`batch`的字节数是否超过配置`max.message.bytes`；
10. 检查`validRecords`字节数是否超过配置`log.segments.bytes`；
11. 取得当前的`LogSegment`对象，将`validRecords`添加进去；
12. 更新LEO为`validRecords`最后1条消息的offset+1；
13. 若未冲刷的消息数量超过了配置`flush.messages`，则将所有`LogSegments`写入本地磁盘。

核心还是用绝对offset替换相对offset。生产者向服务端发送请求时，由于不知道消息集落盘时的offset，所以只能设置offsets为0,1,2,...n-1，也就是相对offset。而分区的leader broker则维护了其LEO，因此收到请求时，会将offsets修改为LEO,LEO+1,LEO+2,...LEO+n-1，最后将LEO更新为LEO+n。而更新的offsets会包含在响应里发送给生产者，这样客户端就可以通过消息送达的回调函数得到发送消息的绝对offset。

每个`Log`对应1个分区的消息日志，而消息日志是分为多个文件（日志片段，Log Segment）对应`LogSegment`对象，负责实际写入磁盘。

这里回顾用到的3个Kafka服务端配置：

- `max.message.bytes`：每个消息集的最大字节数（这是0.11开始的含义，见[upgrade 0.11 message format](https://kafka.apache.org/11/documentation.html#upgrade_11_message_format)；
- `log.segment.bytes`：Log Segment的最大字节数（所以需要检测消息集字节数是否超过这个值，否则即使新建文件写入消息集，也无法容纳整个消息集）；
- `flush.messages`（Topic级别）：允许`LogSegment`对象缓存的消息数量，如果缓存消息数超过了该配置就会调用`fsync`写入磁盘。

此外，**Record Batch**即**消息集（Message Set）**，Record（记录）即Message（消息）。之所以这里区分，是因为从Kafka 0.11版本开始，消息集的概念发生了变化。在此之前，消息集仅仅是一组消息之前加上Log Overhead（即offset和message size）。而Kafka 0.11版本新增了，V2版本的消息集，增加了独有的header，比如第1条消息和最后1条消息的offset可直接通过header计算得到，还有些其他字段是leader epoch以及实现事务相关的字段。而每条消息（记录）的key和value用varint而非固定4字节的int表示长度，并且消息本身也有header。

具体参考：https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol#AGuideToTheKafkaProtocol-Messagesets 