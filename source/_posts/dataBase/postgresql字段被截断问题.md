---
title: postgresql字段被截断问题
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: dataBase
---


# 前言

最近遇到一个问题就是字段名过长，会被pg给截断，导致原始字段和下游用的的字段不一样，就会报错。当然，小伙伴可能会说为什么会用那么长的字段名，每个应用程序里面处理不一样，我们数据字段每次被使用就会加一个后缀，用来标识它是在哪一层里面被使用的，随着被使用的越来越多，就会超长。报错类似于下面这样（字段名是为了测试这个名字故意这样取的，实际不会这样用）:

```
test=# ALTER TABLE "test1" 
 RENAME TO "123456789_123456789_123456789_123456789_123456789_123456789_123456789_";
 ERROR: relation "test1" does not exist

 NOTICE: identifier "123456789_123456789_123456789_123456789_123456789_123456789_123456789_" will be truncated to "123456789_123456789_123456789_123456789_123456789_123456789_123"

```

这只是这个NOTICE，在命令行里面可以报出来，但是如果是在Java程序里面，执行的时候是不会报这个错，只会在最后执行失败的时候抛出字段不存在的异常。



# 解决方式

1.查找资料后发现，pg的默认长度是NAMEDATALEN - 1 ，就是NAMEDATALEN 默认是64，所以字段最大长度是63。

下面这个来自于官方文档：

```
The system uses no more than NAMEDATALEN-1 bytes of an identifier; longer names can be written in commands, but they will be truncated. By default, NAMEDATALEN is 64 so the maximum identifier length is 63 bytes. If this limit is problematic, it can be raised by changing the NAMEDATALEN constant in src/include/pg_config_manual.h.
```

这个文档也说了，如果想修改的话需要修改src/include/pg_config_manual.h的NAMEDATALEN常量，也就是说想把这个长度改长的话，得修改源码在重新编译pg才行。而且这个长度不仅仅是字段名，包括表名也是一样，都不能超过63，创建表超过也会给你截断掉，这个比较坑的就是在应用程序里面不报错，只有当使用到这个表时才发现对不上。

2.如果你不想重新编译pg，那就只能从程序里面去处理最大长度问题，可以在生成字段或者表名里面加一个判断函数，如果超过了63，会被修改为一个比较短的名字，是否需要储存原来超长的字段名也可以根据场景来控制，假设要储存，中间加一张映射表，解决长名字和短名字之间的映射问题。



# 参考资料：



https://eichisanden.hateblo.jp/entry/2018/10/06/231210

 https://www.postgresql.org/docs/current/sql-syntax-lexical.html#SQL-SYNTAX-IDENTIFIERS

 https://www.postgresql.org/docs/9.4/sql-syntax-lexical.html

 https://til.hashrocket.com/posts/8f87c65a0a-postgresqls-max-identifier-length-is-63-bytes