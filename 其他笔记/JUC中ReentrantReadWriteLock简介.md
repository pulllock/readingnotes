ReentrantLock实现了标准的互斥操作，也就是在某一时刻只有一个线程持有锁。ReentrantLock在一定程度上减低了吞吐量，独占的情况下，任何读读，读写，写写都不能同时发生。

ReadWriteLock维护了一对相关的锁，一个只读，一个写入操作。只要没有写操作，多个读操作可以同时进行，写锁是独占的。ReadWriteLock是接口，定义了readLock和writeLock两个方法。

ReentrantReadWriteLock是ReadWriteLock的实现类，具有以下特性：

* 公平性
	- 默认为非公平锁，和独占锁的非公平性一样，由于读线程间没有锁竞争，所以读操作没有公平和非公平之分，写操作时有区分，非公平锁的吞吐量要高于公平锁。
	
	- 公平锁利用AQS的CLH队列，释放当前保持的锁时，优先为等待时间最长的写线程分配写锁。一个线程持有写锁或者有一个写线程已经在等待了，那么试图获取公平锁的所有线程都将被阻塞，直到最先的写线程释放锁。如果读线程等待时间比写线程还要长，一旦上个写线程释放锁，这一组读线程将获取锁。

* 重入性
	- 读写锁允许读线程和写线程按照请求锁的顺序重新获取读取锁或者写锁。只有写线程释放了锁，读线程才能获取重入锁。
	
	- 写线程获取写锁之后可再次获取读锁，但是读线程获取读锁之后不能获取写锁。
	
	- 读写锁最多支持65535个递归写锁和65535的读锁。
	
* 锁降级
	写线程获取写锁后可获取读锁，然后释放写锁，实现了锁降级。

* 锁升级
	- 读锁不能直接升级为写锁。

* 锁获取中断
	- 读锁和写锁都支持获取锁期间被中断，和独占锁一致。
	
* 条件变量
	- 写锁提供了条件变量Condition的支持，和独占锁一致，读锁不允许获取条件变量。
	
* 重入数
	- 读写锁数量最大分别是65535
	
* 监测
	- 此类支持一些确定是保持锁还是争用锁的方法，用于监视系统状态，不是同步控制。
	
ReadLock的lock方法，是共享锁：

```
public void lock() {
    sync.acquireShared(1);
}
```

WriteLock的lock方法，是独占锁：

```
public void lock() {
    sync.acquire(1);
}
```

内部都是使用
AQS的acquire和release来操作的。

## WriteLock
### lock()

```
public void lock() {
    sync.acquire(1);
}
```
与ReentrantLock一样，调用AQS的acquire方法：

```
public final void acquire(int arg) {
        if (!tryAcquire(arg) &&
            acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
            selfInterrupt();
    }
```
首先调用tryAcquire方法，该方法与ReentrantLock中的tryAcquire方法不太一样：

```
protected final boolean tryAcquire(int acquires) {

	//当前线程
    Thread current = Thread.currentThread();
    //当前锁个数
    int c = getState();
    //写锁个数
    int w = exclusiveCount(c);
    //当前锁个数不为零，已经有线程持有锁，线程重入
    if (c != 0) {
    	//写线程为0或者独占锁不是当前线程，返回false
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        //超出最大范围
        if (w + exclusiveCount(acquires) > MAX_COUNT)
            throw new Error("Maximum lock count exceeded");
        //设置锁的线程数量
        setState(c + acquires);
        return true;
    }
    //是否阻塞
    if (writerShouldBlock() ||
        !compareAndSetState(c, c + acquires))
        return false;
    //设置锁为当前线程所有
    setExclusiveOwnerThread(current);
    return true;
}
```