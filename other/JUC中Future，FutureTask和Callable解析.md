这里简单介绍下Callable，Future，FutureTask等的介绍，用法，还会做一些源码的解析。同时还会对比下和Runnable等知识点的不同。

# Callable接口
Callable和Runnable类似，Callable可以又返回值，而Runnable没有返回值。

Runnable接口定义：

```
//FunctionalInterface是Java8中新的类型，函数式接口，lambda表达式中使用
@FunctionalInterface
public interface Runnable {
    public abstract void run();
}
```
可以看到Runnable的run方法无返回值，当线程启动之后，run就会去执行，但是却不会有返回值。

Callable接口：

```
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```
而Callable的call方法是又返回值的，返回值是定义Callable的时候的泛型类型。

# Future接口
Future接口表示异步任务计算的结果，表示Callable或者Runnable任务执行的结果。看下Future接口的定义：

```
public interface Future<V> {
	//用来取消任务
    boolean cancel(boolean mayInterruptIfRunning);
	//判断任务是否被取消
    boolean isCancelled();
	//判断任务是否已经完成
    boolean isDone();
	//获取任务的执行结果，会阻塞直到任务完成
    V get() throws InterruptedException, ExecutionException;
	//获取任务的执行结果，可指定超时时间
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

- cancle方法，用来取消任务，取消成功返回true，失败返回false。mayInterruptIfRunning参数表示是否取消正在执行的任务：
	
	- true表示可以取消正在执行中的任务，如果此时任务已经完成则返回false；如果任务正在执行则返回true；如果任务还没执行则返回true。
	- false表示不取消正在执行的任务，如果此时任务已经完成则返回false；如果任务正在执行则返回false；如果任务还没执行则返回true。

- isCancelled方法，判断任务是否被取消成功，如果任务在正常完成之前被取消了，则返回true。
- isDone，判断任务是否已经完成，已经完成则返回true。
- get方法，用来获取任务执行的结果，此方法会阻塞一直等到任务完成才返回。
- get(timeout,unit)方法，用来获取执行的结构，可以设置超时时间，如果指定的时间内还没有结果返回，则返回null。

# RunnableFuture接口

```
public interface RunnableFuture<V> extends Runnable, Future<V> {
    void run();
}
```
这个接口表示可以作为Runnable的Future，如果run方法执行成功，则可以通过Future来得到结果。

# FutureTask类
FutureTask类是Future接口的实现，其实它直接实现了RunnableFuture接口，对于Future和RunnableFuture中的方法做了具体的实现

# 例子
比如我们要在下载一个很大的文件，花费的时间很长，这是就可能要异步执行了。

```
public class DownloadFileCallable implements Callable<String> {
    @Override
    public String call() throws Exception {
        System.out.println(Thread.currentThread().getName() + "：" + "正在下载文件。。。");
        Thread.sleep(5000);
        System.out.println(Thread.currentThread().getName() + "：" + "下载完成，花费了5秒多。。。");
        return "这是下载的文件id";
    }
}
```

```
public class Main {
    public static void main(String[] args) throws ExecutionException, InterruptedException {
        DownloadFileCallable downloadFileCallable = new DownloadFileCallable();
        FutureTask<String> futureTask = new FutureTask<String>(downloadFileCallable);

        ExecutorService executor = Executors.newFixedThreadPool(1);
        System.out.println(Thread.currentThread().getName() + "：" + "在执行的时候，这里要下载一个文件");
        executor.execute(futureTask);
        System.out.println(Thread.currentThread().getName() + "：" + "继续做其他的事情。。。");
        Thread.sleep(1000);
        System.out.println(Thread.currentThread().getName() + "：" + "做完了其他事情，等着下载完成");
        while(true){

            if(futureTask.isDone()){
                System.out.println(Thread.currentThread().getName() + "：" + futureTask.get());
                System.out.println(Thread.currentThread().getName() + "：" + "下载好了，可以处理文件了。。。");
                executor.shutdown();
                return;
            }

        }

    }
}
```

执行结果：

```
main：在执行的时候，这里要下载一个文件
main：继续做其他的事情。。。
pool-1-thread-1：正在下载文件。。。
main：做完了其他事情，等着下载完成
pool-1-thread-1：下载完成，花费了5秒多。。。
main：这是下载的文件id
main：下载好了，可以处理文件了。。。
```

# FutureTask源码
## 重要说明
FutureTask实现了RunnableFuture接口，可以在开始的时候看到一段说明，这个版本（这里使用的是jdk1.8.0_121）的FutureTask跟之前版本不一样，之前版本使用AQS来做同步控制，现在的版本使用的是通过一个使用CAS来改变的状态和简单的Treiber stack栈来保存等待的线程。

## 变量

```
//任务的运行时候的状态，初始状态为NEW
//只有set，setException和cancel方法能将状态转换成终止状态
//执行完了就会变成COMPLETING
//可能的状态转换：
//NEW -> COMPLETING -> NORMAL
//NEW -> COMPLETING -> EXCEPTIONAL
//NEW -> CANCELLED
//NEW -> INTERRUPTING -> INTERRUPTED
private volatile int state;
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;

//当前执行的Callable任务
private Callable<V> callable;
//调用get方法后要返回的结果或者是要抛出的异常
//不是volatile类型的，但是会用state来保证安全
private Object outcome; reads/writes
//执行Callable任务的线程
private volatile Thread runner;
//等待线程，使用的Treiber stack栈来保存
private volatile WaitNode waiters;
```

## 构造函数

我们使用的时候一般都是new一个FutureTask，参数是一个Callable或者Runnable，然后把FutureTask交给线程只去执行，执行的就是Runnable的run或者Callable的call方法。首先看下构造函数：

```
//接受一个Callable类型的任务
public FutureTask(Callable<V> callable) {
    if (callable == null)
        throw new NullPointerException();
    this.callable = callable;
    //运行状态设为NEW
    this.state = NEW; 
}
```

```
//接收一个Runnable类型的任务和一个用于返回的结果
public FutureTask(Runnable runnable, V result) {
	//将Runnable转换成Callable类型
    //Executors中的方法，里面会使用RunnableAdapter适配器类来转换
    this.callable = Executors.callable(runnable, result);
    //运行状态设为NEW
    this.state = NEW;
}
```

## run方法

当交给线程运行的时候，会执行run方法：

```
public void run() {
    if (state != NEW ||
        !UNSAFE.compareAndSwapObject(this, runnerOffset,
                                     null, Thread.currentThread()))
        return;
    try {
    	//我们在构造函数中提交的Callable任务
        //Runnable也转成了Callable类型
        Callable<V> c = callable;
        if (c != null && state == NEW) {
            V result;
            boolean ran;
            try {
            	//执行我们覆盖的call方法，返回结果
                result = c.call();
                ran = true;
            } catch (Throwable ex) {
                result = null;
                ran = false;
                //有异常，进行异常处理
                setException(ex);
            }
            //执行完成
            if (ran)
                set(result);
        }
    } finally {
		//当前运行Callable任务的线程设为null
        runner = null;
        int s = state;
        //INTERRUPTED
        if (s >= INTERRUPTING)
            handlePossibleCancellationInterrupt(s);
    }
}
```

先看下正常执行完成后，set(result)方法：

```
protected void set(V v) {
	//执行完成，将状态由NEW变为COMPLETING
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
    	//返回的结果
        outcome = v;
        //将状态设为NORMAL，最后的状态不能在改变了
        UNSAFE.putOrderedInt(this, stateOffset, NORMAL);
        //结束完成
        finishCompletion();
    }
}
```

看下finishCompletion()方法：

```
//从等待的栈中移除并唤醒所有等待的线程
//接着执行done方法
//最后将Callable任务设为null
private void finishCompletion() {
    //循环把栈中的等待的每个Thread都唤醒
    for (WaitNode q; (q = waiters) != null;) {
        if (UNSAFE.compareAndSwapObject(this, waitersOffset, q, null)) {
            for (;;) {
                Thread t = q.thread;
                if (t != null) {
                    q.thread = null;
                    LockSupport.unpark(t);
                }
                WaitNode next = q.next;
                if (next == null)
                    break;
                q.next = null; // unlink to help gc
                q = next;
            }
            break;
        }
    }
	//调用done，子类实现
    done();
	//Callable任务设为null
    callable = null;        // to reduce footprint
}
```

再返回run方法，看发生异常的时候调用的setException(ex)方法：

```
protected void setException(Throwable t) {
	//发生异常将NEW变为COMPLETING
    if (UNSAFE.compareAndSwapInt(this, stateOffset, NEW, COMPLETING)) {
    	//返回的异常
        outcome = t;
        //最终状态设为EXCEPTIONAL
        UNSAFE.putOrderedInt(this, stateOffset, EXCEPTIONAL); 
        //调用结束完成方法
        finishCompletion();
    }
}
```

## get方法

get有两个版本，一个是无参数，会阻塞直到完成，另一个可以指定超时时间。

```
public V get() throws InterruptedException, ExecutionException {
	//当前运行的状态
    int s = state;
    //new或者completing状态，需要等待完成
    if (s <= COMPLETING)
    	//等待完成，返回值是状态
        s = awaitDone(false, 0L);
    //根据状态值来判断返回或者抛异常
    return report(s);
}
```

等待完成的方法awaitDone：

```
private int awaitDone(boolean timed, long nanos)
    throws InterruptedException {
    //超时时间
    final long deadline = timed ? System.nanoTime() + nanos : 0L;
    //保存正在等待线程的stack
    WaitNode q = null;
    boolean queued = false;
    for (;;) {
    	//中断，需要移除所有等待的线程
        //并抛异常
        if (Thread.interrupted()) {
            removeWaiter(q);
            throw new InterruptedException();
        }
		//当前运行的状态
        int s = state;
        //正常，异常，中断等状态
        if (s > COMPLETING) {
            if (q != null)
            	//线程设为null
                q.thread = null;
            //返回
            return s;
        }
        //正在完成状态可以让出线程了
        else if (s == COMPLETING) // cannot time out yet
            Thread.yield();
        else if (q == null)
            q = new WaitNode();
        else if (!queued)
            queued = UNSAFE.compareAndSwapObject(this, waitersOffset,
                                                 q.next = waiters, q);
                                                 //设置了超时时间
        else if (timed) {
            nanos = deadline - System.nanoTime();
            //超时
            if (nanos <= 0L) {
            	//移除所有的等待的线程
                removeWaiter(q);
                return state;
            }
            //没有超时，park
            LockSupport.parkNanos(this, nanos);
        }
        //
        else
            LockSupport.park(this);
    }
}
```

```
//跟上面类似
public V get(long timeout, TimeUnit unit)
    throws InterruptedException, ExecutionException, TimeoutException {
    if (unit == null)
        throw new NullPointerException();
    int s = state;
    if (s <= COMPLETING &&
        (s = awaitDone(true, unit.toNanos(timeout))) <= COMPLETING)
        throw new TimeoutException();
    return report(s);
}
```

## cancel方法

```
//就是各种状态的转换
//具体的参见最上面cancel方法的说明
public boolean cancel(boolean mayInterruptIfRunning) {
    if (!(state == NEW &&
          UNSAFE.compareAndSwapInt(this, stateOffset, NEW,
              mayInterruptIfRunning ? INTERRUPTING : CANCELLED)))
        return false;
    try {    // in case call to interrupt throws exception
        if (mayInterruptIfRunning) {
            try {
                Thread t = runner;
                if (t != null)
                    t.interrupt();
            } finally { // final state
                UNSAFE.putOrderedInt(this, stateOffset, INTERRUPTED);
            }
        }
    } finally {
        finishCompletion();
    }
    return true;
}
```

## isDone方法

```
 public boolean isDone() {
    return state != NEW;
}
```

## isCancelled方法

```
public boolean isCancelled() {
    return state >= CANCELLED;
}
```
