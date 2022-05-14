---
title: Pulsar AVRO schema 源码阅读：初识
date: 2022-05-14 22:46:50
tags:
  - Pulsar
top: 1
---

[toc]

## Pulsar AVRO schema 入门

一般而言，Avro 序列化需要提供序列化对象的[格式规范](https://avro.apache.org/docs/current/spec.html )，它使用 JSON 来表达。Pulsar 中则是可以直接传递 对象在内部自动生成。比如对于当前类型：

```java
@AllArgsConstructor
@Data
public class User {

    private String name;
    private int age;
}
```

发送一条消息：

```java
            client.newProducer(Schema.AVRO(User.class))
                    .topic("my-topic")
                    .create()
                    .newMessage()
                    .value(new User("xyz", 11))
                    .send();
```

然后查询该 topic 的 schema：

```bash
$ curl -L http://localhost:8080/admin/v2/schemas/public/default/my-topic/schema
{"version":0,"type":"AVRO","timestamp":0,"data":"{\"type\":\"record\",\"name\":\"User\",\"namespace\":\"pulsar.schema\",\"fields\":[{\"name\":\"age\",\"type\":\"int\"},{\"name\":\"name\",\"type\":[\"null\",\"string\"],\"default\":null}]}","properties":{"__jsr310ConversionEnabled":"false","__alwaysAllowNull":"true"}}
```

查询的结果中 `data` 字段（去掉 `\` 转义符）就是 Avro 对 schema 的定义：

```json
{
  "type": "record",
  "name": "User",
  "namespace": "pulsar.schema",
  "fields": [
    {
      "name": "age",
      "type": "int"
    },
    {
      "name": "name",
      "type": [
        "null",
        "string"
      ],
      "default": null
    }
  ]
}
```

## Schema 在 Pulsar 中的使用

### Pulsar 用到的各种 Schema 接口

首先 Apache Avro 库提供的 `Schema` 对象（位于 `org.apache.avro` 包）我们称之为 AVRO `Schema`。

然后在 `PulsarApi.proto` 中我们可以看到 `Schema` 的定义：

```protobuf
message Schema {
    enum Type {
        // ...
        Avro = 4;
        // ...
    }

    required string name = 1;
    required bytes schema_data = 3;
    required Type type = 4;
    repeated KeyValue properties = 5;
}

message CommandProducer {
    // ...
    optional Schema schema = 7;
    // ...
}

message CommandSubscribe {
    // ...
    optional Schema schema = 12;
```

PB 的 `Schema` 暴露给用户的接口则是 `SchemaInfo`（只列出部分方法）：

```java
public interface SchemaInfo {

    // schema 名字
    String getName();

    // 对于 AVRO 类型，为 Avro schema 序列化后的字节数组
    byte[] getSchema();

    // schema 类型，比如 AVRO
    SchemaType getType();
  
    // 用户自定义的一组属性
    Map<String, String> getProperties();
```

而 Pulsar 在创建生产者和消费者时指定的则是自己的 `Schema` 接口（只列出部分方法）：

```java
public interface Schema<T> extends Cloneable{

    // 将对象按照 schema 定义编码成字节数组
    byte[] encode(T message);

    // 将字节数组按照 schema 定义解码成对象，默认忽略 schema version
    default T decode(byte[] bytes) {
        return decode(bytes, null);
    }
    default T decode(byte[] bytes, byte[] schemaVersion) {
        return decode(bytes);
    }
```

大多数实现中会将 `SchemaInfo` 保存到内部字段，就像前文阅读的 `AbstractStructSchema` 类一样。Pulsar `Schema` 类提供了 `encode`/`decode` 方法负责在用户传入的 `T` 类型和 `byte[]` 中进行互相转换。

最后总结下见到的几个 schema 相关的类：

| 包                                 | 类           | 作用                                                         |
| ---------------------------------- | ------------ | ------------------------------------------------------------ |
| org.apache.avro                    | `Schema`     | 底层使用的 Avro schema 类                                    |
| org.apache.pulsar.common.api.proto | `Schema`     | ProtoBuf 协议定义的 schema 描述，会被持久化，并在 Broker 端进行注册和校验 |
| org.apache.pulsar.common.schema    | `SchemaInfo` | 对 ProtoBuf Schema 的接口包装                                |
| org.apache.pulsar.client.api       | `Schema<T>`  | 构造生产者或消费者对象时可以指定，在内部会对对象进行编解码，同时包含 `SchemaInfo` 用于注册和验证 schema 兼容性。 |

### Schema 在 Broker 端的处理

Broker 主要负责对生产者/消费者上传的 schema 进行注册和验证。Pulsar 的 `Schema` 对象在构造生产者和消费者时都会被设置为内部字段，位于 `ProducerBase` 和 `ConsumerBase` 中，并且在创建生产者或者订阅时会取得 `SchemaInfo` 附在 Producer 或者 Subscribe 命令上传递给 broker。

在 Broker 中并不是直接使用 protobuf 的 `Schema`，而是转换成了 `SchemaData`：

```java
    private SchemaData getSchema(Schema protocolSchema) {
        return SchemaData.builder()
            .data(protocolSchema.getSchemaData())
            .isDeleted(false)
            .timestamp(System.currentTimeMillis())
            .user(Strings.nullToEmpty(originalPrincipal))
            .type(Commands.getSchemaType(protocolSchema.getType()))
            .props(protocolSchema.getPropertiesList().stream().collect(
                Collectors.toMap(
                    KeyValue::getKey,
                    KeyValue::getValue
                )
            )).build();
    }
```

在处理 Producer 请求时：

```java
    private CompletableFuture<SchemaVersion> tryAddSchema(Topic topic, SchemaData schema) {
        if (schema != null) {
            return topic.addSchema(schema);
        } else {
```

调用了 `Topic#addSchema` 注册 schema。

而在处理 Subscribe 请求时则是：

```java
if (schema != null) {
    return topic.addSchemaIfIdleOrCheckCompatible(schema)
            .thenCompose(v -> topic.subscribe(option));
```

调用 `Topic#addSchemaIfIdleOrCheckCompatible` 对 schema 进行兼容性检查。至于实现细节就不深入了，实际上都是调用 Pulsar 实现的 Schema 注册服务（`SchemaRegistryService`）的相关方法。

### Schema 在 Client 端的处理

Client 除了上传 schema 外，最重要的就是利用 schema 对用户传入的对象进行编解码。在 `ProducerBase#newMessage` 方法：

```java
    public TypedMessageBuilder<T> newMessage() {
        return new TypedMessageBuilderImpl<>(this, schema);
    }
```

生产端的 schema 被传入了 `TypedMessageBuilderImpl` 中，在用 `value` 方法指定消息的值时：

```java
    public TypedMessageBuilder<T> value(T value) {
        /* ... */
        this.content = ByteBuffer.wrap(schema.encode(value));
        return this;
    }
```

会用 `encode` 方法得到字节数组以进行网络传输。

而在消费端，收到新消息时，会将消息的 payload 和 `schema` 一起构造成 `MessageImpl` 对象，比如在 `ConsumerImpl#newSingleMessage`：

```java
final MessageImpl<V> message = MessageImpl.create(topicName.toString(), batchMessageIdImpl,
        msgMetadata, singleMessageMetadata, payloadBuffer,
        createEncryptionContext(msgMetadata), cnx(), schema, redeliveryCount, poolMessages, consumerEpoch);
```

在 `MessageImpl#getValue` 中，会利用 schema 将收到的字节数组解码成 `T` 对象：

```java
    public T getValue() {
        SchemaInfo schemaInfo = getSchemaInfo();
        if (schemaInfo != null && SchemaType.KEY_VALUE == schemaInfo.getType()) {
            /* ... */
        } else {
            /* ... */
            return decode(schema.supportSchemaVersioning() ? getSchemaVersion() : null);
        }
    }

    private T decode(byte[] schemaVersion) {
        // 对于设置了 poolMessage 的消息，直接对 payload 内部的 NIO buffer（可能在堆外）进行解码
        // 否则会对 getData() 进行解码，getData() 会拷贝一份 payload
        T value = poolMessage ? schema.decode(payload.nioBuffer(), schemaVersion) : null;
        if (value != null) {
            return value;
        }
        // 注：下面代码有些啰嗦，直接 schema.decode(getData(), schemaVersion) 就行
        if (null == schemaVersion) {
            return schema.decode(getData());
        } else {
            return schema.decode(getData(), schemaVersion);
        }
    }
```

## Schema#AVRO 方法实现

### AvroSchema

用户端一般传递 `Class` 对象即可，实际上底层会将其转换成 `SchemaDefinition` 对象：

```java
    static <T> Schema<T> AVRO(Class<T> pojo) {
        // 实际上也是将 Class 对象转换成 SchemaDefinition 对象来构造
        return DefaultImplementation.getDefaultImplementation()
                .newAvroSchema(SchemaDefinition.builder().withPojo(pojo).build());
    }

    static <T> Schema<T> AVRO(SchemaDefinition<T> schemaDefinition) {
        return DefaultImplementation.getDefaultImplementation().newAvroSchema(schemaDefinition);
    }
```

在 `SchemaDefinition` 中会保存这个 `Class` 类型的字段：

```java
    private Class<T> pojo;
```

回到 `Schema#AVRO` 方法，它实际上是基于 `SchemaDefinition` 对象创建了 `AvroSchema` 作为 schema 对象：

```java
    public static <T> AvroSchema<T> of(SchemaDefinition<T> schemaDefinition) {
        /* 这里会首先处理 schema reader/writer 用于支持 multi-schema，这里我们暂不关心这个特性 */
        // getPojo() 返回的用户提供的 T 对应的 Class 对象，这里取得它的 ClassLoader
        ClassLoader pojoClassLoader = null;
        if (schemaDefinition.getPojo() != null) {
            pojoClassLoader = schemaDefinition.getPojo().getClassLoader();
        }
        return new AvroSchema<>(parseSchemaInfo(schemaDefinition, SchemaType.AVRO), pojoClassLoader);
    }
```

```java
    private AvroSchema(SchemaInfo schemaInfo, ClassLoader pojoClassLoader) {
        super(schemaInfo);
        this.pojoClassLoader = pojoClassLoader;
        /* 设置 schema reader/writer (略) */
    }
```

其继承体系为：

```
       +-----------+
       |[I] Schema |
       +-----^-----+
             |
    +--------+---------+
    |[A] AbstractSchema|
    +--------^---------+
             |
  +----------+-------------+
  |[A] AbstractStructSchema|
  +----------^-------------+
             |
+------------+---------------+
|[A] AbstractBaseStructSchema|
+------------^---------------+
             |
       +-----+------+
       | AvroSchema |
       +------------+
```

再依次看基类的构造方法：

```java
    protected final SchemaInfo schemaInfo;

    public AbstractStructSchema(SchemaInfo schemaInfo) {
        this.schemaInfo = schemaInfo;
    }
```

```java
    // org.apache.avro.schema
    protected final Schema schema;

    public AvroBaseStructSchema(SchemaInfo schemaInfo) {
        super(schemaInfo);
        this.schema = parseAvroSchema(new String(schemaInfo.getSchema(), UTF_8));
    }
```

可见 `Schema#AVRO` 方法最关键的是：

1. 调用 `SchemaUtils#parseSchemaInfo` 得到 `SchemaInfo` 对象。
2. 将 `SchemaInfo` 对象传入 `AvroBaseStructSchema#parseAvroSchema` 对象得到 Avro 库的 `Schema` 对象。

### SchemaUtils#parseSchemaInfo

```java
    public static <T> SchemaInfo parseSchemaInfo(SchemaDefinition<T> schemaDefinition, SchemaType schemaType) {
        return SchemaInfoImpl.builder()
                .schema(createAvroSchema(schemaDefinition).toString().getBytes(UTF_8))
                .properties(schemaDefinition.getProperties())
                .name("")
                .type(schemaType).build();
    }

    // 将 SchemaDefinition 转换成 Avro Schema 对象
    public static Schema createAvroSchema(SchemaDefinition schemaDefinition) {
        Class pojo = schemaDefinition.getPojo();

        if (StringUtils.isNotBlank(schemaDefinition.getJsonDef())) {
            return parseAvroSchema(schemaDefinition.getJsonDef());
        } else if (pojo != null) { // 在之前通过 withPojo 已经设置过 pojo
            ThreadLocal<Boolean> validateDefaults = null;
            try {
                Field validateDefaultsField = Schema.class.getDeclaredField("VALIDATE_DEFAULTS");
                validateDefaultsField.setAccessible(true);
                validateDefaults = (ThreadLocal<Boolean>) validateDefaultsField.get(null);
            } catch (NoSuchFieldException | IllegalAccessException e) {
                throw new RuntimeException("Cannot disable validation of default values", e);
            }

            final boolean savedValidateDefaults = validateDefaults.get();

            try {
                // Disable validation of default values for compatibility
                validateDefaults.set(false);
                return extractAvroSchema(schemaDefinition, pojo);
            } finally {
                validateDefaults.set(savedValidateDefaults);
            }
        } else {
            throw new RuntimeException("Schema definition must specify pojo class or schema json definition");
        }
    }
```

针对 Avro schema，需要将 `Schema#VALIDATE_DEFAULTS` 设为 false（这似乎是为了和 Avro 1.8 生成的类兼容，参考 [#5938](https://github.com/apache/pulsar/pull/5938)）然后调用 `extractAvroSchema`，再改回去。

```java
    public static Schema extractAvroSchema(SchemaDefinition schemaDefinition, Class pojo) {
        try {
            // 一般 pojo 是不包含 SCHEMA$ 这个字段的，因此会进入 catch 分支
            return parseAvroSchema(pojo.getDeclaredField("SCHEMA$").get(null).toString());
        } catch (NoSuchFieldException | IllegalAccessException | IllegalArgumentException ignored) {
            // SchemaDefinition 的 alwaysAllowNull 默认为 true，因此回创建 AllowNull 对象
            ReflectData reflectData = schemaDefinition.getAlwaysAllowNull()
                     ? new ReflectData.AllowNull()
                     : new ReflectData();
            // 给 reflectdata 添加一些逻辑类型（主要是日期和时间）的转换
            AvroSchema.addLogicalTypeConversions(reflectData, schemaDefinition.isJsr310ConversionEnabled(), false);
            return reflectData.getSchema(pojo);
        }
    }
```

实际上是将 POJO 类通过 AVRO 库转换成 AVRO 的 `Schema` 类型。回顾 `parseSchemaInfo` 方法：

```java
                .schema(createAvroSchema(schemaDefinition).toString().getBytes(UTF_8))
```

将 `Schema` 转换成字符串再按照 UTF-8 编码成字节作为 `SchemaInfo` 的 `schema` 字段。

### SchemaUtils#parseAvroSchema

```java
    public static Schema parseAvroSchema(String schemaJson) {
        final Schema.Parser parser = new Schema.Parser();
        parser.setValidateDefaults(false);
        return parser.parse(schemaJson);
    }
```

这里实际上又将转换成字节数组的 schema 又转换回去了。

### encode 和 decode 方法

AVRO schema 的编解码实现在 `AbstractStructSchema` 类：

```java
    protected SchemaReader<T> reader;
    protected SchemaWriter<T> writer;

    @Override
    public byte[] encode(T message) {
        return writer.write(message);
    }

    @Override
    public T decode(byte[] bytes) {
        return reader.read(bytes);
    }
```

这里用到了前文我们忽略的 `SchemaReader` 和 `SchemaWriter`，它们是在 `AvroSchema` 中设置的：

```java
    private AvroSchema(SchemaInfo schemaInfo, ClassLoader pojoClassLoader) {
        /* ... */
        // 用于从 SchemaInfo 的属性中取得 __jsr310ConversionEnabled 来判断是否支持 JSR 310
        // 从而兼容旧的 Pulsar 客户端（不支持 JSR 310 转换）
        boolean jsr310ConversionEnabled = getJsr310ConversionEnabledFromSchemaInfo(schemaInfo);
        setReader(new MultiVersionAvroReader<>(schema, pojoClassLoader,
                getJsr310ConversionEnabledFromSchemaInfo(schemaInfo)));
        setWriter(new AvroWriter<>(schema, jsr310ConversionEnabled));
    }
```

首先来看 `AvroWriter`，也就是编码（将 `T` 转换成 `byte[]`）：

```java
    public AvroWriter(Schema schema, boolean jsr310ConversionEnabled) {
        this.byteArrayOutputStream = new ByteArrayOutputStream();
        this.encoder = EncoderFactory.get().binaryEncoder(this.byteArrayOutputStream, null);
        ReflectData reflectData = new ReflectData();
        // 这里会决定是否进行 JSR 310 转换
        AvroSchema.addLogicalTypeConversions(reflectData, jsr310ConversionEnabled);
        this.writer = new ReflectDatumWriter<>(schema, reflectData);
    }

    @Override
    public synchronized byte[] write(T message) {
        byte[] outputBytes = null;
        try {
            writer.write(message, this.encoder);
        } catch (Exception e) {
            throw new SchemaSerializationException(e);
        } finally {
            try {
                this.encoder.flush();
                // 从 write 的输出流中取出编码后的字节
                outputBytes = this.byteArrayOutputStream.toByteArray();
            } catch (Exception ex) {
                throw new SchemaSerializationException(ex);
            }
            this.byteArrayOutputStream.reset();
        }
        return outputBytes;
    }
```

实际上是构造了 Avro 库的 `ReflectDatumWriter` 进行编码。

再来看 `MultiVersionAvroReader`：

```java
    public MultiVersionAvroReader(Schema readerSchema, ClassLoader pojoClassLoader, boolean jsr310ConversionEnabled) {
        super(new AvroReader<>(readerSchema, pojoClassLoader, jsr310ConversionEnabled), readerSchema);
```

它是构造了 `AvroReader` 负责实际的解码：

```java
    public AbstractMultiVersionReader(SchemaReader<T> providerSchemaReader) {
        this.providerSchemaReader = providerSchemaReader;
    }

    @Override
    public T read(byte[] bytes, int offset, int length) {
        return providerSchemaReader.read(bytes);
    }
```

> 注：multi version schema 的实现更为复杂，内部用了一个 cache 来缓存不同版本的 schema reader。

而 `AvroReader` 实现如下：

```java
    public AvroReader(Schema schema, ClassLoader classLoader, boolean jsr310ConversionEnabled) {
        this.schema = schema;
        if (classLoader != null) {
            ReflectData reflectData = new ReflectData(classLoader);
            AvroSchema.addLogicalTypeConversions(reflectData, jsr310ConversionEnabled);
            this.reader = new ReflectDatumReader<>(schema, schema, reflectData);
        } else {
            this.reader = new ReflectDatumReader<>(schema);
        }
    }

    public T read(InputStream inputStream) {
        try {
            BinaryDecoder decoderFromCache = decoders.get();
            BinaryDecoder decoder = DecoderFactory.get().binaryDecoder(inputStream, decoderFromCache);
            if (decoderFromCache == null) {
                decoders.set(decoder);
            }
            return reader.read(null, DecoderFactory.get().binaryDecoder(inputStream, decoder));
```

实际上是构造了 Avro 库的 `ReflectDatumReader` 进行解码。

## 总结

至此，关于 Pulsar 处理 AVRO schema 的流程大致有了了解。其实就是对 Avro 库进行了包装，用来对用户传入的对象进行编解码。同时，将 AVRO schema 的 JSON 描述信息存放到了 schema info 的 data 字段中，Broker 会将其持久化，并对 Client 上传的 schema 描述信息进行注册和验证。更多的细节可以进一步阅读 Pulsar 的 schema 注册服务。
