### 引语:
阿里巴巴java开发规范说到使用工具类Arrays.asList()方法把数组转换成集合时,不能使用其修改集合相关的方法,它的add/remove/clear方法会抛出UnsupportedOperationException(),我们来看一下为什么会出现这种情况.

### 问题分析:
我们做个测试
```java
public static void main(String[] args) {
       List<String> list = Arrays.asList("a", "b", "c");
       // list.clear();
       // list.remove("a");
       // list.add("g");
   }
```
被注释的三行可以分别解开注释,运行后确实出现了规约中所说的异常.我们来看下Arrays.asList()做了什么操作.
```java
public static <T> List<T> asList(T... a) {
       return new ArrayList<>(a);
   }
```
看上去是个很正常的方法,然而实际上你点进到ArrayList发现,其实ArrayList并不是我们平时用的ArrayList.
```java
private static class ArrayList<E> extends AbstractList<E>
       implements RandomAccess, java.io.Serializable
   {
       private static final long serialVersionUID = -2764017481108945198L;
       private final E[] a;

       ArrayList(E[] array) {
           a = Objects.requireNonNull(array);
       }

       @Override
       public int size() {
           return a.length;
       }

       @Override
       public Object[] toArray() {
           return a.clone();
       }

       @Override
       @SuppressWarnings("unchecked")
       public <T> T[] toArray(T[] a) {
           int size = size();
           if (a.length < size)
               return Arrays.copyOf(this.a, size,
                                    (Class<? extends T[]>) a.getClass());
           System.arraycopy(this.a, 0, a, 0, size);
           if (a.length > size)
               a[size] = null;
           return a;
       }
       // 后面省略了
```
而是Arrays里面的一个内部类.而且这个内部类没有add,clear,remove方法,所以抛出的异常其实来自于AbstractList.
```java
   public void add(int index, E element) {
       throw new UnsupportedOperationException();
   }

   public E remove(int index) {
      throw new UnsupportedOperationException();
  }
```
点进去就会发现抛出异常的地方,clear底层也会调用到remove所以也会抛出异常.

### 总结:
1.Arrays.asList()不要乱用,底层其实还是数组;
2.如果使用了Arrays.asList()的话,最好不要使用其集合的操作方法;
3. List<String> list = new ArrayList<>(Arrays.asList("a", "b", "c"))可以在外面这样包一层真正的ArrayList(数组转集合有很多方式,可以参考[链接](https://stackoverflow.com/questions/157944/create-arraylist-from-array)).
