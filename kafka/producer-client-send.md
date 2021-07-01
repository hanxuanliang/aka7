# kafka producer send「一」

作为开篇，先从 `client` 入手，探究一条消息从客户端出发，是如何被传输 -> 存储 -> 消费：

- `producer` 传输给 `broker`
- `broker` 存储
- `consumer` 消费 

## 示例

以下是示例代码，最简单的演示发送若干消息：

```java
package kafkaclient;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.Producer;

import java.util.Properties;

public class SimpleProducer {
	private static int key;

	public static void main(String[] args) {
		Properties props = new Properties();
		props.put("bootstrap.servers", "127.0.0.1:9092,127.0.0.1:9093");
		props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
		props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

		String topicName = "test";
		int msgNum = 10;

		Producer<String, String> producer = new KafkaProducer<>(props);
		for (int i = 0; i < msgNum; i++) {
			String msg = i + " This is hxl's kafka blog.";
			producer.send(new ProducerRecord<>(topicName, msg));
		}
		producer.close();
	}
}
```

基本流程都是：

1. 构建 `KafkaProducer`，将自己的 `Properties` 传递进去
2. 调用 `send()`，使用 `ProducerRecord<topic, value>` 封装你传输的消息

那么关键传输函数就是：`send()`，进去看看。

## send

```java
// 异步向一个 topic 发送数据，等同于：send(record, null)
@Override
public Future<RecordMetadata> send(ProducerRecord<K, V> record) {
    return send(record, null);
}

// 向 topic 异步地发送数据，当发送确认后唤起回调函数
@Override
public Future<RecordMetadata> send(ProducerRecord<K, V> record, Callback callback) {
    ProducerRecord<K, V> interceptedRecord = this.interceptors == null ? record : this.interceptors.onSend(record);
    return doSend(interceptedRecord, callback);
}
```

消息发送最后调用的是 `doSend`。

### dosend

*具体代码，可以在IDE中自行查看*

总体流程分为：

1. 获取 `topic` 对应的 `metadata` ，检测它的可用性；后续会获取它的一些信息

> ```java
> clusterAndWaitTime = waitOnMetadata(record.topic(), record.partition(), nowMs, maxBlockTimeMs);
> ```

2. 序列化 record 的 key 和 value

> ```java
> // key 序列化
> serializedKey = keySerializer.serialize(record.topic(), record.headers(), record.key());
> // value 序列化
> serializedValue = valueSerializer.serialize(record.topic(), record.headers(), record.value());
> ```

3. 获取该 `record` 的 `partition` 的值（可以指定【在创建 `ProducerRecord` 时指定，或者是给一个 `key` 标识符】，也可以根据算法计算）

> ```java
> int partition = partition(record, serializedKey, serializedValue, cluster);
> ```

4. 向 `accumulator` 中追加数据。也就是缓存区

> ```java
> RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey,
>         serializedValue, headers, interceptCallback, remainingWaitMs, true, nowMs);
> ```

5. 如果缓存区的 `batch` 已经满了，唤醒 发送线程 发送数据

> ```java
> if (result.batchIsFull || result.newBatchCreated) {
>     log.trace("Waking up the sender since topic {} partition {} is either full or getting a new batch", record.topic(), partition);
>     this.sender.wakeup();
> }
> return result.future;
> ```

下面对其中几个部分做详细分析。

## 发送详解

获取 `metadata` 部分的我们后面细聊，先看后面的几个。

### 序列化

序列化调用的是直接调用 `KafkaProducer` 内部属性 `keySerializer/valueSerializer` 。而这个地方的赋值是在初始化 `KafkaProducer` 时，将你配置中指定的：

- `key.serializer`
- `value.serializer`

赋值给 `KafkaProducer` 内部属性。

### 获取 partition

```java
private int partition(ProducerRecord<K, V> record, byte[] serializedKey, byte[] serializedValue, Cluster cluster) {
        Integer partition = record.partition();
        return partition != null ?
                partition :
                partitioner.partition(
                        record.topic(), record.key(), serializedKey, record.value(), serializedValue, cluster);
    }
```

获取 `partition` 分为几种情况：

1. 获取消息本身指定的分区。这个在构建 `ProducerRecord` 时可以指定：`ProducerRecord(topic, partition, key, value)`
2. 没有指定具体分区，但是指定 `key`【走的是：`ProducerRecord(topic, key, value)`】：将 key 的 hash 值与该 `topic` 的 `partition_num` 进行取余得到

```java
public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster, int numPartitions) {
  			// 1. 无partition指向，无key
        if (keyBytes == null) {
            return stickyPartitionCache.partition(topic, cluster);
        }
  			// 2. 无partition指向，有key
        // 对key取hash算法值，然后和 partition_num 进行取余
  			// 然后返回这个
        return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
    }
```

3. 无partition指向，无key：会使用 `ThreadLocalRandom` 生成一个整数（这个整数是递增的），然后将这个值和 `topic` 对应的 `partition num` 取余，返回【其中涉及 `topic` 和返回值的缓存存取，也是一个 `kafka` 优化的点】

```java
public int nextPartition(String topic, Cluster cluster, int prevPartition) {
        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
  			// indexCache 是并发安全的map
        Integer oldPart = indexCache.get(topic);
        Integer newPart = oldPart;
        if (oldPart == null || oldPart == prevPartition) {
            ...
            // 获取取余后的结果：partition，然后设置值
            if (oldPart == null) {
                indexCache.putIfAbsent(topic, newPart);
            } else {
                indexCache.replace(topic, prevPartition, newPart);
            }
          	// 并发安全的get
            return indexCache.get(topic);
        }
        return indexCache.get(topic);
    }
```

这就是 Producer 中默认的 partitioner 实现：`DefaultPartitioner.java`

### 向缓存区写入数据

这个部分是最为核心的部分：

```java
RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, serializedKey, serializedValue, headers, interceptCallback, remainingWaitMs, true, nowMs);
```

具体的我们放到下篇文章着重说说里面的设计，这里给出一个数据流向的设计图：

![image.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/10ae0e2c3a7640ff85e6f6cb70637915~tplv-k3u1fbpfcp-watermark.image)

这个就是 `accumulator` 设计的核心数据结构。当然为什么是这样，我们下篇文章来揭晓。。。

## 总结

还有什么没有讲到呢？

1. `topic metedate` 怎么加载进来，怎么更新？
2. 我们传入的 `config` 有哪些选项会影响到 `send` 的过程？
3. `accumulator` 这个内核缓存区是怎么缓存数据，怎么发送数据的？

这些问题，暂时留给大家可以看看源码中给出了什么提示，我们下篇问题来一一说说。
