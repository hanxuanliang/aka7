# 分布式数据基础

本文介绍分布式数据中常见的两种基本技术: 分区, 复制。

## 分区

由于单机容纳的数据集是有限。而实现可扩展的数据集存储，主要方式之一就是对数据进行分区(Partition)。从而将一个数据集拆分为多个较小的数据集，同时将存储和处理这些较小的数据集的责任分配分布式系统中不同的节点。

那么，数据分区后，我们就可以自由扩展更多的节点从而能存储和处理更大规模的数据。

分区分为垂直分区 (Vertical Partitioning) 和水平分区 (Horizontal Partitioning)。这两种分区方式普遍认为起源千关系型数据库，在设计数据库架构时十分常见。

- **垂直分区**：它是对表的列进行拆分，将不同的列数据放入独立的表中。这种方法缩小了表的宽度并提高了访问性能，特别是对于不常访问或大型数据类型的列。列式数据库可以视为已经进行了垂直分区
- **水平分区**：它是对表的行进行拆分，将不同的行数据放入不同的表中，但每个表都保留了原始表的所有列。例如，将10年订单按照年为单位进行分区

垂直分区和列相关，而一个表中的列是有限的，这就导致了垂直分区不能超过一定的限度；而水平分区则可以无限拆分。另外，表中数据一般是以行为单位不断增长，而列的变动很少，因此水平分区更常见。

分片在不同系统中有不同的叫法，MongoDB 和 ES 中称为 shard, HBase 中称为 region, Bigtable 中称为 tablet。

### 分区算法÷

分区算法用来计算某个数据应该划分到哪个分区上，不同的分区算法有着不同的特性。下面我们将研究一些经典的分区算法，讨论每种算法的优缺点。

- 一致性哈希
  - 虚拟节点解决数据倾斜的问题
  - 一个物理节点不再对应哈希环的一个(实际)节点，而是对应对应多个节点(虚拟节点)
  - 虚拟节点越多，数据分布越均匀。节点被迫下线事，数据也就分摊给其余的节点
  - 对于不同性能的机器，性能越好的机器可以映射更多的节点(权重越高)，让其承担更大的负载【如果这个节点挂了呢?其他节点又该如何处理?】
  - 不存储额外元数据，一致性哈希依然无法搞笑范围查询。任何查询都会发送给多个节点

分区给数据集扩展带来了便利，但是也有一些限制

1. 垂直分区的数据集，进行join查询时会非常低效。一个query需要访问多个节点；而水平分区无需担心，同一行数据存储在同一个节点当中
2. 范围查询下，水平分区下的独立数据集分布在不同节点，query也会访问多个节点
3. 事务。从而引出: 分布式事务的问题
