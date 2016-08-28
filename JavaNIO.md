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