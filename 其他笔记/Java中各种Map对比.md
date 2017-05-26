- HashMap，基于哈希表的实现，存储key-vaule，底层是数组，数组每一项是链表，使用链表来解决冲突，线程不安全，允许key和value为null。
- LinkedHashMap，存放元素是有序的，是HashMap的子类，默认保留插入的顺序，可以设置按照访问的顺序排序，是一个双向链表，允许key和value为null，线程不安全的。
- TreeMap，支持排序，基于红黑树，线程不安全的，根据自然顺序进行排序，还可以自定义Comparator进行排序。线程不安全的。
- IdentityHashMap，使用`==`判断相等，而HashMap使用equals判断相等，底层是数组，逻辑上可以看成是环形数组，存放元素如发现重入，则往后查找，找到空位置插入。
- WeakHashMap，键是弱引用的，能实现对键值的动态回收。
- ConcurrentHashMap，线程安全的，底层还是数组和链表，使用段锁来保证线程安全。

# 含义

# 使用场景
