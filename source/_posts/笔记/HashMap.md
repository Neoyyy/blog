---
title: HashMap 
date: 2019-10-28 21:02:16
comments: true
categories: 
- 笔记
tags: 
- java
---

HashMap主要用来存放键值对，在JDK1.8之前，HashMap的结构有数组+链表组成，数组是HashMap的主体，链表则是用来处理哈希冲突时存放数据用的，在JDK1.8后，当出现哈希冲突的时候，即当链表长度大于阈值（一般是8），会把链表转换成红黑树来存储，以减少节点的搜索时间。

<!-- more -->

## HashMap的结构  

### JDK1.8之前    

JDK1.8之前的HashMap的存储结构由数组和链表组成。  

通过key的hash值经过扰动函数处理后得到该key对应的数据在链表中的存储位置，即（n-1）&hash，n为数组长度，如果得到的位置已经存在数据，则比较hash值和key值，如果相同则覆盖不相同则挂在链表上。 
### JDK1.8之后    

基础结构跟JDK1.8之前一样，都是有数组+链表组成，当链表下挂的节点多于阈值（一般为8）后，会把链表转换成红黑树来存储，便于加快查找速度。  
如果没有转换成红黑树的花，假设有一组数据hash值一样，那么便会不断的下挂在同一个数组节点的链表上，此时HashMap变成了线性的，查找效率会变得很慢。    
JDK1.8之前的hash方法：  

```java
    static int hash(int var0) {
        var0 ^= var0 >>> 20 ^ var0 >>> 12;
        return var0 ^ var0 >>> 7 ^ var0 >>> 4;
    }


```

JDK1.8之后的hash方法：  

```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }



```
可见JDK1.8之后的HashMap，若存入一个key为null值，他的hash值为0，所以HashMap只能存储一个key为null值的元素，再看JDK1.8之前若存入一个key为null的元素，  

```java
    private V putForNullKey(V var1) {
        for(HashMap.Entry var2 = this.table[0]; var2 != null; var2 = var2.next) {
            if (var2.key == null) {
                Object var3 = var2.value;
                var2.value = var1;
                var2.recordAccess(this);
                return var3;
            }
        }

        ++this.modCount;
        this.addEntry(0, (Object)null, var1, 0);
        return null;
    }


```  
### 类的属性  

```java
 // 序列号
    private static final long serialVersionUID = 362498820763181265L;    
    // 默认的初始容量是16
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;   
    // 最大容量
    static final int MAXIMUM_CAPACITY = 1 << 30; 
    // 默认的填充因子
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    // 当桶(bucket)上的结点数大于这个值时会转成红黑树
    static final int TREEIFY_THRESHOLD = 8; 
    // 当桶(bucket)上的结点数小于这个值时树转链表
    static final int UNTREEIFY_THRESHOLD = 6;
    // 桶中结构转化为红黑树对应的table的最小大小
    static final int MIN_TREEIFY_CAPACITY = 64;
    // 存储元素的数组，总是2的幂次倍
    transient Node<k,v>[] table; 
    // 存放具体元素的集
    transient Set<map.entry<k,v>> entrySet;
    // 存放元素的个数，注意这个不等于数组的长度。
    transient int size;
    // 每次扩容和更改map结构的计数器
    transient int modCount;   
    // 临界值 当实际大小(容量*填充因子)超过临界值时，会进行扩容
    int threshold;
    // 加载因子
    final float loadFactor;


```  
* loadFactor  
	加载因子，默认值为0.75，当数组占用大于等于capacity*loadFactor时，数组需要进行扩容，默认capacity大小为16，即当数组占用大于等于12时要进行扩容，也就是resize方法。  
	
### resize扩容时的rehash操作  
当数组需要进行扩容时会重新计算hash并分配位置，该操作也就是rehash，对hash表中的所有数据进行操作，所以会比较耗时。   
#### JDK1.8之前的resize实现   

* 扩容时机    

```java
    void addEntry(int var1, K var2, V var3, int var4) {
        HashMap.Entry var5 = this.table[var4];
        this.table[var4] = new HashMap.Entry(var1, var2, var3, var5);
        if (this.size++ >= this.threshold) {//当数组占用大小大于等于前面说的capacity*loadFactor时会进行扩容
            this.resize(2 * this.table.length);
        }

    }


```   

* 扩容操作   
 
	
```java
    void resize(int var1) {
        HashMap.Entry[] var2 = this.table;
        int var3 = var2.length;
        if (var3 == 1073741824) {
            this.threshold = 2147483647; //达到最大值后就不会再扩张了
        } else {
            HashMap.Entry[] var4 = new HashMap.Entry[var1];//新生成一个两倍大小的数组
            this.transfer(var4);//把原数组和对应的链表都搬移过去
            this.table = var4;
            this.threshold = (int)((float)var1 * this.loadFactor);
        }
    }


```     

* 搬移操作   
 

```java
   void transfer(HashMap.Entry[] var1) {
        HashMap.Entry[] var2 = this.table;
        int var3 = var1.length;

        for(int var4 = 0; var4 < var2.length; ++var4) {//遍历搬运数组上的节点
            HashMap.Entry var5 = var2[var4];
            if (var5 != null) {
                var2[var4] = null;

                HashMap.Entry var6;//遍历搬运每个数组节点上的链表数据节点
                do {
                    var6 = var5.next;
                    int var7 = indexFor(var5.hash, var3);
                    var5.next = var1[var7];
                    var1[var7] = var5;
                    var5 = var6;
                } while(var6 != null);
            }
        }

    }



```    

每次resize操作时会搬运所有的Hash表数据，操作比较耗时。  
由于操作中代码都不是同步的，所以HashMap其实不是线程安全的，如果需要保证线程安全可以使用ConcurrentHashMap，还有一个HashTable也是线程安全的。 不过HashMap的key可以为null，而这两个则不可以。   
## HashTable  
HashTable的get和set方法都加上了synchronized同步锁，每次调用get或者set都要加锁，会把整个表都锁住，所以性能损耗比较大。  
## ConcurrentHashMap  
ConcurrentHashMap跟HashTable不同，不是锁住整个表，ConcurrentHashMap增加了一个Segment结构，在每个Segment结构后面才是数组，再然后下挂链表，所以并发时锁住的是Segment对象而不是整个表，锁粒度比HashTable小的多。  
