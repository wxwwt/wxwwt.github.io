

# 概述：

之前有了解过Java的ShutDown Hook机制，但是因为没有使用场景也没有深入学习，最近刚好又看到ShutDown Hook的一些东西，想着学习总结一下，做下学习记录。Java的Shutdown Hook是一种机制，允许开发者在Java虚拟机（JVM）即将关闭之前执行一些清理或终止操作。Shutdown Hook提供了一个钩子，允许开发者在JVM关闭时捕获到关闭事件并执行相应的逻辑。以下是一些使用场景：

1. 资源释放和清理：当应用程序结束或JVM关闭时，可以使用Shutdown Hook来释放和清理打开的文件、网络连接、数据库连接等资源。这确保资源在程序终止之前得到适当的关闭，避免资源泄露和数据丢失。
2. 日志记录和统计：Shutdown Hook可以用于记录应用程序的关键统计信息或生成最终的日志报告。通过在JVM关闭前执行这些操作，可以捕获应用程序在运行期间的关键数据，并生成相应的日志记录。
3. 缓存刷新：如果应用程序使用了缓存机制，可以在JVM关闭前使用Shutdown Hook来刷新缓存，将缓存中的数据写回到持久化存储或其他目标中，确保数据的持久化和一致性。
4. 任务终止和状态保存：在某些情况下，可能需要在应用程序终止时保存任务的当前状态，以便在下次启动时恢复。通过在Shutdown Hook中执行任务的状态保存操作，可以将任务的状态保存到持久化存储中，并在下次启动时进行恢复。
5. 线程管理：Shutdown Hook还可以用于管理和终止应用程序中的线程。在JVM关闭前，可以使用Shutdown Hook发送终止信号给正在运行的线程，以确保它们在终止之前完成当前任务并进行清理操作。

上面说的这些使用场景，我都没用到过，大家可以先了解一下对ShutDownHook有一个简单的认识。



# 分析：

下面我们从源码上去看一下，ShutDown Hook的方法和原理。

跟它相关的主要有两个类ApplicationShutdownHooks和Runtime

```
class ApplicationShutdownHooks {
    /* The set of registered hooks */
    private static IdentityHashMap<Thread, Thread> hooks;
    static {
        try {
            Shutdown.add(1 /* shutdown hook invocation order */,
                false /* not registered if shutdown in progress */,
                new Runnable() {
                    public void run() {
                        runHooks();
                    }
                }
            );
            hooks = new IdentityHashMap<>();
        } catch (IllegalStateException e) {
            // application shutdown hooks cannot be added if
            // shutdown is in progress.
            hooks = null;
        }
    }


    private ApplicationShutdownHooks() {}

    /* Add a new shutdown hook.  Checks the shutdown state and the hook itself,
     * but does not do any security checks.
     */
    static synchronized void add(Thread hook) {
        if(hooks == null)
            throw new IllegalStateException("Shutdown in progress");

        if (hook.isAlive())
            throw new IllegalArgumentException("Hook already running");

        if (hooks.containsKey(hook))
            throw new IllegalArgumentException("Hook previously registered");

        hooks.put(hook, hook);
    }

    /* Remove a previously-registered hook.  Like the add method, this method
     * does not do any security checks.
     */
    static synchronized boolean remove(Thread hook) {
        if(hooks == null)
            throw new IllegalStateException("Shutdown in progress");

        if (hook == null)
            throw new NullPointerException();

        return hooks.remove(hook) != null;
    }

    /* Iterates over all application hooks creating a new thread for each
     * to run in. Hooks are run concurrently and this method waits for
     * them to finish.
     */
    static void runHooks() {
        Collection<Thread> threads;
        synchronized(ApplicationShutdownHooks.class) {
            threads = hooks.keySet();
            hooks = null;
        }

        for (Thread hook : threads) {
            hook.start();
        }
        for (Thread hook : threads) {
            try {
                hook.join();
            } catch (InterruptedException x) { }
        }
    }
}
```



```
 -- Runtime.class里面的方法
 public void addShutdownHook(Thread hook) {
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            sm.checkPermission(new RuntimePermission("shutdownHooks"));
        }
        ApplicationShutdownHooks.add(hook);
    }

    public boolean removeShutdownHook(Thread hook) {
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            sm.checkPermission(new RuntimePermission("shutdownHooks"));
        }
        return ApplicationShutdownHooks.remove(hook);
    }


    public void halt(int status) {
        SecurityManager sm = System.getSecurityManager();
        if (sm != null) {
            sm.checkExit(status);
        }
        Shutdown.halt(status);
    }
```

结合这两个类的源码，我们可以看到添加Hook其实是往ApplicationShutdownHooks的静态Map里面放入新的线程，但是这些线程只是创建后被保存了起来，只有当程序退出时，runHooks被执行，每一个带有Hook任务的线程才的start()方法才被执行，也因为Hook之间是相互独立的线程，所以它们之间执行是没有顺序的，而且因为主线程调用了每个Hook的线程的join方法，所以主线程会等待Hook全部执行完毕在退出。



##  无法被添加的情况：

```
    if(hooks == null)
            throw new IllegalStateException("Shutdown in progress");
   if (hook.isAlive())
        throw new IllegalArgumentException("Hook already running");

    if (hooks.containsKey(hook))
        throw new IllegalArgumentException("Hook previously registered");
```

1.ApplicationShutdownHooks已经在调用Hook时，hooks会置为null，不能在添加hook

2.Hook的Thread不能是已经在运行状态的线程

3.因为储存的Hook是根据线程是否相同来判断的，所以同样的Hook无法被添加



## 不适用的情况：

1.因为ShutDown Hook只能处理正常退出的情况，kill -9这种是无法处理的

2Shutdown.halt和kill -9一样都是强制退出，不会给Hook执行的机会



# 使用：

下面放了一些简单的测试ShutDown的小例子，[github地址](https://github.com/wxwwt/java-practice/tree/master/src/main/java/com/scott/java/task/shutdown/hook)：

```
 @Test
    public void test1() {
        // 测试正常退出的情况
        Runtime.getRuntime().addShutdownHook(
                new Thread(
                        () -> {
                            System.out.println("hook1 执行了");
                        })
        );
    }
    
    输出：hook1 执行了
```

```
 @Test
    public void test2() {
        // 测试Hook执行顺序是否真的无序
        Runtime.getRuntime().addShutdownHook(
                new Thread(
                        () -> {
                            System.out.println("hook1 执行了");
                        })
        );

        Runtime.getRuntime().addShutdownHook(
                new Thread(
                        () -> {
                            System.out.println("hook2 执行了");
                        })
        );
    }
    
    输出：输出结果hook1和hook2会随机打印，没有固定顺序
```

```
 @Test
    public void test3() {
        // 测试kill -9 会执行Hook吗
        Runtime.getRuntime().addShutdownHook(
                new Thread(
                        () -> {
                            System.out.println("hook1 执行了");
                        })
        );

        while(true) {

        }

    }
    
    输出：
```

```
    @Test
    public void test4() {
        // 测试oom时 会执行Hook吗
        Runtime.getRuntime().addShutdownHook(
                new Thread(
                        () -> {
                            System.out.println("hook1 执行了");
                        })
        );

   
        List<Object> list = Lists.newArrayList();
        while(true) {
           list.add(new ShutDownHookTest());
        }
    }
    
    输出：
     	 java.lang.OutOfMemoryError: GC overhead limit exceeded

         at com.scott.java.task.shutdown.hook.ShutDownHookTest.test4(ShutDownHookTest.java:74)
         。。。省略不重要的日志

         hook1 执行了
```

```
 @Test
    public void test5() {
        // 测试移除Hook后，会执行Hook吗
        Thread thread = new Thread(() -> {
            System.out.println("hook1 执行了");
        });

        Runtime.getRuntime().addShutdownHook(thread);
        Runtime.getRuntime().removeShutdownHook(thread);
    }
    
    输出：
```

```
    @Test
    public void test6() {
        // 测试执行halt方法后，会执行Hook吗
        Thread thread = new Thread(() -> {
            System.out.println("hook1 执行了");
        });

        Runtime.getRuntime().addShutdownHook(thread);
        Runtime.getRuntime().halt(111);
    }
    输出：
```

```
 @Test
    public void test7() {
        // 测试已经执行Hook时，还能添加新的hook吗
        Thread thread = new Thread(() -> {
            System.out.println("hook1 执行了");
            Run
        });

        Runtime.getRuntime().addShutdownHook(thread);
        Runtime.getRuntime().halt(111);
    }
    输出：
    hook1 执行了
	Exception in thread "Thread-0" java.lang.IllegalStateException: Shutdown in progress
```

```
  @Test
    public void test8() {
        // 测试重复注册后，会执行Hook吗
        Thread thread = new Thread(() -> {
            System.out.println("hook1 执行了");
        });

        Runtime.getRuntime().addShutdownHook(thread);
        Runtime.getRuntime().addShutdownHook(thread);
    }
    
    输出：java.lang.IllegalArgumentException: Hook previously registered
```

```
  @Test
    public void test9() {
        // 测试重复注册后，会执行Hook吗
        Thread thread = new Thread(() -> {
            System.out.println("hook1 执行了");
        });

        thread.start();
        Runtime.getRuntime().addShutdownHook(thread);
    }
    
    输出：
        hook1 执行了

    	java.lang.IllegalArgumentException: Hook already running
```



# 总结

1.ShutDown的使用还是比较简单，网上也有分析Spring和Dubbo等开源框架的使用例子，基本上都是用于销毁处理资源释放的问题

2.稍微要注意的就是一些特殊情况，比如hook执行是无序的，不能重复添加相同的hook，已经执行的hook不能再创建新的hook等

3.平时基本没用到过ShutDown Hook，自己想到一个比较有用的场景就是Jvm挂了，在Hook里面给监控程序发通知发邮件之类的，让技术人员来处理



# 参考资料

[1.oracle官网资料](https://docs.oracle.com/javase/8/docs/technotes/guides/lang/hook-design.html)

[2.Java Shutdown Hook 场景使用和源码分析](https://segmentfault.com/a/1190000040167517)

[3.Adding Shutdown Hooks for JVM Applications](https://www.baeldung.com/jvm-shutdown-hooks)