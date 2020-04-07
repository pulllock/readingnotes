MyBatis的DefaultSqlSession使用具体Executor的时候，用的是策略模式。

策略模式是定义一系列算法，并将算法封装起来，使用的时候算法可以互相替换。策略模式的角色：

- 抽象策略（Strategy），一般是接口或者抽象类，定义算法。
- 具体策略（ConcreteStrategy），具体的抽象策略实现类。
- 上下文（Context），持有策略Strategy对象，并且有一个能改变具体策略实现类的方法，来动态改变所向持有的策略。

MyBatis中的DefaultSqlSessionFactory在创建SqlSession的时候，会根据不同的ExecutorType来创建不同的Executor对象，并将这些对象给具体的SqlSession使用。MyBatis中策略模式的角色：

- 抽象策略：Executor
- 具体策略：SimpleExecutor、ReuseExecutor、BatchExecutor、CachingExecutor
- 上下文：DefaultSqlSession

简要代码：

```java
public interface Executor {
    // ...
}
```

```java
public class SimpleExecutor extends BaseExecutor {
    // ...
}
```

```java
public class DefaultSqlSession implements SqlSession {

  private final Executor executor;

  public DefaultSqlSession(Configuration configuration, Executor executor, boolean autoCommit) {
    this.executor = executor;
  }
    // ...
}
```

