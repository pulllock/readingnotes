# 线程管理
## 线程的创建和运行
* 继承Thread类，并且覆盖run方法。
* 实现Runnable接口，使用带参数的Thread构造器来创建Thread对象。

## 线程信息的获取和设置
ID，Name，Priority，Status

## 线程的中断
isInterrupted()不能改变interrupted属性的值，interrupted()能设置interrupted的属性为false。

## 线程中断的控制
InterruptedException异常，检查到线程中断的时候，抛出异常，在run()方法中捕获并处理异常。

## 线程的休眠和恢复
调用sleep()方法之后，线程会释放CPU且不再继续执行任务。

休眠中线程被中断，该方法立即抛出InterruptedException异常。

yield()方法，通知JVM这个线程对象可以释放CPU了，并不保证遵循这个要求。通常只做调试使用。

## 等待线程的终止
当一个线程对象的join()方法被调用时，调用它的线程将被挂起，直到这个线程对象完成它的任务。

## 守护线程的创建和运行
setDemon()方法只能在start()方法被调用之前设置。

## 线程局部变量的使用
ThreadLocal()

InheritableThreadLocal 如果一个线程是从其他某个线程中创建的，这个类将提供继承的值。

## 线程的分组
ThreadGroup类表示一组线程。

## 使用工厂类创建线程
ThreadFactory接口，实现了线程对象工厂。

该接口只有一个方法newThread，以Runnable接口对象作为参数并且返回一个线程对象。

# 线程同步基础
## Synchronized
如果一个对象已用synchronized关键字声明，那么只有一个执行线程被允许访问它，如果其他某个线程试图访问这个对象的其他方法，它将被挂起，直到第一个线程执行完成正在运行的方法。

每一个用synchronized关键字声明的方法都是临界区，同一个对象的临界区在同一时间只有一个允许被访问。

synchronized声明的静态方法，同时只能够被一个执行线程访问，但是其他线程可以访问这个对象的非静态方法。

## 使用非依赖属性实现同步
使用synchronized关键字来保护代码块时，必须把对象引用作为传入参数。使用this来引用执行方法所属的对象，还可以使用其他对象。

## 在同步代码中使用条件
一个线程调用wait()方法时，JVM将这个线程置入休眠，并且释放控制这个同步代码块的对象,同时允许其他线程执行这个对象控制的其他同步代码块。为了唤醒这个线程，必须在这个对象控制的某个同步代码块中调用notify()或者notifyAll()方法。

## 使用锁实现同步
基于Lock接口及其实现类如ReentrantLock，提供了更多的好处。

* 支持灵活的同步代码块结构，只能在同一个synchronized结构中获取和释放控制。Lock接口允许实现更复杂的临界区结构。
* Lock提供了更多功能，tryLock()方法，试图获取锁，如果锁已被其他线程获取，它将返回false并继续往下执行代码。而使用synchronized关键字时，如果线程试图执行一个已被其他线程正在执行的同步代码块，此线程将被挂起。
* Lock接口允许分离读和写操作，允许多个读线程和只有一个写线程。
* 相比synchronized关键字，Lock接口具有更好的性能。

## 使用读写锁实现同步数据访问
ReadWriteLock接口和实现类ReentrantReadWriteLock，这个类有两个锁，一个是读操作锁，一个是写操作锁。

## 锁的公平性
ReentrantLock和ReentrantReadWriteLock类的构造器都含有一个布尔参数fair，默认是false非公平模式，true为公平模式

## 在锁中使用多条件
一个锁可能关联一个或者多个条件，这些条件通过Condition接口声明，目的是允许线程获取锁并且查看等待的某一个条件是否满足，如果不满足就挂起直到某个线程唤醒它们。Condition接口提供了挂起和唤起线程的机制。

与锁绑定的所有条件对象都是通过Lock接口声明的newCondition方法创建的。在使用条件的时候，比须获取这个条件绑定的锁，所以带条件的代码必须在调用Lock对象的lock()方法和unlock()方法之间。

当线程调用条件的await()方法时，它将自动释放这个条件绑定的锁，其他某个线程才可以获取这个锁并且执行相同的操作，或者执行这个锁保护的另一个临界区代码。

线程调用了条件对象的signal()或者signalAll()方法后，一个或者多个在该条件上挂起的线程将被唤醒。

awaitUninterruptibly()不可中断的，这个线程将休眠直到其他某个线程调用了将它挂起的条件的sigal()或signalAll()方法。

awaitUntil(Date date)直到发生以下情况之一之前，线程将一直处于休眠状态：

* 其他某个线程中断当前线程。
* 其他某个线程调用了将它挂起的条件的signal()或signalAll()方法。
* 指定的最后期限到了。

# 线程同步辅助类
* 信号量Semaphore，是一种计数器，用来保护一个或者多个共享资源的访问。
* CountDownLatch 在完成一组正在其他线程中执行的操作之前，它允许线程一直等待。
* CyclicBarrier 它允许多个线程在某个集合点处进行相互等待。
* Phaser 它把并发任务分成多个阶段运行，在开始下一阶段之前，当前阶段中的所有线程都必须执行完成，Java7 新特性
* Exchanger 提供了两个线程之间的数据交换点。

## 资源的并发访问控制
如果线程要访问一个共享资源，它必须先获得信号量，如果信号量的内部计数器大于0，信号量将减1，然后允许访问这个共享资源。

否则如果信号量计数器等于0，信号量将会把线程置入休眠直至计数器大于0。

线程使用完某个共享资源时，信号量必须被释放，使信号量的内部计数器加1。

通过acquire()方法获得信号量，使用release()方法释放信号量。

acquireUninterruptibly()忽略线程的中断并且不会抛出任何异常。

tryAcquire()试图获得信号量，如果能获得就返回true，如果不能就返回false，从而避开线程的阻塞和等待信号量的释放。

信号量的公平性，默认为非公平。

## 资源的多副本的并发访问控制

## 等待多个并发事件的完成
CountDownLatch 在完成一组正在其他线程中执行的操作之前，它允许线程一直等待。

当一个线程要等待某些操作先执行完时，需要调用await()方法，让线程进入休眠直到等待的所有操作都完成。当某一个操作完成后，它将调用countDown()方法，将CountDownLatch类的内部计数器减1。当计数器变成0时，将唤醒所有调用await()方法而进入休眠的线程。

CountDownLatch对象内部计数器被初始化之后就不能再次初始化或者修改。当计数器到达0时，所有因调用await()而等待的线程立刻被唤醒，再执行countDown()将不起作用。

## 在集合点的同步
CyclicBarrier 允许两个或者多个线程在某个点上进行同步。

当一个线程到达指定的点后，调用await()等待其他线程，当最后一个线程调用await()方法时，CyclicBarrier对象将唤醒所有在等待的线程，然后这些线程将继续执行。

CyclicBarrier可以以一个Runnable对象作为初始化参数，当所有线程到达集合点后，这个Runnable对象作为线程执行。

CyclicBarrier和CountDownLatch有很多共性，但CyclicBarrier对象可以被重置回初始状态，并把它的内部计数器重置成初始化时的值。

## 并发阶段任务的运行
Phaser 它允许执行并发阶段任务。

## 并发任务间的数据交换
Exchanger 允许在并发任务之间交换数据。Exchanger类允许在两个线程之间定义同步点，当两个线程都到达同步点时，他们交换数据结构。

# 线程执行器
Executor接口，子接口ExecutorService，实现类ThreadPoolExecutor

这套机制分离了任务的创建和执行。通过使用执行器，仅需要实现Runnable接口的对象，然后将这些对象发送给执行器即可。

Callable类似于Runnable接口，主方法名为call()，可以返回结果；当发送一个Callable对象给执行器时，将获得一个实现了Future接口的对象，可以使用这个对象来控制Callable对象的状态和结果。

## 创建线程执行器
使用执行框架的第一步是创建ThreadPoolExecutor对象，可以ThreadPoolExecutor类提供的四个构造器或者使用Executors工厂类来创建ThreadPoolExecutor对象。有了执行器就可以将Runnable或Callable对象发送给它执行。

一旦创建了执行器，就可以使用执行器的execute()方法来发送Runnable或Callable类型的任务。

结束执行使用shutdown()方法，拒绝执行新任务，抛出RejectedExecutionException异常。

shutdownNow()立即关闭执行器，不在执行正在等待执行的任务。

## 创建固定大小的线程执行器
newFixedThreadPool()；创建具有线程最大数量的执行器，如果发送超过线程数的任务给执行器，剩余的任务将被阻塞直到线程池里有空闲的线程来处理他们。

## 在执行器中执行任务并返回结果
Callable的call()方法，可以在这个方法里实现任务的具体逻辑操作。

Future这个接口声明了一些方法来获取由Callable对象产生的结果，并管理他们的状态。

调用Future对象的get()方法时，如果Future对象所控制的任务并未完成，这方法将一直阻塞到任务完成。

## 运行多个任务并处理第一个结果
ThreadPoolExecutor的invokeAny()方法接收到一个任务列表，然后开始执行，并返回第一个完成任务并且没有抛出异常的任务的执行结果。

## 运行多个任务并处理所有结果
invokeAll()方法等待所有任务的完成，这个方法接收一个Callable对象列表，并返回一个Future对象列表。

## 在执行器中延时执行任务
## 在执行器中周期性执行任务
ScheduledThreadPoolExecutor类来执行周期性任务。

## 在执行器中取消任务
Future接口的cancel()方法执行取消操作。

* 如果任务已完成或者之前已被取消，那么方法将返回false并且任务也不能取消
* 还未执行的，则不会执行；如果已在运行，cancel()方法参数为true，任务将取消，参数为false，不取消。

## 在执行器中控制任务的完成
FutureTask类提供了一个名为done()的方法，允许在执行器中的任务执行结束后还可以执行一些代码。可以被用来执行一些后期处理操作。

## 在执行器中分离任务的启动与结果的处理
CompletionService类有一个方法来发送任务给执行器，还有一个方法为下一个已经执行结束的任务获取Future对象。

## Fork/Join框架
用来解决能够通过分治技术将问题拆分成小任务的问题。在一个任务中，先检查将要解决的问题的大小，如果大于一个设定的大小，那就将问题拆分成可以通过框架来执行的小任务。如果问题的大小比设定的大小要小，就可以直接在任务里解决这个问题，然后根据需要返回任务的结果。

* 分解Fork操作 当需要将一个任务拆分成更小的多个任务时，在框架中执行这些任务。
* 合并Join操作 当一个主任务等待其创建的多个子任务的完成执行。

使用Join操作让一个主任务等待它所创建的子任务的完成，执行这个任务的线程称之为工作者线程。工作者线程寻找其他仍未被执行的任务，然后开始执行。

Fork/Join框架执行的任务有以下限制：

* 任务只能使用fork()和join()操作当做同步机制。
* 任务不能执行I/O操作。
* 任务不能抛出非运行时异常，必须在代码中处理掉这些异常。

Fork/Join框架核心组成:

* ForkJoinPool 这个类实现了ExecutorService接口和工作窃取算法。他管理工作者线程，并提供任务的状态信息，以及任务的执行信息。
* ForkJoinTask 这个类是一个将在ForkJoinPool中执行的任务基类。

为了实现Fork/Join任务，需要实现以下：

* RecursiveAction 用于任务没有返回结果的场景
* RecursiveTask 用于任务有返回结果的场景

## 创建Fork/Join线程池
调用invokeAll()方法来执行一个主任务所创建的多个子任务。这是一个同步调用，这个任务将等待子任务的完成，然后继续执行。

## 合并任务的结果
Fork/Join框架提供了执行任务并返回结果的能力，这些类型的任务都是通过RecursiveTask类来实现的。RecursiveTask继承了ForkJoinTask类，并且实现了由执行器框架提供的Future接口。

## 异步运行任务
在ForkJoinPool中执行ForkJoinTask时，可以采用同步或异步方式。

## 取消任务
在任务开始执行前可以取消它。cancel()方法。

ForkJoinPool类不提供任何方法来取消线程池中正在运行或者等待运行的所有任务。

取消任务时，不能取消已经被执行的任务。

# 并发集合
Java提供了两类适用于并发应用的集合：

* 阻塞式集合
* 非阻塞式集合

## 使用非阻塞式线程安全列表
ConcurrentLinkedDeque实现非阻塞式并发列表。

## 使用阻塞式线程安全列表
LinkedBlockingDeque实现阻塞式列表。

## 使用按优先级排序的阻塞式线程安全列表
PriorityBlockingQueue，所有添加进去的元素必须实现Comparable接口。

## 使用带有延迟元素的线程安全列表
DelayQueue。

## 使用线程安全可遍历映射
ConcurrentNavigableMap

ConcurrentSkipListMap

## 生成并发随机数
ThreadLocalRandom

## 使用原子变量
AtomicLong等。

## 原子数组
AtomicIntegerArray等。


