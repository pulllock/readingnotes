java.util.Queue支持获取元素时等待队列变为非空，存储元素时等待队列变可用。

BlockingQueue中的方法在使用时，如果不能被成功调用，可能出现以下四种情况：

1. 抛异常
2. 返回特殊值，null或者false等
3. 在操作成功之前无限期的阻塞当前线程
4. 在给定的时间内阻塞

|  |抛异常|返回特殊值|阻塞|超时|
|----------|-----|-----|-----|----------|
|插入方法|add(e)|offer(e)|put(e)|offer(e,time,unit)|
|移除方法|remove()|poll()|take()|poll(time,unit)|
|检查方法|element|peek()|-|-|

BlockingQueue不接受null元素，add、put、offer一个null时会抛空指针异常。null用来表示poll操作失败的返回值。

BlockingQueue会有容量的限制，它会有一个剩余容量remainingCapacity，超过了此容量添加元素的put操作将会被阻塞。

不指定BlockingQueue的容量的话，最大容量为Integer.MAX_VALUE。

BlockingQueue的实现主要被设计用来解决生产者-消费者问题，另外还支持java.util.Collection接口。例如可以使用remove(x)方法随意的从队列中移除元素，但是此类操作一般不是很常用，只在个别情形下使用，比如当队列中消息被取消的时候。

BlockingQueue的实现是线程安全的。所有队列方法自动使用内部锁或者是其他形式的并发控制。但是像addAll、containsAll、retainAll、removeAll、不需要自动执行，除非特别说明了。比如，addAll(c)方法添加元素时候，如果只添加成功一部分的话，会失败抛异常。

BlockingQueue不支持close和shutdown操作。

