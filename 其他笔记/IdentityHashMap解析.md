IdentityHashMap使用`==`判断相等，而HashMap使用equals方法判断相等。

在比较key或者value时使用引用相等代替对象相等。在IdentityHashMap中k1和k2相等的条件是`k1==k2`，而HashMap中判断k1和k2相等的条件是`k1.equals(k2)`

IdentityHashMap底层是一个数组，逻辑上可以看做是一个环形数组。

存放元素发现冲突，就会往后查找，找到空位置，进行存放。

使用场景：一键多值记录，序列化，深度复制，记录对象代理。

测试：

```
package me.cxis.test.map.test;

import java.util.HashMap;
import java.util.IdentityHashMap;
import java.util.Map;

/**
 * Created by cheng.xi on 2017-05-25 17:18.
 */
public class IdentityMapTest {

    public static void main(String[] args) {
        String key1 = new String ("key");
        String key2 = new String ( "key");
        //String key2 = key1;

        System.out.println("key1 == key2 : " + (key1 == key2));
        System.out.println("key1.equals(key2) : " + (key1.equals(key2)));

        Map<String,Object> hashMap = new HashMap<>();
        hashMap.put(key1,"xxx");
        hashMap.put(key2,"yyy");
        System.out.println(hashMap.size() + " : " + hashMap);

        Map<String,Object> identityMap = new IdentityHashMap<>();
        identityMap.put(key1,"xxx");
        identityMap.put(key2,"yyy");
        System.out.println(identityMap.size() + " : " + identityMap);
    }
}

```
结果：

```
key1 == key2 : false
key1.equals(key2) : true
1 : {key=yyy}
2 : {key=yyy, key=xxx}
```

测试2：

```
package me.cxis.test.map.test;

import java.util.HashMap;
import java.util.IdentityHashMap;
import java.util.Map;

/**
 * Created by cheng.xi on 2017-05-25 17:18.
 */
public class IdentityMapTest {

    public static void main(String[] args) {
        String key1 = new String ("key");
        //String key2 = new String ( "key");
        String key2 = key1;

        System.out.println("key1 == key2 : " + (key1 == key2));
        System.out.println("key1.equals(key2) : " + (key1.equals(key2)));

        Map<String,Object> hashMap = new HashMap<>();
        hashMap.put(key1,"xxx");
        hashMap.put(key2,"yyy");
        System.out.println(hashMap.size() + " : " + hashMap);

        Map<String,Object> identityMap = new IdentityHashMap<>();
        identityMap.put(key1,"xxx");
        identityMap.put(key2,"yyy");
        System.out.println(identityMap.size() + " : " + identityMap);
    }
}

```

结果2：

```
key1 == key2 : true
key1.equals(key2) : true
1 : {key=yyy}
1 : {key=yyy}
```

IdentityHashMap只有在key是同一个引用的时候，才表示相等，覆盖数据。
