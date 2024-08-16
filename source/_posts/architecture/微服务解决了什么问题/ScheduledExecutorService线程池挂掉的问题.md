---
title: ScheduledExecutorService线程池挂掉的问题
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: 微服务解决了什么问题
---


### 导语：
最近遇到一个问题，有个周期给业务方推送信息的功能，突然就没推送了。日志里面也没有查询到报错的信息，
然后检查代码发现原来写这个推送功能用的是ScheduledExecutorService，再设定好执行时间，就会周期性去用
ScheduledExecutorService的子线程来执行任务。

### 问题排查：
问题的原因最后发现是某些配置修改了，导致运行推送任务的子线程在执行的时候回抛出异常，但是这个线程池是不打印子线程的异常信息的，
所以日志里面根本看不到报错的信息。

#### 测试例子：
写一个简单方法，直接子线程就抛出异常，看看会不会打印堆栈信息：
```
@Slf4j
public class ScheduledExecutorServiceTest {

    public static Integer count = 0;

    public static void main(String[] args) {
        ScheduledExecutorService executorService = Executors.newScheduledThreadPool(1);
        executorService.scheduleAtFixedRate(() -> {
            if (count % 2 == 1) {
                throw new RuntimeException();
            } else {
                log.info("子线程执行了");
            }
            count++;
        }, 0, 1, TimeUnit.SECONDS);
    }
}
```
执行结果
```
Connected to the target VM, address: '127.0.0.1:49300', transport: 'socket'
20:10:42.180 [pool-1-thread-1] INFO com.scott.java.task.thread.scheduled.executor.service.ScheduledExecutorServiceTest - 子线程执行了
```
可以看到只有在第一次执行成功了，当count表位1的时候走到throw new RuntimeException()并没有打印出抛出的异常信息，之后也在没有打印后续的日志。
所以当count等于1的时候线程报错，这个线程就不会在执行任务后续的任务了。只要程序出错，这个线程池就会挂掉，如果有多个子线程的话，一个个子线程慢慢的都会抛出异常，会慢慢的
挂掉。

### 解决方式：
#### 方式一：给子线程用try-catch给包起来
因为是子线程报错，抛了异常才导致线程池挂掉，如果我们把异常捕获了打印出日志，那么子线程下个周期还是可以正常运行的。
例子：
```
public static void main(String[] args) {
        ScheduledExecutorService executorService = Executors.newScheduledThreadPool(1);
        executorService.scheduleAtFixedRate(() -> {
            try {
                if (count % 2 == 1) {
                    throw new RuntimeException();
                } else {
                    log.info("子线程执行了");
                }
                count++;
            } catch (Exception e) {
                log.error("线程执行异常", e);
            }
        }, 0, 1, TimeUnit.SECONDS);
    }
```
输出结果：
```
20:25:54.648 [pool-1-thread-1] ERROR com.scott.java.task.thread.scheduled.executor.service.ScheduledExecutorServiceTest - 线程执行异常
java.lang.RuntimeException: null
	at com.scott.java.task.thread.scheduled.executor.service.ScheduledExecutorServiceTest.lambda$main$0(ScheduledExecutorServiceTest.java:24)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.runAndReset$$$capture(FutureTask.java:308)
	at java.util.concurrent.FutureTask.runAndReset(FutureTask.java)
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$301(ScheduledThreadPoolExecutor.java:180)
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:294)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
20:25:55.642 [pool-1-thread-1] ERROR com.scott.java.task.thread.scheduled.executor.service.ScheduledExecutorServiceTest - 线程执行异常
java.lang.RuntimeException: null
	at com.scott.java.task.thread.scheduled.executor.service.ScheduledExecutorServiceTest.lambda$main$0(ScheduledExecutorServiceTest.java:24)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.runAndReset$$$capture(FutureTask.java:308)
	at java.util.concurrent.FutureTask.runAndReset(FutureTask.java)
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$301(ScheduledThreadPoolExecutor.java:180)
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:294)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
	at java.lang.Thread.run(Thread.java:748)
```
可以看到这次抛出的异常都可以打印出来了，而且线程池继续在执行着。

#### 方式二：使用@Scheduled注解来实现周期性任务
例子：
```
@Slf4j
@SpringBootApplication
@EnableScheduling
public class ScheduledTest {

    public static Integer count = 0;

    public static void main(String[] args) {
        SpringApplication.run(ScheduledTest.class);
    }

    @Scheduled(fixedRate = 1000)
    public void test() {
        if (count % 2 == 1) {
            throw new RuntimeException();
        } else {
            log.info("子线程执行了");
        }
        count++;
    }
}
```
输出结果：
```
2020-12-20 20:36:55.431 ERROR 2164 --- [pool-1-thread-1] o.s.s.s.TaskUtils$LoggingErrorHandler    : Unexpected error occurred in scheduled task.

java.lang.RuntimeException: null
	at com.scott.java.task.thread.scheduled.executor.service.ScheduledTest.test(ScheduledTest.java:27) ~[classes/:na]
	at sun.reflect.GeneratedMethodAccessor28.invoke(Unknown Source) ~[na:na]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_162]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_162]
	at org.springframework.scheduling.support.ScheduledMethodRunnable.run(ScheduledMethodRunnable.java:65) ~[spring-context-5.0.5.RELEASE.jar:5.0.5.RELEASE]
	at org.springframework.scheduling.support.DelegatingErrorHandlingRunnable.run(DelegatingErrorHandlingRunnable.java:54) ~[spring-context-5.0.5.RELEASE.jar:5.0.5.RELEASE]
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511) [na:1.8.0_162]
	at java.util.concurrent.FutureTask.runAndReset$$$capture(FutureTask.java:308) [na:1.8.0_162]
	at java.util.concurrent.FutureTask.runAndReset(FutureTask.java) [na:1.8.0_162]
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$301(ScheduledThreadPoolExecutor.java:180) [na:1.8.0_162]
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:294) [na:1.8.0_162]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [na:1.8.0_162]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [na:1.8.0_162]
	at java.lang.Thread.run(Thread.java:748) [na:1.8.0_162]

2020-12-20 20:36:56.431 ERROR 2164 --- [pool-1-thread-1] o.s.s.s.TaskUtils$LoggingErrorHandler    : Unexpected error occurred in scheduled task.

java.lang.RuntimeException: null
	at com.scott.java.task.thread.scheduled.executor.service.ScheduledTest.test(ScheduledTest.java:27) ~[classes/:na]
	at sun.reflect.GeneratedMethodAccessor28.invoke(Unknown Source) ~[na:na]
	at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43) ~[na:1.8.0_162]
	at java.lang.reflect.Method.invoke(Method.java:498) ~[na:1.8.0_162]
	at org.springframework.scheduling.support.ScheduledMethodRunnable.run(ScheduledMethodRunnable.java:65) ~[spring-context-5.0.5.RELEASE.jar:5.0.5.RELEASE]
	at org.springframework.scheduling.support.DelegatingErrorHandlingRunnable.run(DelegatingErrorHandlingRunnable.java:54) ~[spring-context-5.0.5.RELEASE.jar:5.0.5.RELEASE]
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511) [na:1.8.0_162]
	at java.util.concurrent.FutureTask.runAndReset$$$capture(FutureTask.java:308) [na:1.8.0_162]
	at java.util.concurrent.FutureTask.runAndReset(FutureTask.java) [na:1.8.0_162]
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.access$301(ScheduledThreadPoolExecutor.java:180) [na:1.8.0_162]
	at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:294) [na:1.8.0_162]
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149) [na:1.8.0_162]
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624) [na:1.8.0_162]
	at java.lang.Thread.run(Thread.java:748) [na:1.8.0_162]
```
可以看到周期性的抛出了异常，但是线程还是在继续执行着。


### 总结：
1.ScheduledExecutorService和普通的线程池都会把异常信息给吃掉，会导致线程池也挂掉
2.使用线程池的时候要注意catch住异常或者使用线程池的submit来打印异常信息  
3.可以使用@Scheduled注解来替代ScheduledExecutorService。  
