# Java的I/O演进之路
## I/O基础
### Linux网络I/O模型

1. 阻塞I/O模型
2. 非阻塞I/O模型
3. I/O复用模型
4. 信号驱动I/O模型
5. 异步I/O

### I/O多路复用技术
应用场景：

* 服务器需要同时处理多个处于监听状态或者多个连接状态的套接字。
* 服务器需要同时处理多种网络协议的套接字。

支持I/O多路复用的系统调用有select，pselect，poll，epoll。

# NIO入门
## NIO编程
### NIO类库简介

1. 缓冲区Buffer
	
	Buffer是一个对象，它包含一些要写入或者要读出的数据。
	
	缓冲区实际是一个数组，通常是一个字节数组ByteBuffer。提供了对数据的结构化访问以及维护位置等信息。
	
	最常用的缓冲区是ByteBuffer，提供了一组功能用于操作byte数组。

2. 通道Channel
	
	Channel是一个通道，网络数据通过Channel读取和写入。通道与流的不同之处在于通道是双向的，流只是在一个方向上移动，通道可以用于读，写或者二者同时进行。
	
	Channel可分为两大类：用于网络读写的SelectableChannel和用于文件操作的FileChannel。
	
3. 多路复用器Selector
	
	多路复用器提供选择已经就绪的任务的能力。会不断的轮询注册在其上的Channel，如果某个Channel上发生读或写事件，这个Channel就处于就绪状态，会被Selector轮询出来，然后通过SelectionKey可以获取就绪Channel集合，进行后续的I/O操作。
	
	一个多路复用器可以同时轮询多个Channel。
	
## AIO编程
