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




