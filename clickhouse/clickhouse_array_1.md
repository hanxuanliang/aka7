# 「Clickhouse Array 的力量」1

> 原文：[https://altinity.com/blog/harnessing-the-power-of-clickhouse-arrays-part-1](https://altinity.com/blog/harnessing-the-power-of-clickhouse-arrays-part-1)
> 

ClickHouse 贡献者经常添加超越标准SQL的分析功能。这种设计方法在成功的开源项目中很常见，并且反映了以创造性的方式解决现实世界的问题的偏向。

数组就是一个很好的例子。大多数现代数据库都有数组数据类型，但它们的功能往往是有限的。ClickHouse则完全不同：它创造性地将数组与聚合和函数式编程结合起来，创造了一个全新的处理模型。

这篇文章由多部分组成其中有对 ClickHouse Array 的概述，同时会附带具体的例子驱动。我们从基本功能开始，比如定义和加载数组数据，使用数组函数，以及将数组展开为表格格式。这些使ClickHouse 用户能够处理具有任意参数的时间序列。

我们在此基础上探索聚合和数组的关系，以及展示使用lambdas和复杂数组函数的函数式编程的例子。高级功能使我们能够解决诸如建立分析漏斗和线性插值的问题。

**要记住：我们的目标是展示什么是数组，以及为什么要使用它们。**

## Array 基础

在ClickHouse中，数组只是另一种列数据类型，它代表一个值的集合(vector)。下面是一个在ClickHouse表中使用数组的简单例子：

```sql
/* create table */
CREATE TABLE array_test (
  floats Array(Float64),
  strings Array(String),
  nullable_strings Array(Nullable(String))
) ENGINE=TinyLog
```

使用数组很容易，特别是如果你熟悉 Python/Javascript 等流行的编程语言。

你可以用 `[]` 创建数组值，并从其中选择单个值，同样用 `[]`。下面是一个插入数组值并以不同方式再次选择它们的例子。遵循SQL传统，ClickHouse数组索引从1开始，而不是0：

```sql
INSERT INTO array_test 
VALUES([1.0, 2.5], ['a', 'c'], ['A', NULL, 'C'])

SELECT floats[1] AS v1, strings[2] AS v2, nullable_strings 
FROM array_test

/* sql answer*/
┌─v1─┬─v2─┬─nullable_strings─┐
│  1 │ c  │ ['A',NULL,'C']   │
└────┴────┴──────────────────┘
```

ClickHouse有一个丰富的数组函数库。**你不需要特意去建一个 table 来演示使用，相反它很容易定义数组常量，如下例所示。**这是一个快速测试函数行为的好方法：

```sql
SELECT 
    [1, 2, 4] AS array,
    has(array, 2) AS v1,
    has(array, 5) AS v2,
    length(array) AS v3

/* sql answer*/
┌─array───┬─v1─┬─v2─┬─v3─┐
│ [1,2,4] │  1 │  0 │  3 │
└─────────┴────┴────┴────┘
```

基本的数组行为就这么多了。关于更多的细节，请查看 [ClickHouse关于数组数据类型](https://clickhouse.tech/docs/en/sql-reference/data-types/array/) 的文档。同时，我们将开始把数组用于解决分析问题。

## 可变数据建模

像键值对列表（也就是 dict/map）这样的可变数据结构会反复出现在我们日常的分析场景中，特别是那些涉及时间序列数据的问题。

以监测运行公共云的虚拟机为例。特定的虚拟机有我们想要测量的不同属性（如SSD存储的特定值），以及因操作虚拟机的团队而不同的标签（如应用程序类型）。因此，每条监控记录包含两个键值列表，其键值可能在不同的虚拟机之间和随着时间的推移而改变。

我们可以用一对数组来表示每个键值列表。一个数组提供属性名称，另一个数组提供相同数组索引的值。下面是我们如何在表定义中模拟虚拟机监控数据。因为有两种类型的键值，所以有两组数组：一个用于度量数据，另一个用于标签数据。

```sql
CREATE TABLE vm_data (
  datetime DateTime,  
  date Date default toDate(datetime),
  vm_id UInt32,  
  vm_type LowCardinality(String),  
  metrics_name Array(String),
  metrics_value Array(Float64),
  tags_name Array(String),  
  tags_value Array(String)
)
ENGINE = MergeTree()
PARTITION BY date 
ORDER BY (vm_type, vm_id, datetime)
```

你可以直接使用嵌套的JSON结构加载数组，如下面所示的格式化好的：

```json
[ { 
  "datetime": "2020-09-03 00:00:10",
  "vm_id": 6220,
  "vm_type": "m5.large",
  "metrics_name": [ "usage_idle", "ebs1_cap_gib", "ebs1_used_gib" ],
  "metrics_value": [ 80.2, 10.0, 7.6 ],
  "tags_name": [ "name", "group" ],
  "tags_value": [ "sfg-prod-01", "rtb" ]
},{
{
  "datetime": "2020-09-03 00:00:12",
  "vm_id": 6221,
  "vm_type": "m5ad.xlarge",
  "metrics_name": [ "usage_idle", "ssd1_cap_gib", "ssd1_used_gib" ],
  "metrics_value": [ 59.19, 75.0, 21.9 ],
  "tags_name": [ "name", "group", "owner" ],
  "tags_value": [ "mt-prod-65", "marketing", "casey" ]
} ]
```

我们使用以下命令来加载我们的样本数据到ClickHouse。 `jq` 将记录从JSON数组中剥离出来，并将每个记录放在一个单行上，以符合 ClickHouse JSONEachRow 的输入格式：

```bash
cat vm_data.json |jq -c .[] | clickhouse-client --database arrays \
--query="INSERT INTO vm_data FORMAT JSONEachRow"
```

一旦数据被加载，我们就可以使用SQL对其进行操作。我们将从选择单个记录开始：

```sql
SELECT *
FROM vm_data
LIMIT 1\G

/* sql answer*/
Row 1:
──────
datetime:      2020-09-03 00:00:10
date:          2020-09-03
vm_id:         6220
vm_type:       m5.large
metrics_name:  ['usage_idle','ebs1_cap_gib','ebs1_used_gib']
metrics_value: [80.2,10,7.6]
tags_name:     ['name','group']
tags_value:    ['sfg-prod-01','rtb']
```

正如上文提到的，ClickHouse提供了大量的数组函数来直接处理数组中的数据。

例如，这里有一个快速查找缺少 "name"、"group"和 "owner"标签的任何VM的方法。 我们可以使用hasAll()函数，它可以验证第一个数组参数是否包含第二个参数所定义的数值子集。

```sql
WITH ['name', 'group', 'owner'] AS required_tags
SELECT distinct vm_id
FROM vm_data
WHERE NOT hasAll(tags_name, required_tags)
```

注意到：`WITH` 来定义一个数组用于查询。 这是一个通用表表达式或CTE的例子。

CTEs通过从主查询中移除常量表达式来帮助降低查询的复杂性，是ClickHouse的最佳实践。我们将在其他例子中使用它们来保持事情的可读性。

ClickHouse的数组函数是相当多样的，涵盖了广泛的使用情况。下面是如何寻找 "group"标签值为 "rtb" 虚拟机的名称。正如你可能猜到的，indexOf()函数返回一个值的索引。我们可以用它来引用另一个数组中的值，这允许我们在tags_name和tags_value数组之间建立数值关系。

```sql
SELECT distinct vm_type
FROM vm_data
WHERE tags_value[indexOf(tags_name, 'group')] = 'rtb'
```

## 数组与table JOIN

直接使用数组函数来定位和处理标签值可能会很麻烦，特别是在跨数组的几列工作时。 幸运的是，ClickHouse有一个非常方便的 `**ARRAY JOIN**`，可以很容易地将数组值 "解卷" 到一个名称值对表中。 下面是一个使用 `**ARRAY JOIN`** 的例子：

```sql
SELECT date, vm_id, vm_type, name, value
FROM vm_data
ARRAY JOIN tags_name AS name, tags_value AS value
ORDER BY date, vm_id, name
```

ARRAY JOIN的工作方式如下：

左侧表 vm_data 的列（date, vm_id, vm_type）与 ARRAY JOIN（tags_name, tags_value）后列出的数组中的值 "连接"。
ClickHouse为每个列出的数组创建一个列，并以相同的顺序从每个数组中填充值。 结果看起来像下面这样。

```sql
┌───────date─┬─vm_id─┬─vm_type─────┬─name──┬─value───────┐
│ 2020-09-03 │  6220 │ m5.large    │ group │ rtb         │
│ 2020-09-03 │  6220 │ m5.large    │ name  │ sfg-prod-01 │
│ 2020-09-03 │  6221 │ m5ad.xlarge │ group │ marketing   │
│ 2020-09-03 │  6221 │ m5ad.xlarge │ name  │ mt-prod-65  │
│ 2020-09-03 │  6221 │ m5ad.xlarge │ owner │ casey       │
└────────────┴───────┴─────────────┴───────┴─────────────┘
```

ClickHouse的文档中有一篇关于 [ARRAY JOIN的文章](https://clickhouse.com/docs/en/sql-reference/statements/select/array-join/)，说明了它的灵活性。 

这里有一个例子：下面的查询添加了一个序列号，并使用方便的 `**arrayEnumerate()**` 按照数组序列顺序对行进行排序，该函数以升序返回数组索引值。

```sql
SELECT date, vm_id, vm_type, name, value, seq
FROM vm_data
ARRAY JOIN
  tags_name AS name,
  tags_value AS value,
  arrayEnumerate(tags_name) AS seq
ORDER BY date, vm_id, seq

/* sql answer*/
┌───────date─┬─vm_id─┬─vm_type─────┬─name──┬─value───────┬─seq─┐
│ 2020-09-03 │  6220 │ m5.large    │ name  │ sfg-prod-01 │   1 │
│ 2020-09-03 │  6220 │ m5.large    │ group │ rtb         │   2 │
│ 2020-09-03 │  6221 │ m5ad.xlarge │ name  │ mt-prod-65  │   1 │
│ 2020-09-03 │  6221 │ m5ad.xlarge │ group │ marketing   │   2 │
│ 2020-09-03 │  6221 │ m5ad.xlarge │ owner │ casey       │   3 │
└────────────┴───────┴─────────────┴───────┴─────────────┴─────┘
```

ARRAY JOIN 对于呈现输出很有帮助，因为包含数组的查询结果对人类来说很难阅读，而且可能需要在客户端应用程序中进行专门的反序列化逻辑。它对降低查询的复杂性也很有帮助。 

使用ARRAY JOIN，我们可以最小化甚至消除数组函数表达式。下面的例子是对前面的例子的重写，以寻找 "group rtb" 使用的虚拟机类型：

```sql
SELECT distinct vm_type FROM (
  SELECT date, vm_id, vm_type, name, value
  FROM vm_data
  ARRAY JOIN tags_name AS name, tags_value AS value
  WHERE name = 'group' AND value = 'rtb'
)
```

如果不提及 arrayJoin()，就无法结束我们对使用数组的数据建模的介绍。这个函数可以被添加到SELECT 列表中，以产生未滚动的结果，如下例所示：

```sql
SELECT 1, 2, arrayJoin(['a', 'b']) AS a1

/* sql answer*/
┌─1─┬─2─┬─a1─┐
│ 1 │ 2 │ a  │
│ 1 │ 2 │ b  │
└───┴───┴────┘
```

这完全等同于以下带有ARRAY JOIN的查询:

```sql
SELECT 1, 2 FROM system.one ARRAY JOIN ['a', 'b'] AS a1
```

然而，有一个关键的区别。

正如我们在上面看到的，ARRAY JOIN允许多个数组，并在所有数组上并行展开数值。 **`arrayJoin()`** 行为则不同。如果有多个 **`arrayJoin()`** 调用，它们会产生如下的结果:

```sql
SELECT  1,  2, 
  arrayJoin(['a', 'b']) AS a1, arrayJoin(['i', 'ii']) AS a2

/* sql answer*/
┌─1─┬─2─┬─a1─┬─a2─┐
│ 1 │ 2 │ a  │ i  │
│ 1 │ 2 │ a  │ ii │
│ 1 │ 2 │ b  │ i  │
│ 1 │ 2 │ b  │ ii │
└───┴───┴────┴────┘
```

正如你所看到的，结果是数组值的笛卡尔乘积，这可能不是你想要的结果。在本文的其余部分，我们将重点讨论ARRAY JOIN，因为它允许我们处理具有相关值的数组。这种行为对于数组的更多高级应用是至关重要的。

## 结论

你刚才读的文章介绍了数组在ClickHouse中的基本用法。 我们展示了如何使用成对的数组来表示变量数据，如何使用数组函数来提取数据，以及如何使用 **`ARRAY JOIN`** 和 **`arrayJoin()`** 来连接数组与表的行。

我们所涉及的数组功能已经超越了许多SQL数据库的能力。 对于ClickHouse来说，这仅仅是一个开始。在下一篇文章中，我们将展示数组和SQL GROUP BY是如何紧密相连的。阵列和聚合之间的整合使用户能够识别事件的序列，以及建立漏斗，从而跟踪营销、销售和其他领域的预期目标的进展。这是一个重要的分析工具，可用于广泛的有趣的应用。