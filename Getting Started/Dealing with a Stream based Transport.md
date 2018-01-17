Dealing with a Stream-based Transport 处理一个基于流的传输
=================

##One Small Caveat of Socket Buffer 关于 Socket Buffer的一个小警告

基于流的传输比如 TCP/IP, 接收到数据是存在 socket 接收的 buffer 中。不幸的是，基于流的传输并不是一个数据包队列，而是一个字节队列。意味着，即使你发送了2个独立的数据包，操作系统也不会作为2个消息处理而仅仅是作为一连串的字节而言。因此这是不能保证你远程写入的数据就会准确地读取。举个例子，让我们假设操作系统的 TCP/TP 协议栈已经接收了3个数据包：

![](http://99btgc01.info/uploads/2015/02/buf01.png)

由于基于流传输的协议的这种普通的性质，在你的应用程序里读取数据的时候会有很高的可能性被分成下面的片段

![](http://99btgc01.info/uploads/2015/02/buf02.png)

因此，一个接收方不管他是客户端还是服务端，都应该把接收到的数据整理成一个或者多个更有意思并且能够让程序的业务逻辑更好理解的数据。在上面的例子中，接收到的数据应该被构造成下面的格式：

![](http://99btgc01.info/uploads/2015/02/buf01.png)

##The First Solution 办法一

回到 TIME 客户端例子。同样也有类似的问题。一个32位整型是非常小的数据，他并不见得会被经常拆分到到不同的数据段内。然而，问题是他确实可能会被拆分到不同的数据段内，并且拆分的可能性会随着通信量的增加而增加。

最简单的方案是构造一个内部的可积累的缓冲，直到4个字节全部接收到了内部缓冲。下面的代码修改了 TimeClientHandler 的实现类修复了这个问题

	public class TimeClientHandler extends ChannelInboundHandlerAdapter {
	    private ByteBuf buf;
	
	    @Override
	    public void handlerAdded(ChannelHandlerContext ctx) {
	        buf = ctx.alloc().buffer(4); // (1)
	    }
	
	    @Override
	    public void handlerRemoved(ChannelHandlerContext ctx) {
	        buf.release(); // (1)
	        buf = null;
	    }
	
	    @Override
	    public void channelRead(ChannelHandlerContext ctx, Object msg) {
	        ByteBuf m = (ByteBuf) msg;
	        buf.writeBytes(m); // (2)
	        m.release();
	
	        if (buf.readableBytes() >= 4) { // (3)
	            long currentTimeMillis = (buf.readUnsignedInt() - 2208988800L) * 1000L;
	            System.out.println(new Date(currentTimeMillis));
	            ctx.close();
	        }
	    }
	
	    @Override
	    public void exceptionCaught(ChannelHandlerContext ctx, Throwable cause) {
	        cause.printStackTrace();
	        ctx.close();
	    }
	}

1.[ChannelHandler](http://netty.io/4.0/api/io/netty/channel/ChannelHandler.html) 有2个生命周期的监听方法：handlerAdded()和 handlerRemoved()。你可以完成任意初始化任务只要他不会被阻塞很长的时间。

2.首先，所有接收的数据都应该被累积在 buf 变量里。

3.然后，处理器必须检查 buf 变量是否有足够的数据，在这个例子中是4个字节，然后处理实际的业务逻辑。否则，Netty 会重复调用channelRead() 当有更多数据到达直到4个字节的数据被积累。

##The Second Solution 方法二

尽管第一个解决方案已经解决了 TIME 客户端的问题了，但是修改后的处理器看起来不那么的简洁，想象一下如果由多个字段比如可变长度的字段组成的更为复杂的协议时，你的  [ChannelInboundHandler](http://netty.io/4.0/api/io/netty/channel/ChannelInboundHandler.html) 的实现将很快地变得难以维护。

正如你所知的，你可以增加多个 [ChannelHandler](http://netty.io/4.0/api/io/netty/channel/ChannelHandler.html) 到[ChannelPipeline](http://netty.io/4.0/api/io/netty/channel/ChannelPipeline.html) ,因此你可以把一整个ChannelHandler 拆分成多个模块以减少应用的复杂程度，比如你可以把TimeClientHandler 拆分成2个处理器：

* TimeDecoder 处理数据拆分的问题
* TimeClientHandler 原始版本的实现

幸运地是，Netty 提供了一个可扩展的类，帮你完成 TimeDecoder 的开发。

	public class TimeDecoder extends ByteToMessageDecoder { // (1)
	    @Override
	    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) { // (2)
	        if (in.readableBytes() < 4) {
	            return; // (3)
	        }
	
	        out.add(in.readBytes(4)); // (4)
	    }
	}

1.[ByteToMessageDecoder](http://netty.io/4.0/api/io/netty/handler/codec/ByteToMessageDecoder.html) 是  [ChannelInboundHandler](http://netty.io/4.0/api/io/netty/channel/ChannelInboundHandler.html)  的一个实现类，他可以在处理数据拆分的问题上变得很简单。

2.每当有新数据接收的时候，ByteToMessageDecoder 都会调用 decode() 方法来处理内部的那个累积缓冲。

3.Decode() 方法可以决定当累积缓冲里没有足够数据时可以往 out 对象里放任意数据。当有更多的数据被接收了 ByteToMessageDecoder 会再一次调用 decode() 方法。

4.如果在 decode() 方法里增加了一个对象到 out 对象里，这意味着解码器解码消息成功。ByteToMessageDecoder 将会丢弃在累积缓冲里已经被读过的数据。请记得你不需要对多条消息调用 decode()，ByteToMessageDecoder 会持续调用 decode() 直到不放任何数据到 out 里。

现在我们有另外一个处理器插入到 [ChannelPipeline](http://netty.io/4.0/api/io/netty/channel/ChannelPipeline.html) 里，我们应该在 TimeClient 里修改 ChannelInitializer 的实现：

	b.handler(new ChannelInitializer<SocketChannel>() {
	    @Override
	    public void initChannel(SocketChannel ch) throws Exception {
	        ch.pipeline().addLast(new TimeDecoder(), new TimeClientHandler());
	    }
	});

如果你是一个大胆的人，你可能会尝试使用更简单的解码类[ReplayingDecoder](http://netty.io/4.0/api/io/netty/handler/codec/ReplayingDecoder.html)。不过你还是需要参考一下 API 文档来获取更多的信息。

	public class TimeDecoder extends ReplayingDecoder<Void> {
	    @Override
	    protected void decode(
	            ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
	        out.add(in.readBytes(4));
	    }
	}

此外，Netty还提供了更多开箱即用的解码器使你可以更简单地实现更多的协议，帮助你避免开发一个难以维护的处理器实现。请参考下面的包以获取更多更详细的例子：

* 对于二进制协议请看 [io.netty.example.factorial](http://netty.io/4.0/xref/io/netty/example/factorial/package-summary.html)
* 对于基于文本协议请看 [io.netty.example.telnet](http://netty.io/4.0/xref/io/netty/example/telnet/package-summary.html)

*译者注：翻译版本的项目源码见 <https://github.com/waylau/netty-4-user-guide-demos> 中的`com.waylau.netty.demo.factorial` 和 `com.waylau.netty.demo.telnet` 包下*
