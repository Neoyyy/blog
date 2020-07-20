---
title: nginx代理配置
date: 2020-07-20 13:39:04
comments: true
categories: 
- 记录
tags: 
- nginx
---



本文主要涉及nginx的安装、正向代理、反向代理、负载均衡和对应的配置  

<!-- more -->

## nginx的安装及配置文件

**nginx是什么  **  

> nginx是一个高性能的HTTP和反向代理web服务器，同时也提供了IMAP/POP3/SMTP服务  

### nginx安装

要操作nginx的正向代理、反向代理和负载均衡之前首先要安装一个nginx，以CentOS Linux release 7.3.1611 简单安装为例子  

1. 下载nginx安装包     

```shell
wget http://nginx.org/download/nginx-1.8.1.tar.gz  
```

![downNginx](/images/imageForPost/nginx/nginxFirstUse/downNginx.png)

2. 安装nginx依赖

```shell
yum -y install gcc zlib zlib-devel pcre-devel openssl openssl-devel 
```

3. 解压nginx包

```shell
tar -xvf nginx-1.8.1.tar.gz  
```

4. 执行如下命令进行编译。

```shell
cd nginx-1.8.1
./configure
make
make install
```

5. 安装完成

![finishInstall](/images/imageForPost/nginx/nginxFirstUse/finishInstall.png)

### nginx配置文件

​		当安装完nginx后，在conf文件夹下的nginx.conf文件就是nginx的默认配置文件，配置文件的注释以#开头

主要构成有三个模块：  

- 全局模块：  

  ​	在配置文件开始到events 块之前的内容，该块可以配置启动nginx所使用的用户、错误日志文件存放地址和work进程数量等![nginxGlobalConf](/images/imageForPost/nginx/nginxFirstUse/nginxGlobalConf.png)

- events模块：  

  ​	events块主要配置nginx的网络连接，比如将每个work进程最大可处理连接数量worker_connections设置为1024个  

- http模块：  

  ​	nginx作为服务器主要是定义如何处理请求的URL，在http模块中通过定义一系列的虚拟服务器控制来处理对应的请求 。在http块中对监听端口，请求转发，代理等进行配置  

---

在http配置块中还有server块和location块，其中location块在server块中。  

```json
http{
	server{
		location{
		
		}
	}
}
```

**location**  

​		定义用于匹配处理请求的路径，当nginx处理一个请求时会根据请求中的URL与location定义的路径进行匹配，当匹配上后则通过location中的定义对请求进行处理  

```json
location /test {
	root /tetstdata
}
```

当请求的URl匹配上/test后则会通过这个location进行处理

### nginx进程模型

- nginx启动后会存有一个master主进程以及在配置文件全局块中配置的对应数量的work进程，master进程管理work进程，用于接收命令，监控、重启work进程。  

- work进程处理外界请求    


![nginxWorkModel](/images/imageForPost/nginx/nginxFirstUse/nginxWorkModel.png)



nginx启动后可用如下常用命令来控制：

- nginx -s stop 快速关闭

- nginx -s reload 重新加载配置文件

- nginx -s quit 优雅的关闭    

---


在reload这个重新加载配置文件的命令中，master进程接收到命令会进行如下操作。

1. master进程对配置文件进行检查

2. 尝试根据配置文件进行配置

3. 给现存的work进行发送关闭的消息，现存的work进程会把当前处理的请求处理完成后才会退出

4. 根据配置文件新建对应的work进程  

所以在reload命令执行后work进程的id是会发生变化的，**查看nginx进程PID**如下所示  

![workPID-1](/images/imageForPost/nginx/nginxFirstUse/workPID-1.png)

**执行重载配置文件命令**  

```shell
nginx -s reload  
```

![workPID-2](/images/imageForPost/nginx/nginxFirstUse/workPID-2.png)


## nginx正向代理、反向代理和负载均衡

### nginx正向代理 

 nginx可作为正向代理服务器使用，正向代理的目的就是nginx接收到客户端的请求后，转发该请求给目标服务器，并将目标服务器的响应发送回客户端。  

 在该过程中，客户端知道目标服务器的ip等信息，只不过请求不是由客户端直接发往服务器而是中间通过了nginx这个代理服务器。  

 常见的应用比如由于某些原因无法访问一些国际网站，那么可以通过一台能够访问国际网站的机器作为代理服务器转发请求。

![proxy-1](/images/imageForPost/nginx/nginxFirstUse/proxy-1.png)

**举个例子**:  		

 本机网络配置代理为nginx，访问一个ip查询网站，网站显示在代理前为本机所处网络的公网IP，代理后显示为nginx所处的网络的公网IP。

**nginx配置**

 resolver配置为google的dns服务器，location中的proxy_pass中都是nginx的变量，代表了请求中的ip等参数。



![nginxProxy-1](/images/imageForPost/nginx/nginxFirstUse/nginxProxy-1.png)



**本机浏览器配置**

 Web Proxy Server配置为nginx所在机器的ip以及端口

![macProxySetting](/images/imageForPost/nginx/nginxFirstUse/macProxySetting.png)


点击[查看IP](http://ip.tool.chinaz.com/)查看当前请求的IP   

---

在启用代理前访问

![myIPReal](/images/imageForPost/nginx/nginxFirstUse/myIPReal.png)



启用代理后访问

![myIPProxy](/images/imageForPost/nginx/nginxFirstUse/myIPProxy.png)  

  




### nginx反向代理

>与正向代理不同的是反向代理的服务器IP等信息对客户端透明，客户端只知道>nginx的信息，将请求发送到nginx，由nginx进行转发处理，对客户端来说他只>与nginx进行交互，在nginx背后有多少服务器均不清楚。

**反向代理**

![proxy-2](/images/imageForPost/nginx/nginxFirstUse/proxy-2.png)

**反向代理负载**
![proxy-3](/images/imageForPost/nginx/nginxFirstUse/proxy-3.png)

- 在前后端分离的项目中，由于浏览器的安全控制，会出现需要解决跨域的情况，这时就可以通过nginx配置反向代理来避免出现跨域请求的情况。

- 在应用服务中，通过nginx配置反向代理，只对外暴露出nginx服务来保证内网机器的安全      

    

  ---
**举个例子**

**配置反向代理**

1. 实现nginx反向代理2台tomcat服务
		
		​ 启动2个tomcat应用，分别监听8080和8081端口，修改webapps/ROOT/index.jsp文件来标识tomcat1和tomcat2



2. nginx配置

		​ 因为要代理两台tomcat，通过配置两个location来指向不同的toncat，通过url中的/a和/b来匹配到对应的tomcat。

![proxySetting-2-true](/images/imageForPost/nginx/nginxFirstUse/proxySetting-2-true.png)



此时请求nginx所在机器的IP:端口/a会请求到监听在8080的tomcat1

![firstTomcatPage](/images/imageForPost/nginx/nginxFirstUse/firstTomcatPage.png)

  

此时请求nginx所在机器的IP:端口/b会请求到监听在8081的tomcat2

![secondTomcatPage](/images/imageForPost/nginx/nginxFirstUse/secondTomcatPage.png)  


---


### nginx负载

在前面反向代理两台tomcat的基础上修改成nginx的负载均衡设置   


**nginx设置**

新建一个upstream配置，定义用来处理请求的服务器

![loadBalanceSetting](/images/imageForPost/nginx/nginxFirstUse/loadBalanceSetting.png)

此时访问nginx所在的ip:端口，则会随机在tomcat1和tomcat2中轮询。  

**nginx的负载均衡策略**

1. 轮询 

默认策略，根据请求到来的先后在服务器中轮询

```shell
upstream lagouServer{ 
  server 127.0.0.1:8080; 
  server 127.0.0.1:8081; 
}
```

2. 权重

weight代表负载的权重，默认每一个机器的权重为1，根据设置的权重，权重越高被分配到处理请求的几率越高

```shell
upstream lagouServer{ 
  server 127.0.0.1:8080 weight=1; 
  server 127.0.0.1:8081 weight=2; 
}
```
3. ip_hash    

请求根据ip的hash结果来进行分配，所以每个客户端都有固定的服务器进行处理，可用于session未做分布式共享的场景    

```shell
upstream lagouServer{ 
	ip_hash;
  server 127.0.0.1:8080; 
  server 127.0.0.1:8081; 
}
```







