WeakHashMap的键是弱引用，HashMap的键是强引用。

WeakHashMap能实现对键值对的动态回收。动态回收原理是通过WeakReference和ReferenceQueue实现的。

WeakHashMap也是一个散列表，键值都可以为null。

# 使用场景
适合需要缓存的场景。

简单缓存表的解决方案。
