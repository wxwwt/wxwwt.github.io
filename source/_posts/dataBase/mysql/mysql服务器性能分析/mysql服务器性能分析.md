---
title: mysql服务器性能分析
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: mysql服务器性能分析
---
## mysql服务器性能剖析

#### 最近看了下高性能mysql的服务器性能剖析章节，看完了记录一下，梳理一下学习的东西。

### 性能优化
#### 1.什么是性能优化
书中提出了简单的解释 在服务正常运转的情况下，减少了服务的响应时间
那么竟然我们要减少服务的响应时间，也就是说我们优化之后的rt是小于之前的rt的，这里的前提是
我们知道之前的rt是多少，然后进行优化，再把新的rt和之前的对比。如果新的rt比较小，那么就进行了一个成功的优化。
这里于是出现了另一个关键点，怎么测量或者说度量服务器的性能，响应时间？
我们一步步来，为了降低响应时间我们现需要测量响应时间，然后我们还得对当前的响应时间进行分析，得出是哪个步骤比较
耗时，为什么会导致这么慢，然后对症下药，减少某个环节的耗时，然后就达到了优化的结果。

#### 2.理解性能优化
在mysql中一般说的性能优化都是指的select，然后响应时间其实还是可以细分为两部分：
执行时间和等待时间。性能分析的时候要理解主要是哪一部分的问题比较大，如果主要是等待时间，比如占比90%，执行时间10%，
那么就优先看看等待时间可能是I/O问题。
性能优化的几个规则：
##### 1.值得优化的查询：
1.如果一个响应时间比重没有超过5%，那么就是不值得优化的，无论如何努力收益也不会超过5%
2.如果优化的成本大于收益那么也是不值得优化的，花了大量的时间去提高一个已经很快的查询，那么这个时间花的和不值得，
可以说是逆优化，浪费了时间去做了一些无用功
##### 2.异常情况：
某些低频的查询可能不多，但是耗时很长，直接影响了用户的体验，这种查询也是需要揪出来，优化的。
##### 3. 未知的未知
测量工具可能测量出来的时间和实际应用程序执行的时间并不完全一致，可能会存在一个差值。比如说测量工具是测出来9.6秒，实际程序打印的时间可能是
11秒，相差的1400毫秒没有测量到，程序的有些子任务没有测量到，这里可能就会导致排查问题不精准，从而影响优化的进行。
##### 4.被隐藏的细节
这里举个例子更好理解一点就是，有一万的查询，有1-2个查询非常慢，并且调用频次不多，其他的非常快，平均下来其实，这个1-2个查询的响应时间被
平均了就看不出啥异常。这种查询依然是需要去优化的。可能会在某个时间因为调用的多了，就直接让服务器假死或者直接挂掉。

### 3.对应用程序进行剖析
这里书中是以php为例，讲述了一些特定的性能分析工具。每种语言都有不同的sql性能分析工具，这里就不赘述了。

### 4.剖析mysql查询
1.服务器负载查询
  1.可以通过mysql的慢查询日志来发现服务器的问题。
    分析查询日志建议使用pt-query-digest
  2.通过抓取tcp网络包，然后解析mysql客户端和服务端通信协议来进行精确的解析分析。
2.单条性能查询
  1.使用show profile
  [官方文档地址](https://dev.mysql.com/doc/refman/5.6/en/show-profile.html)
  2.使用show status
  [官方文档地址](https://dev.mysql.com/doc/refman/5.6/en/server-status-variables.html)

  show status能查看会话中的各种信息，也能看到全局的变量。
  如果是要查询全局的变量使用 show globals status。
  基本用法如下：
  show status like '变量名';

  下面这是一些常用的方法：
  ```
  --查看MySQL本次启动后的运行时间(单位：秒)

show status like 'uptime';

--查看select语句的执行数

show [global] status like 'com_select';

--查看insert语句的执行数

show [global] status like 'com_insert';

--查看update语句的执行数

show [global] status like 'com_update';

--查看delete语句的执行数

show [global] status like 'com_delete';

--查看试图连接到MySQL(不管是否连接成功)的连接数

show status like 'connections';

--查看线程缓存内的线程的数量。

show status like 'threads_cached';

--查看当前打开的连接的数量。

show status like 'threads_connected';

--查看当前打开的连接的数量。

show status like 'threads_connected';

--查看创建用来处理连接的线程数。如果Threads_created较大，你可能要增加thread_cache_size值。

show status like 'threads_created';

--查看激活的(非睡眠状态)线程数。

show status like 'threads_running';

--查看立即获得的表的锁的次数。

show status like 'table_locks_immediate';

--查看不能立即获得的表的锁的次数。如果该值较高，并且有性能问题，你应首先优化查询，然后拆分表或使用复制。

show status like 'table_locks_waited';

--查看创建时间超过slow_launch_time秒的线程数。

show status like 'slow_launch_threads';

--查看查询时间超过long_query_time秒的查询的个数。

show status like 'slow_queries';
  ```
  3.使用慢查询日志
  [官方文档地址](https://dev.mysql.com/doc/refman/5.6/en/slow-query-log.html)
  默认慢查询日志是关闭的，slow_query_log变量就是慢查询日志，可以通过设置变量来开启。
  ```
  查询慢查询日志：
  mysql> show variables  like '%slow_query_log%';
+---------------------+-----------------------------------------------+
| Variable_name       | Value                                         |
+---------------------+-----------------------------------------------+
| slow_query_log      | OFF                                           |
| slow_query_log_file | /xxx/mysql/DB-Server-slow.log |
+---------------------+-----------------------------------------------+
  ```
  ```
  设置慢查询日志
  mysql> set global slow_query_log=1;
  ```
  4.使用PERFORMANCE_SCHEMA
  [官方文档地址](https://dev.mysql.com/doc/refman/5.6/en/performance-schema-system-variables.html)

    这里以mysql5.6为例，PERFORMANCE_SCHEMA是一张记录着mysql服务器性能的表。默认是关闭的，需要开启设置才有用。
    如下开启：
    ```
    [mysqld]
    performance_schema=ON
    ```

    查看下是否开启：
    ```
mysql>show variables like 'performance_schema';
+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| performance_schema | ON    |
+--------------------+-------+
    ```

5.间歇性问题排查

6.其他诊断工具
  1.USER_STATISTICS
    Percona Server和MariaDB是支持的，官方的mysql我试了下好像没有找到这个。据说官方分支的mysql是使用的performance
    schema表。
     这个工具可以查看到什么表，什么索引使用最频繁或者最频繁等信息。
     命令：
    ```
    mysql> show tables from information_schema like "%_STATISTICS";
    ```
  2.stace
  stace可以查看和打印出系统调用的情况，是linux的命令，不是mysql自带的。
  ```
  使用方式： strace -p pid or strace command
  ```
  常用语句：
  ```
  1.通用示例:

#strace -o output.txt -T -tt -e trace=all -p 28979

跟踪28979进程的所有系统调用（-e trace=all），并统计系统调用的花费时间，以及开始时间（并以可视化的时分秒格式显示），最后将记录结果存在output.txt文件里面



2.统计httpd进程的耗时时间:

#strace -c -p $(pgrep -n httpd)

按crtl+c,将显示从开始到结束的时间调用



3.跟踪最占cpu的httpd的一个进程,并将信息输出到文件

#strace -t -f  -o httpd-strace  -p $(top -b -n1 | grep "httpd" | head -1 | awk '{print $1}')
  ```

  ### 总结：
  1.性能测量最直接有效的方式是根据响应的时间
  2.最好从应用程序来入手测量性能
  3.响应时间分为：执行时间和等待时间
  4.要先了解是执行时间还是等待时间过长影响了性能
  5.少于5%的响应时间就不用考虑优化了，要把力气花在刀刃上，优化成本高于收益时，不值得去优化。
  6.任何测量工具可能都会有些任务或者地方测量不到，隐藏的细节可能才是关键，需要多种方式结合分析会更加准确
