---
title: JAVA常见线程池 
date: 2020-02-02 21:02:16
comments: true
categories: 
- 笔记
tags: 
- java
---

通过JAVA程序创建线程对象的成本是比较高的，在需要使用线程的应用中为了减少创建线程的开销一般会使用线程池，通过池化技术来尽可能减少线程对象创建和小会的开销。

<!-- more -->  
## 线程池工作原理  
线程池是一组预先初始化完成的线程集合，该集合大小可以进行设置，同时还包括一个队列结构用于存放未能被立即执行的任务。  
当任务提交后，已经创建好的空闲线程便可以立即执行该任务，若没有空闲线程则将任务放置到任务队列中，当线程池中的线程执行完毕当前任务后便去任务队列中获取新的任务进行执行。  
若任务队列中没有可执行任务，线程便会保持活跃状态进行等待任务    

好比是队列中存放着一个个实现了Runnable接口的Task，然后线程不断的在轮询队列queue中是否有能够执行的Task，若有则该线程取到该Task执行他的run方法  

## 使用线程池的优势  
* 使用线程池管理线程能够减少创建线程和销毁线程的开销  
* 重复利用线程  
* 线程池能够提供简单的统计功能   
* 直接从线程池中获取已经创建好的线程执行任务比创建线程对象执行任务的效率高  

## ThreadPoolExecutor  
JDK1.5之后提供了ThreadPoolExecutor类，该类实现了Executor接口及其子接口ExecutorService  
  
使用ThreadPoolExecutor可以创建五种预设的线程池执行器，也可通过他的构造函数指定参数创建自定义的线程池执行器  

### ThreadPoolExecutor构造函数    

```java  
    public ThreadPoolExecutor(int corePoolSize,
                              int maximumPoolSize,
                              long keepAliveTime,
                              TimeUnit unit,
                              BlockingQueue<Runnable> workQueue,
                              ThreadFactory threadFactory,
                              RejectedExecutionHandler handler)


```


核心参数  
* corePoolSize 线程池核心线程数大小  
* maximumPoolSize 线程池最大线程数大小  
* keepAliveTime 线程池中非核心线程的空闲存活时间  
* unit 线程空闲存活时间单位  
* workQueue 存放未能被执行的任务的队列  
* threadFactory 线程创建工厂  
* handler 线程池饱和时的拒绝策略处理器   


### 五种预设的线程池执行器  
通过ThreadPoolExecutor的构造函数可以传入参数创建自定义的线程池，也可以直接使用五种默认的线程池  
* **ExecutorService newFixedThreadPool(int nThreads)  固定线程数量的线程池**  
通过构造函数传入的nThreads指定线程池线程大小，该线程池的执行线程数目固定，当有超过线程池空闲线程数目的任务提交后会进入队列中等待执行  
* **ExecutorService newCachedThreadPool()缓存的线程池**  
当任务提交时，若有空闲线程则会复用该线程，若没有空闲线程则创建新的线程执行，该新线程执行完毕后也会被复用，若是同时提交大量任务则有可能导致线程数目不可控  
* **ScheduledExecutorService newScheduledThreadPool(int corePoolSize)调度线程池**  
该线程池在提交任务时可传入指定的延后时间，在该时间后执行任务，提交任务有四种方法  
1、ScheduledFuture schedule(Runnable command, long delay, TimeUnit unit)  
2、ScheduledFuture schedule(Callable callable, long delay, TimeUnit unit)  
均是在指定延后时间delay之后执行任务  
3、ScheduledFuture scheduleAtFixedRate(Runnable command, long initialDelay, long delay, TimeUnit unit)  
提交的任务在指定延后时间initialDelay之后执行，同时delay指定周期时间，下一周期的执行在上一周期执行后的delay时间后执行，若任务执行时间大于delay时间，下一周期开始执行则是任务执行后  
4、ScheduledFuture scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit)  
与3相同都是周期执行的调度，不同点在于下一周期开始时间不受上一周期的任务时间影响，若任务时间大于周期时间，下一周期的开始时间依旧是上一周期任务结束时间+delay时间    
* **ThreadPoolExecutor executor = (ThreadPoolExecutor) Executors.newSingleThreadExecutor();线程池中执行任务的线程只有一个**  
* **ThreadPoolExecutor executor = (ThreadPoolExecutor) Executors.newWorkStealingPool(4);根据机器的处理器数目创建线程并行执行任务，任务执行不存在顺序**

### 线程池的拒绝策略  
当线程池中的线程数达到最大线程数且所有线程均在执行任务时，若有新任务提交则会触发拒绝策略，主要有四种拒绝策略  
* AbortPolicy 抛出异常  
* DiscardPolicy 直接丢弃任务  
* DiscardOldestPolicy 丢弃队列中最老的任务然后提交当前任务  
* CallerRunsPolicy 由调用线程池的当前线程执行该任务    
----
**自定义拒绝策略**  
在ThreadPoolExecutor的构造函数中可以传入RejectedExecutionHandler来实现自定义的拒绝策略  
通过继承RejectedExecutionHandler类实现public void rejectedExecution(Runnable r, ThreadPoolExecutor executor)方法即可



### 线程池的工作顺序  
既然线程池存在拒绝策略，那么什么时候会触发该策略呢？  

当调用线程池的execute()方法提交任务后，便触发了线程池的执行顺序  
  
  - 当任务提交后，若有空闲的核心线程则使用空闲的核心线程执行任务。若核心线程数小于设置的corePoolSize且没有空闲的核心线程则创建线程执行该任务  
  - 若核心线程数等于设置的corePoolSize且没有空闲核心线程的话则将任务放置到队列中等待
  - 若等待队列也满了则判断线程数是否达到了最大线程数maximumPoolSize，若没有则创建一个非核心线程执行任务  
  - 若线程数达到了maximumPoolSize且没有空闲线程，则触发拒绝策略   

### 线程池队列  
常见的五种线程池队列  
* **ArrayBlockingQueue**  
数组有界阻塞队列，根据FIFO排序
* **LinkedBlockingQueue**  
使用链表结构的阻塞队列，根据FIFO排序，可设置容量，没有设置的话则是个无界队列，最大的Integer.MAX_VALUE，newFixedThreadPool使用到该队列  
* **DelayQueue**  
任务定时周期的延迟执行队列，根据执行时间从小到大排序，newScheduledThreadPool使用该队列  
* **PriorityBlockingQueue**  
具有优先级的无界阻塞队列  
* **SynchronousQueue**  
不存储元素的阻塞队列，每个新任务的进入都要有另一个线程将任务移除    

## 阿里巴巴开发手册不建议使用Executor的静态方法创建线程池  
前面提到了可以通过Executor的静态方法创建预设的线程池，而阿里巴巴开发手册中则禁止该行为，主要是为了避免出现OOM异常（OutOfMemoryError）  
  
预设的五种线程池中newCachedThreadPool、newSingleThreadExecutor、newFixedThreadPool是返回ExecutorService对象，以这三种为例  
* newCachedThreadPool    
其构造函数中，核心线程数为0，最大线程数为Integer.MAX_VALUE，任务队列为SynchronousQueue

```java  
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}


```

由于SynchronousQueue队列不存储元素，可以理解为队列一直处于满的状态，会不断创建非核心线程进行处理，若不断的提交任务则会不断的创建线程，有出现OOM的风险  
* newSingleThreadExecutor  
其构造函数中线程只有一个，但是队列使用的是LinkedBlockingQueue，最大程度为Integer.MAX_VALUE  

```java  
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}



```

在任务不断的提交的情况下，会不断往队列中存放任务，有OOM的风险  
* newFixedThreadPool  
其构造函数中核心线程数由用户指定，队列使用的是LinkedBlockingQueue  

```java  
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}


```

与newSingleThreadExecutor一样不断的提交任务也有可能出现OOM的风险



