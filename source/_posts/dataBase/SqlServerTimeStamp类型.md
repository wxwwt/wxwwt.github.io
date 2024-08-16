---
title: SqlServerTimeStamp类型
date: 2024-01-06 22:08:22
updated: 2024-08-15 22:35:58
tags: dataBase
---
SqlServer timestamp类型

# 前言：

​	最近在对接客户数据的时候发现有SqlServer数据库里面有timestamp类型，一开始以为跟mysql，postgresql的timestamp类型差不多，是时间类型。结果接入之后发现竟然是二进制的，就感觉到很困惑，为什么时间类型要用二进制来存呢？难道是有什么优势可言吗？查了资料之后发现，原来SqlServer比较早期的版本里面timestamp是一个递增的数字，用来记录行增加和更新的次数，而不是用来记录时间的字段，所以这个类型的字段是没有业务含义的字段，数据接入的时候忽略这个字段或者直接置为null就可以了。另外提一嘴，我原本以为SqlServer只会出现在大学教学数据库的时候存在，结果发现它还是有很多的用户在使用的，微软的生态还是挺强大的。



# 介绍：

## 概念

timestamp在sqlServer早期版本比如SqlServer2008,还是叫做timestamp，后面新一些的版本就叫rowversion了，这个也更贴切它的作用，所以timestamp和rowversion是同一个东西，SqlServer的timestamp和ISO标准的timestamp数据类型是不一样的，这个要注意。官网的描述是，每个数据库都有一个计数器，当对数据库中包含 rowversion 列的表执行插入或更新操作时，该计数器值就会增加。 此计数器是数据库行版本。 这可以跟踪数据库内的相对时间，而不是时钟相关联的实际时间。 一个表只能有一个 rowversion 列。 每次修改或插入包含 rowversion 列的行时，就会在 rowversion 列中插入经过增量的数据库 rowversion 值。 这一属性使 rowversion 列不适合作为键使用，尤其是不能作为主键使用，因为它是一直变化的。

## 具体使用：

官网对timestamp的使用并没有很多着笔，可能是因为一般用户使用不到这个字段类型的。

在 CREATE TABLE 或 ALTER TABLE 语句中，不必为 timestamp 数据类型指定列名，例如：

```sql
CREATE TABLE ExampleTable (PriKey int PRIMARY KEY, timestamp);  
```

如果不指定列名，则 SQL Server 数据库引擎将生成 timestamp 列名；但 rowversion 同义词不具有这样的行为。 在使用 rowversion 时，必须指定列名，例如：

```sql
CREATE TABLE ExampleTable2 (PriKey int PRIMARY KEY, VerCol rowversion) ;  
```

稍微要一点的是，不可为空的 rowversion 列在语义上等同于 binary(8) 列， 可为空的 rowversion 列在语义上等同于varbinary(8) 列。

可以使用某行的 rowversion 列轻松确定自上次读取该行后，是否对该行运行过更新语句。 如果对该行运行过更新语句，则会更新 rowversion 值。 如果没有对该行运行过更新语句，则 rowversion 值将与以前读取该行时的 rowversion 值相同。 若要返回数据库的当前行版本值，请使用 [@@DBTS](https://learn.microsoft.com/zh-cn/sql/t-sql/functions/dbts-transact-sql?view=sql-server-ver16)。

因为每次数据更新timestamp都会变化，所以可以利用这个特性来作为乐观锁。

这个是官网给的例子，`<myRv>` 来表示上次读取该行时的 rowversion 值：

```
DECLARE @t TABLE (myKey int);  
UPDATE MyTest  
SET myValue = 2  
    OUTPUT inserted.myKey INTO @t(myKey)   
WHERE myKey = 1   
    AND RV = <myRv>;  
IF (SELECT COUNT(*) FROM @t) = 0  
    BEGIN  
        RAISERROR ('error changing row with myKey = %d'  
            ,16 -- Severity.  
            ,1 -- State   
            ,1) -- myKey that was changed   
    END;  
```

我们在写代码的时候也是一样，先读取到timestamp的值t1，然后更新的时候在where条件里面比较一下当前的timestamp值和上次t1是不是一样的。如果不一样，说明这行数据已经被其他线程修改过了，要重头再来一次更新的操作。如果一样的话，就证明这个数据只有我们修过过了，这就是数据库乐观锁的使用。

## 注意事项：

1.timestamp/rowverion是同一个东西，是记录数据行插入/更新的计数器，和ISO标准的timestamp不一样

2.如果要使用日期或者时间，最好使用datetime2

下面是SqlServer2016-2022的时间类型的表格，可以根据情况使用。



| 数据类型                                                     | 格式                                      | 范围                                                         | 精确度     | 存储大小（字节） | 用户定义的秒的小数部分精度 | 时区偏移量 |
| :----------------------------------------------------------- | :---------------------------------------- | :----------------------------------------------------------- | :--------- | :--------------- | :------------------------- | :--------- |
| [time](https://learn.microsoft.com/zh-cn/sql/t-sql/data-types/time-transact-sql?view=sql-server-ver16) | hh:mm:ss[.nnnnnnn]                        | 00:00:00.0000000 到 23:59:59.9999999                         | 100 纳秒   | 3 到 5           | 是                         | 否         |
| [date](https://learn.microsoft.com/zh-cn/sql/t-sql/data-types/date-transact-sql?view=sql-server-ver16) | YYYY-MM-DD                                | 0001-01-01 到 31.12.99                                       | 1 天       | 3                | 否                         | 否         |
| [smalldatetime](https://learn.microsoft.com/zh-cn/sql/t-sql/data-types/smalldatetime-transact-sql?view=sql-server-ver16) | YYYY-MM-DD hh:mm:ss                       | 1900-01-01 到 2079-06-06                                     | 1 分钟     | 4                | 否                         | 否         |
| [datetime](https://learn.microsoft.com/zh-cn/sql/t-sql/data-types/datetime-transact-sql?view=sql-server-ver16) | YYYY-MM-DD hh:mm:ss[.nnn]                 | 1753-01-01 到 9999-12-31                                     | 0.00333 秒 | 8                | 否                         | 否         |
| [datetime2](https://learn.microsoft.com/zh-cn/sql/t-sql/data-types/datetime2-transact-sql?view=sql-server-ver16) | YYYY-MM-DD hh:mm:ss[.nnnnnnn]             | 0001-01-01 00:00:00.0000000 到 9999-12-31 23:59:59.9999999   | 100 纳秒   | 6 到 8           | 是                         | 否         |
| [datetimeoffset](https://learn.microsoft.com/zh-cn/sql/t-sql/data-types/datetimeoffset-transact-sql?view=sql-server-ver16) | YYYY-MM-DD hh:mm:ss[.nnnnnnn] [+\|-]hh:mm | 0001-01-01 00:00:00.0000000 到 9999-12-31 23:59:59.9999999（以 UTC 时间表示） | 100 纳秒   | 8 到 10          | 是                         | 是         |

3.一张表只能有一个timestamp列，在往这张表上增加timestamp列就会报错

实际测试也是如此，在测试的表上在加一列timestamp就会报错：

![{5A5E1119-68B3-BAAB-184F-FC2A70500E51}](C:\Users\Administrator\AppData\Roaming\Tencent\QQ\Temp\{5A5E1119-68B3-BAAB-184F-FC2A70500E51}.jpg)







# 参考资料：

[1.sql时间类型介绍](https://learn.microsoft.com/zh-cn/sql/t-sql/functions/date-and-time-data-types-and-functions-transact-sql?view=sql-server-ver16)

[2.rowversion类型](https://learn.microsoft.com/zh-cn/sql/t-sql/data-types/rowversion-transact-sql?view=sql-server-ver16)

[3.timetamp介绍](https://www.cnblogs.com/OpenCoder/articles/10411186.html)