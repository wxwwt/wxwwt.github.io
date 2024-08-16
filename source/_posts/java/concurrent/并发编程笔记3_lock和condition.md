---
title: 并发编程笔记3_lock和condition
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: concurrent
---
## 并发编程笔记3_lock和condition

### 前言：
&nbsp;&nbsp;&nbsp;&nbsp;在jdk1.5出现了lock，性能比synchroized好很多，但在jdk1.6版本之后，synchroized经过了很多优化，性能已经提升了很多，那为什么还会有显示锁lock呢？
直接用synchroized不就可以了吗，还不需要自己手动去释放锁，避免了程序员忘记释放锁的问题。其实synchroized还是存在一些问题的，有一些synchroized无法
处理的问题，比如被synchroized修饰的代码部分一直处于处理中，这样线程会一直阻塞住，其他线程也没法再去获取这个资源。有时候我们可能希望有这么一个
超时时间之类的东西，获取不到资源超过了某个时间，就不去获取了。或者A，B两个线程因为竞态条件构成了死锁，那么我们能不能直接中断其中一个线程，
让资源能够释放出来。这时候lock孕育而生。

### lock的使用：
我们先看下lock显示锁是怎么使用的，下面是lock的标准使用形式：
```java
Lock lock = new ReentrantLock();
lock.lock();
try{
 // 进行操作
} finally {
lock.unlock();
}
```

### lock和synchroized的区别和优势：
使用上应该没有什么大问题，语义上和synchroized是一样的。我们再来看看lock的不同之处或者说lock和synchroized相比有哪些不同：
我们先看下lock接口提供的方法：
```java
public interface Lock {

    void lock();

    void lockInterruptibly() throws InterruptedException;

    boolean tryLock();

    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

    void unlock();

    Condition newCondition();
}
```
可以看到lock接口总共六个方法，我们除去正常就应该有的加锁解锁方法lock(),unlock()，
```java
void lockInterruptibly() throws InterruptedException;

boolean tryLock();

boolean tryLock(long time, TimeUnit unit) throws InterruptedException;
```
lockInterruptibly()用来中断获取锁，
tryLock()可以非阻塞的获取锁，没有拿到锁就返回了，不会阻塞，
 tryLock(long time, TimeUnit unit）可以设置超时时间，时间到了没拿到锁也返回了。
 有了这三个方法是不是就可以解决我们之前说的哪些问题了？

lock另外的优势还有
1.synchroized是非公平锁，lock则可以选择是公平锁还是非公平锁。
所谓公平非公平呢，就是线程在获取锁的时候，是否是按照线程的请求顺序去分配资源的。比如A，B，C三个线程先后去竞争资源1，当资源1被释放的时候，如果是无序的那就是
非公平锁。如果是按照A，B，C的先后顺序去分配资源1，那就是公平锁。可以类比我们排队，按照我们的单号来上菜是公平的，我们先来点了菜，结果老板先给后面来的人上菜了，
那就不公平了呀。所以需要实现公平那就只能使用支持公平锁的lock了，顺便提一句哈，为了保证公平需要消耗计算机资源，所以性能比较差，因此一般都是使用非公平锁。
咱们看ReentrantLock的构造方法就是如此。
```java
// 默认的无参构造器就是非公平的
public ReentrantLock() {
  sync = new NonfairSync();
}
// 创建非公平锁
public ReentrantLock(boolean fair) {
  sync = fair ? new FairSync() : new NonfairSync();
 }
```

2.对非块级结构进行加锁，我们知道synchroized的加锁粒度是代码块，然而有时候我们需要更加细粒度的加锁解锁，这时候lock的优势就出来了，
这里可以参考锁分段技术对集合的操作，比如对concurrentHashMap的并发处理。

 所以我理解的lock更像是对synchroized的一个加强版和补充，不过并发编程本来就是比较难写的内容，
 什么时候用synchroized，什么时候用lock，建议参考如下意见：
 引自java并发编程的一句话：
```
当需要使用一些高级功能时才应该使用ReentrantLock（lock的常用实现类），这些功能包括：可定时的，可轮询的与可中断的（对应lock的三个方法）锁获取操作，
公平队列，以及非块结构的锁。否则还是应该优先使用synchroized。
```
### lock保证可见性
这个应该算是lock的核心部分吧，或者说我们的学会使用happen-before来解决并发问题的时候可以参考lock保证可见性的写法。
如下：
```java
Lock lock = new ReentrantLock();
int value;
lock.lock();
try{
  value+=1;
} finally {
lock.unlock();
}
```
我们知道value+=1，进行了加锁解锁，它是怎么对另一个线程保持可见性的呢？或者说怎么保证另一个线程读取到的值不是自己原来保存的缓存呢？
保持可见性我们理所当然会想到volatile关键字，没错，你猜对了，就是用了volatile。过程就不贴出了，ReentrantLock里面东西很多，如果你一个个
看过去会发现一个getState()的方法，来源于AbstractQueuedSynchronizer这个类。
```java
    /**
     * The synchronization state.
     */
    private volatile int state;
```
这个字段是volatile的，获取锁的时候会读写这个state，解锁的时候也会去读写这个state。在结合我们上面的代码，过程也就是在value+=1之前读取了一次volatile
修饰的state，value+=之后又对state进行了读写。
好的，重点来了，结合jvm的happen-before原则（怕有人不清楚，贴一下这里会使用到的几条规则的具体内容，来自《深入理解java虚拟机》）：
1.程序次序规则：在一个线程内，按照程序代码顺序，书写在前面的操作先行发生于书写在后面的操作。
2.volatile 变量规则：对一个volatile变量的写操作线性发生于后面对这个变量的读操作，这里的“后面”指的是时间上的先后顺序
3.传递性：如果操作A先行发生于操作B，操作B先行发生于操作C，那么可以得出操作A线程发生于操作C的结论。
这些规则都是jvm会保证的，我们只需要记住并会使用即可。
好的，墨水喝完，我们用用看：
1.在线程1中，value+=1是先行发生于解锁unlock()，程序次序原则的保证
2.因为线程1解锁都会对state进行读写，此时再有另一个线程2获取到这个lock时要加锁会对state进行读取，根据第二条写先于读，所以第一个线程的解锁先于第二个线程的加锁
3.根据传递性可知，操作value+=1是先于线程2对state的加锁
因此，线程2加锁前，value+=1是已经完成了的。所以线程2能得到正确的value的值。看到这里我们遇到类似的并发问题的时候也可以参考这种思路，使用这三个规则来保证数据的可见性
和程序的正确执行。

### 总结：
1.lock的使用方式
2.lock和synchroized的区别和优势，这两种锁改什么时候使用，或者说两种锁的使用场景（记住文章摘自java并发编程的那句话哦）
3.lock是怎么用happen-before保证可见性的（最好学会这个套路可以扩展开来）
