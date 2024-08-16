---
title: oop-klass模型
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: oop-klass模型
---
jvm对象模型可以从[hotspot7的源代码](https://github.com/wxwwt/jdk7u-hotspot/tree/master/src/share/vm/oops)的github上获取到进行学习.  
注：本文内容都是以jdk7对应的hotspot虚拟机为基础进行分析的.

一.oop-klass的层级关系  
首先，今天讲的东西是java对象在jvm层面的具体映射，它叫做oop-klass模型。咱们先来看看这个模型是怎么组成的。
从[oopsHierarchy.hpp](https://github.com/wxwwt/jdk7u-hotspot/blob/master/src/share/vm/oops/oopsHierarchy.hpp)的源代码，具体可以点进入到官方的github仓库里面看看，我这里放的是我fork的链接。（读者：为啥不直接放官方的链接，我：咱这不是心存私心引一下流量到自己的github嘛，虽然并没有啥内容可看，捂脸哭表情）。
好了废话时间结束，咱们先看看jvm对象模型的层次结构。    

引用Hotspot实战里面的一张图,oop各模块的组成:
![oop各模块组成](https://user-gold-cdn.xitu.io/2020/3/22/17102692b769ddb7?w=769&h=753&f=png&s=507702)

oop层级部分:
![oop层级](https://user-gold-cdn.xitu.io/2020/3/22/171026a00c9e01a6?w=658&h=343&f=png&s=46155)

klass层级部分:
![klass层级](https://user-gold-cdn.xitu.io/2020/3/22/171026a82c29140e?w=644&h=570&f=png&s=45129)
上来就是三张图，c++写的，虽然咱们不能完全看明白，但是从定义属性这些大致也有了个了解，这个模型是由oop类和klass组合起来的，然后这两个大类下面又有很多的子类。

二.oop层级部分:
看完了jvm整个模型的构成，我们先来看看oop部分的作用。
参考hotspot源码中的[oop.hpp](https://github.com/wxwwt/jdk7u-hotspot/blob/master/src/share/vm/oops/oop.hpp)部分.

oop.hpp翻译:  
1.oopDesc是对象类的顶层基类。  
2.Desc类描述了Java对象的格式，以便可以从C ++访问这些字段。  
3.oopDesc是抽象的。（完整的类层次结构见oopHierarchy）.  
4.oop类不允许虚拟功能.  

我们可以将oop的部分理解成java部分的对象，然后klass是给jvm使用的内部对象。
下面是oop各子类继承关系和各个子模块作用。

![oop继承关系](https://user-gold-cdn.xitu.io/2020/3/22/171026c1b71b5790?w=1151&h=585&f=png&s=227503)

![oop的用途](https://user-gold-cdn.xitu.io/2020/3/22/171026c66aa45d4d?w=1283&h=693&f=png&s=447566)

不同类型有不同的用处,比如;创建一个Java对象实例的时候，jvm会创建一个instanceOopDesc对象在java层面来表示这个Java对象。如果是创建一个方法的时候，JVM会创建一个methodOopDesc对象来表示这个方法。


![oop对象头](https://user-gold-cdn.xitu.io/2020/3/22/171026d43a9400dd?w=431&h=217&f=png&s=13603)
所有的子类oop都是继承的oopDesc,我们知道jvm中的对象包含了三部分:对象头，实例数据，对齐填充。  
其中_mark，_metadata一起合成为了对象头，里面包含了锁状态标志、线程持有的锁等标志，_metadata包含了两个指针，指向klass，klass包含了实例对象的元数据。

三.klass层级部分
我们知道Hotspot的底层实现使用C++实现的，C++也是一门面向对象的语言。因此，当讲到jvm的对象模型的实现时候，可能会很自然地想到，只要java对象底层对应一个C++对象，问题就解决了。在查看了一下参考资料发现，设计者为了避免每个对象中都有一个C++VTBL指针，采用了将对象模型一分为二的做法:oop-klass模型。看到这里可能会有点懵逼这个C++VTBL指针是干嘛的。这个不是本文重点，先略过。首先，我们这么理解，一个C++对象要实现两部分功能：
1.语言层面的对象;
2.包含虚函数;(可以先简单的理解为实现多态要用的东西).

然后我们看klass源码中的描述  
klass.hpp翻译:    
Klass是klassoop的一部分，它提供了:  
1.语言层面的java类对象(方法字典等)  
2.为该对象提供了虚拟机的调度行为  
这两个功能完成了一个C++类的功能。顶层的Klass实现第一点，而所有子类提供额外的虚拟功能实现了第二点。
因此，java的oop-klass对象模型中,klass和其子类就实现了C++对象中的这两个功能.
在实现对象模型中出现oop/klass二分法的一个原因是我们不希望每个对象都有一个C++ vtbl指针。因此，正常的oops没有任何虚函数。相反，他们将所有虚函数转发给它们的klass，klass具有一个vtbl并根据对象执行C++调度实际类型。（有关转发代码，请参阅oop.inline.hpp。）.所有实现此调度的函数都以“oop_”为前缀！

klass的层级结构:
![klass继承](https://user-gold-cdn.xitu.io/2020/3/22/171026db38e414e0?w=948&h=747&f=png&s=273389)

四.klass和oop之间的联系

![oop和klass的关系](https://user-gold-cdn.xitu.io/2020/3/22/171026e21184ec3c?w=1206&h=900&f=png&s=197617)

所有的oop对象实例都会有一个指针，指向对应的instanceKlass，同时instanceKlass也会有一个instanceKlassKlass来描述它。这种链条直到KlassKlass终止。因为klassKlass的指针始终是指向自己的。[参考内容](http://openjdk.java.net/groups/hotspot/docs/FOSDEM-2007-HotSpot.pdf)  

__注__:这里看大家源码的时候可能会有一个疑问,Klass_vtbl的源码中([就是Klass.hpp](https://github.com/wxwwt/jdk7u-hotspot/blob/master/src/share/vm/oops/klass.hpp))，没有出现_mark和_metadata，那怎么从instanceKlass指向instanceKlassKlass呢？在JDK8之前(本文章是基于JDK7)，方法区内的描述类型的元数据对象，也是由GC管理的。所有由GC统一管理的对象，都要继承自oopDesc，因为oopDesc有_mark和_metadata，所以作为它的子类的Klass都包含了头信息_mark和metadata。从JDK8开始，类型元数据都移出了GC堆，所以Klass这个属性可以直接指向Klass类了。


参考资料:

1.[深入理解多线程（二）—— Java的对象模型](http://www.hollischuang.com/archives/1910)  
2.[HotSpotVM 对象机制实现浅析#1](https://yq.aliyun.com/articles/20279)  
3.[借HSDB来探索HotSpot VM的运行时数据](http://rednaxelafx.iteye.com/blog/1847971)  
