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


