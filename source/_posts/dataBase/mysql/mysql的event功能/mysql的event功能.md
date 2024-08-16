---
title: mysql的event功能
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: mysql的event功能
---
# mysql的event定时任务


最近用到了mysql的event，就学习了一波，记录一下。
mysql从5.7之后就增加了event的功能，类似于linux的crontab，就是定时任务，用来一次性或者周期性执行某些任务。

### event的主要特征和属性：
1.在mysql中event的是根据它的名称和schema来唯一标识一个事件的。  

2.event中可以是由一句sql语句组成，也可以是由BEGIN-END的代码块组成。event可以执行一次或周期性执行，周期性执行可以设置有规律的开始时间和结束时间，也可以分别单独设置开始时间，结束时间，或者都不设置也可以。
默认情况下，只要定时任务设置成功，它就会一直执行下去，直到event是不可用的或者被删除掉。  
**这里要注意的点：如果在一个周期内event没有执行完毕，另一个event的实例也会开始执行。例如A事件设置的一分钟执行一次，还没有执行完已经到第二分钟，此时会有另外一个新的event实例也开始执行。如果这些实例之间有竞争资源的关系，可能会导致mysql服务变得很慢。**    

3.用户可以创建，修改，删除事件。语法上有错误的创建或者修改语句会直接抛出错误信息。用户可以在事件的动作里面写入用户本身没有权限的操作语句，
创建和修改可以成功，但是执行的时候回抛出异常。例如用户对A表只有select的权限，他在event中对A表进行了update，事件可以创建或者修改成功，但是执行的时候是会报错的（这个略坑，本来写完觉得没问题，结果是只有在
执行的时候在才只知道有没有问题）。  

4.sql语句可以修改事件的名称，时间，执行周期，状态（启用还是禁用）和schema等等。  

5.事件的创建者默认是创造这个事件的用户，但是如果有其他用户也对这个事件进行了修改，那么事件的创建者就会变更为最后一个对这个事件进行修改的用户。只要有改数据库的event权限的用户，都可以修改event的内容。  

### event的设置：
event的参数值叫event_scheduler，它有三个值：NO，OFF，DISABLED。对应的语义是开启，关闭和不可用。
**注意：在命令行设置的参数值，在mysql重启后都会失效，只有在配置文件中修改的值，在重启后不会失效。**

开启event：
在mysql的命令行中输入：
```
SET GLOBAL event_scheduler = ON;
SET @@GLOBAL.event_scheduler = ON;
SET GLOBAL event_scheduler = 1;
SET @@GLOBAL.event_scheduler = 1;
```


关闭event
```
SET GLOBAL event_scheduler = OFF;
SET @@GLOBAL.event_scheduler = OFF;
SET GLOBAL event_scheduler = 0;
SET @@GLOBAL.event_scheduler = 0;
```

尽管ON和1，OFF和0是等价的，但是尽量使用ON和oFF，因为还有一个值DISABLED是没有数字的值来对应的，所以尽量用英文。
注意：
只有在服务器启动时才可以将事件调度程序设置为DISABLED。
如果event_scheduler为ON或OFF，则无法在运行时将其设置为DISABLED。
而且如果在启动时将事件计划程序设置为DISABLED，也无法在运行时更改event_scheduler的值。


将event设置为不可用的方式有两种
命令行：
```
--event-scheduler=DISABLED
```

配置文件
```
event_scheduler=DISABLED
```


### event的语法：
#### 1.event的创建：
```
CREATE
    [DEFINER = user]
    EVENT
    [IF NOT EXISTS]
    event_name
    ON SCHEDULE schedule
    [ON COMPLETION [NOT] PRESERVE]
    [ENABLE | DISABLE | DISABLE ON SLAVE]
    [COMMENT 'string']
    DO event_body;

schedule: {
    AT timestamp [+ INTERVAL interval] ...
  | EVERY interval
    [STARTS timestamp [+ INTERVAL interval] ...]
    [ENDS timestamp [+ INTERVAL interval] ...]
}

interval:
    quantity {YEAR | QUARTER | MONTH | DAY | HOUR | MINUTE |
              WEEK | SECOND | YEAR_MONTH | DAY_HOUR | DAY_MINUTE |
              DAY_SECOND | HOUR_MINUTE | HOUR_SECOND | MINUTE_SECOND}
```

这个是官方的创建规范，看上去有点让人头大，我们举个例子来看下帮助理解下。
我们来创建一个每分钟执行某一个select语句（其实查询没啥意义，仅仅作为演示）
```
CREATE EVENT test_insert   （创建名为test_insert的event）
  ON SCHEDULE EVERY 1 MINUTE   （每一分钟跑一次）
  DO                            (执行查询语句)
    SELECT NOW();
```
上面这个是单个语句的event，我们在看下如果是复合的语句，使用BEGIN-END来完成。

```
delimiter //

CREATE EVENT test_insert
    ON SCHEDULE
      EVERY 1 MINUTE
    COMMENT '事件添加的注释'
    DO
      BEGIN
        INSERT INTO XXX;
        UPDATE XXX;
      END //

delimiter ;
```
如果读者写过存储过程的话，会容易理解这个多行的语句，和写储存过程的语法差的不多，只是外面加了创建event的过程。
event也可以周期性的去调用存储过程，将要执行的内容放到储存过程里面，然后用event去调用会比直接将所有逻辑全部写入到event里面要好。
因为event执行不会有报错信息，也不会有报错的日志，很难去调试。如果是放到储存过程中，可以用手动调用一下储存过程，如果有问题可以直接根据报错信息来排查语句中存在的问题。


#### 2.修改event：
基本上和创建是一样的,直接贴上官方的规范写法。
```
ALTER
    [DEFINER = user]
    EVENT event_name
    [ON SCHEDULE schedule]
    [ON COMPLETION [NOT] PRESERVE]
    [RENAME TO new_event_name]
    [ENABLE | DISABLE | DISABLE ON SLAVE]
    [COMMENT 'string']
    [DO event_body]
```

#### 3.删除event：
删除event是根据event的名称来删除的。
```
DROP EVENT [IF EXISTS] event_name
```


#### 4.查看event的元数据：
```
查看事件的三种方式：
SELECT * FROM INFORMATION_SCHEMA.EVENTS;
SHOW EVENTS;
SELECT * FROM mysql.event;


查看创建event的创建语句：
SHOW CREATE EVENT
```

#### 5.赋予event权限
event的权限就叫EVENT，下面是通用的赋权语句，将用户user在host上赋予database的table表event的权限。
```
GRANT EVENT ON database.table TO user@host;
```


### 总结：
1.event相当于是mysql自带的定时任务，可以用来处理一些周期性的工作，比如每个月定时计算某些表的统计数据。比较方便。  
2.event的功能还不够强大，debug其实比较麻烦，也不打印报错信息，建议在event中去调用储存过程，这样可以直接CALL来调用event中要执行的储存过程，来判断语句中有没有问题。另外一种方式就值执行event的时候往一张日志表去写入执行event的数据信息，这样event有没有执行，是否报错都一目了然。  
3.event的间隔要设置合理，因为event是可以并发执行的，如果第一个event没有执行完，还拿着行锁或者表锁，第二个event在来执行就会阻塞，还没执行完，后面的event又来了。这样就可能会把数据库的性能降得很低。
