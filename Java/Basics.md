# 普通方法、匿名内部类、Lambda

普通方法，比如：

```java
public class Calculator {

    public int add(int a, int b) {
        return a + b;
    }
}
```

这段代码中的 `add()`就是一个普通方法，调用方式如下：

```java
Calculator calculator = new Calculator();

int result = calculator.add(1, 2);
```

特点：
- 有名字： `add`
- 属于某个类： `Calculator`
- 不能独立存在： `add`不能作为一个值存在，比如 `Calculator.add`是不行的

在最初的 `Java`中对象是一等公民，方法不是一等公民。一等公民可以像普通的变量一样进行赋值、传参、返回、存储，比如 `String name = "Tom";`中 `String`是一等公民。但是 `add()`方法不是一等公民， 因此以下操作是不能的：

- 不能赋值给变量： `var x = add;`
- 不能当作参数进行传参： `foo(add);`
- 不能被返回： `return add;`

在很多的时候，需要传递一段行为，而不是传递对象。比如一个按钮的点击： `button.onClick(...)`，到底要执行什么需要由用户来确定，这时候方法中更需要传递的是一段行为。

在 `Java 1.1`中引入了匿名内部类，把行为包装成对象，比如：

```java
public interface Runnable {

    void run();
}
```

```java
public class Main {
    public static void main() {
        Runnable task = new Runnable() {
            @Override
            public void run() {
                System.out.println("Hello");
            }
        };
    }
}

```

匿名内部类实际上会由编译器进行处理，会生成额外的字节码文件，内容如下：

```java
class Main$1 implements Runnable {

    @Override
    public void run() {
        System.out.println("Hello");
    }
}
```

使用：

```java
Runnable task = new Main$1();
```

匿名内部类的代码，就相当于把 `run()`这个行为包装成 `Runnable`对象，使用的时候 `execute(task);`这样就行了。

匿名内部类使用比较啰嗦，真正的业务逻辑实际上是 `run()`方法内部的代码，但是外部固定的模板代码确占了很多。

`Java 8`引入了 `Lambda`，上面同样的逻辑变成了：

```java
public class Main {
    public static void main() {
        Runnable task = () -> System.out.println("Hello");
    }
}
```

形式更简洁了。

`Lambda`表达式主要依赖 `Java 7`引入的 `inokedynamic`指令，在运行时动态生成字节码。

# JHSDB使用及错误解决

# 不同版本JVM使用方式


- Java 8: `/usr/lib/jvm/java-8-openjdk/bin/java -cp /usr/lib/jvm/java-8-openjdk/lib/sa-jdi.jar sun.jvm.hotspot.HSDB`
- Java 9以及以后: `jhsdb hsdb`

## 错误1

错误信息：

```
ERROR: ptrace(PTRACE_ATTACH, ..) failed for 20454: Operation not permitted
```

解决方案：

- 临时设置: `sudo sysctl -w kernel.yama.ptrace_scope=0`
## 错误2

错误信息：

```
Exception in thread "main" java.lang.RuntimeException: Type "GrowableArrayBase", referenced in VMStructs::localHotSpotVMStructs in the remote VM, was not present in the remote VMStructs::localHotSpotVMTypes table (should have been caught in the debug build of that VM). Can not continue.
	at jdk.hotspot.agent/sun.jvm.hotspot.HotSpotTypeDataBase.lookupOrFail(HotSpotTypeDataBase.java:595)
	at jdk.hotspot.agent/sun.jvm.hotspot.HotSpotTypeDataBase.lookupType(HotSpotTypeDataBase.java:120)
	at jdk.hotspot.agent/sun.jvm.hotspot.HotSpotTypeDataBase.lookupOrCreateClass(HotSpotTypeDataBase.java:630)
	at jdk.hotspot.agent/sun.jvm.hotspot.HotSpotTypeDataBase.createType(HotSpotTypeDataBase.java:743)
	at jdk.hotspot.agent/sun.jvm.hotspot.HotSpotTypeDataBase.readVMTypes(HotSpotTypeDataBase.java:195)
	at jdk.hotspot.agent/sun.jvm.hotspot.HotSpotTypeDataBase.<init>(HotSpotTypeDataBase.java:89)
	at jdk.hotspot.agent/sun.jvm.hotspot.HotSpotAgent.setupVM(HotSpotAgent.java:418)
	at jdk.hotspot.agent/sun.jvm.hotspot.HotSpotAgent.go(HotSpotAgent.java:345)
	at jdk.hotspot.agent/sun.jvm.hotspot.HotSpotAgent.attach(HotSpotAgent.java:142)
	at jdk.hotspot.agent/sun.jvm.hotspot.HSDB.attach(HSDB.java:1212)
	at jdk.hotspot.agent/sun.jvm.hotspot.HSDB.run(HSDB.java:450)
	at jdk.hotspot.agent/sun.jvm.hotspot.HSDB.main(HSDB.java:59)
	at jdk.hotspot.agent/sun.jvm.hotspot.SALauncher.runHSDB(SALauncher.java:294)
	at jdk.hotspot.agent/sun.jvm.hotspot.SALauncher.main(SALauncher.java:507)
```

解决方案： 程序运行的虚拟机和运行 `jhsdb`命令的虚拟机要一致