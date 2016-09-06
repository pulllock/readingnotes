# 使用多线程

- 继承Thread类 
- 实现Runnable接口

线程是一个子任务，CPU以不确定的方式或者是以随机的时间来调用线程中的run方法。

## currentThread()
该方法可以返回代码段正在被哪个线程调用的信息。

## isAlive()
判断当前线程是否处于活动状态。

## sleep()
是在指定的毫秒数内让当前正在执行的线程休眠，暂停执行，这个正在执行的线程是指`this.currentThread()`返回的线程。

## getId()
取得线程的唯一标识。

## 停止线程
Java中有3种方法可以终止正在运行的线程：

- 使用退出标志，使正常线程退出，也就是当run方法完成后线程终止。
- 使用stop方法强行终止线程，stop和suspend，resume都是过期方法，使用他们可能产生不可预料的结果。
- 使用interrupt方法中断线程。

调用interrupt方法仅仅是在当前线程中打了一个停止的标记，并不是真正的停止线程。

Thread类提供了两个方法判断线程是否是停止状态：

- interrupted() 测试当前线程是否已经中断。当前线程是指运行interrupted()方法的线程。线程的中断状态由该方法清除，如果连续调用两次该方法，第二次返回false。
- isInterrupted() 测试线程是否已经中断。不清除状态标志。

异常停止线程。

使用stop方法停止线程，方法已经被作废，强制停止的话有可能使一些清理工作得不到完成。

## 暂停线程

暂停意味着还能恢复运行，suspend方法暂停线程，resume方法恢复线程执行。

使用suspend与resume方法时，易造成公共的同步对象的独占，使其他线程无法访问公共同步对象。

## yield
放弃当前cpu资源，让其他任务去占用cpu执行时间，放弃时间不确定，有可能刚放弃马上就获得cpu时间片。

## 线程优先级
使用setPriority()方法设置线程的优先级。

线程的优先级具有继承性，A线程启动B线程，B线程的优先级与A是一样的。

## 守护线程

java中有两种线程：

- 用户线程
- 守护线程

当进程中不存在非守护线程，则守护线程自动销毁。垃圾回收线程就是一个典型的守护线程。

# 对象及变量的并发访问

## synchronized同步方法

- 方法内的变量为线程安全，非线程安全存在于实例变量中，方法内部的私有变量不存在非线程安全问题。
- 实例变量非线程安全

调用关键字synchronized声明的方法一定是排队运行的，只有共享资源的读写访问才需要同步化。

synchronized锁拥有重入的功能。自己可以再次获取自己的内部锁。

## synchronized同步语句块
synchronized(this)代码块是锁定当前对象的。

多个线程同时调用同一个对象中的不同名称的synchronized同步方法或synchronized(this)同步代码块，调用效果就是按顺序执行，也就是同步的，阻塞的。

1. synchronized同步方法
	- 对其他synchronized同步方法或synchronized(this)同步代码块调用呈阻塞状态。
	- 同一时间只有一个线程可以执行synchronized同步方法中的代码。
2. synchronized(this)同步代码块
	- 对其他synchronized同步方法或synchronized(this)同步代码块调用呈阻塞状态。
	- 同一时间只有一个线程可以执行synchronized(this)同步代码块中的代码。

synchronized(非this对象)：

	- 在多个线程持有对象监视器为同一个对象的前提下，同一时间只有一个线程可以执行synchronized(非this对象)同步代码块中的代码。
	
## 静态同步synchronized方法与synchronized(class)代码块
synchronized可以应用在static方法上，是对当前.java文件对应的Class类进行持锁。

## 数据类型String的常量池特性
大多数情况下，同步synchronized代码块不使用String作为锁对象。

## 内部类与静态内部类

## volatile关键字
主要作用是使变量在多个线程间可见。强制从公共堆栈中取得变量的值，而不是从线程私有数据栈中取得变量值。缺点是部支持原子性。

1. volatile是线程同步的轻量级实现，比synchronized性能好，volatile只能修饰变量，synchronized可以修饰方法，代码块。
2. 多线程访问volatile不会发生阻塞，而synchronized会出现阻塞。
3. volatile能保证数据的可见性，但不保证原子性，synchronized可以保证原子性，还可以间接保证可见性。
4. volatile解决的是变量在多个线程间的可见性；而synchronized解决的是多个线程之间访问资源的同步性。

### volatile非原子的特性
volatile在多个线程中可以感知实例变量被更改，可以获得最新只。每次从共享内存中读取变量，而不是从私有内存中读取，保证同步数据的可见性。但是修改实例变量的数据不是一个原子操作，非线程安全！

## 使用原子类进行i++ 操作
可以使用AtomicInteger。

# 线程间通信

## 等待通知机制
wait的作用是使当前执行代码的线程进行等待，wait是Object类的方法，该方法用来将当前线程置入预执行队列中，并且在wait所在代码行处停止执行，直到接到通知或被中断为止。

调用wait之前线程必须获得该对象的对象级别锁，只能在同步方法或者同步块中调用wait方法。在执行wait方法之后，当前线程释放锁。

notify也需要在同步方法或者同步块中调用，调用前必须获得该对象的对象级别锁。必须执行完notify方法所在的同步synchronized代码块后才释放锁。

线程呈wait状态时，调用线程对象的interrupt方法会出现InterruptedException异常。

## 通过管道进行线程间通信
pipeStream用于在不同的线程间直接传送数据。

1. PipedInputStream PipedOutputStream
2. PipedReader PipedWriter

## Join
作用是等待线程对象销毁。join具有使线程排队运行的作用，有类似于同步运行的效果。join与synchronized的区别是：join内部使用wait方法进行等待，synchronized使用对象监视器原理作为同步。

## ThreadLocal
变量值的共享可以使用public static变量形式，所有线程都使用同一个public static变量。

ThreadLocal可以实现每个线程都有自己的共享变量。

第一次调用ThreadLocal类的get方法返回值是null。可以使用initialValue方法进行初始化。

## InheritableThreadLocal
可以在子线程中取得父线程继承下来的值。


# Lock使用

## ReentrantLock
使用synchronized关键字可以实现线程间的同步互斥，在JDK1.5中新增的ReentrantLock也能达到同样效果。

lock方法获得锁，unlock方法释放锁。

## Condition

synchronized与wait和notify/notifyAll方法结合可以实现等待/通知模式，类ReentrantLock也可以，需要借助Condition对象。一个Lock里可以创建多个Condition实例。

notify是虚拟机随机选择的，而Condition可以选择性通知。

Object类中的wait方法相当于Condition类中的await方法。

Object类中的wait(long timeout)方法相当于Condition类中的await(long time,TimeUnit unit)

Object类中的notify方法相当于Condition类中的signal。

Object类中的notifyAll方法相当于Condition类的signalAll方法。

## 公平锁与非公平锁

公平锁表示线程获取的顺序是按照线程加锁的顺序来分配的，FIFO。非公平锁是一种获取锁的抢占机制，是随机获得的。





