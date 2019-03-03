Spring中事件机制是设计模式中观察者模式的应用，如果理解了观察者模式再看Spring中事件机制，其实并不难。这里就spring的事件相关源码进行分析下，以便加深下观察者模式的理解。有关观察者模式不做过多介绍，网上有很多文档可供参考。

# 观察者模式和Spring事件机制

我们先看下观察者模式的几个定义：

- Subject：抽象主题，一般会把所有观察者对象保存到一个列表中，这样就能知道将通知发送给谁了。
- ConcreteSubject：具体主题，具体主题内部状态发生改变的时候，会给保存在列表中的观察者发送通知。
- Observer：抽象观察者，是所有具体观察者的接口。
- ConcreteObserver：具体观察者，也就是可以接收并处理主题发送的通知的地方。

总结一下就是：具体观察者（ConcreteObserver）将自己注册到抽象主题（Subject）中去，具体主题（ConcreteSubject）状态改变后发送通知将这个变化发送给所有已经注册的具体观察者（ConcreteObserver）那里去。

再看看Spring中的几个定义：

- ApplicationEventPublisher：字面理解为应用事件发布者，就是它可以发布应用事件，是一个接口。作用等同于观察者模式的抽象主题，它的实现类就相当于具体主题。
- ApplicationEvent：应用事件，也就是被发布者发布的东西，相当于观察者模式中一个隐藏的概念：状态改变。
- ApplicationListener：应用监听器、监听者，是一个接口，相当于观察者模式中的抽象观察者，它的实现类相当于具体观察者。
- ApplicationEventMulticaster：应用事件广播器。这个角色好像在观察者模式中不存在，但是仔细想想后，发现它也可以是观察者模式中的抽象主题，又或者说ApplicationEventPublisher拿着这个广播器来进行发布事件通知。将publisher和multicaster组装到一起才是观察者模式中的抽象主题。

其实稍微想一下后发现，其实就是观察者模式：具体应用监听器（ApplicationListener）将自己注册到一个应用事件广播器（ApplicationEventMulticaster）中，具体应用事件发布者（ApplicationEventPublisher）拿着这个广播器（ApplicationEventMulticaster）将发生的事件（ApplicationEvent）发送给所有注册的具体应用监听器（ApplicationListener）。

相对于设计模式中的观察者模式，增加了几个概念。在设计模式中主题和观察者其实是耦合在一起的，主题直接调用观察者的方法去做通知。而在Spring中应用事件发布者是通过一个广播器将事件发送给监听器的，这样就解耦了，可以使用不同的广播器去发送事件，实现各种不同功能。

接下来就看看Spring中是怎么对观察者模式进行具体的实现的。

# 初始化应用事件广播器

我们在Spring容器的启动过程中，也就是在AbstractApplicationContext的`refresh()`方法中可以看到有一步是`initApplicationEventMulticaster()`，初始化应用广播器，我们就先从这里开始看起：

```java
protected void initApplicationEventMulticaster() {
    // 拿到当前的BeanFactory
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    /**
		 * 如果说Spring的配置文件中存在名字为applicationEventMulticaster的Bean，
		 * 也就是说我们自己定义了一个事件广播器，就会使用我们自己定义的。
		 * 配置文件中不存在自定义的广播器，就使用默认的SimpleApplicationEventMulticaster。
		 */
    if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
        this.applicationEventMulticaster =
            beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
        if (logger.isDebugEnabled()) {
            logger.debug("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
        }
    }
    else {
        this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
        beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
        if (logger.isDebugEnabled()) {
            logger.debug("Unable to locate ApplicationEventMulticaster with name '" +
                         APPLICATION_EVENT_MULTICASTER_BEAN_NAME +
                         "': using default [" + this.applicationEventMulticaster + "]");
        }
    }
}
```

这里的逻辑很简单，就是使用自定义或者默认的广播器进行实例化。现在我们已经有了广播器了，接下来应该就是要将监听器注册到广播器中去了，回到`refresh()`方法中找到`registerListeners()`方法，这里就是注册监听器到广播器的地方。

# 注册监听器到广播器中

直接看`registerListeners()`方法：

```java
protected void registerListeners() {
    // Register statically specified listeners first.
    /**
		 * 先注册硬编码方式的监听器
		 */
    for (ApplicationListener<?> listener : getApplicationListeners()) {
        getApplicationEventMulticaster().addApplicationListener(listener);
    }

    // Do not initialize FactoryBeans here: We need to leave all regular beans
    // uninitialized to let post-processors apply to them!
    // 我们自己的监听器
    String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
    for (String listenerBeanName : listenerBeanNames) {
        /**
			 * 先获取广播器，之前的步骤初始化广播器已经实例化过了
			 * 然后将这些监听器添加到广播器中
			 */
        getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
    }
}
```

广播器有了，监听器也有了，剩下的就是要有一个地方去发送一些事件了，也就是ApplicationEventPublisher去发布事件，AbstractApplicationContext实现了ApplicationEventPublisher接口，所以它是个可以发送事件的发布者。如果我们自己试了ApplicationEventPublisher接口，就需要我们自己去发布事件了。

# 发布事件

我们可以看下`finishRefresh()`方法中有调用`publishEvent()`方法进行容器刷新完成事件发布：`publishEvent(new ContextRefreshedEvent(this));`。

我们看下发布事件的代码：

```java
public void publishEvent(ApplicationEvent event) {
    Assert.notNull(event, "Event must not be null");
    if (logger.isTraceEnabled()) {
        logger.trace("Publishing event in " + getDisplayName() + ": " + event);
    }
    // 获取应用事件广播器，广播事件
    getApplicationEventMulticaster().multicastEvent(event);
    // 父上下文也广播事件
    if (this.parent != null) {
        this.parent.publishEvent(event);
    }
}
```

先获取广播器，我们已经有了，然后广播事件，这里默认是SimpleApplicationEventMulticaster：

```java
public void multicastEvent(final ApplicationEvent event) {
    for (final ApplicationListener listener : getApplicationListeners(event)) {
        Executor executor = getTaskExecutor();
        if (executor != null) {
            executor.execute(new Runnable() {
                public void run() {
                    listener.onApplicationEvent(event);
                }
            });
        }
        else {
            listener.onApplicationEvent(event);
        }
    }
}
```

可以看到这里实现获取所有的应用监听器，然后调用`onApplicationEvent()`方法执行我们的逻辑。获取所有应用监听器也不太复杂，在bean初始化的时候，如果是ApplicationListener的实现类，就会添加到一个Set中保存起来。

# 监听器添加到广播器中

在AbstractApplicationContext中有这样一个内部类：

```java
private class ApplicationListenerDetector implements MergedBeanDefinitionPostProcessor {

    private final Map<String, Boolean> singletonNames = new ConcurrentHashMap<String, Boolean>(64);

    public void postProcessMergedBeanDefinition(RootBeanDefinition beanDefinition, Class<?> beanType, String beanName) {
        if (beanDefinition.isSingleton()) {
            this.singletonNames.put(beanName, Boolean.TRUE);
        }
    }

    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        return bean;
    }

    public Object postProcessAfterInitialization(Object bean, String beanName) {
        if (bean instanceof ApplicationListener) {
            // potentially not detected as a listener by getBeanNamesForType retrieval
            Boolean flag = this.singletonNames.get(beanName);
            if (Boolean.TRUE.equals(flag)) {
                // singleton bean (top-level or inner): register on the fly
                addApplicationListener((ApplicationListener<?>) bean);
            }
            else if (flag == null) {
                if (logger.isWarnEnabled() && !containsBean(beanName)) {
                    // inner bean with other scope - can't reliably process events
                    logger.warn("Inner bean '" + beanName + "' implements ApplicationListener interface " +
                                "but is not reachable for event multicasting by its containing ApplicationContext " +
                                "because it does not have singleton scope. Only top-level listener beans are allowed " +
                                "to be of non-singleton scope.");
                }
                this.singletonNames.put(beanName, Boolean.FALSE);
            }
        }
        return bean;
    }
}
```

我们根据Sprnig的扩展点汇总中[点击查看这篇文章](https://cxis.me/2019/02/22/Spring%E4%B8%AD%E6%89%A9%E5%B1%95%E7%82%B9%E6%B1%87%E6%80%BB/)可以知道在Bean实例化后会调用实现了MergedBeanDefinitionPostProcessor的`postProcessAfterInitialization`方法，也就是上面的内部类的方法，这里面逻辑很清楚，如果是监听器，就添加到广播其中去。

到此就完了！