# Stack
以栈的形式来存储数据，特点是后进先出（LIFO），继承自Vector，内部是通过动态数据来实现的。

List体系中LinkedList是以双向链表的数据结构来存储数据的，所以使用LinkedList完全可以达到Stack的效果，但是Stack不能作为双向链表来使用，并且LinkedList不是线程安全的。Stack是线程安全的。

对Stack、同其父类一样、早已不推荐使用。如果真要使用栈这种结构来实现数据存储、推荐使用Deque 接口及其实现提供了LIFO 堆栈操作的更完整和更一致的 set，应该优先使用此 set，而非此类。