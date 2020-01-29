---
title: Too Many Files异常处理和Ulimit命令  
date: 2020-01-17 22:02:16  
comments: true  
categories:   
- 记录  
tags:   
- Linux  

---
前天生产环境网关应用报了一个异常，导致网关服务不可用，用户无法登陆。  
查看了下日志，报了**Too Many Files**异常   
<!-- more -->  
**异常日志：**
![](/images/imageForPost/Linux/openTooManyFiles/prod-err.png)  
这种异常也较为常见，当java应用在打开超过系统允许它打开的最大文件数目时会报这个异常。  
之前在测试环境中出现过一次，简单粗暴的重启了测试机器就解决了。   
但是生产环境不能像测试环境一样暴力，于是通过ulimit命令来调整应用可以打开文件的上限来处理。    

### 出现异常的原因  
在Linux系统中有句话叫 万物皆为对象，生产环境中通过socket对接上游系统.  
某天上游系统不知道啥原因挂了，接收了socket请求但是不做任何操作，加上不断有用户请求进来，  
应用不断的与该系统创建连接，每打开一个socket连接对Linux来说就是我们的应用打开了一个文件，不断累加，于是超过了最大打开文件的限制，由于应用再也不能创建、接收socket连接，导致用户端出现了异常。    
##### 查看当前应用可打开的文件数  
通过命令查看当前线程打开的文件数  
`lsof -p PID ｜ wc -l   `

**PID**为应用线程id
在本机通过命令 umilits -a 查询结果如下。  
![](/images/imageForPost/Linux/openTooManyFiles/ulimit-a.png)
 其中open file就是允许打开的文件数。  
 不过不加参数的话默认的时-S即显示soft的限制，通过ulimit设置的话没有指定-S或者-H则是同时设置。    
 ![](/images/imageForPost/Linux/openTooManyFiles/ulimit-n.png)  
 ### soft和hard区别  
 linux对打开文件限制有soft和hard限制。
 
 ### hard和soft限制
 hard和soft都是对资源的限制，对打开的文件数限制来说  
 * hard表示打开的最大数目，soft是警告数目，超过soft会出现警告  
 * soft小于等于hard  
 * root进程可以调整hard值，非root进程只能调整soft值  
 * 进程在开启前可以自主调整soft的值。    
 
### 处理方式  
* **ulimit命令调整**
* **修改limits.conf文件**  
#### ulimit命令
之前测试环境出现这种情况直接简单粗暴的重启了机器，生产环境可不能这么干，于是可以通过ulimit命令增加nofile值  
nofile就是进程可打开的文件数。  
![](/images/imageForPost/Linux/openTooManyFiles/ulimit-adj.png)

进程的ulimit修改对当前shell的进程生效，但是已经启动的进程不受影线，而且单次生效。  
如果要保证应用每次启动的时候都生效则可以在应用的启动脚本里设置。
#### 修改limits.conf文件 
除了手动通过当前命令修改还可以对limits.conf文件新增记录来修改某用户进程可控制的资源数目
文件路径。
**/etc/security/limits.conf**  
举例现有文件中的配置
![](/images/imageForPost/Linux/openTooManyFiles/security-limits.png)

* 第一个为范围，可为用户名、group或者用*表示所有的非root用户
* 第二个字段可以为hard soft或者-，-表示hard和soft
* 第三个字段表示控制什么类型，比如nofile代表可打开的文件数目  
* 第四个字段表示具体的值，1000表示nofile可为10000  

---------
通过查看/proc/PID/limits文件可以查看某个已经运行的进程的资源最大值，PID为进程ID。比如现在随机找了个进程查看
![](/images/imageForPost/Linux/openTooManyFiles/proc-limits.png)
### 复现生产问题
生产环境是因为socket打开过多导致的，那手动模拟一下多开许多文件也是一样的   

注：直接在Mac机器上进行测试，都是类Unix系统
#### 查看当前shell允许的最大打开文件数量。  
当前允许最大打开文件数目为1000
![](/images/imageForPost/Linux/openTooManyFiles/ulimit-a-max.png)

##### 打开文件代码  

```java  
package test;

import java.io.File;
import java.io.FileInputStream;
import java.util.ArrayList;
import java.util.List;
import java.io.IOException;
import java.lang.InterruptedException;
class Ulimit{
    public static void main( String[] args ) throws IOException, InterruptedException
    {
                List<FileInputStream> fileList = new ArrayList<FileInputStream>();
        for(int i=0;i<700;i++) {
            File temp = File.createTempFile("test", ".txt");
            fileList.add(new FileInputStream(temp));
            System.out.println("file:" + i + " " + temp.getAbsolutePath());
        }
    }
}


```

由于jvm也是需要打开一些文件的，所以把循环值设置的比允许值更小一点，代码如下。设置为700

目前允许值为1000，那么程序打开700个文件肯定是可以的

##### 编译代码。  
`javac test.java  `
会生成Ulimit.class

##### 在当前shell中执行代码  
由于调整的是当前shell产生的进程，所以编译后执行该class，由于class包名为test，把class文件放入test文件夹后执行  
`java test.java`
![](/images/imageForPost/Linux/openTooManyFiles/run-command.png)
##### 正常执行  
![](/images/imageForPost/Linux/openTooManyFiles/run-succ.png)
##### 调整open file值
通过`ulimit -n 700`改变当前shell的允许值，然后重新执行代码  

![](/images/imageForPost/Linux/openTooManyFiles/ulimit-a-mac.png)

重新运行，报错。  
![](/images/imageForPost/Linux/openTooManyFiles/run-error.png)


 
 
 不过在mac中通过ulimit -n 1100 调大时报错。
 ![](/images/imageForPost/Linux/openTooManyFiles/mac-set-error.png)
 
 在mac中应该使用
 sudo launchctl limit maxfiles 100000 500000  
 ![](/images/imageForPost/Linux/openTooManyFiles/mac-sysctl-set.png)
 进行设置，由于手滑，后面的参数值设置的小了  
 然后应用动不动报too many files error  
 结果导致无法正常关闭应用和重启，只得强制重启，所以命令要想好了再执行。。。 
 