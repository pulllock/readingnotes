# 多进程多线程概述

Java 多线程的实现有两种方式：

* 继承Thread类
* 实现Runnable接口

# 多线程详解

## 继承Thread创建线程

* 继承Thread类
* 重写run方法
* 调用线程的start方法

### Thread 中的Template Design

Thread使用的是Template Design

## 线程的状态

四个基本状态：

* 初始化（被创建）
* 运行状态
* 冻结状态
* 终止状态（死亡）

### 线程的初始化
就是创建了一个线程，实例化一个Thread的子类。

### 线程的运行状态
创建完线程之后，显式调用了start方法。

### 线程的冻结状态
线程被调用了sleep方法或者wait方法之后，放弃了CPU的执行权。可以继续回到运行状态，也可以直接到死亡状态，比如被中断或者出现异常。

### 线程的死亡状态
出现了致命的异常导致线程被死亡，或者是线程执行逻辑执行完毕线程正常死亡。

## 实现Runnable接口创建线程

一个线程只能被启动一次。

### Runnable和Thread的区别

* Runnable是一个可执行任务的标识，Thread才是线程所有API的体现
* 继承了Thread就没法继承其他的类，实现了Runnable接口也可以继承其他类并且实现其他接口。
* 将任务执行单元和线程的执行控制区分开，才是引入Runnable最主要的目的。

### 线程中的策略模式

# 线程的同步

## 同步代码块
### 如何定义一个锁

* 所谓加锁，就是为了防止多个线程同时操作一份数据，如果多个线程操作的数据都是各自的，没有必要加锁。
* 共享数据的锁对于访问他们的线程来说必须是同一份，否则锁只能是私有的锁，各锁各的，起不到保护共享数据的目的。
* 锁的定义可以时任意一个对象，该对象可以不参与任何运算，只要保证在访问的多个线程看起来是唯一的即可。

## 同步方法
方法名前加synchronized

## this锁与static锁
### this锁
同步方法其实用到的锁时this锁。

### static锁
静态锁，锁是类字节码信息，如果一个类的方法位静态方法，那我们需要通过该类的class信息进行加锁

## 线程的休眠

sleep 让当前的运行线程进入休眠状态，也就是主动放弃CPU执行权

```
//在指定的毫秒数内让当前正在执行的线程休眠，暂停执行
sleep(long millis);
//在指定的毫秒数加指定的纳秒数内让当前正在执行的线程休眠，暂停执行
sleep(long millis,int nanos)
```

## 单例模式
### 饿汉模式

饿汉单例模式就是将类的静态实例作为该类的一个成员变量，在jvm加载它的时候就已经创建了该类的实例，因此不会存在多线程的安全问题

```
public class SingletonTest{
	private final static SingletonTest instance = new SingletonTest();
	private SingletonTest(){}
	
	public static SingletonTest getInstance(){
		return instance;
	}
}
```

不存在线程安全问题，但是存在性能上的问题，提前对实例进行了初始化。

### 懒汉模式

实例虽然作为该类的一个实例变量，但是不主动进行创建，只有在第一次使用的时候才会被创建。

```
public class SingletonTest{
	private static SingletonTest instance = null;
	
	private SingletonTest(){}
	
	public static SingletonTest getInstance(){
		if(null == instance){
			instance = new SingletonTest();
		}
		return instance;
	}
}
```
存在线程安全问题，可以使用`synchronized`关键字：

```
public synchronized static SingletonTest getInstance(){
	if(null == instance){
		instance = new SingletonTest();
	}
	return instance;
}
```

这样做的话效率是相当低的，每次调用都要获取锁。可进行如下改进：

```
public static SingletonTest getInstance(){
	if(null == instance){
		synchronized(SingletonTest.class){
			if(null == instance){
				instance = new SingletonTest();
			}
		}
	}
	return instance;
}
```

## 死锁

# 线程间的通讯
## 生产者消费者
## 多线程下的生产者消费者模式

## Object类wait，notify，notifyAll详解

### wait
wait和sleep一样是放弃cpu执行权，但是wait需要等待另外一个持有相同锁的线程对其进行唤醒操作，并且wait方法必须有一个同步锁，否则抛异常`java.lang.IllegalMonitorStateException: current thread
not owner`。

### notify
是将之前处在临时状态的线程唤醒，并且获取执行权，等待cpu再次调度，必须和之前的wait方法用到的是同一个锁。

### notifyAll
notify是唤醒一个正处在阻塞状态的线程，在Jvm中存在一个线程队列或者线程池的概念，notify将严格按照FIFO的方式唤醒在队列中与自己保持有同样一把锁的线程。

notifyAll是将所有wait中的线程都进行唤醒，唤醒的线程保持有和自己一样的锁。

# 守护线程与线程的优先级

## 守护线程
一个后台线程，主线程一旦结束，他就会随之结束。

设置守护线程`setDaemon(true)`

## 线程的yield
yield方法是短暂放弃CPU执行权。

## 线程的停止
之前调用stop方法可以停止，但是该方法已经被废弃，存在线程安全问题。

- run方法中的业务逻辑执行完毕
- 死循环退出

## 线程的优先级

线程的优先级别高，就是获得CPU执行权的几率高。

`void setPriority(int newPriority)`

## join

临时加入一个线程，等到改线程执行结束后才能运行主线程。

## interrupt
将处在阻塞中的线程打断，也就是将线程从阻塞状态转换到临时状态或者其他状态，执行该方法会抛出一个异常InterruptedException。


# 线程池的实现

线程的销毁和创建是比较耗资源的。

## 线程组

```
//创建一个线程组
public ThreadGroup(String name) {
	this(Thread.currentThread().getThreadGroup(), name);
}

//把一个线程组当做参数，创建一个线程组。被传递的是父线程组
public ThreadGroup(ThreadGroup parent, String name) {
        this(checkParentAccess(parent), parent, name);
 }
 
 int activeCount() 返回此线程组中活动线程的计数。
 
 int activeGroupCount 返回此线程组中活动的线程组的计数
```

# 线程状态的监控






