信号量是一个控制访问多个共享资源的计数器，本质上是一个共享锁。

Java并发提供了两种加锁模式：共享锁和独占锁。ReentrantLock是独占锁，每次只能有一个线程持有，而共享锁不同，允许多个线程并行持有锁，并发访问共享资源。

共享锁放宽了加锁条件，采用了乐观锁机制，允许多个读线程同时访问同一个共享资源。

Semaphore是一个计数信号量，维护了一个许可集，如果有必要，在许可可用之前会阻塞每一个acquire()，然后再获取该许可。每个release()添加一个许可，从而释放一个正在阻塞的获取者。

Semaphore用于限制可以访问某些资源的线程数目。

当Semaphore等于1时，可当做互斥锁使用。等于0时，排他，其他线程必须要等待。

Semaphore和ReentrantLock一样都包含公平锁和非公平锁。

## 信号量的获取acquire()
公平锁和非公平锁的差别：对于公平锁而言，如果当前线程不在CLH队列的头部，需要排队等候，非公平锁则直接获取锁。公平信号量和非公平信号量也是一样。

公平信号量和非公平信号量区别就在tryAcquireShared()方法中。

公平信号量：

```
protected int tryAcquireShared(int acquires) {
    for (;;) {
    	//判断该线程是否位于CLH队列的列头，如果是返回-1
        if (hasQueuedPredecessors())
            return -1;
        //获取当前信号量许可
        int available = getState();
        //剩余信号量许可数
        int remaining = available - acquires;
        //如果剩余信号量大于0，则设置可获取的信号量为剩余的。
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

doAcquireSharedInterruptibly方法，主要两个工作：第一，尝试获取共享锁；第二，阻塞线程直到线程获取共享锁。

非公平锁：

```
final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```
不判断是不是在CLH队列头，直接判断信号量。

## 信号量的释放release
没有公平和非公平之分。release()释放线索所占有的共享锁，它首先通过tryReleaseShared尝试释放共享锁，如果成功直接返回，如果失败则调用doReleaseShared来释放共享锁。