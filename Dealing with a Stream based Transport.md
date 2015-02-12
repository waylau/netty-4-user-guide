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

TimeDecoder 处理数据拆分的问题
TimeClientHandler 原始版本的实现
幸运地是，Netty 提供了一个可扩展的类，帮你完成 TimeDecoder 的开发。