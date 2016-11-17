ArrayList是内部是以动态数组的形式来存储数据的。这里的动态数组不是意味着去改变原有内部生成的数组的长度、而是保留原有数组的引用、将其指向新生成的数组对象、这样会造成数组的长度可变的假象。

ArrayList具有数组所具有的特性、通过索引支持随机访问、所以通过随机访问ArrayList中的元素效率非常高、但是执行插入、删除时效率比较低下。

ArrayList实现了AbstractList抽象类、List接口、所以其更具有了AbstractList和List的功能、前面我们知道AbstractList内部已经实现了获取Iterator和ListIterator的方法、所以ArrayList只需关心对数组操作的方法的实现。

ArrayList实现了RandomAccess接口、此接口只有声明、没有方法体、表示ArrayList支持随机访问。

ArrayList实现了Cloneable接口、此接口只有声明、没有方法体、表示ArrayList支持克隆。

ArrayList实现了Serializable接口、此接口只有声明、没有方法体、表示ArrayList支持序列化、即可以将ArrayList以流的形式通过ObjectInputStream/ObjectOutputStream来写/读。

ArrayList实现了List接口，内部以动态数组存储数据，允许重复。

ArrayList具有数组所具有的特性，通过索引支持随机访问。访问元素性能高，插入，删除效率低。但是在数组末尾插入元素效率高。默认初始容量为10，随着ArrayList中元素的增加，容量也会不断自动增长。每次添加新元素时，都会检查是否需要进行扩容操作，扩容需要重新拷贝数组，比较耗时，所以如果能预先知道数组大小，可以在初始化的时候指定一个初始容量。

ArrayList非线程安全，不是同步的。
# 参考
[http://blog.csdn.net/crave_shy/article/details/17436773](http://blog.csdn.net/crave_shy/article/details/17436773)