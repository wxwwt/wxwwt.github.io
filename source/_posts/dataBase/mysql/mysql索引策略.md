---
title: mysql索引策略
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: mysql
---

### 引语:
  最近看了《高性能mysql》，虽然还没看完，但是觉得确实写得挺好的。索引部分看完还是对自己创建索引和了解
  mysql的索引运行原理有了很大的帮助。做了些关于索引的笔记，遇到问题的时候可以回溯下参考下。

### 1.索引的优点：
要是对mysql索引的基本概念还不太清楚的话，可以看下我之前的两篇博客。
[mysql聚簇索引和非聚簇索引](https://blog.csdn.net/sc9018181134/article/details/104885076)，[大白话btree和b+tree](https://blog.csdn.net/sc9018181134/article/details/104743176)。

1.1 索引大大减少了服务器需要扫描行的数量
1.2 索引可以帮助服务器避免排序和临时表
1.3 索引可以随机I/O变为顺序I/O

### 2.使用索引的策略

2.1 独立的列
查询时索引列要是独立的列,指的是索引不能是表达式的一部分,也不能是函数的参数
错误例子:
1. select user_id from user where user_id + 1 = 7;
    这里完全可以写成user_id = 6,这样索引才会生效
2.select * ...where to_days(current_date)-to_days(date_col) <=10
注：__如果把索引的列，像上面的例子中where后面的字段是加了索引的，但是因为对索引的列进行了操作（变成了user_id + 1这样的表达式），
或者是使用函数对索引列进行了操作。这样都会导致索引失效。__

2.2前缀索引和索引选择性
2.2.1 当有一些需要被查询的列比较长的时候,我们可以创建前缀索引,就是这个列的前面某个长度的索引,比如alibabayushishidadao,我们建立前八位的索引,就是alibaba,用这个前缀来搜索对应的列.但是这里有一个问题就是,怎么来确定这个前缀索引的长度呢?
这个提一个概念叫做,索引选择性.索引选择性是指不重复的索引和数据记录总数T的比值.范围是1/T~1之间.索引的选择性越高,则查询效率越高,因为这代表着索引覆盖的不重复数据越多,能在查询的时候过滤掉更多的行.索引的选择性是1,那么这个索引的性能极高.
因此我们需要设置一个合理的前缀索引长度让索引选择率更高.
例子：
select count(distinct left(phone, 3))/count(*) as prefix3,
count(distinct left(phone, 5))/count(*) as prefix5,
count(distinct left(phone, 7))/count(*) as prefix7
from table;
咱们假设是对phone这个字段进行前缀索引化，通过上面的count可以计算出哪个比率是最接近直接使用完整的phone的索引选择率。
这样咱们就可以对索引改为前缀索引，从而减少索引的长度，提升查询效率。

2.2.2 创建前缀索引:alter table user add key (user_name(7)) 数字就是前缀索引的长度

2.2.3 前缀索引的缺点:
1.前缀索引因为不是列的全部长度,所以无法进行group by和 order by
2.同时也是因为不是列的全部长度,所以无法达到覆盖索引

2.2.4 注：**前缀索引其他的的使用场景是针对较长的数据使用唯一id,或者有时候需要使用到后缀索引(当然mysql是不支持的,但是我们在存储数据的时候将数据翻转过来储存)**

2.3 多列索引
如果发现explain中有出现type=index_merge,那么就得考虑索引创建的合理性的问题了。
这种索引合并通常会消耗大量的cpu和内存资源,更重要的是优化器不会把这些计算到查询的成本中,优化器只关心随机页面读取的数据量有多少。


2.4 选择合适的索引顺序
索引的顺序没有什么固定的法则，要根据实际使用情况来创建。
不过一般情况，咱们将使用频率高，索引选择率高的字段放在前面，涉及到范围查询和使用频次低的放在后面。

3.5聚簇索引
3.5.1 主键最好是自增的id,这样每次有新数据进来的时候对于聚簇索引的数据只需要在最后的一列新增数据即可,即使当前的数据页满了的话,也只需要从新的一页开始新增数据.如果是uuid等没有顺序的主键的,话因为插入的数据在innodb中是按顺序来的,假设插入的uui的主键小于之前的主键,那么之前的的数据都会被移动,让新数据插入,遇到数据页满了的情况更是要消耗更多的资源去处理这样的情况.会不停的有页分裂的情况产生,不停的页分裂就会导致有碎片产生,那么就会比普通的自增主键占用更多的空间.
3.5.2 自增主键的缺点:在有并发的情况下,可能会导致资源的竞争,因为自增id的上界是每个线程都会去竞争的,所有的插入都是需要获取到最新的最大的自增id,而并发会让这个上界不停的在变化中.
3.5.3 mysql不能再索引中执行like操作,如果是最左前缀的like比较是可以走索引的,因为会被转换为简单的比较操作.但是如果是通配符开头的like "%xxx%"这样的范围查询是不走索引的.因为搜索引擎无法拿通配符和具体的索引去作比较.

tips:select sum(description = 3),sum(category_type = 2)  from shop_page_field; 这样可以统计改字段属于某个值的数据有多少条,相较于count这个好像写起来更方便,但是性能比较如何,那就不得而知啦.

4.5.覆盖索引
4.5.1 覆盖索引的好处:
1.覆盖索引数据的条目比总的数据量要小,查询的速度会更快
2.对于innod来说直接从索引上获取了数据就不需要在走聚簇索引,不需要进行二次查询

4.6.未使用的索引
在服务器中打开userstates（默认是关闭的），然后让服务器运行一段时间。再通过查询INFORMATION_SCHEMAINDEX.STATISTCS就能查询到某个索引的使用率。如果某个索引没有被使用的话可以删除掉。

4.7 索引和锁
innodb的锁的粒度是可以到行级，总共有行级锁和表级锁。这里行锁是必须加了索引才能达到，因为索引上保存了主键和索引和信息，
才能精准的锁到那一行数据。如果没有加上索引，进行update等操作的时候是会锁表的，这个要注意。

4.8 索引和排序
如果某个字段经常会用来排序，最好加上索引，通过explain的关键字可以看到extra字段里面会返回filesort（mysql叫做文件排序，虽然不一定会用到磁盘文件），
加上索引之后就不会显示这个filesort。
假设咱们有索引（A，B，C）
然后有查询语句，对象下面排序是否生效的情况
(1)select * from table where A = 'a' order by B, C;(索引生效)
(2)select * from table where A = 'a' order by B;(索引生效)
(3)select * from table where A = 'a' order by A, B;(索引生效)
(4)select * from table where A = 'a' order by C; （索引对A生效，对C排序没有生效）
(5)select * from table where A = 'a' order by B, D (不生效，引用了一个不再索引列的字段)
(6)select * from table where A > 'a' order by B, C(不生效，对于A是范围查询，索引失效)
(7)select * from table where A = 'a' and B in ('b1', 'b2') order by C (失效对于B in的情况也是范围查询，索引失效)

4.9 其他优化策略

1.当查询的内容是类似url的时候,使用btree效率会不那么高,因为url一般都比较长，索引搜索次数和效率并不好。
这个时候我们可以对url进行crc32或者crc64,计算出它的hash值存储起来.
但是crc32/crc64会产生碰撞,所以查询条件要带上原有的url;
select * from url_table where url_hash = "1342134234" and  url = "http://www.baidu.com".
首先会根据url_hash去找到对应的url,可能有碰撞但是查询很快,然后再根据url的值去筛选,
这样查询起来性能就会很高.
这里url_hash还是使用的btree索引,只是用来过滤url会比对直接的长url会快很多.可以成为伪hash索引

注:crc64,fnv64()都需要mysql额外安装插件哦，不是mysql官方自带的。所以如果没有安装的话，咱们可以在程序中写入数据的时候进行MD5
等类似操作保存一个hash值。
