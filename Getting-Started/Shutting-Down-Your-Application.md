Shutting Down Your Application 关闭你的应用
===============

关闭一个 Netty 应用往往只需要简单地通过 shutdownGracefully() 方法来关闭你构建的所有的 [EventLoopGroup](http://netty.io/4.0/api/io/netty/channel/EventLoopGroup.html)。当EventLoopGroup 被完全地终止,并且对应的所有 [channel](http://netty.io/4.0/api/io/netty/channel/Channel.html) 都已经被关闭时，Netty 会返回一个[Future](http://netty.io/4.0/api/io/netty/util/concurrent/Future.html)对象来通知你。