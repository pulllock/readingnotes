# LinkedHashMap
HashMap是无序的，HashMap在put的时候是根据key的hash值决定存放的位置。JDK1.4 以后提供的LinkedHashMap可以实现有序的HashMap。LinkedHashMap是HashMap的一个子类，它保留插入的顺序。

LinkedHashMap，元素有顺序，并允许null键和null值。

LinkedHashMap 维护着一个双向链表。

LinkedHashMap是非线程安全的。

根据链表中元素的顺序可以分为：按插入顺序的链表，和按访问顺序(调用 get 方法)的链表。默认是按插入顺序排序，如果指定按访问顺序排序，那么调用get方法后，会将这次访问的元素移至链表尾部，不断访问可以形成按访问顺序排序的链表。

LinkedHashMap继承自HashMap，实现了Map接口。

成员变量：

```
//双向链表的表头
private transient Entry<K,V> header;
//true为访问顺序，false为插入顺序
private final boolean accessOrder;
```

构造方法：

```
//指定初始容量和负载因子
public LinkedHashMap(int initialCapacity, float loadFactor) {
	//HashMap
    super(initialCapacity, loadFactor);
    //以插入时候的顺序排序
    accessOrder = false;
}
//指定初始化容量
public LinkedHashMap(int initialCapacity) {
    super(initialCapacity);
    accessOrder = false;
}

//使用默认的值进行初始化
public LinkedHashMap() {
    super();
    accessOrder = false;
}
//使用指定的Map初始化
public LinkedHashMap(Map<? extends K, ? extends V> m) {
    super(m);
    accessOrder = false;
}
//指定初始容量，负载因子和排序规则
public LinkedHashMap(int initialCapacity,
                     float loadFactor,
                     boolean accessOrder) {
    super(initialCapacity, loadFactor);
    this.accessOrder = accessOrder;
}
```
# LinkedHashMap访问排序的原理

```
//true为访问顺序，false为插入顺序
private final boolean accessOrder;
```
当指定了accessOrder为true的时候，是按照访问的顺序进行排序。也就是每次访问LinkedHashMap的时候，移除当前的key，然后将这个key放到最后。最后一次访问的元素总是在链表的末尾。

基于HashMap来实现，可以保证移除元素比较快，查找需要移除的元素时候，不需要遍历整个LinkedHashMap，而LinkedList查找元素需要遍历。
