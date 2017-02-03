Lock和Synchronized拥有一样的语义，但比synchronized更强大。可以实现锁的公平性，锁的可重入性，定时锁等候等。

* ReentrantLock，可重入的互斥锁，是Lock接口的主要实现。

* ReadWriteLock，维护了一对相关的锁，一个用于只读，一个用于写入操作。

* ReentrantReadWriteLock。

* Semaphore，信号量。

* Condition，锁的关联条件，目的是允许线程获取锁并且查看等待的某一个条件是否满足。

* CyclicBarrier，同步辅助类，允许一组线程互相等待，直到到达某个公共屏障点。

## ReentrantLock
可重入的互斥锁，是一种递归无阻塞的同步机制，等同于synchronized，但是比synchronized更强大灵活，减少死锁发生的概率。

ReentrantLock提供公平锁机制。默认为非公平锁。

### ReentrantLock和synchronized的区别
ReentrantLock提供了更多的功能，定时锁等候，可中断锁等候，锁投票。

ReentrantLock还提供了条件Condition，对线程的等待，唤醒操作更加详细和灵活，在多个条件变量和高度竞争锁的地方，ReentrantLock更加合适。

ReentrantLock提供了可轮询的锁请求，它会尝试着去获取锁，如果成功则继续，否则可以等到下次运行时处理。synchronized则一旦进入锁请求要么成功要么阻塞，所以ReentrantLock不容易产生死锁。

ReentrantLock支持更加灵活的同步代码块，但是一定要在finally中释放。

ReentrantLock支持中断处理。

### AQS
Java中管理锁的抽象类，该类为实现依赖FIFO等待队列的阻塞锁和相关同步器提供一个框架。维护锁的当前状态和线程等待列表。

CLH，AQS中等待锁的线程队列。

### FairSync公平锁的lock方法

```
final void lock() {
    acquire(1);
}
```
内部调用acquire(1)，ReentrantLock是独占锁，1表示锁的状态，对于独占锁而言，如果锁处于可获取状态，其状态为0，当锁初次被线程获取时状态变为1。

acquire()方法，在AQS类中：

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

tryAcquire方法在FairySync中的实现：

```
protected final boolean tryAcquire(int acquires) {
	//当前线程
    final Thread current = Thread.currentThread();
    //获取锁的状态
    int c = getState();
    //锁的状态为0，说明锁没有被任何线程占用。
    //hasQueuedPredecessors判断当前线程是不是CLH队列中的第一个线程
    //如果是，获取锁，设置锁的状态compareAndSetState
    //并且设置锁的拥有者为当前线程setExclusiveOwnerThread
    if (c == 0) {
        if (!hasQueuedPredecessors() &&
            compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //c不等于0，表示锁已经被占用，判断是否是当前线程占用，若是设置state，否则返回false。
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0)
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
tryAcquire方法主要是尝试获取锁，获取成功则设置锁状态返回true，否则返回false。

hasQueuedPredecessors()方法，判断当前线程是不是在CLH队列的队首，来返回是不是有比当前线程等待更久的线程。

```
public final boolean hasQueuedPredecessors() {
    Node t = tail; // Read fields in reverse initialization order
    Node h = head;
    Node s;
    return h != t &&
        ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

compareAndSetState：设置锁状态，利用CAS设置锁的状态。

```
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
```

setExclusiveOwnerThread：设置当前线程为锁的拥有者：

```
protected final void setExclusiveOwnerThread(Thread t) {
    exclusiveOwnerThread = t;
}
```

addWaiter(Node.EXCLUSIVE)方法：

```
private Node addWaiter(Node mode) {
	//新建一个节点
    Node node = new Node(Thread.currentThread(), mode);
    //CLH队尾节点
    Node pred = tail;
    //队尾节点不为null，表示CLH队列不为null，将线程加入到队列队尾。
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //CLH队列为空，新建CLH队列，将当前线程添加到CLH队列中。
    enq(node);
    return node;
}
```

compareAndSetTail，使用CAS设置队尾：

```
private final boolean compareAndSetTail(Node expect, Node update) {
    return unsafe.compareAndSwapObject(this, tailOffset, expect, update);
}
```

enq()方法：

```
private Node enq(final Node node) {
    for (;;) {
        Node t = tail;
        if (t == null) { // Must initialize
            if (compareAndSetHead(new Node()))
                tail = head;
        } else {
            node.prev = t;
            if (compareAndSetTail(t, node)) {
                t.next = node;
                return t;
            }
        }
    }
}
```

acquireQueued方法：

```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
    	//中断标志位
        boolean interrupted = false;
        for (;;) {
        	//上一个节点，node相当于当前线程，所以上一个节点表示上一个等待锁的线程。
            final Node p = node.predecessor();
            //如果p是head，说明当前线程是head的直接后继，尝试获取锁
            if (p == head && tryAcquire(arg)) {
            	//当前节点设置为头节点
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //如果不是head直接后继，或者获取锁失败，则检查是否要阻塞当前线程，是，则阻塞。
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

shouldParkAfterFailedAcquire 判断当前线程是否需要阻塞。

parkAndCheckInterrupt 阻塞当前线程，并且返回“线程被唤醒之后”的中断状态：

```
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

selfInterrupt方法产生一个中断。

### 公平锁的lock方法总结
首先通过tryAcquire方法尝试获取锁，如果成功直接返回，否则通过acquireQueued方法再次获取。在acquireQueued方法中会先通过addWaiter方法将当前线程加入到CLH队列的队尾，在CLH中等待。等待过程中线程处于休眠状态，直到成功获取锁才会返回。

### 非公平锁NonfairSync的lock方法
非公平锁的lock与公平锁的lock流程上一致，但是获取锁机制不同，公平锁在获取锁时采用公平策略CLH队列，而非公平锁采用非公平策略，无视等待队列，直接尝试获取：

```
final void lock() {
    if (compareAndSetState(0, 1))
        setExclusiveOwnerThread(Thread.currentThread());
    else
        acquire(1);
}
```
直接通过compareAndSetState尝试设置状态，成功了就将锁的拥有者设置为当前线程，否则调用acquire尝试获取锁。

acquire和公平锁实现一样。

tryAcquire实现不同：

```
protected final boolean tryAcquire(int acquires) {
    return nonfairTryAcquire(acquires);
}
```

nonfairyTryAcquire:

```
final boolean nonfairTryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) // overflow
            throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```
c==0的条件下，和公平锁不一样，此处直接获取锁，公平锁需要判断线程是否处于队列头。

### 公平锁和非公平锁的lock方法总结
公平锁要按照CLH队列等待获取锁，费公平锁无视CLH队列，直接获取锁。

### unlock方法
unlock方法不分公平锁和不公平锁。

```
public void unlock() {
	//尝试在当前锁的锁定计数值上减去1，成功返回true
    sync.release(1);
}

public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}

protected final boolean tryRelease(int releases) {
    int c = getState() - releases;
    if (Thread.currentThread() != getExclusiveOwnerThread())
        throw new IllegalMonitorStateException();
    boolean free = false;
    //c==0 释放该锁，同时将当前所持有线程设置为null
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

unlock方法要在finally方法中释放。