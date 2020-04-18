---
title: Netty入门浅析(1)
date: 2018-07-22 12:36:12
categories: 
- 后端
tags:
- Netty
- Java
- 分布式
---

# Netty的简单介绍

Netty 是一个 NIO client-server(客户端服务器)框架，使用 Netty 可以快速开发网络应用，例如服务器和客户 端协议。 Netty 提供了一种新的方式来使开发网络应用程序，这种新的方式使得它很容易使用和有很强的扩展性。 Netty 的内部实现时很复杂的，但是 Netty 提供了简单易用的 api 从网络处理代码中解耦业务逻辑。 Netty 是完全基 于 NIO 实现的，所以整个 Netty 都是异步的。 简单点说就是Netty提供了一个简单，间接的方法来操作网络之间的通讯。

# 使用 Netty 能够做什么

- 开发异步、非阻塞的 TCP 网络应用程序；
- 开发异步、非阻塞的 UDP 网络应用程序；
- 开发异步文件传输应用程序；
- 开发异步 HTTP 服务端和客户端应用程序；
- 提供对多种编解码框架的集成，包括谷歌的 Protobuf、Jbossmarshalling、Java 序列化、压缩编解码、XML 解码、字符串编解码等，这些编解码框架可以被用户直接使用；提供形式多样的编解码基础类库，可以非常方便的实现私有协议栈编解码框架的二次定制和开发；
- 基于职责链模式的 Pipeline-Handler 机制，用户可以非常方便的对网络事件进行拦截和定制；所有的 IO 操作都是异步的，用户可以通过 Future-Listener 机制主动 Get 结果或者由 IO 线程操作完成之后主动 Notify 结果，用户的业务线程不需要同步等待；
- IP 黑白名单控制；
- 打印消息码流；流量控制和整形；
- 性能统计；基于链路空闲事件检测的心跳检测  
- ……

# Netty得到的应用

- 阿里分布式服务框架 Dubbo 的 RPC 框架使用 Dubbo 协议进行节点间通信，Dubbo 协议默认使用 Netty 作为基础通信组件，用于实现各进程节点之间的内部通信。
- 淘宝的消息中间件 RocketMQ 的消息生产者和消息消费者之间，也采用 Netty 进行高性能、异步通信。

# Netty Reactor模式

Netty是典型的Reactor模型结构。Netty通过Reactor模型基于多路复用器接收并处理用户请求，内部实现了两个线程池，Boss线程池和Work线程池，其中Boss线程池的线程负责处理请求的accept事件，当接收到accept事件的请求时，把对应的socket封装到一个NioSocketChannel中，并交给Work线程池，其中Work线程池负责请求的read和write事件。
**流程图：**

{% asset_img 1.png 图片 %}

# Netty核心组件

## Channel

这里的Channel与Java的Channel不是同一个，是netty自己定义的通道；Netty的Channel是对网络连接处理的抽象，负责与网络进行通讯，支持NIO和OIO两种方式；内部与网络socket连接，通过channel能够进行I/O操作，如读、写、连接和绑定。 
通过Channel可以执行具体的I/O操作，如read, write, connect, 和bind。在Netty中，所有I/O操作都是异步的；Netty的服务器端处理客户端连接的Channel创建时可以设置父Channel。例如：ServerSocketChannel接收到请求创建SocketChannel，SocketChannel的父为ServerSocketChannel。

## ChannelHandler与ChannelPipeline

ChannelHandler是通道处理器，用来处理I/O事件或拦截I/O操作，ChannelPipeline字如其名，是一个双向流水线，内部维护了多个ChannelHandler，服务器端收到I/O事件后，每次顺着ChannelPipeline依次调用ChannelHandler的相关方法。 
ChannelHandler是个接口，通常我们在Netty中需要使用下面的子类：
  - ChannelInboundHandler 用来处理输入的I/O事件
  - ChannelOutboundHandler 用来处理输出的I/O事件


另外，下面的adapter类提供了:
  - ChannelInboundHandlerAdapter 用来处理输入的I/O事件
  - ChannelOutboundHandlerAdapter 用来处理输出的I/O事件
  - ChannelDuplexHandler 可以用来处理输入和输出的I/O事件


Netty的ChannelPipeline和ChannelHandler机制类似于Servlet和Filter过滤器/拦截器，每次收到请求会依次调用配置好的拦截器链。Netty服务器收到消息后，将消息在ChannelPipeline中流动和传递，途经的ChannelHandler会对消息进行处理，ChannelHandler分为两种inbound和outbound，服务器read过程中只会调用inbound的方法，write时只寻找链中的outbound的Handler。
ChannelPipeline内部维护了一个双向链表，Head和Tail分别代表表头和表尾，Head作为总入口和总出口，负责底层的网络读写操作；用户自己定义的ChannelHandler会被添加到链表中，这样就可以对I/O事件进行拦截和处理；这样的好处在于用户可以方便的通过新增和删除链表中的ChannelHandler来实现不同的业务逻辑，不需要对已有的ChannelHandler进行修改

{% asset_img 2.png 图片 %}

如图所示，在服务器初始化后，ServerSocketChannel的会创建一个Pipeline，内部维护了ChannelHanlder的双向链表，读取数据时，会依次调用ChannelInboundHandler子类的channelRead()方法，例如：读取到客户端数据后，依次调用解码-业务逻辑-直到Tail。而写入数据时，会从用户自定义的ChannelHandler出发查找ChannelOutboundHandler的子类，调用channelWrite()，最终由Head的write()向socket写入数据。例如：写入数据会通过业务逻辑的组装–编码–写入socket（Head的write）

## EventLoop与EventLoopGroup

EventLoop是事件循环，EventLoopGroup是运行在线程池中的事件循环组，Netty使用了Reactor模型，服务器的连接和读写放在线程池之上的事件循环中执行，这是Netty获得高性能的原因之一。事件循环内部会打开selector，并将Channel注册到事件循环中，事件循环不断的进行select()查找准备就绪的描述符；此外，某些系统任务也会被提交到事件循环组中运行。

## ServerBootstrap

ServerBootstrap是辅助启动类，用于服务端的启动，内部维护了很多用于启动和建立连接的属性。包括：

- EventLoopGroup group 线程池组
- channel是通道
- channelFactory 通道工厂，用于创建channel
- localAddress 本地地址
- options 通道的选项，主要是TCP连接的属性
- attrs 用来设置channel的属性，
- handler 通道处理器

# 参考

[Netty原理浅析](http://blog.leanote.com/post/shawn_yan/Netty%E5%8E%9F%E7%90%86%E6%A6%82%E8%A7%88)

