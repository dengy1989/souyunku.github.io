---
layout: post
title: Java 四种线程池的使用
categories: Thread
description: Java 四种线程池的使用
keywords: Thread
---

介绍new Thread的弊端及Java四种线程池的使用

# 1，线程池的作用

**线程池作用就是限制系统中执行线程的数量。**  

根据系统的环境情况，可以自动或手动设置线程数量，达到运行的最佳效果。  
少了浪费了系统资源，多了造成系统拥挤效率不高。  
用线程池控制线程数量，其他线程排 队等候。
一个任务执行完毕，再从队列的中取最前面的任务开始执行。  
若队列中没有等待进程，线程池的这一资源处于等待。
当一个新任务需要运行时，如果线程池 中有等待的工作线程，就可以开始运行了；否则进入等待队列。

# 2，为什么要用线程池?

1.减少了创建和销毁线程的次数，每个工作线程都可以被重复利用，可执行多个任务。

2.可以根据系统的承受能力，调整线程池中工作线线程的数目，防止因为消耗过多的内存，而把服务器累趴下(每个线程需要大约1MB内存，线程开的越多，消耗的内存也就越大，最后死机)。

Java里面线程池的顶级接口是`Executor`，但是严格意义上讲`Executor`并不是一个线程池，而只是一个执行线程的工具。真正的线程池接口是`ExecutorService`。

# 3，比较重要的几个类

类  | 描述
---|---
ExecutorService | 真正的线程池接口。
ScheduledExecutorService | 能和Timer/TimerTask类似，解决那些需要任务重复执行的问题。
ThreadPoolExecutor | ExecutorService的默认实现。
ScheduledThreadPoolExecutor | 继承ThreadPoolExecutor的ScheduledExecutorService接口实现，周期性任务调度的类实现。

# 4，new Thread的弊端

```java
public class TestNewThread {

    public static void main(String[] args) {
        new Thread(new Runnable() {

            @Override
            public void run() {
                System.out.println("start");
            }
        }).start();
    }
}
```

**执行一个异步任务你还只是如下new Thread吗？**

**那你就out太多了，new Thread的弊端如下：**

1.每次new Thread新建对象性能差。  
2.线程缺乏统一管理，可能无限制新建线程，相互之间竞争，及可能占用过多系统资源导致死机或oom。  
3.缺乏更多功能，如定时执行、定期执行、线程中断。  

**相比new Thread，Java提供的四种线程池的好处在于：**  

1.重用存在的线程，减少对象创建、消亡的开销，性能佳。  
2.可有效控制最大并发线程数，提高系统资源的使用率，同时避免过多资源竞争，避免堵塞。  
3.提供定时执行、定期执行、单线程、并发数控制等功能。  

# 四种线程池

**Java通过Executors提供四种线程池，分别为：**  

## 1,newCachedThreadPoo

创建一个可缓存线程池，如果线程池长度超过处理需要，可灵活回收空闲线程，若无可回收，则新建线程。

## 2,newFixedThreadPool

创建一个定长线程池，可控制线程最大并发数，超出的线程会在队列中等待。 
 
## 3,newScheduledThreadPool

创建一个定长线程池，支持定时及周期性任务执行。
  
## 4,newSingleThreadExecutor

创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。 

# 示例

##  1，newCachedThreadPool

创建一个可缓存的线程池。如果线程池的大小超过了处理任务所需要的线程， 那么就会回收部分空闲（60秒不执行任务）的线程，当任务数增加时，此线程池又可以智能的添加新线程来处理任务。此线程池不会对线程池大小做限制，线程池大小完全依赖于操作系统（或者说JVM）能够创建的最大线程大小。

```java
package io.ymq.thread.demo1;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * 描述: 创建一个可缓存的线程池。如果线程池的大小超过了处理任务所需要的线程，那么就会回收部分空闲（60秒不执行任务）的线程，当任务数增加时，此线程池又可以智能的添加新线程来处理任务。
 * 此线程池不会对线程池大小做限制，线程池大小完全依赖于操作系统（或者说JVM）能够创建的最大线程大小。
 *
 * @author yanpenglei
 * @create 2017-10-12 11:13
 **/
public class TestNewCachedThreadPool {
    public static void main(String[] args) {
        ExecutorService cachedThreadPool = Executors.newCachedThreadPool();
        for (int i = 1; i <= 10; i++) {
            final int index = i;
            try {
                Thread.sleep(index * 1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            cachedThreadPool.execute(new Runnable() {

                @Override
                public void run() {
                    String threadName = Thread.currentThread().getName();
                    System.out.println("执行：" + index + "，线程名称：" + threadName);
                }
            });
        }
    }
}

```

响应：

```
执行：1，线程名称：pool-1-thread-1
执行：2，线程名称：pool-1-thread-1
执行：3，线程名称：pool-1-thread-1
执行：4，线程名称：pool-1-thread-1
执行：5，线程名称：pool-1-thread-1
执行：6，线程名称：pool-1-thread-1
执行：7，线程名称：pool-1-thread-1
执行：8，线程名称：pool-1-thread-1
执行：9，线程名称：pool-1-thread-1
执行：10，线程名称：pool-1-thread-1
```

##  2，newFixedThreadPool

描述:创建固定大小的线程池。每次提交一个任务就创建一个线程，直到线程达到线程池的最大大小。  
线程池的大小一旦达到最大值就会保持不变，如果某个线程因为执行异常而结束，那么线程池会补充一个新线程。

```java
package io.ymq.thread.demo2;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * 描述:创建固定大小的线程池。每次提交一个任务就创建一个线程，直到线程达到线程池的最大大小。
 * 线程池的大小一旦达到最大值就会保持不变，如果某个线程因为执行异常而结束，那么线程池会补充一个新线程。
 *
 * @author yanpenglei
 * @create 2017-10-12 11:30
 **/
public class TestNewFixedThreadPool {

    public static void main(String[] args) {

        ExecutorService fixedThreadPool = Executors.newFixedThreadPool(3);

        for (int i = 1; i <= 10; i++) {
            final int index = i;
            fixedThreadPool.execute(new Runnable() {

                @Override
                public void run() {
                    try {
                        String threadName = Thread.currentThread().getName();
                        System.out.println("执行：" + index + "，线程名称：" + threadName);
                        Thread.sleep(2000);
                    } catch (InterruptedException e) {

                        e.printStackTrace();
                    }
                }
            });
        }

    }
}

```

因为线程池大小为3，每个任务输出index后sleep 2秒，所以每两秒打印3个数字，和线程名称。

响应：

```
执行：2，线程名称：pool-1-thread-2
执行：3，线程名称：pool-1-thread-3
执行：1，线程名称：pool-1-thread-1

执行：4，线程名称：pool-1-thread-1
执行：6，线程名称：pool-1-thread-2
执行：5，线程名称：pool-1-thread-3

执行：7，线程名称：pool-1-thread-1
执行：9，线程名称：pool-1-thread-3
执行：8，线程名称：pool-1-thread-2

执行：10，线程名称：pool-1-thread-1

```

##  3，newScheduledThreadPool

创建一个定长线程池，支持定时及周期性任务执行。延迟执行

```sh
package io.ymq.thread.demo3;

import java.util.concurrent.Executors;
import java.util.concurrent.ScheduledExecutorService;
import java.util.concurrent.TimeUnit;

/**
 * 描述:创建一个定长线程池，支持定时及周期性任务执行。延迟执行
 *
 * @author yanpenglei
 * @create 2017-10-12 11:53
 **/
public class TestNewScheduledThreadPool {

    public static void main(String[] args) {

        ScheduledExecutorService scheduledThreadPool = Executors.newScheduledThreadPool(5);

        scheduledThreadPool.schedule(new Runnable() {

            @Override
            public void run() {
                System.out.println("表示延迟3秒执行。");
            }
        }, 3, TimeUnit.SECONDS);


        scheduledThreadPool.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                System.out.println("表示延迟1秒后每3秒执行一次。");
            }
        }, 1, 3, TimeUnit.SECONDS);
    }

}
```

```sh
表示延迟1秒后每3秒执行一次。
表示延迟3秒执行。
表示延迟1秒后每3秒执行一次。
表示延迟1秒后每3秒执行一次。
表示延迟1秒后每3秒执行一次。
表示延迟1秒后每3秒执行一次。
```

## 4，newSingleThreadExecutor

创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。

```java
package io.ymq.thread.demo4;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * 描述:创建一个单线程化的线程池，它只会用唯一的工作线程来执行任务，保证所有任务按照指定顺序(FIFO, LIFO, 优先级)执行。
 *
 * @author yanpenglei
 * @create 2017-10-12 12:05
 **/
public class TestNewSingleThreadExecutor {

    public static void main(String[] args) {
        ExecutorService singleThreadExecutor = Executors.newSingleThreadExecutor();
        for (int i = 1; i <= 10; i++) {
            final int index = i;
            singleThreadExecutor.execute(new Runnable() {

                @Override
                public void run() {
                    try {
                        String threadName = Thread.currentThread().getName();
                        System.out.println("执行：" + index + "，线程名称：" + threadName);
                        Thread.sleep(2000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            });
        }
    }
}
```

结果依次输出，相当于顺序执行各个任务。

响应：
```
执行：1，线程名称：pool-1-thread-1
执行：2，线程名称：pool-1-thread-1
执行：3，线程名称：pool-1-thread-1
执行：4，线程名称：pool-1-thread-1
执行：5，线程名称：pool-1-thread-1
执行：6，线程名称：pool-1-thread-1
执行：7，线程名称：pool-1-thread-1
执行：8，线程名称：pool-1-thread-1
执行：9，线程名称：pool-1-thread-1
执行：10，线程名称：pool-1-thread-1
```

# Contact

 - 作者：鹏磊  
 - 出处：[http://www.ymq.io](http://www.ymq.io)  
 - Email：[admin@souyunku.com](admin@souyunku.com)
 - 版权归作者所有，转载请注明出处
 - Wechat：关注公众号，搜云库，分享技术，分享生活
 
![关注公众号-搜云库](http://www.ymq.io/images/souyunku.png "搜云库")


