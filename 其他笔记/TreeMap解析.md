TreeMap是支持排序的map，基于红黑树，无容量限制，TreeMap非线程安全。

TreeMap继承AbstractMap，实现NavigableMap、Cloneable、Serializable三个接口。其中AbstractMap表明TreeMap为一个Map即支持key-value的集合， NavigableMap则意味着它支持一系列的导航方法，具备针对给定搜索目标返回最接近匹配项的导航方法 。

根据键值的自然顺序进行排序，也可以根据提供的Comaprator进行排序。

迭代器是fail-fast的。

# TreeMap put()方法


# 源码分析
>jdk1.7.0_71

```
//用于排序的comparator
private final Comparator<? super K> comparator;
//根节点
private transient Entry<K,V> root = null;
//TreeMap的元素数量
private transient int size = 0;
//结构修改次数
private transient int modCount = 0;
```

## 空构造

```
public TreeMap() {
        comparator = null;
    }
```

## 使用给定的comparator初始化空的TreeMap

```
public TreeMap(Comparator<? super K> comparator) {
        this.comparator = comparator;
    }
```

## 使用给定的map初始化TreeMap
```
public TreeMap(Map<? extends K, ? extends V> m) {
        comparator = null;
        putAll(m);
    }
```

## 使用SortedMap初始化
```
public TreeMap(SortedMap<K, ? extends V> m) {}
```


# 参考
