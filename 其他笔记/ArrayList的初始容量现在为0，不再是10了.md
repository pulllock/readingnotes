# 前言

一直记得ArrayList的初始容量大小是10，今天再次看ArrayList的源码（版本：Jdk 7u80）时发现在构造函数的注释上写着初始化容量是10，但是构造函数中却没有指定初始容量，仅仅初始化了一个空的数组。应该是不知道在哪个版本中已经修改了，我却还记着之前从别人口里得来的一句话：初始容量是10。实际上初始容量已经是0了，写出来分享下，有错的地方烦请指出来，说的不一定对。

# 测试

写了下代码来测试下，ArrayList中没有直接获取capacity的方法，只能通过反射获取elementData数组的size来间接获取到capacity。代码如下：

```java
public class ArrayListCapacityTest {

    public static void main(String[] args) {
        ArrayList arrayList = new ArrayList();
        System.out.println("capacity: " + getCapacity(arrayList) + " size: " + arrayList.size());

        arrayList.add("test");
        System.out.println("capacity: " + getCapacity(arrayList) + " size: " + arrayList.size());

        arrayList = new ArrayList(11);
        System.out.println("capacity: " + getCapacity(arrayList) + " size: " + arrayList.size());
        }

    public static int getCapacity(ArrayList arrayList) {
        try {
            Field elementDataField = ArrayList.class.getDeclaredField("elementData");
            elementDataField.setAccessible(true);
            return ((Object[]) elementDataField.get(arrayList)).length;
        } catch (NoSuchFieldException | IllegalAccessException e) {
            e.printStackTrace();
            return -1;
        }
    }
}
```

结果如下：

```
capacity: 0 size: 0
capacity: 10 size: 1
capacity: 11 size: 0
```

# 分析

上面结果也可以看出来，确实是初始容量为0了。接着看下ArrayList的源码（下面所有源码版本为Jdk 7u80）：

```java
 /**
  * Constructs an empty list with an initial capacity of ten.
 */
public ArrayList() {
    super();
    this.elementData = EMPTY_ELEMENTDATA;
}
```

源码中这注释确实很误导人，构造函数中没有初始化大小。但是现在这样有个问题，数组大小为0， 我怎么添加元素进去？应该就是在add的时候初始化，继续跟进add方法的源码：

```java
public boolean add(E e) {
    // 如果刚初始化ArrayList，size肯定是0
    ensureCapacityInternal(size + 1);  // Increments modCount!!
    elementData[size++] = e;
    return true;
}
```

add方法中第一步先确保容量够用，这里面有可能就是初始化容量的方法，继续跟进ensureCapacityInternal的源码：

```java
private void ensureCapacityInternal(int minCapacity) {
	// 由上一步知道minCapacity为1
    // 这里if的条件也一定为true
    if (elementData == EMPTY_ELEMENTDATA) {
        minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
    }
	// 经过上一步之后，minCapacity就等于DEFAULT_CAPACITY，即10。
    ensureExplicitCapacity(minCapacity);
}
```

继续跟进ensureExplicitCapacity源码：

```java
private void ensureExplicitCapacity(int minCapacity) {
    modCount++;

    // overflow-conscious code
    // minCapacity此时为10，if条件成立
    if (minCapacity - elementData.length > 0)
        grow(minCapacity);
}
```

继续跟进grow源码：

```java
private void grow(int minCapacity) {
    // overflow-conscious code
    // oldCapacity = 0
    int oldCapacity = elementData.length;
    // newCapacity = 0
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    if (newCapacity - minCapacity < 0)
        // newCapacity由上层传来为10
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
    // minCapacity is usually close to size, so this is a win:
    // 这里就是数组初始化为10的地方了
    elementData = Arrays.copyOf(elementData, newCapacity);
}
```

源码跟到这里就算完了，确实是在add的时候初始化容量为10。

# 结论

ArrayList的初始化容量已经变了，不再是以前的10了，而是初始化为0，等到第一次add的时候再初始化为10。

做这样的改动，就是延迟初始化ArrayList的实际容量，应该是考虑到空间的问题，如果一开始就初始化为10，这个大小为10的数组中就全部是存的null，如果数量多了，这个也是很大的空间。应该是这样的原因吧。