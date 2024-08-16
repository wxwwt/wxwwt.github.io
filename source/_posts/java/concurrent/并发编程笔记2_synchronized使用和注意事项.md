---
title: 并发编程笔记2_synchronized使用和注意事项
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: concurrent
---
### 并发编程笔记2_synchronized使用和注意事项

#### 前言：
上一篇学习了并发bug的成因，我们知道当一个线程的时间片使用完的时候，操作系统就会切换到另外一个线程，如果这两个线程访问了相同的资源，可能就会导致并发问题。
我们可以想到如果这个共享的资源一次只能一个线程访问，其他线程不能访问的话，就不会因为切换线程而产生的问题了。java并发编程中就提供了这样的机制，互斥锁来保证一次只有一个线程能访问共享的资源。
java中有synchroized和lock，今天我们先来看下synchroized关键字。

#### 使用方法：
synchroized分为修饰方法和代码块：
修饰方法：
```java
// 修饰方法  锁为对象
public synchronized void method1() {
       // 处理过程
   }

// 修饰静态方法 锁是类
public synchronized static void method2() {
       // 处理过程
   }
```

修饰代码块：
```java
// 修饰代码块 锁为对象
public void method3() {
      synchronized (this) {
          // 处理过程
      }
  }

// 修饰代码块 锁为类
  public void method4() {
      synchronized (Test.class) {
          // 处理过程
      }
  }
```

我们来看一个经典的例子对个线程对一个共享变量进行加法
```java
public class Test {

    private int i = 0;

    public void add() {  // (1)
        i++;
    }

    public static void main(String[] args) throws InterruptedException {
        Test test = new Test();
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                test.add();
            }
        });
        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 10000; i++) {
                test.add();
            }
        });
        t1.start();
        t2.start();

        t1.join();
        t2.join();
        System.out.println("i的值是" + test.i); // (2) i的值是18084
    }
}
```
预计的结果是20000，计算出来的值是小于20000的，线程t1和线程t2几乎同时启动，各自会将i的值读入到自己的缓存中，因为一个线程时间片用完了，操作系统会切换到另一个线程，
此时线程t1缓存中的值还没有刷新到内存中，线程t2对i增加是自己缓存中的值，然后t2增加完以后在将i增加后的值，刷到内存中。t1相加的操作等于没做，所以最终的i的值会小于20000.

当在（1）处加上synchroized的关键字后，变成：
public synchronized void add() {
线程t1运行add()函数时，先进入临界区（就是被synchroized修饰的部分），获得到monitor的锁，此时如果进行了线程切换，另一个线程因为没有获得到锁，所以会被阻塞住，当时间片用完。
线程在被切换为t1时，可以继续执行。这样的同时只允许一个线程操作的互斥锁就保证了共享变量不会出现问题。

#### 可重入锁
可重入性：就是当一个线程获得锁以后，此线程可以重复获得锁，并且不会阻塞。比如下面的例子如果线程1调用add1已经获取了锁，在调用add2又获得了锁，线程1还是可以正常运行的，这就是可重入锁，
java的另一种锁ReentrantLock(可重入锁也是一样的效果)

```java
public synchronized void add1() {
              add()2;
          }

public synchronized void add2() {
            // 处理内容
        }
```

#### synchronized的实现
我们下面的Test类进行javac Test.java编译 , 在对class文件进行反编译javap -v Test，查看下附加的信息
```java
public class Test {
    public synchronized void method1(){
    }

    public void method2(){
        synchronized (Test.class){
        }
    }
}
```
查看信息
```java
public synchronized void method1();
   descriptor: ()V
   flags: ACC_PUBLIC, ACC_SYNCHRONIZED
   Code:
     stack=0, locals=1, args_size=1
        0: return
     LineNumberTable:
       line 12: 0

 public void method2();
   descriptor: ()V
   flags: ACC_PUBLIC
   Code:
     stack=2, locals=3, args_size=1
        0: ldc           #2                  // class com/algorithm/leetcode/leetcode/Test
        2: dup
        3: astore_1
        4: monitorenter
        5: aload_1
        6: monitorexit
        7: goto          15
       10: astore_2
       11: aload_1
       12: monitorexit
       13: aload_2
       14: athrow
       15: return

```
可以看到java中实现同步方法是使用的ACC_SYNCHRONIZED控制的，实现同步代码块是使用monitorenter和monitorexit来控制的，monitorenter表示进入临界区，monitorexit表示从临界区退出来。
synchroninzed是由monitor对象控制的，任意一个java对象都可以成为monitor对象（可以简单的理解为synchroized的锁），它里面保存着拥有该锁的线程的信息，从这信息可以知道是哪个线程正持有这个锁。
在jvm中的对象是由实例变量，对象头，填充数据等组成，而monitor就保存在对象头里面，对象头的MarkWord中的LockWord指向monitor的起始地址，这样就知道是哪个线程持有着锁。

#### 注意事项：
下面是java并发编程实战课程里面的一个题目：
```java

class SafeCalc {
  long value = 0L;
  long get() {
    synchronized (new Object()) {
      return value;
    }
  }
  void addOne() {
    synchronized (new Object()) {
      value += 1;
    }
  }
}
```
这样使用synchroized是不生效的哦，看完上面的内容我们知道锁的地址是保存在对象头里面的，new了两个对象，那么就是两个不同锁，所以不会起作用。

#### 参考资料：
1.https://time.geekbang.org/column/intro/159   极客时间并发编程实战
2.《java并发编程实战》 Doug lea
3.http://www.hollischuang.com/archives/1883   Synchronized的实现原理
4.https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-2.html#jvms-2.11.10
5.https://blog.csdn.net/sc9018181134/article/details/80643360 （klass对象  我的另一个博客）
