---
title: java四种引用简介
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: basic
---
### 引语：
&nbsp;&nbsp;&nbsp;&nbsp;我们知道java相比C，C++中没有令人头痛的指针，但是却有和指针作用相似的引用对象（Reference），就是常说的引用，比如，Object obj = new Object()；这个obj就是引用，它指向的是真正的对象Object的地址，不过今天要说的是java中的四种引用。有人可能比较懵逼，四种引用？是的，从JDK1.2之后，java对引用这块的概念进行了扩充，按照引用的强度分为了四种引用：强引用，软引用，弱引用，虚引用。下面就让我们来看看这四种引用都具体的情况吧。

### 1.强引用
#### 1.1介绍：
我们平时代码中使用得最多的引用，对象的类是：StrongReference。就比如上面说的Object obj = new Object()；我们再熟悉不过了，作为最强的引用，只要引用还存在着，垃圾收集器就不会将该引用给回收，即使会出现OOM（内存溢出）。就是说这种引用只要引用还一直指向的对象，垃圾收集器是不会去管它的，所以它被称为强引用。不过如果
```java
Object obj = new Object();
obj = null;
```
obj被赋值为了null，该引用就断了，垃圾收集器会在合适的时候回收改引用的内存。
还有一种情况就是obj是成员变量，方法执行完了，obj随着被栈帧被回收了，obj引用也是一起被回收了。强引用的使用就不介绍了，地球人都知道。

### 2.软引用
#### 2.1介绍：
软引用是用来描述一些有用但是非必须的对象。对应的类是SoftReference，它被回收的时机是系统内存不足的时候，如果内存足够，它不会被回收，内存不足了，可能会发生OOM了，软引用的对象就会被回收。这样的特性是不是就像缓存？是的，软引用可以用来存放缓存的数据，内存足够的时候一直可以访问，内存不足的时候，需要重新创建或者访问原对象。

#### 2.2使用：
其实不管是软引用，弱引用，还是虚引用，代码中使用方式都是像下面这样，使用对应的Reference将对象放入到构造函数当中，然后使用的地方reference.get()来调用具体对象。
```java
Object obj = new Object();
SoftReference<Object> softReference = new SoftReference<>(obj);
softReference.get();
```
同时可以使用ReferenceQueue来把引用和引用队列给关联起来：
```java
Object obj = new Object();
ReferenceQueue<Object> refQueue = new ReferenceQueue<>();
SoftReference<Object> softReference = new SoftReference<>(obj, refQueue);
```
**所谓关联起来，其实就是当引用被回收的时候，会被添加到ReferenceQueue中，使用ReferenceQueue.poll()方法可以返回当前可用的引用，并从队列冲删除**。简单来说就是引用和引用队列关联起来（引用的构造函数传入队列），然后引用被回收的时候会被添加到队列中，然后使用poll()方法可以返回引用。

### 3.弱引用
#### 3.1介绍：
虚引用比上面两个引用就更菜了，只要垃圾收集器扫描到了它，被弱引用关联的对象就会被回收。被弱引用关联对象的生命周期其实就是从对象创建到下一次垃圾回收。对应的类是WeakReference。
#### 3.2使用：
```java
public static void main(String[] args) throws InterruptedException {
      Object obj = new Object();
      ReferenceQueue<Object> refQueue = new ReferenceQueue<>();
      WeakReference<Object> weakRef = new WeakReference<>(obj, refQueue);
      System.out.println("引用：" + weakRef.get());
      System.out.println("队列中的东西：" + refQueue.poll());
      // 清除强引用, 触发GC
      obj = null;
      System.gc();
      Thread.sleep(200);
      System.out.println("引用：" + weakRef.get());
      System.out.println("引用加入队列了吗？ " + weakRef.isEnqueued());
      System.out.println("队列中的东西：" + refQueue.poll());
      /**
       * 输出结果
       * 引用：java.lang.Object@7bb11784
       * 队列中的东西：null
       * 引用：null
       * 引用加入队列了吗？ true
       * 队列中的东西：java.lang.ref.WeakReference@33a10788
       */
  }
```
可以看到当强引用被清除，手动触发GC后，弱引用回收，被加入到队列中了。
#### 3.3扩展：
WeakHashMap跟hashMap很像，差别就在于，当WeakHashMap的key（弱引用），指向的对象被回收了，weakhashMap中的对象也就消失了。不会和HashMap一样一直持有该对象，导致无法回收。
不赘述了，有兴趣的可以了解一下，[WeakHashMap](http://www.importnew.com/23182.html)。

### 4.虚引用
#### 4.1介绍：
虚引用是最弱的一种引用，它不会影响对象的生命周期，对象被回收跟它没啥关系。它引用的对象可以在任何时候被回收，而且也无法根据虚引用来取得一个对象的实例。仅仅当它指向的对象被回收的时候，它会受到一个通知。对应的类是PhantomReference。
#### 4.2使用：
有人就要问既然对对象回收没影响，那它有啥用（其实用处很少），我查阅网上的资料说是，可以用来监控对象的回收，和记录日志。简单点说就是对象被回收的时候，和虚引用相关的队列知道了实例对象被回收了。这个时候我们可以记录下来，知道对象是什么时候被回收的。
从而起到监控的作用。
```java
public static void main(String[] args) throws Exception {
       Object abc = new Object();
       ReferenceQueue<Object> refQueue = new ReferenceQueue<Object>();
       PhantomReference<Object> abcRef = new PhantomReference<Object>(abc, refQueue);
       System.out.println("队列中的东西：" + refQueue.poll());
       abc = null;
       System.gc();
       Thread.sleep(1000);
       System.out.println("引用加入队列了吗？ " + abcRef.isEnqueued());
       System.out.println("队列中的东西：" + refQueue.poll());
       /**
        * 输出：
        * 队列中的东西：null
        * 引用加入队列了吗？ true
        * 队列中的东西：java.lang.ref.PhantomReference@7bb11784
        */
   }
```
发现队列中有引用了，就可以添加日志记录了。

### 5.总结：
将人比作垃圾收集器，引用比作食物，我们来总结下四种引用：
强引用是毒药，即使你很饿了你也不会去吃它；
软引用是零食，不饿的时候不吃，饿了饥不择食，零食也能填饱肚子；
弱引用是饭菜，到了吃饭时间（垃圾回收），就吃饭菜；
虚引用是剩菜，当你吃完东西（回收完对象），就回剩下剩菜，别人就知道你吃过饭了。

#### 5.1表格对比：
|引用 | 回收时机 | 使用场景 |
| ------ | ------ | ------ |
| 强 | 不会被回收 | 正常编码使用 |
| 软 | 内存不够了，被GC | 可作为缓存 |
| 弱 | GC发生时 | 可作为缓存（WeakHashMap） |
| 虚 | 任何时候 | 监控对象回收，记录日志 |

### 参考资料：
1.https://blog.csdn.net/l540675759/article/details/73733763
2.https://www.iteye.com/topic/587995
3.https://www.geeksforgeeks.org/types-references-java/
4.https://blog.csdn.net/aitangyong/article/details/39453365  
