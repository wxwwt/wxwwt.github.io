---
title: mysql insert select 死锁问题
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: mysql insert select死锁问题
---
### 引语：
最近中遇到一个数据库死锁的问题，这里记录一下解决的过程。


### 问题产生：
系统中mysql里面有几个event，每几分钟就会执行一次，用来统计数据之类的功能，然后这个event里面会往一张表里面写入数据。
大致内容：
replace into a from select 需要的字段 from b;
大体结构是这样，select 需要的字段from b这里是简写，实际上非常复杂，有很多表的join的操作。然后这个event是每一分钟就执行一次，在数据量很大的情况下
一分钟可能还执行不完。然后我们会有其他的各种插入，更新的操作去对b表进行操作。此时就会发现，后端日志里面经常会有deadlock和wait lock timeout的报错，
最后测试发现把event关掉就没有这个问题，基本确认是这个event的问题。

### 问题分析：
其实最耗时的是发现是event的问题，查询资料解决问题并没有花太多时间。
1.首先根据后端日志里面的报错信息定位到是哪张表产生了死锁，是哪张表等待锁超时  
2.然后根据这几个表名和打印的sql找到了大概可能是哪里的问题，大致确认了是event中的sql导致的  
3.再验证我们的想法，把event关掉后发现日志就没有lock的问题了  
4.检查event中的语句发现大概就是replace into a from select 需要的字段 from b;  

这里主要是不太清楚mysql哪些情况会上锁，理论上select的操作只会上一个共享锁，对于b表的插入和更新等操作是上排他锁，
这两个是可以兼容的，一个读一个写，并不冲突。但是根据等到所超时的现象上来看，就像是select 需要的字段 from b把b表也给锁住了，
所以插入和更新都在等待锁。

最后在Stack Overflow中找到了有一点眉目的信息，[链接地址](https://stackoverflow.com/questions/37919682/find-mysql-innodb-deadlock-reason-on-insert-into-select)。
这里说要设置成read-committed的级别就可以了。然后也引出了一个mysql配置参数：innodb_locks_unsafe_for_binlog。

于是我们顺着这个信息从官网上去查看，发现有这么一段话：
```
INSERT INTO T SELECT ... FROM S WHERE ... sets an exclusive index record lock (without a gap lock) on each row inserted into T. If the transaction isolation level is READ COMMITTED, or innodb_locks_unsafe_for_binlog is enabled and the transaction isolation level is not SERIALIZABLE, InnoDB does the search on S as a consistent read (no locks). Otherwise, InnoDB sets shared next-key locks on rows from S. InnoDB has to set locks in the latter case: During roll-forward recovery using a statement-based binary log, every SQL statement must be executed in exactly the same way it was done originally.
```
意思是说对于INSERT INTO T SELECT ... FROM S WHERE ...这种情况首先T表上会家伙是哪个记录锁（行级锁），并且是不带间隙锁的。
对于表S，有两种情况下不会加锁：  
1.如果事务隔离级别为READ COMMITTED  
2.或者启用了innodb_locks_unsafe_for_binlog且事务隔离级别不是SERIALIZABLE的  

否则，InnoDB在S的行上设置共享的next-key。如果不清楚next-key的话可以看下官网的这个介绍，[链接地址](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-next-key-locks)。

因此我们要解决所等待超时的方式已经比较明朗了，就是让S表不要被锁住，然而不要被锁住可以使用官网说的两种方式。
这两种都可以，但是根据innodb_locks_unsafe_for_binlog这个参数的介绍来看最好是使用方式1，将事务的隔离级别设置为read-committed。  

原因有下面几点：  
1.是innodb_locks_unsafe_for_binlog这个参数是静态的，必须要在my.cnf中加入一行innodb_locks_unsafe_for_binlog = 1，然后重启数据库才能生效。
在mysql中输入命令：
```
show variables like "%innodb_locks_unsafe_for_binlog%"
```
如果发现是ON就是开启成功了。  
2.事务的隔离级别粒度比较细，可以针对某个session来设置，不同的session可以用不同的隔离级别，而且这个参数是动态的直接在mysql命令行修改就行。  
3.mysql5.7的参数介绍中说，innodb_locks_unsafe_for_binlog这个参数将在后面的mysql版本中废弃掉。这个说的是实话，我去查看了mysql8.0的参数详解发现已经没有这个参数了。  

所以推荐使用事务隔离级别来控制。


### 参考资料：  
1.[Stack Overflow上对于insert select锁的解答](https://stackoverflow.com/questions/37919682/find-mysql-innodb-deadlock-reason-on-insert-into-select)  
2.[innodb锁设置的情况](https://dev.mysql.com/doc/refman/5.7/en/innodb-locks-set.html)  
3.[innodb所种类介绍](https://dev.mysql.com/doc/refman/5.7/en/innodb-locking.html#innodb-next-key-locks)   
4.[innodb_locks_unsafe_for_binlog参数的介绍](https://dev.mysql.com/doc/refman/5.7/en/innodb-parameters.html#sysvar_innodb_locks_unsafe_for_binlog)  
