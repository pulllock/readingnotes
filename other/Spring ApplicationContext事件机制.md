ApplicationContext中事件处理是由ApplicationEvent类和ApplicationListener接口来提供的。如果一个Bean实现了ApplicationListener接口，并且已经发布到容器中去，每次ApplicationContext发布一个ApplicationEvent事件，这个Bean就会接到通知。Spring事件机制是观察者模式的实现。

# Spring中提供的标准事件：

- ContextRefreshEvent，当ApplicationContext容器初始化完成或者被刷新的时候，就会发布该事件。比如调用ConfigurableApplicationContext接口中的refresh()方法。此处的容器初始化指的是所有的Bean都被成功装载，后处理（post-processor）Bean被检测到并且激活，所有单例Bean都被预实例化，ApplicationContext容器已经可以使用。只要上下文没有被关闭，刷新可以被多次触发。XMLWebApplicationContext支持热刷新，GenericApplicationContext不支持热刷新。

- ContextStartedEvent，当ApplicationContext启动的时候发布事件，即调用ConfigurableApplicationContext接口的start方法的时候。这里的启动是指，所有的被容器管理生命周期的Bean接受到一个明确的启动信号。在经常需要停止后重新启动的场合比较适用。

- ContextStoppedEvent，当ApplicationContext容器停止的时候发布事件，即调用ConfigurableApplicationContext的close方法的时候。这里的停止是指，所有被容器管理生命周期的Bean接到一个明确的停止信号。

- ContextClosedEvent，当ApplicationContext关闭的时候发布事件，即调用ConfigurableApplicationContext的close方法的时候，关闭指的是所有的单例Bean都被销毁。关闭上下后，不能重新刷新或者重新启动。

- RequestHandledEvent，只能用于DispatcherServlet的web应用，Spring处理用户请求结束后，系统会触发该事件。

# 实现

ApplicationEvent，容器事件，必须被ApplicationContext发布。

ApplicationListener，监听器，可由容器中任何监听器Bean担任。

实现了ApplicationListener接口之后，需要实现方法onApplicationEvent()，在容器将所有的Bean都初始化完成之后，就会执行该方法。

# 观察者模式
观察者模式，Observer Pattern也叫作发布订阅模式Publish/Subscribe。定义对象间一对多的依赖关系，使得每当一个对象改变状态，则所有依赖与它的对象都会得到通知，并被自动更新。

观察者模式的几角色名称：

- Subject被观察者，定义被观察者必须实现的职责，它能动态的增加取消观察者，它一般是抽象类或者是实现类，仅仅完成作为被观察者必须实现的职责：管理观察者并通知观察者。
- Observer观察者，观察者接受到消息后，即进行更新操作，对接收到的信息进行处理。
- ConcreteSubject具体的被观察者，定义被观察者自己的业务逻辑，同时定义对哪些事件进行通知。
- ConcreteObserver具体的观察者，每个观察者接收到消息后的处理反应是不同的，每个观察者都有自己的处理逻辑。

## 观察者模式的优点

- 观察者和被观察者之间是抽象耦合，不管是增加观察者还是被观察者都非常容易扩展。
- 建立一套触发机制。

## 观察者模式的缺点
观察者模式需要考虑开发效率和运行效率问题，一个被观察者，多个观察者，开发和调试比较复杂，Java消息的通知默认是顺序执行的，一个观察者卡壳，会影响整体的执行效率。这种情况一般考虑异步的方式。

## 使用场景

- 关联行为场景，关联是可拆分的。
- 事件多级触发场景。
- 跨系统的消息交换场景，如消息队列的处理机制。

## Java中的观察者模式
java.util.Observable类和java.util.Observer接口。

## 订阅发布模型
观察者模式也叫作发布/订阅模式。

# Spring中的观察者模式
Spring在事件处理机制中使用了观察者模式：

- 事件，ApplicationEvent，该抽象类继承了EventObject，EventObject是JDK中的类，并建议所有的事件都应该继承自EventObject。
- 事件监听器，ApplicationListener，是一个接口，该接口继承了EventListener接口。EventListener接口是JDK中的，建议所有的事件监听器都应该继承EventListener。
- 事件发布，ApplicationEventPublisher，ApplicationContext继承了该接口，在ApplicationContext的抽象实现类AbstractApplicationContext中做了实现

AbstractApplicationContext类中publishEvent方法实现：

```
public void publishEvent(ApplicationEvent event) {
	Assert.notNull(event, "Event must not be null");
	if (logger.isTraceEnabled()) {
		logger.trace("Publishing event in " + getDisplayName() + ": " + event);
	}
	//事件发布委托给ApplicationEventMulticaster来执行
	getApplicationEventMulticaster().multicastEvent(event);
	if (this.parent != null) {
		this.parent.publishEvent(event);
	}
}
```

ApplicationEventMulticaster的multicastEvent方法的实现在SimpleApplicationEventMulticaster类中：

```
public void multicastEvent(final ApplicationEvent event) {
	//获得监听器集合，遍历监听器，可支持同步和异步的广播事件
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
这就执行了了onApplicationEvent方法，这里是事件发生的地方。

## Spring如何根据事件找到事件对应的监听器
在Spring容器初始化的时候，也就是在refresh方法中：

```
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		......
		try {
			......
			// Initialize event multicaster for this context.
			//初始化一个事件注册表
			initApplicationEventMulticaster();
			......
			// Check for listener beans and register them.
			//注册事件监听器
			registerListeners();

			......
		}
	}
}
```

initApplicationEventMulticaster方法初始化事件注册表：

```
protected void initApplicationEventMulticaster() {
	//获得beanFactory
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	//先查找BeanFactory中是否有ApplicationEventMulticaster
	if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
		this.applicationEventMulticaster =
				beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
	}
	else {//如果BeanFactory中不存在，就创建一个SimpleApplicationEventMulticaster
		this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
		beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
	}
}
```

在AbstractApplicationEventMulticaster类中有如下属性：

```
//注册表
private final ListenerRetriever defaultRetriever = new ListenerRetriever(false);
//注册表的缓存
private final Map<ListenerCacheKey, ListenerRetriever> retrieverCache = new ConcurrentHashMap<ListenerCacheKey, ListenerRetriever>(64);

private BeanFactory beanFactory;
```

ListenerRetriever的结构如下：

```
//用来存放监听事件
public final Set<ApplicationListener> applicationListeners;
//存放监听事件的类名称
public final Set<String> applicationListenerBeans;

private final boolean preFiltered;
```

初始化注册表之后，就会把事件注册到注册表中，registerListeners()：

```
protected void registerListeners() {
	//获取所有的Listener，把事件的bean放到ApplicationEventMulticaster中
	for (ApplicationListener<?> listener : getApplicationListeners()) {
		getApplicationEventMulticaster().addApplicationListener(listener);
	}
	// Do not initialize FactoryBeans here: We need to leave all regular beans
	// uninitialized to let post-processors apply to them!
	String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
	//把事件的名称放到ApplicationListenerBean里去。
	for (String lisName : listenerBeanNames) {
		getApplicationEventMulticaster().addApplicationListenerBean(lisName);
	}
}
```
Spring使用反射机制，通过方法getBeansOfType获取所有继承了ApplicationListener接口的监听器，然后把监听器放到注册表中，所以我们可以在Spring配置文件中配置自定义监听器，在Spring初始化的时候，会把监听器自动注册到注册表中去。

ApplicationContext发布事件可以参考上面的内容。发布事件的时候的一个方法，getApplicationListeners：

```
protected Collection<ApplicationListener> getApplicationListeners(ApplicationEvent event) {
	//获取事件类型
	Class<? extends ApplicationEvent> eventType = event.getClass();
	//或去事件源类型
	Class sourceType = event.getSource().getClass();
	ListenerCacheKey cacheKey = new ListenerCacheKey(eventType, sourceType);
	//从缓存中查找ListenerRetriever
	ListenerRetriever retriever = this.retrieverCache.get(cacheKey);
	//缓存中存在，直接返回对应的Listener
	if (retriever != null) {
		return retriever.getApplicationListeners();
	}
	else {//缓存中不存在，就获取相应的Listener
		retriever = new ListenerRetriever(true);
		LinkedList<ApplicationListener> allListeners = new LinkedList<ApplicationListener>();
		Set<ApplicationListener> listeners;
		Set<String> listenerBeans;
		synchronized (this.defaultRetriever) {
			listeners = new LinkedHashSet<ApplicationListener>(this.defaultRetriever.applicationListeners);
			listenerBeans = new LinkedHashSet<String>(this.defaultRetriever.applicationListenerBeans);
		}
		//根据事件类型，事件源类型，获取所需要的监听事件
		for (ApplicationListener listener : listeners) {
			if (supportsEvent(listener, eventType, sourceType)) {
				retriever.applicationListeners.add(listener);
				allListeners.add(listener);
			}
		}
		if (!listenerBeans.isEmpty()) {
			BeanFactory beanFactory = getBeanFactory();
			for (String listenerBeanName : listenerBeans) {
				ApplicationListener listener = beanFactory.getBean(listenerBeanName, ApplicationListener.class);
				if (!allListeners.contains(listener) && supportsEvent(listener, eventType, sourceType)) {
					retriever.applicationListenerBeans.add(listenerBeanName);
					allListeners.add(listener);
				}
			}
		}
		OrderComparator.sort(allListeners);
		this.retrieverCache.put(cacheKey, retriever);
		return allListeners;
	}
}
```

根据事件类型，事件源类型获取所需要的监听器supportsEvent(listener, eventType, sourceType)：

```
protected boolean supportsEvent(
		ApplicationListener listener, Class<? extends ApplicationEvent> eventType, Class sourceType) {

	SmartApplicationListener smartListener = (listener instanceof SmartApplicationListener ?
			(SmartApplicationListener) listener : new GenericApplicationListenerAdapter(listener));
	return (smartListener.supportsEventType(eventType) && smartListener.supportsSourceType(sourceType));
}
```
这里没有进行实际的处理，实际处理在smartListener.supportsEventType(eventType)和smartListener.supportsSourceType(sourceType)方法中。

smartListener.supportsEventType(eventType)：

```
public boolean supportsEventType(Class<? extends ApplicationEvent> eventType) {
	Class typeArg = GenericTypeResolver.resolveTypeArgument(this.delegate.getClass(), ApplicationListener.class);
	if (typeArg == null || typeArg.equals(ApplicationEvent.class)) {
		Class targetClass = AopUtils.getTargetClass(this.delegate);
		if (targetClass != this.delegate.getClass()) {
			typeArg = GenericTypeResolver.resolveTypeArgument(targetClass, ApplicationListener.class);
		}
	}
	return (typeArg == null || typeArg.isAssignableFrom(eventType));
}
```
该方法主要的逻辑就是根据事件类型判断是否和监听器参数泛型的类型是否一致。

smartListener.supportsSourceType(sourceType)方法的实现为：

```
public boolean supportsSourceType(Class<?> sourceType) {
	return true;
}
```

定义自己的监听器要明确指定参数泛型，表明该监听器支持的事件，如果不指明具体的泛型，则没有监听器监听事件。

## 还可以定义自己的事件
暂先不做解析。

