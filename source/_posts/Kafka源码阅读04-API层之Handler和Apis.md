---
title: Kafka源码阅读04-API层之Handler和Apis
date: 2019-10-08 12:14:48
tags:
  - Kafka
top: 1
---

## 回顾

之前通过网络层的阅读，我们知道了和客户端直接进行读写的是`Processor`，但是它会将请求通过`RequestChannel`发送给`KafkaRequestHandler`，同时也会接收`KafkaApis`通过`RequestChannel`回复的响应。因此从本篇开始阅读API层，也就是Handler和Apis，它们都是位于`server`包内。

## Handler线程的创建

回顾请求的调用链

```
Processor.processCompleteReceives
  \-- requestChannel.sendRequest(request)
    \-- requestQueue.put(request)
```

`Processor`的`requestChannel`字段调用`sendRequest()`方法，该方法将请求放入其`requestQueue`字段中：

```scala
private val requestQueue = new ArrayBlockingQueue[BaseRequest](queueSize)
```

既然有入队，就肯定有出队，找到其`poll()`方法的调用处：

```scala
// 取得下一个请求，或者阻塞直到超时  
def receiveRequest(timeout: Long): RequestChannel.BaseRequest =
  requestQueue.poll(timeout, TimeUnit.MILLISECONDS)
```

继续找到该方法的调用处，在`KafkaRequestHandler.run()`方法中：

```scala
val req = requestChannel.receiveRequest(300) // 300ms超时
```

`KafkaRequestHandler`实现了`Runnable`接口，也就是说，它在调用`start()`方法时就会启动线程，执行`run()`方法，查找使用它的地方，为`KafkaRequestHandlerPool`的字段：

```scala
val runnables = new mutable.ArrayBuffer[KafkaRequestHandler](numThreads)
```

Poll类管理了Handler线程，它默认创建了`numThreads`个Handler线程。

PS：这里使用了`ArrayBuffer`而非固定大小的`Array`，和之前提到的`Processor`和`Acceptor`使用`ConcurrentHashMap`保存一样，都是为了支持`resize`操作。

再看看`Pool`类的使用处，它是`KafkaServer`的字段

```scala
var requestHandlerPool: KafkaRequestHandlerPool = null
```

`KafkaServer`才是真正的Kafka服务器的类，而之前介绍的`SocketServer`类只是它用来管理网络的部分，也是其中的一个字段：

```scala
var socketServer: SocketServer = null
```

再看看`requestHandlerPool`的使用处，位于`KafkaServer`的`startup()`方法：

```scala
requestHandlerPool = new KafkaRequestHandlerPool(config.brokerId,
                                                 socketServer.requestChannel,
                                                 apis, time, config.numIoThreads)
```

第1个参数为配置的`broker.id`，第2个参数为`RequestChannel`对象，第3个参数为`KafkaApis`对象，第4个参数为`KafkaServer`的构造参数，为`SystemTime`对象，位于`common.util`包内。

第5个参数为Handler线程的数量，为`config.numIoThreads`，对应配置文件的`num.io.threads`，默认值为8（见`KafkaConfig.scala`的`Defaults`伴生对象）。

而Pool类中创建Handler线程代码如下：

```scala
  for (i <- 0 until numThreads) {
    createHandler(i)
  }

  def createHandler(id: Int): Unit = synchronized {
    runnables += new KafkaRequestHandler(id, brokerId, aggregateIdleMeter, threadPoolSize, requestChannel, apis, time)
    KafkaThread.daemon("kafka-request-handler-" + id, runnables(id)).start()
  }
```

`id`为Handler线程的编号（从0开始），`aggregateIdleMeter`为度量指标相关（配合`time`字段计算Handler线程闲置的时间，我们依旧忽略之），`threadPoolSize`为线程池大小（即线程数量）。剩余参数都是Pool类的构造参数。

至此，我们知道了`KafkaServer`管理了Handler线程池，会根据配置的`num.io.threads`创建对应数量的Handler线程，并且多个Handler线程共享了`KafkaApis`对象和`RequestChannel`对象。

## Handler线程实现

忽略了日志和度量指标的部分：

```scala
  def run() {
    while (!stopped) {
      // 从requestChannel中取得请求，timeout为300ms
      val req = requestChannel.receiveRequest(300)

      req match {
        // Shutdown类型的请求，直接退出Handler线程
        case RequestChannel.ShutdownRequest =>
          debug(s"Kafka request handler $id on broker $brokerId received shut down command")
          shutdownComplete.countDown()
          return

        // 正常请求
        case request: RequestChannel.Request =>
          try {
            request.requestDequeueTimeNanos = endTime
            // 交给apis处理
            apis.handle(request)
          } catch {  // 处理api.handle()可能抛出的异常
            case e: FatalExitError =>
              shutdownComplete.countDown()
              Exit.exit(e.statusCode)
            case e: Throwable => error("Exception when handling request", e)
          } finally {
            request.releaseBuffer()
          }

        case null => // continue
      }
    }
    shutdownComplete.countDown()
  }
```

逻辑很简单，Handler线程反复地从`requestChannel`中取得请求，交由`apis`进行处理，如果处理出错会捕获异常，并以异常中包含的错误码退出当前线程。这也解释了前一篇的疑问：为何请求的发送对象是Handler线程，而响应却来自于Apis。

值得注意的地方是，这里还有个`ShutdownRequest`，是用来退出Handler线程的，找到它的调用处：

```scala
def sendShutdownRequest(): Unit = requestQueue.put(ShutdownRequest)
```

`RequestChannel`对象调用该方法将Shutdown请求加入队列中，而该方法的调用处位于`KafkaRequestHandler`内：

```scala
def initiateShutdown(): Unit = requestChannel.sendShutdownRequest()
```

进一步往上找，会发现位于`KafkaRequestHandler`的`shutdown()`方法内，用于Handler线程的正常退出（相对而言，`apis.handle()`抛出异常则是异常退出，退出码不为0）：

```scala
  def shutdown(): Unit = synchronized {
    info("shutting down")
    for (handler <- runnables)
      handler.initiateShutdown()
    for (handler <- runnables)
      handler.awaitShutdown()
  }
```

至于`awaitShutdown()`方法，则是Java线程退出的惯用法，即调用了`CountDownLatch`对象的`await()`方法，等待计数归0，可以看到`run()`方法中不同的退出分支都会调用`shutdownComplete.countDown()`方法，即将`CountDownLatch`对象`shutdownComplete`的计数减1，而其初始计数为1：

```scala
private val shutdownComplete = new CountDownLatch(1)
```

## Apis

`KafkaApis`对象的创建在`KafkaRequestHandlerPool`创建之前，其构造参数有18个，因此暂时不详细列出，最为关键的还是`requestChannel`。看看关键的`handle()`方法：

```scala
  def handle(request: RequestChannel.Request) {
    try {
      request.header.apiKey match {
        case ApiKeys.PRODUCE => handleProduceRequest(request)
        case ApiKeys.FETCH => handleFetchRequest(request)
        // 其他类型的的ApiKeys...
      }
    } catch {
      case e: FatalExitError => throw e
      case e: Throwable => handleError(request, e)
    } finally {
      request.apiLocalCompleteTimeNanos = time.nanoseconds
    }
  }
```

取得请求头的`apiKey`（见前一篇的**请求头的解析**一节），根据类型调用不同的`handle*()`方法进行处理，这里看一个比较简单的例子，是对`LIST_OFFSETS`请求的处理：

```scala
  def handleListOffsetRequest(request: RequestChannel.Request) {
    val version = request.header.apiVersion()

    // 根据请求头的API版本进行不同的处理
    val mergedResponseMap = if (version == 0)
      handleListOffsetRequestV0(request)
    else
      handleListOffsetRequestV1AndAbove(request)

    // 参数2为将requestThrottleMs作为输入的函数，会根据该输入创建ListOffsetResponse对象
    // 并且在sendResponseMaybeThrottle类根据requestThrottleMs判断是否创建该对象并发送
    sendResponseMaybeThrottle(request, requestThrottleMs => new ListOffsetResponse(requestThrottleMs, mergedResponseMap.asJava))
  }
```

进一步的分析略过，总之Apis在处理完请求后，如果判断需要发送，则会创建响应的响应（`*Response`）类型，并调用`sendResponse()`方法：

```scala
  private def sendResponse(request: RequestChannel.Request, responseOpt: Option[AbstractResponse]): Unit = {
    // 对响应的每个非0错误码更新度量指标
    responseOpt.foreach(response => requestChannel.updateErrorMetrics(request.header.apiKey, response.errorCounts.asScala))

    responseOpt match {
      case Some(response) =>
        // 若响应存在，则构造响应的Send，并调用requestChannel的sendResponse()方法发送
        val responseSend = request.context.buildResponse(response)
        // 如果RequestChannel伴生对象的isRequestloggingEnabled则构造该字符串
        val responseString =
          if (RequestChannel.isRequestLoggingEnabled) Some(response.toString(request.context.apiVersion))
          else None
        requestChannel.sendResponse(new RequestChannel.Response(request, Some(responseSend), SendAction, responseString))
      case None =>
        requestChannel.sendResponse(new RequestChannel.Response(request, None, NoOpAction, None))
    }
  }
```

可见实际上，还是利用`requestChannel.sendResponse()`方法发送响应（参见前一篇的**处理响应**一节，会将响应加入`Processor`的响应队列中）。这里的`Send`接口表示的是待发送的数据，而`String`则是用于调试的：

```scala
object RequestChannel extends Logging {
  private val requestLogger = Logger("kafka.request.logger")
  // ...

  def isRequestLoggingEnabled: Boolean = requestLogger.underlying.isDebugEnabled
  // ...
}
```

这里需要额外说明下，Kafka的日志系统使用的是`SLF4J`，它本身只是日志的抽象层，而没有具体的实现，因此在编译运行Kafka时会提示警告：

> SLF4J: Failed to load class "org.slf4j.impl.StaticLoggerBinder". SLF4J: Defaulting to no-operation (NOP) logger implementation SLF4J: See http://www.slf4j.org/codes.html#StaticLoggerBinder for further details.

只有将具体的日志jar包放入classpath中，才会成功打印日志，因此从Kafka源码中是无法确定`isRequestLoggingEnabled`在哪里设置为`true`，取决于实际日志包的配置。

## 总结

API层其实还是很简单的，创建`num.io.threads`个Handler线程，从共享的`RequestChannel`中取出请求（使用`ArrayBlockingQueue`请求队列保证线程安全并且限制最大请求数），如果不是Handler调用`shutdown()`方法加入的关闭请求，则将其交给Apis对象进行处理，处理完请求后会构造响应对象，通过`RequestChannel`加入到`Processor`内部的响应队列（使用`LinkedBlockingQueue`响应队列保证线程安全，并且不限制最大响应数量）。

实际的请求的解析和响应的构造则集中于`KafkaApis`类中，接下来则是通过不同的`ApiKey`类依次看看Kafka支持哪些请求，并且内部是怎样处理这些请求。