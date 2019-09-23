---
title: 'Kafka源码阅读02: 网络层阅读之Acceptor和Processor'
date: 2019-09-20 11:38:31
tags:
  - Kafka
top: 1
---
## AbstractServerThread

`Acceptor`和`Processor`的抽象基类，封装了一些辅助的变量和方法（这里重新组织了下代码顺序）：

```scala
private val startupLatch = new CountDownLatch(1)
@volatile private var shutdownLatch = new CountDownLatch(0)

private val alive = new AtomicBoolean(true)
protected def isRunning: Boolean = alive.get

def shutdown(): Unit = {
  // 如果线程仍在运行, 则将 alive 置为true表示线程之后会关闭, 然后调用抽象方法 wakeup()
  if (alive.getAndSet(false))
    wakeup()
  shutdownLatch.await()
}

// 等待线程完全启动
def awaitStartup(): Unit = startupLatch.await

// 标识线程已经启动, 这样就可以等待停止操作了, 因此将 shutdownLatch 指向倒计时为1的对象
// 这样做是为了防止启动时抛出异常, 比如绑定正在使用的地址, 此时应该在处理异常之后仍然能 shutdown,
// 此时 shutdownComplete 的调用会因为异常而被跳过, 如果计数初始化为1会一直阻塞
protected def startupComplete(): Unit = {
  // Replace the open latch with 
  shutdownLatch = new CountDownLatch(1)
  startupLatch.countDown()
}

// 标识线程已经关闭
protected def shutdownComplete(): Unit = shutdownLatch.countDown()

def close(channel: SocketChannel): Unit = {
  if (channel != null) {
    debug("Closing connection from " + channel.socket.getRemoteSocketAddress())
    // 减少 channel 对应地址的连接计数
    connectionQuotas.dec(channel.socket.getInetAddress)
    // 关闭 socket 连接以及 channel 本身, 吞下异常, 也就是说关闭出错不是什么严重错误, 写入日志以供分析就行
    CoreUtils.swallow(channel.socket().close(), this, Level.ERROR)
    CoreUtils.swallow(channel.close(), this, Level.ERROR)
  }
}
```

## Acceptor.run()

由于循环嵌套还是有点深的，先忽略对Channels的处理部分

```scala
  def run() {
    serverChannel.register(nioSelector, SelectionKey.OP_ACCEPT) // 注册OP_ACCEPT事件
    // 标识启动完成, 之后 acceptor.awaitStartup() 才会返回, 回顾 createAcceptorAndProcessors()
    // 也就是说 Server 启动时, 必须等到 acceptor 注册 OP_ACCEPT 事件后才会执行后续步骤:
    //   将acceptor加入acceptors, addProcessors, 创建下个acceptor ...
    startupComplete()
    try {
      var currentProcessor = 0 // 记录当前processor的id
      while (isRunning) {
        try {
          // 轮询Selector直到有channels准备好I/O, 或者超时(500ms)
          val ready = nioSelector.select(500)
          if (ready > 0) {  // 有ready个channels准备好I/O
            // TODO: 处理准备好I/O的channels
          }  // else: ready <= 0
        }
        catch {
          // 假设有特定的channel在select时出错, 或者收到bad request, 我们不想要让其他channels受到影响
          // 因此遇到异常只需要打印错误即可。
          // 但是scala会通过ControlThrowable来进行流程控制, 所以此时需要继续将异常往上抛(这是安全的)
          // 在scala 2.13中可以用 case NonFatal(e) 来避免ControlThrowable被捕获
          case e: ControlThrowable => throw e
          case e: Throwable => error("Error occurred", e)
        }
      }
    } finally {
      // Acceptor线程结束后的清理工作
      debug("Closing server socket and selector.")
      CoreUtils.swallow(serverChannel.close(), this, Level.ERROR)
      CoreUtils.swallow(nioSelector.close(), this, Level.ERROR)
      shutdownComplete()
    }
  }
```

外层`try-finally`块没有`catch`，也就是说一切异常都在`while`循环体内进行处理，循环体内则是一个大的`try-catch`，注意重抛`ControlThrowable`的手法，可以参考[Scala 2.13 ControlThrowable](https://www.scala-lang.org/api/current/scala/util/control/ControlThrowable.html)和[Scala 2.12 ControlThrowable](https://www.scala-lang.org/api/2.12.10/scala/util/control/ControlThrowable.html)。

`Selector`的处理和Linux的`epoll_wait`如出一辙，所以这里还是很熟悉的，不同的是没有处理`ready <= 0`的情况，接口文档里写的是

> @return  The number of keys, possibly zero, whose ready-operation sets were update

`select()`方法不会返回负值，像`epoll_wait`返回-1的情况，`Selector`是直接抛出异常了，文档里也写了3种异常：

- `IOException`: If an I/O error occurs;
- `ClosedSelectorException`: If this selector is closed;
- `IllegalArgumentException`: If the value of the timeout argument is negative.

接下来看`ready > 0`时的代码，也是核心的处理逻辑：

```scala
try {
  // 遍历所有的key, 类型为SelectionKey
  val key = iter.next
  iter.remove() // 从集合中移除该key, 防止
  if (key.isAcceptable) {  // 该key的channel可以接受新的socket连接
    // round robin算法, 将连接均衡分配给计算出的下标对应的processor
    // 比如3个processors, 接收了7个连接, 则分配的processor下标依次为: 0,1,2,0,1,2,0,1
    val processor = synchronized {
      currentProcessor = currentProcessor % processors.size
      processors(currentProcessor)
    }
    accept(key, processor)
  } else
    throw new IllegalStateException("Unrecognized key state for acceptor thread.")

  // round robin算法, 迭代
  currentProcessor = currentProcessor + 1
} catch { // 遍历keys及处理每个key时的异常在此打印
  case e: Throwable => error("Error while accepting connection", e)
}
```

思路很简单，就是用round robin算法简单做下负载均衡，调用`accept()`方法将key对应的连接分配给指定processor，因此核心其实是`accept()`方法。

PS：一个细节，外层`catch`处理了`ControlThrowable`，而内层`catch`并没处理，因为该异常是实现流程控制的，在迭代器到达末尾时才会抛出该异常，所以迭代循环中不会抛出该异常。另一个细节，这里每次迭代都把迭代器移除，这里大概是Java不会像C++一样，对象销毁的时候自动析构吧，而且Jave的`Set`移除迭代器之后不影响继续遍历。

看看`accept()`的实现（删掉了日志语句）：

```scala
  def accept(key: SelectionKey, processor: Processor) {
    // key.channel()向下转型, 从抽象类 SelectableChannel 转型为派生类 ServerSocketChannel
    val serverSocketChannel = key.channel().asInstanceOf[ServerSocketChannel]
    val socketChannel = serverSocketChannel.accept() // socket accept, 返回表示连接的 SocketChannel
    try {
      // 将远程地址对应的连接数加1, 如果超过了配置的最大连接数限额, connectionQuotas会抛出 TooManyConnectionsException
      connectionQuotas.inc(socketChannel.socket().getInetAddress)
      socketChannel.configureBlocking(false) // socket设为非阻塞模式
      socketChannel.socket().setTcpNoDelay(true) // socket设置TCP_NODELAY选项, 禁止Nagle算法
      socketChannel.socket().setKeepAlive(true) // socket设置保活模式, 长时间没有发送心跳则发出RST包重置连接
      // 配置的 socket.send.buffer.bytes 不为默认值, 则设置 SO_SNDBUF 选项重置发送缓冲区大小
      if (sendBufferSize != Selectable.USE_DEFAULT_BUFFER_SIZE)
        socketChannel.socket().setSendBufferSize(sendBufferSize)

      processor.accept(socketChannel)
    } catch {
      case e: TooManyConnectionsException =>
        close(socketChannel)
    }
  }
```

跟socket编程里一样的套路，只不过检查了同一个IP的最大连接数是否超限，并且给表示连接的socket设置了一些选项，然后实际上还是调用了`Processor.accept()`方法：

```scala
  def accept(socketChannel: SocketChannel) {
    newConnections.add(socketChannel)
    wakeup()
  }
```

这里就很简单了，把配置好的`SocketChannel`给加入`Processor`内部的并发队列`newConnections`中，其类型前一篇提过，是`ConcurrentLinkedQueue`。

## Acceptor.run()总结

抛开一些程序设计上的细节性知识，其实`Acceptor`线程的逻辑就是：

1. 循环，从`Selector`中等待I/O事件就绪；
2. 遍历所有的I/O事件，将`isAcceptable`的套接字取出，并调用socket的`accept()`取得新连接；
3. 检查最大连接数，没超限的话进行一些socket选项配置；
4. 将配置后的socket存入`Processor`的内部队列中。

可以看到`Acceptor`仅仅做了中介的作用，它是直接和客户端的连接请求打交道的，将成功的连接处理后传递给`Processor`，这样`Processor`就可以专心去处理网络数据的读写。

另一方面，我们可以看到`Channel`（在这里是`SocketChannel`类）其实就是对socket句柄的封装。

## Processor.run()

```scala
  override def run() {
    startupComplete()
    try {
      while (isRunning) {
        try {
          configureNewConnections() // 处理缓存的新连接
          processNewResponses() // 处理缓存的响应
          poll() // 轮询, 从 Selector 中获取准备好I/O的事件
          // 处理已完成的接收/发送以及断开的channels
          processCompletedReceives()
          processCompletedSends()
          processDisconnected()
        } catch {
          // 这里吞下了所有异常, 因为让 processor 退出对 broker 的影响可能会很大, 但值得商榷的是,
          // 是否存在需要让整个 broker 停止的异常。
          // 通常抛出的异常都是和特定socket或者bad request相关的, 这些异常被捕获, 然后会被独立的方法处理, 因此不会在这里
          // 被捕获, 所以可能这里只会看到 ControlThrowable (仅仅是可见, 没有地方会抛出)
          case e: Throwable => processException("Processor got uncaught exception.", e)
        }
      }
    } finally {
      // 将异常信息写入日志(略)
      shutdownComplete()
    }
  }
```

只用照着`try`作用域内的方法一个个地看下来就行。

### 1. configureNewConnections

```scala
  private def configureNewConnections() {
    while (!newConnections.isEmpty) {
      // newConnections的SocketChannel依次出队
      val channel = newConnections.poll()
      try {
        // 调用链: selector.registerChannel => selector.buildAndAttachKafkaChannel => channelBuilder.buildChannel()
        // channel 会注册 OP_READ 事件(返回 SelectionKey)到 selector 上, 然后和 connectionId, SelectionKey 一起构造
        // KafkaChannel 对象, 以 connectionId 作为key组成键值对加入 selector.channels 中
        selector.register(connectionId(channel.socket), channel)
      } catch {
        // 捕获所有异常, 关闭 对应的socket防止socket泄漏
        case e: Throwable =>
          close(channel)
          // 将异常信息写入日志(略)
      }
    }
  }
```

可见`Acceptor`仅仅将表示连接的`SocketChannel`交给`Processor`，而`Processor`则会为其注册读事件，同时交给`selector`管理时会将其包装为`KafkaChannel`，这个包装过程是由`ChannelBuilder`接口完成的，而接口指向的实际对象是在`Processor.createSelector()`中`ChannelBuilders.serverChannelBuilder()`方法创建的，对`PLAINTEXT`协议，即`PlaintextChannelBuilder`，其`buildChannel()`方法的调用和实现依次为：

```scala
// id: SocketChannel.connectionId
// key: SocketChannel.register() 返回的 SelectionKey
// maxReceiveSize: config.socketRequestMaxBytes, 即配置"socket.request.max.bytes"
// memoryPool: SocketServer.memoryPool
KafkaChannel channel = channelBuilder.buildChannel(id, key, maxReceiveSize, memoryPool);
key.attach(channel);  // key原本是attach之前的SocketChannel的, 现在改变attach的对象
```

```java
    @Override
    public KafkaChannel buildChannel(String id, SelectionKey key, int maxReceiveSize, MemoryPool memoryPool) throws KafkaException {
        try {
            PlaintextTransportLayer transportLayer = new PlaintextTransportLayer(key);
            PlaintextAuthenticator authenticator = new PlaintextAuthenticator(configs, transportLayer);
            return new KafkaChannel(id, transportLayer, authenticator, maxReceiveSize,
                    memoryPool != null ? memoryPool : MemoryPool.NONE);
        } catch (Exception e) {
            // 异常处理(略)
        }
    }
```

```java
    public PlaintextTransportLayer(SelectionKey key) throws IOException {
        this.key = key;
        this.socketChannel = (SocketChannel) key.channel();
    }
```

可以发现`key`和`channel: SocketChannel`被存到了`KafkaChannel.transportLayer`字段中，因此在后面的源码中，给`KafkaChannel`注册和取消读/写事件到`Selector`上时是使用`transportLayer`的`addInterestOps()`和`removeInterestOps()`方法：

```java
    @Override
    public void addInterestOps(int ops) {
        key.interestOps(key.interestOps() | ops);
    }

    @Override
    public void removeInterestOps(int ops) {
        key.interestOps(key.interestOps() & ~ops);
   
```

其实也就是调用了`SelectionKey`的`interestOps()`方法，不过包装了位运算`|`和`&~`来表示添加和移除。

### 2. processNewResponses

```scala
  private def processNewResponses() {
    var curr: RequestChannel.Response = null
    while ({curr = dequeueResponse(); curr != null}) {
      // 将响应从 responseQueue 中依次出队, 这里取得 connectionId 作为 channelId
      val channelId = curr.request.context.connectionId
      try {
        // 根据响应的类型进行不同操作
        curr.responseAction match {
          case RequestChannel.NoOpAction =>
            // 无操作: 无需发送响应给客户端, 因此需要读取更多请求到服务端的socket buffer中
            // 调用链: selector.unmute() => channel.unmute()
            // 会将 channel 从 selector.explicitlyMutedChannels 中移除,
            // 如果该channel处于连接状态, 会在 channel.transportLayer 注册 OP_READ 事件。
            updateRequestMetrics(curr)
            trace("Socket server received empty response to send, registering for read: " + curr)
            openOrClosingChannel(channelId).foreach(c => selector.unmute(c.id))
          case RequestChannel.SendAction =>
            // 发送: 调用链为 sendResponse() => selector.send() => channel.setSend()
            // 将响应加入 inflightResponses 中, 并在 channel.transportLayer 注册 OP_WRITE 事件
            val responseSend = curr.responseSend.getOrElse(
              throw new IllegalStateException(s"responseSend must be defined for SendAction, response: $curr"))
            sendResponse(curr, responseSend)
          case RequestChannel.CloseConnectionAction =>
            // 关闭连接： 关闭channel
            updateRequestMetrics(curr)
            trace("Closing socket connection actively according to the response code.")
            close(channelId)
        }
      } catch {
        // 将异常信息写入日志(略)
      }
    }
  }
```

可以看到，`Processor`仅仅是对缓存在`responseQueue`中的响应进行处理，但是从请求到响应的转换并不是它的工作，查找了`responseQueue`的使用地方，可以看到实际上响应是由`RequestChannel.sendResponse()`方法发送过来的，更上一层，是`KafkaApis.sendResponse()`方法调用该方法，因此实际上是`KafkaApis`（位于`kafka.server`包内）完成对请求的处理。

至于`updateRequestMetrics()`方法和异常处理的部分我们不再关心。

### 3. poll

```scala
  private def poll() {
    // 轮询300ms, 会将读取的请求/发送的响应/断开的连接，放入 selector 的 completeReceives/completedSends/disconnected
    try selector.poll(300)
    catch {
      case e @ (_: IllegalStateException | _: IOException) =>
        // 不会重抛异常, 这样这次轮询的所有完成的 sends/receives/connections/disconnections 事件都会被处理
        error(s"Processor $id poll failed due to illegal state or IO exception")
    }
  }
```

关键是`selector.poll()`方法：

```java
    @Override
    public void poll(long timeout) throws IOException {
        if (timeout < 0) // 检查参数合法性
            throw new IllegalArgumentException("timeout should be >= 0");

        boolean madeReadProgressLastCall = madeReadProgressLastPoll;
        clear(); // 清理前1次 poll() 中设置的一些字段 (理应在此2次 poll() 之间对它们全部进行处理)

        boolean dataInBuffers = !keysWithBufferedRead.isEmpty();

        // 在以下情形时将timeout置为0 (代表已经有一些Channel I/O就绪了, select()会立刻返回)
        // 1. 已经有一些接收数据的 Channel 在上一次 poll() 中读了一些数据;
        // 2. 有可连接但暂为完成连接的 Channels;
        // 3. 上次有 Channel 进行了 read() 操作, 并且 Channel 本身缓存了数据.
        // 最后一种情况比较特殊, 它发生的场景是某些 Channels 有数据在中间缓冲区中但却无法读取(比如因为内存不足)
        if (hasStagedReceives() || !immediatelyConnectedKeys.isEmpty() || (madeReadProgressLastCall && dataInBuffers))
            timeout = 0;

        // 若之前内存池内存耗尽, 而现在又可用了, 将一些因为内存压力而暂时取消读事件的 Channel 重新注册读事件
        if (!memoryPool.isOutOfMemory() && outOfMemory) {
            log.trace("Broker no longer low on memory - unmuting incoming sockets");
            for (KafkaChannel channel : channels.values()) {
                if (channel.isInMutableState() && !explicitlyMutedChannels.contains(channel)) {
                    channel.unmute();
                }
            }
            outOfMemory = false;
        }

        // 检查 I/O就绪 的keys, 记录 select() 用时
        long startSelect = time.nanoseconds();
        int numReadyKeys = select(timeout);
        long endSelect = time.nanoseconds();
        this.sensors.selectTime.record(endSelect - startSelect, time.milliseconds());

        // 1. 存在 I/O就绪 的Channels; 2和3 参见之前将 timeout = 0 部分的注释
        if (numReadyKeys > 0 || !immediatelyConnectedKeys.isEmpty() || dataInBuffers) {
            Set<SelectionKey> readyKeys = this.nioSelector.selectedKeys();

            // Poll 有缓存数据的 Channels (但不Poll底层socket有缓存数据的Channels)
            if (dataInBuffers) {
                keysWithBufferedRead.removeAll(readyKeys); //so no channel gets polled twice
                Set<SelectionKey> toPoll = keysWithBufferedRead;
                keysWithBufferedRead = new HashSet<>(); //poll() calls will repopulate if needed
                pollSelectionKeys(toPoll, false, endSelect);
            }

            // Poll 底层 socket 有缓存数据的 Channels
            pollSelectionKeys(readyKeys, false, endSelect);
            readyKeys.clear();

            // Poll 待连接的 Channels
            pollSelectionKeys(immediatelyConnectedKeys, true, endSelect);
            immediatelyConnectedKeys.clear();
        } else {
            madeReadProgressLastPoll = true; //no work is also "progress"
        }

        long endIo = time.nanoseconds();
        this.sensors.ioTime.record(endIo - endSelect, time.milliseconds());

        // 利用 select() 结束时刻保证我们不会关闭刚刚传进 pollSelectionKeys() 的连接 (避免将其识别未过期连接)
        maybeCloseOldestConnection(endSelect);

        // 在关闭过期连接后, 将完成接收的 Channels 加入 completedReceives.
        addToCompletedReceives();
    }
```

这部分继续深究的话比较复杂，Kafka在这方面考虑了不少，上述分析中对一些字段也只是简单地提了下，到此为止。总之，最重要的是直到`poll()`会填充`Selector`内部维护的**已完成接收**/**已完成发送**/**已断开**的`Channel`，以便之后处理。

PS：在处理完成的发送时，在调用`send()`向socket写入数据的同时取消监听对应`Channel`的`OP_WRITE`事件：

```scala
    // 类 KafkaChannel
    // 调用链: Selector.PollSelectionKeys() => write() => send()
    private boolean send(Send send) throws IOException {
        send.writeTo(transportLayer);
        if (send.completed())
            transportLayer.removeInterestOps(SelectionKey.OP_WRITE);

        return send.completed();
    }
```

### 4.  processCompletedReceives

```scala
  private def processCompletedReceives() {
    // 遍历所有完成接收的 NetworkService, 具体实现在 selector.poll() 方法中, 最后会调用 addToCompletedReceives()
    // 如果 channel 不在 explicitlyMutedChannels 中 (即调用了unmute()方法), 则会将 channel 对应的 NetworkService 队列
    // 弹出队首并加入 completedReceives 中。
   selector.completedReceives.asScala.foreach { receive =>
      try {
        // NetworkServer 的 source 字段记录了连接channel的 connectionId
        openOrClosingChannel(receive.source) match {
          case Some(channel) =>
            // 解析 payload (接收到的ByteBuffer)的头部
            val header = RequestHeader.parse(receive.payload)
            // 将其与 channel 的会话层信息封装成 RequestContext
            val context = new RequestContext(header, receive.source, channel.socketAddress,
              channel.principal, listenerName, securityProtocol)
            // 进一步将上述信息封装成 Request 对象
            val req = new RequestChannel.Request(processor = id, context = context,
              startTimeNanos = time.nanoseconds, memoryPool, receive.payload, requestChannel.metrics)
            // 这里仅仅是将 req 放入 requestChannel 的内部队列 requestQueue
            requestChannel.sendRequest(req)
            // 取消监听该channel的 OP_READ 事件, 并添加到 explicitlyMutedChannels
            selector.mute(receive.source)
          case None =>
            // 抛出异常(略)
        }
      } catch {
        // 异常处理(略)
      }
    }
  }
```

### 4. processCompleteSends

```scala
  private def processCompletedSends() {
    // 遍历所有完成发送的 NetworkService, 具体实现在 selector.poll() 方法中
    selector.completedSends.asScala.foreach { send =>
      try {
        // 将该网络地址的响应从 inflightResponses 中移除
        val resp = inflightResponses.remove(send.destination).getOrElse {
          throw new IllegalStateException(s"Send for ${send.destination} completed, but not in `inflightResponses`")
        }
        updateRequestMetrics(resp)
        // 将对应的 channel 从 explicitlyMutedChannels 中移除, 并且如果未断开连接, 则注册 OP_READ 事件
        selector.unmute(send.destination)
      } catch {
        // 异常处理(略)
      }
    }
```

### 5. processDisconnected

```scala
  private def processDisconnected() {
    // 遍历所有断开连接的channel的 connectionId, 具体实现在 selector.poll() 方法中
    selector.disconnected.keySet.asScala.foreach { connectionId =>
      try {
        // 从 connectionId 中取得网络地址信息
        val remoteHost = ConnectionId.fromString(connectionId).getOrElse {
          throw new IllegalStateException(s"connectionId has unexpected format: $connectionId")
        }.remoteHost
        // 将断开连接的网络地址的响应从 inflightResponses 中移除
        inflightResponses.remove(connectionId).foreach(updateRequestMetrics)
        // the channel has been closed by the selector but the quotas still need to be updated
        // 更新 quotas 的信息, 即将该网络地址上的连接数减1
        connectionQuotas.dec(InetAddress.getByName(remoteHost))
      } catch {
        // 异常处理(略)
      }
    }
  }
```

## Processor.run()总结

`Processor`使用了Kafka自己实现的`Selector`（别名为`KSelector`），比`Acceptor`使用的NIO默认的`Selector`（别名为`NSelector`）有更多的功能，因为`Processor`要维护监听socket的读/写事件状态，即`OP_READ`和`OP_WRITE`。

一些具体的实现在`org.apache.kafka.common`的`network`包和`request`包中（Java实现），这里暂时不细看。

归结其流程为：

1. 从将`Acceptor`收到的新连接全部注册`OP_READ`事件，因为Kafka服务端不主动向客户端发送请求，只被动响应客户端的请求；
2. 根据响应类型处理缓存的响应：`NoOpAction`=>重新注册`Channel`的读事件，`SendAction`=>注册`Channel`的写事件，将响应缓存，并交由`RequestChannel`发送，`CloseConnectionAction`=>关闭`Channel`；
3. 轮询`Selector`得到就绪的I/O事件（可读/可写/断开）；
4. 对所有完成接收的数据（请求），封装后给`RequestChannel`发送；
5. 对所有完成发送的数据（响应），从缓存中移除，并重新监听对应`Channel`的读事件；
6. 对所有断开的连接，更新`connectionQuotas`维护的网络地址=>连接数的映射。

`Processor`本身只是做完成读/写/断开三种事件的处理，发送和接收实际上都是通过`RequestChannel`。至于`Processor`是由`SocketServer.newProcessor()`方法创建的，其内部的`requestChannel`字段就是`SocketServer`的同名字段。

因此，接下来就是阅读`RequestChannel`。