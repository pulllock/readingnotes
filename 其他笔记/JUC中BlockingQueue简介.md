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