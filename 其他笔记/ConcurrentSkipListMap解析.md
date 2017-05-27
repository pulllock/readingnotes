ConcurrentSkipListMap，线程安全，是有序哈希表，和TreeMap类似，TreeMap是非线程安全的。支持高并发，多线程环境下可以代替TreeMap

ConcurrentSkipListMap内部数据结构是跳表（SkipList），支持排序。TreeMap则是使用红黑树实现。

跳表数据结构理解可以参考[http://www.tengleitech.com/archives/1297](http://www.tengleitech.com/archives/1297)这个。
