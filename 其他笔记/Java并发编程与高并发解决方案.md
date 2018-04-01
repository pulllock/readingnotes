# Java并发编程与高并发解决方案

## CPU多级缓存

1. 时间局部性：如果某个数据被访问，那么在不久的将来它很可能被再次访问。
2. 空间局部性：如果某个数据被访问，那么与它相邻的数据很快也可能被访问。

## 缓存一致性

缓存一致性MESI

## 安全发布对象

- 在静态初始化方法中初始化一个对象引用。
- 将对象保存到volatile类型或者AtomicReference对象中。
- 将对象引用保存到某个正确构造对象的final类型域中。
- 将对象的引用保存到一个由锁保护的域中。

## 不可变对象

- 对象创建以后其状态就不能修改。
- 对象所有的域都是final类型。
- 对象是正确创建的，在对象创建期间，this引用没有逸出。

final关键字，可用来修饰类、方法、变量。

Collections.unmodifiableXXX。

Guava的ImmutableXXX。

## 线程封闭

ThreadLocal。

## 线程安全-同步容器

ArrayList -> Vector,Stack。

HashMap -> HashTable。

Collections.synchronizedXXX(List,Set,Map)。

