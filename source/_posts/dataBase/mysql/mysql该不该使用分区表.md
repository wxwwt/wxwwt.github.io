---
title: mysql该不该使用分区表
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: mysql
---

## 什么时候使用分区表？
使用分区表的场景：
1.表非常大，热点数据只在表的最后部分，其他部分都是不常用的历史数据
2.想批量删除大量数据，此时可以使用分区表，单独删除某些不在使用的分区
3.分区表的数据可以分布在不同的物理设备上，从而高效的利用多个硬件设备
4.可以使用分区表来避免某些特殊的瓶颈，比如innodb的单个索引的互斥访问，ext3文件系统的inode锁竞争等
5.需要备份和恢复独立的分区的时候，这个在大量数据集的场景下效果非常好


## 分区表的类型
分区表的类型总共有四种range，list，key，hash，但是partition的表达式返回的值必须是整数，且不能是一个常数。
如果不是用常数，使用的varchar，datetime类型的数据的话。会报错，报错信息大概长这样：
```
1659 - Field 'xxx' is of a not allowed type for this type of partitioning
```

## 分区表的使用
下面我们以range分区为例看一下，分区表的用法：
1.测试添加分区和删除分区

### 添加删除range分区
(1)创建一个分区：
```
CREATE TABLE table1 (
    id      INT NOT NULL,
    user_name       VARCHAR(50)     NOT NULL,
    date_create   DATETIME            NOT NULL,
    PRIMARY KEY (`id`,`user_name`,`date_create`)
) partition by range columns(date_create)
(partition p01 values less than ('1985-12-31'),
partition p02 values less than ('1990-12-31'),
partition p03 values less than ('1995-12-31'),
partition p04 values less than ('2000-12-31')，
partition p05 values less than (MAXVALUE)
);
```


(2)添加分区：
注意：最多不能超过p04的范围，然后新增的每个分区严格递增，即最小不能小于前一个分区
下面新曾了两个分区p06和p07
```
alter table table1
reorganize partition p04 into(
partition p06 values less than('1997-12-31'),
partition p07 values less than('1998-12-31'),
partition p04 values less than ('2000-12-31')
);
```

查看建表语句，发现已经增加了p06，p07的分区
```
SHOW CREATE TABLE table1;
```
```
 CREATE TABLE `table1` (
  `id` int(11) NOT NULL,
  `user_name` varchar(50) NOT NULL,
  `date_create` datetime NOT NULL,
  PRIMARY KEY (`id`,`user_name`,`date_create`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
/*!50500 PARTITION BY RANGE  COLUMNS(date_create)
(PARTITION p01 VALUES LESS THAN ('1985-12-31') ENGINE = InnoDB,
 PARTITION p02 VALUES LESS THAN ('1990-12-31') ENGINE = InnoDB,
 PARTITION p03 VALUES LESS THAN ('1995-12-31') ENGINE = InnoDB,
 PARTITION p06 VALUES LESS THAN ('1997-12-31') ENGINE = InnoDB,
 PARTITION p07 VALUES LESS THAN ('1998-12-31') ENGINE = InnoDB,
 PARTITION p04 VALUES LESS THAN ('2000-12-31') ENGINE = InnoDB,
 PARTITION p05 VALUES LESS THAN (MAXVALUE) ENGINE = InnoDB) */
 ```

(3)删除分区：
删除分区会吧原有在分区中的数据一次性删除掉，我们先在第一个分区中插入了几条数据，然后select看下。
```
mysql> select * from table1 where date_create < '1985-12-31';
+----+-----------+---------------------+
| id | user_name | date_create         |
+----+-----------+---------------------+
|  1 | test      | 1984-12-12 00:00:00 |
+----+-----------+---------------------+
```

```
mysql> alter table table1 drop partition p01;
Query OK, 0 rows affected (0.32 sec)
Records: 0  Duplicates: 0  Warnings: 0
```
```
mysql> select * from table1 where date_create < '1985-12-31';
Empty set
```
可以看到已经查询不到p01中的数据了。



## 分区表的限制和问题
### 限制：
1.一个表最多有1024个分区
2.myql5.1中分区表达式必须是整数或者整数表达式，不支持其他数据类型。mysql5.5
3.如果分区中有主键或者唯一索引的的列，那么所有主键列和唯一索引列都必须包含进来
4.分区表中无法使用外键约束

一般使用分区表是有很大的一张表，而我们仅仅只是需要使用这些分区中的某一些分区。其他都是算历史数据不怎么使用。
在数据量超大的时候btree索引其实就没啥作用了，要查询符合条件的数据，会产生大量的随机I/O，响应时间会变得很长。
另外维护索引也需要磁盘空间和I/O，代价也非常高。这种情况下可以使用分区表，用分区来减少搜索的范围来提高查询效率。
如果数据有明显的热点数据，把热点数据按照某种粒度来切割放到同一个分区中也可以打打提高查询的效率。

不过尽管如此，分区表还是有很多问题。
### 问题：
1.NULL值会使分区过滤无效
分区的表达式可能值会是NULL，而不是我们期望的整数。比如说你的数据在第三个分区，然后查询条件是大于第三个分区的表达式的值。
explain 可能会显示过滤条件里面查询了分区表的第一个分区和第三个分区之后的分区。原因就是分区表达式的值可以是NULL，而当这个值是null的时候，mysql 会把这些记录都放在第一个分区里面，
所以
查询的时候，mysql会去检索第一个分区。这么做有很大的一个问题就是，如果这样为null的数据很多，都放到了第一个分区，第一个分区会变得非常的的。
每次where查询的时候都去扫描一遍第一个分区，这样的话性能消耗就很大，而且要找的数据还不在第一个分区里面。
在mysql5.5之前可使用partition p_nulls values less than (0)来创建第一个分区。如果插入的数据都是有效，第一个分区就是空的，
如果数据有很多无效的，因为是less than (0)所以查询也很快。如果是在mysql5.5之后，可以使用partition by range columns(key)，就也不会有着问题了。

2.分区列和索引列不匹配
每个分区因为都有自己独立的索引，所以扫描分区列上的索引，就需要扫描每一个分区内对应的索引。此时如果分区内的索引在a上面，分区的列时
b。这样如果分区数量比较多，每个分区内的索引又不一样，就会有额外的开销。

3.选择分区的成本可能很高
分区的类型有很多，每次查询要知道数据是属于哪一个分区，需要扫描所有的分区定义才能知道记录的所属分区，分区数量很多的情况下是成本很高的。
根据实践经验，100个左右的分区没有问题。再不断往上，性能就会有降低。

4.打开并锁柱所有底层表的成本可能很高
插叙访问分区的时候，mysql会锁住所有的底层表，这样是会影响所有查询的，性能开销很大。

5.维护分区的成本可能很高
在进行alter操作的时候，重组分区会涉及到临时的分区，然后需要复制数据，这样的操作也是代价很高的。


### 总结：  
1.一般表非常大，热点数据只在表的某些指定部分，可能还想批量删除大量数据，此时可以使用分区表是非常方便的
2.不过分区表的性能开销很大，有很多限制都会使它的性能变低，不是特殊场景最好不要乱使用。
