Writing a Time Client 写个时间客户端
==================

不像 DISCARD 和 ECHO 的服务端，对于 TIME 协议我们需要一个客户端,因为人们不能把一个32位的二进制数据翻译成一个日期或者日历。在这一部分，我们将会讨论如何确保服务端是正常工作的，并且学习怎样用Netty 编写一个客户端。

在 Netty 中,编写服务端和客户端最大的并且唯一不同的使用了不同的[BootStrap](http://netty.io/4.0/api/io/netty/bootstrap/Bootstrap.html) 和 [Channel](http://netty.io/4.0/api/io/netty/channel/Channel.html)的实现。请看一下下面的代码：

	public class TimeClient {
	
		public static void main(String[] args) throws Exception {
			
			String host = args[0];
	        int port = Integer.parseInt(args[1]);
	        EventLoopGroup workerGroup = new NioEventLoopGroup();
	
	        try {
	            Bootstrap b = new Bootstrap(); // (1)
	            b.group(workerGroup); // (2)
	            b.channel(NioSocketChannel.class); // (3)
	            b.option(ChannelOption.SO_KEEPALIVE, true); // (4)
	            b.handler(new ChannelInitializer<SocketChannel>() {
	                @Override
	                public void initChannel(SocketChannel ch) throws Exception {
	                    ch.pipeline().addLast(new TimeClientHandler());
	                }
	            });
	
	            // 启动客户端
	            ChannelFuture f = b.connect(host, port).sync(); // (5)
	
	            // 等待连接关闭
	            f.channel().closeFuture().sync();
	        } finally {
	            workerGroup.shutdownGracefully();
	        }
		}
	}

1.BootStrap 和 [ServerBootstrap](http://netty.io/4.0/api/io/netty/bootstrap/ServerBootstrap.html) 类似,不过他是对非服务端的 channel 而言，比如客户端或者无连接传输模式的 channel。

2.如果你只指定了一个 [EventLoopGroup](http://netty.io/4.0/api/io/netty/channel/EventLoopGroup.html)，那他就会即作为一个 boss group ，也会作为一个 workder group，尽管客户端不需要使用到 boss worker 。

3.代替[NioServerSocketChannel](http://netty.io/4.0/api/io/netty/channel/socket/nio/NioServerSocketChannel.html)的是[NioSocketChannel](http://netty.io/4.0/api/io/netty/channel/socket/nio/NioSocketChannel.html),这个类在客户端channel 被创建时使用。

4.不像在使用 ServerBootstrap 时需要用 childOption() 方法，因为客户端的 [SocketChannel](http://netty.io/4.0/api/io/netty/channel/socket/SocketChannel.html) 没有父亲。

5.我们用 connect() 方法代替了 bind() 方法。

正如你看到的，他和服务端的代码是不一样的。[ChannelHandler](http://netty.io/4.0/api/io/netty/channel/ChannelHandler.html) 是如何实现的?他应该从服务端接受一个32位的整数消息，把他翻译成人们能读懂的格式，并打印翻译好的时间，最后关闭连接:

	import java.util.Date;
	
	public class TimeClientHandler extends ChannelInboundHandlerAdapter {
	    @Override
	    public void channelRead(ChannelHandlerContext ctx, Object msg) {
	        ByteBuf m = (ByteBuf) msg; // (1)
	        try {
	            long currentTimeMillis = (m.readUnsignedInt() - 2208988800L) * 1000L;
	            System.out.println(new Date(currentTimeMillis));
	            ctx.close();
	        } finally {
	            m.release();
	        }
	    }
	
	    @Override
	    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
	        cause.printStackTrace();
	        ctx.close();
	    }
	}

1.在TCP/IP中，Netty 会把读到的数据放到 ByteBuf 的数据结构中。

![](http://99btgc01.info/uploads/2015/02/time.jpg)

这样看起来非常简单，并且和服务端的那个例子的代码也相差不多。然而，处理器有时候会因为抛出 IndexOutOfBoundsException 而拒绝工作。在下个部分我们会讨论为什么会发生这种情况。

