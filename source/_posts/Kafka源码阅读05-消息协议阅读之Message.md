---
title: Kafka源码阅读05-消息协议阅读之Message
date: 2019-10-14 12:42:36
tags:
  - Kafka
top: 1
---
## 回顾

之前阅读了网络层和API层，在阅读API层支持的Kafka协议之前，首先得明确Kafka的消息概念。Kafka服务器被称为**broker**，与其交互的是客户端，分为**生产者（Producer）**和**消费者（Consumer）**，客户端与服务端通过消息进行交互。

Kafka使用日志文件（下文称为**消息日志**）来保存消息，通过`log.dirs`配置指定日志文件的存放目录。注意这里的日志文件不同于Kafka本身的日志（记录运行时的一些信息）。而对于每个分区，都会在`log.dirs`下创建一个子目录来存放消息日志，其命名为`<topic>-<partition>`，在该目录下会有像这样的文件：

```
$ ls
00000000000000000020.index  00000000000000000020.log  00000000000000000020.timeindex  leader-epoch-checkpoint
```

同一分区的不同消息是通过offset来唯一标识的，注意它并不是消息在消息日志中实际存储位置的偏移量，而是类似id一样的概念，从0开始递增，表示分区内第offset条消息。

消息日志的命名规则是`[baseOffset].log`，比如这里的20就是该日志的第**baseOffset**，即消息日志中的第1条消息的offset。相应地，有同名的`.index`文件，为消息建立了索引方便查询消息，但并没有对每条消息都建立了索引。

因此首先看看Kafka的消息实现，即`message`包，本文主要讲`Message`类。

## 消息格式

`Message`类的注释给出了格式说明，如下图所示：

```
字节数 |   4   |   1   |     1     |     8     |   4    |  K  |  4  |    V    |
字段名 | CRC32 | magic | attribute | timestamp | keylen | key | len | payload |
                         /     \
             | 7 | 6 | 5 | 4 | 3 | 2 | 1 | 0 |
                0~2: 压缩类型；3: 时间戳类型 
```

补充说明：

- `magic`代表消息格式，其值为0代表v0，为1代表v1；
- v0版本的消息使用绝对offset，且不包含`timestamp`字段，`attribute`第3位不使用；
- v1版本的消息使用相对offset，且包含`timestamp`字段，`attribute`第3位为时间戳类型；
- `K`是字段`keylen`的值，`V`是字段`len`的值。

## 外部消息和内部消息

看看消息的主构造器

```scala
class Message(val buffer: ByteBuffer,
              private val wrapperMessageTimestamp: Option[Long] = None,
              private val wrapperMessageTimestampType: Option[TimestampType] = None)
```

- `buffer`：消息的字节缓冲区；
- `wrapperMessageTimestamp`：外部消息的时间戳；
- `wrapperMessageTimestampType`：外部消息的时间戳类型；

这里的`wrapperMessage`指的是外部消息，因为Kafka会对多个消息一起进行压缩提高压缩率，所以将N个消息压缩后的消息称为**外部消息**，而这N个消息则称为**内部消息**。

```
外部消息offset | 100 |     105    | 106 | 107 | ...
                         /   \ 
内部消息offset    | 0 | 1 | 2 | 3 | 4 |
```

这样做是因为生产者对一批消息压缩时，它是不知道消息的offset时（因为offset是由broker指定的），所以就简单地将offset字段从0开始依次递增来设置。

而broker在收到这批消息时，它知道前1个消息的offset（比如在这里就是100），也知道生产者发送过来的这批消息的数量（5），那么下一个外部消息的offset就被设置为100+5=105。

消费者取得的是外部消息，当消费者通过解压得到每个消息时，可以用外部offset和内部offset计算出内部消息的绝对offset（101~105）。

## getter方法

对这种基于字节的消息协议的实现很简单，利用`ByteBuffer`对象存储字节序列，然后用伴生对象的常量来指定某个字段的长度（length）和偏移量（offset），从而通过其字节区间[offset, offset + length)访问该字段。

再次注意：这里提到的偏移量指的是字节在缓冲区中的位置，不同于消息的offset。

在`Message`伴生对象中定义一系列常量来记录各字段的offset和length：

```scala
object Message {
  val CrcOffset = 0
  val CrcLength = 4
  val MagicOffset = CrcOffset + CrcLength
  val MagicLength = 1
  val AttributesOffset = MagicOffset + MagicLength
  // ...
}
```

然后对于`crc`和`magic`这种整型字段的getter方法直接调用`ByteBuffer.getInt(index)`方法即可（注意不能用`getInt()`方法，因为它是从内部position开始读的）

```scala
  def checksum: Long = ByteUtils.readUnsignedInt(buffer, CrcOffset)
  def magic: Byte = buffer.get(MagicOffset)
```

对于多字节的字段`crc`，使用的是Java类`ByteUtils`的相关方法，将多个字节转换成目标整型，实际上还是首先调用`ByteBuffer`的`getXXX(index)`方法

```java
    public static long readUnsignedInt(ByteBuffer buffer, int index) {
        return buffer.getInt(index) & 0xffffffffL;
    }
```

对于`key`和`payload`这种运行期才确定长度的字段，其编码方式是用户自定义的，所以只需要返回一个`ByteBuffer`即可，具体编解码应该在客户端进行：

```scala
  def payload: ByteBuffer = sliceDelimited(payloadSizeOffset)
  def key: ByteBuffer = sliceDelimited(keySizeOffset)

  // 由于key字段长度是动态的，所以无法在object中定义payloadSizeOffset常量，而需要在类中取得key的长度后计算而出
  private def payloadSizeOffset = {
    if (magic == MagicValue_V0) KeyOffset_V0 + max(0, keySize)
    else KeyOffset_V1 + max(0, keySize)
  }

  private def sliceDelimited(start: Int): ByteBuffer = {
    val size = buffer.getInt(start) // 取得前4个字节表示的长度
    if(size < 0) {
      null
    } else {
      // 拷贝一份，防止影响buffer的position，注意这里的拷贝并没拷贝内部的字节序列
      var b = buffer.duplicate()
      // 跳过表示长度的4个字节，回顾协议，key和payload之前都有4个字节表示长度
      b.position(start + 4)
      // 取得buffer从position开始的子序列，并重置长度和position
      // 之后b就是指向key或payload字段并且长度合适的ByteBuffer了
      b = b.slice()
      b.limit(size)
      b.rewind
      b
    }
  }
```

## 时间戳

Kafka的消息格式在0.10.0的一个重要变化是加入了时间戳字段，见[upgrade to 0.10.0.0](https://kafka.apache.org/11/documentation.html#upgrade_10_performance_impact)，为了保持旧消息的兼容，才有了`magic`标识是否使用时间戳，并且支持对API版本的请求：[Retrieving Supported API versions](https://kafka.apache.org/protocol.html#api_versions)。值得一看的是`timestamp`的getter：

```scala
  def timestamp: Long = {
    if (magic == MagicValue_V0) // v0版本的消息不使用时间戳
      Message.NoTimestamp
    // 对v1版本的消息，有以下3种Case：
    // 1. 外部消息的时间戳及其类型都为None；
    // 2. 外部消息的时间戳类型为LogAppendTime且时间戳非None；
    // 3. 外部消息的时间戳类型为CreateTime且时间戳非None。
    // Case 2
    else if (wrapperMessageTimestampType.exists(_ == TimestampType.LOG_APPEND_TIME) && wrapperMessageTimestamp.isDefined)
      wrapperMessageTimestamp.get
    else // Case 1, 3
      buffer.getLong(Message.TimestampOffset)
  }
```

后2种Case都代表当前消息是内部消息，也就是说和其他内部消息一起被压缩了，只有时间戳类型为`LogAppendTime`时才使用外部消息的时间戳。

## 辅助构造器

一般不会直接传入`ByteBuffer`，而是传入消息协议的各个字段来构造，也就是辅助构造器

```scala
  def this(bytes: Array[Byte], 
           key: Array[Byte],
           timestamp: Long,
           timestampType: TimestampType,
           codec: CompressionCodec, 
           payloadOffset: Int, 
           payloadSize: Int,
           magicValue: Byte) = {
    // 计算总长度，分配对应大小的ByteBuffer
    this(ByteBuffer.allocate(Message.CrcLength +
                             Message.MagicLength +
                             Message.AttributesLength +
                             (if (magicValue == Message.MagicValue_V0) 0
                              else Message.TimestampLength) +
                             Message.KeySizeLength + 
                             (if(key == null) 0 else key.length) + 
                             Message.ValueSizeLength + 
                             (if(bytes == null) 0 
                              else if(payloadSize >= 0) payloadSize 
                              else bytes.length - payloadOffset)))
    // 验证magic和timestamp是否对应：
    // 1. magic只能为0或1；
    // 2. 时间戳必须为非负数或-1（代表不使用时间戳）；
    // 3. magic为0时timestamp必须为-1，因为v0版本不支持时间戳；
    validateTimestampAndMagicValue(timestamp, magicValue)
    // skip crc, we will fill that in at the end
    // 跳过CRC字段，先填充后面的部分
    buffer.position(MagicOffset)
    buffer.put(magicValue) // 填充 magic
    // 根据压缩类型和时间戳类型计算 attribute 并填充
    val attributes: Byte = LegacyRecord.computeAttributes(magicValue, CompressionType.forId(codec.codec), timestampType)
    buffer.put(attributes)
    // Only put timestamp when "magic" value is greater than 0
    if (magic > MagicValue_V0) // 仅当magic大于0才可以填充时间戳
      buffer.putLong(timestamp)
    if(key == null) { // key为空则将keylen填充为-1，代表key不存在
      buffer.putInt(-1)
    } else { // 否则填充 keylen 和 key
      buffer.putInt(key.length)
      buffer.put(key, 0, key.length)
    }
    // 类似key，若bytes为空，填充len为-1，否则填充 bytes 的指定部分
    // payloadOffset指定起始偏移量，payloadSize指定填充字节数(若<0则填充偏移量之后的所有字节)
    val size = if(bytes == null) -1
               else if(payloadSize >= 0) payloadSize 
               else bytes.length - payloadOffset
    buffer.putInt(size)
    if(bytes != null)
      buffer.put(bytes, payloadOffset, size)
    buffer.rewind() // 重置position为初始，以便之后getXXX()读取

    // now compute the checksum and fill it in
    // 后面字段填充完了，计算CRC校验值并填充到前4个字节，完成后position为4，
    // 也就是对内部buffer调用slice()方法返回的是magic字段至末尾的部分而不包含CRC字段
    ByteUtils.writeUnsignedInt(buffer, CrcOffset, computeChecksum)
  }
```

其他辅助构造器都是基于这个辅助构造器构造的，代码就不一一贴出。

## Record类

`class Message`的`asRecord()`方法和`object Message`的`fromRecord()`方法提供了`Message`类和Java的`LegacyRecord`类的互相转化：

```scala
  // object Message
  def fromRecord(record: LegacyRecord): Message = {
    val wrapperTimestamp: Option[Long] = if (record.wrapperRecordTimestamp == null) None else Some(record.wrapperRecordTimestamp)
    val wrapperTimestampType = Option(record.wrapperRecordTimestampType)
    new Message(record.buffer, wrapperTimestamp, wrapperTimestampType)
  }
```

```scala
  // class Message
  private[message] def asRecord: LegacyRecord = wrapperMessageTimestamp match {
    case None => new LegacyRecord(buffer)
    case Some(timestamp) => new LegacyRecord(buffer, timestamp, wrapperMessageTimestampType.orNull)
  }
```

可见两者的构造器完全一致，其实去看实现的话大多数方法也是一致的。只不过`Record`提供了一系列`write()`方法可以将内部存储的字节写入到`DataOutputStream`类中，而`Message`本身没有，因此要将`Message`写入数据流时需要调用`asRecord`转换成Record对象再调用`write()`方法。

此外`LegacyRecord`还提供了`writeCompressedRecordHeader()`方法在创建消息集（Message Set）时会使用，到时候再去阅读。

## 总结

本文内容比较简单，主要是阅读Kafka的消息格式的实现，Java/Scala使用`ByteBuffer`来实现基于字节的消息协议。Kafka在0.10.0中做出了较大改变，添加了时间戳字段，因此使用了`magic`字段来区分不同版本的消息。最后，Scala类`Message`也提供了与Java类`LegacyRecord`的转换方法，从而实现向数据流的写入。