---
title: mysql分区表的四种类型
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: mysql
---
# mysql分区表四种类型

## 分区表的类型
mysql分区表总共有四种类型range，list，key，hash，这四种方式使用的最多的是range的方式。partition分区子句中可以使用各种函数，
但是partition表达式里面返回的值必须是 一个确定的整数，且不是一个常数。可以对字段使用year(),to_days()函数将字段转化为整数。


## range类型
基于属于一个给定连续区间的列值，把多行分配给分区。一般会使用时间的字段来作为分区的列，记录每天的数据。

```
CREATE TABLE range_table (
    id      INT NOT NULL,
    user_name       VARCHAR(50)     NOT NULL,
    date_create   DATETIME            NOT NULL,
    PRIMARY KEY (`id`,`user_name`,`date_create`)
) partition by range columns(date_create)
(partition p01 values less than ('1985-12-31'),
partition p02 values less than ('1990-12-31'),
partition p03 values less than ('1995-12-31'),
partition p04 values less than ('2000-12-31'),
partition p05 values less than (MAXVALUE)
);
```

## list类型
list分区和range分区类似，区别在于list是离散型数值的集合，range是连续的区间值的集合。建议list分区列是非null列，否则插入null值如果枚举列表里面不存在null值会插入失败。其他分区对于null值，range分区会将其作为最小分区值存储，放在第一个分区里面，而HASH\KEY分为会将其转换成0存储，list分区因为只支持整形，非整形字段需要通过函数转换成整形才能插入。

```
CREATE TABLE list_table (
   id      INT NOT NULL,
   user_id       INT(11)     NOT NULL,
   date_create   DATETIME            NOT NULL,
   PRIMARY KEY (`id`,`user_id`,`date_create`)
) partition by list(user_id)
(partition p01 values in (1,3,5,7,9),
partition p02 values in (0,2,4,6,8)
);
```



## key类型

KEY分区其实跟HASH分区差不多，不同点如下：

1.KEY分区允许多列，而HASH分区只允许一列。
2.如果在有主键或者唯一键的情况下，key中分区列可不指定，默认为主键或者唯一键，如果没有，则必须显性指定列。
3.KEY分区对象必须为列，而不能是基于列的表达式。
4.KEY分区和HASH分区的算法不一样，PARTITION BY HASH (expr)，MOD取值的对象是expr返回的值，而PARTITION BY KEY (column_list)，基于的是列的MD5值。

有主键的情况下：
```
CREATE TABLE primary_key_table (
   id      INT NOT NULL,
   user_id       INT(11)     NOT NULL,
   date_create   DATETIME            NOT NULL,
   PRIMARY KEY (`id`,`user_id`,`date_create`)
) partition by key()
PARTITIONS 2;
```


没有主键的情况下：
```
CREATE TABLE no_key_table (
   id      INT NOT NULL,
   user_id       INT(11)     NOT NULL,
   date_create   DATETIME            NOT NULL
) partition by key(id)
PARTITIONS 2;

```
## hash类型
  基于用户定义的表达式的返回值来进行选择的分区，该表达式使用将要插入到表中的这些行的列值进行计算。这个函数可以包含MySQL 中有效的、产生非负整数值的任何表达式。
1.普通hash类型
```
CREATE TABLE hash_table (
   id      INT NOT NULL,
   user_id       INT(11)     NOT NULL,
   date_create   DATETIME            NOT NULL
) partition by hash(user_id)
PARTITIONS 2;

```
2.linear hash分区
linear hash分区是HASH分区的一种特殊类型，与HASH分区是基于MOD函数不同的是，它基于的是另外一种算法。
优点是在数据量大的场景，譬如TB级，增加、删除、合并和拆分分区会更快，缺点是，相对于HASH分区，它数据分布不均匀的概率更大。

```
CREATE TABLE linear_hash_table (
   id      INT NOT NULL,
   user_id       INT(11)     NOT NULL,
   date_create   DATETIME            NOT NULL
) partition by LINEAR hash(user_id)
PARTITIONS 2;
```
