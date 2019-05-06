在ConcurrentHashMap中基础的结构是数组，数组类型是Segment类型，Segment是ConcurrentHashMap的内部类，继承了ReentrantLock，也就是说Segment是一个锁。ConcurrentHashMap中默认是大小为16的Segment数组，而每个数组都是一个锁，所以理想情况下可以有同时16个线程同时操作ConcurrentHashMap。

当操作ConcurrentHashMap的时候，会首先定位到Segment，然后操作就交给了某一个Segment，Segment在ConcurrentHashMap有着很重要的地位，同时他又是ReentrantLock的子类，所以这里单独拿出来分析一下。

## 成员变量

```
//扫描尝试获取锁的时候，最大的重试次数：
static final int MAX_SCAN_RETRIES = Runtime.getRuntime().availableProcessors() > 1 ? 64 : 1;

//每个Segment中的保存数据的表
//每个Segment其实就相当于一个HashMap
//这个table是volatile类型的
transient volatile HashEntry<K,V>[] table;

//每个Segment中元素的数量
transient int count;

//每个Segment中的修改数量
transient int modCount;

//Segment中的表重新扩容的阈值
transient int threshold;

//Segment中表的负载因子
final float loadFactor;
```

## put方法

```
//onlyIfAbsent参数
//设为true表示如果key存在不更新旧值
//设为false表示如果key存在则更新旧值
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
	//操作之前先获取锁，tryLock
    //能获取到锁的情况下，就将node设为null
    //获取不到锁，调用scanAndLockForPut方法进行锁的获取
    HashEntry<K,V> node = tryLock() ? null :
        scanAndLockForPut(key, hash, value);
    //旧值
    V oldValue;
    try {
    	//当前Segment中的数据表
        //是一个HashEntry数组
        HashEntry<K,V>[] tab = table;
        //此处hash为key的hash值
        //找到了key所在的数组的位置
        int index = (tab.length - 1) & hash;
        //得到key对应的HashEntry
        HashEntry<K,V> first = entryAt(tab, index);
        //遍历根据key找到的HashEntry数组
        for (HashEntry<K,V> e = first;;) {
            //HashEntry不为null
            if (e != null) {
                K k;
                //当前HashEntry的key和我们要找的key相等
                //或者当前HashEntry的hash值和key的hash值相等，并且当前HashEntry的key和我们要找的key相同
                //表示找对了HashEntry
                if ((k = e.key) == key ||
                    (e.hash == hash && key.equals(k))) {
                    //旧值
                    oldValue = e.value;
                    //如果onlyIfAbsent为false，就替换旧值，修改数量加1
                    //如果onlyIfAbsent为true，不替换旧值
                    if (!onlyIfAbsent) {
                        e.value = value;
                        ++modCount;
                    }
                    //找到之后，中断循环的执行
                    break;
                }
                //没找到，就找下一个HashEntry
                e = e.next;
            }
            else {//如果HashEntry为null
                if (node != null)
                    node.setNext(first);
                else
                    node = new HashEntry<K,V>(hash, key, value, first);
                int c = count + 1;
                if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                    rehash(node);
                else
                    setEntryAt(tab, index, node);
                ++modCount;
                count = c;
                oldValue = null;
                break;
            }
        }
    } finally {
        unlock();
    }
    return oldValue;
}
```

## scanAndLockForPut方法

```
private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
	//根据hash找到HashEntry
    HashEntry<K,V> first = entryForHash(this, hash);
    HashEntry<K,V> e = first;
    HashEntry<K,V> node = null;
    int retries = -1; // negative while locating node
    while (!tryLock()) {循环尝试获取锁
        HashEntry<K,V> f; // to recheck first below
        if (retries < 0) {//第一次进入
            if (e == null) {//HashEntry为null
                if (node == null) //node也为null，就新建HashEntry
                    node = new HashEntry<K,V>(hash, key, value, null);
                //重试次数变为0
                retries = 0;
            }
            else if (key.equals(e.key))//如果key和HashEntry的key相等，重试次数设为0
                retries = 0;
            else//当前HashEntry不是，就找下一个
                e = e.next;
        }
        //这里将重试次数加1
        //之后判断重试次数是否超过了最大重试次数
        else if (++retries > MAX_SCAN_RETRIES) {
            //超过最大次数，直接获取锁，非公平
            lock();
            //跳出循环
            break;
        }
        //第一次进入的时候经过上面之后这里retries为1
        else if ((retries & 1) == 0 &&
                 (f = entryForHash(this, hash)) != first) {
            e = first = f; // re-traverse if entry changed
            retries = -1;
        }
    }
    return node;
}
```

