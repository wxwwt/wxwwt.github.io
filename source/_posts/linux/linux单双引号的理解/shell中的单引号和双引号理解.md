---
title: shell中的单引号和双引号理解
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: linux单双引号的理解
---
﻿#### 问题描述:
&nbsp;&nbsp;&nbsp;&nbsp;最近在写shell脚本的时候，涉及到一个使用shell脚本发送json数据的问题，就是发送的json数据双引号不见了，导致数据格式不正确,收到了错误的响应。后来仔细查看了资料才发现自己之前对shell单引号和双引号的理解有一些问题，在此记录一些现象和结果。
#### 问题解析：
&nbsp;&nbsp;&nbsp;&nbsp;1.首先，我这边使用的是bash脚本，放一下[bash脚本的手册地址](https://www.gnu.org/software/bash/manual/html_node/)；
&nbsp;&nbsp;&nbsp;&nbsp;2.然后我们看一下官方的手册里面是怎么介绍的：
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;2.1 单引号：
```
Single Quotes：
Enclosing characters in single quotes (‘'’) preserves the literal value of each character within the quotes. A single quote may not occur between single quotes, even when preceded by a backslash.
```
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;翻译出来就是:用单引号（'''）括起字符可以保留引号中每个字符的字面值。单引号之间可能不会出现单引号，即使前面有反斜杠也是如此。

我的理解是单引号中的值都是会直接输出字符或这字符串的字面量，不会去解析各种变量或者其他的符号，而且必须是成对出现的。如果两个单引号之间有单引号，或者两个单引号之间有反斜杆的单引号都是不会结束的情况，必须等待新的单引号出现，让它们成对了才会结束。（__这里的意思是bash的解释器会对单引号去解析，只有成对的时候才会结束，否则会一直等待，所以呢单引号对号都是成对的使用，虽然我也不知道单个的单引号有什么用__）。
下面举几个栗子用来解释一下刚才说的：

![图一](http://ww1.sinaimg.cn/large/6a1bd1f1ly1g1m17qcbb5j20dw03c744.jpg)
![图二](http://ww1.sinaimg.cn/large/6a1bd1f1ly1g1m18m292zj20d801qq2q.jpg)![在这里插入图片描述](http://ww1.sinaimg.cn/large/6a1bd1f1ly1g1m191p2evj20dk035mx0.jpg)
可以看到前两张图，输入了三个单引号，或者两个单引号之间是一个带反斜杆的单引号。都会出现>的符号，意思是等待继续的输入。第三张图输入了单引号以后，出现了；号，表示结束了。说明解释器对单引号都是要成对的去解析。
&nbsp;&nbsp;&nbsp;&nbsp; 2.2 双引号：
```
Double Quotes
Enclosing characters in double quotes (‘"’) preserves the literal value of all characters within the quotes, with the exception of ‘$’, ‘`’, ‘\’, and, when history expansion is enabled, ‘!’. When the shell is in POSIX mode (see Bash POSIX Mode), the ‘!’ has no special meaning within double quotes, even when history expansion is enabled. The characters ‘$’ and ‘`’ retain their special meaning within double quotes (see Shell Expansions). The backslash retains its special meaning only when followed by one of the following characters: ‘$’, ‘`’, ‘"’, ‘\’, or newline. Within double quotes, backslashes that are followed by one of these characters are removed. Backslashes preceding characters without a special meaning are left unmodified. A double quote may be quoted within double quotes by preceding it with a backslash. If enabled, history expansion will be performed unless an ‘!’ appearing in double quotes is escaped using a backslash. The backslash preceding the ‘!’ is not removed.
The special parameters ‘*’ and ‘@’ have special meaning when in double quotes (see Shell Parameter Expansion).
```
大概意思是说：双引号中的信息会保留字面量，但是同时会对$,`,\，这些符号做出特殊的解析。就是双引号中的变量和转义，和函数操作可以被正常解析出来。这个比较好理解，接下来我们看下单引号和双引号使用的一些栗子，加深一下我们的理解。

#### 实例：
直接上图：![在这里插入图片描述](http://ww1.sinaimg.cn/large/6a1bd1f1ly1g1m1sa6eywj214c0ctgmi.jpg)
输出1,2，应该没有什么问题都是输出的字面量的字符串。
输出3,4，就是展示了单引号和双引号的区别，单引号继续输出了字符串，而双引号输出了变量a的值。
输出5,6呢，其实就是我遇到的问题，脚本中需要使用到日期的变量，并且放入到json的数据中。
输出5如果直接使用单引号，肯定行不通，因为不解析变量。
输出6呢，虽然最外层使用了双引号，内部可以解析变量，但是发现问题没有，变量外面是没有双引号的，而json的数据格式是{"key":"value"}。也是不符合的，__原因在于shell解释器分辨不出来双引号是在第几层，仅仅查到一堆双引号就把它们结为夫妻（一对对的双引号进行解析），所以输出6的解析过程是"'{"解析出'{,第二对双引号":"解析出：，第三对双引号"\$start_date"解析出start_date的值,依次类推。得出了'{startDay:2019-03-31 00:00:00,endDay:2019-03-31 23:59:59}'__。
输出7相当于就是正确的输出了json格式的数据，原理也很简单在输出6已经解释清楚。
#### 总结：
1.__写shell脚本的时候，如果不需要解析里面的内容，就使用单引号，反之，双引号；__
2.__记住shell解析单引号和双引号的规则，是就近原则，遇到一对单/双引号，就会解析出其中的内容，而不是根据什么最外层，最内层这种层级关系去解析的,这点要记住。所以在输入json或者其他的格式的数据的时候，混合使用单/双引号的时候要注意使用的顺序，否则得到的结果并不是你预想的那样__

