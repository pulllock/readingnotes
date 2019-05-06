# HashSet
基于HashMap实现，HashSet底层采用HashMap来保存所有元素,元素实际上由 HashMap的key来保存，HashMap的value存储了一个PRESENT，它是一个静态的 Object 对象。 

Set接口的容器的元素不会有重复的元素,并且是没有顺序的。HashSet也是没有重复元素，不保证元素的顺序，允许null元素，非线程安全，迭代器是fail-fast的。

成员变量：

```
//保存set数组的是一个HashMap，键就是要保存的元素，value无作用，使用一个Object来占位。
private transient HashMap<E,Object> map;

//虚拟的value，无作用，只用来填充HashMap的value
private static final Object PRESENT = new Object();
```

构造方法：

```
//无参构造，直接新建一个HashMap实例
public HashSet() {
    map = new HashMap<>();
}
//使用指定的集合初始化
public HashSet(Collection<? extends E> c) {
    map = new HashMap<>(Math.max((int) (c.size()/.75f) + 1, 16));
    addAll(c);
}
//指定初始容量和负载因子的构造方法
public HashSet(int initialCapacity, float loadFactor) {
    map = new HashMap<>(initialCapacity, loadFactor);
}
//指定初始容量的构造方法
public HashSet(int initialCapacity) {
    map = new HashMap<>(initialCapacity);
}
//指定初始容量，负载因子，这里第三个参数无用
//需要注意这里是使用LinkedHashMap来作为底层实现，元素会有顺序
HashSet(int initialCapacity, float loadFactor, boolean dummy) {
    map = new LinkedHashMap<>(initialCapacity, loadFactor);
}
```

迭代器：

```
//直接返回HashMap的keySet的迭代器，无顺序
public Iterator<E> iterator() {
    return map.keySet().iterator();
}
```

size：

```
//直接返回map的大小
public int size() {
    return map.size();
}
```

isEmpty：

```
//直接返回map是否为空
public boolean isEmpty() {
    return map.isEmpty();
}
```

contains：

```
//使用map的containsKey来查询
public boolean contains(Object o) {
    return map.containsKey(o);
}
```

add：

```
//将元素作为key，虚拟的value作为值放入HashMap中去
public boolean add(E e) {
    return map.put(e, PRESENT)==null;
}
```

remove：

```
//使用map的remove方法移除指定的元素，返回的旧值是虚拟的value
public boolean remove(Object o) {
    return map.remove(o)==PRESENT;
}
```

clear：

```
//调用map的clear方法
public void clear() {
    map.clear();
}
```