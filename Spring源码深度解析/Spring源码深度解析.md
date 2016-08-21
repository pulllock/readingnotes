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
<<<<<<< HEAD

## 解析子元素lookup-method
获取器注入，把一个方法声明为返回某种类型的bean，但实际要返回的bean在配置文件里配置。

```
public void parseLookupOverrideSubElements(Element beanEle, MethodOverrides overrides) {
	NodeList nl = beanEle.getChildNodes();
	for (int i = 0; i < nl.getLength(); i++) {
		Node node = nl.item(i);
		//仅当在Spring默认bean的子元素下且为lookup-method时有效
		if (isCandidateElement(node) && nodeNameEquals(node, LOOKUP_METHOD_ELEMENT)) {
			Element ele = (Element) node;
			//要修饰的方法
			String methodName = ele.getAttribute(NAME_ATTRIBUTE);
			//要返回的bean
			String beanRef = ele.getAttribute(BEAN_ELEMENT);
			LookupOverride override = new LookupOverride(methodName, beanRef);
			override.setSource(extractSource(ele));
			overrides.addOverride(override);
		}
	}
}
```

## 解析子元素replaced-method
方法替换，可以在运行时用新的方法替换现有的方法，与lookup-method不同，replaced-method不但可以动态的替换返回实体bean，而且还能动态的更改原有方法的逻辑。

```
public void parseReplacedMethodSubElements(Element beanEle, MethodOverrides overrides) {
	NodeList nl = beanEle.getChildNodes();
	for (int i = 0; i < nl.getLength(); i++) {
		Node node = nl.item(i);
		//仅当在Spring默认bean的子元素下，并且为replaced-method时有效
		if (isCandidateElement(node) && nodeNameEquals(node, REPLACED_METHOD_ELEMENT)) {
			Element replacedMethodEle = (Element) node;
			//要替换的旧方法
			String name = replacedMethodEle.getAttribute(NAME_ATTRIBUTE);
			//新的替换方法
			String callback = replacedMethodEle.getAttribute(REPLACER_ATTRIBUTE);
			ReplaceOverride replaceOverride = new ReplaceOverride(name, callback);
			// Look for arg-type match elements.
			List<Element> argTypeEles = DomUtils.getChildElementsByTagName(replacedMethodEle, ARG_TYPE_ELEMENT);
			for (Element argTypeEle : argTypeEles) {
				//参数
				String match = argTypeEle.getAttribute(ARG_TYPE_MATCH_ATTRIBUTE);
				match = (StringUtils.hasText(match) ? match : DomUtils.getTextValue(argTypeEle));
				if (StringUtils.hasText(match)) {
					replaceOverride.addTypeIdentifier(match);
				}
			}
			replaceOverride.setSource(extractSource(replacedMethodEle));
			overrides.addOverride(replaceOverride);
		}
	}
}
```

## 解析子元素constructor-arg

```
public void parseConstructorArgElements(Element beanEle, BeanDefinition bd) {
	NodeList nl = beanEle.getChildNodes();
	for (int i = 0; i < nl.getLength(); i++) {
		Node node = nl.item(i);
		if (isCandidateElement(node) && nodeNameEquals(node, CONSTRUCTOR_ARG_ELEMENT)) {
			//具体的解析方法
			parseConstructorArgElement((Element) node, bd);
		}
	}
}
```


```
public void parseConstructorArgElement(Element ele, BeanDefinition bd) {
	//index属性
	String indexAttr = ele.getAttribute(INDEX_ATTRIBUTE);
	//type属性
	String typeAttr = ele.getAttribute(TYPE_ATTRIBUTE);
	//name属性
	String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
	if (StringUtils.hasLength(indexAttr)) {
		try {
			int index = Integer.parseInt(indexAttr);
			if (index < 0) {
				error("'index' cannot be lower than 0", ele);
			}
			else {
				try {
					this.parseState.push(new ConstructorArgumentEntry(index));
					Object value = parsePropertyValue(ele, bd, null);
					ConstructorArgumentValues.ValueHolder valueHolder = new ConstructorArgumentValues.ValueHolder(value);
					if (StringUtils.hasLength(typeAttr)) {
						valueHolder.setType(typeAttr);
					}
					if (StringUtils.hasLength(nameAttr)) {
						valueHolder.setName(nameAttr);
					}
					valueHolder.setSource(extractSource(ele));
					//不允许重复指定相同的参数
					if (bd.getConstructorArgumentValues().hasIndexedArgumentValue(index)) {
						error("Ambiguous constructor-arg entries for index " + index, ele);
					}
					else {
						bd.getConstructorArgumentValues().addIndexedArgumentValue(index, valueHolder);
					}
				}
				finally {
					this.parseState.pop();
				}
			}
		}
		catch (NumberFormatException ex) {
			error("Attribute 'index' of tag 'constructor-arg' must be an integer", ele);
		}
	}
	else {
		try {
			this.parseState.push(new ConstructorArgumentEntry());
			Object value = parsePropertyValue(ele, bd, null);
			ConstructorArgumentValues.ValueHolder valueHolder = new ConstructorArgumentValues.ValueHolder(value);
			if (StringUtils.hasLength(typeAttr)) {
				valueHolder.setType(typeAttr);
			}
			if (StringUtils.hasLength(nameAttr)) {
				valueHolder.setName(nameAttr);
			}
			valueHolder.setSource(extractSource(ele));
			bd.getConstructorArgumentValues().addGenericArgumentValue(valueHolder);
		}
		finally {
			this.parseState.pop();
		}
	}
}
```

* 如果配置中指定了index属性：
	1. 解析constructor-arg的子元素
	2. 使用ConstructorArgumentValues.ValueHolder类型来封装解析出来的元素。
	3. 将type、name、index属性一并封装在ConstructorArgumentValues.ValueHolder中并添加到当前BeanDefinition的indexedArgumentValues属性中。
* 如果没有指定index属性：
	1. 解析constructor-arg的子元素
	2. 使用ConstructorArgumentValues.ValueHolder类型来封装解析出来的元素。
	3. 将type、name、index属性一并封装在ConstructorArgumentValues.ValueHolder中并添加到当前BeanDefinition的genericArgumentValues属性中。

### parsePropertyVaule
解析构造函数配置中子元素的过程

```
public Object parsePropertyValue(Element ele, BeanDefinition bd, String propertyName) {
	String elementName = (propertyName != null) ?
					"<property> element for property '" + propertyName + "'" :
					"<constructor-arg> element";

	// Should only have one child element: ref, value, list, etc.
	NodeList nl = ele.getChildNodes();
	Element subElement = null;
	for (int i = 0; i < nl.getLength(); i++) {
		Node node = nl.item(i);
		if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT) &&
				!nodeNameEquals(node, META_ELEMENT)) {
			// Child element is what we're looking for.
			if (subElement != null) {
				error(elementName + " must not contain more than one sub-element", ele);
			}
			else {
				subElement = (Element) node;
			}
		}
	}
	//ref属性
	boolean hasRefAttribute = ele.hasAttribute(REF_ATTRIBUTE);
	//value属性
	boolean hasValueAttribute = ele.hasAttribute(VALUE_ATTRIBUTE);
	
	//1.有ref属性又有value属性
	//2.存在ref属性或者value属性且又有子元素
	if ((hasRefAttribute && hasValueAttribute) ||
			((hasRefAttribute || hasValueAttribute) && subElement != null)) {
		error(elementName +
				" is only allowed to contain either 'ref' attribute OR 'value' attribute OR sub-element", ele);
	}

	if (hasRefAttribute) {
		String refName = ele.getAttribute(REF_ATTRIBUTE);
		if (!StringUtils.hasText(refName)) {
			error(elementName + " contains empty 'ref' attribute", ele);
		}
		//使用RuntimeBeanReference封装对应的ref名称
		RuntimeBeanReference ref = new RuntimeBeanReference(refName);
		ref.setSource(extractSource(ele));
		return ref;
	}
	else if (hasValueAttribute) {
		//使用TypedStringValue封装value
		TypedStringValue valueHolder = new TypedStringValue(ele.getAttribute(VALUE_ATTRIBUTE));
		valueHolder.setSource(extractSource(ele));
		return valueHolder;
	}
	else if (subElement != null) {
		//解析子元素
		return parsePropertySubElement(subElement, bd);
	}
	else {
		// 没有ref，没有value，没有子元素
		error(elementName + " must specify a ref or value", ele);
		return null;
	}
}
```

## 解析子元素property

## 解析子元素qualifier
更多的使用的是注解形式。

# AbstractBeanDefinition属性

GenericBeanDefinition只是子类实现，而大部分通用属性都保存在了AbstractBeanDefinition中。

## 解析默认标签中的自定义标签元素

`bdHolder = delegate.decorateBeanDefinitionIfRequired(ele,bdHolder);`
当Spring中的bean使用的是默认标签配置，但是子元素使用了自定义配置，就使用此进行解析。

# 注册解析BeanDefinition

`BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());`


```
public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

	// 使用beanName做唯一标识
	String beanName = definitionHolder.getBeanName();
	registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

	// 如果有别名，注册别名
	String[] aliases = definitionHolder.getAliases();
	if (aliases != null) {
		for (String aliase : aliases) {
			registry.registerAlias(beanName, aliase);
		}
	}
}
```

1. 通过beanName注册BeanDefinition

	```
	public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		Assert.hasText(beanName, "Bean name must not be empty");
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");
	
		if (beanDefinition instanceof AbstractBeanDefinition) {
			try {
			//注册前最后一次校验，主要是对于AbstractBeanDefinition属性中的methodOverrides校验，校验methodOverrides是否与工程方法共存或者methodOverrides对应的方法根本不存在。
				((AbstractBeanDefinition) beanDefinition).validate();
			}
			catch (BeanDefinitionValidationException ex) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Validation of bean definition failed", ex);
			}
		}
	
		BeanDefinition oldBeanDefinition;
		//加锁，beanDefinitionMap是个全局变量
		synchronized (this.beanDefinitionMap) {
			//已经存在的beanDefinition
			oldBeanDefinition = this.beanDefinitionMap.get(beanName);
			//如果已经存在
			if (oldBeanDefinition != null) {
				//配置中配置了bean不允许覆盖，抛出异常
				if (!this.allowBeanDefinitionOverriding) {
					throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
							"Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
							"': There is already [" + oldBeanDefinition + "] bound.");
				}
				else {
					if (this.logger.isInfoEnabled()) {
						this.logger.info("Overriding bean definition for bean '" + beanName +
								"': replacing [" + oldBeanDefinition + "] with [" + beanDefinition + "]");
					}
				}
			}
			else {
				//记录beanName
				this.beanDefinitionNames.add(beanName);
				this.frozenBeanDefinitionNames = null;
			}
			//注册beanDefinition
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
		//重置beanName对应的缓存
		if (oldBeanDefinition != null || containsSingleton(beanName)) {
			resetBeanDefinition(beanName);
		}
	}
	```
	
	* 对AbstractBeanDefinition的methodOverrides属性进行校验。
	* beanName已经注册，如设置了不允许bean覆盖，则需要抛出异常，否则直接覆盖。
	* 加入map缓存。
	* 清除解析之前留下的对应beanName缓存。

2. 通过别名注册BeanDefinition
	
	```
	public void registerAlias(String name, String alias) {
		Assert.hasText(name, "'name' must not be empty");
		Assert.hasText(alias, "'alias' must not be empty");
		//alias与beanName相同，不记录alias并删除对应的alias
		if (alias.equals(name)) {
			this.aliasMap.remove(alias);
		}
		else {
			//不允许覆盖，抛异常
			if (!allowAliasOverriding()) {
				String registeredName = this.aliasMap.get(alias);
				if (registeredName != null && !registeredName.equals(name)) {
					throw new IllegalStateException("Cannot register alias '" + alias + "' for name '" +
							name + "': It is already registered for name '" + registeredName + "'.");
				}
			}
			//若A->B存在，再次出现A->B->C时抛异常
			checkForAliasCircle(name, alias);
			this.aliasMap.put(alias, name);
		}
	}
	```
	
	* alias与beanName相同，则不需要处理并删除原来的alias。
	* alias覆盖处理。
	* alias循环检查。
	* 注册alias。

# 通知监听器解析及注册完成

`getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));`

Spring没对此事件做任何处理，留作开发人员扩展用。

# alias标签的解析

```
protected void processAliasRegistration(Element ele) {
	//获取name
	String name = ele.getAttribute(NAME_ATTRIBUTE);
	//获取alias
	String alias = ele.getAttribute(ALIAS_ATTRIBUTE);
	boolean valid = true;
	if (!StringUtils.hasText(name)) {
		getReaderContext().error("Name must not be empty", ele);
		valid = false;
	}
	if (!StringUtils.hasText(alias)) {
		getReaderContext().error("Alias must not be empty", ele);
		valid = false;
	}
	if (valid) {
		try {
			//注册alias
			getReaderContext().getRegistry().registerAlias(name, alias);
		}
		catch (Exception ex) {
			getReaderContext().error("Failed to register alias '" + alias +
					"' for bean with name '" + name + "'", ele, ex);
		}
		//注册之后通知监听器做相应处理
		getReaderContext().fireAliasRegistered(name, alias, extractSource(ele));
	}
}
```

# import标签解析

```
protected void importBeanDefinitionResource(Element ele) {
	//resource属性
	String location = ele.getAttribute(RESOURCE_ATTRIBUTE);
	if (!StringUtils.hasText(location)) {
		getReaderContext().error("Resource location must not be empty", ele);
		return;
	}

	// 解析系统属性 如 "${user.dir}"
	location = getEnvironment().resolveRequiredPlaceholders(location);

	Set<Resource> actualResources = new LinkedHashSet<Resource>(4);

	// 判断location是绝对uri还是相对uri
	boolean absoluteLocation = false;
	try {
		absoluteLocation = ResourcePatternUtils.isUrl(location) || ResourceUtils.toURI(location).isAbsolute();
	}
	catch (URISyntaxException ex) {
		// cannot convert to an URI, considering the location relative
		// unless it is the well-known Spring prefix "classpath*:"
	}

	// 绝对地址，根据地址加载配置文件
	if (absoluteLocation) {
		try {
			int importCount = getReaderContext().getReader().loadBeanDefinitions(location, actualResources);
			if (logger.isDebugEnabled()) {
				logger.debug("Imported " + importCount + " bean definitions from URL location [" + location + "]");
			}
		}
		catch (BeanDefinitionStoreException ex) {
			getReaderContext().error(
					"Failed to import bean definitions from URL location [" + location + "]", ele, ex);
		}
	}
	else {
		// 相对地址，根据相对地址计算出绝对路径
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
		catch (IOException ex) {
			getReaderContext().error("Failed to resolve current resource location", ele, ex);
		}
		catch (BeanDefinitionStoreException ex) {
			getReaderContext().error("Failed to import bean definitions from relative location [" + location + "]",
					ele, ex);
		}
	}
	Resource[] actResArray = actualResources.toArray(new Resource[actualResources.size()]);
	getReaderContext().fireImportProcessed(location, actResArray, extractSource(ele));
}
```

1. 获取resource属性表示的路径。
2. 解析路径中的系统属性。
3. 判断location是绝对路径还是相对路径。
4. 如果是绝对路径递归调用bean解析过程，进行解析。
5. 如果是相对路径则计算出绝对路径进行解析。
6. 通知监听器，解析完成。



# 自定义标签的解析
parseCustomElement

扩展Spring自定义标签步骤：

1. 创建一个需要扩展的组件。
2. 定义一个XSD文件描述组件内容。
3. 创建一个实现BeanDefinitionParser接口，解析XSD文件中定义和组件定义。
4. 创建一个Handler，扩展自NamespaceHandlerSupport，将组件注册到Spring容器。
5. 编写Spring.handlers和Spring.schemas文件。

## 自定义标签解析

```
//containingBd为父类bean，顶层元素为null
public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
	//获取对应命名空间
	String namespaceUri = getNamespaceURI(ele);
	//根据命名空间找到对应的NamespaceHandler
	NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
	if (handler == null) {
		error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
		return null;
	}
	//调用自定义NamespaceHandler进行解析
	return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
}
```

## 获取标签命名空间

```
public String getNamespaceURI(Node node) {
	return node.getNamespaceURI();
}
```

调用jdk中的方法获取。

## 提取自定义标签处理器

```
public NamespaceHandler resolve(String namespaceUri) {
	//获取已经配置的handler映射
	Map<String, Object> handlerMappings = getHandlerMappings();
	//根据命名空间找到对应信息
	Object handlerOrClassName = handlerMappings.get(namespaceUri);
	if (handlerOrClassName == null) {
		return null;
	}
	else if (handlerOrClassName instanceof NamespaceHandler) {
		//已经做过解析的从缓存直接读取
		return (NamespaceHandler) handlerOrClassName;
	}
	else {
		//没有做过解析，返回类路径
		String className = (String) handlerOrClassName;
		try {
			//反射机制将类路径转化为类
			Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
			if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
				throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri +
						"] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
			}
			//初始化类
			NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
			//调用NamespaceHandler的初始化方法
			namespaceHandler.init();
			//记录在缓存中
			handlerMappings.put(namespaceUri, namespaceHandler);
			return namespaceHandler;
		}
		catch (ClassNotFoundException ex) {
			throw new FatalBeanException("NamespaceHandler class [" + className + "] for namespace [" +
					namespaceUri + "] not found", ex);
		}
		catch (LinkageError err) {
			throw new FatalBeanException("Invalid NamespaceHandler class [" + className + "] for namespace [" +
					namespaceUri + "]: problem with handler class file or dependent class", err);
		}
	}
}
```

getHandlerMappings：

```
private Map<String, Object> getHandlerMappings() {
	if (this.handlerMappings == null) {
		synchronized (this) {
			if (this.handlerMappings == null) {
				try {
					//this.handlerMappingsLocation已经初始化为META-INF/Spring.handlers
					Properties mappings =
							PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader);
					if (logger.isDebugEnabled()) {
						logger.debug("Loaded NamespaceHandler mappings: " + mappings);
					}
					Map<String, Object> handlerMappings = new ConcurrentHashMap<String, Object>(mappings.size());
					CollectionUtils.mergePropertiesIntoMap(mappings, handlerMappings);
					this.handlerMappings = handlerMappings;
				}
				catch (IOException ex) {
					throw new IllegalStateException(
							"Unable to load NamespaceHandler mappings from location [" + this.handlerMappingsLocation + "]", ex);
				}
			}
		}
	}
	return this.handlerMappings;
}
```

## 标签解析

`handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));`

```
public BeanDefinition parse(Element element, ParserContext parserContext) {
	return findParserForElement(element, parserContext).parse(element, parserContext);
}
```
寻找元素对应的解析器，调用解析器中的parse方法。

```
private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
	//获取元素名称
	String localName = parserContext.getDelegate().getLocalName(element);
	//找到对应解析器
	//注册的解析器
	BeanDefinitionParser parser = this.parsers.get(localName);
	if (parser == null) {
		parserContext.getReaderContext().fatal(
				"Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
	}
	return parser;
}
```

parse方法

```
public final BeanDefinition parse(Element element, ParserContext parserContext) {
	AbstractBeanDefinition definition = parseInternal(element, parserContext);
	if (definition != null && !parserContext.isNested()) {
		try {
			String id = resolveId(element, definition, parserContext);
			if (!StringUtils.hasText(id)) {
				parserContext.getReaderContext().error(
						"Id is required for element '" + parserContext.getDelegate().getLocalName(element)
								+ "' when used as a top-level tag", element);
			}
			String[] aliases = new String[0];
			String name = element.getAttribute(NAME_ATTRIBUTE);
			if (StringUtils.hasLength(name)) {
				aliases = StringUtils.trimArrayElements(StringUtils.commaDelimitedListToStringArray(name));
			}
			//将AbstractBeanDefinition转换为BeanDefinitionHolder
			BeanDefinitionHolder holder = new BeanDefinitionHolder(definition, id, aliases);
			registerBeanDefinition(holder, parserContext.getRegistry());
			if (shouldFireEvents()) {
				//通知监听器进行处理
				BeanComponentDefinition componentDefinition = new BeanComponentDefinition(holder);
				postProcessComponentDefinition(componentDefinition);
				parserContext.registerComponent(componentDefinition);
			}
		}
		catch (BeanDefinitionStoreException ex) {
			parserContext.getReaderContext().error(ex.getMessage(), element);
			return null;
		}
	}
	return definition;
}
```

```
protected final AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext) {
	BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition();
	String parentName = getParentName(element);
	if (parentName != null) {
		builder.getRawBeanDefinition().setParentName(parentName);
	}
	Class<?> beanClass = getBeanClass(element);
	if (beanClass != null) {
		builder.getRawBeanDefinition().setBeanClass(beanClass);
	}
	else {
		String beanClassName = getBeanClassName(element);
		if (beanClassName != null) {
			builder.getRawBeanDefinition().setBeanClassName(beanClassName);
		}
	}
	builder.getRawBeanDefinition().setSource(parserContext.extractSource(element));
	if (parserContext.isNested()) {
		// Inner bean definition must receive same scope as containing bean.
		builder.setScope(parserContext.getContainingBeanDefinition().getScope());
	}
	if (parserContext.isDefaultLazyInit()) {
		// Default-lazy-init applies to custom bean definitions as well.
		builder.setLazyInit(true);
	}
	doParse(element, parserContext, builder);
	return builder.getBeanDefinition();
}
```

# bean的加载

```
public Object getBean(String name) throws BeansException {
	return doGetBean(name, null, null, false);
}
```

```
protected <T> T doGetBean(
			final String name, final Class<T> requiredType, final Object[] args, boolean typeCheckOnly)
			throws BeansException {
	//提取对应的beanName
	final String beanName = transformedBeanName(name);
	Object bean;

	// Eagerly check singleton cache for manually registered singletons.
	Object sharedInstance = getSingleton(beanName);
	if (sharedInstance != null && args == null) {
		if (logger.isDebugEnabled()) {
			if (isSingletonCurrentlyInCreation(beanName)) {
				logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
						"' that is not fully initialized yet - a consequence of a circular reference");
			}
			else {
				logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
			}
		}
		bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
	}

	else {
		// Fail if we're already creating this bean instance:
		// We're assumably within a circular reference.
		if (isPrototypeCurrentlyInCreation(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}

		// Check if bean definition exists in this factory.
		BeanFactory parentBeanFactory = getParentBeanFactory();
		if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
			// Not found -> check parent.
			String nameToLookup = originalBeanName(name);
			if (args != null) {
				// Delegation to parent with explicit args.
				return (T) parentBeanFactory.getBean(nameToLookup, args);
			}
			else {
				// No args -> delegate to standard getBean method.
				return parentBeanFactory.getBean(nameToLookup, requiredType);
			}
		}

		if (!typeCheckOnly) {
			markBeanAsCreated(beanName);
		}

		try {
			final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
			checkMergedBeanDefinition(mbd, beanName, args);

			// Guarantee initialization of beans that the current bean depends on.
			String[] dependsOn = mbd.getDependsOn();
			if (dependsOn != null) {
				for (String dependsOnBean : dependsOn) {
					getBean(dependsOnBean);
					registerDependentBean(dependsOnBean, beanName);
				}
			}

			// Create bean instance.
			if (mbd.isSingleton()) {
				sharedInstance = getSingleton(beanName, new ObjectFactory<Object>() {
					public Object getObject() throws BeansException {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					}
				});
				bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
			}

			else if (mbd.isPrototype()) {
				// It's a prototype -> create a new instance.
				Object prototypeInstance = null;
				try {
					beforePrototypeCreation(beanName);
					prototypeInstance = createBean(beanName, mbd, args);
				}
				finally {
					afterPrototypeCreation(beanName);
				}
				bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
			}

			else {
				String scopeName = mbd.getScope();
				final Scope scope = this.scopes.get(scopeName);
				if (scope == null) {
					throw new IllegalStateException("No Scope registered for scope '" + scopeName + "'");
				}
				try {
					Object scopedInstance = scope.get(beanName, new ObjectFactory<Object>() {
						public Object getObject() throws BeansException {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						}
					});
					bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
				}
				catch (IllegalStateException ex) {
					throw new BeanCreationException(beanName,
							"Scope '" + scopeName + "' is not active for the current thread; " +
							"consider defining a scoped proxy for this bean if you intend to refer to it from a singleton",
							ex);
				}
			}
		}
		catch (BeansException ex) {
			cleanupAfterBeanCreationFailure(beanName);
			throw ex;
		}
	}

	// Check if required type matches the type of the actual bean instance.
	if (requiredType != null && bean != null && !requiredType.isAssignableFrom(bean.getClass())) {
		try {
			return getTypeConverter().convertIfNecessary(bean, requiredType);
		}
		catch (TypeMismatchException ex) {
			if (logger.isDebugEnabled()) {
				logger.debug("Failed to convert bean '" + name + "' to required type [" +
						ClassUtils.getQualifiedName(requiredType) + "]", ex);
			}
			throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
		}
	}
	return (T) bean;
}
```
