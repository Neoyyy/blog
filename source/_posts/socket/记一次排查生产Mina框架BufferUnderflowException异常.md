---
title: 记一次排查生产Mina框架BufferUnderflowException异常
date: 2019-10-11 15:10:16
comments: true
categories: 
  - 归纳
tags: 
  - java
toc: true
---
有一个老项目，跟上游系统交互通过Socket使用XML报文，代码中使用Mina作为客户端进行连接。  
在传输带有经过编码成字符串的Photo节点数据时，特定的两个用户数据接收时出现BufferUnderflowException异常导致交易无法继续。  
而在测试环境使用图片更大的数据却没有复现该异常，记录排查过程和错误猜测思路。
<!-- more -->    

# 报错场景和异常内容  
使用Mina作为客户端与上游系统交互，返回报文格式如下图  
![](/images/imageForPost/socket/Mina/报文格式.png)  
特定两位用户的数据返回时出现如下异常    
![](/images/imageForPost/socket/Mina/MinaException1.png)      
![](/images/imageForPost/socket/Mina/MinaException2.jpg)  
   
# 处理方法  
面向搜索引擎编程，第一反应自然是进行如下操作  
![](/images/imageForPost/socket/Mina/面向谷歌编程.png)  
大多数网友的处理方法是手动增大接收缓冲区到报文大小.  
在报错日志中确实是有看到ioBuffer进行自动扩大，因为逻辑中没有指定大小，使用的是默认大小.  
Mina会根据当前包大小来判断调整缓冲区大小，可能扩大也可能缩小，当缩小时来了个大包就会导致异常。  
生产日终中从2048扩大到4096然后出现了异常 
![](/images/imageForPost/socket/Mina/iobuffer扩大.JPG)   
于是在创建Mina Client时指定读缓冲区大小，以业务报文为基准设置个合理的值，单次交互不会存在粘包的问题于是可以设置成大与等于业务报文大小。  
  

```java
        connector.getSessionConfig().setReceiveBufferSize(2048*50);
        connector.getSessionConfig().setReadBufferSize(2048*50);

```
connector为NioSocketConnector，在创建完处理链之后指定了读取缓冲区和接收缓冲区大小。  
喜闻乐见，报文正常接收了。  
# 顺便发现的问题   
## 解码器中长度判断有误    
因为是短连接，所以不存在粘包的问题，异常抛错的栈顶方法是ioBuffer的get方法。  
出现java.nio.BufferUnderflowException的情况为读取长度超过缓冲区可用长度。  
所以需要关注下解码器中的读取逻辑，然后发现解码器中的长度判断逻辑有点小问题。  
在非常极端的情况下会出现BufferUnderflowException异常，不过这不是导致当前生产问题的原因。  
先来关注下这个长度判断导致的问题。   
### ProtocolDecoder代码 
继承了CumulativeProtocolDecoder，CumulativeProtocolDecoder是一种积累型的解码器，只要channel中有数据进来就会读取数据积累到ioBuffer缓冲区中，子类通过实现doDecode方法进行拆包，doDecode只要有数据进来就会被调用。  
doDecode方法    
  

```java  
        private static final int fieldLen = 8;
        protected boolean doDecode(IoSession session, IoBuffer in, ProtocolDecoderOutput out) throws Exception {
             if (in.remaining() > 0) { // 如果有数据，先读取Head中的8字节长度来获取总报文长度
                 byte[] sizeBytes = new byte[fieldLen];
                 in.mark(); //1、标记ioBuffer最开头的位置
                 in.get(sizeBytes);
                 String len = new String(sizeBytes, charset);
                 int packageSize = Integer.valueOf(len);
                 in.reset();// 2、重置到标志位置
                 if (in.remaining() < packageSize) { //3、判断剩余数据长度是不是跟报文体一样长
                     return false;
                 } else {
     
                     byte[] content = new byte[packageSize];
                     in.skip(fieldLen); //4、跳过表示报文头长度的数据
                     in.get(content);  //5、读取报文体
                     String xmlContent = new String(content, charset);
                     EgsLog.getEgsChannelLogNoCache().debug("解包后的核心接口返回字符串(编码字符集" + charset + ")：" + xmlContent);
                     out.write(xmlContent);
     
                 }
                 if (in.remaining() > 0) { // 如果内容后面还粘了包 单次交互不管
                     return true;
                 }
                 return true;
             } else {
                 return false; 
             }
         }

```
## 出现由于长度判断导致问题的场景  
在很极端的情况下，假设报文头长度8字节，报文体长度22字节。  
最开始使用默认大小的缓冲区，
* 假设获取了一段10字节的包   
* 步骤1获取了前面8字节知道了报文总长度22  
* 步骤2将标志重置到起始位置，缓冲区满了，自动扩展到25字节大小  
* 重新执行doDecode方法  
* 步骤2，position重置到头部，因为报文头的长度不包括自身长度，步骤3判断缓冲区有25字节大于报文体22字节，开始读取。  
* 步骤4跳过表示报文长度8字节，此时缓冲区剩余17字节  
* 开始读取22字节的报文，超过缓冲区可读取长度，于是抛出BufferUnderflowException异常  
	
这种情况也只在后面一段包还没接收到，在传输的途中，ioBuffer扩展大小之前进行读取的极端情况下出现。但是生产只要是这两个用户数据必定复现，所以不是这种情况导致的，不过也确实发现了一个隐藏问题。  
既然意外发现了也应该修复。  
### 修改逻辑  
步骤3判断条件修改  


```java
if (in.remaining() < packageSize + fieldLen)
```  
Ctrl C + Ctrl V多了也是有可能出事的。 


# 真正导致生产问题的原因
## 接收缓冲区自动缩小 
翻到一篇场景一样的文章  
[mina read方法出现BufferUnderflowException异常的解决办法](https://my.oschina.net/javagg/blog/2)  
Mina会根据接收到的包大小自动调整缓冲区的大小，在接收TCP包的过程中，前面都是小包，Mina会判断，如果当前包的大小的平方倍小与byteBuf的大小的话，会把缓冲区减半。  
此时来了一个大包，超过了缓冲区大小，get获取就出错了。  
具体可以看*AbstractPollingIoProcessor类的read(T session)*  

```java
private void read(T session) {
        IoSessionConfig config = session.getConfig();

        IoBuffer buf = IoBuffer.allocate(config.getReadBufferSize());

        final boolean hasFragmentation =
            session.getTransportMetadata().hasFragmentation();

        try {
            int readBytes = 0;
            int ret;

            try {
                if (hasFragmentation) {
                    while ((ret = read(session, buf)) > 0) {
                        readBytes += ret;
                        if (!buf.hasRemaining()) {
                            break;
                        }
                    }
                } else {
                    ret = read(session, buf);
                    if (ret > 0) {
                        readBytes = ret;
                    }
                }
            } finally {
                buf.flip();
            }

            if (readBytes > 0) {
                session.getFilterChain().fireMessageReceived(buf);
                buf = null;

                if (hasFragmentation) {
                    if (readBytes << 1 < config.getReadBufferSize()) {
                        session.decreaseReadBufferSize();  //1、buffer减小
                    } else if (readBytes == config.getReadBufferSize()) {
                        session.increaseReadBufferSize(); //2、buffer加大
                    }
                }
            }
            if (ret < 0) {
                scheduleRemove(session);
            }
        } catch (Throwable e) {
            if (e instanceof IOException) {
                scheduleRemove(session);
            }
            session.getFilterChain().fireExceptionCaught(e);
        }
    }
```    
注释1中是缓冲区减小的情况，即判断当前包的平方倍是不是比当前分配的缓冲区小，小的话调用decreaseReadBufferSize()减小缓冲区。  
*decreaseReadBufferSize()方法*  

```java
    public final void decreaseReadBufferSize() {
        if (this.deferDecreaseReadBuffer) {
            this.deferDecreaseReadBuffer = false;
        } else {
            if (this.getConfig().getReadBufferSize() > this.getConfig().getMinReadBufferSize()) {
                this.getConfig().setReadBufferSize(this.getConfig().getReadBufferSize() >>> 1);
            }

            this.deferDecreaseReadBuffer = true;
        }
    }
```   
当缓冲区减小后，发送端发来一个大于缓冲区大小的包，就会导致BufferUnderflowException异常。  
## 总结  
对于单次交互的短连接，由于不存在粘包的情况，所以可以将客户端的接收缓冲区根据实际业务报文大小适当调大，可以大于等于业务最大报文，一般缓冲区设置较小为了减少开销，而实际业务的报文大小一般最多十几到几十kb。
