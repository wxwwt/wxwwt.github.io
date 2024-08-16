---
title: Trino dynamic filtering（动态过滤）
date: 2023-12-31 21:36:04
updated: 2024-08-15 22:35:58
tags: leetcode
---
 Trino dynamic filtering（动态过滤）

# 前言：
最近看了下trino的文档，看到里面有提到一个概念，叫dynamic filtering 动态过滤。就想了解下它是怎么一个动态法呢？看了trino的官方文档和一些其他引擎其他相关的动态过滤的资料，写一点自己的理解。



# 介绍：

## 概念：

Dynamic filtering optimizations significantly improve the performance of queries with selective joins by avoiding reading of data that would be filtered by join condition.

这个是官方文档描述动态过滤的第一句话，翻译过来就是动态过滤优化通过减少了join中读取了不必要的数据，显著的提高了查询性能。我们就可以知道动态过滤作用是提高查询性能的，然后通过它自己的一系列优化，能让sql少去读取一些不会用到的数据。数据量减少，传输和计算就意味着减少，那么性能自然就提高了，这也是sql优化的核心宗旨，减少用不到的数据量。

## 执行过程：

我们再来看下它是怎么个动态法? 官网也提供了一个例子：

```
SELECT count(*) FROM store_sales JOIN date_dim ON store_sales.ss_sold_date_sk = date_dim.d_date_sk WHERE d_following_holiday=’Y’ AND d_year = 2000;
```

看这个sql，我们知道是store_sales（就简称为销售表把），join了另外一张date_dim（日期表），并且对日期表有两个谓词表达式，这里官网并没有说d_following_holiday，d_year是日期表的字段，但是能看出来这两个是对时间进行筛选的，而且是d_大概率是日期维度表的。如果没有动态过滤的情况下，sql的谓词会被推送到日期表，但是销售表会被全表扫描，官网也说了销售表是事实表，那么一般情况下，事实表的数据量是很大的，至少比维度表大很多，因为最后sql不会用销售表的全部数据，所以这里其实会有很多计算和数据传输被浪费掉。

如果启用了动态过滤，那么trino会从join右侧的维度表里面收集连接条件，也就是store_sales.ss_sold_date_sk = date_dim.d_date_sk的数据，coordinator会将连接条件就生成运行时的谓词，可能像是store_sales.ss_sold_date_sk in (xxxx)之类的，推送给各个执行的worker，这样每个worker执行传输计算的数据量就都会减少，性能就得到了提高。

## 动态过滤开启/关闭：

从coordinator到work的动态过滤是默认启用的。

可以通过将enable-coordinator-dynamic-filters-distribution 配置属性或会话属性enable_coordinator_dynamic_filters_distribution 设置为 false 来禁用它

## 影响因素：

- Planner 支持 Trino 中给定连接操作的动态过滤。目前支持使用 =、<、<=、>、>= 或 IS NOT DISTINCT FROM 连接条件进行内连接和右连接，并支持使用 IN 条件的半连接。

- 连接器支持利用在运行时推入表扫描的动态过滤器。例如，Hive 连接器可以将动态过滤器推送到 ORC 和 Parquet 读取器中，以执行条带或行组修剪。

- connector支持在拆分枚举阶段使用动态过滤器

- join右表的数据量

  

## 查看动态过滤的方式

  

  1.官网举了一个例子，用来查看指定的sql是否有添加动态过滤器，sql如下：

  ```
  EXPLAIN
  SELECT count(*)
  FROM store_sales
  JOIN date_dim ON store_sales.ss_sold_date_sk = date_dim.d_date_sk
  WHERE d_following_holiday='Y' AND d_year = 2000;
  ```

  返回的结果：

  ```
  Fragment 1 [SOURCE]
      Output layout: [count_3]
      Output partitioning: SINGLE []
      Aggregate(PARTIAL)
      │   Layout: [count_3:bigint]
      │   count_3 := count(*)
      └─ InnerJoin[(""ss_sold_date_sk"" = ""d_date_sk"")][$hashvalue, $hashvalue_4]
         │   Layout: []
         │   Estimates: {rows: 0 (0B), cpu: 0, memory: 0B, network: 0B}
         │   Distribution: REPLICATED
         │   dynamicFilterAssignments = {d_date_sk -> #df_370}
         ├─ ScanFilterProject[table = hive:default:store_sales, grouped = false, filterPredicate = true, dynamicFilters = {""ss_sold_date_sk"" = #df_370}]
         │      Layout: [ss_sold_date_sk:bigint, $hashvalue:bigint]
         │      Estimates: {rows: 0 (0B), cpu: 0, memory: 0B, network: 0B}/{rows: 0 (0B), cpu: 0, memory: 0B, network: 0B}/{rows: 0 (0B), cpu: 0, memory: 0B, network: 0B}
         │      $hashvalue := combine_hash(bigint '0', COALESCE(""$operator$hash_code""(""ss_sold_date_sk""), 0))
         │      ss_sold_date_sk := ss_sold_date_sk:bigint:REGULAR
         └─ LocalExchange[HASH][$hashvalue_4] (""d_date_sk"")
            │   Layout: [d_date_sk:bigint, $hashvalue_4:bigint]
            │   Estimates: {rows: 0 (0B), cpu: 0, memory: 0B, network: 0B}
            └─ RemoteSource[2]
                   Layout: [d_date_sk:bigint, $hashvalue_5:bigint]
  
  Fragment 2 [SOURCE]
      Output layout: [d_date_sk, $hashvalue_6]
      Output partitioning: BROADCAST []
      ScanFilterProject[table = hive:default:date_dim, grouped = false, filterPredicate = ((""d_following_holiday"" = CAST('Y' AS char(1))) AND (""d_year"" = 2000))]
          Layout: [d_date_sk:bigint, $hashvalue_6:bigint]
          Estimates: {rows: 0 (0B), cpu: 0, memory: 0B, network: 0B}/{rows: 0 (0B), cpu: 0, memory: 0B, network: 0B}/{rows: 0 (0B), cpu: 0, memory: 0B, network: 0B}
          $hashvalue_6 := combine_hash(bigint '0', COALESCE(""$operator$hash_code""(""d_date_sk""), 0))
          d_following_holiday := d_following_holiday:char(1):REGULAR
          d_date_sk := d_date_sk:bigint:REGULAR
          d_year := d_year:int:REGULAR
  ```

  可以看到Fragement1InnerJoin里面有一个dynamicFilterAssignments，这里就是说明使用了动态过滤，会将d_date_sk的谓词推送到连接器里面。

2.在webUI界面也可以看到对应动态过滤情况，在coordinator的queryStats里面，有一个dynamicFiltersStats的对象。就像下面展示的那样：

  ```
  "dynamicFiltersStats" : {
        "dynamicFilterDomainStats" : [ {
          "dynamicFilterId" : "df_370",
          "simplifiedDomain" : "[ SortedRangeSet[type=bigint, ranges=3, {[2451546], ..., [2451905]}] ]",
          "collectionDuration" : "2.34s"
        } ],
        "lazyDynamicFilters" : 1,
        "replicatedDynamicFilters" : 1,
        "totalDynamicFilters" : 1,
        "dynamicFiltersCompleted" : 1
  }
  ```

3.还可以通过查看该表扫描的统计信息来查看将动态过滤器下推到work节点上的表扫描中。 dynamicFilterSplitsProcessed 记录动态过滤器下推到表扫描后处理的拆分数量

```
"operatorType" : "ScanFilterAndProjectOperator",
"totalDrivers" : 1,
"addInputCalls" : 762,
"addInputWall" : "0.00ns",
"addInputCpu" : "0.00ns",
"physicalInputDataSize" : "0B",
"physicalInputPositions" : 28800991,
"inputPositions" : 28800991,
"dynamicFilterSplitsProcessed" : 1,
```

4.可以使用EXPLAIN ANALYZE来查看动态过滤信息，可以看到里面有个dynamicFilterAssignments的属性，跟上面说到的EXPLAIN方式差不多

```
...

 └─ InnerJoin[("ss_sold_date_sk" = "d_date_sk")][$hashvalue, $hashvalue_4]
    │   Layout: []
    │   Estimates: {rows: 11859 (0B), cpu: 8.84M, memory: 3.19kB, network: 3.19kB}
    │   CPU: 78.00ms (30.00%), Scheduled: 295.00ms (47.05%), Output: 296 rows (0B)
    │   Left (probe) Input avg.: 120527.00 rows, Input std.dev.: 0.00%
    │   Right (build) Input avg.: 0.19 rows, Input std.dev.: 208.17%
    │   Distribution: REPLICATED
    │   dynamicFilterAssignments = {d_date_sk -> #df_370}
    ├─ ScanFilterProject[table = hive:default:store_sales, grouped = false, filterPredicate = true, dynamicFilters = {"ss_sold_date_sk" = #df_370}]
    │      Layout: [ss_sold_date_sk:bigint, $hashvalue:bigint]
    │      Estimates: {rows: 120527 (2.03MB), cpu: 1017.64k, memory: 0B, network: 0B}/{rows: 120527 (2.03MB), cpu: 1.99M, memory: 0B, network: 0B}/{rows: 120527 (2.03MB), cpu: 4.02M, memory: 0B, network: 0B}
    │      CPU: 49.00ms (18.85%), Scheduled: 123.00ms (19.62%), Output: 120527 rows (2.07MB)
    │      Input avg.: 120527.00 rows, Input std.dev.: 0.00%
    │      $hashvalue := combine_hash(bigint '0', COALESCE("$operator$hash_code"("ss_sold_date_sk"), 0))
    │      ss_sold_date_sk := ss_sold_date_sk:bigint:REGULAR
    │      Input: 120527 rows (1.03MB), Filtered: 0.00%
    │      Dynamic filters:
    │          - df_370, [ SortedRangeSet[type=bigint, ranges=3, {[2451546], ..., [2451905]}] ], collection time=2.34s
    |
...
```





## 局限性：

- Min-max dynamic filter collection is not supported for `DOUBLE`, `REAL` and unorderable data types.

- Dynamic filtering is not supported for `DOUBLE` and `REAL` data types when using `IS NOT DISTINCT FROM` predicate.

- Dynamic filtering is supported when the join key contains a cast from the build key type to the probe key type. Dynamic filtering is also supported in limited scenarios when there is an implicit cast from the probe key type to the build key type. For example, dynamic filtering is supported when the build side key is of `DOUBLE` type and the probe side key is of `REAL` or `INTEGER` type.

- DOUBLE、REAL 和不可排序数据类型不支持最小-最大动态过滤器集合。

- 使用 IS NOT DISTINCT FROM 谓词时，DOUBLE 和 REAL 数据类型不支持动态过滤。

- 当连接键包含从构建键类型到探测键类型的转换时，支持动态过滤。当存在从探测密钥类型到构建密钥类型的隐式转换时，在有限的情况下也支持动态过滤。例如，当构建侧密钥为 DOUBLE 类型且探测侧密钥为 REAL 或 INTEGER 类型时，支持动态过滤

  

# 参考资料

[1.trino 动态过滤](https://trino.io/docs/current/admin/dynamic-filtering.html)

[2.阿里云动态过滤器](https://help.aliyun.com/zh/maxcompute/user-guide/dynamic-filtering)

