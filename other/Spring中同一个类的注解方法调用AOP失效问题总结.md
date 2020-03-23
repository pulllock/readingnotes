Spring中在同一个类中两个方法调用，会导致代理失效，比如同一个类中一个方法A调用另外一个带事务的方法B，会发现B方法的事务不生效；同一个类中一个方法A调用另外一个带注解的清除缓存的方法B，会发现清除缓存不成功。

> Spring中仅支持方法级别的代理。

Spring中代理是动态代理，也就是在运行时生成代理，代理可以大概使用以下代码来描述下：

```java
@Service
public class A {
    
    public void a() {
        ...;
        b();
        ...;
    }
    
    @Transactional
    public void b() {
        ...;   
    }
}
```

而在运行时生成的代理如下：

```java
public class A$Proxy {
    A serviceA = new A();
    
    public void a() {
        ...;
        serviceA.b();
        ...;
    }
    
    public void b() {
        transaction start();
        serviceA.b();
        transaction end();
    }
}
```

