---
title: linux中的零拷贝
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: linux中的零拷贝
---
### linux中的零拷贝技术

大家平时一定听说过零拷贝这个词，通常可能是在使用netty，kafka等框架的时候听到的，
如果你没用听说过这个词，也没有关系咱们今天就来看一看这个零拷贝是啥？


咱们先来看看零拷贝的概念：
摘自维基百科：
零复制（英语：Zero-copy；也译零拷贝）技术是指计算机执行操作时，CPU不需要先将数据从某处内存复制到另一个特定区域。这种技术通常用于通过网络传输文件时节省CPU周期和内存带宽。
从这句话上咱们可以知道不需要CPU来复制数据，提高了传输效率。因此为了提高传输效率，在网络非常发达的今天显得更加有必要，想想看咱们平时动不动就4G，5G看电影刷短视频等，需要大量
的网络传输，数据量越大，复制拷贝的成本就越高。所以咱们还是有必要好好了解一下零拷贝技术，毕竟在生活和工作中都有很大的用处。
咱们知道每种语言中很多特性其实都是参考操作系统的，比如java中的JMM内存模型，就是参考的操作系统的内存模型。所以我们讲java的零拷贝之前，先学习一下操作系统中的零拷贝是怎么回事，
学完之后，那再学习java的零拷贝就回清晰很多。

我们看下linux的传统传输流程图：
首先引入一个概念：
DMA(Direct Memory Access，直接存储器访问) ： 它是所有现代电脑的重要特色，它允许不同速度的硬件装置来沟通，而不需要依赖于 CPU 的大量中断负载。否则，CPU 需要从来源把每一片段的资料复制到暂存器，然后把它们再次写回到新的地方。在这个时间中，CPU 对于其他的工作来说就无法使用。
可以简单的理解为没有DMA之前，咱们把数据从一个地方转移到另一个地方需要CPU来调度，并且在CPU处理的时候，它是阻塞的，其他任务没法处理。有了DMA之后，因为不需要CPU来调度了，
数据可以直接移动，所以CPU可以空出来去干更多其他的事情，数据的储存和移动也变得更加的方便和快速。

![](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/linux%E7%9A%84%E9%9B%B6%E6%8B%B7%E8%B4%9D/%E6%B5%81%E7%A8%8B1.png)

可以看出这里一共复制了四次，上下文切换了四次。

读操作（复制两次，上下文切换两次）：
1.用户进程通过 read() 函数向内核（kernel）发起系统调用，上下文从用户态（user space）切换为内核态（kernel space）。
2.CPU利用DMA控制器将数据从主存或硬盘拷贝到内核空间（kernel space）的读缓冲区（read buffer）。
3。CPU将读缓冲区（read buffer）中的数据拷贝到用户空间（user space）的用户缓冲区（user buffer）。
4.上下文从内核态（kernel space）切换回用户态（user space），read 调用执行返回。

写操作（复制两次，上下文切换两次）：
1.用户进程通过 write() 函数向内核（kernel）发起系统调用，上下文从用户态（user space）切换为内核态（kernel space）。
2.CPU 将用户缓冲区（user buffer）中的数据拷贝到内核空间（kernel space）的网络缓冲区（socket buffer）。
3.CPU 利用 DMA 控制器将数据从网络缓冲区（socket buffer）拷贝到网卡进行数据传输。
4.上下文从内核态（kernel space）切换回用户态（user space），write 系统调用执行返回。




### 零拷贝的方式：
上面这个过程是不是很复杂，进行了很多次的切换和复制，如果用户空间可以直接读取数据的话是不是可以提高效率？
好的，我们看看linux系统的零拷贝都有哪些方式：
#### 直接 I/O：
对于这种数据传输方式来说，应用程序可以直接访问硬件存储，操作系统内核只是辅助数据传输：这类零拷贝技术针对的是操作系统内核并不需要对数据进行直接处理的情况，数据可以在应用程序地址空间的缓冲区和磁盘之间直接进行传输，完全不需要 Linux 操作系统内核提供的页缓存的支持。
![](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/linux%E7%9A%84%E9%9B%B6%E6%8B%B7%E8%B4%9D/%E7%9B%B4%E6%8E%A5io.png)
这个没啥好多说的，直接传过了内核空间，内核空间没有参与数据的复制，用户空间直接与硬件进行了交互。

#### 减少拷贝次数：
在数据传输的过程中，避免数据在操作系统内核地址空间的缓冲区和用户应用程序地址空间的缓冲区之间进行拷贝。有的时候，应用程序在数据进行传输的过程中不需要对数据进行访问，那么，将数据从 Linux 的页缓存拷贝到用户进程的缓冲区中就可以完全避免，传输的数据在页缓存中就可以得到处理。在某些特殊的情况下，这种零拷贝技术可以获得较好的性能。Linux 中提供类似的系统调用主要有 mmap()，sendfile() 以及 splice()。
##### mmap():
和传统的区别就是read操作变成了mmap，之后用户空间会和内核态共享同一块内核缓冲区，读入的数据都在这个内核缓冲区里面。写入的话还是和原来一样。
![](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/linux%E7%9A%84%E9%9B%B6%E6%8B%B7%E8%B4%9D/mmap.png)


##### sendfile():
sendfile对于mmap来说更加优化了一步，数据从缓冲复制到到socket直接都是在内核空间一次性完成的，用户空间只是发起了sendfile的调用，减少了复制和上下文切换的开销。
![](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/linux%E7%9A%84%E9%9B%B6%E6%8B%B7%E8%B4%9D/sendfile.png)


##### splice():
splice又在上面两位前辈的基础上变更更加强大，之前sendfile只能是内核缓冲区向socket复制数据，而splice直接让讷河缓冲区和socket之间建立了一个管道可以直接相互交换数据。
![](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/linux%E7%9A%84%E9%9B%B6%E6%8B%B7%E8%B4%9D/splice.png)


#### 写时复制：
对数据在 Linux 的页缓存和用户进程的缓冲区之间的传输过程进行优化。该零拷贝技术侧重于灵活地处理数据在用户进程的缓冲区和操作系统的页缓存之间的拷贝操作。这种方法延续了传统的通信方式，但是更加灵活
写时复制是计算机编程中的一种优化策略，它的基本思想是这样的：如果有多个应用程序需要同时访问同一块数据，那么可以为这些应用程序分配指向这块数据的指针，在每一个应用程序看来，它们都拥有这块数据的一份数据拷贝，当其中一个应用程序需要对自己的这份数据拷贝进行修改的时候，就需要将数据真正地拷贝到该应用程序的地址空间中去，也就是说，该应用程序拥有了一份真正的私有数据拷贝，这样做是为了避免该应用程序对这块数据做的更改被其他应用程序看到。这个过程对于应用程序来说是透明的，如果应用程序永远不会对所访问的这块数据进行任何更改，那么就永远不需要将数据拷贝到应用程序自己的地址空间中去。这也是写时复制的最主要的优点。
