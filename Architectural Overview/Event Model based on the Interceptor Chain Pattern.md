Event Model based on the Interceptor Chain Pattern 基于拦截链模式的事件模型
===========

一个定义良好并具有扩展能力的事件模型是事件驱动开发的必要条件。Netty 具有定义良好的 I/O 事件模型。由于严格的层次结构区分了不同的事件类型，因此 Netty 也允许你在不破坏现有代码的情况下实现自己的事件类型。这是与其他框架相比另一个不同的地方。很多 NIO 框架没有或者仅有有限的事件模型概念；在你试图添加一个新的事件类型的时候常常需要修改已有的代码，或者根本就不允许你进行这种扩展。

在一个 [ChannelPipeline](http://netty.io/4.0/api/io/netty/channel/ChannelPipeline.html) 内部一个 [ChannelEvent]() 被一组[ChannelHandler](http://netty.io/4.0/api/io/netty/channel/ChannelHandler.html) 处理。这个管道是 [Intercepting Filter (拦截过滤器)](http://java.sun.com/blueprints/corej2eepatterns/Patterns/InterceptingFilter.html)模式的一种高级形式的实现，因此对于一个事件如何被处理以及管道内部处理器间的交互过程，你都将拥有绝对的控制力。例如，你可以定义一个从 socket 读取到数据后的操作：

	public class MyReadHandler implements SimpleChannelHandler {
	  	 public void messageReceived(ChannelHandlerContext ctx, MessageEvent evt) {
	         Object message = evt.getMessage();
	  		 // Do something with the received message.
	            ...
	  	
	         // And forward the event to the next handler.
	         ctx.sendUpstream(evt);
	    }
	}

同时你也可以定义一种操作响应其他处理器的写操作请求：

	public class MyWriteHandler implements SimpleChannelHandler {
	  	public void writeRequested(ChannelHandlerContext ctx, MessageEvent evt) {
	        Object message = evt.getMessage();
	  		// Do something with the message to be written.
	            ...
		
	        // And forward the event to the next handler.
	        ctx.sendDownstream(evt);
	    }
	}

有关事件模型的更多信息，请参考 API 文档 ChannelEvent 和ChannelPipeline 部分。