---
title: 'Kafka源码阅读07: Produce请求(2): 发送响应'
date: 2019-11-07 17:22:33
tags:
  - Kafka
top: 1
---
## 前文回顾

前一篇阅读了`appendToLocalLog`的部分，服务端在收到Produce请求时，会首先将消息写入本地消息日志：

```scala
      val sTime = time.milliseconds // 取得当前毫秒级时间戳
      val localProduceResults = appendToLocalLog(internalTopicsAllowed = internalTopicsAllowed,
        isFromClient = isFromClient, entriesPerPartition, requiredAcks)
      // 调试信息: 再次取得时间戳, 相减得到 appendToLocalLog 的用时
      debug("Produce to local log in %d ms".format(time.milliseconds - sTime))
```

返回的`localProduceResults`类型是`Map[TopicPartition, LogAppendResult]`。

```scala
// 正常结果: info有效，exception为None
// 错误结果: info各字段设为无效值，exception为某种异常，可通过error方法取得其错误信息
case class LogAppendResult(info: LogAppendInfo, exception: Option[Throwable] = None) {
  def error: Errors = exception match {
    case None => Errors.NONE
    case Some(e) => Errors.forException(e)
  }
}
```

`info`字段来自于`Log.appendAsLeader`的返回值，即实际添加到本地日志的消息，包含消息集的第1条消息和最后1条消息的offset（生产者在发送消息集时是不知道最后写入日志文件时消息的offset，只有在服务端写入日志时才会设置）。

接下来阅读`ReplicaManager.appendRecords`中的后续处理。

## ProducePartitionStatus的处理

```scala
      // 将分区对应的处理结果转换成 ProducePartitionStatus 对象
      val produceStatus = localProduceResults.map { case (topicPartition, result) =>
        topicPartition ->
                ProducePartitionStatus(
                  // lastOffset + 1 代表下一批消息的第1条消息的 offset
                  result.info.lastOffset + 1, // required offset
                  // 利用 LogAppendInfo 的各字段构造 PartitionResponse
                  new PartitionResponse(result.error, result.info.firstOffset, result.info.logAppendTime, result.info.logStartOffset)) // response status
      }

      // 通过 KafkaApis.handleProduceRequest 传入的回调更新 KafkaApis.brokerTopicStats
      processingStatsCallback(localProduceResults.mapValues(_.info.recordsProcessingStats))
```

```scala
    public static final class PartitionResponse {
        public Errors error; // 错误信息
        public long baseOffset; // 消息集中第1条消息的offset
        public long logAppendTime; // 消息集写入日志文件时的时间戳
        public long logStartOffset; // 消息集写入日志文件时，日志文件的起始offset
        // ...
    }
```

回顾一下，在使用Kafka客户端时，生产者可以通过回调取得消息的元数据，像主题和分区，是在生产者发送前就已知的，但offset和时间戳则是由服务端在此处设置的。见[Kafka 1.1 Producer API](https://kafka.apache.org/11/javadoc/index.html?org/apache/kafka/clients/producer/KafkaProducer.html)的`RecordMetadata`。

接下来是一个if-else分支

```scala
      if (delayedProduceRequestRequired(requiredAcks, entriesPerPartition, localProduceResults)) {
        // ...
      } else {
        // we can respond immediately
        // 取得 PartitionStatus 作为新的value传进 responseCallback, 即忽略了 offset 字段
        val produceResponseStatus = produceStatus.mapValues(status => status.responseStatus)
        responseCallback(produceResponseStatus)
      }
```

如果`delayedProduceRequestRequired`返回`false`，则可以立刻发送响应，而且忽略了offset字段，因为该字段代表了下一批消息的第1个offset，而`PartitionStatus`本身就包含当前消息集的`baseOffset`。

那么为何else分支就意味着可以立刻发送响应呢？

```scala
  private def delayedProduceRequestRequired(requiredAcks: Short,
                                            entriesPerPartition: Map[TopicPartition, MemoryRecords],
                                            localProduceResults: Map[TopicPartition, LogAppendResult]): Boolean = {
    requiredAcks == -1 &&
    entriesPerPartition.nonEmpty &&
    // exception字段为Option类型，若不为None则isDefined返回true
    localProduceResults.values.count(_.exception.isDefined) < entriesPerPartition.size
  }
```

可见，if分支意味着以下条件满足：

1. `requiredAcks`为-1，即生产者要等待分区的所有ISR收到消息后才会返回；
2. `entriesPerPartition`不为空，即存在需要添加消息的分区；
3. `localProduceResults`中至少存在1条成功的结果。

相应地，else分支对应的是：

1. `requiredAcks`为0或1，即客户端无需等待服务端的响应或者只需要等待leader收到消息；
2. 没有消息需要写入（无论是没有可写入的分区还是全部消息写入出现异常），那么ISR也没必要去从leader复制数据，因此也可以直接返回响应。

PS：第2个条件在处理Produce请求是是多余的判断，因为之前在`KafkaApis.handleProduceRequest`中已经判断过了：

```scala
    if (authorizedRequestInfo.isEmpty)
      sendResponseCallback(Map.empty)
    else { // authorizedRequestInfo非空, 传入参数entriesPerPartition
      // ...
      replicaManager.appendRecords(
        // ...
        entriesPerPartition = authorizedRequestInfo,
        /* ... */)
      // ...
    }
```

也就是说if分支里会等待所有ISR收到消息才会返回，查看if分支：

```scala
        // 构造 DelayedProduce 对象, 注意 timeout 仅在此处使用
        val produceMetadata = ProduceMetadata(requiredAcks, produceStatus)
        val delayedProduce = new DelayedProduce(timeout, produceMetadata, this, responseCallback, delayedProduceLock)

        // 通过 topic 和 partition 创建用于延迟生产操作的key
        val producerRequestKeys = entriesPerPartition.keys.map(new TopicPartitionOperationKey(_)).toSeq

        // 尝试立刻完成请求, 否则会将请求放入 purgatory 中, 因为在创建 DelayedProduce 对象时,
        // 新的请求可能会到达, 从而使得这个操作处于可完成状态
        delayedProducePurgatory.tryCompleteElseWatch(delayedProduce, producerRequestKeys)
```

还是利用了purgatory，先不研究其实现细节，大致可以理解为，创建一个`DelayedProduce`对象，传入带offset和时间戳的消息集，设置timeout和响应回调，就能完成延迟生产。而purgatory只是用来确认是否完成，若没完成则将其扔进purgatory中。

也就是说，响应回调不再是像else分支（以及之前的错误处理分支）中一样由当前线程去调用，而是由`DelayedProduce`对象去调用，从而实现了异步的方式等待所有ISR收到最新的消息，避免leader的`Handler`线程阻塞在`KafkaApis`对请求的处理中。

另外，值得注意的是`timeout`是在构造这个`DelayedProduce`对象时才使用，也就是之前的写入本地日志的时间是不计算在内的，当然网络传输时间也是，可以回顾[上一篇阅读笔记](https://bewaremypower.github.io/2019/11/06/Kafka源码阅读06-Produce请求-1-写入本地日志/) 的**2.1 请求格式**中翻译的官网对`timeout`的说明。

## sendResponseCallback

```scala
    def sendResponseCallback(responseStatus: Map[TopicPartition, PartitionResponse]) {

      // 合并 responseStatus 和之前认证失败或者主题不存在产生的错误响应
      val mergedResponseStatus = responseStatus ++ unauthorizedTopicResponses ++ nonExistingTopicResponses
      var errorInResponse = false

      // 检查是否有错误响应, 若有则将 errorInResponse 置为 true, 并将错误写入debug日志
      mergedResponseStatus.foreach { case (topicPartition, status) =>
        if (status.error != Errors.NONE) {
          errorInResponse = true
          // 写入debug日志(略)
        }
      }

      def produceResponseCallback(bandwidthThrottleTimeMs: Int) {
        if (produceRequest.acks == 0) {
          // acks为0意味着客户端无需等待服务端响应, 因此服务端无需操作
          // 但是如果请求存在错误, 服务端需要关闭socket来通知客户端有错误发生, 然后更新元数据
          if (errorInResponse) { // 存在错误响应
            // 首先转换成 Map[TopicPartition, String], 其value为异常的类型名称, 然后将其字符串表示写入日志
            val exceptionsSummary = mergedResponseStatus.map { case (topicPartition, status) =>
              topicPartition -> status.error.exceptionName
            }.mkString(", ")
            // 写入info日志(略)
            closeConnection(request, new ProduceResponse(mergedResponseStatus.asJava).errorCounts)
          } else { // 不存在错误响应
            sendNoOpResponseExemptThrottle(request)
          }
        } else { // acks为1或者-1
          sendResponseMaybeThrottle(request, requestThrottleMs =>
            new ProduceResponse(mergedResponseStatus.asJava, bandwidthThrottleTimeMs + requestThrottleMs))
        }
      }

      // When this callback is triggered, the remote API call has completed
      // 无论是在哪个处理分支, 这个回调函数必定是在远程API调用结束后才触发
      request.apiRemoteCompleteTimeNanos = time.nanoseconds

      quotas.produce.maybeRecordAndThrottle(
        request.session.sanitizedUser, // session认证用户名(没配置SSL认证则是ANONYMOUS)
        request.header.clientId,
        numBytesAppended,
        produceResponseCallback)
    }
```

由于是接着之前的进行阅读，所以用到了一些之前创建的对象，见[上一篇阅读笔记](https://bewaremypower.github.io/2019/11/06/Kafka源码阅读06-Produce请求-1-写入本地日志/)的**handleProduceRequest**：

- `unauthorizedTopicResponses`：对调用`KafkaApis.authorize`方法认证失败的请求生成的错误响应；
- `nonExistingTopicResponses`：对目标主题不在`KafkaApis.metadataCache`中的请求生产的错误响应；
- `numBytesAppended`：请求的总字节数，包含header部分。

检测出是否有错误响应是为了传给`produceResponseCallback`，从而在acks为0时，关闭与客户端的socket连接来通知其更新元数据。而该回调被传入了`ClientQuotaManager.maybeRecordAndThrottle`方法，在未启用quotas的情况下会直接调用`produceResponseCallback`，分为以下3种情况：

1. acks为0，且存在错误响应：关闭与客户端的连接，会引起客户端更新元数据；

2. acks为0，且不存在错误响应：

   ```scala
   sendNoOpResponseExemptThrottle(request)
   ```

   ```scala
     private def sendNoOpResponseExemptThrottle(request: RequestChannel.Request): Unit = {
       quotas.request.maybeRecordExempt(request)
       sendResponse(request, None)
     }
   ```

   会进入`KafkaApis.sendResponse`的`None`分支：

   ```scala
   requestChannel.sendResponse(new RequestChannel.Response(request, None, NoOpAction, None))
   ```

   回顾[网络层阅读的之Acceptor和Processor](https://bewaremypower.github.io/2019/09/20/Kafka源码阅读02-网络层阅读之Acceptor和Processor/)的**4.2 processNewResponses**，如果响应的类型是`NoOpAction`，只会给`Processor`与客户端的连接`Channel`重新注册读事件，并不会发送响应给客户端。

3. acks不为0：

   ```scala
   sendResponseMaybeThrottle(request,
     requestThrottleMs => new ProduceResponse(mergedResponseStatus.asJava, bandwidthThrottleTimeMs + requestThrottleMs))
   ```

   创建`ProduceResponse`的`throttleMs`为`bandwidthThrottleTimeMs`和`requestThrottleMs`之和，这两者都有各自对应的quotas对象，若未启用则为0。最终也会进入`KafkaApis.sendResponse`中：

   ```scala
   val responseSend = request.context.buildResponse(response)
   val responseString =
     if (RequestChannel.isRequestLoggingEnabled) Some(response.toString(request.context.apiVersion))
     else None
   requestChannel.sendResponse(new RequestChannel.Response(request, Some(responseSend), SendAction, responseString))
   ```

   将`SendAction`类型的响应通过`RequestChannel`交给`Processor`，进一步发送给客户端。

## 总结

本篇阅读了处理Produce请求的流程，接着写入本地日志后的代码继续阅读：

写入本地日志后会返回处理结果，包含了每个请求写入的分区的相关状态，新增了消息集的`baseOffset`和写入日志的时间戳。对于acks字段为-1的情况，将timeout字段/消息集以及发送响应的回调丢给`DelayedOperation`对象进行异步的延迟操作，并通过purgatory字段检查异步处理的结果。

无论是`KafkaApis`本身，还是`DelayedOperation`，处理完后都会调用`sendResponseCallback`，acks不为0则根据Produce响应协议构造响应发送给客户端，acks为0则根据是否有错误响应而有不同的行为，若不包含错误响应则不进行操作，否则关闭socket，触发客户端重新获取元数据。

至此，完成了服务端对Produce请求的阅读，但是有不少细节没有深入，有待进一步阅读：

- `DelayedOperation`与`DelayedOperationPurgatory`：延迟操作的实现；
- `Log`类，对本地日志目录和日志片段（segment）文件直接操作；
- `Partition`类，管理了分区的副本broker，还有leader epoch等。