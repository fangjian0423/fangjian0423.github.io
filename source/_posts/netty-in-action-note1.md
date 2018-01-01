title: Netty in Action笔记(一)
date: 2016-08-19 01:36:52
tags:
- nio
- netty
categories: netty

----------------

一篇读书笔记，根据章节进行下总结。已经看了5章，先对这5个章节做个小总结。
	
<!--more-->


## 第一章

然后介绍了一下Netty出来的背景以及Netty所拥有的一些强大的特性，比如Netty的设计高可用，可扩展，性能高，使用NIO这种非阻塞式的IO模型等特点。

然后介绍了一下异步编程的设计，有两种方式：

1. 基于Callback
2. 基于Future

基于Callback的跟javascript相似，这种方式比较大的问题就是当callback多的时候，代码就变得难以阅读。

基于Future的方式就是jdk里的concurrent包里的Future接口一样，代表着未来的一个值。

Netty内部这2种异步编程方式都有使用。

之后对IO阻塞模型和NIO非阻塞模型进行了一番比较。

其中IO阻塞模型对于每一个Socket都会创建一个线程进行处理，虽然可以使用线程池解决线程过多的问题，但是底层还是使用线程处理每一个请求，系统的瓶颈在于线程的个数，并且线程多了会导致频繁的线程切换，导致CPU利用效率不高。

NIO非阻塞模型采用Reactor模式，一个Reactor线程内部使用一个多路复用器selector，这个selector可以同时注册、监听和轮询成百上千个Channel，一个IO线程可以同时并发处理多个客户端连接，就不会存在频繁的IO线程之间的切换，CPU利用率高。

之后使用JDK的NIO Api编写了一个demo，发现JDK的NIO Api比较难用，比如在ByteBuffer中进行读操作后又要进行写操作，需要调用flip进行切换，api很难用，而netty很好地解决了这个问题，这也是netty的一个优点。

## 第二章

第二章主要就是使用netty的api编写了一个server端和client端，让读者先简单熟悉一下netty中的一些api的使用。

## 第三章

第三章主要对netty中的一些主要的组件做一个简单的介绍。

Channel: 一个Channel表示一个通道，跟io中的stream的角色一样，但是有几点不同。 1. stream是单向的，只支持读或者写，channel是双向的，既支持读也支持写。2. stream是阻塞的，channel是并行的。

EventLoop: 当一个channel注册之后，会将这个channel绑定到EventLoop里来处理channel的IO操作，一个EventLoop对应一个线程。

EventLoopGroup: 多个EventLoop的集合

Bootstrap: 用于配置客户端netty程序，比如连接的host和port，EventLoop等

ServerBootstrap: 用于配置服务端netty程序，比如绑定的port，EventLoop等

ChannelHandler: 主要用于处理关于Channel的业务逻辑。比如转换数据格式、当channel状态改变的时候被通知到，当channel注册到EventLoop的时候被通知到以及通知一些用户执行的特殊事件等。ChannelHandler有很多很多的实现类

ChannelPipeline: 把很多ChannelHandler整合在一起并进行处理

Future or ChannelFuture: 由于所有的netty都是异步操作。异步操作的结果就是ChannelFuture


ChannelInBoundHandler和ChannelOutBoundHandler是ChannelHandler的两种很常见的实现类，他们分别用于读取socket中的数据和写数据到socket中，他们存储在ChannelPipeline中用于处理socket，有一张很经典关于ChannelInboundHandler和ChannelOutboundHandler的图如下：

![image](http://7x2wh6.com1.z0.glb.clouddn.com/netty01.png)

最后讲解了关于java对象的编码和解码，因为使用netty开发的时候需要传输字节数据，而这些字节数据可以跟java对象之间进行互相转换使编程更加方便。

## 第四章

第四章主要讲解了netty中的4种transport，分别是OIO、NIO、Local和Embedded。

其中OIO这种transport在netty中的io.netty.channel.socket.oio包里，底层使用jdk的java.net包里的类，io模型为阻塞io模型

NIO在netty中的io.netty.channel.socket.nio包里，底层使用jdk的java.nio.channels包里的类，netty提供了2种nio的实现，分别是基于selector和基于epoll的实现。

Local在io.netty.channel.local包里，这是一种在同一个JVM中客户端与服务器进行通信的一种transport。

Embedded在io.netty.channel.embedded包里，主要用于测试，可以在不需要网络的情况下进行ChannelHandler的单元测试。

这4种transport的使用场景如下：

OIO: 低连接数，低延迟，需要阻塞时使用
NIO: 搞链接数
Local: 同一个JVM中进行通信时使用
Embedded: ChannelHandler单元测试时使用


## 第五章

第五章主要讲解netty中的ByteBuf的使用。

Jdk的NIO中的ByteBuffer使用成本过多，netty发现了这一缺点并进行了改造，设计出了ByteBuf这个类来代替ByteBuffer，相比ByteBuffer，ByteBuf有如下几个特点：

1. 可以定义自己的buffer类型，比如heap buffer, direct buffer等
2. 可以使用内置的composite buffer类型完成零拷贝
3. buffer容量可以扩展
4. 不需要调用flip来切换读写模式
5. 区分readerIndex和writerIndex
6. 方法链式调用
7. 使用引用计数
8. 可以使用pool来创建buffer


ByteBuf工作原理：内部有2个索引，分别是readerIndex和writerIndex，初始化时，这2个值都是0，当写入数据到buffer中，writerIndex会增加；当读取数据时，readerIndex会增加。当readerIndex = writerIndex的时候，再进行读取将会抛出IndexOutOfBoundsException异常。

ByteBuf是有容量概念的，默认情况下的最大的容量是Integer.MAX_VALUE。当writerIndex超过这个容量大小时，将会抛出异常。

下图就是一个ByteBuf的结构：

![image](http://7x2wh6.com1.z0.glb.clouddn.com/netty02.png)


ByteBuf共有3种类型：

1. heap buffer: 存储在JVM的堆中，内部使用一个字节数组存储字节。可以使用ByteBuf的hasArray方法判断是否是heap buffer
2. direct buffer：在堆外直接内存中分配，效率高，相比于堆内分配的buffer，在堆外直接内存中分配的buffer少了一次缓冲区的内存拷贝(实际上，当使用一个非直接内存buffer的时候，在发送buffer数据出去之前jvm内部会拷贝这个buffer到堆外直接内存中)
3. composite buffer：组合型的buffer，比如一个ByteBuf由2个ByteBuf构成，可以使用Composite Buffer完成，可以避免创建一个新的ByteBuf来整合这2个ByteBuf导致的内存拷贝

以上2、3点再加上netty文件传输采用的transferTo方法(可以直接将文件缓存区的数据发送到目标Channel，不需要通过循环write方式导致的内存拷贝问题)构成了Netty的零拷贝。


接下来就是ByteBuf的一些常用方法介绍，比如copy, slice, duplicate, readInt, writeInt, indexOf, clear, discardReadBytes方法等等。





一些参考资料：

[http://www.open-open.com/news/view/1d31d83](http://www.open-open.com/news/view/1d31d83)

[http://www.infoq.com/cn/articles/netty-high-performance](http://www.infoq.com/cn/articles/netty-high-performance)

[http://www.infoq.com/cn/author/%E6%9D%8E%E6%9E%97%E9%94%8B#%E6%96%87%E7%AB%A0](http://www.infoq.com/cn/author/%E6%9D%8E%E6%9E%97%E9%94%8B#%E6%96%87%E7%AB%A0)






