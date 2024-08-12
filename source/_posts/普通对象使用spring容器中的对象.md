### 引语：
&nbsp;&nbsp;&nbsp;&nbsp;工作中有时候需要在普通的对象中去调用spring管理的对象，但是在普通的java对象直接使用@Autowired或者@Resource的时候会发现被注入的对象是null，会报空指针。我们可以简单的理解为spring是一个公司，它管理的对象就是它的员工，而普通的java对象是其他公司的员工，如果其他公司要找spring公司的员工一起共事没有经过spring公司的同意肯定是不行的。

### 解决方式：
方法一：如果这个普通对象可以被spring管理的话，最好是直接交给spring管理，这样spring管理的bean中注入其他的bean是没有问题的。

方法二：当我们的普通对象没有办法交给spring管理的时候，我们可以创建一个公共的springBeanUtil专门为普通对象提供spring的员工（有点像spring公司的外包部门，把对象外包给其他公司使用，哈哈）。
```java
@Service
public class SpringBeanUtil implements ApplicationContextAware {

    public static ApplicationContext applicationContext;

    @Override
    public void setApplicationContext(ApplicationContext context) throws BeansException {
        applicationContext = context;
    }

    // 这里使用的是根据class类型来获取bean 当然你可以根据名称或者其他之类的方法 主要是有applicationContext你想怎么弄都可以
    public static Object getBeanByClass(Class clazz) {
        return applicationContext.getBean(clazz);
    }
}
```
这个util呢，其实就是实现了ApplicationContextAware接口,有小伙伴要问了这个接口是干嘛的？这里给出链接地址，[ApplicationContextAware参考资料](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/context/ApplicationContextAware.html)。然后我也将文档中的解释给摘录过来了
```java
public interface ApplicationContextAware extends Aware
Interface to be implemented by any object that wishes to be notified of the ApplicationContext that it runs in.
Implementing this interface makes sense for example when an object requires access to a set of collaborating beans. Note that configuration via bean references is preferable to implementing this interface just for bean lookup purposes.
This interface can also be implemented if an object needs access to file resources, i.e. wants to call getResource, wants to publish an application event, or requires access to the MessageSource. However, it is preferable to implement the more specific ResourceLoaderAware, ApplicationEventPublisherAware or MessageSourceAware interface in such a specific scenario.
Note that file resource dependencies can also be exposed as bean properties of type Resource, populated via Strings with automatic type conversion by the bean factory. This removes the need for implementing any callback interface just for the purpose of accessing a specific file resource.
ApplicationObjectSupport is a convenience base class for application objects, implementing this interface.
```
大概意思就是说只要实现了ApplicationContextAware接口的类，期望被告知当前运行的applicationContext是什么。然后又说了如果是想要获取资源最好是用ResourceLoaderAware, ApplicationEventPublisherAware or MessageSourceAware 这几个接口，最后还来了一句我们知道你们要使用这些接口，所以我们帮你弄了一个实现了这些接口的抽象类ApplicationObjectSupport（在spring-context的jar包中）。这里说得很清楚要使用bean的话，实现ApplicationContextAware，因为我们这里不需要使用静态资源之类的所以我们就不用spring为我们提供的ApplicationObjectSupport了，有兴趣的可以自己研究下。

我们这里简单的看一下ApplicationContextAware类里面都有啥？
```java
void setApplicationContext(ApplicationContext applicationContext) throws BeansException;
```
发现就一个方法，spring初始化的时候会将当前的applicationContext传给ApplicationContextAware的setApplicationContext方法，所以只要实现类将这个applicationContext拿到了，就可以通过class类型或者class的名称来获取到spring中的bean了。原理其实很简单吧。使用的时候我们可以
```java
Test test = (Test) SpringBeanUtil.getBeanByClass(Test.class);
```
来调用spring中的bean。
