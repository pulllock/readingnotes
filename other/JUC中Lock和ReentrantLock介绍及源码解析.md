Lock框架是jdk1.5新增的，作用和synchronized的作用一样，所以学习的时候可以和synchronized做对比。在这里先和synchronized做一下简单对比，然后分析下Lock接口以及ReentrantLock的源码和说明。具体的其他的Lock实现的分析在后面会慢慢介绍。

# Lock框架和synchronized
有关synchronized的作用和用法不在具体说明，应该都很熟悉了。而Lock有着和synchronized一样的语意，但是比synchronized多了一些功能，单单就从Lock接口定义来看，比synchronized多出来的功能有：

- 可中断的获取锁，就是获取锁的线程可以响应中断。
- 可以尝试获取锁，也就是非阻塞获取锁，一个线程可以尝试去获取锁，获取成功就持有锁并返回true，否则返回false。
- 带超时的尝试获取锁，就是在尝试获取锁的时候，会有超时时间，当到达了指定的时间后，还未获取到锁，就返回false。

除了定义之外，Lock框架还和synchronized有不一样的是：

- Lock需要显示的加锁和释放锁，而且一定要在finally中去释放锁。而synchronized则不需要我们去关心锁的释放。
- 锁的公平性，Lock接口并没有定义有关公平性的方法，而是在具体的实现类中使用AQS来实现锁的公平性。

# Lock接口源码

```
public interface Lock {

    //获取锁，获取到锁后返回
    //注意一定要记得释放锁
    void lock();

    //可中断的获取锁
    //获取锁的时候，如果线程正在等待获取锁，则该线程能响应中断
    void lockInterruptibly() throws InterruptedException;

    //尝试获取锁，当线程获取锁的时候，获取成功与否都会立即返回
    //不会一直等着去获取锁
    boolean tryLock();

    //带有超时时间的尝试获取锁
    //在一定的时间内获取到锁会返回true
    //在这段时间内被中断了，会返回
    //在这段时间内，没有获取到锁，会返回false
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

    //释放锁
    void unlock();

    //获取一个Condition对象。
    Condition newCondition();
}
```
Lock接口的定义并不复杂，获取锁释放锁以及非阻塞式的获取锁等方法的定义。其实想象一下日常使用的时候，也大概是如此，获取锁，释放锁，获取锁的时候没有得到锁，我就转一圈回来再试试，等到一定时间之后，我就不要了，走了。接口的定义没有太多要说的，接下来看Lock接口的实现。

# Lock接口的实现
Lock接口主要的实现是ReentrantLock重入锁，另外还有ConcurrentHashMap中的Segment继承了ReentrantLock，在ReentrantReadWriteLock中的WriteLock和ReadLock也实现了Lock接口。

## ReentrantLock
ReentrantLock是一个可重入的互斥锁，等同于synchronized，但是比synchronized更强大灵活，减少死锁发生的概率。我们上面说Lock框架提供了公平锁的机制，就是在ReentrantLock中有提供公平锁机制的实现，默认为非公平锁。

在继续看ReentrantLock的各个方法实现之前，首先需要了解下内部是怎么实现公平锁和非公平锁的，其实想一下也简单，比如我就是一可重入锁ReentrantLock，你想从我这获得到公平的还是不公平的，但是不能我说什么就是什么，我这里有一个天平（AQS），这个是大家公认的可以实现公平和不公平的机器，你找我要，我就给天平说一声，他来操作，然后我再把结果给你。（越描述越复杂！）。几乎ReentrantLock中所有的操作都会交给Sync去实现。

有关AQS这里不做介绍，在AQS专门的文章有介绍，请自行查阅把。接下来就看看我拿着的两把锁，公平和不公平。

### Sync
Sync是公平和非公平两种的基类，直接看代码，看不明白的话，可以先看下面公平和非公平的解析，然后再返回来：

```
//继承自AQS
abstract static class Sync extends AbstractQueuedSynchronizer {

    //由具体的子类实现，即公平和非公平的有不同实现
    abstract void lock();

    //非公平的尝试获取
    final boolean nonfairTryAcquire(int acquires) {
    	//当前线程
        final Thread current = Thread.currentThread();
        //当前AQS同步器的状态
        int c = getState();
        //状态为0，说明没有人获取锁
        if (c == 0) {
        	//尝试获取，获取成功设为独占模式
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        //这里解释跟公平的一样，参照下面的
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
	
    //尝试释放
    protected final boolean tryRelease(int releases) {
        int c = getState() - releases;
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
	
    //是否是独占
    protected final boolean isHeldExclusively() {
        // While we must in general read state before owner,
        // we don't need to do so to check if current thread is owner
        return getExclusiveOwnerThread() == Thread.currentThread();
    }
	
    final ConditionObject newCondition() {
        return new ConditionObject();
    }

    // Methods relayed from outer class
	//获取持有者线程
    final Thread getOwner() {
        return getState() == 0 ? null : getExclusiveOwnerThread();
    }
	//获取重入数
    final int getHoldCount() {
        return isHeldExclusively() ? getState() : 0;
    }
	//是否锁了
    final boolean isLocked() {
        return getState() != 0;
    }

    //
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        s.defaultReadObject();
        setState(0); // reset to unlocked state
    }
}
```

### FairSync
公平锁的实现，有关公平的实现，是此类进行处理的。

```
//继承自Sync
static final class FairSync extends Sync {
	//获取锁
    //公平的lock方法交给了AQS的acquire方法去处理
    //acquire方法采用独占模式，并且忽略中断
    //而AQS获取锁的实现是先使用tryAcquire方法获取，获取不到就加入到队列中，一直尝试获取，直到成功返回，
    //tryAcquire的实现又是具体的子类实现的，下面的tryAcquire方法就是公平的tryAcquire实现
    //
    final void lock() {
    	//参数1是AQS的同步状态
        //首先了解下AQS中同步状态的定义，state
   	 	//0表示未被获取到锁，1表示已经被获取到锁了，大于1表示重入数
        //我们要获取锁，肯定是想要现在的同步状态为0，然后我们把状态变成1，这样锁就是我们的了
        acquire(1);
    }

    //公平的tryAcquire方法实现
    protected final boolean tryAcquire(int acquires) {
    	//当前线程
        final Thread current = Thread.currentThread();
        //获取AQS的同步状态
        int c = getState();
        //状态为0的话，说明没有其他人获取到锁
        if (c == 0) {
        	//hasQueuedPredecessors查询是否还有其他线程比当前线程等待获取锁的时间更长
            //compareAndSetState使用cas来设置状态，预期为0，我们想要设置的值为1
            if (!hasQueuedPredecessors() &&
                compareAndSetState(0, acquires)) {
                //如果我们是等待获取时间最长的（这就是公平，我等的时间长，就该我第一个被服务）
                //并且cas设置成功了，表示我们获取到锁了
                //设置当前线程为独占访问
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        //往下是state不为0的情况，也就是1，或者大于1
        //如果当前线程和独占线程是同一个
        else if (current == getExclusiveOwnerThread()) {
        	//当前状态加上我们要获取的参数1
            //现在表示的是重入数
            int nextc = c + acquires;
            //state是一个32位整型，小于0，表示重入数超过了最大数
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            //设置当前状态
            setState(nextc);
            return true;
        }
        return false;
    }
}
```
可以看到公平的tryAcquire在获取锁的开始只调用一次，获取到就获取到了，或者已经获取到增加重入数，没有获取到就返回false，如果返回false的话，AQS就会将其加入到队列中一直尝试获取。

### NonfairSync

```
//也是继承自Sync
static final class NonfairSync extends Sync {
    //非公平的获取锁
    final void lock() {
    	//先尝试直接获取锁
        //如果能获取到锁，就设置为独占模式
        if (compareAndSetState(0, 1))
            setExclusiveOwnerThread(Thread.currentThread());
        else
        	//直接获取不到的话，就会跟公平锁一样的流程去获取
            //tryAcquire在下面
            acquire(1);
    }
	//这里是非公平的tryAcquire，直接调用Sync中的nofairTryAcquire方法
    protected final boolean tryAcquire(int acquires) {
        return nonfairTryAcquire(acquires);
    }
}
```

### 公平和非公平的区别
上面看完了代码，有点模糊，感觉代码都差不多，到底公平和非公平差别在哪里。首先看下公平的锁获取，公平锁获取会直接调用acquire方法，acquire方法并不是直接去获取锁，而是调用公平的tryAcquire方法，公平的tryAcquire方法首先获取到当前同步器状态，如果没有人用同步器，也就是状态为0，会先去判断有没有人比我等的时间更长，有的话我就不能获取锁，而是让别人先去；如果我是等待最长的那个，我就去使用CAS更改状态，获取锁。

而非公平的实现是，我上来就直接使用CAS获取锁，不问别人是不是等着很长时间了，我获取到了就是我的了，我获取不到，再调用acquire方法，然后acquire方法中调用非公平的tryAcquire方法，非公平的tryAcquire方法也是很直接，如果当前锁没有人用，也就是state为0，我不管有没有人比我等的时间长，我就去获取，然后设置独占。

公平锁，获取锁首先去尝试，没有的话就排队，轮到我之后，还要去问一下有没有等的时间更长的。非公平锁则是不排队，直接上，没有获取到，我也尝试获取，尝试获取的时候我还是直接上，不管其他人。

了解完公平和非公平锁，再去看其他方法就没那么难了。

### ReentrantLock构造函数

```
//可以看到，默认是非公平的
public ReentrantLock() {
    sync = new NonfairSync();
}
```
还可以指定公平性

```
public ReentrantLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
}
```

### lock方法

```
//直接调用公平或者非公平的同步器的lock方法
public void lock() {
    sync.lock();
}
```
lock方法有三种情况：

- 如果锁没有被其他线程持有，当前线程立即获得锁并返回，同步器状态设为1。
- 如果当前线程已经持有锁，则状态加1，并立即返回。
- 如果锁被其他线程持有，当前线程挂起直到获取到锁，然后返回，同步器设为1。

### lockInterruptibly方法

```
//可中断的获取锁
public void lockInterruptibly() throws InterruptedException {
	//调用AQS的方法
    sync.acquireInterruptibly(1);
}
```
获取锁，可以被Thread.interrupt打断。也有三种情况：

- 如果锁没有被其他线程持有，当前线程立即获得锁并返回，同步器状态设为1。
- 如果当前线程已经持有锁，则状态加1，并立即返回。
- 如果锁被其他线程持有，当前线程会挂起去获取锁，在这个过程会有两种情况：
	
    + 当前线程获取到了锁，返回，同步器状态设置1。
    + 当前线程被中断了，会抛出InterruptedException异常，并清除中断状态。

### tryLock方法

```
//尝试获取锁，不会阻塞，成功与否都会直接返回
public boolean tryLock() {
	//使用的是非公平锁的获取
    return sync.nonfairTryAcquire(1);
}
```
使用非公平的锁获取，如果使用了公平的，获取的时候还要判断别人是不是了好久，而非公平的nonfairTryAcquire，能获取就直接获取到，获取不到就返回false，比较直接。

如果不想破坏公平性，可以使用带有超时时间的tryLock方法。

### 带超时的tryLock方法

```
//在超时间内，并且没有被打断，锁没有被其他线程持有，就立即获取到锁并返回
public boolean tryLock(long timeout, TimeUnit unit)
        throws InterruptedException {
    return sync.tryAcquireNanos(1, unit.toNanos(timeout));
}
```

如果使用的是公平锁，如果有其他等的时间更长的线程，即便现在锁没有人持有，当前线程也不会获取到锁，给等的时间更长的去获取。

### unlock方法

```
//释放锁
//直接调用AQS来释放锁
public void unlock() {
    sync.release(1);
}
```

### newCondition方法

```
//返回一个新的Condition实例
public Condition newCondition() {
    return sync.newCondition();
}
```

返回的Condition实例的方法，其实和Object的wait，notify，notifyAll方法的作用一样。

### 其他方法

```
//重入次数
 public int getHoldCount() {
    return sync.getHoldCount();
}

//锁是否被当前线程持有
public boolean isHeldByCurrentThread() {
    return sync.isHeldExclusively();
}

//查询锁是否被任何线程持有
public boolean isLocked() {
    return sync.isLocked();
}

//是否是公平锁
public final boolean isFair() {
    return sync instanceof FairSync;
}

//返回当前拥有锁的线程
protected Thread getOwner() {
    return sync.getOwner();
}

//查询是否有线程在排队获取锁
public final boolean hasQueuedThreads() {
    return sync.hasQueuedThreads();
}

//查询给定的线程是否在等待获取锁
public final boolean hasQueuedThread(Thread thread) {
    return sync.isQueued(thread);
}

//得到正在等待获取锁的队列的长度
public final int getQueueLength() {
    return sync.getQueueLength();
}

//获取正在等待获取锁的所有线程
protected Collection<Thread> getQueuedThreads() {
    return sync.getQueuedThreads();
}

//查询是否有线程正在等待给定的Condition
public boolean hasWaiters(Condition condition) {
    if (condition == null)
        throw new NullPointerException();
    if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
        throw new IllegalArgumentException("not owner");
    return sync.hasWaiters((AbstractQueuedSynchronizer.ConditionObject)condition);
}

//得到正在等待一个Condition的队列的长度
public int getWaitQueueLength(Condition condition) {
    if (condition == null)
        throw new NullPointerException();
    if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
        throw new IllegalArgumentException("not owner");
    return sync.getWaitQueueLength((AbstractQueuedSynchronizer.ConditionObject)condition);
}

//获取所有的等待某个Condition的线程集合
protected Collection<Thread> getWaitingThreads(Condition condition) {
    if (condition == null)
        throw new NullPointerException();
    if (!(condition instanceof AbstractQueuedSynchronizer.ConditionObject))
        throw new IllegalArgumentException("not owner");
    return sync.getWaitingThreads((AbstractQueuedSynchronizer.ConditionObject)condition);
}
```

### ReentrantLock和synchronized的区别
ReentrantLock和synchronized很类似，但是比synchronized多了更多功能，比如可中断锁，锁可以带超时时间，可以尝试非阻塞获取锁等。ReentrantLock还提供了条件Condition，跟Object的wait/notify类似，但是在多个条件变量和高度竞争锁的地方，ReentrantLock更加合适。

另外AQS是重点，一定要多学几遍，学会了，才能掌握锁（我还不太明白！）。

有关其他实现，比如ReentrantReadWriteLock的ReadLock和WriteLock以及ConcurrentHashMap中的Segment会另行介绍。