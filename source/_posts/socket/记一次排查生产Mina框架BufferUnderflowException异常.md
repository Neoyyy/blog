---
title: 使用Netty实现简单的RPC调用
date: 2019-10-08 20:10:16
comments: true
categories: 
  - 归纳
tags: 
  - java
toc: true
---
本文主要通过Netty实现一个简单的rpc调用。  
之前项目中用来socket对接的代码都是一样可以复用的，所以每次都是ctrl c + ctrl v，  
没有自己从头用Netty写过一个连接，借此来简单的使用下Netty  
<!-- more -->    

# 主要内容  
* 实现简单的rpc需要什么内容  
* 如何通过Netty实现  

# Netty核心组件  
* Bootstrap、ServerBootstrap  
* Channel  
* ChannelHanlder  
* ChannelPipeline  
* EventLoop  
* ChannelFuture  
