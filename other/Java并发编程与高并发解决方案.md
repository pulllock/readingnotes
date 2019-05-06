# Java并发编程与高并发解决方案

## CPU多级缓存

1. 时间局部性：如果某个数据被访问，那么在不久的将来它很可能被再次访问。
2. 空间局部性：如果某个数据被访问，那么与它相邻的数据很快也可能被访问。

## 缓存一致性

缓存一致性MESI

## 安全发布对象

- 在静态初始化方法中初始化一个对象引用。
- 将对象保存到volatile类型或者AtomicReference对象中。
- 将对象引用保存到某个正确构造对象的final类型域中。
- 将对象的引用保存到一个由锁保护的域中。

## 不可变对象

- 对象创建以后其状态就不能修改。
- 对象所有的域都是final类型。
- 对象是正确创建的，在对象创建期间，this引用没有逸出。

final关键字，可用来修饰类、方法、变量。

Collections.unmodifiableXXX。

Guava的ImmutableXXX。

## 线程封闭

ThreadLocal。

## 线程安全-同步容器

ArrayList -> Vector,Stack。

HashMap -> HashTable。

Collections.synchronizedXXX(List,Set,Map)。



### 并发容器J.U.C

ArrayList -> CopyOnWriteArrayList。

HashSet、TreeSet -> CopyOnWriteArraySet、ConcurrentSkipListSet。

HashMap、TreeMap -> ConcurrentHashMap、ConcurrentSkipListMap。

### AQS

CLH队列。

Sync queue 同步队列，双向列表。

Condition queue 单向列表。

- 使用Node实现FIFO队列，可用于构建锁或者其他同步装置的基础框架。
- 使用int类型表示状态。
- 使用方法是继承。
- 子类通过继承并通过实现它的方法管理其状态的方法操作状态。
- 可以同时实现排它锁和共享锁模式。

### AQS同步组件

- CountDownLatch
- Semaphore
- CyclicBarrier
- ReentrantLock
- Condition
- FutureTask

#### CountDownLatch

countDown

await

#### Semaphore

#### CyclicBarrier

CountDownLatch只能使用一次，CyclicBarrier可以使用多次。

 CountDownLatch表示一个或者多个线程需要等待其他线程完成某项操作后才能继续执行。

CyclicBarrier实现了多个线程之间相互等待，直到所有线程都满足了之后，才能继续往下执行。

#### ReentrantLock与锁

可指定是公平锁还是非公平锁。

提供了一个Condition类，可以分组唤醒需要唤醒的线程。

提供中断等待锁的线程机制，lock.lockInterruptibly()。

#### ReentrantReadWriteLock

#### StampedLock

#### Condition

### FutureTask

#### Callable与Runnable

#### Future

#### FutureTask

### Fork/Join框架

### BlockingQueue

#### ArrayBlockingQueue

#### DelayQueue

#### LinkedBlockingQueue

#### PriorityBlockingQueue

#### SynchronousQueue

## 线程池

### ThreadPoolExecutor

- corePoolSize 核心线程数量
- maximumPoolSize 线程最大线程数
- workQueue 阻塞队列
- keepAliveTime
- unit
- threadFactory 线程工厂
- rejectHandler

状态

- Running
- Shutdown
- stop
- Tidying
- Terminated

方法

- execute() 提交任务，交给线程池执行
- submit() 提交任务，能返回结果
- shutdown()
- shutdownNow()
- getTaskCount
- getCompletedTaskCount()
- getPoolSize()
- getActiveCount()

### Executors

- newCachedTheadPool
- newFixedThreadPool
- newScheduledTheadPool
- newSingleThreadExecutor

## 死锁

必要条件：

- 互斥条件
- 请求和保持条件
- 不剥夺条件
- 环路等待条件

## 多线程并发最佳实践

- 使用本地变量
- 使用不可变类
- 最小化锁的作用域范围 S=1/(1-a+a/n)
- 使用线程池的Executor，不直接使用new Thread执行
- 宁可使用同步，不要使用线程的wait和notify
- 使用BlockingQueue实现生产-消费模式
- 使用并发集合，而不是加锁的同步集合
- 使用Semaphore创建有界的访问
- 使用同步代码块，不使用同步方法
- 避免使用静态变量

## Spring与线程安全

## HashMap与ConcurrentHashMap

## 高并发之扩容思路

- 垂直扩容
- 水平扩容

### 扩容-数据库

- 读操作扩展：memcache、redis、CDN
- 写操作扩展：Cassandra、Hbase