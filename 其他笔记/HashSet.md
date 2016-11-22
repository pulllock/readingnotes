# HashSet
基于 HashMap 实现的，HashSet 底层采用 HashMap 来保存所有元素。

封装了一个 HashMap 对象来存储所有的集合元素，所有放入 HashSet 中的集合元素实际上由 HashMap 的 key 来保存，而 HashMap 的 value 则存储了一个 PRESENT，它是一个静态的 Object 对象。 

Set接口的容器的元素是没有顺序的, 而且不会存有多个重复的元素.