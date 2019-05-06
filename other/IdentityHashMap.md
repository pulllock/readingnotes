# IdentityHashMap
在Java中，有一种key值可以重复的map，就是IdentityHashMap。在IdentityHashMap中，判断两个键值k1和 k2相等的条件是 k1 == k2 。在正常的Map 实现（如 HashMap）中，当且仅当满足下列条件时才认为两个键 k1 和 k2 相等：(k1==null ? k2==null : e1.equals(e2))。

IdentityHashMap只有在key完全相等（同一个引用），才会覆盖，而HashMap则不会。

IdentityHashMap的数据很简单，底层实际就是一个Object数组，在逻辑上需要看成是一个环形的数组，解决冲突的办法是：根据计算得到散列位置，如果发现该位置上已经有元素，则往后查找，直到找到空位置，进行存放，如果没有，直接进行存放。当元素个数达到一定阈值时，Object数组会自动进行扩容处理。

IdentityHashMap只有当key为同一个引用时才认为是相同的，而HashMap还包括equals相等，即内容相同。