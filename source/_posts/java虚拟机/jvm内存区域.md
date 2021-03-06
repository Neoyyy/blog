---
title: JVM内存区域和GC类型
date: 2019-10-15 22:02:16
comments: true
categories: 
- 笔记
tags: 
- java
---

简单记录JVM的内存划分以及几种GC收集器  
<!-- more -->  
## 主要区域  

* 线程公有部分
	* 堆 heap  
		用来存放对象实例。GC的主要区域，采用分代回收的gc算法的话还可以细分成 Eden，From Survivor(S0)，To Survivor(S1)，Tentired。其中Eden，S0，S1属于新生代，Tentired属于老年代，当对象在 Eden经过一次GC后对象年龄+1并且进入S0，S1区，当他的年龄增加到一定程度会进入Tentired老年区，或者s区域相同年龄的对象所占空间大于s总空间的一半，那对象大于这个年龄的都会进入老年代。    
	* 方法区  
		用于存储被虚拟机加载的类信息，常量，静态变量，即时编译后的代码数据等。

	* 运行时常量池
		方法区的一部分，也是线程共享的。

* 线程私有部分  
	* 虚拟机栈  
		每个方法调用都是通过栈传递的，由栈帧组成，每个栈帧都包含了：1、局部变量表 2、操作数栈 3、动态链表 4、方法出口信息等。  
		每次函数调用都会有一个栈帧入栈，调用结束后会有个栈帧出栈，抛出异常也会出栈。  
	* 本地方法栈  
		调用native方法也会创建栈帧入本地方法栈，HotSpot虚拟机中本地方法栈和虚拟机栈是同一个。  
	* 程序计数器
		当前程序执行的行号指示器，可以保证线程切换后能恢复到原来的执行位置。通过程序计数器来获取下一步要执行的字节码指令。  
* 直接内存
	* 元空间
	在jdk1.8之后移除了方法区，原方法区的内容移动到了直接内存中的元空间内  
	
	  
## HotSpot    

### 对象的创建过程    
 
* 类加载检查 
* 分配内存 
* 初始化零值 
* 设置对象头 
* 执行init方法   

### 类加载检查  
  当虚拟机执行到new指令时，会检查是否能在常量池中找到这个类的符号引用，并且检查该类是否被加载、解析和初始化过。若没有则进行对应的类加载操作。  

### 分配内存  
  根据内存是否规整分为：1、指针碰撞 2、空闲列表 两种方式    

  * 指针碰撞  
  	在内存规整的情况下被使用的内存在一则，未被使用的内存在另一侧，中间存在一个指针，当需要分配时指针往未被使用的一则移动所需空间即可。  
  * 空闲列表  
  	内存分配不连续，虚拟机维护一份内存使用列表，当需要分配时查找列表找到一块合适的内存进行分配后更新该列表  
在创建对象时需要保证线程安全，虚拟机通过：1、CAS 2、TLAB 两种方式来保证线程安全
	* CAS  
	CAS是一种乐观锁的实现，每次不加锁且假设没有冲突去操作，若因为冲突而导致失败则不断重试直到成功  
	* TLAB  
	预先给线程在Eden区分配一块内存，当需要分配时在这块内存中进行分配，若该块内存空间不够则进行CAS方式分配  
	
	  
### 初始化零值  
   为了保证不赋初始值就能使用需要将对应的数据设置零值  
     
### 设置对象头  
   虚拟机需要在零值初始化完成后把对象的一些信息存放到对象头中，包括 这个对象是哪个类的实例，如何找到类的元数据信息，对象的哈希值，GC的分年代信息等  
     
### 直销init方法  
   根据代码中的意愿初始化对象  
     
## GC部分  
垃圾回收算法主要有：
* 标记-清除算法 
* 复制算法
* 标记-整理算法
* 分代收集算法   

### 标记-清除算法  
垃圾收集器会标记需要清除的对象，然后清除他们。不过会导致回收后内存有大量不连续碎片。  
### 复制算法  
将内存分为两块，将需要保留的对象复制到一侧，回收另一侧内存。缺点是内存使用率低，会空置一半的内存。  
### 标记-整理算法  
将需要回收的对象进行标记，然后把不回收的对象移动到一起，回收边界以外的内存。  
### 分代收集算法  
根据对象的存活周期，将内存分为新生代和老年代，然后在各个年代采用前面的算法。  
	
## 垃圾收集器  

垃圾收集器主要有：
* Serial收集器   
* ParNew收集器 
* Parallel Scavenge收集器 
* CMS收集器 
* G1收集器  
### Serial收集器
单线程收集器，执行GC时会stop the world，对新生代使用复制算法，对老年代使用标记-整理算法。  
### ParNew收集器  
Serial收集器的多线程版本。  
### Parallel Scavenge收集器  
与ParNew收集器类似，提供很多参数供用户找到最合适的停顿时间或最大吞吐量。   
### CMS收集器  
并发收集器，收集线程与用户线程基本上一同工作。主要步骤有：1、初始标记 2、并发标记 3、重新标记 4、并发清除  
### G1收集器  
使用服务器的多处理器和大内存优势的收集器，1、初始标记 2、并发标记 3、最终标记 4、筛选回收

## 类加载  
类加载过程主要分为：
* 加载 
* 连接 
* 初始化  
其中**连接**又分为：
* 验证 
* 准备 
* 解析三步  
### 加载  
1、通过类的完全限定名获取到类文件的二进制字节流，具体获取方式不做限制。   
2、将字节流所代表的静态存储结构转换为方法区的运行时数据。 
3、在内存中生成一个代表该类的class对象
	
### 连接部分  
* 验证  
	1、验证文件格式是否符合要求  
	2、元数据验证  
	3、字节码验证  
	4、符号引用验证  
* 准备  
	该阶段为类变量分配内存并初始化零值  
* 解析  
	将常量池内的符号引用替换为直接引用  
### 初始化  
执行类代码中的初始化逻辑

