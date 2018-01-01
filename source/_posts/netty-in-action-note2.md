title: Netty in Action笔记(二)
date: 2016-08-29 20:36:52
tags:
- nio
- netty
categories: netty

----------------

主要记录ChannelHandler、Codec以及Bootstrap的作用，还有Netty内置的一些ChannelHandler、Codec，以及Bootstrap的分类。
	
<!--more-->

## 第六章

介绍Netty中的Channel、ChannelHandler、ChannelHandlerContext以及ChannelPipeline。

### Channel

定义：一个Channel表示一个通道，跟io中的stream的角色一样，但是有几点不同。 1. stream是单向的，只支持读或者写，channel是双向的，既支持读也支持写。2. stream是阻塞的，channel是并行的。

其中Channel的生命周期状态如下：

| 状态说明 | 说明 |
|------------|--------|
| channelUnregistered | channel创建之后，还未注册到EventLoop  |
| channelRegistered | channel注册到了对应的EventLoop  |
| channelActive | channel处于活跃状态，活跃状态表示已经连接到了远程服务器，现在可以接收和发送数据  |
| channelInactive | channel未连接到远程服务器 |

一个Channel正常的生命周期如下：

channelRegistered -> channelActice -> channelInactive -> channelUnregistered

在另外一种特殊情况下，会发生多次channelRegistered和channelUnregistered，这是因为用户可以从EventLoop上取消注册Channel来阻止事件的执行并在之后重新注册。状态变化如下：

![](http://7x2wh6.com1.z0.glb.clouddn.com/netty03.png)

### ChannelHandler


ChannelHandler有2种类型：

1. Inbound Handler: 处理inbound数据(接收到的数据)以及所有类型的channel状态修改事件
2. Outbound Handler: 处理outbound数据(发送出去的数据)并且可以拦截各种操作，比如bind、connect、disconnect、close、write等操作

Inbound和Outbound Handler都属于ChannelHandler，它们都可以被添加到ChannelPipeline中，它们内部也提供了handlerAdded、handlerRemoved这两种方法分别在ChannelHandler添加到ChannelPipeline和ChannelHandler从ChannelPipeline中被删除的时候触发。

#### ChannelInboundHandler


ChannelInboundHandler方法在两种情况下触发：channel状态的改变和channel接收到数据。

一些方法说明：

| 方法名 | 描述 |
|------------|--------|
| channelRegistered(..) | Channel注册到EventLoop，并且可以处理IO请求 |
| channelUnregistered(...) | Channel从EventLoop中被取消注册，并且不能处理任何IO请求 |
| channelActive(...) | Channel已经连接到远程服务器，并准备好了接收数据 |
| channelInactive(...) | Channel还没有连接到远程服务器 |
| channelReadComplete(...) | Channel的读取操作已经完成 |
| channelRead(...) | 有数据可以读取的时候触发 |
| userEventTriggered(...) | 当用户调用Channel.fireUserEventTriggered方法的时候触发，用户可以传递一个自定义的对象当这个方法里 |


ChannelInboundHandler有一个实现ChannelInboundHandlerAdapter，它实现了所有的方法，我们只需要继承这个类然后复写需要的方法即可。


ChannelInboundHandler中的channelRead方法中有读取的ByteBuf。由于Netty在ByteBuf的使用上使用了池的概念，当不需要这个ByteBuf的时候需要进行资源的释放以减少内存的消耗。

	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) {
		  // do something
    	  ReferenceCountUtil.release(msg);
	}

Netty内部提供了一个SimpleChannelInboundHandler类，这个类读取数据会自动释放资源。它继承ChannelInboundHandlerAdapter并复写了channelRead方法，在channelRead方法里面finally代码里会自动release资源，并提供了channelRead0方法：

	@Override
	public void channelRead0(ChannelHandlerContext ctx, Object msg) {
		  // do something, do not need release
	}

所以一般使用ChannelInboundHandler的话，有3种方法。 

1. 直接实现ChannelInBoundHandler接口 
2. 继承ChannelInboundHandlerAdapter
3. 继承SimpleChannelInboundHandler

第1种基本不用，第3种用来处理接收消息，第2种用来处理事件状态的改变


#### ChannelOutboundHandler

ChannelOutboundHandler方法在两种情况下触发：发送数据和拦截操作。

ChannelOutboundHandler有一个强大的功能，可以按需推迟一个操作，这使得处理请求可以用到更为复杂的策略。比如，如果写数据到远端被暂停，你可以推迟flush操作，稍后再试。

一些方法说明：

| 方法名 | 描述 |
|------------|--------|
| bind(..) | 请求绑定channel到本地地址 |
| connect(...) | channel连接到远程地址 |
| disconnect(...) | channel从远程服务器上断开 |
| close(...) | 关闭channel |
| deregister(...) | 取消channel在eventloop上的注册 |
| read(...) | 在channel中读数据 |
| flush(...) | flush数据到远程服务器 |
| write(...) | 写数据到远程服务器 |


ChannelOutboundHandler有一个实现ChannelOutboundHandlerAdapter，它实现了所有的方法，我们只需要继承这个类然后复写需要的方法即可。

在outboundhandler中有时候也需要释放资源，当消息被消费并且不再需要传递给下一个outbound handler的时候，调用ReferenceCountUtil.release(message)释放消息。

当消息被写回去或者channel关闭的时候，这个消息的资源会被自动释放，所以没有一个类似SimpleChannelInboundHandler的概念。

### ChannelHandlerContext

当ChannelHandler被添加到ChannelPipeline中的时候，ChannelHandlerContext会被创建。 
所以说ChannelHandlerContext属于ChannelHandler。

可以通过ChannelHandlerContext的channel方法得到Channel和pipeline方法得到ChannelPipeline。

ChannelHandlerContext可以被保留下来并且在其他地方进行调用，比如在其他线程，或者在handler外部进行调用。

可以使用以下方法保留ChannelHandlerContext：

	class WriterHandler extends ChannelHandlerAdapter {
	  private ChannelHandlerContext ctx;

	  @Override
	  public void handlerAdded(ChannelHandlerContext ctx) throws Exception {
    	  this.ctx = ctx;
	  }

	  public void send(String msg) {
    	  ctx.write(msg);
	  }
	}

Netty中提供了一个@Sharable注解用来将一个实例的ChannelHandler添加到多个ChannelPipeline中，如果不加上这个注解，被多个ChannelPipeline使用的话会抛出异常。


### ChannelPipeline

多个ChannelHandler可以组成一个ChannelPipeline，里面的每个ChannelHandler可以转发给下一个ChannelHandler。

ChannelPipeline内部的所有ChannelHandler的处理流程图：

![](http://7x2wh6.com1.z0.glb.clouddn.com/netty01.png)

ChannelPipeline提供了多种方法用于添加或删除或代替ChannelHandler，比如addLast, addFirst, remove, replace等方法。


## 第七章

介绍Netty中的编码器、解码器。我们知道网络中传输的是字节-ByteBuf。我们需要对ByteBuf进行一些解码用于解码成熟悉的POJO，对ByteBuf进行编码用于网络传输。

### Decoder

解码器，针对的是inbound的数据，也就是读取数据的解码。

2种类型：

1. bytes到message的解码(ByteToMessageDecoder和ReplayingDecoder)
2. message到message的解码(MessageToMessageDecoder)

decoders的作用是把inbound中读取的数据从一种格式转换成另一种格式，由于decoders处理的是inbound中的数据，所以它也是ChannelInboundHandler的一种实现类。

ByteToMessageDecoder和ReplayingDecoder属于bytes到message的解码。

一个ByteToMessageDecoder例子，将byte转换成integer：

	public class ToIntegerDecoder extends ByteToMessageDecoder {
    	@Override  protected void decode(ChannelHandlerContext ctx, ByteBuf in, 	List<Object> out) throws Exception {
    	    if(in.readableBytes() >= 4) {
        	    out.add(in.readInt());
	        }
	    }    
	}
	
处理流程图如下：

![](http://7x2wh6.com1.z0.glb.clouddn.com/netty04.png)

如果使用ReplayingDecoder，不需要进行可读字节的判断，直接添加到List里即可。跟ToIntegerDecoder实现一样的功能，ReplayingDecoder只需要这样即可。(但是有一定的局限性：1.不是所有的操作都被ByteBuf支持 2.ByteBuf.readableBytes方法大部分时间不会返回期望的值)

	public class ToIntegerDecoder2 extends ReplayingDecoder<Void> {
    	@Override
	    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> 	out) throws Exception {
    	    out.add(in.readInt());
	    }
	}
	
一个MessageToMessageDecoder例子，将integer转换成string：

	public class IntegerToStringDecoder extends MessageToMessageDecoder<Integer> {
    	@Override
	    protected void decode(ChannelHandlerContext ctx, Integer msg, List<Object> out) throws Exception {
    	    out.add(String.valueOf(msg));
	    }
	}
	
处理流程图如下：

![](http://7x2wh6.com1.z0.glb.clouddn.com/netty05.png)


### Encoder

编码器，针对的是outbound的数据，也就是发送出去的数据的编码。

2种类型：

1. message到message的编码(MessageToMessageEncoder)
2. message到byte的编码(MessageToByteEncoder)

decoders的作用是把outbound中发送出去的数据从一种格式转换成另一种格式，由于eecoders处理的是outbound中的数据，所以它也是ChannelOutboundHandler的一种实现类。


一个MessageToByteEncoder例子，将integer转换成byte：

	public class IntegerToByteEncoder extends MessageToByteEncoder<Integer> {
    	@Override
	    protected void encode(ChannelHandlerContext ctx, Integer msg, ByteBuf out) throws Exception {
    	    out.writeInt(msg);
	    }
	}
	
处理流程图如下：

![](http://7x2wh6.com1.z0.glb.clouddn.com/netty06.png)


一个MessageToMessageEncoder例子，将integer转换成string：

	public class IntegerToStringEncoder extends MessageToMessageEncoder<Integer> {
    	@Override
	    protected void encode(ChannelHandlerContext ctx, Integer msg, List<Object> out) throws Exception {
    	    out.add(String.valueOf(msg));
	    }
	}
	
处理流程图如下：

![](http://7x2wh6.com1.z0.glb.clouddn.com/netty07.png)


### Codec

编解码器，既支持编码器的功能，也支持解码器的功能。

2种类型：

1. ByteToMessageCodec：message到byte的编解码
2. MessageToMessageCodec：message到message的编解码


### CombinedChannelDuplexHandler

结合解码器和编码器在一起可能会牺牲可重用性。为了避免这种方式，可以使用CombinedChannelDuplexHandler。

CombinedChannelDuplexHandler也就是codec的另外一种方式，如果已经有个encoder和decoder，不需要重新写了codec，直接使用CombinedChannelDuplexHandler整合这个encoder和decoder即可。

上面的ToIntegerDecoder和IntegerToByteEncoder就可以构成一个编解码器，直接使用CombinedChannelDuplexHandler即可。

	public class CombinedByteIntegerCodec extends CombinedChannelDuplexHandler<ToIntegerDecoder, IntegerToByteEncoder> {
    	public CombinedByteIntegerCodec() {
        	super(new ToIntegerDecoder(), new IntegerToByteEncoder());
	    }
	}

## 第八章

主要说明jetty内置的一些ChannelHandler和Codec。

使用SSL/TLS加密Netty程序的话，可以使用内置的SslHandler。

要构建Http应用的话，可以使用HttpClientCodec/HttpServerCodec(http客户端和服务端的编解码)以及HttpObjectAggregator(Http的消息聚合)。


可以使用HttpContentDecompressor和HttpContentCompressor对http的内容进行解压和压缩。

还有一些WebSocket、SPDY，一些空置链接、超时链接等处理的内置解决方案。

## 第九章

主要讲解Bootstrap中Netty中的作用。

之前分析过各种ChannelHandler，各种Codec，以及把这两个东西添加到Channel 的ChannelPipeline中。有了这些东西之后，该用什么把他们整合起来呢，那就是Bootstrap。

Bootstrap分别ServerBootstrap(服务端)和Bootstrap(客户端)，它们都继承AbstractBootstrap。

### Bootstrap

客户端服务启动类，内部提供的一些方法如下：

| 方法名 | 描述 |
|------------|--------|
| group(..) | 设置Bootstrap使用的EventLoopGroup，用来处理Channel的IO操作 |
| channel(..) | Channel的类型，比如有NioSocketChannel, OioSocketChannel等 |
| channelFactory(..) | 如果Channel没有没有参数的构造函数，需要使用ChannelFactory构造Channel  |
| localAddress(..) | Channel需要绑定的地址和端口，可以不调用，使用connect或者bind再进行设置 |
| option(..) | 一些可选的设置，使用ChannelOptions完成。比如keep-alive时间，超时时间 |
| handler(..) | 设置ChannelHandler处理事件 |
| clone(..) | 复制一个新的Bootstrap，AbstractBootstrap实现了Cloneable接口 |
| remoteAddress()..) | 设置远程地址，也可以在调用connect方法的时候设置 |
| connect()..) | 链接到远程地址 |
| bind()..) | 绑定本地端口 |

需要注意的是如果EventLoopGroup选择的是NioEventLoopGroup，那么对应的channel需要选择NioSocketChannel，否则会抛出兼容性的错误异常。


### ServerBootstrap

服务端服务启动类，内部提供的一些方法如下：


| 方法名 | 描述 |
|------------|--------|
| group(..) | 设置ServerBootstrap使用的EventLoopGroup，用来处理ServerChannel的IO操作并接收Channel |
| channel(..) | ServerChannel的类型，比如有NioServerSocketChannel, OioServerSocketChannel等 |
| channelFactory(..) | 如果ServerChannel没有没有参数的构造函数，需要使用ChannelFactory构造ServerChannel  |
| localAddress(..) | ServerChannel需要绑定的地址和端口，可以不调用，使用connect或者bind再进行设置 |
| option(..) | 一些ServerChannel可选的设置，使用ChannelOptions完成。比如keep-alive时间，超时时间 |
| childOption(..) | 被ServerChannel接收的Channel的可选的设置，使用ChannelOptions完成 |
| handler(..) | 设置ServerChannel的ChannelHandler处理事件 |
| childHandler(..) | 设置被ServerChannel接收的Channel的ChannelHandler处理事件 |
| clone(..) | 复制一个新的ServerBootstrap，AbstractBootstrap实现了Cloneable接口 |
| bind()..) | 绑定本地端口 |


ServerBootstrap的处理过程：

![](http://7x2wh6.com1.z0.glb.clouddn.com/netty08.png)

ServerBootstrap调用bind绑定地址和端口的时候，会创建ServerChannel。这个ServerChannel会接收客户的各个链接，针对各个链接创建Channel。

handler方法就是为ServerChannel服务的，而childHandler是给被ServerChannel接收的Channel服务器的。所以说只要服务器已起来，handler中的ChannelHandler就会触发，而有链接被ServerChannel接收之后childHandler中的ChannelHandler才会触发。


### 从一个已经存在的Channel中使用Bootstrap启动客户端

在ServerBootstrap接收到新的Channel的时候准备启动Bootstrap客户端的时候，可以使用一个全新的EventLoop用于处理Channel的IO模型。

但是没有必要，可以跟ServerBootstrap共享同一个EventLoop，因为一个EventLoop是跟一个线程绑定的，如果使用了多个EventLoop的话，那就相当于需要进行线程的上下文切换，需要消耗一定的资源。


1个在ServerBootstrap接收到链接之后，使用Bootstrap链接另外一个地址的处理过程：


![](http://7x2wh6.com1.z0.glb.clouddn.com/netty09.png)

其中第3点就是ServerChannel接收到的新的Channel，第5点是Bootstrap创建的连接到远程服务器的Channel，它们使用同一个EventLoop。

