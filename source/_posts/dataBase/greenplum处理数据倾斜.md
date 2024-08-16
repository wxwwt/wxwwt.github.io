---
title: greenplum处理数据倾斜
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: dataBase
---
greenplum处理数据倾斜

# 前言：

最近遇到了几次数据倾斜，导致查询很慢的经历，也是头一次知道这个概念和问题，查找资料后处理了该问题，记录一下数据倾斜的问题和处理过程，希望也能帮助到其他人了解gp和数据倾斜。



# 数据倾斜：

首先我们要了解数据倾斜是什么？我们知道gp是mpp数据库，它的存储架构如下图：

![](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/gp%E9%9B%86%E7%BE%A4%E5%8C%96.png)





![](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/gp%E6%95%B0%E6%8D%AE%E5%82%A8%E5%AD%98.png)

从上面的图可以知道， gp由一个coordinator和多个segment组成，coordinator负责分发和汇总请求，segment负责执行。数据也是储存在segment之上，在每个segment计算完毕后，将结果给coordinator汇总在返回给客户端，返回给用户。因此，每个segment的数据量越均匀越好，因为coordinator返回结果的时长是由最慢的那个节点决定的，这就是木桶效应，最短的那个木板决定了这个水桶能装多少水，其他模木板在高也没有用。

举个例子，100万的数据，4个segment，假设每个segment处理10万的数据要1秒，如果数据90%都是在同一个segmentA上，处理要花9秒，那么10%的数据很快就在其他segment上处理完了，花费时间小于1秒，而segmentA因为数据量大还在处理，请求就会一直阻塞，直到segmentA处理完才返回，一共要花9秒。

如果数据是均匀的，每个segment是分到25万的数据，每个segment同时处理一共只要花2.5秒。现在是不是很容易理解为什么数据要分布越均匀越好了呢？然后当我们在gp中创建一张表的时候，可以选择哪一列，或者哪几列组合起来做为分布键，如果不指定的话，gp就会随机指定一列做为分布键。因为是随机的，所以很容易出现数据倾斜的问题。



# 数据倾斜处理：

以下内容基于greenplum6.16

gp给我们提供了两个视图来查看数据倾斜：

- gp_toolkit.gp_skew_coefficients view通过计算存储在每个节点上的数据的变异系数（CV）来显示数据分布倾斜。 skccoeff列显示变异系数（CV），计算为标准差除以平均值。 它考虑了数据系列平均值附近的平均值和可变性。 值越低越好。 值越高表示数据倾斜越大。

- gp_toolkit.gp_skew_idle_fractions view通过计算表扫描期间空闲的系统百分比来显示数据分布倾斜，这是计算倾斜的指示。 siffraction列显示在表扫描期间空闲的系统百分比。 这是数据分布不均匀或查询处理倾斜的指标。 例如，值0.1表示10％偏斜，值0.5表示50％偏斜，依此类推。 倾斜超过10％的表应评估其分配策略。

  ```
  查看数据变异系数：是个数值，越小越好
  SELECT * FROM gp_toolkit.gp_skew_coefficients WHERE skcrelname = 'table_name'; 
  查看数据倾斜指标：是个小数，可以看成百分比，越小越好
  SELECT * FROM gp_toolkit.gp_skew_idle_fractions WHERE sifrelname = 'table_name';
  
  查看数据分布情况：
  select gp_segment_id,count(*) from table_name group by gp_segment_id;
  
  修改分布键平衡表数据：
  alter table table_name set distributed by (col1,col2...);
  ```

  在发现数据倾斜之后，可以根据业务情况来重建分布键，比如用常用来筛选的字段，join条件等。也可以计算一下每个列的数据重复度distinct，选取数据重复度越小的字段越好。

  

  # 实例

这是我之前遇到的一个实际的例子，是一张超过千万的表，查询会超过一分种，本来这张表是用的随机分布键，使用的一个distinct值就2-3项的一个列。直接查询数据倾斜指标达到了93%，因为原表经常会使用到时间来筛选数据，而且这个时间列的dintinct值很大，选了一个时间列做为分布键，再去查看数据倾斜指标就降了很多。

  ![](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/%E6%95%B0%E6%8D%AE%E5%80%BE%E6%96%9C%E5%9B%BE%E7%89%87.png)

![](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/%E6%95%B0%E6%8D%AE%E5%80%BE%E6%96%9C%E5%9B%BE%E7%89%872.png)

修改后大概是7秒多，虽然还是不快，但是相比之前要超过一分种，已经降了很多，可以证明数据倾斜的优化还是很有效果的。

# 参考资料
[1.分布与倾斜](https://docs-cn.greenplum.org/v6/admin_guide/distribution.html)
[2.GreenPlum中性能调优之数据倾斜](https://blog.csdn.net/wangning0714/article/details/130704775)
[3.greenplum内核揭秘](https://www.slidestalk.com/w/710)
[4.greenplum架构最详细解读](https://cn.greenplum.org/greenplum_architecture/)
