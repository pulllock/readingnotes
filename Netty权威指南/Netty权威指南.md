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

> 源码版本4.1.6.final

`public abstract int capacity();` 缓冲区的容量。

`public abstract ByteBuf capacity(int newCapacity);` 重新调整容量，返回新的ByteBuf；如果新的容量小于现在的容量，buffer将会被截断；如果新的容量大于现在的容量，返回新的ByteBuf，新的ByteBuf是在原来的基础上增加的，新加的容未指定数据。

`public abstract int maxCapacity();` 缓冲区最大容量。

`public abstract ByteBufAllocator alloc();` 返回创建自己的Allocator。

`public abstract ByteBuf unwrap();` 返回一个未被包装的ByteBuf。

`public abstract boolean isDirect();` 如果是直接内存，就返回true。

`public abstract boolean isReadOnly();` buffer如果只读，返回true。

`public abstract ByteBuf asReadOnly();` 返回当前ByteBuf的只读的一个版本。

`public abstract int readerIndex();` 读取索引。

`public abstract ByteBuf readerIndex(int readerIndex);` 设置读取索引。

`public abstract int writerIndex();` 写入索引。

`public abstract ByteBuf writerIndex(int writerIndex);` 设置写入索引。

`public abstract ByteBuf setIndex(int readerIndex, int writerIndex);` 设置读取和写入索引。

`public abstract int readableBytes();` 返回可读的字节数。从readerIndex到writerIndex中间的。

`public abstract int writableBytes();`  返回可写的字节数，从writerIndex到capacity之间的。

`public abstract int maxWritableBytes();` 最大的可写的字节数，从writerIndex到maxCapacity之间。

`public abstract boolean isReadable();`writerIndex减去readerIndex大于零，返回true。

`public abstract boolean isReadable(int size);` 大于或者等于指定的大小的，返回true。

`public abstract boolean isWritable();` 是否可写。

`public abstract boolean isWritable(int size);`是否可写

`public abstract ByteBuf clear();` 设置readerIndex和writerIndex为0，跟NIO buffer不同。

`public abstract ByteBuf markReaderIndex();` 标记readerIndex，可以使用resetReaderIndex来移动当前的readerIndex到标记的readerIndex。

`public abstract ByteBuf resetReaderIndex();` 重置readerIndex。

`public abstract ByteBuf markWriterIndex();`标记writerIndex。

`public abstract ByteBuf resetWriterIndex();` 重置writerIndex。

`public abstract ByteBuf discardReadBytes();`丢弃从0到readerIndex之间的bytes，移动readerIndex与writerIndex之间的数据到0开始，readerIndex=0，writerIndex=oldWriterIndex-oldReaderIndex。

`public abstract ByteBuf discardSomeReadBytes();`跟discardReadBytes类似，实际丢弃的数据跟具体实现类的逻辑相关。

`public abstract ByteBuf ensureWritable(int minWritableBytes);` 确保指定的最小可写字节数能写入。

`public abstract int ensureWritable(int minWritableBytes, boolean force);`确保指定的最小可写字节数能写入，可写并且容量没变返回0，不可写并且容量没变返回1，可写并且容增大了返回2，不可写但是容增到最大容量了返回3。

```
指定的位置返回某种类型的数据，不会改变readerIndex和writerIndex。
public abstract boolean getBoolean(int index);
public abstract byte  getByte(int index);
public abstract short getUnsignedByte(int index);
...(省略)
```

`public abstract ByteBuf getBytes(int index, ByteBuf dst);`把当前的Buffer从index开始写到dst中，直到不可写为止，不会修改当前buffer的readerIndex和writerIndex，但是会修改dst的writerIndex。

`public abstract ByteBuf getBytes(int index, ByteBuf dst, int length);` 指定了传输数据的长度。

`public abstract ByteBuf getBytes(int index, ByteBuf dst, int dstIndex, int length);` 指定了要写入的位置和要传输的长度，该方法不会修改传输和别写入的buffer的readerIndex和writerIndex。

`public abstract ByteBuf getBytes(int index, byte[] dst);`从指定位置传输到dst,不会修改当前buffer的readerIndex和writerIndex。

`public abstract ByteBuf getBytes(int index, byte[] dst, int dstIndex, int length);`

`public abstract ByteBuf getBytes(int index, ByteBuffer dst);` 被写入的ByteBuffer的position会改变。

`public abstract ByteBuf getBytes(int index, OutputStream out, int length) throws IOException;` 写入到OutputStream。

`public abstract int getBytes(int index, GatheringByteChannel out, int length) throws IOException;` 写入到GatheringByteChannel。

`public abstract int getBytes(int index, FileChannel out, long position, int length) throws IOException;` 写入到FileChannel，不会修改Channel的position。

`public abstract CharSequence getCharSequence(int index, int length, Charset charset);` 获取CharSequence。

```
在指定位置设置为指定类型的值，不会改变当前ByteBuf的readerIndex和writerIndex。
public abstract ByteBuf setBoolean(int index, boolean value);
public abstract ByteBuf setByte(int index, int value);
public abstract ByteBuf setShort(int index, int value);
...(省略)
```

```
在当前的readerIndex处获取指定类型的值，readerIndex会加一
public abstract boolean readBoolean();
public abstract byte  readByte();
...(省略)

```

`public abstract ByteBuf readSlice(int length);`
返回当前ByteBuf新创建的子区域，开始位置为readerIndex，长度为length。与原ByteBuf共享缓冲区，但是独立维护自己的readerIndex和writerIndex。

`public abstract ByteBuf readRetainedSlice(int length);` 会调用retain方法增加引用计数。

`public abstract ByteBuf readBytes(ByteBuf dst);`将当前ByteBuf的数据传输到指定的ByteBuf，读取的位置在当前的readerIndex，一直到目标ByteBuf不能写。当前的ByteBuf的readerIndex会增加，被写入的ByteBuf的writerIndex会增加

`public abstract ByteBuf readBytes(ByteBuf dst, int length);`指定读取的长度。

`public abstract ByteBuf readBytes(ByteBuf dst, int dstIndex, int length);`指定读取的长度和目标ByteBuf写入的起始位置。

`public abstract ByteBuf readBytes(byte[] dst);`写入到字节数组中去，当前ByteBuf的readerIndex会增加。

`public abstract ByteBuf readBytes(byte[] dst, int dstIndex, int length);` 指定读取的长度和写入的起始位置。

`public abstract ByteBuf readBytes(ByteBuffer dst);`从readerIndex开始读取数据到指定的ByteBuffer中去。

`public abstract ByteBuf readBytes(OutputStream out, int length) throws IOException;` 传输到OutputStream。

`public abstract int readBytes(GatheringByteChannel out, int length) throws IOException;` 传输到GatheringByteChannel中。

`public abstract CharSequence readCharSequence(int length, Charset charset);` 获取CharSequence，readerIndex会增加。

`public abstract int readBytes(FileChannel out, long position, int length) throws IOException;` 传输到FileChannel。

`public abstract ByteBuf skipBytes(int length);`跳过指定长度的bytes，readerIndex增加。

```
在当前writerIndex处写入执行类型的值，writerIndex增加。
public abstract ByteBuf writeBoolean(boolean value);
public abstract ByteBuf writeByte(int value);
public abstract ByteBuf writeShort(int value);
...(省略)
```

`public abstract ByteBuf writeBytes(ByteBuf src);` 从指定的ByteBuf中传输数据到当前的ByteBuf，从当前的writerIndex开始写入，直到源ByteBuf不可读为止。源ByteBuf的readerIndex增加。

`public abstract ByteBuf writeBytes(ByteBuf src, int length);` 指定了写入的长度。

`public abstract ByteBuf writeBytes(ByteBuf src, int srcIndex, int length);` 指定源ByteBuf开始读取的位置，读取的长度。

`public abstract ByteBuf writeBytes(byte[] src);`  从字节数组中读取。

`public abstract ByteBuf writeBytes(byte[] src, int srcIndex, int length);`

`public abstract ByteBuf writeBytes(ByteBuffer src);`

`public abstract int  writeBytes(InputStream in, int length) throws IOException;`

`public abstract int writeBytes(ScatteringByteChannel in, int length) throws IOException;`

`public abstract int writeBytes(FileChannel in, long position, int length) throws IOException;`

`public abstract ByteBuf writeZero(int length);`从writerIndex开始讲长度为length的数据置为NUL。

`public abstract int writeCharSequence(CharSequence sequence, Charset charset);`写入CharSequence。

`public abstract int indexOf(int fromIndex, int toIndex, byte value);` 查找第一个匹配value的位置，不会改变readerIndex和writerIndex。

```
从当前ByteBuf中定位出首次出现value的位置
public abstract int bytesBefore(byte value);
public abstract int bytesBefore(int length, byte value);
public abstract int bytesBefore(int index, int length, byte value);
```

```
使用指定的processor来迭代当前的ByteBuf的可读字节数组。与processor设置的查找条件进行对比，如果满足条件，则返回位置索引，否则返回-1。
public abstract int forEachByte(ByteProcessor processor);
public abstract int forEachByte(int index, int length, ByteProcessor processor);

逆序迭代
public abstract int forEachByteDesc(ByteProcessor processor);

public abstract int forEachByteDesc(int index, int length, ByteProcessor processor);
```

`public abstract ByteBuf copy();`从当前的ByteBuf复制一份新的ByteBuf，内容和索引都是独立的。复制操作不会修改原来的readerIndex和writerIndex。

`public abstract ByteBuf copy(int index, int length);`从指定的index出开始复制长度为length的数据到新的ByteBuf，新的ByteBuf和原来的相互独立，此操作不会改变原来ByteBuf的readerIndex和writerIndex。

```
public abstract ByteBuf slice();
public abstract ByteBuf slice(int index, int length);
返回当前ByteBuf的可读子缓冲区，起始位置从readerIndex到writerIndex，新的ByteBuf与原ByteBuf共享内容，读写索引独立维护。该操作不会修改原来的readerIndex和writerIndex。
```

```
public abstract ByteBuf retainedSlice();
public abstract ByteBuf retainedSlice(int index, int length);
```

`public abstract ByteBuf duplicate();` 返回当前ByteBuf的复制对象，新的与原来的ByteBuf共享缓冲区内容，但是维护自己独立的对鞋索引。

`public abstract ByteBuf retainedDuplicate();`

`public abstract ByteBuffer nioBuffer();`把当前的ByteBuf可读的缓冲区转换成ByteBuffer，共享同一个缓冲区内容引用，对ByteBuffer的读写操作不会修改原ByteBuf的读写索引，返回后的ByteBuffer无法感知原ByteBuf的动态扩展操作。

`public abstract ByteBuffer nioBuffer(int index, int length);`

`public abstract ByteBuffer internalNioBuffer(int index, int length);`内部使用，转换成ByteBuffer。

`public abstract ByteBuffer[] nioBuffers();`转换成ByteBuffer数组。

`public abstract ByteBuffer[] nioBuffers(int index, int length);`

`public abstract boolean hasArray();`判断是否有字节数组，如果返回true可以安全的使用array()和arrayOffser()方法。

`public abstract byte[] array();`返回字节数组。

`public abstract int arrayOffset();`

`public abstract boolean hasMemoryAddress();`判断是否有指向内存地址的引用。

`public abstract long memoryAddress();` 返回内存地址。

### AbstractByteBuf源码分析
AbstractByteBuf继承自ByteBuf，ByteBuf的一些公共属性和功能会在AbstractByteBuf中实现。
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

> 源码版本4.1.6.final

`static final ResourceLeakDetector<ByteBuf> leakDetector =
            ResourceLeakDetectorFactory.instance().newResourceLeakDetector(ByteBuf.class);` 用于检测对象是否泄漏，被定义为static，所有的ByteBuf实例共享同一个ResourceLeakDetector对象。

##### 重用缓冲区
```
@Override
public ByteBuf discardReadBytes() {
	//确保可以访问
    ensureAccessible();
    //读索引为0，没有可重用的缓冲区，直接返回。
    if (readerIndex == 0) {
        return this;
    }
	//readerIndex不等于writerIndex，说明缓冲区中有已经读过的被丢弃的缓冲区，也有尚未读取的可读缓冲区。
    if (readerIndex != writerIndex) {
    	//字节数组的复制，尚未读取的字节数组复制到缓冲区的起始位置。
        setBytes(0, this, readerIndex, writerIndex - readerIndex);
        //writerIndex设置为之前的writerIndex减去readerIndex，既是重用缓冲区的长度。
        writerIndex -= readerIndex;
        //重新调整markedReaderIndex和markedWriterIndex
        adjustMarkers(readerIndex);
        //readerIndex变成了0
        readerIndex = 0;
    } else {//如果readerIndex和writerIndex相等，说明没有可读的字节数组，不需要复制数组，直接调整mark。
    	//重新调整markedReaderIndex和markedWriterIndex
        adjustMarkers(readerIndex);
        //读写索引都设置为0。
        writerIndex = readerIndex = 0;
    }
    return this;
}
```

##### adjustMarkers()

```
protected final void adjustMarkers(int decrement) {
    int markedReaderIndex = this.markedReaderIndex;
    //备份的markedReaderIndex如果小于等于需要减少的值，markedReaderIndex设为0。
    if (markedReaderIndex <= decrement) {
        this.markedReaderIndex = 0;
        int markedWriterIndex = this.markedWriterIndex;
        //备份的markedWriterIndex如果小于等于需要减少的值，markedWriterIndex设为0。
        if (markedWriterIndex <= decrement) {
            this.markedWriterIndex = 0;
        } else {
        	//markedWriterIndex减去需要减少的值就是新的markedWriterIndex。
            this.markedWriterIndex = markedWriterIndex - decrement;
        }
    } else {
    	//如果需要减小的值小于markedReaderIndex，则也一定小于markedWriterIndex，新值就是减去需要减小的值之后的取值。
        this.markedReaderIndex = markedReaderIndex - decrement;
        markedWriterIndex -= decrement;
    }
}
```

##### ensureWritable
确保能够写入指定的大小

```
public ByteBuf ensureWritable(int minWritableBytes) {
    if (minWritableBytes < 0) {
        throw new IllegalArgumentException(String.format(
                "minWritableBytes: %d (expected: >= 0)", minWritableBytes));
    }
    //实现方法
    ensureWritable0(minWritableBytes);
    return this;
}
```

```
private void ensureWritable0(int minWritableBytes) {
	//writableBytes()==capacity() - writerIndex;
	//要写入的字节数组长度小于当前ByteBuf可写的字节数，说明可以写入，直接返回。
    if (minWritableBytes <= writableBytes()) {
        return;
    }
	//如果要写入的字节数组大于可动态扩展的最大可写字节数，说明缓冲区无法写入超过最大容量的字节数组，抛异常。
    if (minWritableBytes > maxCapacity - writerIndex) {
        throw new IndexOutOfBoundsException(String.format(
                "writerIndex(%d) + minWritableBytes(%d) exceeds maxCapacity(%d): %s",
                writerIndex, minWritableBytes, maxCapacity, this));
    }

    //要写入的字节数虽然大于当前ByteBuf可写的字节数，但是可以通过动态扩展满足新的写入请求，则进行动态扩展。
    int newCapacity = alloc().calculateNewCapacity(writerIndex + minWritableBytes, maxCapacity);

    //重新设置为新的容量
    capacity(newCapacity);
}
```

##### calculateNewCapacity

```
//此方法实现位于AbstractByteBufAllocator类
public int calculateNewCapacity(int minNewCapacity, int maxCapacity) {
    if (minNewCapacity < 0) {
        throw new IllegalArgumentException("minNewCapacity: " + minNewCapacity + " (expectd: 0+)");
    }
    //满足要求的最小容量比最大可扩展的容量大，直接抛异常。
    if (minNewCapacity > maxCapacity) {
        throw new IllegalArgumentException(String.format(
                "minNewCapacity: %d (expected: not greater than maxCapacity(%d)",
                minNewCapacity, maxCapacity));
    }
    //门限阈值为4M
    final int threshold = 1048576 * 4; // 4 MiB page
	//最小需要的新容量正好等于阈值，使用阈值作为新缓冲区的容量。
    if (minNewCapacity == threshold) {
        return threshold;
    }

    //最小需要的新容量大于阈值，不能采用倍增的方式扩展内存，而采用每次步进4MB的方式进行内存扩容。
    if (minNewCapacity > threshold) {
    	//扩容后的新容量
        int newCapacity = minNewCapacity / threshold * threshold;
        //新容量大于缓冲区的最大长度，使用maxCapacity做为扩容后的缓冲区容量。
        if (newCapacity > maxCapacity - threshold) {
            newCapacity = maxCapacity;
        } else {
            newCapacity += threshold;
        }
        return newCapacity;
    }

    // 最小需要的新容量小于阈值，以64为计数进行倍增，直到倍增后的结果大于或者等于需要的容量值。
    int newCapacity = 64;
    while (newCapacity < minNewCapacity) {
        newCapacity <<= 1;
    }

    return Math.min(newCapacity, maxCapacity);
}
```

##### ensureWritable(int minWritableBytes, boolean force)

```
public int ensureWritable(int minWritableBytes, boolean force) {
    if (minWritableBytes < 0) {
        throw new IllegalArgumentException(String.format(
                "minWritableBytes: %d (expected: >= 0)", minWritableBytes));
    }
	//要写的最小容量小于等于当前可写的字节数，说明可写入，返回0。
    if (minWritableBytes <= writableBytes()) {
        return 0;
    }
	//如果要写入的字节数组大于可动态扩展的最大可写字节数，说明缓冲区无法写入超过最大容量的字节数组。
    if (minWritableBytes > maxCapacity - writerIndex) {
        if (force) {
        		//容量等于最大容量，返回1
            if (capacity() == maxCapacity()) {
                return 1;
            }
			//扩容成最大容量，返回3
            capacity(maxCapacity());
            return 3;
        }
    }

    //要写入的字节数虽然大于当前ByteBuf可写的字节数，但是可以通过动态扩展满足新的写入请求，则进行动态扩展。
    int newCapacity = alloc().calculateNewCapacity(writerIndex + minWritableBytes, maxCapacity);

    //扩展新容量，返回2
    capacity(newCapacity);
    return 2;
}
```


### AbstractReferenceCountedByteBuf源码解析
该类主要是对引用进行计数，类似JVM内存回收的对象引用计数器，用于跟踪对象的分配和销毁，做自动内存回收。

#### 成员变量

`private static final AtomicIntegerFieldUpdater<AbstractReferenceCountedByteBuf> refCntUpdater;` 通过原子的方式对成员变量进行更新操作，以实现线程安全，消除锁。

`private volatile int refCnt = 1;` refCnt字段用于跟踪对象的引用次数。

#### 对象引用计数器
每调用一次retain方法，引用计数器就加1。

release方法释放引用计数器。

> 源码版本4.1.6.final


`private static final AtomicIntegerFieldUpdater<AbstractReferenceCountedByteBuf> refCntUpdater;` 通过原子的方式对成员变量进行更新操作，以实现线程安全，消除锁。

`private volatile int refCnt = 1;` refCnt字段用于跟踪对象的引用次数。

##### retain()

```
//每调用一次该方法，引用计数器就会加1
public ByteBuf retain() {
    return retain0(1);
}
```

##### retain0()

```
private ByteBuf retain0(int increment) {
    for (;;) {
    	//当前的引用计数器
        int refCnt = this.refCnt;
        //增加之后的引用计数器
        final int nextCnt = refCnt + increment;

        // Ensure we not resurrect (which means the refCnt was 0) and also that we encountered an overflow.
        if (nextCnt <= increment) {
            throw new IllegalReferenceCountException(refCnt, increment);
        }
        //CAS增加
        if (refCntUpdater.compareAndSet(this, refCnt, nextCnt)) {
            break;
        }
    }
    return this;
}
```
##### release()

```
//调用release0方法，释放引用计数器
public boolean release() {
    return release0(1);
}
```

##### release0()

```
private boolean release0(int decrement) {
    for (;;) {
    	//当前引用计数
        int refCnt = this.refCnt;
        //要减少的大于当前引用计数，抛异常
        if (refCnt < decrement) {
            throw new IllegalReferenceCountException(refCnt, -decrement);
        }
		//CAS进行操作
        if (refCntUpdater.compareAndSet(this, refCnt, refCnt - decrement)) {
        	//申请和释放的相等，该对象需要被释放和垃圾回收。
            if (refCnt == decrement) {
            	//释放ByteBuf对象。
                deallocate();
                return true;
            }
            return false;
        }
    }
}
```

`protected abstract void deallocate();` 释放ByteBuf对象。


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

#### 源码分析 UnpooledHeapByteBuf

> 源码版本4.1.6.final


`private final ByteBufAllocator alloc;` 用于UnpooledHeapByteBuf的内存分配。

`private byte[] array;` 作为缓冲区。

`private ByteBuffer tmpNioBuf;` 用于实现Netty ByteBuf到JDK NIO ByteBuffer的转换。

##### capacity()

```
public ByteBuf capacity(int newCapacity) {
    ensureAccessible();
    if (newCapacity < 0 || newCapacity > maxCapacity()) {
        throw new IllegalArgumentException("newCapacity: " + newCapacity);
    }
	//原来的容量
    int oldCapacity = array.length;
    //新容量比原来的大，扩容
    if (newCapacity > oldCapacity) {
    	//新的字节数组
        byte[] newArray = new byte[newCapacity];
        //复制数据
        System.arraycopy(array, 0, newArray, 0, array.length);
        //需要将原来的tmpNioBuf设置为空。
        setArray(newArray);
    } else if (newCapacity < oldCapacity) {
    //新的容量小于旧的容量，不需要动态扩容，但是需要截取当前缓冲区创建一个新的子缓冲区
    	//新的字节数组
        byte[] newArray = new byte[newCapacity];
        //原来的readerIndex。
        int readerIndex = readerIndex();
        //readerIndex小于新的容量
        if (readerIndex < newCapacity) {
            int writerIndex = writerIndex();
            //writerIndex大于新容量，则将写索引设置为新的容量值。
            if (writerIndex > newCapacity) {
                writerIndex(writerIndex = newCapacity);
            }
            //将可读的字节数组复制到新的缓冲区去。
            System.arraycopy(array, readerIndex, newArray, readerIndex, writerIndex - readerIndex);
        } else {
        	//readerIndex大于新的容量，没有可读的字节数组，将读写索引设置为新容量即可。
            setIndex(newCapacity, newCapacity);
        }
        //替换原来的字节数组。
        setArray(newArray);
    }
    return this;
}
```

##### getBytes(int index, ByteBuf dst, int dstIndex, int length)

把当前ByteBuf的数据从index处开始传输到指定的ByteBuf中的指定destIndex位置处，长度为length，此操作不会改变原来的和目的ByteBuf的readerIndex和writerIndex。

```
public ByteBuf getBytes(int index, ByteBuf dst, int dstIndex, int length) {
	//首先检查目标ByteBuf的要写的index等。
    checkDstIndex(index, length, dstIndex, dst.capacity());
    //如果有在内存中的数据，使用系统底层的方法区复制内存数据，使用Unsafe中的方法。
    if (dst.hasMemoryAddress()) {
        PlatformDependent.copyMemory(array, index, dst.memoryAddress() + dstIndex, length);
    } else if (dst.hasArray()) {
    	//如果目标ByteBuf含有字节数组数据，需要将拷贝的位置移动到已存在的Array的长度加上dstIndex位置处。
        getBytes(index, dst.array(), dst.arrayOffset() + dstIndex, length);
    } else {
    	//目标ByteBuf不存在数据，调用dst.setBytes方法
    	//将array中的数据传输到dst中。
        dst.setBytes(dstIndex, array, index, length);
    }
    return this;
}
```

##### nioBuffer

```
//转换成ByteBuffer，调用ByteBuffer的warp包装成ByteBuffer，然后调用slice()方法复制一份
public ByteBuffer nioBuffer(int index, int length) {
    ensureAccessible();
    return ByteBuffer.wrap(array, index, length).slice();
}
```

##### getByte(int index)

```
public byte getByte(int index) {
    ensureAccessible();
    return _getByte(index);
}
//调用HeapByteBufUtil中的静态方法。
protected byte _getByte(int index) {
    return HeapByteBufUtil.getByte(array, index);
}
```

### deallocate()
释放ByteBuf，将array直接置为null。

```
protected void deallocate() {
    array = null;
}
```

### UnpooledDirectByteBuf源码分析

基于NIO的ByteBuffer缓冲区。建议使用Unpooled.directBuffer（int）和Unpooled.wrappedBuffer（ByteBuffer的），而不是显式调用构造函数。

```
//用于UnpooledDirectByteBuf的内存分配。
private final ByteBufAllocator alloc;
//作为缓冲区
private ByteBuffer buffer;
//用于实现Netty ByteBuf到JDK NIO ByteBuffer的转换。
private ByteBuffer tmpNioBuf;
//容量
private int capacity;
//
private boolean doNotFree;
```

##### 构造方法

```
protected UnpooledDirectByteBuf(ByteBufAllocator alloc, int initialCapacity, int maxCapacity) {
    super(maxCapacity);
    ...做校验的一些代码省略
    this.alloc = alloc;
    //使用ByteBuffer.allocateDirect(initialCapacity)指定出事容量进行新创建一个DirectByteBuffer。
    setByteBuffer(ByteBuffer.allocateDirect(initialCapacity));
}

```

##### setByteBuffer(ByteBuffer buffer)

```
private void setByteBuffer(ByteBuffer buffer) {
	//原来的ByteBuffer缓冲区
    ByteBuffer oldBuffer = this.buffer;
    if (oldBuffer != null) {
        if (doNotFree) {
            doNotFree = false;
        } else {
            freeDirect(oldBuffer);
        }
    }
	//新的buffer作为缓冲区
    this.buffer = buffer;
    //要转换的置为null
    tmpNioBuf = null;
    //设置容量
    capacity = buffer.remaining();
}
```

`public boolean isDirect() {
        return true;
    }`直接返回true。

##### capacity(int newCapacity)

```
public ByteBuf capacity(int newCapacity) {
    ensureAccessible();
    //新容量小于零或者新容量大于最大的容量，抛异常。
    if (newCapacity < 0 || newCapacity > maxCapacity()) {
        throw new IllegalArgumentException("newCapacity: " + newCapacity);
    }
	//获取读索引
    int readerIndex = readerIndex();
    //获取写索引
    int writerIndex = writerIndex();
	//原来的容量
    int oldCapacity = capacity;
    //如果新的容量大于原来的容量，扩容
    if (newCapacity > oldCapacity) {
    	//旧的缓冲区
        ByteBuffer oldBuffer = buffer;
        //使用allocateDirect方法获取新的缓冲区，此方法内部使用ByteBuffer.allocateDirect获取新的缓冲区。
        ByteBuffer newBuffer = allocateDirect(newCapacity);

	//新旧缓冲区的position都设置为0，limit设置为原来缓冲区的容量。        oldBuffer.position(0).limit(oldBuffer.capacity());
        newBuffer.position(0).limit(oldBuffer.capacity());
        //把旧的缓冲区内容传输到新的缓冲区中去。
        newBuffer.put(oldBuffer);
        //将新缓冲区的position设置为0，limit设置为容量大小，mark取消。
        newBuffer.clear();
        //将缓冲区设置为新的缓冲区。
        setByteBuffer(newBuffer);
    } else if (newCapacity < oldCapacity) {//新的容量小于旧的容量，不需要扩容。
    	//原来的缓冲区
        ByteBuffer oldBuffer = buffer;
        //申请一个新的直接缓冲区
        ByteBuffer newBuffer = allocateDirect(newCapacity);
        //readerIndex小于新的缓冲区的容量
        if (readerIndex < newCapacity) {
        	//writerIndex大于新的容量，将writerIndex设置为新的容量的大小。
            if (writerIndex > newCapacity) {
                writerIndex(writerIndex = newCapacity);
            }
            	//新旧缓冲区的position都设置为readerIndex，limit设置为writerIndex
            oldBuffer.position(readerIndex).limit(writerIndex);
            newBuffer.position(readerIndex).limit(writerIndex);
            //原来的缓冲区中的内容传输到新的缓冲区
            newBuffer.put(oldBuffer);
            //将新缓冲区的position设置为0，limit设置为容量大小，mark取消。
            newBuffer.clear();
        } else {
        	//readerIndex大于新的容量，没有可读的字节数组，将读写索引设置为新容量即可。
            setIndex(newCapacity, newCapacity);
        }
        //缓冲区更新为新的缓冲区
        setByteBuffer(newBuffer);
    }
    return this;
}
```


### PooledByteBuf内存池原理分析
#### PoolArena
本身是指一块区域，内存管理中Memory Arena是指内存中的一大块连续的区域，PoolArena是Netty的内存池实现类。

Netty的PoolArena由多个Chunk组成的大块内存区域，每个Chunk由一个或者多个Page组成。

#### PoolChunk
Chunk主要用来组织和管理多个Page的内存分配和释放，Netty中Chunk的Page被构建成一颗二叉树。

#### PoolSubpage
对于小于一个Page的内存，netty在Page中完成分配。每个Page会被切分为大小相等的多个存储块，存储块的大小由第一次申请的内存块大小决定。
一个Page只能用于分配与第一次申请时大小相同的内存。
#### 内存回收策略
Chunk和Page都是通过状态位来标识内存是否可用。

### PooledDirectByteBuf
基于内存池实现，与UnPooledDirectByteBuf的唯一不同就是缓冲区的分配和销毁策略不同。

#### 创建字节缓冲区实例
由于采用内存池实现，所以新创建PooledDirectByteBuf对象时不能直接new一个实例，而是从内存池中获取，然后设置引用计数器的值。

```
static PooledDirectByteBuf newInstance(int maxCapacity) {
    PooledDirectByteBuf buf = RECYCLER.get();
    buf.reuse(maxCapacity);
    return buf;
}

PooledDirectByteBuf中的reuse方法，

final void reuse(int maxCapacity) {
	//设置最大容量
    maxCapacity(maxCapacity);
    //设置引用计数为1
    setRefCnt(1);
    //设置读写索引为0
    setIndex0(0, 0);
    //markedReaderIndex = markedWriterIndex = 0
    discardMarks();
}
```

#### 复制新的字节缓冲区实例
copy(int index,int length)复制一个新的实例，与原来的PooledDirectByteBuf独立。

```
public ByteBuf copy(int index, int length) {
    checkIndex(index, length);
    //分配一个新的ByteBuf
    ByteBuf copy = alloc().directBuffer(length, maxCapacity());
    copy.writeBytes(this, index, length);
    return copy;
}
```

调用AbstractByteBufAllocator的directBuffer方法，newDirectBuffer方法对不同的子类有不同的实现策略，如果是基于内存池的分配器，会从内存池中获取可用的ByteBuf，如果是非池，直接创建新的ByteBuf。


## ByteBuf相关的辅助类功能介绍
### ByteBufHolder
是ByteBuf的容器。

### ByteBufAllocator
字节缓冲区分配器。共有两种不同的分配器，基于内存池的字节缓冲区分配器和普通字节缓冲区分配器。

### CompositeByteBuf
允许将多个ByteBuf的实例组装到一起，形成一个统一的视图。

### ByteBufUtil
提供了一些静态方法用于操作ByteBuf对象。

# Channel和Unsafe
## Channel功能说明
包括但不限于网路的读写，客户端发起连接，主动关闭连接，链路关闭，获取通信双方的网络地址等。

### Channel的工作原理
是Netty抽象出来的网络I/O读写的相关接口。

### Channel的功能介绍
#### 网络I/O操作
1. `Channel read()` 从当前的Channel中读取数据到第一个inbound缓冲区中，如果数据被成功读取，触发`ChannelHandler.channelRead()`事件。读取完成后，接着会触发`ChannelHandler.channelReadComplete()`事件。
2. `ChannelFuture write(Object msg)` 请求将当前的msg通过ChannelPipeline写入到目标Channel中，只是将消息存入到消息发送环形数组中，没有真正被发送，调用flush才会被写入到Channel。
3. `ChannelFuture write(Object msg,ChannelPromise promise)` 携带了ChannelPromise参数负责设置写入操作的结果。
4. `ChannelFuture writeAndFlush(Object msg,ChannelPromise promise)` 将消息写入Channel中发送，等价于单独调用write和flush操作的组合。
5. `ChannelFuture writeAndFlush(Object msg)` 等同于4。
6. `Channel flush()` 将之前写入到环形发送数组中的消息全部写入到目标Channel中，发送给通信对方。
7. `ChannelFuture close(ChannelPromise promise)`主动关闭当前连接，通过ChannelPromise设置操作结果并进行结果通知，无论操作是否成功，都可以通过ChannelPromise获取操作结果。该操作会级联触发ChannelPipeline中的所有ChannelHandler的`ChannelHandler.close(ChannelHandlerContext,ChannelPromise)`事件。
8. `ChannelFuture disconnect(ChannelPromise promise)`请求断开与远程通信对端的连接并使用ChannelPromise来获取操作结果的通知消息。该方法会级联触发`ChannelHandler.disconnect(ChannelHandlerContext,ChannelPromise)`事件。
9. `ChannelFuture connect(SocketAddress remoteAddress)`客户端使用指定的服务端地址remoteAddress发起连接请求，如果连接因为应答超时而失败，ChannelFuture中的操作结果是ConnectTimeoutException异常，如果连接被拒绝，操作结果为ConnectException，该方法会级联触发`ChannelHandler.connect(ChannelHandlerConext,SocketAddress,ChannelPromise)`事件
10. `ChannelFuture connect(SocketAddress remoteAddress,SocketAddress localAddress)`
11. `ChannelFuture connect(SocketAddress remoteAddress,ChannelPromise promise)`
12. `connect(SocketAddress remoteAddress,ChannelPromise promise)`
13. `ChannelFuture bind(SocketAddress localAddress)`绑定指定的本地Socket地址localAddress，该方法会级联触发`ChannelHandler.bind(ChannelHandlerContext,SocketAddress,ChannelPromise)`事件。
14. `ChannelFuture bind(SocketAddress localAddress,ChannelPromise promise)`
15. `ChannelConfig config()` 获取当前Channel的配置信息。
16. `boolean isOpen()` 判断当前Channel是否已经打开
17. `boolean isRegistered()`判断当前Channel是否已经注册到EventLoop上。
18. `boolean isActive()` 判断当前Channel是否已经处于激活状态。
19. `ChannelMetadata metadata()`获取当前Channel的元数据描述信息。
20. `SocketAddress localAddress()`获取当前Channel的本地绑定地址。
21. `SocketAddress remoteAddress()`获取当前Channel通信的远程Socket地址。

eventLoop()，Channel需要注册到EventLoop的多路复用器上，用于处理I/O事件，通过eventLoop()方法可以获取到Channel注册的EventLoop。

Netty中每个Channel对应一个物理连接，每个连接都有自己的TCP参数配置，所以Channel会聚合一个ChannelMetadata来对TCP参数提供元数据描述信息，通过metadata()方法可以获取当前Channel的TCP参数配置。

parent()，服务端Channel的父Channel为空，客户端Channel的父Channel就是创建它的ServerSocketChannel。

id()，返回ChannelId对象，ChannelId是Channel的唯一标识。

## Channel源码分析
### AbstractChannel源码分析
AbstractChannel聚合了所有Channel使用到的能力对象，由AbstractChannel提供初始化和统一封装。

`private final Channel parent;` 父类Channel。

`private final ChannelId id;` 生成的全局ChannelId，唯一的。

`private final Unsafe unsafe;`Unsafe实例。

`private final DefaultChannelPipeline pipeline;`当前Channel对应的DefaultChannelPipeline。

`private final VoidChannelPromise unsafeVoidPromise = new VoidChannelPromise(this, false);`

`private final CloseFuture closeFuture = new CloseFuture(this);`

`private volatile EventLoop eventLoop;` 当前Channel注册的EventLoop。

#### 核心源码分析

当Channel进行I/O操作时会产生对应的I/O事件，然后驱动事件在ChannelPipeline中传播，由对应的ChannelHandler对事件进行拦截和处理，不关心的事件可以直接忽略。

网络I/O操作直接调用DefaultChannelPipeline的相关方法，由DefaultChannelPipeline中对应的ChannelHandler进行具体的逻辑处理。

```
public ChannelFuture connect(SocketAddress remoteAddress) {
    return pipeline.connect(remoteAddress);
}
```



### AbstractNioChannel源码分析

`private final SelectableChannel ch;` 
由于Nio Channel，NioSocketChannel，NioServerSocketChannel需要共用，所以定义了一个java.nio.SocketChannel和java.ServerSocketChannel的公共父类SelectableChannel，用于设置参数和I/O操作。

`protected final int readInterestOp;`readInterestOp代表了JDK SelectionKey的OP_READ。

`volatile SelectionKey selectionKey;`是Channel注册到EventLoop后返回的选择键。

`private ChannelPromise connectPromise;`代表连接操作结果。

`private ScheduledFuture<?> connectTimeoutFuture;` 连接超时定时器。

`private SocketAddress requestedRemoteAddress;` 请求的通信地址信息。
#### 核心API源码分析
##### doRegister()
Channel的注册

```
protected void doRegister() throws Exception {
	//标识注册操作是否成功
    boolean selected = false;
    for (;;) {
        try {
        //调用SelectableChannel的register方法，将当前的Channel注册到EventLoop的多路复用器上。
            selectionKey = javaChannel().register(eventLoop().selector, 0, this);
            return;
        } catch (CancelledKeyException e) {
            if (!selected) {
                //第一次处理该异常，调用多路复用器的selectNow方法将已经取消的selectionKey从多路复用器中删除掉。
                eventLoop().selectNow();
                selected = true;
            } else {
                throw e;
            }
        }
    }
}
```

### AbstractNioByteChannel源码分析
`private Runnable flushTask;` 负责继续写半包消息。

##### doWrite()

```
protected void doWrite(ChannelOutboundBuffer in) throws Exception {
    int writeSpinCount = -1;

    boolean setOpWrite = false;
    for (;;) {
    	//从发送消息环形数组ChannelOutboundBuffer弹出一条消息
        Object msg = in.current();
        if (msg == null) {
            //为空，说明消息发送数组中所有待发送的消息都已经发送完成，清除半包标识。
            clearOpWrite();
            // Directly return here so incompleteWrite(...) is not called.
            return;
        }

        if (msg instanceof ByteBuf) {
            ByteBuf buf = (ByteBuf) msg;
            int readableBytes = buf.readableBytes();
            if (readableBytes == 0) {
                in.remove();
                continue;
            }

            boolean done = false;
            long flushedAmount = 0;
            if (writeSpinCount == -1) {
                writeSpinCount = config().getWriteSpinCount();
            }
            for (int i = writeSpinCount - 1; i >= 0; i --) {
                int localFlushedAmount = doWriteBytes(buf);
                if (localFlushedAmount == 0) {
                    setOpWrite = true;
                    break;
                }

                flushedAmount += localFlushedAmount;
                if (!buf.isReadable()) {
                    done = true;
                    break;
                }
            }

            in.progress(flushedAmount);

            if (done) {
                in.remove();
            } else {
                // Break the loop and so incompleteWrite(...) is called.
                break;
            }
        } else if (msg instanceof FileRegion) {
            FileRegion region = (FileRegion) msg;
            boolean done = region.transferred() >= region.count();

            if (!done) {
                long flushedAmount = 0;
                if (writeSpinCount == -1) {
                    writeSpinCount = config().getWriteSpinCount();
                }

                for (int i = writeSpinCount - 1; i >= 0; i--) {
                    long localFlushedAmount = doWriteFileRegion(region);
                    if (localFlushedAmount == 0) {
                        setOpWrite = true;
                        break;
                    }

                    flushedAmount += localFlushedAmount;
                    if (region.transferred() >= region.count()) {
                        done = true;
                        break;
                    }
                }

                in.progress(flushedAmount);
            }

            if (done) {
                in.remove();
            } else {
                // Break the loop and so incompleteWrite(...) is called.
                break;
            }
        } else {
            // Should not reach here.
            throw new Error();
        }
    }
    incompleteWrite(setOpWrite);
}
```

### AbstractNioMessageChannel源码分析
AbstractNioMessageChannel和AbstractNioByteChannel的消息发送实现比较相似，不同之处是一个发送的是ByteBuf或者FileRegion，他们可以直接被发送，另一个发送的则是POJO对象。

### AbstractNioMessageServerChannel源码分析

### NioServerSocketChannel源码分析

### NioSocketChannel源码分析

## Unsafe功能说明
是Channel接口的辅助接口，不应被用户代码调用。实际的I/O读写操作都是由Unsafe接口负责完成的。

### AbstractUnsafe源码分析

### AbstractNioUnsafe源码分析

### NioByteUnsafe源码分析
