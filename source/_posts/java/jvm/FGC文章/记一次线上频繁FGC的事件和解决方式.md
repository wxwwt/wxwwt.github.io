---
title: 记一次线上频繁FGC的事件和解决方式
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: FGC文章
---
﻿### 问题描述:
&nbsp;&nbsp;&nbsp;&nbsp;早上去公司上班,突然就邮件一直报警,接口报异常,然后去查服务器的运行情况,发现java的cpu爆了.接着就开始排查问题
### 问题解决过程:
1.先服务器(centos7)上,使用了top和uptime命令,发现时java的cpu爆了,超过100%了,导致后续的服务无法正常提供;
2.调整了负载均衡,下掉了有问题的那几台机器;
3.使用jps找到了运行着的tomcat的pid,这里假设为10086;
4.使用jstat -gcutil 10086 500 10 (意思是对pid为10086的线程,每500ms显示各分代的内存使用情况), 这里给一下部分jvm的参数设置,如下:![在这里插入图片描述](https://img-blog.csdnimg.cn/20190310232309220.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NjOTAxODE4MTEzNA==,size_16,color_FFFFFF,t_70)
可以看到对新生代使用的是ParNew收集器,对老年代使用的是CMS收集器，CMSInitiatingOccupancyFraction=80说明当使用率超过80%的时候触发垃圾回收。然后发现,线上老年代一直超过80%的使用率,几乎一秒不到就进行了一次FGC,这么频繁的FGC导致了服务无法正常运行;
5.使用jmap -histo 查看了是哪些对象的数量最多，如果参数是-histo：live的话，会在进行一次FGC后，显示当前的使用数量最多的实例。查看后发现有大量的ConcurrentHashMap的Node节点实例，于是去代码中搜索使用到ConcurrentHashMap的地方，有好几处使用比较频繁的地方，看了代码后分析，可能是图片上传，下载的问题，但也不能断定。
6.使用jmap -dump：format=b，file=/usr/local/tomcat/dump1将内存的情况给拉下来.如下：![在这里插入图片描述](https://img-blog.csdnimg.cn/20190310232804111.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NjOTAxODE4MTEzNA==,size_16,color_FFFFFF,t_70)文件生成后,将dump1放入到eclipse的mat中进行分析。直接显示了一个疑似内存泄漏的问题。然后分析dump文件给出的信息，发现一个叫IdleConnectionReaper的类。dump文件里面说的内存泄漏的大概的意思就是说，IdleConnectionReaper这个类里面的ArrayList存放的东西太多了，爆掉了。如下：![在这里插入图片描述](https://img-blog.csdnimg.cn/2019031023282023.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NjOTAxODE4MTEzNA==,size_16,color_FFFFFF,t_70)
从oss的jar包里面找到这个类以后，简单的看一下这个类的构成，如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20190310233017745.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NjOTAxODE4MTEzNA==,size_16,color_FFFFFF,t_70)
通过查找一些官方的资料和源代码的阅读，发现这个是一个oss的守护线程，用来检测上传或者下载的工作线程，每60秒就会去检查一下空闲的工作线程，并且将它们回收。然后它内部有一个静态的ArrayList，里面保存的是ossClient的链接，默认是1024个。
7.所以大概原因找到了，就是ossClient的链接太多了，扛不住了，所以一直在进行FGC，导致服务不可用了，最后找到相关的代码，发现有个小方法里面在每次上传或者下载的时候，都会去创建一个ossClient。修改了代码将ossClient调用的地方改成了单例。修改完线上跑了一段日子，后来也没有出现过这样的问题。
### 总结：
1.大量的请求，调用的地方要注意是否会导致内存的大量消耗，尽可能使用池化技术，单例等，减少创建，销毁的系统开销；
2.CMS 的几个缺点，可以参考《深入java虚拟机》，对CPU占用会比较高，无法处理浮动垃圾，还有就是CMS使用的是标记-清除算法，会导致大量的空间碎片，碎片过多的话，导致分配大对象很困难，所以不得不进行FGC，也可能是这个原因导致了本文说的一直FGC的问题。解决方式：
-XX:+UseCMSCompactAtFullCollection:使用并发收集器时,开启对年老代的压缩（会整理内存碎片，默认是开启的）.
-XX:CMSFullGCsBeforeCompaction=0:上面配置开启的情况下,这里设置多少次Full GC后,对年老代进行压缩，会使得FGC的时间变长，但是可以提升内存空间的使用率，让大对象可以更容易分配，而不需要多次FGC。

