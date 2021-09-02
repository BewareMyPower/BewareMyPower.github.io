---
title: Pulsar 源码阅读 - 消费者的订阅
date: 2021-09-02 20:58:55
tags:
  - Pulsar
top: 1
---

[toc]

## 前言

之前对 Pulsar 消费端的逻辑不太熟悉，但是一直有印象就是刚接触 Pulsar 时，不记得在哪看到 Pulsar 的消费模型是 push 的，而这点和 Kafka 的 pull 消费模型是完全不同的。之前对于 Kafka 的消费模型已经比较熟悉了，客户端发送 FETCH 请求，其中对于需要拉取（pull）数据的每个 partition，请求中会有一个 `partition_max_bytes` 字段限制该分区获取的最大字节数。而从 FETCH v3 开始，还有个总的 `max_bytes` 字段限制总的最大字节数来针对分区太多的场合。具体协议参见 [Kafka Message Fetch](https://kafka.apache.org/protocol#The_Messages_Fetch)。

那么，Pulsar 采用的 push 消费模型是怎样的呢？为什么要采用 push 消费模型呢？带着这些问题，开始阅读源码，本文采用 Pulsar 2.8.0 的源码（实际是 master 分支），因此和之前的 release 版本可能有些许出入。

> 本文在阅读源码时，会略去一些相对不核心的代码，必须合法性检查/异常处理/错误日志，此时会用 /* ... */ 的风格来略去这一部分，而代码分析则统一使用 // 注释风格。

## 协议

参见 Pulsar 官网文档 [Binary Protocol: Consumer](http://pulsar.apache.org/docs/en/develop-binary-protocol/#consumer)：

![consumer](http://pulsar.apache.org/docs/assets/binary-protocol-consumer.png)

重点是客户端发送 Flow 请求，然后broker 回复消息。重点是流控的处理。这里先看看 `PulsarApi.proto` 中的定义（位于 `org.apache.pulsar.common.api.proto` 包）：

```protobuf
message CommandFlow {
    required uint64 consumer_id       = 1;

    // Max number of messages to prefetch, in addition
    // of any number previously specified
    required uint32 messagePermits     = 2;
}
```

除了消费者 id 外，它只需要一个 permits 参数，表示 prefetch（提前获取）的最大消息数量。

再来看看文档的介绍。典型的消费者实现会在应用程序准备消费之前使用队列积累这些消息，在应用程序队列已经入队了半数以上消息时，消费者发送 permits 给 broker 来请求更多的消息（其数量等于队列大小的一半）。

## Client

首先给出一份最简单的客户端消费的代码：

```java
try (PulsarClient client = PulsarClient.builder()
        .serviceUrl("pulsar://localhost:6650")
        .build()) {
    Consumer<byte[]> consumer = client.newConsumer()
            .topic("my-topic")
            .subscriptionName("my-sub")
            .subscribe();
    Message<byte[]> message = consumer.receive();
    // TODO: handle the message
} catch (PulsarClientException e) {
    e.printStackTrace();
}
```

1. 设置目标主题和订阅名后，调用 `subscribe()` 方法订阅该主题，创建 `Consumer`;
2. consumer 调用 `receive()` 方法接收消息。

> 这里先简单介绍一下，客户端的实现代码位于 `pulsar-client` 模块，而接口定义则位于 `pulsar-client-api` 模块，一些（broker 和客户端等）公用的类位于 `pulsar-common` 模块。而各模块均位于同名目录下。
>
> 另外 Pulsar 所有的同步调用 API 都只是简单等待异步调用 API（方法名后缀是 Async）返回的 CompletableFuture<T> 对象完成，其返回值为 T。

### 消费者的创建

创建 `PulsarClient` 时实际上是创建了 `PulsarClientImpl` 对象，其中 `newConsumer` 方法是创建一个 builder 用于链式调用：

```java
    public ConsumerBuilder<byte[]> newConsumer() {
        // 另外从这里也可以看到默认的 consumer 是使用 bytes schema
        return new ConsumerBuilderImpl<>(this, Schema.BYTES);
    }
```

其中 builder 的方法就不仔细阅读了，主要是对参数进行必要的验证后设置相应字段，比如必要的是主题名和订阅名，都保存在 `conf` 中：

```java
    // ConsumerBuilderImpl<T>
    private ConsumerConfigurationData<T> conf;
```

```java
    // ConsumerConfigurationData<T>
    private Set<String> topicNames = Sets.newTreeSet();
    private String subscriptionName;
```

`ConsumerBuilderImpl#subscribeAsync` 本身只是对 `conf` 的一些参数进行合法性校验，对于只设置主题名和订阅名的情况只是验证这两项配置是否存在，最后实际上是调用 `PulsarClientImpl#subscribeAsync`：

```java
        return interceptorList == null || interceptorList.size() == 0 ?
                client.subscribeAsync(conf, schema, null) : // 默认 intercepto 为 null
                client.subscribeAsync(conf, schema, new ConsumerInterceptors<>(interceptorList));
```

```java
    // PulsarClientImpl<T>
    public <T> CompletableFuture<Consumer<T>> subscribeAsync(ConsumerConfigurationData<T> conf, Schema<T> schema, ConsumerInterceptors<T> interceptors) {
        // 进行一些合法性检查（这里略去代码），包括：
        // 1. client 状态为 Open
        // 2. conf != null
        // 3. conf.topicNames 的每个主题名的格式必须合法
        // 4. 对于 compacted topic，主题必须为 persistent，订阅模式必须为 Exclusive 或者 Failover
        // 5. 对于 ConsumerEventListener，订阅模式必须为 Failover
        /* ... */

        if (conf.getTopicsPattern() != null) {
            // 正则订阅，此时禁止设置具体的主题名字（topicNames）
            if (!conf.getTopicNames().isEmpty()){
                return FutureUtil
                    .failedFuture(new IllegalArgumentException("Topic names list must be null when use topicsPattern"));
            }
            return patternTopicSubscribeAsync(conf, schema, interceptors);
        } else if (conf.getTopicNames().size() == 1) {
            // 单主题订阅
            return singleTopicSubscribeAsync(conf, schema, interceptors);
        } else {
            // 多主题订阅
            return multiTopicSubscribeAsync(conf, schema, interceptors);
        }
    }
```

为求简单，还是只看单主题订阅的情况：

```java
    private <T> CompletableFuture<Consumer<T>> singleTopicSubscribeAsync(ConsumerConfigurationData<T> conf, Schema<T> schema, ConsumerInterceptors<T> interceptors) {
        return preProcessSchemaBeforeSubscribe(this, schema, conf.getSingleTopic())
            .thenCompose(schemaClone -> doSingleTopicSubscribeAsync(conf, schemaClone, interceptors));
    }
```

这里的先后顺序是：

1. 调用 `preProcessSchemaBeforeSubscribe`，此时会对 schema 进行预处理，必须注册 schema。
2. 对前一步得到的 schemaClone 传入 `doSingleTopicSubscribeAsync`。

这里关注第二步：

```java
    private <T> CompletableFuture<Consumer<T>> doSingleTopicSubscribeAsync(ConsumerConfigurationData<T> conf, Schema<T> schema, ConsumerInterceptors<T> interceptors) {
        CompletableFuture<Consumer<T>> consumerSubscribedFuture = new CompletableFuture<>();

        String topic = conf.getSingleTopic();

        // 1. 首先取得 topic 的 metadata（目前仅包含 partitions 字段表示分区数量）
        getPartitionedTopicMetadata(topic).thenAccept(metadata -> {
            /* debug 日志 ... */
            ConsumerBase<T> consumer;
            // 从 executor provider（包含一组 executors）中分配一个 executor
            ExecutorService listenerThread = externalExecutorProvider.getExecutor();
            if (metadata.partitions > 0) {
                // 2.1 多分区订阅，创建的是 MultiTopicsConsumerImpl
                consumer = MultiTopicsConsumerImpl.createPartitionedConsumer(PulsarClientImpl.this, conf,
                    listenerThread, consumerSubscribedFuture, metadata.partitions, schema, interceptors);
            } else {
                // 2.2 单分区订阅，创建的是 ConsumerImpl
                int partitionIndex = TopicName.getPartitionIndex(topic);
                consumer = ConsumerImpl.newConsumerImpl(PulsarClientImpl.this, topic, conf, listenerThread, partitionIndex, false,
                        consumerSubscribedFuture,null, schema, interceptors,
                        true /* createTopicIfDoesNotExist */);
            }

            consumers.add(consumer); // 将创建的 consumer 加入到 client 内部的 consumers 中
        }).exceptionally(/* 异常处理... */);

        return consumerSubscribedFuture;
    }
```

> 多分区订阅和多主题订阅本质上是一样的，都是使用 `MultiTopicsConsumerImpl` 管理多个主题（因为 Pulsar 中分区只不过是一个包含后缀 `-partition-<n>` 的主题）。

这里还是关注单分区订阅，`ConsumerImpl.newConsumerImpl(...)` 只是将参数原封不动传给其构造方法，构造方法包括了 consumer 内部一些字段的初始化，因此比较长，我们还是只关注重点，那就是最简洁的订阅会和 broker 有什么交互。实际上这部分逻辑在构造方法最后，调用 `grabCnx` 方法：

```java
    void grabCnx() {
        this.connectionHandler.grabCnx();
    }
```

### 连接的建立

`connectionHandler`（下文简称 connection）的构造：

```java
this.connectionHandler = new ConnectionHandler(this,
		        new BackoffBuilder()
                        // 设置 BackOff 所需要的参数，参数对应 client 的配置为：
                        // initialTime: 默认 100 ms，ClientBuilder#startingBackoffInternal
                        // max: 默认 60 s，ClientBuilder#maxBackoffInterval
                        // mandatoryStop：固定为 0 ms
                        /* ... */
                        .create(),
        this); // 将 ConsumerImpl 对象自身传入 ConnectionHandler 的构造方法
```

这里简单说下 `BackOff` 对象。查看 backoff 在 `ConnectionHandler` 中的使用，可以看到主要是在重连活着关闭连接时用 `next()` 方法来得到建立连接时对应的 timeout，因为连接对应的是 client，所以这里用的都是 client 的配置。

回到正题，继续看 `ConnectionHandler#grabCnx` 的实现：

```java
    protected void grabCnx() {
        if (CLIENT_CNX_UPDATER.get(this) != null) {
            // 已经连接成功了，无视这次调用
            /* warn 日志... */
            return;
        }

        if (!isValidStateForReconnection()) {
            // 若 connection 状态不可用于重连（比如为 Closed），则无视这次调用
            /* info 日志... */
            return;
        }

        try {
            // 1. 取得主题对应的连接
            state.client.getConnection(state.topic) //
                    // 2.1 若连接成功，则调用 connectionOpened
                    .thenAccept(cnx -> connection.connectionOpened(cnx)) //
                    // 2.2 若连接失败，则调用 handleConnectionError，会进行重连操作
                    .exceptionally(this::handleConnectionError);
        } catch (Throwable t) {
            /* warn 日志... */
            reconnectLater(t);
        }
    }
```

这里还是看看连接成功的处理。注意到 `ConsumerImpl` 是实现了 `Connection` 接口的：

```java
public class ConsumerImpl<T> extends ConsumerBase<T> implements ConnectionHandler.Connection {
```

然后注意到构造 connection 时用的 `Connection` 接口就是 `ConsumerImpl` 对象自己，因此调用的是 `ConsumerImpl#connectionOpened`。

实际上这一小节的内容同样也适用于生产者以及多主题消费者，它们对应的类都实现了 `Connection` 接口，只需要实现各自的回调即可。

### 连接成功的回调

```java
    public void connectionOpened(final ClientCnx cnx) {
        if (getState() == State.Closing || getState() == State.Closed) {
            /* 执行一些清理工作... */
            return;
        }
        // 绑定 client 到 connection，前面 grabCnx() 检查的 CLIENT_CNX_UPDATER 也会在这里设置
        setClientCnx(cnx);

        /* info 日志表示准备订阅对应主题... */

        long requestId = client.newRequestId();

        int currentSize;
        synchronized (this) {
            // incomingMessages 为客户端缓存收到消息的队列（接收队列），这里先取得其大小
            currentSize = incomingMessages.size();
            // 清空接收队列，取得第一条消息的 id（也就是消费者第一条没有传给应用程序的消息的 id）
            startMessageId = clearReceiverQueue();
            // 清空 DLQ
            if (possibleSendToDeadLetterTopicMessages != null) {
                possibleSendToDeadLetterTopicMessages.clear();
            }
        }

        boolean isDurable = subscriptionMode == SubscriptionMode.Durable;
        MessageIdData startMessageIdData = null;
        if (isDurable) {
            // 对持久化订阅，那么将 startMessageIdData 置为 null，因为 broker 负责告诉客户端重新开始消费的消息 id
            startMessageIdData = null;
        } else if (startMessageId != null) {
            // 对非持久化订阅（常用于 Reader API），则要用我们之前取得的第一条消息的 id
            MessageIdData.Builder builder = MessageIdData.newBuilder();
            builder.setLedgerId(startMessageId.getLedgerId());
            builder.setEntryId(startMessageId.getEntryId());
            if (startMessageId instanceof BatchMessageIdImpl) {
                builder.setBatchIndex(startMessageId.getBatchIndex());
            }

            startMessageIdData = builder.build();
            builder.recycle();
        } // else: 非持久化订阅，但是缓存队列里没有消息

        // 取得 schema
        SchemaInfo si = schema.getSchemaInfo();
        if (si != null && (SchemaType.BYTES == si.getType() || SchemaType.NONE == si.getType())) {
            // don't set schema for Schema.BYTES
            si = null;
        }

        // 构造 CommandSubscribe 并发送，具体字段就不贴出来了
        ByteBuf request = Commands.newSubscribe(/* ... */);
        cnx.sendRequestWithId(request, requestId).thenRun(() -> {
            // case 1 发送成功
            synchronized (ConsumerImpl.this) {
                // 如果 State 是 Uninitialized，Connecting，RegisteringSchema，则将其改为 Ready
                if (changeToReadyState()) {
                    // 将 availablePermits 置为 0
                    consumerIsReconnectedToBroker(cnx, currentSize);
                } else {
                    // 其它 state 比如 Failed 则代表异常状态，此时将 state 置为 Closed
                    setState(State.Closed);
                    deregisterFromClientCnx(); // 将 consumer 和 connection 解绑
                    client.cleanupConsumer(this); // 将 consumer 从 client 中移除
                    cnx.channel().close(); // 关闭连接
                    return;
                }
            }

            // 重制 BackOff，因为之前可能重连过导致 BackOff 内部状态发生变化，
            resetBackoff();

            // 因为 subscribeFuture 可能多次 complete，因此这里判断是否为第一次 complete
            boolean firstTimeConnect = subscribeFuture.complete(this);
            // 如果 receiverQueueSize（内部接收队列的大小）大于 0，则代表 consumer 有缓冲区可以接收消息。
            // 此时，会将接收队列大小的一半作为 permits 发送 Flow 请求
            // 但还有一种情况也不会发送，也就是满足以下三个条件：
            // 1. 第一次连接完成
            // 2. 当前 consumer 是一个子 consumer
            // 3. 订阅类型是持久化（NOTE：这个条件可能是早期错误，见 https://github.com/apache/pulsar/pull/3960）
            // 因为当 consumer 是子 consumer 的时候，它归属于 MultiTopicsConsumerImpl，也就是多分区/主题的 consumer
            // MultiTopicsConsumerImpl 会在所有子 consumer 连接完成后再调用 startReceivingMessages
            // 从而对每个子 consumer 调用 increaseAvailablePermits
            if (!(firstTimeConnect && hasParentConsumer && isDurable) && conf.getReceiverQueueSize() != 0) {
                increaseAvailablePermits(cnx, conf.getReceiverQueueSize());
            }
        }).exceptionally((e) -> {/* 异常处理... */});
```

可以看到连接成功后，主要是发送 CommandSubscribe（订阅命令），broker 处理成功后，consumer 就处于 Ready 状态，并且会发送 Flow 请求携带 permits 为接收队列大小的一半。

```java
    protected void increaseAvailablePermits(ClientCnx currentCnx, int delta) {
        // availablePermits 增加 delta，代表可用的接收队列的数量
        int available = AVAILABLE_PERMITS_UPDATER.addAndGet(this, delta);

        // 如果超过了阈值（默认一半接收队列大小）且没有调用 pause，则将其清零，并将清零前的值构造 Flow 命令发送
        while (available >= receiverQueueRefillThreshold && !paused) {
            if (AVAILABLE_PERMITS_UPDATER.compareAndSet(this, available, 0)) {
                sendFlowPermitsToBroker(currentCnx, available);
                break;
            } else {
                available = AVAILABLE_PERMITS_UPDATER.get(this);
            }
        }
    }
```

再看看 Flow 命令的定义和发送：

```protobuf
message CommandFlow {
    required uint64 consumer_id       = 1;

    // 预先拉取的消息数量，也称为 permits
    required uint32 messagePermits     = 2;
}
```

```java
    private void sendFlowPermitsToBroker(ClientCnx cnx, int numMessages) {
        // 仅当连接未断开且消息数大于 0 才会发送 Flow 请求，因为 broker 收到 permits 为 0 的请求会直接抛出异常
        if (cnx != null && numMessages > 0) {
            if (log.isDebugEnabled()) {
                /* ... */
            } else {
                cnx.ctx().writeAndFlush(Commands.newFlow(consumerId, numMessages), cnx.ctx().voidPromise());
            }
        }
    }
```

## Broker 处理

### CommandSubscribe 处理

broker 对 TCP 协议的处理位于 `org.apache.pulsar.broker.service` 包的 `ServerCnx` 类。对于 consumer 而言，比较独有的就是 `CommandSubscribe` 和 `CommandFlow`。

因为代码比较长，所以日志相关代码均略过，不特意用 `/* xxx... */` 说明。

```java
    protected void handleSubscribe(final CommandSubscribe subscribe) {
        /* 略去相关字段的解析... */

        // 验证是否具有 CONSUME 权限
        CompletableFuture<Boolean> isAuthorizedFuture = isTopicOperationAllowed(
                topicName,
                subscriptionName,
                TopicOperation.CONSUME
        );
        isAuthorizedFuture.thenApply(isAuthorized -> {
            if (isAuthorized) {
                /* 验证 metadata 字段...*/
              
                // 这里的 Consumer 是 broker consumer，对应每个 client consumer
                CompletableFuture<Consumer> consumerFuture = new CompletableFuture<>();
                CompletableFuture<Consumer> existingConsumerFuture = consumers.putIfAbsent(consumerId,
                        consumerFuture);

                if (existingConsumerFuture != null) { // 已经创建过 broker consumer
                    if (existingConsumerFuture.isDone() && !existingConsumerFuture.isCompletedExceptionally()) {
                        // 创建完成则直接发送成功的响应
                        commandSender.sendSuccessResponse(requestId);
                        return null;
                    } else {
                        // 之前已经有相同 consumerId 的 subscribe 请求，这是因为 client timeout 小于 broker timeout，
                        // 因此 client 发生重试，此时需要等待之前的 consumer future 完成
                        ServerError error = null;
                        if (!existingConsumerFuture.isDone()) {
                            // 前一个 subscribe 请求还未完成，直接返回 ServiceNotReady
                            error = ServerError.ServiceNotReady;
                        } else {
                            // 前一个 subscribe 请求异常完成，则返回同样的错误码并将其移除 cache 避免下次重新进入此分支
                            error = getErrorCode(existingConsumerFuture);
                            consumers.remove(consumerId, existingConsumerFuture);
                        }
                        commandSender.sendErrorResponse(requestId, error,
                                "Consumer is already present on the connection");
                        return null;
                    }
                }

                // 判断是否自动创建 topic，forceTopicCreation 为 client 填充的字段（NOTE：实际上永远为 true）
                // service 对象则是通过配置或者 system topic 的配置来判断是否允许自动创建 topic
                boolean createTopicIfDoesNotExist = forceTopicCreation
                        && service.isAllowAutoTopicCreation(topicName.toString());

                // 当前 broker 获取或创建 Topic 对象
                service.getTopic(topicName.toString(), createTopicIfDoesNotExist)
                        .thenCompose(optTopic -> {
                            if (!optTopic.isPresent()) {
                                return FutureUtil
                                        .failedFuture(new TopicNotFoundException(
                                                "Topic " + topicName + " does not exist"));
                            }

                            Topic topic = optTopic.get();

                            // 对于 durable cursor 而言，如果该订阅不存在且不允许订阅自动创建，subscribe 会失败
                            boolean rejectSubscriptionIfDoesNotExist = isDurable
                                && !service.isAllowAutoSubscriptionCreation(topicName.toString())
                                && !topic.getSubscriptions().containsKey(subscriptionName);

                            if (rejectSubscriptionIfDoesNotExist) {
                                return FutureUtil
                                        .failedFuture(
                                                new SubscriptionNotFoundException(
                                                        "Subscription does not exist"));
                            }

                            // 若带有 schema 则先检查 schema 兼容性，最后都会调用 Topic#subscribe
                            if (schema != null) {
                                return topic.addSchemaIfIdleOrCheckCompatible(schema)
                                        .thenCompose(v -> topic.subscribe(/* ... */));
                            } else {
                                return topic.subscribe(/* ... */);
                            }
                        })
                        .thenAccept(consumer -> {
                            if (consumerFuture.complete(consumer)) {
                                commandSender.sendSuccessResponse(requestId);
                            } else {
                                // 如果 consumerFuture 已经完成，则当前 consumer 是 client timeout 重新创建的 consumer
                                // 此时需要关闭 consumer 并移除这个 future
                                try {
                                    consumer.close();
                                    log.info("[{}] Cleared consumer created after timeout on client side {}",
                                            remoteAddress, consumer);
                                } catch (BrokerServiceException e) {
                                    log.warn(
                                            "[{}] Error closing consumer created"
                                                    + " after timeout on client side {}: {}",
                                            remoteAddress, consumer, e.getMessage());
                                }
                                consumers.remove(consumerId, consumerFuture);
                            }

                        })
                        .exceptionally(exception -> {
                            /* 根据异常严重性打印对应等级的日志... */

                            // 返回错误码移除订阅失败过程中添加的 consumer
                            if (consumerFuture.completeExceptionally(exception)) {
                                commandSender.sendErrorResponse(requestId,
                                        BrokerServiceException.getClientErrorCode(exception),
                                        exception.getCause().getMessage());
                            }
                            consumers.remove(consumerId, consumerFuture);
                            return null;

                        });
            } else { // 鉴权失败
                String msg = "Client is not authorized to subscribe";
                ctx.writeAndFlush(Commands.newError(requestId, ServerError.AuthorizationError, msg));
            }
            return null;
        }).exceptionally(ex -> { // 鉴权抛出异常
            commandSender.sendErrorResponse(requestId, ServerError.AuthorizationError, ex.getMessage());
            return null;
        });
    }
```

总结下来，处理 subscribe 请求核心调用是：

- `BrokerService#getTopic`：获取当前 broker 所拥有（own）的 `Topic` 对象
- `Topic#subscribe`：在 `Topic` 对象中创建对应的订阅，并得到 `Consumer` 对象

其中 `Topic` 和 `Consumer` 是 broker 端对 topic 和 consumer 的抽象，负责管理对应的资源。均位于 `org.apache.pulsar.broker.service` 包下。这里我们重点看 `PersistentTopic#subscribe`。

### PersistentTopic#subscribe

```java
    public CompletableFuture<Consumer> subscribe(/* ... */) {
        // 只有 Failover 和 Exclusive 模式才支持读取 compacted topic
        if (readCompacted && !(subType == SubType.Failover || subType == SubType.Exclusive)) {
            return FutureUtil.failedFuture(new NotAllowedException(
                    "readCompacted only allowed on failover or exclusive subscriptions"));
        }

        // 通过 NamespaceService 检查 topic owner 是否为当前 broker，若不是则该 future 会以 ServiceUnitNotReady 异常完成
        return brokerService.checkTopicNsOwnership(getName()).thenCompose(__ -> {
            /* 进行一系列检查... （这里略去具体代码，仅用注释说明） */
            // 1. 若 broker 未启用订阅复制，则仅仅是打印 warn 日志，防止 broker 禁止跨地域复制后 consumer 订阅失败
            // 2. 检查 broker 是否支持 Key_Shared 订阅模式
            // 3. 检查非 system topic（以 __change_events 结尾）的 topic 级别配置是否支持该订阅类型
            // 4. 检查订阅名是否为空
            // 5. 检查协议是否支持 batch 消息
            // 6. 禁止对前缀是跨地域复制的前缀（默认 pulsar.repl）或者 pulsar.dedup 创建订阅

            // 用连接地址作为 key，检查 consumer 对应的 RateLimiter 是否存在，限制重连次数，因为重连的连接地址是相同的
            // 参考 https://github.com/apache/pulsar/pull/2977
            if (cnx.clientAddress() != null && cnx.clientAddress().toString().contains(":")) {
                SubscribeRateLimiter.ConsumerIdentifier consumer = new SubscribeRateLimiter.ConsumerIdentifier(
                        cnx.clientAddress().toString().split(":")[0], consumerName, consumerId);
                if (subscribeRateLimiter.isPresent() && (!subscribeRateLimiter.get().subscribeAvailable(consumer)
                        || !subscribeRateLimiter.get().tryAcquire(consumer))) {
                    /* warn 日志... */
                    return FutureUtil.failedFuture(
                            new NotAllowedException("Subscribe limited by subscribe rate limit per consumer."));
                }
            }

            lock.readLock().lock();
            try {
                // 当 topic 被删除或关闭，或者 producer 写消息失败时都会标记为 fence 状态，此时禁止订阅
                if (isFenced) {
                    log.warn("[{}] Attempting to subscribe to a fenced topic", topic);
                    return FutureUtil.failedFuture(new TopicFencedException("Topic is temporarily unavailable"));
                }
                // 实际上是增加 usageCount（连接的 producer 和 consumer 总数），传参数是为了打印 debug 日志
                handleConsumerAdded(subscriptionName, consumerName);
            } finally {
                lock.readLock().unlock();
            }

            // 获取（或创建）订阅
            CompletableFuture<? extends Subscription> subscriptionFuture = isDurable ? //
                    getDurableSubscription(subscriptionName, initialPosition, startMessageRollbackDurationSec,
                            replicatedSubscriptionState)
                    : getNonDurableSubscription(subscriptionName, startMessageId, initialPosition,
                    startMessageRollbackDurationSec);

            // 对于持久化订阅，可以获取最大的未确认的消息
            int maxUnackedMessages = isDurable
                    ? getMaxUnackedMessagesOnConsumer()
                    : 0;

            CompletableFuture<Consumer> future = subscriptionFuture.thenCompose(subscription -> {
                // 创建 Consumer 对象并加入到对应的订阅中
                Consumer consumer = new Consumer(/* ... */);
                return addConsumerToSubscription(subscription, consumer).thenCompose(v -> {
                    checkBackloggedCursors();
                    if (!cnx.isActive()) {
                        // 连接已经断开，则需要关闭 consumer
                        try {
                            consumer.close();
                        } catch (BrokerServiceException e) {
                            /* ... */
                            // 减少 usageCount，对应 handleConsumerAdded
                            decrementUsageCount(); 
                            return FutureUtil.failedFuture(e);
                        }

                        decrementUsageCount();
                        return FutureUtil.failedFuture(
                                new BrokerServiceException("Connection was closed while the opening the cursor "));
                    } else {
                        // 连接仍存活，则继续检查复制订阅的状态，至此完成整个订阅
                        checkReplicatedSubscriptionControllerState();
                        return CompletableFuture.completedFuture(consumer);
                    }
                });
            });

            future.exceptionally(ex -> {
                decrementUsageCount();
                /* 打印日志... */
                return null;
            });
            return future;
        });
    }
```

核心流程：

1. 获取 `Subscription`（broker 对订阅的抽象），若不存在则创建。
2. 创建 `Consumer`（broker 对消费者的抽象）后加入订阅。

这里不进一步阅读 `Subscription` 代码。简单说，在创建订阅时，对于持久化（durable）订阅（`PersistentSubscription`），创建时会打开一个 cursor，对应于 `PersistentTopic` 内部的 ledger，若 cursor 不存在则创建；对于非持久化（non-durable）订阅（`NonPersistentSubscription`），则是直接内存中维护消息 id（`MessageIdImpl`）。两者最大的区别是，持久化订阅在 cursor 已存在时，直接打开 cursor，而无视掉 CommandSubscribe 中的消息 id。

至于将 consumer 加入订阅，实际上是根据订阅类型创建对应的 dispatcher（`Dispatcher`），这里以默认的 Exclusive 订阅为例：

```java
                    switch (consumer.subType()) {
                        case Exclusive:
                            if (dispatcher == null || dispatcher.getType() != SubType.Exclusive) {
                                previousDispatcher = dispatcher;
                                dispatcher = useStreamingDispatcher
                                        ? new PersistentStreamingDispatcherSingleActiveConsumer(
                                                cursor, SubType.Exclusive, 0, topic, this)
                                        : new PersistentDispatcherSingleActiveConsumer(
                                                cursor, SubType.Exclusive, 0, topic, this);
                            }
                            break;
```

> 注：https://github.com/apache/pulsar/pull/9056 引入了 streaming dispatcher。

然后将 consumer 加入 dispatcher：

```java
                try {
                    dispatcher.addConsumer(consumer);
                    return CompletableFuture.completedFuture(null);
                } catch (BrokerServiceException brokerServiceException) {
                    return FutureUtil.failedFuture(brokerServiceException);
                }
```

订阅里实际负责消息分发的就是 dispatcher。

### CommandFlow

首先是 `ServerCnx` 第一步处理：

```java
    protected void handleFlow(CommandFlow flow) {
        checkArgument(state == State.Connected); // 必须是 Connected 状态
        CompletableFuture<Consumer> consumerFuture = consumers.get(flow.getConsumerId());

        // 如果 consumer 已经创建成功，则直接交给 Consumer 处理
        if (consumerFuture != null && consumerFuture.isDone() && !consumerFuture.isCompletedExceptionally()) {
            Consumer consumer = consumerFuture.getNow(null);
            if (consumer != null) {
                consumer.flowPermits(flow.getMessagePermits());
            }
        }
    }
```

然后递交给 `Consumer`：

```java
    public void flowPermits(int additionalNumberOfMessages) {
        checkArgument(additionalNumberOfMessages > 0); // permits 必须大于 0

        // 1. 对于只支持 individual ack 的订阅类型，检查没有确认的（unacked）消息数量
        if (shouldBlockConsumerOnUnackMsgs() && unackedMessages >= maxUnackedMessages) {
            blockedConsumerOnUnackedMsgs = true;
        }
        int oldPermits;
        if (!blockedConsumerOnUnackedMsgs) {
            // 1.1 正常情况下，增加 messagePermits，然后交给 subscription 处理
            oldPermits = MESSAGE_PERMITS_UPDATER.getAndAdd(this, additionalNumberOfMessages);
            subscription.consumerFlow(this, additionalNumberOfMessages);
        } else {
            // 1.2 如果 unacked 消息数量超过限制，则更新 permitsReceivedWhileConsumerBlocked 记录因此阻塞的 flow permits
            oldPermits = PERMITS_RECEIVED_WHILE_CONSUMER_BLOCKED_UPDATER.getAndAdd(this, additionalNumberOfMessages);
        }
    } 
```

可见，更新完 `Consumer` 内部的 `messagePermits` 后，交由对应的 `Subscription` 对象处理。这里仅看 `PersistentSubscription` 的实现：

```java
    public void consumerFlow(Consumer consumer, int additionalNumberOfMessages) {
        this.lastConsumedFlowTimestamp = System.currentTimeMillis();
        dispatcher.consumerFlow(consumer, additionalNumberOfMessages);
    }
```

仅仅是传递给 dispatcher。因此实际的处理是由 dispatcher 完成的。至于 dispatcher 的实现就留给下一篇进行讨论。

## 总结

本文梳理了 client 和 broker 创建消费者订阅某个 topic 的流程。Client 端的实现包含了一些和生产者通用的部分，也就是建立连接的部分。连接成功后的回调，消费者会先发一个 subscribe 命令注册自己，注册成功后发送 flow 请求，携带 permits，其值为内部缓冲区大小的一半（对于无缓冲区的 zero queue consumer 例外）。

Broker 端处理相对复杂，但本质是对一些概念进行了抽象。消费者对应 `Consumer`，每个消费者则包含一个订阅 `Subscription`。而 topic 对应 `Topic`，消费者发送 subscribe 命令时会在 `Topic` 里创建 `Subscription`，因为每个 topic 可以对应多个订阅，创建完成后根据消费者的订阅类型创建对应的 `Dispatcher`。通常意义上一个订阅可以对应多个消费者，但实际上真正维护这组消费者的不是 `Subscription` 而是 `Dispatcher`，订阅本身更多的是维护 cursor（消费进度），比如持久化订阅（`PersistentSubscription`）就会创建一个 durable cursor。而 client 端的 Flow 请求，最终也是 `Dispatcher` 来处理的。

最后解答最初提出的问题，push 和 pull 的区别，以及为什么 Pulsar 是 push 消费模型。Kafka 的消费者是发送 FETCH 请求给 broker，然后 broker 对应的 FETCH 响应里包含读取的消息。而 Pulsar 的消费者发送的是 Flow 请求，它本身是没有对应的 FETCH 响应的。Client 会主动告知 broker 自己可以缓存多少条消息，broker 根据这个提示可以灵活定制 dispatcher，然后主动发消息给 client。因此 Pulsar 的这种 push 模型，实际上是由服务端（broker）来进行流量控制。两者本质区别就是流控到底是 client 还是 server 处理的。
