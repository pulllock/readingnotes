# 第2章 Java内存区域与内存溢出异常

## 2.2 运行时数据区域

运行时数据区域：

- 方法区
- 堆
- 虚拟机栈
- 本地方法栈
- 程序计数器

### 2.2.1 程序计数器

- 程序计数器指向下一条要执行的字节码指令的偏移地址。
- 对应CPU内部的PC寄存器，存放下一条将被执行的指令地址。
- 程序计数器是Java虚拟机在内存中开辟一块空间来模拟物理CPU的PC寄存器。

### 2.2.2 Java虚拟机栈

栈帧：
- 局部变量表：存放方法参数和方法内部定义的局部变量。
	- 容量：在编译期确定：写入了 `.class`文件的 `Code`属性中。
	- 底层单位：基本存储单元是变量槽（Slot），一个Slot长度为32位，可存放基本数据类型、对象引用、 `returnAddress`类型。64位的 `long`或 `double`需要占用两个连续的Slot。
- 操作数栈：JVM执行引擎的临时工作台。所有计算（加减乘除等）和参数传递，都需要先将数据压入操作数栈，由CPU或解释器取出计算后，再将结果压回。
- 动态链接：每个栈帧都包含一个指向运行时常量池中该栈帧所属方法的引用。Java 具备多态性（方法重写），许多方法在编译期是无法确定具体调用哪个类的。在程序运行期间，将符号引用转化为直接引用（找到内存中真正的类方法地址）的过程，就依赖动态连接。
- 方法出口：当一个方法执行完毕后，程序必须知道下一步该回到哪里继续执行。方法出口里就记录了主调方法中该方法调用指令的下一条字节码指令的地址。

### 2.2.3 本地方法栈

### 2.2.4 Java堆

存放对象实例。

### 2.2.5 方法区

方法区存储：

- 类型信息：类的全限定名、父类全限定名、实现的接口列表、访问修饰符（public/private）等。
- 域信息：类中声明的成员变量的名称、类型、修饰符。
- 方法信息：方法名、返回类型、参数列表、方法字节码、异常表。
- 常量（运行时常量池）：包含字面量和符号引用。
- 静态变量
- 即时编译器编译后的代码缓存

### 2.2.6 运行时常量池

### 2.2.7 直接内存

## 2.3 HotSpot虚拟机对象探秘

### 2.3.1 对象的创建

```java
public class ObjectHeaderInspector {

    public static void main(String[] args) {
        // 创建一个普通业务对象
        Order order = new Order();

        System.out.println("1. 刚new出来，未计算HashCode时的内存布局");
        System.out.println(ClassLayout.parseInstance(order).toPrintable());

        // 强行触发Identity HashCode计算
        int hashCode = order.hashCode();
        System.out.println("Calculated Identity HashCode (Hex): " + Integer.toHexString(hashCode));

        System.out.println("\n 2. 计算HashCode之后，Mark Word被强行写入后的布局");
        System.out.println(ClassLayout.parseInstance(order).toPrintable());
    }

}

class Order {
    /**
     * 4字节
     */
    private int orderId = 1001;
}
```

```
1. 刚new出来，未计算HashCode时的内存布局
fun.pullock.utj.c2_3_1.object_header.Order object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000011 (non-biasable; age: 2)
  8   4        (object header: class)    0x01042a10
 12   4    int Order.orderId             1001
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

Calculated Identity HashCode (Hex): 725bef66

2. 计算HashCode之后，Mark Word被强行写入后的布局
fun.pullock.utj.c2_3_1.object_header.Order object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x00000392df7b3011 (hash: 0x92df7b30; age: 2)
  8   4        (object header: class)    0x01042a10
 12   4    int Order.orderId             1001
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total
```

刚 `new`对象时输出的值：

- 8字节的Mark Word：是十六进制 `0x0000000000000011`，十六进制值的最后两位 `11`转换为二进制是 `00010001`，二进制表示对应的含义：
	- 第一位是 `0`，该位是闲置位
	- 第二到五位是 `0010`，表示GC分代年龄，这里的年龄是2
	- 第6位是 `0`，表示的是无偏向锁
	- 最后两位是 `01`，表示无锁状态
- 4字节的类型指针： 是十六进制 `0x01042a10`
	- 这里是4字节（32位）的原因是默认开启了指针压缩
- 4字节的业务数据：代码中自己定义的 `int`类型的 `orderId`

计算HashCode之后的对象：

- 8字节的Mark Word：是十六进制 `0x00000392df7b3011`，其中的 `0x92df7b30`是HashCode，另外最后两位 `11`转换为二进制是 `00010001`，二进制表示对应的含义：
	- 第一位是 `0`，该位是闲置位
	- 第二到五位是 `0010`，表示GC分代年龄，这里的年龄是2
	- 第6位是 `0`，表示的是无偏向锁
	- 最后两位是 `01`，表示无锁状态
- 4字节的类型指针： 是十六进制 `0x01042a10`
	- 这里是4字节（32位）的原因是默认开启了指针压缩
- 4字节的业务数据：代码中自己定义的 `int`类型的 `orderId`


64位Mark Word：

| 锁状态    | 25 bit        | 31 bit               | 1 bit | 4 bit  | 1 bit (偏向锁) | 2 bit (锁标志位) |
| ------ | ------------- | -------------------- | ----- | ------ | ----------- | ------------ |
| 无锁状态   | 闲置            | 对象的Identity HashCode | 闲置    | GC分代年龄 | 0           | 01           |
| 偏向锁    | 拥有锁的线程ID      | Epoch                | 闲置    | GC分代年龄 | 1           | 01           |
| 轻量级锁   | 指向栈中锁记录的指针    | 闲置                   | 闲置    |        |             | 00           |
| 重量级锁   | 指向互斥量的指针      | 闲置                   | 闲置    |        |             | 10           |
| GC标记状态 | 空 (留给GC收集器使用) | 闲置                   | 闲置    |        |             | 11           |
### 2.3.2 对象的内存布局

内存布局：

- 对象头
- 实例数据
- 对齐填充

32位Mark Word：

| 锁状态     | 25 bit                       | 4 bit  | 1 bit (偏向锁) | 2 bit (锁标志位) |
| ------- | ---------------------------- | ------ | ----------- | ------------ |
| 无锁状态    | 对象的Identity HashCode         | GC分代年龄 | 0           | 01           |
| 偏向锁     | 线程ID (23bit) \| Epoch (2bit) | GC分代年龄 | 1           | 01           |
| 轻量级锁    | 指向栈中锁记录的指针 (30bit)           |        |             | 00           |
| 重量级锁    | 指向堆中互斥量的指针 (30bit)           |        |             | 10           |
| GC 标记状态 | 空 / 留给 GC 回收器记录使用 (30bit)    |        |             | 11           |

64位Mark Word：

| 存储内容 / 状态 | 25 bit                | 31 bit            | 1 bit | 4 bit | 1 bit | 2 bit |
| --------- | --------------------- | ----------------- | ----- | ----- | ----- | ----- |
| 无锁状态      | 闲置                    | Identity HashCode | 闲置    | GC年龄  | 0     | 01    |
| 偏向锁       | 线程ID (54bit)          | Epoch (2bit)      | 闲置    | GC年龄  | 1     | 01    |
| 轻量级锁      | 指向栈中锁记录的物理指针 (62bit)  |                   |       |       |       | 00    |
| 重量级锁      | 指向堆中重量级锁的物理指针 (62bit) |                   |       |       |       | 10    |
| GC 标记     | 保留为空 (62bit)          |                   |       |       |       | 11    |

```java
public class ArrayHeaderInspector {

    public static void main(String[] args) {
        // 1. 普通对象
        NormalObject normal = new NormalObject();
        // 2. 数组对象（含有5个元素的int数组）
        int[] array = new int[5];

        System.out.println("A. 普通对象的内存布局");
        System.out.println(ClassLayout.parseInstance(normal).toPrintable());

        System.out.println("B. 数组对象的内存布局");
        System.out.println(ClassLayout.parseInstance(array).toPrintable());
    }
}

public class NormalObject {

    private int value = 100;
}
```

```
A. 普通对象的内存布局
fun.pullock.utj.c2_3_2.array_header.NormalObject object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000019 (non-biasable; age: 3)
  8   4        (object header: class)    0x01042a10
 12   4    int NormalObject.value        100
Instance size: 16 bytes
Space losses: 0 bytes internal + 0 bytes external = 0 bytes total

B. 数组对象的内存布局
[I object internals:
OFF  SZ   TYPE DESCRIPTION               VALUE
  0   8        (object header: mark)     0x0000000000000019 (non-biasable; age: 3)
  8   4        (object header: class)    0x00175cd8
 12   4        (array length)            5
 16  20    int [I.<elements>             N/A
 36   4        (object alignment gap)    
Instance size: 40 bytes
Space losses: 0 bytes internal + 4 bytes external = 4 bytes total
```

数组：

- 8字节的Mark Word `0x0000000000000019`
- 4字节的类型指针 `0x00175cd8`
- 4字节数组长度 `5`
- 20个字节的数组元素，有5个int类型元素： `5 * 4 = 20`
- 4字节的对齐填充

### 2.3.3 对象的访问定位

两种访问方式：

- 句柄
- 直接指针

## 2.4 实战：OutOfMemoryError异常

### 2.4.1 Java堆溢出

```java
/**
 * VM Args：-Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
 */
public class HeapOOM {

    public static void main(String[] args) {
        List<OOMObject> list = new ArrayList<OOMObject>();
        while (true) {
            list.add(new OOMObject());
        }
    }

    static class OOMObject {
    }
}
```

输出结果：

```java
java.lang.OutOfMemoryError: Java heap space
Dumping heap to java_pid61003.hprof ...
Heap dump file created [31067974 bytes in 0.025 secs]
Exception in thread "main" java.lang.OutOfMemoryError: Java heap space
	at java.base/java.util.Arrays.copyOf(Arrays.java:3509)
	at java.base/java.util.Arrays.copyOf(Arrays.java:3478)
	at java.base/java.util.ArrayList.grow(ArrayList.java:238)
	at java.base/java.util.ArrayList.grow(ArrayList.java:245)
	at java.base/java.util.ArrayList.add(ArrayList.java:484)
	at java.base/java.util.ArrayList.add(ArrayList.java:497)
	at fun.pullock.utj.c2_4_1.heap_oom.HeapOOM.main(HeapOOM.java:14)

Process finished with exit code 1

```

### 2.4.2 虚拟机栈和本地方法栈溢出

### 2.4.3 方法区和运行时常量池溢出

### 2.4.4 本机直接内存溢出

# 第3章 垃圾收集器与内存分配策略

## 3.1 概述

## 3.2 对象已死？

### 3.2.1 引用计数法

### 3.2.2 可达性分析算法

固定可作为GC Roots的对象：

- 虚拟机栈中引用的对象：各个线程当前正在执行的方法中的局部变量，只要方法没执行完，变量就是活的。
- 方法区中类静态属性引用的对象：只要这个类没有被卸载，这个静态变量指向的堆中的对象就永远是活的。
- 方法区中常量引用的对象：比如字符串常量池里常驻的字面量对象。
- 本地方法栈中JNI引用的对象：本地方法，C++内部用句柄持有Java对象，JVM无法干涉底层代码。
- Java虚拟机内部的引用：系统的亲儿子，比如 `Object.class`等。
- 所有被同步锁持有的对象
- 反应Java虚拟机内部情况的JMXBean、JVMTI中注册的回调、本地代码缓存等

临时性的GC Roots对象：跨代引用的对象。

### 3.2.3 再谈引用

- 强引用
- 软引用
- 弱引用
- 虚引用

### 3.2.4 生存还是死亡？

### 3.2.5 回收方法区

- 废弃的常量
- 不再使用的类型

## 3.3 垃圾收集算法

### 3.3.1 分代收集理论

### 3.3.2 标记-清除算法

两阶段：

- 标记
- 清除

缺点：

- 效率
- 空间碎片化

### 3.3.3 标记-复制算法

- 半区复制：分成大小相等的两块
- Appel式回收：新生代分为一大块Eden和两小块Survivor，比例为8:1:1

缺点：对象存活率较高时，要进行较多的复制操作
### 3.3.4 标记-整理算法

## 3.4 HotSpot的算法细节实现

### 3.4.1 根节点枚举

### 3.4.2 安全点

### 3.4.3 安全区域

### 3.4.4 记忆集与卡表

### 3.4.5 写屏障

### 3.4.6 并发的可达性分析

并发扫描时对象消失问题的解决方案：

- 增量更新
- 原始快照
	- 产生浮动垃圾

## 3.5 经典垃圾收集器

### 3.5.1 Serial收集器

- 单线程
- 新生代
- 复制算法

### 3.5.2 ParNew收集器

- 多线程
- 新生代
- 复制算法

### 3.5.3 Parallel Scavenge收集器

- 多线程
- 新生代
- 复制算法

### 3.5.4 Serial Old收集器

- 单线程
- 老年代
- 标记-整理算法

### 3.5.5 Parallel Old收集器

- 多线程
- 老年代
- 标记-整理算法
### 3.5.6 CMS收集器

- 并发
- 老年代
- 标记-清除算法

步骤：

- 初始标记
- 并发标记
- 重新标记
- 并发清除

### 3.5.7 Garbage First收集器

内存：

- 面向整个堆内存进行回收
- 堆内存布局：基于Region，没有固定大小及固定数量的分代区域划分，将堆分为多个大小相等的独立区域（Region）
- 每个Region可根据需要作为新生代的Eden空间、Survivor空间，或者老年代空间
- Humongous区域专门存储大对象

跨Region引用：

- 每个Region都有自己的记忆集，记录下别的Region指向自己的指针，同时还记录这些指针分别在哪些卡页的范围之内
- 占用内存大

垃圾回收过程中对象引用改变时，防止对象消失的方案：

- 原始快照（SATB）
	- 会产生浮动

垃圾回收过程中新创建对象：每个Region有两个TAMS（Top at Mark Start）指针，把Region中一部分空间划分出来用于存放新对象，G1默认这部分空间内对象时存活的。

步骤：

- 初始标记：
	- 仅标记一下GC Roots能直接关联到的对象
	- 修改TAMS指针的值，让下一阶段用户线程并发运行时，能在这个区域内分配新对象
	- 该阶段需要停顿线程
	- 耗时很短
- 并发标记：
	- 耗时长
	- 可与用户程序并发执行
	- 还要重新处理STAB记录下的在并发时有引用变动的对象
- 最终标记：
	- 用户线程短暂停顿
	- 处理并发阶段结束后仍遗留下来的少量的SATB中的对象
- 筛选回收：
	- 更新Region统计数据，对各个Region的回收价值和成本进行排序
	- 根据用户期望停顿时间定制回收计划
	- 选择任意多个Region构成回收集
	- 将要回收的Region中的存活对象复制到空的Region中，再清理掉整个旧Region的全部空间
	- 移动对象，需要暂停用户线程

## 3.6 低延迟垃圾收集器

### 3.6.1 Shenandoah收集器

跨Region引用关系记录：连接矩阵。

九个阶段：

- 初始标记：
	- 标记与GC Roots直接关联的对象
	- 会发生STW
- 并发标记：
	- 并发
	- 遍历对象图，标记出全部可达对象
- 最终标记：
	- 处理剩余的SATB扫描
	- 统计回收价值最高的Region，构成回收集
	- 会有短暂停顿
- 并发清理：
	- 清理没有活对象的Region
- 并发回收：
	- 并发
	- 回收集的存活对象复制到其他未被使用的Region中
	- 通过读屏障和Brooks Pointer转发指针来解决
- 初始引用更新：
	- 引用更新：将指向旧对象的引用修改到复制后的新地址
	- 该阶段不做具体的更新操作，只建立一个线程集合点，确保并发回收阶段中进行的收集器线程都已完成它们的对象移动任务
	- 短暂停顿
- 并发引用更新：
	- 进行引用更新操作
	- 并发
- 最终引用更新：
	- 修正GC Roots中的引用
	- 停顿
- 并发清理：
	- 并发
	- 回收Region的内存空间

Brooks Pointer（转发指针）

### 3.6.2 ZGC收集器

内存布局：

- 基于Region的堆内存布局
- Region可动态创建和销毁，动态的区域容量大小：
	- 小型Region：容量2MB，放小于256KB的小对象
	- 中型Region：容量32MB，放大于等于256KB，小于4MB的对象
	- 大型Region：容量不固定，动态变化，放4MB以及以上的大对象，每个大型Region中只放一个大对象

染色指针：

- 将额外信息存储在指针上
	- 高18位没用
	- 4位标志信息：
		- finalizable：是否只能通过 `finalize()`方法访问
		- Remapped：是否进入了重分配集（被移动过）
		- Marked1：
		- Marked0：
	- 42位地址空间
- 4TB内存限制，只有42位实际可用的地址空间
- 不支持32位平台
- 不支持压缩指针

阶段：

- 并发标记
- 并发预备重分配
- 并发重分配
- 并发重映射

## 3.7 选择合适的垃圾收集器

### 3.7.1 Epsilon收集器

### 3.7.2 收集器的权衡

### 3.7.3 虚拟机及垃圾收集器日志

JDK 9开始HotSpot日志开始统一，所有日志功能都使用 `-Xlog`参数，格式如下： `-Xlog[:[selections][:[output][:[decorators][:output-options]]]]`，可以使用命令 `java -Xlog:help`来查看详细信息

### 3.7.4 垃圾收集器参数总结

## 3.8 实战：内存分配与回收策略

### 3.8.1 对象优先在Eden分配

查看当前JVM使用的垃圾收集器：

- Java 8以及以前的版本： `java -XX:+PrintCommandLineFlags -version`
- Java 9以及以后的版本： `java -Xlog:gc -version`

打印收集器日志参数：

- Java 8以及以前的版本
	- `-XX:+PrintGC`： 打印基础GC概要信息，也可使用 `-verbose:gc`代替
	- `-XX:+PrintGCDetails`： 打印详细的GC回收前后内存大小及各分代的占用情况
	- `-XX:+PrintGCDateStamps`： 打印绝对的日期时间
	- `-XX:+PrintGCTimeStamps`： 打印相对于JVM启动的秒数
	- `-XX:+PrintGCCause`： 打印触发GC的原因
	- `-Xloggc:/var/log/jvm/gc-%t.log`： 指定日志输出的文件路径，`%t`会自动生成时间戳后缀，防止重启服务时覆盖旧日志
	- `-XX:+UseGCLogFileRotation`： 开启滚动日志
	- `-XX:NumberOfGCLogFiles=5`： 日志文件只保留的个数
	- `-XX:GCLogFileSize=20M`： 每个文件最大大小
- Java 9以及以后的版本，格式： `-Xlog[:[selections][:[output][:[decorators][:output-options]]]]`
	- `-Xlog:gc`： 打印基础GC概要信息，等价于 `-XX:+PrintGC`
	- `-Xlog:gc*`： 打印详细的GC日志，等价于 `-XX:+PrintGCDetails`
	- `:time`： 显示绝对日期
	- `:uptime`： 显示启动相对时间
	- `:file=/path/to/gc.log`： 指定日志输出到文件
	- `::filecount=5`： 日志文件个数
	- `,filesize=50M`： 每个文件大小


示例代码：

```java
private static final int _1MB = 1024 * 1024;

public static void testAllocation() {
	byte[] allocation1, allocation2, allocation3, allocation4;
	allocation1 = new byte[2 * _1MB];
	allocation2 = new byte[2 * _1MB];
	allocation3 = new byte[2 * _1MB];
	// 出现一次Minor GC
	allocation4 = new byte[4 * _1MB];
}
```

使用命令行运行该示例代码，虚拟机参数：

- Java 8以及以前版本： `-Xms20M -Xmx20M -Xmn10M -XX:SurvivorRatio=8 -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+PrintGCCause`
- Java 9以及以后版本： `-Xms20M -Xmx20M -Xmn10M -XX:SurvivorRatio=8 -Xlog:gc*:stdout:time,uptime,level,tags`

参数解释：

Java 8以及以前版本：

- `-Xms20M -Xmx20M -Xmn10M -XX:SurvivorRatio=8`
	- 初始堆内存（ `-Xms20M`）和最大堆内存（ `-Xmx20M`）均设置为固定大小：20M
	- 新生代内存（ `-Xmn10M`）设置为固定大小：10M
	- 老年代大小可以推出为10M， `老年代大小 = 堆总内存大小 - 新生代大小`即 `老年代大小 = 20M - 10M`
	- `-XX:SurvivorRatio=8`将新生代中Eden区和单个的Survivor区的容量比例设置为8:1，此时：
		- Eden区大小为 `10M x (8/10) = 8M`
		- Survivor 0区大小为 `10M x (1/10) = 1M`
		- Survivor 1区大小为 `10M x (1/10) = 1M`
		- `新生代总内存 = Eden + Survivor 0 + Survivor 1 = 8M + 1M + 1M = 10M`
- `-verbose:gc`： 打印基础GC概要信息，也可使用 `-XX:+PrintGC`代替
- `-XX:+PrintGCDetails`： 打印详细的GC回收前后内存大小及各分代的占用情况
- `-XX:+PrintGCDateStamps`： 打印绝对的日期时间
- `-XX:+PrintGCTimeStamps`： 打印相对于JVM启动的秒数
- `-XX:+PrintGCCause`： 打印触发GC的原因

Java 9以及以后版本：

- `-Xms20M -Xmx20M -Xmn10M -XX:SurvivorRatio=8`（这部分内存参数的设置和以前版本是一致的）
	- 初始堆内存（ `-Xms20M`）和最大堆内存（ `-Xmx20M`）均设置为固定大小：20M，在G1收集器中共20个1MB的Region
	- 新生代内存（ `-Xmn10M`）设置为固定大小：10M
	- 老年代大小可以推出为10M， `老年代大小 = 堆总内存大小 - 新生代大小`即 `老年代大小 = 20M - 10M`
	- `-XX:SurvivorRatio=8`将新生代中Eden区和单个的Survivor区的容量比例设置为8:1，此时：
		- Eden区大小为 `10M x (8/10) = 8M`
		- Survivor 0区大小为 `10M x (1/10) = 1M`
		- Survivor 1区大小为 `10M x (1/10) = 1M`
		- `新生代总内存 = Eden + Survivor 0 + Survivor 1 = 8M + 1M + 1M = 10M`
- `-Xlog:gc*`： 打印详细的GC日志，等价于 `-XX:+PrintGCDetails`加上 `-XX:+PrintGCCause`
- `:stdout`：日志输出到标准控制台
- `:time`： 显示绝对日期
- `:uptime`： 显示启动相对时间
- `:level`：会在日志中输出日志级别
- `:tags`：会在日志中输出 `[gc,heap]` 等标签

Java 8中输出：

```
Heap
 PSYoungGen      total 9216K, used 6816K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
  eden space 8192K, 83% used [0x00000000ff600000,0x00000000ffca8010,0x00000000ffe00000)
  from space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)
  to   space 1024K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x00000000fff00000)
 ParOldGen       total 10240K, used 4096K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  object space 10240K, 40% used [0x00000000fec00000,0x00000000ff000010,0x00000000ff600000)
 Metaspace       used 2683K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 279K, capacity 386K, committed 512K, reserved 1048576K
```

JDK 8中垃圾收集器默认是： `Parallel Scavenge + Parallel Old`。

第一行日志 `heap`表示这些输出的日志是JVM堆内存快照，输出的内容是当前JVM进程在退出前或触发GC后，整个Java堆各个分区的内存使用情况。

第二行日志 `PSYoungGen      total 9216K, used 6816K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)`是新生代的内存信息：

- `PSYoungGen`：表示新生代使用的是Parallel Scavenge垃圾收集器。
- `total 9216K`：新生代总大小是9MB，新生代大小 = Eden大小 + 一个Survivor区大小=8MB + 1MB = 9MB。
- `used 6816K`: 新生代实际被占用的空间。
- `[0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)`：操作系统的虚拟内存地址指针：
	- 第一个值 `0x00000000ff600000`：该区域在操作系统进程空间中的起始内存基地址。
	- 第二个值 `0x0000000100000000`：当前已经向操作系统申请并提交的内存上限地址。
	- 第三个值 `0x0000000100000000`：该区域允许扩容到的最大保留内存地址。

第三行日志 `  eden space 8192K, 83% used [0x00000000ff600000,0x00000000ffca8010,0x00000000ffe00000)`是Eden空间内存信息：

- `eden space`：Eden空间
- `8192K`：Eden区大小为8MB
- `83% used`：Eden区当前被使用了83%。
- `[0x00000000ff600000,0x00000000ffca8010,0x00000000ffe00000)`：
	- 第一个值 `0x00000000ff600000`是Eden的起始地址。
	- 第二个值 `0x00000000ffca8010`是当前的分配指针（Top指针）。
	- 第三个值 `0x00000000ffe00000`是Eden的结束地址。

第四行日志 `from space 1024K, 0% used [0x00000000fff00000,0x00000000fff00000,0x0000000100000000)`是Survivor空间内存信息：

- `from space`：From Survivor空间
- `1024K`：大小为1MB
- `0% used`：没有被使用
- 剩下的是内存空间地址

第五行日志 `to   space 1024K, 0% used [0x00000000ffe00000,0x00000000ffe00000,0x00000000fff00000)`是Survivor空间内存信息：

- to space`：To Survivor空间
- `1024K`：大小为1MB
- `0% used`：没有被使用
- 剩下的是内存空间地址

第六行日志 `ParOldGen       total 10240K, used 4096K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)`和第七行日志 `object space 10240K, 40% used [0x00000000fec00000,0x00000000ff000010,0x00000000ff600000)`是老年代内存信息：

- `ParOldGen`：表示老年代使用的是Parallel Old收集器
- `total 10240K`：老年代总内存空间为10MB
- `used 4096K`：老年代被占用空间为4MB
- 剩下的是内存空间地址
- `object space 10240K`：对象空间总容量为10MB
- `40% used`：使用了40%的空间
- 剩下的是内存空间地址

第八行日志 `Metaspace       used 2683K, capacity 4486K, committed 4864K, reserved 1056768K`和第九行日志 `class space    used 279K, capacity 386K, committed 512K, reserved 1048576K`是元空间内存信息：

- `Metaspace`：元空间
- `used 2683K`：已使用空间，当前加载的类元数据、常量池等实际占用的空间
- `capacity 4486K`：JVM根据当前加载类的数量，在本地内存中申请的空闲块的水位线容量
- `committed 4864K`：JVM已经向操作系统成功申请并承诺分配的内存大小
- `reserved 1056768K`：JVM向操作系统预留的元空间最大大小（大约1GB）
- `class space`：元空间内部分成两部分， `class space`专门用来存放经过压缩的类指针
- `used 279K`
- `capacity 386K`
- `committed 512K`
- `reserved 1048576K`

Java 8中使用Serial收集器后的输出：

```
2026-07-21T15:16:34.011+0800: 0.029: [GC (Allocation Failure) 2026-07-21T15:16:34.011+0800: 0.029: [DefNew: 6651K->258K(9216K), 0.0025232 secs] 6651K->6402K(19456K), 0.0025615 secs] [Times: user=0.00 sys=0.01, real=0.01 secs] 
Heap
 def new generation   total 9216K, used 4680K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  53% used [0x00000000fec00000, 0x00000000ff051888, 0x00000000ff400000)
  from space 1024K,  25% used [0x00000000ff500000, 0x00000000ff540aa0, 0x00000000ff600000)
  to   space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
 tenured generation   total 10240K, used 6144K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,  60% used [0x00000000ff600000, 0x00000000ffc00030, 0x00000000ffc00200, 0x0000000100000000)
 Metaspace       used 2683K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 279K, capacity 386K, committed 512K, reserved 1048576K
```

第一行日志是在最后分配给4MB大小的数组对象时，由于Eden区的空间不足触发的Minor GC。

- `2026-07-21T15:16:34.011+0800`：绝对日期和时间，由参数 `-XX:+PrintGCDateStamps`产生。
- `0.029`：JVM启动后的相对时间，单位是秒，由参数 `-XX:+PrintGCTimeStamps`产生，表示这次GC发生在JVM启动后的第0.029秒。
- `[GC`：表示这是一次Minor GC，如果是Full GC则会显示为 `[Full GC`。
- `(Allocation Failure)`：触发GC的原因，由参数 `-XX:+PrintGCCause`产生，表示由于Eden区空间不足，程序尝试分配新对象时失败，从而触发了此次的垃圾回收。
- `DefNew`：表示新生代使用的垃圾收集器是Serial收集器，Default New Generation。
- `6651K->258K`：表示新生代在GC前后的内存占用变化，GC前新生代占用6651K，GC后是258K。
	- `6651K = 6144K + 507K`，其中6144K是先分配的三个2MB大小的数组对象，总计6MB，507K是系统中自己的对象。
	- `258K`是GC发生之后新生代仅剩的存活对象，这些对象被挪到了Survivor区中。
- `(9216K)`：当前新生代的总的可用大小，也就是Eden加上一个Survivor： `Eden（8192K) + Survivor (1024K) = 9216K`。
- `0.0025232 secs`：新生代垃圾收集消耗额时间，大约2.52毫秒。
- `6651K->6402K`：整个Java堆（新生代+老年代）在GC前后的变化，GC前堆中占用6651KB，GC之后堆占用6402K。
	- `6651K`：GC前堆的占用和新生代中的占用是一样的，此时只有新生代中有对象，老年代中是空的。
	- `6402K`：GC后堆的占用是新生代的Survivor区中的258K加上晋升到老年代的6MB对象： `6144K + 258K = 6402K`。
- `19456K`：当前整个堆的可用内存，新生代 + 老年代 = Eden + 一个Survivor + 老年代 = 8192K + 1024K + 102400K = 19456K。
- `0.0025615 secs`：整个GC（包括空间分配担保和老年代指针移动）引发的总的STW时间，大概2.56毫秒。
- `[Times: user=0.00 sys=0.01, real=0.01 secs]`：操作系统视角的CPU时间统计：
	- `user`：用户态CPU时间
	- `sys`：内核态CPU时间
	- `real`：实际经过的时间
- `heap`表示这些输出的日志是JVM堆内存快照，输出的内容是当前JVM进程在退出前或触发GC后，整个Java堆各个分区的内存使用情况。
- `def new generation`：新生代
- `total 9216K`：新生代总的可用内存
- `used 4680K`：新生代占用内存，Eden中分配的4MB对象+Eden中系统自己的对象+Survivor中的258K
- `[0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)`：新生代内存地址
- `eden space`：Eden区
- `8192K`：Eden区大小
- `53% used`：Eden使用率，Eden已使用内存/Eden总内存=(4680K-258K)/8192K=53%
- `[0x00000000fec00000, 0x00000000ff051888, 0x00000000ff400000)`：Eden区内存地址
- `from space`：From Survivor区
- `1024K`：From Survivor区大小
- `25% used`：From Survivor使用率，From Survivor已使用内存/From Survivor内存=258K/1024K=25%
- `[0x00000000ff500000, 0x00000000ff540aa0, 0x00000000ff600000)`：From Survivor区内存地址（此时由于进行过一次Minor GC，因此 From和To交换了，可以从From和To的内存地址看出来）
- `to   space`：To Survivor区
- `1024K`：To Survivor区大小
- `0% used`：To Survivor区使用率
- `[0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)`：To Survivor区内存地址
- `tenured generation`：老年代
- `total 10240K`：老年代总大小，10MB
- `used 6144K`：老年代已使用，6MB，这是分配的3个2MB大小的对象
- `[0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)`：老年代内存地址
- `the space 10240K`：老年代大小
- `60% used`：老年代使用率，老年代已使用/老年代大小=6144K/10240K=60%
- `[0x00000000ff600000, 0x00000000ffc00030, 0x00000000ffc00200, 0x0000000100000000)`：老年代内存地址
- `Metaspace`：元空间
- `used 2683K`：元空间已使用内存（保存类元数据、方法数据等）
- `capacity 4486K`：当前分配的元空间总容量
- `committed 4864K`：JVM向操作系统申请分配的总容量
- `reserved 1056768K`：JVM为元空间预留的可用内存（约1GB）
- `class space`：类指针压缩空间
- `used 279K`：已使用
- `capacity 386K`：容量
- `committed 512K`：向操作系统申请分配的容量
- `reserved 1048576K`：预留的容量（约1GB）

Java 9中输出：

```
[2026-07-21T16:14:05.305+0800][0.003s][info][gc,heap] Heap region size: 1M
[2026-07-21T16:14:05.308+0800][0.005s][info][gc     ] Using G1
[2026-07-21T16:14:05.308+0800][0.005s][info][gc,heap,coops] Heap address: 0x00000000fec00000, size: 20 MB, Compressed Oops mode: 32-bit
[2026-07-21T16:14:05.355+0800][0.053s][info][gc,start     ] GC(0) Pause Initial Mark (G1 Humongous Allocation)
[2026-07-21T16:14:05.355+0800][0.053s][info][gc,task      ] GC(0) Using 8 workers of 8 for evacuation
[2026-07-21T16:14:05.356+0800][0.054s][info][gc,phases    ] GC(0)   Pre Evacuate Collection Set: 0.0ms
[2026-07-21T16:14:05.356+0800][0.054s][info][gc,phases    ] GC(0)   Evacuate Collection Set: 1.2ms
[2026-07-21T16:14:05.356+0800][0.054s][info][gc,phases    ] GC(0)   Post Evacuate Collection Set: 0.0ms
[2026-07-21T16:14:05.356+0800][0.054s][info][gc,phases    ] GC(0)   Other: 0.1ms
[2026-07-21T16:14:05.356+0800][0.054s][info][gc,heap      ] GC(0) Eden regions: 2->0(9)
[2026-07-21T16:14:05.356+0800][0.054s][info][gc,heap      ] GC(0) Survivor regions: 0->1(2)
[2026-07-21T16:14:05.356+0800][0.054s][info][gc,heap      ] GC(0) Old regions: 0->0
[2026-07-21T16:14:05.356+0800][0.054s][info][gc,heap      ] GC(0) Humongous regions: 9->9
[2026-07-21T16:14:05.356+0800][0.054s][info][gc,metaspace ] GC(0) Metaspace: 4027K->4027K(1056768K)
[2026-07-21T16:14:05.356+0800][0.054s][info][gc           ] GC(0) Pause Initial Mark (G1 Humongous Allocation) 10M->9M(20M) 1.298ms
[2026-07-21T16:14:05.356+0800][0.054s][info][gc,cpu       ] GC(0) User=0.00s Sys=0.00s Real=0.00s
[2026-07-21T16:14:05.356+0800][0.054s][info][gc           ] GC(1) Concurrent Cycle
[2026-07-21T16:14:05.356+0800][0.054s][info][gc,marking   ] GC(1) Concurrent Clear Claimed Marks
[2026-07-21T16:14:05.356+0800][0.054s][info][gc,marking   ] GC(1) Concurrent Clear Claimed Marks 0.004ms
[2026-07-21T16:14:05.356+0800][0.054s][info][gc,marking   ] GC(1) Concurrent Scan Root Regions
[2026-07-21T16:14:05.357+0800][0.054s][info][gc,marking   ] GC(1) Concurrent Scan Root Regions 0.289ms
[2026-07-21T16:14:05.357+0800][0.054s][info][gc,marking   ] GC(1) Concurrent Mark (0.054s)
[2026-07-21T16:14:05.357+0800][0.054s][info][gc,marking   ] GC(1) Concurrent Mark From Roots
[2026-07-21T16:14:05.357+0800][0.054s][info][gc,task      ] GC(1) Using 2 workers of 2 for marking
[2026-07-21T16:14:05.357+0800][0.054s][info][gc,marking   ] GC(1) Concurrent Mark From Roots 0.025ms
[2026-07-21T16:14:05.357+0800][0.054s][info][gc,marking   ] GC(1) Concurrent Mark (0.054s, 0.054s) 0.033ms
[2026-07-21T16:14:05.358+0800][0.056s][info][gc,start     ] GC(1) Pause Remark
[2026-07-21T16:14:05.360+0800][0.058s][info][gc,stringtable] GC(1) Cleaned string and symbol table, strings: 2930 processed, 0 removed, symbols: 18973 processed, 0 removed
[2026-07-21T16:14:05.360+0800][0.058s][info][gc            ] GC(1) Pause Remark 14M->14M(20M) 2.519ms
[2026-07-21T16:14:05.360+0800][0.058s][info][gc,cpu        ] GC(1) User=0.01s Sys=0.00s Real=0.00s
[2026-07-21T16:14:05.360+0800][0.058s][info][gc,marking    ] GC(1) Concurrent Create Live Data
[2026-07-21T16:14:05.360+0800][0.058s][info][gc,marking    ] GC(1) Concurrent Create Live Data 0.029ms
[2026-07-21T16:14:05.360+0800][0.058s][info][gc,start      ] GC(1) Pause Cleanup
[2026-07-21T16:14:05.361+0800][0.058s][info][gc            ] GC(1) Pause Cleanup 14M->14M(20M) 0.097ms
[2026-07-21T16:14:05.361+0800][0.058s][info][gc,cpu        ] GC(1) User=0.00s Sys=0.00s Real=0.00s
[2026-07-21T16:14:05.361+0800][0.058s][info][gc,marking    ] GC(1) Concurrent Cleanup for Next Mark
[2026-07-21T16:14:05.361+0800][0.058s][info][gc,marking    ] GC(1) Concurrent Cleanup for Next Mark 0.128ms
[2026-07-21T16:14:05.361+0800][0.058s][info][gc            ] GC(1) Concurrent Cycle 4.355ms
[2026-07-21T16:14:05.362+0800][0.059s][info][gc,heap,exit  ] Heap
[2026-07-21T16:14:05.362+0800][0.059s][info][gc,heap,exit  ]  garbage-first heap   total 20480K, used 15050K [0x00000000fec00000, 0x00000000fed000a0, 0x0000000100000000)
[2026-07-21T16:14:05.362+0800][0.059s][info][gc,heap,exit  ]   region size 1024K, 2 young (2048K), 1 survivors (1024K)
[2026-07-21T16:14:05.362+0800][0.059s][info][gc,heap,exit  ]  Metaspace       used 4035K, capacity 4486K, committed 4864K, reserved 1056768K
[2026-07-21T16:14:05.362+0800][0.059s][info][gc,heap,exit  ]   class space    used 347K, capacity 386K, committed 512K, reserved 1048576K
```

- `[2026-07-21T16:14:05.305+0800][0.003s][info][gc,heap]`：
	- `[2026-07-21T16:14:05.305+0800]`：绝对时间
	- `[0.003s]`：相对JVM启动的时间
	- `[info]`：日志级别
	- `[gc,heap]`：日志标签分类
- `Heap region size: 1M`：G1将堆划分为大小相等的Region，根据堆的总大小20MB，这里分成了每个Region大小为1MB。
- `Using G1`：使用的垃圾收集器是G1
- `Heap address: 0x00000000fec00000, size: 20 MB, Compressed Oops mode: 32-bit`：
	- `Heap address: 0x00000000fec00000`：堆的起始地址
	- `size: 20 MB`：堆的大小
	- `Compressed Oops mode: 32-bit`：开启了普通对象指针压缩，在64位的JVM上使用32位来表示对象指针，以节省内存开销。
- `GC(0) Pause Initial Mark (G1 Humongous Allocation)`：分配4MB大小对象的时候触发了GC：
	- 由于分配的4MB大于一个Region大小的50%，故需要在老年代分配，但是此时老年代也已经被3个2MB大小的对象占据。
	- `GC(0)`: GC阶段标记
	- `Pause Initial Mark`：初始标记阶段，会STW。G1会借助一次年轻代回收顺便完成堆GC Roots的直接引用的标记。
	- `(G1 Humongous Allocation)`：GC触发的原因是因为申请分配巨型对象失败/达到阈值
- `GC(0) Using 8 workers of 8 for evacuation`：并行回收的线程数，当前机器有8个CPU，G1启用了全部8个并行工作线程来转移对象。
- `GC(0)   Pre Evacuate Collection Set: 0.0ms`：将回收集转移前的准备工作，耗时0.0ms
- `GC(0)   Evacuate Collection Set: 1.2ms`：将存活对象从源Region复制到空闲Region，耗时1.2ms
- `GC(0)   Post Evacuate Collection Set: 0.0ms`：转移后的收尾清理（处理卡表、引用处理等），耗时0.0ms
- `GC(0)   Other: 0.1ms`：其他开销，耗时0.1ms
- `GC(0) Eden regions: 2->0(9)`：Eden区的Region数量从2个清空为0个，预留最大为9个
- `GC(0) Survivor regions: 0->1(2)`：Survivor区从0个增加到1个，总共预留有2个。Eden区清理出的存活对象复制到了这个Survivor区中
- `GC(0) Old regions: 0->0`：普通非巨型老年代Region数量回收前后都是0
- `GC(0) Humongous regions: 9->9`: 巨型对象的Region在GC前后保持9个不变
- `GC(0) Metaspace: 4027K->4027K(1056768K)`：元空间在GC前后不变，保留的虚拟地址空间为1GB左右
- `GC(0) Pause Initial Mark (G1 Humongous Allocation) 10M->9M(20M) 1.298ms`：GC(0)的汇总：
	- `Pause Initial Mark`：本次GC类型
	- `(G1 Humongous Allocation)`：本次GC触发的原因
	- `10M->9M(20M)`：堆内存变化，使用量从10MB降低到9MB，总堆空间为20MB
	- `1.298ms`：本次停顿1.298毫秒
- `GC(0) User=0.00s Sys=0.00s Real=0.00s`：CPU时间统计
- `GC(1) Concurrent Cycle`：并发标记阶段开启，此阶段GC线程与应用线程并发运行，不会发生长时间STW
- `GC(1) Concurrent Clear Claimed Marks`：并发清理上一阶段留下的标记状态位
- `GC(1) Concurrent Clear Claimed Marks 0.004ms`：并发清理上一阶段留下的标记状态位耗时0.004毫秒
- `GC(1) Concurrent Scan Root Regions`：并发扫描根区域。扫描在初始标记阶段幸存并进入Survivor区的对象，寻找指向老年代的引用
- `GC(1) Concurrent Scan Root Regions 0.289ms`：并发扫描根区域耗时
- `GC(1) Concurrent Mark (0.054s)`：并发标记阶段
- `GC(1) Concurrent Mark From Roots`：从GC Roots开始遍历可达对象
- `GC(1) Using 2 workers of 2 for marking`：使用2个线程并发标记
- `GC(1) Concurrent Mark From Roots 0.025ms`：从GC Roots开始的并发标记耗时
- `GC(1) Concurrent Mark (0.054s, 0.054s) 0.033ms`：整个并发标阶段汇总
	- 第一个0.054s是并发标记阶段开始时，JVM已经运行的时间
	- 第二个0.054s是并发标记阶段结束时，JVM已经运行的时间
	- 第三个0.033ms是整个并发标记阶段总耗时
- `GC(1) Pause Remark`：重新标记，STW，解决并发标记期间因应用程序运行导致的对象引用变更，G1使用SATB（原始快照）来解决。
- `GC(1) Cleaned string and symbol table, strings: 2930 processed, 0 removed, symbols: 18973 processed, 0 removed`：清理字符串常量池与符号表
	- `Cleaned string and symbol table`：对JVM内部的字符串常量池表和符号表进行并发标记后的弱引用清理
	- `strings: 2930 processed, 0 removed`：共检查了2930个字符串常量池记录，0个被判定为无引用可回收
	- `symbols: 18973 processed, 0 removed`：共检查了18973个符号，0个被剔除
- `GC(1) Pause Remark 14M->14M(20M) 2.519ms`：重新标记阶段汇总
	- 重新标记阶段开始前整个Java堆占用14MB
	- 重新标记阶段结束后整个Java堆占用14MB
	- 当前堆可用最大容量为20MB
	- 本次STW总时间为2.519ms
- `GC(1) User=0.01s Sys=0.00s Real=0.00s`：重新标记阶段CPU耗时
- `GC(1) Concurrent Create Live Data`：并发创建存活数据，上一阶段重新标记确定了最终存活对象后，G1并发统计各个Region中存活的对象字节数
- `GC(1) Concurrent Create Live Data 0.029ms`：并发创建存活对象数据耗时
- `GC(1) Pause Cleanup`：清理，STW，回收没有存活对象的Region，直接清空
- `GC(1) Pause Cleanup 14M->14M(20M) 0.097ms`：清理前后都是14MB，堆总可用内存20MB，耗时0.097毫秒
- `GC(1) User=0.00s Sys=0.00s Real=0.00s`：CPU耗时
- `GC(1) Concurrent Cleanup for Next Mark`：并发清理/重置标记位，清空和重置位图等内部数据结构，为下一次并发标记周期做准备
- `GC(1) Concurrent Cleanup for Next Mark 0.128ms`：耗时
- `GC(1) Concurrent Cycle 4.355ms`：总周期耗时
- `Heap`：退出时的堆快照信息
- `garbage-first heap   total 20480K, used 15050K [0x00000000fec00000, 0x00000000fed000a0, 0x0000000100000000)`：G1管理的堆信息：
	- `total 20480K`：堆总量20MB
	- `used 15050K`：使用了1505K
	- `[0x00000000fec00000, 0x00000000fed000a0, 0x0000000100000000)`：内存空间地址
- `region size 1024K, 2 young (2048K), 1 survivors (1024K)`：Region信息
	- `region size 1024K`：单个Region大小为1MB
	- `2 young (2048K)`：年轻代占用2个Region，共2MB
	- `1 survivors (1024K)`：年轻代2个Region中有一个Region是Survivor区，另外一个是个Eden区
- `Metaspace       used 4035K, capacity 4486K, committed 4864K, reserved 1056768K`：元空间信息
	- `used 4035K`：已使用
	- `capacity 4486K`：容量
	- `committed 4864K`：向操作系统申请的内存
	- `reserved 1056768K`：保留的内存地址空间，大概1GB
- `class space    used 347K, capacity 386K, committed 512K, reserved 1048576K`：类空间，元空间的一部分
	- `used 347K`：已使用
	- `capacity 386K`：容量
	- `committed 512K`向操作系统申请的内存
	- `reserved 1048576K`：保留的内存地址空间，大概1GB

### 3.8.2 大对象直接进入老年代

示例代码：

```java
private static final int _1MB = 1024 * 1024;

public static void testPretenureSizeThreshold() {
	byte[] allocation;

	// 直接分配在老年代中
	allocation = new byte[4 * _1MB];
}
```

运行参数：

- Java 8以及以前，使用默认Parallel收集器： `-Xms20M -Xmx20M -Xmn10M -XX:SurvivorRatio=8 -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+PrintGCCause`
- Java 8以及以前，使用Serial收集器： `-Xms20M -Xmx20M -Xmn10M -XX:SurvivorRatio=8 -XX:PretenureSizeThreshold=3145728 -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+PrintGCCause -XX:+UseSerialGC`
	- `-XX:PretenureSizeThreshold`：该参数设置一个阈值，超过该阈值的对象作为大对象直接分配到老年代，该值以字节为单位。

Java 8以及以前，使用Serial收集器的垃圾收集日志：

```
Heap
 def new generation   total 9216K, used 672K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,   8% used [0x00000000fec00000, 0x00000000feca8100, 0x00000000ff400000)
  from space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
  to   space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
 tenured generation   total 10240K, used 4096K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,  40% used [0x00000000ff600000, 0x00000000ffa00010, 0x00000000ffa00200, 0x0000000100000000)
 Metaspace       used 2684K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 279K, capacity 386K, committed 512K, reserved 1048576K
```

日志中可以看到 `tenured generation   total 10240K, used 4096K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)`，老年代中占用4MB，上面的新生代日志中只占用672K，大对象直接分配到了老年代。


### 3.8.3 长期存活的对象将进入老年代

示例代码：

```java
private static final int _1MB = 1024 * 1024;

public static void testTenuringThreshold() {
	byte[] allocation1, allocation2, allocation3;
	allocation1 = new byte[_1MB / 4];
	allocation2 = new byte[4 * _1MB];
	allocation3 = new byte[4 * _1MB];
	allocation3 = null;
	allocation3 = new byte[4 * _1MB];
}
```

运行参数：

- Java 8以及以前，使用Serial收集器： `-Xms20M -Xmx20M -Xmn10M -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=1 -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+PrintGCCause -XX:+UseSerialGC -XX:+PrintTenuringDistribution`
	- `-XX:MaxTenuringThreshold`：晋升老年代的年龄阈值，默认为15。
	- `-XX:+PrintTenuringDistribution`：打印Survivor区期望容量和年龄阈值调整信息。


Java 8以及以前，使用Serial收集器日志：

```
2026-07-22T20:42:33.144+0800: 0.077: [GC (Allocation Failure) 2026-07-22T20:42:33.144+0800: 0.077: [DefNew
Desired survivor size 524288 bytes, new threshold 1 (max 1)
- age   1:     527112 bytes,     527112 total
: 4860K->514K(9216K), 0.0032372 secs] 4860K->4610K(19456K), 0.0032854 secs] [Times: user=0.00 sys=0.01, real=0.01 secs] 
2026-07-22T20:42:33.148+0800: 0.081: [GC (Allocation Failure) 2026-07-22T20:42:33.148+0800: 0.081: [DefNew
Desired survivor size 524288 bytes, new threshold 1 (max 1)
: 4610K->0K(9216K), 0.0007770 secs] 8706K->4610K(19456K), 0.0008161 secs] [Times: user=0.00 sys=0.00, real=0.00 secs] 
Heap
 def new generation   total 9216K, used 4421K [0x00000000fec00000, 0x00000000ff600000, 0x00000000ff600000)
  eden space 8192K,  53% used [0x00000000fec00000, 0x00000000ff051588, 0x00000000ff400000)
  from space 1024K,   0% used [0x00000000ff400000, 0x00000000ff400000, 0x00000000ff500000)
  to   space 1024K,   0% used [0x00000000ff500000, 0x00000000ff500000, 0x00000000ff600000)
 tenured generation   total 10240K, used 4610K [0x00000000ff600000, 0x0000000100000000, 0x0000000100000000)
   the space 10240K,  45% used [0x00000000ff600000, 0x00000000ffa808a0, 0x00000000ffa80a00, 0x0000000100000000)
 Metaspace       used 2685K, capacity 4486K, committed 4864K, reserved 1056768K
  class space    used 279K, capacity 386K, committed 512K, reserved 1048576K
```

日志中第一次GC的时候新生代还剩下514K对象活着，等到第二次GC的时候直接就变成了0，这些对象全部晋升到了老年代中。

### 3.8.4 动态对象年龄判定

如果在Survivor空间中相同年龄所有对象大小的总和大于Survivor空间的一半，年龄大于或等于该年龄的对象就可以直接进入老年代，无需等 `-XX:MaxTenuringThreshold`中要求的年龄。

### 3.8.5 空间分配担保

Minor GC之前，会先检查老年代最大可用连续空间是否大于新生代所有对象总空间，如果这个条件成立，那这次的Minor GC是可以确保安全的。如果不成立，则会查看 `-XX：HandlePromotionFailure`参数设置的值是否允许担保失败，如果允许，则会继续检查老年代最大可用连续空间是否大于历次晋升到老年代对象的平均大小，如果大于，将会尝试进行一次Minor GC，尽管这里Minor GC是有风险的；如果小于，或者 `-XX：HandlePromotionFailure`设置不允许冒险，这时就要进行一次Full GC。

# 第4章 虚拟机性能监控、故障处理工具

## 4.2 基础故障处理工具

### 4.2.1 jps：虚拟机进程状况工具

jps：JVM Process Status Tool。

命令格式： `jps [options] [hostid]`：

- `hostid`：RMI注册表中注册的主机名，jps可以通过RMI协议查询开启了RMI服务的远程虚拟机进程状态。
- `options`：
	- `-q`：只输出LVMID，省略竹类的名称
	- `-m`：输出虚拟机进程启动时传递给主类 `main()`函数的参数
	- `-l`：输出主类的全名。如果进程执行的是JAR包，则输出JAR路径
	- `-v`：输出虚拟机进程启动时的JVM参数

### 4.2.2 jstat：虚拟机统计信息监视工具

jstat：JVM Statistics Monitoring Tool，监视虚拟机各种运行状态信息。

命令格式： `jstat [option vmid [interval [s|ms] [count]]]`：

- `vmid`：虚拟机进程ID
- `interval`：查询间隔
- `count`：查询次数
- `option`：主要分三类：类加载、垃圾收集、运行期编译状况
	- `-class`：监视类加载、卸载数量、总空间以及类装载所耗费的时间
	- `-gc`：监视Java堆状况，包括Eden区、两个Survivor区、老年代、永久代等的容量、已用空间、垃圾手机时间合计等信息
	- `-gccapacity`：监视内容与 `-gc`基本相同，但输出主要关注Java堆各个区域使用到的最大、最小空间
	- `-gcutil`：监视内容与 `-gc` 基本相同，但输出主要关注已使用空间占总空间的百分比
	- `-gccause`：与 `-gcutil`功能一样，但是会额外输出导致上一次垃圾收集产生的原因
	- `-gcnew`：监视新生代垃圾收集状况
	- `-gcnewcapacity`：监视内容与 `-gcnew`基本相同，输出主要关注使用到的最大、最小空间，这个在Java 7以及以前使用，Java 8开始使用 `-gcmetacapacity
	- `-gcold`：监视老Bash年代垃圾收集状况，还有非堆/元数据区
	- `-gcoldcapacity`：监视内容与 `-gcold`基本相同，输出主要关注使用到的最大、最小空间
	- `-gcpermcapacity`：输出永久代使用到的最大、最小空间
	- `-compiler`：输出即时编译器编译过的方法、耗时等信息
	- `-printcompilation`：输出已经被即时编译的方法

`jstat -class 12345`输出的信息：

```
Loaded  Bytes  Unloaded  Bytes     Time
406     866.1  0         0.0       0.02
```

- `loaded`：已加载的类总数
- `Bytes`：已加载类的总大小
- `Unloaded`：已卸载的类总数
- `Bytes`：已卸载类的总大小
- `Time`：类加载与卸载的总耗时

`jstat -gc 12345`输出的信息：

```
S0C     S1C     S0U  S1U  EC      EU
1024.0  1024.0  0.0  0.0  8192.0  4257.1

OC       OU
10240.0  4610.7

MC      MU      CCSC   CCSU
4864.0  2677.4  512.0  278.3

YGC  YGCT   FGC  FGCT    CGC  CGCT  GCT
2    0.002  0    0.000   -    -     0.002
```

- `S0C`：Survivor 0 Capacity，Survivor 0的容量
- `S1C`：Survivor 1 Capacity，Survivor 1的容量
- `S0U`：Survivor 0 Used，Survivor 0的使用量
- `S1U`：Survivor 1 Used，Survivor 1的使用量
- `EC`：Eden Capacity，Eden的容量
- `EU`：Eden Used，Eden的使用量
- `OC`：Old Capacity，老年代容量
- `OU`：Old Used，老年代使用量
- `MC`：Metaspace Capacity，元空间容量
- `MU`：Metaspace Used，元空间使用量
- `CCSC`：Compressed Class Space Capacity，类压缩空间容量
- `CCSU`：Compressed Class Space Used，类压缩空间使用量
- `YGC`：Young GC次数
- `YGCT`：Young GC耗时
- `FGC`：Full GC次数
- `FGCT`：Full GC耗时
- `CGC`：并发GC次数
- `CGCT`：并发GC耗时
- `GCT`：GC总耗时

`jstat -gccapacity`输出的日志：

```
NGCMN    NGCMX    NGC      S0C     S1C     EC
10240.0  10240.0  10240.0  1024.0  1024.0  8192.0

OGCMN    OGCMX    OGC      OC
10240.0  10240.0  10240.0  10240.0

MCMN  MCMX       MC       CCSMN  CCSMX      CCSC
0.0   1056768.0  4864.0   0.0    1048576.0  512.0

YGC  FGC CGC
2    0   -
```

- `NGCMN`：New Gen Min，新生代最小容量
- `NGCMX`：New Gen Max，新生代最大容量
- `NGC`：New Gen Current，新生代当前总容量
- `S0C`：Survivor 0 Capacity，Survivor 0当前容量
- `S1C`：Survivor 1 Capacity，Survivor 1当前容量
- `EC`：Eden Capacity，Eden当前容量
- `OGCMN`：Old Gen Min，老年代最小容量
- `OGCMX`：Old Gen Max，老年代最大容量
- `OGC`：Old Gen Current，老年代当前总容量
- `OC`：Old Capacity，老年代当前可供分配的容量
- `MCMN`：元空间最小初始容量
- `MCMX`：元空间最大允许容量
- `MC`：元空间当前已提交容量
- `CCSMN`：类压缩空间最小初始容量
- `CCSMX`：类压缩空间最大允许容量
- `CCSC`：类压缩空间当前已提交容量
- `YGC`：Young GC次数
- `FGC`：Full GC次数
- `CGC`：并发GC次数

`jstat -gcutil 12345`输出的日志：

```
S0     S1    E      O      M      CCS
0.00   0.00  51.97  45.03  55.05  54.36

YGC  YGCT   FGC  FGCT   CGC  CGCT  GCT
2    0.002  0    0.000  -    -     0.002
```

- `S0`：Survivor 0使用率
- `S1`：Survivor 1使用率
- `E`：Eden使用率
- `O`：老年代使用率
- `M`：元空间使用率
- `CCS`：类压缩空间使用率
- `YGC`：Young GC次数
- `YGCT`：Young GC累计耗时
- `FGC`：Full GC次数
- `FGCT`：Full GC累计耗时
- `CGC`：并发GC次数
- `CGCT`：并发GC累计耗时
- `GCT`：GC总耗时

`jstat -gccause 12345`输出的日志：

```
S0     S1    E      O      M      CCS
0.00   0.00  51.97  45.03  55.05  54.36

YGC  YGCT   FGC  FGCT   CGC  CGCT  GCT
2    0.002  0    0.000  -    -     0.002

LGCC                 GCC
Allocation Failure   No GC
```

- `S0`：Survivor 0使用率
- `S1`：Survivor 1使用率
- `E`：Eden使用率
- `O`：老年代使用率
- `M`：元空间使用率
- `CCS`：类压缩空间使用率
- `YGC`：Young GC次数
- `YGCT`：Young GC累计耗时
- `FGC`：Full GC次数
- `FGCT`：Full GC累计耗时
- `CGC`：并发GC次数
- `CGCT`：并发GC累计耗时
- `GCT`：GC总耗时
- `LGCC`：Last Garbage Collection Cause，上次触发GC的原因
- `GCC`：Current Garbage Collection Cause，当前触发GC的原因

`jstat -gcnew 12345`输出的日志：

```
S0C     S1C     S0U  S1U  TT  MTT  DSS    EC      EU       YGC   YGCT
1024.0  1024.0  0.0  0.0  1   1    512.0  8192.0  4257.1   2     0.002
```

- `S0C`：Survivor 0 Capacity，Survivor 0的容量
- `S1C`：Survivor 1 Capacity，Survivor 1的容量
- `S0U`：Survivor 0 Used，Survivor 0的使用量
- `S1U`：Survivor 1 Used，Survivor 1的使用量
- `EC`：Eden Capacity，Eden的容量
- `EU`：Eden Used，Eden的使用量
- `TT`：Tenuring Threshold，实际晋升年龄阈值
- `MTT`：Maximum Tenuring Threshold，最大允许晋升年龄阈值
- `DSS`：Desired Survivor Size，期望的Survivor区大小
- `YGC`：Young GC累计次数
- `YGCT`：Young GC累计耗时

`jstat -gcnewcapacity 12345`输出的日志：

```
NGCMN    NGCMX    NGC      S0CMX   S0C     S1CMX   S1C     ECMX    EC
10240.0  10240.0  10240.0  1024.0  1024.0  1024.0  1024.0  8192.0  8192.0

YGC  FGC  CGC
2    0    -
```

- `NGCMN`：New Gen Min，新生代最小容量
- `NGCMX`：New Gen Max，新生代最大容量
- `NGC`：New Gen Current，新生代当前总容量
- `S0CMX`：Survivor 0 Max，Survivor 0最大允许容量
- `S0C`：Survivor 0 Capacity，Survivor 0当前容量
- `S1CMX`：Survivor 1 Max，Survivor 1最大允许容量
- `S1C`：Survivor 1 Capacity，Survivor 1当前容量
- `ECMX`：Eden最大允许容量
- `EC`：Eden Capacity，Eden当前容量
- `YGC`：Young GC次数
- `FGC`：Full GC次数
- `CGC`：并发GC次数

`jstat -gcold 12345`输出的日志：

```
MC      MU      CCSC   CCSU   OC       OU
4864.0  2677.4  512.0  278.3  10240.0  4610.7

YGC  FGC  FGCT   CGC  CGCT  GCT
2    0    0.000  -    -     0.002
```

- `MC`：Metaspace Capacity，元空间容量
- `MU`：Metaspace Used，元空间使用量
- `CCSC`：Compressed Class Space Capacity，类压缩空间容量
- `CCSU`：Compressed Class Space Used，类压缩空间使用量
- `OC`：Old Capacity，老年代容量
- `OU`：Old Used，老年代使用量
- `YGC`：Young GC次数
- `FGC`：Full GC次数
- `FGCT`：Full GC耗时
- `CGC`：并发GC次数
- `CGCT`：并发GC耗时
- `GCT`：GC总耗时

`jstat -gcoldcapacity 12345`输出的日志：

```
OGCMN    OGCMX    OGC      OC
10240.0  10240.0  10240.0  10240.0

YGC  FGC  FGCT   CGC  CGCT  GCT
2    0    0.000  -    -     0.002
```

- `OGCMN`：Old Gen Min，老年代最小容量
- `OGCMX`：Old Gen Max，老年代最大容量
- `OGC`：Old Gen Current，老年代当前总容量
- `OC`：Old Capacity，老年代当前可供分配的容量
- `YGC`：Young GC次数
- `FGC`：Full GC次数
- `FGCT`：Full GC耗时
- `CGC`：并发GC次数
- `CGCT`：并发GC耗时
- `GCT`：GC总耗时

`jstat -gcmetacapacity 12345`输出的日志：

```
MCMN  MCMX       MC      CCSMN  CCSMX      CCSC 
0.0   1056768.0  4864.0  0.0    1048576.0  512.0

YGC  FGC  FGCT   CGC  CGCT  GCT
2    0    0.000  -    -     0.002
```

- `MCMN`：元空间最小初始容量
- `MCMX`：元空间最大允许容量
- `MC`：元空间当前已提交容量
- `CCSMN`：类压缩空间最小初始容量
- `CCSMX`：类压缩空间最大允许容量
- `CCSC`：类压缩空间当前已提交容量
- `YGC`：Young GC次数
- `FGC`：Full GC次数
- `FGCT`：Full GC耗时
- `CGC`：并发GC次数
- `CGCT`：并发GC耗时
- `GCT`：GC总耗时

`jstat -compiler 12345`输出的日志：

```
Compiled  Failed  Invalid  Time  FailedType  FailedMethod
11        0       0        0.00  0
```

- `Compiled`：已触发JIT编译的方法总数
- `Failed`：编译失败的方法总数
- `Invalid`：失效的编译代码总数
- `Time`：JIT编译累计耗时
- `FailedType`：最后一次失败的编译类型
- `Failed Method`：最后一次失败的方法名

`jstat -printcompilation`输出的日志：

```
Compiled  Size  Type  Method
11        5     1     java/lang/ThreadLocal access$400
```

- `Compiled`：最近一次编译的任务序号
- `Size`：字节码大小
- `Type`：JIT编译类型/级别
- `Method`：被编译的具体类与方法名

### 4.2.3 jinfo：Java配置信息工具

jinfo： Configuration Info for Java，实时查看和调整虚拟机各项参数。

命令格式： `jinfo [option] <pid>`：

- `jinfo <pid>`：打印所有配置信息，包括JVM参数和System属性
- `-flags <pid>`：打印JVM的启动参数与默认生效参数
- `-flag <name> <pid>`：查询指定的名称的参数取值值
- `-sysprops <pid>`：打印Java的系统属性


`jinfo 12345`输出的日志：

```
Java System Properties:
#Wed Jul 29 20:58:47 CST 2026
java.runtime.name=OpenJDK Runtime Environment
sun.boot.library.path=/usr/lib/jvm/java-8-openjdk/jre/lib/amd64
java.vm.version=25.502-b07
java.vm.vendor=Arch Linux
java.vendor.url=http\://java.oracle.com/
path.separator=\:
java.vm.name=OpenJDK 64-Bit Server VM
file.encoding.pkg=sun.io
user.country=US
sun.java.launcher=SUN_STANDARD
sun.os.patch.level=unknown
java.vm.specification.name=Java Virtual Machine Specification
user.dir=/home/fs/develop/ThirtySix/UtJ/target/classes
java.runtime.version=1.8.0_502-b07
java.awt.graphicsenv=sun.awt.X11GraphicsEnvironment
java.endorsed.dirs=/usr/lib/jvm/java-8-openjdk/jre/lib/endorsed
os.arch=amd64
java.io.tmpdir=/tmp
line.separator=\n
java.vm.specification.vendor=Oracle Corporation
os.name=Linux
sun.jnu.encoding=UTF-8
java.library.path=/usr/java/packages/lib/amd64\:/usr/lib64\:/lib64\:/lib\:/usr/lib
java.specification.name=Java Platform API Specification
java.class.version=52.0
sun.management.compiler=HotSpot 64-Bit Tiered Compilers
os.version=7.1.5-arch1-1
user.home=/home/fs
user.timezone=
java.awt.printerjob=sun.print.PSPrinterJob
file.encoding=UTF-8
java.specification.version=1.8
user.name=fs
java.class.path=.
java.vm.specification.version=1.8
sun.java.command=fun.pullock.utj.c3_8_3_allocation.TenuringThreshold
java.home=/usr/lib/jvm/java-8-openjdk/jre
sun.arch.data.model=64
user.language=en
java.specification.vendor=Oracle Corporation
awt.toolkit=sun.awt.X11.XToolkit
java.vm.info=mixed mode
java.version=1.8.0_502
java.ext.dirs=/usr/lib/jvm/java-8-openjdk/jre/lib/ext\:/usr/java/packages/lib/ext
sun.boot.class.path=/usr/lib/jvm/java-8-openjdk/jre/lib/resources.jar\:/usr/lib/jvm/java-8-openjdk/jre/lib/rt.jar\:/usr/lib/jvm/java-8-openjdk/jre/lib/sunrsasign.jar\:/usr/lib/jvm/java-8-openjdk/jre/lib/jsse.jar\:/usr/lib/jvm/java-8-openjdk/jre/lib/jce.jar\:/usr/lib/jvm/java-8-openjdk/jre/lib/charsets.jar\:/usr/lib/jvm/java-8-openjdk/jre/lib/jfr.jar\:/usr/lib/jvm/java-8-openjdk/jre/classes
java.vendor=Arch Linux
java.specification.maintenance.version=6
file.separator=/
java.vendor.url.bug=http\://bugreport.sun.com/bugreport/
sun.io.unicode.encoding=UnicodeLittle
sun.cpu.endian=little
sun.cpu.isalist=

VM Flags:
-XX:CICompilerCount=4 -XX:InitialHeapSize=20971520 -XX:InitialTenuringThreshold=1 -XX:MaxHeapSize=20971520 -XX:MaxNewSize=10485760 -XX:MaxTenuringThreshold=1 -XX:MinHeapDeltaBytes=196608 -XX:NewSize=10485760 -XX:OldSize=10485760 -XX:+PrintGC -XX:+PrintGCCause -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintTenuringDistribution -XX:SurvivorRatio=8 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseSerialGC 

VM Arguments:
jvm_args: -Xms20M -Xmx20M -Xmn10M -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=1 -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+PrintGCCause -XX:+PrintTenuringDistribution -XX:+UseSerialGC 
java_command: fun.pullock.utj.c3_8_3_allocation.TenuringThreshold
java_class_path (initial): .
Launcher Type: SUN_STANDARD
```


- `Java System Properties:`:  Java系统属性
- `#Wed Jul 29 20:58:47 CST 2026`:  时间戳
- `java.runtime.name=OpenJDK Runtime Environment`:  Java运行时环境名称
- `sun.boot.library.path=/usr/lib/jvm/java-8-openjdk/jre/lib/amd64`:  JVM启动依赖的本地原生C/C++动态链接库路径
- `java.vm.version=25.502-b07`:  HotSpot VM底层版本号
- `java.vm.vendor=Arch Linux`:  虚拟机实现的构建方
- `java.vendor.url=http\://java.oracle.com/`:  Java供应商的官方网站
- `path.separator=\:`:  环境变量/类路径环境变量的分隔符
- `java.vm.name=OpenJDK 64-Bit Server VM`:  JVM虚拟机的的名称
- `file.encoding.pkg=sun.io`:  用于字符编码转换处理的底层内部包名
- `user.country=US`:  当前系统的国家/地区设置
- `sun.java.launcher=SUN_STANDARD`:  Java启动器类型
- `sun.os.patch.level=unknown`:  操作系统的补丁等级
- `java.vm.specification.name=Java Virtual Machine Specification`:  JVM规范的名称
- `user.dir=/home/fs/develop/ThirtySix/UtJ/target/classes`:  Java进程的工作目录
- `java.runtime.version=1.8.0_502-b07`:  Java运行时完整版本号，Java 8 Update 502 Build 07
- `java.awt.graphicsenv=sun.awt.X11GraphicsEnvironment`:  基于X11显示协议的图形界面环境
- `java.endorsed.dirs=/usr/lib/jvm/java-8-openjdk/jre/lib/endorsed`:  已废弃的Endorsed标准覆盖机制路径
- `os.arch=amd64`:  操作系统硬件架构为64位
- `java.io.tmpdir=/tmp`: 
- `line.separator=\n`:  操作系统的换行符
- `java.vm.specification.vendor=Oracle Corporation`:  JVM规范的制定者
- `os.name=Linux`:  操作系统类型
- `sun.jnu.encoding=UTF-8`:  Java虚拟机内部使用的字符编码
- `java.library.path=/usr/java/packages/lib/amd64\:/usr/lib64\:/lib64\:/lib\:/usr/lib`:  加载Native动态库的搜索路径
- `java.specification.name=Java Platform API Specification`:  Java平台API规范的名称
- `java.class.version=52.0`:  字节码版本号，Java 8对应的是52
- `sun.management.compiler=HotSpot 64-Bit Tiered Compilers`:  JVM使用的编译器，使用了分层编译（C1+C2编译器协同工作）
- `os.version=7.1.5-arch1-1`:  操作系统内核版本号
- `user.home=/home/fs`:  用户主目录
- `user.timezone=`:  当前用户的时区设置
- `java.awt.printerjob=sun.print.PSPrinterJob`:  默认的打印机作业实现
- `file.encoding=UTF-8`:  Java默认的文件字符集编码
- `java.specification.version=1.8`:  Java平台API规范的版本
- `user.name=fs`: 用户名
- `java.class.path=.`:  应用类路径
- `java.vm.specification.version=1.8`:  JVM规范的版本号
- `sun.java.command=fun.pullock.utj.c3_8_3_allocation.TenuringThreshold`:  启动该Java进程时执行的主类的全限定名
- `java.home=/usr/lib/jvm/java-8-openjdk/jre`:  JRE（Java运行时环境）的安装根目录
- `sun.arch.data.model=64`:  JVM的数据模型位数
- `user.language=en`:  当前系统的语言环境设置
- `java.specification.vendor=Oracle Corporation`:  Java平台规范的制定者
- `awt.toolkit=sun.awt.X11.XToolkit`:  底层的AWT GUI工具包实现类
- `java.vm.info=mixed mode`:  JVM的执行模式，混合模式（解释执行+JIT即时编译）
- `java.version=1.8.0_502`:  Java的标准版本号
- `java.ext.dirs=/usr/lib/jvm/java-8-openjdk/jre/lib/ext\:/usr/java/packages/lib/ext`:  扩展类加载器搜索的扩展jar包目录
- `sun.boot.class.path=/usr/lib/jvm/java-8-openjdk/jre/lib/resources.jar\:/usr/lib/jvm/java-8-openjdk/jre/lib/rt.jar\:/usr/lib/jvm/java-8-openjdk/jre/lib/sunrsasign.jar\:/usr/lib/jvm/java-8-openjdk/jre/lib/jsse.jar\:/usr/lib/jvm/java-8-openjdk/jre/lib/jce.jar\:/usr/lib/jvm/java-8-openjdk/jre/lib/charsets.jar\:/usr/lib/jvm/java-8-openjdk/jre/lib/jfr.jar\:/usr/lib/jvm/java-8-openjdk/jre/classes`:  引导类加载器加载的核心类库jar包路径
- `java.vendor=Arch Linux`:  JDK发行包的打包/提供方
- `java.specification.maintenance.version=6`:  Java规范的维护版本号
- `file.separator=/`:  文件路径分隔符
- `java.vendor.url.bug=http\://bugreport.sun.com/bugreport/`:  提交Java缺陷的网址
- `sun.io.unicode.encoding=UnicodeLittle`:  JVM内部Unicode字符的编码表示形式（小端序）
- `sun.cpu.endian=little`:  CPU的字节序，小端字节序
- `sun.cpu.isalist=`:  CPU支持的指令集列表
- `VM Flags:`:  JVM生效的配置标志
- `-XX:CICompilerCount=4 -XX:InitialHeapSize=20971520 -XX:InitialTenuringThreshold=1 -XX:MaxHeapSize=20971520 -XX:MaxNewSize=10485760 -XX:MaxTenuringThreshold=1 -XX:MinHeapDeltaBytes=196608 -XX:NewSize=10485760 -XX:OldSize=10485760 -XX:+PrintGC -XX:+PrintGCCause -XX:+PrintGCDateStamps -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintTenuringDistribution -XX:SurvivorRatio=8 -XX:+UseCompressedClassPointers -XX:+UseCompressedOops -XX:+UseFastUnorderedTimeStamps -XX:+UseSerialGC `: 
- `VM Arguments:`:  虚拟机启动参数
- `jvm_args: -Xms20M -Xmx20M -Xmn10M -XX:SurvivorRatio=8 -XX:MaxTenuringThreshold=1 -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCTimeStamps -XX:+PrintGCCause -XX:+PrintTenuringDistribution -XX:+UseSerialGC `:  显式传递给JVM的参数字符串
- `java_command: fun.pullock.utj.c3_8_3_allocation.TenuringThreshold`:  入口类的完整包名与类名
- `java_class_path (initial): .`:  启动时初始化的类路径
- `Launcher Type: SUN_STANDARD`:  启动器类型

### 4.2.4 jmap： Java内存映像工具

jmap： Memory Map for Java，用于生成堆转储快照，一般为heapdump或dump文件。

命令格式： `jmap [option] vmid`：

- `-dump`：生成Java堆转储快照，格式为 `-dump:[live,]format=b,file=<filename>`，其中live子参数说明是否只dump存活的对象
- `-finalizerinfo`：显示在F-Queue中等待Finalizer线程执行finalize方法的对象
- `-heap`：显示Java堆详细信息
- `-histo`：显示堆中对象统计信息
- `-permstat`：显示永久代内存状态
- `-F`：当虚拟机进程对 `-dump`选项没有响应时，可使用该选项强制生成dump快照

### 4.2.5 jhat：虚拟机堆转储快照分析工具

jhat: JVM Heap Analysis Tool，分析jmap生成的堆转储快照。

### 4.2.6 jstack：Java堆栈跟踪工具

jstack: Stack Trace for Java，用于生成虚拟机当前时刻的线程快照，一般称为threaddump或javacore文件。

命令格式： `jstack [option] vmid`：

- `-F`： 当正常输出的请求不被响应时，强制输出线程堆栈
- `-l`： 除堆栈外，显示关于锁的附加信息
- `-m`： 如果调用到本地方法的话，可以显示C/C++的堆栈

`jstack -l 12345`输出的日志：

```
2026-07-30 18:59:19
Full thread dump OpenJDK 64-Bit Server VM (25.502-b07 mixed mode):

"Attach Listener" #10 daemon prio=9 os_prio=0 tid=0x00007f98a8001800 nid=0x153d2 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"Service Thread" #9 daemon prio=9 os_prio=0 tid=0x00007f98cc0cf000 nid=0x15354 runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"C1 CompilerThread3" #8 daemon prio=9 os_prio=0 tid=0x00007f98cc0ba000 nid=0x15353 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"C2 CompilerThread2" #7 daemon prio=9 os_prio=0 tid=0x00007f98cc0b7800 nid=0x15352 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"C2 CompilerThread1" #6 daemon prio=9 os_prio=0 tid=0x00007f98cc0b5800 nid=0x15351 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"C2 CompilerThread0" #5 daemon prio=9 os_prio=0 tid=0x00007f98cc0b3800 nid=0x15350 waiting on condition [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"Signal Dispatcher" #4 daemon prio=9 os_prio=0 tid=0x00007f98cc0ad800 nid=0x1534f runnable [0x0000000000000000]
   java.lang.Thread.State: RUNNABLE

   Locked ownable synchronizers:
	- None

"Finalizer" #3 daemon prio=8 os_prio=0 tid=0x00007f98cc07d800 nid=0x1534e in Object.wait() [0x00007f98bc1fd000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	- waiting on <0x00000000ffa10fa8> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:144)
	- locked <0x00000000ffa10fa8> (a java.lang.ref.ReferenceQueue$Lock)
	at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:165)
	at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:188)

   Locked ownable synchronizers:
	- None

"Reference Handler" #2 daemon prio=10 os_prio=0 tid=0x00007f98cc079000 nid=0x1534d in Object.wait() [0x00007f98bc2fd000]
   java.lang.Thread.State: WAITING (on object monitor)
	at java.lang.Object.wait(Native Method)
	- waiting on <0x00000000ffa11160> (a java.lang.ref.Reference$Lock)
	at java.lang.Object.wait(Object.java:502)
	at java.lang.ref.Reference.tryHandlePending(Reference.java:191)
	- locked <0x00000000ffa11160> (a java.lang.ref.Reference$Lock)
	at java.lang.ref.Reference$ReferenceHandler.run(Reference.java:153)

   Locked ownable synchronizers:
	- None

"main" #1 prio=5 os_prio=0 tid=0x00007f98cc009800 nid=0x1534b waiting on condition [0x00007f98d03fe000]
   java.lang.Thread.State: TIMED_WAITING (sleeping)
	at java.lang.Thread.sleep(Native Method)
	at fun.pullock.utj.c3_8_3_allocation.TenuringThreshold.testTenuringThreshold(TenuringThreshold.java:33)
	at fun.pullock.utj.c3_8_3_allocation.TenuringThreshold.main(TenuringThreshold.java:8)

   Locked ownable synchronizers:
	- None

"VM Thread" os_prio=0 tid=0x00007f98cc06f800 nid=0x1534c runnable 

"VM Periodic Task Thread" os_prio=0 tid=0x00007f98cc0d1800 nid=0x15355 waiting on condition 

JNI global references: 5
```

- `2026-07-30 18:59:19`: 堆栈快照生成的时间
- `Full thread dump OpenJDK 64-Bit Server VM (25.502-b07 mixed mode)`:  
	- `Full thread dump`: 表示包含当前JVM内所有线程状态的全量堆栈转储
	- `OpenJDK 64-Bit Server VM`: 当前运行的JVM类型为64位OpenJDK的Server版本
	- `25.502-b07 mixed mode`: 虚拟机版本为Java 8 Update 502，混合模式运行（解释执行+JIT动态编译）
- `"Attach Listener" #10 daemon prio=9 os_prio=0 tid=0x00007f98a8001800 nid=0x153d2 waiting on condition [0x0000000000000000]`: 
	- `"Attach Listener"`: 线程名称。该线程负责接收并处理来自外部诊断工具（如jstack、jmap、jinfo）的连接与命令。
	- `#10`: 线程的编号
	- `daemon`: 守护线程
	- `prio=9`: Java语言层面的线程优先级，取值范围1-10，默认为5
	- `os_prio=0`: 操作系统层面的线程优先级
	- `tid=0x00007f98a8001800`: Thread ID，JVM在C++内部为该线程分配的内存结构体指针地址
	- `nid=0x153d2`: Native Thread ID，对应操作系统底层的真实线程ID/进程PID
	- `waiting on condition`: 线程当前正在等待某个条件发生
	- `[0x0000000000000000]`: 当前线程栈顶的内存地址，全为0表示该JVM内部Native线程不需要展示Java级别的调用栈
- `java.lang.Thread.State: RUNNABLE`: Java语言层面的线程状态为Runnable，表示该线程在JVM看来处于可运行状态
- `Locked ownable synchronizers:`
	- `None`: 该线程当前没有持有任何基于AQS的排他锁
- `Service Thread`: 用于接受JVM内部产生的各种服务通知，如内存低警告、GC统计信号、JMX性能监控指标等
- `C1 CompilerThread`和 `C2 CompilerThread`: 由于启用了分层编译，JVM会同时启动C1（Client级快速编译）和C2（Server级深度优化编译）线程。
- `Signal Dispatcher`: 信号分发线程，负责将操作系统发送给JVM进程的信号分发给内部对应的Handler进行处理
- `Finalizer`: 处理 `finalize()`方法的系统线程
	- `in Object.wait()`: 该线程目前阻塞在 `Object.wait()`方法上
	- `[0x00007f98bc1fd000]`: 当前线程栈在内存中的起始基地址
	- `java.lang.Thread.State: WAITING (on object monitor)`: 线程状态为WAITING，等待对象监视器（synchronized），处于无限期等待状态
	- `at java.lang.Object.wait(Native Method)`: 正在执行 `Object.wait()`原生方法
	- `waiting on <0x00000000ffa10fa8> (a java.lang.ref.ReferenceQueue$Lock)`: 正在等待地址为 `0x00000000ffa10fa8`的锁对象唤醒，该锁对象的类型是 `ReferenceQueue$Lock`
	- `at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:144)`: 调用栈位于 `ReferenceQueue.remove`方法第144行
	- `- locked <0x00000000ffa10fa8> (a java.lang.ref.ReferenceQueue$Lock)`: 已成功获得了地址为 `0x00000000ffa10fa8`的 `ReferenceQueue$Lock`内置锁，随后在 `wait()`中释放了该锁并进入等待状态
	- `at java.lang.ref.ReferenceQueue.remove(ReferenceQueue.java:165)`
	- `at java.lang.ref.Finalizer$FinalizerThread.run(Finalizer.java:188)`: Finalizer Thread的入口调用栈
- `Reference Handler`: 引用处理线程，用于将GC垃圾回收器清理出来的软引用、弱引用、虚引用移入对应的 `ReferenceQueue`中
- `main`: 程序主线程
	- `waiting on condition`: 正处于条件等待状态（此时是睡眠）
	- `java.lang.Thread.State: TIMED_WAITING (sleeping)`: 状态是TIMED_WAITING，带超时的等待，sleeping表示在代码中调用了 `Thread.sleep()`
	- `at java.lang.Thread.sleep(Native Method)`
	- `at fun.pullock.utj.c3_8_3_allocation.TenuringThreshold.testTenuringThreshold(TenuringThreshold.java:33)`: `main`方法调用了 `testTenuringThreshold()`方法，在 `TenuringThreshold.java`第33行处代码执行了 `Thread.sleep()`导致主线程进入了休眠状态
	- `at fun.pullock.utj.c3_8_3_allocation.TenuringThreshold.main(TenuringThreshold.java:8)`: 程序主入口是 `main`方法，位于 `TenuringThreshold.java`第8行
- `VM Thread`: HotSpot的核心线程，负责JVM核心特殊操作，比如发起STW、触发GC、生成Thread Dump、执行类卸载等
- `VM Periodic Task Thread`: JVM内部的定时器、周期性任务线程，负责定期执行一些内部采样与定时任务
- `JNI global references: 5`: 当前JVM内部由Native层面持有的Java全局对象引用数量为5个。垃圾回收时，这些JNI全局引用会被直接作为GC Roots进行可达性分析

### 4.2.7 基础工具总结

基础工具：用于支持基本的程序创建和运行

| 名称           | 作用                                  |
| ------------ | ----------------------------------- |
| appletviewer | 在不使用Web浏览器的情况下运行和调试Applet，JDK11中被移除 |
| extcheck     | 检查JAR冲突的工具，从JDK 9中被移除               |
| jar          | 创建和管理JAR文件                          |
| java         | Java运行工具，用于运行Class文件或JAR文件          |
| javac        | 用于Java编程语言的编译器                      |
| javadoc      | Java的API文档生成器                       |
| javah        | C语言头文件和Stub函数生成器，用于编写JNI方法          |
| javap        | Java字节码分析工具                         |
| jlink        | 将Module和它的依赖打包成一个运行时镜像文件            |
| jdb          | 基于JPDA协议的调试器，以类似于GDB的方式进行调试Java代码   |
| jdeps        | Java类依赖性分析器                         |
| jdeprscan    | 用于搜索JAR包中使用了deprecated的类，从JDK 9开始提供 |


安全工具：用于签名、设置安全测试等

| 名称         | 作用                     |
| ---------- | ---------------------- |
| keytool    | 管理密钥库和证书。              |
| jarsigner  | 生成并验证JAR签名             |
| policytool | 管理策略文件的GUI工具，JDK 10被移除 |


国际化：用于创建本地语言文件

| 名称           | 作用               |
| ------------ | ---------------- |
| native2ascii | 本地编码到ASCII编码的转换器 |


远程方法调用：用于跨Web或网络的服务交互

| 名称          | 作用                                    |
| ----------- | ------------------------------------- |
| rmic        | Java RMI编译器                           |
| rmiregistry | 远程对象注册表服务，用于在当前主机的指定端口上创建并启动一个远程对象注册表 |
| rmid        | 启动激活系统守护进程，允许在虚拟机中注册或激活对象             |
| serialver   | 生成并返回指定类的序列化版本ID                      |


部署工具：用于程序打包、发布和部署

| 名称           | 作用                                  |
| ------------ | ----------------------------------- |
| javapackager | 打包、签名Java和JavaFX应用程序，在JDK 11中被移除    |
| pack200      | 使用Java GZIP压缩器将JAR文件转换为压缩的Pack200文件 |
| unpack200    | 将Pack200生成的打包文件解压提取为JAR文件           |


性能监控和故障处理：用于监控分析Java虚拟机运行信息，排查问题


| 名称        | 全称                                     | 作用                                                                                                   |
| --------- | -------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| jps       | JVM Process Status Tool                | 显示虚拟机进程                                                                                              |
| jstat     | JVM Statistics Monitoring Tool         | 收集虚拟机运行数据                                                                                            |
| jstatd    | JVM Statistics Monitoring Tool Daemon  | jstat的守护程序，启动一个RMI服务器应用程序，用于监视测试的HotSpot虚拟机的创建和终止，并提供一个界面，允许远程监控工具附加到在本地系统上运行的虚拟机。在JDK 9中集成到了JHSDB中。 |
| jinfo     | Configuration Info for Java            | 显示虚拟机配置信息。在JDK 9中集成到了JHSDB中                                                                          |
| jmap      | Memory Map for Java                    | 生成虚拟机的内存转储快照。在JDK 9中集成到了JHSDB中                                                                       |
| jhat      | JVM Heap Analysis Tool                 | 分析堆转储快照，会建立一个HTTP/Web服务器，可以在浏览器查看分析结果。在JDK 9中被JHSDB代替                                                |
| jstack    | Stack Trace for Java                   | 显示虚拟机的线程快照。在JDK 9中集成到了JHSDB中                                                                         |
| jhsdb     | Java HotSpot Debugger                  | 基于Serviceability Agent的HotSpot进程调试器，从JDK 9开始提供                                                       |
| jsagebugd | Java Serviceability Agent Debug Daemon | 适用于Java的可维护性代理调试守护程序，主要用于附加到指定的Java进程、核心文件或充当一个调试服务器                                                 |
| jcmd      | JVM Command                            | 虚拟机诊断命令工具，将诊断命令请求发送到正在运行的Java虚拟机。JDK 7开始提供                                                           |
| jconsole  | Java Console                           | 用于监控Java虚拟机的使用JMX规范的图形工具。可以监控本地和远程Java虚拟机，可监控和管理应用程序                                                 |
| jmc       | Java Mission Control                   | 包含用于监控和管理Java应用程序的工具，而不会引入与这些工具相关联的性能开销                                                              |
| jvisualvm | Java VisualVM                          | 图形化工具                                                                                                |


REPL和脚本工具：

| 名称         | 作用                                                                           |
| ---------- | ---------------------------------------------------------------------------- |
| jshell     | 基于Java的Shell REPL（Read-Eval-Print Loop）交互工具                                  |
| jjs        | 堆Nashorn引擎的调用入口。Nashorn是基于Java实现的一个JavaScript运行环境                            |
| jrunscript | Java命令行脚本外壳工具（Command Line Script Shell），主要用于解释执行JavaScript、Groovy、Ruby等脚本语言 |


## 4.3 可视化故障处理工具

### 4.3.1 JHSDB：基于服务性代理的调试工具

示例代码：

```java
public class JHSDB {

    static class Test {
        static ObjectHolder staticObj = new ObjectHolder();
        ObjectHolder instanceObj = new ObjectHolder();

        void foo() {
            ObjectHolder localObj = new ObjectHolder();

            while (true) {
                try {
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
        }
    }

    private static class ObjectHolder {
    }

    public static void main(String[] args) {
        Test test = new Test();
        test.foo();
    }
}
```

运行命令及参数：

- 程序: `java -Xms10M -Xmx10M -XX:+UseSerialGC fun.pullock.utj.c4_3_1_jhsdb.JHSDB`
- jhsdb: `jhsdb hsdb`

在菜单`Tools --> Head Parameters`中打开 `Heap Parameters`，显示内存布局：

```
Heap Parameters:
Gen 0:   eden [0x00000000ff600000,0x00000000ff703cb0,0x00000000ff8b0000) space capacity = 2818048, 37.76060592296512 used
  from [0x00000000ff8b0000,0x00000000ff8b0000,0x00000000ff900000) space capacity = 327680, 0.0 used
  to   [0x00000000ff900000,0x00000000ff900000,0x00000000ff950000) space capacity = 327680, 0.0 usedInvocations: 0

Gen 1:   old  [0x00000000ff950000,0x00000000ff950000,0x0000000100000000) space capacity = 7012352, 0.0 usedInvocations: 0
```

- `Gen 0`: 年轻代
- `eden`: Eden区
- `from`: From Survivor
- `to`: To Survivor
- `Gen 1`: 老年代
- `old`: 老年代
- `[bottom, top, end)`: 内存地址范围
	- `bottom`: 一个区域的内存起始基地址
	- `top`: 当前指针位置
	- `end`: 一个区域结束的内存地址
- `space capacity`: 一个区域的内存总容量
- `used`: 一个区域已使用的百分比
- `Invocations`: 触发垃圾回收次数

在菜单 `Windows --> Console`中，使用命令在新生代中查找 `ObjectHolder`的实例： `scanoops 0x00000000ff600000 0x00000000ff950000 fun.pullock.utj.c4_3_1_jhsdb.JHSDB$ObjectHolder`，输出的信息如下：

```
hsdb> scanoops 0x00000000ff600000 0x00000000ff950000 fun.pullock.utj.c4_3_1_jhsdb.JHSDB$ObjectHolder
0x00000000ff6fdce8 fun/pullock/utj/c4_3_1_jhsdb/JHSDB$ObjectHolder
0x00000000ff6fdd08 fun/pullock/utj/c4_3_1_jhsdb/JHSDB$ObjectHolder
0x00000000ff6fdd18 fun/pullock/utj/c4_3_1_jhsdb/JHSDB$ObjectHolder
```

在菜单 `Tools --> Inspector`中输入上面对象的地址可以查看每个对象的信息，地址 `0x00000000ff6fdce8`对应的对象信息如下：

```
Oop for fun/pullock/utj/c4_3_1_jhsdb/JHSDB$ObjectHolder @ 0x00000000ff6fdce8
	_mark: 1
	_metadata._compressed_klass: InstanceKlass for fun/pullock/utj/c4_3_1_jhsdb/JHSDB$ObjectHolder
		_java_mirror: Oop for java/lang/Class @ 0x00000000ff6fdbc8
		_super: InstanceKlass for java/lang/Object
		_layout_helper: 16
		_access_flags: 538968096
		_subklass: null
		_next_sibling: InstanceKlass for fun/pullock/utj/c4_3_1_jhsdb/JHSDB$Test
		_vtable_len: 5
		_array_klasses: null
		_nonstatic_field_size: 0
		_static_field_size: 0
		_static_oop_field_count: 0
		_nonstatic_oop_map_size: 0
		_is_marked_dependent: 0
		_init_state: 4
		_itable_len: 2
```

- `Oop for fun/pullock/utj/c4_3_1_jhsdb/JHSDB$ObjectHolder @ 0x00000000ff6fdce8`
	- `Oop`: Ordinary Object Pointer，普通对象指针，这是HotSpot内部在C++层面表示Java对象实例的句柄/指针类型
	- `fun/pullock/utj/c4_3_1_jhsdb/JHSDB$ObjectHolder`: Java对象的类全限定名
	- `@ 0x00000000ff6fdce8`: 对象实例在Java堆内存中的起始地址
- `_mark: 1`: 对象的Mark Word，占用8字节
	- `1`: 对应十六进制为 `0x0000000000000001`，其中二进制最后两位是 `01`，偏向锁标记为 `0`，该对象处于无锁状态，未生成HashCode，年龄为0
- `_metadata._compressed_klass: InstanceKlass for fun/pullock/utj/c4_3_1_jhsdb/JHSDB$ObjectHolder`: 类元数据
	- `_metadata._compressed_klass`:  对象的元数据指针（Klass Word），指向元空间中该类对应的 InstanceKlass C++对象
	- `InstanceKlass for fun/pullock/utj/c4_3_1_jhsdb/JHSDB$ObjectHolder`: HotSpot中用来表示Java类元数据的底层C++数据结构
- `_java_mirror: Oop for java/lang/Class @ 0x00000000ff6fdbc8`
	- `_java_mirror`: Java镜像对象（Class对象）
	- 元空间中的C++ `InstanceKlass`对象不能直接被Java代码访问，JVM会在Java堆中为每个类创建一个 `java.lang.Class`实例作为镜像。Java代码中调用的 `ObjectHolder.class`就是位于这个 `0x00000000ff6fdbc8`地址处的镜像对象
- `_super: InstanceKlass for java/lang/Object`: 当前类的父类元数据指针
- `_layout_helper: 16`: 对象布局，大小为16字节
	- 8字节的Mark Word
	- 4字节的Compressed Klass Pointer
	- 4字节的字节对齐
- `_access_flags: 538968096`: 类的访问标志位，访问修饰符，比如 `public, static, final`等的二进制掩码
- `_subklass: null`: 指向该类的第一个子类的InstanceKlass指针， `null`表示没有任何类继承 `ObjectHolder`
- `_next_sibling: InstanceKlass for fun/pullock/utj/c4_3_1_jhsdb/JHSDB$Test`: 指向同一个ClassLoader的下一个兄弟类的元数据，这里是指向了 `JHSDB$Test`类
- `_vtable_len: 5`: 虚方法表长度为5，用于实现Java的多态与动态分配。这里的5个方法来自 `Object`类
- `_array_klasses: null`: 如果创建类该类的数组 `ObjectHolder[]`，此指针会指向该数组类型的 `ObjArrayKlass`
- `_nonstatic_field_size: 0`: 非静态实例变量（成员变量）占用的内存大小，以字（word）为单位
- `_static_field_size: 0`: 静态变量大小为0
- `_static_oop_field_count: 0`: 静态引用类型变量数量为0
- `_nonstatic_oop_map_size: 0`: GC用的OopMap大小。用于GC Root扫描时快速定位对象内部的引用指针。
- `_is_marked_dependent: 0`: JIT编译器使用的标记位，用于标识该类是否有依赖它的JIT编译优化代码
- `_init_state: 4`: 类的初始化状态，4表示已经完成初始化，执行过了 `<clinit>`步骤
	- 0: allocated
	- 1: loaded
	- 2: linked
	- 3: being_initialized
	- 4: fully_initialized
- `_itable_len: 2`: 接口方法表长度为2，用于处理通过接口调用的方法分派

在菜单 `Tools --> Compute Reverse Ptrs`中可以根据对象实例地址找出引用它们的指针，如果使用这个菜单中的功能报错，可以使用 `Windows --> Console`来通过命令行方式查找，用到的命令： `revptrs 0x00000000ff6fdce8`，使用命令后输入的信息如下：

```
hsdb> revptrs 0x00000000ff6fdce8
Computing reverse pointers...
Done.
null
Oop for java/lang/Class @ 0x00000000ff6fc380
```

输出的 `0x00000000ff6fc380`就是引用 `0x00000000ff6fdce8`的地方，可以继续使用 `Tools --> Inspector`继续查看 `0x00000000ff6fc380`这个实例的信息：

```
Oop for java/lang/Class @ 0x00000000ff6fc380
	staticObj: Oop for fun/pullock/utj/c4_3_1_jhsdb/JHSDB$ObjectHolder @ 0x00000000ff6fdce8
```

- `Oop for java/lang/Class @ 0x00000000ff6fc380`: 这是一个Class类型的对象实例，位于堆地址 `0x00000000ff6fc380`。这个Class对象是 `fun.pullock.utj.c4_3_1_jhsdb.JHSDB$Test`类在Java堆中的镜像对象
- `staticObj: Oop for fun/pullock/utj/c4_3_1_jhsdb/JHSDB$ObjectHolder @ 0x00000000ff6fdce8`: 类中的静态字段 `staticObj`
	- 这是 `JHSDB$Test.class`对象内部持有的一个静态成员变量 `staticObj`
	- `staticObj`的引用被直接嵌入存放在 `Class`对象的实例内存的末尾，Java 7以后，静态变量从方法区挪到了堆中的 `Class`对象末尾


第二个对象 `revptrs 0x00000000ff6fdd08`，输出的信息：

```
hsdb> revptrs 0x00000000ff6fdd08
Computing reverse pointers...
Done.
Oop for fun/pullock/utj/c4_3_1_jhsdb/JHSDB$Test @ 0x00000000ff6fdcf8
```

输出的 `0x00000000ff6fdcf8`就是引用 `0x00000000ff6fdd08`的地方，使用 `Tools --> Inspector`查看 `0x00000000ff6fdcf8`实例的信息：

```
Oop for fun/pullock/utj/c4_3_1_jhsdb/JHSDB$Test @ 0x00000000ff6fdcf8
	<<Reverse pointers>>: 
	_mark: 1
	_metadata._compressed_klass: InstanceKlass for fun/pullock/utj/c4_3_1_jhsdb/JHSDB$Test
	instanceObj: Oop for fun/pullock/utj/c4_3_1_jhsdb/JHSDB$ObjectHolder @ 0x00000000ff6fdd08
```

这个是 `fun.pullock.utj.c4_3_1_jhsdb.JHSDB$Test`类的实例，该实例中持有一个 ObjectHolder`实例，名字就是代码中的 `instanceObj`。

第三个对象 `0x00000000ff6fdd18`是方法中创建的实例，使用 `revptrs 0x00000000ff6fdd18`没法查看栈上的引用信息，输出的信息如下：

```
hsdb> revptrs 0x00000000ff6fdd18
null
```

可以在 `Java Threads`选项卡中选中 `main`线程，点击 `Stack Memory`后会出现 `Stack Memory for main`选项卡，可以查看到地址 `0x00000000ff6fdd18`处对应的信息为 `NewGen fun/pullock/utj/c4_3_1_jhsdb/JHSDB$ObjectHolder`。


### 4.3.2 JConsole：Java监视与管理控制台

JConsole：Java Monitoring and Management Console，基于JMX（Java Management Extensions）的可视化监视、管理工具。通过JMX的MBean（Managed Bean）对系统进行信息收集和参数动态调整。

JConsole标签页：

- Overview: 整个虚拟机主要运行数据概览信息
	- Heap Memory Usage: 堆内存使用情况
		- Used: 实际已被使用的内存
		- Committed: 已提交/已申请内存，JVM当前向操作系统成功申请，保证可用的堆内存总量
		- Max: 最大内存，JVM能够向操作系统申请的堆内存上限
	- Threads: 线程统计
		- Live: 处于存活状态的线程总数
		- Peak: 峰值线程数，自JVM启动以来，存活线程数到达过的历史最高记录
		- Total: 累计线程数，自JVM启动以来，累计创建过的线程总数
	- Classes: 类加载统计
		- Loaded: 当前已加载类数
		- Unloaded: 已卸载类数
		- Total: 累计加载类数
	- CPU Usage: CPU使用情况
		- CPU Usage: JVM进程占用的CPU资源百分比
- Memory: 内存，相当于命令行的 `jstat`
	- Chart: 图标下拉菜单，选择内存池与内存区域
		- Heap Memory Usage: 堆内存使用情况，展示堆内存整体使用曲线，包括Eden、Survivor、Tenured Gen
		- Non-Heap Memory Usage: 非堆内存使用情况，展示非堆区域的总和，包括元空间、压缩类空间、所有CodeHeap内存池
		- Memory Pool "Eden Space": Eden区
		- Memory Pool "Survivor Space": Survivor区
		- Memory Pool "Tenured Gen": 老年代
		- Memory Pool "Metaspace": 元空间，使用本地内存存储类的结构信息，如字节码、常量池、字段和方法描述符等，取代类JDK 7以及之前版本的永久代
		- Memory Pool "CodeHeap 'profiled nmethods'": 带剖析信息的方法代码堆，存放包含分析数据的轻量级JIT编译方法。生命周期较短，会随着代码热度变化被替换或重新编译
		- Memory Pool "CodeHeap 'non-nmethods'": 非方法代码堆，存放非方法级别的JIT编译代码，比如JVM内部的Compiler Buffers、VTables、Adapter代码等，这部分会永久保留，不会发生GC
		- Memory Pool "Compressed Class Space": 压缩类空间，当开启指针压缩时，JVM会专门划出一块连续内存，用来存放 `InstanceKlass`的压缩Klass指针，这部分空间属于Metaspace的一部分
		- Memory Pool "CodeHeap 'non-profiled nmethods'": 无剖析信息的方法代码堆，存放高度优化的、五分析数据的JIT编译方法。生命周期长，追求最高执行性能。
	- Details: 详细数据
		- Heap: 堆内存
			- Time: 采样时间
			- Used: 堆内存已被实际使用大小
			- Committed: 已向操作系统申请的内存
			- Max: 能够申请的最大上限
			- GC time: 垃圾回收耗时和次数统计
				- {x} seconds on Copy ({y} collections): 年轻代收集器名称为Copy， 自JVM启动以来，累计进行了y次垃圾回收，共花费x秒
				- {x} seconds on MarkSweepCompact ({y} collections): 老年代收集器名称为MarkSweepCompact， 自JVM启动以来，累计进行了y次垃圾回收，共花费x秒
		- Non-Heap: 非堆内存统计
			- Time: 采样时间
			- Used: 堆内存已被实际使用大小
			- Committed: 已向操作系统申请的内存
			- GC time
				- {x} seconds on Copy ({y} collections): 年轻代收集器名称为Copy， 自JVM启动以来，累计进行了y次垃圾回收，共花费x秒
				- {x} seconds on MarkSweepCompact ({y} collections): 老年代收集器名称为MarkSweepCompact， 自JVM启动以来，累计进行了y次垃圾回收，共花费x秒
- Threads: 线程，相当于命令行的 `jstack`
	- Number of Threads: 线程数量统计图表
		- Peak: 峰值线程数
		- Live threads: 活动线程数
	- Threads: 线程列表与详情
		- Name: 线程名称
		- State: 线程状态
		- Total blocked: {x} Total waited: {y}: 累计阻塞x次，累计等待y次
		- Stack trace: 线程堆栈追踪
- Classes: 类加载信息
	- Number of Loaded Classes: 已加载类数量统计图表
		- Total Loaded: 累计加载数量
		- Loaded: 当前已加载数量
	- Details: 详细数据统计
		- Time: 采样时间
		- Current classes loaded: 当前已加载类数量
		- Total classes loaded: 累计已加载类数量
		- Total classes unloaded: 累计已卸载类数量
- VM Summary: 虚拟机概要
	- VM Summary
	- Connection name: 连接名称
	- Virtual Machine: 虚拟机名称与版本
	- Vendor: 供应商
	- Name: 进程名称/主类名
	- Uptime: 运行时间
	- Process CPU time: 进程的CPU耗时
	- JIT compiler: JIT编译器
	- Total compile time: 编译总耗时
	- Live threads: 活动线程数
	- Peak: 峰值线程数
	- Daemon threads: 守护线程数
	- Total threads started: 累计启动线程数
	- Current classes loaded: 当前已加载类数量
	- Total classes loaded: 累计加载类数量
	- Total classes unloaded: 累计卸载类数量
	- Current heap size: 当前堆大小
	- Maximum heap size: 最大堆内存
	- Garbage collector: 垃圾收集统计
		- Name='Copy': 年轻代垃圾收集器名称
		- Collections = {x}: 累计触发GC次数
		- Total time spent = {y} seconds: 累计GC时间
	- Garbage collector: 垃圾收集统计
		- Name = 'MarkSweepCompact': 老年代垃圾收集器名称
		- Collections = {x}: 累计触发GC次数
		- Total time spent = {y} seconds : 累计GC时间
	- Committed memory: 已向操作系统申请的内存
	- Pending finalization: 等待终结的对象数，也就是等待执行 `finalize()`方法的对象的数量
	- Operating System: 操作系统名称
	- Architecture: CPU架构
	- Number of processors: CPU核心数
	- Committed virtual memory: 已提交虚拟内存，操作系统保证为JVM进程提供的虚拟内存总量
	- Total physical memory: 物理内存总量
	- Free physical memory: 空闲物理内存
	- Total swap space: 交换空间大小
	- Free swap space: 空间交换空间大小
	- VM arguments: JVM启动参数
	- Class path: 类路径
	- Library path: 本地库路径
	- Boot class path: 引导类路径
- MBeans
	- JMImplementation: JMX基础设施实现，JMX框架本身的元数据，用于管理JMX容器的生命周期与事件
		- MBeanServerDelegate: 代表当前JVM内部的JMX代理服务器本身的实例
			- Attributes: 属性，展示当前MBean暴露的所有状态与变量数据
			- Notifications: 通知，用于接收并展示MBean产生的异步事件与告警通知
	- com.sun.management: Oracle/HotSpot专有扩展
		- DiagnosticCommand: HotSpot内部诊断命令的JMX映射接口，对应命令行工具 `jcmd`
			- Operations: 操作，展示当前MBean暴露的所有可执行方法，相当于JMX提供的远程RPC调用接口
			- Notifications
		- HotSpotDiagnostic: 专门用于控制HotSpot虚拟机诊断功能的MBean
			- Attributes
			- Operations
	- java.lang: 标准平台MXBeans
		- ClassLoading: 类加载系统监控
			- Attributes
		- Compilation: JIT编译器监控
			- Attributes
		- GarbageCollector: 垃圾收集器监控
			- Copy: 年轻代
				- Attributes
				- Notifications
			- MarkSweepCompact: 老年代
				- Attributes
				- Notifications
		- Memory: 内存管理总入口
			- Attributes
			- Operations
			- Notifications
		- MemoryManager: 内存管理器，负责管理一个或多个内存池
			- CodeCacheManager: 管理JIT代码缓存
				- Attributes
				- Notifications
			- Metaspace Manager: 管理元空间
				- Attributes
				- Notifications
		- MemoryPool: 内存池
			- CodeHeap 'non-nmethods': 非方法代码堆
				- Attributes
				- Operations
			- CodeHeap 'non-profiled nmethods': 不带剖析信息的方法代码堆
				- Attributes
				- Operations
			- CodeHeap 'profiled nmethods': 带剖析信息的代码堆
				- Attributes
				- Operations
			- Compressed Class Space: 压缩类空间
				- Attributes
				- Operations
			- Eden Space: Eden区
				- Attributes
				- Operations
			- Metaspace: 元空间
				- Attributes
				- Operations
			- Survivor Space: Survivor区
				- Attributes
				- Operations
			- Tenured Gen: 老年代
				- Attributes
				- Operations
		- OperatingSystem: 操作系统监控
			- Attributes
		- Runtime: JVM远行时环境配置
			- Attributes
		- Threading: 线程管理系统
			- Attributes
			- Operations
	- java.nio: NIO相关
		- BufferPool: 缓冲区池
			- direct: 直接内存
				- Attributes
			- mapped: 内存映射
				- Attributes
	- java.util.logging: 日志管理
		- Logging: 日志管理
			- Attributes
			- Operations

### 4.3.3 VisualVM：多合一故障处理工具

### 4.3.4 Java Mission Control：可持续在线的监控工具

## 4.4 HotSpot虚拟机插件及工具

- Ideal Graph Visualizer: 用于可视化展示C2即时编译器是如何将字节码转化为理想图，然后转化为机器码的。
- Client Compiler Visualizer: 用于查看C1即时编译器生成高级中间表（HIR），转换成低级中间表示（LIR）和做物理寄存器分配的过程。
- MakeDeps: 帮助处理HotSpot的编译依赖的工具。
- Project Creator: 帮忙生成Visual Studio的.project文件的工具。
- LogCompilation: 将-XX:+LogCompilation输出的日志整理成更容易阅读的格式的工具。
- HSDIS: 即时编译器的反汇编插件。