# Java7语法新特性

## switch 中使用字符串
在switch语句中，表达式的值不能是null，会在运行时抛出NullPointerException异常。

case子句中也不能使用null，编译会出错。

### 实现原理
这个新特性是在编译器层次上实现的。在Java虚拟机和字节码这个层次上，还是只支持在switch语句中使用一整数类型兼容的类型。

编译过程中，编译器会根据源代码将字符串类型转换成与整数类型兼容的格式。

使用字符串的哈希值作为switch语句的表达式的值。

### 枚举类型
代码中有多个地方使用switch语句来枚举字符串，考虑使用枚举类型进行替换。

## 数值字面量的改进

### 二进制整数字面量
前面加0b 或 0B

### 数值字面量中使用下划线

## 优化的异常处理
支持在一个catch子句中同时捕获多个异常。每个异常类型之间使用` | `来分隔。

编译器的做法其实是把捕获多个异常的catch子句转换成了多个catch子句。

在捕获并重新抛出异常时的异常类型更加精确。

## try-with-resources语句
能被try语句所管理的资源需要满足一个条件，那就是其Java类要实现java.lang.AutoCloseable接口。

## 优化变长参数的方法调用
@SafeVarargs只能用在参数长度可变的方法或构造方法上，且方法必须声明为static或final。用来确保这个方法中的实现中对泛型类型的参数处理不会引发类型安全问题。

# Java语言的动态性

## 脚本语言支持API

## 反射API
通过反射API可以获取Java程序在运行时刻的内部结构，比如Java类中包含的构造方法，域和方法等元素，兵可以与这些元素进行交互。

### 获取构造方法
getConstructors 获取所有的公开构造方法的列表。

getConstructor 根据参数类型来获取公开的构造方法。

getDeclaredConstructors和getDeclaredConstructor 获取类中真正声明的构造反复噶，忽略从父类继承下来的构造方法。


### 获取域
getFields,getField,getDeclaredFields,getDeclaredField

使用静态域时不需要提供具体的对象实例。

### 获取方法

### 操作数组

### 访问权限与异常处理
反射可以绕过java语言中默认的访问控制权限。

## 动态代理

## 动态语言支持

### 方法句柄
method handler ，通过方法句柄可以直接调用该句柄所引用的底层方法。

（跳过）
### invokedynamic指令

（跳过）

# Java I/O

## 流
### 基本输入流
java.io.InputStream和java.io.OutputStream。

read() 读取数据过程中，对read方法的调用是阻塞的。流中没有数据可用时，对read方法的调用需要等待。

close()方法关闭流，在java7中尽量通过try-with-resources语句来使用流。

### 基本输出流

write()方法写入数据。

flush()方法强制要求OutputStream类的对象对暂时保存在内部缓冲区中的内容立即进行实际的写操作。

### 过滤输入输出流

DataOutputStream,DataInputStream

ObjectOutputStream,ObjectInputStream

FileInputStream,FileOutputStream

ByteArrayInputStream,ByteArrayOutputStream

PipedInputStream,PipedOutputStream

### 字符流
Reader和Writer

字节流转字符流 InputStreamReader，OutputStreamWriter 需要制定字符的编码格式。

从String类对象中创建 StringReader，StringWriter

从字符数组中创建 CharArrayReader，CharArrayWriter

BufferedReader，BufferedWriter 提供内部缓冲区的支持。

## 缓冲区
容量capacity 创建缓冲区时指定，无法创建后修改。

读写限制limit 缓冲区中进行读写操作时的最大允许位置。

读写位置position 当前进行读写操作时的位置

clear方法 把limit设为capacity，position设为0。

flip方法 会把limit设为当前position，把position设置为0。

rewind position设为0。

### 字节缓冲区
ByteBuffer

直接缓冲区 和非直接缓冲区

### 缓冲区视图
视图和原ByteBuffer类对象共享同样的存储空间，但是提供额外的实用功能，在试图中对数据所做的修改会反应在原始的ByteBuffer类的的对象中。

slice

duplicate

asReadOnlyBuffer

## 通道

### 文件通道

FileChannel

使用文件通道之前首先需要获取到一个FileChannel类的对象。FileChannel类的对象以即可通过直接打开文件的方式来创建，也可以从已有的流中得到。

数据传输：transferFrom和transferTo

内存映射文件 FileChannel的map方法可以把文件映射到内存中。通过MappedByteBuffer类的load方法可以把该缓冲区所对应的文件内容加载到物理内存中。

force方法强制要求把更新同步到文件中。

FileChannel类的lock和tryLock方法可以对当前文件通道所对应的文件进行加锁。可以选择锁定文件的全部内容，也可以锁定指定范围区间中的部分内容。

lock阻塞，tryLock不阻塞。成功加锁后得到一个FileLock对象。release方法可以解除锁定。

isShared为true表示是一个共享锁，否则是排他锁。


### 套接字通道
JavaNIO提供了非阻塞式和多路复用的套接字连接。

NetworkChannel表示一个套接字所对应的通道。

SocketChannel和ServerSocketChannel都支持阻塞和费阻塞两种模式。

费阻塞式的套接字通道实现多路复用，或者使用NIO.2中的异步套接字通道。

套接字通道的多路复用的思想比较简单，通过一个专门的selector来同时对多个套接字通道进行监听。基于选择器的做法可以同时管理多个套接字通道，可用通道的选择一般是通过操作系统提供的底层系统调用来实现的，性能较高。

多路复用实现核心是选择器Selector类的对象，非阻塞式的套接字通道可以通过register方法注册到某个Selector类的对象上，需要提供感兴趣的事件。

Selector的open方法可以创建一个新的选择器，有了选择器之后可以把套接字通道注册到选择器上。注册时需要指定套接字感兴趣的事件。

套接字通道注册完成之后，下一步就是调用Selector类的对象的select方法来进行通道选择操作。

## NIO.2

### 文件系统访问
Path接口作为文件系统中路径的一个抽象。

getRoot 获取根 

getNameCount 获取名称元素的总数

getName 获取单个名称元素

getFileName 获取文件名

getParent 获取当前路径的父路径

1. 文件目录列表流 

DirectoryStream。

2. 文件目录树遍历 

FileVisitor

3. 文件属性

AttributeView是所有属性视图的父接口。

FileAttributeView表示的是文件的属性视图。

4. 监视目录变化

NIO.2提供了新的目录监视服务，使用该服务可以在指定目录中的子目录或文件被创建，更改或删除时得到事件通知。

被监事的对象要实现Watchable接口。并通过register方法注册到表示监视服务的WatchService接口的实现对象上，注册是需要指定被监视对象感兴趣的事件类型。

5. 文件操作的实用方法

### zip/jar文件系统

### 异步IO通道
异步通道一般提供两种使用方式：一种是通过Java同步工具包中的Future类的对象来表示异步操作的结果。另外一种是在执行操作时传入一个CompletionHandler接口的实现对象作为操作完成时的回调方法。

异步文件通道AsynchronousFileChannel

### 套接字通道绑定与配置
NetworkChannel

### IP组播通道

# 国际化与本地化
（过）

# Java7其他重要更新

## 关系数据库访问
### 数据库查询的默认模式
getSchema和setSchema 用来获取和设置数据库操作时是用的默认模式名称。通过设置后，在sql语句中酒不在需要使用模式名称作为前缀了。

### 数据库连接超时时间与终止

### 语句自动关闭

# Java虚拟机

## 内存管理

悬挂引用 指的是对摸个对象的引用实际上指向一个错误的内存地址。

内存泄漏 某些对象所占用的内存没有被释放，又没有对象引用指向这些内存。

## 引用类型
### 强引用

最常见的引用类型，也是默认的类型。

### 引用类型基本概念

* 软引用 所指向的对象是可以在需要的时候被回收的。
* 弱引用 在判断一个对象是否存活时，可以不考虑弱引用的存在。
* 幽灵引用 

### 引用队列

## Java本地接口

### JNI
原生方法 使用native关键字来声明。

## HotSpot虚拟机
JIT编译

### 垃圾回收
分代回收

一般划分为年轻代，老年代，永久代。

永久代中包含的是Java虚拟机自身运行所需的对象。

年轻代划分为eden和两个survivor空间。

内存较大的对象直接在老年代中进行分配。

对于老年代和永久代通常采用标记-清除-压缩算法。

虚拟机对字符串的内部化处理，有可能会造成永久代的内存不足。

### 启动参数
`-classpath`和`-cp`可以指定程序运行时的类路径。

`-D`用来设置系统属性的值。

`-ea`和`-da`用来启用和禁用程序中的断言。

`-esa`和`-dsa`启用和禁用系统类中的断言。

`-verbose`输出详细信息，如`-verbose:class`输出加载Java类时的信息，`-verbose:gc`输出垃圾回收器运行时的信息，`-verbose:jni`用来使用原生方法时输出相关信息。

启动参数中以`-X`开头的参数是非标准参数，不是所有的平台上的虚拟机都支持。

`-Xms` 堆内存的初始值

`-Xmx` 堆内存的最大值

以`-XX`开头的参数实现不是很稳定，一般不推荐使用

`-XX:+<option>` 打开

`-XX:-<option>` 关闭

`-XX:<option>=<number>` 数值类型

`-XX:<option>=<string>` 字符串类型

### 分析工具

1. 命令行工具

jmap

jhat

jps

jstack

jinfo

jstat


2. 图形化工具

jconsole

Java VisualVM

3. JMX

### Java虚拟机工具接口
JVM TI

# Java源代码和字节代码操作
## Java字节代码格式

### 常量池的结构
cp_info

# Java类加载器

## OSGI中的类加载器

# 对象生命周期

## Java类的链接
Java类被加载后经过链接和初始化就可以在虚拟机中使用了。

链接过程分为 验证，准备，解析。解析是可选的。

验证 保证字节码在结构上是正确的。

准备 会创建Java类的静态域，并设置为默认值。保证类加载时的类型安全。

解析 处理所加载的Java类中包含的形式引用。

## Java类的初始化
类第一次被使用时，虚拟机会对该类进行初始化。

主要工作是执行Java类中的静态代码块和初始化静态域。

对于静态代码块和静态域，虚拟机严格按照在源代码中的出现顺序执行初始化操作。

可能造成类被初始化的操作：

* 创建一个Java类的对象，new 一个Class
* 调用一个Java类中的静态方法
* 为类活接口中的静态域赋值
* 访问类或接口中声明的静态域，并且该域的值不是final类型的或者String类型的
* 在一个顶层Java类中执行assert语句也会使该Java类初始化
* 调用Class类和反射API进行反射操作的方法也会初始化Java类

## 对象的创建与初始化
类的构造方法的调用分为三步：

1. 调用父类的构造方法
2. 初始化类中的实例域的值
3. 执行类的构造方法中的其他代码

## 对象终止

## 对象复制
clone

Cloneable

Object的clone方法 为原始对象创建了一个浅拷贝。

## 对象序列化

# 多线程与并发编程实践
## 多线程
继承Thread类并覆写run方法，或者实现Runnable接口。

### 可见性

### Java内存模型

### volatile
对共享变量的访问进行同步。

### final
声明一个域的值只能被初始化一次，之后无法修改。不可变对象。

### 原子操作
long，double 需要声明为volatile，保证读取和写入操作的完整性。

## 基本线程同步方式
### synchronized
静态方法对应的监视器对象是所在Java类对应的Class类的对象所关联的监视器对象。

实例方法使用的是当前对象实例所关联的监视器对象。

synchronized代码块，对应的监视器对象是synchronized代码块声明中的对象所关联的监视器对象。

### Object的wait，notify，notifyAll
