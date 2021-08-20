---
title: Netty-and-NIO-buffer
date: 2021-08-21 00:19:40
tags:
  - Java
top: 1
---

[toc]

## 简述

目的是比较 NIO buffer（`java.nio.ByteBuffer`）和 Netty buffer（`io.netty.buffer.ByteBuf`）设计上的区别。因为刚好需要用 Netty 的 `ByteBufAllocator` 去重写 Kafka 的基于 NIO buffer 的 `ByteBufferOutputStream`，所以借此机会学习下两者的区别。

## NIO buffer

参考文档：https://docs.oracle.com/javase/8/docs/api/java/nio/ByteBuffer.html

NIO buffer 的设计很简单，它是定长的，也就是必须在构造时指定容量，且不可扩容，比如：

```java
ByteBuffer buffer = ByteBuffer.allocate(4); // 分配 4 字节容量堆内内存
```

每个 NIO buffer 有四个属性：

- mark：记录的位置
- position：当前读写位置
- limit：可读写字节的上界，初始值是 capacity
- capacity：容量

这里暂时忽略 mark 属性。那么上述代码中，`buffer` 构造完成后各下标为：

```
position: 0, limit: 4, capacity: 4
```

可以用以下方法打印出来：

```java
    private static void printMetadata(final ByteBuffer buffer) {
        System.out.println("position: " + buffer.position() + ", limit: " + buffer.limit()
                + ", capacity: " + buffer.capacity());
    }
```

由于 NIO buffer 是不可扩容的，所以理论上 `capacity` 不会改变。而 `position()` 和 `limit()` 方法都有另一个重载形式，可以接受一个 `int` 参数指定新的值。设置 `limit` 的场景大多是为了截断一个 buffer，比如：

```java
final ByteBuffer buffer = ByteBuffer.allocate(4);
buffer.putInt(0x11223344); // 0x11, 0x22, 0x33, 0x44
printMetadata(buffer); // position: 4, limit: 4, capacity: 4
buffer.position(1).limit(3);
printMetadata(buffer); // position: 1, limit: 3, capacity: 4
System.out.println(Integer.toHexString(buffer.getShort())); // 2233
```

> 注：这里的设计没有采用 `setPosition()` 和 `setLimit()` 这样的命名方式，但是返回的都是 `ByteBuffer`，因此支持上述代码的链式调用方式。

NIO buffer 提供 `put` 和 `get` 方法，两者都针对不同类型提供了多种类似方法，但大体分为两类，一类是不带 index 的，也就是直接从末尾写入或读取，一类是带 index 的，从 index 对应位置写入和读取。

limit 属性则是 `get` 读写的上界，如果即将读取的字节位置已经超过了 limit，那么会抛出 `BufferOverflowException`（因为打破了 `position <= limit` 的约束），比如：

```java
final ByteBuffer buffer = ByteBuffer.allocate(2);
buffer.put((byte) 0x11);
printMetadata(buffer); // position: 1, limit: 2, capacity: 2
// 写入的位置是 position + short size = 1 + 2 = 3 > limit，因此抛出异常
buffer.putShort((short) 0x2233); 
```

无论是 `get` 还是 `put`，都会改变 position，因为它们共用一个下标如果往末尾写入了新数据则会改变 limit。这里需要注意的是，读写共用一个下 position，这会导致一些反直觉的行为，比如：

```java
final ByteBuffer buffer = ByteBuffer.allocate(1);
buffer.put((byte) 0x11);
System.out.println(buffer.get()); // BufferUnderflowException
```

直觉上第一次 `get()` 应该从 position 0 开始，但这里其实是从 position 1 开始，因为 `put` 改变了 position。如果要从 position 0 开始，只能在 `get()` 之前手动调用 `buffer.position(0)` 或者 `buffer.rewind()`。

这里提到 `rewind`，它和 `position(0)` 的唯一区别是它将 `ByteBuffer` 的 `mark` 字段设置为了 -1。

现在看看 mark，NIO buffer 永远满足约束： `mark <= position <= limit <= capacity`。读/写都会改变 position，写入可能改变 limit，而 capacity 无法改变。mark 属性仅在 `mark()` 方法中会设置为 -1 之外的值：

```java
    public final Buffer mark() {
        mark = position;
        return this;
    }
```

也就是记录当前的 position，主要用途是在 `put` 写入之前记下位置。`mark()` 方法要结合 `reset()` 方法使用：

```java
    public final Buffer reset() {
        int m = mark;
        if (m < 0) // 必须 mark() 设置过合法的 position，否则会抛出异常
            throw new InvalidMarkException();
        position = m;
        return this;
    }
```

这样可以方便回滚到写之前的位置进行读取，比如：

```java
final ByteBuffer buffer = ByteBuffer.allocate(2);
buffer.mark();
buffer.put((byte) 0x11);
buffer.reset();
System.out.println(Integer.toHexString(buffer.get())); // 11
buffer.mark();
buffer.put((byte) 0x22);
buffer.reset();
System.out.println(Integer.toHexString(buffer.get())); // 22
```

如果是单纯的先写后读的场景，可以在写结束后调用 `flip()`：

```java
    public final Buffer flip() {
        limit = position; // 禁止写入更多字节，除非重新设置 limit
        position = 0; // 读写重新从 position 0 开始
        mark = -1;
        return this;
    }
```

由于读写都会改变 position，因此如果想要读写的同时不改变 position，可以调用 `duplicate()` 方法创建一个 buffer 共享数据，但是拥有独立的 position 以及其他属性，这些属性的初始值都是调用时原 buffer 的相应值。也可以用 `array()` 直接获取底层数组（下标 0 到 capacity）。

## Netty buffer

参考文档：https://netty.io/4.1/api/io/netty/buffer/ByteBuf.html

### 创建

Netty buffer 支持使用 Netty 的 allocator 来分配内存，具体定制可参考 [Netty ByteBufAllocator](https://netty.io/4.1/api/io/netty/buffer/ByteBufAllocator.html)。也可以直接通过现有的 NIO buffer 或者字节数组构造。

```java
final ByteBuf buf1 = Unpooled.wrappedBuffer(new byte[2]);
final ByteBuf buf2 = Unpooled.wrappedBuffer(ByteBuffer.allocate(2));
final ByteBuf buf3 = ByteBufAllocator.DEFAULT.buffer(2);
```

其中，使用 `wrappedBuffer` 构造时，writer index 均为原来的 buffer/数组的大小。

### 读写位置分离

NIO buffer 的设计很简单，但是由于读写共用一个 position，导致使用起来不是很符合直觉，比如写完之后要重新调用 `position()` 设置位置，或者结合 `mark()` 和 `reset()`，或者 `flip()` 等等。

而 Netty buffer 则将读写位置（这里称为 index 而非 position）分离了，见下图。

```
      +-------------------+------------------+------------------+
      | discardable bytes |  readable bytes  |  writable bytes  |
      |                   |     (CONTENT)    |                  |
      +-------------------+------------------+------------------+
      |                   |                  |                  |
      0      <=      readerIndex   <=   writerIndex    <=    capacity
```

- Discardable bytes：已经读取的区域
- Readable bytes：数据实际存储的位置
- Writable bytes：待填满的区域

Netty buffer 将读写分为了两套 API，read/write 系列方法类似于 NIO buffer 的 get/put，读写成功会更新对应的 index。同时还支持指定 index 的 get/set 系列方法用于随机读写，不会更新 index。另外，无论是 readerIndex 还是 writerIndex，Netty buffer 也提供了相应的 mark/reset 系列方法。

```java
final ByteBuf buf = ByteBufAllocator.DEFAULT.buffer(2);
buf.writeByte(1); // readerIndex: 0, writerIndex: 1
buf.writeByte(2); // readerIndex: 0, writerIndex: 2
buf.readByte(); // readerIndex: 1, writerIndex: 2
buf.readByte(); // readerIndex: 1, writerIndex: 2
```

### 扩容

NIO buffer 本质是对定长数组的包装，因此 capacity 是无法增长的，但是 Netty buffer 实现了自动扩容。这里可以先写个代码看看它的扩容策略。

```java
final ByteBuf buf = ByteBufAllocator.DEFAULT.buffer(1);
for (int i = 0; i < 500; i++) {
    buf.writeByte(1);
    System.out.println(i + " address: " + buf.memoryAddress() + ", capacity: " + buf.capacity());
}
buf.release();
```

可以看到输出是：

```
0 address: 140495611232256, capacity: 1
1 address: 140495611232256, capacity: 16
...
16 address: 140495611240448, capacity: 64
...
64 address: 140495611248640, capacity: 128
...
128 address: 140495611256832, capacity: 256
...
256 address: 140495611265024, capacity: 512
...
```

这里不深究具体扩容策略，姑且看到 capacity 到达 64 之后就开始两倍扩容。这里主要看到，每次扩容之后内存地址发生了改变，对于连续存储的数据结构而言，基本都是这个实现套路。

但是，注意这里我们是用 `ByteBufAllocator` 进行分配的，如果是字节数组或者 NIO buffer 的 wrapper，由于底层是定长数组，因此 Netty buffer 无法获取其分配器，自然地，也不知道扩容时用哪个分配器才能跟原来的分配方式一致，因此这种情况是不允许扩容的。

```java
        final ByteBuf buf = Unpooled.wrappedBuffer(ByteBuffer.allocate(100));
        buf.writerIndex(0);
        for (int i = 0; i < 200; i++) {
            try {
                buf.writeByte(1);
            } catch (IndexOutOfBoundsException e) {
                System.out.println("i = " + i);
                break;
            }
        }
        buf.release();
```

输出 `i = 100`，也就是容量到达 100 时就无法继续写入了。

### 引用计数

需要注意的是每个 Netty buffer 维护了一个引用计数，在调用 `retain()` 的时候自增，在 `release()` 的时候自减，使用 allocator 分配的时候初始为 1，只有引用计数降为 0 才会回收内存，否则会出现内存泄漏。由于在 Java 中，经常是将 buffer 传递给其它对象后就不再使用了，因此惯用法是最后一个使用 buffer 对象负责 release。

Netty 可以设置内存泄漏的检测器，详细用法参考 [ResourceLeakDetector](https://netty.io/4.0/api/io/netty/util/ResourceLeakDetector.html)，它可以设置检测等级，默认是 Disabled。这里给出一个示例检测全局资源泄漏。

```java
    public static void leakDemo(final ByteBufAllocator allocator) {
        final ByteBuf buf = allocator.buffer(1024 * 1024);
        buf.writeByte(1);
        buf.readByte();
        // NOTE: not released
        //buf.release();
    }

    public static void main(String[] args) throws InterruptedException {
        // 等价于加上 JVM 选项 -Dio.netty.leakDetectionLevel=paranoid
        ResourceLeakDetector.setLevel(ResourceLeakDetector.Level.PARANOID);
        final ByteBufAllocator allocator = new PooledByteBufAllocator(false /* preferDirect */);
        for (int i = 0; ; i++) {
            leakDemo(allocator);
            Thread.sleep(100);
        }
    }
```

可以看到输出：

```
2021-08-20 23:15:55:264 [main] ERROR io.netty.util.ResourceLeakDetector - LEAK: ByteBuf.release() was not called before it's garbage-collected. See https://netty.io/wiki/reference-counted-objects.html for more information.
Recent access records: 
#1:
	io.netty.buffer.AdvancedLeakAwareByteBuf.readByte(AdvancedLeakAwareByteBuf.java:400)
...
```

从而方便定位泄漏位置。

> 需要注意的是 paranoid 时最高的检测等级，如果改成 simple 或 advanced，上述错误不会报出来。另外如果 preferDirect 改为 true 或者使用默认的 allocator，这里也检测不出来。原因可以参考回答：https://stackoverflow.com/questions/28822632/netty-4-5-does-not-actually-detect-resource-leak-of-bytebuf
>
> 原因简单说就是 Netty 的资源泄漏检测机制依赖于 `ReferenceQueue` 和 `PhantomReference`，如果 VM 太早终止或者 GC 不够快，那么检测器无法判断是否泄漏

## 总结

本文从 NIO buffer 入手，学习了 Netty buffer 的一些改进，包括读写位置分离，自定义分配器，自动扩容。同时还要注意 Netty buffer 相比 NIO buffer 而言需要谨慎地管理内存，同时还可以用 Netty 提供的检测器检测资源泄漏。

个人认为 Netty 的最大优点在于它提供的池化分配器，可以安全地复用堆外内存，减少了 GC 的同时避免了从系统的堆内存到 JVM 堆的拷贝。因此 Pulsar 无论是 broker 还是 client 在分配内存保存消息时，都是使用的 Netty 的分配器。
