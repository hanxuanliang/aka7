# 「Clickhouse Array 的力量」2

> 原文：[https://altinity.com/blog/harnessing-the-power-of-clickhouse-arrays-part-2](https://altinity.com/blog/harnessing-the-power-of-clickhouse-arrays-part-2)

上篇文章阐述了基本的数组行为：我们介绍了基本的数组语法，使用数组来模拟键值对，以及如何使用ARRAY JOIN将数组的值展开到表中。正如我们所指出的，这些功能已经为用户提供了巨大的力量，但还有更多的东西。

在当前的文章中，我们将挖掘数组和GROUP BY子句之间的整合。这种整合为用户提供了解决问题的工具，如识别事件序列和进行漏斗分析。漏斗分析是一种重要的技术，它可以衡量朝向特定目标的进展，例如让用户在网站上购买产品。

我们将展示它是如何使用数组工作的，然后展示使用 `**ClickHouse windowFunnel()**` 的替代方法。

## 构建 **sequences**

跟踪序列(***可以理解为埋点***)是分析应用中的一个常见问题。它出现在许多用例中，从跟踪用户通过在线服务的路径到计算飞机的行程。在本节中，我们将探讨如何使用数组来跟踪事件的序列。**我们将寻求解决以下问题：显示一架商业飞机在一天内完成的最长行程。**

我们的数据集是 [流行的航空公司准点率数据](https://www.transtats.bts.gov/Tables.asp?DB_ID=120)，该数据集可用于ClickHouse。它可以按照[ClickHouse文档中的说明下载](https://clickhouse.com/docs/en/getting-started/example-datasets/ontime/)。其结果是一个名为 "ontime"的 table，其中包含美国出发地和目的地机场之间的每个商业航空公司航班的一行。

为了追踪一架飞机在一天内穿越的路径，我们需要找到该飞机的所有航班，将他们排序，然后计算由此产生的跳数来进行排序。飞机由其尾号来识别。让我们先算出任何飞机的最大跳数。这不需要数组，用GROUP BY就可以轻松做到。下面是一个例子，找到在2017年1月15日飞行次数最多的飞机：

```sql
SELECT Carrier, TailNum, count(*) AS Hops
FROM ontime
WHERE (FlightDate = toDate('2017-01-15')) AND (DepTime < ArrTime)
GROUP BY Carrier, TailNum
ORDER BY Hops DESC, Carrier, TailNum
LIMIT 5

/* sql answer*/
┌─Carrier─┬─TailNum─┬─Hops─┐
│ HA      │ N488HA  │   14 │
│ HA      │ N492HA  │   14 │
│ HA      │ N493HA  │   14 │
│ HA      │ N483HA  │   12 │
│ HA      │ N489HA  │   12 │
└─────────┴─────────┴──────┘
```

赢家实际上有3个: N488HA、N492HA和N493HA。所有飞机都属于夏威夷航空公司。美国的飞机迷们不会感到惊讶：夏威夷航空公司在岛屿之间运营短途航班，机场之间的距离只有50英里(为了增加乐趣，你可以在这里查询飞机尾号，以了解有关飞机的更多信息)。

这很容易，但我们还远远没有解决整个问题。获胜的行程实际上是什么样子的？在这一点上，我们需要引入数组。让我们只从问题的一部分开始，加入每一跳的出发时间。

```sql
SELECT Carrier, TailNum,
  groupArray(DepTime) as Departures,
  length(groupArray(DepTime)) as Hops
FROM ontime
WHERE (FlightDate = toDate('2017-01-15')) AND (DepTime < ArrTime)
GROUP BY Carrier, TailNum
ORDER BY Hops DESC, Carrier, TailNum
LIMIT 5

/* sql answer*/
┌─Carrier─┬─TailNum─┬─Departures───────────────────────────┬─Hops─┐
│ HA      │ N488HA  │ [1157,1813,1921,1042,658,544,170...] │   14 │
│ HA      │ N492HA  │ [718,824,938,1512,1622,1056,1743...] │   14 │
│ HA      │ N493HA  │ [627,728,1012,1647,1801,904,1523...] │   14 │
│ HA      │ N483HA  │ [845,725,1800,1904,1259,1407,113...] │   12 │
│ HA      │ N489HA  │ [2047,1936,1031,1602,1438,1256,9...] │   12 │
└─────────┴─────────┴──────────────────────────────────────┴──────┘
```

这个查询引入了groupArray()，它是一个聚合函数。 

它收集每个GROUP BY子集的值，并按照处理行的顺序将它们添加到数组中。如果你在不同的列上调用groupArray()，它将为每个列创建一个数组。最重要的是，生成的数组将始终是相同的大小，并且有相同的顺序。**换句话说，groupArray()是我们在上一篇博客文章中学习的ARRAY JOIN操作符的逆向操作(这句话很重要：说白了 → 一个是列转行，一个是行转列)**

构建飞行路径的工作比较多。首先，我们只有出发时间，没有机场。第二，出发时间没有被排序。 让我们在下一个例子中解决这个问题，它增加了更多的数组，并按出发时间顺序进行排序：

```sql
SELECT Carrier, TailNum,
  arraySort(groupArray(DepTime)) as Departures,
  arraySort((x,y) -> y, groupArray(Origin), groupArray(DepTime)) as Origins,
  length(groupArray(DepTime)) as Hops
FROM ontime WHERE (FlightDate = toDate('2017-01-15')) AND (DepTime < ArrTime)
GROUP BY Carrier, TailNum
ORDER BY Hops DESC, Carrier, TailNum
LIMIT 5 \G

/* sql answer*/
Row 1:
──────
Carrier:    HA
TailNum:    N488HA
Departures: [544,658,818,930,1042,1157,1322,1443,1556,1708,1813,1921,2025,2126]
Origins:    ['HNL','KOA','HNL','LIH','OGG','LIH','HNL','OGG','LIH','OGG','HNL','OGG','HNL','OGG']
Hops:       14
...
```

如上面所示，我们依靠的是一个新的、非常通用的函数：arraySort()。

前面的查询说明了它的不同使用方法。在最简单的形式下，arraySort()接收一个数组参数并返回排序后的值。下面是一个例子。

```sql
SELECT arraySort([1224, 1923, 1003, 745]) AS s

/* sql answer*/
┌─s────────────────────┐
│ [745,1003,1224,1923] │
└──────────────────────┘
```

我们可以通过添加一个lambda表达式来使事情变得更加有趣，该表达式指示arraySort如何处理它遇到的每个数组值。

如果存在的话，**lambda表达式总是第一个参数**。下面的例子使用一个lambda表达式来扭转排序顺序。这个lambda表达式被应用于每个连续的数组值，ClickHouse使用所得到的值作为数组中的值的排序键。

```sql
SELECT arraySort(x -> (-x), [1224, 1923, 1003, 745]) AS s

/* sql answer*/
┌─s────────────────────┐
│ [1923,1224,1003,745] │
└──────────────────────┘
```

lambda并不总是必要的。你可以使用另外一种方法：arrayReverseSort()。下面的查询返回完全相同的答案:

```sql
SELECT arrayReverseSort([1224, 1923, 1003, 745]) AS s
```

最后，我们可以完全添加一个不同的键(***另外一个是指定顺序的键。可以理解另类的排序键***)。下面的例子有两个数组，用一个lambda表达式来挑选第二个数组作为排序键。ClickHouse将使用第二个数组的值作为键对第一个数组进行排序。

```sql
SELECT
  arraySort((x, y) -> y, [1224, 1923, 1003, 745], [2, 1, 3, 4]) AS s

/* sql answer*/
┌─s────────────────────┐
│ [1923,1224,1003,745] │
└──────────────────────┘
```

**总结一下，groupArray()从分组的值中建立数组，sortArray()对这些值进行排序，用一个数组作为其他数组的排序键**。下面剩下的就是构建完整的查询。我们将使用ARRAY JOIN将我们的结果以一种更可读的格式展开。

```sql
SELECT Carrier, TailNum, Depart, Arr, Origin, Dest
FROM
(
  SELECT Carrier, TailNum,
    arraySort(groupArray(DepTime)) AS Departures,
    arraySort(groupArray(ArrTime)) AS Arrivals,
    arraySort((x, y) -> y, groupArray(Origin), groupArray(ArrTime)) AS Origins,
    arraySort((x, y) -> y, groupArray(Dest), groupArray(ArrTime)) AS Destinations,
    length(groupArray(DepTime)) AS Hops
  FROM ontime
  WHERE (FlightDate = toDate('2017-01-15')) AND (DepTime < ArrTime)
  GROUP BY Carrier, TailNum
  ORDER BY Hops DESC, Carrier, TailNum
)
ARRAY JOIN Departures AS Depart, Arrivals AS Arr,
    Origins AS Origin, Destinations AS Dest
LIMIT 15

/* sql answer*/
┌─Carrier─┬─TailNum─┬─Depart─┬──Arr─┬─Origin─┬─Dest─┐
│ HA      │ N488HA  │    544 │  632 │ HNL    │ KOA  │
│ HA      │ N488HA  │    658 │  749 │ KOA    │ HNL  │
│ HA      │ N488HA  │    818 │  853 │ HNL    │ LIH  │
│ HA      │ N488HA  │    930 │ 1020 │ LIH    │ OGG  │
│ HA      │ N488HA  │   1042 │ 1126 │ OGG    │ LIH  │
│ HA      │ N488HA  │   1157 │ 1229 │ LIH    │ HNL  │
│ HA      │ N488HA  │   1322 │ 1406 │ HNL    │ OGG  │
│ HA      │ N488HA  │   1443 │ 1535 │ OGG    │ LIH  │
│ HA      │ N488HA  │   1556 │ 1646 │ LIH    │ OGG  │
│ HA      │ N488HA  │   1708 │ 1742 │ OGG    │ HNL  │
│ HA      │ N488HA  │   1813 │ 1857 │ HNL    │ OGG  │
│ HA      │ N488HA  │   1921 │ 1958 │ OGG    │ HNL  │
│ HA      │ N488HA  │   2025 │ 2107 │ HNL    │ OGG  │
│ HA      │ N488HA  │   2126 │ 2207 │ OGG    │ HNL  │
│ HA      │ N492HA  │    501 │  542 │ HNL    │ LIH  │
└─────────┴─────────┴────────┴──────┴────────┴──────┘
```

很高兴看到我们的N488HA飞机的始发机场和出发机场正确排列。

## 漏斗分析

正如介绍中提到的，漏斗分析是一种重要的技术，可以在时间序列数据中跟踪目标的进展。我们可以使用漏斗来回答以下问题：

1. 有多少网站客户把东西放进购物车，然后在同一时段购买？
2. 营销线索是如何从联系人发展到注册客户的？
3. 有多少飞机在同一天内飞往芝加哥，然后飞往亚特兰大？

既然我们有航班数据，我们将用它来建立一个漏斗，以回答最后一个问题。为了增加趣味性，我们将在整个数据集中按年份汇总结果。读取更多的数据使我们更容易评估性能。

就像之前的跟踪序列的例子一样，我们将从一个更简单的查询开始，然后增加功能，直到我们有一个解决方案。我们最初的查询生成了一个按时间排序的每架飞机的每日目的地列表。为了保持快速，我们将只看单日的数据。

```sql
SELECT FlightDate, Carrier, TailNum,
    arraySort(groupArray(ArrTime)) AS Arrivals,
    arraySort((x, y) -> y, groupArray(Dest), groupArray(ArrTime)) AS Dests
  FROM ontime
  WHERE DepTime < ArrTime AND TailNum != ''
    AND toYYYYMM(FlightDate) = toYYYYMM(toDate('2017-01-01'))
  GROUP BY FlightDate, Carrier, TailNum
  ORDER BY FlightDate, Carrier, TailNum
LIMIT 5\G

/* sql answer*/
Row 1:
──────
FlightDate: 2017-01-01
Carrier:    AA
TailNum:    N001AA
Arrivals:   [1021,2213]
Dests:      ['MIA','ATL']
. . .
```

我们最初的查询给了我们正确排序的列表，但我们现在需要寻找经过芝加哥的航班。我们将搜索限制在奥黑尔（'ORD'）和中途岛（'MDW'）上，这两个机场是服务于芝加哥市的主要机场。为了更快地检查结果，我们将添加一个HAVING子句，以便我们只看包括在芝加哥停留的航线。

```sql
SELECT FlightDate, Carrier, TailNum,
    arraySort(groupArray(ArrTime)) AS Arrivals,
    arraySort((x, y) -> y, groupArray(Dest), groupArray(ArrTime)) AS Dests,
    arrayFilter((arr, dest) -> (dest in ('MDW', 'ORD')), Arrivals, Dests)[1] as CHI_Arr,
    length(Arrivals) as Hops
  FROM ontime
  WHERE DepTime < ArrTime AND TailNum != ''
    AND toYYYYMM(FlightDate) = toYYYYMM(toDate('2017-01-01'))
  GROUP BY FlightDate, Carrier, TailNum
  HAVING CHI_Arr > 0
  ORDER BY FlightDate, Carrier, TailNum
LIMIT 5 \G

/* sql answer*/
Row 1:
──────
FlightDate: 2017-01-01
Carrier:    AA
TailNum:    N005AA
Arrivals:   [1238,1757,2158]
Dests:      ['JAC','ORD','ATL']
CHI_Arr:    1757
Hops:       3
. . .
```

我们的示例查询依赖于一个新的函数：**`arrayFilter()`**，它接受两个或多个参数。**第一个是一个lambda表达式，其余的参数是数组。**arrayFilter() 返回第一个数组参数中的元素数组，其中λ表达式的结果为非零。下面是如何返回长于一个字符的字符串数组元素:

```sql
WITH ['a', 'bc', 'def', 'g'] AS array
SELECT arrayFilter(v -> (length(v) > 1), array) AS filtered

/* sql answer*/
┌─filtered─────┐
│ ['bc','def'] │
└──────────────┘
```

这里是另一个例子，它处理成对的数组以返回与标签'name'对应的第一个值。可能有多个这样的标签，但是我们使用[1]索引来返回结果数组中的第一个标签：

```sql
WITH
    ['name', 'owner', 'account'] AS tags,
    ['joe', 'susie', 'website'] AS values
SELECT arrayFilter((v, t) -> (t = 'name'), values, tags)[1] AS name

/* sql answer*/
┌─name─┐
│ joe  │
└──────┘
```