# DefaultListableBeanFactory

XmlBeanFactory继承自DefaultListableBeanFactory，DefaultListableBeanFactory是bean加载的核心，是注册及加载bean的默认实现。

![](XmlBeanFactory.png)

* AliasRegistry定义对alias的简单增删改等操作。
* SimpleAliasRegistry主要使用map作为alias的缓存，并对接口AliasRegistry进行实现。
* SingletonBeanRegistry定义对单例的注册及获取。
* BeanFactory定义获取bean及各种属性。
* DefaultSingletonBeanRegistry对接口SingletonBeanRegistry的实现。
* HierarchicalBeanFactory继承BeanFactory，增加了对parentFactory的支持。
* BeanDefinitionRegistry定义对BeanDefinition的各种增删改操作。
* FactoryBeanRegistrySupport在DefaultSingletonBeanRegistry的基础上增加了对FactoryBean的特殊处理。
* ConfigurableBeanFactory提供配置Factory的各种方法。
* ListableBeanFactory根据条件获取bean的配置清单。
* AbstractBeanFactory综合FactoryBeanRegistrySupport和ConfigurableBeanFactory的功能。
* AutowireCapableBeanFactory提供创建bean，自动注入，初始化以及应用bean的后处理器。
* AbstractAutowireCapableBeanFactory综合AbstractBeanFactory，并对接口AutowireCapableBeanFactory进行实现。
* ConfigurableListableBeanFactory BeanFactory的配置清单，指定忽略类型及接口等。
* DefaultListableBeanFactory 综合上面所有功能，主要是对Bean注册后的处理。

# XMLBeanDefinitionReader

* ResourceLoader 定义资源加载器，主要应用于根据给定的资源文件地址返回对应的Resource。
* BeanDefinitionReader 主要定义资源文件读取并转换为BeanDefinition的各个功能。
* EnvironmentCapale定义获取Environment方法。
* DocumentLoader定义从资源文件加载到转换为Document的功能。
* AbstractBeanDefinitionReader对BeanDefinitionReader，EnvironmentCapale进行实现。
* BeanDefinitionParserDelegate定义解析Element的各种方法。

## XML配置文件读取的大致流程
1. 通过继承自AbstractBeanDefinitionReader中的方法，使用ResourceLoader将资源文件路径转换为对应的Resource文件。
2. 通过DocumentLoader对Resource文件进行转换，将Resource文件转换为Document文件。
3. 通过实现接口BeanDefinitionDocumentReader的DefaultBeanDefinitionDocumentReader类对Document进行解析，并使用BeanDefinitionParserDelegate对Element进行解析。

# XmlBeanFactory

## 配置文件封装

Spring使用Resource接口封装底层资源。

### InputStreamSource
接口
### Resource
接口，继承InputStreamResource

### AbstractResource
抽象类，实现了Resource接口，对接口的方法加以实现。

### WritableResource
接口，继承自Resource。	

### FileSystemResource
继承AbstractResource，实现WritableResource。

构造可以指定一个File或者路径。

```
public InputStream getInputStream() throws IOException {
	return new FileInputStream(this.file);
}
```

直接使用FileInputStream返回一个实例。

### AbstractFileResolvingResource
继承AbstractResource，实现URL到文件引用的转换。

### ClassPathResource
继承自AbstractFileResolvingResource。

```
public InputStream getInputStream() throws IOException {
	InputStream is;
	if (this.clazz != null) {
		is = this.clazz.getResourceAsStream(this.path);
	}
	else if (this.classLoader != null) {
		is = this.classLoader.getResourceAsStream(this.path);
	}
	else {
		is = ClassLoader.getSystemResourceAsStream(this.path);
	}
	if (is == null) {
		throw new FileNotFoundException(getDescription() + " cannot be opened because it does not exist");
	}
	return is;
}
```

使用class或者classLoader提供的底层方法实现。

通过Resource相关类完成了对配置文件封装后，读取工作就交给了XMLBeanDefinitionReader来处理。

# XmlBeanFactory

构造函数源码：

```
public XmlBeanFactory(Resource resource, BeanFactory parentBeanFactory) throws BeansException {
	super(parentBeanFactory);
	this.reader.loadBeanDefinitions(resource);
}
```

XmlBeanFactory有两个构造函数，其中`super(parentBeanFactory);`可以追踪到父类AbstractAutowireCapableBeanFactory中：

```
public AbstractAutowireCapableBeanFactory() {
	super();
	ignoreDependencyInterface(BeanNameAware.class);
	ignoreDependencyInterface(BeanFactoryAware.class);
	ignoreDependencyInterface(BeanClassLoaderAware.class);
}
```

`ignoreDependencyInterface`主要是忽略给定接口的自动装配功能。

真正实现资源加载的是在`this.reader.loadBeanDefinitions(resource);`中，也就是在XmlBeanDefinitionReader中完成的。

# 加载Bean，XmlBeanDefinitionReader

1. 封装资源文件，进入XmlBeanDefinitionReader后先对参数Resource使用EncodedResource进行封装。
2. 获取输入流，从Resource中获取对应的InputStream并构造InputSource。
3. 通过构造的InputSource实例和Resource实例继续调用函数doLoadBeanDefinitions(inputSource, encodedResource.getResource());

## EncodedResource
对资源文件的编码进行处理。

## doLoadBeanDefinitions
真正处理部分。

```
protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
	try {
	//1. 获取XML文件的验证模式
		int validationMode = getValidationModeForResource(resource);
		//2. 加载XML文件，得到Document
		Document doc = this.documentLoader.loadDocument(
				inputSource, getEntityResolver(), this.errorHandler, validationMode, isNamespaceAware());
				//3. 根据返回的Document注册Bean信息
		return registerBeanDefinitions(doc, resource);
	}
	...
}
```

## 获取XML验证模式

```
protected int getValidationModeForResource(Resource resource) {
	//获取当前验证模式
	int validationModeToUse = getValidationMode();
	//如果验证模式是手工指定的，使用指定的验证模式
	if (validationModeToUse != VALIDATION_AUTO) {
		return validationModeToUse;
	}
	//自动检测验证模式，VALIDATION_DTD 2，VALIDATION_XSD 3
	int detectedMode = detectValidationMode(resource);
	if (detectedMode != VALIDATION_AUTO) {
		return detectedMode;
	}
	//没有自动检测到就使用xsd
	return VALIDATION_XSD;
}
```

## 获取Document

```
Document doc = this.documentLoader.loadDocument(
					inputSource, getEntityResolver(), this.errorHandler, validationMode, isNamespaceAware());
```

获取Document委托给DocumentLoader，DocumentLoader是接口，真正的实现在DefaultDocumentLoader。

DefaultDocumentLoader中代码：

```
public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
			ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {

//创建DocumentBuilderFactory
	DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
	if (logger.isDebugEnabled()) {
		logger.debug("Using JAXP provider [" + factory.getClass().getName() + "]");
	}
	//创建DocumentBuilder
	DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
	//解析InputSource返回Document。
	return builder.parse(inputSource);
}
```

## EntityResolver
作用是项目本身提供一个如何寻找DTD声明的方法，由程序来实现寻找DTD声明的过程。

DelegatingEntityResolver：

```
public InputSource resolveEntity(String publicId, String systemId) throws SAXException, IOException {
	if (systemId != null) {
	//如果是dtd从这解析
		if (systemId.endsWith(DTD_SUFFIX)) {
			return this.dtdResolver.resolveEntity(publicId, systemId);
		}
		else if (systemId.endsWith(XSD_SUFFIX)) {
			return 
			//调用"META-INF/spring.schemas";解析
			this.schemaResolver.resolveEntity(publicId, systemId);
		}
	}
	return null;
}
```
DTD类型的由BeansDtdResolver来解析，直接截取systemId最后的xx.dtd然后去当前路进行该寻找。

XSD类型的由PluggableSchemaResolver解析，默认到META-INF/spring.schemas文件中找到systemId所对应的XSD文件并加载。

## 解析及注册BeanDefinitions

文件转换成Document之后，就需要解析注册bean了。代码如下：

```
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
//使用DefaultBeanDefinitionDocumentReader实例化
	BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
	//设置环境变量
	documentReader.setEnvironment(getEnvironment());
	//记录下加载新的BeanDefinition之前的个数
	int countBefore = getRegistry().getBeanDefinitionCount();
	//加载注册bean
	documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
	//返回本次加载的BeanDefinition个数
	return getRegistry().getBeanDefinitionCount() - countBefore;
}
```

DefaultBeanDefinitionDocumentReader中：

```
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
	this.readerContext = readerContext;
	logger.debug("Loading bean definitions");
	//获取root
	Element root = doc.getDocumentElement();
	//真正开始解析的方法
	doRegisterBeanDefinitions(root);
}
```

```
protected void doRegisterBeanDefinitions(Element root) {
//处理profile属性
	String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
	if (StringUtils.hasText(profileSpec)) {
		String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
				profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
		if (!getEnvironment().acceptsProfiles(specifiedProfiles)) {
			return;
		}
	}

//专门处理解析
	BeanDefinitionParserDelegate parent = this.delegate;
	this.delegate = createDelegate(this.readerContext, root, parent);

	//解析前处理，留给子类实现
	preProcessXml(root);
	parseBeanDefinitions(root, this.delegate);
	//解析后处理，留给子类实现
	postProcessXml(root);

	this.delegate = parent;
}
```

### profile属性的使用
可以用于多环境配置。

```
<beans profile="dev">
	...
</beans>
<beans profile="test">
	...
</beans>
...
```

首先获取是否定义了profile属性，如果定义了会需要到环境变量中寻找。

### parseBeanDefinitions
解析并注册BeanDefinition。

```
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
//对默认的root进行处理
	if (delegate.isDefaultNamespace(root)) {
		NodeList nl = root.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (node instanceof Element) {
				Element ele = (Element) node;
				if (delegate.isDefaultNamespace(ele)) {
				//处理默认bean标签
					parseDefaultElement(ele, delegate);
				}
				else {
				//处理自定义bean标签
					delegate.parseCustomElement(ele);
				}
			}
		}
	}
	else {
	//处理自定义root标签
		delegate.parseCustomElement(root);
	}
}
```

Spring配置有默认Bean声明和自定义声明。

# 默认标签的解析
## parseDefaultElement

```
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
	if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
	//解析import标签
		importBeanDefinitionResource(ele);
	}
	else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
	//解析alias标签
		processAliasRegistration(ele);
	}
	else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
	//解析bean标签
		processBeanDefinition(ele, delegate);
	}
	else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
		//对beans标签的处理
		doRegisterBeanDefinitions(ele);
	}
}
```

## bean标签的解析

```
processBeanDefinition(ele, delegate);
```

```
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
	BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
	if (bdHolder != null) {
		bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
		try {
			// Register the final decorated instance.
			BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
		}
		catch (BeanDefinitionStoreException ex) {
			getReaderContext().error("Failed to register bean definition with name '" +
					bdHolder.getBeanName() + "'", ele, ex);
		}
		// Send registration event.
		getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
	}
}
```

1. 首先委托BeanDefinitionDelegate类的parseBeanDefinitionElement方法进行元素解析，返回BeanDefinitionHolder类型的实例bdHolder，处理之后bdHolder包含配置文件中各种属性，class，name，id，alias等。
2. 返回的bdHolder不为空，若存在默认标签的子节点下有自定义属性，需要再次对自定义标签进行解析。
3. 解析完成之后，需要对解析后的bdHolder进行注册，委托给BeanDefinitionReaderUtils的registerBeanDefinition方法。
4. 发出响应事件，通知相关的监听器。

## 解析BeanDefinition
delegate.parseBeanDefinitionElement(ele)：

```
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
	return parseBeanDefinitionElement(ele, null);
}
```

```
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {
	//获取id属性
	String id = ele.getAttribute(ID_ATTRIBUTE);
	//获取name属性
	String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

	//分割name属性
	List<String> aliases = new ArrayList<String>();
	if (StringUtils.hasLength(nameAttr)) {
		String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
		aliases.addAll(Arrays.asList(nameArr));
	}

	String beanName = id;
	//没有id但是有name，使用name作为beanName
	if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
		beanName = aliases.remove(0);
		if (logger.isDebugEnabled()) {
			logger.debug("No XML 'id' specified - using '" + beanName +
					"' as bean name and " + aliases + " as aliases");
		}
	}

	if (containingBean == null) {
		checkNameUniqueness(beanName, aliases, ele);
	}

	AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
	if (beanDefinition != null) {
	//不存在beanName，Spring提供的命名规则生成beanName
		if (!StringUtils.hasText(beanName)) {
			try {
				if (containingBean != null) {
					beanName = BeanDefinitionReaderUtils.generateBeanName(
							beanDefinition, this.readerContext.getRegistry(), true);
				}
				else {
					beanName = this.readerContext.generateBeanName(beanDefinition);
					String beanClassName = beanDefinition.getBeanClassName();
					if (beanClassName != null &&
							beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
							!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
						aliases.add(beanClassName);
					}
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Neither XML 'id' nor 'name' specified - " +
							"using generated bean name [" + beanName + "]");
				}
			}
			catch (Exception ex) {
				error(ex.getMessage(), ele);
				return null;
			}
		}
		String[] aliasesArray = StringUtils.toStringArray(aliases);
		return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
	}

	return null;
}
```

1. 提取元素中id，name属性。
2. 进一步解析其他所有属性统一封装至AbstractBeanDefinition类型实例。
3. 如果没有beanName，使用默认规则生成beanName。
4. 将信息封装到BeanDefinitionHolder实例中。

### 解析其他标签

`AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);`

```
public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, BeanDefinition containingBean) {

	this.parseState.push(new BeanEntry(beanName));

	String className = null;
	//解析class属性
	if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
		className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
	}

	try {
	//解析parent属性
		String parent = null;
		if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
			parent = ele.getAttribute(PARENT_ATTRIBUTE);
		}
		//创建用于承载属性的AbstractBeanDefinition
		AbstractBeanDefinition bd = createBeanDefinition(className, parent);

		//解析默认bean的各种属性
		parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
		//提取description
		bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
		//解析元数据
		parseMetaElements(ele, bd);
		//解析lookup-method属性
		parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
		//解析replace-method属性
		parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
		//解析构造函数参数
		parseConstructorArgElements(ele, bd);
		//解析property子元素
		parsePropertyElements(ele, bd);
		//解析qualifier子元素
		parseQualifierElements(ele, bd);

		bd.setResource(this.readerContext.getResource());
		bd.setSource(extractSource(ele));

		return bd;
	}
	catch (ClassNotFoundException ex) {
		error("Bean class [" + className + "] not found", ele, ex);
	}
	catch (NoClassDefFoundError err) {
		error("Class that bean class [" + className + "] depends on not found", ele, err);
	}
	catch (Throwable ex) {
		error("Unexpected failure during bean definition parsing", ele, ex);
	}
	finally {
		this.parseState.pop();
	}

	return null;
}
```

### 创建用于承载属性的BeanDefinition
BeanDefinition是一个接口，有三种实现：RootBeanDefinition，ChildBeanDefinition，GenericBeanDefinition，都继承了AbstractBeanDefinition。

Spring通过BeanDefinition将配置文件中的bean配置信息转换为容器内部表示，并将这些BeanDefinition注册到BeanDefinitionRegistry中。BeanDefinitionRegistry主要是以map形式保存。

```
protected AbstractBeanDefinition createBeanDefinition(String className, String parentName)
			throws ClassNotFoundException {

	return BeanDefinitionReaderUtils.createBeanDefinition(
			parentName, className, this.readerContext.getBeanClassLoader());
}
```

```
public static AbstractBeanDefinition createBeanDefinition(
			String parentName, String className, ClassLoader classLoader) throws ClassNotFoundException {

	GenericBeanDefinition bd = new GenericBeanDefinition();
	bd.setParentName(parentName);
	if (className != null) {
		if (classLoader != null) {
			bd.setBeanClass(ClassUtils.forName(className, classLoader));
		}
		else {
			bd.setBeanClassName(className);
		}
	}
	return bd;
}
```

## 解析各种属性

```
public AbstractBeanDefinition parseBeanDefinitionAttributes(Element ele, String beanName,
			BeanDefinition containingBean, AbstractBeanDefinition bd) {
	//解析scope属性
	if (ele.hasAttribute(SCOPE_ATTRIBUTE)) {
		// Spring 2.x "scope" attribute
		bd.setScope(ele.getAttribute(SCOPE_ATTRIBUTE));
		//scope和singleton两个属性不能同时出现
		if (ele.hasAttribute(SINGLETON_ATTRIBUTE)) {
			error("Specify either 'scope' or 'singleton', not both", ele);
		}
	}
	//解析singleton属性
	else if (ele.hasAttribute(SINGLETON_ATTRIBUTE)) {
		// Spring 1.x "singleton" attribute
		bd.setScope(TRUE_VALUE.equals(ele.getAttribute(SINGLETON_ATTRIBUTE)) ?
				BeanDefinition.SCOPE_SINGLETON : BeanDefinition.SCOPE_PROTOTYPE);
	}
	else if (containingBean != null) {
		// Take default from containing bean in case of an inner bean definition.
		//在嵌入beanDefinition情况下且没有单独指定scope属性，则使用父类默认属性
		bd.setScope(containingBean.getScope());
	}

//解析abstract属性
	if (ele.hasAttribute(ABSTRACT_ATTRIBUTE)) {
		bd.setAbstract(TRUE_VALUE.equals(ele.getAttribute(ABSTRACT_ATTRIBUTE)));
	}
	//解析lazy-init属性
	String lazyInit = ele.getAttribute(LAZY_INIT_ATTRIBUTE);
	if (DEFAULT_VALUE.equals(lazyInit)) {
		lazyInit = this.defaults.getLazyInit();
	}
	//没有设置或者设置成其他的字符都会被设置为false
	bd.setLazyInit(TRUE_VALUE.equals(lazyInit));
	//解析autowire属性
	String autowire = ele.getAttribute(AUTOWIRE_ATTRIBUTE);
	bd.setAutowireMode(getAutowireMode(autowire));
	//解析dependency-check属性
	String dependencyCheck = ele.getAttribute(DEPENDENCY_CHECK_ATTRIBUTE);
	bd.setDependencyCheck(getDependencyCheck(dependencyCheck));
	//解析depends-on属性
	if (ele.hasAttribute(DEPENDS_ON_ATTRIBUTE)) {
		String dependsOn = ele.getAttribute(DEPENDS_ON_ATTRIBUTE);
		bd.setDependsOn(StringUtils.tokenizeToStringArray(dependsOn, MULTI_VALUE_ATTRIBUTE_DELIMITERS));
	}
	//解析autowire-candidate属性
	String autowireCandidate = ele.getAttribute(AUTOWIRE_CANDIDATE_ATTRIBUTE);
	if ("".equals(autowireCandidate) || DEFAULT_VALUE.equals(autowireCandidate)) {
		String candidatePattern = this.defaults.getAutowireCandidates();
		if (candidatePattern != null) {
			String[] patterns = StringUtils.commaDelimitedListToStringArray(candidatePattern);
			bd.setAutowireCandidate(PatternMatchUtils.simpleMatch(patterns, beanName));
		}
	}
	else {
		bd.setAutowireCandidate(TRUE_VALUE.equals(autowireCandidate));
	}
	
	//解析primary属性
	if (ele.hasAttribute(PRIMARY_ATTRIBUTE)) {
		bd.setPrimary(TRUE_VALUE.equals(ele.getAttribute(PRIMARY_ATTRIBUTE)));
	}
	
	//解析init-method属性
	if (ele.hasAttribute(INIT_METHOD_ATTRIBUTE)) {
		String initMethodName = ele.getAttribute(INIT_METHOD_ATTRIBUTE);
		if (!"".equals(initMethodName)) {
			bd.setInitMethodName(initMethodName);
		}
	}
	else {
		if (this.defaults.getInitMethod() != null) {
			bd.setInitMethodName(this.defaults.getInitMethod());
			bd.setEnforceInitMethod(false);
		}
	}
	
	//解析destory-method属性
	if (ele.hasAttribute(DESTROY_METHOD_ATTRIBUTE)) {
		String destroyMethodName = ele.getAttribute(DESTROY_METHOD_ATTRIBUTE);
		if (!"".equals(destroyMethodName)) {
			bd.setDestroyMethodName(destroyMethodName);
		}
	}
	else {
		if (this.defaults.getDestroyMethod() != null) {
			bd.setDestroyMethodName(this.defaults.getDestroyMethod());
			bd.setEnforceDestroyMethod(false);
		}
	}
	//解析factory-method属性
	if (ele.hasAttribute(FACTORY_METHOD_ATTRIBUTE)) {
		bd.setFactoryMethodName(ele.getAttribute(FACTORY_METHOD_ATTRIBUTE));
	}
	//解析factory-bean属性
	if (ele.hasAttribute(FACTORY_BEAN_ATTRIBUTE)) {
		bd.setFactoryBeanName(ele.getAttribute(FACTORY_BEAN_ATTRIBUTE));
	}

	return bd;
}
```

## 解析元数据 meta

```
<bean id="myBean" class="me.cxis.MyBean">
	<meta key="str" value="sadasd"/>
</bean>
```

```
public void parseMetaElements(Element ele, BeanMetadataAttributeAccessor attributeAccessor) {
//获取所有子元素
	NodeList nl = ele.getChildNodes();
	for (int i = 0; i < nl.getLength(); i++) {
		Node node = nl.item(i);
		if (isCandidateElement(node) && nodeNameEquals(node, META_ELEMENT)) {
			Element metaElement = (Element) node;
			String key = metaElement.getAttribute(KEY_ATTRIBUTE);
			String value = metaElement.getAttribute(VALUE_ATTRIBUTE);
			//使用key，value构造BeanMetadataAttribute
			BeanMetadataAttribute attribute = new BeanMetadataAttribute(key, value);
			attribute.setSource(extractSource(metaElement));
			attributeAccessor.addMetadataAttribute(attribute);
		}
	}
}
```
