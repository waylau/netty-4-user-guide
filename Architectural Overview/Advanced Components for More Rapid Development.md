Advanced Components for More Rapid Development 适用快速开发的高级组件
==========

上述所提及的核心组件已经足够实现各种类型的网络应用，除此之外，Netty 也提供了一系列的高级组件来加速你的开发过程。

##Codec 框架

就像“[使用POJO代替ChannelBuffer](../Getting Started/Speaking in POJO instead of ByteBuf.md)”一节所展示的那样，从业务逻辑代码中分离协议处理部分总是一个很不错的想法。然而如果一切从零开始便会遭遇到实现上的复杂性。你不得不处理分段的消息。一些协议是多层的（例如构建在其他低层协议之上的协议）。一些协议过于复杂以致难以在一台独立状态机上实现。

因此，一个好的网络应用框架应该提供一种可扩展，可重用，可单元测试并且是多层的 codec 框架，为用户提供易维护的 codec 代码。

Netty 提供了一组构建在其核心模块之上的 codec 实现，这些简单的或者高级的 codec 实现帮你解决了大部分在你进行协议处理开发过程会遇到的问题，无论这些协议是简单的还是复杂的，二进制的或是简单文本的。

##SSL / TLS 支持

不同于传统阻塞式的 I/O 实现，在 NIO 模式下支持 SSL 功能是一个艰难的工作。你不能只是简单的包装一下流数据并进行加密或解密工作，你不得不借助于 javax.net.ssl.SSLEngine，SSLEngine 是一个有状态的实现，其复杂性不亚于 SSL 自身。你必须管理所有可能的状态，例如密码套件，密钥协商（或重新协商），证书交换以及认证等。此外，与通常期望情况相反的是 SSLEngine 甚至不是一个绝对的线程安全实现。

在 Netty 内部，[SslHandler](http://netty.io/4.0/api/io/netty/handler/ssl/SslHandler.html) 封装了所有艰难的细节以及使用 SSLEngine 可 能带来的陷阱。你所做的仅是配置并将该 SslHandler 插入到你的  [ChannelPipeline](http://netty.io/4.0/api/io/netty/channel/ChannelPipeline.html) 中。同样 Netty 也允许你实现像 [StartTlS](http://en.wikipedia.org/wiki/Starttls) 那样所拥有的高级特性，这很容易。

##HTTP 实现

HTTP无 疑是互联网上最受欢迎的协议，并且已经有了一些例如 Servlet 容器这样的 HTTP 实现。因此，为什么 Netty 还要在其核心模块之上构建一套 HTTP 实现？

与现有的 HTTP 实现相比 Netty 的 HTTP 实现是相当与众不同的。在HTTP 消息的低层交互过程中你将拥有绝对的控制力。这是因为 Netty 的HTTP 实现只是一些 HTTP codec 和 HTTP 消息类的简单组合，这里不存在任何限制——例如那种被迫选择的线程模型。你可以随心所欲的编写那种可以完全按照你期望的工作方式工作的客户端或服务器端代码。这包括线程模型，连接生命期，快编码，以及所有 HTTP 协议允许你做的，所有的一切，你都将拥有绝对的控制力。

由于这种高度可定制化的特性，你可以开发一个非常高效的HTTP服务器，例如：

* 要求持久化链接以及服务器端推送技术的聊天服务（如，[Comet](http://en.wikipedia.org/wiki/Comet_%28programming%29) )
* 需要保持链接直至整个文件下载完成的媒体流服务（如，2小时长的电影）
* 需要上传大文件并且没有内存压力的文件服务（如，上传1GB文件的请求）
* 支持大规模混合客户端应用用于连接以万计的第三方异步 web 服务。

##WebSockets 实现

[WebSockets](http://en.wikipedia.org/wiki/WebSockets) 允许双向，全双工通信信道，在 TCP socket 中。它被设计为允许一个 Web 浏览器和 Web 服务器之间通过数据流交互。

WebSocket 协议已经被 IETF 列为 [RFC 6455](http://tools.ietf.org/html/rfc6455)规范。

Netty 实现了 RFC 6455 和一些老版本的规范。请参阅[io.netty.handler.codec.http.websocketx](http://netty.io/4.0/api/io/netty/handler/codec/http/websocketx/package-frame.html)包和相关的[例子](http://static.netty.io/3.5/xref/org/jboss/netty/example/http/websocketx/server/package-summary.html)。

##Google Protocol Buffer 整合

[Google Protocol Buffers](http://code.google.com/apis/protocolbuffers/docs/overview.html) 是快速实现一个高效的二进制协议的理想方案。通过使用 [ProtobufEncoder](http://netty.io/4.0/api/io/netty/handler/codec/protobuf/ProtobufEncoder.html) 和 [ProtobufDecoder](http://netty.io/4.0/api/io/netty/handler/codec/protobuf/ProtobufDecoder.html)，你可以把 Google Protocol Buffers 编译器 (protoc) 生成的消息类放入到 Netty 的codec 实现中。请参考“[LocalTime](http://docs.jboss.org/netty/3.2/xref/org/jboss/netty/example/localtime/package-summary.html)”实例，这个例子也同时显示出开发一个由简单协议定义 的客户及服务端是多么的容易。
 

*译者注：翻译版本的项目源码见 <https://github.com/waylau/netty-4-user-guide-demos>* 