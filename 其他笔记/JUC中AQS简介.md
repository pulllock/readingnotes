JUC同步器应该具备两种最基本的功能，获取锁和释放锁。获取锁应该先判断当前状态是否可以获取，如果不可以获取则处于阻塞状态，释放应该是释放后修改状态，让其他线程能够获得到该锁。

JUC中各个同步器获取锁和释放锁的方法都不同，lock.lock(),Semaphore.acquire(),CountDownLatch.await()等。AQS作为核心处理框架，提供了大量的同步操作，用户还可以在此类的基础上进行自定义，实现自己的同步器。

## state
AQS用32位整形来表示同步状态。

```
private volatile int state;
```

在互斥锁中表示线程是否已经获取了锁，0未获取，1已经获取，大于1表示重入数。

AQS提供了getState(),setState(),compareAndSetState()来获取和修改state的值，这些操作需要atomic包的支持，采用CAS操作，保证其原子性和可见性。

## CLH同步队列
AQS内部维护着一个FIFO的CLH队列，所以AQS不支持基于优先级的同步策略。CLH更容易处理cancel和timeout，同时具备进出队列快，无锁，畅通无阻，检查是否有线程在等待也很容易。

原始CLH使用自旋锁，AQS的CLH则在每个node里使用一个状态字段来控制阻塞，不是自旋。

## 共享锁和互斥锁
AQS的CLH队列锁中，每个节点代表着一个需要获取锁的线程，该node中有两个常量SHARED共享模式，EXCLUSIVE独占模式。

```
/** Marker to indicate a node is waiting in shared mode */
static final Node SHARED = new Node();
/** Marker to indicate a node is waiting in exclusive mode */
static final Node EXCLUSIVE = null;
```

共享模式允许多个线程可以获取同一个锁，独占模式则一个锁只能被一个线程持有，其他线程必须要等待。

## 阻塞和唤醒
Java内置锁可以使用wait和notify来阻塞唤醒线程，而AQS中使用LockSupport.park()和LockSupport.unpark()的本地方法来实现线程的阻塞和唤醒。

## AQS锁获取
对于lock.lock()最终都会调用AQS的acquire()方法，Semaphore.acquire()最终会调用AQS的acquireSharedInterruptibly()方法。

```
//以独占模式获取锁，忽略中断
public final void acquire(int arg) {
	//tryAcquire试图在独占模式下获取锁，获取成功返回true，失败返回false。
	//addWaiter将当前线程加入到CLH队列尾。
	//acquireQueued当前线程会根据公平性来进行阻塞等待，知道获取到锁为止，并且返回当前线程在等待过程中有没有中断过。
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        //产生一个中断。
        selfInterrupt();
}
```

1. 首先线程尝试获取锁，如果成功直接返回，如果失败则新建node节点添加到CLH队列中。tryAcquire尝试获取锁，addWaiter新建节点添加到CLH队列。tryAcquire在AQS中没有实现，仅仅是抛出一个异常，具体实现需要各个锁自己实现。
2. acquireQueued主要是根据节点寻找CLH队列的头节点，并尝试获取锁，判断是否需要挂起，并且返回挂起标识。

selfInterrupt产生一个中断，如果在aquireQueued()中当前线程被中断过，需要产生一个中断。

## AQS锁释放

```
//以独占模式释放对象
public final boolean release(int arg) {
	//tryRelease尝试释放锁，AQS没有实现，给子类实现
	if (tryRelease(arg)) {
	    Node h = head;
	    if (h != null && h.waitStatus != 0)
	    	//唤醒节点
	        unparkSuccessor(h);
	    return true;
	}
	return false;
	}
```

## 阻塞，唤醒
当一个线程加入到CLH队列中时，如果不是头结点，需要判断该节点是否需要挂起；释放锁后，需要唤醒该线程的继任节点。

acquireQueued()方法：

```
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
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
parkAndCheckInterrupt()方法，挂起当前线程。

```
private final boolean parkAndCheckInterrupt() {
    LockSupport.park(this);
    return Thread.interrupted();
}
```

LockSupport的park方法，为了线程调度，在许可可用之前禁用当前线程。

释放锁之后，需要唤醒该线程的继任节点：

```
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```
unparkSuccessor方法唤醒线程的继任节点，通过LockSupport的unpark方法来唤醒，如果给定的线程的许可尚不可用，则使其可用。

## LockSupport
用来创建锁和其他同步类的基本线程阻塞原语。每个使用LockSupport的线程都会与一个许可关联，如果该许可可用并且可在进程中使用，则调用park()将会立即返回，否则可能阻塞。如果许可不可用，可调用unpark使其可用。

许可不可重入，只能调用一次park()方法，否则会一直阻塞。

park()和unpark()作用分别是阻塞线程和解除阻塞线程，且park和unpark不会遇到suspend和resume可能引发的死锁问题。

park，如果许可可用，使用该许可，并且该调用立即返回；否则为线程调度禁用当前线程，并在发生以下三种情况之一之前，使其处于休眠状态：
	* 其他某个线程将当前线程作为目标调用unpark
	* 其他某个线程中断当前线程
	* 该调用不合逻辑的返回

unpark，如果给定的线程尚不可用，则使其可用。如果线程在park上受阻塞，则它将解除其阻塞状态。否则，保证下一次调用park不受阻塞。如果给定线程尚未启动，则无法保证此操作有任何效果。

## CLH队列
AQS的CLH队列是CLH队列的一种变形，主要从两方面进行了改造，节点的结构与节点等待机制，结构上引入了头节点和尾节点，分别指向队列的头和尾。尝试获取锁，入队列，释放锁等实现都与头尾节点相关，并且每个节点都引入前驱节点和后续节点的引用。在等待机制上由原来的自旋改为阻塞唤醒。

线程获取锁时，会调用AQS的acquire方法，该方法第一次尝试获取锁失败，会将该线程加入CLH队列中：

```
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```
addWaiter：

```
private Node addWaiter(Node mode) {
    Node node = new Node(Thread.currentThread(), mode);
    // Try the fast path of enq; backup to full enq on failure
    Node pred = tail;
    if (pred != null) {
        node.prev = pred;
        if (compareAndSetTail(pred, node)) {
            pred.next = node;
            return node;
        }
    }
    enq(node);
    return node;
}
```

Node的源码：

```
static final class Node {
    //共享锁
    static final Node SHARED = new Node();
    //独占锁
    static final Node EXCLUSIVE = null;

    //线程已被取消
    static final int CANCELLED =  1;
    //当前线程的后继线程需要被unpark
    static final int SIGNAL    = -1;
    //线程在等待Condition唤醒
    static final int CONDITION = -2;
    
    static final int PROPAGATE = -3;

    volatile int waitStatus;

    //前继节点
    volatile Node prev;

    //后继节点
    volatile Node next;
    volatile Thread thread;

    Node nextWaiter;

    final boolean isShared() {
        return nextWaiter == SHARED;
    }

	//获取前继节点
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

### 入列
在线程尝试获取锁的时候，如果失败了，需要将该线程加入到CLH队列，入列主要流程：tail执行新建node，然后将node的后继节点指向旧tail值，这个过程中有一个CAS操作，使用自旋方式直到成功。

### 出列
当线程释放锁时，需要进行出列，主要工作是唤醒后继节点，让所有线程有序进行下去。

### 取消
线程因为超时或者中断涉及到取消的操作，该节点就不会参与到锁竞争中，会等待GC回收。取消过程主要是将取消状态的节点移除掉，移除过程先将其状态设为cancelled，然后将前驱节点的pred指向其后继节点。

### 挂起
AQS将自旋机制改为阻塞机制。当前线程首先检测是否为头结点，且尝试获取锁，如果当前节点为头结点并且成功获取到锁，直接返回，当前线程不进入阻塞，否则当前线程阻塞。