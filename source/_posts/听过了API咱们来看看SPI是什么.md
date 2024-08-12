### 引语
 平时API倒是听得很多?SPI又是啥.别急我们来先看看面向接口编程的调用关系，来了解一下，API和SPI的相似和不同之处。

### SPI理解
先来一段官话的介绍:SPI 全称为 (Service Provider Interface) ，是JDK内置的一种服务提供发现机制.(听了一脸懵逼)好的，我们结合图片来理解一下。
![](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/%E5%90%AC%E8%BF%87%E4%BA%86API%E5%92%B1%E4%BB%AC%E7%9C%8B%E7%9C%8BSPI%E6%98%AF%E5%95%A5/%E8%B0%83%E7%94%A8%E5%85%B3%E7%B3%BB%E5%9B%BE.png)
&nbsp;&nbsp;&nbsp;&nbsp;简单的来说分为调用方，接口，服务方.接口就是协议，契约，可以调用方定义，也可以由服务方定义，也就是接口是可以位于调用方的包或者服务方的包.
1.接口的定义和实现都在服务方的时候，仅暴露出接口给调用方使用的时候，我们称为API;
2.接口的定义在调用方的时候(实现在服务方)，我们给它也取个名字--SPI。
应该还比较好理解吧？

### SPI的使用场景
SPI在框架中其实有很多广泛的应用，这里列举几个例子：
1.Mysql驱动的选择driverManager根据配置来确定要使用的驱动;
2.dubbo框架中的扩展机制（[dubbo官网链接](http://dubbo.apache.org/zh-cn/blog/introduction-to-dubbo-spi.html)）

### 使用实例
看完上面的简介和SPI在框架中的应用，想必对SPI在读者的大脑中已经产生了一个雏形，talk is cheap!show me the code.说了这么多,我们具体写一个简单的例子来看看效果,验证一下SPI.

1.首先定义一个接口,忍者服务接口
```java
public interface NinjaService {
    void performTask();
}
```
2.接下来写两个实现类,ForbearanceServiceImpl(上忍),ShinobuServiceImpl(下忍)
```java
public class ForbearanceServiceImpl implements NinjaService {

    @Override
    public void performTask() {
        System.out.println("上忍在执行A级任务");
    }
}
```
```java
public class ShinobuServiceImpl implements NinjaService {

    @Override
    public void performTask() {
        System.out.println("下忍在执行D级任务");
    }
}
```
3.接下来我们在main/resources/下创建META-INF/services目录,并且在services目录下创建一个com.scott.java.task.spi.NinjaService(忍者服务类的全限定名)的文件.

4.创建一个Client场景类来调用看看结果
![](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/%E5%90%AC%E8%BF%87%E4%BA%86API%E5%92%B1%E4%BB%AC%E7%9C%8B%E7%9C%8BSPI%E6%98%AF%E5%95%A5/client.png)
很完美的调用了两个实现类的performTask()方法.

5.最后贴一下目录结构
![](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/%E5%90%AC%E8%BF%87%E4%BA%86API%E5%92%B1%E4%BB%AC%E7%9C%8B%E7%9C%8BSPI%E6%98%AF%E5%95%A5/dir.png)
附上一波代码例子的地址,在spi里面,[git链接](https://github.com/wxwwt/java_practice);

### SPI源码简单分析
1.先看下核心类ServiceLoader的定义和属性
```java
// 继承了Iterable类  遍历的时候使用
public final class ServiceLoader<S> implements Iterable<S>
{
  // 这就是为啥需要在META-INF/services/目录下创建服务类的文件
  private static final String PREFIX = "META-INF/services/";

  // 被加载的服务
  private final Class<S> service;

  // 类加载器
  private final ClassLoader loader;

  // 访问控制类
  private final AccessControlContext acc;

  // 实现类的缓存 根据初始化的顺序 也就是在/services/文件中的定义顺序来定义的加载顺序
  private LinkedHashMap<String,S> providers = new LinkedHashMap<>();

  // 懒加载iterator
  private LazyIterator lookupIterator;

```
2.然后从client开始,然后依次debug进去
```java
ServiceLoader<NinjaService> ninjaServices = ServiceLoader.load(NinjaService.class);
```

```java
public static <S> ServiceLoader<S> load(Class<S> service) {
      // 获取当前的类加载器 也就是AppClassLoader
      ClassLoader cl = Thread.currentThread().getContextClassLoader();
      return ServiceLoader.load(service, cl);
  }
```

```java
public static <S> ServiceLoader<S> load(Class<S> service, ClassLoader loader) {
       return new ServiceLoader<>(service, loader);
   }
```
后面的就省略了,因为这里仅仅就是根据NinjaService初始化的事项,没有什么很难理解的点.

3.我们在看看具体的调用过程,这里使用的是client对应的class文件,因为增加for(foreach)在java中是个语法糖,实际上编译后是这样的内容
```java
public static void main(String[] args) {
       ServiceLoader<NinjaService> ninjaServices = ServiceLoader.load(NinjaService.class);
      // 这里一下其实就是增加for解糖后的代码 有兴趣可以去了解下java的语法糖
       Iterator var2 = ninjaServices.iterator();

       while(var2.hasNext()) {
           NinjaService item = (NinjaService)var2.next();
           item.performTask();
       }
   }
```
4.随着断点继续走,我们进入到var2.hasNext()的方法
```java
public boolean hasNext() {
            // knownProviders还没有加载过provider 走下面的分支
             if (knownProviders.hasNext())
                 return true;
             return lookupIterator.hasNext();
         }
```
这里lookupIterator上面ServiceLoader的属性介绍过,它其实是ServiceLoader中的一个Iterator的内部类。然后调用了内部类Iterator的hasNext()方法。
```java
public boolean hasNext() {
          //   acc = (System.getSecurityManager() != null) ? AccessController.getContext() : null;
          //   ServiceLoader初始化没有设置过securityManager,所以acc是null,进入hasNextService()
           if (acc == null) {
               return hasNextService();
           } else {
               PrivilegedAction<Boolean> action = new PrivilegedAction<Boolean>() {
                   public Boolean run() { return hasNextService(); }
               };
               return AccessController.doPrivileged(action, acc);
           }
       }
```
5.hasNextService()分析
```java
private boolean hasNextService() {
           if (nextName != null) {
               return true;
           }
           if (configs == null) {
               try {
               // 这里加载了META-INF/services下的文件 也就是含有两个实现类全限定名的文件
                   String fullName = PREFIX + service.getName();
                   if (loader == null)
                       configs = ClassLoader.getSystemResources(fullName);
                   else
                   // 因为loader是不为null 的AppClassLoader
                       configs = loader.getResources(fullName);
               } catch (IOException x) {
                   fail(service, "Error locating configuration files", x);
               }
           }
           while ((pending == null) || !pending.hasNext()) {
               if (!configs.hasMoreElements()) {
                   return false;
               }
               // 这里是将上面加载的文件中的两个实现类的文件
               pending = parse(service, configs.nextElement());
           }
           nextName = pending.next();
           return true;
       }
```
6.继续看到parse方法,这里最后返回的是含有两个全限定类名的Iterator<String>,其实就是把services/下的文件内容给加载出来
```java
private Iterator<String> parse(Class<?> service, URL u) throws ServiceConfigurationError {
   InputStream in = null;
   BufferedReader r = null;
   ArrayList<String> names = new ArrayList<>();
   try {
       in = u.openStream();
       r = new BufferedReader(new InputStreamReader(in, "utf-8"));
       int lc = 1;
       while ((lc = parseLine(service, u, r, lc, names)) >= 0);
   } catch (IOException x) {
       fail(service, "Error reading configuration file", x);
   } finally {
       try {
           if (r != null) r.close();
           if (in != null) in.close();
       } catch (IOException y) {
           fail(service, "Error closing configuration file", y);
       }
   }
   return names.iterator();
}
```
附带的说一下parseLine(service, u, r, lc, names),检查类名是否符合规范,符合的话添加到Iterator中,到这里var2
.hasNext()执行完毕,结果是加载了services下的文件内容
```java
private int parseLine(Class<?> service, URL u, BufferedReader r, int lc,
                         List<String> names)
       throws IOException, ServiceConfigurationError
   {
       String ln = r.readLine();
       if (ln == null) {
           return -1;
       }
       int ci = ln.indexOf('#');
       if (ci >= 0) ln = ln.substring(0, ci);
       ln = ln.trim();
       int n = ln.length();
       if (n != 0) {
           if ((ln.indexOf(' ') >= 0) || (ln.indexOf('\t') >= 0))
               fail(service, u, lc, "Illegal configuration-file syntax");
           int cp = ln.codePointAt(0);
           if (!Character.isJavaIdentifierStart(cp))
               fail(service, u, lc, "Illegal provider-class name: " + ln);
           for (int i = Character.charCount(cp); i < n; i += Character.charCount(cp)) {
               cp = ln.codePointAt(i);
               if (!Character.isJavaIdentifierPart(cp) && (cp != '.'))
                   fail(service, u, lc, "Illegal provider-class name: " + ln);
           }
           if (!providers.containsKey(ln) && !names.contains(ln))
               names.add(ln);
       }
       return lc + 1;
   }
```
```java
private boolean hasNextService() {
          if (nextName != null) {
              return true;
          }
          if (configs == null) {
              try {
                  String fullName = PREFIX + service.getName();
                  if (loader == null)
                      configs = ClassLoader.getSystemResources(fullName);
                  else
                      configs = loader.getResources(fullName);
              } catch (IOException x) {
                  fail(service, "Error locating configuration files", x);
              }
          }
          while ((pending == null) || !pending.hasNext()) {
              if (!configs.hasMoreElements()) {
                  return false;
              }
              pending = parse(service, configs.nextElement());
          }
          // 这里将下一个实现类的名字赋值给了LazyIterator的属性nextName
          nextName = pending.next();
          return true;
      }
```
7.接下来执行的是 NinjaService item = (NinjaService)var2.next()的next(方法),然后继续debug进去,这里我省略了一些方法的调用,只展示出有用的方法这个是ServiceLoader的内部类LazyIterator的nextService()方法.
```java
private S nextService() {
           if (!hasNextService())
               throw new NoSuchElementException();
           String cn = nextName;
           nextName = null;
           Class<?> c = null;
           try {
           // 这里nextName在上面已经赋值过了 所以反射创建实例
               c = Class.forName(cn, false, loader);
           } catch (ClassNotFoundException x) {
               fail(service,
                    "Provider " + cn + " not found");
           }
           // 类型判断
           if (!service.isAssignableFrom(c)) {
               fail(service,
                    "Provider " + cn  + " not a subtype");
           }
           try {
           // 强转类型
               S p = service.cast(c.newInstance());
          //  将类添加到ServiceLoader的providers属性中 然后返回
               providers.put(cn, p);
               return p;
           } catch (Throwable x) {
               fail(service,
                    "Provider " + cn + " could not be instantiated",
                    x);
           }
           throw new Error();          // This cannot happen
       }
```
8.到这里子类的实现类返回,分析就结束了.

### 总结:
1.了解了什么是SPI；
2.SPI和API的简单区别和联系；
3.学习了怎么使用SPI来扩展服务；
4.分析了ServiceLoader的源码加载过程，这里扯一句，简单的就是META-INF/services定义好要实现的接口(文件名)和实现类(文件内容),
ServiceLoader加载的时候没有实例化实现类,而是在Iterator遍历的时候去用反射创建了实例.

**觉得写得还行的可以点个赞,关注一波,后面会继续写更好的文章~ XD**


### 参考资料:
1.http://cr.openjdk.java.net/~mr/jigsaw/spec/api/java/util/ServiceLoader.html
2.https://www.cnblogs.com/happyframework/archive/2013/09/17/3325560.html
3.http://dubbo.apache.org/zh-cn/blog/introduction-to-dubbo-spi.html
4.https://www.cnblogs.com/googlemeoften/p/5715262.html
5.https://zhuanlan.zhihu.com/p/28909673
