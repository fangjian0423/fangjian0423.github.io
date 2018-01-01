title: 使用Netty编写自定义的协议
date: 2016-08-30 22:04:52
tags:
- nio
- netty
categories: netty

----------------

Netty内部提供的Codec用于处理消息的编解码问题，比如Http，Ftp，SMTP这些协议，都需要进行编解码。

Netty内部直接提供了Http协议的Codec组件用于编解码Http协议。

在开发过程中，有时候我们需要构建一些适应自己业务的应用层协议，Netty可以很方便地实现自定义协议。

比如我们的自定义协议CucstomProtocol结构如下：

	| version | header | content |
	
其中version表示版本号，long类型；header是一个uuid，string类型，content则是具体的内容，string类型。
	
<!--more-->

对应的POJO如下：

	public class CustomProtocol {
	    private long version; // 版本
	    private String header; // 头信息(UUID)
	    private String content; // 具体内容
	    // GET SET ...
	    @Override
	    public String toString() {
	        return "CustomProtocol{" +
	                "version=" + version +
	                ", header='" + header + '\'' +
	                ", content='" + content + '\'' +
	                '}';
	    }
	}


解码器把byte转换成CustomProtocol，在server中使用：

	public class CustomProtocolDecoder extends ByteToMessageDecoder {
	    @Override
	    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) throws Exception {
	        long version = in.readLong(); // 读取version

	        byte[] headerBytes = new byte[36];
	        in.readBytes(headerBytes); // 读取header

	        String header = new String(headerBytes);

	        byte[] contentBytes = new byte[in.readableBytes()]; // 读取content
	        in.readBytes(contentBytes);

	        out.add(new CustomProtocol(version, header, new String(contentBytes)));
	    }
	}

编码器把CustomProtocol转换成byte，在client中使用：

	public class CustomProtocolEncoder extends MessageToByteEncoder<CustomProtocol> {
	    @Override
	    protected void encode(ChannelHandlerContext ctx, CustomProtocol msg, ByteBuf out) throws Exception {
	        out.writeLong(msg.getVersion());
	        out.writeBytes(msg.getHeader().getBytes());
	        out.writeBytes(msg.getContent().getBytes());
	    }
	}

server代码：

	ServerBootstrap serverBootstrap = new ServerBootstrap();
    EventLoopGroup eventLoopGroup = new NioEventLoopGroup();
    EventLoopGroup childEventLoopGroup = new NioEventLoopGroup();

    try {
        serverBootstrap
                .group(eventLoopGroup, childEventLoopGroup)
                .channel(NioServerSocketChannel.class)
                .childHandler(new ChannelInitializer<Channel>() {
                    @Override
                    protected void initChannel(Channel ch) throws Exception {
                        ch.pipeline().addLast(new CustomProtocolDecoder()); // 解码器
                        ch.pipeline().addLast(new ServerHandler()); // 打印数据
                    }
                });

        ChannelFuture future = serverBootstrap.bind("localhost", 9999).sync();

        future.channel().closeFuture().sync();
    } catch (Exception e) {
        e.printStackTrace();
    } finally {
        try {
            eventLoopGroup.shutdownGracefully().sync();
            childEventLoopGroup.shutdownGracefully().sync();
        } catch (Exception e1) {
            e1.printStackTrace();
        }
    }

ServerHandler用于打印接收到的数据，并写数据回去给客户端表示接收到了数据：

	public class ServerHandler extends SimpleChannelInboundHandler<CustomProtocol> {
	    @Override
	    protected void channelRead0(ChannelHandlerContext ctx, CustomProtocol msg) throws Exception {
	        System.out.println("server receive: " + msg);
	        ctx.writeAndFlush(Unpooled.copiedBuffer("server get", CharsetUtil.UTF_8)).addListener(ChannelFutureListener.CLOSE);
	    }
	}
	
client代码：

	Bootstrap bootstrap = new Bootstrap();

	EventLoopGroup eventLoopGroup = new NioEventLoopGroup();

	try {
	    bootstrap
	            .group(eventLoopGroup)
	            .channel(NioSocketChannel.class)
	            .handler(new ChannelInitializer<Channel>() {
	                @Override
	                protected void initChannel(Channel ch) throws Exception {
	                    ch.pipeline().addLast(new CustomProtocolEncoder()); // 编码器
	                    ch.pipeline().addLast(new ClientHandler()); // 接收服务端数据
	                }
	            });
	    ChannelFuture channelFuture = bootstrap.connect("localhost", 9999).sync();

	    channelFuture.channel().writeAndFlush(new CustomProtocol(1024l, UUID.randomUUID().toString(), "content detail"));

	    channelFuture.channel().closeFuture().sync();
	} catch (Exception e) {
	    e.printStackTrace();
	} finally {
	    try {
	        eventLoopGroup.shutdownGracefully().sync();
	    } catch (InterruptedException e) {
	        e.printStackTrace();
	    }
	}

ClientHandler用于接收服务端返回回来的数据：

	public class ClientHandler extends SimpleChannelInboundHandler<ByteBuf> {
	    @Override
	    protected void channelRead0(ChannelHandlerContext ctx, ByteBuf msg) throws Exception {
	        System.out.println("send success, response is: " + msg.toString(CharsetUtil.UTF_8));
	    }
	}
	
	
启动服务端和客户端之后，服务端收到客户端发来的CustomProtocol：

	server receive: CustomProtocol{version=1024, header='6ed7fa3d-7d54-4add-9081-d659d4b37d3f', content='content detail'}
	
之后客户端也收到服务端成功接收数据的反馈：

	send success, response is: server get
	
	