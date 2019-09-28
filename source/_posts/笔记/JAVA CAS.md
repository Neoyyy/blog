---
title: JAVA CAS 
date: 2019-09-26 21:02:16
comments: false
categories: 
- 买的课怎么也得学完
tags: 
- java
---

# CAS 和 FFA 是什么
CAS和FFA都是硬件同步原语，通过计算机硬件实现的原子操作。  
CAS全称叫 Compare And Set，参数有个旧值和新值，当通过CAS更改值时会比对原值是否相同，若相同则执行更新操作，不同则放弃该操作，是一个无锁的更新。  
FFA 全称叫 Fetch And Add，参数有增加的值 inc，即获取当前操作时的原值，并增加inc后返回原值，java中的FFA是通过CAS实现的。  
CAS和FFA操作存在多步，通过编程语言是无法直接实现原子操作的，而原语是由计算机硬件实现的，所以能够有效的实现无锁的并发操作。  
在JAVA中 CAS和FFA都是通过JNI来实现的native 方法。  
# 使用场景  
CAS和FFA都是无锁的并发操作，那么接下来通过一个账户服务的场景来实现。  
## 实现内容  
存在一个账户余额balance为0，通过多个线程对账户余额进行增加，每次增加1，直到账户余额为10000。  
通过CAS、FFA和锁三种方式实现  
## CAS实现 
  

```java
public class CASTest {

    private static AtomicInteger balance = new AtomicInteger(0);

    @Test
    public void cas(){
        long startTime = System.currentTimeMillis();

        Lock lock = new ReentrantLock();

        for(int i = 0;i < 10000; i ++ ){
            CompletableFuture.runAsync(()->transferByCAS(1));
        }

        while(balance.get() < 10000){
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        long endTime = System.currentTimeMillis();
        System.out.println(endTime - startTime);
        System.out.println(balance);

    }

    public static void transferByCAS(int amount){
        while(true){
            int oldValue = balance.get();
            int newValue = oldValue + amount;
            if (balance.compareAndSet(oldValue, newValue)){
                break;
            }

        }
    }

}

```






执行结果如下。  
![](/images/imageForPost/笔记/CAS/CASResult.png)
多执行几遍，除了运行时间不一样之外，账户余额仍然是被增加到了10000。通过transferByCAS可以看到当准备执行compareAndSet方法时若旧值被其他线程修改后，会放弃本次操作并不断重试，

### FFA实现。
把原代码中的transferByCAS修改为transferByFFA，代码如下  
  

```java
    public static void transferByFFA(int amount){  
        balance.addAndGet(amount);  
    }

```







执行结果
![](/images/imageForPost/笔记/CAS/FFAResult.png)  
可见FFA和CAS一样都是原子操作，不会受多线程并发的影响。  
### 锁实现  
修改原代码为如下    

```java
public class CASTest {

    private static int balance = 0;

    @Test
    public void cas(){
        long startTime = System.currentTimeMillis();

        Lock lock = new ReentrantLock();

        for(int i = 0;i < 10000; i ++ ){
            CompletableFuture.runAsync(()->transferByLock(1, lock));
        }

        while(balance < 10000){
        //while(balance.get() < 10000){
            try {
                Thread.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }

        long endTime = System.currentTimeMillis();
        System.out.println(endTime - startTime);
        System.out.println(balance);

    }




    public static void transferByLock(int amount, Lock lock){
        lock.lock();
        System.out.println("thread " + Thread.currentThread().getId() + " get lock");
        try {
            balance += amount;
        }finally {
            lock.unlock();
        }
    }
}

```




执行结果如下  
在线程获取到锁的时候顺便打印了线程信息，可见CompletableFuture.runAsync()是启动了一个线程池来执行异步任务的。
![](/images/imageForPost/笔记/CAS/LockResult.png)   


## CAS、FFA具体实现  
代码中的CAS和FFA都是通过AtomicInteger类来操作的，查看AtomicInteger的组成可见  
  

```java
       private static final long serialVersionUID = 6214790243416807050L;

    // setup to use Unsafe.compareAndSwapInt for updates
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;

```






对AtomicInteger类对象的操作都是对value进行操作，value是被volatile修饰的，对该值的操作对所有线程可见，所以不会存在脏读的情况。  
### AtomicInteger的compareAndSet(int expect, int update)方法  
跟进该方法可以看到他的实现为    

```java
    public final boolean compareAndSet(int expect, int update) {
        return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
    }

```







unsafe.compareAndSwapInt通过JNI实现的调用。
### AtomicInteger的addAndGet(int delta)  

```java
    public final int addAndGet(int delta) {
        return unsafe.getAndAddInt(this, valueOffset, delta) + delta;
    }


```







继续跟进 **getAndAddInt(Object var1, long var2, int var4)方法可以看到**  
  

```java
    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }

```







最终还是调用到了this.compareAndSwapInt方法，所以在JAVA中CAS和FFA实现都是相同的，FFA相比较于CAS，使用范围更小一点，只能进行增加，而CAS则可以进行其他的逻辑操作。  
虽然CAS和FFA是一种无锁的并发操作，相比较于锁实现可以减少获取锁的开销，但是通过不间断的循环实现操作，在业务并发量大的时候也是有不小的循环开销的。而且CAS和FFA只能保证对一个共享变量的原子操作，如果业务逻辑中需要对多个共享变量进行原子操作则需要使用锁来实现。  
## ABA问题  
如果存在线程1和线程2，共享变量value，值为A。  
当线程1要将value由A变成B时，获取到旧值A，线程2在线程1获取完value旧值后通过CAS操作将value从A变成B，并再次通过CAS操作将value从B变成A，此时线程1开始了它的CAS操作，并且发现value值为A，与旧值相同则可以进行操作。虽然线程1进行CAS比较的时候value值是A，但是不是最开始线程1获取到的值，而是线程2变更之后的值。  
#### 解决方法  
可以通过对变量value增加一个版本号，CAS操作时比较版本号，若版本号相同则进行操作，操作完成后版本号+1，版本号不同则放弃此次操作。JAVA中可以通过AtomicStampedReference来实现版本号的控制。该类的构造函数如下    

```java
    private static class Pair<T> {
        final T reference;
        final int stamp;
        private Pair(T reference, int stamp) {
            this.reference = reference;
            this.stamp = stamp;
        }
        static <T> Pair<T> of(T reference, int stamp) {
            return new Pair<T>(reference, stamp);
        }
    }

```






可见比AtomicInteger的value属性多了个int stamp，即对应的版本号，通过比对这个版本号来进行CAS和FFA操作，版本号一致则修改，不一致则放弃操作并循环重试。


