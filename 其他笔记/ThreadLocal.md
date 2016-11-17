# ThreadLocal
Java中的ThreadLocal类允许我们创建只能被同一个线程读写的变量，如果一段代码含有一个ThreadLocal变量的引用，即使两个线程同时执行这段代码，它们也无法访问到对方的ThreadLocal变量。

## 创建ThreadLocal变量

```
private ThreadLocal myThreadLocal = new ThreadLocal();
```

只需要实例化对象一次，并且也不要知道它是被哪个线程实例化。虽然所有线程都能访问到这个ThreadLocal实例，但是每个线程只能访问到自己通过调用ThreadLocal的set()方法设置的值。即使是两个不同的线程在同一个ThreadLocal对象上设置了不同的值，它们仍然无法访问到对方的值。

## 访问ThreadLocal变量
设置需要保存的值：
`myThreadLocal.set("ThreadLocal value");`

读取保存在ThreadLocal变量中的值：
`String threadLocalVlaue = (String) myThreadLocal.get();`

get()方法返回一个Object对象，set对象需要传入一个Object类型的参数。

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

# 文章链接
[http://ifeve.com/java-threadlocal%E7%9A%84%E4%BD%BF%E7%94%A8/](http://ifeve.com/java-threadlocal%E7%9A%84%E4%BD%BF%E7%94%A8/)

[https://my.oschina.net/lichhao/blog/111362](https://my.oschina.net/lichhao/blog/111362)