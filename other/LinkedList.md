LinkedList 与 ArrayList 一样实现 List 接口，只是 ArrayList 是 List 接口的大小可变数组的实现，LinkedList 是 List 接口链表的实现。基于链表实现的方式使得 LinkedList 在插入和删除时更优于 ArrayList，而随机访问则比 ArrayList 逊色些。

LinkedList 实现所有可选的列表操作，并允许所有的元素包括 null。

除了实现 List 接口外，LinkedList 类还为在列表的开头及结尾 get、remove 和 insert 元素提供了统一的命名方法。这些操作允许将链接列表用作堆栈、队列或双端队列。

此类实现 Deque 接口，为 add、poll 提供先进先出队列操作，以及其他堆栈和双端队列操作。

所有操作都是按照双重链接列表的需要执行的。在列表中编索引的操作将从开头或结尾遍历列表（从靠近指定索引的一端）。

同时，与 ArrayList 一样此实现不是同步的。

LinkedList基于双向链表实现。

LinkedList相对于Arraylist来说,get和set等随机访问会比较慢,LinkedList需要移动指针；add和remove会比较快。

