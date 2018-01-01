title: Netty的单元测试
date: 2016-09-03 21:04:52
tags:
- nio
- netty
categories: netty

----------------

我们在编写完Netty程序之后，会需要进行ChannelHandler的一些测试。

在初学Netty的时候，我们都是直接起一个Server，然后再写一个Client去连接Server，看传输的数据和接收到的数据是否正确。

在[Netty in Action](https://book.douban.com/subject/24700704/)书中的Transport和Unit-test your code这两章就提出了可以使用EmbeddedChannel进行ChanndlHandler的单元测试。
	
<!--more-->

## EmbeddedChannel介绍

EmbeddedChannel中提供了一些方法：

| 方法名 | 说明 |
|------------|--------|
| writeInbound | 写数据到Channel中，但是这些数据只会被ChannelPipeline中的inbound handler处理。如果可以从EmbeddedChannel的readInbound方法中读取出数据的话，就返回true  |
| readInbound | 读取在EmbeddedChannel上被所有inbound handler处理过的数据，如果已经没有数据可以读取就返回null  |
| writeInbound | 写数据到Channel中，但是这些数据只会被ChannelPipeline中的outbound handler处理。如果可以从EmbeddedChannel的readOutbound方法中读取出数据的话，就返回true  |
| readOutbound | 读取在EmbeddedChannel上被所有outbound handler处理过的数据，如果已经没有数据可以读取就返回null  |
| finish | 标记EmbeddedChannel已经完成。如果可以从inbound或者outbound中返回数据，该方法就返回true。这个方法还会关闭Channel  |


下图就是EmbeddedChannel的处理流程图：

![](http://7x2wh6.com1.z0.glb.clouddn.com/netty10.png)

使用writeInbound方法写入的数据，并经过Pipeline中所有的inbound handler，之后可以使用readInbound方法读取经过inbound handler之后的数据。

使用writeOutbound方法写入的数据，并经过Pipeline中所有的outbound handler，之后可以使用readOutbound方法读取经过outbound handler之后的数据。

调用finish方法可以标记EmbeddedChannel已经完成。


## Inbound Handler的测试


Netty内部提供了一个FixedLengthFrameDecoder解码器用于把长度不固定的字节转换成固定长度的字节，处理流程图如下：

![](http://7x2wh6.com1.z0.glb.clouddn.com/netty11.png)

针对这个Decoder编写单元测试。

由于是个Decoder，针对的是inbound中的数据，所以需要使用的方法是writeInbound和readInbound：

	ByteBuf buf = Unpooled.buffer(); // 构造heap buffer
	for(int i = 0; i < 9; i ++) { // 写入9个字节
	    buf.writeByte(i);
	}

	ByteBuf input = buf.copy();

	// 构造EmbeddedChannel，并在Pipeline中加入FixedLengthFrameDecoder
	EmbeddedChannel channel = new EmbeddedChannel(new FixedLengthFrameDecoder(3));

	// 使用writeInbound方法写入数据
	Assert.assertTrue(channel.writeInbound(input));
	// 标记EmbeddedChannel状态已经complete
	Assert.assertTrue(channel.finish());

	// 读取经过FixedLengthFrameDecoder处理过后的字节
	Assert.assertEquals(buf.readBytes(3), channel.readInbound());
	Assert.assertEquals(buf.readBytes(3), channel.readInbound());
	Assert.assertEquals(buf.readBytes(3), channel.readInbound());
	Assert.assertNull(channel.readInbound());
	
	
我们在[使用Netty编写自定义的协议](http://fangjian0423.github.io/2016/08/30/netty-custom-protocol/)文章中编写的自定义协议CustomProtocol的解码器，并最后通过一个server和client的编写完成了测试。

现在我们可以使用EmbeddedChannel进行Decoder的unit test：

	EmbeddedChannel channel = new EmbeddedChannel(new CustomProtocolDecoder());
	String uuid = UUID.randomUUID().toString();
	channel.writeInbound(new CustomProtocol(1024l, uuid, "content content"));
	Assert.assertTrue(channel.finish());

	CustomProtocol customProtocol = (CustomProtocol) channel.readInbound();
	// 判断是否正确
	Assert.assertEquals(1024l, customProtocol.getVersion());
	Assert.assertEquals(uuid, customProtocol.getHeader());
	Assert.assertEquals("content content", customProtocol.getContent());
	Assert.assertNull(channel.readInbound());
	
## Outbound Handler的测试

AbsIntegerEncoder对所有的int数据取绝对值。处理流过图如下：

![](http://7x2wh6.com1.z0.glb.clouddn.com/netty12.png)

	public class AbsIntegerEncoder extends MessageToMessageEncoder<ByteBuf> {
	    @Override
	    protected void encode(ChannelHandlerContext ctx, ByteBuf msg, List<Object> out) throws Exception {
	        while(msg.readableBytes() >= 4) {
	            int value = Math.abs(msg.readInt());
	            out.add(value);
	        }
	    }
	}

针对这个Encoder编写单元测试。

由于是个Encoder，针对的是outbound中的数据，所以需要使用的方法是writeOutbound和readOutbound：

	ByteBuf buf = Unpooled.buffer(); // 构造heap buffer
	for(int i = 1; i < 10; i ++) { // 写入10个负数
	    buf.writeInt(i * -1);
	}
	// 构造EmbeddedChannel，并在Pipeline中加入AbsIntegerEncoder
	EmbeddedChannel channel = new EmbeddedChannel(new AbsIntegerEncoder());
	// 使用writeOutbound方法写入数据
	Assert.assertTrue(channel.writeOutbound(buf));
	// 标记EmbeddedChannel状态已经complete
	Assert.assertTrue(channel.finish());

	// 测试是否所有的int数据都取了绝对值
	for(int i = 1; i < 10; i ++) {
	    Assert.assertEquals(i, (int)channel.readOutbound());
	}
	Assert.assertNull(channel.readOutbound());

同理我们可以使用EmbeddedChannel进行CustomProtocol的Encoder的unit test：

	EmbeddedChannel channel = new EmbeddedChannel(new CustomProtocolEncoder());
	String uuid = UUID.randomUUID().toString();

	channel.writeOutbound(new CustomProtocol(1024l, uuid, "content content"));
	Assert.assertTrue(channel.finish());

	ByteBuf buf = (ByteBuf) channel.readOutbound();
	Assert.assertEquals(1024l, buf.readLong());
	byte[] headerBytes = new byte[36];
	buf.readBytes(headerBytes);
	Assert.assertEquals(uuid, new String(headerBytes));
	byte[] contentBytes = new byte[buf.readableBytes()];
	buf.readBytes(contentBytes);
	Assert.assertEquals("content content", new String(contentBytes));
	Assert.assertNull(channel.readOutbound());