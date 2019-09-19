---
title: 'Kafka源码阅读01: 网络层阅读之服务器的启动'
date: 2019-09-18 19:47:06
tags:
  - Kafka
top: 1
---
## 前言
今天正式开始阅读Kafka源码，作为阅读笔记的第一篇，先简单地介绍下背景。

阅读的Kafka版本是1.1.0，服务端源码在`core.main.scala.kafka`目录下，该目录下的源码文件仅有`Kafka.scala`，也就是服务端的启动入口，其他的若干个模块都阻止在各子目录下，这里首先阅读的是网络层，也就是`network`子目录下的代码。

阅读思路是直接看公用方法，然后再给一些逻辑以及用到的字段作注释，否则单看某些字段不看语境也不知道做什么。注释里会给英文两边加空格，逗号也使用英文逗号，方便vim快捷键按词前进/后退。

使用Intellij Idea阅读的，之前用得比较少，也很折腾了下配置过程，记录一些阅读源码的方法：

- 光标选中+单击鼠标左键：跳转至变量/函数定义处；
- Navigate菜单栏的Back和Forward，快捷键是`Ctrl+Alt+Left/Right`：后退/前进到前/后一次阅读的地方，一般时配合跳转功能回退；
- 光标选中+鼠标右键，选择Find Usages，快捷键是`Alt+Shift+F7`：查看变量/函数所有使用的地方；
- 快捷键`Alt+F7`：查看类的所有字段和方法。

对应我阅读C/C++源码时vim的`Ctrl+J`（`YCM`）/`Ctrl+]`（`ctags`）跳转，`Ctrl+O`回退，`LeaderF`查看类的字段和方法。之前vim一直没配置查找所有调用处的功能，一直是手动写个简单脚本用`egrep`在当前目录下递归搜关键词的……

不过IDE优点就是功能更大更全，上手新语言时直接使用，不必每接触一门语言旧学习怎么定制功能。

## SocketServer

注释表明了它是一个NIO套接字服务器，其线程模型是：

- 1个Acceptor线程处理新连接；
- Acceptor有N个Processor线程，其中每个都有自己的Selector从套接字中读取请求；
- M个Handler线程，处理请求，并将回应发给Processor线程用于写入。

`SocketServer`的主构造器有以下参数：

```scala
val config: KafkaConfig // 配置文件
val metric: Metrics // 度量指标
val time: Time // 对象创建时间
val credentialProvider: CredentialProvider // 证书提供者
```

混入了`Logging`和`KafkaMetricsGroup`特质，后者继承自前者，但并没重写`info()`等日志方法，而是将不同类型的metric（度量指标）给组织起来，提供了各种metric的工厂方法。

对于日志，设置了类相关的前缀表明broker id：

```scala
  private val logContext = new LogContext(s"[SocketServer brokerId=${config.brokerId}] ")
  this.logIdent = logContext.logPrefix
```

涉及到的一些字段，我添上了注释并按相关度整合了下：

```scala
  // 每个ip的最大连接数, 配置: max.connections.per.ip
  private val maxConnectionsPerIp = config.maxConnectionsPerIp
  // 指定ip的最大连接数, 会覆盖 maxConnectionsPerIp, 配置: max.connections.per.ip.overrides
  private val maxConnectionsPerIpOverrides = config.maxConnectionsPerIpOverrides
  // 由以上参数创建
  private var connectionQuotas: ConnectionQuotas = _

  // 请求队列的最大容量, 配置: queued.max.requests
  private val maxQueuedRequests = config.queuedMaxRequests
  // 内部维护了一个请求队列
  val requestChannel = new RequestChannel(maxQueuedRequests)

  // 每个 Processor 拥有自己的 Selector, 用于从连接中读取请求和写回响应
  // Processor 会将请求发送至 requestChannel, 会从 responseChannels 中读取响应
  private val processors = new ConcurrentHashMap[Int, Processor]() // key: id
  private var nextProcessorId = 0 // 递增作为每个 processor 的id

  // key: EndPoint, 即配置 listeners 指定的 ip/port 以及其 SecurityProtocol (默认PLAINTEXT)
  // 对每个绑定的 ip/port 都创建唯一对应的 Acceptor, 用于创建连接 Channel
  // PS: 该字段在 network包 内可访问，实际上目前也就这个类会访问。
  private[network] val acceptors = new ConcurrentHashMap[EndPoint, Acceptor]()
```

## 服务器启动源码分析

```scala
  def startup() {
    this.synchronized {
      // connectionQuotas 用于限制IP的最大连接数
      connectionQuotas = new ConnectionQuotas(maxConnectionsPerIp, maxConnectionsPerIpOverrides)
      // 创建 Acceptors 和 Processors
      createAcceptorAndProcessors(config.numNetworkThreads, config.listeners)
    }

    // 忽略了剩下的代码, 都是调用 KafkaMetricsGroup.newGauge 创建 Gauge对象, 
    // 用于测量某些指标, 比如Processor的平均闲置百分比/内存池可用大小/内存池占用大小
  }
```

再看看 Acceptor 和 Processor 的创建：

```scala
  private def createAcceptorAndProcessors(processorsPerListener: Int,
                                          endpoints: Seq[EndPoint]): Unit = synchronized {

    // socket内部缓冲区大小，底层可用C函数 setsockopt 设置 SO_SNDBUF/SO_RCVBUF
    val sendBufferSize = config.socketSendBufferBytes // socket.send.buffer.bytes
    val recvBufferSize = config.socketReceiveBufferBytes // socket.receive.buffer.bytes
    val brokerId = config.brokerId // broker.id

    endpoints.foreach { endpoint =>
      val listenerName = endpoint.listenerName
      val securityProtocol = endpoint.securityProtocol

      val acceptor = new Acceptor(endpoint, sendBufferSize, recvBufferSize, brokerId, connectionQuotas)
      KafkaThread.nonDaemon(s"kafka-socket-acceptor-$listenerName-$securityProtocol-${endpoint.port}", acceptor).start()
      acceptor.awaitStartup() // 等待acceptor线程完全启动
      acceptors.put(endpoint, acceptor) // 构建键值对 <EndPoint, Acceptor> 加入 acceptors
      // 单个 Acceptor 对应 processorsPerListener(即numNetworkThreads)个 Processors
      addProcessors(acceptor, endpoint, processorsPerListener)
    }
  }
```

对每个`Acceptor`，会创建多个`Processor`，类似地，也存入`ConcurrentHashMap`中：

```scala
  private def addProcessors(acceptor: Acceptor, endpoint: EndPoint, newProcessorsPerListener: Int): Unit = synchronized {
    val listenerName = endpoint.listenerName
    val securityProtocol = endpoint.securityProtocol
    val listenerProcessors = new ArrayBuffer[Processor]()

    for (_ <- 0 until newProcessorsPerListener) {
      val processor = newProcessor(nextProcessorId, connectionQuotas, listenerName, securityProtocol, memoryPool)
      listenerProcessors += processor
      // 内部调用了 putIfAbsent() 方法将其加入了 requestChannel 内部维护的 ConcurrentHashMap[Int, Processor],
      // key为 processor.id, 使用 putIfAbsent() 方法是因为理论上 processer.id 是唯一的, 因此在插入重复的id时,
      // 不应插入新对象, 而是仅仅返回一个非空 Processor 并根据返回值打印日志
      requestChannel.addProcessor(processor)
      // 仅在此递增 SocketServer 的 nextProcessId 字段，因此保证了对不同 Acceptor 的 Processors,
      // 其 id 是不同的，因此每个 Processor 在 processors 字段中都对应唯一的 key
      nextProcessorId += 1
    }
    listenerProcessors.foreach(p => processors.put(p.id, p))
    // 启动 listenerProcessors 的所有 Processor 线程, 并将其添加到内部维护的 processor: ArrayBuffer[Processor] 中
    acceptor.addProcessors(listenerProcessors)
  }
```

## RequestChannel/Acceptor/Processor的字段

从上述代码可知，重点是`RequestChannel`/`Acceptor`/`Processor`这3个类型，于是现在看看它们创建时除去传入构造器的参数外初始化的其他字段（依然忽略metrics相关的）。

首先是`RequestChannel`的字段：

```scala
// 创建了固定长度的请求队列，queueSize由 queued.max.requests 决定
private val requestQueue = new ArrayBlockingQueue[BaseRequest](queueSize)
// 之前已经提过，创建 Processors 时添加进的
private val processors = new ConcurrentHashMap[Int, Processor]()
```

此外，配置是使用`kafka.server.KafkaConfig`类实现的，默认配置在伴生对象`kafka.server.Defaults`中（比如默认的`queueSize`为500），并在`KafkaConfig`的伴生对象的`configDef`字段创建时加载。

再就是`Acceptor`的字段：

```scala
// NIO Selector, 用于注册 connect, read, write 等事件，并将事件分发给 Acceptor, Processors
private val nioSelector = NSelector.open()
// 服务通道, 绑定 EndPoint, 用于接收客户端的连接, 类型为 ServerSocketChannel
val serverChannel = openServerSocket(endPoint.host, endPoint.port)
// 之前提过的，保存每个 Acceptor 对应的 Processors, 没有存为映射, 因为 Acceptor 要将
// 连接均衡地分配给 Processors, 不涉及查询操作, 更多地需要遍历, 比如round-robin算法
private val processors = new ArrayBuffer[Processor]()
```

其中 `NSelector` 就是 `Selector` 的别名：

```scala
import java.nio.channels.{Selector => NSelector}
```

最后是`Processor`的字段：

```scala
// SocketChannel 的并发队列, 用于管理 Acceptor 分配的socket连接
private val newConnections = new ConcurrentLinkedQueue[SocketChannel]()
// ConnectionId 到 Response 的映射, 缓存待发送的响应
private val inflightResponses = mutable.Map[String, RequestChannel.Response]()
// 缓存产生的响应
private val responseQueue = new LinkedBlockingDeque[RequestChannel.Response]()

// 创建的是 kafka.common.network.Selector, 也就是自己实现的 Selector
private val selector = createSelector(
  ChannelBuilders.serverChannelBuilder(listenerName,
    listenerName == config.interBrokerListenerName,
    securityProtocol,
    config,
    credentialProvider.credentialCache,
    credentialProvider.tokenCache))

// 用于生成连接的 index, 类似 SocketServer 生成 Processor.id, 不过保证了 index 非负
private var nextConnectionIndex = 0
```

不得不说这里为何要使用 `ConcurrentLinkedQueue` 和 `LinkedBlockingDeque` 还是不清楚，但还是先不要在意细节，注意这里保存了2份`Response`，一个只是临时缓存处理后的响应，另一个则是真正待发送的响应，因为用key记录了连接信息：

```scala
  // 作为 inflightResponses 的key, 记录了本地地址/远程地址, 以及连接对应的索引
  // 该索引是通过 nextConnectionIndex 自增生成的, 而且非负
  private[network] case class ConnectionId(localHost: String, localPort: Int, remoteHost: String, remotePort: Int, index: Int) {
    override def toString: String = s"$localHost:$localPort-$remoteHost:$remotePort-$index"
  }
```

## 服务器启动总结

总结下来，启动`SocketServer`时其实就是根据配置参数创建了1个`RequestChannel`，M个`Acceptor`，M*N个`Processor`。其中M是监听地址的数量，N是`num.network.thread`配置的Acceptor对应的Processors的数量。

每个监听地址除了ip和port外，还有协议类型和名称，这些共同组成了`EndPoint`类。

- `SocketServer`保存`EndPoint`到`Acceptor`的映射和`Processor.id`到`Processor`的映射；
- `requestChannel`持有M*N个`Processor`的`id`到其自身的映射；
- 每个`Acceptor`持有1个`Selector`；
- 每个`Acceptor`持有1个监听`EndPoint`的`ServerSocketChannel`；
- 每个`Acceptor`持有N个`Processor`组成的数组；
- 每个 `Processor` 持有1个`Selector`（Kafka自己实现的`Selectable`接口）；
- 每个`Processor`持有一组socket连接；
- `acceptors`和`processors`都启动了线程（供`M*(N+1)`个）构成了整个网络层的处理。

Kafka的网络层是使用Reactor模式的，使用了Java NIO，所有的socket读写都是非阻塞模式，具体框架可以参考《Apacha Kafka源码剖析》一书，我目前也是照着这本书的思路去看源码。

不过对Java NIO不熟悉，虽然看了眼核心还是分发事件的`Selector`（I/O多路复用），但是封装得比较好。抽空去看看。

网络层运转的核心还是`Acceptor`和`Processor`的线程函数，也就是这2个类的`run()`方法，也是接下来要读的部分。

## 为什么使用ConcurrentHashMap？

《Apacha Kafka源码剖析》书中使用的是Kafka 0.10.0.1版本，其中`acceptors`和`processors`的类型是：

```scala
  private val processors = new Array[Processor](totalProcessorThreads)
  private[network] val acceptors = mutable.Map[EndPoint, Acceptor]()
```

而1.1.0版本就都用`ConcurrentHashMap`来保存了，看源码时我也在想为什么不用数组去存processors，因为key就是从0到N-1。搜了下这个结构在Java 8用了不同于7的实现，抽空去看看。

然后看到了`addListeners`/`removeListeners`方法，前者根据新的`Seq[EndPoint]`重新创建`acceptors`和`processors`，后者则将指定的`Seq[EndPoint]`对应的`Acceptor`从`acceptors`中删除。而这两个方法在0.10.0.1版本中没有，所以就能用固定长度的数组来保存`processors`，也能用不支持并发访问的`mutable.Map`来保存`acceptors`。

不过还有个不明白的地方，看到直接访问`acceptors`和`processors`的都是`SocketServer`内部，而除了`boundPort()`方法和`stopProcessingRequests()`外，所在访问它们的方法都直接用`synchronized`关键字保护了，而`boundPort()`方法仅在`xxxTest.scala`中被调用了，这样的话使用`ConcurrentHashMap`是否必要？
