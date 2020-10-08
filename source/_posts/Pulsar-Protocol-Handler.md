---
title: Pulsar Protocol Handler
date: 2020-10-08 23:49:23
tags:
  - Pulsar
top: 1
---

[toc]

## 概述

本文主要目的是阅读 [Pulsar](https://github.com/apache/pulsar) protocol handler（比如 KoP，AoP，MoP）在 broker 中如何运作的，protocol handler（下文简称 **handler**）对应的是包 `org.apache.pulsar.broker.protocol`（下文将**略去包括 `broker`  之前的包前缀**）的接口 `ProtocolHandler`，只要实现了该接口，并打包成 `*.nar` 后缀以供 broker 加载，即相当于实现了一个 handler。

## handler 初始化

`ProtocolHandler` 本身位于 `protocol` 包下，注意到初始化方法 `initialize`，找到它的使用，在 `ProtocolHandlerWithClassLoader` 类的同名方法被调用，简单看下代码实现：

```java
@Slf4j
@Data
@RequiredArgsConstructor
class ProtocolHandlerWithClassLoader implements ProtocolHandler {

    private final ProtocolHandler handler;
    private final NarClassLoader classLoader;

    @Override
    public void initialize(ServiceConfiguration conf) throws Exception {
        handler.initialize(conf);
    }
    /* 其他方法都是 override handler 的同名方法，比如 xxx()，都是直接调用 handler.xxx() */

    @Override
    public void close() {
        handler.close();
        try {
            classLoader.close();
        } catch (IOException e) {
            log.warn("Failed to close the protocol handler class loader", e);
        }
    }
}
```

仅仅是在构造时多传入了一个 `NarClassLoader` 对象，并且在 override `close()` 方法时关闭这个对象，其他方法都是直接调用 handler 的同名方法。

继续查找 `initialize()` 方法的调用，又被 `ProtocolHandlers` 类的同名方法调用：

```java
    public void initialize(ServiceConfiguration conf) throws Exception {
        for (ProtocolHandler handler : handlers.values()) {
            handler.initialize(conf);
        }
    }

    private final Map<String, ProtocolHandlerWithClassLoader> handlers;

    ProtocolHandlers(Map<String, ProtocolHandlerWithClassLoader> handlers) {
        this.handlers = handlers;
    }
```

可见 `ProtocolHandlers` 维护了一系列 `ProtocolHandlerWithClassLoader`，并在构造时传入。而它的构造则在静态方法 `load` 中：

```java
    public static ProtocolHandlers load(ServiceConfiguration conf) throws IOException {
        // 1. 在配置 protocolHandlerDirectory 对应的目录下查找所有的 handler definitions，并暂时解压到配置 narExtractionDirectory 目录下
        ProtocolHandlerDefinitions definitions =
            ProtocolHandlerUtils.searchForHandlers(conf.getProtocolHandlerDirectory(), conf.getNarExtractionDirectory());

        ImmutableMap.Builder<String, ProtocolHandlerWithClassLoader> handlersBuilder = ImmutableMap.builder();

        // 2. 遍历配置 messagingProtocols 列表的每个 protocol
        conf.getMessagingProtocols().forEach(protocol -> {

            // 2.1 取得 protocol 名字对应的 definition
            ProtocolHandlerMetadata definition = definitions.handlers().get(protocol);
            if (null == definition) {
                /* 抛出异常表示 protocol handler is found */
            }

            ProtocolHandlerWithClassLoader handler;
            try {
                // 2.2 加载 definition 对应的 handler
                handler = ProtocolHandlerUtils.load(definition, conf.getNarExtractionDirectory());
            } catch (IOException e) {
                /* 记录错误日志并抛出异常表示 Failed to load the protocol handler */
            }

            // 2.3 通过 handler 的 accept 方法判断 protocol 是否可接受
            if (!handler.accept(protocol)) {
                /* 关闭 handler，记录错误日志并抛出异常表示 Malformed protocol handler found */
            }

            // 2.4 将 protocol 和 handler 作为 key-value 加入 map
            handlersBuilder.put(protocol, handler);
            log.info("Successfully loaded protocol handler for protocol `{}`", protocol);
        });

        return new ProtocolHandlers(handlersBuilder.build());
    }
```

上面代码注释给出了初始化流程，比如修改配置（`conf/broker.conf` 或 `conf/standalone.conf`）：

```properties
# 默认目录就是 ./protocols
protocolHandlerDirectory=./protocols
messagingProtocols=kafka
```

就会在 `./protocols` 目录下面查找所有 handler definition，然后找到协议名 `kafka` 对应的 definition，期间会调用 handler 的 `initialize` 方法进行初始化。初始化完成后还要调用 handler 的 `accept` 方法判断是否被接受。最后和 `kafka` 组成键值对交由 `Protocols` 类管理。

这里回顾下 handler 的这两个接口（略去 javadoc 注释）：

```java
    boolean accept(String protocol);

    void initialize(ServiceConfiguration conf) throws Exception;
```

## definition

在上一节，引入了 handler definition 的概念，这里看看其实现。首先是取得所有 definition 的 `searchForHandlers` 方法：

```java
public static ProtocolHandlerDefinitions searchForHandlers(String handlersDirectory,
                                                           String narExtractionDirectory) throws IOException {
    Path path = Paths.get(handlersDirectory).toAbsolutePath();
    log.info("Searching for protocol handlers in {}", path);

    ProtocolHandlerDefinitions handlers = new ProtocolHandlerDefinitions();
    if (!path.toFile().exists()) {
        /* warn 日志提示目录不存在 */
        return handlers;
    }

    try (DirectoryStream<Path> stream = Files.newDirectoryStream(path, "*.nar")) {
        for (Path archive : stream) {
            try {
                // 1. 从 nar 包中取得 definition
                ProtocolHandlerDefinition phDef =
                    ProtocolHandlerUtils.getProtocolHandlerDefinition(archive.toString(), narExtractionDirectory);
                log.info("Found protocol handler from {} : {}", archive, phDef);

                checkArgument(StringUtils.isNotBlank(phDef.getName()));
                checkArgument(StringUtils.isNotBlank(phDef.getHandlerClass()));

                // 2. 将 definition 和 nar 包路径组成 metadata
                ProtocolHandlerMetadata metadata = new ProtocolHandlerMetadata();
                metadata.setDefinition(phDef);
                metadata.setArchivePath(archive);

                // 3. 将 definition name 作为 key 加入返回的 map
                handlers.handlers().put(phDef.getName(), metadata);
            } catch (Throwable t) {
                /* warn 日志提示加载失败 */
            }
        }
    }

    return handlers;
}
```

可以看到会从 `protocolHandlerDirectory` 所在目录下面找到所有 `*.nar` 后缀的文件，然后取得 definition：

```java
    public static ProtocolHandlerDefinition getProtocolHandlerDefinition(String narPath, String narExtractionDirectory) throws IOException {
        // 1. 解压 *.nar 包到 narExtractionDirectory，并从解压目录构造 NarClassLoader
        try (NarClassLoader ncl = NarClassLoader.getFromArchive(new File(narPath), Collections.emptySet(), narExtractionDirectory)) {
            // 2. 从 NarClassLoader 中取得 definition
            return getProtocolHandlerDefinition(ncl);
        }
    }

    private static ProtocolHandlerDefinition getProtocolHandlerDefinition(NarClassLoader ncl) throws IOException {
        // 取得 META-INF/services/pulsar-protocol-handler.yml 的配置内容，以 KoP 为例：
        //   name: kafka
        //   description: Kafka Protocol Handler
        //   handlerClass: io.streamnative.pulsar.handlers.kop.KafkaProtocolHandler
        // 也就是 ProtocolHandlerDefinition 的 3 个字段
        String configStr = ncl.getServiceDefinition(PULSAR_PROTOCOL_HANDLER_DEFINITION_FILE);

        // 解析 YAML 配置，从中得到 definition 的 class，也就是将配置文件的三个字段填充进来
        return ObjectMapperFactory.getThreadLocalYaml().readValue(
            configStr, ProtocolHandlerDefinition.class
        );
    }
```

至此，我们知道了 definition 其实就是从 nar 包中解析 `pulsar-protocol-handler.yml` 得到对应的三个字段（都是 `String` 类型）：

- `name`：协议名，比如 KoP 的协议名为 kafka
- `description`：handler 的描述信息
- `handlerClass`：handler 的主类

回顾前一节的 `load()` 方法注释 2.3，definition 的协议名是返回的 definition map 的 key，因此可以通过用户配置的 `messagingProtocols` 来找到对应的 definition，从而找到对应的 handler：

```java
// 这里的 definition 变量其实是 definition + handler 解压目录的路径
ProtocolHandlerMetadata definition = definitions.handlers().get(protocol);
```

## handler 的启动

前文提到了 `ProtocolHandlers#load` 从配置文件中找到 nar 包解压后并加载得到 handlers，而 `load` 方法在 `PulsarService#start` 中被调用，并且对加载的 handlers 进行其他处理：

```java
            protocolHandlers = ProtocolHandlers.load(config);
            protocolHandlers.initialize(config);
            /* 其他初始化及启动流程（略） */
            this.protocolHandlers.start(brokerService);
            // 第一个 key 为协议名，第二个 key 为 handler 绑定的地址
            Map<String, Map<InetSocketAddress, ChannelInitializer<SocketChannel>>> protocolHandlerChannelInitializers =
                this.protocolHandlers.newChannelInitializers();
            this.brokerService.startProtocolHandlers(protocolHandlerChannelInitializers);

            state = State.Started; // 加载完 handlers 后 broker 的状态才改为 Started
```

最后一步涉及到了 `BrokerService#startProtocolHandlers` 会启动 handler 创建的 `ChannelInitializer<SocketChannel>`，其实现为：

```java
    public void startProtocolHandlers(
        Map<String, Map<InetSocketAddress, ChannelInitializer<SocketChannel>>> protocolHandlers) {

        protocolHandlers.forEach((protocol, initializers) -> {
            initializers.forEach((address, initializer) -> {
                try {
                    startProtocolHandler(protocol, address, initializer);
                } catch (IOException e) {
                    /* 异常处理... */
                }
            });
        });
    }

    private void startProtocolHandler(String protocol,
                                      SocketAddress address,
                                      ChannelInitializer<SocketChannel> initializer) throws IOException {
        ServerBootstrap bootstrap = defaultServerBootstrap.clone();
        bootstrap.childHandler(initializer);
        try {
            bootstrap.bind(address).sync(); // handler 对应的服务绑定了相应的端口
        } catch (Exception e) {
            /* 异常处理 */
        }
        log.info("Successfully bind protocol `{}` on {}", protocol, address);
    }
```

## 总结

至此，加载并启动 handlers 的流程就出来了：

1. 通过配置 `protocolHandlerDirectory` 找到目录下所有 nar 包并解压，通过解析 YAML 文件得到 handler 的名字和主类，通过 `NarClassLoader` 加载主类并转型为 `ProtocolHandler` 接口；

2. 调用 handler 的 `accept` 方法判断协议是否被接受，加载所有接受的 handlers；

   ```java
   boolean accept(String protocol);
   ```

3. 传入 broker 的配置到 `initialize` 中初始化加载的 handlers；

   ```java
   void initialize(ServiceConfiguration conf) throws Exception;
   ```

4. 将 `BrokerService` 对象传入各 handler 的 `start` 方法启动

   ```java
   void start(BrokerService service);
   ```

5. 创建 handler 对应的 channel initializers 交由 `BrokerService` 启动，也就是在这一步，handler 被单独作为一个服务启动，比如 KoP 在这里就会默认绑定 9092 端口提供服务

   ```java
   Map<InetSocketAddress, ChannelInitializer<SocketChannel>> newChannelInitializers();
   ```

这几步完成后，broker 就不会干预 handler 了，除非 broker 本身关闭。而 handler 作为一个相对独立的服务，和 broker 的交互全部借由 `start()` 方法中得到的 `BrokerService` 对象来进行。
