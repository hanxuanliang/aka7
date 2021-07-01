# kafka HW 备份机制

先对 `kafka` 高可用机制的基本原理开个篇：

1. `kafka` 高可用是通过 `leader-follower` 多副本机制同步完成
2. 每一个分区含有多个副本 (分区是真正存储日志消息的物理存储介质)
3. 多个副本只有一个 `leader`，也只有它对外提供服务 (读写服务都是)
4. 其他 `follower副本` 只是一个备份冗余
5. `follower副本` 是通过不断向 `leader 副本` 发送 `fetch` 以同步消息 

所以如果 `leader 副本` 挂了，其中一个 `follower副本` 就会成为该分区下新的 `leader副本`。问题就来了，在选为新的 `leader副本` 时，`old leader 副本` 消息没有同步完全，会导致消息丢失或者离散吗？Kafka 是如何解决 `leader副本` 变更时消息不会出错？

## HW 机制

先说几个关键术语：

1. `HW`：HW 一定不会大于日志末端位移，`offset < HW` 的日志被认为是 **已提交**，**已备份**，对消费者可见
2. `LEO`：`last end offset`，日志末端位移。下一个日志要写入的地方

这两个指标是怎么存储？

1. `leader` 会保存自己的 `LEO, HW`，同时会保存 `remote leader` 的 `LEO, HW`
2. 每一个 `follower` 只会保存自己的 `LEO, HW`
3. 以上说明：`remote follower` 的 `LEO, HW` 会被存放在两个地方

下面说说这两个指标的更新机制：

### LEO 更新

现在 `Producer` 发送一批消息到 `broker`，各个副本的变化：

1. `leader副本` 自身 `LEO+1` (因为读写都是由目前的 `leader副本` 提供功能)
2. `follower副本` 从 `leader副本` fetch 日志消息到本地，然后 `LEO+1`
3. 每次 `follower` fetch会将自身的 `LEO` 携带发送，`leader副本` 中的 `remote follower LEO` 更新

### HW 更新

无图🧂🔨：

![](https://oscimg.oschina.net/oscnet/up-bbc513ce55dc1873d2cb23936d5b3fc5ace.png)

基本流程过一下：

1. `leader` 分区接收到 `producer` 发送的消息，该分区磁盘中写入消息，并 `LEO++`
2. `follower` 向 `leader` fetch消息，携带 `fetchOffset=N` (当前需要fetch到的消息offset)，更新 `leader remote LEO`，根据每个 `leader remote LEO, leaderHW` 更新 `leaderHW`
3. 随即 `leader` 发送当前的 `leaderHW` 以及日志消息响应消息，发送给 `follower`。`follower` 该分区写入消息，更新自己的 `follower LEO++`
4. `follower` 发送第二轮 fetch，携带当前最新的 `fetchOffset = N+1`，`leader` 接收到请求，更新 `remote LEO = N+1`，根据 `leaderHW` 计算最新的的值，同时将 `leaderHW` 发送给 `follower`
5. `follower` 对比当前最新的 `LEO` 与 `leaderHW`，取最小的作为新的 `followerHW` 值

## HW 机制缺陷

总体来说，我们需要两轮 `fetch req` 才能完成 `leader` 对 `leaderHW & followerHW` 的更新。如果我们在这个过程中发生 **leader切换**，就会出现数据丢失/数据不一致

## leader epoch

在 0.11 版本后推出，主要是为了弥补 HW 机制的问题。

![](https://oscimg.oschina.net/oscnet/up-6da50acf24aa55da41104ad196fb1dea442.png)

我们可以看到这个 `leader-epoch-checkpoint` 文件，就是用来保存 `leader` 的 `epoch data`：

```shell
$ cat leader-epoch-checkpoint
0
1
0 0
3 10898
```

格式：<epoch, offset>。`epoch` 表示 `leader` 版本，保证单调递增，每次 `leader` 变更，`epoch++`；`offset` 表示每一代 `leader` 写入第一条消息的位移值。

### 工作机制

我们直接从缺陷这边看，就知道整个机制运行：

![](https://oscimg.oschina.net/oscnet/up-653cae940d4a12b87dcdf6d78cf26aa0e14.png)

`follower` 宕机之后不会出现以前那种日志截断的情况

![](https://oscimg.oschina.net/oscnet/up-5bcf53cda4a31289bb67f8634d5a7fe9d73.png)

说明运行以上出现的情况：

1. 宕机之前，`follower` 已不在 ISR 列表中，`unclean.leader.election.enable=true`，即允许非 ISR 中副本成为 leader
2. `follower` 消息写入到 pagecache，但尚未 flush 到磁盘。此时 HW 可能是整个同步写入的中间态

`epoch` 这里解决消息不一致的情况：

1. 新的 `follower` 发送 `epochReq` 会携带自己曾经的 `epoch`
2. 新的 `leader` 判断发送来的 `epoch != nowEpoch`，然后会发送自己的 `epoch start offset`
3. 新的 `follower` 收到新的 `epoch start offset`，会从这个位置往后截断自己的日志 (说白了就是要和成为 `leader` 的 `leader` 保持一致的过去)
4. 发送 `fetchreq`，开始新的同步，那么之后的消息都是一致的。
