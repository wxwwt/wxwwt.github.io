### ThreadLocal的使用和坑点


### 概念：
ThreadLocal的概念：摘自ThreaLocal的注释
```
This class provides thread-local variables.  These variables differ from
 * their normal counterparts in that each thread that accesses one (via its
 * {@code get} or {@code set} method) has its own, independently initialized
 * copy of the variable.  {@code ThreadLocal} instances are typically private
 * static fields in classes that wish to associate state with a thread (e.g.,
 * a user ID or Transaction ID).
```
这段话的大概意思是ThreadLocal是保存的线程的本地变量，访问get/set方法都是对线程独立的。
大白话就是ThreadLocal是和线程相关的，在一个线程没有结束之前，在任意方法中get/set在ThreadLocal中设置的值都是只和当前线程有关。
因此呢，ThreadLocal的使用场景也可以推测出来，可以用来在一个线程中传递参数，或者某些情况（比如session，数据库操作句柄）只跟线程相关的时候来使用。

### 源码分析：
我们接下来在看看源代码中，ThreadLocal是什么样的？
简答的贴了一下类和常用的几个方法的源代码（此博客是基于JDK1.8）
类定义：
```
public class ThreadLocal<T>
```
从类定义上可以看出ThreadLocal是支持泛型的

ThreadLocalMap:
```
static class ThreadLocalMap {
        static class Entry extends WeakReference<ThreadLocal<?>> {
            /** The value associated with this ThreadLocal. */
            Object value;

            Entry(ThreadLocal<?> k, Object v) {
                super(k);
                value = v;
            }
        }
    // 后面代码省略
}
```
这里首先我们要看到ThreadLocal中有一个静态类叫做ThreadLocalMap，它里面有一个静态类Entry（可以类比Map中的entry，保存实际的key和value，），ThreadLocalMap其实就是保存了ThreadLocal调用set方法设置的value，key就是ThreadLocal。
总结一下就是当我们使用ThreadLocal的set方法时，ThreadLocal为key，保存的泛型对象为value，存到了ThreadLocal的内部类ThreadLocalMap中，然后ThreaLocalMap的键值对实际上是放在静态类Entry里面。这里稍微提一句
Entry是继承了WeakReference弱引用，key是被弱引用的构造函数给创建的，value是强引用。（java的四种引用可以参考我之前写的文章[Java四种引用简介](https://juejin.im/post/5cea6cbd6fb9a07ee062f3a7)）

get：
```
public T get() {
       Thread t = Thread.currentThread();
       ThreadLocalMap map = getMap(t);
       if (map != null) {
           ThreadLocalMap.Entry e = map.getEntry(this);
           if (e != null) {
               @SuppressWarnings("unchecked")
               T result = (T)e.value;
               return result;
           }
       }
       return setInitialValue();
   }
```
get方法可以看出，会先获取当前线程，然后获取ThreadLocalMap，然后把值从ThreadLocalMap中取出来，如果没有ThreadLocalMap就去调用setInitialValue()设置完初始值，并返回。
set：
```
public void set(T value) {
      Thread t = Thread.currentThread();
      ThreadLocalMap map = getMap(t);
      if (map != null)
          map.set(this, value);
      else
          createMap(t, value);
  }
```
set方法根据当前的线程从ThreadLocalMap中取得，没有map就创建一个。thread中的 ThreadLocal.ThreadLocalMap threadLocals变量会指向刚才创建的ThreadLocalMap。

### 其他用法
**1.初始化值**
这样看下来，应该对ThreadLocal的实现和原理有了一个大概的认识。这里提一句，ThreadLocal直接new出来，然后去get的值是null。某些情况下所有的线程都需要有一个初始化的值，这时候可以重写 protected T initialValue()方法，如下：
```
   private static ThreadLocal<String> threadLocal2 = new ThreadLocal<String>() {
       @Override
       protected String initialValue() {
           return "override initialValue的初始值";
       }
   };
```
或者jdk1.8可以使用ThreadLocal.withInitial初始化
```
private static ThreadLocal<String> threadLocal1 = ThreadLocal.withInitial(() -> "withInitial的初始值");
```
这样所有的线程使用ThreadLocal的get都会是相同的一个初始化值。

**2.多个线程有可以相同的值**
可以使用InheritableThreadLocal，父线程中设置的值，子线程中可以访问到。这个用法和上面差不多就不举例子了，有兴趣的可以自己研究下~


### 坑点：
1.上面看ThreadLocalMap的时候，咱们知道key是弱引用，gc的时候key会被回收，但是value和ThreadLocalMap的引用不会被回收。如果这种情况的Thread很多，而且一直没有执行完，就可能会出现内存泄漏。  
2.在使用线程池的时候，当使用ThreadLocal调用set方法后，然后没有调用remove的话，因为线程池的线程是复用的，如果同一个线程再去调用get方法可能拿到的值并不是当时set进去的，导致程序数据异常之类的。尽量用完值后就remove掉。

### 总结：
1.ThreadLocal在同一个线程传值，或者只跟线程相关的场景使用
2.初始化ThreadLocal的值可以用ThreadLocal.withInitial，或重写initialValue()
3.父子线程共享相同的值使用InheritableThreadLocal
4.**不管是正常使用还是线程池使用ThreadLocal都一定要使用完就remove，否则会内存泄漏或者数据出错**
