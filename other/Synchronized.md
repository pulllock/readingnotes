# Synchronized

## 同步的基础
Java中的每一个对象都可以作为锁。

* 对于同步方法，锁是当前实例对象。
* 对于静态同步方法，锁是当前对象的Class对象。
* 对于同步方法块，锁是synchronized括号里配置的对象。

## 同步的原理
JVM规范规定JVM基于进入和退出Monitor对象来实现方法同步和代码块同步，但两者的实现细节不一样。代码块同步是使用monitorenter和monitorexit指令实现，而方法同步是使用另外一种方式实现的，细节在JVM规范里并没有详细说明，但是方法的同步同样可以使用这两个指令来实现。monitorenter指令是在编译后插入到同步代码块的开始位置，而monitorexit是插入到方法结束处和异常处， JVM要保证每个monitorenter必须有对应的monitorexit与之配对。任何对象都有一个 monitor 与之关联，当且一个monitor 被持有后，它将处于锁定状态。线程执行到 monitorenter 指令时，将会尝试获取对象所对应的 monitor 的所有权，即尝试获得对象的锁。



# 文章链接
[http://www.infoq.com/cn/articles/java-se-16-synchronized](http://www.infoq.com/cn/articles/java-se-16-synchronized)


[http://www.cnblogs.com/GnagWang/archive/2011/02/27/1966606.html](http://www.cnblogs.com/GnagWang/archive/2011/02/27/1966606.html)