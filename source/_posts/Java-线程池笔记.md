---
title: Java 线程池笔记
date: 2022-09-14 18:40:44
tags:
---


# ThreadPoolExecutor
ThreadPoolExecutor 继承了 AbstractExecutorService，成员变量 ctl 是一个 Integer 的原子变量，用来记录线程池状态和线程池中线程个数。

> 高 3 位用来表示线程池状态，低 29 位用来记录线程池线程个数。即，线程池状态理论上可以至多拥有 8 种。

实际上，线程池的状态如下：

|线程池状态|描述|
|:-|:-|
|RUNNING|接收新任务并且处理阻塞队列里的任务|
|SHUTDOWN|拒绝新任务但是处理阻塞队列里的任务|
|STOP|拒绝新任务并且抛弃阻塞队列里的任务，同时会中断正在处理的任务|
|TIDYING|所有任务都执行完（包含阻塞队列里面的任务）后当前线程池活动线程数位 0，将要调用 terminated 方法|
|TERMINATED|终止状态。terminated 方法调用完成以后的状态|

线程池参数：

|参数|描述|
|:-|:-|
|corePoolSize|核心线程数，即使他们空闲也会保持在线程池中|
|maximumPoolSize|线程池中允许的最大线程数|
|keepAliveTime|保持存活时间，如果线程数超过核心线程数，而且超过的线程不在工作（空闲），他们允许有keepAliveTime 的时间存活，以便等待新任务。| 
|TimeUnit|时长单位，用于 keepAliveTime|
|workQueue|用于保存等待执行的任务的阻塞队列|
|threadFactory|executor 用于创建线程的工厂|
|RejectedExecutionHandler|饱和策略，当队列满并且线程个数达到 maximunPoolSize 后采取的策略|


- keepAliveTime

一般情况下，keepAliveTime 可以设置为 0，表示线程运行完毕立即销毁；keepAliveTime < 0，初始化才会报错

如果调用了 allowCoreThreadTimeOut(true)，那么 keepAlive <= 0 就会报错，这表示允许核心线程等待任务超时，而不是常驻。



## newFixedThreadPool

`newFixedThreadPool` 创建一个核心线程个数和最大线程个数都为 nThreads 的线程池：

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>());
 }
 //使用自定义线程创建工厂
 public static ExecutorService newFixedThreadPool(int nThreads, ThreadFactory threadFactory) {
        return new ThreadPoolExecutor(nThreads, nThreads,
                                      0L, TimeUnit.MILLISECONDS,
                                      new LinkedBlockingQueue<Runnable>(),
                                      threadFactory);
 }
```

keepAliveTime 为 0，说明只要线程个数比核心线程个数多并且当前空闲则回收。

这里传递了 `new LinkedBlockingQueue<Runnable>()` 作为阻塞队列，默认大小为 `Integer.MAX_VALUE`，因此可以认为是一个无界队列。


## newSingleThreadExecutor

创建一个核心线程数和最大线程数都为 1 的线程池：

```java
ExecutorService executor = Executors.newSingleThreadExecutor();
```


有界队列禁止设置长度为 0，至少是 1，因此似乎没有办法做到仅固定线程活跃，其他任务拒绝：

```java
// 错误
ThreadPoolExecutor threadPoolExecutor = new ThreadPoolExecutor(
            THREAD_POOL_SIZE, THREAD_POOL_SIZE, 0L, TimeUnit.SECONDS,
            new ArrayBlockingQueue<>(0), new ThreadPoolExecutor.AbortPolicy());
```




## Executor
Executor > ExecutorService > AbstractExecutorService > ThreadPoolExecutor

execute() 和 submit() 有什么区别？

1. execute 无法返回值；submit 可以返回值

2. submit 底层调用了 execute

```java
public ThreadPoolExecutor(
	int corePoolSize,
	int maximumPoolSize,
	long keepAliveTime,
	TimeUnit unit,
	BlockingQueue<Runnable> workQueue,
	ThreadFactory threadFactory,
	RejectedExecutionHandler handler)
```
## shutdown
调用 shutdown 方法后，线程池就不会再接受新的任务了，但是工作队列里面的任务还是要执行的。该方法会立即返回，并不等待队列任务完成再返回。

## shutdownNow
调用 shutdownNow 方法后，线程池就不会再接受新的任务了，并且会丢弃工作队列里面的任务，正在执行的任务会被中断，该方法会立即返回，并不等待激活的任务执行完成。


# org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor

创建一个 Spring 线程池 ThreadPoolTaskExecutor

它暴露了Executor的配置参数作为bean属性

```java
ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
//配置核心线程数
executor.setCorePoolSize(100);
//配置最大线程数
executor.setMaxPoolSize(200);
//配置队列大小
executor.setQueueCapacity(2000000);
//配置线程池中的线程的名称前缀
executor.setThreadNamePrefix("mythread-");
//线程执行时间
executor.setKeepAliveSeconds(customPool.getKeepAliveSeconds());

// rejection-policy：当pool已经达到max size的时候，如何处理新任务
// CALLER_RUNS：不在新线程中执行任务，而是有调用者所在的线程来执行
executor.setRejectedExecutionHandler(new ThreadPoolRejectedPolicyHandler());
//执行初始化
executor.initialize();
return executor;
```


# 线程池 main 线程等待所有线程结束

```java
@Test
public void mainWait() throws InterruptedException {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(5); //核心池大小
    executor.setMaxPoolSize(10); //最大线程数
    executor.setQueueCapacity(10); //队列程度
    executor.setThreadNamePrefix("sub-thread-");//线程前缀名称
    executor.initialize(); //初始化

    int count = 5; // 任务数量
    CountDownLatch countDownLatch = new CountDownLatch(count); // 同步工具
    for (int i = 0; i < count; i++) {
        executor.execute(() -> task(countDownLatch));
    }
    System.out.println("main 线程等待子线程完成...");
    countDownLatch.await();
    System.out.println("main 线程工作结束.");
    executor.shutdown();
}

private void task(CountDownLatch countDownLatch) {
    try {
        System.out.println(Thread.currentThread().getName() + " 工作开始！");
        Thread.sleep((long) (Math.random() * 2000));
        System.out.println(Thread.currentThread().getName() + " 工作结束！");
        countDownLatch.countDown();
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
}
```