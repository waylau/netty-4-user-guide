Rich Buffer Data Structure 丰富的缓冲实现
============

Netty 使用自建的 buffer API，而不是使用 NIO 的 [ByteBuffer](http://docs.oracle.com/javase/7/docs/api/java/nio/ByteBuffer.html?is-external=true) 来表示一个连续的字节序列。与 ByteBuffer 相比这种方式拥有明显的优势。Netty 使用新的 buffer 类型 [ByteBuf](http://netty.io/4.0/api/io/netty/buffer/ByteBuf.html)，被设计为一个可从底层解决 ByteBuffer 问题，并可满足日常网络应用开发需要的缓冲类型。这些很酷的特性包括：

* 如果需要，允许使用自定义的缓冲类型。
* 复合缓冲类型中内置的透明的零拷贝实现。
* 开箱即用的动态缓冲类型，具有像 [StringBuffer](http://docs.oracle.com/javase/7/docs/api/java/lang/StringBuffer.html?is-external=true) 一样的动态缓冲能力。
* 不再需要调用的flip()方法。
* 正常情况下具有比 ByteBuffer 更快的响应速度。

更多信息请参考：[io.netty.buffer 包描述](http://netty.io/4.0/api/io/netty/buffer/package-summary.html#package_description)

###Extensibility 可扩展性

ByteBuf 具有丰富的操作集,可以快速的实现协议的优化。例如，ByteBuf 提供各种操作用于访问无符号值和字符串，以及在缓冲区搜索一定的字节序列。你也可以扩展或包装现有的缓冲类型用来提供方便的访问。自定义缓冲仍然实现自 ByteBuf 接口，而不是引入一个不兼容的类型

###Transparent Zero Copy 透明的零拷贝

举一个网络应用到极致的表现，你需要减少内存拷贝操作次数。你可能有一组缓冲区可以被组合以形成一个完整的消息。网络提供了一种复合缓冲，允许你从现有的任意数的缓冲区创建一个新的缓冲区而无需内存拷贝。例如，一个信息可以由两部分组成；header 和 body。在一个模块化的应用，当消息发送出去时，这两部分可以由不同的模块生产和装配。

<pre> +--------+----------+
 | header |   body   |
 +--------+----------+
 </pre>

如果你使用的是 ByteBuffer ，你必须要创建一个新的大缓存区用来拷贝这两部分到这个新缓存区中。或者，你可以在 NiO做一个收集写操作，但限制你将复合缓冲类型作为 ByteBuffer 的数组而不是一个单一的缓冲区，打破了抽象，并且引入了复杂的状态管理。此外，如果你不从 NIO channel 读或写，它是没有用的。

	// 复合类型与组件类型不兼容。
	ByteBuffer[] message = new ByteBuffer[] { header, body };

通过对比， ByteBuf 不会有警告，因为它是完全可扩展并有一个内置的复合缓冲区。

	// 复合类型与组件类型是兼容的。
	ByteBuf message = Unpooled.wrappedBuffer(header, body);
	
	// 因此，你甚至可以通过混合复合类型与普通缓冲区来创建一个复合类型。
	ByteBuf messageWithFooter = Unpooled.wrappedBuffer(message, footer);
	
	// 由于复合类型仍是 ByteBuf，访问其内容很容易，
	//并且访问方法的行为就像是访问一个单独的缓冲区，
	//即使你想访问的区域是跨多个组件。
	//这里的无符号整数读取位于 body 和 footer
	messageWithFooter.getUnsignedInt(
	     messageWithFooter.readableBytes() - footer.readableBytes() - 1);

###Automatic Capacity Extension 自动容量扩展

许多协议定义可变长度的消息，这意味着没有办法确定消息的长度，直到你构建的消息。或者，在计算长度的精确值时，带来了困难和不便。这就像当你建立一个字符串。你经常估计得到的字符串的长度，让 StringBuffer 扩大了其本身的需求。

	// 一种新的动态缓冲区被创建。在内部，实际缓冲区是被“懒”创建，从而避免潜在的浪费内存空间。
	ByteBuf b = Unpooled.buffer(4);
	
	// 当第一个执行写尝试，内部指定初始容量 4 的缓冲区被创建
	b.writeByte('1');
	
	b.writeByte('2');
	b.writeByte('3');
	b.writeByte('4');

	// 当写入的字节数超过初始容量 4 时，
	//内部缓冲区自动分配具有较大的容量
	b.writeByte('5');

###Better Performance 更好的性能

最频繁使用的缓冲区  ByteBuf 的实现是一个非常薄的字节数组包装器（比如，一个字节）。与 ByteBuffer 不同，它没有复杂的边界和索引检查补偿，因此对于 JVM 优化缓冲区的访问更加简单。更多复杂的缓冲区实现是用于拆分或者组合缓存，并且比 ByteBuffer 拥有更好的性能。
