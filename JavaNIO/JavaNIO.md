# 缓冲区

## 属性

* 容量 Capacity 缓冲区能容纳的数据元素数量，创建时被设定，无法改变
* 上界Limit 缓冲区第一个不能被读或写的元素，即缓冲区现存元素的计数
	
	写模式下Buffer的limit表示你能最多往Buffer里写多少数据，limit等于capacity。
	
	切换到读模式下，limit表示能读多少数据，limit会被设置成写模式下的position值，你能读到之前写入的所有数据。
	
* 位置Position 下一个要被读或写的元素索引，由get和put方法更新
	
	写数据到Buffer中时，position表示当前的位置，初始position为0，当一个byte、long等数据写到Buffer后，position会向前移动到下一个可插入数据的Buffer单元，position最大可为capacity - 1。
	
	当读取数据时，从某个特定位置读，当将Buffer从写模式切换到读模式，position会被重置为0，当从buffer的position处读取数据时，position向前移动到下一个可读的位置。
	
* 标记Mark 备忘位置 mark()设定 mark=position，reset()设定position=mark，标记在设定前是未定义的

0 <= mark <= position <=limit <=capacity

## 缓冲区API

> 源码Jdk1.7

```
public abstract class Buffer {

    //mark <= position <= limit <= capacity
    //标记
    private int mark = -1;
    //位置
    private int position = 0;
    //上界
    private int limit;
    //容量
    private int capacity;

    //仅用于direct buffers
    long address;

    Buffer(int mark, int pos, int lim, int cap) {
        if (cap < 0)
            throw new IllegalArgumentException("Negative capacity: " + cap);
        this.capacity = cap;
        limit(lim);
        position(pos);
        if (mark >= 0) {
            if (mark > pos)
                throw new IllegalArgumentException("mark > position: ("
                                                   + mark + " > " + pos + ")");
            this.mark = mark;
        }
    }

    //返回容量
    public final int capacity() {
        return capacity;
    }

    //返回位置
    public final int position() {
        return position;
    }

    //设置位置
    public final Buffer position(int newPosition) {
        if ((newPosition > limit) || (newPosition < 0))
            throw new IllegalArgumentException();
        position = newPosition;
        if (mark > position) mark = -1;
        return this;
    }

    //返回上界
    public final int limit() {
        return limit;
    }

    //设置上界
    public final Buffer limit(int newLimit) {
        if ((newLimit > capacity) || (newLimit < 0))
            throw new IllegalArgumentException();
        limit = newLimit;
        if (position > limit) position = limit;
        if (mark > limit) mark = -1;
        return this;
    }

    //设置标记
    public final Buffer mark() {
        mark = position;
        return this;
    }

    //重置位置
    public final Buffer reset() {
        int m = mark;
        if (m < 0)
            throw new InvalidMarkException();
        position = m;
        return this;
    }

    /清空缓冲区
    public final Buffer clear() {
        position = 0;
        limit = capacity;
        mark = -1;
        return this;
    }

    //flip方法将Buffer从写模式切换到读模式。调用flip()方法会将position设回0，并将limit设置成之前position的值。

	//换句话说，position现在用于标记读的位置，limit表示之前写进了多少个byte、char等 —— 现在能读取多少个byte、char等。
    public final Buffer flip() {
        limit = position;
        position = 0;
        mark = -1;
        return this;
    }

    //将position设为0，可以重读buffer中所有数据，limit保持不变，表示能从Buffer中读取多少个元素
    public final Buffer rewind() {
        position = 0;
        mark = -1;
        return this;
    }

    //返回当前position和limit之间的数据数
    public final int remaining() {
        return limit - position;
    }

    //是否还有数据
    public final boolean hasRemaining() {
        return position < limit;
    }

    //是否是只读的
    public abstract boolean isReadOnly();

    //是否是可访问数组支持
    public abstract boolean hasArray();

    //返回数组
    public abstract Object array();

    public abstract int arrayOffset();

    //是否是direct Buffer
    public abstract boolean isDirect();


    // -- Package-private methods for bounds checking, etc. --
    final int nextGetIndex() {
        if (position >= limit)
            throw new BufferUnderflowException();
        return position++;
    }

    final int nextGetIndex(int nb) {
        if (limit - position < nb)
            throw new BufferUnderflowException();
        int p = position;
        position += nb;
        return p;
    }

       final int nextPutIndex() {
        if (position >= limit)
            throw new BufferOverflowException();
        return position++;
    }

    final int nextPutIndex(int nb) { 
        if (limit - position < nb)
            throw new BufferOverflowException();
        int p = position;
        position += nb;
        return p;
    }

        final int checkIndex(int i) {
        if ((i < 0) || (i >= limit))
            throw new IndexOutOfBoundsException();
        return i;
    }

    final int checkIndex(int i, int nb) {
        if ((i < 0) || (nb > limit - i))
            throw new IndexOutOfBoundsException();
        return i;
    }

    final int markValue() { 
        return mark;
    }

    final void truncate() {
        mark = -1;
        position = 0;
        limit = 0;
        capacity = 0;
    }

    final void discardMark() {
        mark = -1;
    }

    static void checkBounds(int off, int len, int size) {
        if ((off | len | (off + len) | (size - (off + len))) < 0)
            throw new IndexOutOfBoundsException();
    }

}
```

## 比较

两个缓冲区相等的充要条件：

* 两个对象类型相同。
* 两个对象都剩余同样数量的元素。
* 在每个缓冲区中应被get函数返回的剩余数据元素序列必须一致。

## 创建缓冲区

以CharBuffer为例，对于IntBuffer，DoubleBuffer，ShortBuffer，LongBuffer，FloatBuffer，ByteBuffer也适用。

分配容量为100个char变量的CharBuffer：

`CharBuffer charBuffer = CharBuffer.allocate(100);`

隐含的从堆中分配了一个char型数组作为备份存储器来存储100个char变量。

用自已的数组做缓冲区的备份存储器，使用wrap()方法：

```
char [] myArray = new char[100];
CharBuffer charBuffer = CharBuffer.wrap(myArray);
```

通过allocate()或wrap()方法创建的缓冲区通常都是间接的，使用备份数组。

hasArray()方法表示是否有一个可存取的备份数组，如果返回true，array()方法会返回这个缓冲区对象所使用的数组存储空间的引用。

## 复制缓冲区

duplicat()方法创建一个与原始缓冲区相似的新缓冲区，两个缓冲区共享数据元素，有同样的容量，每个缓冲区有各自的位置，上界，标记。对一个缓冲区内的数据元素做的改变会反应在另一个缓冲区上。

slice()，分割缓冲区，此方法创建一个从原始缓冲区的当前位置开始的新缓冲区，并且其容量是原始缓冲区的剩余元素数量（limit - position）。这个新缓冲区与原始缓冲区共享一段数据元素子序列。

## 字节缓冲区

### 直接缓冲区
直接缓冲区被用于与通道和固有I/O例程交互。

直接ByteBuffer通过ByteBuffer.allocateDirect()方法创建。

# 通道Channel

channel

```
public interface Channel extends Closeable {

    //检查一个通道是否打开
    public boolean isOpen();

    //关闭一个打开的通道
    public void close() throws IOException;

}
```

InterruptibleChannel是一个标记接口，当被通道使用时，可标志该通道是可以中断的。

WritableByteChannel和ReadableByteChannel，面向字节的子接口，通道只能在字节缓冲区上操作。

## 打开通道

FileChannel，SocketChannel，ServerSocketChannel，DatagramChannel。

Socket通道有可以直接创建新socket通道的工厂方法。

FileChannel对象只能通过在一个打开的RandomAccessFile，FileInputStream，FileOutputStream对象上调用getChannel()方法来获取。

```
SocketChannel sc = SocketChannel.open( );sc.connect (new InetSocketAddress ("somehost", someport));
```

```ServerSocketChannel ssc = ServerSocketChannel.open( );ssc.socket( ).bind (new InetSocketAddress (somelocalport));
```

```DatagramChannel dc = DatagramChannel.open( );
```

```RandomAccessFile raf = new RandomAccessFile ("somefile", "r");FileChannel fc = raf.getChannel( );
```

通道可以以阻塞或非阻塞方式运行，非阻塞模式的通道永远不会让调用的线程休眠，请求的操作要么立即完成，要么返回一个结果表明未进行任何操作，只有面向流的通道如sockets和pipe才能使用非阻塞模式。

socket 通道类从 SelectableChannel 引申而来。从 SelectableChannel 引申而 来的类可以和支持有条件的选择(readiness selectio)的选择器(Selectors)一起使用。将非阻塞 I/O 和选择器组合起来可以使您的程序利用多路复用 I/O(multiplexed I/O)。

## Scatter/Gather
是指在多个缓冲区上实现一个简单的I/O操作。

对于一个write操作，数据是从几个缓冲区按顺序抽取（gather），并沿着通道发送的；

对read操作，从通道读取的数据会按顺序被散步（scatter）到多个缓冲区，将每个缓冲区填满直至通道中的数据或缓冲区的最大空间被消耗完。

## 文件通道
文件通道总是阻塞式的，不能被设置为非阻塞。

FileChannel对象是线程安全的。多个进程可以在同一个实例上并发调用方法而 不会引起任何问题,不过并非所有的操作都是多线程的(multithreaded)。影响通道位置或者影响 文件大小的操作都是单线程的(single-threaded)。如果有一个线程已经在执行会影响通道位置或文 件大小的操作,那么其他 试进行此类操作之一的线程必须等待。并发行为也会受到底层的操作系 统或文件系统影响。

每个 FileChannel 都有一个 “file position”的概念。这个 position 值 决定文件中哪一处的数据接下来将被读或者写。


## 内存映射文件
通过内存映射机制来访问一个文件会比使用常规方法读写高效得多,甚至比使用通道的效率都高。因为不需要做明确的系统调用,那会很消耗时间。更重要的是,操作系统的虚拟内存可以自动 缓存内存页(memory page)。这些页是用系统内存来缓存的,所以不会消耗 Java 虚拟机内存  (memory heap)。


## Channel-to-Channel 传输

transferTo( )和 transferFrom( )方法允许将一个通道交叉连接到另一个通道,而不需要通过一个 中间缓冲区来传递数据。只有 FileChannel 类有这两个方法,因此 channel-to-channel 传输中通道之 一必须是 FileChannel。

## Socket通道
新的 socket 通道类可以运行非阻塞模式并且是可选择的。

### ServerSocketChannel
ServerSocketChannel 是一个基于通道的 socket 监听器。它同我们所熟悉的 java.net.ServerSocket 执行相同的基本任务,不过它增加了通道语义,因此能够在非阻塞模式下运行。

### SocketChannel

Socket 通道是线程安全的。并发访问时无需特别措施来保护发起访问的多个线程,不过任何时候都只有一个读操作和一个写操作在进行中。

### DatagramChannel
DatagramChannel 则模拟包导向的无连接协议(如 UDP/IP)

## 管道


# 选择器

