# ThreadLocal简介
Java中的ThreadLocal类给每个线程分配一个只属于该线程的变量副本，可以用来实现线程间的数据隔离，当前线程的变量不能被其他线程访问。

# ThreadLocal使用
## 创建ThreadLocal变量

```
private ThreadLocal myThreadLocal = new ThreadLocal();
```

## 访问ThreadLocal变量
设置需要保存的值：
`myThreadLocal.set("ThreadLocal value");`

读取保存在ThreadLocal变量中的值：
`String threadLocalVlaue = (String) myThreadLocal.get();`

## ThreadLocal范型
`private ThreadLocal myThreadLocal = new ThreadLocal<String>()`

## 初始化ThreadLocal的值

```
private ThreadLocal myThreadLocal = new ThreadLocal<String>(){
	protected String initialVlaue(){
		return "initial value";
	}
};

```

# 源码分析
源码版本：
> jdk7u80

最常用的方法就是get和set方法，所以先从这两个方法入手，分析下使用。

## set(T value)
将当前的线程局部变量的副本的值设置为指定的值。子类一般不需要重写该方法，只需要使用initialValue方法去设置初始值。

```
public void set(T value) {
	//获取当前线程
    Thread t = Thread.currentThread();
    //从当前线程中得到当前线程的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null)
    	//不为空的话，调用ThreadLocalMap的set方法设置值
        map.set(this, value);
    else
    	//ThreadLocalMap为null，还没有被初始化，创建新的map
        createMap(t, value);
}
```
### getMap(Thread t)
获取指定的线程t的ThreadLocalMap
```
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

### createMap(t, value)
为当前线程t初始化一个ThreadLocalMap用来存储值，初始值是value。

在Thread类中`ThreadLocal.ThreadLocalMap threadLocals = null;`是用来存储当前线程对应的ThreadLocalMap，属于线程私有的。所以createMap方法使用`t.threadLocals = new ThreadLocalMap(this, firstValue);`来设置。

```
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

ThreadLocal用来把变量的副本存储到线程中，变量的副本就只能是当前线程私有，而在线程中是通过ThreadLocalMap来存储副本的，所以有必要了解下ThreadLocalMap是怎么实现的。

## ThreadLocalMap
ThreadLocalMap是一个自定义的HashMap，用来存储线程本地变量的值，类似与Map。ThreadLocalMap内部是使用Entry对象来存储。

### Entry
Entry继承了WeakReference，使用ThreadLocal作为key，value为ThreadLocal的值。

```
static class Entry extends WeakReference<ThreadLocal> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal k, Object v) {
        super(k);
        value = v;
    }
}
```

`private static final int INITIAL_CAPACITY = 16;`ThreadLocalMap的初始容量为16。

`private Entry[] table;`存放线程本地变量的数组。

`private int size = 0;` 线程本地变量的数目。

`private int threshold` 扩容的阈值。

扩容的阈值为指定长度的三分之二

```
private void setThreshold(int len) {
    threshold = len * 2 / 3;
}
```

//构造方法，当我们第一次使用的时候会构造一个新的ThreadLocalMap

```
ThreadLocalMap(ThreadLocal firstKey, Object firstValue) {
	//存放线程本地变量的数组，初始容量16
    table = new Entry[INITIAL_CAPACITY];
    //得到存放Entry的数组下标
    int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);
    //在得出的位置处新建一个Entry对象
    table[i] = new Entry(firstKey, firstValue);
    //大小设为1
    size = 1;
    //设置阈值 16*2/3
    setThreshold(INITIAL_CAPACITY);
}
```

使用给定的父map来构造一个ThreadLocalMap

```
private ThreadLocalMap(ThreadLocalMap parentMap) {
	//父map中存放的线程本地变量数据
    Entry[] parentTable = parentMap.table;
    //父map的长度
    int len = parentTable.length;
    //设置阈值
    setThreshold(len);
    //新建长度为len的Entry数组
    table = new Entry[len];
	//循环把父map的数组的元素放到新数组中去，中间需要重新计算数组下标。
    for (int j = 0; j < len; j++) {
        Entry e = parentTable[j];
        if (e != null) {
            ThreadLocal key = e.get();
            if (key != null) {
                Object value = key.childValue(e.value);
                Entry c = new Entry(key, value);
                int h = key.threadLocalHashCode & (len - 1);
                while (table[h] != null)
                    h = nextIndex(h, len);
                table[h] = c;
                size++;
            }
        }
    }
}
```
根据key获取Entry

```
private Entry getEntry(ThreadLocal key) {
	//计算数组下标
    int i = key.threadLocalHashCode & (table.length - 1);
    //获取元素
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
    	//没有找到key的时候的处理
        return getEntryAfterMiss(key, i, e);
}
```

//当没有找到对应的key时候

```
//key ThreadLocal对象
//i 计算出来的数组下标
//e 在i处的entry
private Entry getEntryAfterMiss(ThreadLocal key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
    	//获取key
        ThreadLocal k = e.get();
        //找到key，返回e
        if (k == key)
            return e;
        //key为null，找不到，清除掉
        if (k == null)
            expungeStaleEntry(i);
        else
        	//key不为null，计算下一个数组下标
            i = nextIndex(i, len);
        //返回下一个entry
        e = tab[i];
    }
    return null;
}
```

//存放指定的key和value

```
private void set(ThreadLocal key, Object value) {

	//当前存放的数组
    Entry[] tab = table;
    //数组长度
    int len = tab.length;
    //根据key获取存放的数组下标
    int i = key.threadLocalHashCode & (len-1);
	
	//从第i个元素开始挨个遍历
    for (Entry e = tab[i];
         e != null;
         e = tab[i = nextIndex(i, len)]) {
         //获取到i处的key
        ThreadLocal k = e.get();
	//i处的key和要存放的key相等，将原来的值替换成新的值，返回。
        if (k == key) {
            e.value = value;
            return;
        }
	//i处key为null
        if (k == null) {
        	//替换原来的Entry
            replaceStaleEntry(key, value, i);
            return;
        }
    }
	//不存在key，新建一个Entry
    tab[i] = new Entry(key, value);
    //size加1
    int sz = ++size;
    //不能移除一些旧的entry并且新的size已经大于等于阈值了，需要重新扩容
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

//rehash()

```
private void rehash() {
	//首先清除旧的entry
    expungeStaleEntries();

    // size大于等于阈值的四分之三，将容量扩展为两倍
    if (size >= threshold - threshold / 4)
        resize();
}
```
ThreadLocalMap内部的Entry的get和set基本就这些，接下来继续看ThreadLocal的get方法

## public T get()

```
public T get() {
	//获取当前线程
    Thread t = Thread.currentThread();
    //获取当前线程的ThreadLocalMap
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null)
            return (T)e.value;
    }
    //map为空，设置初始值，并返回
    return setInitialValue();
}
```

## private T setInitialValue()

```
private T setInitialValue() {
	//这里初始值为null
    T value = initialValue();
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
    return value;
}
```

## remove()
移除当前线程的ThreadLocalMap中的值

```
public void remove() {
     ThreadLocalMap m = getMap(Thread.currentThread());
     if (m != null)
         m.remove(this);
 }
```

# 总结一下get和set方法
get方法：

首先获取到当前线程，然后获取当前线程内部的ThreadLocalMap，如果map不为空，就查找到Entry中的值，返回；如果map为空，设置初始值，并返回。

set方法：

首先获取到当前线程，然后获取当前线程内部的ThreadLocalMap，如果map不为空，直接使用Entry的set设置值，此方法会替换原来的值；如果map为空，说明没有使用过，新建一个map并使用当前线程和指定的值初始化。
