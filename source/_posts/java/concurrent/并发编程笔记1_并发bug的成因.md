---
title: 并发编程笔记1_并发bug的成因
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: concurrent
---
#### 前言:  
&nbsp;&nbsp;&nbsp;&nbsp;最近在学习并发编程的知识,打算好好学习下并发这块,之前有处理过并发的问题,但是学的不够体系,知识比较零散.
所以买了极客时间的并发课程（java并发编程实战）和《并发编程实战》从头系统化的学一遍。这里记录一下自己的学习过程和心得之类的。

&nbsp;&nbsp;&nbsp;&nbsp;首先咱们得知道为啥会产生并发编程的bug，究其根本原因是因为cpu处理速度>>内存>>I/O，这里用了">>",是远远大于的意思，极客时间里面的例子更形象说是
天上一天，人间一年。天上指的是cpu对于内存，内存对于I/O.用来凸显其处理速度的差距。因为有了这三者的差距，cpu的利用率就比较低，一直得等待内存，I/O处理完。
所以为了减少这三者的差距，做出了三种优化：  
1.CPU 增加了缓存，以均衡与内存的速度差异；  
2.操作系统增加了进程、线程，以分时复用 CPU，进而均衡 CPU 与 I/O 设备的速度差异；  
3.编译程序优化指令执行次序，使得缓存能够得到更加合理地利用。  

这三类优化相应的耶带来了三种问题，都可能会导致并发问题：  

1.缓存的可见性问题：  
&nbsp;&nbsp;&nbsp;&nbsp;上面的第一种优化，在cpu和内存之间增加了缓存，在多核cpu的情况下，每个cpu可能都有自己的缓存。现在假设一个value=0，当线程a在cpu1的缓存中给value加1，线程b在cpu的缓存中给value加1，它们再往内存中刷新value的值的时候，可能顺序并不是我们想的value变为2.
可能的执行过程是线程a，b如果都将value从内存中读入到自己的cpu缓存中，线程a先给value加了1，然后刷新到内存中value变为1，此时线程b也将value加1，刷新到内存中，覆盖了a写入的值，value还是1.这就是缓存的可见性问题，各自的cpu缓存并不知道，其他线程也对相同的值做了操作。
比较经典的例子就是，启动多个线程对同一个变量进行增加操作，得出来的值不是预想的值。

2.线程切换导致的原子性问题：  
&nbsp;&nbsp;&nbsp;&nbsp;这个问题其实比较好理解，我们学过操作系统知道，多进程操作系统在执行任务的时候，不是串行的进程a执行完，再去执行进程b，而是每个进程有一个操作系统分配的执行时间（称为是时间片），当进程a时间片消耗完了，
就切换到另一个进程。这样进程之间交替的执行，又因为时间片的时间非常短，用户是感知不出来这种切换的。介绍完进程的切换，咱们再说说像java这种的高级语言，对应的底层操作系统指令可能是好多条。
比如我们想的count+=1，对于操作系统就不是原子操作，可能是1,2,3...好几条，那么有两个线程都对count加1的时候，如果进程a中count的值还没有加完，此时操作系统给进程a的时间片用完了，切换到进程b，这时候进程b读到的还是没有增加的的值，那么计算结果就不符合我们的预期。

3.编译优化带来的指令执行次序问题：  
&nbsp;&nbsp;&nbsp;&nbsp;我们写java的知道volatile关键字有使变量可见和禁止指令重排序的作用，其实就是针对的这个问题。编译器的优化能大大提高执行效率，但是也会带来一些问题，“int a = 1；int b = 2”这种被优化成“int b = 2；int a = 1”倒是问题不大。
这里引用极客时间的例子：
```java
public class Singleton {
  static Singleton instance;
  static Singleton getInstance(){
    if (instance == null) {
      synchronized(Singleton.class) {
        if (instance == null)
          instance = new Singleton();
        }
    }
    return instance;
  }
}
```
这是双重校验锁的单例，代码执行顺序是  
1.分配一块内存M；  
2.在内存M上初始化 Singleton 对象；  
3.然后M的地址赋值给 instance 变量。  
可能会被优化成：  
1.分配一块内存M；  
2.将M的地址赋值给 instance 变量；  
3.最后在内存M上初始化 Singleton 对象。  
这样就可能会导致下面这种情况，线程a调用getInstance(),a如果处于被优化的步骤2，然后时间片用完了，此时线程b进来，执行到第一个instance == null这里，发现instance不为null，就返回了未初始化的instance。此时访问instance对象可能抛出空指针异常。
