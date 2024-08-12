### 遍历集合类的时候删除元素
### 导语：

最近写了一个bug就是在遍历list的时候删除了里面一个元素，其实之前看过阿里的java开发规范，知道在遍历的时候删除元素会产生问题，但是写的快的时候还是会没注意到，那正好研究下里面的机制看。我们看看阿里规范怎么写的：
![](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/remove%E9%9B%86%E5%90%88%E7%B1%BB%E5%85%83%E7%B4%A0/%E9%98%BF%E9%87%8C%E8%A7%84%E8%8C%83.png)


首先提出一个概念：fail-fast
摘自百度百科：
fail-fast 机制是java集合(Collection)中的一种错误机制。当多个线程对同一个集合的内容进行操作时，就可能会产生fail-fast事件。
例如：当某一个线程A通过iterator去遍历某集合的过程中，若该集合的内容被其他线程所改变了；那么线程A访问集合时，就会抛出ConcurrentModificationException异常，产生fail-fast事件。
简单来说是java为了防止出现并发异常的一个机制，但是其实在单线程下也可以产生。

### 实例分析：
接下来，我会通过6个例子来探究下，遍历list时删除元素会发生什么，和它的源码探究。
前提先造了一个list：
```java
private List<String> list = new ArrayList<String>() {{
       add("元素1");
       add("元素2");
       add("元素3");
   }};
```
#### 1.普通的for循环
```java
public void test1() {
       for (int i = 0; i < list.size() - 1; i++) {
           if ("元素3".equals(list.get(i))) {
               System.out.println("找到元素3了");
           }

           if ("元素2".equals(list.get(i))) {
               list.remove(i);
           }
       }
   }
   //  这里不会输出找到元素3 因为遍历到元素2的时候删除了元素2 list的size变小了
   //  所以就产生问题了
```

#### 2.for循环另一种情况
```java
public void test2() {
       for (int i = 0; i < list.size() - 1; i++) {
           if ("元素2".equals(list.get(i))) {
               list.remove(i);
           }

           if ("元素3".equals(list.get(i))) {
               System.out.println("找到元素3了");
           }
       }
   }
   // 这里会输出元素3 但是其实是在遍历到元素2的时候输出的
   // 遍历到元素2 然后删除 到了判断元素3的条件的时候i是比原来小了1
   // 所以又阴差阳错的输出了正确结果
```

#### 3.增强for循环
```java
public void test3() {
       for (String item : list) {
           if ("元素2".equals(item)) {
               list.remove(item);
           }

           if ("元素3".equals(item)) {
               System.out.println("找到元素3了");
           }
       }
   }
   // 这里和上面的结果有点不一样 但是还是没有输出元素3的打印语句
   // 这里反编译下java文件就可以知道是为啥啦

   public void test3() {
         Iterator var1 = this.list.iterator();

         while(var1.hasNext()) {
           // 为了显示区别这里
             String var2 = (String)var1.next();
             if ("元素2".equals(var2)) {
                 this.list.remove(var2);
             }

             if ("元素3".equals(var2)) {
                 System.out.println("找到元素3了");
             }
         }

     }
```
反编译的文件可以知道这里增强for反编译是用了迭代器，来判断是否还有元素，然后remove用的是list的方法。
我们看下ArrayList中的hasNext(),next()是怎么写的。下面是ArrayList中的内部类Itr实现了Iterator接口，
```java
private class Itr implements Iterator<E> {
        int cursor;       // index of next element to return
        int lastRet = -1; // index of last element returned; -1 if no such
        int expectedModCount = modCount;

        Itr() {}

        public boolean hasNext() {
            return cursor != size;
        }

        @SuppressWarnings("unchecked")
        public E next() {
            checkForComodification();
            int i = cursor;
            if (i >= size)
                throw new NoSuchElementException();
            Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length)
                throw new ConcurrentModificationException();
            cursor = i + 1;
            return (E) elementData[lastRet = i];
        }

        public void remove() {
            if (lastRet < 0)
                throw new IllegalStateException();
            checkForComodification();

            try {
                ArrayList.this.remove(lastRet);
                cursor = lastRet;
                lastRet = -1;
                expectedModCount = modCount;
            } catch (IndexOutOfBoundsException ex) {
                throw new ConcurrentModificationException();
            }
        }

        @Override
        @SuppressWarnings("unchecked")
        public void forEachRemaining(Consumer<? super E> consumer) {
            Objects.requireNonNull(consumer);
            final int size = ArrayList.this.size;
            int i = cursor;
            if (i >= size) {
                return;
            }
            final Object[] elementData = ArrayList.this.elementData;
            if (i >= elementData.length) {
                throw new ConcurrentModificationException();
            }
            while (i != size && modCount == expectedModCount) {
                consumer.accept((E) elementData[i++]);
            }
            // update once at end of iteration to reduce heap write traffic
            cursor = i;
            lastRet = i - 1;
            checkForComodification();
        }

        final void checkForComodification() {
            if (modCount != expectedModCount)
                throw new ConcurrentModificationException();
        }
    }
```
这里cursor在next()执行时，cursor = i + 1，所以当运行到"元素2".equal(item)这里，元素2被移除，当遍历当元素3的时候size = 2.cursor = i + 1 = 1 + 1 也是2.
hasNext()中cursor == size就直接退出了。



#### 4.foreach
```java
public void test4() {
        list.forEach(
                item -> {
                    if ("元素2".equals(item)) {
                        list.remove(item);
                    }

                    if ("元素3".equals(item)) {
                        System.out.println("找到元素3了");
                    }
                }
        );
    }
    // 这里抛出了我们期待已经的fail-fast java.util.ConcurrentModificationException
```
点进去看下ArrayList的源码
```java
@Override
   public void forEach(Consumer<? super E> action) {
       Objects.requireNonNull(action);
       final int expectedModCount = modCount;
       @SuppressWarnings("unchecked")
       final E[] elementData = (E[]) this.elementData;
       final int size = this.size;
       for (int i=0; modCount == expectedModCount && i < size; i++) {
           action.accept(elementData[i]);
       }
       if (modCount != expectedModCount) {
           throw new ConcurrentModificationException();
       }
   }
```
根据报错信息我们可以知道是
if (modCount != expectedModCount) {
    throw new ConcurrentModificationException();
}
抛出的异常，那么这个modCount元素是从哪里来的？
我们找到AbstractList,这个是ArrayList的父类。
看看源码里面是怎么说的
```java
/**
    * The number of times this list has been <i>structurally modified</i>.
    * Structural modifications are those that change the size of the
    * list, or otherwise perturb it in such a fashion that iterations in
    * progress may yield incorrect results.
    *
    * <p>This field is used by the iterator and list iterator implementation
    * returned by the {@code iterator} and {@code listIterator} methods.
    * If the value of this field changes unexpectedly, the iterator (or list
    * iterator) will throw a {@code ConcurrentModificationException} in
    * response to the {@code next}, {@code remove}, {@code previous},
    * {@code set} or {@code add} operations.  This provides
    * <i>fail-fast</i> behavior, rather than non-deterministic behavior in
    * the face of concurrent modification during iteration.
    *
    * <p><b>Use of this field by subclasses is optional.</b> If a subclass
    * wishes to provide fail-fast iterators (and list iterators), then it
    * merely has to increment this field in its {@code add(int, E)} and
    * {@code remove(int)} methods (and any other methods that it overrides
    * that result in structural modifications to the list).  A single call to
    * {@code add(int, E)} or {@code remove(int)} must add no more than
    * one to this field, or the iterators (and list iterators) will throw
    * bogus {@code ConcurrentModificationExceptions}.  If an implementation
    * does not wish to provide fail-fast iterators, this field may be
    * ignored.
    */
protected transient int modCount = 0;
```
英文好的同学可以自己看下，英文不好的同学可以默默打开谷歌翻译或者有道词典了。

剩下的同学可以听一下我的理解，其实只看第一段基本上就知道它是干嘛的了。意思是这个字段是记录了list的结构性修改次数（我们可以理解为add，remove这些会改变list大小的操作）。如果是其他方式导致的就回抛出ConcurrentModificationException异常。
所以我们再去看看子类的ArrayList的add，remove方法。
```java
// 这个是add里面的子方法
private void ensureExplicitCapacity(int minCapacity) {
      modCount++;

      // overflow-conscious code
      if (minCapacity - elementData.length > 0)
          grow(minCapacity);
  }

// 这个是remove(int index)
  public E remove(int index) {
        rangeCheck(index);

        modCount++;
        E oldValue = elementData(index);

        int numMoved = size - index - 1;
        if (numMoved > 0)
            System.arraycopy(elementData, index+1, elementData, index,
                             numMoved);
        elementData[--size] = null; // clear to let GC do its work

        return oldValue;
    }
// 这是remove(Object o)
private void fastRemove(int index) {
       modCount++;
       int numMoved = size - index - 1;
       if (numMoved > 0)
           System.arraycopy(elementData, index+1, elementData, index,
                            numMoved);
       elementData[--size] = null; // clear to let GC do its work
   }    
```
看完这几个方法，是不是都看到了modCount++这个操作？是的，添加和删除的时候都会对这个modCount加一。好了，我们在回头看看为啥test4()会抛出异常。
首先我们知道list里面add了三个元素所以modCount在添加完元素的时候是3，然后它开始遍历，当发现元素2的时候，去移除了元素2，此时modCount变为4.
我们在会到ArrayList的
```java
@Override
   public void forEach(Consumer<? super E> action) {
       Objects.requireNonNull(action);
       final int expectedModCount = modCount;
       @SuppressWarnings("unchecked")
       final E[] elementData = (E[]) this.elementData;
       final int size = this.size;
       for (int i=0; modCount == expectedModCount && i < size; i++) {
           action.accept(elementData[i]);
       }
       if (modCount != expectedModCount) {
           throw new ConcurrentModificationException();
       }
   }
```
可以看到一开始把modCoun赋值给了expectedModCount，然后for循环里面和最后的if条件均对这个modCount有跑断，if里面如果发现modCount和expectedModCount不相等了，就抛出异常。当遍历到元素2，action.accept(elementData[i]);这一行执行完，再去遍历元素3的时候因为modCount == expectedModCount不相等了，所以循环推出，因此不会打印出找到元素3了，并且执行到if条件直接抛出异常。

#### 5.迭代器
我们在来看看正确的使用方式，使用迭代器
```java
public void test5() {
       Iterator<String> iterator = list.iterator();
       while (iterator.hasNext()) {
           String temp = iterator.next();
           if ("元素2".equals(temp)) {
               iterator.remove();
           }

           if ("元素3".equals(temp)) {
               System.out.println("找到元素3了");
           }
       }
   }
   // 这里打印出了 找到元素3了
```
上面的test3我们已经看到这个迭代器的代码了，那为啥它没问题呢？其实原理非常的沙雕，不信你看！  
![](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/remove%E9%9B%86%E5%90%88%E7%B1%BB%E5%85%83%E7%B4%A0/%E5%A5%87%E6%80%AA%E7%9A%84%E7%9F%A5%E8%AF%86.jfif)  
这是迭代器的remove方法
```java
public void remove() {
    if (lastRet < 0)
        throw new IllegalStateException();
    checkForComodification();

    try {
        ArrayList.this.remove(lastRet);
        cursor = lastRet;
        lastRet = -1;
        expectedModCount = modCount;
        // 看到没有直接把modeCount重新赋值给了expectedModCount 所以它们一直会相等啊
    } catch (IndexOutOfBoundsException ex) {
        throw new ConcurrentModificationException();
    }
}
```
所以它是可以找到元素3的，因为直接忽略了remove对modCount的影响。

#### 6.removeIf
jdk8还出了一种新写法，封装了在遍历元素时删除元素，removeIf(),如下：
```java
 list.removeIf(item -> "元素2".equals(item)
```
点进去可以看到代码和上一个差不多只是封装了一层罢了。
```java
default boolean removeIf(Predicate<? super E> filter) {
        Objects.requireNonNull(filter);
        boolean removed = false;
        final Iterator<E> each = iterator();
        while (each.hasNext()) {
            if (filter.test(each.next())) {
                each.remove();
                removed = true;
            }
        }
        return removed;
    }
```

总结：
1.我们探究了好几种遍历时删除元素的方式，也知道了fail-fast的基本概念。
2.学会了以后遍历的时候删除元素要使用迭代器哦，而且发现即使是单线程依然可以抛出ConcurrentModificationException
3.如果是多线程操作list的话建议使用CopyOnWriteArrayList，或者对迭代器加锁也行，看个人习惯吧~
