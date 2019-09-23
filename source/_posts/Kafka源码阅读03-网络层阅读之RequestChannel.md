---
title: 'Kafka源码阅读03: 网络层阅读之RequestChannel'
date: 2019-09-23 20:01:56
tags:
  - Kafka
top: 1
---

## 回顾

前2篇分析了`SocketServer`的启动以及`Acceptor`/`Processor`，对配置`listener`的网络地址，都会创建1个`Acceptor`和N个`Processor`，其中N为配置`num.network.thread`。每个`Acceptor`会创建1个默认的NIO `Selector`，每个`Processor`则都会创建1个Kafka自行实现`Selector`接口的`KSelector`。

socket都会被封装成`Channel`，即通道，代表socket两端的连接。`Acceptor`会创建一个`Channel`监听网络地址，并在其`Selector`注册读事件，然后负责将所有客户端的连接转换成`Channel`均衡地发送给`Processor`。

`Processor`则在其`Selector`上注册从`Acceptor`收到的`Channel`的读/写/关闭连接等事件，并分别处理。但是`Processor`只负责读取请求(Request)和写入响应(Response)，对于已完成的请求，会将其作为参数传给`requestChannel`调用`sendRequest()`。另一方面，`Processor`内部的响应队列，则是由`requestChannel`调用`sendResponse()`得到的。

而`RequestChannel`包含`requestQueue`字段缓存请求，另外它也和`SocketServer`一样保存了所有`Processor`的id和自身组成的`ConcurrentHashMap`。

## 处理请求

### 发送请求

在`Processor.processCompletedReceives()`方法中，会将封装了网络地址信息的请求传递给`sendRequest()`方法：

```scala
val header = RequestHeader.parse(receive.payload)
val context = new RequestContext(header, receive.source, channel.socketAddress,
  channel.principal, listenerName, securityProtocol)
val req = new RequestChannel.Request(processor = id, context = context,
  startTimeNanos = time.nanoseconds, memoryPool, receive.payload, requestChannel.metrics)
requestChannel.sendRequest(req)
```

`sendRequest()`方法的实现：

```scala
  // 发送待处理的请求, requestQueue 有容量上限, 由 queue.max.requests 配置, 若达到了上限则会阻塞。
  def sendRequest(request: RequestChannel.Request) {
    requestQueue.put(request)
  }
```

“发送”只是将请求放到了内部的请求队列中，而出队方法是在`server.KafkaRequestHandler`中调用的，不属于网络层的事情，暂时不管。因此看看`Request`类型的构造，传入了`header`和`context`。

### 请求头的解析

```java
    public static RequestHeader parse(ByteBuffer buffer) {
        try {
            // 前2个字节为 Api Key, 代表请求的类型
            short apiKey = buffer.getShort();
            // 后2个字节为 Api Version, 即客户端使用的API版本
            short apiVersion = buffer.getShort();
            // 通过上述字段创建 Schema 对象, 即完整的消息头
            Schema schema = schema(apiKey, apiVersion);
            // ByteBuffer.getXXX() 会修改内部偏移量, 因此需要将偏移量重置为最开始以便从头读取 buffer
            // 从头读取 buffer, 进而用 scheme.read() 构造请求头
            buffer.rewind();
            return new RequestHeader(schema.read(buffer));
        }  // 异常处理(略)
    }
```

Api Key的类型参考[Api Key](https://kafka.apache.org/protocol.html#protocol_api_keys)，消息协议类型参考[消息协议](https://kafka.apache.org/protocol.html#protocol_messages)：

```
Request Header => api_key api_version correlation_id client_id 
  api_key => INT16
  api_version => INT16
  correlation_id => INT32
  client_id => NULLABLE_STRING
```

这个头部即`Schema`类，其`read()`方法会把后面的correlation id和client id给读入，构造消息头。

### 请求上下文的构造

```scala
val context = new RequestContext(header, receive.source, channel.socketAddress,  channel.principal,
                                 listenerName, securityProtocol)
```

其实就是简单地将对应参数赋值给内部字段：

```java
    public final RequestHeader header;  // 消息头
    public final String connectionId; // 连接id，包含本地和远程的地址和表示连接的index
    public final InetAddress clientAddress; // 客户端地址
    public final KafkaPrincipal principal; // Channel的principal字段，用于信息认证
    public final ListenerName listenerName; // 配置 listeners 的名字部分(比如PLAINTEXT) 
    public final SecurityProtocol securityProtocol; // 安全协议, 根据listenerName解析的
```

消息头上一小节刚看完，连接id也是阅读源码至今一直见到的用于标识一条TCP连接的字符串，最后2个字段均为解析`listener`配置时解析出的`EndPoint`类的字段。

### 请求对象的创建

`RequestChannel.Request`字段比较多就不一一解释了（很多都是metric相关的），核心是请求body（0.10版本是字段现在是方法）：

```scala
    def body[T <: AbstractRequest](implicit classTag: ClassTag[T], nn: NotNothing[T]): T = {
      bodyAndSize.request match {
        case r: T => r
        case r =>
          throw new ClassCastException(s"Expected request with type ${classTag.runtimeClass}, but found ${r.getClass}")
      }
    }
```

该方法仅仅是检查请求类型T是否合法，若不合法则抛出异常。

关键部分是`bodyAndSize`进行类型匹配，该字段初始化：

```scala
private val bodyAndSize: RequestAndSize = context.parseRequest(buffer)
```

`context`解析请求的方法：

```java
    public RequestAndSize parseRequest(ByteBuffer buffer) {
        if (isUnsupportedApiVersionsRequest()) {
            // 未支持的 ApiVersion 被视为v0请求并且不被处理
            ApiVersionsRequest apiVersionsRequest = new ApiVersionsRequest((short) 0, header.apiVersion());
            return new RequestAndSize(apiVersionsRequest, 0);
        } else {
            ApiKeys apiKey = header.apiKey();
            try {
                short apiVersion = header.apiVersion();
                // 根据API版本将字节缓存解析成Struct的各个字段
                Struct struct = apiKey.parseRequest(apiVersion, buffer);
                // 根据请求类型/API版本/Struct字段创建实际的请求类型
                AbstractRequest body = AbstractRequest.parseRequest(apiKey, apiVersion, struct);
                return new RequestAndSize(body, struct.sizeOf());
            } catch (Throwable ex) {
                // 异常处理(略)
            }
        }
    }
```

```java
    // 类 AbstractRequest 的方法
    public static AbstractRequest parseRequest(ApiKeys apiKey, short apiVersion, Struct struct) {
        switch (apiKey) {
            case PRODUCE: 
                return new ProduceRequest(struct, apiVersion);
            case FETCH:
                return new FetchRequest(struct, apiVersion);
            // 其他类型的请求...(略)
            default:
                // 异常处理(略)
        }
    }
```

这部分代码都是Java实现的，为了能根据不同请求类型/API版本得到对应的请求类型的示例，实现得较为复杂，细节也不深入去看，总之，在Kafka中可以像这样调用`body()`方法得到实际的请求对象：

```scala
// 将请求的 ByteBuffer 解析成 MetadataRequest 对象
val metadataRequest = request.body[MetadataRequest]
```

### 取出请求

之前介绍了`Processor`仅仅是将请求加入阻塞队列`requestQueue`中，那么何时取出呢？找到其`poll()`方法的调用处：

```scala
  // 取得下个请求, 或者阻塞直到超时
  def receiveRequest(timeout: Long): RequestChannel.BaseRequest =
    requestQueue.poll(timeout, TimeUnit.MILLISECONDS)
```

再看看上述方法的调用处：

```scala
  // 类 KafkaRequestHandler 的方法
  def run() {
    while (!stopped) {
      // ...
      val req = requestChannel.receiveRequest(300) // 300ms超时
      // ...
      req match {
        case RequestChannel.ShutdownRequest =>
          // ...
        case request: RequestChannel.Request =>
          // ...
        case null => // continue
      }
    }
    // ...
  }
```

由于是API层的代码，所以略去了其他代码，只保留了`req`相关的。可以看到请求`Handler`线程会反复地从请求队列中取出请求，然后根据请求地类型进行不同处理。

### 多线程取出请求安全吗？

由于`ArrayBlockingQueue`是线程安全的，所以多个`Handler`线程从中取出请求是线程安全的。另一方面，关于顺序性，即来自同一个客户端的多个请求，必须保证取出的顺序也一致。《Apache Kafka源码剖析》书上给出了解释：`Processor.run()`方法通过多处注册/取消 读/写事件 来保证每个连接上只有一个请求和一个对应的响应来实现的。

具体而言，可以回顾我上一篇源码分析中`Processor`的部分：

1. 在`processCompletedReceives()`中，一旦接收到完整的请求`req`，在调用`sendRequest(req)`后会取消监听该`Channel`的读事件；
2. 在`processCompleteSends()`中，只有当响应成功返回客户端（将响应从缓存的`inflightResponses`移除）后，才会重新注册该`Channel`的读事件；
3. 在`processNewResponses()`中判断请求类型是`SendAction`时，会注册`Channel`的写事件；
4. 在`poll()`中向底层socket发送数据时，如果判断数据完毕，则会取消注册`Channel`的写事件。

## 处理响应

`Processor`自己维护了响应队列，并在`processNewResponses()`中调用`dequeueResponse()`方法依次出队， 那么，可以找到其对应方法`enqueueResponse()`的调用处：

```scala
  def sendResponse(response: RequestChannel.Response) {
    if (isTraceEnabled) {
      // 判断响应类型并打印日志(略)
    }

    val processor = processors.get(response.processor)
    // 如果 processor 已经关闭了, 可能会被移出 processors (此时返回null), 因此直接丢掉响应
    if (processor != null) {
      processor.enqueueResponse(response)
    }
  }
```

值得注意的是这里对`processor != null`的判断，Kafka 1.1.0的`SocketServer`支持`resizeThreadPoll()`方法来改变网络线程数量（也就是`Acceptor`对应`Processor`的数量），如果网络线程数减少的话，那么多出的`Processor`会调用`shutdown()`方法关闭，并通过`connectionId`将其从`Acceptor`和`RequestChannel`中的`processors`字段移除。

继续找到该方法的调用处，位于`KafkaApis`的`sendResponse()`方法中：

```scala
  private def sendResponse(request: RequestChannel.Request, responseOpt: Option[AbstractResponse]): Unit = {
    // 更新metrics(略)

    responseOpt match {
      case Some(response) =>
        val responseSend = request.context.buildResponse(response)
        val responseString =
          if (RequestChannel.isRequestLoggingEnabled) Some(response.toString(request.context.apiVersion))
          else None
        requestChannel.sendResponse(new RequestChannel.Response(request, Some(responseSend), SendAction, responseString))
      case None =>
        requestChannel.sendResponse(new RequestChannel.Response(request, None, NoOpAction, None))
    }
  }
```

此时`RequestChannel`只是将响应转发给了`Processor`，它本身并不维护响应队列（在0.10.0.1版本中则是维护了多个响应队列），真正维护响应队列的是`Processor`本身。

## 总结

关于Kafka网络层的阅读至此就告一段落，不得不说但作为Kafka的基础设施的Java NIO实现的部分更为复杂（`KafkaChannel`和`KafkaSelector`），但阅读源码不应太陷入细节（感觉我已经有些陷进去了……）。对于Kafka，最重要的还是它的业务层，也就是对[Kafka协议](https://kafka.apache.org/protocol.html)的实现。

`RequestChannel`的作用很简单：

- 维护请求队列，接收来自`Processor`的请求，并转发给`KafkaRequestHandler`进行处理；
- 从`KafkaApis`获取响应，发送给`Processor`。

加上文章开始总结的`Acceptor`/`Processor`，以及统筹全局的`SocketServer`，构成了Kafka的网络层，简单描述：

```
|             Network Layer                 |       API Layer        |
| Acceptor -> Processors <-> RequestChannel | -> KafkaRequestHandler |
|                                           | <- KafkaApis           |
| Client -> SocketChannel -> Acceptor       |                        |
| Client <-> KafkaChannel <-> Processor     |                        |
```

可以发现不管是什么`Channel`，都是起到了连接的作用。`Acceptor`通过最简单的`SocketChannel`与监听套接字连接，监听连接事件，并将接受的连接转发给`Processor`，之后`Processor`通过比较复杂的`KafkaChannel`与客户端连接，监听读/写事件并和客户端进行数据的交互。

数据分为来自客户端的请求和来自服务端的响应，`Processor`不负责这部分，而是通过`RequestChannel`将请求发送给`KafkaRequestHandler`，再从`RequestChannel`接收响应。

问题来了，而实际发送响应给`RequestChannel`的却是`KafkaApis`，因此**请求**=>**响应**的过程是由它们共同完成的，也就是接下来要阅读的API层。