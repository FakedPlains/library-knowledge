[[Kafka 概述|Kafka]]

## 生产者客户端

```
public class KafkaProducerAnalysis {
	
	public static final String brokerList = "localhost:9092";
	public static final String topic = "topic-demo";
	
	public static Properties initConfig() (
		Properties props = new Properties();
		props.put("bootstrap.servers", brokerList);
		props.put(ProducerConfig.KEY_SERIALIZER_CLASS—CONFIG, StringSerializer.class.getName());
		props.put(ProducerConfig.VALUE—SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
		properties.put("client.id", "producer.client.id.demo");
		return props;
	}

	public static void main(String[] args) {
		Properties props = initConfig();
		KafkaProducer<String, String> producer = new KafkaProducer<>(props);
		ProducerRecord<S七ring, String> record = new ProducerRecord<>(topic, "hello, Kafka1 ");
		try {
			producer.send(record);
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}
```

KafkaProducer 是线程安全的，可以在多个线程中共享单个 KafkaProducer

### 发送消息

1. 创建 ProducerRecord 对象
2. 发送消息

1. 发后即忘（fire-and- forget）：只管向 Kafka 发送消息，不关心消息是否正确送达
2. 同步（sync）：利用 send 方法返回的 Future 对象
3. 异步（async）：在 send() 中指定 Callback 回调函数

3. 关闭 KafkaProducer

### 序列化

生产者通过序列化器将对象转化成字节数组发送到 Kafka，消费者通过反序列化器从 Kafka 中接收相应的字节数组转化成对象

#### 自定义序列化器

实现`org.apache.kafka.serialization.Serializer`接口

```
public interface Serializer<T> {
	public void configure(Map<String, ?> configs, boolean isKey);
	public byte[] serialize(String topic, T data);
	public void close();
}
```

### 分区器

消息发送到 broker 之前，会经过拦截器、序列化器、分区器的处理

- 如果消息 ProducerRecord 中指定了 partition 字段，则不需要经过分区器，partition 就是要发往的分区
- 如果未指定 partition 字段，则需要依赖分区器为消息分配分区

- Kafka 中提供了默认分区器`org.apache.kafka.clients.producer.internals.DefaultPartitioner`，如果 key 不为 null，默认分区器对 key 进行哈希，根据哈希值计算分区号，拥有相同 key 对消息会发往同一个分区；如果 key 为 null，那么消息会以轮训的方式发往主题中的各个可用分区
- 在不改变主题分区数量的情况下，key 与分区的映射关系可以保持不变

#### 自定义分区器

实现`org.apache.kafka.clients.producer.Partitioner`接口

```
public interface Partitioner extends Configurable, Closeable {
    public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster);
    public void close();
}
```

通过配置参数指定使用自定义分区器

```
props.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, DemoPartitioner.class.getName());
```

### 生产者拦截器

生产者拦截器用于在消息发送之前做一些处理

自定义实现`org.apache.kafka.clients.producer.ProducerInterceptor`接口

```
public interface ProducerInterceptor<K, V> extends Configurable {
    public ProducerRecord<K, V> onSend(ProducerRecord<K, V> record);

    public void onAcknowledgement(RecordMetadata metadata, Exception exception);

    public void close();
}
```

- KafkaProducer 在消息序列化和计算分区之前调用生产者拦截器的`onSend()`方法
- KafkaProducer 在消息被应答之前或消息发送失败时调用生产者拦截器的`onAcknowledgement()`方法，在 Callback 之前执行，且此方法运行在 Producer 的 I/O 线程中

通过配置参数指定链接器，多个拦截器之间使用 “,” 分隔，形成拦截器链

```
properties.put(ProducerConfig.INTERCEPTOR_CLASSES_CONFIG, ProducerinterceptorPrefix.class.getName());
```

在拦截器链中，如果某个拦截器执行失败，那么下一个拦截器会接着从上一个执行成功的拦截器继续执行

## 原理分析

### 整体架构

![](https://cdn.nlark.com/yuque/0/2023/jpeg/26094224/1684484622709-ce5962d8-a695-4ae0-8caa-ed43f8a199e2.jpeg)

Kafka Producer 由两个线程共同工作：main 线程和 sender 线程，main 线程将消息缓存到消息累加器 RecordAccumulator 中，sender 线程从 RecordAccumulator 获取消息发送到 broker 集群

RecordAccumulator 中将各主题的消息按分区存储在不同的双端队列 `Dequeue<ProducerBatch>`  中，ProducerBatch 包含一个或多个 ProducerRecord

sender 线程获取到 RecordAccumulator 中的消息后，最终将其转换为 `Map<Node, Request>`  向各个 broker 节点发送请求

### 重要生产者参数


|**参数**|**默认值**|**描述**|
|---|---|---|
|acks|1|用来指定分区中必须有多少个副本收到消息才认为成功写入<br><br>1：leader<br><br>0：无<br><br>-1 或 all：ISR 中所有副本|
|buffer.memory|33554432(32M)|生产者客户端缓存消息的缓冲区大小|
|max.block.ms|60000|生产者客户端缓冲区已满，或没有可用的元数据时，`send()`和`partitionsFor()`方法阻塞时间|
|batch.size|16384（16K）|ProducerBatch 可复用内存区域大小|
|linger.ms|0|sender 线程发送消息的等待时间，会在 ProducerBatch 被填满或等待时间超过设置的值时发送|
|max.in.flight.requests.per.connection|5|限制每个连接最多缓存的请求数|

