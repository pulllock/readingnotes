Condition，条件，对于线程而言，它为线程提供了一个含义，以便在某种状态可能为true的另一个线程通知它之前，一直挂起该线程。因为访问此共享状态信息发生在不同的线程中，所以必须受保护，因此需要将某种形式的锁与该条件相关联。

Lock代替了synchronized方法和语句的使用，Condition代替了Object监视器方法（wait，notify，notifyAll）的使用。

Condition实例实质上被绑定到一个锁上，要为特定Lock实例获得Condition实例，请使用newCondition方法。

Condition中用await()替换wait()，signal()替换notify()，signalAll()替换notifyAll()。