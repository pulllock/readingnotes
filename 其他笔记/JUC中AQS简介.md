AQS，在java.util.concurrent.locks包中，AbstractQueuedSynchronizer这个类是并发包中的核心，了解其他类之前，需要先弄清楚AQS。在JUC的很多类中都会存在一个内部类Sync，Sync都是继承自AbstractQueuedSynchronizer，相信不用说就能明白AQS有多重要。

# AQS原理

AQS就是一个同步器，要做的事情就相当于一个锁，所以就会有两个动作：一个是获取，一个是释放。获取释放的时候该有一个东西来记住他是被用还是没被用，这个东西就是一个状态。如果锁被获取了，也就是被用了，还有很多其他的要来获取锁，总不能给全部拒绝了，这时候就需要他们排队，这里就需要一个队列。这大概就清楚了AQS的主要构成了：

- 获取和释放两个动作
- 同步状态（原子操作）
- 阻塞队列

# state
AQS用32位整形来表示同步状态。

```
private volatile int state;
```

在互斥锁中表示线程是否已经获取了锁，0未获取，1已经获取，大于1表示重入数。

AQS提供了getState(),setState(),compareAndSetState()来获取和修改state的值，这些操作需要atomic包的支持，采用CAS操作，保证其原子性和可见性。

# AQS的CLH锁队列
CLH其实就是一个FIFO的队列，只不过稍微做了点改进。AQS中内部使用内部类Node来实现，是一个链表队列，原始CLH使用自旋锁，AQS的CLH则在每个node里使用一个状态字段来控制阻塞，不是自旋。直接看代码：

```
/**
      +------+  prev +-----+       +-----+
 head |      | <---- |     | <---- |     |  tail
      +------+       +-----+       +-----+
/**
static final class Node {
	//作为共享模式
    static final Node SHARED = new Node();
	//作为独占模式
    static final Node EXCLUSIVE = null;
	//等待状态：表示节点中线程是已被取消的
    static final int CANCELLED =  1;
	//等待状态：表示当前节点的后继节点的线程需要被唤醒
    static final int SIGNAL    = -1;
	//等待状态：表示线程正在等待条件
    static final int CONDITION = -2;
	//等待状态：表示下一个共享模式的节点应该无条件的传播下去
    static final int PROPAGATE = -3;
	//等待状态，初始化为0，剩下的状态就是上面列出的
    volatile int waitStatus;
	//当前节点的前驱节点
    volatile Node prev;
	//后继节点
    volatile Node next;
	//当前节点的线程
    volatile Thread thread;
	//
    Node nextWaiter;
	//是否是共享节点
    final boolean isShared() {
        return nextWaiter == SHARED;
    }
	//当前节点的前驱节点
    final Node predecessor() throws NullPointerException {
        Node p = prev;
        if (p == null)
            throw new NullPointerException();
        else
            return p;
    }

    Node() {    // Used to establish initial head or SHARED marker
    }

    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    Node(Thread thread, int waitStatus) { // Used by Condition
        this.waitStatus = waitStatus;
        this.thread = thread;
    }
}
```


# 共享锁和互斥锁
AQS的CLH队列锁中，每个节点代表着一个需要获取锁的线程，该node中有两个常量SHARED共享模式，EXCLUSIVE独占模式。

```
/** Marker to indicate a node is waiting in shared mode */
static final Node SHARED = new Node();
/** Marker to indicate a node is waiting in exclusive mode */
static final Node EXCLUSIVE = null;
```

共享模式允许多个线程可以获取同一个锁，独占模式则一个锁只能被一个线程持有，其他线程必须要等待。

# AQS源码

```
//阻塞队列的队列头
private transient volatile Node head;
//队列尾
private transient volatile Node tail;
//同步状态，这就是上面提到的需要原子操作的状态
private volatile int state;
//返回当前同步器的状态
protected final int getState() {
    return state;
}
//设置同步器的状态
protected final void setState(int newState) {
    state = newState;
}
//原子的设置当前同步器的状态
protected final boolean compareAndSetState(int expect, int update) {
    return unsafe.compareAndSwapInt(this, stateOffset, expect, update);
}
//
static final long spinForTimeoutThreshold = 1000L;
```

## 独占模式的获取
### acquire，独占，忽略中断

```
//独占模式的获取方法，会忽略中断
//tryAcquire方法会被至少调用一次，由子类实现
//如果tryAcquire不能成功，当前线程就会进入队列排队
public final void acquire(int arg) {
	//首先调用tryAcquire尝试获取
    //获取不成功，就使用acquireQueued使线程进入等待队列
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

tryAcquire方法：

```
//由子类来实现
//尝试在独占模式下获取，会查询该对象的状态是否允许在独占模式下获取
protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
}
```

使用指定的模式创建一个节点，添加到AQS链表队列中：

```
private Node addWaiter(Node mode) {
	//当前线程，指定的mode，共享或者独占
    Node node = new Node(Thread.currentThread(), mode);
    //先尝试使用直接添加进队列
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    //使用添加节点的方法
    enq(node);
    return node;
}
```

向队列中插入节点：

```
//会插入节点到对列中
private Node enq(final Node node) {
    for (;;) {
    	//尾节点
        Node t = tail;
        //需要实例化一个队列
        if (t == null) { // Must initialize
        	//使用cas创建头节点
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

tryAcquire没有获取到，就会先使用addWaiter添加进队列，然后使用acquireQueued从队列获取，如果这时候获取成功，则替换当前节点为队列头，然后返回：

```
//独占模式处理正在排队等待的线程。
//自旋，直至获取成功才返回
final boolean acquireQueued(final Node node, int arg) {
	//当前获取是否失败
    boolean failed = true;
    try {
    	//获取是否被中断
        boolean interrupted = false;
        for (;;) {
        	//获取当前节点的前驱节点
            final Node p = node.predecessor();
            
            //head节点要么是刚才初始化的节点
            //要么就是成功获取锁的节点
            //如果当前节点的前驱节点是head，当前节点就应该去尝试获取锁了
            //当前节点的前驱节点是头节点，就尝试获取
            if (p == head && tryAcquire(arg)) {
            	//获取成功的话，就把当前节点设置为头节点
                setHead(node);
                //之前的head节点的next引用设为null
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //查看当前节点是否应该被park
            //如果应该，就park当前线程
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
    	//失败了，取消当前线程
        if (failed)
            cancelAcquire(node);
    }
}
```


设置头节点，只能被获取方法调用：

```
private void setHead(Node node) {
    head = node;
    node.thread = null;
    node.prev = null;
}
```

shouldParkAfterFailedAcquire方法，查看是否应该被park：

```
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
	//前驱节点中保存的等待状态
    int ws = pred.waitStatus;
    //等待状态是signal，也就是当前节点在等着被唤醒
    //此时当前节点应该park
    if (ws == Node.SIGNAL)
        return true;
        
    //等待状态大于0表示前驱节点已经取消
    //会向前找到一个非取消状态的节点
    if (ws > 0) {
        do {
            node.prev = pred = pred.prev;
        } while (pred.waitStatus > 0);
        pred.next = node;
    } else {
       //将前驱节点的waitStatus设置为signal，表示当前需要被park
        compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
    }
    return false;
}
```

看下parkAndCheckInterrupt方法：

```
//挂起当前线程，并返回当前中断状态
private final boolean parkAndCheckInterrupt() {
    //挂起当前线程
    LockSupport.park(this);
    return Thread.interrupted();
}
```

cancelAcquire取消当前节点：

```
private void cancelAcquire(Node node) {
    //节点不存在
    if (node == null)
        return;
	//节点的线程引用设为null
    node.thread = null;

    //前驱节点
    Node pred = node.prev;
    //大于0表示前驱节点被取消
    while (pred.waitStatus > 0)
        node.prev = pred = pred.prev;

    //前驱节点的下一个是需要移除的节点
    Node predNext = pred.next;

    //设置节点状态为取消
    node.waitStatus = Node.CANCELLED;

    //如果是尾节点，直接取消，将前一个节点设置为尾节点
    if (node == tail && compareAndSetTail(node, pred)) {
        compareAndSetNext(pred, predNext, null);
    } else {//不是尾节点，说明有后继节点，将前驱节点的next纸箱后继节点
        int ws;
        if (pred != head &&
            ((ws = pred.waitStatus) == Node.SIGNAL ||
             (ws <= 0 && compareAndSetWaitStatus(pred, ws, Node.SIGNAL))) &&
            pred.thread != null) {
            Node next = node.next;
            if (next != null && next.waitStatus <= 0)
                compareAndSetNext(pred, predNext, next);
        } else {
            unparkSuccessor(node);
        }

        node.next = node; // help GC
    }
}
```

### acquireInterruptibly 独占，可中断
跟独占忽略中断类似，不再解释。
### tryAcquireNanos，独占，可超时，可中断
跟上面类似，但是在doAcquireNanos中会获取当前时间，并获取LockSupport.parkNanos之后的时间在做超时时间的重新计算，到了超时时间，就返回false。

## 独占模式的释放

### release，独占，忽略中断

```
public final boolean release(int arg) {
	//尝试释放，修改状态
    if (tryRelease(arg)) {
    	//成功释放
        //head代表初始化的节点，或者是当前占有锁的节点
        //需要unpark后继节点
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
unparkSuccessor：

```
private void unparkSuccessor(Node node) {
	//头节点中保存的waitStatus
    int ws = node.waitStatus;
    //重置头节点状态为0
    if (ws < 0)
        compareAndSetWaitStatus(node, ws, 0);
	//后继节点
    Node s = node.next;
    //后继节点为null或者已经取消
    if (s == null || s.waitStatus > 0) {
        s = null;
        //从最后往前找有效的节点
        for (Node t = tail; t != null && t != node; t = t.prev)
            if (t.waitStatus <= 0)
                s = t;
    }
    //unpark
    if (s != null)
        LockSupport.unpark(s.thread);
}
```

## 共享模式的获取
### acquireShared，共享，忽略中断
### acquireSharedInterruptibly，共享，可中断
### tryAcquireSharedNanos，共享，可设置超时，可中断

## 共享模式的释放
### releaseShared

共享模式的和独占模式基本差不多，和独占式的acquireQueued方法区别就是在获取成功的节点后会继续unpark后继节点，将共享状态向后传播。


# LockSupport
用来创建锁和其他同步类的基本线程阻塞原语。每个使用LockSupport的线程都会与一个许可关联，如果该许可可用并且可在进程中使用，则调用park()将会立即返回，否则可能阻塞。如果许可不可用，可调用unpark使其可用。

许可不可重入，只能调用一次park()方法，否则会一直阻塞。

park()和unpark()作用分别是阻塞线程和解除阻塞线程，且park和unpark不会遇到suspend和resume可能引发的死锁问题。

park，如果许可可用，使用该许可，并且该调用立即返回；否则为线程调度禁用当前线程，并在发生以下三种情况之一之前，使其处于休眠状态：
	* 其他某个线程将当前线程作为目标调用unpark
	* 其他某个线程中断当前线程
	* 该调用不合逻辑的返回

unpark，如果给定的线程尚不可用，则使其可用。如果线程在park上受阻塞，则它将解除其阻塞状态。否则，保证下一次调用park不受阻塞。如果给定线程尚未启动，则无法保证此操作有任何效果。




