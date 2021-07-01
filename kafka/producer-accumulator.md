# kafka producer accumulator

`Producer` 会将 `record` 加入到一个 `buffer`。这就是 `RecordAccumulator`，而这个 `buffer` 承载的主体是：`ConcurrentMap<TopicPartition, Deque<RecordBatch>> batches`，每个 `partition` 对应一个 `deque`。添加数据的时候，会选定一个 `partition` (没有就创建一个key)，然后对这个 `partition` 对应的 `deque` 最新的一个 `RecordBatch` (没有就创建) 加入 `record`。发送时，会将 `deque` 头部的 `RecordBatch` 弹出发送【满足 `queue` 的先入先出】。


## 源码阅读

将 `key, value` 加入到指定的 `portition` 中的流程：

```java
public RecordAppendResult append(TopicPartition tp,
                                    long timestamp,
                                    byte[] key,
                                    byte[] value,
                                    Header[] headers,
                                    Callback callback,
                                    long maxTimeToBlock) throws InterruptedException {
    ...
    try {
        // 1. 从 batches_map 找 tp 这个key对应的 Deque；找不到就创建一个，然后加入 batches_map
        Deque<ProducerBatch> dq = getOrCreateDeque(tp);
        synchronized (dq) {
            ...
            // 2. 尝试向这个 deque 中最后的一个 bathc 加入 record；如果加进去了，返回result；没有成功，返回 NULL
            RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq);
            if (appendResult != null)
                return appendResult;
        }

        ...
        // 3. 上面没有塞进去，说明满了，需要创建一个新的 RecordBatch
        //    3-1. 需要确定size：batchSize > abstractRecordSize，那就取 batchSiz
        int size = Math.max(this.batchSize, AbstractRecords.estimateSizeInBytesUpperBound(maxUsableMagic, compression, key, value, headers));
        ...
        // 4. 按照 size 开辟内存空间，供 record 存放
        buffer = free.allocate(size, maxTimeToBlock);
        synchronized (dq) {
            ...
            // 5. 再次尝试是否可以加入
            RecordAppendResult appendResult = tryAppend(timestamp, key, value, headers, callback, dq);
            if (appendResult != null) {
                // Somebody else found us a batch, return the one we waited for! Hopefully this doesn't happen often...
                return appendResult;
            }
            // 6. 创建一个基于 buffer 的 RecordBatch
            MemoryRecordsBuilder recordsBuilder = recordsBuilder(buffer, maxUsableMagic);
            ProducerBatch batch = new ProducerBatch(tp, recordsBuilder, time.milliseconds());
            // 7. 向这个 RecordBatch 追加数据
            FutureRecordMetadata future = Utils.notNull(batch.tryAppend(timestamp, key, value, headers, callback, time.milliseconds()));
            // 8. 将 RecordBatch 添加到当前的 queue 中
            dq.addLast(batch);
            // 9. 向未 ack 的 batch 集合添加这个 batch
            incomplete.add(batch);

            // Don't deallocate this buffer in the finally block as it's being used in the record batch
            buffer = null;
            // 10. 如果 dp.size()>1 就证明这个 queue 有一个 batch 是可以发送了
            return new RecordAppendResult(future, dq.size() > 1 || batch.isFull(), true);
        }
    } finally {
        if (buffer != null)
            free.deallocate(buffer);
        ...
    }
}
```

加入到指定的 `partition`，最后这些在 `buffer` 中的数据怎么发送出去？这里就看到 `The main run loop for the sender thread`：

```java
public void run() {
    ...
    // 1. 主循环，一直运行直到 close 被调用
    while (running) {
        try {
            runOnce();
        } catch (Exception e) {
            log.error("Uncaught error in kafka producer I/O thread: ", e);
        }
    }
    ...
    // 2. 不是强制关闭 && (缓冲区中还有消息未发出 || client 还有正在被处理的请求)
    // 此时得完成剩下这些没有完成的任务 (很简单，你退出了，没有完成的当然要完成了再完全退出)
    while (!forceClose && (this.accumulator.hasUndrained() || this.client.inFlightRequestCount() > 0)) {
        try {
            runOnce();
        } catch (Exception e) {
            log.error("Uncaught error in kafka producer I/O thread: ", e);
        }
    }
    // 3. 是强制关闭，accumulator 会放弃未处理完的 batch
    if (forceClose) {
        ...
        this.accumulator.abortIncompleteBatches();
    }
    try {
        // 4. 关闭客户端
        this.client.close();
    } catch (Exception e) {
        log.error("Failed to close network client", e);
    }
    ...
}
```

这里我们需要关注的就是 `runOnce()`。这个里面才是正确处理缓冲区消息的地方：

```java
void runOnce() {
	// 此处省略与事务消息相关的逻辑，此处不关注
    ...
    long currentTimeMs = time.milliseconds();
    long pollTimeout = sendProducerData(currentTimeMs);   
    client.poll(pollTimeout, currentTimeMs);                    
}
```

1. `sendProducerData` 真正进行 `RecordBatch` 发送 -> 合并一个node的 `RecordBatch` 放在一个 `produce` 请求发送
2. `poll` 做真正的 `socket` 读写

其中提一嘴：`client.poll()` 中的 `client` 其实就是 `NetworkClient`。负责实际的网络请求

### accumulator.ready()

在客户端发送到服务端之前，需要确定一下此时集群中符合接收条件的节点集合。条件如下，满足一个即可加入：

1. `Deque` 中有多个 `RecordBatch` 或是第一个 `RecordBatch` 满了
2. 达到了需要发送 `RecordBatch` 的时间间隔
3. 等待 `bufferPool` 释放空间 (已经没有空间)
4. `Sender` 被 `close()` 需要把最后的空间发送出去

来看看 `ready` 是怎么准备符合条件的 `node`：

```java
public ReadyCheckResult ready(Cluster cluster, long nowMs) {
    // 1. 可以接收消息的 node set
    Set<Node> readyNodes = new HashSet<>();
    long nextReadyCheckDelayMs = Long.MAX_VALUE;
    // 2. 元数据中找不到 leader 副本的分区
    Set<String> unknownLeaderTopics = new HashSet<>();

    // 3. 判断现在是否在等待 bufferpool 是否空间
    boolean exhausted = this.free.queued() > 0;
    // 4. 开始遍历 batches，对缓冲区中所记录的每一个分区(key)的 leader 副本所在的 node 都进行判断
    for (Map.Entry<TopicPartition, Deque<ProducerBatch>> entry : this.batches.entrySet()) {
        TopicPartition part = entry.getKey();
        Deque<ProducerBatch> deque = entry.getValue();
        
        // 4-1. 查找分区的 leader 副本所在的 node
        Node leader = cluster.leaderFor(part);
        // 4-2. 对当前 queue 加锁，读取其消息
        synchronized (deque) {
            // 4-3. 当前元数据中对应的 leader 找不到，则不能发送消息
            if (leader == null && !deque.isEmpty()) {
                // 4-3-1. 并加入到 `找不到 leader 副本` 的集合中，这里不为空会触发 metadata 的更新
                unknownLeaderTopics.add(part.topic());
            } else if (!readyNodes.contains(leader) && !isMuted(part, nowMs)) {
                // 4-3-2. 只取 deque.first。只要不为空就发送消息
                ProducerBatch batch = deque.peekFirst();
                if (batch != null) {
                    long waitedTimeMs = batch.waitedTimeMs(nowMs);
                    boolean backingOff = batch.attempts() > 0 && waitedTimeMs < retryBackoffMs;
                    long timeToWaitMs = backingOff ? retryBackoffMs : lingerMs;
                    boolean full = deque.size() > 1 || batch.isFull();
                    boolean expired = waitedTimeMs >= timeToWaitMs;
                    // 对应以上说的4个条件
                    boolean sendable = full || expired || exhausted || closed || flushInProgress();
                    if (sendable && !backingOff) {
                        // 4-3-4. 将 leader 副本作为 node 加入
                        readyNodes.add(leader);
                    } else {
                        long timeLeftMs = Math.max(timeToWaitMs - waitedTimeMs, 0);
                        // Note that this results in a conservative estimate since an un-sendable partition may have
                        // a leader that will later be found to have sendable data. However, this is good enough
                        // since we'll just wake up and then sleep again for the remaining time.
                        nextReadyCheckDelayMs = Math.min(timeLeftMs, nextReadyCheckDelayMs);
                    }
                }
            }
        }
    }
    return new ReadyCheckResult(readyNodes, nextReadyCheckDelayMs, unknownLeaderTopics);
}
```

得到 `readyNodes`，此时还需要检测是否可以连接 (有连接说明可以，没有连接就创建一个连接。还是没有连接，则过滤掉这个node)。

**至此，需要发送的 node 就准备好了，以后需要发送的 `ProducerBatch` 也准备好了**

### accumulator.drain()

现在我们就需要转换到 `node` 层面的发送。我们之前准备的都是 `TopicPartition -> ProducerBatch`，现在需要转换为 `NodeId -> RecordBatch`，因为对 `broker` 连接本身就是针对 `node` 的，不存在说对某一个分区的发送，所以这个地方需要有一个从 **逻辑层到网络层** 的转换；而且也正是因为针对的是 `node`，所以要把在这个 `node` 上的分区统一成一个 `request` 发送出去，所以这里也承担着 **合并RecordBatch** 的职责。

这个工作在 `RecordAccumulator.drain()`：

```java
public Map<Integer, List<ProducerBatch>> drain(Cluster cluster, Set<Node> nodes, int maxSize, long now) {
    if (nodes.isEmpty())
        return Collections.emptyMap();
    
    // 1. 可以从这个 map 就知道最后的结果：nodeId -> RecordBatch 的映射
    Map<Integer, List<ProducerBatch>> batches = new HashMap<>();
    // 2. 遍历当前准备好的 ready Node
    for (Node node : nodes) {
        // 3. 将 node 上的分区集合，将这些分区对应的 ProducerBatch 插入到这个 ready 中
        //    也就是将范围聚焦到每一个 node 上
        List<ProducerBatch> ready = drainBatchesForOneNode(cluster, node, maxSize, now);
        // 4. nodeId -> RecordBatch
        batches.put(node.id(), ready);
    }
    return batches;
}
```

当然拿到这个 `<nodeId, ProducerBatch>` 的map集合，也需要像上一步检测一下当前 node 的这些连接是否可用 (因为有连接是有时间限制的)。过滤完了，剩下就是发送这些 `ProducerBatch`。

简单来说：最后是将同一个 `node` 下的分区集合对应的 ProducerBatch 放到一个 `produce request` 中发送出去。这个就涉及到 `client -> server` 协议问题：

1. 如何封装请求？
2. 请求头？数据具体在哪块

这个会在 `server` 端协议分析里面说道说道

### send 核心图解

```txt
             +--------------------+                                     
             |  sendProducerData  |                                     
             +--------------------+                                     
                        |                                               
           +------------+--------------------+                          
           |                                 |                          
           v                                 v                          
+---------------------+           +---------------------+               
| accumulator.ready() |           | accumulator.drain() |-------+       
+---------------------+           +---------------------+       |       
           |                                 ^                  |       
           v                                 |                  v       
     .-----------.                           |            .-----------. 
    (  send node  )------avaliable node------+           ( send batchs )
     `-----------'                                        `-----------' 
                                                                |       
                               +----------------------+         |       
                               |  sendProduceRequests |<--------+       
                               +----------------------+                            
```
