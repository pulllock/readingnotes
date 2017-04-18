ReadWriteLock接口，读写锁，有一对关联的锁，一个是读锁，另外一个是写锁。读锁可以被多个线程同时持有，但是写锁只能由一个线程持有，是独占锁。不能同时存在读写线程。

# ReadWriteLock接口

```
public interface ReadWriteLock {
    //读锁
    Lock readLock();

    //写锁
    Lock writeLock();
}
```
ReadWriteLock是一个接口，并不是Lock接口的子类或者扩展之类的，而是内部使用了Lock来实现读写锁。

# ReadWriteLock的实现

## ReentrantReadWriteLock
ReentrantReadWriteLock是ReadWriteLock的唯一实现类。跟ReentrantLock语意差不多。看下ReadWriteLock的特性：

- 获取锁的顺序，不是使用读或者写来决定锁的顺序，而是使用可选的公平策略来决定顺序。

	+ 非公平模式，默认就是非公平模式。不确定顺序。
	+ 公平模式，按照锁获取的顺序，当前锁被释放之后，等待时间最长的写锁得到写锁或者等待时间最长的一组读线程得到读锁。一个线程试图以公平模式获取读锁，但是此时写锁被持有，或者有读线程正在等待获取锁，当前线程会被阻塞，获取不到读锁；一个线程试图以公平模式获取写锁，只有当其他线程把读写锁全部都释放了，才能获取到。

- 重入性，读写锁都可以像ReentrantLock一样，可重入。另外，一个写线程可以获取读锁，但是读线程不能获取写锁。
- 锁降级，可以从写锁降级到读锁，但是不能从读锁升级到写锁。
- 可中断的锁获取，读写锁都支持中断。
- 同样的支持Condition，跟ReentrantLock一样。但是这里只有写锁能用Condition。

首先看一下ReadLock这个内部类的实现，然后中间介绍同步器以及公平和非公平的代码，然后再是WriteLock，最后再过一遍其他方法。

### ReadLock
ReentrantReadWriteLock中的读锁，看下代码：

```
//实现了Lock接口，内部实现跟ReentrantLock差不多，其加锁解锁等操作还是由AQS来实现
public static class ReadLock implements Lock, java.io.Serializable {
    //同步器
    private final Sync sync;

    //使用外部的lock来构造一个ReadLock实例
    protected ReadLock(ReentrantReadWriteLock lock) {
        sync = lock.sync;
    }

    //获取读锁，如果读锁没有被其他线程持有，直接获取返回
    //如果读锁被其他线程持有，当前线程会一直阻塞直到读锁被获取到
    public void lock() {
    	//调用的是AQS的acquireShared方法
        //acquireShared在共享模式下获取，忽略中断
        //acquireShared方法首先使用tryAcquireShared先尝试获取，如果获取不到就加入队列中去
        //而tryAcquireShared方法是由ReentrantReadWriteLock中的Sync实现的
        sync.acquireShared(1);
    }

    //可中断的获取锁，跟lock流程差不多
    public void lockInterruptibly() throws InterruptedException {
        sync.acquireSharedInterruptibly(1);
    }

    //尝试获取，非阻塞
    //如果写锁没有被其他线程持有，可以直接获取到读锁并返回true。
    //即便有公平策略，当调用此方法时候，如果读锁可用也会直接获取到
    //如果不想破坏公平性，可以使用带有超时时间的tryLock方法
    public  boolean tryLock() {
    	//由Sync来实现
        return sync.tryReadLock();
    }

    //带所有超时时间的tryLock
    public boolean tryLock(long timeout, TimeUnit unit)
            throws InterruptedException {
        return sync.tryAcquireSharedNanos(1, unit.toNanos(timeout));
    }

    //释放锁
    public  void unlock() {
    	//首先tryReleaseShared
        //没有成功就doReleaseShared
        sync.releaseShared(1);
    }

    //读锁没有Condition操作
    public Condition newCondition() {
        throw new UnsupportedOperationException();
    }
}
```

可以看到读锁基本跟ReentrantLock的套路差不多，但是因为读锁可以被很多线程使用，所以读锁很多都调用的shared方法。这些方法基本都是在AQS中实现的。

以lock方法为例，调用的是AQS的acquireShared方法试下，在acquireShared中首先调用tryAcquireShared来获取锁，如果成功就返回了，没成功的话调用doAcquireShared方法加入队列中去，然后继续获取。其中tryAcquireShared方法都是在Sync中实现的，首先看看Sync的代码。

### Sync

```
//继承AQS，是一个同步器
abstract static class Sync extends AbstractQueuedSynchronizer {

    //锁的状态分成了两个短整型，低位表示独占锁的状态，高位表示共享锁的状态

    static final int SHARED_SHIFT   = 16;
    static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
    static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
    static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

    //读锁状态
    static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
    //写锁状态
    static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }

    //持有每个线程的读锁的数量
    static final class HoldCounter {
        int count = 0;
        final long tid = Thread.currentThread().getId();
    }

    //继承自ThreadLocal
    static final class ThreadLocalHoldCounter
        extends ThreadLocal<HoldCounter> {
        public HoldCounter initialValue() {
            return new HoldCounter();
        }
    }

    //当前线程的读锁持有的重入的数量
    private transient ThreadLocalHoldCounter readHolds;

    //
    private transient HoldCounter cachedHoldCounter;

    //
    private transient Thread firstReader = null;
    private transient int firstReaderHoldCount;

    Sync() {
        readHolds = new ThreadLocalHoldCounter();
        setState(getState()); // ensures visibility of readHolds
    }

    //
    abstract boolean readerShouldBlock();

    //
    abstract boolean writerShouldBlock();

    //写锁用来释放锁
    protected final boolean tryRelease(int releases) {
        if (!isHeldExclusively())
            throw new IllegalMonitorStateException();
        int nextc = getState() - releases;
        boolean free = exclusiveCount(nextc) == 0;
        if (free)
            setExclusiveOwnerThread(null);
        setState(nextc);
        return free;
    }
	//尝试获取，这个是写锁用的方法
    protected final boolean tryAcquire(int acquires) {
       	//当前线程
        Thread current = Thread.currentThread();
        //当前同步器的状态
        int c = getState();
        //独占锁的数量
        int w = exclusiveCount(c);
        if (c != 0) {//同步器状态不为0，那么表示已经有线程获取到锁了
            // (Note: if c != 0 and w == 0 then shared count != 0)
            //w为0，表示写锁为0，说明读锁肯定不为0，而读锁只能在没有读锁和写锁的情况下才能获取，所以返回false
            //如果w不为0的话，表示现在只有写锁，但是当前线程不是持有独占锁的线程，也会返回false
            if (w == 0 || current != getExclusiveOwnerThread())
                return false;
            //到这里，w肯定不为0，首先判断是否超多锁的总数量
            if (w + exclusiveCount(acquires) > MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            //设置同步器状态，表示重入获取
            setState(c + acquires);
            return true;
        }
        //到这里，表示c为0，也就是没有线程持有锁
        //writerShouldBlock方法分为公平和不公平的两种实现
        //在公平的实现中，会查询是否有线程比当前线程等待的时间更长，有的话，返回true
        //不公平实现，直接返回false
        //CAS设置状态
        if (writerShouldBlock() ||
            !compareAndSetState(c, c + acquires))
            return false;
        //设置当前线程为独占锁的拥有者
        setExclusiveOwnerThread(current);
        return true;
    }
	//读锁用
    protected final boolean tryReleaseShared(int unused) {
        Thread current = Thread.currentThread();
        if (firstReader == current) {
            // assert firstReaderHoldCount > 0;
            if (firstReaderHoldCount == 1)
                firstReader = null;
            else
                firstReaderHoldCount--;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != current.getId())
                rh = readHolds.get();
            int count = rh.count;
            if (count <= 1) {
                readHolds.remove();
                if (count <= 0)
                    throw unmatchedUnlockException();
            }
            --rh.count;
        }
        for (;;) {
            int c = getState();
            int nextc = c - SHARED_UNIT;
            if (compareAndSetState(c, nextc))
                // Releasing the read lock has no effect on readers,
                // but it may allow waiting writers to proceed if
                // both read and write locks are now free.
                return nextc == 0;
        }
    }

    private IllegalMonitorStateException unmatchedUnlockException() {
        return new IllegalMonitorStateException(
            "attempt to unlock read lock, not locked by current thread");
    }
	//读锁用来获取锁的方法
    protected final int tryAcquireShared(int unused) {
        //当前线程
        Thread current = Thread.currentThread();
        //AQS状态
        int c = getState();
        //写锁不为0，并且写锁被其他线程拥有，返回-1
        if (exclusiveCount(c) != 0 &&
            getExclusiveOwnerThread() != current)
            return -1;
        //到这里有两种情况：
        //一种是写锁不为0，但是写锁被当前线程拥有
        //一种是写锁为0
        //r表示读锁数量
        int r = sharedCount(c);
        //readerShouldBlock也有公平和非公平实现
        //公平实现中，会查询是否有线程比当前线程等待的时间更长，有的话，返回true
        //非公平实现中，没明白！！！
        if (!readerShouldBlock() &&
            r < MAX_COUNT &&
            compareAndSetState(c, c + SHARED_UNIT)) {
            if (r == 0) {//表示没有读锁
                firstReader = current;
                firstReaderHoldCount = 1;
            } else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                HoldCounter rh = cachedHoldCounter;
                if (rh == null || rh.tid != current.getId())
                    cachedHoldCounter = rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                rh.count++;
            }
            return 1;
        }
        //使用循环来尝试获取读锁
        return fullTryAcquireShared(current);
    }

    /**
     * Full version of acquire for reads, that handles CAS misses
     * and reentrant reads not dealt with in tryAcquireShared.
     */
    final int fullTryAcquireShared(Thread current) {
        /*
         * This code is in part redundant with that in
         * tryAcquireShared but is simpler overall by not
         * complicating tryAcquireShared with interactions between
         * retries and lazily reading hold counts.
         */
        HoldCounter rh = null;
        for (;;) {
            int c = getState();
            if (exclusiveCount(c) != 0) {
                if (getExclusiveOwnerThread() != current)
                    return -1;
                // else we hold the exclusive lock; blocking here
                // would cause deadlock.
            } else if (readerShouldBlock()) {
                // Make sure we're not acquiring read lock reentrantly
                if (firstReader == current) {
                    // assert firstReaderHoldCount > 0;
                } else {
                    if (rh == null) {
                        rh = cachedHoldCounter;
                        if (rh == null || rh.tid != current.getId()) {
                            rh = readHolds.get();
                            if (rh.count == 0)
                                readHolds.remove();
                        }
                    }
                    if (rh.count == 0)
                        return -1;
                }
            }
            if (sharedCount(c) == MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            if (compareAndSetState(c, c + SHARED_UNIT)) {
                if (sharedCount(c) == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    if (rh == null)
                        rh = cachedHoldCounter;
                    if (rh == null || rh.tid != current.getId())
                        rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                    cachedHoldCounter = rh; // cache for release
                }
                return 1;
            }
        }
    }

    /**
     * Performs tryLock for write, enabling barging in both modes.
     * This is identical in effect to tryAcquire except for lack
     * of calls to writerShouldBlock.
     */
    final boolean tryWriteLock() {
        Thread current = Thread.currentThread();
        int c = getState();
        if (c != 0) {
            int w = exclusiveCount(c);
            if (w == 0 || current != getExclusiveOwnerThread())
                return false;
            if (w == MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
        }
        if (!compareAndSetState(c, c + 1))
            return false;
        setExclusiveOwnerThread(current);
        return true;
    }

    /**
     * Performs tryLock for read, enabling barging in both modes.
     * This is identical in effect to tryAcquireShared except for
     * lack of calls to readerShouldBlock.
     */
    final boolean tryReadLock() {
        Thread current = Thread.currentThread();
        for (;;) {
            int c = getState();
            if (exclusiveCount(c) != 0 &&
                getExclusiveOwnerThread() != current)
                return false;
            int r = sharedCount(c);
            if (r == MAX_COUNT)
                throw new Error("Maximum lock count exceeded");
            if (compareAndSetState(c, c + SHARED_UNIT)) {
                if (r == 0) {
                    firstReader = current;
                    firstReaderHoldCount = 1;
                } else if (firstReader == current) {
                    firstReaderHoldCount++;
                } else {
                    HoldCounter rh = cachedHoldCounter;
                    if (rh == null || rh.tid != current.getId())
                        cachedHoldCounter = rh = readHolds.get();
                    else if (rh.count == 0)
                        readHolds.set(rh);
                    rh.count++;
                }
                return true;
            }
        }
    }

    protected final boolean isHeldExclusively() {
        // While we must in general read state before owner,
        // we don't need to do so to check if current thread is owner
        return getExclusiveOwnerThread() == Thread.currentThread();
    }

    // Methods relayed to outer class

    final ConditionObject newCondition() {
        return new ConditionObject();
    }

    final Thread getOwner() {
        // Must read state before owner to ensure memory consistency
        return ((exclusiveCount(getState()) == 0) ?
                null :
                getExclusiveOwnerThread());
    }

    final int getReadLockCount() {
        return sharedCount(getState());
    }

    final boolean isWriteLocked() {
        return exclusiveCount(getState()) != 0;
    }

    final int getWriteHoldCount() {
        return isHeldExclusively() ? exclusiveCount(getState()) : 0;
    }

    final int getReadHoldCount() {
        if (getReadLockCount() == 0)
            return 0;

        Thread current = Thread.currentThread();
        if (firstReader == current)
            return firstReaderHoldCount;

        HoldCounter rh = cachedHoldCounter;
        if (rh != null && rh.tid == current.getId())
            return rh.count;

        int count = readHolds.get().count;
        if (count == 0) readHolds.remove();
        return count;
    }

    /**
     * Reconstitute this lock instance from a stream
     * @param s the stream
     */
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();
        readHolds = new ThreadLocalHoldCounter();
        setState(0); // reset to unlocked state
    }

    final int getCount() { return getState(); }
}
```
