# TCP IP
IP首部：
![](tcp-ip/ip_header.png)
## TCP连接的建立与终止
TCP首部：
![](tcp-ip/tcp_header.png)

连接建立与终止的时序：
![](tcp-ip/connection_create_finish_sequence.png)

### 建立连接

1. 请求端(通常称为客户)发送一个SYN段指明客户打算连接的服务器的端口，以及初始序号(ISN，在这个例子中为1415531521)。这个SYN段为报文段1。
2. 服务器发回包含服务器的初始序号的SYN报文段(报文段2)作为应答。同时，将确认序号设置为客户的ISN加1以对客户的SYN报文段进行确认。一个SYN将占用一个序号。
3. 客户必须将确认序号设置为服务器的ISN加1以对服务器的SYN报文段进行确认(报文段3)。

这三个报文段完成连接的建立。这个过程也称为三次握手(three-way handshake)。

### 连接终止

图中的报文段4发起终止连接，当服务器收到这个FIN，它发回一个ACK，确认序号为收到的序号加1(报文段5)。和SYN一样，一个FIN将占用一个序号。同时TCP服务器还向应用程序传送一个文件结束符。接着这个服务器程序就关闭它的连接，导致它的TCP端发送一个FIN(报文段6)，客户必须发回一个确认，并将确认序号设置为收到序号加1(报文段7)。

## TCP一些概念和用处

- Sequence Number：是包的序号，用来解决网络包乱序（reordering）问题。
- Acknowledgement Number：就是ACK——用于确认收到，用来解决不丢包的问题。
- Window：又叫Advertised-Window，也就是著名的滑动窗口（Sliding Window），用于解决流控的。
- TCP Flag ：也就是包的类型，主要是用于操控TCP的状态机的。

## 为什么建链接要3次握手，断链接需要4次挥手？

- 对于建链接的3次握手：主要是要初始化Sequence Number 的初始值。通信的双方要互相通知对方自己的初始化的Sequence Number（缩写为ISN：Inital Sequence Number）——所以叫SYN，全称Synchronize Sequence Numbers。也就上图中的 x 和 y。这个号要作为以后的数据通信的序号，以保证应用层接收到的数据不会因为网络上的传输的问题而乱序（TCP会用这个序号来拼接数据）。
- 对于4次挥手：其实你仔细看是2次，因为TCP是全双工的，所以，发送方和接收方都需要Fin和Ack。只不过，有一方是被动的，所以看上去就成了所谓的4次挥手。如果两边同时断连接，那就会就进入到CLOSING状态，然后到达TIME_WAIT状态。

# 参考

- 《TCP/IP详解 卷一：协议》
- [http://www.52im.net/thread-513-1-1.html](http://www.52im.net/thread-513-1-1.html)