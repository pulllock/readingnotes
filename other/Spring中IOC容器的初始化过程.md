Spring IOC容器初始化过程分为Resource定位，载入解析，注册。IOC容器初始化过程中不包含Bean的依赖注入。Bean的依赖注入一般会发生在第一次通过getBean向容器索取Bean的时候。

大致步骤：

- 读取xml文件。
- 将xml文件读取后，变成Resource。
- 从Resource中解析出BeanDefinition。
- 将BeanDefinition注册到BeanFactory中。

完成了初始化过程之后，Bean就存在与BeanFactory中，等待被使用。

# ClassPathXmlApplicationContext初始化过程
实际的构造方法：

```
public ClassPathXmlApplicationContext(String[] configLocations, boolean refresh, ApplicationContext parent)
		throws BeansException {
	//super方法为容器设置好Bean资源加载器
	//该方法最终会调用到AbstractApplicationContext的无参构造方法
	//这里会默认设置解析路径的模式为Ant-style
	super(parent);
	//设置Bean定义资源文件的路径，将位置信息保存到configLocations中，供后面解析使用。
	setConfigLocations(configLocations);
	if (refresh) {
		//调用容器的refresh，载入BeanDefinition的入口
		refresh();
	}
}
```

refresh()方法：

```
public void refresh() throws BeansException, IllegalStateException {
	synchronized (this.startupShutdownMonitor) {
		// 调用容器准备刷新的方法，此方法中会获取容器的当前时间，给容器设置同步标识
		//初始化前的准备工作，例如对系统属性或环境变量进行准备以及验证。
		prepareRefresh();

		//通知子类启动refreshBeanFactory的调用
		//初始化BeanFactory，并进行XML文件读取
		//ClassPathXmlApplicationContext包含着BeanFactory所提供的一切特征
		//这一步会复用BeanFactory中的配置文件读取解析以及其他功能
		//这一步之后ClassPathXmlApplicationContext已经包含了BeanFactory所提供的功能，可以进行Bean的提取等基础操作了。
		//这一步是IOC初始化的整个过程。
		ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

		//准备当前上下文用的BeanFactory，为BeanFactory配置容器特性，例如类加载器，事件处理器等各种功能填充。
		//对BeanFactory各种功能的填充，比如@Qualifier和@Autowired注解就是在这一步增加的支持
		prepareBeanFactory(beanFactory);

		try {
			//为子类设置BeanFactory的后置处理器
			//子类覆盖方法做额外的处理。
			postProcessBeanFactory(beanFactory);

			//调用BeanFactoryPostProcessor，激活各种BeanFactory处理器
			invokeBeanFactoryPostProcessors(beanFactory);

			// Register bean processors that intercept bean creation.
			//注册拦截Bean创建的Bean处理器，这里只是注册，真正调用实在getBean的时候。
			registerBeanPostProcessors(beanFactory);

			// Initialize message source for this context.
			//为上下文初始化Message源，国际化处理
			initMessageSource();

			// Initialize event multicaster for this context.
			//初始化应用消息广播器，并放入applicationEventMulticaster bean中
			initApplicationEventMulticaster();

			// Initialize other special beans in specific context subclasses.
			留给子类来初始化其他的Bean
			onRefresh();

			// Check for listener beans and register them.
			在所有注册的bean中查找Listener bean，注册到消息广播器中
			registerListeners();

			// Instantiate all remaining (non-lazy-init) singletons.
			//初始化剩下的单实例，非惰性的
			finishBeanFactoryInitialization(beanFactory);

			// Last step: publish corresponding event.
			完成刷新过程，通知生命周期处理器lifecycleProcessor刷新过程，同时发出ContextRefreshEvent通知别人
			finishRefresh();
		}

		catch (BeansException ex) {
			// Destroy already created singletons to avoid dangling resources.
			destroyBeans();

			// Reset 'active' flag.
			cancelRefresh(ex);

			// Propagate exception to caller.
			throw ex;
		}
	}
}
```

重点看下`ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();`这句，告诉子类启动refreshBeanFactory方法以及通过getBeanFactory获得BeanFactory：

```
protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
	refreshBeanFactory();
	ConfigurableListableBeanFactory beanFactory = getBeanFactory();
	if (logger.isDebugEnabled()) {
		logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
	}
	return beanFactory;
}
```

refreshBeanFactory方法在AbstractRefreshableApplicationContext实现：

```
//创建载入BeanFactory
protected final void refreshBeanFactory() throws BeansException {
	//如果已经存在BeanFactory，销毁并关闭
	if (hasBeanFactory()) {
		destroyBeans();
		closeBeanFactory();
	}
	try {
		//创建IOC容器，创建了DefaultListableBeanFactory容器，给ApplicationContext使用
		DefaultListableBeanFactory beanFactory = createBeanFactory();
		beanFactory.setSerializationId(getId());
		customizeBeanFactory(beanFactory);
		//启动对BeanDefinitions的载入
		loadBeanDefinitions(beanFactory);
		synchronized (this.beanFactoryMonitor) {
			this.beanFactory = beanFactory;
		}
	}
}
```

对BeanDefinition的载入，loadBeanDefinitions方法：

```
protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
	//创建bean读取器，容器使用该读取器去读取Bean定义资源
	XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

	//配置bean读取器
	beanDefinitionReader.setEnvironment(this.getEnvironment());
	beanDefinitionReader.setResourceLoader(this);
	beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

	//当Bean读取器读取Bean定义的xml资源文件时，启用xml的校验机制
	initBeanDefinitionReader(beanDefinitionReader);
	//通过beanDefinitionReader加载BeanDefinitions
	loadBeanDefinitions(beanDefinitionReader);
}
```

loadBeanDefinitions(beanDefinitionReader)方法：

```
protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
	//获得Bean配置文件的资源位置
	Resource[] configResources = getConfigResources();
	if (configResources != null) {
		reader.loadBeanDefinitions(configResources);
	}
	String[] configLocations = getConfigLocations();
	if (configLocations != null) {
		//最终还是转换成Resource的形式去加载资源
		reader.loadBeanDefinitions(configLocations);
	}
}
```

在AbstractBeanDefinitionReader中的loadBeanDefinitions(configResources)方法：

```
public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
	Assert.notNull(resources, "Resource array must not be null");
	int counter = 0;
	for (Resource resource : resources) {
		//此处loadBeanDefinitions并没有实现，具体实现在各个子类中
		//比如XmlBeanDefinitionReader中
		counter += loadBeanDefinitions(resource);
	}
	return counter;
}
```

XmlBeanDefinitionReader中的loadBeanDefinitions方法：

```
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
	Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
	if (currentResources == null) {
		currentResources = new HashSet<EncodedResource>(4);
		this.resourcesCurrentlyBeingLoaded.set(currentResources);
	}
	try {
		//得到xml文件的InputStream
		InputStream inputStream = encodedResource.getResource().getInputStream();
		try {
			//得到InputSource
			InputSource inputSource = new InputSource(inputStream);
			if (encodedResource.getEncoding() != null) {
				inputSource.setEncoding(encodedResource.getEncoding());
			}
			//doLoadBeanDefinitions是从xml文件中加载BeanDefinitions
			return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
		}
		finally {
			inputStream.close();
		}
	}
	catch (IOException ex) {......}
	finally {......}
}
```

# xml配置文件的读取以及转化为Bean对象的过程

当Spring定位到xml之后，将xml转换为文件流的形式，作为InputSource和Resource对象传递给文档解析器进行解析，文档解析的开始是XmlDefinitionReader的doLoadBeanDefinitions方法。

```
//inputSource是SAX的InputSource
//resource对象是对xml文件描述的一个对象
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource) throws BeanDefinitionStoreException {
	try {
		//xml的解析模式
		int validationMode = getValidationModeForResource(resource);
		//将inputResource解析为Document对象
		Document doc = this.documentLoader.loadDocument(
				inputSource, getEntityResolver(), this.errorHandler, validationMode, isNamespaceAware());
		//Document传递给registerBeanDefinitions方法
		//此方法才是真正把Document对象解析为BeanDefinition对象的具体实现
		return registerBeanDefinitions(doc, resource);
	}
}
```

registerBeanDefinitions方法：

```
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    // Support old XmlBeanDefinitionParser SPI for backwards-compatibility.
    if (this.parserClass != null) {
        XmlBeanDefinitionParser parser =
                (XmlBeanDefinitionParser) BeanUtils.instantiateClass(this.parserClass);
        return parser.registerBeanDefinitions(this, doc, resource);
    }
    //先实例化一个BeanDefinitionDocumentReader，这个对象是通过BeanUtils.instantiateClass方法实例化出来的
    //实际上BeanUtils.instantiateClass中是封装了Java的反射的一些方法，通过基本的Java反射来构造实例。
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();

    //记录下注册之前BeanDefinition中对象的个数。
    int countBefore = getRegistry().getBeanDefinitionCount();
    //开始解析Document
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

DefaultBeanDefinitionDocumentReader类中的registerBeanDefinitions方法：

```
//在这里解析Document
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
	this.readerContext = readerContext;
	logger.debug("Loading bean definitions");
	Element root = doc.getDocumentElement();
	doRegisterBeanDefinitions(root);
}
```

doRegisterBeanDefinitions方法：

```
protected void doRegisterBeanDefinitions(Element root) {
	String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
	if (StringUtils.hasText(profileSpec)) {
		Assert.state(this.environment != null, "environment property must not be null");
		String[] specifiedProfiles = StringUtils.tokenizeToStringArray(profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
		if (!this.environment.acceptsProfiles(specifiedProfiles)) {
			return;
		}
	}

	//BeanDefinitionParserDelegate对象描述了Spring中bean节点中定义的所有属性和子节点
	BeanDefinitionParserDelegate parent = this.delegate;
	this.delegate = createHelper(readerContext, root, parent);

	//xml解析的预处理，可以自己定义一些节点属性等，此方法Spring默认实现为空
	preProcessXml(root);
	//把Document对象解析为BeanDefinition对象
	parseBeanDefinitions(root, this.delegate);
	//xml解析的后处理，可以在解析完xml之后，实现自己的逻辑。Spring默认实现为空。
	postProcessXml(root);

	this.delegate = parent;
}
```

parseBeanDefinitions方法：

```
//解析在root级别的元素，比如import，alias，bean
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
	//校验是不是Spring的默认命名空间。
	//默认命名空间是http://www.springframework.org/schema/beans
	//如果是Spring的默认命名空间，就按照默认命名空间来解析，否则就按照自定义标签来解析
	if (delegate.isDefaultNamespace(root)) {
		NodeList nl = root.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (node instanceof Element) {
				Element ele = (Element) node;
				//查看子节点是不是默认命名空间，是就按照默认解析，不是就按照自定义标签进行解析
				if (delegate.isDefaultNamespace(ele)) {
					//解析Spring默认的标签
					parseDefaultElement(ele, delegate);
				}
				else {
					//解析自定义标签
					delegate.parseCustomElement(ele);
				}
			}
		}
	}
	else {
		//解析自定义标签
		delegate.parseCustomElement(root);
	}
}
```

解析默认的标签，parseDefaultElement方法：

```
//此方法会依次解析文档中的import，alias，bean，beans标签
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
	if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
		importBeanDefinitionResource(ele);
	}
	else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
		processAliasRegistration(ele);
	}
	else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
		processBeanDefinition(ele, delegate);
	}
	else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
		// recurse
		doRegisterBeanDefinitions(ele);
	}
}
```

import标签的解析，importBeanDefinitionResource方法：

```
protected void importBeanDefinitionResource(Element ele) {
	//获取import标签中的resource属性，此属性表示资源地址
	//resource属性不可为空
	String location = ele.getAttribute(RESOURCE_ATTRIBUTE);
	if (!StringUtils.hasText(location)) {
		getReaderContext().error("Resource location must not be empty", ele);
		return;
	}

	// 解析resource中的占位符为真正的路径，比如"${user.dir}"
	location = environment.resolveRequiredPlaceholders(location);

	Set<Resource> actualResources = new LinkedHashSet<Resource>(4);

	// 解析路径，判断是相对路径还是绝对路径
	boolean absoluteLocation = false;
	try {
		absoluteLocation = ResourcePatternUtils.isUrl(location) || ResourceUtils.toURI(location).isAbsolute();
	}
	catch (URISyntaxException ex) {...}
	//绝对路径
	if (absoluteLocation) {
		try {
			//递归调用Bean的解析过程
			int importCount = getReaderContext().getReader().loadBeanDefinitions(location, actualResources);
		}
	}
	else {//相对路径，计算出绝对路径，进行递归调用解析过程
		try {
			int importCount;
			Resource relativeResource = getReaderContext().getResource().createRelative(location);
			if (relativeResource.exists()) {
				importCount = getReaderContext().getReader().loadBeanDefinitions(relativeResource);
				actualResources.add(relativeResource);
			}
			else {
				String baseLocation = getReaderContext().getResource().getURL().toString();
				importCount = getReaderContext().getReader().loadBeanDefinitions(
						StringUtils.applyRelativePath(baseLocation, location), actualResources);
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Imported " + importCount + " bean definitions from relative location [" + location + "]");
			}
		}
		catch (IOException ex) {......}
	}
	//解析完成后进行监听器激活处理
	Resource[] actResArray = actualResources.toArray(new Resource[actualResources.size()]);
	getReaderContext().fireImportProcessed(location, actResArray, extractSource(ele));
}
```

### bean标签的解析
processBeanDefinition(ele, delegate);这里解析和注册bean。

```
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
	//解析，将XML的元素解析成BeanDefinition，然后保存在BeanDefinitionHolder中
	BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
	if (bdHolder != null) {
		bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
		try {
			//注册
			BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
		}
		//发送注册事件
		getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
	}
}
```

parseBeanDefinitionElement方法对xml元素进行解析：

```
//走了好几步，才找到解析的方法
public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, BeanDefinition containingBean) {

	this.parseState.push(new BeanEntry(beanName));

	String className = null;
	if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
		className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
	}

	try {
		String parent = null;
		if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
			parent = ele.getAttribute(PARENT_ATTRIBUTE);
		}
		//创建BeanDefinition
		AbstractBeanDefinition bd = createBeanDefinition(className, parent);

		parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
		bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

		parseMetaElements(ele, bd);
		parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
		parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
		//处理构造器属性
		parseConstructorArgElements(ele, bd);
		//处理属性元素
		parsePropertyElements(ele, bd);
		parseQualifierElements(ele, bd);

		bd.setResource(this.readerContext.getResource());
		bd.setSource(extractSource(ele));

		return bd;
	}
	finally {
		this.parseState.pop();
	}

	return null;
}
```

处理完元素之后，组装成了BeanDefinition，然后就开始要注册了，registerBeanDefinition：

```
public static void registerBeanDefinition(
		BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
		throws BeanDefinitionStoreException {

	//使用唯一的id作为名字注册
	String beanName = definitionHolder.getBeanName();
	registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

	//使用别名注册
	String[] aliases = definitionHolder.getAliases();
	if (aliases != null) {
		for (String aliase : aliases) {
			registry.registerAlias(beanName, aliase);
		}
	}
}
```

registry.registerBeanDefinition方法中最主要的就是`this.beanDefinitionMap.put(beanName, beanDefinition)`，将beanName作为key，BeanDefinition作为value放到一个ConcurrentHashMap中去。

到此IOC容器初始化就完成了，接下来就是依赖注入。

## 总结

- IOC容器初始化入口是在构造方法中调用refresh开始的。
- 通过ResourceLoader来完成资源文件位置的定位，DefaultResourceLoader是默认的实现，同时上下文本身就给除了ResourceLoader的实现。
- 创建的IOC容器是DefaultListableBeanFactory。
- IOC对Bean的管理和依赖注入功能的实现是通过对其持有的BeanDefinition进行相关操作来完成的。
- 通过BeanDefinitionReader来完成定义信息的解析和Bean信息的注册。
- XmlBeanDefinitionReader是BeanDefinitionReader的实现了，通过它来解析xml配置中的bean定义。
- 实际的处理过程是委托给BeanDefinitionParserDelegate来完成的。得到Bean的定义信息，这些信息在Spring中使用BeanDefinition对象来表示。
- BeanDefinition的注册是由BeanDefinitionRegistry实现的registerBeanDefiition方法进行的。内部使用ConcurrentHashMap来保存BeanDefiition。
