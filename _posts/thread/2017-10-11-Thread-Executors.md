---
layout: post
title: 使用 Executors，ThreadPoolExecutor，创建线程池，源码分析理解
categories: Thread
description: 使用 Executors，ThreadPoolExecutor，创建线程池，源码分析理解
keywords: Thread
---

之前创建线程的时候都是用的 `newCachedThreadPoo`,`newFixedThreadPool`,`newScheduledThreadPool`,`newSingleThreadExecutor` 这四个方法。  
当然 `Executors` 也是用不同的参数去 `new ThreadPoolExecutor` 实现的，本文先分析前四种线程创建方式，后在分析 `new ThreadPoolExecutor` 创建方式

# 使用 Executors 创建线程池 

## 1.newFixedThreadPool()

由于使用了`LinkedBlockingQueue`所以`maximumPoolSize`没用，当`corePoolSize`满了之后就加入到`LinkedBlockingQueue`队列中。
每当某个线程执行完成之后就从`LinkedBlockingQueue`队列中取一个。
所以这个是创建固定大小的线程池。

源码分析

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
	return new ThreadPoolExecutor(
			nThreads,
			nThreads,
			0L,
			TimeUnit.MILLISECONDS,
			new LinkedBlockingQueue<Runnable>());
}
```
## 2.newSingleThreadPool()

创建线程数为1的线程池，由于使用了`LinkedBlockingQueue`所以`maximumPoolSize` 没用，`corePoolSize`为1表示线程数大小为1,满了就放入队列中，执行完了就从队列取一个。

源码分析

```java
public static ExecutorService newSingleThreadExecutor() {
	return new Executors.FinalizableDelegatedExecutorService
			(
					new ThreadPoolExecutor(
							1,
							1,
							0L,
							TimeUnit.MILLISECONDS,
							new LinkedBlockingQueue<Runnable>())
			);
}

public ThreadPoolExecutor(int corePoolSize,
						  int maximumPoolSize,
						  long keepAliveTime,
						  TimeUnit unit,
						  BlockingQueue<Runnable> workQueue) {
	this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
			Executors.defaultThreadFactory(), defaultHandler);
}
```
## 3.newCachedThreadPool()

创建可缓冲的线程池。没有大小限制。由于`corePoolSize`为0所以任务会放入`SynchronousQueue`队列中，`SynchronousQueue`只能存放大小为1，所以会立刻新起线程，由于`maxumumPoolSize`为`Integer.MAX_VALUE`所以可以认为大小为`2147483647`。受内存大小限制。

源码分析

```sh
public static ExecutorService newCachedThreadPool() {
	return new ThreadPoolExecutor(
			0,
			Integer.MAX_VALUE,
			60L,
			TimeUnit.SECONDS,
			new SynchronousQueue<Runnable>());
}

public ThreadPoolExecutor(int corePoolSize,
						  int maximumPoolSize,
						  long keepAliveTime,
						  TimeUnit unit,
						  BlockingQueue<Runnable> workQueue) {
	this(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue,
			Executors.defaultThreadFactory(), defaultHandler);
}
```


# 使用 ThreadPoolExecutor 创建线程池

源码分析 ,`ThreadPoolExecutor` 的构造函数

```java
public ThreadPoolExecutor(int corePoolSize,
						  int maximumPoolSize,
						  long keepAliveTime,
						  TimeUnit unit,
						  BlockingQueue<Runnable> workQueue,
						  ThreadFactory threadFactory,
						  RejectedExecutionHandler handler) {
	if (corePoolSize < 0 ||
			maximumPoolSize <= 0 ||
			maximumPoolSize < corePoolSize ||
			keepAliveTime < 0)
		throw new IllegalArgumentException();
	if (workQueue == null || threadFactory == null || handler == null)
		throw new NullPointerException();
	this.corePoolSize = corePoolSize;
	this.maximumPoolSize = maximumPoolSize;
	this.workQueue = workQueue;
	this.keepAliveTime = unit.toNanos(keepAliveTime);
	this.threadFactory = threadFactory;
	this.handler = handler;
}
```

## 构造函数参数

1、`corePoolSize` 核心线程数大小，当线程数 < corePoolSize ，会创建线程执行 runnable

2、`maximumPoolSize` 最大线程数， 当线程数 >= corePoolSize的时候，会把 runnable 放入 workQueue中

3、`keepAliveTime`  保持存活时间，当线程数大于corePoolSize的空闲线程能保持的最大时间。

4、`unit` 时间单位

5、`workQueue` 保存任务的阻塞队列

6、`threadFactory` 创建线程的工厂

7、`handler` 拒绝策略


## 任务执行顺序

1、当线程数小于 `corePoolSize`时，创建线程执行任务。

2、当线程数大于等于 `corePoolSize`并且 `workQueue` 没有满时，放入`workQueue`中

3、线程数大于等于 `corePoolSize`并且当 `workQueue` 满时，新任务新建线程运行，线程总数要小于 `maximumPoolSize`

4、当线程总数等于 `maximumPoolSize` 并且 `workQueue` 满了的时候执行 `handler` 的 `rejectedExecution`。也就是拒绝策略。

## 四个拒绝策略

ThreadPoolExecutor默认有四个拒绝策略：


1、`ThreadPoolExecutor.AbortPolicy()`   直接抛出异常`RejectedExecutionException`

2、`ThreadPoolExecutor.CallerRunsPolicy()`    直接调用run方法并且阻塞执行

3、`ThreadPoolExecutor.DiscardPolicy()`   直接丢弃后来的任务

4、`ThreadPoolExecutor.DiscardOldestPolicy()`  丢弃在队列中队首的任务

当然可以自己继承RejectedExecutionHandler来写拒绝策略.

# TestThreadPoolExecutor 示例

## TestThreadPoolExecutor.java

```java
package io.ymq.thread.TestThreadPoolExecutor;

import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.ThreadPoolExecutor;
import java.util.concurrent.TimeUnit;

/**
 * 描述:
 *
 * @author yanpenglei
 * @create 2017-10-12 15:39
 **/
public class TestThreadPoolExecutor {
    public static void main(String[] args) {

        long currentTimeMillis = System.currentTimeMillis();

        // 构造一个线程池
        ThreadPoolExecutor threadPool = new ThreadPoolExecutor(5, 6, 3,
                TimeUnit.SECONDS, new ArrayBlockingQueue<Runnable>(3)
        );

        for (int i = 1; i <= 10; i++) {
            try {
                String task = "task=" + i;
                System.out.println("创建任务并提交到线程池中：" + task);
                threadPool.execute(new ThreadPoolTask(task));

                Thread.sleep(100);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        try {
            //等待所有线程执行完毕当前任务。
            threadPool.shutdown();

            boolean loop = true;
            do {
                //等待所有线程执行完毕当前任务结束
                loop = !threadPool.awaitTermination(2, TimeUnit.SECONDS);//等待2秒
            } while (loop);

            if (loop != true) {
                System.out.println("所有线程执行完毕");
            }

        } catch (InterruptedException e) {
            e.printStackTrace();
        } finally {
            System.out.println("耗时：" + (System.currentTimeMillis() - currentTimeMillis));
        }


    }
}

```

## ThreadPoolTask.java

```java
package io.ymq.thread.TestThreadPoolExecutor;

import java.io.Serializable;

/**
 * 描述:
 *
 * @author yanpenglei
 * @create 2017-10-12 15:40
 **/
public class ThreadPoolTask implements Runnable, Serializable {

    private Object attachData;

    ThreadPoolTask(Object tasks) {
        this.attachData = tasks;
    }

    public void run() {

        try {

            System.out.println("开始执行任务：" + attachData + "任务，使用的线程池，线程名称：" + Thread.currentThread().getName());

            System.out.println();

        } catch (Exception e) {
            e.printStackTrace();
        }
        attachData = null;
    }

}
```

**遇到java.util.concurrent.RejectedExecutionException**

**第一**

你的线程池 `ThreadPoolExecutor` 显示的 `shutdown()` 之后，再向线程池提交任务的时候。 如果你配置的拒绝策略是 `AbortPolicy` 的话，这个异常就会抛出来。

**第二**

当你设置的任务缓存队列过小的时候，或者说， 你的线程池里面所有的线程都在干活（线程数== `maxPoolSize`),并且你的任务缓存队列也已经充满了等待的队列， 这个时候，你再向它提交任务，则会抛出这个异常。


**响应**

可以看到线程 pool-1-thread-1 到5 循环使用

```sh
创建任务并提交到线程池中：task=1
开始执行任务：task=1任务，使用的线程池，线程名称：pool-1-thread-1

创建任务并提交到线程池中：task=2
开始执行任务：task=2任务，使用的线程池，线程名称：pool-1-thread-2

创建任务并提交到线程池中：task=3
开始执行任务：task=3任务，使用的线程池，线程名称：pool-1-thread-3

创建任务并提交到线程池中：task=4
开始执行任务：task=4任务，使用的线程池，线程名称：pool-1-thread-4

创建任务并提交到线程池中：task=5
开始执行任务：task=5任务，使用的线程池，线程名称：pool-1-thread-5

创建任务并提交到线程池中：task=6
开始执行任务：task=6任务，使用的线程池，线程名称：pool-1-thread-1

创建任务并提交到线程池中：task=7
开始执行任务：task=7任务，使用的线程池，线程名称：pool-1-thread-2

创建任务并提交到线程池中：task=8
开始执行任务：task=8任务，使用的线程池，线程名称：pool-1-thread-3

创建任务并提交到线程池中：task=9
开始执行任务：task=9任务，使用的线程池，线程名称：pool-1-thread-4

创建任务并提交到线程池中：task=10
开始执行任务：task=10任务，使用的线程池，线程名称：pool-1-thread-5

所有线程执行完毕
耗时：1015
```
