# Sprig IOC容器的设计

有两个主要的容器系列：

* 一个是实现BeanFactory接口的简单容器系列，只实现了容器最基本功能
* 一个是ApplicationContext应用上下文，作为容器的高级形态存在，增加了许多面向框架的特性，对应用环境做了许多适配。

BeanDefinition抽象了对bean的定义。

![](IOCContainer.png)

* 从接口BeanFactory到HierarchicalBeanFactory，再到ConfigurableBeanFactory是一条主要的BeanFactory设计路径。BeanFactory接口定义了基本的IOC容器规范。HierarchicalBeanFactory接口继承了BeanFactory，增加了getParentBeanFactory，使得BeanFactory具备了双亲IOC容器的管理功能。ConfigurableBeanFactory接口主要定义了一些对BeanFactory的配置功能。
* 第二条主线以ApplicationContext应用上下文接口为核心，从BeanFactory到ListableBeanFactory，再到ApplicationContext再到WebApplicationContext或者ConfigurableApplicationContext接口。ListableBeanFactory和HierarchicalBeanFactory连接BeanFactory接口定义和ApplicationContext的接口功能。ApplicationContext通过集成MessageSource，ResourceLoader，ApplicationEventPublisher接口，在IOC容器的基础上添加了许多对高级容器的特性支持。
* 这里设计的是主要的接口关系，具体IOC容器是在这些接口下实现的，DefaultLisableBeanFactory实现了ConfigurableBeanFactory，成为一个简单的IOC实现。XMLBeanFactory都是在DefaultListableBeanFactory的基础上做扩展。
* 这个接口体系是一BeanFactory和ApplicationContext为核心。BeanFactory是IOC容器最基本的接口。web环境中设计了WebApplicationContext接口。

# BeanFactory
接口定义了IOC容器最基本的形式，提供了IOC容器所应遵守的最基本的服务契约。

使用`&`符号得到FactoryBean本身，区分通过容器来获取FactoryBean产生的对象和FactoryBean本身。

getBean方法是主要方法，可以取得IOC容器中管理的Bean。

```
public interface BeanFactory {
	//使用`&`符号得到FactoryBean本身，
	//区分通过容器来获取FactoryBean产生的对象和FactoryBean本身。
	String FACTORY_BEAN_PREFIX = "&";

	//取得IOC容器中管理的Bean
	Object getBean(String name) throws BeansException;

	//根据名字和类型获取Bean
	<T> T getBean(String name, Class<T> requiredType) throws BeansException;

	//根据类型获取bean
	<T> T getBean(Class<T> requiredType) throws BeansException;

	//获取的Bean是prototype类型的，可以为这个类型的bean生成指定构造函数对应参数
	Object getBean(String name, Object... args) throws BeansException;

	//判断容器是否含有指定名字的bean
	boolean containsBean(String name);

	//查询指定名字的bean是否是单例bean，单例属性可以再BeanDefinition中指定
	boolean isSingleton(String name) throws NoSuchBeanDefinitionException;

	//查询指定名字的bean是否是原型bean
	boolean isPrototype(String name) throws NoSuchBeanDefinitionException;

	//根据名字查询bean是否是特定的class类型
	boolean isTypeMatch(String name, Class<?> targetType) throws NoSuchBeanDefinitionException;

	//查询指定名字的bean的类型
	Class<?> getType(String name) throws NoSuchBeanDefinitionException;

	//获取bean的所有别名
	String[] getAliases(String name);

}
```

## XmlBeanFactory

Spring 3.1 之后被废除，官方建议使用DefaultListableBeanFactory和XmlBeanDefinitionReader来代替XmlBeanFactory。

只提供最基本的IOC容器的功能，可以认为直接的BeanFactory实现是IOC容器的基本形式，而各种ApplicationContext的实现是IOC容器的高级表现形式。

![](XmlBeanFactory.png)


XmlBeanFactory继承自DefaultListableBeanFactory，后者非常重要，是经常要用到的一个IOC容器的实现，ApplicationContext也会用到它。DefaultListableBeanFactory实际上已经包含了基本IOC容器所具有的重要功能。

Spring中实际上是把DefaultListableBeanFactory作为一个默认的功能完整的IOC容器来使用。

在XmlBeanFactory中初始化了一个XmlBeanDefinitionReader对象来处理BeanDefinition。

```
public class XmlBeanFactory extends DefaultListableBeanFactory {
	//初始化一个reader
	private final XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(this);
	//构造方法
	public XmlBeanFactory(Resource resource) throws BeansException {
		this(resource, null);
	}

	//构造方法
	public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
		super(parentBeanFactory);
		//从Resource中载入BeanDefinitions的过程，
		//也是IOC初始化的重要组成部分
		this.reader.loadBeanDefinitions(resource);
	}

}
```

使用编程式IOC容器：

```
ClassPathResource res = new ClassPathResource("beans.xml");
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(res);
```

# ApplicationContext
高级形态意义的IOC容器，在BeanFactory基础上添加附加功能。

* 支持不同的信息源 扩展了MessageSource接口，支持国际化的实现。
* 访问资源 体现在对ResourceLoader和Resource的支持上，可以从不同地方得到Bean定义资源。
* 支持应用事件 继承了接口ApplicationEventPublisher，在上下文引入了事件机制。
* 在ApplicationContext中提供附加功能，一般建议ApplicationContext作为IOC容器的基本形式

## FileSystemXmlApplicationContext
ApplicationContext应用上下文的主要功能已经在FileSystemXmlApplicationContext的基类AbstractXmlApplicationContext中实现了。

```
//实例化应用上下文支持，同时启动IOC容器的refresh过程
//refresh会牵涉ioc容器启动的一些列复杂操作。
public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)throws BeansException {

	super(parent);
	setConfigLocations(configLocations);
	if (refresh) {
		refresh();
	}
}
//可以为在文件系统中读取以xml形式存在的BeanDefinition做准备
protected Resource getResourceByPath(String path) {
	if (path != null && path.startsWith("/")) {
		path = path.substring(1);
	}
	return new FileSystemResource(path);
}
```

# IOC容器的初始化过程
初始化是由refresh()方法启动的，标志着ioc容器的正式启动

* BeanDefinition的Resource定位
* BeanDefinition载入
* 向IOC容器注册BeanDefinition

Resource定位过程，是指BeanDefinition的资源定位，由ResourceLoader通过统一的Resource接口完成。

BeanDefinition的载入，是把用户定义好的Bean表示成IOC容器内部的数据结构，内部数据结构就是BeanDefinition。

向容器注册BeanDefinition，是通过调用BeanDefinitionRegistry接口来实现的。在IOC容器内部将BeanDefinition注入到一个HashMap中区，通过这个map来持有BeanDefinition数据。

初始化过程一般不包含Bean依赖注入的实现，Bean定义的载入和依赖注入是两个独立过程，依赖注入一般发生在应用第一次通过getBean向容器获取bean的时候。

但是如果我们对某个bean设置了lazyinit属性，那么这个bean的依赖注入在容器初始化就预先完成了。



## BeanDefinition的Resource定位
FileSystemXmlBeanFactory已经通过集成AbstractApplicationContext具备了ResourceLoader读入以Resource定义的BeanDefinition的能力。

```
public class FileSystemXmlApplicationContext extends AbstractXmlApplicationContext {
	public FileSystemXmlApplicationContext() {
	}
	public FileSystemXmlApplicationContext(ApplicationContext parent) {
		super(parent);
	}

	//configLocation包含的是BeanDefinition所在的文件路径
	public FileSystemXmlApplicationContext(String configLocation) throws BeansException {
		this(new String[] {configLocation}, true, null);
	}

	//允许包含多个路径
	public FileSystemXmlApplicationContext(String... configLocations) throws BeansException {
		this(configLocations, true, null);
	}

	//允许多个路径，并允许自定自己的双亲容器
	public FileSystemXmlApplicationContext(String[] configLocations, ApplicationContext parent) throws BeansException {
		this(configLocations, true, parent);
	}

	public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh) throws BeansException {
		this(configLocations, refresh, null);
	}

	//在对象初始化过程中，调用refresh方法载入BeanDefinition
	//这个refresh启动了BeanDefinition的载入过程
	public FileSystemXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
			throws BeansException {

		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}

	//应用于文件系统中Resource的实现，构造一个FileSystemResource来得到一个在文件系统中定位的BeanDefinition
	//这个getResourceByPath是在BeanDefinitionReader的loadBeanDefinition中被调用的，
	//loadBeanDefinition采用模板模式，具体的实现实际是子类完成的
	@Override
	protected Resource getResourceByPath(String path) {
		if (path != null && path.startsWith("/")) {
			path = path.substring(1);
		}
		return new FileSystemResource(path);
	}

}
```

```
protected final void refreshBeanFactory() throws BeansException {
	//如果已建立BeanFactory，则销毁并关闭该BeanFactory
	if (hasBeanFactory()) {
		destroyBeans();
		closeBeanFactory();
	}
	//这里创建并设置持有的DefaultListableBeanFactory的地方
	//同时调用loadBeanDefinitions再载入BeanDefinition的信息
	try {
		DefaultListableBeanFactory beanFactory = createBeanFactory();
		beanFactory.setSerializationId(getId());
		customizeBeanFactory(beanFactory);
		loadBeanDefinitions(beanFactory);
		synchronized (this.beanFactoryMonitor) {
			this.beanFactory = beanFactory;
		}
	}
}
```


```
//这是在上下文中创建DefaultListableBeanFactory的地方
//getInternalParentBeanFactory的具体实现可以参考AbstractApplicationContext中的实现，会根据容器已有的双亲IOC容器来生成DefaultListableBeanFactory的双亲IOC容器
protected DefaultListableBeanFactory createBeanFactory() {
	return new DefaultListableBeanFactory(getInternalParentBeanFactory());
}
```

loadBeanDefinitions：

是使用BeanDefinitionReader载入bean定义的地方

```
public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionStoreException {
	//获取resourceLoader，使用的是DefaultResourceLoader
	ResourceLoader resourceLoader = getResourceLoader();
	//解析Resource路径模式
	//比如我们设定的各种Ant格式的路径定义，得到需要的Resource集合
	//这些Resource集合指向我们已经定义好的BeanDefinition信息，可以是多个文件
	if (resourceLoader instanceof ResourcePatternResolver) {
		try {
			//调用DefaultResourceLoader的getResource完成具体的resource定位
			Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
			int loadCount = loadBeanDefinitions(resources);
			if (actualResources != null) {
				for (Resource resource : resources) {
					actualResources.add(resource);
				}
			}
			return loadCount;
		}
	}
	else {
		//调用DefaultResourceLoader的getResource完成具体的resource定位
		Resource resource = resourceLoader.getResource(location);
		int loadCount = loadBeanDefinitions(resource);
		if (actualResources != null) {
			actualResources.add(resource);
		}
		return loadCount;
	}
}
```

getResource：

DefaultResourceLoader


```
public Resource getResource(String location) {
	//处理带有classpath标识的resource
	if (location.startsWith(CLASSPATH_URL_PREFIX)) {
		return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
	}
	else {
		try {
			//处理带URL的resource定位
			URL url = new URL(location);
			return new UrlResource(url);
		}
		catch (MalformedURLException ex) {
			//既不是ClassPath也不是url的，把getResource任务交给getResourceByPath，默认的实现是得到一个ClassPathContextResource，由子类来实现
			return getResourceByPath(location);
		}
	}
}
```

FileSystemXmlApplicationContext实现了getResourceByPath。

# BeanDefinition的载入和解析

对IOC容器来说，载入过程相当于把定义的BeanDefinition在IOC容器中转化成一个Spring内部表示的数据结构的过程。BeanDefinition通过HashMap来保持和维护。

refresh方法：描述了整个ApplicationContext的初始化过程。

```
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		prepareRefresh();

		//这是在子类中启动refreshBeanFactory的地方
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		// Prepare the bean factory for use in this context.
		prepareBeanFactory(beanFactory);

		try {
			//设置BeanFactory的后置处理
			postProcessBeanFactory(beanFactory);

			//调用BeanFactory的后处理器，这些后处理器是在Bean定义中向容器注册的
			invokeBeanFactoryPostProcessors(beanFactory);

			//注册bean的后处理器，在bean创建过程中调用
			registerBeanPostProcessors(beanFactory);

			//对上下文中的消息源进行初始化
			initMessageSource();

			//对上下文中的事件机制进行初始化
			initApplicationEventMulticaster();

			//初始化其他特殊bean
			onRefresh();

			//检查监听bean，并将这些bean向容器注册
			registerListeners();

			// 实例化所有的 (non-lazy-init)单例.
			finishBeanFactoryInitialization(beanFactory);

			// 发布容器事件，结束refres过程
			finishRefresh();
		}catch (BeansException ex) {
			//销毁前面过程生成的单例bean
			destroyBeans();
			// 重置active标志
			cancelRefresh(ex);
		}
	}
}
```

在AbstractRefreshableApplicationContext的refreshBeanFactory方法中，创建了BeanFactory，在创建容器之前，如果已有容器存在，需要把已有容器销毁和关闭。在建立好当前的IOC容器以后，开始对容器的初始化过程，比如BeanDefinition的载入。

loadBeanDefinitions抽象方法，实际在AbstractXmlApplicationContext中实现，在这个loadBeanDefinitions中初始化读取器XMLBeanDefinitionReader，然后把这个读取器在ioc容器中设置好，最后启动读取器来完成BeanDefinition的载入

```
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
	//创建XmlBeanDefinitionReader，并通过回调设置到BeanFactory中去
	//使用的是DefaultListableBeanFactory
	XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
beanDefinitionReader.setEnvironment(this.getEnvironment());
	//这里设置XmlBeanDefinitionReader，为XmlBeanDefinitionReader配置ResourceLoader，因为DefaultResourceLoader是父类，所以this可以直接被使用
	beanDefinitionReader.setResourceLoader(this);
	beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

	//启动bean定义信息载入过程
	initBeanDefinitionReader(beanDefinitionReader);
	loadBeanDefinitions(beanDefinitionReader);
}
```

接着调用loadBeanDefinitions，首先得到BeanDefinition信息的Resource定位，然后调用XmlBeanDefinitionReader来读取，具体载入过程是委托给BeanDefinitionReader完成。

```
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
	Resource[] configResources = getConfigResources();
	if (configResources != null) {
		reader.loadBeanDefinitions(configResources);
	}
	String[] configLocations = getConfigLocations();
	if (configLocations != null) {
		reader.loadBeanDefinitions(configLocations);
	}
}
```

初始化FileSystemXmlApplicationContext的过程是通过调用IOC的refresh来启动BeanDefinition的载入过程，初始化是通过定义的XmlBeanDefinitionReader来完成。

具体的Resource载入在XmlBeanDefinitionReader读入BeanDefinition时实现。

```
public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
	//载入BeanDefinition的过程会遍历整个Resource集合所包含的BeanDefinition信息
	int counter = 0;
	for (Resource resource : resources) {
		counter += loadBeanDefinitions(resource);
	}
	return counter;
}
```

loadBeanDefinitions：在读取器中需要得到XMl的resource，得到xml之后，就可以解析了，解析交给了BeanDefinitionParserDelegate来完成。

```
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {

	Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
	if (currentResources == null) {
		currentResources = new HashSet<EncodedResource>(4);
		this.resourcesCurrentlyBeingLoaded.set(currentResources);
	}
	if (!currentResources.add(encodedResource)) {}
	//这里得到xml文件，并得到IO的InputSource准备进行读取
	try {
		InputStream inputStream = encodedResource.getResource().getInputStream();
		try {
			InputSource inputSource = new InputSource(inputStream);
			if (encodedResource.getEncoding() != null) {
				inputSource.setEncoding(encodedResource.getEncoding());
			}
			return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
		}
		
	}
}
```

具体的读取过程在doLoadBeanDefinitions方法中，从特定的xml文件中实际载入BeanDefinition的地方：

```
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
	try {
		int validationMode = getValidationModeForResource(resource);
		//取得xml文件的Document对象，解析过程是由documentLoader（DefaultDocumentLoader）完成的
		Document doc = this.documentLoader.loadDocument(
				inputSource, getEntityResolver(), this.errorHandler, validationMode, isNamespaceAware());
		//启动对BeanDefinition解析的详细过程，这个解析会使用到Spring的Bean配置规则
		return registerBeanDefinitions(doc, resource);
	}
}
```

registerBeanDefinitions 按照Spring的Bean语义要求进行解析并转化为容器内部数据结构。

```
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
	//创建BeanDefinitionDocumentReader来对xml的BeanDefinition进行解析
	BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
	documentReader.setEnvironment(getEnvironment());
	int countBefore = getRegistry().getBeanDefinitionCount();
	//具体解析过程在registerBeanDefinitions中完成
	documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
	return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

创建BeanDefinitionDocumentReader：

```
protected BeanDefinitionDocumentReader createBeanDefinitionDocumentReader() {
	return BeanDefinitionDocumentReader.class.cast(BeanUtils.instantiateClass(this.documentReaderClass));
}
```

得到reader之后，具体解析过程：`registerBeanDefinitions->doRegisterBeanDefinitions->parseBeanDefinitions->parseDefaultElement->processBeanDefinition`
BeanDefinitionParserDelegate来解析：

```
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
	//封装了BeanDefinition，Bean的名字和别名，来完成向ioc容器注册
	//得到这个holder意味着对xml的解析是按照spring的bean规则进行解析得到的
	BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
	if (bdHolder != null) {
		bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
		try {
			//这里是向容器注册解析得到的BeanDefinition的地方
			BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
		}
		//注册以后，发送消息
		getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
	}
}
```

具体BeanDefinition解析是在BeanDefinitionParserDelegate中完成的，这个类包含了对bean定义规则的处理。

其他元素的解析，比如各种bean属性的配置，是由parseBeanDefinitionElement来完成。

上面是BeanDefinition依据xml的<bean>定义被创建的过程。

![](AbstractBeanDefinition.png)
这个BeanDefinition可以看成是对<bean>的抽象。


# BeanDefinition在IOC容器中的注册




