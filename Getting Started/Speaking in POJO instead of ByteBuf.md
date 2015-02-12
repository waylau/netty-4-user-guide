Speaking in POJO instead of ByteBuf 用POJO代替ByteBuf
============

我们回顾了迄今为止的所有例子使用 [ByteBuf](http://netty.io/4.0/api/io/netty/buffer/ByteBuf.html) 作为协议消息的主要数据结构。在本节中,我们将改善的 TIME 协议客户端和服务器例子，使用 POJO 代替 ByteBuf。

在 [ChannelHandler](http://netty.io/4.0/api/io/netty/channel/ChannelHandler.html) 使用 POIO 的好处很明显：通过从ChannelHandler 中提取出 ByteBuf 的代码，将会使 ChannelHandler的实现变得更加可维护和可重用。在 TIME 客户端和服务器的例子中，我们读取的仅仅是一个32位的整形数据，直接使用 ByteBuf 不会是一个主要的问题。然而，你会发现当你需要实现一个真实的协议，分离代码变得非常的必要。

首先，让我们定义一个新的类型叫做 UnixTime。

	public class UnixTime {
	
	    private final long value;
	
	    public UnixTime() {
	        this(System.currentTimeMillis() / 1000L + 2208988800L);
	    }
	
	    public UnixTime(long value) {
	        this.value = value;
	    }
	
	    public long value() {
	        return value;
	    }
	
	    @Override
	    public String toString() {
	        return new Date((value() - 2208988800L) * 1000L).toString();
	    }
	}

现在我们可以修改下 TimeDecoder 类，返回一个 UnixTime，以替代ByteBuf

	@Override
	protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
	    if (in.readableBytes() < 4) {
	        return;
	    }
	
	    out.add(new UnixTime(in.readUnsignedInt()));
	}

下面是修改后的解码器，TimeClientHandler 不再任何的 ByteBuf 代码了。

	@Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) {
	    UnixTime m = (UnixTime) msg;
	    System.out.println(m);
	    ctx.close();
	}

是不是变得更加简单和优雅了？相同的技术可以被运用到服务端。让我们修改一下 TimeServerHandler 的代码。

	@Override
	public void channelActive(ChannelHandlerContext ctx) {
	    ChannelFuture f = ctx.writeAndFlush(new UnixTime());
	    f.addListener(ChannelFutureListener.CLOSE);
	}

现在,唯一缺少的功能是一个编码器,是[ChannelOutboundHandler](http://netty.io/4.0/api/io/netty/channel/ChannelOutboundHandler.html)的实现，用来将 UnixTime 对象重新转化为一个 ByteBuf。这是比编写一个解码器简单得多,因为没有需要处理的数据包编码消息时拆分和组装。


	public class TimeEncoder extends ChannelOutboundHandlerAdapter {
	    @Override
	    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
	        UnixTime m = (UnixTime) msg;
	        ByteBuf encoded = ctx.alloc().buffer(4);
	        encoded.writeInt((int)m.value());
	        ctx.write(encoded, promise); // (1)
	    }
	}

1.在这几行代码里还有几个重要的事情。第一，通过 [ChannelPromise](http://netty.io/4.0/api/io/netty/channel/ChannelPromise.html)，当编码后的数据被写到了通道上 Netty 可以通过这个对象标记是成功还是失败。第二， 我们不需要调用 cxt.flush()。因为处理器已经单独分离出了一个方法 void flush(ChannelHandlerContext cxt),如果像自己实现 flush() 方法内容可以自行覆盖这个方法。

进一步简化操作，你可以使用 [MessageToByteEncode](http://netty.io/4.0/api/io/netty/handler/codec/MessageToByteEncoder.html):

	public class TimeEncoder extends MessageToByteEncoder<UnixTime> {
	    @Override
	    protected void encode(ChannelHandlerContext ctx, UnixTime msg, ByteBuf out) {
	        out.writeInt((int)msg.value());
	    }
	}

最后的任务就是在 TimeServerHandler 之前把 TimeEncoder 插入到ChannelPipeline。 但这是不那么重要的工作。