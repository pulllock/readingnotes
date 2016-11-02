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
```