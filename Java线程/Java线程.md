# Thread的创建与管理
## 创建Thread

* 使用Thread类
* 使用Runnable接口，需要创建一个Thread的实例

# 数据同步
## synchronized
## volatile
## Lock

```
lock()
unlock()
tryLock()
tryLock(long time,TimeUnit unit)

```

# Thread Notification
## 等待与通知

```
void wait() 必须从synchronized方法或块中调用。
void notify() 必须从synchronized方法或块中调用。
void notifyAll() 必须从synchronized方法或块中调用。
```

## 条件变量
Condition：

```
await()
awaitUninterruptibly()
awaitNanos()
awaitUntil(Date deadline)
signal()
signalAll()
```

# 极简同步技巧
## Atomic变量

```
compareAndSet()
weakCompareAndSet()
这两个方法需要两个参数，预期数据所具有的值，以及要把数据所设定成的值。方法只会在变量具有预期值的时候才会将它设定成新值，如果当前值不等于预期值，该变量不会被重新赋值并返回false；如果当前值等于预期值，返回true，值会被设置为新值。

incrementAndGet()
decrementAndGet()
getAndIncrement()
getAndDecrement()

addAndGet()
getAndAdd()
```

## Thread局部变量
ThreadLocal

```
protected T initialvalue();
public T get();
public void set(T value);
public void remove();
```

## 可继承的Thread局部变量
InheritableThreadLocal

# 高级同步议题
## J2SE 5.0中加入的同步Class
### Semaphore
信号量

会记录它可以发出的许可数目。

```
public void acquire();
public boolean tryAcquire();
public void release();
```

### Barrier
Barrier就是一个等待的点，所有的Thread可以在这个点上合并结果或者是安全的接下去工作。

Barrier可以重复使用。

CyclicBarrier

```
public int await()
public void reset()
public boolean isBroken();
public int getParties();
public int getNumberWaiting();
```

### 倒数的Latch
CountDownLatch

Thread是在指定的数目倒数到0的时候被释放。

```
public void await();
public boolean await();
public void countDown();
public long getCount();
```

### Exchanger

```
public V exchange(V x);
```

exchange()方法要使用与其他thread交换的数据对象来被调用，如果其他的Thread已经在等待，exchange()方法会以此Thread的数据来返回；如果没有其他的Thread在等待，则该方法会等待一个Thread。

### 读写锁
ReadWriteLock,ReentrantReadWriteLock

```
Lock readLock();
Lock writeLock();
```

# Thread与Collection Class

## 集合接口
List

Map

Set

Queue

## 线程安全的集合类
Vector

Stack

Hashtable

ConcurrentHashMap

CopyOnWriteArrayList

CopyOnWriteArraySet

ConcurrentLinkedQueue

## 非线程安全的集合类
BitSet

HashSet

HashMap

WeakHashMap

TreeMap

ArrayList

LinkedList

LinkedHashSet

IdentityHashMap

EnumSet

EnumMap

PriorityQueue

## Thread-Notification 集合类
ArrayBlockingQueue

LinkedBlockingQueue

SynchronousQueue

PriorityBlockingQueue

DelayQueue


# Thread Pool
## Executor
## ExecutorService
## ThreadPoolExecutor

## Callable Task与Future结果
### Callable
### Future
### FutureTask

# Task的调度

## Timer
TimerTask

## ScheduledThreadPoolExecutor

# Thread与I/O
## 新的I/O服务器
### Noblocking I/O
