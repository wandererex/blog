---
title: netty源码解读-新连接接入
date: 2020-10-17 17:54:17
tags: netty
---
## 前言
前面分析了netty服务端启动的过程，看到首先创建serversocketchannel，再注册到eventloop的selector上，当注册完毕，会bind端口，然后触发active事件，关注op_accept事件。这样就可以接受新连接了，在这篇博客，我分析一下新连接接入的过程。
## debug过程
首先在NioEventLoop的run方法打一个断点
[netty](netty源码解读-新连接接入/image0.png)
然后telnet一个新连接，进入processSelectedKeysOptimized方法
[netty](netty源码解读-新连接接入/image1.png)
进入processSelectedKey方法，处理accept事件
[netty](netty源码解读-新连接接入/image2.png)
进入read方法，调用doReadMessages拿新连接
[netty](netty源码解读-新连接接入/image3.png)
然后将拿到的连接发送出去
[netty](netty源码解读-新连接接入/image4.png)
继续跟，进入ServerBootstrapAcceptor的channelRead方法，正式配置新接入的连接
[netty](netty源码解读-新连接接入/image5.png)
主要做这几件事
1. 添加childHandler
2. 设置childOptions
3. 设置childAttrs
4. 将拿到的连接channel注册到childeventgroup
注册与服务端启动的过程相似，主要有以下区别
当channel注册到selector上后就已经active了，进入以下逻辑
[netty](netty源码解读-新连接接入/image6.png)
然后和服务端启动一样，进入doBeginRead方法，关注op_read事件
[netty](netty源码解读-新连接接入/image7.png)
## 总结
关于新连接接入，其实和服务端启动很像。首先在eventloop轮询selector拿到selectedkeys，然后调用processSelectedKey处理，拿到channel后，要回调之前在服务端启动时设置的ServerBootstrapAcceptor，配置新连接，然后注册到worker group上，把op_read作为感兴趣事件放入selectedkey中。
