Looking into the Received Data 查看收到的数据
========================


现在我们已经编写出我们第一个服务端，我们需要测试一下他是否真的可以运行。最简单的测试方法是用 telnet 命令。例如，你可以在命令行上输入`telnet localhost 8080`或者其他类型参数。

![](http://99btgc01.info/uploads/2015/02/telnet%281%29.jpg)

![](http://99btgc01.info/uploads/2015/02/telnet2%281%29.jpg)

然而我们能说这个服务端是正常运行了吗？事实上我们也不知道，因为他是一个 discard 服务，你根本不可能得到任何的响应。为了证明他仍然是在正常工作的，让我们修改服务端的程序来打印出他到底接收到了什么。

我们已经知道 channelRead() 方法是在数据被接收的时候调用。让我们放一些代码到 DiscardServerHandler 类的 channelRead() 方法。

    @Override
	public void channelRead(ChannelHandlerContext ctx, Object msg) {
	    ByteBuf in = (ByteBuf) msg;
	    try {
	        while (in.isReadable()) { // (1)
	            System.out.print((char) in.readByte());
	            System.out.flush();
	        }
	    } finally {
	        ReferenceCountUtil.release(msg); // (2)
	    }
	}

1.这个低效的循环事实上可以简化为:System.out.println(in.toString(io.netty.util.CharsetUtil.US_ASCII))

2.或者，你可以在这里调用 in.release()。

如果你再次运行 telnet 命令，你将会看到服务端打印出了他所接收到的消息。

![](http://99btgc01.info/uploads/2015/02/telnet3%281%29.jpg)

完整的discard server代码放在了[io.netty.example.discard](http://netty.io/4.0/xref/io/netty/example/discard/package-summary.html)包下面。

*译者注：翻译版本的项目源码见 <https://github.com/waylau/netty-4-user-guide-demos> 中的`com.waylau.netty.demo.discard` 包下*