Writing a Discard Server 写个丢弃服务器
========================

世上最简单的协议不是'Hello, World!' 而是 [DISCARD(丢弃)](http://tools.ietf.org/html/rfc863)。这个协议将会丢掉任何收到的数据，而不响应。

为了实现 DISCARD 协议，你只需忽略所有收到的数据。让我们从 handler （处理器）的实现开始，handler 是由 Netty 生成用来处理 I/O 事件的。