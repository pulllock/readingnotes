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
NIO2.0引入了新的一步通道的概念。异步通道提供两种方式获取操作结果：

* 使用java.util.concurrent.Future类来表示异步操作的结果。
* 在异步执行操作的时候传入一个java.nio.channels。

CompletionHandler接口的实现类作为操作完成的回调。

# TCP粘包/拆包
### LineBasedFrameDecoder和StringDecoder
LineBasedFrameDecoder的工作原理是它依次遍历ByteBuf中的可读字节判断看是否有`\n`或者`\r\n`，如果有就以此为结束位置，从可读索引到结束位置区间的字节就组成了一行。它是以换行符为结束标志的解码器，支持携带结束符或者不携带结束符两种解码方式，同时支持配置单行的最大长度。如果连续读取到最大长度后仍有没有发现换行符，就抛出异常，同时忽略之前读到的异常码流。

StringDecoder就是将接收到的对象转换成字符串，然后继续调用后面的handler。

可利用LineBasedFrameDecoder+StringDecoder来解决TCP的粘包/拆包问题。

# 分隔符和定长解码器的应用
## DelimiterBasedFrameDecoder
以分隔符作为码流结束标识的消息的解码。

## FixedLengthFrameDecoder
固定长度解码器，能够按照指定的长度对消息进行自动解码。

# 编解码技术
## Java序列化的缺点

* 无法跨语言
* 序列化后的码流太大
* 序列化性能太低

## 业界主流的编解码框架

* Protobuf
* Thrift
* Jboss Marshalling

# MessagePack编解码

# 服务端创建

ServerBootstrap是服务端的启动辅助类。

步骤：

1. 创建ServerBootstrap实例
2. 设置并绑定Reactor线程池
3. 设置并绑定服务端Channel
4. TCP链路建立时创建ChannelPipeline
5. 添加并设置ChannelHandler
6. 绑定监听端口并启动服务端
7. Selector轮询
8. 网络事件通知
9. 执行Netty系统和业务HandlerChannel

步骤1 创建ServerBootstrap实例，是netty服务端的启动辅助类，它提供一系列的方法用于设置服务端启动相关的参数。底层通过门面模式对各种能力进行抽象和封装。

步骤2 设置并绑定Reactor线程池，netty的Reactor线程池是EventLoopGroup，实际就是EventLoop的数组，EventLoop的职责是处理所有注册到本线程多路复用器Selector上的Channel，Selector的轮询操作由绑定的EventLoop线程run方法驱动，在一个循环体内循环执行。

步骤3 设置并绑定服务端Channel，作为NIO服务端，需要创建ServerSocketChannel，netty对原生的NIO类库进行了封装，对应实现是NioServerSocketChannel。ServerBootstrap方法提供了channel方法用于指定服务端Channel的类型。

步骤4 链路建立的时候创建并初始化ChannelPipeline，不是NIO服务端必须的，本质是一个处理网络事件的职责链，负责管理和执行ChannelHandler。网络事件以事件流的形式在ChannelPipeline中流转，由ChannelPipeline根据ChannelHandler的执行策略调度ChannelHandler的执行。

步骤5 初始化ChannelPipeline完成后，添加并设置ChannelHandler，ChannelHandler是netty提供给用户定制和扩展的接口，利用ChannelHandler可以完成大多数功能定制，如消息编解码，心跳，安全认证，TSL/SSL认证，流量控制和流量整形等。

步骤6 绑定并启动监听端口，绑定监听端口之前会做一系列的初始化和检测工作，完成之后，会启动监听端口，并将ServerSocketChannel注册到Selector上监听客户端连接。

步骤7 Selector轮询，由Reactor线程NioEventLoop负责调度和执行Selector轮询操作，选择准备就绪的Channel集合。

步骤8 当轮询到就绪的Channel后，由Reactor线程NioEventLoop执行ChannelPipeline的相应方法，最终调度并执行ChannelHandler。

步骤9 执行netty系统ChannelHandler和用户定制的ChannelHandler。


backlog指定了内核为此套接口排队的最大连接个数。

## Netty服务端创建源码分析

# ByteBuf和相关辅助类
## ByteBuf功能说明
进行数据传输的时候，需要用到缓冲区，常用的缓冲区是Jdk NIO类库的Buffer。

7种基础类型除了Boolean都有自己的缓冲区实现。主要使用ByteBuffer。

ByteBuffer缺点：

* ByteBuffer长度固定，一旦分配完成，容量不能动态扩展收缩。
* ByteBuffer只有一个标识位置的指针position，读写的时候需要手工调用flap和rewind等。
* ByteBuffer功能有限。

### ByteBuf的工作原理
ByteBuf通过两个位置指针来协助缓冲区的读写操作，读使用readerIndex，写使用writerIndex。

readerIndex和writerIndex的取值一开始都是0，随着数据的写入writerIndex会增加，读取会使readerIndex增加，但不会超过writerIndex。

在读取之后0-readerIndex被视为discard的，调用discardReadBytes方法可释放这部分空间，作用类似ByteBuffer的compact。

readerIndex和writerIndex之间的数据是可读取的，等价于ByteBuffer的position和limit之间的数据。writerIndex和capacity之间的空间是可写的，等价于ByteBuffer的limit和capacity之间的可用空间。

ByteBuf对write操作进行了封装，缓冲区不足会自动进行动态扩展。

### ByteBuf的功能介绍
#### 顺序读操作read
read操作类似于ByteBuffer的get操作。

#### 顺序写操作write
write操作类似于ByteBuffer的put操作。

#### readerIndex和writerIndex
读索引和写索引

#### Discardable bytes
调用discardReadBytes会发生字节数组的内存复制。

#### Readable bytes和Writable bytes

#### Clear操作
ByteBuffer的clear操作并不会清空缓冲区内容本身，它主要用来操作位置指针如position，limit，mark。对于ByteBuf也是操作readerIndex和writerIndex，见他们还原为初始分配的值。

#### mark和rest
ByteBuffer调用mark操作会将当前的位置指针备份到mark变量中，当调用rest操作后，重新将指针的当前位置恢复为备份在mark中的位置。

Netty的ByteBuf也有类似的rest和mark接口。

* markReaderIndex将当前的readerIndex备份到markedReaderIndex中
* resetReaderIndex将当前的readerIndex设置为markedReaderIndex
* markWriterIndex将当前writerIndex备份到markedWriterIndex
* resetWriterIndex将当前writerIndex设置为markedWriterIndex

#### 查找操作
#### Derived buffers
类似于数据库的视图

1. duplicate 返回当前ByteBuf的复制对象，复制后返回的ByteBuf与操作的ByteBuf共享缓冲区内容，但是维护自己独立的读写索引。修改复制后的内容，之前原来的内容也随着改变，双方持有的是同一个内容指针引用。
2. copy 复制一个新对象，内容和索引都是独立的，复制操作本身并不修改原来的读写索引。
3. copy(int index,int length) 从指定的索引开始复制，复制的字节长度为length，复制后的ByteBuf内容和读写索引都与之前的独立。
4. slice 返回当前ByteBuf的可读子缓冲区，起始位置从readerIndex到writerIndex，返回后的ByteBuf与原ByteBuf共享内容，但是读写索引独立维护。
5. slice(int index,int length)返回当前ByteBuf的可读子缓冲区，起始位置从index到index+length，与原ByteBuf共享内容，但是读写索引独立维护。

#### 转换成标准的ByteBuffer

1. ByteBuffer nioBuffer()将当前ByteBuf可读的缓冲区转换成ByteBuffer，两者共享同一个缓冲区内容引用，对ByteBuffer的读写操作并不会修改原ByteBuf的读写索引，返回后的ByteBuffer无法感知原ByteBuf的动态扩展操作。
2. ByteBuffer nioBuffer(int index,int length) 将当前ByteBuff从index开始长度为length的缓冲区转换成ByteBuffer。

#### 随机读写set和get
无论是get还是set操作ByteBuf都会对齐索引和长度进行合法性校验。set不支持动态扩展缓冲区。

## ByteBuf源码分析
## ByteBuf的主要继承关系
内存分配的角度：

1. 堆内存（HeapByteBuf）字节缓冲区，内存的分配和回收速度快，可以被JVM自动回收，缺点是如果进行Socket的I/O读写，需要额外做一次内存复制，将堆内存对应的缓冲区复制到内核Channel中，性能会有一定程度下降。
2. 直接内存（DirectByteBuf）字节缓冲区，在堆外进行内存分配，分配和回收速度会慢一些，但是将它写入或者从Socket Channel中读取时，速度比堆内存快。

内存回收角度：

基于对象池的ByteBuf和普通ByteBuf，基于对象池的ByteBuf可以重用ByteBuf对象。

### AbstractByteBuf源码分析
#### 主要成员变量
ResourceLeakDetector对象，定义为static，所有的ByteBuf实例共享同一个对象，用于检测对象是否泄漏。

#### 读操作簇
读操作以及其他的一些公共功能由父类实现，差异化功能由子类实现。

#### 写操作簇

#### 操作索引
与索引相关的操作主要涉及设置读写索引，mark，reset等。

#### 重用缓冲区
discardReadBytes等。

#### skipBytes
需要丢弃非法的数据报，或者跳过不需要读取的字节或者字节数组，skip非常方便。

### AbstractReferenceCountedByteBuf源码解析
该类主要是对引用进行计数，类似JVM内存回收的对象引用计数器，用于跟踪对象的分配和销毁，做自动内存回收。

#### 成员变量

`private static final AtomicIntegerFieldUpdater<AbstractReferenceCountedByteBuf> refCntUpdater;` 通过原子的方式对成员变量进行更新操作，以实现线程安全，消除锁。

`private volatile int refCnt = 1;` refCnt字段用于跟踪对象的引用次数。

#### 对象引用计数器
每调用一次retain方法，引用计数器就加1。

release方法释放引用计数器。

### UnpooledHeapByteBuf源码分析
UnpooledHeapByteBuf是基于堆内存进行内存分配的字节缓冲区，没有基于对象池，意味着每次I/O的读写都会创建一个新的UnpooledHeapByteBuf，频繁进行大块内存分配和回收会对性能造成影响。

#### 成员变量
`private final ByteBufAllocator alloc;` 用于UnpooledHeapByteBuf的内存分配。

`private byte[] array;` 作为缓冲区。

`private ByteBuffer tmpNioBuf;` 用于实现Netty ByteBuf到JDK NIO ByteBuffer的转换。

#### 动态扩展缓冲区
#### 字节数组复制
#### 转换成JDK ByteBuffer
nioBuffer 调用ByteBuffer的wrap方法。

#### 子类实现相关方法

* isDirect 如果是基于堆内存实现的ByteBuf，返回false.
* HasArray 由于UnpooledHeapByteBuf基于字节数组实现，所以返回true。
* array 由于UnpooledHeapByteBuf基于字节数组实现，所以返回的是内部的字节数组成员变量。

内存地址相关的接口主要是UnsafeByteBuf使用。

UnpooledDirectByteBuf内部缓冲区由java.nio.DirectByteBuffer实现。


