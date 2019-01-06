---
layout: post
title:  "SpringBoot多线程任务执行"
description: "SpringBoot Multi-Thread Task Executor"
date:   2018-11-28 16:22:39 +0800
categories: SpringBoot
---

>本文记录一下在开发过程中遇到的SpringBoot框架使用多线程执行定时任务的问题。

最近在项目开发过程中遇到一个需求，需要异步地调用同一个定时器，也就是下一次执行无需等待上一次执行完成后再进行，原本以为只需要使用``@fixedRate``注解即可, 结果发现方法并没有被异步地执行，而是等到上一次执行完成之后才被再次执行：

```java
@Component
public class MultiThreadScheduler {
    private static final Logger logger = LoggerFactory.getLogger(MultiThreadScheduler.class);

    /**
     * fixedRate 每次以指定的时间间隔执行
     * @throws InterruptedException
     */
    @Scheduled(fixedRate = 2 * 1000, initialDelay = 1000)
    public void task() throws InterruptedException {
        logger.info("task start : thread ---> " + Thread.currentThread().getName());
        Thread.sleep(3000);//休眠3秒使得方法的执行完成时间超过了两次被调度的间隔时间
        logger.info("task end : thread ---> " + Thread.currentThread().getName());
    }
}
```
执行的输出结果(每一个start都要等到上一个end之后)：
```bash
2018-11-26 10:28:58.811 n.j.demo.MultiThreadScheduler  : task start : thread ---> pool-1-thread-1
2018-11-26 10:29:01.811 n.j.demo.MultiThreadScheduler  : task end : thread ---> pool-1-thread-1
2018-11-26 10:29:01.813 n.j.demo.MultiThreadScheduler  : task start : thread ---> pool-1-thread-1
2018-11-26 10:29:04.818 n.j.demo.MultiThreadScheduler  : task end : thread ---> pool-1-thread-1
2018-11-26 10:29:04.818 n.j.demo.MultiThreadScheduler  : task start : thread ---> pool-1-thread-1
2018-11-26 10:29:07.819 n.j.demo.MultiThreadScheduler  : task end : thread ---> pool-1-thread-1
```

查了一下发现，原来<b>默认的定时器只有单线程来调度所有的计划任务</b>，即使两个定时器的initialDelay在同一时刻，但<b>谁先抢到谁就先执行，而其他则需要等待该任务执行完成后被调度</b>；而且无论fixedDelay或fixedRate也是一样，执行时间到了依然需要等待被调度。

```java
@Component
public class MultiThreadScheduler {
    private static final Logger logger = LoggerFactory.getLogger(MultiThreadScheduler.class);

    /**
     * fixedDelay 上次执行结束和下次执行开始之间的时间间隔
     * @throws InterruptedException
     */
   @Scheduled(fixedDelay = 2 * 1000, initialDelay = 1000)
    public void task1() throws InterruptedException {
        logger.info("task1 start : thread ---> " + Thread.currentThread().getName());
        Thread.sleep(3000);
        logger.info("task1 end : thread ---> " + Thread.currentThread().getName());
    }

    /**
     * fixedRate 每次以指定的时间间隔执行
     * @throws InterruptedException
     */
    @Scheduled(fixedRate = 2 * 1000, initialDelay = 1000)
    public void task2() throws InterruptedException {
        logger.info("task2 start : thread ---> " + Thread.currentThread().getName());
        Thread.sleep(3000);
        logger.info("task2 end : thread ---> " + Thread.currentThread().getName());
    }
}
```
上面这段代码的一次执行居然输出了以下内容，task2居然被连续执行了好几次，而task1居然等待了好久都没有进入执行，所以<b>两个在竞争的方法谁被调度执行完全是随机的</b>，这有可能导致一些非预期的执行结果；
```bash
2019-11-26 10:44:41.335  INFO 3740 --- [pool-1-thread-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 start : thread ---> pool-1-thread-1
2019-11-26 10:44:44.340  INFO 3740 --- [pool-1-thread-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 end : thread ---> pool-1-thread-1
2019-11-26 10:44:44.341  INFO 3740 --- [pool-1-thread-1] n.j.demo.scheduler.MultiThreadScheduler  : task1 start : thread ---> pool-1-thread-1
2019-11-26 10:44:47.342  INFO 3740 --- [pool-1-thread-1] n.j.demo.scheduler.MultiThreadScheduler  : task1 end : thread ---> pool-1-thread-1
2019-11-26 10:44:47.342  INFO 3740 --- [pool-1-thread-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 start : thread ---> pool-1-thread-1
2019-11-26 10:44:50.347  INFO 3740 --- [pool-1-thread-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 end : thread ---> pool-1-thread-1
2019-11-26 10:44:50.347  INFO 3740 --- [pool-1-thread-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 start : thread ---> pool-1-thread-1
2019-11-26 10:44:53.353  INFO 3740 --- [pool-1-thread-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 end : thread ---> pool-1-thread-1
2019-11-26 10:44:53.353  INFO 3740 --- [pool-1-thread-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 start : thread ---> pool-1-thread-1
2019-11-26 10:44:56.357  INFO 3740 --- [pool-1-thread-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 end : thread ---> pool-1-thread-1
2019-11-26 10:44:56.357  INFO 3740 --- [pool-1-thread-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 start : thread ---> pool-1-thread-1
2019-11-26 10:44:59.358  INFO 3740 --- [pool-1-thread-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 end : thread ---> pool-1-thread-1
2019-11-26 10:44:59.358  INFO 3740 --- [pool-1-thread-1] n.j.demo.scheduler.MultiThreadScheduler  : task1 start : thread ---> pool-1-thread-1
2019-11-26 10:45:02.363  INFO 3740 --- [pool-1-thread-1] n.j.demo.scheduler.MultiThreadScheduler  : task1 end : thread ---> pool-1-thread-1
2019-11-26 10:45:02.364  INFO 3740 --- [pool-1-thread-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 start : thread ---> pool-1-thread-1
```
于是查了一下，发现可以自定义一个<b>ThreadPoolTaskScheduler</b>来并发地执行定时任务：

```java
@Configuration
public class Configure implements SchedulingConfigurer {
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        //实例化一个ThreadPoolTaskScheduler
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(5);
        scheduler.setBeanName("taskScheduler");
        scheduler.setWaitForTasksToCompleteOnShutdown(true);
        scheduler.setAwaitTerminationSeconds(10);
        scheduler.initialize();//初始化该scheduler

        taskRegistrar.setScheduler(scheduler);//注册
    }
}
```

观察打印结果，两个定时任务被不同的线程独立执行：
```bash
2018-11-26 13:01:10.694  INFO 4357 --- [taskScheduler-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 start : thread ---> taskScheduler-1
2018-11-26 13:01:10.694  INFO 4357 --- [taskScheduler-2] n.j.demo.scheduler.MultiThreadScheduler  : task1 start : thread ---> taskScheduler-2
2018-11-26 13:01:13.697  INFO 4357 --- [taskScheduler-2] n.j.demo.scheduler.MultiThreadScheduler  : task1 end : thread ---> taskScheduler-2
2018-11-26 13:01:13.697  INFO 4357 --- [taskScheduler-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 end : thread ---> taskScheduler-1
2018-11-26 13:01:13.698  INFO 4357 --- [taskScheduler-2] n.j.demo.scheduler.MultiThreadScheduler  : task2 start : thread ---> taskScheduler-2
2018-11-26 13:01:15.699  INFO 4357 --- [taskScheduler-3] n.j.demo.scheduler.MultiThreadScheduler  : task1 start : thread ---> taskScheduler-3
2018-11-26 13:01:16.699  INFO 4357 --- [taskScheduler-2] n.j.demo.scheduler.MultiThreadScheduler  : task2 end : thread ---> taskScheduler-2
2018-11-26 13:01:16.699  INFO 4357 --- [taskScheduler-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 start : thread ---> taskScheduler-1
2018-11-26 13:01:18.703  INFO 4357 --- [taskScheduler-3] n.j.demo.scheduler.MultiThreadScheduler  : task1 end : thread ---> taskScheduler-3
2018-11-26 13:01:19.700  INFO 4357 --- [taskScheduler-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 end : thread ---> taskScheduler-1
```

同时还发现了另外一种实现方法，就是使用<b>@Async</b>注解,该注解可以<b>用于任何类的方法</b>上，使用了该注解的方法将被异步的执行，也就是说，你可以在你的每个定时任务上加上``@Async``注解，这样这些方法就可以独立执行了：

```java
@Component
public class MultiThreadScheduler {
    private static final Logger logger = LoggerFactory.getLogger(MultiThreadScheduler.class);

    /**
     * Async 该注解使方法可以被异步调用，也就是主线程无需等待该方法执行完再调度其他方法
     *       为了让Async可以生效，还必须加上@EnableAsync;
     *       该注释可以用在任何方法上
     * @throws InterruptedException
     */
    @Async
    @Scheduled(fixedDelay = 2 * 1000, initialDelay = 1000)
    public void task3() throws InterruptedException {
        logger.info("task3 start : thread ---> " + Thread.currentThread().getName());
        Thread.sleep(3000);
        logger.info("task3 end : thread ---> " + Thread.currentThread().getName());
    }

    /**
     * fixedDelay 上次执行结束和下次执行开始之间的时间间隔
     * @throws InterruptedException
     */
    @Scheduled(fixedDelay = 2 * 1000, initialDelay = 1000)
    public void task1() throws InterruptedException {
        logger.info("task1 start : thread ---> " + Thread.currentThread().getName());
        Thread.sleep(3000);
        logger.info("task1 end : thread ---> " + Thread.currentThread().getName());
    }

    /**
     * fixedRate 每次以指定的时间间隔执行
     * @throws InterruptedException
     */
    @Scheduled(fixedRate = 2 * 1000, initialDelay = 1000)
    public void task2() throws InterruptedException {
        logger.info("task2 start : thread ---> " + Thread.currentThread().getName());
        Thread.sleep(3000);
        logger.info("task2 end : thread ---> " + Thread.currentThread().getName());
    }
}
```
和TaskScheduler类似，你也可以自定义一个TaskExecutor(不指定也没关系，<b>默认情况下使用了@Async注解以后系统会使用SimpleAsyncTaskExecutor来进行异步调用</b>)：
```java
@Configuration
public class Configure implements AsyncConfigurer {
    @Override
    public Executor getAsyncExecutor() {
        //实例化一个ThreadPoolTaskExecutor
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(3);
        executor.setMaxPoolSize(10);
        executor.initialize();//初始化

        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return null;
    }
}
```
同时使用了taskScheduler和taskExecutor之后的定时器执行结果：
```bash
2018-11-26 13:21:01.373  INFO 4782 --- [taskScheduler-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 start : thread ---> taskScheduler-1
2018-11-26 13:21:01.373  INFO 4782 --- [lTaskExecutor-1] n.j.demo.scheduler.MultiThreadScheduler  : task3 start : thread ---> ThreadPoolTaskExecutor-1
2018-11-26 13:21:01.373  INFO 4782 --- [taskScheduler-3] n.j.demo.scheduler.MultiThreadScheduler  : task1 start : thread ---> taskScheduler-3
2018-11-26 13:21:03.377  INFO 4782 --- [lTaskExecutor-2] n.j.demo.scheduler.MultiThreadScheduler  : task3 start : thread ---> ThreadPoolTaskExecutor-2
2018-11-26 13:21:04.373  INFO 4782 --- [taskScheduler-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 end : thread ---> taskScheduler-1
2018-11-26 13:21:04.374  INFO 4782 --- [taskScheduler-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 start : thread ---> taskScheduler-1
2018-11-26 13:21:04.378  INFO 4782 --- [taskScheduler-3] n.j.demo.scheduler.MultiThreadScheduler  : task1 end : thread ---> taskScheduler-3
2018-11-26 13:21:04.378  INFO 4782 --- [lTaskExecutor-1] n.j.demo.scheduler.MultiThreadScheduler  : task3 end : thread ---> ThreadPoolTaskExecutor-1
2018-11-26 13:21:05.381  INFO 4782 --- [lTaskExecutor-3] n.j.demo.scheduler.MultiThreadScheduler  : task3 start : thread ---> ThreadPoolTaskExecutor-3
2018-11-26 13:21:06.382  INFO 4782 --- [lTaskExecutor-2] n.j.demo.scheduler.MultiThreadScheduler  : task3 end : thread ---> ThreadPoolTaskExecutor-2
2018-11-26 13:21:06.383  INFO 4782 --- [taskScheduler-5] n.j.demo.scheduler.MultiThreadScheduler  : task1 start : thread ---> taskScheduler-5
2018-11-26 13:21:07.376  INFO 4782 --- [taskScheduler-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 end : thread ---> taskScheduler-1
2018-11-26 13:21:07.376  INFO 4782 --- [taskScheduler-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 start : thread ---> taskScheduler-1
2018-11-26 13:21:07.382  INFO 4782 --- [lTaskExecutor-1] n.j.demo.scheduler.MultiThreadScheduler  : task3 start : thread ---> ThreadPoolTaskExecutor-1
2018-11-26 13:21:08.386  INFO 4782 --- [lTaskExecutor-3] n.j.demo.scheduler.MultiThreadScheduler  : task3 end : thread ---> ThreadPoolTaskExecutor-3
2018-11-26 13:21:09.387  INFO 4782 --- [taskScheduler-5] n.j.demo.scheduler.MultiThreadScheduler  : task1 end : thread ---> taskScheduler-5
2018-11-26 13:21:09.387  INFO 4782 --- [lTaskExecutor-2] n.j.demo.scheduler.MultiThreadScheduler  : task3 start : thread ---> ThreadPoolTaskExecutor-2
2018-11-26 13:21:10.377  INFO 4782 --- [taskScheduler-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 end : thread ---> taskScheduler-1
2018-11-26 13:21:10.378  INFO 4782 --- [taskScheduler-1] n.j.demo.scheduler.MultiThreadScheduler  : task2 start : thread ---> taskScheduler-1
2018-11-26 13:21:10.383  INFO 4782 --- [lTaskExecutor-1] n.j.demo.scheduler.MultiThreadScheduler  : task3 end : thread ---> ThreadPoolTaskExecutor-1
2018-11-26 13:21:11.392  INFO 4782 --- [taskScheduler-4] n.j.demo.scheduler.MultiThreadScheduler  : task1 start : thread ---> taskScheduler-4
2018-11-26 13:21:11.392  INFO 4782 --- [lTaskExecutor-3] n.j.demo.scheduler.MultiThreadScheduler  : task3 start : thread ---> ThreadPoolTaskExecutor-3

```
可以看出加上``@Async``注解的task3方法使用的是lTaskExecutor来进行多线程调用，其他两个没有使用该注解的task1，task2方法使用的是taskScheduler进行多线程调度。

总结一下：
<b>ThreadPoolTaskScheduler用于多线程定时器，而ThreadPoolTaskExecutor可以用于所有多线程任务。</b>

>可以通过 [这里](https://github.com/JohnnyChenS/spring-test-demo.git) 下载到我的Demo代码









